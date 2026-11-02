# 后端工程化每日训练 Day 82：JMM、happens-before、volatile 与并发原子性复盘

## 一、今天学习什么

今天从前两天的 Spring 主线切到 Java 并发底层，开始补后端工程里非常重要但之前覆盖较少的一块：

**Java Memory Model（JMM）、happens-before、volatile、synchronized、原子性、Lost Update、check-then-act，以及 AI 并发修复方案审核。**

今天真正要建立的不是一句：

```text
volatile 保证可见性，不保证原子性
```

而是形成下面这条判断链：

```text
多个线程共享状态
↓
线程间到底有没有同步关系
↓
是否存在 happens-before
↓
问题属于可见性、原子性还是互斥
↓
是否是复合操作
↓
应该使用 volatile、synchronized、Atomic 类还是更高层并发工具
```

今天最重要的变化，是开始从：

> “看到并发问题就找一个并发关键字”

转向：

> **先判断并发问题到底属于哪一种，再决定工具。**

---

## 二、JMM 不是简单的“主内存 + 线程缓存”

最开始容易把 Java 内存模型理解成：

```text
主内存
+
每个线程自己的缓存
```

这个类比只能帮助入门，但不够准确。

今天建立的更重要的理解是：

> **JMM 规定了在多线程程序中，哪些写入必须被哪些读取观察到，以及编译器、JIT、CPU 在什么范围内可以进行重排序。**

真正要判断的问题不是：

```text
两个线程是不是同时运行
```

而是：

```text
Thread A 的写入
↓
有没有规则保证
↓
Thread B 能够观察到
```

这就是后面理解 happens-before 的基础。

---

## 三、第一题：volatile 不只是“这个变量自己可见”

代码：

```java
class TaskState {

    int result;

    volatile boolean finished;
}
```

Thread A：

```java
state.result = 42;
state.finished = true;
```

Thread B：

```java
if (state.finished) {
    System.out.println(state.result);
}
```

前提：

```text
Thread B 已经明确读取到 finished == true
```

### 我最开始的判断

最开始我认为：

> 不能保证 B 读到 result == 42，因为 volatile 只保证可见性，不保证原子性。

这个判断是错的。

错误点在于：

> **把可见性和原子性混在了一起。**

这道题没有：

```java
result++;
```

也没有多个线程同时修改一个复合状态。

这里的关键是 happens-before 链。

完整关系：

```text
Thread A:

result = 42
↓
程序顺序
finished = true
↓
volatile write
↓
happens-before
↓
Thread B volatile read finished == true
↓
程序顺序
读取 result
```

由于 happens-before 具有传递关系：

```text
A: result = 42
↓
happens-before
↓
B: 读取 result
```

所以在题目给定条件下：

> **Thread B 有保证读取到 result == 42。**

今天这里真正建立的概念是：

> **volatile 写不仅影响 volatile 字段本身，还可以通过 happens-before 把写入之前的普通状态安全发布给后续读取线程。**

---

## 四、happens-before 到底是什么

今天不能再把 happens-before 理解成：

```text
现实时间上 A 一定先执行
```

更准确的理解是：

> **happens-before 是 Java 内存模型提供的一种线程间可见性与顺序保证关系。**

今天重点接触了几种常见来源。

### 1. 同一线程程序顺序

例如：

```java
a = 1;
b = 2;
```

在同一线程内，可以建立对应的程序顺序关系。

### 2. volatile

```text
对某 volatile 字段的写
↓
happens-before
↓
后续对同一 volatile 字段的读
```

### 3. synchronized

```text
线程 A 对某 monitor 解锁
↓
happens-before
↓
线程 B 后续对同一 monitor 加锁
```

### 4. Thread.start / join / 并发工具

今天还建立了一个重要意识：

> Java 并发 API 不是只有“功能”，很多 API 自己就定义了内存一致性语义。

所以以后看：

```text
Executor
Future
ConcurrentHashMap
CountDownLatch
CompletableFuture
```

不能只理解 API 怎么调用，还需要逐渐理解它们建立了什么线程间同步关系。

---

## 五、第二题：volatile 为什么救不了 count++

代码：

```java
volatile int count = 10;
```

Thread A 和 Thread B 各执行一次：

```java
count++;
```

今天判断：

> 最终完全可能得到 11，而不是 12。

因为：

```java
count++;
```

不是一个不可拆分的动作。

它本质接近：

```text
读取 count
↓
count + 1
↓
写回 count
```

