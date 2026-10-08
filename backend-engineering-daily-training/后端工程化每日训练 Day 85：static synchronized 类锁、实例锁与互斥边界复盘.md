# 后端工程化每日训练 Day 85：static synchronized 类锁、实例锁与互斥边界复盘

## 一、今天学习什么

Day 82～84 已经沿着下面这条链路逐步补齐 Java 并发基础：

```text
Day 82：volatile、可见性、原子性
↓
Day 83：CAS、AtomicInteger、原子更新
↓
Day 84：synchronized、this、锁对象与实例范围
↓
Day 85：实例 synchronized 与 static synchronized 的锁对象区别
```

今天只补 Day 84 留下的一个自然缺口：

> **普通实例 synchronized 方法锁当前实例 this；static synchronized 方法锁当前类对应的 Class 对象，例如 Counter.class。**

今天依然不展开：

```text
AQS / ReentrantLock
Monitor 底层实现
锁升级
偏向锁 / 轻量级锁
ABA
可重入机制细节
```

最终只要能根据**具体锁对象**判断两个线程是否互斥，就达到本次训练目标。

---

## 二、真实问题：static synchronized 到底锁的是什么

先看一个类级操作：

```java
class Counter {

    public static synchronized void resetAll() {
        // 修改当前 JVM 内的某份类级共享状态
    }
}
```

两个线程：

```text
Thread A → Counter.resetAll()
Thread B → Counter.resetAll()
```

假设线程 A 已经进入 `resetAll()`，尚未退出。

那么 B 不能直接进入，因为两边争用的是**同一个锁对象**：

```java
Counter.class
```

过程可以理解成：

```text
A 进入 static synchronized resetAll()
↓
A 持有 Counter.class 这把锁
↓
B 也需要 Counter.class
↓
B 等待
↓
A 退出同步方法并释放锁
↓
B 才有机会获取锁并进入
```

注意，这里的等待是“等待同一把锁可用”，**不是调用失败**。

---

## 三、核心机制：实例锁与类锁

### 1. 普通 synchronized 实例方法

```java
class Counter {

    public synchronized void add() {
        // 修改当前实例的数据
    }
}
```

创建：

```java
Counter c1 = new Counter();
Counter c2 = new Counter();
```

锁对象分别是：

```text
c1.add() → 锁 c1（即该次调用中的 this）
c2.add() → 锁 c2（即该次调用中的 this）
```

因为：

```text
c1 != c2
```

所以两个线程分别进入 `c1.add()` 与 `c2.add()` 时，不会因为这些同步方法而互相等待。

### 2. static synchronized 方法

```java
class Counter {

    public static synchronized void resetAll() {
        // ...
    }
}
```

静态方法没有实例级的 `this`。

它使用的是：

```java
Counter.class
```

在本次讨论的同一 JVM、同一个类的前提下，无论从哪里调用 `Counter.resetAll()`，都会尝试获取该 Class 对象对应的锁。

可以建立一个直觉上的等价模型：

```java
public static void resetAll() {
    synchronized (Counter.class) {
        // ...
    }
}
```

关键区别：

| 方法形式 | 锁对象 | 判断关键 |
| --- | --- | --- |
| `public synchronized void add()` | 当前实例 `this` | 是否同一个实例 |
| `public static synchronized void resetAll()` | `Counter.class` | 是否争用同一个 Class 对象 |

> **两个 synchronized 是否互斥，不取决于方法名是否相同，而取决于锁对象是否相同。**

---

## 四、第一题：两个线程调用同一个 static synchronized 方法

题目：

```java
class Counter {
    public static synchronized void resetAll() {
        // ...
    }
}
```

```text
Thread A → Counter.resetAll()
Thread B → Counter.resetAll()
```

条件：A 已经进入，尚未执行结束。

**我的回答：B 必须等待。**

我能够指出：

- `static synchronized` 属于类级锁；
- 两个线程都在调用 `Counter.resetAll()`；
- 因为同一把锁的互斥性，A 没有释放前，B 不能进入。

这个结论正确。

但是当时有一个不够精确的说法：

> “static synchronized 锁的是整个类，范围更宽。”

应修正为：

> **它锁的是 Counter.class 这个具体对象，并不会让这个类的所有方法自动停止执行。**

