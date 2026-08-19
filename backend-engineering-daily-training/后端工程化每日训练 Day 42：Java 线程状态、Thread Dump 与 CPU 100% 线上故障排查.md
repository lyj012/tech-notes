# 后端工程化每日训练 Day 42：Java 线程状态、Thread Dump 与 CPU 100% 线上故障排查

## 一、今天学习的知识点

今天学习的是：

**Java 线程状态、Linux `top`、`top -Hp`、Thread Dump、Linux TID 与 JVM `nid` 的对应关系、Tomcat HTTP 工作线程、锁竞争、线程池耗尽，以及 Java 服务 CPU 飙高时的线上排查思路。**

一句话理解：

> Java 服务 CPU 飙高、接口 RT 暴涨或者“服务还活着但请求几乎处理不动”时，不能只停留在“Java 很吃 CPU”，而要从 Linux 进程继续追到具体线程，再通过 Thread Dump 追到 Java 调用栈和具体代码。

今天一开始多个概念比较陌生：

```text
Thread Dump
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
HTTP 工作线程
TID
nid
top -Hp
jstack
```

训练以后，最核心的排障主线已经可以压缩成：

```text
CPU / RT 异常
↓
top 找 Java 进程
↓
top -Hp 找高 CPU 线程
↓
Linux TID 转十六进制
↓
Thread Dump 中找 nid
↓
看线程状态和 Java 调用栈
↓
定位具体业务代码 / 锁 / 下游等待
```

今天最重要的几个工程判断是：

```text
CPU 高 ≠ 一定死循环

CPU 不高 ≠ Java 服务一定正常

大量 BLOCKED ≠ 一定死锁

线程数量多 ≠ 并发能力一定强

重启能恢复服务 ≠ 根因已经解决
```

---

## 二、`top` 到底是干什么的

Linux 中：

```bash
top
```

可以先把它理解成：

> **Linux 的实时任务管理器。**

它可以帮助查看：

```text
CPU 使用情况
内存使用情况
进程 PID
进程名称
进程 CPU / 内存占用
```

例如：

```text
PID      %CPU    COMMAND
23781    760.0   java
```

说明：

```text
PID = 23781
Java 进程 CPU 使用非常高
```

但这里还只能得出：

```text
Java 这个进程很吃 CPU
```

还不能知道：

```text
到底是哪一个 Java 线程有问题？
到底执行了哪段 Java 代码？
```

所以 `top` 只是第一层定位。

---

## 三、为什么 8 核机器上 Java CPU 可以显示 750%

今天第一题：

```text
8 核 CPU
Java CPU = 750%
```

一开始容易产生疑问：

```text
CPU 不是最多 100% 吗？
```

在 Linux `top` 的常见显示方式下，多核 CPU 可以累计计算。

因此：

```text
100%
≈ 持续占用 1 个 CPU 核心

750%
≈ 持续使用约 7.5 个 CPU 核心的计算能力
```

所以 8 核机器上：

```text
Java CPU = 750%
```

大致意味着：

> **这个 Java 进程已经接近把 8 个 CPU 核心都吃满。**

但仍然不能直接判断：

```text
一定存在 Bug
```

因为还要结合：

```text
QPS
RT
错误率
任务吞吐量
GC
线程状态
```

判断到底是：

```text
正常高负载
```

还是：

```text
异常 CPU 消耗
```

---

## 四、为什么 `top` 后面还要继续 `top -Hp <PID>`

假设通过：

```bash
top
```

已经找到：

```text
Java PID = 23781
CPU = 760%
```

下一步常见做法是：

```bash
top -Hp 23781
```

这里可以先理解成：

```text
top
→ 看整个进程

top -Hp <PID>
→ 继续往这个进程内部看线程
```

例如：

```text
TID      %CPU
23891    99.8
23892    98.7
23905    97.6
```

现在排查范围已经从：

```text
Java 进程 CPU 很高
```

缩小成：

```text
线程 23891 很高
线程 23892 很高
线程 23905 很高
```

今天第二题已经能够判断出：

> **`top -Hp <PID>` 的价值是找到 Java 进程内部到底是哪几个线程在吃 CPU。**

这一步非常重要，因为一个 Spring Boot 进程里面可能同时存在：

```text
Tomcat HTTP 工作线程
GC 线程
MQ 消费线程
定时任务线程
数据库相关线程
自定义线程池
JVM 内部线程
```

如果不继续缩小范围，直接去翻整个项目业务代码，效率会非常低。

