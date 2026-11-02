# 后端工程化每日训练 Day 63：ACL、防火墙访问控制与监控 Timeout 排障

## 一、今天学习什么

今天继续补全监控系统网络排障链路，核心学习：

**ACL / 防火墙访问控制：为什么网络明明能 Ping 通，但是 SNMP、Java 端口、数据库端口仍然可能访问失败。**

这是运维监控场景中非常高频的问题。

今天建立一个核心认知：

> 路由决定“有没有路”，ACL / 防火墙决定“有路以后允不允许你走”。

---

## 二、核心原理

## 1. Ping 通不代表服务可达

Ping 使用：

```text
ICMP
```

例如：

```text
Collector
   |
 ICMP
   |
设备
```

Ping 成功只能说明：

```text
ICMP通信正常
```

不能证明：

```text
TCP 8082
TCP 2883
UDP 161
UDP 162
```

一定正常。

例如：

```text
ICMP       ALLOW
UDP 161    DENY
```

结果：

```text
Ping成功
SNMP Timeout
```

完全合理。

---

## 2. 不同协议需要分别排查

常见协议：

| 功能 | 协议 |
|---|---|
| Ping | ICMP |
| SNMP采集 | UDP 161 |
| SNMP Trap | UDP 162 |
| HTTP | TCP 80 |
| HTTPS | TCP 443 |
| Java服务 | TCP业务端口 |
| 数据库 | TCP数据库端口 |

网络可达不是简单的：

```text
A 能不能访问 B
```

而是：

```text
谁
通过什么协议
访问谁
访问什么端口
是否允许
```

---

## 三、ACL是什么

ACL 可以理解为：

```text
网络设备上的流量过滤规则
```

匹配信息包括：

```text
源IP
目标IP
协议
源端口
目标端口
```

例如：

允许：

```text
192.168.10.20
访问
10.20.1.50
UDP 161
```

拒绝：

```text
192.168.10.0/24
访问
10.20.1.50
TCP 22
```

---

## 四、ACL规则顺序

很多ACL采用：

```text
从上到下匹配
匹配后停止
```

例如：

```text
规则1:
DENY
192.168.1.20 -> 10.10.1.50 UDP 161

规则2:
ALLOW
192.168.1.0/24 -> 10.10.1.0/24 UDP 161
```

请求：

```text
192.168.1.20
访问
10.10.1.50
UDP161
```

首先命中规则1：

```text
DENY
```

后面的ALLOW不会执行。

所以：

> 有允许规则，不代表一定允许，需要看实际命中的规则。

---

## 五、为什么 Timeout 很难定位

例如 SNMP：

```text
Collector
   |
 UDP161
   |
交换机
```

可能出现：

### 情况1：请求没有到设备

```text
Collector
  X
设备
```

原因：

- 路由
- VLAN
- ACL
- 防火墙

### 情况2：请求到了，响应回不来

```text
Collector
    X
设备
```

原因：

- 回程ACL
- 防火墙策略
- 路由不对称

### 情况3：请求响应都正常

问题可能：

- Java代码
- SNMP库
- 数据解析
- 状态判断

最终应用层都可能看到：

```text
SNMP Timeout
```

---

## 六、DROP 和 REJECT

DROP：

```text
收到数据
↓
直接丢弃
↓
不给回应
```

发送方表现：

```text
等待
↓
timeout
```

REJECT：

```text
收到数据
↓
返回拒绝信息
```

通常更快看到失败。

---

## 七、监控系统中的实际场景

## 场景1：SNMP采集失败

链路：

```text
Collector
↓
UDP161
↓
交换机
```

排查：

1. 确认目标IP
2. 确认UDP161
3. 检查ACL、防火墙
4. 检查SNMP Agent
5. 检查Community和版本

---

## 场景2：Java服务访问失败

现象：

```text
Ping服务器正常
8082 Timeout
```

如果：

```bash
ss -lntp
```

看到：

```text
8082 LISTEN
```

说明Java服务已经启动。

下一步检查：

```text
监听地址
服务器防火墙
ACL
VPN策略
网络链路
```

---

## 场景3：Collector迁移导致全部设备Offline

旧Collector：

```text
192.168.1.20
```

新Collector：

```text
192.168.1.30
```

代码、模板、OID、Community全部没有变化。

但是：

```text
所有设备SNMP Timeout
```

优先考虑：

```text
ACL白名单仍允许旧IP
```

因为故障点是公共变化：

```text
Collector IP变化
```

---

## 八、联合排查流程

真实故障：

```text
Collector可以Ping通防火墙
但是SNMP持续Timeout
```

### 1. Codex检查

关注代码逻辑：

```text
目标IP来源
SNMP版本
Community来源
UDP161配置
timeout/retry
Offline判断逻辑
```

Codex负责回答：

> 程序想做什么。

---

### 2. Linux检查

检查：

```bash
ip addr

ip route

ip route get 目标IP
```

确认：

```text
IP
出口网卡
路由
下一跳
```

抓包：

```bash
tcpdump -i eth0 udp port 161
```

确认：

```text
请求有没有发出
响应有没有回来
```

---

### 3. 网络设备检查

重点：

```text
ACL
防火墙策略
源IP白名单
UDP161是否允许
规则命中日志
回程策略
```

---

### 4. 监控平台检查

关注：

```text
是否全部设备失败
最后成功时间
失败日志
是否刚迁移Collector
是否集中在一个网段
```

---

## 九、今天建立的排障模型

以后遇到：

```text
Ping正常
服务Timeout
```

不要直接判断：

```text
网络没问题
```

正确思路：

```text
现象
↓
确认协议
↓
确认端口
↓
确认路由
↓
确认ACL/防火墙
↓
确认服务
↓
确认业务配置
```

---

## 十、训练题复盘

### 1. Ping成功为什么8082可能访问失败？

因为Ping使用ICMP，8082使用TCP，两者受不同规则控制。

---

### 2. Ping正常，SNMP UDP161 Timeout排查什么？

优先：

```text
目标配置
ACL/防火墙
SNMP Agent
抓包分析
```

---

### 3. 为什么DENY在前ALLOW在后仍然失败？

因为ACL按顺序匹配，先命中的DENY直接结束。

---

### 4. ss看到8082 LISTEN但访问Timeout怎么办？

说明服务已监听，需要继续排查：

```text
监听地址
防火墙
ACL
VPN
网络策略
```

---

### 5. SNMP请求到达但响应被拦为什么Timeout？

因为Collector最终没有收到响应，UDP没有可靠连接机制，只能等待超时。

---

### 6. Collector换IP后全部Timeout为什么像ACL问题？

因为所有设备共同失败，而唯一变化是Collector源IP。

---

## 总结

今天真正需要掌握的不是ACL命令，而是：

> Ping通只能证明一种通信方式成功，判断监控服务是否真正可达，必须继续验证具体协议、端口以及中间访问控制策略。
