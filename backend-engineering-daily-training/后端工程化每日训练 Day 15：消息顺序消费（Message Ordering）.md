# 后端工程化每日训练 Day 15：消息顺序消费（Message Ordering）

## 一、今天学习的知识点

今天学习的是：

**消息顺序消费（Message Ordering）**

一句话理解：

> 当业务事件必须按照发生顺序处理时，不仅要保证消息最终被消费，还要保证同一个业务对象的消息按照正确的先后顺序执行。

例如订单状态：

```text
订单创建
↓
支付成功
↓
发货
↓
签收
```

如果业务执行顺序变成：

```text
发货
↓
支付成功
```

即使每条消息都没有丢失，最终业务数据仍然可能错误。

常见需要顺序的场景包括：

```text
订单状态流转
用户余额变更
库存扣减
AI 异步任务状态更新
广告投放状态更新
支付和退款事件
```

---

## 二、消息成功不等于业务正确

很多人刚接触 MQ 时会认为：

```text
消息没有丢失
+
消费者最终处理成功
=
业务一定正确
```

这个判断并不成立。

假设某个 AI 任务依次产生：

```text
QUEUED
↓
RUNNING
↓
SUCCESS
↓
ARCHIVED
```

如果消费端最终执行顺序变成：

```text
SUCCESS
↓
RUNNING
```

数据库最终可能停留在：

```text
RUNNING
```

真实任务已经成功，但用户看到的仍然是运行中。

所以消息系统需要同时考虑：

```text
消息可靠性
消息幂等
消息顺序
状态机约束
异常重试
```

---

## 三、顺序消费通常不是整个 Topic 全局有序

顺序消费并不意味着：

```text
整个 Topic 的所有消息只能一条一条处理
```

真正常见的业务要求是：

> 同一个业务对象的事件必须有序，不同业务对象之间可以并行处理。

例如：

```text
Task-1001：QUEUED → RUNNING → SUCCESS
Task-2001：QUEUED → RUNNING → FAILED
Task-3001：QUEUED → RUNNING → SUCCESS
```

要求是：

```text
Task-1001 内部必须有序
Task-2001 内部必须有序
Task-3001 内部必须有序
```

但不需要：

```text
Task-2001 必须等待 Task-1001 全部完成
```

这种设计可以兼顾：

```text
同一业务对象的顺序
不同业务对象之间的吞吐
```

---

## 四、Kafka 顺序消费的核心基础

### 1. 使用稳定业务 Key

Kafka Producer 发送消息时，可以为消息指定 Key。

例如：

```java
producer.send(
    new ProducerRecord<>("task-status-topic", taskId, message)
);
```

这里使用：

```text
taskId
```

作为消息 Key。

Kafka 默认会根据 Key 计算消息应该进入哪个 Partition：

```text
hash(key)
↓
选择 Partition
```

例如：

```text
Task-1001
↓
Partition 3
```

只要 Partition 数量和分区策略没有变化，同一个 `taskId` 产生的消息通常会进入同一个 Partition。

### 2. Kafka 保证同一 Partition 内的日志顺序

假设同一个 Partition 中写入：

```text
offset=100  RUNNING
offset=101  SUCCESS
offset=102  ARCHIVED
```

Kafka 会按照 Partition 内的 offset 保存这些消息。

因此 Kafka 可以保证：

> 同一个 Partition 中的消息读取顺序与日志中的 offset 顺序一致。

Kafka 不保证：

```text
Partition 0
和
Partition 1
```

之间存在全局顺序。

### 3. Consumer Group 中一个 Partition 同一时刻只分配给一个 Consumer

例如：

```text
Partition 0 → Consumer A
Partition 1 → Consumer B
Partition 2 → Consumer C
```

在同一个 Consumer Group 中，一个 Partition 不会同时由 Consumer A 和 Consumer B 共同消费。

因此，只要满足：

