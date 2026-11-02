# 后端工程化每日训练 Day 09：状态机与异步任务状态流转复盘

## 一、今日学习主题

今天学习的知识点是：

> **状态机（State Machine）**

状态机用于定义一个业务对象具有哪些状态，以及这些状态之间允许如何流转。

在简单项目中，状态可能只是数据库中的一个 `status` 字段；但随着业务逐渐复杂，如果没有统一的状态流转规则，不同代码就可能随意修改状态，最终造成非法流转、并发冲突和数据混乱。

状态机的核心思想是：

```text
读取当前状态
↓
判断目标状态是否合法
↓
合法才允许更新
↓
记录状态变化
```

而不是直接执行：

```java
task.setStatus(TaskStatus.SUCCESS);
```

---

## 二、状态机解决的核心问题

以 AI 异步任务执行平台为例，任务可能包含以下状态：

```text
CREATED
QUEUED
RUNNING
SUCCESS
FAILED
TIMEOUT
```

正常的状态流转可能是：

```text
CREATED
↓
QUEUED
↓
RUNNING
├─→ SUCCESS
├─→ FAILED
└─→ TIMEOUT
```

如果没有状态机约束，系统中可能出现：

```text
SUCCESS → RUNNING
```

```text
RUNNING → CREATED
```

```text
QUEUED → SUCCESS
```

这些状态变化可能绕过正常的业务流程，使数据库中的状态无法真实反映任务执行情况。

因此，状态机主要解决：

1. 哪些状态可以互相转换；
2. 哪些状态不能回退；
3. 哪些操作允许在当前状态下执行；
4. 多个线程并发修改状态时，谁能够修改成功；
5. 任务异常后应该进入失败、超时还是重试状态；
6. 如何还原任务完整的执行生命周期。

---

## 三、我最初的理解

一开始我认为状态机比较简单：

```text
CREATED → QUEUED → RUNNING → SUCCESS
```

任务执行失败时：

```text
RUNNING → FAILED
```

Worker 崩溃后，如果任务长时间停留在 `RUNNING`，可以设置一个超时时间，例如半小时或者一小时，超过时间后将任务判定为失败。

对于两个 Worker 同时修改状态的问题，我最初想到的是：

```text
使用Redis分布式锁
↓
让两个Worker排队修改
```

对于重新执行任务，我最初认为所有状态似乎都可以重试。

但是继续分析后发现，状态机的概念虽然简单，真正落地时会同时涉及：

```text
事务
幂等
并发控制
条件更新
超时检测
自动重试
MQ重复消费
状态日志
```

复杂的不是状态名称本身，而是状态变化过程中可能发生的异常。

---

## 四、Worker 崩溃后的超时机制

### 4.1 问题场景

Worker 从消息队列中获取任务后，将任务状态修改为：

```text
RUNNING
```

如果 Worker 在执行过程中宕机，任务就可能永远停留在 `RUNNING`。

仅仅设置一个 `RUNNING` 状态无法发现这个问题，因此需要增加超时检测机制。

---

### 4.2 状态设计

Worker 崩溃和业务执行失败不是同一种异常。

业务明确返回错误：

```text
RUNNING → FAILED
```

Worker 宕机或者任务长期没有进展：

```text
RUNNING → TIMEOUT
```

超时后可以根据重试次数决定下一步：

```text
TIMEOUT
↓
RETRYING
↓
QUEUED
↓
RUNNING
```

超过最大重试次数后：

```text
TIMEOUT → FINAL_FAILED
```

这样可以区分：

* 普通业务失败；
* Worker 崩溃；
* 执行超时；
* 可以继续重试的任务；
* 已经无法继续自动重试的任务。

---

### 4.3 需要记录的字段

任务表可以增加：

```text
started_at
heartbeat_at
timeout_at
retry_count
max_retry_count
worker_id
```

字段作用：

| 字段                | 作用                 |
| ----------------- | ------------------ |
| `started_at`      | 任务开始执行的时间          |
| `heartbeat_at`    | Worker 最近一次上报心跳的时间 |
| `timeout_at`      | 任务允许执行到什么时间        |
| `retry_count`     | 当前已经重试的次数          |
| `max_retry_count` | 最大重试次数             |
| `worker_id`       | 当前执行任务的 Worker     |

