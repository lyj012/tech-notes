# 后端工程化每日训练（第 79 天）

## 今日主题

Spring 事务边界设计与 AI 辅助事务方案审核

---

## 一、今日训练目标

今天从故障排查型训练切回 Java 后端核心机制，重点不是背 Spring 注解，而是理解：

- 事务为什么可能不生效；
- 事务边界如何设计；
- AI 生成事务修改方案时如何审核。

核心问题：

> 代码写了事务，不代表业务真的按照设计执行。

---

## 二、@Transactional 的真实含义

错误理解：

```java
@Transactional = 自动开启数据库事务
```

实际流程：

```
@Transactional
        ↓
Spring Proxy
        ↓
TransactionInterceptor
        ↓
事务管理器
        ↓
commit / rollback
```

注解只是事务元数据，真正执行事务的是 Spring 事务基础设施。

---

## 三、事务代理边界：为什么 REQUIRES_NEW 可能失效

示例：

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(){
        orderMapper.insertOrder();
        saveLog();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog(){
        logMapper.insertLog();
    }
}
```

容易误认为：

```
createOrder
    ↓
事务 T1

saveLog
    ↓
事务 T2
```

实际：

```java
this.saveLog();
```

属于同一个对象内部调用，没有经过 Spring Proxy。

因此：

- saveLog 的 REQUIRES_NEW 不会生效；
- 两个 SQL 可能处于同一个事务。

工程判断：

> 判断事务问题，第一步不是看注解，而是看调用链是否经过代理。

---

## 四、事务不要包含不可控操作

错误设计：

```java
@Transactional
public void pay(){
    createOrder();
    reduceStock();
    paymentApi.pay();
}
```

问题：

第三方 HTTP 调用具有不可控性：

- 网络异常；
- 响应慢；
- 第三方服务不可用。

如果数据库事务等待第三方接口：

- 长时间占用连接；
- 持有锁；
- 降低并发能力。

正确思路：

```
事务1：创建订单 WAIT_PAY

事务外：调用支付

事务2：支付回调更新状态
```

复杂场景使用：

- MQ；
- 本地消息表；
- 状态机；
- 补偿机制。

---

## 五、REQUIRES_NEW 与连接池风险

假设：

```yaml
hikari:
  maximum-pool-size: 100
```

外层事务：

```
事务 T1
占用连接 C1
```

进入 REQUIRES_NEW：

```
挂起 T1
创建 T2
申请连接 C2
```

一个请求可能需要两个数据库连接。

100 个请求同时进入：

```
100 × 2 = 200 个连接需求
```

但是连接池只有 100。

可能出现：

- 线程等待；
- Connection timeout；
- 应用卡顿。

生产排查需要关注：

- Hikari active connections；
- idle connections；
- pending threads；
- 数据库连接状态；
- 线程堆栈。

---

## 六、AI 修改事务代码时的审核

AI 可能给出：

> 日志很重要，使用 REQUIRES_NEW 保证日志保存。

不能直接接受。

需要判断：

### 1. 日志是否真的需要强一致？

不同日志可靠性要求不同：

- 审计日志；
- 业务日志；
- 调试日志。

### 2. 是否引入资源风险？

检查：

- 并发量；
- 连接池；
- 延迟；
- 数据库压力。

### 3. 是否有更合适方案？

例如：

```
业务事务
    ↓
本地消息表
    ↓
异步处理
```

或者：

```
业务服务
    ↓
MQ
    ↓
日志服务
```

---

## 七、今日核心总结

今天重点不是记忆：

- REQUIRED；
- REQUIRES_NEW；
- Transactional 参数。

而是形成工程判断：

1. @Transactional 不等于事务一定生效。

2. 事务生效依赖调用路径和 Spring Proxy。

3. 数据库事务应该保护短时间、强一致的数据变化。

4. HTTP、MQ、AI 调用等不可控操作不应该随意放入事务。

5. AI 可以生成代码，但不能替代事务边界和架构判断。

6. 可靠性问题通常需要：

- MQ；
- 状态机；
- 补偿机制；
- 本地消息表。

---

## 今日训练能力

从：

> 会写 @Transactional

提升到：

> 能判断一个事务设计是否合理，并审核 AI 给出的修改方案。
