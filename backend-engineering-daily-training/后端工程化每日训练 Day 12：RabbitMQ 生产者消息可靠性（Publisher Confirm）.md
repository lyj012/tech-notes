# 后端工程化每日训练 Day 12：RabbitMQ 生产者消息可靠性（Publisher Confirm）

## 一、今天学习的知识点

今天学习的是：

**RabbitMQ 生产者消息可靠性（Publisher Confirm）**

Publisher Confirm 主要解决：

> 生产者尝试发送消息之后，如何确认 RabbitMQ Broker 是否真正接收到了这条消息。

它是 RabbitMQ 消息可靠性链路中的第一道保障。

完整的消息链路可以先简化为：

```text
Producer
↓
RabbitMQ Broker
↓
Consumer
```

但是在真实系统中，仅仅执行了发送方法，并不能证明 Broker 已经成功接收消息。

例如：

```java
rabbitTemplate.convertAndSend(...);
```

这个方法正常返回，只能说明：

```text
生产者执行了发送动作
```

不能直接等价于：

```text
RabbitMQ Broker 已经成功接管消息
```

所以需要 Publisher Confirm，让 Broker 把接收结果返回给 Producer。

---

## 二、为什么正常消息流程中还需要返回 Confirm

我一开始的疑惑是：

```text
正常流程不是：

Producer
↓
RabbitMQ
↓
Consumer

为什么 RabbitMQ 收到消息以后，
还要反过来给 Producer 返回一个 Confirm？
```

这里的关键是：

**消息传递方向和结果反馈方向不是一回事。**

业务消息的传递方向仍然是：

```text
Producer
↓
RabbitMQ
↓
Consumer
```

而 Publisher Confirm 是一条反向的结果通知：

```text
Producer
↓ publish
RabbitMQ
↑ confirm
Producer
```

Confirm 并不是把原来的业务消息重新发送回生产者。

它只是告诉生产者：

```text
ACK
→ Broker 已经接收了这次发布

NACK
→ Broker 没有成功接收或处理这次发布

超时 / 没有 Confirm
→ Producer 暂时无法确定最终结果
```

所以完整理解应该是：

```text
业务通道：
Producer → RabbitMQ → Consumer

可靠性反馈通道：
Producer ← RabbitMQ
```

---

## 三、发送了不等于对方收到了

Publisher Confirm 存在的根本原因是：

> 分布式系统中，发送方执行了发送动作，不代表接收方一定已经收到。

例如：

```text
Producer 调用发送方法
↓
消息进入客户端缓冲区
↓
发送方法返回
↓
网络突然中断
↓
Broker 实际没有收到消息
```

还可能发生：

```text
RabbitMQ 重启
TCP 连接中断
网络闪断
Broker 内部异常
Broker 拒绝接收消息
```

如果没有 Confirm，Producer 最多只能知道：

```text
我尝试发送过
```

有了 Confirm，Producer 才能进一步知道：

```text
RabbitMQ 是否已经接管这条消息
```

这类似于快递签收回执。

```text
寄件人
↓
快递系统
↓
收件人
```

包裹的方向没有改变，但是快递系统仍然会给寄件人返回：

```text
已揽收
已签收
```

这些状态不是把包裹送回来，而是让发送方知道责任是否已经成功转移。

---

## 四、Publisher Confirm 的责任边界

消息可靠性可以理解为责任逐段转移。

### 第一段：Producer 把消息交给 RabbitMQ

```text
Producer
↓
RabbitMQ Broker
```

Publisher Confirm 负责确认：

```text
Broker 是否已经接收这次发布
```

收到 Confirm ACK 后，Producer 才能认为：

```text
消息已经进入 RabbitMQ 负责的范围
```

### 第二段：RabbitMQ 把消息交给 Consumer

```text
RabbitMQ Broker
↓
Consumer
```

Consumer ACK 负责确认：

```text
消费者是否已经成功处理消息
```

因此完整链路是：

```text
Producer
  │
  │ publish
  ▼
RabbitMQ Broker
  │
  ├── Publisher Confirm ACK / NACK → Producer
  │
  │ deliver
  ▼
Consumer
  │
  └── Consumer ACK / NACK → RabbitMQ
```

