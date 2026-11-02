# 后端工程化每日训练 Day 61：VLAN、Access、Trunk 与二层网络隔离复盘

## 一、今天学习什么

今天继续补全网络通信链路，核心学习：

**VLAN 如何把一台物理交换机逻辑划分成多个相互隔离的二层网络，以及这种隔离为什么会最终表现成 Ping 不通、SNMP Timeout、设备 Offline。**

最近几天的链路已经逐渐串起来：

```text
Day 58
路由：
数据应该往哪里走？

↓

Day 59
ARP：
下一跳对应的 MAC 是什么？

↓

Day 60
MAC 地址表：
交换机应该从哪个端口转发？

↓

Day 61
VLAN：
这个二层帧允许在哪个逻辑网络里传播？
```

今天最重要的一句话：

> **物理上连在一起，不代表逻辑网络一定相通。**

---

## 二、VLAN 到底是什么

VLAN：

```text
Virtual LAN
Virtual Local Area Network
虚拟局域网
```

可以先理解为：

> **把一台物理交换机，逻辑上切分成多个相互隔离的二层广播域。**

例如一台交换机：

```text
port1 → VLAN 10
port2 → VLAN 10
port3 → VLAN 20
```

物理上：

```text
port1
port2
port3

都插在同一台交换机
```

但逻辑上更接近：

```text
VLAN 10

port1 ↔ port2


VLAN 20

port3
```

所以：

```text
同一台物理交换机
≠
同一个二层网络
```

这里必须注意：

VLAN 是：

```text
逻辑隔离
```

不是：

```text
把物理交换机真的切成几块
```

---

## 三、为什么同一个 VLAN 可以直接进行二层通信

假设：

```text
port1 → PC A → VLAN 10
port2 → PC B → VLAN 10
```

A 要访问 B。

如果 A 已经根据 IP 和掩码判断：

```text
B 和我在同一网段
```

那么 A 会尝试直接找到 B 的 MAC。

链路：

```text
A
↓
ARP Request
“谁是目标 IP？”
↓
ARP 广播在 VLAN 10 内传播
↓
B 收到
↓
如果 B 正好拥有这个目标 IP
B 回复自己的 MAC
↓
A 得到目标 MAC
↓
交换机根据 MAC 地址表转发
↓
完成二层通信
```

这里有两个容易说错的点。

### 1. 不是“同一个端口收到 ARP”

应该是：

```text
同一个 VLAN 内的相关端口
可以收到这个 ARP 广播
```

### 2. 不是所有收到 ARP 的设备都会回复

ARP Request 是广播。

但真正回复的是：

```text
拥有目标 IP 的设备
```

其他设备即使收到广播，也不会因为收到就必须回复。

---

## 四、为什么同网段但不同 VLAN 仍然可能完全不通

这是今天非常关键的一个场景。

Collector：

```text
192.168.10.20/24
VLAN 10
```

目标设备：

```text
192.168.10.50/24
VLAN 20
```

从 IP 看：

```text
192.168.10.20/24
192.168.10.50/24
```

都属于：

```text
192.168.10.0/24
```

Collector 会认为：

```text
目标和我同网段
```

因此不会先把数据交给默认网关，而是尝试：

```text
直接 ARP
```

例如：

```text
谁是 192.168.10.50？
```

但是：

```text
Collector 在 VLAN 10
目标设备在 VLAN 20
```

ARP 广播只能在 VLAN 10 内传播。

所以：

```text
Collector
↓
ARP Request
↓
VLAN 10
↓
× VLAN 隔离
↓
VLAN 20 中的目标设备收不到
```

最终：

```text
ARP 失败
↓
拿不到目标 MAC
↓
Ping 不通
↓
SNMP Request 到不了目标
↓
SNMP Timeout
```

这说明：

> **IP 同网段，不代表一定能通信；二层 VLAN 仍然可能把双方隔离。**

---

## 五、不同 VLAN 应该如何通信

不同 VLAN：

```text
默认不能直接二层通信
```

但不等于：

```text
永远不能通信
```

