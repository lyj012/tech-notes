# 后端工程化每日训练 Day 16：XXL-JOB 分布式任务调度

## 一、今天学习的知识点

今天学习的是：

**XXL-JOB 分布式任务调度**

对于已经接触过 XXL-JOB 的开发者，今天真正需要掌握的重点并不是：

```text
如何写 @XxlJob
如何配置 Cron
Admin 和 Executor 分别是什么
```

而是：

```text
为什么调度任务可能重复执行
为什么执行失败不等于业务没有成功
为什么 XXL-JOB 不能保证业务只执行一次
如何通过幂等、重试、补偿和日志保证任务可靠
```

一句话理解：

> XXL-JOB 负责在指定时间选择 Executor 并触发任务，但无法从根本上保证业务只执行一次。生产环境必须默认任务可能被重复触发，并由业务幂等、状态机、重试记录和补偿机制保证最终正确。

---

## 二、为什么不能只使用 `@Scheduled`

Spring 提供了：

```java
@Scheduled(cron = "0 */5 * * * ?")
public void syncData() {
    // 执行业务逻辑
}
```

单实例部署时，这种方式通常可以正常工作。

但生产环境可能部署多个实例：

```text
Pod A
Pod B
Pod C
Pod D
```

每个实例都是独立的 JVM，每个 JVM 都会维护自己的定时任务。

到了凌晨 2 点：

```text
Pod A 执行一次
Pod B 执行一次
Pod C 执行一次
Pod D 执行一次
```

如果四个实例扫描的是同一张任务表，就可能出现：

```text
同一批异常任务被扫描四次
↓
同一批任务被重新投递四次
↓
MQ 中出现重复消息
↓
Worker 重复执行 AI、OCR 或 PDF 处理
```

因此，在多实例环境中，不能只依赖本地 JVM 的 `@Scheduled` 来保证任务只执行一次。

常见解决方向包括：

```text
使用 XXL-JOB 等分布式调度系统
使用数据库或 Redis 分布式锁
使用数据库状态抢占
业务逻辑保持幂等
```

其中最不能省略的是：

> 即使调度层已经做了防重，业务层仍然必须保证幂等。

---

## 三、XXL-JOB 的基本架构

XXL-JOB 的核心角色包括：

```text
XXL-JOB Admin
        ↓
    调度任务
        ↓
Executor
        ↓
执行 Java 业务代码
```

### Admin 负责

```text
维护任务配置
保存 Cron 表达式
选择 Executor
发起调度请求
记录调度日志
查看执行结果
支持手动触发
支持失败重试
```

### Executor 负责

```text
注册执行器
接收调度请求
找到对应 JobHandler
执行 Java 方法
记录执行日志
返回执行结果
```

最小示例：

```java
@XxlJob("scanTimeoutTask")
public void scanTimeoutTask() {
    // 扫描超时任务并进行补偿
}
```

简化执行流程：

```text
到达 Cron 时间
↓
Admin 生成一次调度记录
↓
Admin 请求某个 Executor
↓
Executor 执行 JobHandler
↓
Executor 返回执行结果
↓
Admin 更新调度日志
```

---

## 四、XXL-JOB 能不能保证任务只执行一次

不能。

XXL-JOB 可以降低多实例环境下无控制重复执行的概率，但不能提供绝对的业务“只执行一次”。

原因在于一次任务执行包含多个步骤：

```text
Admin 发起调度
↓
Executor 收到请求
↓
业务代码开始执行
↓
业务数据修改成功
↓
Executor 返回执行结果
↓
Admin 收到结果并记录成功
```

其中任意环节发生超时、网络异常或进程重启，都可能出现：

```text
业务已经成功
但调度系统不知道业务已经成功
```

这时调度系统只能看到：

```text
没有收到明确的成功结果
```

它不能直接判断：

```text
Executor 完全没有执行
```

还是：

```text
Executor 已经执行成功，只是结果没有成功返回
```

