# 后端工程化每日训练 Day 62：交换机端口状态、错误包与丢弃包复盘

## 一、今天学习什么

今天继续补全交换机监控与网络排障链路，核心学习：

**交换机端口状态、Error、CRC、Discard 分别说明什么，以及为什么端口显示 UP，网络仍然可能抖动、丢包，甚至导致 SNMP Timeout 和 Java Connection Timeout。**

今天最重要的判断是：

> **交换机端口 UP，只说明链路已经建立；Error、CRC、Discard 才能进一步判断这条链路的通信质量是否健康。**

以前看到：

```text
Oper Status = UP
```

容易直接判断：

```text
端口已经 UP
↓
网络肯定正常
```

这个判断不够准确。

真实情况可能是：

```text
端口 UP

但是：
CRC Error 持续增长
Input / Output Error 增长
Discard 增长
Ping 丢包
链路抖动
```

最终上层可能表现为：

```text
SNMP Timeout
Java Connection timed out
设备 ONLINE / OFFLINE 抖动
```

---

## 二、Admin Status 和 Oper Status

交换机端口需要先区分两个状态。

### 1. Admin Status

表示：

```text
管理员希望这个端口是什么状态
```

例如：

```text
Admin Status = DOWN
```

可能只是管理员主动执行了 `shutdown`。

所以：

```text
Admin DOWN
+
Oper DOWN
```

不一定代表故障。

它可能只是一个未使用端口、备用端口，或者管理员主动关闭的端口。

### 2. Oper Status

表示：

```text
端口实际运行状态
```

例如：

```text
Admin Status = UP
Oper Status  = DOWN
```

含义是：

```text
配置上希望端口工作
↓
但是实际链路没有建立
```

这时候优先排查：

```text
对端设备是否开机
↓
对端端口是否正常
↓
网线 / 光纤
↓
光模块
↓
交换机接口
↓
物理链路
```

这里不能一开始就跳到 Java、SNMP、数据库。

因为：

> **如果物理链路本身都没有建立，上层协议和应用排查的优先级就很低。**

---

## 三、为什么 Oper Status = UP 仍然不能证明网络健康

`Oper Status = UP` 只能证明：

```text
链路已经建立
```

不能证明：

```text
每一个数据帧都能正确传输
```

例如：

```text
Server
  │
  │ 质量较差的网线 / 光链路
  │
Switch
```

物理连接仍然可能保持：

```text
UP
```

但是传输过程中部分帧发生损坏：

```text
数据帧传输
↓
数据发生错误
↓
CRC 校验失败
↓
交换机丢弃坏帧
↓
上层出现丢包
```

所以端口 UP 以后，还应该继续看：

```text
Error
CRC / FCS
Discard
RX / TX
带宽利用率
端口是否 flap
Ping 是否丢包
```

---

## 四、Error 和 CRC

### 1. Error 是什么

交换机端口会维护错误计数器，例如：

```text
Input Errors
Output Errors
```

可以先理解为：

```text
Error 增长
↓
数据传输过程中存在异常
```

其中非常常见的一类就是：

```text
CRC / FCS Error
```

### 2. CRC Error 是什么

以太网帧会携带校验信息。

接收方重新计算后，如果发现校验结果不一致：

```text
发送的数据
↓
传输过程中发生损坏
↓
接收方校验失败
↓
CRC / FCS Error
```

常见排查方向包括：

```text
网线质量
水晶头 / 接头
光纤
光模块
接口硬件
链路干扰
速率 / 双工协商
```

但是要注意：

```text
CRC 高
≠
一定是网线坏了
```

CRC 是一个**定位方向的证据**，不是最终根因。

它告诉我们：

> **这段二层 / 物理链路存在数据帧损坏，需要继续向链路质量方向排查。**

---

## 五、Error 和 Discard 的区别

这个区别是今天最重要的知识点之一。

### Error

更偏向：

```text
数据本身出现问题
```

例如：

```text
收到数据帧
↓
CRC 校验失败
↓
Error
↓
丢弃
```

### Discard

更偏向：

```text
数据本身可能没坏
但是设备没有继续处理
```

例如：

```text
大量流量进入
↓
发送速度跟不上
↓
队列堆积
↓
缓存被占满
↓
Discard
```

所以：

```text
Error
→ 更关注链路质量 / 数据损坏

Discard
→ 更关注拥塞 / 队列 / 缓存 / 策略
```

---

## 六、为什么带宽 96% + Output Discard 增长更像拥塞

场景：

```text
CRC = 0
Input Error = 0
Output Error = 0
Output Discard 持续快速增加
带宽利用率 = 96%
```

这时候优先怀疑：

```text
流量拥塞 / 队列问题
```

而不是优先怀疑：

```text
网线 / 光纤损坏
```

原因是现有证据组合为：

