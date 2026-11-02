# 后端工程化每日训练 Day 20：ELK 集中日志、TraceId 链路关联与 Kibana 故障排查

## 一、今天学习的知识点

今天学习的是：

**ELK 集中日志、TraceId 链路关联与 Kibana 故障排查**

在单机小项目中，查看日志通常只需要：

```text
SSH 登录服务器
↓
tail -f app.log
```

但真实生产环境可能包含：

```text
Nginx
API-1
API-2
Worker-1
Worker-2
Scheduler
Kafka
MySQL
Redis
第三方 AI 服务
```

一次业务请求可能跨越多个服务和多个进程：

```text
客户端
↓
Nginx
↓
API
↓
Kafka
↓
Worker
↓
AI 服务
↓
MySQL
```

如果日志分散在不同机器和容器中，排查问题时就需要反复执行：

```text
登录机器 A
↓
grep API 日志
↓
登录机器 B
↓
grep Worker 日志
↓
登录机器 C
↓
检查 Scheduler 日志
↓
人工根据时间拼接调用过程
```

这会导致：

```text
排查速度慢
容易漏日志
跨服务日志难以关联
并发请求容易混淆
无法方便地统计错误趋势
无法统一设置日志保留周期
```

ELK 解决的核心问题是：

> 将分散在多个服务、多个容器和多台服务器上的日志统一采集、集中存储，并通过结构化字段和 TraceId 在 Kibana 中完成快速检索、过滤、关联和分析。

对于 Java 后端开发者，今天真正需要掌握的重点不是只记住：

```text
Filebeat 负责采集
Elasticsearch 负责存储
Kibana 负责查询
```

而是理解：

```text
什么日志值得打印
日志字段应该如何统一
TraceId、TaskId、RequestId 有什么区别
TraceId 如何跨 HTTP 和 MQ 传递
Worker 开始执行后没有日志应该如何排查
为什么没有日志不等于进程一定卡死
为什么不能看到 RUNNING 就直接认定任务失败
如何根据已有证据缩小排查范围
如何控制日志量、敏感数据和存储成本
日志、指标和链路追踪之间有什么边界
```

一句话理解：

> ELK 的价值不只是把日志放到一个网页里，而是建立一套能够跨服务定位真实生产问题的日志工程体系。

---

## 二、为什么生产环境需要集中日志

### 1. 单机日志的局限

假设系统只有一个 Spring Boot 服务：

```text
Client
↓
Spring Boot
↓
MySQL
```

出现问题时，可以直接执行：

```bash
grep "orderNo=202607270001" app.log
```

但当服务扩展成多实例后：

```text
Nginx
├── API-1
├── API-2
└── API-3
```

同一个用户请求可能进入任意一个实例。

如果不知道请求进入了哪一台机器，就需要逐台检查：

```text
API-1 没有
API-2 没有
API-3 找到
```

当系统进一步拆分成异步任务架构：

```text
API
↓
Kafka
↓
Worker
```

问题可能出现在：

```text
API 创建任务失败
消息发送失败
Kafka 未收到消息
Worker 未消费
Worker 执行阻塞
第三方接口超时
数据库状态更新失败
日志采集失败
```

单机日志已经无法支撑高效排查。

### 2. 集中日志解决的问题

集中日志系统希望做到：

```text
所有服务日志统一进入一个平台
通过业务字段直接检索
通过 TraceId 串联调用链
通过 serviceName 区分服务
通过 podName 或 hostName 区分实例
通过 level、errorCode、status 过滤问题
通过时间范围定位故障窗口
通过聚合统计观察错误趋势
```

例如用户反馈：

```text
TaskId=10001
一直处于 RUNNING
```

在 Kibana 中搜索：

```text
taskId:"10001"
```

可能看到：

```text
10:00:01 task-api     任务创建成功
10:00:01 task-api     Kafka 发送成功
10:00:02 task-worker  开始执行任务
10:00:03 ai-client    开始调用 AI 接口
10:01:03 ai-client    AI 接口读取超时
10:01:03 task-worker  更新任务失败状态
```

也可能只看到：

```text
10:00:01 task-api     任务创建成功
10:00:02 task-worker  开始执行任务
```

第二种情况不能直接下结论说：

```text
Worker 一定卡死
```

因为还存在多个可能性，需要继续结合容器、JVM、Kafka、数据库和日志采集链路排查。

---

## 三、ELK 和 Filebeat 的整体架构

常见日志链路是：

```text
Application
↓
日志文件或标准输出
↓
Filebeat
↓
Logstash（可选）
↓
Elasticsearch
↓
Kibana
```

也可以是：

```text
Application
↓
Filebeat
↓
Elasticsearch
↓
Kibana
```

在容器平台中，也可能变成：

```text
Spring Boot 容器标准输出
↓
容器运行时日志文件
↓
Filebeat / Fluent Bit
↓
Elasticsearch
↓
Kibana
```

不同组件的职责必须区分清楚。

### 1. Application

应用负责：

```text
生成日志
选择日志级别
打印业务字段
打印异常栈
控制敏感数据
控制日志内容和日志量
```

ELK 无法自动补救一份设计很差的日志。

如果应用只打印：

```text
执行失败
```

即使日志成功进入 Elasticsearch，也仍然无法知道：

```text
哪个任务失败
哪个用户失败
执行到哪一步
失败原因是什么
是否可以重试
```

因此日志工程的第一责任仍然在应用代码。

### 2. Filebeat

Filebeat 主要负责：

```text
监听日志文件
读取新增日志
记录读取位置
增加基础元数据
将日志发送到下游
```

它可以理解为轻量日志采集器。

Filebeat 通常不负责复杂业务分析，也不应该承载大量复杂转换逻辑。

常见能力包括：

```text
指定日志路径
识别多行异常栈
增加 serviceName、environment 等字段
发送到 Elasticsearch 或 Logstash
记录文件读取 offset
处理日志轮转
```

### 3. Logstash

Logstash 是可选组件，主要用于：

```text
解析非结构化日志
字段转换
字段重命名
数据清洗
过滤无效日志
脱敏
多路输出
复杂路由
```

例如原始日志是：

```text
2026-07-27 10:00:01 INFO taskId=10001 userId=20008 Start task
```

Logstash 可以将其转换成：

```json
{
  "@timestamp": "2026-07-27T10:00:01+08:00",
  "level": "INFO",
  "taskId": "10001",
  "userId": "20008",
  "message": "Start task"
}
```

但如果应用本身已经输出规范 JSON 日志，很多场景可以直接：

```text
Filebeat
↓
Elasticsearch
```

减少一层组件和运维成本。

### 4. Elasticsearch

Elasticsearch 负责：

```text
存储日志文档
建立倒排索引
按字段过滤
全文检索
聚合统计
时间范围查询
```

它不是传统关系数据库。

日志通常按照文档存储，每条日志是一条 JSON 文档：

```json
{
  "@timestamp": "2026-07-27T10:00:01.123+08:00",
  "level": "INFO",
  "serviceName": "task-worker",
  "traceId": "abc123",
  "taskId": "10001",
  "status": "RUNNING",
  "message": "Start AI task"
}
```

### 5. Kibana

Kibana 是开发、测试、运维和 SRE 经常使用的查询入口。

主要能力包括：

```text
Discover 日志检索
字段过滤
时间范围筛选
保存查询
聚合统计
Dashboard 可视化
错误趋势分析
索引和数据视图管理
```

