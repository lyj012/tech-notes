# 后端工程化每日训练 Day 56：Linux 网卡 RX TX、丢包与网络排障复盘

## 一、今天学习什么

今天学习 Linux 服务器网络监控的基础概念和排障思路，核心包括：

- 磁盘与内存的区别
- `ens33` 是什么
- RX / TX 分别代表什么
- `rx_errors` / `rx_dropped` 的区别
- 为什么 Counter 类指标要看“是否持续增长”
- 网卡 UP、Ping 正常为什么不能证明网络完全正常
- 大量设备同时 Timeout 时为什么优先找公共依赖
- 如何让 Codex 帮助追 Java Agent 中的完整网络采集链路

今天最重要的主线是：

```text
Linux 网络异常
↓
先确认网卡
↓
再看链路质量
↓
再看 IP / 路由
↓
再看目标协议和端口
↓
再进入应用
```

---

## 二、磁盘和内存先区分清楚

### 1. 磁盘是什么

在 Windows 中平时看到的：

```text
C 盘
D 盘
```

可以先理解为磁盘上的分区。

磁盘主要负责：

```text
长期保存数据
```

例如：

- Java jar 包
- 日志文件
- 数据库文件
- 上传文件
- 操作系统文件

断电以后，磁盘上的数据通常仍然存在。

---

### 2. 内存是什么

内存可以先理解成：

> 程序运行时使用的临时空间。

例如启动 IDEA：

```text
IDEA 安装文件
原本在磁盘
↓
启动 IDEA
↓
运行过程中需要的代码和数据
进入内存
↓
CPU 从内存中读取并处理
```

所以最简单的区分是：

```text
磁盘
=
东西长期存在哪里

内存
=
程序运行时数据临时放在哪里
```

---

## 三、ens33 是什么

Linux 中常见网卡接口名：

```text
eth0
ens33
ens160
enp0s3
```

其中：

```text
ens33
=
Linux 的一个网络接口 / 网卡名称
```

例如：

```bash
ip addr
```

可能看到：

```text
ens33
inet 192.168.1.100/24
```

表示这台服务器通过 `ens33` 这个网络接口进行网络通信。

可以类比 Windows 中的：

```text
以太网
WLAN
```

---

## 四、RX 和 TX

### 1. RX

```text
RX
=
Receive
=
接收
```

站在当前 Linux 网卡角度：

```text
外部
↓
ens33
↓
Linux / Java
```

属于：

```text
RX
```

例如客户端调用 Java API：

```text
客户端
↓
ens33 RX
↓
Java 服务
```

---

### 2. TX

```text
TX
=
Transmit
=
发送
```

例如 Java 返回数据：

```text
Java 服务
↓
ens33 TX
↓
客户端
```

所以可以简单记：

```text
进 ens33 = RX
出 ens33 = TX
```

但真正判断方向时，一定要记住：

> RX / TX 永远站在当前接口自己的角度判断。

---

## 五、Linux 网卡常见原始指标

Linux 常见可以从：

```text
/proc/net/dev
```

或者：

```text
/sys/class/net/ens33/statistics/
```

读取网卡统计数据。

常见字段：

```text
rx_bytes
tx_bytes

rx_packets
tx_packets

rx_errors
tx_errors

rx_dropped
tx_dropped
```

含义：

```text
rx_bytes / tx_bytes
=
累计接收 / 发送多少字节

rx_packets / tx_packets
=
累计接收 / 发送多少个包

rx_errors / tx_errors
=
收发过程中出现错误的数据包

rx_dropped / tx_dropped
=
数据包到达某个处理环节后没有继续被处理，最终被丢弃
```

---

## 六、RX/TX Bytes 为什么不能直接当当前网速

例如：

```text
rx_bytes = 982734982734
```

这个值一般是累计值，不代表当前每秒接收速度。

它和之前学习 Counter64 的思想一样：

```text
第一次采样
↓
第二次采样
↓
计算增量 delta
↓
除以时间间隔
↓
得到速率
```

例如：

```text
10:00
rx_bytes = 10 GB

10:01
rx_bytes = 10.6 GB
```

一分钟增加：

```text
0.6 GB
```

再除以 60 秒，才能得到这一分钟的平均接收速率。

所以：

```text
累计 Counter
≠
当前速率
```

---

## 七、Error 和 Dropped 的区别

可以先简单理解：

```text
Error
=
包本身或者收发过程出现错误

Dropped
=
包可能本身没有坏
但系统没有继续处理它
```

`rx_errors` 可能与这些方向有关：

```text
物理链路
网卡硬件
驱动
帧错误
CRC
```

`rx_dropped` 可能与这些方向有关：

```text
接收队列满
系统来不及处理
内核缓冲区不足
突发流量过大
```

但监控指标只能提供排查方向，不能仅凭一个数值直接下根因结论。

---

## 八、Dropped 最重要的是看是否继续增长

例如：

```text
10:00 = 100
10:01 = 300
10:02 = 700
10:03 = 1400
```

