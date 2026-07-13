# 后端工程化每日训练 Day 06：分布式锁复盘

## 一、今天学习的知识点

今天学习的是：

**分布式锁（Distributed Lock）**

一句话概括：

> 分布式锁用于保证多个服务实例同时运行时，同一份业务资源在同一时刻只能被一个实例处理。

它主要解决的是分布式环境下的**并发竞争问题**。

---

## 二、为什么需要分布式锁

假设 AI 异步任务平台部署了三个 Worker：

```text
Worker-1
Worker-2
Worker-3
```

因为 MQ 重复投递，同一个任务：

```text
Task-1001
```

同时被三个 Worker 收到。

如果系统没有任何并发控制，三个 Worker 都可能执行：

```text
调用 GPT
保存结果
修改任务状态
产生费用
```

最终可能导致：

```text
同一个任务调用 GPT 三次
同一个结果保存三次
同一笔费用产生三次
```

分布式锁的目标就是：

```text
三个 Worker 同时竞争同一把锁

只有一个 Worker 获取成功

获取成功的 Worker 执行业务

其他 Worker 直接退出
```

---

## 三、为什么 `synchronized` 不够

Java 中的：

```java
synchronized
```

只能保证当前 JVM 内部的线程互斥。

例如系统部署在两台服务器：

```text
服务器 A：Worker-1
服务器 B：Worker-2
```

服务器 A 中的 `synchronized` 无法限制服务器 B。

因为两个服务实例拥有：

```text
不同的 JVM
不同的内存
不同的锁对象
```

所以在分布式部署环境下，需要将锁放在所有实例都可以访问的公共组件中。

常见实现包括：

```text
Redis
数据库
ZooKeeper
```

实际 Java 项目中，Redis 和 Redisson 比较常见。

---

## 四、为什么普通数据库状态判断不一定可靠

一开始我的理解是：

> 三个消费者是相互独立的，如果没有做限制，当然可能同时执行。

这个方向没有问题，但真正的技术原因是：

```text
查询状态
判断状态
修改状态
```

这几个操作不是一个原子操作。

假设任务初始状态是：

```text
CREATED
```

三个 Worker 几乎同时查询：

```text
Worker-1 查询到 CREATED
Worker-2 查询到 CREATED
Worker-3 查询到 CREATED
```

此时三个 Worker 都认为：

```text
当前任务可以执行
```

然后都继续修改状态并调用 GPT。

所以，下面这种逻辑存在并发问题：

```java
Task task = taskMapper.selectById(taskId);

if ("CREATED".equals(task.getStatus())) {
    taskMapper.updateStatus(taskId, "RUNNING");
    callGpt();
}
```

问题在于：

```text
查询和更新之间存在时间窗口
```

多个 Worker 可能同时通过判断。

---

## 五、数据库条件更新也可以抢占任务

不是所有并发问题都必须使用 Redis 分布式锁。

对于简单的任务抢占，可以直接使用数据库条件更新：

```sql
UPDATE ai_task
SET status = 'RUNNING',
    worker_id = ?
WHERE id = ?
  AND status = 'CREATED';
```

然后判断影响行数：

```text
affectedRows = 1
```

说明当前 Worker 抢占成功，可以继续执行。

```text
affectedRows = 0
```

说明任务已经被其他 Worker 抢走，当前 Worker 直接退出。

示例：

```java
int affectedRows = taskMapper.markRunning(taskId, workerId);

if (affectedRows == 0) {
    return;
}

callGpt();
```

数据库会保证：

```text
同一条任务记录

只能有一个 Worker

成功从 CREATED 更新为 RUNNING
```

因此，是否使用分布式锁需要根据业务复杂度判断。

### 数据库条件更新适合

```text
单表任务抢占
状态变更简单
并发量可以接受
临界区较短
```

### 分布式锁适合

```text
需要保护一整段复杂业务
涉及多个数据库操作
涉及多个外部服务
涉及跨资源并发控制
```

