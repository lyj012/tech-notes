# 后端工程化每日训练 Day 57：TCP 连接状态与 Java 服务端口排障复盘

## 一、今天学习什么

今天继续沿着 Linux 网络监控与服务排障往上学习，核心解决一个真实工作里非常常见的问题：

```text
服务器 Ping 得通

但是

Java 服务 8082 访问不了
数据库 2883 连不上
后端接口 Connection refused / timed out
```

今天重点学习：

- Ping 通到底能证明什么
- Java 进程存在为什么不等于服务正常
- TCP 三次握手和连接建立
- `LISTEN`、`ESTABLISHED`、`TIME_WAIT`、`CLOSE_WAIT`
- `Connection refused` 和 `Connection timed out` 的区别
- 为什么 TCP 正常不代表 Java 业务正常
- 如何按层次排查 Java 服务端口不可用
- 在 Java / Agent 项目中如何追端口状态检测完整链路

今天最重要的主线：

```text
IP 能到
↓
TCP 端口能通
↓
TCP 连接能建立
↓
应用能够正常处理请求
```

这四层不能混成一件事。

---

## 二、Ping 通到底证明什么

平时看到：

```bash
ping 192.168.10.20
```

返回正常，只能先说明：

```text
至少部分 IP / ICMP 网络通信正常
```

它不能继续推出：

```text
Java 进程正常
8082 正在监听
TCP 一定能够建立连接
HTTP 接口一定正常
数据库一定正常
业务一定正常
```

所以：

```text
Ping 通
≠
Java 服务正常
```

例如：

```text
服务器在线
Ping 正常

但是
Java 服务根本没有启动
8082 没有任何程序监听
```

这时客户端依然无法访问 Java 服务。

今天第一个需要建立的工程思维就是：

> 主机可达、协议可达、端口可达、应用可用必须分层判断。

---

## 三、端口监听 LISTEN 是什么

假设 Spring Boot 配置：

```properties
server.port=8082
```

服务正常启动后，大致过程：

```text
Java 进程
↓
向 Linux 申请 TCP 8082
↓
bind
↓
listen
↓
等待客户端连接
```

Linux 上可能看到：

```text
0.0.0.0:8082
LISTEN
```

`LISTEN` 可以先理解为：

> 有程序正在等待别人连接这个 TCP 端口。

常用命令：

```bash
ss -lntp
```

例如：

```text
Java 进程存在

但是

ss -lntp
没有 8082
```

不能说明 Java 已经正常对外提供 8082。

因为可能存在：

```text
服务仍在启动过程中

实际监听的是 8083

8082 bind 失败

配置读取错误

当前 Java 进程根本不是目标服务
```

所以不能形成错误判断：

```text
有 Java 进程
→
Java 服务正常
```

更合理的是：

```text
Java 进程存在
↓
目标端口是否 LISTEN
↓
监听 IP 是否正确
↓
再继续验证 TCP 和应用
```

---

## 四、TCP 连接是怎么建立的

客户端访问 Java 服务以前，通常先建立 TCP 连接。

最基础的是三次握手：

```text
客户端
↓
SYN
“我想连接你”

服务器
↓
SYN + ACK
“可以，我也准备好了”

客户端
↓
ACK
“收到”

↓
连接建立
↓
ESTABLISHED
```

例如浏览器访问：

```text
192.168.1.20:8082
```

完整链路更接近：

```text
找到服务器 IP
↓
找到正确路由
↓
TCP 三次握手
↓
连接建立
↓
发送 HTTP 请求
↓
Spring Boot 处理请求
↓
数据库 / Redis / 下游调用
↓
返回响应
```

所以 TCP 只是应用请求链路中的一层。

---

## 五、ESTABLISHED 能证明什么

当 TCP 三次握手完成：

```text
客户端
↔
服务器
```

连接进入：

```text
ESTABLISHED
```

它可以理解为：

> TCP 连接已经建立，可以开始正常收发数据。

