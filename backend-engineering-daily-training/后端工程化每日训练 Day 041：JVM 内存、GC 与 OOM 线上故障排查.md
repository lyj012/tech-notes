# 后端工程化每日训练 Day 41：JVM 内存、GC 与 OOM 线上故障排查

## 一、今天学习的知识点

今天学习的是：

**JVM 内存、GC、内存泄漏、Java Heap OOM、进程内存、容器 OOMKilled，以及 Java 服务内存异常时的线上排查思路。**

一句话理解：

> JVM 内存问题的核心不是只背“堆、栈、元空间”这些概念，而是 Java 服务出现内存上涨、频繁 Full GC、CPU 升高、接口 RT 变慢甚至进程被杀时，能够逐步判断到底是哪类资源出了问题，并最终追到具体对象、引用链和代码。

今天一开始看这一块会比较陌生，因为它同时涉及：

```text
Heap
GC
OOM
Metaspace
Thread Stack
Direct Memory
Native Memory
OOMKilled
Heap Dump
```

但训练以后，最核心的主线已经可以压缩成：

```text
内存高
↓
先看 GC 后能不能降下来
↓
如果 GC 越来越频繁却回收不动
↓
怀疑大量对象长期存活
↓
继续找哪些对象占内存
↓
找谁在引用这些对象
↓
最终定位具体代码
```

今天最重要的三个工程判断是：

```text
内存高 ≠ 内存泄漏

-Xmx 只限制 Heap，不限制整个 Java 进程

OOMKilled 不等于 Java Heap OOM
```

---

## 二、Heap、GC 与内存泄漏

### 1. Heap 是什么

普通 Java 对象主要创建在 Heap 中，例如：

```java
new User();
new Order();
new ArrayList<>();
```

可以先简单理解为：

```text
Java 不断创建对象
↓
对象占用 Heap
```

`-Xmx` 控制的主要就是 Java Heap 的最大值。

例如：

```text
-Xmx4G
```

表示 JVM Heap 最大大约允许到 4G。

它并不是：

```text
整个 Java 进程最多只能占 4G
```

这个区别在今天后面的 Kubernetes OOMKilled 场景里非常重要。

---

### 2. GC 是什么

Java 对象不再被需要，并且已经没有有效引用能够到达它时，GC 才有机会把这部分内存回收。

可以先记成：

```text
创建对象
↓
占用 Heap
↓
对象不再被引用
↓
GC 判断对象不可达
↓
回收内存
```

所以 GC 的作用不是“看到内存高就无条件删对象”。

它必须先判断：

```text
这个对象是不是还活着？
还有没有地方引用它？
```

只要对象仍然可以通过引用链访问到，GC 就不能随便回收。

---

### 3. 什么是真正的内存泄漏

今天一个需要记住的定义是：

> **内存泄漏不是 GC 坏了，而是程序仍然保留着本来已经不需要的对象引用。**

例如：

```java
private static final List<Order> LIST = new ArrayList<>();

public void process(Order order) {
    LIST.add(order);
}
```

如果每个请求都：

```text
LIST.add(order)
```

但从来不：

```text
remove
clear
```

那么这些 `Order` 即使已经没有业务价值，仍然被 `LIST` 引用。

对于 GC 来说：

```text
LIST
↓
仍然引用 Order
↓
Order 仍然可达
↓
不能回收
```

长期运行以后可能变成：

```text
Heap
1G
↓
2G
↓
3G
↓
4G
↓
Java heap space
```

今天练习里一开始把它说成了“GC 泄漏”，这里要纠正：

```text
正确术语：内存泄漏
```

另外，不能简单理解成：

```text
用了 static 就一定内存泄漏
```

真正的问题是：

```text
长期存在的引用
+
持续保存对象
+
对象业务上已经没用
+
引用却一直不释放
```

`static List`、`static Map` 只是很典型的一种场景。

---

## 三、为什么 Heap 使用率 90% 不能直接判断内存泄漏

今天第一题：

```text
线上 Java 服务
Heap 使用率 = 90%
```

能不能直接判断发生内存泄漏？

结论：

```text
不能
```

因为高内存可能只是系统刚刚处理过一次大批量任务，短时间创建了很多临时对象。

例如：

```text
GC 前：3.2G
↓
GC 后：800M
```

这种情况说明：