一种可能的执行顺序：

```text
初始 count = 10

Thread A              Thread B

读取 10               读取 10
计算 11               计算 11
写入 11               写入 11
```

两个线程都执行了一次 `count++`，但最终只增加了 1。

这就是：

> **Lost Update（更新丢失）。**

今天这里需要牢牢记住：

```text
volatile
→ 提供可见性和顺序保证
→ 不提供互斥
→ 不会把复合操作自动变成原子操作
```

---

## 六、原子性不是“有读有写”

今天还纠正了一个表达。

不能简单说：

> “原子性就是读和写。”

更准确的是：

> **原子性描述的是一个操作能否作为一个不可分割的整体完成。**

例如：

```java
count++;
```

实际包含：

```text
读
+
修改
+
写
```

如果其他线程可以插入到这些步骤之间，那么这个复合操作整体就不是原子的。

所以以后判断时应该问：

```text
这个业务操作是不是一个复合动作？
↓
这些步骤之间能不能被其他线程插入？
↓
如果插入，会不会破坏最终结果？
```

---

## 七、第三题：synchronized 到底解决什么

代码：

```java
class Counter {

    private int count = 0;

    public synchronized void increment() {
        count++;
    }
}
```

今天最开始对 `synchronized` 本身的作用并不熟悉。

最后建立的核心模型是：

```text
synchronized
=
互斥
+
内存可见性
```

### 1. 互斥

假设：

```text
Thread A 已经进入 increment()
```

此时 Thread B 也调用同一个对象的 `increment()`：

```text
A 获取锁
↓
A 执行 count++
↓
A 释放锁
↓
B 才能获取锁
↓
B 执行 count++
```

B 不能在 A 还持有同一把锁时进入临界区。

所以：

```text
初始 count = 10

A 完成后 = 11
B 再执行 = 12
```

不会再出现上一题两个线程同时读取 10 的 Lost Update。

### 2. 可见性

`synchronized` 不只是“挡住别人”。

线程 A 对同一个 monitor 解锁，与后续线程 B 对同一个 monitor 加锁之间存在 happens-before，因此前一个线程在临界区里的修改，对后续拿到同一把锁的线程具有可见性保证。

今天形成的基础对照：

```text
volatile
→ 可见性
→ 顺序保证
→ 没有互斥

synchronized
→ 互斥
→ 可见性
```

---

## 八、synchronized 实例方法锁的是什么

代码：

```java
public synchronized void increment() {
    count++;
}
```

今天先建立一个简化但实用的理解：

```java
public synchronized void increment()
```

实例方法上加 `synchronized`，可以先理解为：

```java
synchronized (this)
```

所以重要前提是：

> **竞争的是同一个对象上的同一把锁。**

两个完全不同的 `Counter` 实例，并不会因为方法都写了 `synchronized` 就互相阻塞。

这也是以后分析锁时必须继续问的一层：

```text
锁住的到底是谁？
不同线程拿的是不是同一把锁？
```

---

## 九、第四题：check-then-act 为什么单靠 volatile 不安全

代码：

```java
class Task {

    private volatile boolean processing = false;

    public void execute() {
        if (!processing) {
            processing = true;
            doTask();
        }
    }
}
```

今天的判断：

> **两个线程仍然可能都执行 doTask()。**

一种可能的顺序：

```text
初始 processing = false

Thread A                   Thread B

读取 false                 读取 false
processing = true          processing = true
doTask()                   doTask()
```

问题就在：

```java
if (!processing) {
    processing = true;
}
```

这是：

```text
检查
+
修改
```

两个动作组成的复合操作。

`volatile` 没有能力把：

```text
检查 processing == false
+
设置 processing = true
```

变成一个原子整体。

这类问题就是典型的：

> **check-then-act 并发错误。**

---

## 十、为什么改成 synchronized 后就不同

如果改成：

```java
public synchronized void execute() {
    if (!processing) {
        processing = true;
        doTask();
    }
}
```

那么：

```text
Thread A 获取锁
↓
读取 processing
↓
修改 processing
↓
执行 doTask
↓
释放锁
↓
Thread B 才能进入
```

所以 Thread B 不能在 A 的：

```text
检查
↓
修改
```

之间插进来。

今天这里真正需要形成的是：

> **如果业务正确性要求多个步骤作为一个整体执行，就要保护整个复合操作，而不是只让其中某个字段“可见”。**

---

## 十一、第五题：80 万次 count++ 为什么最终可能不到 80 万

