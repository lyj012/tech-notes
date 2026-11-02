# 后端工程化每日训练 Day 28：事务边界设计、长事务与异步任务最终一致性

## 一、今天学习的知识点

今天学习的是：

**数据库事务边界（Transaction Boundary）设计、长事务的资源风险，以及 AI、对象存储、Kafka 和补偿机制组合下的最终一致性。**

一句话理解：

> 事务不是越大越安全，而是应该尽可能短，只保护必须原子提交的数据库操作；AI 调用、HTTP 请求、对象存储上传、MQ 网络发送等不可控耗时操作，应放在数据库事务之外。

今天重点解决以下问题：

```text
为什么 AI 调用不操作数据库，却会导致数据库连接池耗尽
为什么不能把整个业务流程都放进 @Transactional
哪些操作应该保留在事务内
哪些操作应该移到事务外
AI 已经成功，但数据库更新失败时如何补偿
Outbox 本地消息表能解决什么问题，不能解决什么问题
Kafka、对象存储、任务状态机、补偿和幂等如何组合
Spring 同类方法调用为什么可能导致事务不生效
```

---

## 二、问题背景：事务为什么不能覆盖整个业务流程

假设一个 AI 平台的任务执行流程是：

```text
保存任务
↓
调用 AI，平均耗时 20 秒
↓
上传结果到 OSS
↓
更新数据库
```

最直接的实现可能是：

```java
@Transactional
public void executeTask(Long taskId) {
    taskMapper.insert(...);

    AiResult result = aiClient.call(...);

    String objectKey = ossClient.upload(result);

    taskMapper.updateResult(taskId, objectKey);
}
```

监控发现：

```text
平均事务时间：22 秒
数据库连接池等待数量持续增加
接口响应越来越慢
```

这类问题不一定是 SQL 本身慢，而可能是事务边界设计错误。

事务真正持续的时间不是：

```text
SQL 执行时间
```

而是：

```text
进入事务
↓
执行第一条数据库操作并获取连接
↓
执行 AI、OSS、HTTP 等后续逻辑
↓
方法结束
↓
事务提交或回滚
↓
连接归还连接池
```

只要事务没有结束，已经获取的数据库连接和相关事务资源就可能一直被占用。

---

## 三、为什么 AI 调用会导致数据库连接池压力增加

今天最需要修正的理解是：

```text
AI 调用本身占用数据库连接
```

这个说法不准确。

AI 调用通常通过 HTTP 请求访问外部模型服务，它本身不需要使用数据库连接。

真正的因果链是：

```text
进入 @Transactional 方法
↓
保存任务，执行第一条 SQL
↓
事务获取数据库连接
↓
开始调用 AI，等待 20 秒
↓
事务还没有提交
↓
数据库连接无法归还连接池
```

因此更准确的表达是：

> AI 调用没有直接使用数据库连接，但它位于数据库事务中，把事务持续时间拉长，导致事务已经持有的连接长期不能归还。

假设连接池最大连接数为 20：

```text
20 个并发任务
×
每个事务持有连接 20 秒
```

可能出现：

```text
20 个连接全部被长事务占用
↓
第 21 个请求无法获取连接
↓
请求进入等待队列
↓
等待数量持续增加
↓
达到 connectionTimeout
↓
接口报错
```

同时，长事务还可能长期占用：

```text
数据库连接
行锁或间隙锁
Undo Log
事务快照
JVM 请求线程
下游连接
```

最终可能形成：

```text
连接池耗尽
锁等待增加
死锁概率上升
Undo Log 膨胀
接口超时
系统吞吐下降
```

---

## 四、事务边界应该如何划分

事务的目标是：

```text
保证必须一起成功或一起失败的数据库操作
```

事务不应该被用来包住整个业务流程。

合理的事务边界通常是：

```text
一组必须原子提交的数据库操作
↓
快速提交
```

而不是：

```text
数据库操作
↓
网络调用
↓
文件上传
↓
AI 推理
↓
MQ 发送
↓
数据库操作
↓
最后提交
```

### 应该保留在事务内的操作