---

### 4.4 心跳检测

Worker 执行长任务时，可以定期更新：

```text
heartbeat_at
```

例如每隔 30 秒更新一次。

定时任务每分钟扫描长时间没有心跳的任务：

```sql
SELECT id, status, heartbeat_at
FROM ai_task
WHERE status = 'RUNNING'
  AND heartbeat_at < NOW() - INTERVAL 5 MINUTE;
```

扫描到异常任务后，不能直接无条件修改状态，而应该使用条件更新：

```sql
UPDATE ai_task
SET status = 'TIMEOUT'
WHERE id = ?
  AND status = 'RUNNING';
```

必须保留：

```sql
AND status = 'RUNNING'
```

因为在定时任务扫描完成后，Worker 可能恰好已经执行成功。

如果不校验旧状态，就可能出现：

```text
Worker：RUNNING → SUCCESS
定时任务：SUCCESS → TIMEOUT
```

条件更新可以避免定时任务覆盖已经成功的状态。

---

## 五、哪些状态允许重试

我最初认为所有状态似乎都可以重试，但这个判断不准确。

首先需要区分：

### 重试

原任务因为异常没有完成，在原有任务流程上再次尝试执行。

### 重新执行

任务已经正常结束，但用户希望使用相同参数重新运行一次。

这两个概念不能混在一起。

---

### 5.1 CREATED

```text
CREATED → QUEUED
```

这是正常提交任务，不属于重试。

因此 `CREATED` 状态不需要点击重试。

---

### 5.2 QUEUED

`QUEUED` 表示任务已经进入消息队列等待消费。

此时不应该重试，否则可能再次发送一条 MQ 消息，造成重复消费。

---

### 5.3 RUNNING

`RUNNING` 表示任务正在执行。

此时不应该重试，否则可能导致两个 Worker 同时处理同一个任务。

---

### 5.4 SUCCESS

`SUCCESS` 表示任务已经成功完成，因此不能叫作重试。

如果用户需要再次运行，应创建一条新任务：

```text
原任务：SUCCESS
新任务：CREATED
```

新任务可以通过以下字段和原任务关联：

```text
source_task_id
parent_task_id
retry_of_task_id
```

这样既可以重新执行，又不会破坏原任务已经成功的事实。

---

### 5.5 FAILED

`FAILED` 不一定都允许重试，需要判断失败原因。

可以重试的失败：

```text
网络连接失败
模型接口超时
第三方服务暂时不可用
数据库短暂异常
```

这些问题可能在下一次执行时恢复。

不应该重试的失败：

```text
请求参数错误
文件已经损坏
文件格式不支持
用户没有权限
任务数据缺失
```

这些问题重复执行也不会成功。

因此可以增加：

```text
retryable
```

或者：

```text
error_type
```

例如：

```text
error_type = RETRYABLE
error_type = NON_RETRYABLE
```

---

### 5.6 TIMEOUT

`TIMEOUT` 通常允许重试。

但需要限制最大重试次数：

```text
retry_count < max_retry_count
```

如果超过最大次数，则进入：

```text
FINAL_FAILED
```

等待人工处理。

---

### 5.7 重试规则总结

| 当前状态         | 是否允许重试 | 原因              |
| ------------ | -----: | --------------- |
| CREATED      |      否 | 应正常提交，不属于重试     |
| QUEUED       |      否 | 已经进入消息队列        |
| RUNNING      |      否 | 当前正在执行          |
| SUCCESS      |      否 | 已经成功，重新执行应创建新任务 |
| FAILED       |    视情况 | 只有可恢复异常允许重试     |
| TIMEOUT      |      是 | Worker 崩溃或任务超时  |
| FINAL_FAILED |   人工决定 | 已超过自动重试次数       |
| CANCELLED    |  视业务决定 | 通常重新创建任务        |

---

## 六、两个 Worker 并发修改状态

### 6.1 我最初的方案

我最初想到使用 Redis 分布式锁：

```text
lock:task:{taskId}
```

两个 Worker 修改同一个任务前先抢锁，抢到锁的 Worker 才能继续执行。

这个方案可以限制并发，但对于单纯的数据库状态更新来说，Redis 锁不是最简单的方案。

