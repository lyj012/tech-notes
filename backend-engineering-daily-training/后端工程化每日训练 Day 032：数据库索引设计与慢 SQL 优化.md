# 后端工程化每日训练 Day 32：数据库索引设计与慢 SQL 优化

## 一、今天学习的知识点

今天学习的是：

**数据库索引（Index）设计、联合索引、索引选择性、`SELECT *` 的真实影响，以及通过 `EXPLAIN` / `EXPLAIN ANALYZE` 排查慢 SQL。**

一句话理解：

> 索引的核心不是“给查询字段都加索引”，而是根据真实查询路径，让数据库尽可能少扫描数据、少排序、少回表，并通过执行计划验证优化是否真正生效。

今天重点解决以下问题：

```text
为什么数据量越大，SQL 可能越来越慢
SELECT * 是否等于全表扫描
为什么 WHERE + ORDER BY 常常需要联合索引
为什么两个单列索引不等于一个联合索引
status 这种低选择性字段为什么通常不适合单独建索引
LIMIT 20 为什么不代表数据库一定只处理 20 行
SELECT * 应该怎么优化
EXPLAIN 应该重点看什么
走了索引为什么 SQL 仍然可能很慢
rows 很大说明什么
如何结合数据分布、回表、排序、分页方式继续排查
```

---

## 二、索引到底解决什么问题

假设任务表已经有：

```text
5000 万条数据
```

执行：

```sql
SELECT *
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

如果没有适合这条查询的索引，数据库可能需要经历：

```text
扫描大量数据
↓
判断 status 是否等于 RUNNING
↓
找到符合条件的数据
↓
按照 create_time 排序
↓
最后返回前 20 条
```

数据量从：

```text
100 万
↓
1000 万
↓
5000 万
```

扫描、比较、排序和 IO 成本都可能不断增加。

所以索引最直接的价值是：

> **减少数据库为了得到最终结果必须检查的数据量。**

可以简单理解成：

```text
没有合适索引
↓
从大量数据里找答案

有合适索引
↓
从已经组织好的索引结构里快速定位答案
```

索引通常用额外的磁盘空间和写入维护成本，换取查询效率。

因此：

> **索引本质上是空间换时间，但不是索引越多越好。**

---

## 三、今天第一个需要纠正的认知：SELECT * 不等于全表扫描

今天一开始的判断是：

```text
SELECT *
↓
全表扫描
```

这个判断不准确。

`SELECT *` 表示：

```text
查询结果需要返回这一行的全部列
```

而数据库是否进行全表扫描，主要取决于：

```text
WHERE 条件
索引设计
优化器选择
数据分布
SQL 写法
```

例如：

```sql
SELECT *
FROM task
WHERE task_id = 10086;
```

如果 `task_id` 是主键：

```text
PRIMARY KEY(task_id)
```

那么即使使用：

```sql
SELECT *
```

数据库仍然可以通过主键索引快速定位数据。

所以：

> **`SELECT *` 会增加读取和传输成本，但它本身不是“全表扫描”的定义。**

`SELECT *` 常见问题包括：

```text
读取不需要的字段
网络传输数据更多
Java 对象映射数据更多
大字段可能造成更多 IO
更难形成覆盖索引
可能增加回表成本
```

尤其表中存在：

```text
TEXT
LONGTEXT
JSON
BLOB
大 VARCHAR
```

时，无脑 `SELECT *` 的代价会更加明显。

---

## 四、LIMIT 20 不等于数据库只处理 20 条数据

这也是今天一个重要认知。

SQL：

```sql
SELECT *
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

最终确实只返回：

```text
20 条
```

但是如果没有合适索引，数据库可能先处理大量数据：

```text
扫描 5000 万条中的大量记录
↓
筛出 RUNNING
↓
对符合条件的数据排序
↓
最后才取 20 条
```

所以：

```text
返回行数 = 20
```

不等于：

```text
扫描行数 = 20
```

真正优化目标应该是：

> **让数据库能够尽可能早地定位到最终需要的 20 条，并停止继续扫描。**

