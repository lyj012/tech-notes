# 后端工程化每日训练 Day 21：Prometheus、Grafana 服务监控、业务指标与告警排障

## 一、今天学习的知识点

今天学习的是：

**Prometheus、Grafana 服务监控、业务指标与告警排障**

一句话理解：

> 监控系统用于持续采集服务运行指标、观察变化趋势，并在异常影响用户之前主动告警，而不是等用户反馈系统已经不可用后才开始排查。

在真实生产环境中，系统故障不一定表现为：

```text
服务直接宕机
接口全部报错
进程无法启动
```

更多时候，故障会经历一个逐步恶化的过程：

```text
第三方接口逐渐变慢
↓
单个任务执行时间增加
↓
Worker 消费吞吐下降
↓
Kafka Lag 持续增加
↓
用户排队时间变长
↓
线程池或连接池逐渐饱和
↓
大量请求超时
↓
服务最终不可用
```

如果只有日志，团队往往要等到错误真正发生后才能看到异常。

如果建立了监控体系，就可以更早发现：

```text
AI 接口 P95 耗时上升
Kafka Lag 开始增长
任务端到端耗时增加
Worker 活跃线程接近上限
支付成功率持续下降
```

今天真正需要掌握的重点不是只会安装 Prometheus 和 Grafana，而是理解：

```text
应该监控什么
指标为什么异常
多个指标之间如何关联
如何设计有意义的告警
如何根据监控曲线缩小故障范围
如何结合日志继续定位根因
```

---

## 二、日志、监控指标和链路追踪的区别

### 1. 日志回答“发生了什么”

日志通常记录离散事件：

```text
任务创建成功
消息发送成功
Worker 开始执行
AI 接口调用超时
数据库状态更新失败
```

日志适合回答：

```text
哪个任务失败
执行到哪一步
具体异常是什么
错误栈是什么
第三方返回了什么错误码
```

### 2. 监控回答“系统是否健康”

监控指标通常是持续变化的数值：

```text
每秒请求数
接口平均耗时
接口 P95 耗时
错误率
Kafka Lag
线程池活跃数
数据库连接池使用率
任务成功率
```

监控适合回答：

```text
系统什么时候开始变慢
异常是突然发生还是逐渐恶化
受影响范围有多大
当前故障是否仍在持续
发布前后指标是否发生变化
容量是否即将达到上限
```

### 3. 链路追踪回答“请求经过了哪里”

一次请求可能经过：

```text
Nginx
↓
API
↓
Kafka
↓
Worker
↓
第三方 AI
↓
MySQL
```

链路追踪用于观察：

```text
请求经过了哪些服务
每一段耗时多少
哪一段最慢
哪一个下游发生错误
```

三者关系可以理解为：

```text
Metrics：发现异常
↓
Tracing：缩小到具体链路
↓
Logs：定位具体原因
```

不能用其中一种完全替代另外两种。

---

## 三、Prometheus、Grafana 和 Alertmanager 的职责

整体架构：

```text
Spring Boot Application
↓
暴露 Metrics
↓
/actuator/prometheus
↓
Prometheus 定时拉取
↓
存储时间序列数据
↓
Grafana 查询并展示
↓
告警规则触发
↓
Alertmanager 发送通知
```

### 1. Spring Boot Application

应用负责生成并暴露指标，例如：

```text
JVM 内存
GC 次数与耗时
线程数量
HTTP 请求次数
HTTP 请求耗时
数据库连接池
自定义业务指标
```

### 2. Micrometer

Micrometer 可以理解为 Java 应用中的指标门面。

业务代码可以通过统一 API 创建：

```text
Counter
Timer
Gauge
DistributionSummary
```

再由 Prometheus Registry 转换为 Prometheus 能识别的格式。

### 3. Prometheus

Prometheus 主要负责：

```text
定时抓取指标
存储时间序列数据
使用 PromQL 查询
计算聚合结果
执行告警规则
```

Prometheus 常见采用 Pull 模式：