因此，分布式任务调度通常只能做到：

```text
尽量可靠地触发
+
记录执行结果
+
失败后进行重试或补偿
```

最终业务正确性仍然依赖：

```text
幂等
状态机
唯一约束
条件更新
业务执行记录
人工补偿
```

---

## 五、最典型的重复执行过程

假设凌晨 2 点执行 AI 异常任务补偿：

```text
扫描 RUNNING 超过 30 分钟的任务
↓
将异常任务重新投递到 MQ
```

第一次执行：

```text
02:00:00 Admin 发起调度
↓
02:00:01 Executor 开始执行
↓
02:00:05 查询出 Task-A、Task-B、Task-C
↓
02:00:08 三个任务已经成功投递 MQ
↓
02:00:09 Executor 向 Admin 返回执行成功
```

如果最后一步发生网络超时：

```text
Task-A、Task-B、Task-C 已经进入 MQ
↓
Executor 的成功回调没有到达 Admin
↓
Admin 认为本次执行失败
↓
触发失败重试
↓
Task-A、Task-B、Task-C 再次进入 MQ
```

核心问题是：

> Admin 没有收到成功结果，不等于 Executor 没有成功完成业务。

这是分布式系统中常见的不确定状态：

```text
调用方不知道被调用方是否已经成功
```

因此，失败重试可能带来重复执行。

---

## 六、为什么会发生重复执行

重复执行不只有失败重试一种原因。

### 1. 失败重试

第一次任务已经修改数据库或发送 MQ，但执行结果没有被 Admin 正确确认。

随后：

```text
Admin 判断失败
↓
触发重试
↓
业务再次执行
```

### 2. 人工重复触发

运维或开发人员看到任务状态异常，点击了手动执行。

但第一次任务可能仍然在运行：

```text
Cron 自动触发一次
+
人工又触发一次
=
同一任务执行两次
```

### 3. 外部接口重复触发

如果系统允许通过 API 触发任务，调用方可能因为超时重试而重复调用。

```text
调用方发起触发请求
↓
请求实际成功
↓
调用方没有收到响应
↓
调用方再次请求
```

### 4. 调度周期小于执行时间

例如任务每分钟执行一次，但每次需要五分钟：

```text
02:00 第一次开始
02:01 第二次开始
02:02 第三次开始
```

如果没有合适的阻塞策略，就可能出现多个执行实例同时扫描同一批数据。

### 5. 配置了重复任务

可能存在两个不同的 XXL-JOB 任务：

```text
任务 101：凌晨 2 点扫描异常任务
任务 208：凌晨 2 点扫描异常任务
```

两者调用的是同一个 JobHandler，导致业务执行两次。

### 6. 分片广播使用错误

分片广播会让多个 Executor 分别执行。

正确思路应该是：

```text
Executor 0 处理自己的分片
Executor 1 处理自己的分片
Executor 2 处理自己的分片
Executor 3 处理自己的分片
```

如果每个 Executor 都忽略分片参数，直接扫描全量数据：

```text
Executor 0 扫描全部任务
Executor 1 扫描全部任务
Executor 2 扫描全部任务
Executor 3 扫描全部任务
```

同一批任务就可能被处理四次。

### 7. Executor 重启或任务超时

任务执行过程中：

```text
业务已经处理一部分
↓
Executor 进程重启
↓
任务被标记失败
↓
后续重新执行
```

如果任务没有记录每条业务数据的处理进度，重试时可能重新处理全部数据。

---

## 七、调度重复和业务重复不是一个问题

需要区分两个层次。

### 第一层：为什么任务被触发两次

可能原因包括：

```text
失败重试
人工触发
API 重复触发
重复任务配置
任务重叠
分片使用错误
网络或回调异常
```

这是：

> 调度层或触发层问题。

### 第二层：为什么触发两次后造成了数据错误

例如：