这正是 `(status, create_time)` 这种联合索引在该场景中的价值。

---

## 五、为什么这里应该优先考虑联合索引

当前查询：

```sql
SELECT *
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

查询需求实际上包含两件事：

```text
1. 找到 status = RUNNING
2. 按 create_time 从新到旧取前 20 条
```

一个常见索引设计是：

```sql
CREATE INDEX idx_status_create_time
ON task(status, create_time);
```

可以把它理解成索引数据先按照：

```text
status
```

组织，同一个 `status` 内部再按照：

```text
create_time
```

组织。

示意：

```text
RUNNING
    2026-08-09 19:00
    2026-08-09 18:59
    2026-08-09 18:58
    ...

SUCCESS
    2026-08-09 19:00
    2026-08-09 18:57
    ...
```

查询时数据库可以：

```text
定位 RUNNING
↓
沿 create_time 对应的索引顺序扫描
↓
取最新 20 条
↓
尽早结束
```

对于 MySQL 常见 B+Tree 索引，即使索引定义没有显式写 `DESC`，优化器在适合的场景下也可以反向扫描索引来满足 `ORDER BY create_time DESC`。

因此核心不是背：

```text
(status, create_time)
```

而是理解：

> **索引字段顺序应该服务于整条查询路径，而不是看到哪个字段出现在 WHERE 里就给哪个字段单独加索引。**

---

## 六、为什么不是 status 和 create_time 各建一个索引

今天一开始对联合索引不熟悉，直觉方案是：

```sql
INDEX(status)
INDEX(create_time)
```

这两个索引分别只能很好地服务一部分需求：

```text
INDEX(status)
→ 帮助筛选 status

INDEX(create_time)
→ 帮助按照 create_time 定位 / 排序
```

但当前 SQL 需要的是：

```text
先满足 status = RUNNING
并且在 RUNNING 这一批数据里
继续利用 create_time 的索引顺序
直接拿前 20 条
```

联合索引：

```text
(status, create_time)
```

可以在同一棵索引结构里同时服务：

```text
WHERE status = ?
+
ORDER BY create_time
+
LIMIT 20
```

而两个独立单列索引，并不等价于一个针对查询模式设计的联合索引。

MySQL 在部分场景可能使用 Index Merge 等策略组合多个索引，但不能因此认为：

```text
两个单列索引
=
一个联合索引
```

实际设计仍然应该优先根据高频 SQL 的完整查询模式判断。

今天可以先记一个常见经验：

> **等值查询字段在前，排序字段跟在后面，是联合索引设计中非常常见的一种模式。**

例如：

```sql
WHERE user_id = ?
ORDER BY create_time DESC
```

首先可以考虑：

```text
(user_id, create_time)
```

再例如：

```sql
WHERE department_id = ?
ORDER BY id DESC
```

首先可以考虑：

```text
(department_id, id)
```

但这只是候选方案，不是机械公式。

最终必须结合：

```text
真实数据量
字段选择性
查询频率
写入频率
EXPLAIN
EXPLAIN ANALYZE
压测结果
```

验证。

---

## 七、联合索引与最左前缀原则

假设建立：

```text
(user_id, status, create_time)
```

典型情况下，可以很好地支持从最左侧开始的查询：

```text
user_id

user_id + status