```text
同一业务对象使用相同 Key
↓
进入同一 Partition
↓
Consumer 按照拉取顺序处理
```

就具备了顺序消费的基础。

---

## 五、Kafka 的读取顺序不等于业务执行顺序

Kafka 能保证消息按照 offset 顺序被拉取，但不能保证开发者自己的业务线程也按照这个顺序完成。

例如 Consumer 按照以下顺序拉取：

```text
offset=100  RUNNING
offset=101  SUCCESS
offset=102  ARCHIVED
```

如果直接交给线程池：

```text
线程 1 处理 RUNNING
线程 2 处理 SUCCESS
线程 3 处理 ARCHIVED
```

各任务执行时间不同：

```text
SUCCESS 100ms 完成
ARCHIVED 300ms 完成
RUNNING 3s 完成
```

数据库实际更新顺序就可能变成：

```text
SUCCESS
↓
ARCHIVED
↓
RUNNING
```

最终状态被旧事件覆盖。

所以需要明确：

> Kafka 保证的是 Partition 内消息的存储和拉取顺序，不保证业务线程的完成顺序。

---

## 六、串行、并发和并行的区别

### 1. 串行（Serial）

串行表示：

> 前一个任务执行完成以后，后一个任务才能开始。

例如：

```text
任务 A 完成
↓
任务 B 开始
↓
任务 B 完成
↓
任务 C 开始
```

在消息顺序消费中：

```text
Task-1001 RUNNING 处理完成
↓
Task-1001 SUCCESS 开始处理
↓
Task-1001 ARCHIVED 开始处理
```

这就是同一个 Task 内部串行。

### 2. 并发（Concurrency）

并发表示：

> 多个任务在同一段时间内都在推进，但某个瞬间不一定真正同时执行。

可以类比一个人处理多件家务：

```text
做一会儿饭
↓
等待水烧开时去扫地
↓
扫一会儿再回来做饭
```

任务通过切换交替推进。

### 3. 并行（Parallelism）

并行表示：

> 多个任务在同一个时刻真正同时执行。

例如：

```text
线程 1 在 CPU Core 1 执行 Task-A
线程 2 在 CPU Core 2 执行 Task-B
```

对应家务类比：

```text
甲做饭
乙扫地
丙洗衣服
```

多个人在同一时刻分别做不同事情。

### 4. 三者在顺序消费中的关系

目标通常是：

```text
同一个 Task 内部串行
不同 Task 之间并发或并行
```

例如：

```text
线程 1：串行处理 Task-1001 的所有状态消息
线程 2：串行处理 Task-2001 的所有状态消息
线程 3：串行处理 Task-3001 的所有状态消息
```

整个系统具有吞吐能力，但每个 Task 自己不会乱序。

---

## 七、线上出现 SUCCESS 后又变回 RUNNING，可能是谁的问题

不能看到最终状态错误就直接断定：

```text
一定是 Kafka 自己把消息顺序打乱了
```

生产端和消费端都可能导致业务顺序被破坏。

### 1. 生产端可能的问题

#### 同一个 Task 使用了不同 Key

例如：

```text
RUNNING：key = taskId
SUCCESS：key = UUID
```

两条消息可能进入不同 Partition，不同 Partition 之间没有顺序保证。

#### 多个线程并发发送

例如：

```text
线程 A 发送 RUNNING
线程 B 发送 SUCCESS
```

由于线程调度、异步发送和失败重试，实际写入 Broker 的顺序可能与业务预期不同。

#### 多个服务实例产生旧事件

例如：

```text
Worker A 已经发送 SUCCESS
补偿程序 B 又发送旧的 RUNNING
```

这不是 Kafka 改变顺序，而是生产端后来又产生了一条旧状态消息。

### 2. 消费端可能的问题

#### 使用线程池并行执行

Kafka 按照：

```text
RUNNING → SUCCESS
```

拉取，但线程池可能按照：

```text
SUCCESS → RUNNING
```

完成。