但是：

```text
ESTABLISHED
≠
Java 业务一定正常
```

即使 TCP 已经建立，后面仍然可能：

```text
HTTP 请求进入 Java
↓
Java 代码异常

或者

Java 线程池堵塞

或者

数据库查询很慢 / 数据库异常

或者

Redis / 下游服务异常

或者

CPU / 磁盘 / GC 出现问题

或者

业务数据本身异常
```

最后接口仍然可能：

```text
500
超时
返回错误数据
```

所以：

> `ESTABLISHED` 只能证明 TCP 连接已经成功建立，不能证明业务请求已经成功。

---

## 六、Connection refused 是什么意思

假设客户端访问：

```text
192.168.1.20:8082
```

立即返回：

```text
Connection refused
```

更应该优先怀疑：

```text
Java 服务没启动

Java 服务启动失败

目标端口没有 LISTEN

实际监听端口不是 8082

服务只监听 127.0.0.1

客户端访问错了端口

防火墙主动 REJECT
```

例如：

```text
Java 实际监听：8083

客户端访问：8082
```

就可能直接：

```text
Connection refused
```

### 今天答题中的一个错误

一开始看到 `Connection refused`，第一反应想到：

```text
账号 / 密码是不是错了？
```

这个优先级不对。

原因是：

```text
Connection refused
通常发生在 TCP 建连阶段
```

此时很多情况下连 Java 应用、数据库认证逻辑都还没有真正进入。

所以正确优先级应该是：

```text
进程
+
端口
+
LISTEN
+
监听地址
```

而不是第一时间查用户名密码。

可以先简单记：

```text
refused
≈
我已经找到目标附近了
但这个端口没有正常接收连接
```

---

## 七、Connection timed out 是什么意思

另一种错误：

```text
连接 192.168.1.20:8082

等很久

Connection timed out
```

它和 `refused` 的排障线索不同。

Timeout 更像：

```text
客户端发出了连接请求
↓
但是迟迟没有得到正常结果
```

更应该提高这些方向的排查优先级：

```text
路由
防火墙 DROP
ACL
VPN
目标服务器不可达
网络丢包
中间网络设备
目标机器压力极大
```

所以先形成粗略判断：

```text
Connection refused
→
优先服务 / 端口 / LISTEN / REJECT

Connection timed out
→
优先网络路径 / 路由 / DROP / ACL / VPN
```

但这不是绝对规则。

例如防火墙如果主动 `REJECT`，也可能让客户端很快收到拒绝结果。

---

## 八、TIME_WAIT 是什么

TCP 连接关闭后，经常会看到：

```text
TIME_WAIT
```

今天不需要背特别细的 TCP 协议定义，只需要记住工程化版本：

> 连接主动关闭以后，会留下一段正常的“收尾状态”。

例如 Java 不断调用另一个 HTTP 服务：

```text
建立 TCP
↓
请求
↓
返回
↓
关闭

建立 TCP
↓
请求
↓
返回
↓
关闭
```

如果大量使用短连接，就可能看到很多：

```text
TIME_WAIT
```

例如：

```text
TIME_WAIT = 20000
```

不能仅凭这个数字直接判断：

```text
服务器故障
```

真正更值得关注的是：

```text
数量趋势
+
是否影响业务
```

例如：

```text
100
↓
5000
↓
30000
↓
60000
```

同时出现：

```text
新连接失败
接口超时
端口资源压力
```

这时才值得进一步调查：

```text
短连接是否过多
连接是否没有复用
请求量是否突然暴增
```

所以：

```text
TIME_WAIT 多
≠
服务器一定异常
```

更合理的是：

```text
TIME_WAIT 持续暴涨
+
业务出现异常
→
重点调查
```

---

## 九、CLOSE_WAIT 是什么

`CLOSE_WAIT` 和 `TIME_WAIT` 的诊断意义不同。

可以先这样理解：