user_id + status + create_time
```

而如果只查询：

```text
status
```

或者：

```text
create_time
```

通常不能直接按照“从联合索引中跳过最左字段”的方式获得同等效果。

这就是常说的：

> **最左前缀原则。**

但也不能把它背成绝对口号。

数据库优化器版本、索引跳跃扫描等能力可能影响实际执行方式。

工程上真正应该做的是：

```text
理解最左前缀
+
使用 EXPLAIN 验证
```

而不是只靠背规则判断。

---

## 八、为什么 status 通常不适合单独建立索引

`status` 可能只有：

```text
RUNNING
SUCCESS
FAILED
WAITING
```

4 种取值。

假设 5000 万条任务的数据分布为：

```text
SUCCESS   3000 万
RUNNING   1000 万
FAILED     500 万
WAITING    500 万
```

这时候查询：

```sql
WHERE status = 'RUNNING'
```

即使建立：

```text
INDEX(status)
```

也可能一次命中：

```text
1000 万条
```

说明这个字段本身不能把结果集缩小到很小。

这种字段通常称为：

```text
低基数 / 低选择性字段
```

所以今天自己的理解：

> 状态只有 4 种，单独筛选的优化能力有限；联合其他业务查询条件通常更符合实际查询路径。

这个方向是正确的。

需要进一步补充的是：

> **低选择性字段并不是绝对不能出现在索引里，而是单独作为索引时价值经常有限。**

例如当前查询：

```sql
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20
```

`(status, create_time)` 仍然可能很有价值。

因为它不只是为了：

```text
筛选 status
```

还为了：

```text
利用 create_time 顺序
+
配合 LIMIT 尽早结束扫描
```

这就是为什么：

```text
INDEX(status)
```

和：

```text
INDEX(status, create_time)
```

不能只从 `status` 的选择性一个维度判断。

---

## 九、SELECT * 应该怎么优化

后台页面实际上只展示：

```text
任务 ID
任务名称
状态
创建时间
```

那么 SQL 可以改成：

```sql
SELECT task_id,
       task_name,
       status,
       create_time
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

相比：

```sql
SELECT *
```

收益主要是：

```text
减少读取字段数量
减少网络传输
减少 Java 对象映射数据量
避免无意义读取大字段
增加形成覆盖索引的可能性
```

但今天需要明确优先级：

> **`SELECT *` 通常不是一条 6 秒慢 SQL 的第一根因。**

对于当前案例，更应该先排查：

```text
是否有正确索引
扫描了多少数据
是否发生额外排序
是否大量回表
数据分布是否异常
```

然后再继续减少返回字段。

也就是说：

```text
先减少扫描与排序成本
↓
再减少每一行的读取成本
```

---

## 十、什么是回表

InnoDB 中常见的二级索引并不一定保存查询需要的全部列。

例如索引：

```text
(status, create_time)
```

SQL 还需要：

```text
task_name
```

数据库先通过二级索引找到符合条件的记录后，可能还需要根据主键再去聚簇索引读取完整行数据。

可以简单理解成：

```text
查二级索引
↓
拿到主键
↓
根据主键再查一次完整数据
```

这就是常说的：

> **回表。**

如果最终只返回 20 条：

```text
回表 20 次
```

通常成本可能并不大。

真正危险的是某些 SQL：

```text
先通过索引命中几十万 / 几百万条
↓
产生大量随机回表
```

这时候即使：

```text
EXPLAIN 显示 key 不为空
```

SQL 仍然可能很慢。

所以：

> **走索引，不等于没有 IO 成本。**

---

## 十一、什么是覆盖索引

如果一个查询需要的字段都能够直接从某个索引中获得，就可以避免为了读取其他列再次回到聚簇索引查完整行。

这类情况通常称为：

> **覆盖索引（Covering Index）。**

例如某些极简查询：

```sql
SELECT status, create_time
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

如果索引是：

```text
(status, create_time)
```

就更容易直接从索引中完成查询。

但不能为了覆盖所有查询字段，无限制把大量字段塞进联合索引。

因为索引越宽：

```text
占用磁盘越多
缓存效率下降
INSERT 维护成本增加
UPDATE 索引字段成本增加
页分裂与写放大成本增加
```

所以覆盖索引仍然需要考虑：

```text
查询收益
vs
写入和存储成本
```

---

## 十二、慢 SQL 第一原则：先 EXPLAIN，不要先猜

今天第四题的回答是：

```text
先分析
EXPLAIN
```

这是正确的排查方向。

例如：

```sql
EXPLAIN
SELECT task_id,
       task_name,
       status,
       create_time
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

重点先关注：

```text
type
possible_keys
key
rows
Extra
```

### 1. type

如果看到：

```text
ALL
```

通常意味着全表扫描，需要高度关注。

