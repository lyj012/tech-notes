# 后端工程化每日训练 Day 11：操作日志

## 一、今天学习的知识点

今天学习的是：

**操作日志（Operation Log / Audit Log）**

操作日志和普通程序日志不是一回事。

程序日志主要解决：

* 系统为什么报错
* 接口为什么失败
* SQL 是否执行异常
* MQ 是否消费成功
* 程序运行过程中发生了什么

而操作日志主要解决：

> 谁在什么时间，对什么业务数据，执行了什么操作。

它的核心价值是：

**业务操作可追溯、可审计。**

---

## 二、操作日志和程序日志的区别

普通程序日志，例如：

```java
logger.info("删除任务成功");
```

这种日志只能说明：

```text
程序执行了删除操作
```

但是可能无法直接知道：

```text
是谁删除的
删除了哪个任务
什么时候删除的
删除前的数据是什么
操作最终是否成功
```

而操作日志通常会记录到数据库中，例如：

```text
operatorId
operatorName
operationType
targetType
targetId
oldValue
newValue
operateTime
ip
result
```

因此后台系统可以根据这些字段进行查询、筛选、分页和审计。

---

## 三、哪些操作需要记录操作日志

我一开始的判断是：

```text
删除任务              需要记录
修改 GPT 模型         需要记录
修改任务状态           需要记录
修改超时时间           需要记录
重新执行任务           可以不记录
```

后面发现：

**重新执行任务同样应该记录。**

因为重新执行任务并不是一个没有影响的操作。

例如：

```text
FAILED
↓
管理员点击 RETRY
↓
QUEUED
↓
RUNNING
↓
SUCCESS / FAILED
```

这个操作可能会：

* 再次发送 MQ 消息
* 再次调用 GPT 接口
* 产生新的 API 调用成本
* 再次修改数据库
* 改变任务状态
* 覆盖或者产生新的任务结果
* 出现重复执行问题

因此，只要人工操作可能影响：

```text
业务数据
业务状态
系统配置
权限
资金
业务流程
```

就应该考虑记录操作日志。

对于 AI 异步任务执行平台来说，以下操作基本都应该记录：

```text
删除任务
重新执行任务
人工修改任务状态
修改 GPT 模型
修改超时时间
```

---

## 四、状态修改应该记录哪些内容

例如管理员人工把任务：

```text
FAILED
↓
SUCCESS
```

至少应该记录：

```text
operatorId
operatorName

operationType = UPDATE_STATUS

targetType = TASK
targetId = 10086

oldValue = FAILED
newValue = SUCCESS

operateTime

ip

result
```

这里非常重要的是：

```text
旧值 oldValue
新值 newValue
```

因为操作日志不仅需要知道：

```text
谁操作了
```

还需要知道：

```text
把什么从什么改成了什么
```

例如：

```text
FAILED → SUCCESS
```

这种人工修改属于比较特殊的操作，因为它可能绕过正常的任务状态机。

正常情况下应该是：

```text
RUNNING
↓
程序执行完成
↓
SUCCESS
```

如果管理员直接：

```text
FAILED
↓
SUCCESS
```

实际上属于：

**人工干预状态机。**

因此可以额外记录：

```text
operationType = MANUAL_STATUS_CHANGE
```

方便后续区分：

```text
系统自动状态流转
```

和：

```text
人工修改状态
```

---

## 五、操作原因是不是必须记录

一开始认为操作日志应该记录：

```text
操作原因 reason
```

但后来进一步分析发现：

**不是所有操作都需要让用户填写操作原因。**

例如普通操作：

```text
修改 GPT 模型
修改超时时间
普通配置修改
```

用户正常修改之后，系统自动记录：

```text
谁修改的
什么时候修改
旧值
新值
```

一般就足够了。

如果所有操作都要求填写原因，会增加用户操作成本，实际业务中也没有必要。

---

### 高风险操作

对于一些高风险人工操作，可以要求填写原因。

例如：

```text
人工把 FAILED 改成 SUCCESS
手动补发会员
人工退款
修改核心权限
强制执行特殊补偿
```

前端可以弹出确认框：

```text
确定将任务 FAILED 修改为 SUCCESS？

修改原因：
[________________]

取消    确认
```

此时可以强制要求管理员填写原因。

因此更合理的设计是：

```text
普通操作
↓
系统自动记录操作日志
↓
用户无感
```

而：

```text
高风险人工操作
↓
二次确认
↓
必要时要求填写原因
↓
记录操作日志
```

在 AI 异步任务平台中，可以设计为：

```text
修改 GPT 模型
→ 不强制填写原因

修改超时时间
→ 不强制填写原因

普通 RETRY
→ 不强制填写原因

删除普通任务
→ 二次确认即可

FAILED 强制改 SUCCESS
→ 建议填写或强制填写原因
```

所以：

> 操作原因不是所有操作日志的必填字段，而是高风险人工操作的附加审计信息。

---

## 六、为什么操作日志一般保存到数据库

操作日志通常保存到数据库，而不是只写普通日志文件。

主要原因不是数据库天然更加安全，而是数据库更加适合：

```text
结构化存储
条件查询
分页
筛选
排序
统计
后台展示
Excel 导出
长期审计
```