说明：

```text
当前仍然持续有新的包被丢弃
```

如果一直是：

```text
1400
1400
1400
1400
```

说明：

```text
历史累计丢过 1400 个包
但当前没有继续增加
```

所以 Counter 类型异常指标不能只问：

```text
当前累计值大不大？
```

更重要的是：

```text
最近还在不在增长？
```

持续增长通常比一个很大的历史累计值更值得立即调查。

---

## 九、RX 很高不代表网络故障

例如：

```text
RX = 800 Mbps
TX = 300 Mbps
Errors = 0
Dropped = 0
Java 接口正常
```

不能因为：

```text
RX = 800 Mbps
```

就直接判断网络故障。

因为高流量可能只是：

```text
业务高峰
数据同步
批量传输
备份
```

还需要结合：

```text
网卡最大带宽
Errors 是否增长
Dropped 是否增长
接口响应时间
高流量持续时间
```

一起判断。

核心：

```text
流量高
≠
网络一定异常
```

---

## 十、网卡 UP 和 Ping 正常也不能证明网络完全正常

例如：

```text
ens33 = UP
Ping 网关正常

但是
Java 无法连接数据库 TCP 2883
```

这里不能得出：

```text
网络完全正常
```

原因是：

```text
网卡 UP
=
接口被系统启用
```

而 Ping 通主要说明：

```text
ICMP 基本可达
```

Java 连接数据库还需要：

```text
IP
↓
路由
↓
TCP 2883
↓
防火墙 / ACL
↓
目标数据库服务监听
↓
数据库认证
```

如果连 TCP 连接都没有建立，就不应该优先检查用户名、密码、权限。

更合理的顺序：

```text
网卡
↓
IP
↓
路由
↓
目标 IP 可达性
↓
TCP 2883
↓
防火墙 / ACL
↓
目标服务是否监听
↓
最后才是账号、密码、权限等应用层问题
```

---

## 十一、大量设备同时 Timeout：优先找公共依赖

例如：

```text
Collector 管理 200 台交换机

同一分钟
180 台出现 SNMP Timeout
```

不应该优先：

```text
逐台检查 180 台交换机
```

更应该先检查共同路径：

```text
Collector 进程
↓
Collector 所在服务器网卡
↓
服务器路由
↓
防火墙 / ACL
↓
上联交换网络
↓
公共网络链路
```

因为：

```text
大量对象
同一时间
出现相同类型故障
```

大概率存在公共依赖问题。

但这里不能直接说：

```text
绝对是网卡坏了
```

正确判断应该是：

> 优先怀疑公共依赖，网卡只是其中一个候选。

这是今天非常重要的排障思想：

> 先找异常对象之间的共同点，比逐个检查对象效率高得多。

---

## 十二、Java 偶发超时 + rx_dropped 持续增长怎么排

假设：

```text
CPU 正常
内存正常
磁盘正常

Java 请求偶发超时
rx_dropped 持续增长
```

首先要明确：

```text
CPU / 内存 / 磁盘正常
```

只能说明一部分本机资源没有明显异常，不能排除：

```text
Linux 网卡
网络链路
交换机
远端服务
```

可以按下面顺序排：

```text
第一层：服务器网卡

ens33 是否存在？
ens33 是否 UP？

↓

第二层：链路质量

carrier 是否正常？
rx_errors 是否增长？
rx_dropped 是否继续增长？
tx_errors / tx_dropped 是否增长？
RX / TX 是否异常？

↓

第三层：IP

IP 地址是否正确？
子网是否正确？

↓

第四层：路由

目标地址从哪个接口出去？
下一跳网关是谁？

↓

第五层：协议 / 端口

目标 TCP / UDP 是否真正可达？
防火墙 / ACL 是否拦截？

↓

第六层：交换机 / 公共链路

交换机端口是否异常？
链路是否丢包？

↓

第七层：远端应用

目标 Java / 数据库 / SNMP 服务是否真的正常？
```

核心仍然是：

```text
先底层
↓
再网络
↓
再协议
↓
最后应用
```

---

## 十三、Linux 网卡监控在 Agent 中的大致采集链路

Linux Agent 可能从：

```text
/proc/net/dev
```

读取：

```text
ens33

RX bytes
RX packets
RX errors
RX dropped

TX bytes
TX packets
TX errors
TX dropped
```

然后进入监控系统：

```text
Linux
↓
Agent
↓
读取网卡统计数据
↓
识别具体 interface，例如 ens33
↓
解析 RX/TX 原始 Counter
↓
保存前后两次采样
↓
计算 delta / 速率
↓
上报
↓
后端接收
↓
时序数据库 / 数据库存储
↓
后端接口
↓
前端展示
```

如果服务器同时存在：

```text
ens33
docker0
lo
virbr0
```

还必须搞清楚平台究竟监控的是哪个网络接口，不能把 `lo` 等接口的数据错误当成服务器对外流量。

---

## 十四、进入 Java Agent 源码怎么追

如果要追：