#### 旧消息重试

`RUNNING` 第一次处理失败，被放入重试队列；`SUCCESS` 正常执行完成以后，旧的 `RUNNING` 又重试成功。

#### 重复消费

Consumer 在业务执行成功后、offset 提交前崩溃，消息重新投递，也可能造成旧状态再次执行。

#### 错误的 offset 提交

同一 Partition 内多个消息并行处理时，如果后面的 offset 先完成并被错误提交，可能跨过前面尚未完成的消息，造成丢失、重复或恢复顺序异常。

---

## 八、如何判断消息是否进入了不同 Partition

不能只根据 Key 的 Hash 自己推算。

因为实际 Partition 还受到以下因素影响：

```text
Topic 的 Partition 数量
默认或自定义分区器
消息是否真的携带预期 Key
Topic 是否扩容
分区策略是否调整
```

最可靠的方法是记录 Kafka 消息的实际元数据。

### 1. Producer 发送回调记录

```java
ProducerRecord<String, TaskStatusMessage> record =
        new ProducerRecord<>("task-status-topic", taskId, message);

producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        log.error(
                "send task status failed, taskId={}, status={}",
                taskId,
                message.getStatus(),
                exception
        );
        return;
    }

    log.info(
            "send task status success, taskId={}, status={}, key={}, partition={}, offset={}, eventSeq={}",
            taskId,
            message.getStatus(),
            taskId,
            metadata.partition(),
            metadata.offset(),
            message.getEventSeq()
    );
});
```

### 2. Consumer 处理时记录

```java
public void consume(ConsumerRecord<String, TaskStatusMessage> record) {
    TaskStatusMessage message = record.value();

    log.info(
            "consume task status, taskId={}, status={}, key={}, partition={}, offset={}, eventSeq={}",
            message.getTaskId(),
            message.getStatus(),
            record.key(),
            record.partition(),
            record.offset(),
            message.getEventSeq()
    );

    taskStatusService.handle(message);
}
```

### 3. 根据日志判断

如果日志是：

```text
RUNNING  partition=1 offset=100
SUCCESS  partition=3 offset=52
```

说明两条消息进入了不同 Partition，Kafka 不保证它们之间的顺序。

如果日志是：

```text
RUNNING  partition=1 offset=100
SUCCESS  partition=1 offset=101
```

说明 Kafka 中的 Partition 和 offset 顺序正确。

此时应重点排查：

```text
消费端线程池
业务执行耗时
消息重试
重复消费
offset 提交
数据库更新日志
```

线上排查顺序问题时，建议至少记录：

```text
eventId
taskId
status
messageKey
partition
offset
eventSeq
producerTime
consumeTime
finishTime
consumerInstance
```

---

## 九、消费端使用线程池时如何兼顾吞吐和顺序

顺序和性能确实存在矛盾。

对于同一个 Task：

```text
前一条没有完成
↓
后一条不能执行
```

因此，同一个 Task 的处理速度不可能通过无约束并行无限提高。

真正可行的设计是：

> 同一个 Key 串行，不同 Key 并行。

### 方案一：使用 Partition 作为并发单位

```text
Task-1001 → Partition 0 → Consumer A 串行处理
Task-2001 → Partition 1 → Consumer B 串行处理
Task-3001 → Partition 2 → Consumer C 串行处理
```

通过增加合理数量的：

```text
Partition
Consumer
```

提高不同 Task 之间的并行度。

限制是：

```text
同一个 Partition 中的慢消息
```

会阻塞后续消息。

### 方案二：按业务 Key 分桶

假设应用内部创建 8 个单线程执行器：

```java
public class KeyOrderedExecutor {

    private final ExecutorService[] executors;

    public KeyOrderedExecutor(int bucketCount) {
        this.executors = new ExecutorService[bucketCount];
        for (int i = 0; i < bucketCount; i++) {
            executors[i] = Executors.newSingleThreadExecutor();
        }
    }

    public void submit(String taskId, Runnable task) {
        int bucket = Math.floorMod(taskId.hashCode(), executors.length);
        executors[bucket].submit(task);
    }
}
```

