# 后端工程化每日训练 Day 08：线上日志设计与 TraceId 复盘

## 一、今日学习主题

今天学习的主题是：

> **线上日志设计与 TraceId 请求链路追踪。**

TraceId 的核心作用，是为一次请求或一次任务执行生成唯一的链路标识，并将它传递到 Gateway、业务服务、消息队列、消费者、第三方接口和数据库等组件中。

当线上出现问题时，可以通过一个 TraceId，快速查询本次执行产生的全部日志，而不需要在不同服务的海量日志中人工拼接调用过程。

---

## 二、TraceId 解决的问题

以 AI 异步任务执行平台为例，系统调用链可能是：

```text
前端
  ↓
Gateway
  ↓
Task Service
  ↓
RabbitMQ
  ↓
Worker
  ↓
GPT
  ↓
MySQL
```

用户反馈：

```text
TaskId：10086
任务一直处于 RUNNING 状态
```

如果系统没有 TraceId，只能依赖以下信息排查：

```text
TaskId
UserId
接口路径
日志时间
消息内容
```

排查过程通常是：

1. 查询数据库中的任务记录。
2. 根据 TaskId 搜索 Task Service 日志。
3. 检查 RabbitMQ 消息是否发送成功。
4. 搜索 Worker 消费日志。
5. 检查 GPT 调用日志。
6. 检查数据库结果保存和状态更新日志。

这种方式存在几个明显问题：

* 不同服务的日志分散在不同文件或不同服务器中。
* 同一时间可能存在大量并发请求。
* MQ 是异步执行，请求时间和消费时间可能相差较大。
* 同一个任务可能发生多次重试。
* 无法准确判断哪些日志属于同一次执行。
* 日志字段不完整时，甚至无法还原完整调用链。

因此，没有 TraceId 时，排查问题通常依赖人工推测，速度慢且容易判断错误。

---

## 三、TraceId 应该在哪里生成

我最开始不确定 TraceId 应该在 Gateway、Controller 还是 Worker 中生成，并猜测可能是在消费者中生成。

这个理解是不正确的。

正确原则是：

> **TraceId 应该在请求进入系统的最外层生成一次，后续组件只负责传递和复用。**

如果系统调用链是：

```text
前端
  ↓
Gateway
  ↓
Task Service
  ↓
RabbitMQ
  ↓
Worker
```

那么 TraceId 应优先在 Gateway 中处理。

Gateway 的逻辑应该是：

```text
检查请求头中是否已有 TraceId
        ↓
有：继续使用原来的 TraceId
        ↓
没有：生成新的 TraceId
```

伪代码：

```java
String traceId = request.getHeader("X-Trace-Id");

if (traceId == null || traceId.isBlank()) {
    traceId = UUID.randomUUID().toString();
}

MDC.put("traceId", traceId);
```

如果系统没有 Gateway，请求直接进入某个后端服务，则可以在该服务的 Filter 中统一生成。

不能在 Worker 中重新生成 TraceId。

如果前半段链路使用：

```text
TraceId=A
```

而 Worker 重新生成：

```text
TraceId=B
```

最终日志会变成：

```text
Gateway、Task Service：TraceId=A
Worker、GPT、数据库：TraceId=B
```

此时整条调用链被截断，搜索 TraceId=A 时无法找到消费者之后的日志。

因此需要记住：

```text
入口生成一次
后续只传递
没有才生成
不能随意重新生成
```

---

## 四、TraceId 与 TaskId 的区别

TraceId 和 TaskId 都可以用于日志排查，但它们不是同一个概念。

### TraceId

TraceId 表示：

> 一次具体的请求或执行链路。

例如：

```text
用户创建任务
  ↓
Gateway
  ↓
Task Service
  ↓
RabbitMQ
  ↓
Worker
```

这一次执行可以使用同一个 TraceId。

### TaskId

TaskId 表示：

> 一个长期存在的业务任务。

例如：

```text
TaskId=10086
```

这个任务可能经历：

```text
第一次执行
第二次重试
人工重新执行
延迟队列重新检查
```

这些执行可能拥有不同的 TraceId，但 TaskId 始终是 10086。

因此日志排查时，通常需要结合：

```text
TraceId
TaskId
MessageId
RetryCount
```

其中：

* TraceId：查询一次执行链路。
* TaskId：查询某个业务任务的全部历史。
* MessageId：判断消息是否重复投递。
* RetryCount：判断当前是第几次执行。

---

## 五、RabbitMQ 如何传递 TraceId