```text
库存恢复两次
MQ 发送两次
通知发送两次
AI 任务执行两次
用户扣费两次
```

这是：

> 业务幂等问题。

正确判断应该是：

> 重复执行的直接原因可能出现在调度层，但重复执行造成业务错误，说明业务层没有做好幂等保护。

生产环境不能假设：

```text
调度系统永远只触发一次
```

而应该假设：

```text
任何任务都可能被重复触发
```

---

## 八、真实业务场景：AI 异步任务补偿

任务表中存在：

```text
task_id = 10001
status = RUNNING
updated_at = 01:10
```

凌晨 2 点执行补偿任务：

```text
当前时间 - updated_at > 30 分钟
```

系统认为 Worker 可能已经异常退出，需要将任务重新入队。

最简单的错误实现：

```java
@XxlJob("scanTimeoutTask")
public void scanTimeoutTask() {
    List<Task> tasks = taskMapper.selectTimeoutRunningTasks();

    for (Task task : tasks) {
        mqProducer.send(task.getId());
    }
}
```

如果 JobHandler 执行两次：

```text
第一次查询 Task-10001
↓
发送一次 MQ

第二次又查询 Task-10001
↓
再次发送 MQ
```

结果：

```text
同一个 taskId 出现两条消息
```

如果消费者也没有幂等：

```text
Worker A 执行一次
Worker B 又执行一次
```

可能造成：

```text
重复调用付费模型
重复调用 OCR
重复生成文件
重复上传对象存储
重复写入任务结果
重复发送完成通知
```

---

## 九、如何让补偿任务保持幂等

不能只写：

```text
查询超时任务
↓
直接发送 MQ
```

更稳妥的设计是先进行状态抢占。

### 1. 增加补偿状态

例如：

```text
NONE
RETRY_WAIT
ENQUEUE_PROCESSING
ENQUEUED
ENQUEUE_FAILED
```

任务进入补偿范围时：

```text
RUNNING 超时
+
retry_status = RETRY_WAIT
```

### 2. 使用条件更新抢占任务

```sql
UPDATE ai_task
SET retry_status = 'ENQUEUE_PROCESSING',
    retry_version = retry_version + 1,
    retry_started_at = NOW()
WHERE id = ?
  AND status = 'RUNNING'
  AND retry_status = 'RETRY_WAIT';
```

只有更新行数为 1 的执行线程，才获得本次重新投递资格。

```java
int updated = taskMapper.claimForRetry(taskId);

if (updated == 0) {
    // 已经被其他调度实例或线程处理
    return;
}
```

这样即使任务扫描执行两次：

```text
第一次条件更新成功
第二次条件更新失败
```

真正发送 MQ 的只有第一次。

### 3. 发送消息时携带幂等键

例如：

```text
taskId = 10001
retryVersion = 3
idempotencyKey = 10001:3
```

消息内容：

```json
{
  "taskId": 10001,
  "retryVersion": 3,
  "idempotencyKey": "10001:3"
}
```

### 4. 消费端再次校验幂等

不能因为生产端已经防重，就省略消费端幂等。

消费者可以建立执行记录：

```sql
CREATE UNIQUE INDEX uk_task_retry
ON task_execution(task_id, retry_version);
```

消费时先尝试写入：

```text
task_id = 10001
retry_version = 3
```

如果唯一键冲突，说明同一轮补偿消息已经被处理，应直接跳过。

### 5. 使用状态机限制非法执行

例如任务只有处于指定状态时才能被重新领取：

```sql
UPDATE ai_task
SET status = 'RUNNING',
    worker_id = ?,
    lease_expire_time = ?
WHERE id = ?
  AND status IN ('WAITING', 'RETRY_WAIT');
```

如果任务已经进入：

```text
SUCCESS
```

旧的补偿消息不能再把它重新改为：

```text
RUNNING
```

---

## 十、数据库更新成功和 MQ 发送成功之间仍然有问题

假设先更新数据库：

