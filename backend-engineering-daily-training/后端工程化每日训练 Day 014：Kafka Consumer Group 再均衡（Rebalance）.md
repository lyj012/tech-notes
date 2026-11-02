# 后端工程化每日训练 Day 14：Kafka Consumer Group 再均衡（Rebalance）

## 一、今天学习的知识点

今天学习的是：

**Kafka Consumer Group 再均衡（Rebalance）**

Rebalance 可以理解为：

> 当 Consumer Group 的成员、订阅关系或 Topic 的 Partition 发生变化时，Kafka 重新计算每个 Consumer 应该负责哪些 Partition。

例如：

```text
Topic：6 个 Partition
Consumer Group：Consumer A、B、C

Partition 0、1 → Consumer A
Partition 2、3 → Consumer B
Partition 4、5 → Consumer C
```

如果 Consumer B 宕机，它负责的 Partition 2、3 必须重新分配给其他 Consumer，否则这些 Partition 会持续积压。

Rebalance 本身是 Kafka 保证故障接管和动态扩缩容的必要机制，但频繁 Rebalance 会造成消费暂停、Lag 上升、吞吐下降和重复消费。

---

## 二、Consumer Group、Partition 和 Consumer

同一个 Consumer Group 中：

> 一个 Partition 在同一时刻只能由一个 Consumer 负责。

例如 4 个 Partition、2 个 Consumer：

```text
Consumer A → Partition 0、1
Consumer B → Partition 2、3
```

如果增加到 4 个 Consumer，可以形成 4 路 Partition 级并行。

如果增加到 8 个 Consumer，仍然最多只有 4 个 Consumer 真正工作，因为有效并行度通常不会超过 Partition 数量。

所以只增加 Consumer 实例，不考虑 Partition 数量，可能无法提高吞吐。

---

## 三、哪些情况会触发 Rebalance

常见触发条件包括：

```text
新增 Consumer 实例
Consumer 正常下线
Consumer 进程崩溃或网络中断
Consumer 长时间没有调用 poll()
Topic 增加 Partition
Consumer 订阅范围变化
Pod 频繁重启或实例频繁扩缩容
```

简化过程：

```text
Consumer Group 成员或 Partition 变化
↓
Group Coordinator 检测到变化
↓
撤销旧 Partition 分配
↓
重新计算分配关系
↓
Consumer 获得新的 Partition
↓
从已提交的 offset 继续消费
```

Rebalance 期间，正常消费会暂停或明显减少，因此 Lag 可能上升。

---

## 四、Group Coordinator

每个 Consumer Group 都由某个 Kafka Broker 上的 Group Coordinator 负责协调。

它主要负责：

```text
维护 Consumer Group 成员
接收 Consumer 心跳
检测成员加入和退出
协调 Rebalance
管理 Consumer Group 的 offset 提交
```

Consumer 通过心跳表示自己仍然存活，同时还必须持续调用 `poll()`，表示自己仍然能够正常推进消费循环。

---

## 五、`session.timeout.ms` 和 `max.poll.interval.ms`

### `session.timeout.ms`

主要判断 Consumer 是否还活着。

如果 Kafka 长时间收不到心跳，就认为 Consumer 已经失效。

常见原因：

```text
进程崩溃
网络中断
长时间 Stop-The-World
Consumer 与 Broker 连接异常
```

### `max.poll.interval.ms`

主要判断 Consumer 是否还能正常消费。

即使进程仍然活着，如果两次 `poll()` 之间间隔过长，Kafka 也会认为 Consumer 无法正常继续消费，并将它移出 Consumer Group。

两个参数不一定同时超时。

今天题目中的直接原因是：

```text
任务耗时 8～15 分钟
>
max.poll.interval.ms 5 分钟
```

Consumer 可能仍然能够发送心跳，因此 `session.timeout.ms` 不一定超时；但超过 `max.poll.interval.ms` 已经足以触发 Rebalance。

---

## 六、真实业务场景：AI 异步任务

Kafka 中有一条任务消息：

```text
taskId = 10001
taskType = PDF_OCR
```

Consumer 收到消息后同步执行：

```text
下载 PDF
↓
调用 OCR
↓
调用 AI 模型
↓
保存结果
↓
更新任务状态
```

