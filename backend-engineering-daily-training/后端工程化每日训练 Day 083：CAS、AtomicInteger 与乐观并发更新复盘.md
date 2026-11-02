# 后端工程化每日训练 Day 83：CAS、AtomicInteger 与乐观并发更新复盘

## 一、今天学习什么

Day 82 已经学习了 JMM、happens-before、volatile、synchronized，以及一个最关键的并发事实：

```text
count++
```

不是一个不可分割的操作，而可以粗略理解成：

```text
读取
↓
计算
↓
写回
```

今天没有继续扩展 AQS、ConcurrentHashMap、Virtual Threads 等更多并发知识，而是只向前推进一个核心机制：

> **CAS（Compare And Set / Compare And Swap）：只有当当前值仍然等于我预期的旧值时，才允许完成更新。**

AtomicInteger 只作为理解 CAS 和原子更新的具体 Java 工具。

今天暂时不展开：

```text
ABA
AQS
LongAdder
CAS CPU 指令细节
ConcurrentHashMap
无锁算法设计
```

今天的目标不是把 Java 并发一次学完，而是把“为什么 count++ 会丢更新”与“原子更新如何避免旧值覆盖”真正连接起来。

---

## 二、从 count++ 的 Lost Update 开始

假设：

```java
int count = 5;
```

两个线程都执行一次：

```java
count++;
```

如果它们交错执行：

```text
初始 count = 5

Thread A 读取 5
Thread B 读取 5

A 计算 5 + 1 = 6
B 计算 5 + 1 = 6

A 写回 6
B 写回 6
```

最终结果：

```text
count = 6
```

但业务预期是：

```text
count = 7
```

问题不是某一个线程“没有执行”。

两个线程都执行了。

真正的问题是：

> **两个线程都基于同一个旧值 5 计算出了 6，后一个写入没有意识到别人已经完成了一次更新。**

这就是今天理解 CAS 的入口。

---

## 三、CAS 的核心直觉

假设线程读取到：

```text
count = 5
```

它准备把值更新成：

```text
6
```

CAS 不会直接无条件写入，而是表达：

```text
我刚才看到的是 5。

现在真正修改之前，
再检查一次当前值是不是仍然等于 5。

如果仍然是 5：
    把它改成 6。

如果已经不是 5：
    说明别人修改过，
    我手里的旧值已经失效，
    本次更新失败。
```

可以抽象成：

```text
CAS(expected = 5, newValue = 6)
```

其中：

```text
expected
=
我之前读取到的旧值，
我预期它现在仍然应该是这个值

newValue
=
我现在准备写入的新值
```

今天最重要的一句话：

> **CAS 不允许一个线程拿着已经过期的旧状态，直接覆盖最新状态。**

---

## 四、两个线程同时 CAS 会发生什么

假设：

```text
count = 5
```

Thread A 和 Thread B 都先读取到了：

```text
5
```

它们都准备：

```text
CAS(5 → 6)
```

A 先执行：

```text
expected = 5
实际值   = 5
```

两者一致，所以：

```text
CAS 成功
count = 6
```

B 随后执行时，自己手里的旧值仍然是：

```text
expected = 5
```

但是共享变量真实值已经是：

```text
实际值 = 6
```

因此：

```text
expected != 实际值
↓
CAS 失败
```

B 不能继续把之前计算好的 6 无条件写进去，否则又会把 A 的那次更新效果覆盖掉。

它需要重新：

```text
读取当前值 6
↓
重新计算 7
↓
尝试 CAS(6 → 7)
```

如果这期间没有其他线程再次修改：

```text
CAS 成功
最终 count = 7
```

完整过程：

```text
初始 count = 5

A 读 5
B 读 5

A CAS(5 → 6)
成功
↓
count = 6

B CAS(5 → 6)
失败
↓
重新读 6
↓
计算 7
↓
B CAS(6 → 7)
成功

最终 count = 7
```

---

## 五、为什么 CAS 失败后必须重新读取

今天反复确认的一个核心点是：

> **CAS 失败，本质上说明“我刚才看到的状态已经过期了”。**

例如：

```text
expected = 8
实际值   = 9
```

这说明线程基于 8 计算出来的结果已经不再建立在最新状态上。

如果继续拿：

```text
expected = 8
```

原样重试：

```text
CAS(8 → 9)
CAS(8 → 9)
CAS(8 → 9)
```