---

### 6.2 更合适的方案：数据库条件更新

两个 Worker 同时尝试将任务从 `RUNNING` 修改为 `SUCCESS` 时，可以执行：

```sql
UPDATE ai_task
SET status = 'SUCCESS',
    finished_at = NOW()
WHERE id = ?
  AND status = 'RUNNING';
```

然后检查受影响行数：

```java
int affectedRows = taskMapper.markSuccess(taskId);

if (affectedRows == 0) {
    throw new IllegalStateException("任务状态已发生变化");
}
```

执行过程：

```text
Worker A：
RUNNING → SUCCESS
影响1行
```

```text
Worker B：
尝试执行相同SQL
但状态已经不是RUNNING
影响0行
```

这样只有一个 Worker 可以成功更新。

这种方式本质上类似 CAS：

```text
只有当前值仍然等于预期值时
才允许修改为新值
```

---

### 6.3 乐观锁方案

也可以增加：

```text
version
```

更新时同时校验状态和版本：

```sql
UPDATE ai_task
SET status = 'SUCCESS',
    version = version + 1
WHERE id = ?
  AND status = 'RUNNING'
  AND version = ?;
```

只有版本号没有发生变化的 Worker 可以更新成功。

---

### 6.4 为什么不优先使用 Redis 锁

Redis 分布式锁还需要考虑：

```text
锁的过期时间
Worker崩溃后的锁释放
任务执行时间超过锁时间
锁自动续期
Redis不可用
解锁时误删其他Worker的锁
```

因此，对于单纯状态修改冲突：

> 优先使用数据库条件更新或乐观锁。

但需要注意：

> 条件更新只能避免两个 Worker 重复修改状态，不能自动避免两个 Worker 都调用一次 GPT。

因此，在 Worker 领取任务时，也应该抢占执行权：

```sql
UPDATE ai_task
SET status = 'RUNNING',
    worker_id = ?
WHERE id = ?
  AND status = 'QUEUED';
```

只有影响行数为 `1` 的 Worker，才获得执行资格。

完整防线是：

```text
领取任务：条件更新抢占执行权
执行任务：幂等控制
完成任务：条件更新状态
复杂临界区：必要时使用分布式锁
```

---

## 七、状态变更日志设计

如果需要查看一个任务的完整生命周期，就不能只保存任务表中的当前状态。

任务表只能告诉我们：

```text
任务现在是什么状态
```

但无法告诉我们：

```text
任务之前经历过什么
什么时候失败
为什么失败
由哪个Worker执行
是谁触发重试
一共重试了几次
```

因此需要单独设计状态变更日志表。

---

### 7.1 日志表字段

```sql
CREATE TABLE task_status_log (
    id BIGINT PRIMARY KEY,
    task_id BIGINT NOT NULL,
    from_status VARCHAR(32),
    to_status VARCHAR(32) NOT NULL,
    trigger_type VARCHAR(32),
    operator_id VARCHAR(64),
    worker_id VARCHAR(64),
    reason VARCHAR(500),
    error_code VARCHAR(64),
    trace_id VARCHAR(64),
    retry_count INT,
    created_at DATETIME NOT NULL
);
```

字段含义：

| 字段             | 作用           |
| -------------- | ------------ |
| `task_id`      | 对应的任务 ID     |
| `from_status`  | 修改前状态        |
| `to_status`    | 修改后状态        |
| `trigger_type` | 状态变化由什么触发    |
| `operator_id`  | 操作人员         |
| `worker_id`    | 执行任务的 Worker |
| `reason`       | 状态变化原因       |
| `error_code`   | 失败错误码        |
| `trace_id`     | 查询完整日志链路     |
| `retry_count`  | 当前重试次数       |
| `created_at`   | 状态变化时间       |

---

### 7.2 触发类型

`trigger_type` 可以包含：

```text
USER
ADMIN
SYSTEM
WORKER
SCHEDULER
RETRY
TIMEOUT_CHECK
```

这样可以知道一次状态变化是：

* 用户提交任务；
* Worker 执行完成；
* 定时任务检测超时；
* 系统自动重试；
* 运营人员手动处理。

---

### 7.3 完整生命周期示例