只有同样需要争用 `Counter.class` 的同步路径，才会因这把锁而互斥。

---

## 五、第二题：实例同步方法与静态同步方法会互斥吗

代码：

```java
class Counter {

    public synchronized void add() {
        // ...
    }

    public static synchronized void resetAll() {
        // ...
    }
}

Counter c1 = new Counter();
```

两个线程：

```text
Thread A → c1.add()
Thread B → Counter.resetAll()
```

条件：A 已经进入 `add()`。

**我的最初判断：B 不需要等待。结论正确，但理由不对。**

当时解释为：

> “调用的方法不同，所以不是同一把锁。”

这里把**方法身份**误当成了**锁身份**。

真正关系是：

```text
Thread A → c1.add()           → 锁 c1
Thread B → Counter.resetAll() → 锁 Counter.class
```

因为：

```text
c1 != Counter.class
```

因此两者不会仅仅因为都使用 synchronized 就互斥。

需要纠正的是判断依据：

```text
错误：两个不同方法 → 两把不同锁

正确：两个不同锁对象 → 不会因该锁互斥
```

---

## 六、第三题：同一个实例，两个不同同步方法

为了纠正上一题的判断依据，继续使用一个更简单的反例：

```java
class Counter {

    public synchronized void add() {
        // ...
    }

    public synchronized void subtract() {
        // ...
    }
}

Counter c1 = new Counter();
```

线程：

```text
Thread A → c1.add()
Thread B → c1.subtract()
```

条件：A 已进入 `add()`。

**我第一次回答错误：认为“对象相同，但方法不同，所以锁不同”，B 可以进入。**

实际是：

```text
c1.add()      → 锁 c1
c1.subtract() → 锁 c1
```

**两个方法不同，但锁对象相同，B 必须等待。**

这题纠正了 Day 85 最关键的误区：

> **synchronized 不会按方法名称为每个实例方法自动分配一把独立的锁。**

对于普通 synchronized 实例方法，只要是在同一个对象上调用，争用的就是那个实例对象的锁。

---

## 七、第四题：Account 存钱与取钱，建立业务直觉

代码：

```java
class Account {

    public synchronized void deposit() {
        // 存钱
    }

    public synchronized void withdraw() {
        // 取钱
    }
}

Account account = new Account();
```

两个线程：

```text
Thread A → account.deposit()
Thread B → account.withdraw()
```

条件：A 已经进入 `deposit()`，尚未退出。

这次我能正确回答：

> **B 必须等待。**

因为：

```text
deposit()  → 锁 account
withdraw() → 锁 account
```

无论是存钱还是取钱，它们都通过同一个 `account` 对象进入普通 synchronized 实例方法，因此互斥。

但在回答时，我一度用“必须先存完钱再取钱，保证数据一致性”来解释等待。

这个业务直觉有帮助，但要和同步机制分开：

- **synchronized 提供的是互斥**：持有同一把锁时，其他需要该锁的线程无法同时进入。
- **synchronized 不规定存款一定先于取款**：如果 B 先拿到锁，B 可以先执行 `withdraw()`。
- **余额是否允许扣减，需要业务校验**：不能仅依赖 synchronized 自动保证余额充足或业务正确。

例如真实取款实现还需要在受保护的范围内检查余额与扣款条件。

> **同一把锁解决并发互斥；业务执行顺序和业务约束仍然要单独设计。**

这道题帮助我把“同一实例的不同同步方法仍然互斥”从抽象规则迁移到了业务场景。

---

## 八、第五题：返回实例锁与类锁，确认能否迁移

最后回到今天的核心：

```java
class Counter {

    public synchronized void add() {
        // ...
    }

    public static synchronized void resetAll() {
        // ...
    }
}

Counter c1 = new Counter();
```

线程：

```text
Thread A → c1.add()
Thread B → Counter.resetAll()
```

条件：A 已经进入 `c1.add()`。

**我的回答：B 不需要等待，因为锁对象不同。**

对应关系：

```text
A → c1.add()           → c1
B → Counter.resetAll() → Counter.class
```

判断正确。

这一轮我还指出，原题省略了 `c1` 的创建代码，导致对象来源不够明确。补齐 `Counter c1 = new Counter();` 后，判断条件才完整。