```text
大量对象只是正常的临时对象
GC 可以有效回收
```

所以：

```text
Heap 使用率高
```

只是当前现象，不等于：

```text
内存泄漏
```

真正更应该关注的是：

> **GC 以后到底还能剩多少内存。**

---

## 四、判断内存泄漏时为什么要看“GC 后基线”

今天第二题：

两个服务 GC 前都是：

```text
3.8G
```

服务 A：

```text
GC 后：700M
```

服务 B：

```text
GC 后：3.5G
```

更值得怀疑的是：

```text
服务 B
```

原因：

### 服务 A

```text
3.8G
↓ GC
700M
```

说明 GC 回收掉了大量对象。

这些对象大概率只是正常的临时对象。

### 服务 B

```text
3.8G
↓ GC
3.5G
```

说明 GC 已经执行了，但绝大多数对象仍然存活。

这意味着：

```text
大量对象仍然被引用
```

因此 B 更值得怀疑内存泄漏。

但这里还要保留一个边界：

> **一次 GC 后剩 3.5G，也不能 100% 证明一定发生了内存泄漏。**

更有说服力的是连续趋势：

```text
第一次 GC 后：800M
第二次 GC 后：1.2G
第三次 GC 后：1.8G
第四次 GC 后：2.5G
```

如果：

```text
GC 后存活内存基线
持续上涨
```

才越来越像：

```text
长期存活对象不断增加
↓
对象无法释放
↓
疑似内存泄漏
```

今天可以把这一点记成：

> **不要只看 GC 前多高，更要看 GC 后能降到多低。**

---

## 五、Java heap space 不一定只有一种原因

今天第三题看到日志：

```text
java.lang.OutOfMemoryError: Java heap space
```

第一层含义是：

> **Java Heap 已经无法继续为新的 Java 对象分配足够空间。**

但原因不能只回答：

```text
堆太小
```

至少要想到两类常见情况。

### 1. 内存泄漏

例如：

```java
private static final List<Order> LIST = new ArrayList<>();
```

不断：

```java
LIST.add(order);
```

但对象一直不释放。

结果：

```text
对象越来越多
↓
GC 回收不了
↓
Heap 越来越满
↓
最终 OOM
```

这种更像：

```text
慢慢积累
↓
最后撑爆
```

---

### 2. 一次性加载的数据太大

例如：

```java
List<Order> list = orderMapper.selectAll();
```

数据库里有：

```text
300 万 / 500 万条订单
```

于是一次性构造几百万个 Java 对象：

```text
MySQL
↓
几百万行数据
↓
几百万个 Order
↓
全部进入 Heap
↓
瞬间 OOM
```

这里完全可能：

```text
没有内存泄漏
```

只是：

```text
瞬时内存峰值太大
```

所以今天要区分：

```text
内存泄漏
→ 对象长期被引用，慢慢积累

一次加载海量数据
→ 瞬时创建太多对象，把 Heap 直接撑爆
```

---

## 六、大批量数据为什么优先分批，而不是只增大 -Xmx

今天第五题：

线上一次查询：

```text
300 万条订单
```

全部装入：

```java
List<Order>
```

最后 Heap OOM。

两个方案：

### 方案 A

```text
-Xmx4G → -Xmx16G
```

### 方案 B

```text
每次 5000 条
分批读取
分批处理
```

今天选择的是：

```text
方案 B
```

这个判断是对的。

但训练里有两个表达需要修正。

### 修正 1：这不叫“降低内存泄漏”

这个案例的问题不一定是泄漏，而是：

```text
一次加载的数据量过大
↓
内存峰值过高
```

分批以后从：

```text
300 万条对象同时存在
```

变成：

```text
每批大约 5000 条对象
```

核心收益是：

> **降低内存峰值，让 JVM 的内存占用更可控。**

---

### 修正 2：不能说分批处理一定更快

分批以后需要：

```text
多次查询
多次循环
多次处理
```

所以纯执行时间不一定更短。

它真正的工程优势是：

```text
内存占用可控
OOM 风险更低
故障影响更小
服务稳定性更好
```

因此不能简单追求：

```text
一次全部加载最快
```

而应该考虑：

```text
系统能不能稳定完成任务
```

---

### 为什么只扩大 Heap 通常不是根治

如果本来：

```text
-Xmx4G
```