---

## 五、Linux TID 为什么还要转换成十六进制

假设：

```bash
top -Hp 23781
```

找到高 CPU 线程：

```text
TID = 23891
CPU ≈ 100%
```

Linux 这里看到的是十进制线程 ID。

而 JVM Thread Dump 中常见线程信息里会出现：

```text
nid=0x....
```

为了把：

```text
Linux 高 CPU 线程
```

和：

```text
JVM Thread Dump 中的 Java 线程
```

对应起来，可以把十进制 TID 转成十六进制：

```bash
printf "%x\n" 23891
```

例如得到：

```text
5d53
```

然后在 Thread Dump 中搜索：

```text
nid=0x5d53
```

最终可能找到：

```text
"http-nio-8080-exec-43"
java.lang.Thread.State: RUNNABLE
    at com.example.OrderService.calculate(OrderService.java:128)
```

于是完整关系变成：

```text
Linux TID = 23891
↓
十六进制 = 5d53
↓
Thread Dump 搜 nid=0x5d53
↓
找到 Java 线程
↓
找到 Java 调用栈
↓
OrderService.calculate()
```

今天这里有一个重要概念纠正：

```text
这不是“为了看内存快照”
```

而是：

> **为了在 Thread Dump（线程快照）中找到 Linux 高 CPU 线程对应的 Java 线程。**

---

## 六、Thread Dump 到底是什么

今天专门补了这个概念。

Thread Dump 可以理解成：

> **某一个时间点，JVM 中所有线程正在干什么的一张快照。**

它通常可以看到：

```text
线程名称
线程状态
调用栈
线程正在执行的方法
锁等待关系
某些死锁信息
```

常见获取方式：

```bash
jstack <PID>
```

例如：

```bash
jstack 23781 > /tmp/jstack-1.txt
```

可能看到：

```text
"http-nio-8080-exec-43"
java.lang.Thread.State: RUNNABLE
    at com.example.OrderService.calculate(OrderService.java:128)
    at com.example.OrderController.query(OrderController.java:56)
```

可以理解成：

```text
抓取快照的这个瞬间
↓
这个线程正在这条 Java 调用链附近执行
```

---

### Thread Dump 和 Heap Dump 不能混

今天训练中一开始把两者混在了一起，这里需要明确区分：

```text
Thread Dump
→ 看线程现在在干什么

Heap Dump
→ 看 Java Heap 里有哪些对象、对象占多少内存、谁在引用它们
```

一句话记忆：

> **Thread Dump 看线程现场，Heap Dump 看堆对象现场。**

---

## 七、四种常见线程状态怎么理解

今天重点学习了：

```text
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
```

不要求死背 JVM 定义，先建立工程直觉。

### 1. RUNNABLE

可以先理解为：

```text
线程正在运行
或者已经准备好运行
```

人话：

> **我在干活 / 我准备干活。**

例如：

```java
while (true) {
    count++;
}
```

如果某个线程：

```text
长期 RUNNABLE
+
CPU 接近 100%
+
多次 Thread Dump 都停在同一段业务代码
```

就非常值得怀疑：

```text
死循环
超重计算
异常算法
正则灾难性回溯
频繁重试
```

---

### 2. BLOCKED

可以先理解为：

```text
我要进入 synchronized 临界区
↓
但是锁被别的线程拿着
↓
我只能等
```

人话：

> **门被别人锁住了，我进不去。**

例如：

```java
synchronized (lock) {
    doSomething();
}
```

线程 A 已经拿到 `lock`。

线程 B、C、D 也想进入：

```text
B → BLOCKED
C → BLOCKED
D → BLOCKED
```

所以大量 `BLOCKED` 往往要继续看：

```text
大家是不是在等同一把锁？
谁持有这把锁？
持锁线程为什么这么久不释放？
```

---

### 3. WAITING

可以先理解为：

```text
线程主动等待某个条件
没有固定超时时间
```

常见：

```text
Object.wait()
Thread.join()
LockSupport.park()
```

人话：

> **我先等着，等别人把我叫醒。**

但看到 `WAITING` 不能直接判定故障。

很多线程池没有任务时，本来就可能处于等待状态。

---

### 4. TIMED_WAITING

和 `WAITING` 类似，但有时间限制。

例如：

```java
Thread.sleep(1000);
```

人话：

> **我等一会儿，时间到了就继续。**

因此：

```text
WAITING
TIMED_WAITING
```

本身都不等于：

```text
线程挂了
```

