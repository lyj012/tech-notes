# 后端工程化每日训练 Day 96：ThreadLocal 线程隔离、CompletableFuture 异步上下文传递与线程池复用风险复盘

> 日期：2026-10-27  
> 主线：Java 后端 × Java 并发 × 异步任务 × 后端工程化  
> 训练方式：V3.3；先讲概念和最小示例，再一次一题验证；按当天真实问答复盘  
> 状态：当日互动训练与上一正式训练日回忆均完成；本文为学习记录，非实机或生产验证报告

## 一、今天学习什么

上一正式训练日 Day 95 学习了 Spring `@Transactional` 的默认异常回滚规则、异常传播与 `rollbackFor`。Day 96 转向 Java 线程上下文，补一个新核心机制：

> **同一个 `ThreadLocal` 对象，在不同线程中访问到的是各自线程保存的值；值不会因切换到 `CompletableFuture` 的工作线程而自动传递。但线程池复用同一线程时，未清理的旧值可能被后续任务读到。**

今天的学习目标：

1. 解释 `ThreadLocal` 是什么，为什么适合保存线程内上下文。
2. 判断不同线程 `set/get` 的结果和从未赋值时的默认返回值。
3. 判断 `CompletableFuture.runAsync()` 切换线程后为什么可能读不到请求线程的用户 ID。
4. 理解先读取普通局部变量、再交给异步任务使用的传值方式。
5. 识别线程池复用和忘记 `remove()` 导致的残留上下文风险。
6. 最后回忆 Day 95 中 `IOException` 与 `@Transactional` 默认回滚规则。

**训练边界：** 今天没有研究 `ThreadLocalMap` 的完整内部实现、弱引用键与内存泄漏细节、`InheritableThreadLocal`、Spring Security 上下文传播组件，也没有编写或运行并发测试。所有请求、用户 ID 和线程 A/B 均为帮助理解的教学条件，不代表某个真实服务的执行记录。

---

## 二、先教学：ThreadLocal 是什么、解决什么问题？

假设 Java 后端同时处理两个用户请求：

~~~text
用户 1001 → 请求线程 A
用户 2002 → 请求线程 B
~~~

如果把“当前用户 ID”直接放到所有线程共享的普通可变字段中，就可能发生线程间覆盖。

`ThreadLocal<T>` 是 Java 的线程本地变量机制。**它不是给共享变量加锁，而是让每个线程对同一个 `ThreadLocal` 键拥有独立的取值。**

最小示例：

~~~java
public class Demo {
    private static final ThreadLocal<String> USER = new ThreadLocal<>();

    public static void main(String[] args) {
        USER.set("1001");
        System.out.println(USER.get()); // 1001
        USER.remove();
    }
}
~~~

本日先认识三个 API：

| 方法 | 含义 |
| --- | --- |
| `set(value)` | 在**当前线程**中保存该 `ThreadLocal` 对应的值 |
| `get()` | 从**当前线程**中读取对应的值 |
| `remove()` | 删除**当前线程**中该 `ThreadLocal` 对应的值 |

本例只有主线程，所以打印 `1001`。

关键直觉：`USER` 虽然被声明为 `static`，但这不意味着所有线程共用一个 `USER` 的值。共享的是 `ThreadLocal` 对象本身；每个线程的关联值是分开的。可以把 `ThreadLocal` 想象成“查找钥匙”，不同线程有各自的储物柜。

### 核心机制的简化图

~~~text
              同一个 ThreadLocal USER
                         │
            ┌────────────┴────────────┐
            │                         │
         线程 A                    线程 B
    USER → "1001"             USER → "2002"
~~~

更精确地说：每个 `Thread` 可以维护自己的 `ThreadLocalMap`；`ThreadLocal` 对象作为查找键参与定位该线程中的值。本日只需要掌握这个因果关系，不深入内部哈希结构。

---

## 三、真实问答：从线程隔离到异步传值

