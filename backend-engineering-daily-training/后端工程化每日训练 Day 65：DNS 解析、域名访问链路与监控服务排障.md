# 后端工程化每日训练 Day 65：DNS 解析、域名访问链路与监控服务排障

## 一、今天学习什么

今天继续补全监控系统网络排障链路。

之前已经学习：

- Day 59：ARP、MAC 地址与邻居表；
- Day 60：交换机 MAC 地址表与二层转发；
- Day 61：VLAN、Access、Trunk 与二层隔离；
- Day 63：ACL、防火墙访问控制与 Timeout 排障；
- Day 64：TCP 三次握手、连接状态与监控端口排障。

今天继续补充一个经常被忽略的前置环节：

**DNS 解析、域名访问链路，以及 Java 服务和监控采集中的域名故障排查。**

实际工作中目标地址不一定直接配置成 IP，也可能是：

```text
db.monitor.local
device.company.com
api.monitor.com
```

程序真正访问目标服务前，需要先把域名解析成 IP。

完整链路可以理解为：

```text
域名
 ↓
DNS解析
 ↓
目标IP
 ↓
路由选择
 ↓
ARP / 二层转发
 ↓
TCP / UDP
 ↓
应用协议
```

---

## 二、DNS 是什么

DNS：

```text
Domain Name System
```

核心作用：

> 把域名解析成网络通信真正需要使用的 IP 地址。

例如 Java 配置：

```yaml
spring:
  datasource:
    url: jdbc:mysql://db.monitor.local:3306/monitor
```

程序不能直接拿 `db.monitor.local` 去进行 IP 层通信。

它需要先得到类似：

```text
db.monitor.local
        ↓
      DNS
        ↓
   10.10.20.50
```

然后才能继续访问：

```text
10.10.20.50:3306
```

---

## 三、直接访问 IP 与访问域名的区别

例如：

```bash
ping 10.0.0.10
```

成功。

这只能说明：

> 当前机器到 `10.0.0.10` 的 ICMP 网络通信基本可达。

不能直接证明：

- `db.xxx.com` 一定解析到 `10.0.0.10`；
- TCP 3306 一定开放；
- MySQL 一定正常；
- 应用一定能正常访问数据库。

直接访问 IP：

```text
客户端
 ↓
已经知道目标IP
 ↓
路由
 ↓
TCP / UDP
 ↓
目标服务
```

访问域名：

```text
db.xxx.com
 ↓
DNS解析
 ↓
得到目标IP
 ↓
路由
 ↓
TCP / UDP
 ↓
目标服务
```

所以：

```text
IP访问成功
≠
域名访问一定成功
```

因为域名访问比直接访问 IP 多了 DNS 解析这一步。

---

## 四、DNS 与 TCP 三次握手的关系

访问一个 TCP 服务时，客户端首先要知道目标 IP。

例如：

```text
db.xxx.com
 ↓
DNS解析
 ↓
10.0.0.10
 ↓
向10.0.0.10发送SYN
 ↓
TCP三次握手
```

### 情况 1：DNS 完全解析失败

```text
db.xxx.com
 ↓
没有得到IP
 ↓
无法确定目标地址
 ↓
不会向目标数据库发送SYN
 ↓
目标服务的TCP三次握手不会开始
```

Java 中常见：

```text
java.net.UnknownHostException
```

### 情况 2：DNS 解析到了错误 IP

例如本来应该解析到：

```text
10.0.0.10
```

却返回：

```text
10.0.0.20
```

此时 TCP 不一定完全没有发生，而是可能：

```text
客户端
 ↓
向错误IP发送SYN
 ↓
连接了错误目标
```

可能表现为：

```text
Connection timeout
```

也可能：

```text
Connection refused
```

甚至错误 IP 上刚好存在服务时，TCP 还可能建立成功，只是在后续应用层才发现访问目标不对。

因此需要区分：

```text
DNS解析失败
→ 没拿到目标IP

DNS解析错误
→ 拿到了IP，但目标可能错了
```

---

## 五、DNS 本身也依赖网络

今天需要避免一个误区：

> DNS 在目标业务服务连接之前，不代表 DNS 在“网络之前”。

DNS 查询本身也是一次网络通信。

例如服务器向 DNS Server 查询：

```text
客户端
 ↓
查找到DNS服务器的路由
 ↓
ARP / 二层转发
 ↓
UDP/TCP 53
 ↓
DNS服务器
 ↓
返回解析结果
```

所以更准确的理解是：

```text
DNS：先解决目标是谁
路由：再解决怎么到它
TCP：再解决怎么建立连接
应用协议：最后进行业务通信
```

---

## 六、DNS 排查命令

### 1. nslookup

```bash
nslookup db.monitor.local
```

重点看：

```text
1. 能不能解析出IP
2. 解析出来的IP是不是预期地址
```

