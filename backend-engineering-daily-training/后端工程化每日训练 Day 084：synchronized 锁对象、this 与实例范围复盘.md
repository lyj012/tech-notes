# 后端工程化每日训练 Day 84：synchronized 锁对象、this 与实例范围复盘

## 一、今天学习什么

Day 83 已经从：

```text
count++
↓
Lost Update
↓
CAS
↓
AtomicInteger
```

建立了“单变量原子更新”的基础。

今天没有继续进入：

```text
AQS
ReentrantLock
ABA
LongAdder
ConcurrentHashMap
锁升级
Monitor 细节
```

而是继续补 synchronized 最基础、但非常容易在代码 Review 中判断错的一块：

> **synchronized 能不能让两个线程互斥，关键不是代码里有没有 synchronized，也不是两个线程是不是调用同一个方法，而是它们竞争的是不是同一个锁对象。**

今天真正建立的判断链是：

```text
共享状态是谁？
↓
哪些线程会访问它？
↓
每条访问路径锁的对象是谁？
↓
这些线程拿的是不是同一个锁对象？
↓
如果是同一个对象 → 形成互斥
如果不是同一个对象 → 互不影响
```

这是今天唯一需要真正掌握的核心。

---

## 二、第一种失败写法：synchronized(new Object())

最典型的错误代码：

```java
class Counter {

    private int count = 0;

    public void increment() {
        synchronized (new Object()) {
            count++;
        }
    }
}
```

表面上看：

```text
count++
外面已经有 synchronized
```

但这里每次进入方法都会：

```java
new Object()
```

产生一个新的对象。

例如：

```text
Thread A
↓
new Object()
↓
得到对象 A
↓
锁对象 A

Thread B
↓
new Object()
↓
得到对象 B
↓
锁对象 B
```

结果：

```text
对象 A != 对象 B
```

所以：

```text
A 可以进入
B 也可以同时进入
```

它们没有竞争同一把锁。

因此：

> **“代码里有 synchronized”不等于“真正形成了互斥”。**

---

## 三、同一个实例里，两把不同的锁不会互斥

训练第一题：

```java
class Counter {

    private int count = 0;

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void addA() {
        synchronized (lockA) {
            count++;
        }
    }

    public void addB() {
        synchronized (lockB) {
            count++;
        }
    }
}
```

前提：

```text
只有同一个 Counter 实例
```

Thread A：

```text
addA()
↓
锁 lockA
```

Thread B：

```text
addB()
↓
锁 lockB
```

今天能够正确判断：

> **B 不需要等 A。**

原因不是：

```text
方法名不同
```

而是：

```text
lockA != lockB
```

真正判断依据是锁对象。

---

## 四、不同方法也可以互斥

把代码改成：

```java
class Counter {

    private int count = 0;

    private final Object lock = new Object();

    public void addA() {
        synchronized (lock) {
            count++;
        }
    }

    public void addB() {
        synchronized (lock) {
            count++;
        }
    }
}
```

今天一开始容易产生的误区是：

> addA 和 addB 是不同方法，所以可能互不影响。

这个判断是错的。

真正关系是：

```text
Thread A → addA() → synchronized(lock)
Thread B → addB() → synchronized(lock)
```

虽然方法不同，但：

```text
两边锁的是同一个 lock
```

所以如果 A 已经持有 lock：

```text
B 尝试进入
↓
拿不到同一个 lock
↓
等待
↓
A 离开 synchronized 并释放 lock
↓
B 再继续竞争
```

今天形成的第一个核心结论：

> **是否互斥，不看方法是不是同一个，先看锁对象是不是同一个。**

---

## 五、同一把锁可以保护多个不同操作

同样的逻辑可以用于：

```java
class Counter {

    private int count = 0;

    private final Object lock = new Object();

    public void add() {
        synchronized (lock) {
            count++;
        }
    }

    public void reset() {
        synchronized (lock) {
            count = 0;
        }
    }
}
```

如果：

```text
Thread A 已经进入 add()
并持有 lock
```

此时 Thread B 调用：

```text
reset()
```

因为：

```text
add()   → synchronized(lock)
reset() → synchronized(lock)
```

所以 B 仍然必须等待。

这说明 synchronized 的互斥边界不是：

```text
方法
```

而是：

```text
锁对象
```

---

## 六、锁字段属于哪个实例非常重要

继续使用：

```java
class Counter {

    private int count = 0;

    private final Object lock = new Object();

    public void add() {
        synchronized (lock) {
            count++;
        }
    }
}
```