```text
对方已经说：
“我要断开连接了”
↓
Linux 已经知道对方关闭
↓
但是当前本地应用
还没有把自己的 Socket 正常关闭
↓
CLOSE_WAIT
```

例如：

```text
10:00 = 50
10:10 = 500
10:20 = 2000
10:30 = 5000
```

而且持续不下降。

此时更应该优先怀疑：

```text
当前 Java 应用没有正确释放连接
```

常见方向：

```text
HTTP Client
Socket
InputStream
OutputStream
数据库连接
异常路径没有正常 close
```

最终可能造成：

```text
文件描述符不断消耗
↓
Socket 资源累积
↓
新连接建立失败
↓
服务异常
```

今天需要记住的区别：

```text
TIME_WAIT
→
连接已经主动关闭
留下正常收尾状态

CLOSE_WAIT
→
对方已经关闭
本地应用还没有完成 close
```

大量 `CLOSE_WAIT` 长时间不释放，比单纯看到很多 `TIME_WAIT` 更应该提高 Java 应用资源释放问题的排查优先级。

---

## 十、Ping 正常但 Java 8082 不可用怎么排

假设：

```text
服务器：192.168.10.20
Java：Spring Boot
端口：8082
```

浏览器打不开。

### 第一层：确认 IP

```text
访问的是不是正确服务器？
```

然后：

```bash
ping 192.168.10.20
```

Ping 正常只能证明主机网络基本可达，不能结束排查。

---

### 第二层：确认 Java 进程

检查：

```text
Java 服务进程是否存在？
```

如果进程不存在，问题已经可以往：

```text
启动失败
systemd
JVM
配置文件
数据库依赖
应用日志
```

方向收缩。

---

### 第三层：确认目标端口 LISTEN

```bash
ss -lntp
```

确认：

```text
8082 有没有 LISTEN？
监听在哪个 IP？
```

如果：

```text
Java 进程存在
但是没有 8082 LISTEN
```

仍然不能说明服务已经正常对外提供。

---

### 第四层：服务器本机访问

例如：

```bash
curl localhost:8082
```

如果本机正常，说明：

```text
Java 本身大概率正常
本机 TCP 路径基本正常
```

---

### 第五层：远程 TCP

如果本机正常，但是远程访问超时：

```text
问题范围进一步缩小到服务器外部访问路径
```

继续排：

```text
防火墙
路由
ACL
VPN
网络策略
中间设备
```

---

### 第六层：观察 TCP 状态

继续看：

```text
LISTEN
ESTABLISHED
TIME_WAIT
CLOSE_WAIT
```

判断：

```text
连接是否真的建立
连接是否大量堆积
是否存在资源释放异常
```

---

### 第七层：进入应用

如果：

```text
8082 LISTEN
TCP ESTABLISHED
```

但接口仍然 10 秒才返回，就应该降低 TCP 建连问题的优先级。

继续往上：

```text
Java 线程
↓
线程池
↓
数据库
↓
Redis
↓
下游服务
↓
业务代码
↓
CPU / 磁盘 / GC
```

核心原则：

> 先证明哪一层正常，再继续往上一层追。

---

## 十一、数据库 2883 场景怎么判断

例如 Java 连接 OceanBase / OBProxy：

```text
Java
↓
TCP
↓
OceanBase / OBProxy
2883
```

如果日志是：

```text
Connection refused
```

优先想到：

```text
目标 IP 是否正确？
2883 有没有 LISTEN？
OBProxy / OceanBase 是否启动？
端口配置是否正确？
```

而不是先查：

```text
数据库用户名密码
```

如果是：

```text
Connect timed out
```

则增加：

```text
路由
VPN
防火墙
ACL
网络链路
```

的排查优先级。

这和今天 Java 8082 的排障思想完全一致。

---

## 十二、SNMP 场景不能机械套 TCP 状态

今天学习的是 TCP。

但常见 SNMP GET：