真正重要的是上下文：

```text
它在等什么？
为什么等？
是不是大量关键业务线程都在等同一个东西？
```

---

## 八、为什么最好连续抓多次 Thread Dump

今天第四题：

某个高 CPU 线程连续三次 Thread Dump 都显示：

```text
RUNNABLE
at OrderService.calculate(OrderService.java:128)
```

并且：

```text
线程 CPU 一直接近 100%
```

首先应该怀疑：

```text
这段代码可能存在死循环
超重计算
异常算法
```

为什么不只抓一次？

因为一次 Thread Dump 只能说明：

```text
拍照这一瞬间
线程刚好执行到这里
```

它可能只是正常经过这个方法。

但如果：

```text
第 1 次：OrderService.java:128
第 2 次：OrderService.java:128
第 3 次：OrderService.java:128
```

同时：

```text
CPU 一直 100%
线程一直 RUNNABLE
```

那么可信度会高很多。

可以记成：

> **单次 Thread Dump 是一个瞬时证据，多次 Thread Dump 的重复模式更接近真正的故障特征。**

---

## 九、HTTP 工作线程是什么

今天做到锁竞争问题时，对“HTTP 线程”一开始比较陌生。

Spring Boot 使用 Tomcat 时，可以先把 Tomcat HTTP 工作线程理解成：

> **负责处理用户 HTTP 请求的工人。**

例如：

```text
用户 A 请求 /order/create
→ http-nio-8080-exec-1

用户 B 请求 /order/create
→ http-nio-8080-exec-2

用户 C 请求 /user/info
→ http-nio-8080-exec-3
```

一个请求进来后，会占用某个工作线程去执行：

```text
Controller
↓
Service
↓
数据库 / Redis / 远程接口 / 文件 IO
↓
返回响应
```

当这个请求处理完成以后，工作线程才能继续处理别的任务。

因此如果大量 HTTP 工作线程长期被占住，就可能出现：

```text
线程池被占满
↓
新请求只能排队
↓
RT 上升
↓
大量请求超时
```

---

## 十、CPU 只有 20%，Tomcat 200 个线程全部占满，为什么仍然可能严重故障

今天第五题给出的现象：

```text
CPU = 20%
接口 RT = 10s
Tomcat 200 个线程全部占满
```

不能得出：

```text
CPU 不高
↓
Java 服务没问题
```

因为线程可能根本没有在大量计算，而是在：

```text
等数据库
等远程接口
等网络 IO
等锁
等连接池
```

假设：

```text
Tomcat 最大工作线程 = 200
```

然后下游接口突然每次要等 30 秒。

可能形成：

```text
请求 1 → 占住线程 1 等 30 秒
请求 2 → 占住线程 2 等 30 秒
...
请求 200 → 占住线程 200 等 30 秒
↓
200 个线程全部被占住
↓
第 201 个请求只能排队
↓
后续请求越来越多
↓
接口全面超时
```

这里今天有一个重要纠正。

一开始容易说：

```text
“Tomcat 线程太多了，200 个线程用完了”
```

更准确的表达应该是：

> **不是线程太多，而是现有 200 个工作线程全部被某些慢操作长期占住，无法及时释放。**

所以真正要问的是：

```text
为什么这 200 个线程迟迟不能完成请求？
```

而不是只问：

```text
线程数量是不是应该再加大？
```

---

## 十一、150 个 HTTP 线程全部 BLOCKED，应该怎么分析

今天这一题一开始不太会分析：

```text
150 个 HTTP 线程
全部 BLOCKED
全部等待同一个锁对象
```

第一反应不应该直接是：

```text
死锁
```

而应该先：

> **找到是谁持有这把锁，再看这个持锁线程正在执行什么，为什么迟迟不释放。**

可以把现场想象成：

```text
线程 A
→ 已经进入房间
→ 手里拿着 lock

线程 B/C/D/.../150
→ 全部在门外
→ 等 A 释放 lock
→ BLOCKED
```

这更像：

```text
严重锁竞争
持锁时间过长
```

不一定是死锁。

---

## 十二、大量线程等同一把锁和死锁有什么区别

这是今天一个非常重要的纠正。

### 普通锁竞争

```text
线程 A
拿着 lock
↓
执行 30 秒
↓
释放 lock

线程 B / C / D
等待 lock
↓
A 释放以后
↓
后面的线程还能继续
```

虽然系统可能非常慢，但：

```text
最终还能继续运行
```

这更像：