最终不再靠“方法不同”推导，而是明确说出两边各自的锁对象。

这说明前面暴露的误区经过同构题和业务题纠正后，已能在简单场景中迁移。

---

## 九、今天的锁对象判断表

默认在同一 JVM 内，且 A 已持有其所需锁，B 此时尝试进入：

```java
class Counter {

    public synchronized void add() {}

    public synchronized void subtract() {}

    public static synchronized void resetAll() {}
}

Counter c1 = new Counter();
Counter c2 = new Counter();
```

| 线程 A | 线程 B | A 的锁 | B 的锁 | B 是否因该锁等待 |
| --- | --- | --- | --- | --- |
| `c1.add()` | `c1.add()` | `c1` | `c1` | 是 |
| `c1.add()` | `c1.subtract()` | `c1` | `c1` | 是 |
| `c1.add()` | `c2.add()` | `c1` | `c2` | 否 |
| `Counter.resetAll()` | `Counter.resetAll()` | `Counter.class` | `Counter.class` | 是 |
| `c1.add()` | `Counter.resetAll()` | `c1` | `Counter.class` | 否 |

从这张表可以看到：

```text
同一方法 ≠ 必然互斥
不同方法 ≠ 必然不互斥

同一个锁对象 → 需要互斥
不同的锁对象 → 不会因为这两把锁而互斥
```

这里的判断只讨论锁本身，不排除业务代码中还有其他等待条件或同步机制。

---

## 十、两个必要边界

### 1. static synchronized 不是“整个类的方法全部上锁”

假设：

```java
class Counter {

    public synchronized void add() {}

    public static synchronized void resetAll() {}

    public void query() {}
}
```

当一个线程进入 `resetAll()` 时：

- 另一个线程调用同一类的 `static synchronized` 方法，若需要同一个 `Counter.class` 锁，就要竞争这把锁；
- 另一个线程调用 `c1.add()`，锁的是 `c1`，不会仅因为 `resetAll()` 持有类锁而等待；
- 普通未同步方法 `query()` 也不会自动因为类锁而被阻塞。

所以所谓“类锁”，应理解成**锁对象是 Class 对象**，不是“冻结这个类”。

### 2. static synchronized 不是分布式锁

假设部署两个独立的 Java 服务进程：

```text
JVM A → 自己的 Counter.class 和锁状态
JVM B → 自己的 Counter.class 和锁状态
```

它们不共享同一把 Java 对象锁。

因此：

> **在一个 JVM 内有效的 static synchronized，不会自动阻止另一个 JVM 执行相同代码。**

本次只确认单 JVM / 多 JVM 边界，不继续展开分布式锁方案。

---

## 十一、设计选择：锁范围与共享状态范围匹配

今天不把类锁理解成“更高级、更安全”的实例锁。

要先问：

```text
我要保护的状态属于谁？
↓
是某一个 Counter 实例自己的状态？
还是同一 JVM 中多个实例共同访问的一份共享状态？
↓
所有修改路径会不会争用同一把锁？
```

一般而言：

- 如果保护的是某个实例自己的状态，优先考虑实例级同步；
- 如果确实保护当前 JVM 内、类级别共享的一份状态，才考虑是否需要类级同步；
- 如果状态跨 JVM 共享，单靠实例锁或类锁都不足以构成跨进程互斥。

**Trade-off：锁范围扩大，会使更多原本可以并行的操作发生竞争。**

因此不能因为 `static synchronized` 看起来“范围更大”，就直接认为修复质量更高。

---

## 十二、AI / Codex Review：保留后续审核问题，不冒充已经完成

今天的训练目标主要是基本机制理解和简单迁移，**没有完整开展生产级 AI Review 或并发修复验收**。

以后如果 Codex 提议把：

```java
public synchronized void add() {
    // ...
}
```

改成：

```java
public static synchronized void resetAll() {
    // ...
}
```

不能简单接受“static synchronized 锁得更大，所以更安全”的理由。

真正需要审核的是：

```text
共享状态是什么？
↓
实际访问这份状态的路径有哪些？
↓
每条路径持有哪个锁对象？
↓
它们真的在竞争同一把锁吗？
↓
会不会放大无谓的锁竞争？
↓
系统是否存在多个 JVM？
```