任务耗时 8～15 分钟，而 `max.poll.interval.ms` 只有 5 分钟。

结果：

```text
Consumer A poll 到任务
↓
同步执行长任务
↓
5 分钟内没有再次 poll
↓
超过 max.poll.interval.ms
↓
Consumer A 被移出组
↓
触发 Rebalance
↓
Partition 交给 Consumer B
```

如果 Consumer A 仍然在后台执行，Consumer B 又从未提交的 offset 重新读取，就可能导致同一个 AI 任务执行两次。

---

## 七、为什么 Kafka Lag 会持续波动

Rebalance 和 Lag 波动的关系：

```text
Rebalance 开始
↓
有效消费暂停或减少
↓
Producer 继续写入
↓
Lag 上升
↓
Rebalance 完成
↓
消费恢复
↓
Lag 下降
```

如果长任务不断超过 `max.poll.interval.ms`，就会反复出现：

```text
消费
→ poll 超时
→ Rebalance
→ 恢复
→ 再次超时
→ 再次 Rebalance
```

因此 Lag 上升不一定只是 Consumer 性能不足，也可能是频繁 Rebalance 压缩了有效消费时间。

---

## 八、为什么同一个任务会执行两次

假设：

```text
Consumer A 读取 offset=100
↓
开始执行任务
↓
AI 调用和数据库更新已经成功
↓
offset 还没有提交
↓
发生 Rebalance 或进程崩溃
↓
Consumer B 接管 Partition
```

如果最后已提交的位置仍然是 offset=99，Consumer B 会再次读取 offset=100。

核心原因是：

```text
业务执行成功
```

和：

```text
offset 提交成功
```

不是一个原子操作。

Kafka 常见可靠消费语义是至少一次：消息不会轻易丢失，但可能重复。

对于 AI 任务，重复执行可能造成：

```text
重复消耗模型 Token
重复调用付费 OCR
重复生成结果文件
重复上传对象存储
重复扣费
重复发送通知
```

---

## 九、自动提交 offset 的主要风险

自动提交的核心风险是：

> 业务还没有真正完成，offset 已经被推进。

例如：

```text
Consumer poll 到消息
↓
任务交给线程池
↓
Consumer 继续 poll
↓
Kafka 自动提交 offset
↓
AI 任务仍在执行
↓
进程崩溃或任务失败
```

服务恢复后，Kafka 认为这条消息已经完成，不会再次投递，但实际业务结果没有成功落地，最终造成任务丢失。

---

## 十、自动提交和乱序提交的区别

我一开始把自动提交风险回答成：

```text
offset 100、101、102 并行执行
102 先完成
直接提交 102
```

这个风险存在，但它属于同一 Partition 内并行处理导致的乱序提交问题，不是自动提交的直接定义。

### 自动提交风险

```text
业务未完成
↓
offset 已自动推进
↓
服务崩溃后任务不再重投
```

### 并行乱序提交风险

```text
同一 Partition 的后续 offset 先完成
↓
错误提交较大 offset
↓
跨过前面尚未完成的消息
```

两者都可能造成任务丢失，但原因不同。

---

## 十一、为什么不能只把 `max.poll.interval.ms` 无限调大

把 5 分钟提高到 20 分钟，可以降低正常长任务触发 Rebalance 的概率。

但如果无限调大：

```text
Consumer 真正卡死
↓
Kafka 很久以后才发现
↓
Partition 长时间无法被其他 Consumer 接管
↓
故障恢复变慢
```

参数应根据任务耗时 P95/P99、网络波动、GC 停顿和故障恢复要求设置，而不能代替架构改造。

---

## 十二、最小配置调整

```yaml
spring:
  kafka:
    consumer:
      enable-auto-commit: false
      max-poll-records: 1
      properties:
        max.poll.interval.ms: 1200000
    listener:
      ack-mode: manual
```

含义：

```text
enable-auto-commit: false
→ 关闭自动提交

max-poll-records: 1
→ 每次只拉取少量长任务

max.poll.interval.ms: 1200000
→ 两次 poll 最长间隔示例为 20 分钟

ack-mode: manual
→ 业务成功后手动确认
```

这个配置只是示例，不能直接照搬。

手动提交也不等于精确一次，因为仍然存在：

```text
业务已经成功
↓
offset 提交前进程崩溃
↓
消息再次投递
```