```text
锁竞争严重
持锁时间太长
```

---

### 死锁

典型结构：

```text
线程 A
已经拿着锁 1
↓
等待锁 2

线程 B
已经拿着锁 2
↓
等待锁 1
```

形成：

```text
A 等 B
B 等 A
```

谁都无法继续。

这才更像：

```text
死锁
```

Thread Dump 有时甚至可以直接看到类似：

```text
Found one Java-level deadlock
```

所以今天必须记住：

> **大量线程等待同一把锁不等于死锁；循环等待、彼此持有对方需要的锁，才是典型死锁结构。**

另外还要区分：

```text
数据库死锁
→ MySQL / 数据库锁

Java 线程死锁
→ JVM 中 Java 对象锁
```

不是同一个层面的故障。

---

## 十三、为什么下面的 `synchronized` 设计很危险

代码：

```java
synchronized (lock) {
    queryDatabase();
    callRemoteService();
    generateFile();
}
```

今天已经能够直接判断出问题：

```text
锁里面放的东西太多
+
每一步都可能很慢
↓
持锁时间太长
```

例如线程 A 拿到锁后：

```text
查数据库 2 秒
↓
远程接口卡 5 秒
↓
生成文件 3 秒
↓
整个过程 10 秒都不释放锁
```

那么其他请求：

```text
B → BLOCKED
C → BLOCKED
D → BLOCKED
...
```

结果可能变成：

```text
一个线程慢慢执行
+
大量 HTTP 工作线程排队
↓
Tomcat 工作线程被占住
↓
RT 暴涨
↓
请求超时
```

这里不能简单得出：

```text
synchronized 不能用
```

真正的问题是：

> **锁范围太大、持锁时间太长，并且锁里面包含数据库、远程调用、文件 IO 等慢操作。**

`synchronized` 本身可以正常使用，但更适合：

```text
临界区小
执行快
竞争低
```

---

## 十四、CPU 90% + Full GC 频繁，为什么不能直接判死循环

今天第八题：

```text
Java CPU = 90%
Full GC 非常频繁
```

不能直接说：

```text
肯定某段业务代码死循环
```

因为：

> **GC 自己也会消耗大量 CPU。**

例如：

```text
对象创建速度非常快
↓
Heap 很快被填满
↓
GC 不断工作
↓
CPU 被 GC 大量消耗
```

还有另一种情况：

```text
大量对象长期被引用
↓
GC 回收不了多少
↓
Heap 一直很满
↓
Full GC 不断重复
↓
CPU 被 GC 吃掉
```

所以看到：

```text
CPU 高
+
Full GC 频繁
```

应该联动检查：

```text
CPU
Heap
GC 频率
GC 前后内存变化
线程状态
Thread Dump
```

尤其看：

### 情况 A

```text
Full GC 前：3.9G
Full GC 后：800M
```

说明：

```text
GC 能回收大量对象
```

可能需要继续怀疑：

```text
对象分配速度是不是太快
```

### 情况 B

```text
Full GC 前：3.9G
Full GC 后：3.7G
```

说明：

```text
GC 几乎回收不下来
```

更值得怀疑：

```text
大量长期存活对象
内存泄漏
Heap 压力严重
```

所以今天 Day 42 和 Day 41 已经开始连起来：

```text
CPU
↔
线程
↔
Heap
↔
GC
```

线上不能只看某一个指标就下结论。

---

## 十五、8 核升级 32 核什么时候有效，什么时候只是拖延故障

今天第九题一开始从“线程数量有限 / 无限”去判断是否该加 CPU。

这个角度不算完全错误，但不是核心标准。

真正应该判断的是：

> **CPU 高，是正常业务计算量确实超过机器能力，还是 Bug / 异常行为在浪费 CPU。**

### 1. 加 CPU 可能真的有效

例如：

```text
正常批处理
压缩
加密
大量数据计算
CPU 密集型任务
```

代码本身没有异常，并且任务能够有效并行。

如果：

```text
业务量持续增加
↓
8 核已经成为真正瓶颈
↓
任务持续积压
```

那么：

```text
8 核 → 32 核
```

可能真的提高吞吐量。

---

### 2. 加 CPU 只是推迟故障

如果根因是：

```java
while (true) {
    calculate();
}
```

或者：

```text
无限重试
异常日志风暴
GC 风暴
异常算法
线程创建失控
```

那么：

```text
8 核 → 32 核
```

并没有修复根因。

它最多可能：

```text
短期多给系统一点资源
```