开发人员真正排查线上问题时，通常直接面对的是 Kibana，而不是直接操作 Elasticsearch API。

---

## 四、完整日志链路是如何工作的

一条日志从代码到 Kibana，大致经历：

```text
Spring Boot 执行业务代码
↓
Logback 生成日志
↓
日志写入文件或标准输出
↓
Filebeat 读取新增内容
↓
Filebeat 增加主机、容器、环境等元数据
↓
发送到 Elasticsearch
↓
Elasticsearch 建立索引
↓
Kibana 根据数据视图查询
```

如果 Kibana 中没有看到日志，故障点可能在任意一层：

```text
代码根本没有打印
日志级别被过滤
异常被 catch 后吞掉
日志写错文件
容器标准输出未被采集
Filebeat 路径配置错误
Filebeat 没有权限读取文件
多行异常栈解析错误
Filebeat 发送失败
Elasticsearch 磁盘满
Elasticsearch 索引只读
字段映射冲突
Kibana 时间范围选错
Kibana 数据视图选错
查询条件写错
```

因此：

> Kibana 中没有日志，只能说明当前查询没有找到日志，不能直接说明业务代码没有执行。

---

## 五、结构化日志为什么重要

### 1. 非结构化日志的问题

以下日志人眼可以看懂：

```text
任务 10001 执行失败，用户 20008，原因是 AI timeout
```

但如果所有信息都放在 message 字符串中：

```json
{
  "message": "任务 10001 执行失败，用户 20008，原因是 AI timeout"
}
```

会导致：

```text
字段无法稳定聚合
格式变化后查询失效
难以按 taskId 精确过滤
难以按 errorCode 统计
难以按 status 制作 Dashboard
```

### 2. 结构化日志

更合理的方式是：

```json
{
  "@timestamp": "2026-07-27T10:00:01.123+08:00",
  "level": "ERROR",
  "serviceName": "task-worker",
  "traceId": "trace-a1b2c3",
  "taskId": "10001",
  "userId": "20008",
  "status": "FAILED",
  "errorCode": "AI_TIMEOUT",
  "durationMs": 60000,
  "message": "AI request timed out"
}
```

这样可以直接查询：

```text
taskId:"10001"
```

```text
errorCode:"AI_TIMEOUT"
```

```text
serviceName:"task-worker" and level:"ERROR"
```

```text
status:"FAILED" and durationMs > 30000
```

结构化日志的核心收益是：

```text
精确查询
稳定过滤
方便聚合
便于告警
减少人工阅读
```

### 3. 不要求所有日志完全相同

统一日志字段不等于所有服务只能打印完全一样的内容。

更合理的设计是：

```text
公共字段统一
+
业务字段按场景扩展
```

公共字段例如：

```text
@timestamp
level
serviceName
environment
traceId
requestId
hostName
podName
message
```

AI 任务业务字段：

```text
taskId
model
provider
status
retryCount
durationMs
```

支付业务字段：

```text
orderNo
outTradeNo
channel
payStatus
amount
callbackId
```

广告业务字段：

```text
campaignId
accountId
creativeId
platform
operation
thirdPartyCode
```

---

## 六、团队应该统一哪些日志字段

日志字段不能无限增加，但以下字段具有较高通用价值。

### 1. 时间和级别

```text
@timestamp
level
```

用途：

```text
确定发生时间
按时间窗口排查
区分 INFO、WARN、ERROR
```

### 2. 服务和实例

```text
serviceName
environment
hostName
podName
instanceId
```

用途：

```text
确定是哪一个服务
确定是测试还是生产环境
确定是哪一个容器或实例
判断是否只有单节点异常
```

### 3. 调用链字段

```text
traceId
spanId
requestId
```

用途：

```text
串联跨服务调用
区分一次请求中的不同步骤
定位单次请求
```

### 4. 业务标识字段

```text
taskId
orderNo
campaignId
userId
messageId
jobId
```

用途：

```text
定位具体业务对象
关联数据库记录
关联 MQ 消息
关联调度任务
```

### 5. 状态和结果字段

```text
status
success
errorCode
errorType
retryCount
```

用途：

```text
判断业务执行结果
统计失败类型
区分可重试和不可重试错误
```

### 6. 性能字段

```text
durationMs
queueWaitMs
thirdPartyDurationMs
dbDurationMs
```

用途：

```text
定位慢接口
区分排队时间和执行时间
判断第三方接口是否变慢
```

### 7. HTTP 字段

```text
httpMethod
requestPath
statusCode
clientIp
userAgent
```

用途：

```text
定位接口
统计 HTTP 错误
分析请求来源
```

### 8. 异常字段

```text
exceptionClass
exceptionMessage
exceptionStack
```

用途：

```text
查看异常类型
保留完整堆栈
定位代码位置
```

需要注意：

```text
message 可以给人阅读
errorCode 用于稳定分类
exceptionStack 用于代码定位
```

不能只打印一句：

```text
处理失败
```

---

## 七、TraceId、RequestId、TaskId 和 MessageId 的区别

这是今天必须明确区分的概念。

### 1. TraceId

TraceId 表示：

```text
一次调用链
```

例如：

```text
客户端请求
↓
API
↓
用户服务
↓
订单服务
↓
支付服务
```

这整条链路可以使用同一个 TraceId：

```text
traceId=trace-A
```

TraceId 的主要作用是：

```text
将多个服务中的日志关联起来
```

### 2. RequestId

RequestId 通常表示：

```text
一次具体 HTTP 请求
```

一个 Trace 中可能包含多个下游请求，因此可能存在：

```text
traceId=trace-A
requestId=request-1
requestId=request-2
requestId=request-3
```

实际项目中，简单系统也可能暂时将 TraceId 和 RequestId 设计成相同值，但概念上仍然不同。

### 3. TaskId

TaskId 表示：

```text
一个业务任务
```

例如：

```text
AI 生成任务 10001
PDF 转换任务 20001
广告发布任务 30001
```

同一个任务可能经历：

```text
第一次执行
自动重试
人工重试
补偿执行
定时扫描恢复
```

这些执行可能产生不同 TraceId：

```text
taskId=10001 traceId=trace-A 第一次执行
taskId=10001 traceId=trace-B 自动重试
taskId=10001 traceId=trace-C 人工重试
```

所以不能说：

```text
TraceId 就是定义一个任务的编号
```

正确理解是：

```text
TaskId 标识业务任务
TraceId 标识一次调用链或一次执行链路
```

### 4. MessageId

MessageId 表示：

```text
一条 MQ 消息
```

同一个 TaskId 可能发送多条消息：

```text
任务创建消息
任务执行消息
任务完成事件
任务失败事件
```

因此：

```text
taskId != messageId
```

### 5. JobId

JobId 通常表示：

```text
一个调度任务定义
```

如果使用 XXL-JOB，还可能有：

```text
jobId
logId
executorAddress
triggerTime
```

一个 Job 可以扫描和处理多个 Task。

### 6. 推荐理解方式

```text
TraceId
一次调用链

RequestId
一次 HTTP 请求

TaskId
一个业务任务

MessageId
一条 MQ 消息

JobId
一个调度任务定义
```

排查时常见组合：

```text
先用 taskId 找业务任务
↓
找到相关 traceId
↓
用 traceId 串联本次执行链路
↓
再根据 messageId、requestId、podName 缩小范围
```

