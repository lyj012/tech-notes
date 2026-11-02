# 后端工程化每日训练 Day 75：Pika、Redis 协议兼容、RocksDB 与分层排障复盘

## 一、今天学习什么

今天从当前项目里已经部署的 Pika 出发，重点解决四个问题：

1. **Pika 到底是什么，为什么 Java 项目明明使用 Redis 客户端，后端却可以运行 Pika？**
2. **OceanBase、Pika、IoTDB 为什么可以同时存在，它们分别适合什么数据？**
3. **当 Java 报 `RedisConnectionException` 时，怎么判断问题是在 Java、网络、Pika 还是更底层？**
4. **为什么 Pika 明明还能 `PING → PONG`，真实业务请求却可能越来越慢？**

今天最重要的不是背 Redis 命令，而是建立一条完整链路：

```text
Java
 ↓
Redis 客户端 / Redis 协议
 ↓
TCP 网络
 ↓
Pika
 ↓
RocksDB
 ↓
磁盘
```

以及建立一个排障原则：

> **先根据异常判断失败发生在哪一层，再用这一层对应的证据逐步缩小范围。不要一上来把 Java、网络、Pika、RocksDB、磁盘全部乱查一遍。**

---

## 二、Pika 到底是什么

可以先把 Pika 理解成：

> **一个兼容 Redis 协议、底层使用 RocksDB 做持久化存储的 KV 数据库。**

它支持 Redis 常见的数据访问方式，因此 Java 应用可以继续使用 Redis 客户端与它交互。

例如 Java 代码：

```java
redisTemplate.opsForValue().get("host:1001:status");
```

Java 更关心的是：

```text
目标 IP
目标端口
Redis 协议
认证信息
命令
响应
```

它不一定要求目标服务必须是官方 Redis Server。

所以可能出现：

```text
Java
 ↓
RedisTemplate / Lettuce / Jedis
 ↓
Redis 协议
 ↓
Pika
```

这里最重要的认知是：

> **API / 协议，不等于具体实现产品。**

以后看到：

```text
Java 代码使用 RedisTemplate
服务器运行的却是 Pika
```

两者并不矛盾。

---

## 三、OceanBase、Pika、IoTDB 为什么可以同时存在

今天用三类数据做了区分。

### 1. 用户、主机、模板、权限等配置数据

更适合：

```text
OceanBase / MySQL 一类关系型数据库
```

因为这些数据往往强调：

```text
结构化
数据之间存在关系
事务
条件查询
JOIN
唯一约束
排序
分页
```

例如：

```text
用户
 ↓
角色
 ↓
权限

主机
 ↓
模板
 ↓
监控项
```

因此不能只说：

```text
因为 OceanBase 可以查询和更新
```

因为 Pika、IoTDB 也能读写数据。

更准确的是：

> **结构化业务数据 + 数据之间存在关系 + 事务 / 复杂查询，更适合关系型数据库。**

---

### 2. `host:1001:status = ONLINE`

更适合：

```text
Pika
```

因为访问模式很像：

```text
Key
 ↓
Value
```

例如：

```text
host:1001:status
↓
ONLINE
```

它通常不需要：

```text
复杂 JOIN
大量关系计算
按时间范围聚合
```

而是希望：

```text
通过一个 Key
快速拿到当前 Value
```

因此 Pika 这类 KV 存储非常自然。

需要注意：

> **Pika 不能简单理解成“只是缓存”。**

它本身可以进行持久化，底层数据会落到 RocksDB / 磁盘中。

---

### 3. CPU 历史数据

例如：

```text
10:00  CPU = 30%
10:01  CPU = 35%
10:02  CPU = 42%
```

更适合：

```text
IoTDB
```

因为这里的核心是：

```text
对象
+
指标
+
时间
+
值
```

真实查询通常是：

```text
这台主机最近 1 小时 CPU 怎么变化？
过去 24 小时平均 CPU 是多少？
CPU > 90% 出现在哪些时间段？
```

所以可以把三种存储先压缩成：

```text
OceanBase
→ Relation
→ 关系

Pika
→ Key
→ 键值访问

IoTDB
→ Time
→ 时间序列
```

再进一步：

```text
OceanBase：
“这些数据之间是什么关系？”

Pika：
“这个 Key 当前对应什么 Value？”

IoTDB：
“这个指标在这段时间发生了什么？”
```

---

## 四、Pika 的数据链路怎么理解

一次读取可以先简化成：

```text
Java
 ↓
GET host:1001:status
 ↓
TCP
 ↓
Pika
 ↓
查找 Key
 ↓
RocksDB / 缓存层
 ↓
磁盘数据
 ↓
返回 Value
 ↓
Java
```