```text
插入任务记录
更新订单状态
更新任务状态
保存结果元数据
插入 Outbox 本地消息
扣减或冻结数据库中的业务额度
需要同时成功或同时失败的多表写入
```

### 应该移到事务外的操作

```text
AI 模型调用
HTTP / RPC 第三方接口调用
OSS / OBS / S3 文件上传和下载
Kafka / RabbitMQ 网络发送
Thread.sleep()
长时间 CPU 计算
等待外部系统返回结果
```

判断标准不是“这个操作和业务是否有关”，而是：

```text
它是否属于当前数据库能够原子控制的操作
它的耗时是否可控
失败后是否能够由数据库回滚
```

Kafka 发送虽然和业务高度相关，但 MySQL 本地事务不能直接回滚已经发送到 Kafka 的消息。

OSS 上传虽然和文件记录高度相关，但数据库回滚不能自动删除已经上传的对象。

---

## 五、AI 任务应该拆成两个短事务

推荐流程如下：

```text
用户提交任务
↓
事务①：
插入 ai_task
状态 = PENDING
插入 outbox_event
↓
提交事务
↓
Outbox Publisher 异步发送 Kafka
↓
Worker 消费消息
↓
幂等校验
↓
短事务更新状态为 PROCESSING
↓
提交事务
↓
调用 AI
↓
上传结果到 OSS
↓
事务②：
保存 objectKey、结果摘要等元数据
更新状态为 SUCCESS
↓
提交事务
```

这里不存在一个持续 20 秒以上的大事务。

### 事务①负责什么

```text
任务记录一定成功创建
任务事件一定进入 Outbox 待发送状态
```

业务表和 Outbox 表位于同一个数据库中，因此可以由同一个本地事务原子提交。

### 事务②负责什么

```text
结果元数据写入
任务状态更新为 SUCCESS
必要的业务统计更新
```

### 事务外负责什么

```text
Kafka 实际网络发送
AI 模型调用
OSS 上传
重试等待
```

这样即使 AI 调用耗时 20 秒，数据库连接也不会因为等待 AI 而被长期占用。

---

## 六、推荐的任务状态机

异步任务不能只依赖方法是否抛异常，还需要显式状态。

可以设计：

```text
PENDING
↓
QUEUED
↓
PROCESSING
↓
SUCCESS
```

失败分支：

```text
PROCESSING
↓
RETRY_WAIT
↓
PROCESSING
```

超过重试上限：

```text
PROCESSING / RETRY_WAIT
↓
FAILED
```

状态含义：

```text
PENDING
→ 任务已创建，等待发送消息

QUEUED
→ 任务消息已进入 Kafka

PROCESSING
→ Worker 已领取并开始执行

RETRY_WAIT
→ 当前执行失败，等待下一次重试

SUCCESS
→ AI 结果和文件元数据已经可靠保存

FAILED
→ 超过自动重试次数，需要告警或人工处理
```

状态更新应带条件，避免重复消费或并发 Worker 同时执行。

例如：

```sql
UPDATE ai_task
SET status = 'PROCESSING',
    start_time = NOW(),
    worker_id = ?
WHERE id = ?
  AND status IN ('PENDING', 'QUEUED', 'RETRY_WAIT');
```

只有返回影响行数为 1 的 Worker 才能继续执行。

---

## 七、AI 成功但数据库更新失败，应该如何处理

场景如下：

```text
AI 调用成功
↓
OSS 上传成功
↓
更新数据库失败
```

此时可能出现：

```text
OSS 中已经存在结果文件
数据库中的任务仍然是 PROCESSING
用户看不到结果
```

这不是 Outbox 本地消息表直接解决的问题。

Outbox 主要解决的是：

```text
数据库业务操作已经提交
但 MQ 消息没有可靠发送
```

而当前问题是：

```text
外部调用已经成功
但后续数据库落库失败
```

更适合使用：

```text
稳定的业务幂等键
结果执行记录
数据库重试
补偿扫描
状态机
告警
```

### 方案一：数据库更新有限重试

OSS 上传成功后已经得到：

```text
objectKey
```

更新数据库发生短暂网络异常时，可以针对数据库更新进行有限次数重试。