---

## 八、TraceId 如何在 HTTP 请求中传递

典型流程：

```text
客户端请求进入系统
↓
网关或第一个 Spring Boot 服务读取请求头
↓
如果请求头没有 TraceId，则生成新的 TraceId
↓
写入 MDC
↓
日志自动打印 TraceId
↓
调用下游 HTTP 服务时继续透传 TraceId
↓
请求结束后清理 MDC
```

### 1. 生成或读取 TraceId

示例：

```java
@Component
public class TraceIdFilter extends OncePerRequestFilter {

    private static final String TRACE_ID = "traceId";
    private static final String TRACE_HEADER = "X-Trace-Id";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String traceId = request.getHeader(TRACE_HEADER);
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }

        try {
            MDC.put(TRACE_ID, traceId);
            response.setHeader(TRACE_HEADER, traceId);
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(TRACE_ID);
        }
    }
}
```

### 2. 为什么必须清理 MDC

MDC 通常基于线程上下文。

Tomcat 线程会被线程池复用：

```text
线程 thread-1 处理请求 A
↓
请求 A 结束
↓
thread-1 被复用处理请求 B
```

如果没有清理 MDC：

```text
请求 B 可能错误地携带请求 A 的 TraceId
```

因此必须使用：

```java
try {
    MDC.put("traceId", traceId);
    // 业务执行
} finally {
    MDC.remove("traceId");
}
```

或者：

```java
MDC.clear();
```

但使用 `clear()` 时要确认不会误删当前线程中其他需要保留的上下文字段。

### 3. 调用下游服务时继续传递

例如使用 HTTP 客户端调用下游服务时，需要增加请求头：

```text
X-Trace-Id: trace-A
```

如果不传递：

```text
API 日志 traceId=trace-A
下游服务自己生成 traceId=trace-B
```

这两段日志就无法直接关联。

---

## 九、TraceId 如何跨 Kafka 或 RabbitMQ 传递

HTTP 请求结束后，线程上下文会消失。

异步任务链路：

```text
API
↓
Kafka
↓
Worker
```

Worker 无法自动继承 API 线程中的 MDC。

因此生产者必须主动把 TraceId 写入消息。

### 1. 放在消息体中

```json
{
  "messageId": "msg-10001",
  "traceId": "trace-A",
  "taskId": "10001",
  "eventType": "TASK_CREATED",
  "payload": {}
}
```

### 2. 放在消息 Header 中

也可以放在 MQ Header：

```text
traceId=trace-A
messageId=msg-10001
```

### 3. 消费者恢复 MDC

```java
public void consume(TaskMessage message) {
    try {
        MDC.put("traceId", message.getTraceId());
        MDC.put("taskId", message.getTaskId());

        log.info("Start task execution");
        taskService.execute(message);
    } finally {
        MDC.remove("traceId");
        MDC.remove("taskId");
    }
}
```

### 4. 重试时 TraceId 如何处理

重试场景有两种设计。

#### 方案一：沿用原 TraceId

适合希望把原始执行和快速重试视为同一条链路的场景。

```text
原执行 traceId=trace-A
重试 traceId=trace-A
```

优点：

```text
查询简单
可以看到完整重试过程
```

缺点：

```text
多次执行边界不够清晰
```

#### 方案二：生成新 TraceId，并保留 ParentTraceId

```text
第一次执行
traceId=trace-A

第二次重试
traceId=trace-B
parentTraceId=trace-A
```

优点：

```text
每次执行边界清晰
仍然可以追溯来源
```

对于长周期任务、人工重试和补偿任务，第二种方式通常更容易分析。

---

## 十、Spring Boot 日志格式设计

### 1. 最低可用文本格式

```text
2026-07-27 10:00:01.123 INFO
service=task-worker
traceId=trace-A
taskId=10001
status=RUNNING
message=Start AI task
```

### 2. 推荐 JSON 格式

```json
{
  "@timestamp": "2026-07-27T10:00:01.123+08:00",
  "level": "INFO",
  "serviceName": "task-worker",
  "environment": "prod",
  "traceId": "trace-A",
  "taskId": "10001",
  "status": "RUNNING",
  "message": "Start AI task"
}
```

JSON 日志更容易被 Filebeat 和 Elasticsearch 解析。

### 3. Logback Pattern 示例

```xml
<property name="LOG_PATTERN"
          value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level service=${spring.application.name} traceId=%X{traceId:-} taskId=%X{taskId:-} %logger{36} - %msg%n"/>
```

示例输出：

```text
2026-07-27 10:00:01.123 [task-worker-1] INFO service=task-worker traceId=trace-A taskId=10001 TaskService - Start task
```

### 4. JSON Encoder 示例

实际项目可以使用支持 JSON 输出的 Logback Encoder，将 MDC 字段作为独立字段输出。

核心目标不是依赖某个固定组件，而是确保：

```text
日志天然是结构化数据
字段名稳定
异常栈完整
MDC 字段能够进入 JSON
```

---

## 十一、Filebeat 最小配置示例

假设应用日志路径是：

```text
/var/log/myapp/*.log
```

基础配置示例：

```yaml
filebeat.inputs:
  - type: filestream
    id: task-service-log
    enabled: true
    paths:
      - /var/log/myapp/*.log

    fields:
      serviceName: task-service
      environment: prod

    fields_under_root: true

output.elasticsearch:
  hosts:
    - "http://elasticsearch:9200"

  index: "backend-log-%{+yyyy.MM.dd}"

setup.kibana:
  host: "http://kibana:5601"
```

如果应用输出 JSON：

```yaml
filebeat.inputs:
  - type: filestream
    id: task-service-json-log
    paths:
      - /var/log/myapp/*.json

    parsers:
      - ndjson:
          target: ""
          add_error_key: true
```

需要注意：

```text
不同 Filebeat 版本支持的具体配置可能不同
生产使用前必须以当前官方文档和实际版本为准
```

### 1. 多行异常栈

Java 异常通常包含多行：

```text
java.lang.NullPointerException
    at com.example.TaskService.execute(TaskService.java:100)
    at com.example.Worker.consume(Worker.java:50)
```

如果没有正确处理多行日志，Elasticsearch 可能把每一行都当成独立日志：

```text
第一条：java.lang.NullPointerException
第二条：at com.example.TaskService...
第三条：at com.example.Worker...
```

这会破坏异常上下文。

所以需要根据日志格式配置 multiline，或者直接使用 JSON 日志并将异常栈作为一个字段输出。

### 2. 文件轮转

应用日志通常会轮转：

```text
app.log
app.2026-07-26.log
app.2026-07-25.log.gz
```

Filebeat 需要正确处理：

```text
文件重命名
文件截断
新文件创建
旧文件删除
```

否则可能产生：

```text
重复采集
漏采集
读取位置错误
```

---

## 十二、Logstash 什么时候值得使用

不是所有系统都必须上 Logstash。

### 1. 可以不使用 Logstash 的情况

```text
应用已经输出 JSON 日志
字段命名统一
只需要简单增加环境字段
不需要复杂转换
日志量不大
链路希望尽量简单
```

架构：

```text
Filebeat
↓
Elasticsearch
```

优点：

```text
组件更少
资源消耗更低
故障点更少
维护成本更低
```

### 2. 适合使用 Logstash 的情况

