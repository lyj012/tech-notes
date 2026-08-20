# 后端工程化每日训练 Day 43：Kubernetes Pod 健康检查与流量摘除复盘

## 一、今天学习什么

Kubernetes 中的三类健康检查机制：

- Startup Probe
- Liveness Probe
- Readiness Probe

核心目标：理解 Kubernetes 如何判断一个服务是否启动完成、是否需要重启、是否应该继续接收流量。

一句话理解：

> Pod Running 不代表业务一定健康，Kubernetes 需要通过不同 Probe 判断服务状态。

---

# 二、Pod 重启到底是什么

一个 Pod 中通常运行容器，容器内部运行 Java 应用：

```
Pod
 |
 └── Container
        |
        └── Spring Boot 应用
```

Pod 重启通常意味着：

```
容器退出
↓
Kubernetes 发现异常
↓
重新启动容器
↓
Java 应用重新启动
```

重启原因可能包括：

- Java 进程异常退出
- OOMKilled
- Liveness Probe 检查失败
- 配置错误导致启动失败
- 节点异常

注意：

> 重启只是 Kubernetes 的恢复尝试，不代表问题已经解决。

---

# 三、三个 Probe 的区别

## 1. Startup Probe

解决：

> 应用启动慢的问题。

例如 Spring Boot：

```
加载 Bean
↓
初始化连接池
↓
读取配置
↓
缓存预热
↓
启动完成
```

如果应用需要 40 秒启动，但是 Liveness 10 秒开始检查，可能导致：

```
应用还没启动完成
↓
Liveness 判断失败
↓
Kubernetes 重启
↓
再次启动
↓
继续失败
```

Startup Probe 给应用提供启动窗口。

---

## 2. Liveness Probe

解决：

> 应用自身是否已经坏到需要重启。

例如：

```
Java进程存在
端口存在
但是线程死锁
接口全部超时
```

此时：

```
Liveness失败
↓
达到阈值
↓
重启容器
```

Liveness 主要判断：

```
重启这个实例是否可能恢复问题？
```

不要简单把所有外部依赖加入 Liveness。

---

## 3. Readiness Probe

解决：

> 当前实例是否适合接收流量。

例如：

```
Java正常运行
但是数据库暂时异常
```

此时：

```
Readiness=false
↓
Service停止发送流量
↓
Pod继续运行
```

数据库恢复：

```
Readiness=true
↓
重新接收流量
```

---

# 四、Liveness 和 Readiness 核心区别

## Java线程死锁

状态：

```
Liveness=false
Readiness=false
```

处理：

```
重启Pod
```

---

## MySQL短暂异常

状态：

```
Liveness=true
Readiness=false
```

处理：

```
暂时摘流
等待恢复
```

因为：

```
MySQL坏了
≠
Java服务坏了
```

---

# 五、错误配置 Liveness 的风险

错误设计：

```
/health/live
检查：
MySQL
Redis
MQ
第三方接口
AI服务
```

任何一个失败：

```
Liveness=false
```

可能导致：

```
外部依赖故障
↓
所有Pod重启
↓
启动压力增加
↓
故障扩大
```

原则：

> Liveness 判断的是当前应用自身是否死亡，不是整个系统是否完美。

---

# 六、Pod 同时重启如何排查

看到：

```
多个Pod同时重启
```

不能直接认为：

```
一定是OOM
```

需要先确认退出原因。

排查思路：

## 1. 查看 Pod 状态和事件

关注：

- Last State
- Reason
- Events

判断：

- OOMKilled
- Liveness probe failed
- 应用异常退出

## 2. 查看上一次容器日志

因为容器已经重启，需要查看之前退出前的日志。

## 3. 分析共同原因

多个 Pod 同时异常，优先考虑：

- 公共配置错误
- 公共依赖故障
- Probe 配置问题
- 发布问题

---

# 七、真实业务设计

假设支付服务：

```
启动时间：35秒

3个Pod

依赖：
MySQL
Redis
MQ
第三方支付
```

## Startup Probe

检查：

```
应用是否启动完成
配置是否加载完成
初始化是否完成
```

---

## Liveness Probe

检查：

```
Java应用自身是否还能工作
```

例如：

- 应用死锁
- 无法响应请求

不要简单检查所有外部依赖。

---

## Readiness Probe

检查：

```
当前是否具备处理支付请求的能力
```

根据业务决定依赖是否影响流量。

---

# 八、今天形成的工程认知

以前理解：

```
服务启动
=
服务正常
```

现在理解：

```
Running
≠
健康
```

线上真正需要关注：

```
Pod状态
Ready状态
接口RT
错误率
依赖状态
业务成功率
```

核心思想：

> 自动恢复机制必须正确设计，否则自愈可能变成故障放大。
