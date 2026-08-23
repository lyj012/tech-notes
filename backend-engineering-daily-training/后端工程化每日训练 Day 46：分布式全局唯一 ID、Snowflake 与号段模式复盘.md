# 后端工程化每日训练 Day 46：分布式全局唯一 ID、Snowflake 与号段模式复盘

## 一、今天学习什么

今天主要学习：

- 为什么单库 `AUTO_INCREMENT` 到分库分表后不能天然保证全局唯一
- “单库唯一”和“全局唯一”的区别
- UUID 为什么生成方便，但不适合很多大型 MySQL 表做主键
- Snowflake 的 `timestamp + workerId + sequence` 组成
- `workerId` 和 `sequence` 分别解决什么问题
- Snowflake 的时钟回拨风险
- Kubernetes 多 Pod 环境下如何管理 `workerId`
- 12 bit `sequence` 的容量含义
- Snowflake 为什么性能高、为什么只是趋势递增
- 数据库号段模式与批量预分配思想
- 双 Buffer 为什么能够减少号段切换时的等待
- 号段模式为什么允许跳号
- Java `Long` Snowflake ID 传给 JavaScript 后的精度问题
- 高并发场景下 Snowflake 与号段模式如何做工程选择

今天最核心的判断：

> 分布式 ID 的目标不是让编号“看起来漂亮”，而是在多节点、多数据库、高并发环境中，以可控的复杂度稳定生成全局唯一 ID。

---

# 二、单库自增 ID 为什么不能直接解决分布式唯一性

只有一个 MySQL 时：

```sql
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY
);
```

数据库可以正常生成：

```text
10001
10002
10003
```

这种场景完全没有必要为了“架构高级”强行改成 Snowflake。

如果现有方案：

```text
单库
没有分库分表
自增主键稳定
没有全局 ID 冲突需求
```

那么继续使用 `AUTO_INCREMENT` 往往就是更简单、更合理的方案。

但是系统变成 8 个订单库以后：

```text
DB0 → id = 10001
DB1 → id = 10001
```

在各自数据库内部：

```text
都没有主键冲突
```

但这并不代表：

```text
整个订单系统不存在 ID 冲突
```

因为数据库唯一约束只约束当前库。

---

# 三、真正的问题是 ID 进入“全局语境”

一开始容易把问题理解成：

> 只有把 8 个数据库的数据全部导到一个地方时，才会产生冲突。

更准确的理解是：

> 只要 ID 离开自己的数据库上下文，进入 MQ、Redis、Elasticsearch、跨库查询、数据仓库或其他服务，就已经进入全局语境。

例如：

```text
DB0：orderId = 10001
DB1：orderId = 10001
```

进入 Kafka：

```json
{
  "orderId": 10001
}
```

消费者只看到：

```text
10001
```

就无法知道它到底来自：

```text
DB0
还是
DB1
```

Redis 更直接：

```text
order:10001
```

如果两个订单都使用同一个 Key，就可能发生覆盖。

因此需要记住：

> 单库唯一 ≠ 全局唯一。

当然也可以设计组合标识：

```text
(DB0, 10001)
(DB1, 10001)
```

但这时真正的唯一标识已经不是单独的 `orderId`，而是整个组合。

---

# 四、UUID 为什么方便，但大型订单表通常不优先使用随机 UUID 主键

UUID 最大的优点是：

```text
不依赖数据库
不依赖中心服务
各节点自己生成
实现简单
冲突概率极低
```

例如：

```java
String id = UUID.randomUUID().toString();
```

非常适合：

```text
TraceId
文件名
请求 ID
对象存储 Key
临时任务编号
```

但随机 UUID 作为大型 MySQL 表主键时，需要考虑 InnoDB 的 B+Tree 特性。

趋势递增 ID：

```text
10001
10002
10003
10004
```

新数据通常更接近索引右侧追加。

随机 UUID：

```text
8af...
12c...
f91...
3ae...
```

插入位置可能到处跳，可能带来：