甚至变成：

```text
以前可以吃满 8 核
现在有机会继续吃满 32 核
```

所以可以记成：

> **正常负载把资源吃满，扩容可能有效；异常代码把资源吃满，扩容最多止血，根因仍然必须修。**

---

## 十六、真实事故中为什么不能“直接重启”，也不能为了 Debug 一直不恢复服务

今天最后一题：

```text
接口 RT 暴涨
CPU = 780%
Java 服务仍然存活
大量用户请求超时
```

现在有两个目标：

```text
A. 尽快恢复服务
B. 保存故障现场，找到根因
```

一开始的想法是：

```text
先把根因彻底找到
↓
修复
↓
最后再恢复服务
```

这个思路需要调整。

真实生产事故里，业务正在持续受损，不能为了研究故障让用户无限等待。

更合理的顺序是：

```text
① 在极短时间内快速保存关键故障现场
↓
② 如果不能立刻定位并修复，就先止血恢复服务
↓
③ 利用已经保存的现场继续做根因分析
↓
④ 修复代码 / 配置 / 架构问题
↓
⑤ 防止再次发生
```

在条件允许时，重启前可以快速保存：

```text
top 信息
线程 CPU
Thread Dump
GC 状态
关键日志
```

例如：

```bash
top

top -Hp <PID>

jstack <PID> > /tmp/jstack-1.txt
jstack <PID> > /tmp/jstack-2.txt
jstack <PID> > /tmp/jstack-3.txt
```

然后如果服务已经严重不可用：

```text
重启
摘流
切换实例
```

先让业务恢复。

之后继续利用刚才的证据：

```text
高 CPU 线程
↓
TID 转十六进制
↓
Thread Dump 找 nid
↓
调用栈
↓
具体业务代码
↓
永久修复
```

所以今天最重要的事故处理判断是：

> **既不能一上来无脑重启把故障现场全部清掉，也不能为了 Debug 无限延迟业务恢复。应该快速保留关键证据，然后及时止血，再继续根因分析。**

---

## 十七、今天训练中的几个关键纠正

### 1. “Thread Dump 是内存快照”

错误。

应该是：

```text
Thread Dump
→ 线程快照
→ 看 JVM 中线程正在干什么

Heap Dump
→ 堆内存快照
→ 看 Java Heap 中有哪些对象
```

---

### 2. “CPU 750% 是不可能的”

错误。

在多核机器常见显示方式下：

```text
750%
≈ 7.5 个 CPU 核心的计算能力
```

---

### 3. “top 已经看到 Java CPU 高，直接翻代码就行”

不合理。

应该继续：

```text
top
↓
Java PID
↓
top -Hp
↓
高 CPU TID
↓
Thread Dump
↓
调用栈
↓
具体代码
```

---

### 4. “Tomcat 200 个线程占满 = Tomcat 线程太多”

不准确。

真正应该问：

```text
为什么现有 200 个工作线程都被长期占住？
```

可能是：

```text
下游慢
数据库慢
锁等待
网络 IO
连接池耗尽
业务代码执行时间过长
```

---

### 5. “150 个线程 BLOCKED 在同一把锁 = 死锁”

错误。

更常见的是：

```text
严重锁竞争
持锁线程执行太久
```

真正典型的死锁更像：

```text
A 拿锁 1 等锁 2
B 拿锁 2 等锁 1
```

---

### 6. “synchronized 有问题，所以不能用”

错误。

真正需要控制的是：

```text
锁范围
持锁时间
竞争程度
```

危险的是在锁里面执行大量慢操作。

---

### 7. “CPU 高 = 一定业务代码死循环”

错误。

还要看：

```text
GC
线程
Heap
RT
QPS
错误率
```

GC 风暴本身也可能大量消耗 CPU。

---

### 8. “为了找根因，先不恢复服务”

不适合真实线上事故。

更合理的是：

```text
快速保留现场
↓
及时止血恢复
↓
事后根因分析和永久修复
```

---

## 十八、今天形成的工程判断

### 判断 1

```text
Java CPU 高
```

不能停在：

```text
服务器 CPU 不够
```

要继续追：

```text
哪个进程？
哪个线程？
什么线程状态？
什么调用栈？
哪段代码？
```

---

### 判断 2

CPU 高时的核心排查链路：

```text
top
↓
Java PID
↓
top -Hp <PID>
↓
高 CPU TID
↓
printf 转十六进制
↓
Thread Dump 找 nid
↓
Java 调用栈
↓
具体业务代码
```

