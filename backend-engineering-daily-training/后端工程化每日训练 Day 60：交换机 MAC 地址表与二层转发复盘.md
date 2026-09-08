# 后端工程化每日训练 Day 60：交换机 MAC 地址表与二层转发复盘

## 一、今天学习什么

今天学习网络通信链路中的下一层：

**交换机如何通过 MAC 地址表判断数据帧应该从哪个端口转发。**

前面的学习解决了：

```text
IP
↓
路由
↓
下一跳
↓
ARP 获取 MAC
```

但是数据帧到达交换机以后，还有一个问题：

```text
交换机有很多端口

port1
port2
port3
...
```

交换机如何知道目标 MAC 应该从哪个端口出去？

答案：

```text
MAC 地址表
(MAC Address Table / CAM Table)
```

完整链路：

```text
服务器
↓
网卡生成以太网帧
↓
交换机收到数据帧
↓
学习源 MAC
↓
查询目标 MAC
↓
选择出口端口
↓
转发数据
```

---

## 二、交换机工作在哪一层

交换机主要工作在：

```text
OSI 第二层
数据链路层
```

关注：

```text
MAC 地址
```

而不是：

```text
IP 地址
```

区别：

```text
路由器：
看 IP
决定下一跳

交换机：
看 MAC
决定端口
```

---

## 三、什么是 MAC 地址表

例如交换机连接：

```text
port1:
服务器A

port2:
服务器B

port3:
管理设备
```

交换机内部记录：

```text
MAC 地址              端口

AA-AA-AA-AA-AA-AA     port1
BB-BB-BB-BB-BB-BB     port2
CC-CC-CC-CC-CC-CC     port3
```

含义：

> 某个 MAC 地址当前位于交换机哪个端口。

---

## 四、交换机如何学习 MAC

交换机不会手动配置所有 MAC。

它会自动学习。

例如：

服务器 A：

```text
MAC:
AA-AA
```

发送数据：

```text
服务器A
↓
交换机 port1
```

交换机收到以后看到：

```text
源 MAC:
AA-AA
```

于是记录：

```text
AA-AA
=
port1
```

注意：

交换机学习的是：

```text
源 MAC
```

不是：

```text
目标 MAC
```

原因：

源 MAC 来自哪个端口，交换机可以确定。

目标 MAC 只是想访问的对象，交换机不一定知道它在哪里。

核心：

```text
源 MAC告诉交换机：我是谁，我从哪里来

目标 MAC告诉交换机：我要找谁
```

---

## 五、目标 MAC 已知时如何转发

例如：

MAC 表：

```text
AA-AA  port1
BB-BB  port2
```

服务器 A 访问服务器 B：

```text
源 MAC:
AA-AA

目标 MAC:
BB-BB
```

交换机查询：

```text
BB-BB
↓
port2
```

于是：

```text
port1
↓
交换机
↓
port2
```

直接发送。

这种叫：

```text
已知单播
Known Unicast
```

不会发送给所有端口，因为交换机已经知道目标位置。

---

## 六、目标 MAC 未知怎么办

如果交换机收到：

```text
目标 MAC:
CC-CC
```

但是 MAC 地址表没有：

```text
CC-CC
```

交换机不知道 CC-CC 在哪里。

因此会：

```text
未知单播泛洪
Unknown Unicast Flooding
```

发送给：

```text
除了来源端口之外的其他端口
```

等待目标设备响应。

目标回复后，交换机会根据回复帧学习：

```text
CC-CC
=
port3
```

以后即可精准转发。

---

## 七、MAC 地址表为什么会老化

设备位置可能变化。

例如：

原来：

```text
服务器
↓
port1
```

迁移后：

```text
服务器
↓
port5
```

如果永久保存旧记录：

```text
MAC = port1
```

会导致转发错误。

所以 MAC 表具有：

```text
老化时间
```

长期没有通信：

```text
删除记录
```

下一次通信重新学习。

---

## 八、MAC 变化可能导致什么问题

例如：

```text
192.168.1.100
```

上午：

```text
MAC:
AA-AA
```

下午：

```text
MAC:
BB-BB
```

可能原因：

- IP 冲突
- 网卡更换
- 虚拟机迁移
- HA 主备切换
- 设备替换

不能直接判断一定是 IP 冲突。

需要结合：

```text
时间
交换机端口
设备日志
ARP
```

确认。

---

## 九、交换机 MAC 表在监控系统中的作用

监控系统链路：

```text
Collector
↓
交换网络
↓
目标设备
↓
SNMP Agent
↓
Response
```

当出现：

```text
设备 Offline
SNMP Timeout
```

不能直接认为：

```text
Java代码错误
SNMP配置错误
```

可能问题发生在：

```text
Collector
↓
交换机端口
↓
MAC转发
↓
VLAN
↓
设备
```

---

## 十、实际排障案例

### 场景一：交换机在线，但是服务器访问异常

现象：

```text
交换机管理正常
某服务器访问失败
```

排查：

```text
交换机
↓
查看端口状态
↓
查看 MAC 地址表
↓
确认服务器 MAC 是否存在
↓
确认 MAC 所在端口
```

---

### 场景二：服务器迁移后网络异常

服务器：

原来：

```text
port5
```

迁移：

```text
port20
```

检查：

```text
MAC Address Table
```

确认：

```text
MAC 是否已经学习到正确端口
```

---

### 场景三：大量设备同时 Offline

不要逐个检查：

```text
设备
SNMP
代码
```

应该寻找共同故障域：

```text
共同交换机
共同端口
共同 VLAN
共同链路
```

---

## 十一、和 Java 后端排障的联系

Java 报：

```text
Connection timeout
```

不一定是代码问题。

完整链路：

```text
Java
↓
Linux网络栈
↓
ip route
↓
ARP
↓
MAC
↓
交换机
↓
目标服务
```

可能失败的位置：

- 应用配置错误
- IP错误
- 路由错误
- ARP异常
- MAC转发异常
- VLAN错误
- 防火墙限制
- 服务未启动

排障需要先定位问题所在层。

---

## 十二、今日核心总结

记住三个模型：

### 模型一：MAC 学习

```text
看到源 MAC
↓
记录来源端口
```

### 模型二：MAC 转发

```text
目标 MAC 已知
↓
单播转发

目标 MAC 未知
↓
泛洪寻找
```

### 模型三：监控排障

看到：

```text
SNMP Timeout
设备 Offline
```

不要直接修改代码。

应该按照：

```text
应用
↓
SNMP
↓
IP/路由
↓
ARP
↓
MAC
↓
交换网络
↓
目标设备
```

逐层定位。

今天补全了网络链路中：

**交换机如何通过 MAC 地址表完成二层转发。**