因为错误的数据处理方式导致 OOM。

改成：

```text
-Xmx16G
```

可能只是把故障推迟。

例如真正存在内存泄漏时：

```text
4G
→ 4 天挂一次

16G
→ 更久以后再挂
```

根因仍然存在。

所以：

> **扩大 JVM Heap 可以作为临时止血手段，但不能替代对业务代码和对象生命周期的分析。**

---

## 七、Full GC 风暴为什么会把接口拖慢

今天第六题给出的现象：

```text
Full GC 每 2 秒一次
CPU = 95%
接口 RT：100ms → 5s
```

而且每次 Full GC：

```text
3.9G → 3.7G
```

首先应该怀疑：

```text
大量对象长期存活
↓
GC 无法有效回收
↓
可能存在内存泄漏
```

同时服务已经非常像进入：

```text
GC Thrashing
GC 风暴
```

完整链路：

```text
Heap 快满
↓
触发 Full GC
↓
3.9G 只能回收到 3.7G
↓
几乎没释放多少
↓
Heap 很快再次不足
↓
继续 Full GC
↓
Full GC 越来越频繁
↓
大量 CPU 时间被 GC 占用
↓
业务线程获得的 CPU 资源减少
↓
请求开始堆积
↓
接口 RT 飙升
```

今天已经能够判断出：

```text
GC 回收不掉
→ 不断继续 GC
→ CPU 被大量消耗
→ 业务请求变慢
```

这里需要纠正一个说法：

```text
“这基本上就是 OOM”
```

还不能直接这么说。

因为题目目前只出现：

```text
频繁 Full GC
CPU 高
RT 高
GC 回收效果很差
```

但还没有真正看到：

```text
java.lang.OutOfMemoryError
```

更准确的表达是：

> **服务已经发生严重 GC 风暴，并且很可能存在大量长期存活对象；如果继续恶化，最终可能发展成 Heap OOM。**

---

## 八、为什么 -Xmx4G 的 Java 进程可能占 6G 甚至更多

今天第七题是一个很重要的概念纠正。

一开始容易把：

```text
-Xmx4G
```

理解成：

```text
Java 进程所有内存最多 4G
```

这是错误的。

`-Xmx` 主要控制：

```text
Java Heap 最大值
```

但是一个 Java 进程还包含：

```text
Java 进程
│
├── Heap
├── Metaspace
├── Thread Stack
├── Direct Memory
├── Code Cache
└── JVM Native Memory
```

所以完全可能出现：

```text
Heap = 3G
Metaspace = 一部分
线程栈 = 一部分
Direct Memory = 一部分
Native Memory = 一部分
↓
整个 Java 进程 RSS = 6G+
```

因此：

```text
-Xmx4G
```

不等于：

```text
Java 进程最多 4G
```

---

### MySQL、Redis 不能算进 Java 进程内存

训练里一开始提到：

```text
Java 进程可能还包括 MySQL、Redis
```

这个理解不对。

通常：

```text
Java 服务
MySQL
Redis
```

是不同的操作系统进程。

它们可以一起消耗整台机器的内存，但：

```text
MySQL 的内存
Redis 的内存
```

不能算成：

```text
Java 进程自己的内存
```

所以排查时要区分：

```text
整台机器内存

某个 Java 进程的总内存

Java Heap 内存
```

三个概念不能混在一起。

---

## 九、Kubernetes memory limit 与 -Xmx 为什么不能简单设成一样

今天第八题：

```text
Kubernetes memory limit = 4Gi
```

JVM：

```text
-Xmx4g
```

乍看像：

```text
刚刚好
```

实际上风险很高。

原因是：

```text
memory limit
→ 限制容器 / 进程整体可使用的内存
```

而：

```text
-Xmx
→ 主要限制 Java Heap
```

Java 进程真正需要的是：

```text
Heap
+
Metaspace
+
Thread Stack
+
Direct Memory
+
Native Memory
+
其他 JVM 内存
```

所以可能出现：

```text
Heap 还没达到 4G
↓
但整个 Java 进程已经超过 4Gi
↓
容器达到 memory limit
↓
进程被杀
↓
OOMKilled
```

这也是为什么：

> **容器 memory limit 必须给 Heap 之外的内存留空间。**

---

## 十、OOMKilled 和 Java OutOfMemoryError 必须区分

