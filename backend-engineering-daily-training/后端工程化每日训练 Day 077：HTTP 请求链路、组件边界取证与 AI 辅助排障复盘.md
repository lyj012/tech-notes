# 后端工程化每日训练 Day 77：HTTP 请求链路、组件边界取证与 AI 辅助排障复盘

## 一、今天学习什么

今天学习主题：

**一次 HTTP 请求进入 Java 后端后的完整链路，以及如何利用组件边界快速缩小故障范围。**

核心链路：

```text
浏览器 / 前端
↓
Tengine / Nginx
↓
Spring Boot :8082
↓
Tomcat
↓
Filter / Spring Security
↓
DispatcherServlet
↓
Interceptor
↓
Controller
↓
Service
↓
Mapper / MyBatis
↓
数据库连接池 / JDBC
↓
OceanBase / MySQL
```

今天真正需要掌握的不是背住所有 Spring MVC 名词，而是建立一个可以迁移的模型：

> **一次请求不是简单的“前端 → 后端”，而是经过多个明确的组件边界。每个边界都可以单独验证。**

只要能够不断证明：

```text
上一层正常
↓
下一层异常
```

问题范围就会越来越小。

---

## 二、一次 HTTP 请求到底经过哪些层

假设前端访问：

```http
GET /api/monitor/host/123
```

实际链路可能是：

```text
Vue 前端
↓
Tengine :80 / :443
↓
Spring Boot :8082
↓
Tomcat
↓
Filter / Spring Security
↓
DispatcherServlet
↓
Interceptor
↓
HostController
↓
HostService
↓
HostMapper / MyBatis
↓
HikariCP
↓
OceanBase
```

返回过程则反过来：

```text
OceanBase
↓
Mapper
↓
Service
↓
Controller
↓
Spring MVC
↓
Tomcat
↓
Tengine
↓
前端
```

所以一句：

```text
“后端接口有问题”
```

实际可能涉及完全不同的层。

---

## 三、Tengine、Java 与 Spring MVC 各自负责什么

### 1. Tengine / Nginx

Tengine 主要负责：

```text
接收外部请求
↓
匹配 location
↓
通过 proxy_pass / upstream
↓
转发给 Java
```

例如：

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:8082/;
}
```

如果：

```text
Java 直连 200
正式域名 404 / 502
```

就不应该继续优先折腾数据库，而应该先看：

```text
Tengine
location
proxy_pass
upstream
路径改写
```

---

### 2. Tomcat

Spring Boot Web 项目通常由内嵌 Tomcat 提供 HTTP 服务。

例如：

```yaml
server:
  port: 8082
```

检查：

```bash
ss -lntp | grep ':8082'
```

只能回答：

> **有没有程序正在监听 8082。**

不能直接推出：

```text
所有 Controller 正常
Service 正常
数据库正常
Pika 正常
IoTDB 正常
```

---

### 3. Filter / Spring Security

Filter 位于 Controller 之前，常见用途：

```text
Token 校验
认证
TraceId
字符编码
跨域
安全过滤
```

因此请求可能：

```text
进入 Java
↓
Security Filter
↓
Token 无效
↓
401
```

Controller 根本没有执行。

所以：

> **Controller 没日志，不代表请求没有进入 Java。**

---

### 4. DispatcherServlet 与 Interceptor

DispatcherServlet 可以理解为 Spring MVC 的请求调度中心。

它负责根据：

```text
URL
+
HTTP Method
```

找到对应 Controller。

Interceptor 更靠近 Spring MVC，常用于：

```text
权限判断
用户上下文
操作日志
性能统计
```

工程上先记住：

```text
Filter
→ 更靠外层，属于 Servlet 体系

Interceptor
→ 更靠近 Controller
```

---

### 5. Controller → Service → Mapper

Controller 主要负责：

```text
接收参数
↓
基础校验 / 参数转换
↓
调用 Service
↓
返回结果
```

Service 负责业务逻辑。

Mapper / MyBatis 才进入真正的数据访问链：

```text
Java代码
↓
MyBatis
↓
HikariCP
↓
JDBC
↓
网络
↓
OceanBase / MySQL
```

---

## 四、服务“活着”和业务“正常”不是一回事

今天一个非常重要的判断：

```text
进程存在
≠
端口正常

