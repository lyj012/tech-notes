# 后端工程化每日训练 Day 54：Linux CPU 使用率、Load Average 与 iowait 原理复盘

## 一、今天学习什么

今天学习 Linux 服务器 CPU 指标如何判断。

核心知识点：

- CPU 使用率高不代表一定 CPU 不够
- Load Average 高不代表一定 CPU 算力不足
- iowait 高不代表 CPU 自己很忙

排查 CPU 异常时，需要先回答：

> CPU 时间到底消耗在哪里？系统任务为什么排队？真正瓶颈是计算还是 IO？

---

## 二、Linux CPU 时间分类

Linux CPU 时间常见包括：

```text
user
system
idle
iowait
```

### user

用户态程序消耗的 CPU 时间。

例如：

- Java 业务代码
- 大量计算
- JSON 编解码
- 压缩解压
- 加密运算

如果：

```text
CPU 高
user 高
iowait 低
```

通常更像 CPU 计算压力。

---

### system

CPU 执行 Linux 内核代码的时间。

例如：

- 系统调用
- 网络包处理
- 上下文切换
- 内核操作

如果 system 异常升高，需要关注系统层行为，而不是只看业务代码。

---

### iowait

iowait 表示：

> CPU 没有进行大量计算，但是系统存在任务等待 IO 完成。

例如：

```text
Java 请求
↓
查询数据库
↓
磁盘读取数据慢
↓
线程等待 IO
↓
CPU 没有继续执行计算
↓
iowait 升高
```

因此：

```text
iowait 高
≠
CPU 算力不足
```

更应该关注：

- 磁盘性能
- 存储延迟
- IO 队列
- 数据库读写压力

---

## 三、CPU 使用率如何计算

CPU 使用率不是读取一次得到的。

Linux `/proc/stat` 保存的是累计 CPU 时间：

```text
cpu user nice system idle iowait irq softirq ...
```

这些数据类似 Counter64：

```text
累计值
↓
两次采样
↓
delta 差值
↓
计算时间段内比例
```

例如：

第一次：

```text
total = 10000
idle = 6000
```

第二次：

```text
total = 11000
idle = 6200
```

计算：

```text
总 CPU 时间增加 = 1000
空闲时间增加 = 200

使用时间 = 1000 - 200 = 800

CPU 使用率 = 800 / 1000 * 100%
```

所以 CPU 使用率本质是：

> 某个时间窗口内 CPU 工作时间占比。

---

## 四、Load Average 原理

Linux Load Average：

```text
load average: 1.20, 2.50, 3.10
```

分别表示：

- 1 分钟平均负载
- 5 分钟平均负载
- 15 分钟平均负载

Load 统计：

```text
正在运行任务
+
不可中断等待任务
```

例如等待磁盘 IO 的任务，也可能影响 Load。

所以：

```text
Load 高
≠
CPU 使用率高
```

---

## 五、Load 必须结合 CPU 核数判断

例如：

服务器 A：

```text
2 核 CPU
Load = 6
```

服务器 B：

```text
32 核 CPU
Load = 6
```

两个 Load 数值相同，但是压力完全不同。

粗略理解：

```text
Load / CPU 核数
```

越高，资源竞争越明显。

---

## 六、典型故障场景分析

### 场景 1：CPU 95%，user 85%

```text
CPU = 90%
user = 85%
system = 3%
iowait = 1%
```

判断：

```text
CPU 计算压力
```

排查：

```text
top
↓
找到高 CPU 进程
↓
查看线程
↓
Java Thread Dump
↓
定位代码
```

---

### 场景 2：CPU 65%，iowait 50%

```text
CPU = 65%
user = 8%
system = 7%
iowait = 50%
```

不要优先增加 CPU。

原因：

CPU 并没有大量用于计算。

应该检查：

```text
iostat -x
↓
磁盘 util
await
读写压力

↓

找到产生 IO 的进程
```

工具：

```text
iotop
```

---

### 场景 3：CPU 不高，但是 Load 很高

例如：

```text
8 核服务器
CPU = 30%
Load = 20
iowait = 45%
```

原因：

```text
大量任务等待 IO
↓
进入不可中断等待状态
↓
影响 Load
```

瓶颈更可能：

```text
磁盘 / 存储 IO
```

---

## 七、监控系统中的 CPU 数据链路

追踪：

> Linux CPU 82%、user 60%、iowait 10% 是如何显示到前端的？

完整链路：

```text
Linux
↓
/proc/stat
↓
Agent 读取
↓
保存历史采样值
↓
第二次采样
↓
delta 计算
↓
计算 user/system/iowait 百分比
↓
上报
↓
后端接收
↓
数据库存储
↓
前端展示
```

---

## 八、源码排查关键词

如果进入 Java Agent 项目追 CPU 采集逻辑，可以搜索：

```text
/proc/stat
cpu
user
system
idle
iowait
cpuUsage
previous
current
delta
loadavg
Collector
Metric
```

重点关注：

```text
第一次采样值保存在哪里
第二次采样在哪里获取
差值如何计算
百分比公式在哪里
数据如何上报
```

---

## 九、今天总结

今天核心不是记 CPU 命令，而是建立监控排查思维：

```text
看到 CPU 异常
↓
不要直接判断 CPU 不够
↓
拆分 user/system/iowait
↓
结合 Load 和 CPU 核数
↓
判断计算瓶颈还是 IO 瓶颈
↓
继续追进程和代码
```

核心认知：

> 监控指标不是答案，而是定位问题的入口。

CPU、Load、iowait 三个指标必须结合分析，才能判断真实瓶颈。
