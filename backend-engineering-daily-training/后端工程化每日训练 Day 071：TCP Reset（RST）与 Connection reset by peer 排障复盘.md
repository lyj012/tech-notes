# 后端工程化每日训练 Day 71：TCP Reset（RST）与 Connection reset by peer 排障复盘

## 一、今天学习什么

今天学习 TCP 故障中非常常见、也非常容易和 `timeout`、`refused` 混淆的一类问题：

**TCP Reset（RST）以及 `Connection reset by peer` 到底意味着什么，怎么判断是谁把连接断掉了。**

线上常见三类错误：

```text
Connection refused
Connection timed out
Connection reset by peer
```

今天最核心的一句话是：

> **Connection reset by peer 表示当前 TCP 连接收到了 RST，连接被强制终止。排障重点不是简单判断“网络通不通”，而是继续确认：谁发了 RST，为什么发。**

---

## 二、RST 是什么

正常 TCP 建连：

```text
客户端                      服务端

SYN        ───────────────→
           ←────────────── SYN+ACK
ACK        ───────────────→

        TCP 连接建立
```

正常关闭通常通过 FIN 完成，而 RST（Reset）可以简单理解为：

> **这条 TCP 连接不要了，立即终止。**

例如：

```text
Java 客户端                  服务端

请求 ─────────────────────→

     ←──────────────────── RST

Connection reset by peer
```

因此可以先记成：

```text
timeout：等不到
reset：被掐断
```

---

## 三、reset 不能直接等于“网络不通”

Java 出现：

```text
java.net.SocketException:
Connection reset by peer
```

不能直接判断“网络不通”。更准确地说，是当前 TCP 连接收到了 RST，因此连接被异常终止。

同时也不能仅凭这一条 Java 异常断言：

```text
一定是对端 Java 应用主动发了 RST
```

RST 可能与以下位置有关：

```text
服务端应用
服务端操作系统 TCP 栈
服务进程异常退出
防火墙
NAT
负载均衡
连接状态失效
协议或连接状态异常
```

所以：

```text
Connection reset by peer
↓
只是现象
↓
还要定位 RST 来源和真正根因
```

---

## 四、Connection refused 和 reset 的区别

假设访问：

```text
10.0.0.20:8082
```

### 场景 A：Connection refused

典型过程：

```text
客户端 SYN
↓
10.0.0.20 收到
↓
8082 没有进程监听
↓
目标主机返回 RST
↓
Connection refused
```

优先排查：

```bash
ss -lntp | grep 8082
```

然后继续确认服务状态、端口配置、启动日志；某些防火墙主动 `REJECT` 也可能产生 refused。

### 场景 B：Connection reset by peer

如果已经明确：

```text
TCP 已建立
↓
请求已经发送
↓
Connection reset by peer
```

这时再优先从 `ip addr`、`ip route`、ARP 从头检查，价值已经比较低。

因为 TCP 已经成功建立，本身已经证明：

```text
IP 基本可用
路由至少能把流量送到对端
二层邻居解析基本完成
TCP 三次握手成功
```

此时更有价值的是：

```text
抓包确认 RST
↓
确认 RST 来源方向
↓
再查对应应用、OS 或中间设备
```

---

## 五、为什么继续反复 ping 价值很低

场景：

```text
Collector 能 ping 通目标
TCP 连接也能建立
采集过程中经常 reset
```

这时继续反复 `ping` 信息增量很低：

```text
ping 正常
→ 只能证明 ICMP 基本可达

TCP 能建立
→ 已经证明三次握手成功

采集过程中 reset
→ 故障发生在 TCP 建立以后
```

下一步真正想获得的关键证据应该是：

> **RST 是从哪个方向发出的。**

---

## 六、tcpdump：确认 RST 来源

只观察目标主机：

```bash
tcpdump -nn host 10.0.0.20
```

只观察 8082：

```bash
tcpdump -nn host 10.0.0.20 and port 8082
```

假设看到：

```text
10.0.0.20 > 10.0.0.10: Flags [R.]
```

可以读成：

```text
10.0.0.20
↓
向 10.0.0.10
↓
发送了带 RST 标志的 TCP 包
```

常见标志：

```text
R = RST
. = ACK
```

`Flags [R.]` 通常表示 RST + ACK。

