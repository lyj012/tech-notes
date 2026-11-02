# 后端工程化每日训练 Day 93：CompletableFuture exceptionally 异常恢复、thenApply 继续执行与业务成功误判复盘

> 日期：2026-10-24  
> 主线：Java 后端 × 异步任务 × AI 工程化 × AI 辅助开发的工程判断  
> 训练方式：V3.3，单主题、一次一题、先讲清前置概念再互动；依据当天真实问答复盘  
> 状态：互动训练完成；本文是学习记录，不是生产代码或实机验证报告

## 一、今天学习什么

上一正式训练日 Day 92 学习了：\`CompletableFuture\` 的上游阶段异常完成时，依赖正常结果的 \`thenApply\` 处理函数不会执行；Java 异步计算异常完成，也不代表数据库中的业务任务状态自动由 \`PROCESSING\` 变成 \`FAILED\`。

今天只向前推进一个核心机制：

> **\`exceptionally()\` 可以把上游的异常完成恢复为一个正常结果；恢复后的下游 \`thenApply()\` 因此可能继续执行，但“异步链正常完成”不代表“原始业务操作成功”。**

本次目标：

1. 从熟悉的 \`try-catch\` 认识“异常被处理后，程序为什么还能继续执行”；
2. 分清 \`catch\` 正常结束与再次 \`throw\` 的控制流差异；
3. 理解 \`exceptionally()\` 返回备用值后，下游 \`thenApply()\` 的执行条件；
4. 判断正常解析与解析失败恢复两条路径各自传递什么结果；
5. 审核 AI / Codex 把失败业务错误标为 \`SUCCESS\` 的方案；
6. 最后回忆 Day 92 的异常传播与数据库任务状态边界。

**范围限制：** 本日不展开 \`handle()\`、\`whenComplete()\` 等其他 API 的系统比较，也没有针对异常路径完成生产级设计。以下 PDF、订单、数据库和 Codex 代码均为**教学假设**；未运行 Java 程序、未连接数据库、未查看真实业务代码或真实 Codex 补丁。

---

## 二、前置概念：\`try-catch\` 捕获异常后，控制流往哪里走？

首先从同步代码开始，而不是一上来就考 \`CompletableFuture\`：

~~~java
String result;

try {
    result = parsePdf();
} catch (Exception e) {
    result = "默认文本";
}

saveResult(result);
~~~

假设 \`parsePdf()\` 因为 PDF 损坏而抛出一个能被此处 \`catch\` 捕获的异常。

执行路径：

~~~text
parsePdf() 抛出异常
    ↓
进入 catch
    ↓
把 result 设置为 "默认文本"
    ↓
catch 正常结束
    ↓
继续执行 saveResult("默认文本")
~~~

这里的“恢复”不是把损坏的 PDF 修好，而是用替代值继续后续流程。

如果 \`catch\` 中再次抛出异常：

~~~java
String result;

try {
    result = parsePdf();
} catch (Exception e) {
    result = "默认文本";
    throw new RuntimeException("解析失败");
}

saveResult(result);
~~~

那么 \`saveResult(result)\` 在这条路径下就不会执行，因为 \`catch\` 没有正常结束，当前方法的普通执行路径被再次抛出的异常打断。

要精确区分：

- 异常被捕获并处理，\`catch\` 正常结束 → 后续语句可以继续执行；
- \`catch\` 内再次 \`throw\` → 后续普通语句不会按原路径执行；
- “当前方法没继续往下走”不等于“整个 Java 程序或所有线程停止了”；
- 上述判断都有给定前提：没有其他异常、提前返回等改变流程的情况。

---

## 三、今天唯一的新核心 API：\`exceptionally()\`

从 Day 92 已经认识的 \`supplyAsync()\` 开始：

~~~java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> {
        return parsePdf();
    }).exceptionally(ex -> {
        return "默认文本";
    });
~~~

可以先用同步编程建立直觉：\`exceptionally()\` 与 \`catch\` 提供备用值的思路相似，但它处理的是 \`CompletableFuture\` 的**异常完成**，不是普通同步代码块的异常。

| 阶段 | 本次理解的职责 |
| --- | --- |
| \`supplyAsync(...)\` | 安排异步计算，产生正常结果或异常完成 |
| \`exceptionally(ex -> ...)\` | 上游异常完成时调用恢复函数；正常返回替代值，可使返回的新阶段正常完成 |
| \`thenApply(result -> ...)\` | 上游阶段正常完成时，使用其结果继续处理 |

**特别重要：要区分上下游两个阶段的状态。**

~~~text
解析阶段：parsePdf() 抛异常
    ↓
上游 CompletableFuture：异常完成
    ↓
exceptionally()：返回 "默认文本"
    ↓
恢复后的 CompletableFuture：正常完成，结果为 "默认文本"
~~~

原来的 PDF 解析没有成功；是后续阶段恢复为正常完成。

\`exceptionally()\` 不是无条件把结果变成默认值：

- 上游**正常完成**：恢复函数不执行，原正常结果继续传给下游；
- 上游**异常完成**，恢复函数正常返回备用值：返回的新阶段正常完成；
- 恢复函数本身又抛异常：不能再断言恢复后的阶段正常完成，需要考虑新的异常完成。

---

## 四、叠加 \`thenApply()\`：异常恢复为何可能掩盖失败？

示例：

~~~java
CompletableFuture<String> future =
    CompletableFuture.supplyAsync(() -> {
        return parsePdf();
    }).exceptionally(ex -> {
        return "默认文本";
    }).thenApply(result -> {
        saveResult(result);
        return result;
    });
~~~

在 \`saveResult()\` 自身不抛异常的前提下：

| 场景 | \`supplyAsync\` 结果 | \`exceptionally\` 的恢复函数 | \`thenApply\` | 保存内容 |
| --- | --- | --- | --- | --- |
| PDF 正常 | 返回真实文本 | 不执行 | 执行 | 真实文本 |
| PDF 损坏 | 异常完成 | 执行，返回“默认文本” | 执行 | 默认文本 |
| PDF 损坏且没有任何恢复环节 | 异常完成 | 不存在 | 处理函数跳过 | 不会由该处理函数保存 |

更准确地说，\`exceptionally\` 在上游正常完成时**不会调用它的异常恢复函数**，但它返回的阶段仍然可以正常完成，继续将真实结果交给 \`thenApply\`。不能把“跳过异常恢复函数”误解成“跳过剩下的整个异步链”。

由此引出核心工程风险：

> **技术层面的“正常完成”，可能只是得到了备用值；业务层面的“成功”，必须由真实业务条件决定。**

---

## 五、真实训练问答与纠错轨迹

下面只记录当天对话中实际发生的回答、纠错和验证，不把后来答对的内容回填成最初就掌握。

### 第 1 题：\`try-catch\` 后的保存语句还执行吗？

**问题：** \`parsePdf()\` 抛异常，被 \`catch\` 捕获并把 \`result\` 设置为“默认文本”，后面的 \`saveResult(result)\` 是否执行？

**我的首次回答（按口述整理）：**

> 不会。因为抛出了异常，被 catch 拿到了；最终 result 等于默认文本，但后面的方法就不会调用了。

**评估：首次判断错误。**

正确部分：知道异常进入 \`catch\`，且 \`result\` 被改为默认文本。

错误部分：把“进入 \`catch\`”误认为“必然终止后续执行”。实际上本题的 \`catch\` 正常结束，所以后面仍会执行 \`saveResult("默认文本")\`。

这是当天暴露的第一个前置知识缺口，不能记成“首次独立答对异常恢复”。

### 第 2 题：\`catch\` 中多一个 \`throw\` 会怎样？

**问题：** 若 \`catch\` 在赋备用值后又执行 \`throw new RuntimeException("解析失败")\`，是否继续调用 \`saveResult()\`？

**我的回答（按口述整理）：**

> 不会执行。因为 catch 里又 throw new RuntimeException，重新抛异常，后面的调用不会继续。

**评估：独立答对。** 能识别“捕获后正常结束”和“捕获后再抛出”这两种控制流的差别。

### 第 3 题：\`exceptionally()\` 返回备用文本，最终 future 是什么状态？

**问题：** PDF 解析失败，上游异常完成；\`exceptionally()\` 正常返回“默认文本”，最终 \`future\` 是异常完成还是正常完成？

**我的回答（按口述整理）：**

> 它有异常恢复逻辑，最终应该是正常完成，拥有默认文本这个结果；不过当时是猜的。

**评估：结论正确，但自述信心不足。** 经解释后建立了“原始解析失败 ≠ 恢复后的 future 失败”的区别。

当时口述中还把 \`exceptionally\` 的触发条件说反过一次；已纠正为**上游异常完成时执行恢复函数**。

### 第 4 题：\`exceptionally()\` 之后的 \`thenApply()\` 会执行吗？

**问题：** PDF 损坏后恢复函数返回“默认文本”，其后 \`thenApply(result -> saveResult(result))\` 是否执行，保存什么？

**我的回答（按口述整理）：**

> exceptionally 提供备用结果，变成正常返回；后面的 thenApply 能拿到 result，所以会正常执行，保存“默认文本”。

**评估：独立答对，因果链条清楚。** 这是本日真正开始从 API 语义推导后续执行路径的证据。

### 第 5 题：Codex 的 PDF 状态更新方案有问题吗？

**教学用候选方案：**

~~~java
CompletableFuture.supplyAsync(() -> {
    return parsePdf();
}).exceptionally(ex -> {
    return "默认文本";
}).thenApply(result -> {
    saveResult(result);
    updateTaskStatus("SUCCESS");
    return result;
});
~~~

业务约束：解析成功才能标记 \`SUCCESS\`；解析失败应标记 \`FAILED\`。

**我的回答（按口述整理）：**

> 这段不符合要求。PDF 损坏后返回默认文本，继续走 thenApply，结果仍然更新 SUCCESS，明明失败却被当作成功。

**评估：核心业务 Bug 独立识别正确。**

但随后分析正常路径时，曾误以为 \`parsePdf()\` 成功返回后，就不会继续执行后面的 \`thenApply()\`。这里还存在一个执行链理解错误。

**纠正点：**

- 解析成功：\`exceptionally\` 的恢复函数不执行，但真实结果继续传递，\`thenApply\` 仍执行；
- 解析失败并被恢复：备用结果继续传递，\`thenApply\` 也执行；
- 所以两条路径都可能执行 \`updateTaskStatus("SUCCESS")\`，其中失败路径的业务含义是错误的。

注意“执行了 UPDATE 代码”和“数据库实际成功持久化”仍是两个层面，后者在本日假设中才被给定为正常。

### 第 6 题：正常路径会打印什么？

**纠错题：**

~~~java
CompletableFuture.supplyAsync(() -> {
    return "真实PDF内容";
}).exceptionally(ex -> {
    return "默认文本";
}).thenApply(result -> {
    System.out.println(result);
    return result;
});
~~~

**我的回答（按口述整理）：**

> 正常返回“真实PDF内容”，exceptionally 的异常处理函数跳过，然后进入 thenApply，打印的是真实 PDF 内容。

**评估：纠错后独立回答正确。** 正常路径与异常恢复路径的边界得到进一步验证。

### 第 7 题：迁移到订单系统进行 AI Review

**题目代码：**

~~~java
CompletableFuture.supplyAsync(() -> {
    return createOrder();
}).exceptionally(ex -> {
    return "备用订单号";
}).thenApply(orderNo -> {
    updateOrderStatus("SUCCESS");
    return orderNo;
});
~~~

业务要求：创建订单成功记为 \`SUCCESS\`；失败记为 \`FAILED\`。假设 \`createOrder()\` 因数据库异常失败，但后续状态更新在教学条件下正常执行。

**我的回答（按口述整理）：**

> createOrder 失败抛异常，exceptionally 把异常恢复成正常结果，返回备用订单号；thenApply 继续执行，最后仍更新 SUCCESS。这明明失败了还当成功，我会拒绝这个方案。

**评估：独立迁移正确。** 说明本日理解不只局限于 PDF 场景，能够根据异常恢复机制识别另一类后端业务中的状态误判。

**现实边界：** 如果数据库故障仍存在，\`updateOrderStatus\` 本身也可能失败；本题只验证逻辑上会尝试错误的 \`SUCCESS\` 更新，不证明任何真实数据库最终落库。

---

## 六、AI / Codex Review：为什么这不是一个可接受的“修复”？

假设问题是：文件解析失败后，不希望任务一直停留在 \`PROCESSING\`。

某个看似简单的修改是在 \`thenApply()\` 前插入：

~~~java
.exceptionally(ex -> "默认文本")
~~~

这样确实能让后续依赖正常结果的处理函数继续执行，却同时引入新的业务风险：

~~~text
真实业务：解析失败
    ↓
技术动作：exceptionally 返回备用值
    ↓
异步链表现：恢复后的阶段正常完成
    ↓
thenApply 执行
    ↓
保存备用文本 + 更新 SUCCESS
    ↓
数据库可能把失败业务错误记录为成功
~~~

**审核结论：拒绝把“恢复了 CompletableFuture 异常”直接当作“解决了业务失败”。**

审查时应要求开发实现说明以下边界：

1. 业务的“成功”究竟需要哪些条件？是仅有非空字符串，还是解析结果真正有效？
2. 什么情况下允许使用默认值？默认值是合法业务降级结果，还是必须标为失败？
3. 失败路径会不会误执行成功保存和 \`SUCCESS\` 状态更新？
4. 失败状态更新由谁执行？是否确实可达，且能在失败场景独立持久化？
5. 成功写入、失败写入、数据库再次异常等路径，分别有什么可观察、可测试的结果？

**不预设唯一生产实现。** 可以根据真实业务选择显式结果类型、异常链处理、状态机和持久化边界等方案；但无论采用哪种方案，都必须让“技术状态”和“业务状态”含义一致。

---

## 七、Day 93 Definition of Done：按实际证据评估

| 验证项 | 当天证据 | 结论 |
| --- | --- | --- |
| 理解捕获异常后能继续执行 | 第 1 题首次误判，讲解后建立区别 | **纠错后理解** |
| 区分 \`catch\` 正常结束与再次 \`throw\` | 第 2 题独立正确 | 通过 |
| 理解 \`exceptionally()\` 返回备用值 | 第 3 题答对，但自述“猜的” | 基础理解建立，需后续巩固 |
| 理解异常恢复后 \`thenApply()\` 的执行条件 | 第 4 题能解释结果传递 | 通过 |
| 分清上游正常完成时的执行路径 | 第 5 题出现误判，第 6 题纠错后正确 | **纠错后通过** |
| 识别 PDF 失败被误标为 SUCCESS | 第 5 题独立拒绝候选方案 | 通过 |
| 迁移至订单创建场景审查 AI 代码 | 第 7 题独立判断并说明因果关系 | 通过 |
| 真实编译运行、数据库验证、真实 PR 审查 | 本次没有执行 | 未开展 |
| 独立设计生产级失败补偿与状态持久化 | 本次只做 Review，未开展系统设计验收 | 未开展 |

**阶段判断：** 对 \`exceptionally()\` 的异常恢复机制已建立基础理解，可以结合 \`thenApply()\` 识别简单的失败路径业务状态误判，并在订单场景做初步迁移。不能因此宣称熟练掌握 \`CompletableFuture\` 的全部异常处理 API、复杂多阶段组合、事务一致性或生产级容错设计。

本日的学习轨迹是**先错、经解释纠正，再实现跨场景迁移**，不是从一开始就全部答对。

---

## 八、Day 92 间隔回忆：没有异常恢复时会怎样？

本日新概念结束后，按 V3.3 规则回忆上一正式训练日 Day 92。

**回忆题：**

~~~java
CompletableFuture.supplyAsync(() -> {
    return parsePdf();
}).thenApply(result -> {
    updateTaskStatus("FAILED");
    return result;
});
~~~

数据库初始为：

~~~text
task_id = 1001
status = PROCESSING
~~~

已知 \`parsePdf()\` 抛异常，且没有任何其他状态更新代码。

**我的回答（按口述整理）：**

> thenApply 不会往下走，因为它需要上游正常返回结果。updateTaskStatus("FAILED") 不会执行，数据库不会自动变为 FAILED，依然保持 PROCESSING。

**评估：独立回答正确，Day 92 关键机制间隔回忆通过。**

两天的对照关系：

| 机制 | 上游解析失败后 | \`thenApply\` 处理函数 | 数据库业务状态 |
| --- | --- | --- | --- |
| Day 92：只有 \`thenApply\` | 异常完成并传播 | 不执行 | 不会自动由 PROCESSING 变 FAILED |
| Day 93：先 \`exceptionally\` 返回备用值 | 经过恢复的新阶段正常完成 | 可以执行 | **可能错误地写 SUCCESS**，取决于业务代码和数据库更新结果 |

请记住这两类故障不是一回事：

- **异常未处理**：失败路径上的业务状态更新可能根本不执行；
- **异常被不当恢复**：后续处理可能把失败误当成成功。

---

## 九、今天的真实训练轨迹

~~~text
收到 Day 93 初始题目
    ↓
先要求讲清核心概念，不直接答陌生 API
    ↓
从 try-catch 讲异常恢复
    ↓
第 1 题：catch 后是否保存
    → 首次误判“不会执行”
    → 解释 catch 正常结束，仍会保存默认文本
    ↓
第 2 题：catch 内再 throw
    → 独立判断不会继续执行
    ↓
第 3 题：exceptionally 返回备用结果
    → 判断正常完成正确，但信心不足
    ↓
第 4 题：恢复后 thenApply 是否执行
    → 独立正确，能解释默认值传递
    ↓
第 5 题：PDF 失败被 Codex 标记 SUCCESS
    → 独立发现业务 Bug
    → 同时暴露“正常解析后是否继续执行”的误解
    ↓
第 6 题：正常路径打印真实 PDF 内容
    → 纠错后独立正确
    ↓
第 7 题：订单创建失败 + 备用订单号
    → 独立识别错误的 SUCCESS 更新，拒绝方案
    ↓
Day 92 间隔回忆：无恢复时 thenApply 被跳过
    → 独立答对；数据库仍 PROCESSING
    ↓
Day 93 互动训练结束
~~~

---

## 十、值得复用的工程判断模型

> **异常恢复改变的是异步链后续阶段的完成结果，不会追溯改变原始业务操作已经失败的事实。**

以后审查 Java 异步链、AI 代码修复或任务状态异常时，先问：

~~~text
上游真实业务操作成功了吗？
    ↓
上游是正常完成，还是异常完成？
    ↓
异常是否被 exceptionally 转换成备用值？
    ↓
下游拿到的是业务真实结果，还是降级/替代结果？
    ↓
thenApply 是否会执行？
    ↓
SUCCESS / FAILED 的判定是否与业务事实一致？
    ↓
数据库状态更新是否实际执行并成功持久化？
    ↓
是否有成功、失败、恢复后错误标记、持久化失败的测试证据？
~~~

**一句话总结：异常恢复可以让异步流程继续正常完成，但不能把业务失败自动变成业务成功。**

**后续间隔复习建议：** 使用一个新业务场景，让自己独立区分“上游异常完成”“异常恢复为备用值”“下游继续执行”“数据库业务状态成功持久化”四个不同层面；基础稳定后再单独学习其他异常处理 API，不在同一天堆叠。