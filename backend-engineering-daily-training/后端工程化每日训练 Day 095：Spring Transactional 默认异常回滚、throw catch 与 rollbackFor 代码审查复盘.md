# 后端工程化每日训练 Day 95：Spring Transactional 默认异常回滚、throw catch 与 rollbackFor 代码审查复盘

> 日期：2026-10-26  
> 主线：Java 后端 × Spring 事务 × 异常控制流 × AI 辅助代码审查  
> 训练方式：V3.3，单主题、先解释概念和作用、一次一题、根据真实回答纠错与迁移  
> 状态：互动训练完成；本文为真实学习问答复盘，不是编译运行、数据库实测或真实 Codex PR 审查报告

## 一、今天学习什么

上一个正式训练日 Day 94 学了 `CompletableFuture.whenComplete()`：无论上游正常完成还是异常完成，都可以执行观察回调；“任务结束”不等于“业务成功”。

Day 92～94 连续学习了异步 API，今天切换回 Spring。Day 79 讨论过 `@Transactional` 的代理边界、短事务、外部调用和 `REQUIRES_NEW` 的风险；**今天不是重新学习“事务是什么”，而是补上一个具体的回滚判断机制**：

> **在 Spring 声明式事务的默认规则下，并不是 Java 方法抛出任何异常都会回滚；是否回滚取决于异常类型、异常是否传播到事务拦截器，以及实际配置和事务状态。**

今天仅围绕这一个核心机制展开，必要的前置知识是 Java 的 `throw` / `catch` 与受检异常、非受检异常的区别。

学习目标：

1. 说清 `RuntimeException` 与 `IOException` 的基本区别；
2. 区分“异常被抛出”和“异常被捕获”；
3. 判断 `@Transactional` 默认规则下运行时异常、受检异常对提交/回滚的影响；
4. 理解 `rollbackFor = Exception.class` 的作用和触发前提；
5. 在简化 Codex 候选代码中识别“异常被吞掉，回滚配置形同无触发条件”的风险；
6. 最后完成 Day 94 的 `whenComplete()` 旧知识回忆。

**范围和证据限制：** 本文所述订单系统、文件生成、事务和 Codex 代码均为教学设定。本次没有编译 Java，没有连接 MySQL、执行 SQL、实际观察 COMMIT/ROLLBACK，也没有读取真实 Codex 提交。文中“会提交/会回滚”均以题目给定前提及 Spring 常规规则为条件，不等于对真实生产系统的执行证明。

---

## 二、先复习已有基础：事务解决什么问题，为什么应当短

教学用订单业务需要创建订单和扣减库存。需要放在同一个数据库事务里的写入，在失败需要撤销时应当整体回滚，避免只完成一部分数据库改动。

Spring 的声明式事务通常由代理/事务拦截器、事务管理器和数据库资源共同完成；注解本身不是一条立即执行的数据库 `BEGIN` 命令。

本次问答中，我先主动复述了之前学过的知识（按口述整理）：

> `@Transactional` 是用来保障事务的；事务范围最好短一些，不要随便把第三方调用放进去，尽量只包需要事务保护的数据库操作。

**评估：** 这部分已有合理的工程直觉，与 Day 79 的学习方向一致，但“事务一致性”“要么成功要么失败”还不足以回答 Spring 何时触发回滚。

短事务的理由包括：避免在持有数据库连接、锁或事务资源时等待耗时且不可控的外部 HTTP/AI/文件操作。**“应当短”不等于只允许一条 SQL；事务边界应由需要原子提交的业务数据变化决定。**

---

## 三、今天必须弄清的 Java 异常前置概念

### 1. `throw` 是主动抛出异常，不是“只有 catch 才算出错”

~~~java
public void test() {
    throw new RuntimeException("出错了");
}
~~~

只要执行到 `throw`，异常就已经抛出。没有 `catch` 并不表示“没有异常”，而是表示这段代码没有在本地接住它，异常会沿调用栈向上传播。

- `throw`：抛出异常；
- `catch`：捕获匹配的异常并执行相应处理；
- 未捕获：异常确实发生，继续向外传播；
- 捕获后正常返回：异常发生过，但不再自动向外传播；
- 捕获后再 `throw`：异常仍然向外传播。

### 2. `RuntimeException` 和 `IOException`

~~~java
public void demo() {
    int value = 10 / 0; // 运行时抛出 ArithmeticException
}
~~~

