# 后端工程化每日训练 Day 94：CompletableFuture whenComplete 完成观察、异步日志时序与业务状态误判复盘

> 日期：2026-10-25  
> 主线：Java 后端 × 异步任务 × AI 工程化 × AI 辅助开发的工程判断  
> 训练方式：V3.3，单主题、一次一题、先解释前置知识再互动；依据当天实际问答复盘  
> 状态：互动训练完成；本文是学习记录，不是生产代码或实机验证报告

## 一、今天学习什么

上一正式训练日 Day 93 学习了：`CompletableFuture.exceptionally()` 能在上游异常完成时返回备用结果，使其返回的阶段正常完成；但异步阶段恢复正常，不等于原始 PDF 解析等业务操作成功。

今天只增加一个核心机制：

> **`whenComplete()` 用来观察前一个异步阶段正常或异常完成的情况；回调正常执行时，它不会把上游失败自动恢复为成功。任务“结束”与任务“成功”是两件事。**

本日目标：

1. 理解 `whenComplete()` 的作用，以及为何不能用异步任务外的一条普通日志代替其完成时机；
2. 区分 `Thread.sleep()` 阻塞当前线程与 `supplyAsync()` 提交异步任务后的主线程继续执行；
3. 判断解析失败后 `whenComplete()` 是否执行、返回阶段是否仍异常完成；
4. 分清 `CompletableFuture` 的完成状态与数据库 `PROCESSING / SUCCESS / FAILED` 业务状态；
5. 审核 AI / Codex 无条件标记 `SUCCESS` 的错误方案；
6. 最后回忆 Day 93 的 `exceptionally()` 与 `thenApply()` 执行链。

**范围边界：** 今天不系统学习 `handle()`、`whenCompleteAsync()`、线程池调优、复杂阶段组合或生产级任务状态机。所有 PDF / 数据库 / Codex 代码均为教学假设；没有编译、运行 Java 程序，没有连接数据库，没有实际读取某个 Codex PR，也没有执行真实业务修复。

---

## 二、为什么不直接写一条 `log.info()`？

训练开始时，先讲解一个基础示例：

~~~java
CompletableFuture.supplyAsync(() -> {
    return parsePdf();
}).whenComplete((result, error) -> {
    log.info("PDF解析任务结束");
});
~~~

本题中：

- `supplyAsync()`：提交 PDF 解析的异步计算；
- `whenComplete((result, error) -> ...)`：注册前一阶段完成时的回调，不论正常完成或异常完成都可运行；
- `result`：正常完成时的结果（也可能合法地是 `null`）；
- `error`：前一阶段异常完成时的异常；正常完成时为 `null`。

我随即提出了一个实际的疑问：

> 既然已经有 `log.info()`，为什么还需要 `whenComplete()`？直接打印日志不就行了吗？

关键在于：**日志 API 负责记录信息，`whenComplete()` 负责把这段记录逻辑挂在“异步阶段完成”这个时点上。**

~~~java
CompletableFuture.supplyAsync(() -> parsePdf());

log.info("PDF解析任务结束");
~~~

上面这段代码中，主线程提交异步任务后通常可直接继续往下执行，因此 `log.info()` 可能在 PDF 实际解析结束之前执行。这条日志并不可靠地表示“解析已经结束”。

改成：

~~~java
CompletableFuture.supplyAsync(() -> parsePdf())
    .whenComplete((result, error) -> {
        log.info("PDF解析任务结束");
    });
~~~

才是把该日志挂到前一个异步阶段的完成事件上。

这不是说“日志必须写在 `whenComplete()` 中”。在异步任务内部使用适当的 `try-finally` 等方式，同样可以记录结束信息；`whenComplete()` 的价值在于**把业务计算与完成后的观察逻辑分离**。

---

## 三、前置知识纠错：`sleep(5000)` 会阻塞谁？

课堂为了检查对异步执行时序的理解，给出简化题：

~~~java
CompletableFuture.supplyAsync(() -> {
    Thread.sleep(5000);
    return "PDF解析成功";
});

log.info("任务结束");
~~~

题目假设解析要等待 5 秒，问：外面的日志是否也必须等 5 秒？

**我的首次回答（按口述整理）：**

> 那只能等五秒了呀。你都来了一个 `sleep` 函数，那只能等了。

**评估：首次误判。** 我把工作线程中的 `Thread.sleep()`，误认为会让主线程也等待。

纠错后的执行模型：

~~~text
主线程                         工作线程
   |                               |
   |-- supplyAsync 提交任务 -------->|
   |                               |-- 解析 / sleep(5000)
   |-- 继续执行 log.info("任务结束")  |
   |                               |-- 返回结果
~~~