```text
retry_status = ENQUEUE_PROCESSING
```

然后发送 MQ。

可能发生：

```text
数据库更新成功
↓
MQ 发送失败
↓
任务永久停留在 ENQUEUE_PROCESSING
```

反过来，如果先发 MQ：

```text
MQ 发送成功
↓
数据库更新失败
↓
下一次扫描又会重新发送
```

这属于数据库和 MQ 之间的一致性问题。

常见解决方向包括：

### 方案一：失败状态和再次扫描

发送失败时记录：

```text
retry_status = ENQUEUE_FAILED
last_error
next_retry_time
retry_count
```

后续定时任务只扫描：

```text
ENQUEUE_FAILED
且 next_retry_time <= 当前时间
```

### 方案二：本地消息表

在同一个数据库事务中写入：

```text
业务状态变化
+
待发送消息记录
```

独立任务再扫描本地消息表，将消息发送到 MQ。

### 方案三：事务消息或 Outbox

通过事务消息或 Outbox 模式，将数据库状态和消息发送之间的不一致窗口显式管理起来。

无论采用哪种方案，都不能认为：

```text
调用 mq.send() 没有抛异常
=
消息和业务状态绝对一致
```

---

## 十一、失败重试应该怎么设计

重试不是简单地：

```text
失败以后再执行三次
```

首先需要区分异常类型。

### 1. 可重试异常

常见包括：

```text
短暂网络超时
MQ 暂时不可用
数据库连接暂时失败
下游服务短暂限流
临时 DNS 或连接异常
```

这类问题可能在短时间后自动恢复，可以进行有限重试。

### 2. 不可重试异常

常见包括：

```text
参数错误
业务状态非法
数据缺失
权限配置错误
SQL 语法错误
代码空指针
业务规则冲突
```

盲目重试通常不会自动恢复，只会重复制造失败日志和系统压力。

应该：

```text
立即记录失败
↓
触发告警
↓
修复数据、配置或代码
↓
人工确认后再补偿
```

### 3. 使用递增退避

错误示例：

```text
第 1 次：1 分钟后
第 2 次：5 分钟后
第 3 次：3 分钟后
```

更合理的示例：

```text
第 1 次：1 分钟后
第 2 次：5 分钟后
第 3 次：15 分钟后
第 4 次：30 分钟后
```

还可以加入随机抖动：

```text
15 分钟 ± 随机 30 秒
```

避免大量失败任务在同一时刻集中重试。

### 4. 限制最大重试次数

例如：

```text
max_retry_count = 4
```

超过后进入：

```text
MANUAL_REQUIRED
```

并触发告警。

---

## 十二、不要轻易重试整个扫描任务

假设扫描出 100 个任务：

```text
90 个发送成功
10 个发送失败
```

如果整个 XXL-JOB 任务直接失败并重试：

```text
100 个任务全部重新处理
```

这会让已经成功的 90 个任务再次进入 MQ。

更合理的方式是：

```text
每条业务任务独立记录处理状态
↓
成功的任务标记 ENQUEUED
↓
失败的任务标记 ENQUEUE_FAILED
↓
下一轮只重试失败的任务
```

调度任务本身可以成功结束，但在业务表中保留部分失败记录：

```text
本次扫描 100 条
成功 90 条
失败 10 条
```

然后：

```text
10 条失败任务进入后续重试或人工补偿
```

这比“一个任务失败导致整个批次全部重跑”更稳定。

---

## 十三、人工补偿机制怎么设计

超过自动重试次数后，不能只打印一条日志。

至少需要建立失败任务记录：

```text
业务任务 ID
任务类型
失败阶段
失败原因
异常堆栈
已重试次数
最近重试时间
下一次重试时间
XXL-JOB jobId
XXL-JOB logId
Executor 地址
MQ messageId
当前业务状态
```

后台管理页面可以提供：

```text
查看失败详情
重新执行
修改参数后重试
跳过任务
标记已处理
终止后续重试
```

