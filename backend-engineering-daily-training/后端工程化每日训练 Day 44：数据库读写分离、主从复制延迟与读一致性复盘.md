# 后端工程化每日训练 Day 44：数据库读写分离、主从复制延迟与读一致性复盘

## 一、今天学习什么

今天主要学习：

- Master / Slave 主从架构
- 数据库读写分离
- 主从复制延迟
- Read-After-Write Consistency（读后写一致性）
- 强一致查询与弱一致查询的路由判断
- Slave 故障后的流量切换风险
- 唯一索引、唯一约束与 Redis 在“名称判重”中的作用

今天最核心的判断：

> 不能因为一条 SQL 是 `SELECT`，就机械地让它读取 Slave。真正应该判断的是：读到旧数据会造成什么业务后果。

---

# 二、为什么要做读写分离

一个数据库同时承担：

```text
INSERT
UPDATE
DELETE
SELECT
```

高峰期可能出现：

```text
CPU 上升
磁盘 IO 上升
连接数上涨
查询 RT 上升
请求超时
```

很多业务通常是：

```text
读请求 >> 写请求
```

因此可以设计：

```text
INSERT / UPDATE / DELETE → Master
SELECT                   → Slave A / B / C
```

核心收益：

```text
大量 SELECT 分散到多个 Slave
↓
提高读吞吐量
↓
降低 Master 压力
```

需要注意：

> 读写分离主要扩展的是读能力，并不是增加几个 Slave 就能直接提高写能力。

---

# 三、主从复制延迟

Master 写成功后，Slave 不一定同一瞬间已经同步完成。

例如：

```text
10:00:00.000
Master 更新 nickname = liu

10:00:00.150
Slave 才同步完成
```

那么这 150ms 内：

```text
Master = 新数据
Slave  = 旧数据
```

如果此时查询被路由到 Slave，就可能读到旧值。

例如用户刚修改昵称：

```text
user001 → liu
```

接口已经返回：

```text
修改成功
```

用户立刻刷新：

```text
SELECT → Slave
↓
Slave 尚未同步
↓
仍然返回 user001
```

过一会再次查询：

```text
Slave 已同步
↓
返回 liu
```

这就是典型的：

> 主从复制延迟导致的读后写不一致。

复制延迟并不是固定值，还可能因为以下因素扩大：

- Master 写入量暴涨
- Slave CPU 不足
- Slave 磁盘 IO 变慢
- 大事务
- 网络抖动
- 复制线程异常

所以不能假设：

```text
“Slave 一定在 500ms / 1s 内同步完成”
```

---

# 四、Read-After-Write Consistency

读后写一致性的核心要求：

> 我刚刚写进去的数据，我下一次读取时应该能够看到。

例如创建订单：

```text
POST /orders
↓
INSERT → Master
↓
创建成功
```

紧接着：

```text
GET /orders/10001
```

如果读取 Slave，而 Slave 还没有同步：

```text
Master：订单 10001 已存在
Slave：订单 10001 不存在
```

前端可能出现：

```text
刚提示“创建成功”
↓
马上又提示“订单不存在”
```

因此这类关键的写后立即读，更适合：

```text
写 Master
↓
立即查询仍读 Master
```

但这不代表订单以后永远都要读取 Master。

例如：

```text
刚创建后的订单详情 → Master
普通历史订单列表   → Slave
```

---

# 五、不能把所有 SELECT 都机械路由到 Slave

两个接口都是 `SELECT`：

```text
商品首页列表
账户余额
```

但业务风险完全不同。

## 商品首页

如果数据晚几百毫秒甚至几秒：

```text
商品描述刚修改
↓
用户暂时看到旧内容
↓
稍后刷新恢复
```

通常只是展示延迟。

所以：

```text
商品首页 → Slave
```

通常合理。

## 余额、支付、权限、库存

如果读取旧数据：

```text
支付状态错误
余额判断错误
权限状态错误
库存核心判断错误
退款状态错误
```

就可能影响真正的业务决策。

因此更应该关注：