只要实际值仍然不是 8，这些重试就没有意义。

正确思路是：

```text
CAS 失败
↓
重新读取最新状态
↓
基于最新状态重新计算
↓
再次尝试原子更新
```

因此 CAS 的关键并不只是“失败了再试”，而是：

> **失败后必须承认旧计算依据已经失效，并基于最新状态重新计算。**

---

## 六、CAS 与乐观锁思想

今天还区分了乐观并发与悲观并发的基本思路。

### 悲观锁

典型思想：

```text
我认为别人很可能和我竞争
↓
先获得独占权
↓
再修改
```

例如：

```java
synchronized
```

或者：

```java
ReentrantLock
```

典型行为：

```text
拿不到锁
↓
等待 / 阻塞
```

### CAS / 乐观并发

CAS 更接近：

```text
先读取状态
↓
准备更新
↓
真正提交时检查状态是否仍然和之前一样
↓
没变 → 更新成功
变了 → 本次失败，重新竞争
```

它体现的是乐观并发控制思想。

今天纠正的术语是：

> **CAS 本身不是 Java 的“关键字”，也不应该简单等同于一把传统意义上的锁。它是一种原子的比较并更新机制，常用于实现乐观并发更新。**

---

## 七、AtomicInteger 是什么

Java 中：

```java
java.util.concurrent.atomic.AtomicInteger
```

AtomicInteger 是一个类，不是 Java 关键字。

例如：

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

今天可以用下面这个概念模型理解它的原子自增：

```text
读取旧值 old
↓
计算 newValue = old + 1
↓
原子地尝试更新
↓
如果发现状态已经变化
    重新基于最新状态更新
```

用 CAS 模型表达就是：

```text
old = 5
newValue = 6

CAS(5, 6)
```

如果失败：

```text
重新读取 6
↓
计算 7
↓
再次尝试原子更新
```

这里需要保持一个边界：

> 这个 CAS 循环是今天用于理解 AtomicInteger 原子更新语义的模型，不应机械理解成所有 JDK 版本的具体源码都一定手写成完全相同的 CAS 循环。实际实现可能进一步使用 JVM / VarHandle / Unsafe / CPU 提供的原子读改写能力。

真正应该掌握的是：

> **AtomicInteger 提供的是单个整数状态的原子更新能力。**

---

## 八、volatile 为什么救不了 count++

Day 82 的知识今天再次被验证。

例如：

```java
volatile int count = 0;

count++;
```

volatile 可以提供：

```text
可见性
+
一定的有序性保证
```

但：

```text
count++
```

仍然是一个 read-modify-write 复合操作。

也就是：

```text
读取
↓
计算
↓
写回
```

多个线程仍然可能：

```text
A 读到 5
B 读到 5

A 写 6
B 写 6
```

所以：

```text
volatile
≠
让 count++ 自动变成原子操作
```

而：

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

提供的是原子更新语义，因此能够解决这个单变量自增场景中的 Lost Update。

今天重新建立的区别：

```text
volatile
→ 重点解决可见性 / 顺序关系

AtomicInteger
→ 可以提供单变量原子更新

synchronized
→ 可以通过互斥保护一整个临界区
```

---

## 九、AtomicInteger 的边界：单个原子操作不等于整个业务事务

训练中进一步出现了：

```java
AtomicInteger stock = new AtomicInteger(10);
AtomicInteger sold = new AtomicInteger(0);
```

一次卖货要求：

```text
stock - 1
同时
sold + 1
```

即使：

```text
stock.decrementAndGet()
```

本身是原子操作，

以及：

```text
sold.incrementAndGet()
```

本身也是原子操作，

也不能自动推出：

```text
stock - 1
+
sold + 1
```

这两个操作组合起来就是一个不可分割的整体。

例如：

```text
stock：10 → 9
↓
程序异常
↓
sold 仍然是 0
```

此时两个字段的业务不变量已经被破坏。

所以：

> **多个“各自原子”的操作组合起来，不自动等于一个更大的原子业务操作。**

这也是 AtomicInteger 的重要边界。

---

## 十、今天暴露出的事务与锁概念混淆

在讨论：

```text
stock - 1
sold + 1
```

时，最开始想到：

```java
@Transactional
```

这是今天暴露出的一个重要边界问题。

如果 stock 和 sold 只是 JVM 内存中的普通字段 / AtomicInteger：

```text
没有数据库修改
```

那么 Spring 的：

