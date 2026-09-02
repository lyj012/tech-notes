# 后端工程化每日训练 Day 55：Linux 磁盘容量与磁盘 IO 监控原理复盘

## 一、今天学习什么

今天学习 Linux 磁盘监控中的两个核心问题：

- 磁盘容量不足
- 磁盘 IO 性能不足

这两个问题经常被混淆，但本质完全不同。

```text
磁盘容量问题
=
还能不能继续存数据

磁盘 IO 性能问题
=
读写数据是否足够快
```

排查磁盘异常时，需要先判断：

```text
磁盘异常
↓
容量问题 or 性能问题
↓
容量：Used / Available
↓
性能：IOPS / 吞吐量 / await / 队列
↓
定位产生 IO 的进程
↓
进入 Java、数据库、IoTDB 等应用
```

---

## 二、磁盘容量监控

### 1. df -h 查看磁盘空间

Linux 常用命令：

```bash
df -h
```

示例：

```text
Filesystem   Size   Used   Avail   Use%
/dev/sda1    100G    90G    10G    90%
```

含义：

- Size：总容量
- Used：已使用空间
- Avail：剩余空间
- Use%：容量使用率

这个指标回答：

> 文件系统还有多少空间可以继续写入？

注意：

```text
磁盘容量使用率 90%
≠
磁盘 IO 性能差
```

---

### 2. 磁盘满会发生什么

当磁盘达到 100%：

```text
No space left on device
```

可能影响：

- Java 日志无法写入
- 数据库无法新增数据
- IoTDB 无法保存时序数据
- 临时文件创建失败
- 上传文件失败

所以容量监控属于基础监控指标。

---

## 三、磁盘 IO 性能指标

磁盘空间足够，不代表磁盘一定正常。

例如：

```text
磁盘使用率 = 30%
await = 180ms
iowait = 45%
```

说明：

虽然空间很多，但是磁盘处理 IO 请求速度很慢。

---

## 四、IOPS 和吞吐量

### 1. IOPS

IOPS：

```text
Input / Output Operations Per Second
```

表示：

> 磁盘每秒处理多少次 IO 请求。

关注：

```text
次数
```

不是数据大小。

例如：

```text
IOPS = 10000
```

表示每秒完成 10000 次 IO 操作。

---

### 2. MB/s（吞吐量）

MB/s 表示：

> 每秒传输了多少数据。

例如：

10:00：

```text
累计写入 = 100GB
```

10:01：

```text
累计写入 = 106GB
```

计算：

```text
(106GB - 100GB) / 60s
≈ 100MB/s
```

---

### 3. 为什么 IOPS 高不代表吞吐量高

因为单次 IO 大小不同。

例如：

小文件：

```text
10000 次 IO
每次 4KB
```

IOPS 很高，但是吞吐量可能只有几十 MB/s。

大文件：

```text
100 次 IO
每次 10MB
```

IOPS 很低，但是吞吐量可能很高。

关系：

```text
吞吐量 ≈ IOPS × 单次 IO 大小
```

---

## 五、await 与 iowait

### 1. await

await 表示：

> 一个 IO 请求从提交到完成平均耗时。

单位通常是：

```text
ms
```

例如：

```text
await = 2ms
```

说明 IO 响应较快。

如果：

```text
await = 250ms
```

说明请求等待明显增加。

---

### 2. iowait

iowait 表示：

> CPU 时间中等待 IO 完成的比例。

例如：

```text
Java 请求
↓
查询数据库
↓
等待磁盘读取
↓
线程阻塞
↓
iowait 升高
```

所以：

```text
iowait 高
≠
CPU 算力不足
```

需要继续检查磁盘 IO。

---

## 六、Linux 磁盘数据采集原理

磁盘性能指标通常不是 Linux 直接返回的。

常见来源：

```text
/proc/diskstats
```

里面保存累计数据：

- 读次数
- 写次数
- 读取字节数
- 写入字节数
- IO 时间

类似 Counter64：

```text
累计值
↓
两次采样
↓
delta 差值
↓
除以时间间隔
↓
得到速率
```

例如：

第一次：

```text
writeBytes = 100GB
```

第二次：

```text
writeBytes = 106GB
```

计算：

```text
6GB / 60s
=
100MB/s
```

Write IOPS 同理：

```text
写操作次数增量 / 时间间隔
```

---

## 七、监控系统中的磁盘数据链路

追踪：

> Linux 磁盘写入速率 50MB/s 是怎么显示到前端的？

完整链路：

```text
Linux
↓
/proc/diskstats
↓
Agent采集
↓
读取累计 IO 数据
↓
保存历史采样值
↓
第二次采样
↓
delta计算
↓
计算 Write MB/s / Write IOPS
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

进入 Java Agent 项目追磁盘采集逻辑，可以搜索：

```text
/proc/diskstats
diskstats
disk
readBytes
writeBytes
readOps
writeOps
iops
await
ioTime
delta
Collector
Metric
```

排查顺序：

```text
采集入口
↓
Linux 原始数据来源
↓
累计值保存
↓
两次采样计算
↓
指标转换
↓
上报
↓
存储
↓
前端接口
↓
页面展示
```

---

## 九、典型排查场景

### 场景 1：容量问题

```text
磁盘使用率 = 95%
await = 2ms
iowait = 1%
```

判断：

```text
容量风险
```

排查：

```text
df -h
↓
查看大目录
↓
定位日志、数据库、临时文件
```

---

### 场景 2：IO 性能问题

```text
磁盘使用率 = 30%
await = 180ms
iowait = 45%
```

判断：

```text
磁盘 IO 瓶颈
```

排查：

```text
iostat
↓
查看磁盘压力
↓
iotop
↓
定位产生 IO 的进程
↓
进入应用分析
```

---

## 十、今天总结

今天核心理解：

```text
磁盘监控
=
容量监控 + 性能监控
```

容量：

```text
df -h
回答：还能不能存
```

性能：

```text
IOPS
吞吐量
await
队列
回答：读写是否够快
```

最终排查思路：

```text
看到磁盘异常
↓
先区分容量还是性能
↓
容量看空间
↓
性能看 IO 指标
↓
定位进程
↓
进入具体应用
```

监控指标不是最终答案，而是定位问题的入口。