```text
随机写
页分裂
索引碎片
缓存局部性变差
索引空间增大
```

另外今天还纠正了一个概念：

```text
BIGINT = 8 字节
```

不是：

```text
“只有 8 位数字”
```

而 UUID 字符串常见形式是 36 个字符。

InnoDB 的二级索引中还需要保存主键值，因此主键越大，很多二级索引也会跟着变大。

所以对于大规模订单表：

> `BIGINT` 类型、趋势递增的主键通常比随机字符串 UUID 更适合数据库索引。

---

# 五、Snowflake 的核心结构

Snowflake 可以把一个 64 位整数理解成：

```text
时间戳 + workerId + sequence
```

一个常见划分：

```text
时间戳      41 bit
workerId   10 bit
sequence   12 bit
```

可以简单理解为：

```text
当前是什么时间
+
我是哪个节点
+
这一毫秒我是第几个请求
```

最终得到一个 `long` 类型 ID。

例如：

```text
198273645817263104
```

---

# 六、为什么必须同时有 workerId 和 sequence

如果只有毫秒时间戳：

```text
timestamp = 100500
```

同一个 Java 实例在这一毫秒内可能收到很多请求。

如果所有请求只使用时间戳：

```text
100500
100500
100500
```

就会直接重复。

因此需要 `sequence`：

```text
100500 + worker1 + 0
100500 + worker1 + 1
100500 + worker1 + 2
```

它解决的是：

> 同一台机器、同一毫秒内多个请求如何区分。

而 `workerId` 解决的是不同节点之间的区分：

```text
Java-1：100500 + worker1 + 0
Java-2：100500 + worker2 + 0
```

即使：

```text
时间戳相同
sequence 相同
```

只要：

```text
workerId 不同
```

最终 ID 仍然不同。

因此可以直接记：

```text
时间戳   → 区分不同时间
workerId → 区分不同节点
sequence → 区分同一节点同一毫秒内的请求
```

---

# 七、workerId 重复为什么危险

假设两个 Java 实例错误配置：

```text
Java-1 workerId = 5
Java-2 workerId = 5
```

从 Snowflake 算法的视角看，这两个实例已经无法通过 workerId 区分。

如果它们恰好出现：

```text
时间戳相同
workerId 相同
sequence 相同
```

例如：

```text
Java-1：100500 + 5 + 0
Java-2：100500 + 5 + 0
```

最终就可能生成完全相同的 ID。

因此：

> workerId 唯一不是建议，而是 Snowflake 正确性的基础条件之一。

---

# 八、Kubernetes 下 workerId 为什么更麻烦

传统服务器比较稳定：

```text
server01 → workerId 1
server02 → workerId 2
server03 → workerId 3
```

但是 Kubernetes Pod 会：

```text
创建
销毁
扩容
漂移
重建
```

如果 100 个 Pod 全部写死：

```text
WORKER_ID=1
```

那么从 Snowflake 的视角看：

```text
100 个 Pod
≈
同一个 worker
```

就存在重复 ID 风险。

常见的 workerId 管理方式包括：

```text
StatefulSet 稳定序号
Redis / DB 注册分配
Etcd / ZooKeeper
独立 workerId 分配服务
```

例如 StatefulSet：

```text
order-service-0 → workerId 0
order-service-1 → workerId 1
order-service-2 → workerId 2
```

相比随机 Pod 名，这种稳定序号更容易映射 workerId。

但无论采用哪种方案，核心要求都是：

```text
自动分配
避免重复
能够检测冲突
```

---

# 九、12 bit sequence 到底意味着什么

如果 `sequence` 使用 12 bit：

```text
2^12 = 4096
```

可以表示：

```text
0 ~ 4095
```

所以：

> 单个 worker 在同一毫秒内最多生成 4096 个不同 ID。

理论上：

```text
4096 × 1000
≈ 409 万 ID / 秒 / worker
```

如果某一毫秒真的用完 4096 个序号：

```text
sequence 用完
↓
等待下一毫秒
↓
sequence = 0
↓
继续生成
```