常见访问类型不应该只背排名，而应该理解：

```text
数据库为了找到结果
到底要扫描多大范围
```

---

### 2. possible_keys

表示：

```text
优化器认为可能可以使用哪些索引
```

它不代表最终一定会使用。

---

### 3. key

表示：

```text
优化器最终选择的索引
```

如果：

```text
key = NULL
```

说明当前执行计划没有选择索引。

但需要注意：

> **key 不为 NULL，也不等于 SQL 一定快。**

---

### 4. rows

`EXPLAIN` 中的 `rows` 主要是优化器对需要检查行数的估算。

如果看到：

```text
rows = 10000000
```

至少说明需要高度警惕：

```text
索引过滤能力可能不足
数据分布可能导致命中范围过大
统计信息可能不准确
```

但不能简单把：

```text
EXPLAIN rows
```

直接当成：

```text
实际扫描行数
```

它是估算值。

---

### 5. Extra

需要重点关注例如：

```text
Using filesort
Using temporary
Using index
Using index condition
```

其中：

```text
Using filesort
```

说明 MySQL 需要额外进行排序处理，并不代表一定使用磁盘文件排序，但对于大结果集需要重点分析。

```text
Using temporary
```

说明执行过程中使用了临时表，复杂 GROUP BY / ORDER BY 等场景尤其需要关注。

---

## 十三、EXPLAIN ANALYZE 为什么更进一步

如果数据库版本支持，可以进一步使用：

```sql
EXPLAIN ANALYZE
SELECT task_id,
       task_name,
       status,
       create_time
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

与普通 `EXPLAIN` 相比，它能够提供实际执行过程中的更多信息，例如：

```text
实际执行时间
实际循环次数
实际处理行数
各执行节点成本
```

因此排查真实线上慢 SQL 时，可以形成一个基本思路：

```text
先 EXPLAIN 看执行计划
↓
需要进一步验证时看 EXPLAIN ANALYZE
↓
结合慢 SQL 日志和监控判断真实瓶颈
```

注意：

> `EXPLAIN ANALYZE` 会真正执行 SQL，在线上使用前需要评估查询本身的成本和影响，不能对高风险写操作或极重查询随意执行。

---

## 十四、走了索引为什么仍然可能很慢

今天第五题的场景：

```text
key = idx_status_create_time
rows = 10000000
```

第一反应是：

```text
扫描的数据量还是太大
需要继续优化 SQL
```

方向正确。

需要继续补全的是：

> **“走了索引”只说明优化器使用了某个索引，不代表这个索引已经把查询范围缩得足够小。**

常见原因包括：

```text
字段选择性太低
查询条件本身会命中海量数据
联合索引字段顺序不匹配
索引只能过滤一部分条件
发生大量回表
排序没有完全利用索引
统计信息不准确
数据冷热分布不均
分页 OFFSET 太深
磁盘 IO 压力高
Buffer Pool 命中率低
存在锁等待
```

因此：

```text
走索引 ≠ SQL 一定快
```

真正应该看：

```text
这个索引减少了多少扫描
这个 SQL 还做了多少额外工作
```

---

## 十五、数据分布为什么会影响索引效果

假设：

```text
5000 万条任务
```

其中：

```text
RUNNING = 1000 万
```

那么：

```sql
WHERE status = 'RUNNING'
```

本身就是一个很宽的查询范围。

如果真实业务实际上是：

```text
查询某个租户当前运行中的任务
```

SQL 应该可能更接近：

```sql
SELECT task_id,
       task_name,
       status,
       create_time
FROM task
WHERE tenant_id = ?
  AND status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

这时候可以考虑：

```text
(tenant_id, status, create_time)
```

假设：

```text
5000 万条
↓ tenant_id
10 万条
↓ status
500 条
↓ create_time
取最新 20 条
```

这种索引的选择性可能明显高于：

```text
(status, create_time)
```

所以索引设计必须回到业务语义：

> **用户究竟想查哪一小部分数据？**

而不是脱离业务单独研究字段。

---

## 十六、常见索引失效或难以充分利用的 SQL 写法