```text
读到旧数据会造成什么损失？
```

而不是只看：

```text
这是不是 SELECT？
```

---

# 六、为什么 Thread.sleep 不能解决复制延迟

错误方案：

```java
updateMaster();

Thread.sleep(1000);

querySlave();
```

想法是：

```text
“等 1 秒，Slave 应该同步好了。”
```

但复制延迟可能是：

```text
正常：50ms
高峰：2s
异常：30s
```

因此：

```text
sleep(1000)
```

没有任何一致性保证。

同时它还会伤害 Java 服务本身：

```text
请求进入
↓
占用 Tomcat 工作线程
↓
sleep 1 秒
↓
线程什么都没做，但仍然被占用
```

高并发下可能出现：

```text
大量线程 sleep
↓
线程池被占满
↓
后续请求排队
↓
RT 上升
↓
请求超时
```

所以正确思路应该是：

```text
要求强一致
↓
明确读取 Master
```

而不是：

```text
猜一个等待时间
↓
再读取 Slave
```

---

# 七、并发 + 主从延迟：优惠券重复领取

代码：

```java
if (!couponMapper.exists(userId, couponId)) {
    couponMapper.insert(userId, couponId);
}
```

假设：

```text
exists() → Slave
insert() → Master
```

两个请求同时到达：

```text
请求 A → Slave 查询 → 没有
请求 B → Slave 查询 → 没有
```

然后：

```text
请求 A → Master INSERT → 成功
请求 B → Master INSERT → 再次插入
```

如果数据库没有约束，就可能出现两条相同领取记录。

数据库最后应该有：

```sql
UNIQUE (user_id, coupon_id)
```

最终：

```text
A INSERT → 成功
B INSERT → 唯一键冲突
```

数据库中只能保留一条记录。

需要进一步注意：

> 即使 `exists()` 改成查询 Master，也不能单靠“先查再插”解决并发问题。

仍然可能：

```text
A 查 Master → 没有
B 查 Master → 没有
A INSERT
B INSERT
```

因此最终正确性应该由多层保护完成：

```text
正确的数据路由
+
业务幂等
+
数据库唯一约束
```

---

# 八、Slave 全挂后自动切 Master 的风险

假设：

```text
Slave A = 8000 QPS
Slave B = 8000 QPS
Master  = 3000 QPS
```

两个 Slave 同时故障。

自动切换：

```text
所有 SELECT
↓
全部进入 Master
```

好处：

```text
Slave 故障后
仍然有机会继续提供查询能力
```

但风险是：

```text
16000 QPS 查询流量回流 Master
+
Master 原有 3000 QPS
↓
Master 瞬间接近 19000 QPS
```

如果 Master 极限只有 10000 QPS：

```text
Slave 故障
↓
流量全部切 Master
↓
Master CPU / IO / 连接数打满
↓
Master 也不可用
↓
读写全部受影响
```

这就是：

> 故障转移引发级联故障。

因此故障切换不能只考虑“能不能切”，还要考虑：

```text
剩余节点有没有容量接住流量？
```

需要配合：

- 限流
- 降级
- 容量保护

例如：

```text
关键查询 → Master
非关键查询 → 限流 / 降级
报表统计 → 暂停
```

优先保护核心交易。

---

# 九、所有 SELECT 永久读取 Master 是否合理

如果所有查询都改成：

```text
SELECT → Master
```

确实可以避免：

```text
因为 Slave 同步延迟而读到旧数据
```

但代价是：

```text
Slave 的读扩展能力被浪费
↓
Master QPS 上升
↓
CPU / IO / 连接数压力上升
↓
读容量下降
```

这相当于重新把读压力放回 Master。

但如果系统本身：

```text
QPS 不高
Master 资源充足
没有明显读瓶颈
```

那么全部读取 Master，甚至直接使用单库，反而可能更合理。

核心原则：

> 不要为了架构看起来高级而增加复杂度，应该用满足当前业务规模的最小复杂度。

---

# 十、报表与统计不要和交易主库抢资源

后台日报、经营统计通常包含：

