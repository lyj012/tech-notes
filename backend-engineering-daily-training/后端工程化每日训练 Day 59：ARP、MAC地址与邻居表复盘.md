# 后端工程化每日训练 Day 59：ARP、MAC 地址与邻居表复盘

## 一、今天学习什么

今天学习网络通信链路中的一个关键环节：

**Linux 已经通过路由知道数据应该往哪里走之后，如何找到下一跳设备对应的 MAC 地址，并真正通过网卡发送数据。**

Day 58 解决的问题：

```text
目标 IP 是谁
↓
目标是否在本地网络
↓
Linux 查询路由表
↓
确定出口网卡和下一跳
```

Day 59 继续向下：

```text
下一跳 IP 已经知道
↓
但是网卡发送以太网帧需要 MAC 地址
↓
MAC 地址从哪里获取？
↓
ARP
```

核心理解：

> IP 负责确定最终目标，MAC 负责当前这一跳的数据交付。

---

## 二、IP 地址和 MAC 地址的区别

服务器网卡：

```text
IP:
192.168.10.20

MAC:
00:11:22:33:44:55
```

IP 地址：

- 网络层地址
- 用于判断目标在哪里
- 用于路由选择

MAC 地址：

- 数据链路层地址
- 用于局域网内实际发送以太网帧

一次通信通常同时包含：

```text
IP 数据包
+
以太网帧
```

例如：

```text
目标 IP:
192.168.10.50

目标 MAC:
？？？
```

Linux 知道目标 IP，但不知道目标网卡地址，因此需要 ARP。

---

## 三、ARP 是什么

ARP（Address Resolution Protocol）：

> 根据 IPv4 地址查询对应 MAC 地址。

例如：

服务器：

```text
192.168.10.20
```

访问：

```text
192.168.10.50
```

如果不知道 MAC：

```text
谁是 192.168.10.50？
请告诉我你的 MAC 地址。
```

这个请求通过广播发送。

目标设备收到后回复：

```text
我是 192.168.10.50
我的 MAC 是 AA:BB:CC:DD:EE:FF
```

最终得到：

```text
192.168.10.50
↓
AA:BB:CC:DD:EE:FF
```

---

## 四、邻居表（Neighbor Table）

Linux 不会每次通信都重新 ARP。

它会缓存：

```text
IP
↓
MAC
```

查看：

```bash
ip neigh
```

例如：

```text
192.168.10.50 dev ens33 lladdr AA:BB:CC REACHABLE
```

表示：

```text
访问 192.168.10.50
通过 ens33 网卡
对应 MAC 为 AA:BB:CC
```

常见状态：

- REACHABLE：确认可达
- STALE：存在缓存，但需要重新确认
- FAILED：邻居解析失败

---

## 五、同网段和跨网段的区别

### 1. 同网段通信

服务器：

```text
192.168.10.20/24
```

访问：

```text
192.168.10.50
```

流程：

```text
Linux 查路由
↓
发现目标同网段
↓
ARP 查询目标 MAC
↓
发送以太网帧
```

即：

```text
同网段：
ARP 找目标 MAC
```

---

### 2. 跨网段通信

服务器：

```text
192.168.10.20
```

访问：

```text
10.20.30.40
```

默认网关：

```text
192.168.10.1
```

流程：

```text
Linux 查询路由
↓
发现目标不在本地网络
↓
下一跳为 192.168.10.1
↓
ARP 查询网关 MAC
↓
数据交给网关
↓
网关继续路由
```

注意：

不会 ARP：

```text
10.20.30.40 的 MAC
```

因为它不在当前二层网络。

记忆：

```text
同网段：目标 MAC

跨网段：下一跳 MAC
```

---

## 六、IP 和 MAC 在转发过程中的变化

跨网段通信：

```text
服务器
↓
路由器
↓
目标服务器
```

IP 通常保持：

```text
源 IP
目标 IP
```

MAC 会变化：

第一跳：

```text
源 MAC：服务器 MAC
目标 MAC：网关 MAC
```