---

## 六、Redis 分布式锁如何设计

### 1. 锁 Key

锁 Key 应该唯一表示当前正在保护的业务资源。

AI 任务场景可以设计为：

```text
lock:ai-task:1001
```

通用格式：

```text
lock:{业务类型}:{业务ID}
```

例如：

```text
lock:payment-order:ORDER1001
lock:member-benefit:ORDER1001
lock:report:2026-07-13
lock:ai-task:1001
```

设计原则：

```text
同一个业务资源必须生成相同的 Key
不同业务资源必须生成不同的 Key
```

---

### 2. 锁 Value

锁 Value 不能简单地写成：

```text
1
```

应该保存当前锁持有者的唯一标识，例如：

```text
UUID
```

或者：

```text
UUID + Worker ID
```

例如：

```text
550e8400-e29b-41d4-a716-446655440000:worker-2
```

Value 的作用是证明：

```text
当前这把锁是谁创建的
```

释放锁时需要先判断：

```text
Redis 中的 Value
是否等于当前 Worker 持有的 Value
```

只有一致时，当前 Worker 才能删除锁。

否则可能误删其他 Worker 后来获得的新锁。

---

### 3. 锁过期时间

锁必须设置过期时间。

如果没有过期时间：

```text
Worker 获取锁
↓
Worker 突然宕机
↓
锁无法主动释放
↓
其他 Worker 永远无法处理任务
```

这会形成死锁。

过期时间不能太短，也不能无限长。

假设 GPT 调用时间：

```text
平均 30 秒
大部分请求 60 秒内完成
极端情况可能需要 180 秒
```

可以考虑设置：

```text
3～5 分钟
```

但固定过期时间仍然存在问题：

```text
业务还没有执行完
锁却已经过期
```

此时其他 Worker 可能重新获取锁并开始执行。

因此，业务执行时间不确定时，可以使用 Redisson 的自动续期机制。

---

## 七、Redis 加锁的基本流程

Redis 常见加锁命令本质上是：

```text
SET key value NX PX
```

含义：

```text
NX：只有 Key 不存在时才设置成功
PX：设置毫秒级过期时间
```

完整流程：

```text
Worker 收到任务
↓
尝试创建锁
↓
创建成功
↓
执行业务
↓
释放锁
```

如果创建失败：

```text
说明其他 Worker 已经持有锁
↓
当前 Worker 直接退出
```

---

## 八、为什么必须原子释放锁

错误做法：

```java
if (value.equals(redisTemplate.opsForValue().get(lockKey))) {
    redisTemplate.delete(lockKey);
}
```

虽然看起来先比较 Value，再删除锁，但这两个操作不是原子的。

可能发生：

```text
Worker A 查询 Value，发现是自己的
↓
Worker A 暂停
↓
锁过期
↓
Worker B 获得新锁
↓
Worker A 恢复执行
↓
Worker A 删除了 Worker B 的锁
```

所以：

```text
比较 Value
删除 Key
```

必须作为一个原子操作执行。

通常可以使用：

```text
Lua 脚本
```

或者直接使用：

```text
Redisson
```

Lua 脚本示例：

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

---

## 九、Worker 调用 GPT 时突然宕机怎么办

这个问题需要分成两部分处理。

### 1. Redis 锁怎么办

如果锁设置了过期时间：

```text
Worker 宕机
↓
无法主动释放锁
↓
锁到期后自动删除
```

如果使用 Redisson 自动续期：

```text
Worker 正常运行
↓
Redisson 定期续期

Worker 宕机
↓
续期停止
↓
锁最终自动过期
```

所以锁本身不会永久存在。

---

### 2. 数据库中的 `RUNNING` 状态怎么办

这是比分布式锁本身更容易忽略的问题。

Worker 可能已经执行：

```text
status = RUNNING
```

然后突然宕机。