```text
10:00:00  NULL      → CREATED   USER
10:00:01  CREATED   → QUEUED    SYSTEM
10:00:03  QUEUED    → RUNNING   WORKER-01
10:05:30  RUNNING   → TIMEOUT   TIMEOUT_CHECK
10:05:31  TIMEOUT   → RETRYING  SCHEDULER
10:05:32  RETRYING  → QUEUED    SYSTEM
10:05:35  QUEUED    → RUNNING   WORKER-02
10:07:20  RUNNING   → SUCCESS   WORKER-02
```

通过这些日志，可以完整还原：

1. 任务什么时候创建；
2. 什么时候进入消息队列；
3. 第一次由哪个 Worker 执行；
4. 第一次为什么超时；
5. 谁触发了重试；
6. 第二次由哪个 Worker 执行；
7. 最终什么时候成功。

---

### 7.4 状态和日志的一致性

状态更新和状态日志插入应该放在同一个事务中：

```java
@Transactional
public void changeStatus(
        Long taskId,
        TaskStatus fromStatus,
        TaskStatus toStatus,
        String reason) {

    int affectedRows = taskMapper.updateStatus(
            taskId,
            fromStatus,
            toStatus
    );

    if (affectedRows == 0) {
        throw new IllegalStateException(
                "任务状态已经变化或发生并发冲突"
        );
    }

    TaskStatusLog log = new TaskStatusLog();
    log.setTaskId(taskId);
    log.setFromStatus(fromStatus);
    log.setToStatus(toStatus);
    log.setReason(reason);
    log.setCreatedAt(LocalDateTime.now());

    taskStatusLogMapper.insert(log);
}
```

否则可能出现：

```text
任务状态已经更新
但状态日志插入失败
```

或者：

```text
状态日志插入成功
但任务状态更新失败
```

将两步放入同一个数据库事务，可以保证它们同时成功或者同时回滚。

---

## 八、状态机的最小实现

对于当前的 AI 异步任务执行平台 MVP，不需要立即引入复杂的状态机框架。

可以先通过：

```text
枚举
合法流转规则
统一状态服务
数据库条件更新
状态日志
```

完成最小实现。

### 8.1 状态枚举

```java
public enum TaskStatus {

    CREATED,
    QUEUED,
    RUNNING,
    SUCCESS,
    FAILED,
    TIMEOUT,
    RETRYING,
    FINAL_FAILED
}
```

---

### 8.2 合法流转规则

```java
private static final Map<TaskStatus, Set<TaskStatus>>
        TRANSITIONS = Map.of(
        TaskStatus.CREATED,
        Set.of(TaskStatus.QUEUED),

        TaskStatus.QUEUED,
        Set.of(TaskStatus.RUNNING),

        TaskStatus.RUNNING,
        Set.of(
                TaskStatus.SUCCESS,
                TaskStatus.FAILED,
                TaskStatus.TIMEOUT
        ),

        TaskStatus.FAILED,
        Set.of(TaskStatus.RETRYING),

        TaskStatus.TIMEOUT,
        Set.of(
                TaskStatus.RETRYING,
                TaskStatus.FINAL_FAILED
        ),

        TaskStatus.RETRYING,
        Set.of(TaskStatus.QUEUED)
);
```

---

### 8.3 校验状态流转

```java
public boolean canTransition(
        TaskStatus currentStatus,
        TaskStatus targetStatus) {

    return TRANSITIONS
            .getOrDefault(currentStatus, Set.of())
            .contains(targetStatus);
}
```

---

### 8.4 统一修改状态

```java
@Transactional
public void changeStatus(
        Long taskId,
        TaskStatus currentStatus,
        TaskStatus targetStatus,
        String reason) {

    if (!canTransition(currentStatus, targetStatus)) {
        throw new IllegalStateException(
                "非法状态流转："
                        + currentStatus
                        + " → "
                        + targetStatus
        );
    }

    int affectedRows = taskMapper.updateStatus(
            taskId,
            currentStatus,
            targetStatus
    );

    if (affectedRows == 0) {
        throw new IllegalStateException(
                "任务状态已发生变化"
        );
    }

    taskStatusLogMapper.insert(
            buildStatusLog(
                    taskId,
                    currentStatus,
                    targetStatus,
                    reason
            )
    );
}
```