两种 ACK 完全不同。

| 机制 | 确认方向 | 解决的问题 |
| --- | --- | --- |
| Publisher Confirm | Broker → Producer | 消息是否成功到达 Broker |
| Consumer ACK | Consumer → Broker | 消息是否被消费者成功处理 |

因此：

```text
Publisher Confirm ACK
≠
消费者执行成功
```

它只说明：

```text
Broker 已经接收到消息
```

消费者可能：

```text
还没有开始消费
消费过程中报错
消费逻辑卡住
处理成功但没有 ACK
```

---

## 五、RabbitMQ 中更完整的消息路径

RabbitMQ 中通常不是 Producer 直接把消息发送给 Queue。

更常见的路径是：

```text
Producer
↓
Exchange
↓
Queue
↓
Consumer
```

其中：

```text
Producer
→ 生产消息

Exchange
→ 根据 routingKey 和绑定规则进行路由

Queue
→ 保存等待消费的消息

Consumer
→ 从队列接收并处理消息
```

因此线上出现：

```text
数据库已有任务
Worker 没收到消息
用户一直等待
```

不能直接判断：

```text
一定是 Producer 发送失败
```

因为至少有三种可能。

---

## 六、Worker 没收到消息时的三种主要情况

### 情况一：Producer 没有成功发送到 Broker

可能表现为：

```text
Confirm ACK = false
```

或者：

```text
长时间没有收到 Confirm
```

可能原因：

```text
网络中断
RabbitMQ 连接断开
Broker 异常
消息发布失败
```

故障位置大致是：

```text
Producer
×
RabbitMQ Broker
```

---

### 情况二：Broker 收到了消息，但没有路由到 Queue

例如生产者发送：

```text
exchange = task.exchange
routingKey = task_execute
```

但是 RabbitMQ 中的实际绑定键是：

```text
task.execute
```

routingKey 不匹配时，Exchange 无法找到目标 Queue。

此时可能出现：

```text
Publisher Confirm ACK = true
```

因为 Broker 的 Exchange 已经接收了消息。

但是：

```text
消息没有进入任何 Queue
```

如果开启了 Publisher Returns，可以通过 ReturnCallback 发现这种问题。

典型结果：

```text
Confirm ACK = true
ReturnCallback 被触发
replyText = NO_ROUTE
```

这表示：

```text
Broker 收到了消息
但是没有成功路由到目标队列
```

所以：

```text
Publisher Confirm
→ 关注消息是否到达 Broker

Publisher Returns
→ 关注消息是否成功路由到 Queue
```

---

### 情况三：消息进入 Queue，但 Consumer 没有正常消费

如果：

```text
Confirm ACK = true
没有触发 ReturnCallback
```

通常说明消息已经被 Broker 接收，并成功完成路由。

此时需要继续检查 Queue 和 Consumer。

RabbitMQ 管理后台常见指标：

```text
Ready
Unacked
Consumers
```

#### Ready 数量增加

```text
Ready > 0
Consumers = 0
```

说明：

```text
消息正在队列中等待
但是当前没有正常工作的消费者
```

可能原因：

```text
Worker 服务没有启动
Worker 与 RabbitMQ 断开
Worker 监听了错误的 Queue
Consumer 初始化失败
消费者数量为 0
```

#### Unacked 数量增加

```text
Unacked > 0
```

说明：

```text
Consumer 已经拿到消息
但还没有返回 ACK
```

可能原因：

```text
消费逻辑卡死
数据库操作阻塞
外部接口超时
任务执行时间过长
消费者发生异常
手动 ACK 代码存在问题
```

因此：

> Worker 没有收到或没有完成任务，只能说明完整消息链路没有走通，不能仅凭最终现象确定故障一定发生在 Producer。

---

## 七、Consumer 和 Worker 的关系

我今天还有一个疑惑：

```text
Consumer 是什么？
它和 Worker 消费者有什么区别？
```

### Consumer

Consumer 是 RabbitMQ 消息体系中的一个角色。

它主要负责：

```text
监听 Queue
接收消息
调用消费逻辑
返回 ACK / NACK
```