可以经过：

```text
路由器

或者

三层交换机
```

进行三层转发。

典型规划：

```text
VLAN 10
192.168.10.0/24

VLAN 20
192.168.20.0/24
```

例如：

```text
服务器 A
192.168.10.20
VLAN 10

访问

服务器 B
192.168.20.50
VLAN 20
```

A 根据掩码判断：

```text
目标不在本地网段
```

于是完整过程更接近：

```text
服务器 A
↓
查路由
↓
默认网关
↓
ARP 获取下一跳网关 MAC
↓
交换机二层转发
↓
三层网关 / 路由
↓
VLAN 20
↓
服务器 B
```

因此可以记住：

> **VLAN 负责隔离，路由负责让不同网络在策略允许的情况下互通。**

---

## 六、Access 端口是什么

普通服务器、PC、打印机等终端通常只需要属于一个 VLAN。

例如：

```text
服务器
↓
交换机 port1
↓
Access VLAN 10
↓
VLAN 10
```

服务器正常发送普通以太网帧即可。

它不需要每次都主动声明：

```text
“我是 VLAN 10”
```

交换机知道：

```text
port1 是 Access VLAN 10
```

因此可以把从这个端口进入的普通终端流量归到 VLAN 10。

现阶段可以先记：

```text
Access
→ 通常承载一个 VLAN
→ 常用于服务器、PC、打印机等普通终端接入口
```

---

## 七、Trunk 端口是什么

假设：

```text
交换机 A
有 VLAN 10 / 20 / 30

交换机 B
也有 VLAN 10 / 20 / 30
```

如果每个 VLAN 单独拉一根线：

```text
VLAN 10 → 一根线
VLAN 20 → 一根线
VLAN 30 → 一根线
```

非常浪费端口和线路。

所以交换机之间通常使用：

```text
Trunk
```

例如：

```text
交换机 A
     │
     │ Trunk
     │ VLAN 10 / 20 / 30
     │
交换机 B
```

一条链路可以同时承载多个 VLAN。

为了让对端知道：

```text
这个帧属于 VLAN 10？
还是 VLAN 20？
还是 VLAN 30？
```

Trunk 链路上的帧通常会使用：

```text
802.1Q Tag
```

现阶段最重要的区别：

```text
Access
→ 通常一个 VLAN

Trunk
→ 可以承载多个 VLAN
```

可以进一步记成：

> **Access 用于把终端接入某一个 VLAN；Trunk 用于让多个 VLAN 在网络设备之间一起通过。**

---

## 八、为什么 Trunk UP 不等于所有 VLAN 都正常

场景：

```text
交换机 A
   │
   │ Trunk
   │
交换机 B
```

状态：

```text
Trunk = UP

VLAN 10 设备正常
VLAN 20 设备正常
VLAN 30 设备全部 Offline
```

不能因为：

```text
Trunk = UP
```

就判断：

```text
VLAN 30 一定正常
```

因为可能存在：

```text
Trunk UP

allowed VLAN:
VLAN 10
VLAN 20

漏了：
VLAN 30
```

于是：

```text
VLAN 10 → 正常
VLAN 20 → 正常
VLAN 30 → 完全不通
```

所以 Trunk UP 更多说明：

```text
这条接口 / 链路本身处于 UP 状态
```

不能证明：

```text
所有 VLAN 都已经正确配置并被允许通过
```

---

## 九、看到整组设备异常，要先找共同故障域

监控平台突然出现：

```text
VLAN 100
40 台设备
全部 SNMP Timeout
```

而：

```text
VLAN 10 正常
VLAN 20 正常
VLAN 30 正常
```

这时不应该第一反应：

```text
40 台设备是不是同时各自坏了？
```

应该先看：

```text
40 台设备的共同点是什么？
```

答案非常明显：

```text
共同 VLAN 100
```

因此优先检查：

```text
VLAN 100 是否存在
↓
相关 Access 端口 VLAN 是否正确
↓
上联 Trunk 是否允许 VLAN 100
↓
VLAN 100 的 Vlanif / 网关是否正常
↓
三层路由是否正常
↓
ACL / 防火墙策略是否允许
↓
最后才逐台检查设备
```