```text
多个历史系统日志格式不同
需要复杂 Grok 解析
需要统一字段
需要删除或脱敏敏感字段
需要根据条件路由到不同索引
需要同时输出到多个下游
```

架构：

```text
Filebeat
↓
Logstash
↓
Elasticsearch
```

### 3. Logstash 的代价

```text
额外资源消耗
额外部署组件
配置复杂度增加
解析失败可能丢字段
处理能力不足时形成积压
```

所以正确判断不是：

```text
ELK 必须四个组件全部部署
```

而是：

```text
根据日志格式和转换需求决定是否需要 Logstash
```

---

## 十三、Kibana 中常见查询方式

下面使用类似 KQL 的表达方式说明查询思路。

### 1. 根据 TaskId 查询

```text
taskId:"10001"
```

### 2. 根据 TraceId 查询

```text
traceId:"trace-A"
```

### 3. 查询某个服务的错误日志

```text
serviceName:"task-worker" and level:"ERROR"
```

### 4. 查询支付回调失败

```text
serviceName:"payment-service"
and eventType:"PAY_CALLBACK"
and status:"FAILED"
```

### 5. 查询第三方限流

```text
thirdPartyCode:"429"
```

### 6. 查询慢请求

```text
durationMs > 3000
```

### 7. 查询某个任务的多次执行

```text
taskId:"10001"
```

然后重点观察：

```text
traceId
retryCount
messageId
podName
@timestamp
```

### 8. 排除健康检查日志

```text
not requestPath:"/actuator/health"
```

### 9. 查询生产环境

```text
environment:"prod"
```

### 10. 查询某时间范围内的错误趋势

Kibana 中可以选择时间窗口，例如：

```text
最近 15 分钟
最近 1 小时
自定义故障时间段
```

再筛选：

```text
level:"ERROR"
```

排查时必须注意：

```text
Kibana 默认时间范围可能不是故障发生时间
时区配置可能造成时间偏差
字段可能是 keyword 或 text
查询语法要与实际数据视图匹配
```

---

## 十四、真实业务场景一：AI 异步任务一直 RUNNING

用户反馈：

```text
TaskId=10001
一直 RUNNING
```

### 1. 第一步：确认业务状态

先查数据库：

```sql
SELECT
    id,
    status,
    created_at,
    started_at,
    updated_at,
    finished_at,
    retry_count,
    worker_id
FROM ai_task
WHERE id = 10001;
```

需要确认：

```text
任务什么时候创建
什么时候进入 RUNNING
最后更新时间
是否超过正常执行时长
是否有重试次数
是否记录 Worker 实例
```

不能只看到：

```text
status=RUNNING
```

就立刻认定异常。

因为可能是：

```text
任务本身正常需要较长时间
任务仍在排队
任务正在下载大文件
AI 模型仍在推理
PDF 仍在转换
OCR 仍在识别
```

所以需要先知道：

```text
正常耗时范围
当前已运行多久
是否有阶段进度
```

### 2. 第二步：查询 TaskId

```text
taskId:"10001"
```

可能看到：

```text
API：任务创建成功
API：Kafka 发送成功
Worker：开始执行
AI Client：开始请求
```

然后没有后续日志。

### 3. 第三步：不要重新从 Nginx 开始

既然已经确认：

```text
API 创建成功
Worker 已经开始执行
```

说明请求入口和消息消费链路至少已经走到 Worker。

此时 Nginx 不是第一优先级。

更合理的排查顺序是：

```text
数据库任务状态和更新时间
↓
Worker 容器是否存活或重启
↓
任务当前运行时长是否超出正常范围
↓
JVM、线程和连接池状态
↓
第三方接口、数据库、对象存储等下游依赖
↓
Kafka offset、重复消费和提交状态
↓
Filebeat、Elasticsearch 日志采集链路
```

Nginx 只有在以下问题中优先级较高：

```text
用户请求是否进入系统
接口是否返回 502、504
上传是否被 413 拒绝
SSE 或 WebSocket 是否被代理中断
```

对于已经由 Worker 执行的后台任务，Nginx 通常不在核心执行链路中。

---

## 十五、Worker 开始执行后没有日志的可能原因

### 1. 任务仍在正常执行

这是最容易被忽略的可能性。

例如：

```text
大文件下载需要 3 分钟
OCR 识别需要 5 分钟
AI 推理需要 8 分钟
PDF 合并需要 2 分钟
```

如果代码只打印：

```text
开始执行
```

和：

```text
执行完成
```

中间没有阶段日志，就会出现长时间空白。

改进方式：

```text
记录阶段开始和结束
记录处理进度
记录本阶段耗时
记录外部调用开始和返回
```

例如：

```text
10:00:00 Start task
10:00:01 Start download file
10:01:10 Download completed durationMs=69000
10:01:11 Start OCR
10:03:30 OCR completed durationMs=139000
10:03:31 Start AI analysis
```

### 2. 卡在第三方 HTTP 请求

例如：

```text
Worker 调用 AI 服务
↓
连接建立成功
↓
对方迟迟不返回完整响应
```

如果没有合理配置：

```text
connect timeout
read timeout
call timeout
```

线程可能长时间等待。

需要检查：

```text
第三方接口开始日志
HTTP 客户端超时配置
连接池状态
对方服务监控
线程栈
```

### 3. 卡在数据库锁或慢 SQL

可能情况：

```text
等待行锁
等待表锁
SQL 扫描大量数据
数据库连接池耗尽
事务长时间未提交
```

需要检查：

```text
MySQL 当前连接
慢 SQL
锁等待
事务状态
连接池 active、idle、pending
```

### 4. 线程池耗尽

例如 Worker 消费线程又将任务提交到业务线程池：

```text
Kafka Consumer
↓
业务线程池
```

如果线程池已经满：

```text
任务可能排队
任务可能被拒绝
任务可能阻塞提交线程
```

需要检查：

```text
线程池 activeCount
queueSize
completedTaskCount
rejectedCount
```

### 5. 线程死锁或线程饥饿

例如：

```text
线程 A 等待线程 B
线程 B 又等待线程 A
```

或者线程池中的任务都在等待同一个线程池中的子任务。

需要使用：

```text
jstack
线程 dump
JVM 诊断工具
```

检查线程状态：

```text
RUNNABLE
WAITING
TIMED_WAITING
BLOCKED
```

### 6. JVM Full GC 或 OOM

可能表现：

```text
服务长时间无响应
日志突然中断
容器重启
任务状态停留在 RUNNING
```

需要检查：

```text
GC 日志
Heap 使用率
容器内存限制
OOMKilled 状态
JVM 退出日志
```

### 7. 容器被杀死或重启

例如：

```text
容器内存超限
健康检查失败
宿主机重启
部署覆盖旧容器
人工重启
```

需要检查：

```bash
docker ps -a
docker inspect <container>
docker logs <container>
```

重点关注：

```text
ExitCode
OOMKilled
RestartCount
StartedAt
FinishedAt
```

### 8. 异常被吞掉

错误代码示例：

```java
try {
    executeTask();
} catch (Exception ignored) {
}
```

或者：

```java
try {
    executeTask();
} catch (Exception e) {
    log.info("Task finished");
}
```

后果：

```text
真实异常没有 ERROR 日志
任务状态没有更新
数据库仍然 RUNNING
```

正确方式至少应包括：

```java
try {
    executeTask();
} catch (Exception e) {
    log.error("Task execution failed, taskId={}", taskId, e);
    markTaskFailed(taskId, e);
    throw e;
}
```