本日采用“先解释 → 演示 → 独立答题 → 反馈 → 小幅迁移”的节奏，五道新知识问题依次进行。以下“我的回答”均根据实际口述整理，保留初步犹豫、术语口误和自我纠正，不把它们改写成零失误的背诵。

### 第 1 题：两线程分别 set，A 读取谁的值？

~~~java
static ThreadLocal<String> USER = new ThreadLocal<>();

// 线程 A
USER.set("1001");

// 线程 B
USER.set("2002");

// 回到线程 A
System.out.println(USER.get());
~~~

**我的回答（按口述整理）：**

> 应该是线程 A 自己设置的值，因为 `ThreadLocal` 对不同线程隔离，线程 B 设置的值不会覆盖线程 A；口头把 1001 说成了 101。

**评估：因果解释正确；数字口误，正确输出为 `1001`。**

这题验证的不是“记住输出”，而是能说明为什么 A、B 使用同一个 `ThreadLocal` 对象，却不共用同一份值。

### 第 2 题：B 从未 set，直接 get 会怎样？

~~~java
static ThreadLocal<String> USER = new ThreadLocal<>();

// 线程 A
USER.set("1001");

// 线程 B：从未设置 USER，也未配置 initialValue
System.out.println(USER.get());
~~~

**我的回答（按口述整理）：**

> B 应该拿不到 A 的 1001，因为它们隔离。输出是 None；B 并没有设置自己的值。

**评估：判断逻辑正确，Java 术语纠正为 `null`，不是 Python 常见的 `None`。** 使用 `new ThreadLocal<>()` 时，当前线程尚未赋值的 `get()` 默认返回 `null`；如果自定义了初始值逻辑，则需按具体实现判断。

### 第 3 题：CompletableFuture 的工作线程能读取请求线程的值吗？

~~~java
static ThreadLocal<String> USER = new ThreadLocal<>();

public void process() {
    // 假设请求线程 A 执行
    USER.set("1001");

    CompletableFuture.runAsync(() -> {
        // 假设由另一个线程 B 执行，且 B 之前没有设置过 USER
        System.out.println(USER.get());
    });
}
~~~

**我的回答（按口述整理）：**

> 输出 `null`。因为普通 `ThreadLocal` 的值不会自动传递到异步工作线程；B 没有设置过 `USER`，所以获取不到 A 的 1001。

**评估：独立答对，并说明了跨线程边界。**

`CompletableFuture.runAsync()` 提交异步任务；如果工作在线程 B 执行，它调用的 `USER.get()` 查找的是 **B 的线程本地值**。今天题目特意设定 B 从未赋值，因此结果为 `null`。

工程限定：`runAsync()` 默认使用异步执行机制，通常由工作线程执行，不能据此断言“每次都新建一个线程”或“任意线程上必然是 `null`”。若复用的工作线程此前残留过 `USER`，反而可能读到旧值，正是第五题的风险。

### 第 4 题：传普通变量，和在 B 中重新 get，有什么区别？

先解释一遍：如果异步任务需要用户 ID，可以在请求线程中取出 ID，让 Lambda 捕获这个局部变量供异步任务使用，而不是要求工作线程重新读取请求线程的 `ThreadLocal`。

~~~java
USER.set("1001");                 // 线程 A
String userId = USER.get();      // A 中取出 "1001"

CompletableFuture.runAsync(() -> {
    // 假设线程 B 执行，B 从未设置过 USER
    System.out.println(userId);
    System.out.println(USER.get());
});
~~~

**我的实际回答轨迹（按口述整理）：**

> 起初顺口说第一个可能是 None，然后发现前面有 `String userId = USER.get()`，自己改正为：第一个打印 1001，第二个打印 null。第一个是 A 已经取出来、传给异步任务使用的普通值；第二个是 B 重新从自己的 `ThreadLocal` 中读取，所以不同。

**评估：出现短暂误判，随即在同一回答中自主纠正；最终结论和原因正确。**

两个输出在题目条件下依次为：

~~~text
1001
null
~~~