```java
@Transactional
```

不会因为方法中途抛异常，就自动把：

```text
stock = 9
```

恢复成：

```text
stock = 10
```

数据库事务与 JVM 内存并发控制不是同一个问题层次。

可以先建立最简单的区分：

```text
@Transactional
→ 主要处理受事务管理器控制的事务资源
→ 典型是数据库事务
→ 可以提供提交 / 回滚语义

synchronized / Lock
→ 主要解决线程之间的互斥访问
→ 控制谁能同时进入临界区
→ 不提供“异常后自动撤销内存修改”的事务回滚能力
```

例如：

```java
synchronized void sell() {
    stock--;
    throw new RuntimeException();
}
```

即使抛异常：

```text
stock 已经从 10 变成 9
```

synchronized 也不会自动恢复成 10。

今天因此纠正了一个容易混淆的判断：

> **互斥 ≠ 回滚。线程安全机制和事务回滚机制必须分层理解。**

---

## 十一、synchronized 与 CAS 的一个基础区别

今天没有深入 synchronized，只做了和 CAS 必要的对照。

### synchronized

如果线程 A 已经持有锁：

```text
A 进入临界区
↓
B 也尝试进入
↓
B 拿不到锁
↓
等待 / 阻塞
↓
A 释放锁
↓
B 再竞争锁
```

B 不是“调用失败”，而是通常需要等待锁释放。

### CAS

CAS 的核心模型则是：

```text
尝试原子更新
↓
发现 expected 与实际值不同
↓
本次更新失败
↓
重新读取 / 重新计算 / 再尝试
```

可以先粗略记成：

```text
synchronized：
先获得独占权，再进入临界区

CAS：
允许并发竞争，但旧状态不能直接覆盖新状态
```

今天不继续展开锁升级、Monitor、AQS、自旋优化等内容。

---

## 十二、今天答题中的几个关键纠正

### 1. B 会自动读到最新值？

最开始在 A 把 5 改成 6 后，容易直接认为：

```text
B 接下来就读 6，再加成 7
```

这里要区分：

```text
B 之前已经读取的局部旧值
```

和：

```text
共享变量当前真实值
```

如果 B 之前已经读取了 5，它手里的旧计算依据不会因为 A 更新了共享变量就自动变成 6。

B 必须在 CAS 失败后重新读取。

---

### 2. expected 是什么？

expected 不是：

```text
“我希望最终得到的值”
```

而是：

> **我之前看到的旧值，我预期共享状态现在仍然应该等于它。**

例如：

```text
CAS(expected = 8, newValue = 9)
```

如果真正执行时：

```text
实际值 = 9
```

则：

```text
8 != 9
↓
CAS 失败
```

---

### 3. AtomicInteger 是关键字？

不是。

```text
volatile
→ Java 关键字

AtomicInteger
→ java.util.concurrent.atomic 包中的类

CAS
→ 原子的比较并更新机制 / 操作思想
```

---

### 4. CAS 就等于乐观锁？

更准确地说：

> **CAS 常用于实现乐观并发控制，体现乐观锁思想，但 CAS 本身不是传统意义上那种“先获得一把锁再执行”的锁。**

---

### 5. synchronized 可以自动回滚？

不能。

```text
synchronized
→ 互斥

事务回滚
→ 另一类能力
```

不能因为一段代码被 synchronized 保护，就认为异常发生后内存状态会自动恢复。

---

### 6. 线程拿不到 synchronized 锁就调用失败？

不是。

典型行为是：

```text
拿不到
↓
等待 / 阻塞
↓
锁释放后再次竞争
```

---

## 十三、今天形成的并发更新判断模型

看到类似：

```java
count++;
```

以后可以先沿着下面的链路判断：

```text
这个变量是否被多个线程共享？
↓
是否存在并发修改？
↓
这个操作是不是 read-modify-write？
↓
多个线程会不会基于同一个旧值计算？
↓
是否可能发生 Lost Update？
↓
业务只需要单变量原子更新，
还是需要保护多个状态组成的业务不变量？
```

如果只是：

```text
一个 JVM 内
一个整数
要求精确原子自增
```

AtomicInteger 是一个合理候选。

如果是：

```text
多个字段必须作为整体保持一致
```

就不能因为每个字段分别是 AtomicInteger，就宣布整个业务线程安全。

如果涉及：

```text
数据库事务
跨 JVM
多实例
分布式共享状态
```

