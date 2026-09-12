# 后端工程化每日训练 Day 64：TCP 三次握手、连接状态与监控端口排障

## 一、今天学习什么

今天继续补全监控系统网络排障链路。

之前学习：

- Day 60：交换机 MAC 地址表与二层转发；
- Day 61：VLAN、Access、Trunk 与二层隔离；
- Day 63：ACL、防火墙访问控制与 Timeout 排障。

今天继续向上传输层学习：

**TCP 三次握手、TCP连接状态，以及 Java 服务、监控采集中的端口异常排查。**

实际工作中经常遇到：

```
服务器 Ping 正常
但是：
8082访问失败
3306连接失败
监控显示 Offline
```

此时问题可能已经不是网络层，而是 TCP 连接阶段。

---

# 二、TCP基础

## 1. TCP是什么

TCP（Transmission Control Protocol）是一种：

- 面向连接；
- 可靠传输；
- 保证数据顺序。

Java服务访问流程：

```
Collector
   |
   | TCP connect
   ↓
Java服务8082
```

---

# 三、TCP三次握手

## 1. 握手流程

客户端发送：

```
SYN
```

表示：我要建立连接。

服务端回复：

```
SYN + ACK
```

表示：收到请求，可以建立连接。

客户端回复：

```
ACK
```

表示：确认收到响应。

最终进入：

```
ESTABLISHED
```

---

## 2. 为什么需要三次握手

核心目的：

确认双方具备双向通信能力。

客户端确认：

- 可以发送；
- 可以收到服务器响应。

服务端确认：

- 可以收到请求；
- 可以返回响应。

---

# 四、Ping通为什么不能证明8082可用

Ping使用：

```
ICMP协议
```

只能证明：

```
IP层基本可达
```

不能证明：

```
TCP 8082连接成功
```

因为TCP需要：

```
客户端
 ↓
SYN
 ↓
服务器端口监听
 ↓
三次握手
 ↓
应用响应
```

可能出现：

```
Ping成功
8082失败
```

原因：

- Java服务未启动；
- 端口没有监听；
- 防火墙阻断；
- ACL限制。

---

# 五、TCP连接状态

查看：

```bash
ss -ant
```

## LISTEN

表示服务正在监听端口。

例如：

```
0.0.0.0:8082
```

---

## ESTABLISHED

表示TCP连接已经建立。

---

## TIME_WAIT

表示连接关闭后，TCP仍保留状态。

作用：

防止旧连接的数据影响新的连接。

大量TIME_WAIT不一定故障，需要结合：

- 请求量；
- 短连接数量；
- 端口资源。

---

## CLOSE_WAIT

表示：

> 对方已经关闭连接，但是本地程序没有关闭socket。

流程：

```
客户端关闭
 ↓
Java收到FIN
 ↓
程序没有close()
 ↓
CLOSE_WAIT增加
```

大量CLOSE_WAIT：

优先检查：

- Java代码资源释放；
- HTTP客户端关闭；
- 数据库连接释放；
- 文件流关闭。

---

# 六、Connection refused 与 Connection timeout

## Connection refused

含义：

请求已经到达服务器，但是目标端口没有服务监听。

流程：

```
客户端 SYN
 ↓
服务器检查8082
 ↓
没有监听
 ↓
返回RST
```

常见原因：

- Java服务未启动；
- 端口配置错误；
- 应用启动失败。

---

## Connection timeout

含义：

请求没有收到响应。

可能原因：

- 防火墙丢弃；
- ACL限制；
- 路由异常；
- VPN问题；
- 服务无响应。

区别：

|错误|含义|
|-|-|
|refused|到了，但是没人监听|
|timeout|没有收到有效响应|

---

# 七、Java进程存在但是端口不存在

现象：

```
ps看到Java进程

ss看不到8082监听
```

可能原因：

## 1. Spring Boot启动失败

例如：

- 数据库连接失败；
- Bean创建失败；
- 配置错误。

结果：

```
JVM存在
应用没有启动完成
```

## 2. 端口配置变化

例如：

8082改成8083。

## 3. service启动错误

启动了错误jar包。

排查：

```bash
ps -ef | grep java
ss -lntp | grep 8082
journalctl -u 服务名
```

---

# 八、监控Offline排障流程

场景：

```
hope-web-app Offline

服务器在线
Java进程存在
```

## 1. 监控采集层

查看：

- Collector日志；
- timeout信息；
- 配置参数；
- 判断逻辑。

确认为什么判定Offline。

---

## 2. TCP层

测试：

```bash
nc -vz IP 8082
```

判断：

- refused；
- timeout；
- 成功。

---

## 3. 应用层

检查：

```bash
ss -lntp | grep 8082
curl localhost:8082
```

确认Java服务状态。

---

## 4. 网络层

如果TCP无法建立，检查：

```bash
ip addr
ip route
ip neigh
```

以及：

- VLAN；
- ACL；
- 防火墙；
- VPN；
- 路由。

必要时：

```bash
tcpdump -i eth0 port 8082
```

---

# 九、Java服务健康检测设计

完整健康检测应该包含：

```
Java健康检测

├── 进程状态
├── 端口监听
├── TCP连接状态
├── HTTP健康接口
├── JVM指标
│   ├── Heap
│   ├── GC
│   └── Thread
└── 外部依赖
    ├── MySQL
    ├── Redis
    └── MQ
```

---

# 十、今日总结

生产排障不能看到Offline直接认为代码问题。

应该按照链路分析：

```
监控判断
 ↓
TCP连接
 ↓
端口监听
 ↓
应用状态
 ↓
网络链路
```

核心认知：

```
Ping通
≠
端口可用

端口可用
≠
应用健康

应用健康
≠
监控判断正确
```

今日关键词：

- TCP三次握手
- SYN
- ACK
- LISTEN
- ESTABLISHED
- TIME_WAIT
- CLOSE_WAIT
- Connection refused
- Connection timeout
- Java服务健康检测
- 监控Offline排障
