# 后端工程化每日训练（第 62 天）

# 交换机端口状态、错误包与丢弃包

## 一、核心知识

交换机端口 UP 不代表网络一定健康。

端口 UP 只能说明链路已经建立，但是仍可能存在：

- CRC Error
- Input/Output Error
- Discard 增长
- 丢包
- 网络抖动
- SNMP Timeout
- Java Connection Timeout

核心判断：

> 端口能连接，与端口通信质量正常，是两回事。

---

## 二、Admin Status 与 Oper Status

### Admin Status

表示管理员配置希望端口处于什么状态。

### Oper Status

表示端口实际运行状态。

### 常见情况

### Admin DOWN + Oper DOWN

通常表示管理员主动关闭端口，不一定是故障。

### Admin UP + Oper DOWN

表示配置允许端口工作，但是实际链路没有建立。

优先排查：

- 对端设备
- 网线
- 光纤
- 光模块
- 接口

---

## 三、为什么 UP 仍然可能异常

例如：

服务器 → 网线 → 交换机

链路可能保持 UP，但是如果链路质量差：

数据帧损坏 → CRC 校验失败 → 丢弃 → 丢包

所以：

> UP 只是链路存在，不代表数据传输质量正常。

---

## 四、Error、CRC 与 Discard

### Error / CRC

表示数据帧传输过程中出现异常。

可能原因：

- 网线问题
- 光模块问题
- 光纤问题
- 接口问题
- 链路干扰

CRC 增长速度比累计值更重要。

监控应该关注：

当前值 - 上一次值 = 新增错误数量

---

### Discard

Discard 不一定代表数据损坏。

可能是：

流量过高 → 队列堆积 → 缓存不足 → 丢弃数据

更偏向：

- 拥塞
- 队列压力
- 带宽瓶颈

---

## 五、交换机接口监控指标

一个接口通常需要关注：

### 接口身份

- ifIndex
- ifName
- ifDescr
- ifAlias

### 接口状态

- ifAdminStatus
- ifOperStatus

### 流量

- RX
- TX
- 带宽利用率

### 错误

- Input Error
- Output Error
- CRC/FCS

### 丢弃

- Input Discard
- Output Discard

---

## 六、真实排障思路

设备 Offline 时，不应该直接认为是 Java 或 SNMP 代码问题。

排查链路：

物理链路
↓
交换机端口状态
↓
CRC/Error/Discard
↓
IP
↓
TCP/SNMP
↓
Java应用

---

## 七、联合排查方法

### Codex

查看：

- Offline 判断逻辑
- SNMP 采集流程
- OID 配置
- Collector 数据流转
- 状态计算逻辑

### 监控平台

查看：

- 最后采集时间
- Offline 时间点
- 状态趋势
- CRC 趋势
- Error 趋势
- Discard 趋势
- 流量趋势

### 交换机

查看：

- Admin Status
- Oper Status
- CRC
- Error
- Discard
- 流量

### Linux

常用命令：

```bash
ip addr
ip neigh
ip route
ping
ss -ant
```

分别检查：

- IP
- ARP邻居
- 路由
- 网络丢包
- TCP连接状态

---

## 八、今天总结

监控研发排障不能只看最终异常。

应该沿着完整链路分析：

物理链路
↓
交换机
↓
网络协议
↓
SNMP
↓
应用

核心理解：

> 交换机端口 UP，只说明链路建立；Error、CRC、Discard 才能进一步判断链路通信质量是否健康。