---

## 十三、任务执行架构如何调整

### 方案一：同步执行长任务

```text
poll
↓
执行 AI 任务
↓
业务成功
↓
提交 offset
↓
再次 poll
```

优点是流程简单、offset 边界清晰。

缺点是 Consumer 长时间占用 Partition，容易超过 `max.poll.interval.ms`，吞吐和故障恢复能力较差。

### 方案二：交给内存线程池

```text
Kafka Consumer
↓
提交线程池
↓
Consumer 继续 poll
```

这种方案可以避免 poll 线程长期阻塞，但会引入：

```text
offset 何时提交
同一 Partition 完成顺序
线程池满了怎么办
进程退出时任务是否丢失
Rebalance 时未完成任务怎么办
```

不能简单设计为：

```text
poll
→ 丢进线程池
→ 立即提交 offset
```

否则进程崩溃时，任务可能永久丢失。

### 方案三：Consumer 可靠落库 + Worker 执行

更适合 AI、OCR、PDF 转换等长任务：

```text
Kafka Consumer
↓
根据 taskId 校验和幂等落库
↓
提交 Kafka offset
↓
Worker 从任务表领取任务
↓
执行 AI / OCR / PDF 转换
↓
更新任务状态
```

Kafka Consumer 只执行短操作，长耗时任务由独立 Worker 处理。

任务状态可以设计为：

```text
WAITING
→ RUNNING
→ SUCCESS

WAITING
→ RUNNING
→ FAILED
→ RETRY_WAIT
```

同时可以增加：

```text
retry_count
worker_id
lease_expire_time
next_retry_time
last_error
```

Worker 崩溃后，可以通过执行租约超时重新恢复任务。

---

## 十四、任务幂等怎么做

使用 `taskId` 作为业务幂等键，并建立唯一约束：

```sql
ALTER TABLE ai_task
ADD CONSTRAINT uk_ai_task_task_id UNIQUE (task_id);
```

Consumer 收到重复消息时：

```text
第一次 taskId 到达
→ 插入任务成功

第二次同一个 taskId 到达
→ 唯一约束冲突
→ 识别为重复消息
→ 不创建第二条任务
```

同时结合任务状态判断：

```text
SUCCESS
→ 直接跳过

RUNNING 且租约有效
→ 不重复执行

FAILED 且允许重试
→ 进入重试流程

RUNNING 但租约过期
→ 允许故障恢复
```

幂等不代表代码绝对只运行一次，而是同一个请求执行多次，最终有效业务结果仍然只出现一次。

---

## 十五、为什么调整参数后仍然必须幂等

以下情况仍然可能造成重复：

```text
业务成功但 offset 提交前崩溃
Rebalance 发生在提交之前
offset 提交响应丢失
Producer 因确认不确定而重试发送
管理员重复执行人工补偿
```

因此最终可靠性必须落到业务层：

```text
同一个 taskId 无论到达多少次
最终只能产生一次有效业务结果
```

---

## 十六、Rebalance 风暴排查

如果日志频繁出现 Rebalance，需要检查：

```text
max.poll.interval.ms exceeded
heartbeat timeout
Consumer 加入和退出日志
Pod 重启次数
OOMKilled
健康检查失败
网络异常
Full GC
部署和扩缩容记录
单条任务 P95/P99 耗时
```

监控指标包括：

```text
Consumer Group 状态
Rebalance 次数和持续时间
每个 Partition 的 Lag
poll 调用间隔
offset 提交失败次数
Consumer 实例重启次数
线程池队列长度和拒绝次数
重复 taskId 数量
幂等拦截次数
```

---

## 十七、思考题完整回答

### 1. 为什么消费者频繁 Rebalance

单个任务耗时 8～15 分钟，超过 `max.poll.interval.ms` 的 5 分钟。Consumer 长时间没有再次调用 `poll()`，Kafka 将它移出 Consumer Group，触发 Partition 重新分配。

`session.timeout.ms` 不一定同时超时。

### 2. 为什么同一个 AI 任务执行两次

Consumer A 已经开始或完成任务，但 offset 尚未提交；发生 Rebalance 后，Consumer B 从最后已提交的 offset 继续消费，再次读取同一条消息。

### 3. 自动提交还有什么风险