实际吞吐还会受 CPU、同步实现等因素影响，但容量已经很高。

---

# 十、Snowflake 最经典的风险：时钟回拨

Snowflake 强依赖时间戳。

假设：

```text
lastTimestamp    = 100500
currentTimestamp = 100300
```

说明机器时间突然倒退了 200ms。

如果算法什么都不管，继续生成：

```text
时间重新进入已经使用过的区间
+
workerId 可能相同
+
sequence 可能再次出现相同值
↓
产生重复 ID 风险
```

因此检测到：

```text
currentTimestamp < lastTimestamp
```

绝不能装作什么都没发生。

轻微回拨可以考虑：

```text
等待系统时间追上 lastTimestamp
```

例如：

```text
回拨 2ms
↓
暂停生成
↓
时间追上
↓
继续
```

如果回拨很大，例如 30 秒：

```text
让大量业务线程等待 30 秒
```

就可能让服务大面积阻塞。

更合理的处理可能包括：

```text
拒绝生成
报警
切换备用 workerId
使用逻辑时间
```

具体阈值应该根据业务和实现决定。

今天需要记住的不是“200ms 一定等待”，而是：

> 发现时钟回拨后必须进入专门的异常处理逻辑，不能继续普通生成。

---

# 十一、Snowflake 为什么性能高

Snowflake 生成 ID 通常不需要：

```text
查 MySQL
查 Redis
调用远程 ID 服务
网络 IO
```

每个实例可以直接在本地计算：

```text
时间戳
+
workerId
+
sequence
↓
得到 long ID
```

因此：

```text
Java-1 → 本地生成
Java-2 → 本地生成
Java-3 → 本地生成
```

不会出现：

```text
所有业务请求
↓
抢同一张 id_generator 表
↓
抢同一行数据
```

所以 Snowflake 很适合高并发、多实例场景。

这里还纠正了一个容易混淆的点：

```text
Snowflake = 本地计算
号段模式  = 批量领取
```

Snowflake 本身不是“从数据库分一段 ID 再生成”。

---

# 十二、为什么 Snowflake 只是趋势递增

Snowflake 的高位主要来自时间戳。

因此一般来说：

```text
现在生成的 ID
>
很久以前生成的 ID
```

所以它具有：

```text
趋势递增
```

这对 B+Tree 索引比较友好。

但是：

```text
趋势递增
≠
全局严格递增
```

因为多个 worker 可以并发生成：

```text
worker1
worker2
worker3
```

同一毫秒下：

```text
timestamp 相同
workerId 不同
sequence 不同
```

最终 ID 的大小关系并不能严格表达整个业务系统中谁先完成、谁后完成。

因此不能写出这种业务假设：

```text
ID 更大
=
业务一定发生得更晚
```

如果真正需要按业务时间排序，应该保留：

```text
create_time
```

并使用：

```sql
ORDER BY create_time
```

而不是把 Snowflake ID 完全当作业务时间字段。

---

# 十三、数据库号段模式是什么

号段模式不是每生成一个 ID 就访问一次数据库。

而是数据库维护类似：

```text
biz_tag     max_id     step
order       100000     10000
payment      50000      5000
```

Java 服务一次领取：

```text
100001 ~ 110000
```

放到本地内存。

后续真正生成 ID 时：

```text
100001
100002
100003
...
```

只是在内存中递增。

快用完以后，再去数据库领取下一段：

```text
110001 ~ 120000
```

这就是：

```text
Segment / 号段模式
```

---

# 十四、为什么不能每生成一个 ID 都 UPDATE 数据库 +1

方案 A：

```text
每生成 1 个订单 ID
↓
UPDATE id_generator SET id = id + 1
```

如果订单 QPS 是：

```text
50000
```

那么 ID 生成数据库也可能需要承受巨大的写压力。

更严重的是：

```text
大量请求竞争同一张表 / 同一行
↓
形成中心热点
```

真正的订单数据库可能完全正常，但是：

