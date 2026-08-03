# 后端工程化每日训练 Day 27：Outbox Pattern、本地消息表与数据库 MQ 最终一致性

## 一、今天学习的知识点

今天学习的是：

**Outbox Pattern（本地消息表）、数据库与消息队列的最终一致性，以及 Publisher Confirm、Consumer ACK 的责任边界。**

一句话理解：

> 不直接要求“数据库更新”和“MQ 发送”同时成功，而是在同一个数据库事务中提交业务数据和待发送消息，再由 Sender 异步、可靠地把消息发送到 MQ。

今天重点解决以下问题：

```text
为什么 @Transactional 不能同时控制 MySQL 和 RabbitMQ
为什么数据库提交成功后，MQ 仍然可能发送失败
为什么 MQ 发送成功后，数据库事务仍然可能回滚
Outbox 本地消息表应该由谁写入
Sender 在整个流程中扮演什么角色
Publisher ACK / NACK 和 Consumer ACK / NACK 有什么区别
为什么 Outbox 仍然可能产生重复消息
为什么消费者必须实现幂等
```

---

## 二、问题背景：数据库与 MQ 双写一致性

很多后端业务都需要同时完成两件事：

```text
修改数据库中的业务状态
+
向消息队列发送事件
```

例如支付成功后：

```text
更新订单为已支付
↓
发送 OrderPaid 消息
↓
积分系统消费消息
↓
发放积分
```

最直接的代码可能是：

```java
@Transactional
public void paySuccess(Long orderId) {
    orderMapper.updateStatus(orderId, "PAID");

    rabbitTemplate.convertAndSend(
        "order.exchange",
        "order.paid",
        new OrderPaidEvent(orderId)
    );
}
```

这段代码看起来处于同一个方法中，并且方法上有 `@Transactional`，但它并不能保证：

```text
订单状态修改
和
RabbitMQ 消息发送
```

一定同时成功或同时失败。

本质原因是：

```text
MySQL
和
RabbitMQ
```

是两个彼此独立的系统。

Spring 的普通 `@Transactional` 默认只控制当前数据库连接上的本地事务，不能自动把 RabbitMQ 纳入同一个原子事务。

---

## 三、为什么数据库成功，但 MQ 可能失败

假设业务流程是：

```text
开始 MySQL 事务
↓
更新订单状态
↓
发送 RabbitMQ 消息
↓
提交 MySQL 事务
```

可能出现以下异常。

### 1. 网络异常

```text
订单更新完成
↓
应用准备发送 MQ
↓
网络闪断
↓
RabbitMQ 没有收到消息
```

如果异常被错误捕获且没有继续抛出，数据库事务仍然可能提交。

例如：

```java
@Transactional
public void paySuccess(Long orderId) {
    orderMapper.updateStatus(orderId, "PAID");

    try {
        rabbitTemplate.convertAndSend(...);
    } catch (Exception e) {
        log.error("MQ 发送失败", e);
        // 错误：异常被吞掉，方法继续正常结束
    }
}
```

最终结果：

```text
订单：PAID
MQ：没有 OrderPaid 消息
积分：没有到账
```

### 2. RabbitMQ 暂时不可用

可能原因包括：

```text
RabbitMQ 服务宕机
连接被断开
Broker 拒绝接收
Exchange 不存在
客户端连接池异常
发送超时
```

数据库和 RabbitMQ 的可用性并不绑定。

### 3. 应用在两个动作之间崩溃

如果采用先提交数据库、再发送消息的流程：

```text
数据库事务提交成功
↓
应用进程突然崩溃
↓
MQ 发送代码还没有执行
```

最终仍然是：

```text
数据库有业务结果
消息队列没有对应事件
```

### 4. 发送结果不确定

还可能发生：

```text
生产者发送消息
↓
RabbitMQ 实际已经收到
↓
Confirm ACK 返回途中网络中断
↓
生产者没有收到确认
```

此时生产者无法判断：

```text
消息到底没到
还是已经到了但确认丢失
```

如果生产者重新发送，就可能产生重复消息。

---

## 四、为什么 MQ 成功，但数据库仍可能回滚

反过来也可能发生：

```text
开始 MySQL 事务
↓
更新订单
↓
RabbitMQ 已经接收消息
↓
后续数据库代码报错
↓
MySQL 事务回滚
```

最终结果：

```text
数据库：订单仍然未支付
RabbitMQ：已经存在 OrderPaid 消息
积分消费者：可能已经发放积分
```

原因是：

> MySQL 回滚时，只能回滚 MySQL 中尚未提交的数据，无法把已经进入 RabbitMQ 的消息自动撤回。

所以，把 `rabbitTemplate.convertAndSend()` 写在 `@Transactional` 方法内部，不等于 MySQL 和 RabbitMQ 形成了同一个事务。