是否重新抛出异常要根据 MQ 重试和业务语义决定，但不能无声吞掉异常。

### 9. 业务已经完成，但状态更新失败

链路可能是：

```text
AI 调用成功
↓
结果已生成
↓
更新数据库 SUCCESS 失败
↓
事务回滚
↓
任务仍然显示 RUNNING
```

这时任务并没有卡在执行逻辑，而是卡在：

```text
最终状态落库
```

需要检查：

```text
结果是否已经写入对象存储
状态更新 SQL 是否执行
事务是否回滚
乐观锁是否冲突
数据库是否短暂不可用
```

### 10. 日志采集链路异常

Worker 可能正常执行，但 Kibana 没有新日志。

可能原因：

```text
日志写入了新文件
日志路径变化
Filebeat 未采集该路径
Filebeat 挂掉
Filebeat 无读取权限
Elasticsearch 写入失败
索引只读
Kibana 查询时间范围错误
```

所以：

```text
没有 Kibana 日志
!=
没有本地日志
!=
代码没有执行
```

---

## 十六、针对 RUNNING 任务的标准排查流程

### 第一步：确认是否真的超时

```text
查询任务创建时间
查询开始时间
查询最后更新时间
对比该任务类型正常耗时
```

例如：

```text
正常耗时 10 分钟
当前执行 3 分钟
```

此时不应立即认定故障。

如果：

```text
正常耗时 2 分钟
当前执行 40 分钟
```

则应进入异常排查。

### 第二步：查业务标识

```text
taskId:"10001"
```

查看：

```text
最后一条日志是什么
来自哪个 serviceName
来自哪个 podName
对应哪个 traceId
是否存在多个 traceId
是否发生过重试
```

### 第三步：查本次 TraceId

```text
traceId:"trace-A"
```

按时间排序，确认链路执行到了哪一步。

### 第四步：确认 Worker 实例状态

```text
Worker 容器是否还在
是否刚刚重启
是否 OOMKilled
CPU 和内存是否异常
```

### 第五步：查看 JVM 状态

```text
GC 是否异常
线程是否阻塞
线程池是否耗尽
连接池是否耗尽
是否存在死锁
```

### 第六步：检查下游依赖

```text
AI 服务
MySQL
Redis
对象存储
第三方广告接口
文件系统
```

### 第七步：检查 Kafka

```text
消息是否已经消费
offset 是否提交
consumer group 是否 rebalance
是否发生重复消费
是否进入重试 Topic 或死信队列
```

### 第八步：检查日志采集链路

```text
Worker 本地日志是否存在
Filebeat 是否运行
Filebeat 是否有发送错误
Elasticsearch 是否可写
索引是否正常
Kibana 查询是否正确
```

### 第九步：恢复任务

根据原因决定：

```text
继续等待
人工终止
标记失败
重新投递
执行补偿
转移到其他 Worker
```

恢复动作必须考虑幂等，避免：

```text
原任务其实还在运行
+
人工又重试一次
↓
同一任务并发执行两次
```

---

## 十七、Kafka 应该检查什么

如果 Worker 已经打印：

```text
开始执行任务
```

说明消息大概率已经被某个消费者获取。

此时检查 Kafka 的目的不是简单判断：

```text
消息有没有发送
```

而是进一步确认：

```text
是否发生 Rebalance
offset 是否已经提交
消息是否会被重新消费
是否存在重复执行风险
消费者是否仍然存活
同一 Partition 是否被阻塞
```

需要关注：

```text
topic
partition
offset
consumerGroup
consumerInstance
messageId
retryCount
```

例如日志：

```json
{
  "serviceName": "task-worker",
  "traceId": "trace-A",
  "taskId": "10001",
  "messageId": "msg-88",
  "topic": "ai-task",
  "partition": 3,
  "offset": 18201,
  "consumerGroup": "ai-worker-group",
  "message": "Start consuming task"
}
```

如果 Worker 长时间执行，且超过：

```text
max.poll.interval.ms
```

可能触发 Rebalance，消息可能被其他消费者重新读取。

因此长任务需要考虑：

```text
消费线程与执行线程解耦
合理设置 max.poll.interval.ms
控制 max.poll.records
正确提交 offset
业务幂等
任务状态机防并发
```

---

## 十八、Docker 容器应该检查什么

### 1. 容器是否存在

```bash
docker ps -a
```

### 2. 容器是否重启

```bash
docker inspect task-worker
```

检查：

```text
RestartCount
StartedAt
FinishedAt
ExitCode
OOMKilled
```

### 3. 查看容器日志

```bash
docker logs --since 30m task-worker
```

如果 Kibana 没日志，但 `docker logs` 有日志，说明问题更可能在：

```text
日志采集链路
```

如果 `docker logs` 也没有日志，可能是：

```text
代码没有打印
线程阻塞
进程退出
日志写到其他文件
日志级别被过滤
```

### 4. 检查资源

```bash
docker stats task-worker
```

关注：

```text
CPU
Memory
Network IO
Block IO
```

### 5. 检查容器时间

如果容器时区错误，日志时间可能与 Kibana 查询时间不一致。

表现为：

```text
日志其实存在
但出现在错误的时间窗口
```

---

## 十九、JVM 应该检查什么

### 1. 线程状态

使用线程 dump 检查：

```text
是否大量 BLOCKED
是否大量 WAITING
是否存在死锁
是否卡在 HTTP 客户端
是否卡在数据库驱动
是否卡在线程池队列
```

### 2. GC 状态

关注：

```text
Young GC 频率
Full GC 次数
GC 停顿时间
Heap 使用率
Old 区增长
```

### 3. 内存问题

```text
Java Heap OOM
Direct Memory OOM
Metaspace OOM
容器内存限制触发 OOM Kill
```

### 4. 线程池

```text
核心线程数
最大线程数
活动线程数
队列长度
拒绝次数
```

### 5. 连接池

```text
数据库连接池 active
数据库连接池 idle
等待连接线程数
HTTP 连接池 leased
HTTP 连接池 pending
```

如果连接池耗尽，业务线程可能一直等待连接，看起来像任务卡死。

---

## 二十、Nginx 日志什么时候值得看

今天的场景中，Worker 已经开始执行，因此 Nginx 不是第一排查对象。

但以下场景应该优先检查 Nginx：

### 1. 用户请求根本没有进入 API

```text
前端报错
API 没有任何访问日志
```

检查：

```text
Nginx access.log
Nginx error.log
路由配置
upstream 状态
```

### 2. 返回 502

可能原因：

```text
后端容器退出
端口未监听
upstream 地址错误
网络不通
后端提前断开连接
```

### 3. 返回 504

可能原因：

```text
后端处理太慢
下游接口太慢
proxy_read_timeout 太小
```

### 4. 上传返回 413

检查：

```nginx
client_max_body_size
```

### 5. SSE 或 WebSocket 中断

检查：

```text
代理缓冲
连接超时
Upgrade Header
长连接配置
```

结论：

> 排查应该根据已有证据缩小范围，而不是每次都机械地从 Nginx 开始把所有组件检查一遍。

---

## 二十一、没有 TraceId 会增加哪些排查成本

用户当前的初步理解是：

```text
没有 TraceId 就需要一个一个看日志
```

这个方向正确，但可以进一步拆解。

### 1. 需要人工根据时间拼接

例如：

