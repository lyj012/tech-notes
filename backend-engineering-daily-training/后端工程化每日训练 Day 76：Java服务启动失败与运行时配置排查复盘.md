# 后端工程化每日训练 Day 76：Java 服务启动失败与运行时配置排查复盘

## 一、今天学习什么

今天学习主题：

**Java 服务启动失败与运行时配置排查：从 JAR 到 Linux 进程的完整链路。**

今天重点解决：

1. 为什么 Java 进程存在，不代表服务正常？
2. Spring Boot 启动过程中数据库连接失败为什么可能导致服务启动失败？
3. systemd 显示 active，但是端口没有监听应该如何排查？
4. 手动启动 JAR 正常，但是 systemd 启动失败的原因？
5. 如何设计线上 Java 服务健康检查体系？

核心目标：

建立一个完整认知：

```text
代码
 ↓
Maven构建
 ↓
Spring Boot JAR
 ↓
JVM启动
 ↓
Spring容器初始化
 ↓
Bean加载
 ↓
数据库/MQ/缓存连接
 ↓
HTTP端口监听
 ↓
业务提供服务
```

线上排障不能只关注代码，而需要理解整个运行链路。

---

## 二、Java 服务启动完整流程

执行：

```bash
java -jar xxx.jar
```

实际过程：

```text
启动JVM
 ↓
读取配置文件
 ↓
创建Spring ApplicationContext
 ↓
扫描并初始化Bean
 ↓
创建DataSource
 ↓
连接数据库、缓存等依赖
 ↓
启动Tomcat
 ↓
监听HTTP端口
```

只有：

```text
Tomcat started
Started Application
```

并且端口正常监听，服务才真正可用。

---

## 三、为什么 Java 进程存在不能证明服务正常

命令：

```bash
ps -ef | grep java
```

只能证明：

```text
Java进程存在
```

不能证明：

```text
Spring启动成功
Bean初始化成功
数据库连接正常
HTTP接口可用
```

例如：

```text
JVM启动
 ↓
Spring初始化失败
 ↓
8082没有监听
```

此时：

```text
ps
可以看到Java进程

ss
没有HTTP端口
```

因此：

> 证据只能支持对应范围的结论。

---

## 四、数据库连接失败为什么可能导致启动失败

Spring Boot 中：

```text
DataSource
```

本身就是 Spring Bean。

启动过程：

```text
Spring启动
 ↓
创建DataSource
 ↓
连接数据库
 ↓
失败
 ↓
ApplicationContext启动失败
```

例如：

```yaml
spring:
 datasource:
   url: jdbc:mysql://xxx:3306/test
```

数据库地址错误：

```text
Connection refused
```

可能导致：

```text
Bean创建失败
 ↓
Spring启动失败
 ↓
服务退出
```

但是如果某些配置只有接口调用时才使用，则可能：

```text
服务启动成功
 ↓
访问接口
 ↓
业务失败
```

关键区别：

**配置什么时候被读取。**

---

## 五、systemd active 但是端口不存在如何排查

现象：

```bash
systemctl status xxx
```

显示：

```text
active(running)
```

但是：

```bash
ss -lntp | grep 8082
```

没有输出。

不能认为服务正常。

排查顺序：

### 1. 查看日志

```bash
journalctl -u 服务名 -n 200
```

关注：

```text
Application failed to start
BeanCreationException
Connection refused
Port already in use
```

### 2. 检查端口

```bash
ss -lntp
```

确认是否监听。

### 3. 验证接口

```bash
curl localhost:8082
```

确认业务链路。

---

## 六、为什么手动启动正常，systemd启动失败

原因：

两种启动环境不同。

主要区别：

### 1. 环境变量

手动启动可能继承：

```text
JAVA_HOME
PATH
```

systemd 不一定存在。

---

### 2. 工作目录

手动：

```bash
cd /opt/app
java -jar app.jar
```

systemd：

```ini
WorkingDirectory=/xxx
```

可能导致配置文件路径不同。

---

### 3. 权限

systemd可能使用：

```ini
User=app
```

导致：

```text
Permission denied
```

例如：

- 无法读取配置文件
- 无法写日志
- 无法访问目录

---

### 4. 启动参数

例如：

手动：

```bash
java -jar app.jar --spring.profiles.active=prod
```

systemd没有参数。

导致加载配置不同。

---

## 七、线上 Java 服务健康检查设计

完整健康检查不能只看进程。

应该覆盖多个层级。

---

## 1. JVM

监控：

- 堆内存使用率
- GC次数
- Full GC次数
- GC耗时
- 活跃线程数

目的：

发现：

```text
内存泄漏
GC风暴
线程耗尽
```

---

## 2. HTTP服务

监控：

- 端口监听
- 健康接口
- QPS
- 响应时间
- 错误率

例如：

```text
P95响应时间持续升高
500错误增加
```

需要告警。

---

## 3. 数据库

监控：

- 连接池大小
- 活跃连接数
- 空闲连接数
- 慢SQL
- 连接失败次数

避免：

```text
数据库连接耗尽
```

---

## 4. 缓存/MQ

缓存：

- 内存使用率
- 命中率
- 连接数

MQ：

- 消息堆积
- 消费速度

---

## 5. 日志

关注：

```text
ERROR
Exception
Timeout
Connection refused
```

重点看异常趋势，而不是单个错误。

---

## 八、生产排障思维

以后遇到：

```text
页面打不开
接口502
Java服务异常
```

不要直接猜代码。

按照证据链排查：

```text
进程
 ↓
日志
 ↓
Spring启动状态
 ↓
端口
 ↓
依赖组件
 ↓
业务接口
```

核心原则：

> 不要用低层证据推导高层结论。

例如：

```text
Java进程存在
```

只能说明：

```text
JVM存在
```

不能说明：

```text
服务正常
```

---

## 九、今日总结

今天建立了 Java 服务从代码到线上运行的完整模型：

```text
源码
 ↓
Maven
 ↓
JAR
 ↓
JVM
 ↓
Spring Boot
 ↓
Bean
 ↓
数据库/缓存/MQ
 ↓
HTTP端口
 ↓
业务请求
```

以后学习：

- Docker
- Kubernetes
- 微服务
- 服务治理

本质都是继续扩展这条运行链路。

目标：

从“会写代码”提升到：

**理解代码如何在线上运行，并根据证据定位问题。**