---

## 五、“把 MQ 放进事务”到底是什么意思

今天一开始容易产生的误解是：

```text
把 MQ 放进事务
是不是指消费者处理完后再提交数据库？
是不是指消费者 ACK？
```

这里讨论的不是消费者，而是生产端的双写问题。

所谓“把 MQ 发送放到事务方法里”，通常只是指：

```java
@Transactional
public void business() {
    updateDatabase();
    sendMq();
}
```

但普通本地事务只能控制：

```text
updateDatabase()
```

不能原子控制：

```text
sendMq()
```

因此这只是代码写在同一个方法里，不是真正的跨系统原子事务。

理论上可以研究分布式事务、XA 或 RabbitMQ 自身事务机制，但这类方案通常复杂、性能成本高，并且仍然要处理跨系统故障边界。

对于大量允许最终一致的业务，Outbox Pattern 通常更加直接、稳定、可排查。

---

## 六、Outbox Pattern 的核心原理

Outbox Pattern 的核心不是：

```text
数据库事务内直接发送 MQ
```

而是：

```text
数据库事务内写业务数据
+
数据库事务内写一条待发送消息
```

因为业务表和 Outbox 表都位于同一个 MySQL 数据库中，所以它们可以被同一个本地事务控制。

例如：

```text
BEGIN

UPDATE orders
SET status = 'PAID'
WHERE id = 1001;

INSERT INTO outbox_message (...)
VALUES (..., 'NEW');

COMMIT
```

事务只有两种结果：

```text
订单更新成功 + Outbox 插入成功
```

或者：

```text
订单更新失败 + Outbox 插入失败
```

不会出现：

```text
订单已支付
但数据库中完全没有对应的待发送消息
```

因此，Outbox 先把原来的跨系统双写：

```text
MySQL + RabbitMQ
```

转换为一个可靠的数据库本地事务：

```text
业务表 + Outbox 表
```

然后再通过异步 Sender、失败重试和补偿机制，把消息最终发送到 RabbitMQ。

---

## 七、完整流程和角色划分

完整流程如下：

```text
支付回调成功
↓
PaymentService 开启 MySQL 事务
↓
更新订单为 PAID
↓
插入 Outbox 消息，状态 NEW
↓
提交 MySQL 事务
↓
Outbox Sender 扫描 NEW 消息
↓
Sender 调用 rabbitTemplate 发送 RabbitMQ
↓
RabbitMQ 返回 Publisher Confirm
↓
确认成功后，Outbox 改为 SENT
↓
RabbitMQ 把消息投递给积分消费者
↓
消费者执行幂等校验
↓
发放积分
↓
消费者向 RabbitMQ 返回 Consumer ACK
↓
RabbitMQ 删除队列中的消息
```

### 1. PaymentService

负责：

```text
处理支付回调
更新订单状态
写入 Outbox 待发送消息
```

它是业务事件的产生方。

### 2. Outbox Sender

负责：

```text
扫描待发送消息
调用 RabbitMQ 客户端发送消息
处理 Confirm ACK / NACK / 超时
更新 Outbox 状态
执行有限重试和异常告警
```

从 RabbitMQ 客户端角色来看：

> 真正执行 `rabbitTemplate.convertAndSend()` 的 Sender，就是 MQ Producer（生产者）。

### 3. RabbitMQ

负责：

```text
接收 Producer 发布的消息
通过 Exchange 和 Routing Key 路由消息
把消息保存到 Queue
把消息投递给 Consumer
```

### 4. 积分消费者

负责：

```text
监听积分队列
接收 OrderPaid 事件
执行幂等校验
发放积分
处理成功后返回 ACK
```

---

## 八、Sender 到底是不是生产者

今天最容易混淆的问题是：

```text
完整流程里到底谁是生产者？
```

可以从两个角度理解。

### 业务角度

```text
支付服务
```

产生了“订单支付成功”这个业务事件，所以支付服务是事件生产方。

### RabbitMQ 客户端角度

```text
Outbox Sender
```

真正调用：

```java
rabbitTemplate.convertAndSend(...)
```

因此 Sender 是直接向 RabbitMQ 发布消息的 Producer。

在实际项目中，PaymentService 和 OutboxSender 不一定是两个独立服务。

它们可以位于同一个 Spring Boot 项目中：

```text
payment-service
├── PaymentService
│   └── 更新订单 + 写 Outbox
│
└── OutboxSender
    └── 扫描 Outbox + 发送 RabbitMQ
```

也可以把 Sender 独立成一个专门的消息投递服务。

所以不要把 Sender 理解成必须存在于另一台服务器，它首先是一个明确的后台发送职责。

---