```text
10:00:01 API 收到请求
10:00:01 API 创建任务
10:00:02 Worker 开始执行
10:00:03 另一个 Worker 也开始执行
```

没有 TraceId 时，很难确认哪些日志属于同一次调用。

### 2. 并发请求容易混淆

同一秒可能有大量相同接口请求：

```text
用户 A 创建任务
用户 B 创建任务
用户 C 创建任务
```

日志内容高度相似，仅靠时间无法准确关联。

### 3. 重试链路难以区分

同一个 TaskId 可能执行多次：

```text
第一次执行
自动重试
人工重试
补偿任务
```

没有 TraceId 时，容易把多次执行混成一次。

### 4. 跨服务无法直接关联

需要在：

```text
API
Worker
Scheduler
第三方客户端
```

之间反复切换，并人工判断日志关系。

### 5. 容易误判根因

例如看到：

```text
Worker A 超时
Worker B 成功
```

如果无法区分不同调用链，可能错误地认为同一个任务先失败后成功，或者误把其他任务的错误当成本任务错误。

### 6. 排查耗时明显增加

有 TraceId：

```text
搜索一个字段
↓
查看完整链路
```

没有 TraceId：

```text
查时间
查用户
查 TaskId
查实例
查上下文
人工拼接
反复验证
```

面试中可以表达为：

> 如果没有 TraceId，跨服务日志无法直接关联，只能根据时间、业务编号、实例和日志内容人工拼接。在高并发、重试和异步消费场景下，不同请求的日志容易混杂，会显著增加检索、关联和确认根因的成本。

---

## 二十二、日志级别应该如何使用

### 1. DEBUG

用于：

```text
开发调试
详细参数
内部流程细节
```

生产环境通常不会长期大量开启 DEBUG。

### 2. INFO

用于正常业务关键节点：

```text
任务创建
任务开始
任务完成
支付回调收到
消息发送成功
状态流转
```

INFO 不是打印所有代码执行细节，而是记录可理解的业务过程。

### 3. WARN

用于：

```text
发生异常但系统能够恢复
触发重试
数据不完整但有默认处理
第三方接口短暂失败
接近资源阈值
```

例如：

```text
第一次调用超时，准备重试
```

### 4. ERROR

用于：

```text
业务最终失败
数据一致性风险
无法自动恢复
关键依赖不可用
需要人工介入
```

### 5. 常见错误

#### 所有异常都打印 ERROR

后果：

```text
ERROR 数量巨大
告警失去意义
真实故障被淹没
```

#### 所有日志都打印 INFO

后果：

```text
无法快速筛选故障
```

#### 同一个异常重复打印多次

例如：

```text
DAO 层打印 ERROR
Service 层再次打印 ERROR
Controller 层再次打印 ERROR
```

一次异常产生三份堆栈，增加噪音和存储成本。

更合理的原则是：

```text
在最有业务上下文、能够决定处理结果的位置记录一次完整异常
```

---

## 二十三、异常日志应该怎么写

错误写法：

```java
log.error("执行失败：{}", e.getMessage());
```

问题：

```text
只有异常消息
没有堆栈
不知道代码行
```

推荐写法：

```java
log.error(
    "Task execution failed, taskId={}, status={}, errorCode={}",
    taskId,
    taskStatus,
    errorCode,
    e
);
```

这样同时保留：

```text
业务上下文
稳定错误码
完整异常栈
```

但也不能将整个大对象都拼进日志：

```java
log.error("Request={}, Response={}, User={}, Task={}", request, response, user, task, e);
```

可能导致：

```text
日志过大
敏感信息泄露
序列化失败
循环引用
查询性能下降
```

---

## 二十四、为什么不能打印完整大对象

例如 AI 服务返回：

```text
5MB JSON
```

如果每次都完整记录：

```text
请求体 5MB
响应体 5MB
重试三次
```

一个任务可能产生几十 MB 日志。

后果：

```text
应用磁盘快速增长
Filebeat 网络流量增加
Elasticsearch 写入压力增加
索引体积暴涨
Kibana 查询变慢
存储成本上升
```

更合理的方式：

```text
记录请求摘要
记录响应摘要
记录 bodySize
记录对象存储地址
记录 hash
记录 traceId
```

例如：

```json
{
  "traceId": "trace-A",
  "taskId": "10001",
  "provider": "example-ai",
  "requestSize": 10240,
  "responseSize": 5242880,
  "responseObjectKey": "ai-response/2026/07/27/10001.json",
  "durationMs": 61000
}
```

---

## 二十五、日志中的敏感信息必须控制

不能直接记录：

```text
密码
Access Token
Refresh Token
API Key
银行卡号
身份证号
完整手机号
支付签名密钥
Cookie
Authorization Header
完整用户隐私数据
```

错误示例：

```text
Authorization: Bearer eyJhbGciOi...
```

```text
password=123456
```

```text
cardNo=622202xxxxxxxxxx
```

推荐方式：

```text
Token 不打印
手机号仅保留部分位数
身份证脱敏
订单金额按业务需要记录
第三方响应只保留必要错误字段
```

例如：

```text
phone=138****8888
```

同时需要注意：

```text
日志系统通常有较多人可以访问
日志保留时间可能很长
日志备份可能复制到多个位置
```

所以日志脱敏不能只依赖使用者自觉，而应尽可能在代码规范、公共组件和 Logstash 规则中统一处理。

---

## 二十六、日志生命周期和存储成本

如果日志永久保存：

```text
每天 20GB
30 天 600GB
一年约 7.3TB
```

还没有计算：

```text
副本
索引开销
备份
冷热迁移
```

因此需要设置日志生命周期。

常见阶段：

```text
热数据
温数据
冷数据
自动删除
```

### 1. 热数据

特点：

```text
最近日志
查询频繁
写入频繁
```

适合保留在性能较高节点。

### 2. 温数据

特点：

```text
查询频率下降
仍可能用于问题复盘
```

### 3. 冷数据

特点：

```text
很少查询
主要用于审计或历史追溯
```

### 4. 自动删除

根据业务要求设置：

```text
普通应用 INFO 日志保留 7 到 30 天
ERROR 日志保留更长
审计日志按合规要求保留
```

具体时间必须由：

```text
业务价值
合规要求
磁盘成本
故障追溯周期
```

共同决定。

不能简单照搬固定天数。

---

## 二十七、索引设计和字段映射风险

### 1. 按环境和服务区分

索引可以按以下维度设计：

```text
backend-prod-task-api-2026.07.27
backend-prod-task-worker-2026.07.27
backend-test-task-api-2026.07.27
```

也可以使用较统一的索引，再通过字段过滤：

```text
backend-prod-2026.07.27
```

选择取决于：

```text
日志量
权限隔离
查询习惯
生命周期策略
服务数量
```

### 2. 字段类型必须稳定

例如 `taskId` 一会儿是数字：

```json
{"taskId": 10001}
```

一会儿是字符串：

```json
{"taskId": "task-10001"}
```

可能导致映射冲突。

推荐：

```text
业务 ID 统一按 keyword 字符串处理
```

因为业务 ID 通常用于：

```text
精确匹配
聚合
关联
```

而不是数值计算。

### 3. 字段爆炸

如果把动态参数名直接作为字段：

```json
{
  "customField_user_10001": "x",
  "customField_user_10002": "y"
}
```

会导致大量动态字段，增加 Elasticsearch Mapping 压力。