---

### 2. dig

```bash
dig db.monitor.local
```

用于查看更详细的 DNS 查询结果。

---

### 3. DNS 配置

Linux 可以检查：

```bash
cat /etc/resolv.conf
```

关注：

```text
nameserver
```

是否配置正确。

---

### 4. 不要把 ping 当成 DNS 专用检查

例如：

```bash
ping db.monitor.local
```

实际上同时包含：

```text
DNS解析
+
ICMP通信
```

如果失败，还需要继续区分：

```text
是域名没解析出来？
还是解析成功但ICMP不通？
```

因此检查 DNS 时，优先使用：

```bash
nslookup
```

或：

```bash
dig
```

---

## 七、ping 不能检测 TCP 端口

今天还纠正了一个容易混淆的点。

```bash
ping db.monitor.local
```

不能证明 MySQL 3306 可以访问。

因为 Ping 使用：

```text
ICMP
```

而 MySQL 常见连接使用：

```text
TCP 3306
```

测试 TCP 端口可以使用：

```bash
nc -vz db.monitor.local 3306
```

或者环境中存在 telnet 时：

```bash
telnet db.monitor.local 3306
```

所以：

```text
Ping成功
≠
TCP端口可用
```

---

## 八、Java 数据库连接失败如何排查

场景：

```text
监控平台提示数据库连接失败
```

配置：

```yaml
host: db.monitor.local
```

真正高效的排查方式，不是机械地从网络第一层全部检查一遍，而是：

> 先看 Java 完整异常，再根据异常缩小故障范围。

### 1. UnknownHostException

```text
java.net.UnknownHostException: db.monitor.local
```

优先判断：

```text
域名解析失败
```

检查：

```bash
nslookup db.monitor.local
```

---

### 2. Connection timeout

如果已经成功解析出 IP，但连接一直没有有效响应，需要继续检查：

```text
目标IP
路由
ACL
防火墙
目标服务响应
```

---

### 3. Connection refused

通常说明请求已经到达目标主机，但目标端口没有正常监听，或者被主动拒绝。

重点检查：

```text
服务是否启动
端口是否监听
端口配置是否正确
```

---

### 4. Access denied for user

例如 MySQL 返回：

```text
Access denied for user
```

这时网络问题通常已经不是首要矛盾。

因为请求基本已经走到：

```text
DNS
 ↓
IP
 ↓
网络
 ↓
TCP
 ↓
MySQL
 ↓
认证阶段
```

下一步应该重点检查：

```text
用户名
密码
来源主机权限
数据库授权
```

---

## 九、几个 Linux 命令分别看什么

今天进一步明确了不同命令的职责。

### ip addr

```bash
ip addr
```

主要查看：

```text
本机网卡
本机IP
接口状态
```

不是用来确认目标服务器 IP 是否正确。

---

### ip route get

假设 DNS 已经解析出：

```text
10.10.20.50
```

可以执行：

```bash
ip route get 10.10.20.50
```

确认：

```text
目标走哪个网卡
下一跳是谁
源IP是什么
```

---

### ip neigh

```bash
ip neigh
```

检查邻居表状态。

如果目标或下一跳在本地二层网络中，可以辅助判断 ARP / 二层邻居是否异常。

---

## 十、SNMP Timeout 与 DNS

场景：

```text
Collector采集失败
SNMP timeout
```

配置不是 IP，而是：

```text
device.company.com
```

完整链路：

```text
Collector配置
 ↓
DNS解析
 ↓
目标IP
 ↓
路由
 ↓
ARP / VLAN / 二层转发
 ↓
ACL / 防火墙
 ↓
UDP 161
 ↓
设备SNMP Agent
 ↓
SNMP Response
 ↓
Collector
```

需要注意：

> SNMP 常见使用 UDP 161，没有 TCP 三次握手。

`SNMP timeout` 只能说明：

```text
在规定时间内没有收到有效SNMP响应
```

它不能直接说明故障在哪一层。

可能是：

- DNS 解析失败或解析错误；
- 路由异常；
- VLAN / 二层链路异常；
- ACL / 防火墙丢弃；
- SNMP Agent 未开启；
- SNMP Version / Community / 用户认证错误；
- OID 或设备 MIB 不匹配；
- 响应返回途中被丢弃。

---

## 十一、SNMP 故障先看影响范围

今天进一步明确：

遇到 SNMP Timeout，不应该一上来就固定按照“硬件 → 软件”全部检查。

应该先看故障范围。

### 只有一台设备失败

优先怀疑：

```text
设备自身
设备地址
SNMP参数
设备ACL
SNMP Agent
```

### 整个 VLAN 都失败

优先怀疑公共链路：

```text
VLAN
Trunk
三层路由
ACL
防火墙
```

### 所有设备突然失败

优先怀疑：