## 九、Outbox 表如何设计

最小表结构可以包含：

```text
id
message_id
aggregate_type
aggregate_id
event_type
exchange_name
routing_key
payload
status
retry_count
next_retry_time
last_error
created_at
updated_at
sent_at
```

示例：

```sql
CREATE TABLE outbox_message (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id      VARCHAR(64)  NOT NULL,
    aggregate_type  VARCHAR(64)  NOT NULL,
    aggregate_id    VARCHAR(64)  NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    exchange_name   VARCHAR(100) NOT NULL,
    routing_key     VARCHAR(100) NOT NULL,
    payload         JSON         NOT NULL,
    status          VARCHAR(20)  NOT NULL,
    retry_count     INT          NOT NULL DEFAULT 0,
    next_retry_time DATETIME     NULL,
    last_error      VARCHAR(1000) NULL,
    created_at      DATETIME     NOT NULL,
    updated_at      DATETIME     NOT NULL,
    sent_at         DATETIME     NULL,
    UNIQUE KEY uk_message_id (message_id),
    KEY idx_status_retry_time (status, next_retry_time)
);
```

### 常见状态

```text
NEW
SENDING
SENT
FAILED
```

含义：

```text
NEW
→ 已写入数据库，等待 Sender 发送

SENDING
→ 某个 Sender 已经领取，正在发送

SENT
→ 已确认成功交给 RabbitMQ

FAILED
→ 已超过自动重试上限，需要告警或人工处理
```

状态设计不是越多越好，但必须能区分：

```text
尚未发送
正在处理
已经成功
自动重试失败
```

---

## 十、事务内如何写入业务数据和 Outbox

示例代码：

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final OrderMapper orderMapper;
    private final OutboxMessageMapper outboxMessageMapper;
    private final ObjectMapper objectMapper;

    @Transactional
    public void handlePaySuccess(Long orderId) {
        int updated = orderMapper.markPaid(orderId);
        if (updated == 0) {
            throw new IllegalStateException("订单不存在或状态不允许支付");
        }

        String messageId = UUID.randomUUID().toString();

        OrderPaidEvent event = new OrderPaidEvent(
            messageId,
            orderId,
            LocalDateTime.now()
        );

        OutboxMessage message = new OutboxMessage();
        message.setMessageId(messageId);
        message.setAggregateType("ORDER");
        message.setAggregateId(orderId.toString());
        message.setEventType("ORDER_PAID");
        message.setExchangeName("order.exchange");
        message.setRoutingKey("order.paid");
        message.setPayload(writeJson(event));
        message.setStatus("NEW");
        message.setRetryCount(0);
        message.setCreatedAt(LocalDateTime.now());
        message.setUpdatedAt(LocalDateTime.now());

        outboxMessageMapper.insert(message);
    }

    private String writeJson(Object value) {
        try {
            return objectMapper.writeValueAsString(value);
        } catch (JsonProcessingException e) {
            throw new IllegalStateException("事件序列化失败", e);
        }
    }
}
```

关键点是：

```text
markPaid()
和
outboxMessageMapper.insert()
```

处于同一个 MySQL 事务中。

如果事件序列化失败、Outbox 插入失败或订单更新失败，事务整体回滚。

---

## 十一、Sender 如何扫描并发送消息

最简单的 Sender 可以由：

```text
Spring @Scheduled
XXL-JOB
独立消息投递服务
```

实现。

简化流程：

```text
定时扫描 NEW
↓
领取一批消息
↓
逐条发送 RabbitMQ
↓
等待 Publisher Confirm
↓
成功改为 SENT
失败安排下次重试
```

示例伪代码：

```java
@Component
@RequiredArgsConstructor
public class OutboxSender {

    private final OutboxMessageMapper outboxMessageMapper;
    private final RabbitTemplate rabbitTemplate;

    @Scheduled(fixedDelay = 1000)
    public void sendPendingMessages() {
        List<OutboxMessage> messages =
            outboxMessageMapper.findSendableMessages(100);

        for (OutboxMessage message : messages) {
            sendOne(message);
        }
    }