更合理的设计是固定结构：

```json
{
  "customFields": {
    "key": "value"
  }
}
```

或者只记录经过筛选的固定字段。

---

## 二十八、常见日志系统故障

### 1. Filebeat 停止运行

表现：

```text
应用本地日志正常
Kibana 没有新日志
```

### 2. Filebeat 无权限读取

表现：

```text
指定日志文件存在
但采集器读取失败
```

### 3. Elasticsearch 磁盘水位过高

可能导致：

```text
索引写入受限
索引只读
日志无法继续写入
```

### 4. Mapping 冲突

例如同一字段不同类型：

```text
statusCode 有时是数字
有时是字符串
```

可能导致部分文档写入失败。

### 5. Kibana 查询时间错误

例如：

```text
故障发生在 10:00
当前查询最近 15 分钟
现在已经 11:00
```

自然看不到日志。

### 6. 时区不一致

```text
应用使用 UTC
Kibana 使用本地时区
数据库使用 Asia/Shanghai
```

排查时容易错位。

### 7. 多行异常解析失败

异常栈被拆成很多条无上下文日志。

### 8. 日志轮转配置错误

可能导致：

```text
重复采集
日志遗漏
磁盘未释放
```

### 9. 日志量突增

例如代码进入错误循环：

```text
每毫秒打印一次 ERROR
```

可能快速打满：

```text
应用磁盘
网络带宽
Elasticsearch 磁盘
```

需要监控：

```text
每秒日志量
索引写入速率
磁盘使用率
ERROR 增长率
```

---

## 二十九、业务关键节点应该如何记录日志

以 AI 异步任务为例。

### 1. 创建任务

```json
{
  "event": "TASK_CREATED",
  "taskId": "10001",
  "status": "PENDING"
}
```

### 2. 发送 MQ

```json
{
  "event": "MESSAGE_PUBLISHED",
  "taskId": "10001",
  "messageId": "msg-88",
  "topic": "ai-task"
}
```

### 3. Worker 开始消费

```json
{
  "event": "MESSAGE_CONSUMED",
  "taskId": "10001",
  "messageId": "msg-88",
  "consumerGroup": "ai-worker-group"
}
```

### 4. 任务开始

```json
{
  "event": "TASK_STARTED",
  "taskId": "10001",
  "status": "RUNNING",
  "workerId": "worker-2"
}
```

### 5. 阶段执行

```json
{
  "event": "TASK_STAGE_STARTED",
  "taskId": "10001",
  "stage": "AI_INFERENCE"
}
```

### 6. 第三方调用结果

```json
{
  "event": "THIRD_PARTY_RESPONSE",
  "taskId": "10001",
  "provider": "example-ai",
  "statusCode": 200,
  "durationMs": 58000
}
```

### 7. 任务完成

```json
{
  "event": "TASK_COMPLETED",
  "taskId": "10001",
  "status": "SUCCESS",
  "durationMs": 61000
}
```

### 8. 任务失败

```json
{
  "event": "TASK_FAILED",
  "taskId": "10001",
  "status": "FAILED",
  "errorCode": "AI_TIMEOUT",
  "retryable": true
}
```

日志应该帮助回答：

```text
发生了什么
作用于哪个业务对象
执行到哪一步
耗时多久
结果是什么
失败是否可重试
```

---

## 三十、支付回调场景如何排查

用户反馈：

```text
已经支付
订单仍然待支付
```

首先查询：

```text
orderNo:"202607270001"
```

理想日志链路：

```text
收到支付回调
↓
验签成功
↓
检查回调幂等
↓
查询订单
↓
校验支付金额
↓
更新支付单
↓
更新订单状态
↓
提交事务
↓
返回成功
```

可能发现：

```text
回调收到
验签成功
更新数据库失败
事务回滚
```

还需要检查：

```text
是否重复回调
是否金额不一致
是否订单状态不允许流转
是否乐观锁更新失败
是否事务范围正确
是否异常后错误返回成功
```

推荐字段：

```text
traceId
orderNo
outTradeNo
wechatTradeNo
callbackId
payStatus
orderStatus
amount
errorCode
```

敏感支付字段必须脱敏。

---

## 三十一、广告投放场景如何排查

广告发布失败，可以查询：

```text
campaignId:"888"
```

理想链路：

```text
读取账户授权
↓
获取 Token
↓
上传素材
↓
创建创意
↓
创建广告
↓
保存第三方 ID
```

如果第三方返回：

```text
HTTP 429
```

日志应包含：

```text
platform
accountId
campaignId
operation
thirdPartyCode
thirdPartyRequestId
retryCount
durationMs
```

这样可以判断：

```text
自身参数错误
授权失效
第三方限流
第三方服务异常
重复提交
```

不能只打印：

```text
广告创建失败
```

---

## 三十二、日志、指标和链路追踪的区别

ELK 主要解决日志问题，但日志不是全部可观测性。

### 1. Logs：日志

回答：

```text
具体发生了什么
```

适合：

```text
查看异常栈
查看业务字段
查看单次执行上下文
```

### 2. Metrics：指标

回答：

```text
系统整体是否异常
```

例如：

```text
QPS
错误率
P95 延迟
CPU
内存
Kafka Lag
线程池队列长度
```

指标适合发现：

```text
现在出了问题
```

### 3. Traces：分布式链路追踪

回答：

```text
一次请求在哪个服务、哪个 Span 上耗时
```

例如：

```text
API 20ms
用户服务 10ms
订单服务 50ms
第三方支付 1200ms
```

### 4. 三者组合

```text
Metrics 发现异常
↓
Trace 定位慢在哪个服务
↓
Logs 查看具体错误和业务上下文
```

只有日志，可能难以及时发现整体趋势。

只有指标，无法知道某个具体任务为什么失败。

只有 Trace，可能缺少详细业务状态和异常上下文。

---

## 三十三、从日志查询走向日志告警

只会在用户投诉后打开 Kibana，仍然属于被动排查。

更成熟的做法是对关键错误建立告警。

例如：

```text
5 分钟内 ERROR 数量超过阈值
AI_TIMEOUT 错误率超过 5%
支付回调失败连续出现
Kafka 消费失败数量上升
同一 TaskId 重试次数超过限制
日志采集停止
Elasticsearch 磁盘接近上限
```

但告警不能简单设置为：

```text
出现一条 ERROR 就报警
```

否则容易形成告警风暴。

告警设计需要考虑：

```text
错误率
绝对数量
持续时间
业务重要性
是否可自动恢复
是否需要人工处理
```

---

## 三十四、日志设计的常见错误

### 错误一：只打印技术信息，没有业务标识

```text
NullPointerException
```

缺少：

```text
taskId
orderNo
userId
operation
```

### 错误二：只打印业务描述，没有异常栈

```text
任务失败
```

无法定位代码。

### 错误三：日志字段命名不统一

```text
taskId
task_id
taskID
bizTaskId
```

导致查询困难。

### 错误四：状态值不统一

```text
SUCCESS
success
成功
done
completed
```

导致聚合失真。

### 错误五：把敏感数据完整打印

造成安全风险。

### 错误六：长任务没有阶段日志

只记录开始和结束，中间数分钟无任何信息。

### 错误七：异常被吞掉

任务停留在 RUNNING，但没有错误日志。

### 错误八：日志和状态更新不一致

先打印：

```text
任务成功
```

再更新数据库，但更新失败。