### 1. 对普通索引列做函数计算

例如：

```sql
WHERE YEAR(create_time) = 2026
```

对于普通的 `create_time` B+Tree 索引，这种写法通常不利于直接按原始列范围定位。

更常见的写法：

```sql
WHERE create_time >= '2026-01-01'
  AND create_time <  '2027-01-01'
```

把条件改成明确范围。

---

### 2. 前置模糊匹配

例如：

```sql
WHERE task_name LIKE '%report%'
```

普通 B+Tree 索引通常无法像：

```sql
LIKE 'report%'
```

那样很好地利用有序前缀定位。

如果业务确实需要大量全文检索，则应该考虑：

```text
全文索引
Elasticsearch
其他搜索方案
```

而不是强行依赖普通索引。

---

### 3. 隐式类型转换

例如数据库字段是字符串，但 SQL 参数类型和字段类型处理不一致。

这类情况可能影响索引使用方式。

因此接口参数、Java 类型和数据库字段类型应该保持合理一致。

---

## 十七、索引不是越多越好

每增加一个索引，写入时数据库都需要维护它。

例如：

```sql
INSERT INTO task ...
```

除了写入表数据，还可能要维护：

```text
主键索引
status 索引
create_time 索引
status + create_time 联合索引
tenant_id + status + create_time 联合索引
其他业务索引
```

索引过多会带来：

```text
INSERT 更慢
UPDATE 更慢
DELETE 成本增加
磁盘占用增加
Buffer Pool 被更多索引页占用
DDL 管理更复杂
重复 / 冗余索引增加
```

所以不应该：

```text
这个字段有人查
↓
建索引

那个字段也有人查
↓
再建索引
```

而应该：

```text
统计高频查询
↓
归纳查询模式
↓
设计少量高价值索引
↓
EXPLAIN 验证
↓
压测
↓
上线监控
```

---

## 十八、分页也可能让 SQL 越翻越慢

虽然今天案例使用：

```sql
LIMIT 20
```

但后台管理经常出现：

```sql
LIMIT 1000000, 20
```

这类深分页可能导致数据库为了得到最终 20 条，仍然需要跳过前面大量记录。

常见优化方向是基于上一页最后一个有序键继续查询。

例如：

```sql
SELECT task_id,
       task_name,
       status,
       create_time
FROM task
WHERE status = 'RUNNING'
  AND create_time < ?
ORDER BY create_time DESC
LIMIT 20;
```

如果时间可能重复，还可以结合唯一 ID 形成稳定游标。

这种方式通常称为：

```text
Keyset Pagination
游标分页
```

核心思想是：

> **不要让数据库每翻一页都重新跳过前面几十万、几百万条。**

---

## 十九、5000 万任务表慢 SQL 的完整排查顺序

线上出现：

```text
SQL：6 秒
数据库 CPU：95%
```

不应该一上来执行：

```text
加 CPU
加机器
把数据库规格翻倍
```

也不应该凭感觉连续加索引。

可以按下面顺序排查。

### 第一步：确认慢 SQL

看：

```text
慢 SQL 日志
APM
接口日志
数据库监控
```

确认：

```text
真正慢的是哪条 SQL
平均耗时
P95 / P99
执行频率
每秒调用次数
```

---

### 第二步：EXPLAIN

看：

```text
type
key
rows
Extra
```

先确认：

```text
有没有走预期索引
有没有大范围扫描
有没有额外排序
有没有临时表
```

---

### 第三步：检查现有索引

重点判断：

```text
是否缺索引
是否存在重复索引
联合索引字段顺序是否匹配查询
索引是否服务 WHERE + ORDER BY + LIMIT
```

---

### 第四步：检查数据分布

例如：

```sql
SELECT status, COUNT(*)
FROM task
GROUP BY status;
```

判断：

```text
RUNNING 到底有多少
各状态占比如何
是否存在极端倾斜
```

索引效果必须结合真实数据分布判断。

---

### 第五步：检查返回字段与回表

判断：