即使 Redis 锁已经过期，数据库中的任务仍然可能一直保持：

```text
RUNNING
```

如果没有恢复机制，这个任务可能永远卡死。

因此任务表可以增加：

```text
worker_id
started_at
heartbeat_at
retry_count
```

例如：

```sql
CREATE TABLE ai_task (
    id BIGINT PRIMARY KEY,
    status VARCHAR(32) NOT NULL,
    worker_id VARCHAR(64),
    started_at DATETIME,
    heartbeat_at DATETIME,
    retry_count INT NOT NULL DEFAULT 0
);
```

Worker 执行过程中定期更新：

```text
heartbeat_at
```

定时任务扫描长时间没有心跳的任务：

```sql
SELECT *
FROM ai_task
WHERE status = 'RUNNING'
  AND heartbeat_at < NOW() - INTERVAL 5 MINUTE;
```

再根据业务规则进行处理：

```text
改为 RETRY
重新投递 MQ
```

或者：

```text
改为 FAILED
进入死信队列
等待人工处理
```

---

## 十、GPT 调用成功后 Worker 宕机的问题

还有一种更加复杂的情况：

```text
Worker 调用 GPT 成功
↓
GPT 已经返回结果
↓
Worker 还没有保存数据库
↓
Worker 突然宕机
```

系统恢复后只能看到：

```text
任务仍然是 RUNNING
或者任务执行超时
```

如果直接重试：

```text
GPT 会再次被调用
```

可能导致：

```text
重复调用
重复扣费
结果不一致
```

所以外部接口调用必须考虑：

```text
调用结果未知
```

可以采取的措施：

```text
为每次调用生成唯一 requestId
记录调用尝试日志
记录请求参数和调用状态
支持查询上游调用结果
限制最大重试次数
不确定状态进入人工处理
```

任务调用记录可以设计为：

```text
task_id
request_id
provider
request_status
request_time
response_time
error_message
```

---

## 十一、为什么分布式锁不能代替幂等

这是今天最重要的区别。

### 分布式锁解决的是

```text
多个请求同时执行
```

也就是：

```text
并发互斥
```

它保证的是：

> 同一个时间点，只有一个执行者进入临界区。

---

### 幂等解决的是

```text
同一个请求先后重复执行
```

它保证的是：

> 同一个业务请求执行多次，最终只产生一次有效业务结果。

---

### 会员权益发放案例

第一次 MQ 消息到达：

```text
Worker-1 获取锁
↓
发放会员
↓
业务执行成功
↓
锁释放
```

但因为 ACK 发送失败，MQ 重新投递消息：

```text
Worker-2 再次收到消息
↓
此时旧锁已经释放
↓
Worker-2 可以重新获得锁
↓
再次发放会员
```

分布式锁没有失效。

它已经正确保证了：

```text
两个 Worker 没有同时执行
```

但它无法阻止：

```text
两个 Worker 先后各执行一次
```

所以仍然需要幂等。

---

## 十二、幂等如何防止重复执行

以会员权益发放为例，可以建立唯一索引：

```sql
UNIQUE KEY uk_order_benefit (
    order_id,
    benefit_type
);
```

第一次处理：

```text
插入权益记录成功
```

第二次处理：

```text
再次插入相同订单和权益类型
↓
触发唯一键冲突
↓
不重复发放
```

也可以在业务执行前查询：

```text
当前订单是否已经发放权益
```

但仅查询仍然可能存在并发问题，数据库唯一索引才是最终兜底。

完整设计通常是：

```text
分布式锁
+
业务状态判断
+
数据库唯一索引
+
幂等记录
```

---

## 十三、AI 异步任务完整执行流程

一个相对完整的流程可以设计为：