```text
UDP 161
```

SNMP Trap：

```text
UDP 162
```

所以不能把：

```text
ESTABLISHED
TIME_WAIT
CLOSE_WAIT
```

机械套到 SNMP UDP 上。

真正应该先问：

```text
这个监控项到底使用什么协议？
↓
它到底采用什么探测方式？
↓
再使用对应的协议排障方法
```

这也是今天最后一道源码题里最重要的纠正。

---

## 十三、平台为什么判断 Java 8082 服务不可用

真实监控平台不会因为：

```text
Ping 正常
```

就直接认为：

```text
所有服务都正常
```

更合理的服务监控应该分层：

```text
主机可达
↓
TCP 端口可达
↓
应用健康
```

例如：

```text
ICMP：正常
TCP 8082：失败
```

正确状态应该更接近：

```text
服务器在线

但是

Java 8082 服务端口异常
```

而不是：

```text
整台服务器离线
```

---

## 十四、Agent / Collector 中端口状态的大致检测链路

如果平台监控的是 TCP 端口可用性，大致链路可能是：

```text
Agent / Collector
↓
读取目标 IP + 端口
↓
发起 TCP Connect
↓
得到连接结果

success
refused
timeout
↓
状态解析 / 状态判断
↓
监控项
↓
存储
↓
告警
↓
后端接口
↓
前端展示
```

例如最终页面显示：

```text
Java 8082 = DOWN
```

不能只看页面就断言：

```text
Java 服务一定挂了
```

还应该确认：

```text
Agent 从哪里发起检测？

检测的是 localhost 还是远程 IP？

目标 IP / Port 从哪里来的？

连接超时是多少？

success / refused / timeout 怎么映射状态？

状态有没有连续失败次数判断？

Agent 和服务器之间网络路径是否不同？
```

---

## 十五、今天源码追踪题的一个关键纠正

题目：

> 服务器 Ping 通，但是平台为什么判断 Java 8082 服务不可用？这个端口状态到底怎么检测出来的？

一开始直觉判断：

```text
采集类型肯定是 JMX
```

这个判断不能直接成立。

因为：

```text
“Java 服务”
≠
“一定通过 JMX 判断端口可用性”
```

如果目标是判断：

```text
TCP 8082 能不能建立连接
```

更可能需要追：

```text
socket
tcp
connect
port
listen
health
check
timeout
ConnectException
SocketTimeoutException
```

但是最终不能靠猜。

正确做法是：

> 先让 Codex 根据真实源码确认协议和探测实现，再继续往下追完整数据流转。

---

## 十六、交给 Codex 的真实调查目标

以后如果工作中要查：

> 为什么服务器 Ping 通，但是平台判断 Java 8082 服务不可用？

没有必要自己机械地一个关键词一个关键词搜。

更高效的方式是直接把完整调查目标交给 Codex：

```text
帮我追踪“平台为什么判断服务器 Java 8082 服务不可用”的完整源码链路。

重点定位：

1. 这个状态到底使用什么协议、什么采集 / 探测方式，不要预设是 JMX、SNMP 或其他方式。
2. Agent / Collector 从哪里发起检测。
3. 目标 IP 和端口从哪里读取。
4. 是否通过 Socket / TCP Connect 检测。
5. 如何区分 success、Connection refused、timeout。
6. 超时时间在哪里配置。
7. 检测结果如何转换成平台内部状态。
8. 是否存在连续失败次数、恢复次数、状态机或告警判断。
9. 状态如何进入监控项和存储。
10. 后端哪个接口返回给前端。
11. 前端哪个页面 / 组件最终显示“8082 不可用”。

请输出关键类、关键方法、调用关系、配置来源、数据结构，以及完整的数据流转链路。
```

最终希望 Codex 还原：

```text
主机
↓
Java 进程
↓
LISTEN
↓
Agent / Collector
↓
TCP Connect
↓
success / refused / timeout
↓
状态判断
↓
监控项
↓
存储
↓
告警
↓
后端接口
↓
前端
```