```text
第一次更新失败
↓
等待短暂退避
↓
再次更新
↓
成功则标记 SUCCESS
```

不能无限重试，否则会长期占用 Worker。

### 方案二：保存可恢复的执行记录

Worker 应尽量保留：

```text
taskId
requestId
AI provider requestId
objectKey
resultHash
执行阶段
最后错误
重试次数
```

如果主任务表更新失败，补偿程序可以根据这些信息恢复。

### 方案三：补偿任务扫描异常状态

使用 XXL-JOB、Spring Scheduler 或独立补偿服务定期扫描：

```text
status = PROCESSING
并且 update_time 超过合理执行时长
```

补偿程序进一步检查：

```text
是否已有 OSS objectKey
是否存在执行记录
AI 提供方是否可以通过 requestId 查询结果
是否可以安全重试当前阶段
```

然后执行：

```text
补写数据库结果
重新进入重试队列
标记 FAILED
触发告警
```

### 方案四：无法确认外部结果时不要盲目重复调用

如果 AI 服务已经执行成功，但应用在接收结果后宕机，而 AI 服务又不支持通过 requestId 查询历史结果，系统可能无法确认之前是否已经成功。

这时不能宣称能够实现严格的“恰好执行一次”。

更现实的目标是：

```text
至少一次执行
+
业务幂等
+
可查询的 requestId
+
结果去重
+
人工兜底
```

---

## 八、幂等应该设计在哪些位置

补偿和重试会带来重复执行，因此必须配套幂等。

### 1. 任务创建幂等

客户端提交任务时携带：

```text
requestId
```

数据库建立唯一索引：

```sql
UNIQUE KEY uk_request_id (request_id)
```

相同请求重复提交时返回原任务，而不是创建多个任务。

### 2. Kafka 消费幂等

Kafka 可能因为：

```text
消费成功但 Offset 提交失败
Consumer Rebalance
Worker 宕机
消息重试
```

导致同一任务消息被重复消费。

Worker 应使用：

```text
taskId
或
messageId
```

进行幂等判断，并结合条件更新抢占任务。

### 3. OSS 上传幂等

Object Key 可以由 taskId 和固定结果类型生成：

```text
ai-result/{taskId}/final.json
ai-result/{taskId}/final.pdf
```

重复上传时覆盖同一个业务对象，或者先检查对象是否已经存在。

不能每次重试都生成一个随机 UUID，否则可能产生大量孤儿文件。

### 4. 数据库结果更新幂等

```sql
UPDATE ai_task
SET status = 'SUCCESS',
    object_key = ?,
    result_hash = ?,
    finish_time = NOW()
WHERE id = ?
  AND status IN ('PROCESSING', 'RETRY_WAIT');
```

如果任务已经是 SUCCESS，重复消息不应再次改变业务结果。

### 5. 额度扣减和计费幂等

AI 平台通常涉及次数、Token 或余额扣减。

需要为每次扣费建立唯一业务流水：

```text
billing_record.task_id 唯一
```

避免任务重试导致重复扣费。

---

## 九、Kafka、对象存储和补偿机制如何组合

这种场景适合组合 Day 13、Day 26、Day 7 和 Day 27 的知识重新设计。

完整架构如下：

```text
Client
↓
Task API
↓
本地事务：
插入 ai_task(PENDING)
插入 outbox_event(NEW)
↓
提交
↓
Outbox Publisher
↓
Kafka
↓
Worker Consumer
↓
任务幂等与状态抢占
↓
更新 PROCESSING 后立即提交
↓
调用 AI
↓
上传 OSS
↓
短事务更新 SUCCESS + objectKey
↓
Kafka Offset 提交
```

异常处理：

```text
Kafka 消息发送失败
→ Outbox Publisher 重试

Kafka 重复投递
→ Worker 幂等处理

AI 调用失败
→ 退避重试，超过上限标记 FAILED

OSS 上传失败
→ 只重试上传阶段，不重复调用 AI

OSS 成功、数据库失败
→ 使用 taskId + objectKey 补写数据库

PROCESSING 长时间不结束
→ 补偿任务扫描并恢复

补偿仍失败
→ 告警和人工处理
```