下一跳：

```text
源 MAC：当前路由器 MAC
目标 MAC：下一跳 MAC
```

理解：

```text
IP = 最终去哪
MAC = 当前这一站交给谁
```

---

## 七、ARP 失败意味着什么

例如：

```text
192.168.10.50 FAILED
```

说明：

```text
Linux 知道目标 IP
↓
但是无法获得 MAC
↓
无法完成二层封装
↓
数据无法正常发送
```

表面可能看到：

```text
Ping Timeout
TCP Timeout
SNMP Timeout
```

但根因可能在：

```text
二层网络
VLAN
交换机端口
目标设备状态
IP冲突
```

而不是 Java 或 SNMP。

---

## 八、监控系统中的实际应用

### 场景一：同网段设备 SNMP Timeout

Collector：

```text
192.168.1.20
```

交换机：

```text
192.168.1.50
```

现象：

```text
设备 Offline
SNMP Timeout
```

排查：

```bash
ip addr
```

确认 Collector 地址。

```bash
ip route
```

确认目标属于本地网络。

```bash
ip neigh
```

如果：

```text
192.168.1.50 FAILED
```

优先检查：

- 交换机是否在线
- VLAN 是否正确
- 交换机端口
- 二层链路
- IP 是否冲突

不要先查：

- OID
- Community
- SNMP模板

因为二层通信还没有建立。

---

### 场景二：多个远端网段全部失败

Collector：

```text
192.168.1.20
```

远端：

```text
10.20.x.x
10.30.x.x
```

路由：

```text
default via 192.168.1.1
```

邻居表：

```text
192.168.1.1 FAILED
```

说明：

```text
所有跨网段流量
↓
都依赖默认网关
↓
网关 MAC 无法解析
↓
所有远端网络失败
```

排查公共故障点，而不是逐个检查设备。

---

## 九、Java 服务排障中的应用

Java 报：

```text
Connection timeout
```

不一定是 Java 代码问题。

完整链路：

```text
Java 发起连接
↓
Linux 网络栈
↓
ip route 判断路径
↓
ARP 获取下一跳 MAC
↓
网卡发送
↓
交换网络
↓
目标机器
↓
目标服务响应
```

可能失败的位置：

- Java 配置错误
- IP 错误
- 路由错误
- ARP失败
- VLAN异常
- 防火墙拦截
- 服务未启动

工程排障不能看到 timeout 就直接修改代码。

---

## 十、Codex + Linux 联合排查思路

### Codex 查代码

关注：

### 1. 设备 IP 来源

可能来自：

- 配置文件
- 数据库
- 缓存
- 前端录入

搜索：

```text
deviceIp
targetIp
host
```

---

### 2. SNMP 请求发送位置

搜索：

```text
snmp
161
send
collect
```

确认：

- 请求目标
- 请求流程
- 参数来源

---

### 3. timeout 配置

确认：

- 超时时间
- 重试次数
- 失败处理

---

### 4. Offline 判断逻辑

确认：

- 单次失败是否离线
- 连续失败次数
- 状态更新逻辑

---

### Linux 查环境

顺序：

```bash
ip addr
```

确认本机网络。

```bash
ip route
```

确认路径和下一跳。

```bash
ip neigh
```

确认 IP 到 MAC 映射。

---

## 十一、今日核心总结

记住三个模型：

### 模型一：

```text
IP：我要去哪里
路由：下一步走哪里
MAC：这一跳交给谁
```

### 模型二：

```text
同网段：
目标 IP → 目标 MAC

跨网段：
目标 IP → 下一跳网关 → 网关 MAC
```

### 模型三：

看到：

```text
Timeout
```

不要直接判断应用问题。

应该逐层排查：

```text
应用
↓
TCP/UDP
↓
IP/路由
↓
ARP/MAC
↓
交换网络
↓
目标服务
```

今天补全了从 Java 请求到底层网卡发送之间缺失的一层：

**Linux 如何通过 ARP 和邻居表完成二层交付。**