    private void sendOne(OutboxMessage message) {
        try {
            CorrelationData correlationData =
                new CorrelationData(message.getMessageId());

            rabbitTemplate.convertAndSend(
                message.getExchangeName(),
                message.getRoutingKey(),
                message.getPayload(),
                correlationData
            );

            // 真实项目中应在 Publisher Confirm ACK 回调中标记 SENT，
            // 不能只因为 convertAndSend 没抛异常就立即认为 Broker 已收到。
        } catch (Exception e) {
            outboxMessageMapper.scheduleRetry(
                message.getId(),
                e.getMessage()
            );
        }
    }
}
```

不能把下面两件事直接画等号：

```text
convertAndSend() 正常返回
=
RabbitMQ 已可靠接收消息
```

更可靠的做法是结合 Publisher Confirm。

---

## 十二、Publisher Confirm：RabbitMQ 通知生产者

Sender 向 RabbitMQ 发布消息后，Broker 会向 Producer 返回 Confirm 结果。

方向是：

```text
Producer
↓ publish
RabbitMQ Broker
↑ confirm
Producer
```

### 1. Publisher ACK

```text
ACK
```

表示：

> RabbitMQ Broker 已经接收了本次发布。

Sender 收到 ACK 后可以执行：

```text
Outbox：SENDING / NEW → SENT
```

含义是：

```text
这条消息已经交给 RabbitMQ 负责
Sender 不需要继续按未发送消息重试
```

但 Publisher ACK 不代表：

```text
消费者已经收到
消费者已经处理完成
积分已经发放成功
```

它只确认 Producer 到 Broker 这一段。

### 2. Publisher NACK

```text
NACK
```

表示：

> RabbitMQ 没有成功接收或处理本次发布。

Sender 收到 NACK 后应该：

```text
记录失败原因
retry_count + 1
计算 next_retry_time
后续重新发送
超过上限后标记 FAILED 并告警
```

不能只打印一条错误日志后结束。

### 3. Confirm 超时或没有结果

```text
没有 ACK
也没有 NACK
```

这意味着：

```text
结果未知
```

消息可能：

```text
没有进入 RabbitMQ
```

也可能：

```text
已经进入 RabbitMQ，但确认返回途中丢失
```

所以超时后的重新发送可能制造重复消息。

这也是为什么 Outbox 不能提供“绝对只发送一次”，而是通常提供：

```text
至少发送一次
+
消费者幂等
```

---

## 十三、Publisher Return：Broker 收到但无法路由

RabbitMQ 常见消息路径是：

```text
Producer
↓
Exchange
↓
Queue
↓
Consumer
```

Publisher Confirm 主要确认消息是否到达 Broker / Exchange。

但可能出现：

```text
Exchange 已经收到消息
↓
Routing Key 不匹配
↓
找不到任何 Queue
```

此时可能出现：

```text
Publisher Confirm ACK = true
但是消息没有进入目标 Queue
```

因此还需要开启：

```text
mandatory
+
Publisher Returns / ReturnCallback
```

用于发现：

```text
NO_ROUTE
```

职责区分：

```text
Publisher Confirm
→ 消息是否到达 Broker / Exchange

Publisher Return
→ 消息是否因为路由失败而被退回
```

所以更完整的生产端成功判断应该结合：

```text
Confirm ACK
+
没有发生 Return
```

而不是仅看发送方法是否抛异常。

---

## 十四、Consumer ACK：消费者通知 RabbitMQ

RabbitMQ 把消息投递给积分消费者后，消费者处理成功需要返回 Consumer ACK。

方向是：

```text
RabbitMQ
↓ deliver
Consumer
↑ ACK
RabbitMQ
```

Consumer ACK 表示：

> 这条消息已经被消费者成功处理，RabbitMQ 可以从队列中删除它。

流程：

```text
RabbitMQ 投递 OrderPaid
↓
积分消费者收到消息
↓
幂等校验
↓
发放积分成功
↓
Consumer ACK
↓
RabbitMQ 删除队列中的消息
```

Consumer ACK 不会直接返回给原始 Producer。

生产者通常只负责确认：

```text
消息是否可靠交给 RabbitMQ
```

消费者是否完成业务，是另一段责任链。

---

## 十五、Consumer NACK 和重新入队

如果消费者处理失败，可以返回 Consumer NACK。

例如：

```text
积分数据库异常
外部服务超时
消费代码抛异常
```

NACK 常见处理方式包括：

```text
requeue = true
→ 重新放回队列，后续再次消费

requeue = false
→ 不重新放回原队列，可能丢弃或进入死信队列
```

不能对永久性错误无限 `requeue = true`。

否则可能形成：

```text
消费失败
↓
重新入队
↓
立即再次失败
↓
再次重新入队
↓
形成高频死循环
```

更合理的策略通常是：

```text
有限次数重试
+
延迟重试
+
死信队列
+
告警和人工补偿
```

---

## 十六、四种 ACK / NACK 的完整区别

### 生产端确认

```text
Publisher ACK
RabbitMQ → Producer
含义：Broker 已经接收消息
结果：Outbox 可以改为 SENT
```

```text
Publisher NACK
RabbitMQ → Producer
含义：Broker 没有成功接收或处理本次发布
结果：安排重试或进入失败处理
```

### 消费端确认

```text
Consumer ACK
Consumer → RabbitMQ
含义：消费者已成功处理消息
结果：RabbitMQ 可以删除队列中的消息
```

```text
Consumer NACK
Consumer → RabbitMQ
含义：消费者处理失败
结果：根据 requeue 配置重新入队、丢弃或进入死信队列
```

记忆方式：

```text
ACK = 成功确认
NACK = 失败确认
```

完整方向：

```text
Sender / Producer
    │
    │ publish
    ▼