分配过程：

```text
hash(Task-1001) % 8 → 队列 3
hash(Task-2001) % 8 → 队列 6
hash(Task-3001) % 8 → 队列 1
```

同一个 `taskId` 永远进入同一个单线程队列，因此能够串行执行。

不同 Task 进入不同队列时，可以并行处理。

这种方式本质是：

```text
桶内串行
桶间并行
```

### 分桶方案的限制

不同 Task 可能 Hash 到同一个桶：

```text
Task-1001 → 队列 3
Task-9009 → 队列 3
```

它们会互相等待，但不会破坏顺序。

桶数量越多，并行度通常越高，但也会增加：

```text
线程数量
内存占用
调度成本
关闭和恢复复杂度
```

### offset 提交仍然是难点

即使业务按 Key 分桶，也不能简单地：

```text
poll 消息
↓
提交线程池
↓
立即提交 offset
```

否则消息还没有处理完成，进程就可能崩溃，导致任务丢失。

同一 Partition 中：

```text
offset=100 未完成
offset=101 已完成
offset=102 已完成
```

也不能直接提交 offset=102，否则可能跨过尚未完成的 offset=100。

因此异步消费还需要：

```text
跟踪每个 Partition 的连续完成 offset
只提交已经连续完成的最大位置
Rebalance 时停止接收并等待或回收未完成任务
业务层保证幂等
```

如果业务非常复杂，更稳妥的方案可能是：

```text
Kafka Consumer 快速幂等落库
↓
提交 offset
↓
独立 Worker 根据数据库任务表执行
```

---

## 十、状态机如何作为最后一道保护

状态机的核心是：

> 明确规定当前状态允许流转到哪些目标状态，不允许任意状态直接覆盖。

AI 任务状态可以设计为：

```text
QUEUED
↓
RUNNING
↓
SUCCESS
↓
ARCHIVED
```

失败路径：

```text
RUNNING → FAILED
```

允许的状态流转可以是：

```text
QUEUED  → RUNNING
RUNNING → SUCCESS
RUNNING → FAILED
SUCCESS → ARCHIVED
FAILED  → RETRY_WAIT
RETRY_WAIT → RUNNING
```

不允许：

```text
SUCCESS → RUNNING
ARCHIVED → RUNNING
FAILED → SUCCESS（没有经过合法重试流程时）
```

### 1. 条件更新防止状态倒退

错误写法：

```sql
UPDATE task
SET status = 'RUNNING'
WHERE id = ?;
```

无论当前状态是什么，都会被覆盖。

更安全的写法：

```sql
UPDATE task
SET status = 'RUNNING'
WHERE id = ?
  AND status = 'QUEUED';
```

如果数据库已经是：

```text
SUCCESS
```

旧的 `RUNNING` 消息执行时：

```text
status = QUEUED
```

条件不成立，更新行数为 0。

Java 代码需要检查受影响行数：

```java
int affectedRows = taskMapper.markRunning(taskId);

if (affectedRows == 0) {
    log.warn(
            "reject illegal state transition, taskId={}, targetStatus=RUNNING",
            taskId
    );
}
```

### 2. 使用事件序号防止旧事件覆盖新事件

每条消息增加单调递增的事件序号：

```text
QUEUED   eventSeq=1
RUNNING  eventSeq=2
SUCCESS  eventSeq=3
ARCHIVED eventSeq=4
```

数据库保存：

```text
status
last_event_seq
```

更新 SQL：

```sql
UPDATE task
SET status = #{status},
    last_event_seq = #{eventSeq}
WHERE id = #{taskId}
  AND last_event_seq < #{eventSeq};
```

如果数据库已经处理：

```text
SUCCESS eventSeq=3
```

旧消息：