- `Thread.sleep(5000)` 使**执行它的当前线程**休眠，而不是使 Java 应用中的所有线程休眠；
- `supplyAsync()` 通常不会等待异步计算结束，就把控制权交还调用线程；
- 因此，任务外面的日志可能比解析真正结束更早打印；
- 这里的“主线程”和“工作线程”是帮助理解的简化模型，不表示每个 `CompletableFuture` 操作都必然新建一个线程。

**代码严谨性备注：** 上述课堂示例中的 `Thread.sleep(5000)` 没有处理受检异常 `InterruptedException`，直接复制无法通过 Java 编译。它只用于讲解时序，不应作为可运行样例。能编译的教学写法可以是：

~~~java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException("解析等待被中断", e);
    }
    return "PDF解析成功";
});

log.info("主线程继续运行");
~~~

这些代码未在本次训练中实际运行。

---

## 四、真实互动问答与纠错轨迹

只记录本次真实回答中的判断，不将纠错后的答案伪装成首次就已掌握。

### 第 1 题：PDF 抛异常后，`whenComplete()` 还会打印日志吗？

~~~java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> {
        throw new RuntimeException("PDF解析失败");
    }).whenComplete((result, error) -> {
        System.out.println("任务已结束");
    });
~~~

**我的回答（按口述整理）：**

> 它会打印出来。无论成功还是失败，`whenComplete` 都会继续执行，它主要是观察成功或者失败，然后打印日志。

**评估：独立判断正确。** 理解了“观察前一阶段完成”与“必须成功才执行”不同。

### 第 2 题：外部日志是不是一定要等工作线程 `sleep(5000)`？

**我的首次回答：**

> 只能等五秒，因为有 `Thread.sleep(5000)`。

**评估：错误，经解释纠正。** 暴露出对异步提交与线程阻塞范围的混淆。本题纠错过程见第三节。

### 第 3 题：A（`whenComplete` 内）和 B（主线程外）谁通常先打印？

~~~java
CompletableFuture.supplyAsync(() -> parsePdf())
    .whenComplete((result, error) -> {
        log.info("A：PDF解析结束");
    });

log.info("B：主线程继续运行");
~~~

条件：PDF 解析实际需要约 5 秒。

**我的回答（按口述整理）：**

> `supplyAsync` 是异步任务，主线程和工作线程分开。主线程会继续执行 B，而 A 需要等解析完，所以大概率 B 更先打印。

**评估：纠错后独立答对，并且能够解释两个执行路径。**

注意“**通常 B 先**”不是 Java 对日志顺序的硬性保证；线程调度、主线程暂停、任务完成时机等都可能影响真实顺序。可靠的约束是：**A 的回调不早于它观察的前一个异步阶段完成。**

### 第 4 题：打印了“任务结束”，最终 `future` 是否成功？

条件：`parsePdf()` 抛出异常，`whenComplete()` 正常打印日志，没有其他恢复逻辑。

**我的回答（按口述整理）：**

> 肯定是异常完成。它只是观察状态，不会把解析失败修复成正常，只是把状态反映出来。

**评估：独立答对，并解释了原因。**

~~~text
parsePdf 抛异常
    ↓
上游阶段异常完成
    ↓
whenComplete 执行日志回调
    ↓
回调本身正常返回
    ↓
whenComplete 返回的阶段仍异常完成
~~~

**必须加上“回调本身正常执行”的限定。** `whenComplete()` 也返回一个新的阶段，回调自身若抛异常，可能影响这个阶段的完成状态；今天只掌握观察机制的正常回调路径，不展开全部异常优先级规则。

### 第 5 题：数据库是否自动从 `PROCESSING` 变成 `FAILED`？

~~~text
task_id = 1001
status = PROCESSING
~~~

~~~java
CompletableFuture.supplyAsync(() -> parsePdf())
    .whenComplete((result, error) -> {
        log.info("PDF解析任务结束");
    });
~~~

假设解析失败，没有任何其他数据库更新代码。

**我的回答（按口述整理）：**

> 肯定不会。你要修改数据库状态，就得有对应的修改逻辑。`supplyAsync` 是异步任务，`whenComplete` 只是观察，没有修改数据库的操作。

**评估：独立答对，能主动区分异步阶段状态和业务表状态。**

`whenComplete()` 可以**承载**数据库状态更新代码，但不会**自动替你生成和执行**数据库更新。即使写了更新方法，也需要另行验证数据库事务、连接、异常以及最终持久化结果。

### 第 6 题：Codex 无条件标记 `SUCCESS` 是否有 Bug？

业务要求：解析成功才允许标记 `SUCCESS`；解析失败应标记 `FAILED`。

**教学候选代码：**

~~~java
CompletableFuture.supplyAsync(() -> parsePdf())
    .whenComplete((result, error) -> {
        updateTaskStatus("SUCCESS");
        log.info("PDF解析任务结束");
    });