RabbitMQ Broker
    │
    ├── Publisher ACK / NACK ──→ Producer
    │
    │ deliver
    ▼
Consumer
    │
    └── Consumer ACK / NACK ──→ RabbitMQ
```

---

## 十七、为什么消费者必须具备幂等能力

即使使用 Outbox，也不能假设每条消息只会发送一次。

典型重复场景：

```text
Sender 发送消息成功
↓
RabbitMQ 已经收到
↓
Publisher ACK 返回途中丢失
↓
Sender 认为结果未知
↓
Sender 再次发送
```

或者：

```text
RabbitMQ 返回 ACK
↓
Sender 还没来得及把 Outbox 改为 SENT
↓
Sender 服务突然崩溃
↓
数据库状态仍然是 NEW / SENDING
↓
服务重启后再次发送
```

消费者可能收到两次同一个事件：

```text
OrderPaid(messageId=msg-001, orderId=1001)
OrderPaid(messageId=msg-001, orderId=1001)
```

如果没有幂等：

```text
第一次消费：积分 +100
第二次消费：积分再 +100
```

所以消费者必须基于：

```text
messageId
eventId
orderId + benefitType
业务状态机
数据库唯一索引
```

实现幂等。

---

## 十八、消费者幂等的实现方式

### 方式一：消费记录表

建立：

```text
consumer_message_record
```

字段例如：

```text
consumer_name
message_id
processed_at
```

增加唯一约束：

```sql
UNIQUE (consumer_name, message_id)
```

消费逻辑：

```text
开始数据库事务
↓
尝试插入消费记录
↓
唯一键冲突
→ 说明已经消费，直接 ACK

插入成功
→ 执行业务操作
→ 提交事务
→ ACK
```

业务操作和消费记录应尽量处于同一个本地数据库事务中。

### 方式二：业务唯一索引

例如同一个订单只能发放一次支付积分：

```sql
UNIQUE (order_id, points_type)
```

即使消息重复到达，数据库也拒绝重复插入。

### 方式三：状态机判断

例如任务只能从：

```text
WAITING → RUNNING
```

第一次消费更新成功。

第二次重复消费发现任务已经是：

```text
RUNNING / SUCCESS
```

则不再重复执行。

### 方式四：Redis 幂等键

可以使用：

```text
SETNX consume:{consumer}:{messageId}
```

做快速拦截。

但对支付、积分、库存等关键业务，不能只依赖 Redis 临时记录，最好还有数据库唯一约束或状态机兜底。

---

## 十九、Sender 并发扫描时的重复领取问题

如果部署多个 Sender 实例：

```text
Sender A
Sender B
```

两者可能同时扫描到同一条 `NEW` 消息。

```text
A 查到 message-1
B 也查到 message-1
↓
两边都发送
↓
重复消息
```

可以采用：

```text
数据库行锁
SELECT ... FOR UPDATE SKIP LOCKED
乐观锁 version
条件更新抢占状态
分片任务
```

例如通过条件更新领取消息：

```sql
UPDATE outbox_message
SET status = 'SENDING',
    updated_at = NOW()
WHERE id = ?
  AND status = 'NEW';