代码：

```java
volatile int count = 0;
```

8 个线程，每个执行：

```java
for (int i = 0; i < 100_000; i++) {
    count++;
}
```

总共实际执行：

```text
8 × 100000
=
800000 次 count++
```

今天最开始有一个理解偏差：

> “80 万可能只是最大执行次数。”

这个说法不准确。

正确的是：

> **80 万次 count++ 确实都会被执行，但最终成功体现在 count 数值上的累计结果可能小于 80 万。**

原因还是 Lost Update：

```text
A 读取旧值 N
B 也读取旧值 N

A 写 N+1
B 也写 N+1
```

代码执行了两次，但数值只增长了一次。

所以今天必须分开：

```text
执行了多少次 count++
≠
最终成功累计了多少次
```

并发 Bug 很典型的一点就是：

> **代码确实执行了，但结果被其他线程覆盖。**

---

## 十二、第六题：AI 把 int 改成 volatile，能不能上线

生产代码：

```java
class Metrics {

    private int successCount;

    public void success() {
        successCount++;
    }
}
```

几十个线程同时调用。

线上观察：

```text
真实成功请求：1,000,000
successCount：987,431
```

Codex 给出修复：

```java
private volatile int successCount;
```

并表示：

> 并发问题已修复，可以上线。

今天的结论：

> **不能批准。**

但这里还纠正了一个表达。

```text
successCount = 987431
```

不代表：

> 有一万多次真实请求失败。

真实业务请求已经成功了 100 万次。

少掉的是：

> **计数结果。**

根因仍然是：

```java
successCount++;
```

的复合读改写过程发生了 Lost Update。

Codex 增加 `volatile` 只解决了：

```text
可见性
```

但真正缺少的是：

```text
复合更新的原子性
```

所以属于：

> **发现了并发问题，但修错了层次。**

---

## 十三、候选修复：AtomicLong

如果业务确实要求一个 JVM 内做精确计数，可以考虑：

```java
AtomicLong successCount = new AtomicLong();

public void success() {
    successCount.incrementAndGet();
}
```

这里的关键不是记住 `AtomicLong` 这个类，而是：

> **它提供了与业务需求对应的原子更新能力。**

但工程审核不能在看到 `AtomicLong` 后就立即结束。

还需要继续判断：

```text
这个计数是否必须绝对精确？
是不是高并发热点？
是否允许最终聚合？
是否应该考虑 LongAdder？
统计是否应该存在 JVM 本地？
服务是不是多实例部署？
```

---

## 十四、单 JVM 正确，不代表系统正确

假设服务部署三个 Pod：

```text
Pod A：500000
Pod B：300000
Pod C：200000
```

即使三个 Pod 内部全部使用线程安全计数器，也只是分别保证：

```text
单 JVM 内正确
```

它并不会自动得到：

```text
全系统统一计数 = 1000000
```

所以今天非常重要的一层是：

> **代码正确性 ≠ 系统设计正确性。**

以后看到一个并发问题，不能只停在：

```text
字段加什么关键字
```

还要继续问：

```text
这个状态的作用域是什么？
线程内？
单 JVM？
多实例？
跨服务？
最终业务语义是什么？
```

---

## 十五、今天答题中暴露出的几个易错点

### 1. “volatile 不保证原子性，所以 result 不一定是 42”

错误。

第一题的关键是：

```text
普通写
↓
volatile write
↓
volatile read
↓
普通读
```

通过 happens-before 建立了安全发布关系。

### 2. volatile 是锁？

错误。

今天需要明确：

```text
volatile
≠
锁
```

它不会阻止另一个线程同时操作共享状态。

### 3. 原子性就是“读和写”？

不准确。

原子性强调的是：

> **一个操作是否作为不可分割的整体完成。**

### 4. synchronized 只是“加锁”？

不完整。

它至少提供：

```text
互斥
+
可见性
```

### 5. 80 万是最大执行次数？

不准确。

80 万次 `count++` 实际会被执行。

可能少的是最终累计结果。

### 6. 计数少了就代表请求失败？

错误。

真实请求成功数量与本地计数器是否正确，是两个不同问题。

---

## 十六、今天形成的并发问题判断模型

以后遇到共享状态并发问题，不应该先问：

> 加 volatile 还是 synchronized？

应该先判断：