`ArithmeticException` 是 `RuntimeException` 的子类。`NullPointerException`、`IllegalArgumentException` 等也属于该类异常体系。Java 编译器不强制对它们使用 `catch` 或 `throws` 声明。

~~~java
public void readFile() throws IOException {
    Files.readString(Path.of("order.txt"));
}
~~~

`IOException` 是受检异常（checked exception）；这类异常通常要求显式捕获或声明。受检和非受检分类代表 **Java 编译期处理规则不同**，不表示一种异常“更严重”或“更真实”。

### 3. 默认 Spring 回滚表

假设使用常规 Spring 声明式事务，代理正常生效，数据库操作参加同一事务，异常传出了事务方法，没有自定义规则，也没有其他回滚标记或资源层面故障：

| 向外传播的异常 | 默认事务处理 | 说明 |
| --- | --- | --- |
| `RuntimeException` 及其子类 | 回滚 | 非受检异常 |
| `Error` 及其子类 | 回滚 | Java `Error` 体系 |
| `IOException` 等受检异常 | 不因该异常回滚，通常提交 | 不是默认回滚触发条件 |

注意：**“`IOException` 会提交”只是上述简化条件下的结论，不是 `IOException` 这个异常自身会执行 COMMIT。** 实际结果仍依赖事务状态和提交是否成功。

---

## 四、第一题：未捕获的 IOException，订单会回滚吗？

### 题目

~~~java
@Service
public class OrderService {

    @Transactional
    public void createOrder() throws IOException {
        orderMapper.insert(order);
        throw new IOException("订单文件生成失败");
    }
}
~~~

已知：Spring 事务代理生效；InnoDB；`INSERT` 已执行成功但事务尚未提交；`IOException` 未被捕获；`@Transactional` 使用默认回滚配置。

### 我的首次回答（按实际口述整理）

> 事务不是要么一起成功、要么一起失败吗？现在抛出了异常，那么应该不会提交，应该回滚。

**评估：首次判断错误。** 我将事务原子性理解成了“发生任何异常就一定会触发 Spring 回滚”，忽略了 Spring 的默认异常回滚规则。

### 正确判断与纠错

~~~text
Spring 开启事务
  ↓
INSERT 成功（尚未提交）
  ↓
向外抛出 IOException
  ↓
Spring 检查默认回滚规则
  ↓
受检 IOException 不满足默认回滚条件
  ↓
在本题无其他故障的前提下，COMMIT
  ↓
订单保留在数据库；IOException 仍会继续向调用方传播
~~~

**核心教训：数据库 SQL 已执行成功 ≠ 已经提交；方法抛出了异常 ≠ Spring 一定回滚。**

我随后要求暂停答题，指出自己并未先理解 `RuntimeException`、`IOException`、`throw`、`catch`，如果直接不断做题只会猜测。于是从 Java 异常类别和异常传播开始重新解释。

**学习方式纠偏：先知道“是什么、解决什么问题、代码如何执行”，再做单题判断；不要把不懂术语时的猜测算成掌握。**

---

## 五、第二个误解：“没有捕获异常”被误认为“没有发生异常”

### 题目

~~~java
@Transactional
public void createOrder() {
    orderMapper.insert(order);
    throw new RuntimeException("订单创建失败");
}
~~~

条件与前面相同：事务正常生效；异常没有被捕获。

### 我的首次回答（按口述整理）

> 既然没有捕获异常，那不就说明没有异常吗？`INSERT` 成功了，应该保留订单。

**评估：再次误判，问题不在 `@Transactional`，而在对 Java 异常控制流的理解。**

纠正重点：

~~~text
throw new RuntimeException(...)
     ↓
异常已经发生
     ↓
没有 catch 接住
     ↓
异常向外传播到 Spring 事务拦截器
     ↓
匹配默认回滚规则
     ↓
ROLLBACK
~~~

紧接着用更小的题验证：

~~~java
public void test() {
    throw new RuntimeException("出错了");
}
~~~

问：没有 `try-catch`，究竟是“没有异常”，还是“异常已经发生但未被捕获”？

**我的回答（按口述整理）：**

> 肯定发生异常了，只是没有被捕获。我明白了，前面订单创建失败也是异常已经发生，只是没有被捕获。

**评估：纠错后的前置概念判断正确。** 这次能够将 `throw` 与 `catch` 分开理解，但不能把此前的两次首次误判改写成独立答对。