```

只有更新行数为 1 的 Sender 才获得发送权。

还要处理 `SENDING` 卡死：

```text
Sender 把消息改为 SENDING
↓
发送前服务崩溃
↓
消息永久停留在 SENDING
```

因此需要超时恢复：

```text
SENDING 超过一定时间
↓
重新转为 NEW 或进入补偿流程
```

---

## 二十、失败重试不能无限进行

RabbitMQ 长期不可用、Exchange 配置永久错误或消息格式非法时，如果每秒无限重试，会造成：

```text
数据库扫描压力
RabbitMQ 连接压力
错误日志爆炸
CPU 和线程资源浪费
告警噪声
真正可恢复消息被淹没
```

应该设计：

```text
最大重试次数
指数退避
下一次重试时间
失败原因
最终失败状态
监控告警
人工补偿入口
```

例如：

```text
第 1 次失败：10 秒后重试
第 2 次失败：30 秒后重试
第 3 次失败：2 分钟后重试
第 4 次失败：10 分钟后重试
超过 10 次：标记 FAILED 并告警
```

重试策略必须考虑：

```text
故障是否暂时可恢复
下游当前承载能力
重复发送的业务成本
消息是否已经过期
```

---

## 二十一、Outbox 表为什么会无限增长

消息发送成功后如果长期保留：

```text
每天 100 万条消息
↓
一年约 3.65 亿条
```

会带来：

```text
索引体积增加
扫描变慢
备份恢复成本增加
数据库存储成本上升
DDL 和归档操作困难
```

应制定生命周期策略：

```text
SENT 保留 7 天或 30 天
↓
迁移到历史表、归档库或对象存储
↓
从在线 Outbox 表删除
```

可以采用：

```text
按时间分区
冷热表拆分
定时归档
分批删除
```

删除时不能一次删除几千万行，应小批量执行，避免长事务、锁竞争和主从复制延迟。

---

## 二十二、Payload 和事件版本问题

Outbox 中通常会保存序列化后的事件 Payload。

例如：

```json
{
  "messageId": "msg-001",
  "eventType": "ORDER_PAID",
  "eventVersion": 1,
  "orderId": 1001,
  "paidAt": "2026-08-04T00:30:00"
}
```

需要考虑：

```text
字段新增或删除
消费者版本不一致
旧消息延迟数小时后才发送
消息序列化失败
Payload 太大
敏感信息泄露
```

建议：

```text
事件包含 messageId
事件包含 eventType
事件包含 eventVersion
只携带消费者真正需要的数据
敏感字段脱敏或不写入
对 Payload 大小设置上限
保持向后兼容
```

不要直接把复杂数据库实体完整序列化后作为长期事件契约。

---

## 二十三、Outbox 不是“绝对只发送一次”

Outbox 能保证的是：

```text
只要业务事务成功
数据库中就存在一条可补偿、可重试的待发送消息
```

它主要解决：

```text
数据库已经成功
但消息因为一次发送失败而永久丢失
```

它不能天然保证：

```text
消息永远只发送一次
消费者永远只执行一次
MQ 永不丢失
Sender 永不崩溃
```

更准确的可靠性模型是：

```text
本地事务
+
Outbox 持久化
+
Sender 重试
+
Publisher Confirm
+
Publisher Return
+
消费者幂等
+
Consumer ACK
+
死信与人工补偿
+
监控告警
```

最终实现：

```text
至少一次投递
+
业务效果最终只发生一次
```

---

## 二十四、Outbox 与消费者 ACK 的责任边界

Outbox 的状态 `SENT` 通常表示：

```text
生产者已经确认消息交给 RabbitMQ
```

并不表示：

```text
积分已经发放完成
```

因为生产者和消费者是解耦的。

如果业务必须知道消费者最终是否处理成功，可以另外设计：

```text
业务结果表
消费状态查询
回执事件
补偿任务
Saga 状态
```

但不能把 Consumer ACK 误认为会直接返回给原始 Producer。

Consumer ACK 的直接接收方是 RabbitMQ。

---

## 二十五、支付系统中的完整设计

假设支付成功后需要：

```text
更新订单
发放积分
发放优惠券
发送通知
```

推荐流程：

```text
支付平台回调
↓
验证签名、金额和订单状态
↓
开始 MySQL 本地事务
↓
幂等更新订单为 PAID
↓
插入 OrderPaid Outbox 消息，状态 NEW
↓
提交事务
↓
返回支付平台成功响应
```

后台发送：

```text
Sender 领取 NEW 消息
↓
状态改为 SENDING
↓
发送 RabbitMQ
↓
Confirm ACK 且未发生 Return
↓
状态改为 SENT
```

各业务消费者：

```text
积分消费者
优惠券消费者
通知消费者
```

分别监听自己的队列，并独立实现：

```text
幂等
有限重试
死信
告警
```

这样一个消费者失败不会阻塞其他消费者。

---

## 二十六、AI 异步任务平台中的应用

用户提交 AI 任务：

```text
POST /tasks
```

事务中：

```text
插入 task，状态 WAITING
+
插入 TaskCreated Outbox，状态 NEW
```

提交后：

```text
Outbox Sender
↓
RabbitMQ / Kafka
↓
Worker Consumer
↓
幂等领取任务
↓
任务状态 WAITING → RUNNING
↓
调用模型
↓
保存结果
↓
任务状态 RUNNING → SUCCESS / FAILED
↓
Consumer ACK
```

即使 RabbitMQ 暂时故障：

```text
task 仍然存在
Outbox 消息仍然存在
Sender 恢复后继续发送
```

不会因为一次 MQ 故障让任务永久消失。

---

## 二十七、文件处理平台中的应用

文件上传成功后可能要异步执行：

```text
OCR
病毒扫描
缩略图生成
文本索引
内容审核
```

事务中可以：

```text
插入 file_info
+
插入 FileUploaded Outbox
```

Sender 将消息发送到 MQ，多个消费者分别处理不同任务。

需要注意：

> 对象存储文件本体不受 MySQL 本地事务控制。

如果流程包含：

```text
上传 OBS
+
写 MySQL
+
写 Outbox
```

仍然需要结合 Day 26 学过的：

```text
文件状态
临时对象
补偿任务
孤儿对象清理
存在性校验
```

Outbox 主要解决 MySQL 业务数据和 MQ 事件的一致性，不能自动把对象存储也纳入 MySQL 事务。

---

## 二十八、常见错误设计

### 错误一：只在事务中直接发送 MQ

```java
@Transactional
public void paySuccess() {
    updateOrder();
    rabbitTemplate.convertAndSend(...);
}
```

问题：

```text
MySQL 和 RabbitMQ 不是同一个本地事务
```

### 错误二：先提交数据库，再直接发送 MQ

```text
数据库提交
↓
发送前应用崩溃
```

结果：消息永久缺失。

### 错误三：先发送 MQ，再提交数据库

```text
MQ 已收到
↓
数据库回滚
```

结果：消费者处理了不存在或未生效的业务数据。

### 错误四：把 convertAndSend 正常返回当作可靠成功

问题：

```text
只能说明客户端调用没有立即抛异常
不能证明 Broker 已可靠接管
```

### 错误五：收到 NACK 只打印日志

问题：

```text
没有重试
没有补偿
没有告警
消息仍然永久丢失
```

### 错误六：收到 ACK 就认为消费者处理完成

Publisher ACK 只表示：

```text
RabbitMQ 收到消息
```

不表示积分已经到账。

### 错误七：消费者没有幂等

结果：

```text
重复积分
重复扣库存
重复发券
重复通知
```

### 错误八：多个 Sender 同时发送同一条消息

原因：

```text
没有领取锁或状态条件更新
```

### 错误九：无限快速重试

结果：

```text
故障放大
日志爆炸
MQ 和数据库压力进一步增加
```

### 错误十：Outbox 永不归档

结果：表持续膨胀，扫描和维护成本越来越高。

---

## 二十九、线上排查顺序

假设出现：

```text
订单已经 PAID
积分没有到账
```

排查顺序：

### 第一步：检查 Outbox

根据 `orderId` 或 `messageId` 查询：

```text
是否存在 Outbox 记录
status 是什么
retry_count 是多少
last_error 是什么
next_retry_time 是什么
```

判断：

```text
没有 Outbox
→ 事务代码、事件生成或历史数据存在问题