人工操作必须记录审计日志：

```text
操作人
操作时间
操作原因
操作前状态
操作后状态
关联任务 ID
```

否则出现二次事故时，很难判断：

```text
是系统自动重试
还是人工再次触发
```

---

## 十四、阻塞策略解决什么问题

当上一次任务还没有结束，下一次调度又到达时，需要决定如何处理。

常见思路包括：

### 单机串行

```text
上一次任务未完成
↓
后续任务排队等待
```

适合：

```text
任务不能并发
每次任务都不能丢
```

风险：

```text
如果执行速度长期低于调度速度
等待队列会越来越长
```

### 丢弃后续调度

```text
上一次任务未完成
↓
新的调度直接丢弃
```

适合：

```text
只关心最新一次扫描
允许跳过中间调度
```

风险：

```text
如果上一次长期卡死
后续任务可能一直无法执行
```

### 覆盖之前调度

```text
终止旧任务
↓
执行新任务
```

风险较高，因为 Java 线程被终止时，业务可能已经执行了一部分：

```text
数据库已经更新
MQ 还没有发送
```

或者：

```text
部分数据处理成功
部分数据未处理
```

因此，不能把阻塞策略当成业务一致性的替代品。

---

## 十五、大批量扫描任务的风险

定时任务经常被写成：

```sql
SELECT *
FROM ai_task
WHERE status = 'RUNNING'
  AND updated_at < NOW() - INTERVAL 30 MINUTE;
```

如果一次查询几十万条数据，可能造成：

```text
慢 SQL
数据库连接长时间占用
大量对象进入 JVM 内存
长事务
锁竞争
MQ 瞬时流量过大
下游 Worker 被压垮
```

更合理的方式包括：

```text
分页扫描
限制单批数量
使用合适索引
按主键游标推进
控制发送速率
分片处理
记录扫描进度
```

例如：

```sql
SELECT id
FROM ai_task
WHERE status = 'RUNNING'
  AND updated_at < ?
  AND id > ?
ORDER BY id
LIMIT 500;
```

索引可以根据查询条件设计为：

```text
(status, updated_at, id)
```

实际索引仍需结合数据分布、执行计划和查询频率验证。

---

## 十六、日志和监控如何定位重复执行

你之前想到的是：

```text
traceId
```

`traceId` 可以串联一次调用链：

```text
XXL-JOB 调度
↓
Executor
↓
数据库查询
↓
MQ 发送
```

但只靠 `traceId` 不够。

因为两次独立调度通常会产生两个不同的 `traceId`。

排查重复执行时，建议记录：

```text
jobId
logId
traceId
triggerType
attemptNo
batchId
executorAddress
taskId
retryVersion
messageId
startTime
endTime
result
```

### 字段作用

#### `jobId`

表示是哪一个 XXL-JOB 任务。

#### `logId`

表示某一次具体调度记录。

同一个 `jobId` 可以有很多不同的 `logId`。

#### `traceId`

串联这一次调度内部的调用链。

#### `triggerType`

表示触发来源，例如：

```text
CRON
RETRY
MANUAL
API
```

#### `batchId`

表示本次业务扫描批次。

例如：

```text
20260723-020000
```

#### `taskId`

表示具体业务任务。

#### `messageId`

表示发送到 MQ 的具体消息。

---

## 十七、推荐日志格式

第一次 Cron 调度：

```text
jobId=16
logId=893421
traceId=ab12cd34
triggerType=CRON
attemptNo=0
batchId=20260723-020000
executor=10.0.1.12:9999
taskId=AI_10086
retryVersion=3
action=REQUEUE
result=SUCCESS
messageId=MQ_889900
```

后续失败重试：

```text
jobId=16
logId=893447
traceId=ef56gh78
triggerType=RETRY
attemptNo=1
batchId=20260723-020000
executor=10.0.1.15:9999
taskId=AI_10086
retryVersion=3
action=REQUEUE
result=SUCCESS
messageId=MQ_889955
```