---

## 六、第三题：RuntimeException 被 catch 吞掉，订单还回滚吗？

### 题目

~~~java
@Transactional
public void createOrder() {
    orderMapper.insert(order);

    try {
        throw new RuntimeException("订单创建失败");
    } catch (RuntimeException e) {
        System.out.println("异常已捕获");
    }
}
~~~

已知：数据库写入成功；`catch` 自身正常执行；无显式 `setRollbackOnly`；无其他异常或事务状态问题。

### 我的回答（按口述整理）

> 这次应该保留订单。因为 `catch` 已经把异常捕获了，它不会从 `createOrder()` 方法里传出去，方法可以正常结束，所以会提交。

**评估：独立答对，而且能从异常传播范围解释原因。**

按题目条件：

~~~text
INSERT 成功
  ↓
throw RuntimeException
  ↓
catch 捕获并记录
  ↓
createOrder 正常返回
  ↓
Spring 未收到向外抛出的异常，也没有其他回滚标记
  ↓
COMMIT
~~~

关键：**异常“曾经发生过”不等于事务拦截器“最终收到未处理异常”。**

边界：如果底层数据库资源已经让事务处于不可提交状态，或者程序显式将事务标记为 `rollback-only`，即便异常在方法里被捕获，也不能保证最终提交。本题明确排除了这些情况。

---

## 七、第四题：rollbackFor = Exception.class 到底改变什么？

### 题目

~~~java
@Transactional(rollbackFor = Exception.class)
public void createOrder() throws IOException {
    orderMapper.insert(order);
    throw new IOException("文件生成失败");
}
~~~

`IOException` 没有被捕获，事务调用正常。

### 我的回答（按口述整理）

> 如果没有 `rollbackFor = Exception.class`，这个 `IOException` 按默认规则会提交。现在加了回滚配置，所以这次会回滚订单。

**评估：独立答对，已经能将默认规则和自定义规则对照使用。**

`rollbackFor = Exception.class` 的含义：为 `Exception` 及其子类增加回滚匹配规则，包括常见受检异常 `IOException`。原本默认就回滚的运行时异常仍然回滚；`Error` 也仍按默认规则回滚。

也可以更精确指定：

~~~java
@Transactional(rollbackFor = IOException.class)
~~~

这个配置不是“自动追踪方法内出现过的全部异常”；主要是**事务拦截器遇到匹配异常时应如何决定事务结局**。

---

## 八、AI / Codex Review：写了 rollbackFor，为什么订单还是可能被提交？

### 候选代码与业务前提

~~~java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    orderMapper.insert(order);

    try {
        generateOrderFile();
    } catch (IOException e) {
        log.error("文件生成失败", e);
    }
}
~~~

已知：

- `INSERT` 成功但尚未提交；
- `generateOrderFile()` 抛出 `IOException`；
- `catch` 只是记录错误，不重新抛出；
- 没有其他回滚标记、嵌套事务异常或数据库提交故障。

模拟 Codex 的声明：

> 我加了 `rollbackFor = Exception.class`，文件生成失败时订单一定回滚。

### 我的实际回答轨迹（按口述整理）

> `IOException` 被 `catch` 捕获了，没有离开 `createOrder()`。但上面又写了回滚，我感觉 Codex 说“必然回滚”很奇怪，可是一开始还不能完整说出为什么。

**评估：能自主察觉矛盾，尚不能独立给出明确、完整的 Review 理由；经讲解后理清逻辑。** 不能记录成“一眼独立完成 Code Review”。

再拆开执行链：

~~~text
INSERT 订单成功
   ↓
generateOrderFile() 抛出 IOException
   ↓
catch 捕获异常，只写日志
   ↓
方法正常返回
   ↓
rollbackFor 的异常触发条件并未在此调用路径满足
   ↓
Spring 在题目条件下提交事务
   ↓
订单在库，文件生成失败
~~~

### 纠错后的口头复述

> 回滚规则虽然写了，但是异常被 `catch` 捕获了，没有传播到外面的事务代理，因此不会自动触发这个回滚规则，订单依然会提交。

**评估：经过解释后能够正确复述因果关系。**

更严谨地表达：

> 不是 `rollbackFor` “根本无法执行”，而是 **`rollbackFor` 只定义异常匹配规则；当前没有符合规则的异常传播到事务拦截器，也没有其他强制回滚条件，所以 Spring 正常提交事务**。