如果创建：

```java
Counter counterA = new Counter();
Counter counterB = new Counter();
```

实际上存在：

```text
counterA.lock
counterB.lock
```

虽然代码字段名都叫：

```text
lock
```

但它们不是同一个 Java 对象。

可以理解成：

```text
new Counter()
↓
创建 counterA
↓
同时创建 counterA 自己的 lock

new Counter()
↓
创建 counterB
↓
同时创建 counterB 自己的 lock
```

因此：

```text
Thread A → counterA.add() → 锁 counterA.lock
Thread B → counterB.add() → 锁 counterB.lock
```

结果：

> **B 不会因为 A 正在执行而等待。**

这就是今天第二个重要结论：

> **锁字段写在类里，不代表所有实例共享同一把锁。实例字段默认属于各自实例。**

---

## 七、今天重点纠正：this 不是“就近原则”

今天在第一次看到：

```java
synchronized (this)
```

时，对 `this` 的含义发生了混淆，最开始联想到：

> “this 是不是就近原则？”

这里进行了纠正：

> **Java 实例方法中的 this，表示当前正在调用这个实例方法的那个对象。**

例如：

```java
Counter counterA = new Counter();
Counter counterB = new Counter();
```

执行：

```java
counterA.add();
```

进入实例方法后：

```text
this == counterA
```

执行：

```java
counterB.add();
```

进入实例方法后：

```text
this == counterB
```

因此：

```java
public void add() {
    synchronized (this) {
        count++;
    }
}
```

实际锁的是：

> **当前这个 Counter 实例本身。**

---

## 八、synchronized(this) 在不同实例之间不会互斥

代码：

```java
class Counter {

    private int count = 0;

    public void add() {
        synchronized (this) {
            count++;
        }
    }
}

Counter counterA = new Counter();
Counter counterB = new Counter();
```

线程：

```text
Thread A → counterA.add()
Thread B → counterB.add()
```

进入方法后：

```text
A 的 this == counterA
B 的 this == counterB
```

所以：

```text
A → synchronized(counterA)
B → synchronized(counterB)
```

由于：

```text
counterA != counterB
```

因此不会互相等待。

---

## 九、今天最容易混淆的一点：线程不同，不等于 this 不同

后面出现：

```java
Counter c = new Counter();

Thread A → c.add()
Thread B → c.add()
```

而：

```java
public void add() {
    synchronized (this) {
        count++;
    }
}
```

最开始出现的错误判断是：

> A 是 A，B 是 B，所以它们的 this 可能不同。

这里把“线程”和“对象”混在了一起。

正确关系是：

```text
Thread A 调用的是 c.add()
↓
this == c

Thread B 调用的也是 c.add()
↓
this == c
```

所以：

```text
Thread A → synchronized(c)
Thread B → synchronized(c)
```

两个线程虽然不同，但它们访问的是同一个对象。

因此：

```text
A 已经拿到 c 的锁
↓
B 也需要 c 的锁
↓
B 等待
```

今天这一点经过多次同构题后已经能够正确迁移。

最终记忆：

> **线程不同 ≠ 锁对象不同。**

---

## 十、不同方法 + synchronized(this) 仍然可以互斥

代码：

```java
class Counter {

    public void add() {
        synchronized (this) {
            // ...
        }
    }

    public void reset() {
        synchronized (this) {
            // ...
        }
    }
}

Counter c = new Counter();
```

线程：

```text
Thread A → c.add()
Thread B → c.reset()
```

进入方法以后：

```text
A 的 this == c
B 的 this == c
```

所以：

```text
add()   → synchronized(c)
reset() → synchronized(c)
```

即使方法完全不同，也仍然竞争同一个对象锁。

因此：

> **A 已经持有 c 的锁时，B 调用 reset() 仍然要等待。**

再次说明：

```text
方法是否相同
不是核心判断依据

锁对象是否相同
才是核心判断依据
```

---

## 十一、实例 synchronized 方法锁的也是 this

今天继续把显式代码：

```java
public void add() {
    synchronized (this) {
        // ...
    }
}
```

迁移到：

```java
public synchronized void add() {
    // ...
}
```

对于普通实例方法，可以先建立这个理解模型：

```text
public synchronized void add()
≈
进入方法时锁住当前实例 this
```

例如：

```java
Counter c = new Counter();

c.add();
```

这里锁的是：

```text
c
```

因此两个线程：

```text
Thread A → c.add()
Thread B → c.add()
```