所以当 Java 报：

```text
RedisConnectionException
```

背后至少可能存在这些层：

```text
Java 配置
↓
Redis 客户端 / 连接池
↓
TCP 网络
↓
Pika 监听端口
↓
Pika 进程
↓
Pika 内部
↓
RocksDB
↓
磁盘
```

因此：

```text
RedisConnectionException
```

不能直接推出：

```text
Pika 数据坏了
```

甚至不能直接推出：

```text
Pika 挂了
```

可能只是：

```text
IP 配错
端口配错
监听地址错误
网络不通
防火墙拦截
认证失败
连接池异常
Pika 没启动
```

排障必须逐层取证。

---

## 五、RocksDB 在 Pika 里是什么角色

今天不深入 LSM Tree、SST、Compaction 算法，只建立工程直觉。

可以先理解：

> **RocksDB 是 Pika 用来保存大量 KV 数据的底层持久化存储引擎。**

因此 Pika 和一个“纯内存缓存”的运维思路不同。

需要同时关注：

```text
Pika 进程
端口
CPU
内存
磁盘容量
磁盘 IO
数据目录
日志
```

数据链路可以理解成：

```text
Java 写入 Pika
↓
Pika 处理
↓
RocksDB
↓
磁盘
```

所以：

```text
磁盘空间不足
磁盘 IO 很高
数据目录权限异常
底层存储异常
```

最终都可能表现成：

```text
Java 访问 Pika 失败
或者
Java 访问 Pika 变慢
```

这就是：

```text
Java
+
中间件
+
Linux
```

真正串起来的地方。

---

## 六、进程存在为什么不能证明 Pika 正常

执行：

```bash
ps -ef | grep pika | grep -v grep
```

假设看到：

```text
root  1823  1  ./pika -c /opt/pika/conf/pika.conf
```

能够证明：

```text
Pika 进程存在
```

同时还能看到：

```text
-c /opt/pika/conf/pika.conf
```

说明当前运行中的 Pika 使用的是：

```text
/opt/pika/conf/pika.conf
```

这比直接猜：

```text
/etc/pika.conf
```

更加可靠。

但是：

```text
进程存在
≠
端口监听

端口监听
≠
协议正常

协议正常
≠
整个业务完全正常
```

这和 Java 服务完全一样：

```text
Java PID 存在
≠
8082 一定监听
≠
业务接口一定可用
```

所以看到 Pika 进程以后，还必须继续验证服务边界。

---

## 七、为什么下一步要看端口

假设 Java 报：

```text
Unable to connect to 127.0.0.1:9221
```

下一步可以检查：

```bash
ss -lntp | grep ':9221'
```

这条命令回答的是：

> **9221 这个 TCP 端口现在真的有进程在监听吗？**

如果：

```text
Pika 进程存在
+
9221 正常监听
```

证据比只看 `ps` 更强。

但依然不能直接推出：

```text
Pika 整个服务完全正常
```

下一步可以继续验证协议层。

---

## 八、Redis 的 `PING → PONG` 到底是什么意思

今天容易混淆的一点是：

```bash
redis-cli -h 127.0.0.1 -p 9221 PING
```

这里的：

```text
PING
```

不是 Linux 的：

```bash
ping 10.0.0.20
```

这两个不是同一个东西。

这里的 `PING` 是 Redis 协议中的一个命令。

如果 Pika 返回：

```text
PONG
```

可以理解成：

```text
客户端：
“你能不能正常处理一个基础 Redis 协议命令？”

Pika：
“可以，我收到了。”
```

它至少能证明：

```text
TCP 可以建立
+
目标端口能够响应
+
Redis 协议基础交互成功
```

但是不能证明：

```text
所有 Key 都能正常读写
RocksDB 性能完全正常
Java 配置完全正确
远程服务器一定能够访问
整个业务链路没有问题
```

所以：

> **PING → PONG 是一个很强的边界证据，但不是“整个系统绝对正常”的证明。**

---

## 九、本机能 PONG 为什么不能直接判断是 Java 代码问题

假设 Pika 位于：

```text
10.0.0.20
```

在 Pika 自己的服务器执行：

```bash
redis-cli -h 127.0.0.1 -p 9221 PING
```

返回：

```text
PONG
```

这里只测试了：

```text
Pika 服务器
↓
127.0.0.1:9221
↓
Pika
```

而 Java 真实访问链路可能是：

```text
Java 服务器 10.0.0.10
↓
网络
↓
Pika 服务器 10.0.0.20:9221
↓
Pika
```

所以：

```text
Pika 本机可以 PONG
```

