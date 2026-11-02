# 后端工程化每日训练 Day 53：Linux 服务器内存、MemAvailable、Cache、Swap 与 OOM 排障复盘

## 一、今天学习什么

今天继续沿着运维监控方向学习，只解决一个核心问题：

> **Linux 服务器的内存到底应该怎么看，以及看到“内存使用率 95%”以后，怎么判断它到底是真缺内存，还是大量内存只是被 Linux 拿去做了 Cache。**

今天重点建立了下面这条判断链：

```text
MemTotal
↓
MemFree
↓
MemAvailable
↓
Buffers / Cached
↓
Swap Used / Swap In / Swap Out
↓
进程 RSS
↓
Linux OOM / JVM OOM
```

今天最重要的第一个认识是：

```text
Free 很低
≠
服务器马上没内存

Used 很高
≠
一定发生内存故障
```

真正判断 Linux 服务器有没有物理内存压力，不能只盯着：

```text
Used
Free
```

而应该重点结合：

```text
MemAvailable
Cache
Swap
趋势
进程 RSS
系统性能
```

今天第二个重要认识是：

> **服务器物理内存和 JVM Heap 是两个不同层次的问题。**

可能出现：

```text
JVM Heap 很健康
但是整台 Linux 服务器已经快没内存
```

也可能出现：

```text
服务器还有大量可用 RAM
但是 Java 自己的 -Xmx 太小
最终发生 JVM Heap OOM
```

所以内存问题必须分层看。

---

# 二、先建立 Linux 内存的整体模型

假设一台 Linux 服务器有：

```text
16 GB RAM
```

这 16 GB 并不是全部都可以直接理解成：

```text
“给 Java 用的内存”
```

实际上它可能同时被下面这些东西使用：

```text
16 GB 物理内存
│
├── Java 服务
├── OceanBase
├── IoTDB
├── Pika
├── Collector / Agent
├── Linux 内核
├── Page Cache
├── Buffers
└── 真正完全空闲的内存
```

因此以后看到：

```text
服务器总内存 = 16 GB
```

不能直接认为：

```text
Java 最大可以使用 16 GB
```

因为操作系统、数据库、中间件、文件缓存和其他进程都需要共享这块物理内存。

今天顺便把一个基础概念彻底确认：

```text
RAM
=
物理内存
```

如果是在 VMware 里给 Linux 虚拟机分配：

```text
10 GB RAM
```

那么 Linux 看到的就是大约 10 GB 可管理的虚拟机物理内存。

---

# 三、MemFree：真正完全空着的内存

`MemFree` 可以先简单理解成：

> **当前完全没有被任何东西使用的物理内存。**

例如：

```text
MemTotal = 16 GB
MemFree  = 500 MB
```

它只说明：

```text
现在真正完全空着的 RAM
大约只有 500 MB
```

但它不能说明：

```text
系统最多只能再使用 500 MB
```

原因就在于 Linux 会把大量暂时不用的 RAM 主动拿去做缓存。

今天训练第一题：

```text
MemTotal     = 16 GB
MemFree      = 500 MB
MemAvailable = 6 GB
```

不能直接判断：

```text
“服务器只剩 500 MB，马上就要 OOM。”
```

因为真正更有价值的信息是：

```text
MemAvailable = 6 GB
```

这说明系统仍然大约还有 6 GB 内存，可以在不造成明显系统压力的情况下继续提供给应用使用。

所以以后先记住：

```text
MemFree 很低
≠
真正没有可用内存
```

---

# 四、Cache：Linux 会把闲置 RAM 转化成性能

Linux 并不追求：

```text
Free 越大越好
```

它的思路更接近：

```text
既然 RAM 暂时没人用
↓
不如拿来缓存经常访问的数据
↓
减少磁盘 IO
↓
提高系统性能
```

例如 Java 服务读取一个 1 GB 文件：

```text
磁盘
↓
Linux Page Cache
↓
Java 程序
```