---

## 七、抓到 RST 能证明什么，不能证明什么

假设抓包看到：

```text
192.168.1.50 → 192.168.1.20  Flags [R.]
```

其中：

```text
192.168.1.20 = Collector
192.168.1.50 = 目标服务
```

能够证明：

> **在当前抓包位置确实看到了一个源地址为 .50、目标为 .20 的 TCP RST 包，Collector 确实收到了来自 .50 方向的 reset。**

但是不能仅凭这一包就证明：

```text
一定是 .50 上面的 Java 代码主动发了 RST
一定是服务崩溃
一定是 OOM
一定是代码 Bug
一定是网络设备坏了
```

现在只知道“发生了什么”和“RST 从哪个方向来”，还不知道“为什么”。

下一步要结合：

```text
应用日志
systemd 日志
dmesg
服务重启记录
防火墙 / NAT / LB 状态
```

继续确认。

> **tcpdump 先告诉我“RST 从哪边来”，日志再告诉我“为什么会来”。**

---

## 八、OOM、reset、systemd restart 的完整故障链

场景：

```text
Collector
↓
Connection reset by peer
```

目标 Java 服务同时出现：

```text
journalctl：服务刚刚被 systemd 重启
dmesg：OOM Killer 杀掉了 Java 进程
```

这里最重要的是因果顺序。

不是：

```text
systemd 先重启
↓
导致内存不足
↓
OOM Killer 杀进程
```

更合理的是：

```text
Java 服务运行
↓
服务器内存压力过大
↓
内核触发 OOM Killer
↓
Java 进程被杀
↓
原有 TCP 连接异常失效
↓
客户端出现 reset / 连接异常
↓
systemd 发现进程退出
↓
根据 Restart 策略重新拉起 Java 服务
```

因此：

```text
OOM
→ 根因方向

systemd restart
→ 进程死亡后的恢复动作
```

如果时间线还能对齐：

```text
14:30:01  TCP RST
14:30:01  OOM Killer killed java
14:30:02  systemd restart app.service
```

证据链就会非常强。

---

## 九、固定 300 秒后 reset：idle timeout

场景：

```text
某个长连接
↓
总是在空闲约 300 秒后 reset

两端应用没有重启
CPU、内存正常
链路中存在防火墙和 NAT
```

第一优先假设应该是：

> **防火墙或 NAT 的 TCP 会话状态存在约 300 秒的 idle timeout。**

可能过程：

```text
TCP 长连接建立
↓
300 秒没有数据
↓
中间设备认为连接长时间空闲
↓
删除连接跟踪 / NAT 状态
↓
客户端继续使用旧 Socket
↓
连接状态异常
↓
出现 RST / 连接异常
```

这里被清理的是 TCP 会话状态，不是 Java 进程。

验证顺序：

```text
1. tcpdump 确认 RST 时间和方向
2. 查防火墙 / NAT 的 TCP idle timeout 配置和日志
3. 做 keepalive 对照，例如每 60 秒发送少量数据
4. 条件允许时绕过中间设备做对照测试
```

生产环境不应该第一步直接关闭防火墙。

---

## 十、timeout 还要区分 connect timeout 和 read timeout

不能看到 `timeout` 就统一理解成“请求发出后对方没有返回业务响应”。

### 1. Connect timeout

例如：

```text
java.net.ConnectException: Connection timed out
```

更偏向：

```text
TCP 三次握手没有在规定时间内完成
```

优先排查：

```text
路由
ACL
防火墙 DROP
网络路径
目标主机可达性
```

### 2. Read timeout

例如：

```text
java.net.SocketTimeoutException: Read timed out
```

说明：

```text
TCP 通常已经建立
↓
请求可能已经发送
↓
规定时间内没有读到预期响应
```

优先排查：

```text
服务处理过慢
线程阻塞
数据库 / 下游变慢
网络丢包
响应没有及时返回
```

所以以后看到 timeout，先问：

> **到底是 connect timeout，还是 read timeout？**

---

## 十一、refused、timeout、reset 与 TCP 生命周期

### Connection refused

```text
阶段：TCP 建连时
含义：目标明确拒绝连接
优先：监听端口 / 服务状态 / 防火墙 REJECT
```

### Connect timeout