只能说明：

```text
Pika 在本机至少可以处理基础请求
```

不能直接证明：

```text
Java 到 Pika 的远程网络也正常
```

因此还需要从 Java 所在服务器验证：

```bash
nc -vz 10.0.0.20 9221
```

或者：

```bash
redis-cli -h 10.0.0.20 -p 9221 PING
```

这里真正验证的是：

```text
Java 所在服务器
↓
10.0.0.20:9221
↓
Pika
```

也就是 Java 实际需要走的路径。

---

## 十、127.0.0.1 和 0.0.0.0 必须分清

今天一个非常关键的知识点是监听地址。

### 1. `127.0.0.1:9221`

如果：

```bash
ss -lntp | grep ':9221'
```

看到：

```text
127.0.0.1:9221
```

表示：

> **Pika 只监听本机回环地址。**

于是：

```text
Pika 服务器自己
↓
127.0.0.1:9221
↓
可以访问
```

但另一台 Java 服务器：

```text
Java
↓
10.0.0.20:9221
↓
访问失败
```

就非常合理。

此时不能笼统地只说：

```text
网络坏了
```

因为已经有比较强的证据指向：

```text
Pika bind / 监听地址配置
```

---

### 2. `0.0.0.0:9221`

如果看到：

```text
0.0.0.0:9221
```

表示：

> **Pika 正在监听本机所有 IPv4 网卡。**

需要特别注意：

```text
0.0.0.0
```

不是 Java 应该连接的目标 IP。

Java 仍然应该连接 Pika 服务器真实 IP，例如：

```text
10.0.0.20:9221
```

所以：

```text
127.0.0.1
→ 只监听本机

0.0.0.0
→ 监听本机所有 IPv4 网卡

10.0.0.20
→ 服务器真实 IP
→ 其他服务器实际连接的目标
```

这是三个完全不同的概念。

---

## 十一、0.0.0.0 正常监听但远程 timeout 怎么判断

假设 Pika 服务器：

```text
0.0.0.0:9221 正常监听
```

本机：

```bash
redis-cli -h 127.0.0.1 -p 9221 PING
```

返回：

```text
PONG
```

但是 Java 服务器执行：

```bash
nc -vz 10.0.0.20 9221
```

一直：

```text
timeout
```

这时候已经可以确认几件事：

```text
Pika 进程存在
Pika 端口正常
Pika 本机协议基础交互正常
Pika 不是只监听 localhost
```

所以排查重点应该转向：

```text
两台服务器之间的网络
防火墙
路由
安全策略
ACL
端口放行
```

这时候不能继续反复重启 Pika。

真正的问题已经从：

```text
Pika 本机是否工作
```

收敛为：

```text
Java 服务器能不能通过网络到达 Pika 的 9221
```

---

## 十二、为什么只 Ping IP 还不够

例如：

```bash
ping 10.0.0.20
```

成功只能说明：

```text
当前 ICMP 探测能够到达目标 IP
```

它不能证明：

```text
10.0.0.20:9221
```

这个 TCP 端口一定能访问。

所以对于：

```text
Java → Pika
```

真正关心的是：

```text
Java 服务器
↓
TCP
↓
10.0.0.20:9221
```

因此：

```bash
nc -vz 10.0.0.20 9221
```

或者：

```bash
redis-cli -h 10.0.0.20 -p 9221 PING
```

比单纯：

```bash
ping 10.0.0.20
```

更接近真实业务路径。

这和以前学 Java 服务完全一致：

```text
Ping 通
≠
8082 一定通

IP 可达
≠
目标 TCP 服务可达
```

---

## 十三、连接不上和能连接但很慢是两类问题

今天最后把故障分成了两个大方向。

### 第一类：根本连不上

典型异常：

```text
Unable to connect
Connection refused
Connection timeout
RedisConnectionException
```

排查重点：

```text
Java 配置
↓
目标 IP / 端口
↓
网络可达性
↓
Pika 监听
↓
监听地址
↓
Pika 进程
↓
认证
↓
连接池 / 客户端配置
```

这类问题优先回答：

> **连接为什么建立不了？**

---

### 第二类：能连，但是越来越慢

例如：

```text
Java 接口以前 100ms
现在 3～5 秒
```

同时：

```text
Pika 进程存在
9221 正常监听
PING 能返回 PONG
```

这时候问题已经不是：

```text
完全无法建立连接
```

而是：

```text
真实数据访问性能为什么下降
```

此时才应该重点检查：

```text
CPU
内存
磁盘容量
磁盘 IO
RocksDB
Pika 日志
```

---

## 十四、`df -h` 看的是磁盘，不是内存