今天第九题：

Java 服务突然挂了。

Java 日志里没有：

```text
OutOfMemoryError
```

但 Kubernetes 显示：

```text
OOMKilled
```

这时候不能只分析 Java Heap。

原因是：

```text
Java Heap 可能还没满
```

但整个进程还有：

```text
Metaspace
Thread Stack
Direct Memory
Native Memory
```

结果：

```text
整个容器内存
超过 memory limit
↓
Kubernetes / 操作系统
直接杀掉 Java 进程
```

所以它甚至可能：

```text
还没等 JVM 自己抛出 Java OOM
进程就已经被外部杀死
```

这两个概念必须分开：

### Java OOM

例如：

```text
java.lang.OutOfMemoryError: Java heap space
```

更像：

```text
JVM 自己发现资源无法继续分配
```

### OOMKilled

更像：

```text
操作系统 / 容器发现整个进程内存超过限制
↓
直接杀进程
```

因此看到：

```text
OOMKilled
```

排查范围必须扩大到：

```text
Heap
+
堆外内存
+
线程
+
整个进程内存
+
容器 memory limit
```

---

## 十一、从“内存一直涨”怎么追到具体代码

今天最后一道题要求的是完整线上排查链路。

一开始只能想到：

```text
看日志
↓
看内存
↓
再继续查
```

这个方向没有错，但还需要进一步细化。

一个更完整的思维顺序是：

```text
① 先确认故障现象
内存上涨？CPU 高？RT 变慢？Full GC？进程退出？
        ↓
② 判断到底是 Heap 高，还是整个 Java 进程内存高
        ↓
③ 看有没有 OOM
如果有，先看完整 OOM 类型
        ↓
④ 看 GC 是否异常
Full GC 是否越来越频繁？
        ↓
⑤ 看 GC 后内存能不能明显下降
        ↓
⑥ 如果 GC 后基线持续上涨
怀疑长期存活对象越来越多
        ↓
⑦ 获取 / 分析 Heap Dump
        ↓
⑧ 找哪些对象数量最多、占用最大
        ↓
⑨ 继续追谁在引用这些对象
        ↓
⑩ 沿引用链定位到业务字段 / 集合 / 类
        ↓
⑪ 回到代码确认为什么对象一直没有释放
```

可以进一步压缩成：

```text
机器
↓
Java 进程
↓
JVM 内存区域
↓
Heap 对象
↓
引用链
↓
具体代码
```

---

### 一个典型例子

假设 Heap Dump 分析以后发现：

```text
HashMap / ConcurrentHashMap
占用大量内存
```

继续追引用链发现：

```java
private static final Map<Long, ExportTask> TASK_MAP =
        new ConcurrentHashMap<>();
```

创建任务时：

```java
TASK_MAP.put(taskId, task);
```

但任务结束以后从来没有：

```java
TASK_MAP.remove(taskId);
```

于是：

```text
ExportTask 业务上已经完成
↓
TASK_MAP 仍然引用它
↓
GC 判断对象仍然可达
↓
不能回收
↓
对象越来越多
↓
Heap GC 后基线持续上涨
```

真正修复的方向应该是：

```java
try {
    process(task);
} finally {
    TASK_MAP.remove(taskId);
}
```

这时候排障才真正从：

```text
“服务器内存高”
```

追到了：

```text
“具体哪段代码一直持有对象引用”
```

---

## 十二、今天训练中的几个关键纠正

### 1. “Heap 90% = 内存泄漏”

错误。

应该是：

```text
Heap 90%
→ 只表示当前内存使用高
→ 继续看 GC 后能否回落
```

---

### 2. “GC 后 3.5G = 一定内存泄漏”

也不能绝对判断。

应该继续看：

```text
连续多次 GC 后的存活内存基线
```

如果持续：

```text
700M
↓
1.2G
↓
1.8G
↓
2.6G
```

才越来越值得怀疑。

---

### 3. “static 对象一定会泄漏”

错误。

真正的问题是：

```text
长期引用
+
持续持有无用对象
+
没有释放
```

---

### 4. “分批处理是为了降低内存泄漏”

不准确。

这个案例主要解决：

```text
瞬时内存峰值太大
```

分批后让：

```text
同一时间存活的对象数量更少
```

---

### 5. “分批处理一定更快”

