# 后端工程化每日训练 Day 80：Spring IoC、Bean 生命周期与 AI 装配方案审核复盘

## 一、今天学习什么

今天继续 Spring 主线，承接 Day 79 的事务代理问题，补上更底层的一层：

**Spring IoC / DI、Bean 创建与依赖解析、Bean 生命周期、BeanPostProcessor、AOP Proxy，以及 AI 修改 Bean 装配问题时如何做工程审核。**

Day 79 已经知道：

```text
@Transactional
↓
Spring Proxy
↓
事务拦截器
↓
事务管理器
```

今天继续往前追：

> **这个被代理的 Bean，本身到底是怎么被 Spring 创建和管理出来的？**

核心链路可以先建立成：

```text
@Component / @Service / @Bean
        ↓
注册 BeanDefinition
        ↓
解析 Bean 依赖关系
        ↓
创建 Bean
        ↓
依赖注入
        ↓
初始化回调
        ↓
BeanPostProcessor
        ↓
必要时生成 Proxy
        ↓
最终交给业务代码使用
```

今天真正要留下的不是某几个 Spring 注解的记忆，而是：

> **知道一个对象为什么能成为 Spring Bean、依赖为什么能被注入、代理为什么会生效，以及 AI 的“能启动”修改是否真的解决了根因。**

---

## 二、BeanDefinition 不等于 Bean 对象

Spring 扫描到：

```java
@Service
public class OrderService {
}
```

不能简单理解为：

```text
看到 @Service
↓
立刻 new OrderService()
```

更准确的模型是：

```text
扫描组件
↓
注册 BeanDefinition
↓
后续根据定义创建 Bean
```

BeanDefinition 可以粗略理解为 Bean 的“说明书”，里面描述：

```text
Bean 类型
Bean 名称
作用域
是否懒加载
构造方式
依赖关系
初始化方式
...
```

所以：

```text
BeanDefinition
≠
已经创建好的 Java 对象
```

这个区别非常重要，因为后面的依赖解析、生命周期和代理增强都建立在 Spring 容器对 Bean 的管理之上。

---

## 三、构造器注入本质是在解析依赖图

例如：

```java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring 创建 `OrderService` 时，不能只执行：

```java
new OrderService();
```

因为构造器明确要求 `PaymentService`。

真实过程更接近：

```text
准备创建 OrderService
↓
发现构造器需要 PaymentService
↓
去容器里解析 PaymentService
↓
如果 PaymentService 还没创建，则先创建它
↓
拿到 PaymentService Bean
↓
调用 OrderService 构造器
↓
OrderService 创建成功
```

因此整个应用会形成一张依赖图：

```text
Controller
↓
Service
↓
Repository / Mapper
↓
DataSource
```

构造器注入的一个重要价值是：

> **对象创建成功时，它的必需依赖应该已经完整。**

这也是为什么构造器参数很多时，不应该第一反应只是用 Lombok 省代码，而应该继续问：

> **这个类是不是承担了过多职责？**

---

## 四、多个 Bean 冲突：Spring 不是不知道“技术”，而是不知道“业务选择”

第一道题：

```java
public interface StorageClient {
}
```

```java
@Component
public class OssStorageClient implements StorageClient {
}
```

```java
@Component
public class MinioStorageClient implements StorageClient {
}
```

```java
@Service
public class FileService {

    private final StorageClient storageClient;