NEW
→ Sender 没有扫描到或尚未发送

SENDING
→ 可能发送中，也可能 Sender 崩溃造成卡死

SENT
→ 生产端已确认交给 RabbitMQ

FAILED
→ 自动重试耗尽，需要人工处理
```

### 第二步：检查 Sender 日志

查看：

```text
是否领取该 messageId
是否执行发送
exchange 和 routingKey 是否正确
是否收到 Confirm ACK / NACK
是否发生 Confirm 超时
是否触发 ReturnCallback
```

### 第三步：检查 RabbitMQ

查看：

```text
Queue 是否存在
绑定关系是否正确
Ready
Unacked
Consumers
死信队列
```

### 第四步：检查 Consumer

查看：

```text
是否收到 messageId
幂等校验结果
积分事务是否成功
是否返回 ACK / NACK
是否进入重试或死信
```

### 第五步：检查业务结果

查看：

```text
积分流水
消费记录表
订单与积分关联记录
唯一索引冲突
状态机流转
```

完整排查链路：

```text
业务事务
↓
Outbox 记录
↓
Sender
↓
Publisher Confirm / Return
↓
Exchange / Queue
↓
Consumer
↓
幂等记录
↓
业务结果
↓
Consumer ACK
```

---

## 三十、需要监控哪些指标

生产环境至少监控：

```text
NEW 消息数量
最老 NEW 消息等待时长
SENDING 超时数量
每分钟发送成功数
每分钟发送失败数
Publisher NACK 数量
Publisher Confirm 超时数量
ReturnCallback / NO_ROUTE 数量
平均重试次数
FAILED 消息数量
Outbox 表总行数和增长速度
Sender 扫描耗时
消息端到端延迟
Consumer 重复消息数量
Consumer 失败和死信数量
```

其中最重要的告警之一是：

```text
最老待发送消息等待时长
```

因为即使 `NEW` 数量不大，只要某条关键消息长时间未发送，也可能说明链路已经卡住。

---

## 三十一、今天思考题的完整回答

### 1. 为什么数据库事务成功，但 MQ 可能没有发送成功

因为：

```text
MySQL 和 RabbitMQ 是两个独立系统
```

数据库事务成功只说明 MySQL 提交成功，MQ 仍可能因为：

```text
网络中断
RabbitMQ 宕机
连接异常
发布失败
应用崩溃
异常被吞掉
```

而没有成功收到消息。

### 2. 为什么不能简单把 MQ 发送放进事务方法里解决

因为普通 `@Transactional` 只能控制数据库本地事务，不能自动回滚已经发送到 RabbitMQ 的消息，也不能让 RabbitMQ 失败时天然补偿数据库。

代码写在同一个方法中，不代表两个系统形成同一个原子事务。

### 3. 采用 Outbox Pattern 后如何设计

```text
支付回调
↓
开始 MySQL 事务
↓
幂等更新订单为 PAID
↓
插入 Outbox，状态 NEW
↓
提交事务
↓
Sender 扫描并领取 NEW 消息
↓
Sender 作为 Producer 发送 RabbitMQ
↓
收到 Publisher ACK 且未发生路由退回
↓
Outbox 改为 SENT
↓
消费者收到消息
↓
幂等发放积分
↓
Consumer ACK
↓
RabbitMQ 删除队列消息
```

### 4. 为什么消费者必须幂等

因为以下情况都可能产生重复消息：

```text
Confirm ACK 丢失
Sender 标记 SENT 前崩溃
多个 Sender 重复领取
Consumer 处理成功但 ACK 丢失
RabbitMQ 重新投递
人工补偿再次发送
```

如果消费者没有幂等，会出现重复发积分、重复扣库存、重复发券等严重问题。

---

## 三十二、面试表达

可以这样回答：

> 在需要同时更新数据库和发送 MQ 的业务中，我不会只把 `rabbitTemplate.convertAndSend` 写进 `@Transactional` 方法，因为普通本地事务只能控制 MySQL，不能保证 MySQL 和 RabbitMQ 原子提交。
>
> 我会采用 Outbox Pattern，在同一个数据库事务内更新业务表并写入一条状态为 NEW 的本地消息。事务提交后，由独立的 Outbox Sender 扫描并领取待发送消息，Sender 作为 MQ Producer 发布到 RabbitMQ，并结合 Publisher Confirm 和 Publisher Return 判断消息是否成功进入 Broker 并路由到 Queue。发送失败时进行有限次数的退避重试，超过上限后告警和人工补偿。
>
> 由于 Confirm 超时、Sender 标记状态前宕机、Consumer ACK 丢失等情况都可能导致重复投递，所以消费者还必须基于 messageId、业务唯一键或状态机实现幂等。最终通过本地事务、Outbox 持久化、可靠发送、消费者幂等和监控补偿实现数据库与 MQ 的最终一致性。

---

## 三十三、今天需要记住的核心边界

```text
@Transactional
→ 控制 MySQL 本地事务