自动提交可能在业务完成前推进 offset。如果任务执行失败或进程崩溃，Kafka 不会再投递该消息，造成任务丢失。

### 4. 如何调整配置和架构

短期：

```text
关闭自动提交
降低 max.poll.records
根据任务 P99 合理提高 max.poll.interval.ms
业务成功后手动提交
```

长期：

```text
Kafka Consumer 只负责幂等落库
独立 Worker 执行长任务
配合任务状态机、重试、执行租约和超时恢复
```

### 5. 为什么必须保留幂等

Kafka 参数只能降低重复概率，无法消除业务成功后提交前崩溃、Rebalance、提交响应丢失和 Producer 重试等情况，所以必须通过 `taskId`、唯一约束和状态机保证最终业务结果只生效一次。

---

## 十八、面试表达

> 在 Kafka 异步任务系统中，我会关注 Consumer Group Rebalance。Consumer 加入、退出、心跳超时或长时间没有调用 `poll()`，都可能触发 Partition 重新分配。
>
> 对于 AI、OCR、PDF 转换这类 8～15 分钟的长耗时任务，如果直接在消费线程中同步执行，而 `max.poll.interval.ms` 只有 5 分钟，Consumer 会因为 poll 间隔超时被移出组，导致频繁 Rebalance、Lag 波动和重复消费。
>
> 配置层面我会关闭自动提交，减少 `max.poll.records`，根据任务 P99 合理设置 `max.poll.interval.ms`，并在业务成功后手动提交 offset。但参数调整只能降低风险，不能保证业务精确一次。
>
> 架构上我更倾向于让 Kafka Consumer 只负责根据 `taskId` 幂等落库，再由独立 Worker 通过任务状态机、执行租约、重试和超时恢复执行长任务。同时使用唯一约束和原子状态更新，避免重复调用模型、重复扣费或重复生成业务结果。

---

## 十九、今天最大的认知修正

### 第一

原来的理解：

```text
频繁 Rebalance 是两个超时参数都超了
```

修正后：

```text
两个参数不一定同时超时
本题直接原因是超过 max.poll.interval.ms
```

### 第二

原来的理解：

```text
自动提交风险就是 102 比 100、101 先完成
```

修正后：

```text
自动提交的核心风险是业务未完成但 offset 已推进
102 先完成属于并行乱序提交问题
```

### 第三

原来的理解：

```text
手动提交就不会重复消费
```

修正后：

```text
业务成功但提交前崩溃仍然会重复
```

### 第四

原来的理解：

```text
调大 max.poll.interval.ms 就能彻底解决
```

修正后：

```text
调参只能止血
长耗时任务需要任务执行架构解耦
```

### 第五

原来的理解：

```text
幂等就是代码只执行一次
```

修正后：

```text
代码可能重复执行
但最终只能产生一次有效业务结果
```

---

## 二十、最终理解

> Rebalance 是 Kafka Consumer Group 在消费者成员、订阅范围或 Topic Partition 发生变化时，重新分配 Partition 的必要机制。它保证 Consumer 宕机后，其负责的 Partition 可以被其他 Consumer 接管。
>
> 对于 AI、OCR、PDF 转换等长耗时任务，如果 Consumer 同步执行 8～15 分钟，而 `max.poll.interval.ms` 只有 5 分钟，即使进程仍然存活，也可能因为长时间没有再次调用 `poll()` 被移出 Consumer Group。频繁发生时会形成 Rebalance 风暴，导致 Lag 波动、吞吐下降和重复消费。
>
> 自动提交可能造成业务未完成但 offset 已推进，从而丢任务；业务完成后再手动提交可以降低丢失风险，但业务成功和 offset 提交之间仍然存在故障窗口，因此可能重复消费。
>
> 对长耗时任务，短期可以关闭自动提交、减少 `max.poll.records` 并合理提高 `max.poll.interval.ms`。长期更稳妥的方案是让 Kafka Consumer 只负责根据 `taskId` 幂等落库，再由独立 Worker 通过任务状态机、原子领取、执行租约、重试和超时恢复完成真正的 AI 任务。
>
> Kafka 参数和手动提交不能替代业务幂等。最终需要通过 `taskId` 唯一约束、状态判断、原子更新和下游幂等键，保证同一个任务即使被重复投递或执行，也只能产生一次有效业务结果。