Linux 可能会把读取过的文件内容保留在 Page Cache 中。

下一次程序再读取同样的数据时，就可能直接：

```text
RAM Cache
↓
应用
```

而不需要重新走慢得多的磁盘读取。

所以可能看到：

```text
程序真实占用：7 GB
Cache：8.5 GB
Free：500 MB
```

表面上：

```text
Free 只剩 500 MB
```

但如果应用突然还需要 2 GB 内存，Linux 可以回收一部分可回收 Cache，再把 RAM 交给应用。

因此：

> **Cache 不是“浪费掉的内存”，而是 Linux 利用闲置 RAM 提高 IO 性能的一种正常机制。**

今天训练第三题最终形成的判断就是：

> **Linux 会尽量把闲置内存转化成性能，而不是单纯追求 Free 数字越大越好。**

---

# 五、Buffers 是什么

今天又补充了一个容易和 Cache 混在一起的概念：

```text
Buffers
```

可以先理解成：

> **Linux 在 RAM 中为块设备、文件系统相关操作保留的一部分缓存。**

简单模型：

```text
程序产生磁盘相关操作
↓
Linux Buffer / Cache
↓
磁盘
```

现代 Linux 的 `free` 命令通常会把很多这类可回收缓存合并展示为：

```text
buff/cache
```

所以以后看到：

```text
buff/cache 很大
```

不能直接判断：

```text
“内存泄漏了。”
```

更合理的判断是：

```text
Linux 正在利用 RAM 做缓存
↓
先继续看 MemAvailable
↓
再判断有没有真实内存压力
```

这里还有一个今天纠正过的点：

> **正常情况下不要把“手动清空 Cache”当成处理 Linux 内存高的第一方案。**

因为应用真正需要内存时，Linux 自己会回收可回收缓存。

人为频繁清 Cache 反而可能让后续请求重新访问磁盘，降低 IO 性能。

---

# 六、MemAvailable：判断真实内存压力更重要的指标

今天最关键的指标是：

```text
MemAvailable
```

它大致回答的问题是：

> **如果现在应用还要继续申请内存，在不明显破坏系统正常运行的情况下，大约还能拿出多少物理内存。**

它不是只看：

```text
MemFree
```

还会综合考虑一部分：

```text
可回收 Page Cache
可回收内核内存
其他可以释放的内存
```

所以经常会出现：

```text
MemFree      = 500 MB
MemAvailable = 6 GB
```

这并不矛盾。

可以用一个简单思路理解服务器内存使用率：

```text
可用内存率
≈
MemAvailable / MemTotal × 100%
```

反过来：

```text
实际内存压力视角下的使用率
≈
1 - MemAvailable / MemTotal
```

但这里必须注意：

> **真正的监控平台到底使用什么公式，不能靠猜，必须追它实际的源码或模板计算表达式。**

因为不同平台可能使用不同算法。

例如：

```text
方案 A
used = total - free
```

这种算法会把大量 Cache 也算入已使用内存，最终看到的百分比可能很高。

也可能是：

```text
方案 B
usage = 1 - MemAvailable / MemTotal
```

还有一些旧式计算方式：

```text
方案 C
used = total - free - buffers - cache
```

所以页面显示：

```text
内存使用率 = 82%
```

第一反应不应该只是：

```text
“82% 很高还是不高？”
```

还应该继续问：

> **这个 82% 到底是怎么算出来的？**

---

# 七、Swap：磁盘上的备用交换空间，不是真正的 RAM

`Swap` 可以先简单理解成：

> **当物理内存紧张时，Linux 可以把一部分暂时不活跃的内存页放到磁盘上的交换空间，从而释放 RAM。**

例如虚拟机：

```text
RAM  = 10 GB
Swap = 4 GB
```

并不等于：

```text
真正拥有 14 GB RAM
```

而是：

```text
10 GB RAM
=
真正的内存，速度快

4 GB Swap
=
磁盘上的交换空间，速度慢得多
```