不准确。

它的主要收益是：

```text
可控
稳定
降低 OOM 风险
```

不保证纯执行时间一定更短。

---

### 6. “频繁 Full GC 就已经等于 OOM”

错误。

正确关系更像：

```text
GC 越来越频繁
+
回收效果越来越差
↓
GC 风暴
↓
服务性能严重下降
↓
继续恶化以后可能发展成 OOM
```

---

### 7. “-Xmx4G = Java 进程最大 4G”

错误。

应该是：

```text
-Xmx4G
→ Heap 最大约 4G
```

Java 进程还有其他内存区域。

---

### 8. “Java 进程内存包含 MySQL、Redis”

错误。

MySQL、Redis 通常是独立进程。

应该区分：

```text
机器总内存
Java 进程总内存
Java Heap
```

---

## 十三、今天形成的工程判断

今天不需要把 JVM 所有细节一次性背完，真正需要先形成下面几条判断。

### 判断 1

```text
内存高
≠
内存泄漏
```

先看：

```text
GC 后还能剩多少
```

---

### 判断 2

如果出现：

```text
Full GC 越来越频繁
+
每次只能回收很少
+
CPU 很高
+
RT 明显上涨
```

要优先想到：

```text
大量长期存活对象
GC 风暴
可能存在内存泄漏
```

---

### 判断 3

看到：

```text
java.lang.OutOfMemoryError: Java heap space
```

不能只回答：

```text
Heap 太小
```

还要继续考虑：

```text
内存泄漏？
一次性加载数据太大？
对象生命周期设计有问题？
```

---

### 判断 4

看到：

```text
-Xmx4G
```

必须立即知道：

```text
这只是 Heap 上限
不是整个 Java 进程的上限
```

---

### 判断 5

看到 Kubernetes：

```text
OOMKilled
```

不能只盯 Java Heap。

还要看：

```text
整个进程 RSS
Metaspace
线程栈
Direct Memory
Native Memory
容器 memory limit
```

---

### 判断 6

真正排查内存泄漏，最终不能停留在：

```text
“内存很高”
```

而要继续追：

```text
什么对象在增长？
↓
谁在引用这些对象？
↓
为什么业务结束以后引用还没释放？
↓
具体是哪段代码？
```

---

## 十四、今天最值得记住的三句话

### 第一句

> **内存高不等于内存泄漏，判断时更重要的是看 GC 后存活内存能不能有效下降，以及这个基线是否持续上涨。**

### 第二句

> **`-Xmx` 主要限制 Java Heap，而不是整个 Java 进程，所以 Java 进程实际内存完全可能高于 `-Xmx`。**

### 第三句

> **排查内存泄漏最终要从“内存高”追到“什么对象长期存活”，再追引用链，最后定位到具体代码。**

---

## 十五、今日复盘

今天是第一次系统接触 JVM 内存、GC 和 OOM 线上排障，开始时对多个概念比较陌生，但经过逐题训练以后，已经能够建立以下基础判断：

```text
Heap 90%
→ 不能直接判断内存泄漏

GC 后 3.8G → 700M
→ GC 回收效果正常

GC 后 3.8G → 3.5G
→ 更值得怀疑大量对象长期存活

Java heap space
→ 可能是内存泄漏，也可能是一次性加载数据太大

300 万订单一次性进入 List
→ 优先分批控制内存峰值

Full GC 每 2 秒一次且几乎回收不动
→ 可能进入 GC 风暴

-Xmx4G
→ 只代表 Heap 上限，不代表 Java 进程总上限

Pod memory limit = 4Gi + -Xmx4g
→ 没给堆外内存留下空间，存在 OOMKilled 风险

Kubernetes OOMKilled
→ 不等于 Java Heap OOM
```

目前还不要求一次性掌握：

```text
GC Roots
Retained Size
各种 GC 算法细节
复杂 JVM 参数调优
```

现阶段更重要的是先建立正确排障顺序：

```text
现象
↓
Heap / 进程内存
↓
OOM 类型
↓
GC 情况
↓
GC 后基线
↓
Heap 对象
↓
引用链
↓
具体代码
```

这套思维以后遇到 Java 服务：

```text
内存持续上涨
接口越来越慢
Full GC 频繁
进程突然退出
容器 OOMKilled
```

时，可以作为第一层排障框架。