例如查询某个任务：

```sql
SELECT *
FROM audit_log
WHERE target_id = 10086;
```

可以很方便地查看：

```text
这个任务被谁修改过
修改过多少次
什么时候重新执行过
有没有人工修改过状态
```

而普通程序日志更加适合：

```text
异常堆栈
接口错误
SQL 错误
MQ 消费情况
系统运行状态
```

因此正确理解应该是：

```text
程序日志
→ 系统运行排查

操作日志
→ 业务审计和追溯
```

---

## 七、如何查询某个任务过去六个月的人工操作

假设未来需要支持：

> 查看 TaskId = 10086 过去六个月所有人工操作记录。

操作日志表可以设计：

```text
audit_log
----------------

id

operator_id
operator_name

target_type
target_id

operation_type

old_value
new_value

operation_source

reason

result

ip

operate_time
```

例如：

```text
target_type = TASK

target_id = 10086

operation_source = MANUAL
```

查询 SQL：

```sql
SELECT *
FROM audit_log
WHERE target_type = 'TASK'
  AND target_id = 10086
  AND operation_source = 'MANUAL'
  AND operate_time >= ?
ORDER BY operate_time DESC;
```

为了提高查询速度，可以建立联合索引：

```sql
INDEX idx_target_time (
    target_type,
    target_id,
    operate_time
);
```

如果经常查询人工操作，也可以考虑：

```sql
INDEX idx_target_source_time (
    target_type,
    target_id,
    operation_source,
    operate_time
);
```

这里需要理解：

```text
字段
↓
决定查询条件
```

而：

```text
索引
↓
帮助数据库更快找到数据
```

不是简单增加一个：

```text
is_manual
```

字段，查询就一定会变快。

真正需要结合：

```text
业务字段设计
+
索引设计
```

---

## 八、实际项目中的实现方式

如果项目使用若依等后台框架，可以通过：

```java
@Log(
    title = "AI任务",
    businessType = BusinessType.UPDATE
)
```

配合 AOP 自动记录操作日志。

整体流程：

```text
管理员操作接口
↓
Controller
↓
业务执行
↓
AOP 拦截
↓
获取当前登录用户
↓
获取操作类型
↓
获取请求参数
↓
获取执行结果
↓
保存 audit_log
```

这样可以避免每个业务接口都手动写：

```java
auditLogMapper.insert(...)
```

减少重复代码。

---

## 九、操作日志需要注意的问题

### 1. 不要把程序日志当操作日志

```java
logger.info("修改成功");
```

不能代替数据库中的业务审计记录。

---

### 2. 重要修改需要记录旧值和新值

例如：

```text
timeout

60
↓
120
```

应该记录：

```text
oldValue = 60
newValue = 120
```

否则以后只能知道有人修改过，却不知道具体修改内容。

---

### 3. 必须能确定操作人

至少应该记录：

```text
operatorId
operatorName
```

否则日志无法完成责任追溯。

---

### 4. 敏感信息需要脱敏

不能直接记录：

```text
密码
Token
AccessKey
身份证号
```

需要过滤或者脱敏。

---

### 5. 操作失败也应该正确记录结果

不能出现：

```text
数据库修改失败
```

但操作日志却记录：

```text
SUCCESS
```

应该根据实际结果记录：

```text
SUCCESS
```

或者：

```text
FAIL
```

必要时记录异常信息。

---

## 十、和之前知识点的联系

操作日志可以和之前学习的：

```text
状态机
幂等
MQ
重试
```

串联起来。

例如管理员重新执行一个失败任务：

```text
管理员点击 RETRY
↓
记录操作日志
↓
FAILED → QUEUED
↓
发送 MQ
↓
消费者获取任务
↓
幂等检查
↓
RUNNING
↓
执行 GPT
↓
SUCCESS / FAILED
```

如果以后出现问题，可以通过：

```text
TaskId
```

查询：

```text
任务状态
执行日志
TraceId
操作日志
```

还原整个任务发生过什么。

---

## 十一、今天最大的认知修正

今天有两个比较重要的认知修正。

### 第一

原本认为：

```text
重新执行任务影响不大
不需要记录操作日志
```

实际上：

```text
RETRY
```

会重新触发完整业务流程，因此属于重要人工操作，应该留下审计记录。

---

### 第二

原本认为操作日志应该统一记录：

```text
操作原因
```

但实际工程中：

**不应该让所有普通操作都填写原因。**

更合理的是：

```text
普通操作
→ 自动记录

高风险操作
→ 必要时要求填写原因
```

否则会增加无意义的用户操作成本。

---

## 十二、最终理解

我目前对操作日志的理解是：

> 操作日志是一套用于业务审计和操作追溯的机制。只要人工操作可能修改重要业务数据、业务状态、系统配置、权限、资金或者触发新的业务流程，就应该考虑记录操作日志。日志通常保存到数据库，通过操作人、目标对象、操作类型、旧值、新值、操作时间和执行结果等字段实现查询和追溯。对于普通操作应该自动记录、尽量做到用户无感，而对于高风险人工干预，可以增加二次确认和操作原因，提高系统的可审计性。