在 VMware 场景下，Linux 的 Swap 位于 Linux 自己的虚拟磁盘中，而虚拟磁盘文件最终又可能存放在宿主机 Windows 的 C 盘或 D 盘。

但不能理解成：

```text
“Linux 直接把整个内存复制到 C 盘 / D 盘。”
```

更准确的是：

```text
RAM 紧张
↓
选择暂时不活跃的内存页
↓
换出到 Swap
↓
释放 RAM
```

等程序以后又需要这些数据：

```text
Swap
↓
重新读回 RAM
↓
程序继续访问
```

问题在于：

```text
RAM
远快于
磁盘
```

所以如果系统开始频繁：

```text
RAM → Swap
Swap → RAM
RAM → Swap
Swap → RAM
```

性能会明显下降。

可能表现为：

```text
Java 接口 RT 上升
磁盘 IO 增加
Load 上升
系统整体变慢
应用像“卡住”一样
```

因此：

```text
Swap Used > 0
```

本身不一定立即等于故障。

真正更危险的是：

```text
MemAvailable 很低
+
Swap Used 持续增长
+
Swap In / Swap Out 持续活跃
+
业务性能下降
```

这几个信号组合在一起。

---

# 八、训练中的典型场景判断

## 1. Used 95%，但 Available 还有 35%

场景：

```text
Used = 95%
MemAvailable = 35%
Swap = 0
```

不能仅凭：

```text
Used = 95%
```

直接判断严重内存故障。

因为：

```text
MemAvailable = 35%
```

表示仍有较明显的可用物理内存空间。

同时：

```text
Swap = 0
```

说明当前至少没有看到明显依赖 Swap 缓解物理内存压力的迹象。

今天这里纠正了一个表达：

```text
MemAvailable = 35%
```

不是：

```text
“MEM / VM 占用了 35%”
```

而应该理解成：

```text
大约还有 35% 的物理内存可供系统继续使用
```

---

## 2. Available 持续下降

如果看到：

```text
12:00  Available 40%
13:00  Available 30%
14:00  Available 20%
15:00  Available 10%
16:00  Available 5%
```

这比某一个时间点看到：

```text
Used = 95%
```

更值得关注。

因为它说明：

```text
真正还能继续分配的物理内存
正在持续减少
```

下一步应该重点追：

```text
哪个进程的 RSS 在持续上涨？
```

而不是第一时间重启 Java。

---

## 3. Available 下降 + Swap 增长 + Java 变慢

训练第四题：

```text
MemAvailable
40% → 30% → 20% → 10% → 5%

Swap Used
0 → 1 GB → 3 GB

同时
Java 接口越来越慢
```

这个组合已经高度提示：

> **整台 Linux 服务器正在出现越来越明显的物理内存压力。**

链路可以理解成：

```text
真正可用 RAM 持续下降
↓
Linux 开始把不活跃内存页换到 Swap
↓
Swap 使用增加
↓
磁盘 IO 压力增加
↓
程序访问换出数据时需要重新换入
↓
接口 RT 上升
```

下一步重点看：

```text
MemAvailable 趋势
Swap Used 趋势
Swap In / Swap Out
各进程 RSS
磁盘 IO
Load
CPU
```

---

## 4. Host 内存 96%，JVM Heap 只有 2 GB

场景：

```text
服务器总内存 = 16 GB
监控平台内存使用率 = 96%
JVM Heap = 2 GB
```

不能直接判断：

```text
Java 内存泄漏
```

因为其他内存消费者可能包括：

```text
OceanBase
IoTDB
Pika
其他 Java 进程
Linux 内核
Page Cache
其他系统进程
```

而且今天还补充了一个必须分清的概念：

```text
JVM Heap = 2 GB
≠
整个 Java 进程只占 2 GB
```

Java 进程还可能包含：

```text
Heap
Metaspace
Direct Memory
Thread Stack
Code Cache
其他 Native Memory
```

所以正确顺序应该是：