工作重点不是把所有源码都背下来，而是能够让 AI 帮助快速定位真实实现，然后自己理解关键链路和判断逻辑。

---

## 十七、今天训练问题复盘

### 问题 1：Ping 正常，但是 8082 Connection refused

结论：

```text
不能判断服务器网络完全不通
```

因为 Ping 已经说明 IP / ICMP 基本可达。

但第一反应不应该是查密码，而是：

```text
Java 进程
↓
8082 是否 LISTEN
↓
监听地址
↓
端口配置
↓
防火墙 REJECT
```

---

### 问题 2：Java 进程存在，但是没有 8082 LISTEN

结论：

```text
不能证明 Java 已经正常对外提供 8082
```

因为：

```text
进程存在
≠
目标端口已经成功绑定并监听
```

---

### 问题 3：TCP ESTABLISHED

结论：

```text
不能证明 Java 接口一定返回正常业务数据
```

后面还可能是：

```text
Java 代码
数据库
Redis
下游服务
线程池
GC
业务数据
```

出现问题。

---

### 问题 4：refused 和 timeout

```text
refused
→
优先服务 / 端口 / LISTEN

timeout
→
优先网络路径 / 防火墙 / ACL / VPN
```

密码认证通常不是 `Connection refused` 的第一排查项。

---

### 问题 5：TIME_WAIT = 20000

结论：

```text
不能仅凭这个数字判断服务器故障
```

需要继续结合：

```text
是否持续增长
新连接是否失败
接口是否超时
端口资源是否出现压力
```

如果服务正常、接口正常、连接正常，不应该看到一个高值就立即修改系统参数。

---

### 问题 6：CLOSE_WAIT 持续从 50 增长到 5000

结论：

```text
优先怀疑当前 Java 应用没有正确释放连接
```

重点看：

```text
Socket
HTTP Client
流资源
数据库连接
异常路径资源释放
```

---

### 问题 7：追 Java 8082 为什么被平台判断不可用

一开始错误地预设：

```text
肯定是 JMX
```

正确方法应该是：

```text
先确认真实协议 / 探测方式
↓
再追 Agent / Collector
↓
再追 TCP Connect 或真实检测实现
↓
再追状态判断
↓
监控项
↓
存储
↓
告警
↓
前端
```

---

## 十八、今天真正需要记住的内容

不需要死记所有 TCP 协议细节，当前阶段先牢牢记住下面几条：

### 1. Ping 通不等于服务正常

```text
Ping
只证明 IP / ICMP 基本可达
```

---

### 2. Java 进程存在不等于端口正常

```text
进程存在
↓
还要确认 8082 LISTEN
```

---

### 3. ESTABLISHED 不等于业务正常

```text
TCP 成功
≠
Java / 数据库 / Redis / 业务成功
```

---

### 4. refused 和 timeout 给出的线索不同

```text
refused
→
先查服务、端口、LISTEN

timeout
→
先查网络路径、DROP、ACL、VPN
```

---

### 5. TIME_WAIT 看趋势和业务影响

```text
存在很多
≠
一定异常
```

---

### 6. CLOSE_WAIT 持续堆积要警惕本地应用资源释放

```text
对方已经关闭
↓
本地还没 close
```

---

### 7. 排障必须分层

```text
IP
↓
进程
↓
端口 LISTEN
↓
本机 TCP
↓
远程 TCP
↓
网络策略
↓
TCP 状态
↓
Java 应用
↓
监控链路
```

---

### 8. 追源码时先确认真实实现，不要先猜采集类型

```text
先问：
这个状态到底怎么得到？

再问：
代码在哪里？

最后追：
采集 / 探测
→ 计算 / 判断
→ 存储
→ 告警
→ 接口
→ 前端
```

这套方法比背某一个 JMX、SNMP 或 TCP 名词更重要。