各组件的职责必须清晰：

```text
Kafka
→ 削峰、异步解耦、任务投递

OSS / OBS / S3
→ 保存 AI 结果文件本体

MySQL
→ 保存任务状态、文件元数据和执行记录

Outbox
→ 保证任务创建后，Kafka 消息最终可发送

补偿任务
→ 发现并恢复跨系统操作中断后的不一致状态

幂等
→ 保证重试和重复消息不会产生重复业务结果
```

---

## 十、Outbox 能解决什么，不能解决什么

### Outbox 能解决

```text
业务数据提交成功
但应用在发送 MQ 前宕机
```

通过同一数据库事务写入：

```text
业务表
+
Outbox 表
```

即使应用随后宕机，Outbox Sender 仍可以在恢复后继续发送消息。

### Outbox 不能直接解决

```text
AI 已经调用成功，但数据库更新失败
OSS 上传成功，但数据库记录失败
消费者处理外部系统成功，但本地状态更新失败
```

这些问题需要：

```text
阶段记录
业务状态机
幂等键
重试
补偿扫描
外部 requestId
必要时人工处理
```

因此不能把“本地消息表”当作所有跨系统一致性问题的统一答案。

---

## 十一、事务里直接发送 MQ 的风险

错误设计：

```java
@Transactional
public void createTask() {
    taskMapper.insert(...);
    kafkaTemplate.send(...);
}
```

可能出现：

```text
Kafka 已经收到消息
↓
后续数据库操作报错
↓
MySQL 事务回滚
↓
Worker 消费到一个数据库中不存在的任务
```

反过来也可能出现：

```text
MySQL 提交成功
↓
应用在发送 Kafka 前宕机
↓
任务永远没有进入消息队列
```

推荐流程：

```text
本地事务：
插入任务
插入 Outbox
↓
提交
↓
Outbox Publisher 在事务外发送 Kafka
```

Kafka 发送成功后再把 Outbox 状态更新为 SENT。

即使 ACK 丢失导致重复发送，消费者也通过 messageId 或 taskId 幂等处理。

---

## 十二、文件上传事务边界

错误流程：

```text
开始数据库事务
↓
上传 OSS，耗时 10 秒
↓
保存文件记录
↓
提交事务
```

问题是：

```text
数据库事务为了等待 OSS 持续 10 秒
```

推荐流程：

```text
生成稳定 objectKey
↓
上传 OSS
↓
上传成功
↓
开启短事务
↓
保存文件元数据
↓
提交
```

如果 OSS 成功但数据库失败：

```text
记录 objectKey
↓
重试数据库写入
↓
仍失败则进入补偿任务
↓
确认无法恢复后删除孤儿对象
```

需要注意：

> 先上传 OSS 再写数据库，虽然缩短了数据库事务，但仍然存在 OSS 成功、数据库失败的不一致窗口，因此必须设计补偿，不能只调整执行顺序。

---

## 十三、支付回调事务边界

错误设计：

```java
@Transactional
public void handleCallback() {
    updateOrder();
    callPointsService();
    sendSms();
    sendMq();
}
```

任何外部接口超时都会把订单事务拉长。

推荐设计：

```text
支付回调
↓
本地事务：
校验支付幂等
更新订单为 PAID
插入 Outbox(OrderPaid)
↓
提交
↓
发送 MQ
↓
积分消费者发放积分
↓
短信消费者异步发送短信
```

数据库事务只保证：

```text
订单状态更新
和
支付成功事件进入待发送状态
```

积分、短信等后续能力通过异步消息实现最终一致性。

---

## 十四、Spring `@Transactional` 的代理陷阱

下面的代码看起来把数据库保存拆成了独立方法：

```java
@Service
public class FileService {

    public void upload() {
        ossClient.upload(...);
        saveMetadata();
    }

    @Transactional
    public void saveMetadata() {
        fileMapper.insert(...);
    }
}
```

但 `upload()` 和 `saveMetadata()` 位于同一个类中。

内部调用：

```text
this.saveMetadata()
```

通常不会经过 Spring AOP 代理，因此 `@Transactional` 可能不会生效。

更稳妥的方式是拆分职责：