```text
Prometheus
↓
主动请求目标服务的 Metrics 地址
```

例如每 15 秒抓取一次：

```text
http://task-api:8080/actuator/prometheus
http://task-worker:8081/actuator/prometheus
```

### 4. Grafana

Grafana 主要负责：

```text
仪表盘
趋势图
多指标关联展示
时间范围对比
变量筛选
发布前后对比
值班排障视图
```

Grafana 本身通常不是指标存储系统，而是从 Prometheus 等数据源查询并展示数据。

### 5. Alertmanager

Alertmanager 主要负责：

```text
告警分组
告警去重
告警抑制
静默
路由
发送通知
```

通知渠道可能包括：

```text
企业微信
钉钉
飞书
邮件
短信
电话
PagerDuty
```

Prometheus 负责判断告警条件是否满足，Alertmanager 负责如何处理和发送告警。

---

## 四、时间序列指标是什么

Prometheus 存储的不是普通业务表，而是时间序列数据。

一条时间序列通常由以下内容组成：

```text
指标名称
+
标签集合
+
时间戳
+
数值
```

例如：

```text
ai_request_duration_seconds{
  provider="openai",
  model="gpt-5",
  result="success"
}
```

不同标签组合代表不同时间序列：

```text
provider="openai", model="gpt-5"
provider="openai", model="gpt-4.1"
provider="claude", model="sonnet"
```

标签的价值是支持按维度分析：

```text
按厂商
按模型
按接口
按环境
按服务
按状态码
按任务类型
```

但标签不能无限增加。

如果把以下高基数字段直接作为标签：

```text
userId
orderId
taskId
requestId
完整 URL
异常消息
```

可能产生海量时间序列，导致：

```text
Prometheus 内存上涨
查询变慢
磁盘占用增加
抓取和聚合成本上升
```

因此：

> 指标标签适合低基数、可聚合的维度；具体 TaskId、OrderId、TraceId 更适合放在日志和链路追踪中。

---

## 五、Spring Boot 暴露 Prometheus 指标

典型依赖包括：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

基础配置：

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      show-details: when_authorized
```

启动服务后，可以访问：

```text
/actuator/prometheus
```

可能看到：

```text
jvm_memory_used_bytes
jvm_gc_pause_seconds_count
jvm_threads_live_threads
http_server_requests_seconds_count
hikaricp_connections_active
process_cpu_usage
system_cpu_usage
```

生产环境不能简单地将所有 Actuator 端点直接暴露给公网。

需要考虑：

```text
只开放必要端点
限制访问来源
通过内网访问
配置认证
避免泄露环境和配置信息
```

---

## 六、常见的指标类型

## 1. Counter：计数器

Counter 只会累计增加，适合记录：

```text
请求总数
任务成功总数
任务失败总数
重试总数
支付成功总数
第三方接口错误总数
```

例如：

```java
Counter aiRequestCounter = Counter.builder("ai_request_total")
        .tag("provider", provider)
        .tag("result", result)
        .register(meterRegistry);

aiRequestCounter.increment();
```

Counter 本身的累计值通常意义有限，更常见的是观察单位时间增长速率。

例如：

```promql
rate(ai_request_total[5m])
```

### 2. Gauge：仪表值

Gauge 可以上升也可以下降，适合记录当前状态：

```text
WAITING 任务数量
RUNNING 任务数量
Kafka Lag
线程池队列长度
当前在线用户数
连接池活跃连接数
```

例如：

```java
Gauge.builder("ai_task_waiting", taskService, service -> service.countWaiting())
        .register(meterRegistry);
```

### 3. Timer：耗时和次数

Timer 适合记录：

```text
AI 接口调用耗时
数据库查询耗时
任务处理耗时
支付接口耗时
文件转换耗时
```

例如：

```java
Timer.Sample sample = Timer.start(meterRegistry);