竞争的是同一个：

```text
c
```

A 已经进入时，B 必须等待。

---

## 十二、同一个 synchronized 实例方法，不同实例仍然不会互斥

代码仍然是：

```java
public synchronized void add() {
    // ...
}
```

如果：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

线程：

```text
Thread A → c1.add()
Thread B → c2.add()
```

则：

```text
A 锁 c1
B 锁 c2
```

所以：

> **B 不需要等待 A。**

最后的组合题：

```text
Thread A → c1.add()
Thread B → c1.add()
Thread C → c2.add()
```

如果 A 已经进入：

```text
B → 同一个 c1 → 等待
C → 另一个 c2 → 可以进入
```

这道题已经能够独立正确判断。

---

## 十三、今天真正建立起来的 synchronized 判断模型

以后看到 synchronized，不能只问：

```text
有没有加锁？
```

而应该按下面顺序判断：

```text
1. 被保护的共享状态是什么？

2. 哪些线程 / 哪些代码路径会访问或修改它？

3. 每条路径 synchronized 的具体锁对象是什么？

4. 这些竞争线程拿的是不是同一个 Java 对象？

5. 有没有路径使用了另一把锁？

6. 有没有路径完全绕过 synchronized？

7. 如果有多个业务实例 / Bean 实例，它们是否实际上拥有不同锁对象？

8. 当前状态是不是只存在于一个 JVM？
```

今天最重要的工程判断可以压缩成一句：

> **不要审核“有没有 synchronized”，要审核“所有竞争者是不是在争同一把锁”。**

---

## 十四、synchronized 的 JVM 范围边界

今天的核心机制继续带出了一个很重要的边界。

假设单 JVM 内：

```text
所有线程
↓
都访问同一个 Counter 实例
↓
都 synchronized(this)
```

那么它们可以通过同一个 Java 对象形成互斥。

但如果系统部署成：

```text
Java 服务实例 A
Java 服务实例 B
```

它们运行在两个 JVM 中。

即使代码完全一样：

```java
public synchronized void add()
```

也会变成：

```text
JVM A
↓
自己的 Counter 对象

JVM B
↓
自己的 Counter 对象
```

两个 JVM 不共享同一个 Java 对象。

因此：

> **单 JVM 中 synchronized 正确，不代表多个服务实例之间也能互斥。**

今天只建立这个边界，不继续展开分布式锁。

---

## 十五、今天答题中的几个关键纠正

### 1. 两个不同方法就不会互斥？

错误。

例如：

```text
add()   → synchronized(lock)
reset() → synchronized(lock)
```

如果 lock 是同一个对象，它们仍然互斥。

---

### 2. 同一个类里的 lock 字段天然是同一把锁？

错误。

如果：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

那么：

```text
c1.lock
c2.lock
```

是不同对象。

---

### 3. this 是“就近原则”？

错误。

在当前学习场景里：

```text
this
=
当前正在调用实例方法的对象
```

---

### 4. Thread A 和 Thread B 不同，所以 this 不同？

错误。

如果：

```text
A → c.add()
B → c.add()
```

两边的：

```text
this == c
```

线程身份和对象身份不是一回事。

---

### 5. synchronized 方法锁的是“这个方法”？

不准确。

普通实例 synchronized 方法可以理解成：

```text
锁当前实例 this
```

不是把“方法”本身当成独立锁对象。

---

### 6. A 释放锁必须理解成“主动退出”？

今天还纠正了一个措辞。

更准确是：

```text
A 执行完 synchronized 保护范围
↓
释放对应锁
↓
等待线程再竞争
```

重点不是“主动”这个词，而是同步块 / 同步方法结束后锁被释放。

---

## 十六、AI / Codex Review

以后 Codex 修并发问题，给出：

```java
synchronized (...) {
    // 修改共享状态
}
```

不能因为看到 synchronized 就接受。

至少先审核：

```text
1. 共享状态是谁？

2. synchronized 里面锁的是谁？

3. 所有修改这个状态的线程是否都走这里？

4. 所有这些路径锁的是不是同一个对象？

5. 是否存在不同实例导致每个实例各自一把锁？

6. 是否存在 synchronized(new Object()) 这种“每次一把新锁”的伪互斥？

7. 当前保护目标是单 JVM 内共享状态，
   还是多 JVM / 多实例共享业务状态？
```

例如 Codex 修改：

```java
public void increment() {
    synchronized (new Object()) {
        count++;
    }
}
```

应该直接识别：