```text
没有明显错误帧
+
端口接近带宽上限
+
Discard 快速增长
```

更符合：

```text
流量接近接口能力上限
↓
交换机内部队列压力增加
↓
缓存不够
↓
正常数据被丢弃
```

这也是排障时非常重要的思维：

> **不要只看一个指标，要看多个指标组合起来形成的证据链。**

---

## 七、Counter 累计值与增长速度

CRC、Error、Discard 等很多网络指标本质上都是 Counter。

例如：

```text
CRC Errors

10:00 = 100
10:01 = 102
10:02 = 800
10:03 = 3000
```

真正重要的不是：

```text
CRC 累计值 = 3000
```

而是：

```text
10:00 → 10:01：+2
10:01 → 10:02：+698
10:02 → 10:03：+2200
```

因为累计值可能是设备运行几个月甚至几年积累下来的。

所以监控系统更应该关注：

```text
当前值 - 上一次值
=
本周期新增错误量
```

以及：

```text
错误增长速度
```

不能简单使用：

```text
累计值 > 0
↓
立即告警
```

例如一台运行一年的设备：

```text
CRC 累计 = 3
```

和一分钟内：

```text
CRC 100 → 10000
```

完全不是一个严重程度。

---

## 八、一批设备同时 SNMP Timeout：共同故障域

场景：

```text
一条交换机上联端口
↓
下面有 30 台设备
↓
30 台设备同时 SNMP Timeout
```

同时发现：

```text
Admin Status = UP
Oper Status  = DOWN
```

这时候不能逐台检查：

```text
设备1 Community
设备2 Community
设备3 Community
...
设备30 Community
```

应该优先检查：

```text
上联端口
```

因为 30 台设备共享同一条网络路径。

完整判断：

```text
30台设备同时异常
↓
寻找共同依赖
↓
发现共同上联端口
↓
Admin UP + Oper DOWN
↓
优先排查上联物理链路
```

重点检查：

```text
光纤
光模块
接口
对端交换机
对端端口
链路协商
```

这体现的是：

> **共同故障域思维：一批对象同时异常，优先寻找它们共享的网络路径和基础设施。**

---

## 九、48 口交换机接口监控模板应该理解哪些指标

实际工作中，设备厂商资料、客户文档和公司现有 Schema 往往已经定义了很多采集项，开发时可以根据已有模板和规范实现。

但是不能只停留在“照着字段写”。

需要理解为什么这些指标存在。

### 1. 接口身份

```text
ifIndex
ifName
ifDescr
ifAlias
```

作用：

```text
确认这个指标到底属于哪个真实接口
```

其中 `ifIndex` 常用于采集和关联，`ifName / ifDescr / ifAlias` 更方便展示和确认真实端口。

### 2. 管理状态

```text
ifAdminStatus
```

回答：

```text
管理员有没有启用这个端口？
```

### 3. 实际状态

```text
ifOperStatus
```

回答：

```text
这个端口实际有没有连通？
```

例如：

```text
Admin UP + Oper DOWN
```

比单独的“端口 Down”信息量更高。

### 4. 流量

```text
RX
TX
带宽利用率
```

流量 Counter 一般需要通过两个采集周期之间的差值计算速率。

### 5. 错误

```text
Input Error
Output Error
CRC / FCS Error
```

重点看增量和趋势。

### 6. 丢弃

```text
Input Discard
Output Discard
```

重点结合流量、带宽利用率判断是否存在拥塞。

所以监控模板并不是单纯“采到越多越好”，而应该考虑：

```text
这个值是什么？
↓
怎么计算？
↓
怎么展示？
↓
什么时候值得告警？
↓
它出现异常时能帮助定位什么问题？
```

---

## 十、真实场景：端口 UP，服务器仍然反复 Offline

场景：

```text
服务器
ONLINE
↓
OFFLINE
↓
ONLINE
```

交换机端口始终：

```text
UP
```

但是同时：

```text
CRC 快速增长
Ping 存在丢包
```

完整链路可以理解为：

```text
交换机端口仍然 UP
↓
说明物理连接没有完全断开
↓
但是 CRC 快速增长
↓
二层出现错误帧
↓
部分帧被丢弃
↓
IP 层出现丢包
↓
TCP 发生重传 / 建连变慢
↓
SNMP 请求偶发超时
↓
监控采集失败
↓
平台判断设备 Offline
```

因此：

```text
端口没有 Down
≠
设备一定不会 Offline
```

这时候题目已经给出 `CRC 快速增长 + Ping 丢包` 两个强证据，所以优先级应放在二层 / 物理链路质量，而不是先跳到 IP 冲突或者 TCP Connection refused。

---

## 十一、Codex + 监控平台 + 交换机 + Linux 联合排查

真实故障：