    public FileService(StorageClient storageClient) {
        this.storageClient = storageClient;
    }
}
```

Spring 创建 `FileService` 时需要一个 `StorageClient`，但容器里同时有：

```text
OssStorageClient
MinioStorageClient
```

于是：

```text
需要 StorageClient
↓
找到两个候选 Bean
↓
无法唯一确定
↓
FileService 依赖解析失败
↓
应用启动失败
```

### 今天纠正的一个点

最开始容易把它说成：

> “发生在 Bean 创建前。”

更准确的是：

> **问题发生在 FileService 创建过程中的依赖解析阶段，FileService 实例还没有成功创建。**

所以以后遇到：

```text
required a single bean, but 2 were found
```

应该先想到：

```text
依赖解析歧义
```

而不是初始化回调或 AOP Proxy。

---

## 五、为什么不能用 required=false 掩盖必需依赖

假设 Codex 为了让项目启动，建议：

```java
@Autowired(required = false)
private StorageClient storageClient;
```

今天的判断是：

**拒绝直接接受。**

因为根因是：

```text
存在两个 StorageClient
↓
业务到底应该用 OSS 还是 MinIO 没有被明确表达
```

把依赖改成 optional 并没有回答这个问题。

它可能只是把：

```text
启动期明确失败
```

推迟成：

```text
运行期潜在失败
```

例如后续：

```java
storageClient.upload(...);
```

如果依赖没有正确注入，就可能变成运行时异常。

工程判断：

> **不能为了“让 Spring Boot 起起来”，把一个本来必须存在的依赖偷偷变成可选依赖。**

---

## 六、@Primary：解决默认选择，不等于表达真实业务规则

如果修改为：

```java
@Component
@Primary
public class OssStorageClient implements StorageClient {
}
```

那么当 Spring 需要一个 `StorageClient` 时，会优先选择 OSS 实现。

可以先把 `@Primary` 记成：

> **多个同类型 Bean 都满足注入条件时，没有进一步指定的情况下，优先选我。**

所以：

```text
OssStorageClient   ← @Primary
MinioStorageClient
↓
FileService 默认注入 OSS
```

技术上可以完成依赖解析。

但今天第二层判断是：

> **能启动，不代表这个修改一定合理。**

假设真实需求是：

```text
storage.type=oss
→ 使用 OSS

storage.type=minio
→ 使用 MinIO
```

那么直接给 OSS 永久加 `@Primary` 并没有表达真实业务规则。

更合理的设计通常应该让：

```text
配置
↓
决定哪些 Bean 在当前环境注册 / 生效
```

这里要区分：

```text
@Primary
解决：
多个候选都存在时默认选谁

条件装配
解决：
当前环境本来应该有哪些候选
```

这两个不是一个层级的问题。

---

## 七、Bean 生命周期：@PostConstruct 为什么可能拖垮启动

示例：

```java
@Component
public class ModelLoader {

    @PostConstruct
    public void init() {
        remoteAiClient.loadModels();
    }
}
```

平时远程调用：

```text
200ms
```

某天变成：

```text
30s timeout
```

这时风险链路是：

```text
Bean 进入初始化阶段
↓
执行 @PostConstruct
↓
远程调用卡 30s
↓
Bean 初始化迟迟不能完成
↓
ApplicationContext 启动被拖慢
↓
服务迟迟不能 Ready
```

如果超时异常继续向上抛出，还可能：

```text
Bean 初始化失败
↓
ApplicationContext 启动失败
↓
Spring Boot 启动失败
```

### 今天形成的判断

> **不应该随便把耗时、不可控的远程调用塞进 @PostConstruct。**

因为这相当于把：

```text
外部服务可用性
```

绑定成：

```text
本 Java 服务能否启动
```

不过也不能绝对化。

如果这个远程依赖本来就是系统的硬前置条件，没有它整个服务就没有业务意义，那么启动失败可能反而是合理行为。

所以真正应该问：

> **这个依赖是不是服务启动的硬前置条件？**

---

## 八、BeanPostProcessor + AOP：为什么手动 new 会让 @Transactional 失效

代码：

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder() {
        // 保存订单
    }
}
```

正常注入：

```java
@Autowired
private OrderService orderService;
```

调用：

```java
orderService.createOrder();
```

在 Spring Proxy 模型下，链路可以理解为：

```text
Spring 创建 OrderService Bean
↓
BeanPostProcessor 参与处理
↓
发现需要事务增强
↓
生成 Proxy
↓
其他对象拿到的是被增强后的对象
↓
调用 createOrder()
↓
代理先执行事务逻辑
↓
再调用真实业务方法
```

但是如果有人写：

```java
OrderService orderService = new OrderService();
orderService.createOrder();
```

链路变成：

```text
JVM 直接 new
↓
普通 Java 对象
↓
没有交给 Spring IoC 容器创建
↓
没有经过对应的 Spring Bean 后处理和代理增强链路
↓
没有事务 Proxy
↓
直接调用真实方法
```

因此：

> **手动 new 出来的普通对象不能因为类上存在 Spring 注解，就自动获得 Spring 容器提供的能力。**

今天形成了一个很实用的排查条件反射：

如果遇到：

```text
@Transactional 不生效
@Async 不生效
@Cacheable 不生效
某些 AOP 行为不生效
```

先问：