```text
id_generator 卡住
↓
订单拿不到 ID
↓
整个创建订单链路无法继续
```

所以方案 A 的核心问题是：

```text
每个请求都依赖中心数据库
+
热点竞争
+
中心瓶颈
+
单点故障影响核心链路
```

号段模式则是：

```text
一次数据库操作
↓
领取 10000 个 ID
↓
后续 10000 次本地生成
```

把大量数据库访问摊薄成极少数批量申请。

---

# 十五、号段模式带来的新问题

高并发场景下，相比“每个 ID 都 UPDATE 数据库”，更倾向号段模式。

但性能提高以后，也会引入新的工程问题。

## 1. 号段耗尽

当前号段：

```text
100000 ~ 109999
```

马上用完时，需要再向数据库申请下一段。

如果这时数据库突然变慢：

```text
当前号段耗尽
↓
新号段还没拿到
↓
ID 生成暂停
```

所以不能等完全耗尽再准备下一段。

## 2. 需要双 Buffer

例如：

```text
Segment A 正在使用
↓
使用到 80% / 90%
↓
后台异步申请 Segment B
↓
A 用完
↓
立即切 B
```

这样数据库申请下一号段的延迟尽量不会直接打到业务线程。

这体现了一个通用思想：

> 不要等关键资源完全耗尽再补充，应该提前预取。

## 3. 服务重启会浪费 ID

例如拿到：

```text
100000 ~ 109999
```

只用到：

```text
105000
```

服务崩溃。

剩余：

```text
105001 ~ 109999
```

通常直接废弃。

重启后申请：

```text
110000 ~ 119999
```

于是中间出现跳号。

这不是数据错误。

因为普通订单 ID 的第一要求通常是：

```text
唯一
```

而不是：

```text
绝对连续
```

为了回收几千个未使用 ID 去增加复杂状态管理，通常不值得。

---

# 十六、唯一和连续不是一个要求

例如订单 ID：

```text
10001
10005
10098
```

对于大多数业务完全没有问题。

真正需要的是：

```text
每个订单能够唯一识别
```

而不是：

```text
10001
10002
10003
一个都不能少
```

如果强制要求绝对连续，往往需要更强的：

```text
串行化
锁竞争
中心协调
```

会增加高可用难度并降低吞吐量。

所以设计 ID 前应该先问：

> 业务真的需要连续，还是只是觉得连续编号看起来更整齐？

监管票据、会计凭证等特殊编号另当别论，不能直接套普通订单 ID 的设计。

---

# 十七、Java Long 传前端的精度坑

假设数据库里保存：

```text
198273645817263104
```

Java 后端使用：

```text
Long
```

数据库使用：

```text
BIGINT
```

都没有问题。

但是前端收到 JSON：

```json
{
  "id": 198273645817263104
}
```

如果 JavaScript 把它当普通 `Number`，最后几位可能发生变化。

原因是 JavaScript `Number` 的安全整数上限只有：

```text
2^53 - 1
=
9007199254740991
```

而 Snowflake ID 很容易超过这个值。

所以看到：

```text
数据库 ID 正确
后端 Long 正确
前端最后几位偶尔变化
```

第一反应应该优先检查：

```text
JavaScript Number 精度丢失
```

常见处理方式：

```text
Java Long
↓
JSON 序列化成 String
↓
前端按字符串处理
```

例如：

```json
{
  "id": "198273645817263104"
}
```

---

# 十八、Snowflake 和号段模式怎么选

假设场景：

```text
Kubernetes 中 100 个 Java Pod
订单峰值 5 万 QPS
MySQL 已经分库
支付、退款也需要全局 ID
不能接受重复 ID
```

第一反应可以优先考虑 Snowflake：

```text
100 个 Pod
↓
每个 Pod 本地生成 ID
↓
不需要每个请求访问中心数据库
↓
吞吐可以分散到多个节点
```

优势：

```text
高性能
低延迟
BIGINT
趋势递增
无每次请求的中心网络调用
```

但 Snowflake 不能只说一句：