例如：

```java
@RabbitListener(queues = "task.execute.queue")
public void consume(TaskMessage message) {
    executeTask(message);
}
```

这里：

```text
@RabbitListener + consume 方法
```

属于 Consumer 消费逻辑。

### Worker

Worker 通常是：

```text
运行 Consumer 代码的服务、进程或执行节点
```

同时它还会真正执行具体业务任务，例如：

```text
PDF 转换
OCR 识别
AI 分析
生成广告素材
调用第三方接口
```

所以可以简单理解为：

```text
Consumer
→ 负责拿任务

Worker
→ 承载 Consumer，并真正干任务
```

完整关系：

```text
RabbitMQ Queue
↓
Worker 服务中的 Consumer
↓
接收 TaskMessage
↓
Worker 内部业务代码
↓
执行 PDF / OCR / AI 任务
```

一个 Worker 服务中可以存在多个 Consumer。

例如：

```text
pdf-worker
├── PDF 转换 Consumer
├── OCR Consumer
└── AI 分析 Consumer
```

但是在很多简单项目中，Worker 服务的主要职责就是消费消息，所以开发人员经常混用：

```text
Worker 没收到消息
Consumer 没收到消息
消费者服务没收到消息
```

这些说法在具体上下文里往往指向同一个问题。

严格区分时：

```text
Consumer
→ MQ 体系中的消费角色或消费组件

Worker
→ 系统架构中的执行服务或执行进程
```

---

## 八、线上事故应该如何排查

假设 AI 异步任务平台发生事故：

```text
数据库中已有任务
↓
任务状态一直是 WAITING
↓
Worker 没有执行
↓
用户一直等待
```

正确排查顺序应该是：

### 第一步：检查 Producer 是否执行了发送

查看日志：

```text
是否生成 messageId
是否记录 MQ_SEND
exchange 是否正确
routingKey 是否正确
发送代码是否抛出异常
```

如果连发送日志都没有，可能是：

```text
业务代码没有执行到发送步骤
数据库事务提前异常
条件判断错误
程序在发送前崩溃
```

---

### 第二步：检查 Publisher Confirm

查看：

```text
ack = true
ack = false
是否长时间没有 Confirm
cause 是什么
```

判断：

```text
ack = false
→ Broker 没有成功接管消息

没有 Confirm
→ 结果暂时未知，需要进入异常处理或补偿
```

---

### 第三步：检查 Publisher Return

如果：

```text
Confirm ACK = true
```

继续看是否发生：

```text
NO_ROUTE
```

如果触发 ReturnCallback，说明：

```text
Exchange 收到了消息
但是没有匹配到 Queue
```

重点检查：

```text
exchange 名称
routingKey
Queue 绑定关系
交换机类型
```

---

### 第四步：检查 Queue 指标

查看：

```text
Ready
Unacked
Consumers
```

判断消息：

```text
是否还在 Queue 中等待
是否已经投递给 Consumer
是否没有消费者连接
```

---

### 第五步：检查 Consumer / Worker 日志

查看：

```text
是否记录 MQ_CONSUME_START
是否执行业务逻辑
是否发生异常
是否返回 ACK / NACK
是否进入重试或死信队列
```

最终排查链路可以总结为：

```text
Producer 发送日志
↓
Publisher Confirm
↓
Publisher Return
↓
Queue 指标
↓
Consumer 接收日志
↓
业务执行日志
↓
Consumer ACK / NACK
```

---

## 九、应该记录哪些日志

线上排查不能只记录：

```text
消息发送成功
消息消费失败
```

这种日志缺少业务标识，无法知道具体是哪一个任务。

需要使用统一标识贯穿完整链路。

### 关键标识

```text
taskId
messageId
traceId
```

三者作用不同。

```text
taskId
→ 业务任务标识

messageId
→ 本次 MQ 消息标识

traceId
→ 整个请求和调用链路标识
```

如果一个 Task 因为重试发送了多条消息，可能出现：

```text
同一个 taskId
对应多个 messageId
```

因此不能只记录 taskId，也不能只记录 messageId。

---

### Producer 发送日志