```text
是否真的需要 SELECT *
是否读取了大字段
是否产生大量回表
是否适合覆盖索引
```

---

### 第六步：检查分页和排序方式

确认是否存在：

```text
深分页
大范围 ORDER BY
filesort
临时表
```

必要时考虑：

```text
游标分页
调整联合索引
改变查询入口
```

---

### 第七步：检查数据库资源

如果 SQL 设计已经合理，再继续看：

```text
CPU
磁盘 IO
IOPS
Buffer Pool
锁等待
长事务
连接池
数据库并发
```

避免把：

```text
SQL 设计问题
```

误判成：

```text
数据库机器性能不足
```

---

## 二十、今天练习题复盘

### 问题一：为什么 SQL 会越来越慢

最初回答：

```text
因为 SELECT * 算全表扫描
数据越来越大当然越来越慢
没有加索引
```

需要修正为：

> `SELECT *` 不等于全表扫描。真正的问题是没有适合 `WHERE status + ORDER BY create_time + LIMIT` 的索引时，数据库可能需要扫描大量数据并进行额外排序；`SELECT *` 会进一步增加读取、传输和回表成本。

---

### 问题二：应该建立什么索引

最初思路：

```text
status 加一个索引
create_time 加一个索引
联合索引不会
```

今天掌握后的答案：

```text
(status, create_time)
```

原因：

```text
status 用于等值筛选
+
create_time 用于排序
+
LIMIT 20 尽早停止扫描
```

---

### 问题三：为什么两个单列索引不如这里的联合索引

自己的理解：

```text
联合索引查询更快
两个单独索引还要分别筛选
```

更准确的表达：

> 两个单列索引只能分别服务查询的一部分需求，而 `(status, create_time)` 能在同一索引路径里同时利用 `status` 条件和 `create_time` 顺序，更适合当前 `WHERE + ORDER BY + LIMIT` 的完整查询模式。

---

### 问题四：SELECT * 怎么优化

自己的修改：

```sql
SELECT task_id,
       task_name,
       status,
       create_time
FROM task
WHERE status = 'RUNNING'
ORDER BY create_time DESC
LIMIT 20;
```

方向正确。

核心是：

```text
只查询页面真正需要的列
```

而不是默认把整行数据全部拉出来。

---

### 问题五：加了索引仍然很慢怎么办

自己的回答：

```text
先分析
EXPLAIN
```

这是今天最重要的线上排查习惯之一：

> **慢 SQL 不要先猜，先看执行计划。**

---

### 问题六：走了索引但是 rows 很大说明什么

自己的回答：

```text
扫描的数据量太大
考虑继续优化 SQL
```

需要进一步补全：

```text
索引可能使用了
但过滤能力仍然不够
```

继续检查：

```text
数据分布
字段选择性
联合索引设计
回表
排序
分页
统计信息
```

---

### 问题七：为什么 status 不适合单独建索引

自己的回答：

```text
状态只有 4 种
筛选结果有限
单独优化效果有限
联合起来更符合逻辑筛选
```

可以整理成更专业的表达：

> `status` 的取值种类很少，属于低选择性字段，单独建立索引往往无法把结果集缩小很多。实际项目中会结合其他过滤条件或排序字段设计联合索引，并使用执行计划验证。

---

## 二十一、面试怎么表达

如果面试官问：

```text
你平时怎么设计索引？
```

可以回答：

> 我不会看到某个字段出现在 WHERE 条件里就直接单独建索引，而是先看真实高频 SQL 的查询模式，结合 WHERE、ORDER BY、LIMIT 和字段选择性设计联合索引。比如 `WHERE status = ? ORDER BY create_time DESC LIMIT 20`，会优先考虑 `(status, create_time)`，让数据库在筛选状态的同时利用时间顺序快速取前 N 条。优化后会通过 `EXPLAIN` 或 `EXPLAIN ANALYZE` 验证实际执行计划，重点关注索引选择、扫描范围、排序、临时表和回表成本，同时也会考虑索引对 INSERT、UPDATE 的写入影响，避免索引过多。