我最开始认为 TraceId 应该放在 RabbitMQ 的消息体中，因为日志需要完整信息。

这个理解不准确。

RabbitMQ 传递的不是完整日志，而是用于关联日志的 TraceId。

工程中通常采用：

```text
消息 Header：链路追踪和消息元数据
消息 Body：具体业务数据
```

例如：

```text
RabbitMQ Message

Headers:
  traceId: 8d41f7...
  messageId: msg-10001
  retryCount: 0

Body:
  taskId: 10086
  userId: 20001
  model: gpt-5
```

业务消息体：

```json
{
  "taskId": 10086,
  "userId": 20001,
  "model": "gpt-5"
}
```

消息 Header：

```text
traceId
messageId
retryCount
timestamp
contentType
```

生产者发送消息时，可以把 TraceId 写入消息 Header：

```java
String traceId = MDC.get("traceId");

rabbitTemplate.convertAndSend(
    "task.exchange",
    "task.execute",
    taskMessage,
    message -> {
        message.getMessageProperties()
                .setHeader("traceId", traceId);
        return message;
    }
);
```

消费者接收到消息后，从 Header 中读取 TraceId：

```java
String traceId = message.getMessageProperties()
        .getHeader("traceId");
```

完整传递链路为：

```text
Gateway 生成 TraceId
        ↓ HTTP Header
Task Service 获取 TraceId
        ↓ RabbitMQ Header
Worker 获取 TraceId
        ↓ MDC
消费者日志自动携带 TraceId
```

TraceId 放在消息体中并不是完全错误，但更常见、更清晰的设计是：

```text
业务数据放 Body
链路信息放 Header
```

---

## 六、MDC 是什么

MDC 的全称是：

```text
Mapped Diagnostic Context
```

可以把 MDC 理解为：

> 当前线程专用的日志上下文容器。

将 TraceId 放入 MDC：

```java
MDC.put("traceId", traceId);
```

之后当前线程产生的日志，就可以自动携带 TraceId：

```java
log.info("开始创建任务");
log.info("发送RabbitMQ消息");
log.info("更新任务状态");
```

日志输出：

```text
[traceId=8d41f7] 开始创建任务
[traceId=8d41f7] 发送RabbitMQ消息
[traceId=8d41f7] 更新任务状态
```

这样不需要在每一条日志中手动拼接 TraceId：

```java
log.info("traceId={}, 开始创建任务", traceId);
```

Logback 可以通过 `%X{traceId}` 自动读取 MDC 中的数据：

```xml
<pattern>
%d{yyyy-MM-dd HH:mm:ss.SSS}
[%thread]
[traceId=%X{traceId}]
%-5level
%logger{36}
- %msg%n
</pattern>
```

---

## 七、为什么必须清理 MDC

RabbitMQ 消费者、线程池和 Web 容器通常都会复用线程。

假设消费者线程：

```text
worker-1
```

先处理任务 A：

```text
TraceId=AAA
```

任务 A 执行完成后，如果没有清理 MDC，这个线程随后又被用于处理任务 B。

任务 B 如果没有正确设置 TraceId，可能继续使用任务 A 的 TraceId：

```text
任务 B 的日志
TraceId 却显示为 AAA
```

这会导致不同任务的日志被错误关联，甚至比没有 TraceId 更危险，因为错误的 TraceId 会直接误导排查方向。

正确写法：

```java
public void consume(Message message) {
    try {
        String traceId = message.getMessageProperties()
                .getHeader("traceId");

        MDC.put("traceId", traceId);

        log.info("开始消费任务");

        executeTask();

        log.info("任务消费完成");
    } finally {
        MDC.clear();
    }
}
```

必须放在 `finally` 中清理，因为即使业务代码抛出异常，`finally` 仍然会执行。

核心原因是：

> MDC 通常基于 ThreadLocal，而线程池中的线程会被复用，因此使用结束后必须清理。

---

## 八、任务一直 RUNNING 的排查顺序

场景：

```text
TaskId=10086
Status=RUNNING
```

推荐按照实际执行链路进行排查。

### 第一步：查询数据库任务记录

```sql
SELECT
    id,
    status,
    trace_id,
    retry_count,
    created_at,
    updated_at,
    error_message
FROM ai_task
WHERE id = 10086;
```

重点关注：

```text
当前状态
最后更新时间
TraceId
重试次数
错误信息
```

如果任务几个小时没有更新，说明执行过程可能已经卡住或异常中断。

---

### 第二步：通过 TaskId 和 TraceId 搜索日志

先搜索：

```text
taskId=10086
```