```text
每次调用都创建新锁对象
↓
竞争线程没有争同一把锁
↓
并发问题没有真正解决
```

如果修改为：

```java
private final Object lock = new Object();

public void increment() {
    synchronized (lock) {
        count++;
    }
}
```

还不能立刻宣布整个系统安全。

还要继续确认：

```text
这个 Counter 是不是单例 / 同一个实例？
有没有多个 Counter 实例？
有没有绕过这把锁的修改路径？
系统是不是多实例部署？
```

---

## 十七、Definition of Done

Day 84 没有继续追求更多锁机制。

今天真正需要通过的是下面三个问题：

### 1. 为什么 synchronized(new Object()) 通常保护不了共享变量？

因为每次都会创建一个新的锁对象。

不同线程：

```text
各锁各的
```

没有形成竞争。

---

### 2. 为什么两个不同方法仍然可能互斥？

因为互斥取决于：

```text
锁对象
```

而不是：

```text
方法名
```

如果两个方法都锁同一个对象，它们仍然互斥。

---

### 3. 为什么 synchronized 的正确性还取决于对象实例范围？

因为实例字段和 `this` 都跟具体实例绑定。

```text
同一个实例
→ 可能共享同一把锁

不同实例
→ 通常是不同锁对象
```

今天经过同构迁移后已经能够正确判断：

```text
同一个 c1
→ A / B 互斥

另一个 c2
→ C 不受 c1 的锁影响
```

因此今天的核心知识已经达到“能解释 + 能简单迁移”的掌握闸门。

没有继续为了题数重复训练。

---

## 十八、训练节奏校准

今天训练过程中，同构题一度连续较多。

后面进行了节奏校准：

> **题目数量不是训练完成目标。只要核心知识已经能够解释和迁移，就不需要继续重复堆题。**

这与当前训练规则保持一致：

```text
先建立直觉
↓
出现错误
↓
只纠正一个核心误区
↓
做同构题
↓
能迁移
↓
及时停止重复
```

今天真正有价值的不是问了多少题，而是把两个主要误区纠正掉：

```text
方法不同 ≠ 锁不同

线程不同 ≠ 对象不同
```

---

## 十九、昨日核心知识回忆

最后回忆 Day 83：

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
```

问题：

> 为什么多个线程同时执行 incrementAndGet() 时，不会像普通 count++ 那样容易发生 Lost Update？

今天能够回忆出：

```text
AtomicInteger
↓
用于原子更新
↓
核心思想和 CAS 有关
↓
体现乐观并发控制思想
```

同时再次纠正：

```text
AtomicInteger
不是 Java 关键字
而是一个类
```

更完整的理解：

```text
读取旧值
↓
基于旧值计算
↓
原子比较并更新
↓
如果发现状态已经被别人改过
↓
本次更新失败
↓
重新读取最新状态并重试
```

所以它不会像普通：

```java
count++;
```

那样允许两个线程基于同一个旧值无条件覆盖最新结果。

Day 83 的核心知识回忆完成。

---

## 二十、今日总结

今天最重要的工程判断：

1. **synchronized 是否有效，关键不是“有没有 synchronized”，而是竞争线程是不是锁同一个对象。**

2. **两个不同方法只要锁同一个对象，仍然会互斥。**

3. **两个相同方法如果运行在不同实例上，也可能完全不互斥。**

4. **实例字段 lock 属于具体对象；new 两个 Counter，通常就有两个不同的 lock。**

5. **this 表示当前调用实例方法的对象，不是“就近原则”。**

6. **线程不同不代表 this 不同；两个线程调用同一个对象的方法时，this 可以完全相同。**

7. **synchronized(this) 锁的是当前实例对象。**

8. **普通实例 synchronized 方法可以理解为锁当前实例 this。**

9. **synchronized(new Object()) 每次产生新锁，通常无法保护多个线程共享的状态。**

10. **单 JVM 内 synchronized 成立，不代表两个 JVM / 两个服务实例之间也能互斥。**

今天真正建立起来的核心模型：

```text
看到 synchronized
↓
不要先看方法名
↓
找到锁对象
↓
找到对象实例
↓
判断竞争线程拿的是不是同一个对象
↓
同一个 → 互斥
不同 → 不互斥
```

今天真正训练的能力不是背：

```text
synchronized
this
lock
```

而是：

> **看到 AI 或真实代码中的加锁方案时，能够沿着“共享状态 → 访问路径 → 锁对象 → 实例范围”判断这把锁到底有没有真正保护到竞争线程。**