> Linux ens33 的 RX/TX 流量、rx_dropped、rx_errors 到底从哪里采、怎么算、怎么存储、最后怎么显示到前端？

可以给 Codex 的搜索线索：

```text
/proc/net/dev
network
interface
networkInterface
rxBytes
txBytes
rxPackets
txPackets
rxErrors
txErrors
rxDropped
txDropped
```

但实际工作中，没有必要自己机械地一个关键词一个关键词搜索。

更高效的方式是直接把完整调查目标交给 Codex，让它基于真实源码定位链路。

可以直接这样描述：

```text
请在当前 Java Agent 项目源码中完整追踪 Linux 网卡 ens33 的监控链路。

我要搞清楚以下指标：
- RX/TX 流量
- rx_dropped
- rx_errors

请基于真实源码，从头到尾追踪：

Linux 原始数据来源
→ Agent 从哪里读取
→ 如何识别 ens33/interface
→ 原始字段如何解析
→ RX/TX 累计 Counter 如何计算成速率
→ dropped/errors 如何处理
→ 数据如何上报
→ 后端如何接收
→ 存储到哪里
→ 哪个接口返回给前端
→ 前端哪个页面/组件展示

要求：
1. 不要只解释概念，必须定位真实类、方法、字段和调用链。
2. 每一步给出文件路径、类名、方法名。
3. 某一段不在当前项目中时，明确指出链路在哪里断了。
4. 优先搜索 /proc/net/dev、rxBytes、txBytes、rxErrors、rxDropped、networkInterface、interface 等线索，但不要只依赖这些关键词。
5. 最后整理成一条完整数据流。
```

真正需要自己掌握的不是死记所有搜索关键词，而是看懂 Codex 给出的调查结果。

例如 Codex 找到：

```text
/proc/net/dev
↓
LinuxNetworkCollector.collect()
↓
NetworkInterfaceDTO
↓
rxBytes
↓
calculateRate()
↓
AgentReporter.report()
↓
...
```

接下来应该继续确认：

```text
为什么从这里采？
原始字段是累计值还是瞬时值？
速率在哪里计算？
时间差在哪里取？
ens33 怎么被识别和筛选？
数据在哪里上报？
最终存储字段是什么？
后端接口是哪一个？
前端单位是什么？
```

所以：

> AI 可以替我搜索源码，但最终仍然要由我建立对系统数据链路的理解，并判断 AI 有没有漏链路、追错对象或误解业务。

---

## 十五、今天答题中纠正的几个错误判断

### 1. 错误：RX 很高就是网络故障

正确：

```text
RX 很高
只能说明接收流量很大
```

必须结合带宽、Errors、Dropped、RT 等指标判断。

---

### 2. 错误：CPU、内存、磁盘正常，就能排除交换机

正确：

```text
CPU / 内存 / 磁盘正常
只能排除一部分服务器本机资源问题
```

不能证明：

```text
网卡正常
网络正常
交换机正常
远端服务正常
```

---

### 3. 错误：180 台交换机同时 Timeout，绝对是网卡问题

正确：

```text
大量设备同时出现同类异常
↓
优先找公共依赖
```

但公共依赖可能是：

```text
Collector
网卡
路由
防火墙
ACL
交换网络
其他公共链路
```

不能过早锁死根因。

---

### 4. 错误：数据库连不上先检查账号权限

如果表现为：

```text
Connection timed out
Connection refused
```

应该优先确认：

```text
网络
路由
目标端口
防火墙
目标服务监听
```

TCP 连接建立以后，才进入用户名、密码、权限等数据库认证问题。

---

## 十六、今天总结

今天需要真正记住的核心不是命令，而是几个判断模型。

### 1. 网卡方向

```text
ens33

进来的数据 = RX
出去的数据 = TX
```

---

### 2. Counter 指标

```text
rx_bytes
rx_dropped
rx_errors
```

很多都是累计值。

因此要关注：

```text
当前值
+
一段时间内的增量 / 变化趋势
```

---

### 3. 网络异常判断

```text
流量高
≠
网络故障

Ping 通
≠
业务端口一定正常

网卡 UP
≠
整条网络链路正常
```

---

### 4. 大规模异常

```text
大量对象
同一时间
相同故障
↓
先找公共依赖
```

不要逐个对象低效排查。

---

### 5. 网络排障顺序

```text
网卡
↓
链路质量
↓
IP
↓
路由
↓
协议 / 端口
↓
交换网络
↓
远端应用
```

---

### 6. 源码排查方式

```text
先把业务问题和目标链路描述清楚
↓
让 Codex 基于真实源码搜索
↓
拿到类 / 方法 / 字段 / 调用链
↓
自己理解数据从哪里来、在哪里计算、怎么流转
↓
检查 AI 是否漏链路或追错
```

今天真正需要形成的能力不是“自己手动搜索得多快”，而是：

> 能把问题描述清楚，能让 AI 高效定位源码，并且自己能够理解、验证和复述完整的数据链路。