```text
“用 Snowflake 就行”
```

至少还必须解决：

```text
workerId 如何保证唯一
时钟回拨如何处理
时间同步如何监控
ID 重复如何报警
跨数据中心以后 workerId 怎么划分
```

而如果业务：

```text
不希望依赖机器时钟
愿意维护中心号段数据库
希望 ID 更接近连续
```

号段模式可能更稳。

所以真正的工程判断是：

> Snowflake 和号段模式都不是“绝对正确答案”，而是用不同的复杂度交换不同的性能、可用性和运维成本。

---

# 十九、ID 服务本身也存在故障爆炸半径

如果公司所有业务：

```text
订单
支付
退款
用户
```

全部依赖：

```text
Central ID Service
```

但是 ID 服务只有一个实例，那么：

```text
ID Service 挂掉
↓
多个核心系统同时无法创建新数据
```

这就是共享基础服务带来的故障爆炸半径。

所以中心 ID 架构还需要考虑：

```text
多实例
数据库高可用
本地缓存 / 号段缓存
超时
降级
监控
```

不能只关注：

```text
“ID 会不会重复”
```

还要关注：

```text
“ID 服务挂了以后业务还能不能活”
```

---

# 二十、今天容易混淆和纠正的点

## 1. BIGINT 是 8 字节，不是 8 位数字

错误理解：

```text
UUID 36 位
数据库 ID 只有 8 位
```

正确理解：

```text
BIGINT 占 8 字节
UUID 字符串常见为 36 个字符
```

---

## 2. Snowflake 不是号段模式

Snowflake：

```text
本地计算
=
时间戳 + workerId + sequence
```

号段模式：

```text
数据库批量领取一段
↓
Java 内存递增使用
```

---

## 3. workerId 相同不一定立刻重复，但破坏了唯一性基础

只有当：

```text
时间戳相同
workerId 相同
sequence 相同
```

才会生成完全相同的 Snowflake ID。

但两个实例 workerId 一旦重复，就已经产生严重风险。

---

## 4. 时钟回拨不能继续普通生成

```text
currentTimestamp < lastTimestamp
```

必须进入异常处理逻辑。

轻微回拨可以等待，大幅回拨需要拒绝、报警或使用其他策略。

---

## 5. 数据库 ID 正确、前端 ID 变化时不要只盯数据库类型

还应该第一时间想到：

```text
JavaScript Number 安全整数范围
```

---

## 6. Snowflake ID 大小不能当作业务时间的绝对真相

需要业务时间排序时：

```sql
ORDER BY create_time
```

而不是完全依赖：

```sql
ORDER BY id
```

---

# 二十一、今天形成的工程认知

以前容易把问题理解成：

```text
“怎么生成一个不会重复的数字？”
```

现在应该进一步理解成：

```text
多数据库是否会产生相同 ID
多节点如何区分身份
同一毫秒并发如何区分
时钟异常以后怎么办
ID 服务是否会成为单点
高并发下数据库会不会形成热点
服务重启后 ID 是否允许跳号
前后端协议是否会丢精度
跨机房以后如何继续保证唯一
```

今天最终形成几个核心判断：

```text
1. 单库自增正常时，不要为了高级而上 Snowflake。
2. 单库唯一不等于整个系统全局唯一。
3. Snowflake 的核心是 timestamp + workerId + sequence。
4. workerId 重复和时钟回拨是 Snowflake 最需要防的风险。
5. Snowflake 是本地计算，号段模式是批量领取，两者不要混淆。
6. 普通业务 ID 重点是唯一，不是绝对连续。
7. 分布式 ID 方案不仅要看性能，还要看高可用、监控和故障爆炸半径。
8. 真正的架构能力不是会写一个 SnowflakeUtil，而是知道什么时候该用、什么时候不该用，以及它出故障后系统会发生什么。
```

核心思想：

> 架构设计不是追求最复杂的方案，而是在业务规模、性能、正确性、可用性和维护成本之间选择足够可靠且复杂度可控的方案。