```text
服务器内存高
↓
先找哪个进程 RSS 最大
↓
如果确认 Java 是主要消费者
↓
再进入 JVM 层继续拆
```

---

# 九、Linux OOM 和 JVM Heap OOM 必须分清楚

这是今天和 Day 41 联系最紧密的一块。

Day 41 学的是：

```text
Java JVM 内存
Heap
Metaspace
Direct Memory
GC
JVM OOM
```

今天学的是：

```text
整台 Linux 服务器物理内存
```

两者不是一个层次。

## JVM Heap OOM

典型情况：

```text
Java Heap 达到上限
↓
JVM 无法继续分配对象
↓
出现
java.lang.OutOfMemoryError
```

这是 Java/JVM 自己发现内存分配失败。

## Linux 系统 OOM

如果整台服务器物理内存和可用 Swap 都无法继续支撑系统分配，Linux 内核可能触发：

```text
OOM Killer
```

然后选择某个进程杀掉。

训练第六题：

```text
Java 服务突然消失

Java 日志没有：
java.lang.OutOfMemoryError

Linux 系统日志却出现：
Out of memory
Killed process xxxx (java)
```

这里应该判断：

> **这是 Linux 系统层面的 OOM Killer 杀掉了 Java，而不是 JVM Heap OOM。**

关键证据是：

```text
Linux 内核日志明确出现
Out of memory
Killed process xxxx (java)
```

而不是单纯依赖：

```text
“Java 日志里没看到 OOM。”
```

因为 Linux 可以直接把 Java 进程杀掉，JVM 甚至可能没有机会自己输出 `OutOfMemoryError`。

所以以后看到：

```text
Java 进程突然没了
```

不能只查 Java 日志。

还应该查操作系统层：

```bash
dmesg -T | grep -i -E 'out of memory|killed process|oom'
```

或者在使用 systemd 的系统上继续查内核日志。

---

# 十、Linux 内存告警的分层排查顺序

以后看到：

```text
服务器内存使用率 = 96%
```

不要第一时间：

```text
重启 Java
```

今天形成的排查顺序是：

## 第一层：确认监控指标本身

先问：

```text
MemTotal 是多少？
MemAvailable 是多少？
平台的 96% 到底怎么算出来的？
```

因为如果公式本身使用：

```text
total - free
```

大量 Cache 也可能被算进“已使用”。

---

## 第二层：判断 Cache / Buffer

继续看：

```text
Cached
Buffers
buff/cache
```

判断高 Used 是因为：

```text
大量可回收 Cache
```

还是：

```text
MemAvailable 真的已经很低
```

---

## 第三层：看趋势

不要只看一个点。

例如：

```text
Available 一直稳定 8%
```

和：

```text
40%
↓
30%
↓
20%
↓
8%
```

风险完全不同。

监控判断应该尽量结合：

```text
当前值
+
持续时间
+
趋势
```

---

## 第四层：看 Swap

检查：

```text
Swap Used
Swap In
Swap Out
```

重点判断：

```text
只是历史上用过一点 Swap
```

还是：

```text
现在正在持续发生频繁换入换出
```

---

## 第五层：找真正的内存消费者

例如：

```bash
ps aux --sort=-%mem | head
```

或者：

```bash
top
```

重点关注：

```text
RSS
```

然后判断到底是谁在吃内存：

```text
Java
OceanBase
IoTDB
Pika
Collector
其他进程
```

---

## 第六层：进入具体进程

如果确认是 Java：

```text
Java
↓
Heap
Metaspace
Direct Memory
Native Memory
Thread Stack
GC
```

如果确认是数据库：

```text
数据库
↓
内存配置
Buffer / Cache
连接数
查询负载
```

---

## 第七层：查系统层异常

继续确认：

```text
是否触发 OOM Killer
磁盘 IO 是否异常
Load 是否异常
CPU 是否异常
```

---

## 第八层：反查监控链路

如果系统本身数据显示正常，但监控平台明显异常，还要继续查：