建议记录：

```text
taskId
messageId
traceId
exchange
routingKey
发送时间
重试次数
消息类型
```

示例：

```text
[MQ_SEND]
taskId=1001
messageId=msg-8fd21
traceId=trace-abc123
exchange=task.exchange
routingKey=task.execute
retryCount=0
```

---

### Confirm 日志

建议记录：

```text
taskId
messageId
traceId
ack
cause
confirmTime
```

示例：

```text
[MQ_CONFIRM]
taskId=1001
messageId=msg-8fd21
traceId=trace-abc123
ack=true
cause=null
```

如果 NACK：

```text
[MQ_CONFIRM]
taskId=1001
messageId=msg-8fd21
ack=false
cause=broker internal error
```

---

### Return 日志

建议记录：

```text
taskId
messageId
traceId
replyCode
replyText
exchange
routingKey
```

示例：

```text
[MQ_RETURN]
taskId=1001
messageId=msg-8fd21
replyCode=312
replyText=NO_ROUTE
exchange=task.exchange
routingKey=task.execute
```

---

### Consumer 接收日志

建议记录：

```text
taskId
messageId
traceId
queueName
redelivered
receiveTime
```

示例：

```text
[MQ_CONSUME_START]
taskId=1001
messageId=msg-8fd21
traceId=trace-abc123
queue=task.execute.queue
redelivered=false
```

---

### Consumer 执行结果日志

处理成功：

```text
[MQ_CONSUME_SUCCESS]
taskId=1001
messageId=msg-8fd21
cost=3280ms
ack=true
```

处理失败：

```text
[MQ_CONSUME_FAILED]
taskId=1001
messageId=msg-8fd21
error=OCR_SERVICE_TIMEOUT
nack=true
requeue=false
```

通过统一的 taskId、messageId 和 traceId，可以还原完整链路：

```text
任务入库
↓
准备发送
↓
Broker Confirm
↓
路由结果
↓
Consumer 接收
↓
业务执行
↓
Consumer ACK
```

---

## 十、没有收到 Confirm 时为什么不能直接无限重试

如果 Producer 没有收到 Confirm，第一反应通常是：

```text
重新发送消息
```

但是这里存在一个重要问题：

> 没有收到 Confirm，不一定代表 Broker 没有收到消息。

可能发生：

```text
Producer 发送消息
↓
Broker 成功收到消息
↓
Broker 返回 Confirm ACK
↓
网络突然中断
↓
ACK 在返回途中丢失
↓
Producer 没有收到 Confirm
```

此时真实情况是：

```text
Broker 已经有一条消息
```

但 Producer 认为：

```text
发送结果未知
```

如果 Producer 再发送一次：

```text
Queue 中可能出现两条相同业务消息
```

所以：

```text
重试
必须配合
幂等
```

否则消息可靠性提高以后，反而可能制造重复执行问题。

---

## 十一、自动重试应该如何设计

### 1. 有限次数重试

不要无限重试。

可以设计：

```text
第一次失败
↓
等待 1 秒
↓
第二次重试
↓
等待 3 秒
↓
第三次重试
↓
等待 10 秒
↓
仍然失败，进入补偿流程
```

这种方式属于退避重试。

目的：

```text
避免瞬时网络问题导致任务直接失败
避免 RabbitMQ 故障时产生高频重试风暴
```

需要控制：

```text
最大重试次数
重试间隔
下次重试时间
失败原因
```

---

### 2. 记录消息发送状态

可以建立本地消息记录表：

```text
mq_message_record
```

字段示例：

```text
id
message_id
task_id
trace_id
exchange
routing_key
message_body
status
retry_count
next_retry_time
last_error
created_time
updated_time
```

状态可以设计为：

```text
INIT
→ 待发送

SENDING
→ 正在发送

SUCCESS
→ 收到 Confirm ACK

FAILED
→ 明确发送失败

UNKNOWN
→ Confirm 超时，最终结果未知
```

发送前：

```text
status = INIT
```

准备发送：

```text
status = SENDING
```

收到 ACK：

```text
status = SUCCESS
```

收到 NACK：

```text
status = FAILED
```