```text
阶段：TCP 建连阶段
含义：三次握手未在规定时间完成
优先：网络路径 / 路由 / ACL / 防火墙 DROP
```

### Read timeout

```text
阶段：TCP 已建立以后
含义：规定时间内没有读到业务响应
优先：应用处理 / 线程 / 数据库 / 下游 / 网络质量
```

### Connection reset by peer

```text
阶段：通常是 TCP 已建立或通信过程中
含义：收到 RST，连接被强制终止
优先：tcpdump 找 RST 来源，再查应用 / OS / 防火墙 / NAT / LB
```

---

## 十二、为什么线上可能同时出现三种错误

“同时出现”不是指同一条 TCP 连接同时发生 timeout、refused、reset，而是同一段时间内不同请求、不同连接、甚至不同节点分别发生不同故障。

例如：

```text
10:01
某条长连接被中间设备清理
→ reset

10:02
Java 服务崩溃，8082 没人监听
→ refused

10:03
防火墙开始 DROP 新连接
→ connect timeout
```

所以不同错误实际上是在告诉我们：

> **不同连接在 TCP 生命周期的不同阶段失败。**

---

## 十三、今天形成的实际排障顺序

以后看到明确 TCP 异常，不要固定每次都从 ping、IP、路由、ARP 从头查。

应该先利用错误信息判断故障阶段。

```text
Connection refused
↓
优先查 ss -lntp
↓
服务状态 / 启动日志

Connect timeout
↓
三次握手没有完成
↓
网络路径 / 路由 / ACL / 防火墙 DROP

Read timeout
↓
TCP 已建立
↓
应用处理 / 线程 / 数据库 / 下游 / 网络质量

Connection reset by peer
↓
tcpdump
↓
RST 从哪边来？
↓
结合 journalctl / dmesg / 应用日志 / 中间设备
↓
定位真正根因
```

核心原则：

> **已经获得的证据，要用来排除前面的层级，不要每次故障都从网络最底层重新开始。**

---

## 十四、今天需要记住的命令

查看监听端口：

```bash
ss -lntp | grep 8082
```

查看 TCP 连接：

```bash
ss -antp
```

指定目标：

```bash
ss -antp | grep 10.0.0.20
```

抓指定主机：

```bash
tcpdump -nn host 10.0.0.20
```

抓指定主机和端口：

```bash
tcpdump -nn host 10.0.0.20 and port 8082
```

查看 systemd 日志：

```bash
journalctl -u app.service
```

查看内核日志：

```bash
dmesg -T
```

---

## 十五、今日训练问题复盘

1. `Connection reset by peer` 不能直接说明网络不通，它表示当前 TCP 连接收到了 RST，被强制终止。

2. `Connection refused` 优先查端口监听和服务状态；`reset` 优先抓包确认 RST 来源，再查应用、OS 和中间设备。

3. ping 正常、TCP 已建立但采集过程中 reset 时，继续反复 ping 价值很低，因为故障已经发生在 TCP 建立以后。

4. 抓包看到 `.50 → .20 Flags [R.]`，能够证明当前抓包位置看到了来自 `.50` 方向的 RST，但不能直接证明一定是 `.50` 上的 Java 代码主动发出。

5. OOM 场景的链路应该是：内存压力过大 → OOM Killer 杀 Java → 原 TCP 连接异常 → 客户端出现连接异常 → systemd 根据 Restart 策略重新拉起。

6. 长连接固定空闲 300 秒后 reset，要重点怀疑防火墙 / NAT 的 idle timeout，并通过抓包、设备配置日志、keepalive 对照和绕路实验逐步验证。

7. `timeout` 需要继续区分 `connect timeout` 和 `read timeout`，不能把所有 timeout 都归为同一个 TCP 阶段。

---

## 十六、今天最终形成的排障模型

```text
先看异常是什么
↓
判断故障发生在哪个 TCP 阶段
↓
利用已有证据排除不必要的层级
↓
需要 RST 证据时抓包
↓
确认 RST 来源方向
↓
把抓包时间和应用 / 系统 / 网络设备日志对齐
↓
找到连接为什么被终止
```

最终可以压缩成一句话：

> **错误信息告诉你“发生了什么”，抓包帮助你判断“谁在做”，日志和系统状态帮助你继续找到“为什么做”。**