```text
Agent 原始数据
字段解析
单位换算
使用率公式
上报数据
存储数据
后端接口
前端展示
```

今天最后形成的排障原则是：

> **先确认“服务器到底是不是真的缺内存”，再确认“是谁在占内存”，最后才进入具体应用排查。**

---

# 十一、Linux 服务器内存指标是怎么进入监控平台的

今天还把 Linux 内存和前面交换机 SNMP 采集做了区分。

对于交换机，常见链路可能是：

```text
Collector
↓
SNMP GET
↓
OID
↓
设备 SNMP Agent
↓
返回原始值
```

但是 Linux 服务器如果安装了平台自己的 Agent，更常见的链路是：

```text
Linux
↓
Agent 本机读取系统指标
↓
/proc/meminfo
↓
解析字段
↓
单位转换
↓
计算内存使用率
↓
上报 Collector / Server
↓
时序数据库 / 数据库
↓
后端接口
↓
前端显示
```

今天训练第七题一开始把 Linux 内存采集链路和 SNMP OID 混在了一起。

这里需要明确纠正：

> **Linux 服务器内存并不应该默认套用“SNMP Agent + OID”的交换机采集模型。真实项目里更可能由 Agent 直接读取 `/proc/meminfo` 或系统 API。**

当然，最终仍然必须以当前项目源码为准。

常见 `/proc/meminfo` 字段包括：

```text
MemTotal
MemFree
MemAvailable
Buffers
Cached
SwapTotal
SwapFree
```

可以直接查看：

```bash
cat /proc/meminfo
```

或者过滤关键字段：

```bash
grep -E 'MemTotal|MemFree|MemAvailable|Buffers|Cached|SwapTotal|SwapFree' /proc/meminfo
```

常用的整体查看命令：

```bash
free -h
```

需要观察 Swap 换入换出时还可以继续看：

```bash
vmstat 1
```

---

# 十二、如果页面显示“内存使用率 82%”，源码应该怎么追

今天最有工作价值的一部分，不是背一个固定公式，而是：

> **学会追这个 82% 是怎么来的。**

优先可以让 Codex 搜：

```text
/proc/meminfo
MemTotal
MemAvailable
MemFree
Cached
Buffers
memoryUsage
usedMemory
availableMemory
swap
memory
```

然后不要停留在：

```text
“我找到了 memoryUsage 字段。”
```

而是继续把整条真实源码链路映射出来：

```text
Linux 原始数据
↓
Agent 哪个类读取
↓
哪个类解析 MemTotal / MemAvailable 等字段
↓
KB / MB / GB 怎么转换
↓
内存使用率公式到底是什么
↓
哪个 DTO / VO 上报
↓
Collector / Server 哪个入口接收
↓
存到哪里
↓
哪个后端接口查询
↓
前端哪个字段显示成 82%
```

真正需要 Codex 最终交付的是：

```text
关键类
关键方法
关键字段
调用顺序
计算公式
```

而不是只给一个概念解释。

今天这里还形成了一个很重要的源码阅读原则：

> **不要先假设“内存使用率一定等于 (Total - Free) / Total”，再去源码里寻找证明。应该让源码告诉你平台真正使用了什么公式。**

---

# 十三、和当前监控工作的直接关系

## 1. 编写服务器内存监控模板

以后配置 Linux 内存告警时，不能机械地写：

```text
MemFree < 10%
→ 严重告警
```

应该先确认模板使用的到底是：

```text
MemAvailable
MemFree
Used - Cache
还是其他平台自定义公式
```

否则 Linux 正常利用 Cache，都可能被误报成“内存不足”。

---

## 2. 页面看到 95%，先追公式

以后页面出现：

```text
内存使用率 = 95%
```

应该立即产生第二个问题：

```text
这个 95% 到底怎么算出来的？
```

然后去找：

```text
原始监控项
↓
计算表达式
↓
MemAvailable / MemFree / Cache
```

---

## 3. 平台数据和 Linux 本机不一致

例如：