长时间没有 Confirm：

```text
status = UNKNOWN
```

---

### 3. 定时补偿任务

后台定时任务扫描：

```text
status = FAILED
或者
status = UNKNOWN
```

并且满足：

```text
retry_count < 最大重试次数
next_retry_time <= 当前时间
```

然后重新发送。

流程：

```text
扫描失败消息
↓
抢占消息记录
↓
增加 retryCount
↓
重新发送 RabbitMQ
↓
等待 Confirm
↓
更新消息状态
```

为了避免多个补偿实例同时发送同一条消息，需要考虑：

```text
条件更新
数据库锁
分布式锁
任务抢占状态
```

---

## 十二、消费者为什么必须做幂等

补偿和重试可能产生重复消息。

例如：

```text
第一次消息已经到达 Broker
↓
Confirm ACK 丢失
↓
Producer 再次发送
↓
Queue 中存在两条相同任务消息
```

如果 Consumer 每次收到消息都直接执行：

```text
同一个 AI 任务可能执行两次
```

可能造成：

```text
重复调用 GPT
重复扣费
重复生成结果
重复写数据库
重复发送通知
重复创建广告素材
```

因此 Consumer 必须使用：

```text
messageId
或者
taskId
```

进行幂等控制。

### 基于任务状态机实现幂等

例如任务状态：

```text
WAITING
↓
RUNNING
↓
SUCCESS / FAILED
```

消费者收到消息后，可以先执行条件更新：

```sql
UPDATE task
SET status = 'RUNNING'
WHERE task_id = 1001
  AND status = 'WAITING';
```

如果影响行数为 1：

```text
当前 Consumer 成功获得任务执行权
```

如果影响行数为 0：

```text
任务可能已经被其他 Consumer 执行
或者任务已经完成
当前消息属于重复消息
```

此时不应该重复执行完整业务。

可以根据具体状态：

```text
直接 ACK
记录重复消息日志
进入人工检查
```

因此消息可靠性链路不是单独依靠 Confirm，而是：

```text
Publisher Confirm
+
Publisher Returns
+
有限重试
+
补偿机制
+
Consumer 幂等
+
Consumer ACK
+
死信队列
```

共同完成。

---

## 十三、数据库事务和 MQ 发送的一致性问题

AI 异步任务平台常见流程：

```text
insert task
↓
发送 MQ
```

可能出现：

```text
数据库事务提交成功
↓
MQ 发送失败
```

结果：

```text
数据库中有 WAITING 任务
RabbitMQ 中没有消息
Worker 永远不知道任务存在
```

Publisher Confirm 可以帮助发现：

```text
MQ 发送失败或结果未知
```

但是它不能自动解决：

```text
数据库和 MQ 的原子性问题
```

因为：

```text
数据库事务
和
RabbitMQ 消息发送
```

属于两个不同系统。

---

### 本地消息表 / Outbox 方案

更可靠的做法是：

```text
数据库事务开始
↓
insert task
↓
insert mq_message_record
↓
数据库事务提交
```

任务记录和待发送消息记录在同一个数据库事务中提交。

然后由独立发送程序执行：

```text
扫描 INIT / FAILED / UNKNOWN 消息
↓
发送 RabbitMQ
↓
收到 Confirm ACK
↓
更新为 SUCCESS
```

即使应用在数据库提交后立即宕机：

```text
mq_message_record 仍然存在
```

系统恢复后仍然可以继续补发。

这种方案的本质是：

> 不要求数据库和 RabbitMQ 在同一个事务里同时成功，而是先保证待发送事实不会丢失，再通过补偿实现最终一致性。

---

## 十四、Spring Boot 最小配置

配置示例：

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
```

含义：

```text
publisher-confirm-type: correlated
→ 开启关联式 Publisher Confirm
→ 可以通过 CorrelationData 关联具体消息

publisher-returns: true
→ 开启无法路由消息的返回机制

template.mandatory: true
→ 消息无法路由时返回给 Producer
→ 避免消息被静默丢弃
```

---

## 十五、Spring Boot 最小代码示例

### 发送消息

```java
String messageId = UUID.randomUUID().toString();