~~~

已知：PDF 损坏抛异常，数据库更新方法在题目假设下可以正常执行。

**我的回答轨迹（按口述整理）：**

> 最初口头说了一句“没有 Bug”，随后顺着代码重新分析：`whenComplete` 无论成功还是失败都会执行，`updateTaskStatus("SUCCESS")` 就会把失败也更新为成功。这是严重问题，我不会接受，必须区分成功与失败。

**评估：出现过即时口误 / 初判摇摆，但在同一次回答中自主纠正，并独立定位核心 Bug。**

随后又提出“不能把这个方法写在 `whenComplete()` 里面”的说法；这句话**不准确**。不是不能写在这里，而是不能**无条件标记成功**。

简化教学中的一种写法：

~~~java
.whenComplete((result, error) -> {
    if (error != null) {
        updateTaskStatus("FAILED");
    } else {
        updateTaskStatus("SUCCESS");
    }
});
~~~

以上仅表达“按上游异常与否分支”的思路。生产中还要确认“上游正常返回”是否满足业务成功定义，处理状态写入失败、幂等、重复回调、事务边界等问题。本次未开展该实现的完整设计与验证。

---

## 五、核心机制：`exceptionally()` 和 `whenComplete()` 的区别

| 机制 | `exceptionally()` | `whenComplete()` |
| --- | --- | --- |
| 主要用途 | 异常时提供备用结果或恢复逻辑 | 观察前一阶段的正常或异常完成 |
| 正常完成的上游 | 异常恢复函数不调用；正常值传递 | 执行完成回调，`error` 为 `null` |
| 异常完成的上游 | 恢复函数正常返回备用值，可使新阶段正常完成 | 执行完成回调；正常结束的回调不把失败恢复为成功 |
| 返回值特点 | 恢复函数可返回替代结果 | 回调使用 `BiConsumer`，不返回替代结果 |
| 典型工程风险 | 备用值被当成真实业务成功 | “完成日志”被误认为成功证明；回调无条件更新 `SUCCESS` |

今天最重要的三个边界：

1. **提交异步任务 ≠ 异步任务已经完成**；
2. **异步任务完成 ≠ 异步任务成功**；
3. **异步阶段完成状态 ≠ 数据库中的业务任务状态已经正确持久化**。

---

## 六、AI / Codex Review：识别“结束”被错误解释成“成功”

教学候选方案的故障链：

~~~text
原始业务：PDF 解析失败
    ↓
supplyAsync 对应的计算异常完成
    ↓
whenComplete 被触发
    ↓
无条件执行 updateTaskStatus("SUCCESS")
    ↓
数据库在题目给定条件下完成更新
    ↓
失败任务被错误记录为 SUCCESS
~~~

**Code Review 结论：拒绝。**

审核时至少需要追问：

1. 业务成功条件是什么？是否仅仅因为 future 正常完成就算成功？
2. 完成回调能否拿到前一阶段的异常信息，并分支处理？
3. 失败路径是否可能执行成功状态写入？
4. 数据库状态更新是否真的成功持久化，还是只执行了某行 Java 代码？
5. 日志中的“任务结束”是否被监控、前端或上下游误解成“任务成功”？
6. 更新失败、重试和重复更新时是否会造成状态覆盖？

**不凭空预设真实项目的唯一代码修改方案。** 如果将来交给 Codex，应该先让它查看实际业务定义、异步调用链和持久化方式，再设计修改并提供可复现的测试证据。

---

## 七、Day 93 间隔回忆：`exceptionally()` 恢复后是否继续 `thenApply()`？

在 Day 94 的新知识与 AI Review 完成后，按 V3.3 做上一正式训练日的单点回忆：

~~~java
CompletableFuture.supplyAsync(() -> parsePdf())
    .exceptionally(error -> {
        return "默认文本";
    })
    .thenApply(result -> {
        updateTaskStatus("SUCCESS");
        return result;
    });
~~~

条件：PDF 损坏抛异常；`exceptionally()` 正常返回备用文本；数据库更新在教学条件下可以成功。

**我的回答（按口述整理）：**

> `supplyAsync` 是异步计算；`exceptionally` 把异常完成恢复为正常结果，返回默认文本；`thenApply` 因而能够拿到结果继续运行，可能把失败任务更新成 `SUCCESS`。

**评估：独立答对 Day 93 的核心回忆。** 能从“异常恢复 → 下游正常处理函数执行 → 业务状态误判”解释风险。

需要保留精确表述：不是把原本损坏的 PDF 修复了，而是**恢复后的新异步阶段**正常完成了；原始 PDF 解析依旧失败。

两天对照：