try {
    callAiService();
} finally {
    sample.stop(Timer.builder("ai_request_duration")
            .tag("provider", provider)
            .tag("model", model)
            .register(meterRegistry));
}
```

### 4. DistributionSummary：数值分布

适合记录：

```text
请求体大小
文件大小
Token 数量
批处理条数
消息体大小
```

---

## 七、四个黄金信号

Google SRE 中常见的四个黄金信号是：

```text
Latency：延迟
Traffic：流量
Errors：错误
Saturation：饱和度
```

### 1. Latency：延迟

观察：

```text
接口 P50
接口 P95
接口 P99
AI 调用耗时
数据库查询耗时
任务排队时间
端到端任务耗时
```

### 2. Traffic：流量

观察：

```text
每秒请求数
每分钟创建任务数
Kafka 每秒生产消息数
Kafka 每秒消费消息数
支付请求数
AI 调用次数
```

### 3. Errors：错误

观察：

```text
HTTP 5xx 比例
AI 调用失败率
任务失败率
支付失败率
数据库异常数
超时率
429 限流率
```

### 4. Saturation：饱和度

饱和度表示系统距离容量上限还有多远。

常见指标：

```text
CPU 使用率
内存使用率
线程池活跃线程比例
线程池队列长度
数据库连接池使用率
HTTP 连接池使用率
Kafka Lag
磁盘使用率
磁盘 I/O 等待
```

只监控 CPU 和内存是不够的。

系统可能出现：

```text
CPU 20%
内存 40%
HTTP 连接池 100%
Worker 线程全部等待第三方接口
Kafka Lag 持续增加
```

此时机器资源看起来正常，但业务已经严重异常。

---

## 八、为什么平均值会掩盖问题

假设 100 个 AI 请求中：

```text
90 个请求耗时 3 秒
10 个请求耗时 60 秒
```

平均耗时约为：

```text
8.7 秒
```

平均值看起来没有达到 60 秒，但实际上已经有 10% 的用户遭遇严重超时。

因此真实监控通常需要观察：

```text
P50
P90
P95
P99
最大值
```

例如：

```text
P50 = 3 秒
P95 = 60 秒
P99 = 60 秒
```

P95 表示：

```text
95% 的请求耗时不超过该值
```

如果 P50 正常、P99 很高，可能说明：

```text
只有少量请求异常
部分模型变慢
某个实例异常
某个地区网络异常
某些大输入任务耗时过长
```

如果 P50、P95、P99 同时升高，则说明影响范围更广。

---

## 九、系统指标和业务指标必须同时存在

### 1. 系统指标

基础系统指标包括：

```text
CPU
内存
磁盘
网络
JVM Heap
GC
线程
线程池
数据库连接池
HTTP 连接池
```

它们用于判断基础资源和运行时是否异常。

### 2. 业务指标

业务指标直接描述业务是否正常：

```text
支付成功率
任务最终成功率
任务积压数量
AI 调用成功率
订单创建成功率
PDF 转换成功率
用户端到端等待时间
```

例如：

```text
CPU：20%
内存：40%
JVM：正常
AI 调用成功率：5%
```

如果只看系统指标，会错误地认为服务完全健康。

因此应该建立两套视图：

```text
基础设施与应用运行视图
+
业务健康视图
```

真正的稳定性判断必须同时结合两类指标。

---

## 十、AI 异步任务平台应该监控什么

AI 异步任务平台的典型链路：

```text
用户提交任务
↓
API 写入任务表
↓
发送 Kafka 消息
↓
Worker 消费
↓
调用第三方 AI
↓
解析结果
↓
保存结果
↓
更新任务状态
↓
通知用户
```

只监控 AI 接口成功率是不够的，因为 AI 接口成功后，后续仍可能失败：

```text
结果解析失败
数据库保存失败
对象存储上传失败
任务状态更新失败
回调失败
消息确认失败
```

最重要的业务指标至少包括以下五组。

### 1. AI 调用成功率和错误率

建议按以下维度拆分：

```text
provider
model
result
errorType
statusCode
```

错误类型可以区分：

```text
认证失败
429 限流
连接超时
读取超时
第三方 5xx
响应解析失败
内容安全拒绝
业务参数错误
```

### 2. AI 调用耗时

重点观察：

```text
P50
P95
P99
超时率
```

并按：

```text
厂商
模型
任务类型
```

进行拆分。

### 3. Kafka Lag 和最老消息等待时间

Kafka Lag 表示尚未消费的消息数量。

但只看 Lag 还不够。

例如：

```text
Lag = 10000
```

在每秒消费数万条消息的系统中可能很快恢复。

在每秒只能消费几十条消息的系统中可能已经非常严重。

还应该观察：

```text
生产速率
消费速率
Lag 增长速度
最老消息等待时间
预计清空积压时间
```

### 4. 任务状态分布和最终成功率

监控：

```text
WAITING 数量
RUNNING 数量
SUCCESS 数量
FAILED 数量
TIMEOUT 数量
长时间停留在 RUNNING 的任务数
```

任务最终成功率比单个组件成功率更接近用户真实结果。

### 5. 任务端到端耗时

端到端耗时包括：

```text
排队等待时间
+
Worker 执行时间
+
AI 调用时间
+
结果处理时间
+
状态持久化时间
```

例如：

```text
AI 调用耗时只有 5 秒
任务排队等待 20 分钟
```

如果只看 AI 调用耗时，会错误地认为系统性能正常。

---

## 十一、PDF 工具平台应该监控什么

PDF 平台常见业务链路：

```text
用户上传文件
↓
创建转换任务
↓
进入任务队列
↓
Worker 下载文件
↓
执行压缩、拆分、水印或转换
↓
上传结果文件
↓
更新任务状态
↓
用户下载
```

关键业务指标包括：

```text
上传成功率
任务创建成功率
WAITING 任务数量
PDF 转换成功率
不同工具的处理耗时
文件平均大小和 P95 大小
对象存储上传失败率
结果文件下载成功率
任务端到端耗时
```

如果出现：

```text
WAITING：100 → 1000 → 10000
```

可能原因包括：

```text
任务流量突然增加
Worker 数量不足
单任务处理时间增加
对象存储变慢
PDF 转换进程阻塞
大文件比例增加
下游服务异常
```

不能看到积压后只做 Worker 扩容，还需要确认单位任务耗时为什么变化。

---

## 十二、支付系统应该监控什么

支付系统不能只看 HTTP 200。

第三方支付接口返回成功，不代表本地订单最终一定正确。

需要监控：

```text
支付请求量
支付成功率
支付失败率
支付处理中订单数
支付回调成功率
支付回调重复次数
支付状态更新失败数
金额不一致数量
第三方成功但本地未成功数量
超时未关闭订单数量
退款成功率
```

例如：

```text
第三方支付成功率：99.9%
本地订单最终成功率：80%
```

说明问题可能发生在：

```text
支付回调接收
签名验证
订单查询
数据库事务
幂等处理
状态机流转
异常补偿
```

业务指标可以比机器指标更早暴露真实事故。

---

## 十三、为什么 CPU 和内存正常，系统仍可能已经故障

今天思考题给出的现象：

```text
AI 调用平均耗时：3 秒 → 25 秒
CPU：正常
内存：正常
JVM：正常
Kafka Lag：持续增加
```

这很可能属于：

```text
I/O 等待型故障
```

Worker 调用第三方 AI 时，大部分时间可能在等待：

```text
DNS 解析
建立 TCP 连接
TLS 握手
代理转发
第三方排队
首字节返回
响应数据传输
```

等待期间不需要大量 CPU 计算，因此可能出现：

```text
CPU 使用率不高
内存使用率正常
Worker 线程长时间阻塞
单任务处理耗时上升
单位时间完成任务数下降
Kafka Lag 持续增长
```

类似现象还可能由以下原因造成：

```text
数据库连接池耗尽
HTTP 连接池耗尽
Redis 请求阻塞
数据库锁等待
对象存储响应变慢
线程池队列堆积
外部网络异常
```

因此必须记住：

> CPU 和内存正常，只能说明当前故障不一定是计算资源或内存资源不足，不能证明业务系统健康。

---

## 十四、AI 耗时上升并伴随 Kafka Lag 时的排查顺序

题目已经提供了一个最强线索：

```text
AI 调用耗时从 3 秒上升到 25 秒
```

因此排查不能凭感觉把所有组件平均排列，而应该根据指标证据确定优先级。

合理顺序是：

```text
1. 第三方 AI 服务响应变慢
2. 本机到第三方 AI 的网络链路异常
3. Worker 并发能力、线程池或消费者实例不足
4. 数据库、Redis、对象存储等其他下游异常
```

### 1. 第一优先：第三方 AI 服务

先按厂商和模型拆分：

```text
OpenAI P95 耗时
Claude P95 耗时
DeepSeek P95 耗时
```

例如：

```text
OpenAI：25 秒
Claude：3 秒
DeepSeek：4 秒
```

大概率说明问题集中在 OpenAI 链路。

还要查看：

```text
429 比例
5xx 比例
超时率
重试次数
不同模型耗时
不同区域耗时
```

### 2. 第二优先：网络链路

第三方调用变慢不一定完全是服务商自身问题，也可能是：

```text
DNS 解析变慢
连接建立变慢
TLS 握手变慢
出口代理异常
跨境网络抖动
丢包和重传增加
```

最好将 HTTP 调用耗时拆分为：

```text
DNS 耗时
连接耗时
TLS 耗时
首字节耗时
完整响应耗时
```

### 3. 第三优先：Worker 能力

如果单个任务从 3 秒增加到 25 秒，同样数量的 Worker 能完成的任务会显著下降。

链路可能是：

```text
AI 服务变慢
↓
Worker 线程被占用更久
↓
消费吞吐下降
↓
Kafka Lag 增长
```

增加 Worker 只能暂时缓解积压，还可能造成：

```text
第三方接口并发进一步增加
触发 429 限流
网络连接池压力增加
数据库写入压力增加
成本快速上涨
```

因此不能先盲目扩容。

### 4. 第四优先：数据库和其他下游

数据库问题并非不可能，但当前证据相对较弱。

只有当以下指标也异常时，才应提高排查优先级：

```text
SQL P95 耗时上升
慢 SQL 数量增加
数据库连接池等待增加
活跃连接接近上限
锁等待时间增加
任务状态更新失败率上升
```

---

## 十五、如果只有日志，没有监控曲线，会增加哪些排查难度

### 1. 无法快速确认异常开始时间

难以判断：

```text
14:00 突然变慢
还是
10:00 开始逐渐恶化
```

### 2. 无法判断恶化速度

Kafka Lag 可能是：

```text
100 → 110 → 120
```

也可能是：

```text
100 → 1000 → 10000
```

两者紧急程度完全不同。

### 3. 难以建立多个指标之间的时间关系

监控曲线可能显示：

```text
14:05 AI P95 耗时升高
14:07 Worker 消费速率下降
14:10 Kafka Lag 开始增长
14:15 任务端到端耗时上升
```

由此可以形成较强的因果推断：

```text
AI 服务变慢
↓
Worker 吞吐下降
↓
Kafka 消息积压
↓
用户等待时间增加
```

日志是大量离散事件，很难在短时间内人工形成完整趋势。

### 4. 无法快速判断是否与发布有关

如果 Grafana 中标记了发布事件：

```text
13:58 发布新版本
14:00 AI 调用耗时开始上升
```

就可以优先怀疑：

```text
新版本 HTTP 客户端配置
连接池参数
重试策略
超时策略
并发控制
代理配置
```

### 5. 缺少正常基线

单条日志显示：

```text
AI 调用耗时 25 秒
```

但无法知道过去正常水平。

监控可以对比：

```text
过去 7 天 P95：4 秒
今天 P95：28 秒
```

### 6. 无法量化受影响范围

日志可以证明某一次请求失败，但不容易直接回答：

```text
失败率是 0.1% 还是 30%
影响一个模型还是全部模型
影响一个实例还是整个集群
影响持续了 2 分钟还是 2 小时
```

---

## 十六、PromQL 的基础思路

PromQL 不应该只靠死记语法，而应先明确要回答的问题。

### 1. AI 请求速率

```promql
sum(rate(ai_request_total[5m]))
```

按厂商拆分：

```promql
sum by (provider) (rate(ai_request_total[5m]))
```

### 2. AI 请求失败率

```promql
sum(rate(ai_request_total{result="failure"}[5m]))
/
sum(rate(ai_request_total[5m]))
```

### 3. HTTP 5xx 比例

```promql
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m]))
```

### 4. 数据库连接池使用率

```promql
hikaricp_connections_active
/
hikaricp_connections_max
```

### 5. AI P95 耗时

如果指标使用 Histogram：

```promql
histogram_quantile(
  0.95,
  sum by (le, provider) (
    rate(ai_request_duration_seconds_bucket[5m])
  )
)
```

PromQL 结果是否有价值，取决于指标命名、标签设计和采集方式是否正确。

---

## 十七、告警规则应该如何设计

错误的告警方式：

```text
CPU > 80%
立即报警
```

如果 CPU 在阈值附近波动：

```text
79%
81%
79%
82%
```

就可能产生告警风暴。

更合理的方式：

```yaml
groups:
  - name: backend-alerts
    rules:
      - alert: HighCpuUsage
        expr: process_cpu_usage > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "服务 CPU 持续超过 80%"