CorrelationData correlationData = new CorrelationData(messageId);

rabbitTemplate.convertAndSend(
        "task.exchange",
        "task.execute",
        taskMessage,
        correlationData
);
```

这里的：

```text
CorrelationData
```

用于把 Confirm 结果和具体 messageId 对应起来。

否则系统收到一个 Confirm 时，可能不知道它对应哪一条业务消息。

---

### ConfirmCallback

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    String messageId = correlationData == null
            ? null
            : correlationData.getId();

    if (ack) {
        log.info("[MQ_CONFIRM] messageId={}, ack=true", messageId);
        // 更新本地消息表状态为 SUCCESS
    } else {
        log.error(
                "[MQ_CONFIRM] messageId={}, ack=false, cause={}",
                messageId,
                cause
        );
        // 更新为 FAILED，等待重试或补偿
    }
});
```

---

### ReturnsCallback

```java
rabbitTemplate.setReturnsCallback(returned -> {
    log.error(
            "[MQ_RETURN] message={}, replyCode={}, replyText={}, exchange={}, routingKey={}",
            returned.getMessage(),
            returned.getReplyCode(),
            returned.getReplyText(),
            returned.getExchange(),
            returned.getRoutingKey()
    );

    // 记录 NO_ROUTE 等路由异常
    // 更新本地消息状态并进入补偿或告警
});
```

需要注意：

```text
ConfirmCallback
→ 判断 Broker 是否接收发布

ReturnsCallback
→ 判断消息是否无法路由到 Queue
```

两者不能互相替代。

---

## 十六、常见错误理解

### 错误一：convertAndSend 没报错就表示 MQ 收到了

错误：

```text
发送方法正常返回
=
Broker 已收到消息
```

正确理解：

```text
发送方法正常返回
→ Producer 完成了一次发送尝试

Publisher Confirm ACK
→ Broker 确认接收了这次发布
```

---

### 错误二：收到 Confirm ACK 就表示业务完成

错误：

```text
Confirm ACK
=
Consumer 执行成功
```

正确理解：

```text
Confirm ACK
→ Broker 收到了

Consumer ACK
→ Consumer 处理成功
```

---

### 错误三：Worker 没执行一定是 Producer 发送失败

错误：

```text
Worker 没执行
→ Producer 一定没发送成功
```

正确理解：

```text
可能是 Producer 发送失败
可能是 Exchange 无法路由
可能是 Queue 中没有 Consumer
可能是 Consumer 已接收但卡住
可能是 Consumer 处理失败
```

必须通过完整链路排查。

---

### 错误四：没有 Confirm 就直接无限重试

错误：

```text
没有 Confirm
↓
不断重新发送
```

风险：

```text
ACK 可能只是返回途中丢失
Broker 可能已经有消息
无限重试会产生重复消息和重试风暴
```

正确做法：

```text
有限退避重试
+
本地消息状态
+
定时补偿
+
Consumer 幂等
```

---

### 错误五：NACK 后只打印日志

错误做法：

```text
收到 NACK
↓
log.error(...)
↓
结束
```

问题：

```text
线上日志可能没人及时查看
消息依然会永久丢失
```

正确做法：

```text
记录失败状态
↓
有限重试
↓
定时补偿
↓
超过阈值后告警
↓
必要时人工处理
```

---

## 十七、和之前知识点的联系

Publisher Confirm 可以和之前学习的知识组合成完整可靠性链路。

### Day 1：幂等

解决：

```text
消息重试或重复投递后
如何避免业务重复执行
```

### Day 3：Consumer ACK

解决：

```text
Consumer 是否成功处理消息
```

### Day 4：死信队列

解决：

```text
多次消费失败后的异常消息如何隔离和处理
```

### Day 7：事务一致性与补偿

解决：

```text
数据库提交成功但 MQ 发送失败
如何通过本地消息表和补偿实现最终一致性
```

### Day 8：TraceId

解决：

```text
如何把任务入库、发送、路由、消费和执行日志串成完整链路
```

### Day 9：状态机

解决：

```text
任务如何从 WAITING 流转到 RUNNING、SUCCESS 或 FAILED
并辅助实现消费幂等
```