技术表述补充：这里不是 `ThreadLocal` 跨线程传播，而是 Lambda **捕获有效 final 的局部变量 `userId`**，异步代码读取这个已捕获的值。对于此处不可变的 `String` 用户 ID，适合作为简单的显式传值示例；复杂可变对象仍需考虑并发修改与线程安全。

如果真实业务采用此写法，还必须按请求线程的生命周期清理其 `ThreadLocal`；传了局部变量，不等于已经完成上下文清理。

---

## 四、第五题：线程池复用时为什么会出现“脏上下文”？

这一题是在基础线程隔离和异步传值通过后才引入的工程风险。

线程池会复用已有线程。`ThreadLocal` 的值跟**线程**绑定，而不是跟“一次请求”或“一次任务”自动绑定。

假设线程池只有一个工作线程 B：

~~~java
// 第一次任务，在线程 B 中
USER.set("1001");
// 任务结束，但没有 USER.remove()

// 第二次任务，仍由线程 B 执行
// 没有调用 USER.set()
System.out.println(USER.get());
~~~

**我的回答（按口述整理）：**

> 打印第一次任务设置的值，因为这两次用的是同一个线程 B。第一次没有清理，第二次又没有设置新值，自然可能取到上一次任务的值。应该用 `try-finally`，最后调用 `remove()` 清理。

**评估：独立答对，能从“同一线程复用 + 未清理 + 后一任务直接读取”的因果链说明问题。** 口述中仍把 `1001` 偶尔说成 `101`，但并非线程机制混淆。

正确输出：`1001`。

真实风险不仅是日志或数据不准确。如果业务误把旧 `userId` 当成当前用户身份，可能导致用户上下文串用，进而形成越权或数据泄露问题。是否实际发生安全漏洞取决于系统后续如何使用这个值，不能仅凭有残留就宣称已发生越权。

### 清理的基本写法

~~~java
private static final ThreadLocal<String> USER = new ThreadLocal<>();

public void handleTask(String userId) {
    USER.set(userId);
    try {
        // 在当前线程内执行业务
        doWork();
    } finally {
        USER.remove();
    }
}
~~~

`finally` 使正常结束或抛出普通异常时都能执行清理；它不保证在进程被强制终止等极端情况下仍运行。

**尤其要注意：必须在设置该值的线程的合适生命周期边界清理。** 请求线程 A 的 `USER.remove()` 不会替工作线程 B 清除 B 的值。

---

## 五、三个容易混淆的事实

| 场景 | 读到什么 | 原因 |
| --- | --- | --- |
| A 设置 1001，B 设置 2002，A 再 `get()` | `1001` | 不同线程存储各自的值 |
| A 设置 1001；B 从未设置，也无初始值 | `null` | 普通 `ThreadLocal` 不自动跨线程传值 |
| 同一线程 B 的任务 1 设置 1001，未清理；任务 2 直接读取 | `1001` | 线程池复用时旧值仍留在线程内 |

三句话分别对应：

1. **线程隔离**：A、B 互不覆盖。
2. **不自动传播**：B 不会自动继承 A 的普通 `ThreadLocal` 值。
3. **同线程残留**：B 也不会因为一个任务结束就自动清空自己的 `ThreadLocal`。

所以“线程隔离”和“任务隔离”不是一回事。

---

## 六、可复用工程写法：异步显式传值 + 各线程各自清理

以下仅是针对今天业务示例的最小设计思路，并非已经经过编译、压测、异常链路验证的生产模板：

~~~java
private static final ThreadLocal<String> USER = new ThreadLocal<>();

public void processRequest() {
    USER.set("1001");
    try {
        String userId = USER.get();
        CompletableFuture.runAsync(() -> sendNotification(userId));
    } finally {
        USER.remove();  // 清理当前请求线程的值
    }
}
~~~

这里区分两件事：

- `userId` 作为普通值由异步任务使用，不依赖线程 B 的 `USER.get()`。
- `finally` 清理的是执行 `processRequest()` 的请求线程自己的 `ThreadLocal`。