```sql
COUNT(*)
SUM(amount)
GROUP BY
```

可能扫描大量数据。

这种查询通常：

```text
不要求秒级强一致
+
CPU / IO 消耗高
```

如果长期直接跑在交易 Master：

```text
报表查询
↓
占用 CPU / IO / Buffer Pool
↓
和下单、支付、退款争资源
```

规模扩大后更合理的方向可能是：

```text
普通查询 → Slave
报表 / 经营统计 → 报表库 / OLAP / 分析库
核心交易 → Master
```

这说明数据库设计不仅要考虑：

```text
SQL 能不能跑
```

还要考虑：

```text
不同工作负载应该不应该放在一起
```

---

# 十一、综合业务分类

今天最后按业务风险进行了分类。

## 可以接受 Slave 延迟

```text
商品首页列表
历史订单列表
普通历史记录
```

短暂旧数据通常只影响展示。

## 需要强一致 / 写后读 Master

```text
支付状态
刚修改完的个人资料
实时库存核心判断
余额
权限
退款状态
```

这些数据如果读取旧值，可能直接影响业务正确性。

## 应该考虑独立分析资源

```text
后台日报
经营统计
大型聚合报表
```

这些查询既允许一定延迟，又可能非常消耗资源，更适合报表库、OLAP 或分析库。

---

# 十二、延伸：游戏 ID / 用户名为什么能瞬间判断重复

今天额外思考了一个实际问题：

> 一个游戏可能有几千万甚至上亿用户，为什么输入一个 ID 后，系统可以很快告诉我“已被占用”或“可以使用”？

关键不是把所有用户名从头扫描一遍，而是建立索引。

例如：

```sql
CREATE UNIQUE INDEX uk_username
ON user(username);
```

查询：

```sql
SELECT 1
FROM user
WHERE username = ?
LIMIT 1;
```

数据库可以通过索引快速判断目标值是否存在，而不是执行全表扫描。

同时：

```text
INDEX
→ 解决快速查找

UNIQUE
→ 解决最终不能重复
```

大型系统还可能使用 Redis 做加速：

```text
username:liu123 = occupied
```

流程可能是：

```text
用户输入 username
↓
Redis 快速检查
↓
必要时查询数据库索引
↓
最终注册时执行 INSERT
↓
数据库 UNIQUE 兜底
```

注册成功以后，也可能把占用信息写入 Redis，减少后续数据库查询压力。

但需要注意：

> Redis 只是可选的性能加速层，不能作为最终唯一性的保证。

因为可能出现：

```text
数据库已经注册成功
↓
Redis 尚未更新
↓
另一个请求检查 Redis
↓
暂时认为名字可用
```

最终注册仍然必须依赖数据库唯一约束：

```text
请求 A → INSERT → 成功
请求 B → INSERT → UNIQUE 冲突
```

因此可以总结成：

```text
Redis
→ 降低查询压力、提高速度

数据库索引
→ 快速判断是否存在

数据库 UNIQUE
→ 最终保证不能重复
```

这和优惠券场景是同一个工程原则：

> “先查有没有”只是预检查，最终正确性必须有不可绕过的约束兜底。

---

# 十三、今天形成的工程认知

以前看到读写分离，容易理解成：

```text
写 → Master
读 → Slave
```

今天进一步理解为：

```text
弱一致读取 → Slave
强一致读取 → Master
重型分析读取 → 独立分析资源
```

同时形成几个重要判断：

```text
查询慢
≠
读取旧数据
```

```text
节点存活 + HTTP 200
≠
业务数据一定正确
```

```text
先查再插
≠
并发场景下绝对安全
```

```text
自动故障转移
≠
无脑把所有流量切到剩余节点
```

```text
缓存
→ 主要解决性能

数据库约束
→ 负责最终正确性
```

今天最重要的工程思想：

> 数据库架构不能只从 SQL 类型出发，而要从一致性、性能、业务损失、容量和故障风险出发决定数据应该从哪里读，以及最终正确性由什么机制保证。