---

### 判断 3

如果：

```text
CPU 不高
+
RT 很高
+
Tomcat 工作线程全部占满
```

仍然可能是严重故障。

应该重点怀疑：

```text
数据库等待
远程接口等待
锁竞争
连接池耗尽
线程池耗尽
其他慢 IO
```

---

### 判断 4

如果大量线程：

```text
BLOCKED
+
等待同一把锁
```

首先：

```text
找持锁线程
↓
看它正在执行什么
↓
为什么迟迟不释放锁
```

而不是立即喊：

```text
死锁
```

---

### 判断 5

如果：

```text
CPU 高
+
Full GC 高频
```

不能只查业务死循环。

还需要判断：

```text
CPU 是业务线程吃掉的？
还是 GC 吃掉的？
GC 后 Heap 能不能明显下降？
```

---

### 判断 6

扩容前先判断：

```text
这是正常容量瓶颈？
还是异常代码 / 异常重试 / GC / 死循环在浪费资源？
```

扩容可以止血，但不能替代根因修复。

---

### 判断 7

真实事故处理需要同时考虑：

```text
恢复业务
+
保存故障证据
```

正确目标不是只追求其中一个，而是：

```text
用尽可能短的时间保留关键现场
↓
及时止血
↓
再根因分析
```

---

## 十九、今天最值得记住的三句话

### 第一句

> **CPU 高不能停在“Java 很吃资源”，要从进程继续追到线程，再从 Thread Dump 追到调用栈和具体代码。**

### 第二句

> **CPU 不高也可能发生严重线程池故障；大量线程被锁、数据库、远程接口或其他慢操作占住，同样会让 RT 暴涨。**

### 第三句

> **线上事故不能无脑重启，也不能为了 Debug 无限拖延恢复；先快速保存关键现场，再及时止血，最后完成根因修复。**

---

## 二十、今日复盘

今天是第一次系统训练 Java 线程状态、Thread Dump 和 CPU 异常排障，理解难度比前面的单点知识明显更高，因为它不再是：

```text
一个现象
→ 一个固定答案
```

而是需要不断缩小范围：

```text
CPU / RT 异常
↓
进程
↓
线程
↓
线程状态
↓
调用栈
↓
锁 / GC / 下游 / 业务代码
↓
根因
```

经过逐题训练以后，目前已经可以建立以下基础判断：

```text
8 核机器 CPU 750%
→ 大约使用 7.5 个 CPU 核心的计算能力

Java CPU 高
→ top 先找 PID

进程内部继续定位
→ top -Hp <PID> 找高 CPU 线程

Linux TID 和 JVM 线程对应
→ TID 转十六进制，再到 Thread Dump 找 nid

Thread Dump
→ 看线程现在在干什么

Heap Dump
→ 看堆里有什么对象

RUNNABLE
→ 正在执行 / 准备执行

BLOCKED
→ 想进入 synchronized，但锁被别人持有

WAITING
→ 等待条件，没有固定超时时间

TIMED_WAITING
→ 带时间限制的等待

CPU 20% + Tomcat 200 线程占满 + RT 10s
→ 仍然可能发生严重线程池耗尽

150 个线程 BLOCKED 同一把锁
→ 先找持锁线程，不要直接判断死锁

synchronized 中包含 DB / RPC / 文件 IO
→ 持锁时间可能过长

CPU 高 + Full GC 高频
→ 联动分析 CPU、Heap、GC 和线程

8 核 → 32 核
→ 只有真实容量瓶颈时才可能根治；异常代码只能暂时止血

线上严重故障
→ 快速留现场 → 止血恢复 → 根因分析 → 永久修复
```

目前还不要求一次性掌握：

```text
JVM 所有线程状态细节
复杂锁实现
各种 jstack 输出字段
操作系统调度器细节
高级 CPU Profiling 工具
```

现阶段最重要的是把这一条真正记住：

```text
top
↓
top -Hp
↓
高 CPU TID
↓
十六进制
↓
Thread Dump / nid
↓
线程状态 + 调用栈
↓
具体代码
```

以及另一条容易被忽略的链路：

```text
CPU 不高但 RT 高
↓
不要排除 Java 问题
↓
看线程是不是都被慢操作占住
↓
看锁 / 数据库 / 网络 / 下游 / 线程池
```

这两条可以作为以后处理 Java 服务 CPU 飙高、接口卡顿和线程阻塞问题的第一层排障框架。