**不是说凡是异步任务都必须使用 ThreadLocal。** 如果需要的只是明确的业务参数（如用户 ID、订单 ID、租户 ID），优先考虑显式传参，更容易追踪数据来源。若确实需要传递日志 MDC、认证安全上下文或跟踪上下文，应依据具体框架选择经过验证的上下文传播机制，并确保在线程池工作线程执行结束后正确恢复或清理上下文。

这部分是今天知识的工程推论，**没有单独做 AI Review 或真实项目验收**，不计作已通过的高级能力。

---

## 七、以后 Review AI/Codex 代码时怎么检查？

如果 AI 生成了如下代码：

~~~java
USER.set(currentUserId);

CompletableFuture.runAsync(() -> {
    String userId = USER.get();
    sendNotification(userId);
});
~~~

不能只看它“有 `set()`，也有 `get()`”就认定正确。后续审查应追问：

1. `set()` 和 `get()` 到底在哪个线程执行？是否可能切换线程？
2. 异步任务需要的是哪一个确定的业务参数，能否显式传值？
3. 工作者线程是否可能复用？此前有没有留下旧 `ThreadLocal` 值？
4. 入口在哪里设置上下文，退出时是否在相同线程中 `remove()` 或恢复旧上下文？
5. 登录身份、租户等敏感上下文是否有可靠的认证与授权来源，而非仅依赖线程本地的旧值？
6. 如何测试“不同请求、线程复用、正常结束、异常结束”四种情况？

**本节是待实践的审查清单，不是今天已经完成的 Codex 代码审查。** 真正接入项目时，应先由能读取完整仓库的工具确认线程池配置、调用链和认证边界，再决定实现并用测试验证。

---

## 八、最后回忆上一正式训练日 Day 95：Spring 默认回滚规则

遵守 V3.3 的顺序：先完成 Day 96 新知识的逐题验证，最后才回忆 Day 95。

回忆题：

~~~java
@Transactional
public void createOrder() throws IOException {
    orderMapper.insert(order);
    throw new IOException("文件处理失败");
}
~~~

已知：事务代理生效，`insert()` 成功但尚未提交；异常向外抛出，没有自定义 `rollbackFor`、额外回滚标记或数据库提交故障。

**我的回答（按口述整理）：**

> 应该提交。`RuntimeException` 那一类异常默认回滚；`IOException` 这一种默认不会因为异常而回滚，只是我当时忘了它的正式分类名称。

**评估：独立答对 Day 95 的回忆；忘记的术语是“受检异常（Checked Exception）”。**

| 异常类型 | `@Transactional` 默认规则 |
| --- | --- |
| `RuntimeException` 及其子类 | 回滚 |
| `Error` 及其子类 | 回滚 |
| `IOException` 等受检异常 | 默认不因该异常回滚，满足题目其他条件时尝试提交 |

因此，题目中订单通常保留，`IOException` 仍继续向调用方传播。**“方法抛异常”不能直接推导“数据库已回滚”。**

如果希望这类受检异常触发回滚，可以在满足业务需求时使用 `@Transactional(rollbackFor = Exception.class)` 等规则；但如果异常在事务方法内部被捕获且方法正常返回，又没有其他回滚条件，不能认为这项配置必然让事务回滚。

这次是间隔记忆验证，不表示已重新验证 Day 95 的全部事务边界。

---

## 九、Day 96 Definition of Done：按真实回答评估