所有状态更新都应该经过：

```text
TaskStatusService.changeStatus(...)
```

Controller、Worker、定时任务和后台管理功能都不能直接修改数据库状态。

---

## 九、今天纠正的几个错误

### 错误一：超时任务直接设置为 FAILED

修正：

```text
业务明确执行失败 → FAILED
Worker崩溃或任务超时 → TIMEOUT
```

不同异常应该使用不同状态，方便后续判断是否重试和排查原因。

---

### 错误二：所有状态都可以重试

修正：

```text
QUEUED和RUNNING不能重试
SUCCESS需要重新创建任务
FAILED需要判断错误是否可恢复
TIMEOUT可以在限制次数后重试
```

---

### 错误三：并发修改状态优先使用 Redis 锁

修正：

对于简单的数据库状态竞争，优先使用：

```text
数据库条件更新
乐观锁
```

Redis 分布式锁更适合保护复杂的跨资源临界区，而不是所有并发问题都直接上锁。

---

### 错误四：任务表中有 status 就能查看生命周期

修正：

任务表只能保存当前状态。

完整生命周期必须通过追加式状态日志记录：

```text
旧状态
新状态
触发来源
原因
Worker
TraceId
时间
重试次数
```

---

## 十、今天知识点之间的关系

今天的状态机将之前学习的多个知识点串联起来：

```text
状态机
负责限制状态如何流转
```

```text
事务
负责保证状态、结果和日志的一致性
```

```text
幂等
负责防止同一任务被重复执行
```

```text
数据库条件更新
负责防止并发状态冲突
```

```text
MQ
负责异步投递和驱动任务执行
```

```text
超时机制
负责发现长期停留在RUNNING的异常任务
```

```text
重试机制
负责处理暂时性失败
```

```text
状态日志
负责还原完整任务生命周期
```

因此，状态机并不是一个孤立的知识点，而是异步任务系统中连接事务、并发、幂等、MQ、重试和日志的重要业务模型。

---

## 十一、面试表达

在异步任务系统中，我不会把任务状态只当成一个普通字段，而是按照状态机进行设计。

例如任务按照：

```text
CREATED → QUEUED → RUNNING → SUCCESS/FAILED/TIMEOUT
```

进行流转，所有状态修改都统一经过状态服务校验，避免出现 `SUCCESS → RUNNING` 等非法状态。

在并发场景下，我会使用带旧状态条件的 SQL 或乐观锁进行原子更新，只有满足预期状态的 Worker 才能修改成功。

对于长时间停留在 `RUNNING` 的任务，可以通过开始时间、超时时间和 Worker 心跳进行检测，由定时任务将异常任务更新为 `TIMEOUT`，再根据重试次数决定重新入队还是进入最终失败状态。

同时，每次状态变化都会记录旧状态、新状态、触发来源、原因、Worker、TraceId 和修改时间，便于排查任务的完整生命周期。

---

## 十二、今日总结

今天学习后，我对状态机的最终理解是：

> 状态机的概念并不复杂，它的核心只是规定状态之间的合法流转；但在真实工程中，状态变化会受到并发、事务、Worker 崩溃、MQ 重复投递、超时、重试和日志一致性的影响，因此实际设计会比概念复杂得多。

对于当前 AI 异步任务执行平台，第一阶段不需要使用复杂的状态机框架，先完成以下功能即可：

1. 使用枚举定义任务状态；
2. 明确合法状态流转规则；
3. 所有状态更新统一经过状态服务；
4. 使用数据库条件更新处理并发冲突；
5. 增加状态变更日志；
6. 增加任务超时和重试次数；
7. 区分重试与重新执行；
8. 区分可重试失败和不可重试失败。

今天最重要的一句话是：

> **简单系统中，状态机是枚举加流转校验；复杂系统中，状态机是事务、并发、幂等、超时、重试和审计的交汇点。**

## 今日能力等级

**中级后端工程能力。**

当前已经理解了状态机的基本原理以及它与异步任务、并发更新、超时重试和状态日志之间的关系。后续还需要在实际项目中完成状态流转服务、条件更新、超时扫描和状态日志，才能真正转化为工程能力。