```java
@Service
@RequiredArgsConstructor
public class FileApplicationService {

    private final ObjectStorageService objectStorageService;
    private final FileMetadataService fileMetadataService;

    public Long upload(FileData file) {
        UploadResult result = objectStorageService.upload(file);
        return fileMetadataService.save(result);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class FileMetadataService {

    private final FileMapper fileMapper;

    @Transactional
    public Long save(UploadResult result) {
        FileEntity entity = FileEntity.from(result);
        fileMapper.insert(entity);
        return entity.getId();
    }
}
```

这样外部调用会经过 Spring 代理，事务边界也更清晰。

另一种方式是使用 `TransactionTemplate` 显式定义短事务，但不应为了调用代理而进行难以理解的自我注入。

---

## 十五、常见错误

### 1. 认为 `@Transactional` 能控制所有调用

普通数据库事务只能控制对应数据库事务管理器管理的资源。

它不能自动回滚：

```text
已经发送的 Kafka 消息
已经上传的 OSS 文件
已经成功的 AI 请求
已经发送的短信
```

### 2. 认为事务越大越安全

事务越大通常意味着：

```text
锁竞争更严重
连接占用更久
死锁概率更高
回滚成本更高
系统吞吐更低
```

### 3. 在 Controller 上直接加事务

Controller 可能包含：

```text
参数解析
权限校验
文件处理
HTTP 调用
响应组装
```

如果整个 Controller 方法处于事务中，事务范围通常过大，而且职责不清晰。

事务边界一般应放在 Service 层明确的业务用例中。

### 4. 把所有失败都重新执行完整流程

如果流程已经执行到：

```text
AI 成功
OSS 失败
```

应该优先重试 OSS 上传，而不是重新调用一次 AI。

否则会增加：

```text
模型成本
执行时间
重复结果
重复扣费风险
```

### 5. 只设计重试，不设计幂等

任何重试机制都可能导致重复执行。

没有幂等的重试只是把偶发失败转换为重复数据和重复扣费。

### 6. 只设计补偿，不设计可观测性

补偿任务至少需要记录：

```text
taskId
当前状态
执行阶段
retryCount
nextRetryTime
lastError
workerId
requestId
objectKey
开始和更新时间
```

否则系统即使知道任务失败，也无法判断应该从哪个阶段恢复。

---

## 十六、最小代码示例

### 1. 创建任务与 Outbox

```java
@Service
@RequiredArgsConstructor
public class TaskCommandService {

    private final AiTaskMapper aiTaskMapper;
    private final OutboxEventMapper outboxEventMapper;

    @Transactional
    public Long createTask(CreateTaskCommand command) {
        AiTask task = AiTask.pending(
            command.requestId(),
            command.prompt()
        );
        aiTaskMapper.insert(task);

        OutboxEvent event = OutboxEvent.newEvent(
            UUID.randomUUID().toString(),
            "AI_TASK_CREATED",
            task.getId().toString()
        );
        outboxEventMapper.insert(event);

        return task.getId();
    }
}
```

### 2. Worker 在事务外执行 AI 和 OSS

```java
@Service
@RequiredArgsConstructor
public class AiTaskWorker {

    private final TaskStateService taskStateService;
    private final AiClient aiClient;
    private final ObjectStorageService objectStorageService;

    public void execute(Long taskId) {
        boolean acquired = taskStateService.markProcessing(taskId);
        if (!acquired) {
            return;
        }

        try {
            AiResult result = aiClient.call(taskId);

            String objectKey = "ai-result/" + taskId + "/final.json";
            objectStorageService.upload(objectKey, result.content());

            taskStateService.markSuccess(
                taskId,
                objectKey,
                result.hash()
            );
        } catch (RetryableException e) {
            taskStateService.scheduleRetry(taskId, e.getMessage());
        } catch (Exception e) {
            taskStateService.markFailed(taskId, e.getMessage());
        }
    }
}
```

### 3. 状态更新使用短事务