今天答题时一个容易混淆的点：

```bash
df -h
```

假设输出：

```text
/dev/vda1  100G  97G  3G  97%
```

它不是：

```text
内存只剩 3G
```

而是：

> **这块文件系统总共 100G，已经使用 97G，只剩 3G 磁盘空间。**

内存应该看：

```bash
free -h
```

所以必须区分：

```text
df -h
→ 磁盘容量

free -h
→ 内存

iostat -x 1
→ 磁盘 IO 设备压力 / 延迟等指标
```

这是 Linux 排障里必须形成的基本反射。

---

## 十五、为什么 Pika 能 PONG，但 Java 接口还是很慢

假设：

```text
Pika PING → PONG
```

同时：

```bash
df -h
```

发现：

```text
磁盘使用率 97%
```

再执行：

```bash
iostat -x 1
```

发现：

```text
磁盘 IO 压力很高
```

为什么这两件事可以同时存在？

因为：

```text
PING
```

是一个非常轻量的 Redis 协议命令。

链路大致只是：

```text
客户端
↓
PING
↓
Pika
↓
PONG
```

它主要证明：

```text
TCP 还能通信
+
Pika 还能处理基础协议请求
```

但真实业务的 GET / SET 可能需要：

```text
Java
↓
Pika
↓
查找 / 修改数据
↓
RocksDB
↓
磁盘读写
↓
IO 排队
↓
Pika 很晚才能返回
↓
Java 一直等待
↓
接口 RT 上升
```

所以：

```text
PING 成功
≠
真实数据读写性能正常
```

这和 Java 服务里的：

```text
/health 10ms
```

但：

```text
/order/list 5s
```

完全可以同时存在。

因为健康检查可能几乎不做重活，而真实业务还需要访问数据库、缓存、磁盘和其他下游。

---

## 十六、磁盘 IO 为什么会影响 Pika

因为 Pika 的数据链路里存在：

```text
Pika
↓
RocksDB
↓
磁盘
```

如果磁盘出现：

```text
空间越来越紧
IO 排队严重
读写延迟升高
```

那么 RocksDB 的真实数据访问就可能变慢。

进一步还可能存在：

```text
Flush
Compaction
后台文件处理
```

这些工作也会消耗磁盘 IO。

所以可能形成：

```text
数据量增长
↓
RocksDB 磁盘读写压力增加
↓
磁盘成为瓶颈
↓
Pika GET / SET 变慢
↓
Java 等待时间增加
↓
接口 RT 升高
↓
最终甚至出现 timeout
```

此时只去调：

```text
Java 线程池
```

可能完全没有解决真正根因。

---

## 十七、完整 RedisConnectionException 排障顺序

今天最后形成了一条比较完整的排障链。

假设线上突然大量出现：

```text
RedisConnectionException
```

推荐先按下面顺序收敛。

### 第一步：先确认 Java 到底要连谁

先确认：

```text
IP 是多少？
端口是多少？
认证信息是什么？
当前环境配置有没有写错？
```

例如 Java 真正配置的是：

```text
10.0.0.20:9221
```

先把目标确认清楚。

---

### 第二步：从 Java 所在服务器测试目标 TCP 端口

例如：

```bash
nc -vz 10.0.0.20 9221
```

或者直接：

```bash
redis-cli -h 10.0.0.20 -p 9221 PING
```

这里回答：

> **Java 所在服务器能不能走真实路径访问 Pika？**

---

### 第三步：如果远程连不上，再到 Pika 服务器检查

检查：

```text
Pika 进程
9221 端口监听
监听地址
```

例如：

```bash
ps -ef | grep pika | grep -v grep
ss -lntp | grep ':9221'
```

---

### 第四步：判断监听地址

如果：

```text
127.0.0.1:9221
```

重点怀疑：

```text
只监听 localhost
```

如果：

```text
0.0.0.0:9221
```

说明已经监听所有 IPv4 网卡，继续排查网络路径。

---

### 第五步：Pika 本机直接验证协议

```bash
redis-cli -h 127.0.0.1 -p 9221 PING
```

如果：

```text
PONG
```

说明：

```text
Pika 本机基础协议服务可以响应
```

---

### 第六步：本机正常、远程失败

此时重点查：

```text
防火墙
网络策略
路由
ACL
端口放行
中间网络设备
```

而不是继续无差别重启 Pika。

---

### 第七步：TCP 和 Pika 都正常

如果：

```text
Java 服务器可以连接 10.0.0.20:9221
Pika 本机可以 PONG
远程也可以 PONG
```

但 Java 应用仍然报错，则重点回到 Java 侧：