这种思路叫：

```text
寻找共同故障域
```

对于监控系统非常重要。

因为监控平台同时掌握大量设备状态，异常分布本身就是排障线索。

---

## 十、网卡 UP、交换机端口 UP，为什么仍然不能证明网络正常

场景：

服务器：

```text
网卡 UP
IP 正确
```

交换机：

```text
端口 UP
```

但是：

```text
服务器访问不了目标
```

这时不能说：

```text
网卡和交换机端口都正常
所以网络一定没问题
```

因为这只能证明：

```text
最基础的物理链路大致存在
```

后面仍然需要逐层证明。

### 二层

```text
VLAN 是否正确
ARP 是否正常
下一跳 MAC 是否能获取
交换机 MAC 地址表是否学到正确端口
```

### 三层

```text
IP / 掩码是否正确
默认网关是否正确
路由是否存在
下一跳是否正确
```

### 网络策略

```text
ACL 是否允许
防火墙是否允许
```

### 传输层

```text
TCP / UDP 目标端口是否可达
```

### 应用层

```text
SNMP Agent 是否正常
Java 服务是否监听
数据库服务是否正常
认证配置是否正确
```

所以排障不能只看：

```text
UP / DOWN
```

而要证明整条路径。

---

## 十一、Linux 上怎么检查这条链路

出现 VLAN 20 所有设备都 SNMP Timeout 时，可以先在 Collector 所在 Linux 检查。

### 1. 看 IP、掩码、网卡

```bash
ip addr
```

确认：

```text
Collector IP
子网掩码
网卡状态
```

### 2. 看路由

```bash
ip route
```

重点确认：

```text
去目标网段走哪张网卡？
下一跳是谁？
默认网关是谁？
```

### 3. 测试目标可达性

```bash
ping <目标IP>
```

Ping 失败不能直接说明一定是 VLAN 问题，但可以证明网络路径还没有被完全验证。

### 4. 看邻居表

```bash
ip neigh
```

这里需要特别注意。

如果目标和 Collector 同网段：

```text
Collector 可能直接 ARP 目标设备
```

如果目标在另一个网段：

```text
Collector 不会直接 ARP 远端目标 MAC
```

它通常会先：

```text
ARP 下一跳网关 MAC
```

因此跨网段排障时：

> **ip neigh 更重要的是看下一跳邻居是否正常，而不是机械寻找远端目标设备 MAC。**

---

## 十二、Codex、Linux、网络设备应该怎么分工

真实故障：

```text
Collector 可以正常采集 VLAN 10

但是

VLAN 20 中所有设备
全部 SNMP Timeout
```

不要把所有问题都交给 Codex，也不要一开始只盯 Java 代码。

### 1. Codex：还原应用采集链路

让 Codex 查：

```text
设备目标 IP 从哪里读取？
↓
采集任务在哪里生成 / 调度？
↓
SNMP 请求在哪里真正发出？
↓
timeout 怎么配置？
↓
失败之后如何判断 Offline？
↓
状态如何写数据库 / 返回前端？
```

目标不是让 Codex泛泛地：

```text
“帮我看看代码有没有问题”
```

而是：

```text
证明应用到底做了什么
```

而且：

```text
VLAN 10 正常
VLAN 20 全部失败
```

本身就说明：

```text
同一套 SNMP 代码至少不是整体完全不可用
```

更应该继续检查 VLAN 20 的公共网络路径。

---

### 2. Linux：证明 Collector 的网络路径

检查：

```text
ip addr
↓
ip route
↓
ping
↓
ip neigh
```

需要回答：

```text
Collector 自己地址是否正确？
↓
去 VLAN 20 的路由是否正确？
↓
下一跳是谁？
↓
下一跳 MAC 是否能正常解析？
↓
基础网络是否可达？
```

---

### 3. 网络设备：检查 VLAN 的公共网络路径

重点看：