```java
@Service
@RequiredArgsConstructor
public class TaskStateService {

    private final AiTaskMapper aiTaskMapper;

    @Transactional
    public boolean markProcessing(Long taskId) {
        return aiTaskMapper.markProcessing(taskId) == 1;
    }

    @Transactional
    public void markSuccess(
            Long taskId,
            String objectKey,
            String resultHash) {

        int updated = aiTaskMapper.markSuccess(
            taskId,
            objectKey,
            resultHash
        );

        if (updated == 0 && !aiTaskMapper.isSuccess(taskId)) {
            throw new IllegalStateException("任务结果更新失败");
        }
    }

    @Transactional
    public void scheduleRetry(Long taskId, String error) {
        aiTaskMapper.scheduleRetry(taskId, error);
    }

    @Transactional
    public void markFailed(Long taskId, String error) {
        aiTaskMapper.markFailed(taskId, error);
    }
}
```

核心结构是：

```text
短事务更新状态
↓
事务外执行耗时操作
↓
短事务保存结果
```

---

## 十七、今天思考题的回答与修正

### 问题一：为什么 AI 调用导致连接池压力增加

原回答方向：

```text
AI 调用会占用数据库连接，调用次数多后连接池耗尽
```

修正后：

> AI 调用本身不直接使用数据库连接，但保存任务后，当前事务已经获取连接。由于 AI 调用位于事务内部并持续约 20 秒，连接会一直持有到整个事务提交，导致并发任务增多时连接无法及时归还，最终出现连接池等待和超时。

### 问题二：哪些步骤放在事务内

事务内：

```text
保存任务
更新任务状态
保存结果元数据
插入 Outbox 消息
需要原子提交的数据库多表操作
```

事务外：

```text
AI 调用
OSS 上传
Kafka 网络发送
其他第三方接口
```

### 问题三：AI 成功但数据库失败如何保证最终一致性

不能只回答“增加本地消息表”。

本地消息表主要保证数据库与 MQ 之间的最终一致性。

当前场景应使用：

```text
稳定 taskId 和 requestId
固定 OSS objectKey
执行阶段记录
数据库有限重试
状态机
补偿扫描
幂等更新
告警和人工兜底
```

### 问题四：是否适合结合 Kafka、对象存储和补偿机制

适合。

推荐职责划分：

```text
Kafka 负责异步解耦和削峰
对象存储负责保存 AI 结果文件
MySQL 负责任务状态和元数据
Outbox 负责消息可靠投递
补偿机制负责恢复跨系统中断
幂等负责抵消重复投递和重复重试
```

---

## 十八、面试表达

可以这样表达：

> 在异步任务或支付等业务中，我会尽量缩小事务边界，只把必须保证原子性的数据库操作放入事务。AI 调用、对象存储上传、HTTP 请求和 MQ 网络发送都属于耗时且不可由数据库回滚的外部操作，应放在事务之外。
>
> 例如 AI 任务提交时，我会在一个短事务中保存任务并写入 Outbox，再异步投递 Kafka。Worker 消费后通过条件更新抢占任务，在事务外调用 AI 和上传 OSS，最后使用另一个短事务保存 objectKey 和任务结果。对于 AI 或 OSS 已成功但数据库更新失败的情况，会通过 taskId、固定 objectKey、执行记录、有限重试和补偿扫描恢复，并在任务创建、Kafka 消费、文件上传、状态更新和计费环节设计幂等。

---

## 十九、今天的核心结论

```text
AI 调用不直接占用数据库连接
但事务中的 AI 调用会延长连接持有时间
```

```text
事务只保护必须原子提交的数据库操作
不负责包住整个业务流程
```

```text
数据库事务、Kafka、OSS、AI 是不同故障边界
不能依赖一个 @Transactional 同时回滚
```

```text
Outbox 解决数据库提交与 MQ 发送之间的可靠性
不能替代所有跨系统补偿
```

```text
最终一致性 = 状态机 + 重试 + 补偿 + 幂等 + 可观测性
```

```text
短事务提高吞吐
明确状态帮助恢复
幂等保证重复执行不产生重复业务结果
```

事务边界设计的本质不是减少代码，而是明确：

> 哪些操作必须原子提交，哪些外部操作允许异步完成，以及每一个跨系统失败窗口应该由什么机制恢复。