> **这个对象到底是不是 Spring 创建和管理的 Bean？**

---

## 九、循环依赖：@Lazy 能打断创建环，但不能自动证明设计合理

代码：

```java
@Service
public class AService {

    private final BService bService;

    public AService(BService bService) {
        this.bService = bService;
    }
}
```

```java
@Service
public class BService {

    private final AService aService;

    public BService(AService aService) {
        this.aService = aService;
    }
}
```

创建链：

```text
创建 AService
↓
需要 BService
↓
创建 BService
↓
又需要 AService
↓
AService 还没有创建完成
↓
形成依赖环
↓
启动失败
```

今天第一反应是正确的：

> **这不仅是 Spring 技术问题，也可能暴露 Service 职责边界不清、双向耦合等设计问题。**

### 对 @Lazy 的纠正

如果 Codex 建议给其中一个依赖加：

```java
@Lazy
```

项目可能因此正常启动。

但不能简单认为：

```text
@Lazy 只是把报错往后推，后面一定会报错
```

这个说法过度了。

更准确的是：

> **@Lazy 可能确实打断 Bean 创建阶段的依赖环，运行期也可能正常；但它并不能自动证明 A 与 B 的双向依赖在架构上就是合理的。**

所以需要继续检查：

```text
A 为什么必须依赖 B？
B 为什么又必须依赖 A？
是否可以抽取第三方职责？
第一次真正调用 Lazy Bean 在哪里？
运行期有没有真正的业务递归？
```

这里必须区分两种“循环”：

### Bean 创建循环

```text
A 创建需要 B
B 创建需要 A
```

### 业务调用递归

```text
A.method()
→ B.method()
→ A.method()
→ B.method()
→ ...
```

`@Lazy` 可能帮助处理前者，但不会自动消除后者，也不会自动解决职责双向耦合。

---

## 十、AI Review：不能让 Codex 自己修改、自己解释、自己宣布验收通过

今天最后一题暴露出一个实际工作中的习惯：

```text
Codex 修改代码
↓
Codex 解释为什么这样改
↓
再让 Codex 自己检查有没有问题
↓
Codex 说没问题
↓
准备发布
```

这个流程不够。

原因不是 AI 没有价值，而是：

> **修改方案和验收判断如果完全来自同一个判断来源，缺少独立证据。**

更合理的方式是：

```text
AI 负责：
搜索代码
梳理依赖
说明修改原因
给出验证方式
扫描潜在影响

工程师负责：
确认根因
确认业务语义
执行真实运行验证
看日志和指标
做 Smoke Test
决定是否可以发布
```

核心变化是：

> **不要只让 AI“再确认一次”，而要让 AI 给出可验证的证据，再用运行结果独立验证。**

---

## 十一、Bean 装配问题的 Definition of Done

修完一个 Bean 装配问题，不能只用：

```text
Spring Boot 启动成功
```

作为完成标准。

至少应该继续确认：

```text
1. ApplicationContext 正常初始化

2. 所有必需 Bean 能唯一且正确解析

3. 没有通过 required=false 之类方式隐藏必需依赖

4. 没有留下未解释的循环依赖

5. 关键 @Transactional / @Async / AOP 行为仍然生效

6. 受影响的关键业务接口完成 Smoke Test

7. Readiness 正常

8. 启动耗时没有明显异常
```

例如修复 `StorageClient` 装配问题后，不能只验证：

```text
项目启动成功
```

还要继续验证：

```text
当前实际注入的是 OSS 还是 MinIO？
是否符合当前配置和业务要求？
上传/下载接口是否走到了正确实现？
是否因为 @Primary / @Lazy / optional 改变了原业务语义？
```

真正的闭环是：

```text
根因
↓
修改
↓
启动验证
↓
业务验证
↓
关键框架行为验证
↓
运行状态验证
↓
才算完成
```

---

## 十二、今天答题中的几个易错点

### 1. 两个实现冲突发生在 Bean 创建前？

不准确。

更准确：

```text
FileService 创建过程中
↓
依赖解析 StorageClient
↓
发现多个候选
↓
解析失败
```

### 2. @Primary = 这个实现永远正确？

错误。

`@Primary` 只表达：

```text
多个候选同时存在时默认优先选谁
```

它不能证明业务上真的应该永远使用这个实现。

### 3. @PostConstruct 里不能有任何远程调用？

不能绝对化。