如果面试官继续问：

```text
为什么 status 不单独建索引？
```

可以回答：

> `status` 一般只有少量枚举值，选择性比较低，单独索引可能一次仍然命中大量数据，所以过滤效果有限。但如果查询同时需要按照创建时间排序并限制返回条数，那么 `(status, create_time)` 联合索引仍然可能非常有效，因为它不仅用于筛选，还可以利用索引顺序减少额外排序，并配合 LIMIT 提前结束扫描。

---

## 二十二、实际开发中如何正确使用 AI 帮助索引设计

实际开发中完全可以把 SQL、表结构和数据规模交给 AI 辅助分析。

但不能只做：

```text
复制 SQL
↓
问 AI 建什么索引
↓
直接执行 CREATE INDEX
```

更可靠的流程应该是：

```text
把表结构发给 AI
+
把高频 SQL 发给 AI
+
说明数据量和字段分布
+
让 AI 给候选索引和理由
↓
自己判断 WHERE / ORDER BY / LIMIT 是否匹配
↓
EXPLAIN 验证
↓
必要时 EXPLAIN ANALYZE
↓
测试环境压测
↓
评估写入成本
↓
再上线
```

AI 可以帮助：

```text
快速提出候选索引
分析执行计划
发现冗余索引
解释 filesort / temporary
分析分页问题
提供 SQL 改写方案
```

但工程师必须能够判断：

```text
为什么这个索引合理
它解决了哪一段执行成本
它会增加什么写入成本
执行计划是否真的改善
```

否则很容易出现：

```text
AI 给了索引
↓
索引确实创建成功
↓
线上 SQL 仍然慢
↓
甚至写入性能下降
```

---

## 二十三、今天需要真正记住的结论

### 1. SELECT * 不等于全表扫描

```text
是否全表扫描
主要取决于查询条件、索引和优化器执行计划
```

`SELECT *` 的主要问题是：

```text
读取数据更多
传输更多
回表可能更多
覆盖索引更困难
```

---

### 2. LIMIT 20 不代表只处理 20 条

```text
没有合适索引
↓
可能先扫描 / 排序大量数据
↓
最终才返回 20 条
```

---

### 3. 两个单列索引不等于一个联合索引

```text
INDEX(status)
+
INDEX(create_time)
```

不等于：

```text
INDEX(status, create_time)
```

联合索引应该围绕完整查询路径设计。

---

### 4. 低选择性字段单独建索引往往价值有限

```text
status 只有几个枚举值
```

单独索引可能仍然命中大量数据。

但：

```text
(status, create_time)
```

仍然可能因为排序 + LIMIT 获得很高价值。

---

### 5. 走索引不等于 SQL 一定快

真正要继续分析：

```text
扫描范围
数据分布
选择性
回表
排序
临时表
分页
IO
锁等待
```

---

### 6. 慢 SQL 不要先猜，先看执行计划

基本动作：

```sql
EXPLAIN ...
```

需要进一步验证时：

```sql
EXPLAIN ANALYZE ...
```

---

## 二十四、最终形成的慢 SQL 排查思维

今天最重要的不是记住：

```text
(status, create_time)
```

而是形成下面这条思维链：

```text
线上 SQL 慢
↓
确认真实慢 SQL 和调用频率
↓
EXPLAIN 看执行计划
↓
判断是否走预期索引
↓
看扫描范围和数据分布
↓
看 WHERE + ORDER BY + LIMIT 是否能被同一索引服务
↓
看是否有 filesort / temporary
↓
看是否大量回表
↓
看 SELECT * 是否读取无用字段
↓
看是否深分页
↓
再看数据库 CPU / IO / Buffer Pool / 锁
↓
优化并重新验证
```

最终应该从：

```text
SQL 慢
↓
让 AI 给我建个索引
```

逐渐升级为：

```text
SQL 慢
↓
我先理解数据库为什么需要做这么多工作
↓
再让 AI 辅助提出候选方案
↓
用执行计划和数据验证方案
```

这才是数据库性能优化真正有价值的工程能力。