> 平台显示交换机 Gi0/20 后的一台服务器频繁 Offline，同时 Java 服务偶发连接数据库 Timeout。

不能只盯着 Java 日志。

需要从多个视角还原完整故障链。

### 1. Codex：追代码和数据流转

让 Codex 重点查：

```text
设备 Offline 是怎么判断的？
↓
状态来自哪个采集项？
↓
OID 在哪里配置？
↓
Collector 怎么采？
↓
采集周期和 Timeout 是多少？
↓
失败几次后判 Offline？
↓
数据存到哪里？
↓
告警怎么产生？
↓
前端怎么展示？
```

可以重点搜索：

```text
offline
status
timeout
snmp
collector
alarm
ifOperStatus
ifInErrors
ifOutErrors
ifInDiscards
ifOutDiscards
```

Codex 的主要作用是：

> **把代码中的监控链路和状态判断逻辑还原出来。**

它不能替代真实交换机和 Linux 网络状态检查。

### 2. 监控平台：看时间和趋势

重点看：

```text
最后采集时间
设备 ONLINE / OFFLINE 时间点
端口状态趋势
CRC 趋势
Error 趋势
Discard 趋势
流量趋势
告警时间
```

不能只看“现在是 Offline”。

需要关联：

```text
设备什么时候 Offline？
↓
同一时间 CRC 是否暴涨？
↓
Discard 是否增加？
↓
端口有没有 flap？
↓
流量是否异常？
```

### 3. 交换机：看真实链路

针对 `Gi0/20` 重点检查：

```text
Admin Status
Oper Status
CRC
Input / Output Error
Input / Output Discard
RX / TX
带宽利用率
端口是否反复 Up / Down
光模块
物理链路
```

### 4. Linux：验证服务器侧网络

#### IP 和网卡

```bash
ip addr
```

看：

```text
IP 地址
网卡状态
```

#### ARP / 邻居表

```bash
ip neigh
```

看：

```text
IP ↔ MAC
邻居是否 REACHABLE / STALE / FAILED
```

注意：

```text
ip neigh
```

看的是邻居 / ARP，不是路由。

#### 路由

```bash
ip route
```

看：

```text
目标网段走哪条路由
默认网关是什么
```

#### Ping

```bash
ping <目标IP>
```

看：

```text
是否丢包
延迟是否抖动
```

#### 网卡统计

```bash
ip -s link
```

看：

```text
errors
dropped
```

#### TCP

```bash
ss -ant
```

看 TCP 连接状态，并结合 Java 的连接超时现象分析。

---

## 十二、今天形成的完整排障链路

以前看到：

```text
SNMP Timeout
```

容易直接进入：

```text
SNMP 配置
OID
Java 代码
```

现在应该建立完整链路：

```text
Collector
↓
Linux 网卡
↓
路由
↓
ARP / 邻居
↓
VLAN
↓
交换机转发
↓
交换机端口状态
↓
Error / CRC / Discard
↓
目标设备
↓
UDP 161
↓
SNMP Agent
↓
Community / OID
```

如果同时出现 Java 数据库连接 Timeout，还需要考虑：

```text
物理链路异常
↓
二层帧错误 / 丢包
↓
IP 丢包
↓
TCP 重传 / 建连变慢
↓
超过 Java timeout
↓
Connection timed out
```

---

## 十三、今天答题中的几个关键修正

### 1. Oper UP 不是“数据一定正确”

只能说明：

```text
链路建立
```

还必须继续检查通信质量。

### 2. CRC 快速增长时不要先跳到应用层

优先：

```text
二层 / 物理链路
```

检查网线、光纤、模块、接口、协商等。

### 3. `CRC + Ping 丢包` 已经是强证据

这时候 IP 冲突、TCP refused 等虽然理论上可能发生，但不是当前证据下的第一排查方向。

### 4. `ip neigh` 和 `ip route` 不要混淆

```text
ip neigh
→ 邻居 / ARP / IP-MAC

ip route
→ 路由表 / 下一跳
```

### 5. 一批设备一起异常先找共同故障点

```text
30台设备全部Timeout
+
共同上联 Oper DOWN
↓
先查共同上联
```

而不是逐台检查 30 个 SNMP Community。

---

## 十四、今天总结

今天真正需要掌握的不是几个 SNMP 字段，而是：

```text
端口状态
↓
链路质量
↓
二层转发
↓
IP通信
↓
TCP / UDP
↓
SNMP / Java
↓
监控平台状态
```

最终形成三个核心判断：

> **1. 交换机端口 UP，只说明链路建立，不代表通信质量正常。**

> **2. Error / CRC 更偏向链路和数据帧异常，Discard 更偏向拥塞、队列和资源压力。**

> **3. 排障时不要围绕最终报错猜问题，而要根据指标和时间线逐层缩小故障域。**