Outbox Pattern
→ 保证业务数据和待发送消息在同一个数据库事务中提交

Outbox Sender
→ 真正向 RabbitMQ 发布消息的 Producer

Publisher ACK
→ RabbitMQ 已接收发布，Outbox 可以标记 SENT

Publisher NACK
→ RabbitMQ 未成功接收或处理，需要重试

Publisher Return
→ Broker 收到消息，但无法路由到 Queue

Consumer ACK
→ 消费者处理成功，RabbitMQ 可以删除队列消息

Consumer NACK
→ 消费者处理失败，决定重新入队、丢弃或进入死信

消费者幂等
→ 防止重复消息造成重复业务结果
```

一句话总结：

> Outbox 负责让消息不因一次发送失败而永久丢失，Publisher Confirm 负责确认 Producer 到 RabbitMQ，Consumer ACK 负责确认 Consumer 到 RabbitMQ，而消费者幂等负责消除重复投递带来的业务副作用。

---

## 三十四、能力等级

**高级后端工程能力。**

Outbox Pattern 是支付、订单、库存、积分、优惠券、AI 异步任务和文件处理平台中解决数据库与消息队列最终一致性的经典方案。

它与以下能力共同构成可靠异步系统：

```text
Day 01：幂等设计
Day 03：Consumer ACK
Day 04：失败重试与死信队列
Day 07：事务边界与最终一致性
Day 09：状态机
Day 12：Publisher Confirm
Day 13：Kafka 消息积压
Day 16：XXL-JOB 补偿调度
Day 20：链路日志排查
Day 21：监控与告警
Day 24：熔断与降级
Day 25：线程池与流量保护
Day 26：对象存储与数据一致性补偿
```

真正可靠的系统不是只依赖某一个注解或某一个中间件，而是建立：

```text
明确事务边界
持久化中间状态
可靠消息投递
消费者幂等
失败重试
状态机约束
监控告警
人工补偿
```

组成的完整工程化闭环。