```text
RUNNING eventSeq=2
```

执行时：

```text
3 < 2
```

不成立，因此旧事件无法覆盖新状态。

### 3. 状态机和事件序号的区别

状态机回答：

```text
这个状态能不能变成目标状态？
```

事件序号回答：

```text
这条事件是否比已经处理的事件更新？
```

两者结合更可靠：

```text
状态流转合法
+
事件序号更新
+
业务操作幂等
```

---

## 十一、状态机不能完全代替顺序消费

假设数据库当前状态是：

```text
QUEUED
```

因为消息乱序，`SUCCESS` 先到。

严格状态机判断：

```text
QUEUED → SUCCESS
```

属于非法跳转，因此拒绝。

随后 `RUNNING` 到达并更新成功，数据库最终仍然可能停留在：

```text
RUNNING
```

因此状态机虽然防止了错误跳转，但没有自动恢复缺失的 `SUCCESS`。

可以根据业务选择以下方案：

### 方案一：乱序事件暂存和重试

```text
SUCCESS 提前到达
↓
发现前置状态不满足
↓
写入乱序事件表或延迟队列
↓
等待 RUNNING 完成
↓
重新执行 SUCCESS
```

适用于每一步都必须执行的业务。

### 方案二：使用事件序号接受最新状态

如果状态只是展示任务最新进度，可以允许更高 `eventSeq` 直接覆盖较低版本。

例如：

```text
数据库 QUEUED eventSeq=1
收到 SUCCESS eventSeq=3
↓
直接更新为 SUCCESS eventSeq=3
```

后到的 `RUNNING eventSeq=2` 会被拒绝。

### 方案三：定期对账和修复

任务真实执行结果可能存储在：

```text
任务结果表
对象存储
Worker 执行记录
第三方平台
```

可以通过定时任务对账：

```text
数据库状态仍为 RUNNING
但结果文件已经生成
↓
修复为 SUCCESS
```

---

## 十二、Topic 增加 Partition 的顺序风险

相同 Key 并不意味着它永远进入同一个编号的 Partition。

默认分区逻辑通常类似：

```text
hash(key) % partitionCount
```

如果 Partition 数量从：

```text
4
```

增加到：

```text
8
```

同一个 Key 后续可能映射到新的 Partition。

例如：

```text
扩容前 Task-1001 → Partition 1
扩容后 Task-1001 → Partition 5
```

扩容前旧消息和扩容后新消息分布在不同 Partition，中间没有全局顺序保证。

因此需要严格顺序的 Topic 在扩容 Partition 前应评估：

```text
是否存在尚未消费的旧消息
是否需要暂停生产
是否需要等待旧消息处理完成
是否可以切换到新 Topic
是否有事件序号和状态机兜底
```

Partition 数量增加后通常不能直接缩回，因此不能把增加 Partition 当成没有代价的临时操作。

---

## 十三、完整工程保护链路

可靠的状态消息处理不应只依赖 Kafka Partition 顺序。

更完整的保护链路是：

```text
生产端使用稳定业务 Key
↓
同一业务对象进入同一 Partition
↓
记录 eventId、partition、offset、eventSeq
↓
消费端同 Key 串行、不同 Key 并行
↓
正确管理 offset
↓
数据库状态机条件更新
↓
事件序号防止旧事件覆盖
↓
业务操作幂等
↓
异常事件重试或对账修复
```

其中每一层解决的问题不同：

```text
业务 Key
→ 尽量保证进入同一 Partition

Partition 顺序
→ 保证 Kafka 日志和拉取顺序

Key 串行执行
→ 防止消费线程执行乱序

状态机
→ 阻止非法状态跳转

eventSeq
→ 阻止旧事件覆盖新事件

幂等
→ 防止重复消息产生重复业务结果

对账
→ 修复前面机制仍未覆盖的异常状态
```

---

## 十四、今天思考题复盘

题目：某 AI 平台中，一个任务依次发送：