```text
MQ 投递任务
↓
Worker 收到消息
↓
检查任务是否已经 SUCCESS
↓
尝试获取分布式锁
↓
获取失败，直接退出
↓
获取成功
↓
再次检查任务状态
↓
通过数据库条件更新抢占任务
↓
修改状态为 RUNNING
↓
记录 worker_id、started_at、heartbeat_at
↓
生成唯一 requestId
↓
调用 GPT
↓
保存调用结果
↓
修改任务状态为 SUCCESS
↓
发送 ACK
↓
释放分布式锁
```

如果调用失败：

```text
记录失败原因
↓
retry_count + 1
↓
判断是否可以重试
```

超过最大重试次数：

```text
修改为 FAILED
↓
进入死信队列
↓
人工处理
```

---

## 十四、Redisson 最小示例

```java
String lockKey = "lock:ai-task:" + taskId;

RLock lock = redissonClient.getLock(lockKey);

boolean locked = false;

try {
    locked = lock.tryLock(
        0,
        5,
        TimeUnit.MINUTES
    );

    if (!locked) {
        return;
    }

    int affectedRows = taskMapper.markRunning(taskId, workerId);

    if (affectedRows == 0) {
        return;
    }

    gptService.execute(taskId);

    taskMapper.markSuccess(taskId);

} catch (Exception e) {

    taskMapper.markFailed(taskId, e.getMessage());

    throw e;

} finally {

    if (locked && lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

需要注意：

```text
固定 5 分钟租约适合执行时间相对可预测的业务
```

如果执行时间不可预测，可以使用 Redisson 看门狗机制，但仍然需要设置：

```text
GPT 请求超时
任务执行超时
最大重试次数
```

不能让任务无限执行。

---

## 十五、今天纠正的几个认识

### 1. 三个消费者独立只是表面原因

真正的并发问题在于：

```text
查询、判断、修改不是原子操作
```

---

### 2. 数据库状态并不是完全没用

普通查询后更新存在问题，但：

```text
带状态条件的 UPDATE
+
判断影响行数
```

可以实现任务抢占。

---

### 3. 锁过期不等于任务恢复

Redis 锁过期后，数据库任务可能仍然停留在：

```text
RUNNING
```

所以还需要：

```text
心跳
超时扫描
状态恢复
重试机制
```

---

### 4. 分布式锁和幂等解决的问题不同

```text
分布式锁：解决同时执行
幂等：解决重复执行
```

二者经常同时使用，但不能互相替代。

---

## 十六、面试表达

可以这样回答：

> 在分布式部署环境下，同一个任务可能因为 MQ 重复投递或调度重复，被多个服务实例同时处理。`synchronized` 只能保证单 JVM 内互斥，无法限制其他实例，因此可以使用 Redis 或 Redisson 实现分布式锁。通常以业务 ID 作为锁 Key，以 UUID 或 Worker ID 作为锁 Value，并设置合理的过期时间，释放锁时必须校验锁持有者，防止误删其他实例的锁。对于任务抢占，也可以通过数据库条件更新和影响行数实现。分布式锁只能解决并发互斥，不能解决 ACK 丢失后产生的先后重复执行，因此支付、会员权益、AI 任务等场景仍然需要结合幂等、唯一索引、状态机、重试和超时恢复机制。

---

## 十七、今日核心总结

```text
synchronized
只能解决单 JVM 并发
```

```text
分布式锁
解决多个实例同时执行同一份业务
```

```text
锁 Key
用于标识被保护的业务资源
```

```text
锁 Value
用于标识锁持有者
```

```text
过期时间
用于防止 Worker 宕机后形成永久死锁
```

```text
Lua 或 Redisson
用于保证锁的安全释放
```

```text
数据库条件更新
可以实现简单任务抢占
```

```text
心跳和超时扫描
用于恢复长期停留在 RUNNING 的任务
```

```text
分布式锁
防止同时执行
```

```text
幂等
防止先后重复执行
```

最终需要记住的一句话：

> 分布式锁只能保证某一个时刻只有一个执行者，不能保证同一项业务永远只执行一次。