| 验证项 | 本次实际证据 | 结论 |
| --- | --- | --- |
| 理解不同线程的 `ThreadLocal` 值互不覆盖 | 第 1 题说明 A 与 B 独立 | 通过；数值口误 |
| 理解未设置默认返回 `null` | 第 2 题说明 B 无值，口头说成 None | 通过；术语已纠正 |
| 判断异步线程无法自动读取 A 的普通 `ThreadLocal` 值 | 第 3 题独立说明跨线程隔离 | 通过 |
| 区分 Lambda 捕获局部值与工作线程重新 `get()` | 第 4 题起初犹豫，随即自主纠正 1001 / null | 自主纠正后通过 |
| 识别线程池复用、未清理导致旧值残留 | 第 5 题独立说出因果并提出 `try-finally/remove` | 通过 |
| 回忆 Day 95 受检异常默认不回滚 | 正确判断 `IOException` 情形通常提交 | 通过；忘记术语 |
| 理解 `ThreadLocalMap` 内部结构、弱引用与内存泄漏 | 仅介绍最小原理 | 未系统训练 |
| 真实 AI / Codex PR Review | 只提供了后续审查清单 | 未开展 |
| Java 编译、并发测试、线程池复用实测、Spring Security 上下文验证 | 本日没有操作 | 未开展 |

**阶段结论：** Day 96 `ThreadLocal` 基础机制达到 L2（可用自己的话解释并完成简单迁移）；对线程池残留具有初步工程风险判断能力。不能仅根据五道教学题宣称已经达到完整的 L3 失败模式设计、L4 架构取舍或 L5 AI Review 与验收能力。

本日五道新知识问题最终判断均正确，其中第四题出现即时自我纠正；另有“1001/101”和“null/None”的口述小错误。记录它们，是为了区分真正的机制误解与术语、数字口误。

---

## 十、今天实际的学习轨迹

~~~text
对 Day 95 训练提出反馈：
不能只出题，必须先解释概念、作用、机制和最小示例
   ↓
Day 96 先介绍 ThreadLocal 与线程本地储值
   ↓
第 1 题：A/B 各自 set，A get
   → 独立解释线程隔离，数字口误
   ↓
第 2 题：B 从未 set，B get
   → 判断无值；修正 Java 术语为 null
   ↓
第 3 题：CompletableFuture 异步任务在线程 B get
   → 独立解释不会自动传播，输出 null
   ↓
第 4 题：先取 userId，再在异步线程分别打印 userId / USER.get()
   → 开始犹豫；自主修正为 1001 / null
   → 理解显式传值与线程本地读取不同
   ↓
第 5 题：同一线程 B 先 set、不 remove，再复用
   → 判断会读到上次的 1001
   → 主动提出 try-finally + remove
   ↓
最后回忆 Day 95：@Transactional 默认 IOException 处理
   → 判断符合题目条件时提交，回忆通过
   ↓
Day 96 完成互动学习，整理本文归档
~~~

---

## 十一、最值得复用的工程判断模型

> **`ThreadLocal` 解决的是“不同线程的数据相互隔离”，不是“不同任务的数据自动隔离”，也不是“异步调用自动传播上下文”。**

以后遇到线程上下文问题，用这个顺序推导：

~~~text
当前数据到底存在普通局部变量、共享字段，还是 ThreadLocal？
   ↓
set() 在哪个线程执行？
   ↓
get() 在哪个线程执行？
   ↓
两个操作是否确实发生在同一线程？
   ├─ 否：是否明确传参或采用受控上下文传播？
   └─ 是：该线程是否曾执行其他任务，可能残留旧值？
   ↓
该线程生命周期的哪个边界负责清理或恢复上下文？
   ↓
是否有线程切换 / 线程复用 / 异常结束的测试证据？
~~~

**一句话总结：跨线程需要显式考虑传值，同线程复用需要显式考虑清理。**

### 后续轻量复习方向

下一次涉及异步线程上下文时，用一个“请求线程 A → 线程池 B → 线程池 B 被复用”的场景，检验是否能在没有提示的情况下同时说明“为什么 A 的值不自动传给 B”和“为什么 B 的旧值会留下来”；再进入具体框架级上下文传播方案与验收。

### 参考资料

- Java SE `ThreadLocal` API：https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html
- Java SE `CompletableFuture` API：https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/CompletableFuture.html
- Spring Framework 事务回滚规则：https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html

**归档定位：** 本文是 2026-10-27 的真实互动学习和纠错复盘；没有执行生产修改、运行线程池实验或验证真实 AI 代码。