端口正常
≠
HTTP Server 正常

HTTP Server 正常
≠
Health 正常

Health 正常
≠
业务接口正常

业务接口正常
≠
所有依赖正常
```

例如：

```bash
ss -lntp | grep ':8082'
```

看到：

```text
LISTEN
```

再执行：

```bash
curl http://127.0.0.1:8082/actuator/health
```

返回：

```json
{"status":"UP"}
```

仍然不能证明整个 Java 服务完全正常。

还可能存在：

```text
某个业务 Controller 抛异常
OceanBase 连接失败
Pika 异常
IoTDB 查询超时
连接池耗尽
某个 SQL 很慢
某个接口被权限拦截
```

所以：

> **Health UP 只说明当前健康检查认为应用可用，不能替代真实业务链路验证。**

---

## 五、利用状态码快速判断故障层级

状态码不能直接等于根因，但可以提供很强的方向信息。

### 1. 502 Bad Gateway

链路：

```text
浏览器
↓
Tengine
↓
Java
```

出现：

```text
502 Bad Gateway
```

第一反应应该放在：

```text
Tengine → Java
```

这个代理边界。

先验证：

```bash
ss -lntp | grep ':8082'
curl -v http://127.0.0.1:8082/api/host/list
```

如果 Java 直连正常，再查：

```text
location
proxy_pass
upstream
目标 IP / 端口
```

而不是一上来查数据库。

---

### 2. Java 直连 200，正式域名 404

例如：

```bash
curl http://127.0.0.1:8082/api/host/list
```

返回：

```text
200 OK
```

但：

```bash
curl https://monitor.xxx.com/api/host/list
```

返回：

```text
404
```

说明：

```text
Java 这个接口本身存在
+
直接访问可以正常处理
```

此时应该优先检查：

```text
Tengine location
proxy_pass
路径改写
```

核心问题是：

> **/api/host/list 经过 Tengine 后，最终到底变成什么路径发给 8082？**

---

### 3. 请求进入 Java，但 Controller 没执行，并返回 401

典型链路：

```text
Tomcat
↓
Filter / Spring Security
↓
DispatcherServlet
↓
Interceptor
↓
Controller
```

如果：

```text
请求已经进入 Java
Controller 日志没有出现
最终返回 401
```

优先检查：

```text
Token 是否携带
Token 是否过期
Token 是否合法
Spring Security
Filter
其次再看 Interceptor
```

这比直接查 Controller 或数据库更符合证据。

---

## 六、用日志把耗时范围“夹出来”

假设代码：

```java
log.info("query host list start");

hostMapper.selectList(...);

log.info("query host list end");
```

日志：

```text
12:00:00 query host list start

—— 20 秒 ——

12:00:20 query host list end
```

这说明：

> **两个日志之间这一段代码总共花了大约 20 秒。**

也就是当前耗时范围已经缩小到：

```text
hostMapper.selectList(...)
```

但不能直接下结论：

```text
SQL 本身执行了 20 秒
```

因为这一行背后还有：

```text
MyBatis
↓
等待连接池
↓
JDBC
↓
网络
↓
OceanBase
↓
SQL 执行
↓
结果返回
```

所以后续应该检查：

1. SQL 是否全表扫描、索引失效、数据量过大；
2. 数据库是否存在锁等待、负载高、慢查询；
3. HikariCP 是否在等待连接；
4. Java 到数据库的网络和 JDBC 是否异常。

核心方法：

```text
start 日志
↓
某段代码
↓
end 日志
```

如果中间耗时异常，就继续在这个区间内部增加证据，而不是重新回头查 Tengine。

---

## 七、只有“历史曲线查询”接口 timeout 怎么判断

现象：

```text
端口正常
health 正常
大部分接口正常
只有历史曲线查询 timeout
```

调用链：

```text
Controller
↓
Service
↓
IoTDB
```

这时候第一选择不应该是：

```text
重启 Java
```

原因：

1. 大部分接口正常，说明 Java 整体没有明显挂掉；
2. 重启可能让问题暂时消失，但没有得到根因；
3. 连接池积压、线程阻塞、慢查询等现场状态可能被重启清掉；
4. 会损失非常有价值的故障证据。

更合理的方式：

```text
先复现
↓
确认 Controller 是否进入
↓
确认 Service 是否进入
↓
IoTDB 调用前打日志
↓
调用 IoTDB
↓
IoTDB 调用后打日志
```

例如：

```java
log.info("query history start");