### Code Review 结论

**拒绝“配置了 rollbackFor，所以捕获的 IOException 必然回滚”的说法。**

审核真实系统时应先查清业务需求：

1. 文件失败是否必须使订单数据库写入撤销？还是允许订单保存为“文件生成失败/待重试”状态？
2. 异常是被捕获后吞掉、重新抛出，还是转换为其他异常？
3. 事务代理是否真正生效？数据库写入是否处于同一个事务？
4. 是否有显式回滚标记、事务嵌套、其他失败状态或提交异常？
5. 文件是数据库外部副作用，数据库回滚能否撤销已经发生的文件操作？（**一般不能自动撤销**）
6. 如何用集成测试覆盖成功/失败分支并核验数据库最终数据？

**不是所有文件生成失败都应该简单回滚订单。** 是否需要数据库和文件一起实现可靠的业务结果，要根据业务成功定义、重试补偿与事务边界决定。今天只练习识别候选代码与其声明不符，不预设唯一生产修复方案。

---

## 九、今天最重要的规则对照

以下表格都假设 Spring 代理调用生效，数据库操作参加本事务，`INSERT` 已成功执行但尚未提交，且没有其他回滚标记、事务资源错误或规则覆盖。

| 场景 | 异常是否传到事务拦截器 | 最终结果（按题目条件） |
| --- | --- | --- |
| 默认 `@Transactional`，未捕获 `RuntimeException` | 是 | `ROLLBACK` |
| 默认 `@Transactional`，未捕获 `IOException` | 是 | `COMMIT` |
| 默认 `@Transactional`，`RuntimeException` 被方法内 `catch` 捕获并正常返回 | 否 | `COMMIT` |
| `rollbackFor = Exception.class`，未捕获 `IOException` | 是 | `ROLLBACK` |
| `rollbackFor = Exception.class`，`IOException` 被方法内 `catch` 捕获且正常返回 | 否 | `COMMIT` |

进一步要分清四层：

~~~text
① Java 发生了什么异常？
          ↓
② 异常是否沿调用栈传播到 Spring 事务拦截器？
          ↓
③ Spring 的回滚规则是否匹配，事务是否另有 rollback-only 等状态？
          ↓
④ 最终数据库事务成功提交还是回滚？（需要真实观测才能证明）
~~~

**异常、日志、方法返回、数据库最终持久化不是同一件事。**

---

## 十、Day 95 Definition of Done：按真实回答而不是讲解次数评估

| 验证项 | 本次真实证据 | 结论 |
| --- | --- | --- |
| 知道事务保护数据库操作且应控制范围 | 一开始主动复述短事务和外部调用风险 | 已有基础 |
| 默认规则下 `IOException` 不一定回滚 | 第一题误判为所有异常都会回滚；随后理解并能对比 | **首次未通过，经解释纠正** |
| `throw` 与 `catch` 的区别 | 曾将“未捕获”误认成“未发生”；最小题纠错后答对 | **纠错后通过** |
| 未捕获 `RuntimeException` 对事务的影响 | 第一次误判为提交；纠错后理解会回滚 | **纠错后理解，需后续间隔复测** |
| 方法内捕获 `RuntimeException` 后正常返回 | 独立判断订单会提交并解释原因 | **通过** |
| `rollbackFor = Exception.class` 对未捕获 `IOException` 的作用 | 独立正确判断回滚，并与默认行为对照 | **通过** |
| `rollbackFor` 与 `catch` 组合代码审查 | 自主发现冲突但当场未能讲完整，经提示后复述准确 | **部分通过，经引导完成** |
| Day 94 `whenComplete()` 间隔回忆 | 独立判断日志仍会打印，并说明成功失败都触发观察 | **通过** |
| 复杂事务规则、事务传播、rollback-only 与嵌套事务 | 仅提及边界，没有系统练习 | **未开展** |
| Java 编译、Spring 集成测试、MySQL 实测、真实 Codex PR 审查 | 没有进行 | **未开展** |

**本次结论：Day 95 核心规则经纠错后已建立 L2 基础理解，并完成简单迁移；代码审查有初步风险识别，但尚不能宣称已稳定掌握复杂事务失败边界或具备完整生产级事务验收能力。**