根据日志可以看到：

```text
同一个 taskId
同一个 batchId
同一个 retryVersion
出现两个不同 logId
第一次 triggerType=CRON
第二次 triggerType=RETRY
```

由此可以判断：

> 第一次 Cron 执行以后，又发生了一次失败重试，并且同一轮业务补偿缺少幂等保护。

---

## 十八、监控指标怎么设计

关键调度任务至少应监控：

```text
调度成功次数
调度失败次数
执行耗时
超时次数
自动重试次数
人工触发次数
单批扫描数量
单批成功数量
单批失败数量
重复处理拦截次数
连续失败次数
待人工补偿数量
```

对于 AI 异步任务，还可以增加：

```text
超时 RUNNING 任务数量
重新入队数量
重复消息数量
Worker 重复领取次数
失败任务积压量
补偿任务恢复成功率
```

告警示例：

```text
同一任务连续失败 3 次
↓
立即告警

待人工补偿任务超过 20 条
↓
立即告警

单批扫描数量突然增长 10 倍
↓
立即告警

重复处理拦截次数异常升高
↓
排查是否存在重复调度
```

---

## 十九、最小代码示例

### 1. 调度入口

```java
@XxlJob("scanTimeoutTask")
public void scanTimeoutTask() {
    String batchId = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyyMMdd-HHmmss"));

    long lastId = 0L;

    while (true) {
        List<Long> taskIds =
                taskMapper.selectTimeoutTaskIds(lastId, 500);

        if (taskIds.isEmpty()) {
            break;
        }

        for (Long taskId : taskIds) {
            retryService.tryRequeue(taskId, batchId);
            lastId = taskId;
        }
    }
}
```

### 2. 条件更新抢占

```java
public void tryRequeue(Long taskId, String batchId) {
    int updated = taskMapper.claimRetryTask(taskId);

    if (updated == 0) {
        log.info(
                "skip duplicated retry, taskId={}, batchId={}",
                taskId,
                batchId
        );
        return;
    }

    try {
        int retryVersion = taskMapper.selectRetryVersion(taskId);

        String idempotencyKey =
                taskId + ":" + retryVersion;

        String messageId = mqProducer.send(
                taskId,
                retryVersion,
                idempotencyKey
        );

        taskMapper.markEnqueued(taskId, messageId);

        log.info(
                "requeue success, taskId={}, retryVersion={}, " +
                "batchId={}, messageId={}",
                taskId,
                retryVersion,
                batchId,
                messageId
        );
    } catch (RetryableException exception) {
        taskMapper.markRetryFailed(
                taskId,
                exception.getMessage()
        );

        throw exception;
    } catch (Exception exception) {
        taskMapper.markManualRequired(
                taskId,
                exception.getMessage()
        );

        log.error(
                "requeue failed and requires manual handling, " +
                "taskId={}, batchId={}",
                taskId,
                batchId,
                exception
        );
    }
}
```

代码重点不是可以直接复制，而是体现：

```text
先抢占
再发送
记录重试版本
携带幂等键
区分可重试和不可重试异常
保留失败状态
记录批次和消息 ID
```

---

## 二十、今天思考题复盘

### 问题一：为什么会发生重复执行

可能原因包括：

```text
第一次业务已经成功，但执行结果回调失败
Admin 判断失败后触发重试
人工或 API 又触发了一次
任务周期小于执行时间，出现重叠
配置了两个相同任务
分片广播时每个 Executor 都扫描了全量数据
```

最典型的情况是：

```text
业务实际成功
↓
成功结果没有被 Admin 确认
↓
Admin 触发重试
↓
业务再次执行
```

### 问题二：是调度问题、部署问题还是业务幂等问题

需要分层判断：

```text
为什么被执行两次
→ 调度、部署或触发层问题

为什么执行两次后产生错误
→ 业务幂等问题
```