```

`for: 5m` 表示条件持续 5 分钟后才真正触发告警。

告警设计需要考虑：

```text
阈值
持续时间
告警级别
影响范围
通知对象
是否自动恢复
是否需要抑制下游告警
是否存在值班处理手册
```

---

## 十八、告警风暴为什么危险

如果一个上游服务异常，可能同时触发：

```text
AI 调用超时告警
Worker 线程池告警
Kafka Lag 告警
任务失败率告警
任务端到端耗时告警
HTTP 5xx 告警
```

如果每个实例、每个模型、每个 Topic 都单独发送通知，值班人员可能收到数百条消息。

结果是：

```text
真正根因被淹没
告警渠道被刷屏
值班人员无法判断优先级
最终关闭通知
```

Alertmanager 可以进行：

```text
分组
去重
路由
抑制
静默
```

例如：

```text
第三方 AI 服务整体故障
```

可以抑制大量由它引起的下游任务失败告警，优先保留根因告警和核心业务影响告警。

---

## 十九、一个有效告警必须具备什么

一个有价值的告警应该尽量说明：

```text
哪里异常
异常指标是多少
持续了多久
影响什么业务
可能原因是什么
应该先查看什么
仪表盘地址
日志查询入口
处理手册地址
```

差的告警：

```text
系统异常
```

好的告警：

```text
生产环境 task-worker 的 AI 调用 P95 耗时持续 10 分钟超过 20 秒，
OpenAI 模型受影响，Kafka Lag 同期从 500 增长到 8000，
请优先查看 AI Provider Dashboard、出口网络和 Worker HTTP 连接池。
```

告警的目标不是告诉值班人员“某个数字超过阈值”，而是帮助其快速进入正确的排查路径。

---

## 二十、常见监控设计错误

### 错误一：只监控机器，不监控业务

只看：

```text
CPU
内存
磁盘
```

却没有：

```text
任务成功率
支付成功率
AI 调用成功率
Kafka Lag
端到端耗时
```

最终服务器看起来正常，业务已经不可用。

### 错误二：指标越多越好

几千个指标没有明确负责人和使用场景，只会增加：

```text
存储成本
查询成本
维护成本
认知负担
```

### 错误三：只看平均值

平均值会掩盖尾部慢请求。

### 错误四：标签基数失控

将 userId、taskId、orderId 作为标签，导致时间序列爆炸。

### 错误五：告警没有持续时间

瞬时波动导致大量误报。

### 错误六：所有告警都一个级别

无法区分：

```text
需要立即处理
工作时间处理
仅记录趋势
```

### 错误七：没有发布标记

指标异常后无法快速判断是否与版本发布有关。

### 错误八：没有处理手册

值班人员收到告警，但不知道第一步该查什么。

### 错误九：只告警，不验证告警

规则长期没有演练，真正故障时才发现：

```text
通知渠道失效
表达式错误
阈值不合理
告警没有负责人
```

### 错误十：监控系统自身没有监控

Prometheus 抓取失败、磁盘满或 Alertmanager 通知失败后，团队可能误以为系统一直正常。

---

## 二十一、今天思考题的完整回答

题目现象：

```text
AI 调用平均耗时：3 秒 → 25 秒
CPU：正常
内存：正常
JVM：正常
Kafka Lag：持续增加
```

### 问题一：为什么 CPU、内存正常，系统仍可能出现故障

因为当前故障很可能属于 I/O 等待型故障。

Worker 线程可能长时间等待第三方 AI、网络、连接池或数据库响应，没有大量消耗 CPU 和内存，但任务处理吞吐已经显著下降。

因此：

```text
资源指标正常
!=
业务系统正常
```

### 问题二：优先怀疑什么

合理优先级：

```text
第一：第三方 AI 服务响应变慢
第二：到第三方 AI 的网络链路异常
第三：Worker 并发、线程池或消费者实例不足
第四：数据库、Redis、对象存储等其他下游异常
```

之所以 AI 服务优先，是因为题目已经明确显示 AI 调用耗时从 3 秒升到 25 秒，这是当前最强证据。

数据库不是第一优先级，除非 SQL 耗时、连接池、锁等待或状态更新失败率也同时异常。

### 问题三：只有日志，没有监控曲线的困难

```text
无法快速确认异常开始时间
无法判断是突然故障还是逐渐恶化
无法判断 Kafka Lag 增长速度
难以关联 AI 耗时、消费吞吐和 Lag 的时间关系
难以判断是否与版本发布有关
缺少历史基线
难以量化受影响范围
```

### 问题四：AI 平台最重要的五个业务指标

```text
1. AI 调用成功率和错误率
2. AI 调用 P95、P99 耗时与超时率
3. Kafka Lag、消费速率和最老消息等待时间
4. 任务状态分布、最终成功率和长期 RUNNING 数量
5. 任务端到端完成耗时
```

可以继续扩展：

```text
Token 消耗
单任务平均成本
不同模型使用量
重试率
降级率
用户取消率
回调成功率
```

但最先建立的应该是能直接反映系统健康和用户体验的指标。

---

## 二十二、面试中怎么表达

可以这样表达：

> 在多实例 Java 服务中，我会同时建立基础监控和业务监控。基础监控包括 CPU、内存、JVM、GC、线程池、HTTP 连接池和数据库连接池；业务监控包括 AI 调用成功率与 P95/P99 耗时、Kafka Lag、异步任务状态分布、任务最终成功率和端到端耗时。Spring Boot 通过 Actuator 和 Micrometer 暴露指标，由 Prometheus 定时抓取，Grafana 展示趋势，Alertmanager 负责告警分组、去重和通知。

> 我不会只根据 CPU 和内存判断服务是否健康。例如 AI 调用耗时从 3 秒上升到 25 秒时，Worker 可能长时间处于 I/O 等待状态，CPU 和内存仍然正常，但单位时间完成任务数会下降，最终导致 Kafka Lag 持续增加。此时我会先按 AI 厂商和模型查看 P95、P99、超时率、429 和 5xx，再检查网络链路、HTTP 连接池、Worker 活跃线程和消费吞吐，而不是先盲目扩容。

> 如果只有日志，没有监控曲线，就很难确定异常开始时间、恶化速度、多个指标之间的时间关系和发布影响。我的排查思路是先通过监控发现异常和缩小范围，再结合 TraceId、TaskId 和错误日志定位具体请求和代码原因。

> 告警规则不能只配置一个瞬时阈值，还需要设置持续时间、告警等级、影响范围、通知对象和处理手册，并通过 Alertmanager 做分组、去重和抑制，避免告警风暴导致真正重要的故障被淹没。

---

## 二十三、今天内容的实际价值判断

只知道：

```text
Prometheus 采集指标
Grafana 展示图表
```

属于：

```text
基础概念认知
```

能够接入：

```text
Actuator
Micrometer
/actuator/prometheus
```

属于：

```text
基础监控接入能力
```

能够设计：

```text
系统指标
业务指标
标签维度
P95/P99
告警阈值
持续时间
```

属于：

```text
初级到中级监控工程能力
```

能够根据：

```text
AI 耗时
Kafka Lag
消费吞吐
线程池
连接池
任务成功率
端到端耗时
```

形成故障因果链并确定排查优先级，属于：

```text
中级后端线上排障能力
```

当前求职阶段合理学习边界是：

```text
理解 Prometheus Pull 模式
会接入 Spring Boot Actuator 和 Micrometer
理解 Counter、Gauge、Timer、Histogram
会区分系统指标和业务指标
会设计 AI、支付、PDF、MQ 业务指标
理解 P95、P99 和平均值的区别
理解四个黄金信号
会根据指标关联推断故障链路
会设计基础告警和避免告警风暴
```

暂时不需要优先深入：

```text
Prometheus 底层存储引擎源码
超大规模联邦集群
跨地域长期指标存储
自研监控平台
复杂 SRE 组织治理
```

当前更高回报的方向仍然是：

```text
Java 业务开发
MySQL
Redis
MQ
事务
并发
幂等
状态机
接口设计
日志
监控
故障排查
系统设计
```

---

## 二十四、今天的最终结论

第一个必须记住的区别：

```text
日志：发生了什么
监控：系统是否健康以及何时开始变坏
链路追踪：请求经过哪里以及哪一段最慢
```

第二个必须记住的结论：

```text
CPU 和内存正常
!=
业务系统正常
```

I/O 等待、线程池耗尽、连接池耗尽、第三方服务变慢和 Kafka 积压都可能在 CPU 不高时发生。

第三个必须记住的排障原则：

```text
根据已有指标证据确定排查优先级
而不是凭感觉平均怀疑所有组件
```

第四个必须记住的 AI 平台指标：

```text
AI 成功率
AI P95/P99 耗时
Kafka Lag 和最老消息等待时间
任务最终成功率与状态分布
任务端到端耗时
```

第五个必须记住的指标设计原则：

```text
指标名称清晰
标签低基数
系统指标与业务指标并存
平均值与分位数并存
指标能够支持实际排障和告警
```

第六个必须记住的告警原则：

```text
有阈值
有持续时间
有级别
有负责人
有上下文
有处理手册
有分组和抑制
```

第七个必须记住的完整排障链路：

```text
监控发现异常
↓
关联多个指标
↓
缩小故障范围
↓
链路追踪定位慢点
↓
日志确认具体原因
↓
处理并观察指标恢复
```

今天最核心的一句话是：

> 真正有价值的监控能力不是会搭一个 Grafana 仪表盘，而是能够选择真正影响业务的指标，根据延迟、流量、错误和饱和度发现异常，并结合 AI 调用耗时、Kafka Lag、Worker 吞吐和任务端到端耗时建立故障因果链，在用户投诉之前完成告警和定位。

能力等级：

```text
理解 Prometheus、Grafana、Alertmanager 的职责
→ 监控基础认知

能够接入 Actuator、Micrometer 并展示 JVM 和 HTTP 指标
→ 初级监控接入能力

能够设计业务指标、分位数、标签和告警规则
→ 初级到中级监控工程能力

能够结合监控、日志、链路追踪和业务状态定位复杂故障
→ 中级后端工程化与稳定性能力
```