完整体系：

```text
数据库写入任务
↓
记录待发送消息
↓
Producer 发布消息
↓
Publisher Confirm
↓
Publisher Return 检查路由
↓
Queue 保存消息
↓
Consumer 接收消息
↓
幂等检查
↓
执行状态机
↓
Consumer ACK
↓
失败重试 / 死信 / 补偿
```

---

## 十八、面试表达

可以这样表达：

> 在异步任务系统中，我不会仅依赖 `convertAndSend` 方法是否正常返回来判断消息发送成功，而是会开启 RabbitMQ Publisher Confirm，通过 CorrelationData 将 Confirm 结果和具体 messageId 关联。如果收到 NACK 或长时间没有收到 Confirm，我会把消息记录为失败或未知状态，并通过有限次数的退避重试和定时补偿任务重新发送。
>
> 同时，我会开启 Publisher Returns，用于发现 Broker 已接收消息但因为 exchange、routingKey 或绑定关系错误而无法路由到 Queue 的情况。线上排查时，会按照 Producer 发送日志、Confirm、Return、Queue 的 Ready/Unacked/Consumers 指标、Consumer 接收日志和 Consumer ACK 的顺序定位故障。
>
> 由于 Confirm 超时不一定代表 Broker 没收到消息，重新发送可能产生重复消息，所以 Consumer 必须基于 taskId、messageId 或任务状态机实现幂等。对于数据库已经提交但 MQ 发送失败的问题，还需要结合本地消息表和定时补偿实现最终一致性。

---

## 十九、今天最大的认知修正

今天主要有四个认知修正。

### 第一

原本认为：

```text
消息正常方向是 Producer → RabbitMQ → Consumer
RabbitMQ 再返回 Confirm 很多余
```

实际理解：

```text
业务消息向后传递
Confirm 负责反向反馈接收结果
```

Confirm 不是多余流程，而是可靠性反馈通道。

---

### 第二

原本认为：

```text
Producer 调用了发送方法
就说明 RabbitMQ 已经收到
```

实际理解：

```text
发送动作完成
不等于
接收方已经成功接收
```

必须由 Broker 返回 Confirm 才能确认责任是否完成转移。

---

### 第三

原本认为：

```text
Worker 没收到消息
就说明 Producer 没发送成功
```

实际理解：

```text
Producer 发送失败
Broker 路由失败
Queue 没有 Consumer
Consumer 卡住或异常
```

都可能表现为 Worker 没有完成任务。

所以必须通过完整链路日志和 RabbitMQ 指标判断。

---

### 第四

原本没有区分：

```text
Consumer
Worker
```

实际理解：

```text
Consumer
→ MQ 中负责接收消息的角色或组件

Worker
→ 承载 Consumer 并执行具体业务任务的服务或进程
```

简单项目中两者经常被混用，但严格意义上并不完全相同。

---

## 二十、最终理解

我目前对 RabbitMQ Publisher Confirm 的理解是：

> Publisher Confirm 是 RabbitMQ Broker 给 Producer 的消息接收结果反馈。业务消息仍然按照 Producer → Exchange → Queue → Consumer 的方向流转，而 Confirm 通过反向通道告诉 Producer，Broker 是否已经接管了这次消息发布。发送方法正常返回只能证明 Producer 执行了发送尝试，不能证明 Broker 一定收到。
>
> 当 Worker 没有执行任务时，不能直接判断是 Producer 发送失败，还需要依次检查 Publisher Confirm、Publisher Return、Queue 的 Ready 和 Unacked 指标、消费者连接数量、Consumer 接收日志以及 Consumer ACK。Confirm 只保证消息到达 Broker，不保证消息成功路由到 Queue，更不保证消费者业务执行成功。
>
> 如果 Producer 没有收到 Confirm，应该进行有限次数的退避重试，并把消息状态持久化到本地消息表，由定时补偿任务继续处理。由于 Confirm 可能只是在返回途中丢失，重试有可能产生重复消息，因此必须结合 Consumer 幂等、任务状态机、Consumer ACK、死信队列和补偿机制，才能构成完整的消息可靠性体系。