找到对应的 TraceId：

```text
traceId=8d41f7
```

再搜索完整 TraceId，查看链路执行到了哪一步：

```text
创建任务
发送 MQ
Worker 收到消息
更新状态为 RUNNING
开始调用 GPT
GPT 返回
保存结果
更新状态为 SUCCESS
```

最后一条成功日志通常可以帮助判断问题发生的位置。

---

### 第三步：检查 MQ 是否发送成功

检查 Task Service 日志：

```text
任务创建成功
准备发送 RabbitMQ 消息
RabbitMQ 消息发送成功
```

可能出现的问题：

```text
数据库任务创建成功
RabbitMQ 消息发送失败
```

此时数据库中存在任务记录，但消费者永远收不到任务。

如果使用 RabbitMQ Publisher Confirm，还应检查：

```text
Broker 是否确认消息
消息是否成功路由
消息是否进入正确队列
```

---

### 第四步：检查 RabbitMQ 队列状态

重点观察：

```text
Ready 消息数量
Unacked 消息数量
消费者数量
消息积压情况
```

如果 Ready 很多：

```text
消息仍在队列中，没有被消费者处理
```

可能原因：

```text
消费者宕机
消费者连接失败
消费能力不足
消费监听器未启动
```

如果 Unacked 很多：

```text
消息已被消费者获取，但一直没有 ACK
```

可能原因：

```text
GPT 调用卡住
数据库操作阻塞
线程死锁
消费者异常
没有设置调用超时
```

---

### 第五步：检查 Worker 消费日志

搜索：

```text
taskId=10086
```

查看是否存在：

```text
开始消费任务
状态更新为 RUNNING
开始调用 GPT
GPT 调用完成
结果保存成功
状态更新为 SUCCESS
```

如果日志停留在：

```text
开始调用 GPT
```

说明问题可能发生在模型调用阶段。

---

### 第六步：检查 GPT 调用

GPT 调用日志至少应记录：

```text
模型名称
请求开始时间
请求结束时间
执行耗时
响应状态码
第三方 RequestId
异常类型
重试次数
```

例如：

```text
taskId=10086
model=gpt-5
duration=120000ms
exception=SocketTimeoutException
retryCount=1
```

如果只记录：

```text
GPT调用失败
```

则很难判断是连接失败、读取超时、限流还是模型服务异常。

---

### 第七步：检查数据库结果保存

可能出现：

```text
GPT 调用成功
        ↓
保存任务结果失败
        ↓
任务状态没有更新
        ↓
任务一直保持 RUNNING
```

需要检查：

```text
数据库连接超时
事务回滚
字段长度超限
JSON 格式错误
唯一索引冲突
SQL 执行异常
```

---

### 第八步：检查状态更新逻辑

例如：

```java
task.setStatus(TaskStatus.SUCCESS);
int rows = taskMapper.updateById(task);
```

如果影响行数为 0，可能原因包括：

```text
任务记录不存在
乐观锁版本冲突
WHERE 条件不匹配
事务发生回滚
状态机不允许当前状态流转
```

状态更新后应检查影响行数：

```java
int rows = taskMapper.updateById(task);

if (rows != 1) {
    log.error(
        "任务状态更新失败, taskId={}, targetStatus={}, affectedRows={}",
        taskId,
        TaskStatus.SUCCESS,
        rows
    );
}
```

---

## 九、日志中应该记录哪些字段

我最开始想到的字段是：

```text
UserId
TaskId
```

这两个字段是必要的，但还不够。

AI 异步任务平台建议记录：

```text
TraceId
TaskId
UserId
MessageId
TaskStatus
RetryCount
ModelName
QueueName
ConsumerName
Duration
ErrorCode
Exception
```

示例：

```text
traceId=8d41f7
taskId=10086
userId=20001
messageId=msg-10001
status=RUNNING
retryCount=1
model=gpt-5
queue=ai.task.queue
consumer=worker-1
duration=3500ms
```

这些字段分别用于：

| 字段           | 作用            |
| ------------ | ------------- |
| TraceId      | 查询一次完整执行链路    |
| TaskId       | 查询业务任务的全部执行历史 |
| UserId       | 定位受到影响的用户     |
| MessageId    | 判断消息是否重复投递    |
| TaskStatus   | 查看任务状态流转      |
| RetryCount   | 判断当前重试次数      |
| ModelName    | 确认调用的 AI 模型   |
| QueueName    | 确认消息进入的队列     |
| ConsumerName | 定位具体消费者实例     |
| Duration     | 判断是否存在慢调用     |
| ErrorCode    | 对错误进行分类       |
| Exception    | 查看完整异常堆栈      |