```text
Collector
公共网络出口
公共配置
DNS
统一访问策略
```

核心原则：

> 故障范围越大，越应该优先寻找公共故障点。

---

## 十二、结合当前监控平台的实际排障方式

实际工作中，模板导入后如果采集失败，监控平台本身已经提供很多信息，例如：

```text
采集次数
错误信息
Java异常日志
采集详情
```

因此不需要脱离现有证据，从零猜测故障原因。

更高效的流程：

```text
导入模板
 ↓
采集失败
 ↓
打开采集详情
 ↓
收集错误信息 + Java异常栈 + 目标设备基本信息
 ↓
使用AI辅助判断故障层级
 ↓
再用Linux命令 / 配置 / 网络设备信息验证
 ↓
最终定位问题
```

AI 的价值是：

```text
快速缩小范围
```

但最终仍然需要理解判断依据并验证。

例如：

```text
UnknownHostException
→ 优先查DNS

Connection refused
→ 优先查目标端口和服务监听

Timeout
→ 继续查网络、ACL、防火墙、目标响应

认证 / Community失败
→ 查SNMP参数

noSuchObject / noSuchInstance
→ 查OID、MIB、设备型号和模板匹配

字段解析 / 映射异常
→ 查模板、代码、返回数据格式
```

真正需要形成的能力不是：

```text
完全不用AI，靠记忆解决所有故障
```

而是：

```text
系统提供证据
 ↓
AI辅助缩小范围
 ↓
自己理解为什么
 ↓
实际命令验证
 ↓
确认最终故障点
```

---

## 十三、今天建立的完整排障链路

以后遇到域名形式的目标地址，可以先在脑中还原：

```text
配置
 ↓
域名
 ↓
DNS解析
 ↓
目标IP
 ↓
路由选择
 ↓
ARP / 二层转发
 ↓
VLAN / Trunk
 ↓
ACL / 防火墙
 ↓
TCP / UDP
 ↓
应用协议
 ↓
目标服务
 ↓
返回链路
```

但实际排障不应该每次全部从头检查。

更高效的方式是：

```text
先看现象和日志
 ↓
判断大概在哪一层
 ↓
只验证最可能的故障点
 ↓
证据不支持再继续向上下游扩展
```

---

## 十四、训练题复盘

### 1. 为什么 IP 可以访问，但域名可能失败？

因为直接访问 IP 不需要 DNS，而访问域名必须先解析出目标 IP。

DNS 失败或解析错误时，即使某个 IP 本身网络可达，域名访问仍然可能失败。

---

### 2. DNS 在目标服务访问链路中的什么位置？

对于目标业务服务来说：

```text
域名
↓
DNS
↓
目标IP
↓
路由
↓
TCP/UDP
```

DNS 是连接目标业务服务之前的前置步骤。

但 DNS 查询本身也需要网络通信。

---

### 3. 为什么 DNS 失败后 TCP 三次握手可能根本不会发生？

因为没有解析出目标 IP，就无法确定 SYN 应该发送给哪个目标地址。

---

### 4. ping 目标 IP 成功，但域名失败，优先查什么？

优先：

```bash
nslookup 域名
```

或：

```bash
dig 域名
```

检查是否能解析，以及解析出的 IP 是否正确。

---

### 5. Java 出现 UnknownHostException 说明什么？

优先说明域名没有成功解析成目标 IP。

此时对目标业务服务的 TCP 三次握手通常还没有开始。

---

### 6. 数据库连接失败应该怎么查？

第一步不是机械地查所有网络层，而是先看完整 Java 异常。

根据异常决定：

```text
UnknownHost
→ DNS

Timeout
→ 网络 / 路由 / ACL / 防火墙 / 响应

Refused
→ 端口监听 / 服务

Access denied
→ 认证 / 权限
```

---

### 7. SNMP Timeout 怎么查？

先判断影响范围，再沿着：

```text
配置
→ DNS
→ IP
→ 路由
→ 二层
→ ACL/防火墙
→ UDP161
→ SNMP参数
→ OID/模板
→ 返回链路
```

逐步验证。

---

## 总结

今天真正需要掌握的是：

> 域名只是入口，真正通信之前必须先得到目标 IP。DNS 一旦异常，后面的业务访问链路可能根本无法开始，或者被带到错误的目标。

同时进一步明确了现场排障方法：

> 不要看到错误就机械地从网络第一层全部检查。先利用监控平台日志、Java异常和故障范围定位大致层级，再使用 AI 辅助分析，最后通过实际命令和配置验证。

今日关键词：

- DNS
- Domain Name System
- UnknownHostException
- nslookup
- dig
- /etc/resolv.conf
- ip route get
- ip neigh
- Connection timeout
- Connection refused
- UDP 161
- SNMP timeout
- OID / MIB
- 日志驱动排障