```text
Spring Redis 配置
密码 / 认证
连接池
客户端参数
连接超时
代码调用
环境变量
配置中心
```

---

## 十八、完整排障模型

可以最终压缩成：

```text
Java 报 RedisConnectionException
        ↓
1. Java 实际配置的 IP / 端口是什么？
        ↓
2. Java 服务器能否访问 Pika_IP:9221？
        ↓
   ┌────┴────┐
   │         │
  不能       能
   │         │
   ↓         ↓
3. Pika     6. 再查 Java
   进程        认证
   端口        连接池
   监听地址    客户端
   │           配置
   ↓
4. Pika 本机 PING → PONG？
   │
   ├─ 否
   │  ↓
   │  查 Pika / 配置 / 日志
   │
   └─ 是
      ↓
5. 本机正常，远程不通
      ↓
   查防火墙 / 路由 / ACL / 网络
```

如果问题不是“连接失败”，而是：

```text
能连接
但是慢
```

则转成：

```text
Pika 是否本身就慢？
↓
CPU
内存
磁盘容量
磁盘 IO
RocksDB
日志
↓
还是只有 Java 慢？
↓
Java 连接池
线程
客户端
调用逻辑
```

---

## 十九、今天容易混淆的几个点

### 1. Pika = Redis？

不准确。

更准确：

```text
Pika
→ Redis 协议兼容的 KV 数据库
→ 底层使用 RocksDB 做持久化存储
```

Java 可以使用 Redis 客户端与它交互，但 Pika 不等于 Redis Server 本身。

---

### 2. Pika = 缓存？

不准确。

Pika 可以承担很多 Redis 类 KV 访问场景，但它本身具有持久化存储能力。

所以：

```text
缓存类访问模式
≠
数据一定只在内存里
```

---

### 3. 进程存在 = 服务正常？

错误：

```text
Pika PID 存在
→ Pika 完全正常
```

正确：

```text
PID 存在
→ 只能证明进程还存在

还需要继续验证：
端口
监听地址
协议
真实业务路径
```

---

### 4. `PING → PONG` = 整个系统正常？

错误。

正确：

```text
PING → PONG
→ 证明基础 TCP + Redis 协议交互成功
→ 不能证明所有真实数据操作和业务链路都正常
```

---

### 5. `127.0.0.1` 和 `0.0.0.0`

```text
127.0.0.1
→ 只监听本机回环地址

0.0.0.0
→ 监听本机所有 IPv4 网卡
→ 不是给 Java 连接的目标 IP
```

Java 实际连接的是：

```text
Pika 服务器真实 IP
```

例如：

```text
10.0.0.20
```

---

### 6. `df -h` = 看内存？

错误。

```text
df -h
→ 磁盘容量

free -h
→ 内存

iostat -x 1
→ 磁盘 IO
```

---

### 7. IP 能 Ping 通 = Pika 端口一定能通？

错误。

```text
Ping IP 成功
→ ICMP 可达

nc 10.0.0.20 9221 成功
→ 目标 TCP 端口可达

redis-cli ... PING → PONG
→ Redis 协议基础交互成功
```

证据是一层一层加强的。

---

## 二十、今天形成的最终模型

今天最需要记住的是：

```text
OceanBase
→ 关系型业务数据

Pika
→ Key-Value 快速访问 + 持久化

IoTDB
→ 时间序列历史数据
```

Pika 链路：

```text
Java
↓
Redis 客户端 / 协议
↓
TCP 网络
↓
Pika
↓
RocksDB
↓
磁盘
```

服务可用性判断：

```text
进程存在
≠
端口监听

端口监听
≠
协议正常

协议正常
≠
远程网络正常

远程网络正常
≠
Java 配置一定正确

PING 能 PONG
≠
真实 GET / SET 性能正常
```

故障排查：

```text
连不上
→
优先查配置 / 网络 / 监听 / 进程 / 认证

能连但很慢
→
继续查 Pika / RocksDB / CPU / 内存 / 磁盘 / IO
```

今天真正训练的不是几个 Pika 命令，而是三种长期可迁移能力：

> **第一，理解“协议”和“具体实现产品”是两个层次，Java 使用 Redis 客户端并不代表后端一定运行 Redis Server。**
>
> **第二，把 Java、中间件、网络、底层存储和 Linux 资源串成一条完整链路，在组件边界直接取证。**
>
> **第三，根据异常类型选择排查方向：连接失败先查连接链路，性能下降再查资源与底层存储，不进行无差别排查。**

这些方法以后即使 Pika 换成 Redis、数据库换成 MySQL / OceanBase、消息中间件换成 Kafka / RocketMQ，仍然可以继续复用。