```text
1. 这个状态是不是多个线程共享？

2. 当前线程之间有没有可靠同步关系？

3. 问题属于：
   - 可见性？
   - 原子性？
   - 互斥？
   - 顺序？
   - 多字段一致性？

4. 当前操作是不是复合操作？
   例如：
   - count++
   - check-then-act
   - read-modify-write

5. 是否需要一段业务逻辑整体互斥？

6. 状态作用域是不是只在单 JVM？

7. 多实例部署后这个状态还有没有业务意义？
```

再决定候选工具：

```text
volatile
synchronized
Atomic*
并发容器
Lock
更高层并发结构
外部聚合
```

---

## 十七、AI 并发修复方案的审核方式

AI 很适合做机械分析：

```text
扫描共享字段
↓
找所有读取点
↓
找所有写入点
↓
找线程池 / @Async / CompletableFuture
↓
找 ++ / -- / check-then-act
↓
找已有 synchronized / Lock
↓
找 Atomic / ConcurrentHashMap
↓
生成并发测试
```

工程师需要负责判断：

```text
这个状态是不是共享状态？
业务到底要求什么一致性？
需要可见性还是原子性？
是否需要互斥？
是不是多个字段共同维护不变量？
锁粒度是否合理？
有没有跨 JVM 问题？
吞吐与正确性应该如何权衡？
```

今天对 Codex 的审核原则可以总结成：

> **AI 可以发现并发风险，但工程师必须确认它修的是不是正确的问题层级。**

---

## 十八、并发修复的 Definition of Done

假设真的修复生产计数问题，不能只验证：

```text
项目启动成功
```

也不能只让 Codex：

```text
自己改
↓
自己检查
↓
自己说没有问题
```

至少应该确认：

```text
1. 明确共享状态和并发写线程来源

2. 明确业务要求
   精确计数 / 近似统计 / 最终聚合

3. 使用真实并发测试
   例如：
   8 线程 × 100000 次

4. 重复运行
   不能只跑一次

5. 如果业务要求精确
   最终结果必须稳定符合预期

6. 检查修复是否引入严重锁竞争

7. 在接近真实并发量下压测

8. 观察：
   吞吐量
   P95 / P99
   CPU
   线程等待

9. 多实例部署时单独确认统计语义

10. 准备监控和回滚方式
```

真正的上线条件应该是：

```text
并发正确性成立
+
业务统计语义明确
+
性能没有不可接受的回退
+
多实例边界清楚
+
有监控和回滚能力
```

而不是：

```text
Codex 说修好了
=
可以上线
```

---

## 十九、今天真正建立起来的能力

今天不是要求一次把 JMM 全部学透。

真正补上的第一层地基是：

```text
看到共享变量
↓
先问同步关系
↓
看到 volatile
↓
想到 happens-before / 可见性
↓
看到 count++
↓
想到 read-modify-write / 原子性
↓
看到 synchronized
↓
想到互斥 + 可见性
↓
看到 if (...) 再修改
↓
想到 check-then-act
↓
看到 AI 修改
↓
判断它修的是哪个并发层次
```

后续继续学习 CAS、AQS、Atomic 类、ConcurrentHashMap、CompletableFuture、线程池和 Virtual Threads 时，都可以建立在这套模型之上。

---

## 二十、今日总结

今天最重要的几个工程判断：

1. **JMM 关注的是线程之间哪些写入必须被哪些读取观察到。**

2. **happens-before 是线程间可见性和顺序保证关系，不等于现实时间上的简单先后。**

3. **volatile 可以建立线程间可见性与顺序保证，但不会提供互斥。**

4. **volatile 写之前的普通写，可以通过 happens-before 安全发布给读取到该 volatile 状态的线程。**

5. **count++ 是 read-modify-write 复合操作，volatile 不能防止 Lost Update。**

6. **synchronized 同时提供互斥和可见性，适合保护需要作为整体执行的临界区。**

7. **check-then-act 是典型复合并发错误，单靠 volatile 不能保证安全。**

8. **代码执行次数和最终累计结果不是一回事，并发覆盖会导致结果丢失。**

9. **AI 把字段改成 volatile，并不代表并发问题已经修复，必须判断真正缺的是可见性、原子性还是互斥。**

10. **单 JVM 线程安全不等于分布式多实例下业务语义正确。**

今天真正训练的能力不是背：

```text
volatile
synchronized
AtomicLong
```

而是：

> **看到并发代码时，能先判断共享状态、happens-before、可见性、原子性、互斥和状态作用域，再审核 AI 给出的修复方案是不是解决了真正的问题。**