---

## 十、有价值的日志应该记录什么

没有价值的日志：

```text
开始执行
执行结束
发生错误
```

这种日志无法帮助排查业务问题。

更合理的日志：

```java
log.info(
    "开始执行AI任务, taskId={}, userId={}, model={}, retryCount={}",
    taskId,
    userId,
    model,
    retryCount
);
```

```java
log.info(
    "AI任务执行成功, taskId={}, model={}, duration={}ms",
    taskId,
    model,
    duration
);
```

```java
log.error(
    "AI任务执行失败, taskId={}, model={}, retryCount={}",
    taskId,
    model,
    retryCount,
    exception
);
```

异常日志不能只打印：

```java
log.error(exception.getMessage());
```

因为这种方式只记录异常信息，不包含完整调用堆栈。

应该使用：

```java
log.error("调用GPT失败, taskId={}", taskId, exception);
```

这样才能定位具体代码位置和异常调用链。

---

## 十一、今日理解修正

### 原理解一

```text
TraceId 在后端生成，每个请求都生成一个 UUID。
```

修正后：

```text
TraceId 在系统最外层入口检查和生成。
上游已有 TraceId 时应继续复用。
没有 TraceId 时才生成新的 UUID。
```

### 原理解二

```text
TraceId 应该在消费者中生成。
```

修正后：

```text
消费者不能重新生成 TraceId。
消费者应该从 RabbitMQ Header 中读取生产者传递的 TraceId。
```

### 原理解三

```text
TraceId 应放在 RabbitMQ 消息体中。
```

修正后：

```text
TraceId 属于链路追踪元数据，通常放在消息 Header 中。
TaskId、UserId 等业务字段放在消息 Body 中。
```

### 原理解四

```text
不了解 MDC，也不知道为什么要清理。
```

修正后：

```text
MDC 是当前线程的日志上下文。
线程池会复用线程，如果不清理 MDC，可能导致不同任务的日志使用错误的 TraceId。
因此必须在 finally 中调用 MDC.clear()。
```

---

## 十二、完整排查流程总结

当 AI 异步任务一直处于 RUNNING 状态时，可以按照以下顺序排查：

```text
1. 查询数据库任务状态和最后更新时间
2. 根据 TaskId 找到 TraceId
3. 通过 TraceId 搜索完整调用链日志
4. 检查生产者是否成功发送 RabbitMQ 消息
5. 检查 RabbitMQ Ready 和 Unacked 数量
6. 检查 Worker 是否收到消息
7. 检查 GPT 调用是否超时或失败
8. 检查结果是否成功保存到数据库
9. 检查任务状态是否成功更新
10. 检查是否进入重试或死信队列
```

---

## 十三、今日核心结论

今天需要记住以下几个核心结论：

```text
TraceId 在系统入口生成一次，后续只传递。
```

```text
TraceId 用于查询一次执行链路，TaskId 用于查询业务任务历史。
```

```text
RabbitMQ 中，业务数据通常放 Body，TraceId 等链路信息通常放 Header。
```

```text
MDC 可以让当前线程的日志自动携带 TraceId。
```

```text
线程池会复用线程，因此 MDC 必须在 finally 中清理。
```

```text
线上日志不能只记录“开始、结束、失败”，还必须包含业务ID、状态、耗时和完整异常堆栈。
```

---

## 十四、今日复盘

今天的主要知识缺口不是 UUID 如何生成，而是：

* TraceId 应该由哪个组件生成。
* TraceId 如何跨 HTTP 和 RabbitMQ 继续传递。
* TraceId 与 TaskId 的区别。
* MDC 与线程池复用之间的关系。
* 如何按照完整调用链排查 RUNNING 任务。
* 如何设计真正有排查价值的业务日志。

通过今天的学习，我对线上日志的理解从：

```text
在代码中打印一些运行信息
```

提升到了：

```text
通过 TraceId、业务字段、消息字段、耗时和异常堆栈，还原一次完整任务执行过程
```

后续在 AI 异步任务执行平台中，可以逐步加入：

```text
Gateway / Filter 统一生成 TraceId
HTTP Header 传递 TraceId
RabbitMQ Header 传递 TraceId
Worker 消费时写入 MDC
finally 中清理 MDC
Logback 统一输出 TraceId
日志中补充 TaskId、MessageId、RetryCount 和 Duration
```

这套设计将直接提升项目的线上可排查性和工程完整度。