则还需要继续判断更高层的问题，不能把 CAS 当成万能方案。

---

## 十四、AI / Codex Review

假设 Codex 看到：

```java
private volatile int count;

public void increment() {
    count++;
}
```

它修改成：

```java
private final AtomicInteger count = new AtomicInteger();

public void increment() {
    count.incrementAndGet();
}
```

今天能够先做的第一层审核是：

```text
原问题是不是：
多个线程对同一个 JVM 内整数进行 ++，
并且业务要求不能丢更新？
```

如果答案是“是”，那么：

```text
volatile int + count++
→
AtomicInteger + incrementAndGet()
```

在“单变量原子自增”这一层，修复方向是合理的。

但不能因此继续推导：

```text
整个业务线程安全
整个系统线程安全
多实例统计一定正确
性能一定更好
```

AI Review 仍然要先确定：

```text
它修复的是哪个并发问题层级？
```

而不是只看：

```text
代码能不能编译
项目能不能启动
```

---

## 十五、今天的 Definition of Done

Day 83 没有追求继续扩展更多并发概念。

今天真正需要通过的只有四个核心问题：

### 1. 为什么两个 count++ 会丢一次更新？

因为两个线程可能同时读取同一个旧值，并分别计算出相同新值，后一次写入覆盖掉前一次更新效果。

### 2. CAS 的 expected 是什么？

是：

```text
我之前看到的旧值，
我预期它当前仍然应该是这个值
```

### 3. CAS 为什么会失败？

因为：

```text
expected != 当前真实值
```

说明共享状态已经被其他线程修改，我之前的计算依据已经过期。

### 4. CAS 失败后为什么要重新读取？

因为原来的旧值已经失效。

继续拿旧值计算或无条件写入，会重新制造 Lost Update。

需要：

```text
读取最新值
↓
重新计算
↓
再次尝试原子更新
```

今天这四个问题已经能够独立解释，并能在简单变化场景中判断。

因此 Day 83 的核心目标已经达到，没有为了凑题数继续扩展 AQS、ABA、LongAdder 等新概念。

---

## 十六、昨日核心知识回忆

最后回忆 Day 82 的核心点：

```text
volatile 能保证什么？
为什么仍然不能保证 count++ 线程安全？
```

今天能够回答：

```text
volatile 主要保证可见性，并提供相应的顺序保证；
但 count++ 是 read-modify-write 复合操作，
volatile 不会把这三个步骤合成一个原子操作，
因此多个线程仍然可能发生 Lost Update。
```

这里还纠正了一个口误：

```text
不是“可读性”
而是“可见性”
```

---

## 十七、今日总结

今天最重要的工程判断：

1. **count++ 线程不安全的核心是 read-modify-write 不是一个整体，多个线程可能基于同一个旧值更新。**

2. **CAS 的核心不是无条件写，而是只有当前真实值仍然等于 expected 时才允许更新。**

3. **expected 是之前读取到的旧值，不是最终希望得到的新值。**

4. **CAS 失败意味着旧计算依据已经失效，因此必须重新读取并重新计算。**

5. **CAS 体现乐观并发控制思想：允许竞争，但不允许过期状态无条件覆盖最新状态。**

6. **AtomicInteger 是类，不是关键字；CAS 也不是 Java 关键字。**

7. **volatile 解决可见性问题，但不能把 count++ 变成原子操作。**

8. **AtomicInteger 可以解决单个整数状态的原子更新问题，但多个 AtomicInteger 的组合操作不会自动变成一个原子业务事务。**

9. **synchronized 主要提供互斥；拿不到锁的线程通常等待，而不是简单“调用失败”。**

10. **synchronized 不提供异常后的内存状态自动回滚，不能和数据库事务能力混为一谈。**

今天真正建立起来的核心模型是：

```text
普通 count++
↓
多个线程可能读取同一个旧值
↓
发生 Lost Update
↓
CAS 在更新前检查旧状态是否仍然有效
↓
状态没变 → 原子更新
状态变了 → 失败
↓
重新读取
↓
重新计算
↓
再次竞争
```

今天真正训练的能力不是背：

```text
CAS
AtomicInteger
volatile
synchronized
```

而是：

> **看到并发更新代码时，能够判断一个线程手里的状态是否已经过期，理解为什么旧结果不能直接覆盖最新状态，并知道单变量原子更新与整个业务一致性不是同一层问题。**