真正要判断：

```text
这个外部依赖是不是启动硬依赖？
失败是否应该阻止整个服务 Ready？
```

### 4. 手动 new 只是“没有依赖注入”？

不够准确。

更重要的是：

```text
手动 new
→ 不属于 Spring 管理对象
→ 绕过 Bean 生命周期和相关代理增强
```

因此很多 Spring 注解能力可能失效。

### 5. @Lazy 一定只是把错误推迟到运行期？

不一定。

它可能真的打断 Bean 创建环并正常运行。

真正需要继续审核的是：

```text
双向依赖本身是否合理
+
运行期是否存在逻辑递归和职责耦合
```

### 6. Codex 自检通过 = 可以上线？

错误。

AI 自检属于证据之一，但不能替代：

```text
启动结果
日志
实际 Bean 选择
业务 Smoke Test
关键 AOP 行为
Readiness
运行指标
```

---

## 十三、今天形成的 Spring 底层模型

把 Day 79 和 Day 80 串起来：

```text
组件扫描 / 配置
↓
BeanDefinition
↓
依赖解析
↓
Bean 实例化
↓
依赖注入
↓
初始化回调
↓
BeanPostProcessor
↓
AOP Proxy
↓
@Transactional / @Async / @Cacheable 等增强能力
↓
业务调用
```

所以以后遇到 Spring 问题，不要只问：

> 这个注解怎么写？

应该先定位问题处于哪一层：

```text
BeanDefinition 注册？
依赖解析？
Bean 实例化？
初始化回调？
BeanPostProcessor？
Proxy？
业务调用？
```

这会比单纯记注解更容易理解 Spring 的行为。

---

## 十四、今天形成的 AI 工程审核模型

以后 Codex 遇到 Spring 启动问题并给出：

```text
加 @Primary
加 @Lazy
required=false
手动 new
调整 Bean 注册
```

不能只问：

> **改完能不能启动？**

而应该继续问：

```text
1. 原始根因是什么？

2. 修改发生在哪个 Spring 生命周期阶段？

3. 修改是否表达了真实业务语义？

4. 是否把启动期错误推迟到了运行期？

5. 是否绕过了 Spring 容器或 AOP Proxy？

6. 是否隐藏了架构双向依赖？

7. 修改后需要什么证据才能证明真正修复？
```

AI 非常适合：

```text
扫描 @Component / @Service
找构造器依赖
找 @Autowired
找 @Primary / @Qualifier
找 @Lazy
画 Bean 依赖链
找 BeanPostProcessor / AOP 相关代码
检查修改影响范围
```

工程师更应该负责：

```text
依赖是否真的必要
业务到底应该选哪个实现
Bean 生命周期是否合理
故障是否被推迟
AOP 是否仍生效
架构职责是否合理
上线验收证据是否充分
```

---

## 十五、今日总结

今天从 Day 79 的：

```text
@Transactional 为什么依赖 Proxy
```

继续往底层走到了：

```text
Proxy 前面的 Bean 到底怎么来的
```

最终形成完整链路：

```text
BeanDefinition
↓
依赖解析
↓
实例化
↓
依赖注入
↓
初始化
↓
BeanPostProcessor
↓
Proxy
↓
业务调用
```

今天最重要的几个工程判断：

1. **BeanDefinition 不等于 Bean 实例。**

2. **构造器注入本质上是在解析 Bean 依赖图。**

3. **多个 Bean 冲突时，Spring 缺的是业务选择规则，不应该用 optional 掩盖问题。**

4. **@Primary 解决默认优先级，不等于表达完整的环境/业务选择规则。**

5. **@PostConstruct 会影响 Bean 初始化和应用启动，不应该随便绑定不可控远程依赖。**

6. **手动 new 会绕开 Spring 容器管理，可能直接导致事务、异步、缓存等代理增强失效。**

7. **@Lazy 可能打断 Bean 创建环，但不能自动证明双向依赖设计合理。**

8. **AI 修改完成后，不能让 AI 自己宣布验收通过；必须建立可验证的 Definition of Done。**

今天真正训练的能力不是“会背 IoC、DI、@Primary、@Lazy”，而是：

> **看到 Spring Bean 装配、生命周期和代理问题时，能判断问题发生在哪一层，能审核 AI 的修复是否只是让项目启动，还是确实解决了根因并保持业务语义正确。**