```text
Linux free 命令：
Available = 6 GB

监控平台：
可用内存 = 600 MB
```

这时候不能马上怀疑服务器。

应该沿监控链路排：

```text
Agent 读取是否正确
↓
字段解析是否正确
↓
KB / MB / GB 单位是否搞错
↓
计算公式是否正确
↓
数据库存储是否正确
↓
后端返回是否正确
↓
前端展示是否正确
```

这和前面学习 SNMP、Counter64、自动发现时形成的数据链路思维是一样的：

```text
原始指标
↓
采集
↓
解析
↓
转换
↓
存储
↓
展示
↓
告警
```

采集方式不同，但监控系统的数据处理思想是相通的。

---

# 十四、今天训练中需要长期记住的几个纠正

今天不是只学新概念，还纠正了几个很容易在工作中误判的问题。

## 纠正 1

```text
MemFree 很低
≠
马上 OOM
```

优先结合：

```text
MemAvailable
```

判断真实内存空间。

## 纠正 2

```text
Used = 95%
≠
严重内存故障
```

还要看：

```text
Available
Cache
Swap
趋势
性能
```

## 纠正 3

```text
Cache 很大
≠
内存泄漏
```

Linux 会主动利用闲置 RAM 做缓存。

## 纠正 4

```text
手动清 Cache
```

不应该成为内存高时的默认处理动作。

Linux 本身就具备缓存回收机制。

## 纠正 5

```text
Swap Used > 0
≠
服务器已经故障
```

真正需要警惕的是：

```text
Available 很低
+
Swap 持续增长
+
Swap In / Out 活跃
+
服务性能下降
```

## 纠正 6

```text
Host Memory 高
≠
JVM Heap 高
```

必须先区分：

```text
服务器层
Java 进程层
JVM Heap 层
```

## 纠正 7

```text
Java 进程消失
≠
一定 JVM Heap OOM
```

如果 Linux 日志明确出现：

```text
Out of memory
Killed process xxxx (java)
```

就要优先判断 Linux OOM Killer。

## 纠正 8

```text
Linux 内存采集
≠
默认 SNMP OID 链路
```

Linux 服务器更常见的是：

```text
Agent
↓
/proc/meminfo
```

交换机才更常见：

```text
SNMP GET
↓
OID
```

---

# 十五、最终总结

今天真正要留下来的不是几个字段名称，而是一套 Linux 内存判断模型。

以后看到：

```text
服务器内存 95%
```

脑子里应该自动展开成：

```text
第一步
这个 95% 的公式是什么？

第二步
MemAvailable 还有多少？

第三步
Cache / Buffer 占多少？

第四步
Available 是稳定还是持续下降？

第五步
Swap 有没有持续增长？
Swap In / Out 是否活跃？

第六步
哪个进程 RSS 最大？

第七步
如果是 Java，再进入 JVM Heap / Native Memory 等层次

第八步
有没有 Linux OOM Killer 记录？

第九步
如果本机正常、平台异常，再反查 Agent → 解析 → 单位 → 公式 → 存储 → 前端
```

今天最核心的几句话：

> **Free 很低，不等于 Linux 马上没内存。**

> **判断真实物理内存压力，MemAvailable 通常比单独的 MemFree 更有意义。**

> **Cache 是 Linux 把闲置 RAM 转化成 IO 性能的正常机制。**

> **Swap 是磁盘上的交换空间，是缓解内存压力的保险，不是真正的 RAM。**

> **Available 持续下降 + Swap 持续增长 + Swap In/Out 活跃 + 服务变慢，才是非常值得警惕的组合。**

> **Host Memory、Java 进程内存、JVM Heap 必须分层看。**

> **先确认服务器真的缺内存，再找谁在占内存，最后才进入具体应用排查。**

这套判断以后可以直接用于：

```text
服务器监控模板配置
Linux 内存告警判断
Java 服务异常排查
Agent 采集链路排查
监控平台数据校验
源码链路追踪
```