最终日志显示成功，数据库仍然 RUNNING。

更合理的是：

```text
业务结果生成
↓
状态更新成功
↓
打印最终成功日志
```

或者明确区分：

```text
业务执行成功
状态持久化失败
```

### 错误九：日志过多，失去信噪比

每个循环、每条数据都打印 INFO，真正问题被淹没。

### 错误十：日志没有生命周期

最终磁盘被占满，反过来导致系统故障。

---

## 三十五、今天思考题的完整回答

题目：

```text
TaskId=10001
一直 RUNNING

Kibana：
API 任务创建成功
Worker 开始执行
之后没有任何日志
```

### 问题一：Worker 真的是卡死了吗

不能直接判断。

可能性包括：

```text
任务仍在正常执行
任务本身耗时较长
中间没有阶段日志
卡在第三方接口
卡在数据库锁
线程池耗尽
连接池耗尽
线程死锁
JVM Full GC
容器 OOM 或重启
异常被吞掉
状态更新失败
日志采集链路故障
```

### 问题二：下一步查看哪些信息

合理顺序：

```text
数据库任务状态和时间
↓
Worker 容器状态
↓
任务实际运行时长和正常耗时
↓
JVM、线程池、连接池
↓
第三方服务、MySQL、对象存储
↓
Kafka 消费和 offset
↓
Filebeat、Elasticsearch、Kibana
```

Nginx 不是第一优先级，因为 Worker 已经开始执行。

### 问题三：没有 TraceId 增加哪些成本

```text
需要人工按时间拼接
并发请求容易混淆
跨服务日志无法直接关联
重试和补偿难以区分
容易误判其他任务日志
排查耗时显著增加
```

### 问题四：统一哪些字段

最核心字段：

```text
traceId
serviceName
taskId
requestId
status
durationMs
errorCode
podName
```

根据场景扩展：

```text
orderNo
campaignId
messageId
consumerGroup
retryCount
thirdPartyCode
exceptionStack
```

---

## 三十六、面试中怎么表达

可以这样表达：

> 在多服务和多实例环境中，我会通过集中日志体系完成线上排障。Spring Boot 服务输出统一的结构化日志，公共字段包括 serviceName、environment、traceId、requestId、podName，业务字段包括 taskId、orderNo、campaignId、messageId、status、durationMs 和 errorCode。日志由 Filebeat 采集，必要时通过 Logstash 清洗后写入 Elasticsearch，再通过 Kibana 按业务标识和 TraceId 查询。

> TraceId 用于串联一次调用链，TaskId 用于标识业务任务，两者不能混淆。同一个 TaskId 在自动重试、人工重试和补偿执行时可能对应多个 TraceId。HTTP 调用中会通过请求头透传 TraceId，异步 MQ 场景则需要将 TraceId 写入消息 Header 或消息体，消费者收到消息后恢复到 MDC，并在 finally 中清理，避免线程复用造成 TraceId 串号。

> 如果 AI 任务一直处于 RUNNING，Kibana 中只看到 Worker 开始执行，我不会直接判断 Worker 卡死。首先会确认任务正常耗时和数据库最后更新时间，然后检查 Worker 容器是否重启或 OOM，再查看 JVM 线程、线程池、连接池、第三方接口和数据库锁等待，之后检查 Kafka offset 和日志采集链路。因为已经确认 Worker 开始执行，所以此时 Nginx 通常不是第一排查对象。

> 除了可检索和可关联，我还会控制日志级别、异常栈、敏感数据和大对象内容，避免无效日志导致 Elasticsearch 存储和查询成本上升，并通过生命周期策略对热、温、冷数据和自动删除进行管理。

---

## 三十七、今天内容的实际价值判断

只会说：

```text
ELK 是 Elasticsearch、Logstash、Kibana
```

属于：

```text
概念记忆
```

会安装 ELK，并把日志导入 Kibana：

```text
基础部署能力
```

能够设计：

```text
统一日志格式
TraceId
TaskId
日志级别
错误码
异常栈
敏感信息脱敏
```

属于：

```text
初级到中级日志工程能力
```

能够根据：

```text
业务状态
日志证据
容器状态
JVM 状态
Kafka 状态
下游依赖
```

逐层缩小故障范围，属于：

```text
中级线上排障能力
```

当前 Java 后端求职阶段，合理学习边界是：

```text
理解 ELK 整体链路
会设计结构化日志
会使用 TraceId 和业务 ID
会使用 Kibana 查询
会排查日志缺失
会区分应用故障和采集故障
会控制日志量和敏感数据
会理解日志生命周期
```

暂时不需要优先深入：

```text
Elasticsearch 底层 Lucene 源码
复杂集群容量规划
超大规模日志平台架构
自研日志采集器
跨地域 Elasticsearch 集群治理
```

因为当前更高回报的方向仍然是：

```text
Java 业务能力
MySQL
Redis
MQ
事务
并发
幂等
状态机
接口设计
日志和监控
系统设计
真实故障排查
```

ELK 应该作为：

```text
后端服务可观测性和线上排障能力的一部分
```

而不是脱离 Java 主线单独投入过多时间。

---

## 三十八、今天的最终结论

第一个必须记住的结论：

```text
Kibana 没有日志
!=
代码一定没有执行
```

还可能是：

```text
没有打印
日志级别过滤
日志文件变化
Filebeat 故障
Elasticsearch 写入失败
查询时间或数据视图错误
```

第二个必须记住的结论：

```text
Worker 开始执行后没有日志
!=
Worker 一定卡死
```

也可能是：

```text
任务仍在正常运行
长任务缺少阶段日志
第三方调用阻塞
数据库锁等待
线程池或连接池耗尽
容器重启
状态更新失败
日志采集故障
```

第三个必须记住的区别：

```text
TaskId 标识业务任务
TraceId 标识一次调用链
RequestId 标识一次请求
MessageId 标识一条 MQ 消息
```

第四个必须记住的 TraceId 传递方式：

```text
HTTP 请求头透传
MQ Header 或消息体透传
消费者恢复 MDC
finally 清理 MDC
```

第五个必须记住的排障原则：

```text
根据已有证据缩小范围
而不是每次机械地从所有组件重新查起
```

如果已经看到：

```text
Worker 开始执行
```

就优先排查：

```text
Worker
JVM
线程池
连接池
下游依赖
Kafka
日志采集
```

而不是优先检查 Nginx。

第六个必须记住的日志字段：

```text
traceId
serviceName
taskId
status
durationMs
errorCode
podName
```

第七个必须记住的日志质量标准：

```text
可检索
可关联
字段统一
异常完整
敏感数据安全
日志量可控
生命周期明确
```

今天最核心的一句话是：

> 真正有价值的 ELK 能力不是会打开 Kibana，而是能够设计可关联的结构化日志，并根据 TaskId、TraceId、实例、状态和耗时快速判断问题发生在业务代码、Worker、JVM、MQ、下游依赖还是日志采集链路。

能力等级：

```text
能够说明 Filebeat、Elasticsearch、Kibana 的职责
→ 集中日志基础认知

能够使用业务字段和 TraceId 查询日志
→ 初级线上排障能力

能够设计结构化日志、MDC 传递、错误码和日志生命周期
→ 初级到中级日志工程能力

能够结合日志、容器、JVM、Kafka 和下游依赖定位复杂故障
→ 中级后端工程化与稳定性能力
```