iotDBService.queryHistory(...);

log.info("query history end");
```

如果：

```text
19:00:00 query history start
19:00:30 timeout
没有 query history end
```

说明问题已经收敛到：

```text
Java 调用 IoTDB
↓
IoTDB / 客户端 / 网络 / 查询
```

再直接验证 IoTDB：

```text
IoTDB 进程
端口监听
Java → IoTDB 连通性
同样查询在 IoTDB 里执行是否也慢
```

如果：

```text
IoTDB 直接查询 100ms
Java 接口 30s
```

继续查 Java 客户端、连接池、线程、结果处理。

如果：

```text
IoTDB 自己查询也 30s
```

则主要问题已经进一步指向 IoTDB 查询侧。

---

## 八、组件边界 A/B 对照是最有价值的排障方法之一

以后遇到长链路，不一定要从头到尾机械排查。

可以主动绕过中间层。

例如：

```text
前端
↓
Tengine
↓
Java
```

做两个测试：

```text
A：直接访问 Java
B：通过 Tengine 访问
```

如果：

```text
A = 200
B = 502
```

范围马上缩小到：

```text
Tengine → Java
```

类似思想可以迁移：

### Java → IoTDB

```text
Java 查询慢
vs
直接 IoTDB 查询
```

### Java → Pika

```text
Java 报连接异常
vs
Java 所在服务器直接 redis-cli / nc
```

### Java → 数据库

```text
接口查询异常
vs
数据库手工执行等价 SQL
```

核心原则：

> **在组件边界直接取证，把一条长链切成两半。**

---

## 九、“监控页面打不开”应该怎么拆

一句：

```text
监控页面打不开
```

信息量非常低。

第一步不是猜 Tengine、Java、Pika 还是 IoTDB。

而是先把现象具体化：

```text
页面完全加载不了？
页面可以打开但没有数据？
某个接口 404？
401？
500？
502？
504？
timeout？
```

然后：

```text
1. 明确现象
↓
2. 找到对应请求
↓
3. 画出真正调用链
↓
4. 在组件边界测试
↓
5. 找到“上一层正常、下一层异常”
↓
6. 把范围缩到一个组件
↓
7. 再深入代码 / 配置 / 数据库
↓
8. 修复
↓
9. 按原链路完整验证
```

最终模型：

```text
现象
→ 假设
→ 证据
→ 缩小范围
→ 根因
→ 修复
→ 验证
```

---

## 十、AI / Codex 时代为什么还要学这些

今天训练最后形成了一个很重要的判断：

**现实开发中，很多具体排障完全可以交给 Codex。**

实际工作流更可能是：

```text
发现问题
↓
把代码 / 日志 / 环境交给 Codex
↓
AI 搜索项目并定位调用链
↓
AI 给出命令 / 修改方案
↓
人工判断风险
↓
执行
↓
验证结果
```

因此，没有必要为了证明自己“会手工排障”，机械记忆大量：

```text
第一条命令是什么
第二条命令是什么
第三条命令是什么
```

这种操作型知识的训练收益正在下降。

但是调用链、组件边界、故障机制仍然必须理解。

原因是 AI 可以帮忙：

```text
搜索代码
读配置
找调用关系
生成命令
分析日志
修改代码
```

但工程师仍然要判断：

```text
这个结论有没有证据？
这条命令会不会破坏环境？
为什么查这个组件？
为什么不是另一个组件？
这个修改能不能上生产？
修复以后验证闭环完整吗？
```

例如 Codex 如果提出：

```text
历史曲线 timeout
→ 删除 OceanBase 数据
→ 重启 Java
```

只要知道真实调用链：

```text
Controller
↓
Service
↓
IoTDB
```

就会立即发现：

> **OceanBase 并不是当前主要证据指向的链路，为什么要先做高风险操作？**

所以今天这类知识的价值不是：

> **和 AI 比谁排障更快。**

而是：

> **让自己能够给 AI 更准确的上下文、理解 AI 为什么这么排、审核方案是否合理，并控制线上修改风险。**

---

## 十一、今天训练后的学习方向调整

今天也发现一个问题：

> **如果后续训练继续大量围绕“手工执行排障命令”，边际收益会越来越低。**

后续更值得投入的方向应该逐渐转向：

```text
Java 后端原理
↓
线程池 / 连接池 / JVM
↓
事务 / 锁 / 并发 / 幂等
↓
缓存 / MQ / 数据库机制
↓
分布式系统
↓
限流 / 熔断 / 降级 / 容错
↓
系统设计与架构 Trade-off
↓
AI 辅助下的工程判断
```

训练重点应该从：

```text
“这个报错下一条命令是什么？”
```

逐渐升级成：

```text
“为什么系统会这样？”
“故障为什么会传播？”
“这个设计解决什么问题，又增加什么代价？”
“AI 给出的方案为什么能上 / 不能上？”
```

操作型知识可以在真实工作中按需由 AI 加速。

原理、架构和工程判断才更值得长期积累。

---

## 十二、今天容易混淆的几个点

### 1. 8082 LISTEN = 服务完全正常？

错误。

```text
8082 LISTEN
→ 只能说明 HTTP 入口有进程监听
```

---

### 2. /actuator/health = UP = 所有业务正常？

错误。

Health 正常并不能证明：

```text
OceanBase
Pika
IoTDB
每一个 Controller
每一条业务链路
```

都正常。

---

### 3. Controller 没日志 = 请求没到 Java？

错误。

请求可能已经在：

```text
Filter / Spring Security
```

被拦截。

---

### 4. Mapper 调用 20 秒 = SQL 一定执行了 20 秒？

错误。

还可能包含：

```text
等待连接池
JDBC
网络
数据库锁等待
结果返回
```

---

### 5. 某个接口 timeout = 先重启 Java？

不一定。

尤其当：

```text
health 正常
大部分接口正常
只有一个依赖 IoTDB 的接口异常
```

应该先保护现场证据，沿真实调用链取证。

---

### 6. 使用 Codex = 不需要理解后端链路？

错误。

更合理的是：

```text
AI负责加速搜索、分析和修改
+
工程师负责模型、判断、风险与验收
```

---

## 十三、今日总结

今天建立的核心请求模型：

```text
Browser
↓
Tengine
↓
Tomcat
↓
Filter / Security
↓
DispatcherServlet
↓
Interceptor
↓
Controller
↓
Service
↓
Mapper
↓
连接池 / JDBC
↓
数据库 / 中间件
```

真正需要留下的排障模型：

```text
现象
↓
画调用链
↓
找到组件边界
↓
A/B 对照取证
↓
上一层正常 / 下一层异常
↓
缩小范围
↓
找到根因
↓
修复
↓
沿原链路完整验证
```

今天最重要的认知不是学会更多“手工命令”，而是：

> **AI 可以比人工更快地搜索代码、分析日志和执行大量机械排障，但工程师仍然需要理解调用链、组件边界和故障机制，才能判断 AI 的方案是否合理、是否安全、是否真正解决根因。**

因此后续训练应减少低收益的机械排障记忆，逐渐提高：

```text
后端原理
中间件机制
分布式系统
架构设计
Trade-off
AI 辅助下的工程判断
```

的训练比例。

最终目标不是：

```text
不用 AI 也能手工排所有故障
```

而是：

```text
理解系统
+
正确使用 AI
+
审核 AI
+
控制风险
+
完成可靠交付
```