因此不能只选一个。

完整结论是：

> 重复触发的原因可能在调度层，但重复入队并产生业务影响，说明业务层没有做好幂等。

### 问题三：四个实例仅使用 `@Scheduled` 会发生什么

通常会发生：

```text
四个 JVM 各执行一次
↓
同一个任务总共执行四次
```

如果四个实例扫描同一张表，可能同时查到同一批异常任务并分别发送 MQ。

### 问题四：失败重试和人工补偿怎么设计

可重试异常：

```text
网络超时
MQ 暂时不可用
数据库连接异常
下游服务临时限流
```

采用：

```text
有限次数
+
递增退避
+
随机抖动
```

不可重试异常：

```text
参数错误
状态非法
代码错误
配置错误
数据缺失
```

处理方式：

```text
直接记录失败
↓
触发告警
↓
进入人工补偿列表
```

人工补偿后台需要支持：

```text
查看失败原因
重新执行
修改参数后重试
跳过
终止后续重试
记录操作审计
```

### 问题五：如何定位是哪次调度导致重复执行

通过以下字段串联：

```text
jobId
logId
traceId
triggerType
attemptNo
batchId
executorAddress
taskId
retryVersion
messageId
```

重点判断：

```text
同一个 taskId 是否出现多个 logId
触发类型是 CRON、RETRY、MANUAL 还是 API
是否来自不同 Executor
是否属于同一个 batchId
是否生成了多个 messageId
```

---

## 二十一、面试表达

可以这样回答：

> 在多实例部署中，我不会只依赖 Spring 的 `@Scheduled`，因为每个 JVM 都会独立执行定时任务，可能造成重复扫描和重复处理。我会使用 XXL-JOB 统一管理任务调度、Cron、执行日志和手动触发。
>
> 但 XXL-JOB 本身不能保证业务绝对只执行一次。例如 Executor 已经完成数据库更新或 MQ 投递，但执行结果回调超时，Admin 可能认为任务失败并触发重试。因此我会默认任务存在重复执行的可能，并在业务层通过状态抢占、条件更新、唯一幂等键和状态机保证重复执行不会产生重复结果。
>
> 对于 AI 异步任务补偿，我会让调度任务分页扫描超时任务，每条任务独立记录补偿状态和重试版本，只重试失败的数据，而不是让整个批次全部重跑。可重试异常采用递增退避，参数错误、状态非法和代码问题则直接告警并进入人工补偿。
>
> 排查重复执行时，我会结合 XXL-JOB 的 jobId、logId、触发类型和 Executor 地址，以及业务 taskId、batchId、retryVersion、traceId 和 MQ messageId，判断是 Cron、自动重试、人工触发、分片错误还是业务重复处理。

---

## 二十二、今天的最终结论

今天最核心的结论不是：

```text
XXL-JOB 可以防止多实例定时任务重复执行
```

而是：

```text
XXL-JOB 负责统一调度
不等于业务只执行一次
```

完整的可靠任务设计需要同时考虑：

```text
调度是否重复触发
任务是否重叠执行
失败重试是否会扩大重复范围
数据库和 MQ 是否存在一致性窗口
每条业务数据是否有独立状态
任务是否支持幂等
异常是否正确分类
是否有人工补偿入口
日志能否定位具体调度批次
失败是否能够及时告警
```

今天需要记住的核心结论是：

> 调度系统负责“什么时候、让谁执行”，业务系统负责“即使重复执行也不能出错”。失败重试只能提高任务被执行的概率，幂等、状态机、执行记录和补偿机制才负责保证最终业务正确。

能力等级：**中级后端能力**。

其中 XXL-JOB 的基础接入属于常规开发能力，真正体现中级能力的是：

```text
理解重复执行的故障窗口
区分调度重复和业务重复
设计细粒度重试
保证任务幂等
处理数据库与 MQ 一致性
建立日志、监控和人工补偿闭环
```