这是一条**后续 Review 检查链**，不是今天已完成的生产验证。

---

## 十三、今天答题中的关键纠正

### 1. 两个 synchronized 方法不同，就不会互斥？

**错误。**

如果都由同一个对象 `c1` 调用，且都是普通的 synchronized 实例方法，它们锁的都是 `c1`。

### 2. static synchronized “锁住整个类”，其他方法都不能进入？

**不准确。**

它锁的是 `Counter.class` 对象，不是把类的全部方法无条件暂停。

### 3. 实例 synchronized 和 static synchronized 都写在 Counter 类里，就会互斥？

**错误。**

```text
实例 synchronized → c1 / this
static synchronized → Counter.class
```

两个锁对象不同。

### 4. 存钱和取钱互斥，所以同步机制保证一定先存再取？

**错误。**

互斥不等于业务顺序保证。哪个线程先获得锁，要看具体执行情况；余额和业务规则还需要独立校验。

### 5. 判断锁时，只看方法名就够了？

**错误。**

还要知道调用的是哪个实例，以及方法是普通实例同步方法还是静态同步方法。

---

## 十四、Day 85 Definition of Done 与实际评估

今天的目标只有三个：

| 掌握目标 | 本次证据 | 结果 |
| --- | --- | --- |
| 能解释普通 synchronized 实例方法锁 `this` | 能识别 `c1.add()` 锁 `c1` | 通过 |
| 能解释 static synchronized 锁 `Counter.class` | 第 1 题独立答对，并说明类级锁 | 通过 |
| 能判断实例锁和类锁是否为同一把锁 | 第 2 题结论对但理由错；第 5 题修正理由后答对 | 通过简单迁移 |

另外，同一实例不同同步方法的判断在第 3 题出错，在第 4 题通过业务场景纠正。

本次掌握程度：

> **核心机制达到 L2（理解），简单场景迁移通过；不把这一结果夸大为已掌握复杂锁设计或生产级 AI Review。**

本次训练可以结束，不再为了题目数量继续重复。

---

## 十五、昨日核心知识回忆：Day 84

按照训练规则，今日主题完成以后，最后只做一道 Day 84 回忆题。

```java
class Counter {
    public synchronized void add() {}
}

Counter c1 = new Counter();
Counter c2 = new Counter();
```

```text
Thread A → c1.add()
Thread B → c2.add()
```

条件：A 已进入，还没退出。

**我的回答：B 不需要等待。**

理由：

```text
A 锁的是 c1
B 锁的是 c2
c1 != c2
```

两个线程调用的是相同的方法，但使用的是不同的实例对象，因此不争同一把锁。

**Day 84 的核心知识回忆通过。**

---

## 十六、今日总结

今天没有继续扩展更多 Java 并发名词，而是补齐了 Day 84 后面最自然的一块：

1. **普通实例 synchronized 方法锁当前对象 this。**
2. **static synchronized 方法锁对应的 Class 对象，例如 Counter.class。**
3. **两个不同方法仍然可能争用同一把锁；不能用方法名判断互斥。**
4. **实例锁和类锁不是同一把锁，不会自动互相等待。**
5. **类锁不等于整个类全部方法都被暂停。**
6. **锁保证互斥，不自动保证存款先于取款，也不代替业务校验。**
7. **static synchronized 的互斥范围不是跨 JVM 的。**
8. **锁范围应与真实共享状态匹配，而不是越大越好。**

今天最终形成的判断模型：

```text
看到 synchronized
↓
先找共享状态
↓
再找锁对象
↓
实例方法？→ this / c1 / c2
静态同步方法？→ Counter.class
↓
两条竞争路径获取的是不是同一个对象？
↓
同一个 → 因该锁互斥
不同 → 不因这两把锁互斥
↓
最后再检查 JVM 边界
```

> **真正掌握的不是“static synchronized 更大”，而是能够明确说出每条线程正在争哪一个锁对象。**

后续训练可以从并发基础暂时切换到数据库内部机制，逐步补 Buffer Pool、Redo/Undo、WAL、MVCC 等长期缺口；下一次仍只引入一个可真正吸收的核心概念。