```text
VLAN 20 是否存在
↓
Collector 接口属于哪个 VLAN
↓
目标设备接口属于哪个 VLAN
↓
Trunk 是否允许 VLAN 20
↓
VLAN 20 的 Vlanif / 网关是否正常
↓
相关路由是否存在
↓
ACL / 防火墙策略是否允许
```

这里也要避免两个不准确的说法。

不是：

```text
“交换机是不是设置了 VLAN20 timeout”
```

更准确是：

```text
SNMP Timeout 是最终表现
真正故障可能发生在 VLAN / Trunk / 路由 / ACL 等更前面的层
```

也不是：

```text
“防火墙把 VLAN 当病毒杀了”
```

而是：

```text
防火墙 / ACL 策略可能阻断相关网络流量
```

---

## 十三、完整监控采集链路

今天最终应该能够还原出这条链：

```text
监控平台设备配置
↓
Collector 获取目标 IP
↓
采集任务开始执行
↓
Linux 根据目标 IP 查路由
↓
判断下一跳
↓
ARP / ip neigh 获取下一跳 MAC
↓
交换机根据 MAC 地址表进行二层转发
↓
VLAN 限定二层帧传播范围
↓
Trunk 承载相关 VLAN
↓
三层网关 / 路由
↓
ACL / 防火墙
↓
目标设备
↓
UDP 161
↓
SNMP Agent
↓
SNMP Response
↓
Collector
↓
timeout / online / offline 状态判断
↓
数据库
↓
告警 / 前端
```

以后看到：

```text
SNMP Timeout
```

不能直接把它理解为：

```text
SNMP 配置错误
```

它只是：

```text
最终没有在规定时间内收到期望响应
```

真正的故障可能发生在整条链路中的任何位置。

---

## 十四、今天纠正的几个错误理解

### 1. VLAN 不是物理切割

错误：

```text
VLAN 把交换机物理分割了
```

正确：

```text
VLAN 把一台物理交换机
逻辑划分成多个二层网络
```

---

### 2. ARP 广播不是所有设备都必须回复

正确逻辑：

```text
ARP Request 在当前广播域传播
↓
拥有目标 IP 的设备回复自己的 MAC
```

---

### 3. 不同 VLAN 不代表永远不能通信

正确：

```text
不能直接二层通信
```

但可以通过：

```text
路由器 / 三层交换机
```

完成三层通信。

---

### 4. Trunk UP 不代表所有 VLAN 正常

```text
Trunk UP
```

可能同时存在：

```text
VLAN 10 allowed
VLAN 20 allowed
VLAN 30 missing
```

所以需要检查 allowed VLAN。

---

### 5. 没有“VLAN Timeout”这个排障概念

更准确的思路是：

```text
应用看到 SNMP Timeout

但网络侧可能是：
VLAN 不通
Trunk 未放行
路由错误
ACL 阻断
```

Timeout 是结果，不一定是故障发生的层。

---

## 十五、今日核心总结

今天记住四个模型。

### 模型一：VLAN

```text
一台物理交换机
↓
逻辑划分
↓
多个相互隔离的二层广播域
```

### 模型二：Access / Trunk

```text
Access
→ 通常一个 VLAN
→ 终端接入

Trunk
→ 多个 VLAN
→ 网络设备之间传递
```

### 模型三：整组异常先找公共故障域

```text
VLAN 100
40 台设备全部 Offline

↓

先查共同 VLAN
共同 Trunk
共同网关
共同路由
共同策略

而不是逐台检查40台设备
```

### 模型四：分层证明网络正常

```text
网卡 UP
交换机端口 UP

≠

整个网络一定正常
```

还需要继续证明：

```text
VLAN
↓
ARP
↓
MAC
↓
路由
↓
网关
↓
ACL / 防火墙
↓
TCP / UDP
↓
应用服务
```

今天真正补上的不是两个命令，也不是 `Access`、`Trunk` 两个单词，而是一个更重要的工程判断：

> **看到网络或监控故障时，要从异常分布和完整链路判断故障域；物理链路正常，不代表逻辑网络正常。**