需要保留的薄弱点是：① 看到 `@Transactional` 时容易直接套“有异常就回滚”；② 曾把“异常没有被捕获”误解成“异常没有发生”；③ `rollbackFor` 与 `catch` 结合时，起初只能感觉不对而无法完整推导。后续应该间隔复测，不需要明天再重讲整套入门概念。

---

## 十一、最后回忆上一正式训练日 Day 94：whenComplete() 的观察语义

按 V3.3 规则，先完成今天的新知识问答与 Codex 简化审查，**最后**才安排上一正式训练日的核心知识回忆。

### 回忆题

~~~java
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("PDF解析失败");
}).whenComplete((result, error) -> {
    System.out.println("任务结束");
});
~~~

问题：PDF 解析失败后，`whenComplete()` 的日志还会打印吗？为什么？

### 我的回答（按实际口述整理）

> 肯定会打印。`whenComplete` 是观察前一个阶段的状态，无论成功还是失败，都会执行这个回调，打印任务结束。

**评估：独立回答正确，Day 94 本次间隔回忆通过。**

限定：

- 上游异常完成也会触发 `whenComplete` 的观察回调；
- 回调正常执行并打印“任务结束”，不会自动把上游失败恢复为成功；
- 这只是代表观察到了异步完成，不等于数据库里的业务状态已经更新；
- 实际程序需要让异步任务有机会执行完毕，才能观察到日志；主程序提前终止时不保证打印。

---

## 十二、今天实际的学习和纠错顺序

~~~text
开始：已学过 @Transactional 的基础用途和短事务
   ↓
Day 95 直接问 IOException 会不会回滚
   → 首次误判：发生异常就应该回滚
   ↓
主动要求停止猜题，先讲 RuntimeException / IOException / throw / catch
   ↓
听完解释后，能口头复述默认回滚规则
   ↓
未捕获 RuntimeException 的订单题
   → 再次误判：没捕获 = 没发生异常
   ↓
缩小到最小 throw 题
   → 理解异常发生与异常捕获不同
   ↓
try-catch 吞掉 RuntimeException
   → 独立判断正常返回 → 事务提交
   ↓
rollbackFor = Exception.class + 未捕获 IOException
   → 独立判断回滚，说明配置改变了默认规则
   ↓
Codex 候选方案：rollbackFor + catch(IOException) 只写日志
   → 自主感到矛盾，最初说不完整
   → 引导后正确说明：异常未传播、不会自动回滚
   ↓
最后回忆 Day 94 whenComplete()
   → 独立答对“解析失败仍执行完成日志”
   ↓
Day 95 互动训练完成，待本篇正式归档
~~~

---

## 十三、可复用的工程判断模型

> **Spring 事务的自动回滚规则看的是事务拦截器观察到的异常及回滚配置，而不是方法内部“曾经出现过异常”这一历史事实。**

以后 Review AI/Codex 写出的事务代码，可以沿着下面的因果链检查：

~~~text
业务要求是什么？
  ↓
这个方法是否真正进入 Spring 管理的事务？
  ↓
SQL 是否参与这一个事务？
  ↓
是否真的执行到 throw？是什么异常类型？
  ↓
异常有没有被 catch 吞掉、转成正常结果或重新抛出？
  ↓
默认回滚规则 / rollbackFor / 其他回滚标记分别是什么？
  ↓
事务最终是 COMMIT 还是 ROLLBACK？
  ↓
是否有实际集成测试证明数据库最终状态？
~~~

结合 Day 92～94 的异步任务训练，形成一个跨主题的共同判断：

**异常发生、异常传播、回调执行、业务成功、事务提交与数据库最终持久化，是不同层次的事实；不能只凭注解、日志或 AI 的解释就认定功能正确。**

### 后续轻量复习建议

未来使用库存扣减或账户转账的新例子，只考一处边界：**`RuntimeException` 被捕获后再次抛出，与捕获后正常返回，数据库提交/回滚有什么区别？** 在能独立解释后，再逐步进入事务代理失效、嵌套事务和实际集成测试。

### 官方参考

- Spring Framework — Rolling Back a Declarative Transaction：https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html
- Java Language Specification — Exceptions：https://docs.oracle.com/javase/specs/jls/se21/html/jls-11.html

**归档定位：** 本文记录一次存在真实误判、教学节奏调整、纠错、迁移和间隔回忆的 Day 95 学习过程；不把“老师讲过”包装成“从一开始就会”，也不把教学示例当成生产代码实测。