| 训练日 | 上游失败后的处理 | 后续阶段结果 | 容易出现的误判 |
| --- | --- | --- | --- |
| Day 93 | `exceptionally()` 返回备用值 | 恢复后的阶段可以正常完成，`thenApply()` 可运行 | 把备用结果误当原业务成功 |
| Day 94 | `whenComplete()` 正常执行日志 | 不自动恢复上游异常完成 | 把“已结束”日志误当成功，或无条件写 `SUCCESS` |

---

## 八、Day 94 Definition of Done：按真实问答评估

| 验证项 | 实际证据 | 结论 |
| --- | --- | --- |
| 知道 `whenComplete()` 在成功 / 失败完成时都能执行回调 | 第 1 题独立回答并说明观察语义 | 通过 |
| 区分异步任务提交与主线程继续执行 | `sleep(5000)` 题首次误判等待范围 | **首次未通过，经纠错** |
| 根据执行路径比较 A、B 日志时序 | 随后的 A / B 题能主动说明两个线程及等待条件 | **纠错后通过** |
| 分清“完成日志”和 future 完成状态 | 第 4 题独立回答异常完成 | 通过 |
| 分清 future 状态和数据库业务状态 | 第 5 题独立指出没有数据库更新逻辑 | 通过 |
| 审查 Codex 无条件更新 `SUCCESS` | 第 6 题同一回答中自纠，定位失败误报成功 | **自主修正后通过** |
| 判断状态更新能否写进回调 | 曾说“不能写在这里”，经解释纠正为“可以写，但要分支” | **概念纠正，未单独复测** |
| 回忆 Day 93 异常恢复链 | `exceptionally` → `thenApply` → 错误 `SUCCESS` 的因果分析正确 | 通过 |
| 实机编译运行、真实任务状态持久化、真实 PR Review | 本日未进行 | 未开展 |
| 生产级异常回调、线程调度与状态一致性设计 | 本日未开展完整训练 | 未验证 |

**阶段判断：** 对 `whenComplete()` 的基本触发语义、“只观察不自动恢复失败”、异步日志时机和简单状态误判已建立理解；已能识别一类 AI 代码错误。线程阻塞边界曾首次误判，经过纠错后能在相近场景独立解释，但还需要之后跨场景回忆确认稳定性。不能据此宣称已掌握生产级 `CompletableFuture` 异常处理或异步任务状态一致性。

---

## 九、今天的真实训练轨迹

~~~text
收到 Day 94 题目
    ↓
要求先解释 whenComplete 的用途，不直接做陌生 API 的题
    ↓
理解“成功或失败都会触发完成回调”
    ↓
主动追问：为什么不用 log.info 直接打印？
    → 补齐完成时机与异步提交的区别
    ↓
sleep(5000) 题：
    → 首次误判“主线程也要等五秒”
    → 纠正 sleep 只阻塞执行它的当前线程
    ↓
A / B 日志顺序题：
    → 纠错后独立判断通常 B 先、A 后
    ↓
失败后 whenComplete 打印，future 是什么状态？
    → 独立答对：仍异常完成
    ↓
数据库 PROCESSING 是否自动变 FAILED？
    → 独立答对：没有更新逻辑就不会自动改变
    ↓
Codex 无条件 updateTaskStatus("SUCCESS")
    → 口头初判摇摆，在同次回答中自主发现失败误标成功
    → 纠正“不能在 whenComplete 写更新”的过度结论
    ↓
Day 93 间隔回忆
    → 独立解释异常恢复、thenApply 继续执行、业务成功误判
    ↓
Day 94 训练完成
~~~

---

## 十、最值得复用的工程判断模型

> **完成回调可以告诉你“上一阶段执行结束了”，不能单凭这件事证明“业务成功了”，更不能证明“数据库状态已经写对了”。**

以后审查 Java 异步任务、任务状态和 AI 生成的代码时，按下面的顺序拆解：

~~~text
异步计算何时提交？是否已经真正完成？
    ↓
它是正常完成、异常完成，还是在后续被恢复？
    ↓
whenComplete 观察到的 result / error 是什么？
    ↓
回调里只是日志，还是还有实际副作用？
    ↓
失败路径会不会误写 SUCCESS？
    ↓
FAILED / SUCCESS 更新是否真正执行并持久化？
    ↓
日志、数据库状态和业务真实结果是否一致？
    ↓
有没有可重复的成功 / 失败 / 更新失败测试证据？
~~~

**一句话总结：`whenComplete()` 让我们在异步阶段完成时执行观察逻辑；“观察到结束”并不意味着“异常已恢复”或“业务已成功”。**

**后续间隔复习建议：** 用不涉及 PDF 的订单或文件导出场景，独立判断“主线程日志、完成回调、异常恢复、数据库状态”分别何时发生；在基础稳定后，再单独学习回调自身抛异常时 `whenComplete()` 返回阶段的异常语义。