```text
QUEUED
↓
RUNNING
↓
SUCCESS
↓
ARCHIVED
```

线上偶尔发现：

```text
SUCCESS
↓
RUNNING
```

数据库最终状态变成：

```text
RUNNING
```

### 问题一：可能是生产端还是消费端导致顺序被破坏

不能只判断为消费端。

生产端可能存在：

```text
同一个 Task 使用不同 Key
多个线程并发发送
异步发送或失败重试改变实际写入顺序
多个实例产生旧状态事件
```

消费端可能存在：

```text
线程池并行执行
旧消息重试
重复消费
offset 提交错误
数据库更新缺少状态约束
```

消费端线程池是高概率原因之一，但需要通过日志验证。

### 问题二：如何判断是否进入不同 Partition

不能只根据 Key 推测。

应该在 Producer 和 Consumer 日志中记录：

```text
taskId
key
partition
offset
eventSeq
```

如果两条消息的 Partition 不同，则没有顺序保证。

如果 Partition 相同且 offset 顺序正确，则重点排查消费线程池、重试和数据库更新顺序。

### 问题三：线程池如何兼顾吞吐和顺序

原则是：

> 同一个 Task 串行，不同 Task 并行。

可以使用：

```text
Partition 级并行
```

或者：

```text
按 taskId Hash 分桶
每个桶单线程执行
桶与桶之间并行
```

不能把同一 Partition 的消息无约束地提交给普通线程池。

### 问题四：状态机如何作为最后保护

数据库更新不能直接覆盖状态，应限制合法来源状态：

```sql
UPDATE task
SET status = 'RUNNING'
WHERE id = ?
  AND status = 'QUEUED';
```

同时可以增加 `eventSeq`：

```sql
UPDATE task
SET status = #{status},
    last_event_seq = #{eventSeq}
WHERE id = #{taskId}
  AND last_event_seq < #{eventSeq};
```

这样旧的 `RUNNING` 消息即使晚到，也不能覆盖已经处理的 `SUCCESS`。

---

## 十五、面试表达

可以这样回答：

> Kafka 只能保证同一个 Partition 内消息按照 offset 顺序存储和拉取，不能保证整个 Topic 全局有序，也不能保证消费端业务线程按照相同顺序完成。对于订单状态或 AI 异步任务状态这类场景，我会让同一个业务对象使用稳定的 taskId 或 orderId 作为消息 Key，使其进入同一个 Partition。在消费端不会把同一个 Key 的消息无约束地提交到普通线程池，而是采用 Partition 串行或者按业务 Key 分桶的方式，实现同 Key 串行、不同 Key 并行。
>
> 线上排查乱序时，我会记录 eventId、Key、Partition、Offset 和 eventSeq。如果消息位于同一 Partition 且 Offset 顺序正确，说明问题更可能出在线程池并发、重试、重复消费或数据库更新阶段。数据库层还会通过状态机条件更新、事件序号和幂等设计，防止旧状态覆盖新状态。对于必须经过每一步的业务，提前到达的事件还需要暂存重试，而不能只依赖状态机拒绝。

---

## 十六、今天的最终结论

消息顺序消费不能只理解为：

```text
同一个 Key 发送到同一个 Partition
```

真正完整的工程问题包括：

```text
生产端是否按正确顺序产生事件
消息是否使用稳定业务 Key
消息实际进入哪个 Partition
Partition 内 offset 是否正确
消费端是否发生并发执行乱序
offset 是否正确提交
旧消息是否重试或重复消费
数据库是否允许状态倒退
异常后是否能够恢复和对账
```

今天需要记住的核心结论是：

> Kafka 的 Partition 顺序只是顺序消费的基础，不是业务最终正确性的全部保证。真实系统需要通过稳定业务 Key、同 Key 串行执行、状态机、事件序号、幂等和对账共同保证状态正确。

能力等级：**中级后端能力**。
