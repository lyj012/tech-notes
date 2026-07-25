# 后端工程化每日训练 Day 18：Docker 容器化部署与 MySQL 连接故障排查

## 一、今天学习的知识点

今天学习的是：

**Docker 容器化部署与 MySQL 连接故障排查**

Docker 的基础作用是：

```text
把应用程序
+
运行时环境
+
系统依赖
+
启动方式

封装成可重复运行的镜像
```

对于 Java 后端开发者，今天真正需要理解的重点不是只记住：

```text
Image 是镜像
Container 是容器
Registry 是镜像仓库
```

而是理解：

```text
为什么 Java 后端需要学习 Docker
Docker 和运维工作的边界在哪里
为什么本地正常而容器中可能启动失败
容器中的 localhost 到底指向哪里
配置、网络、数据库和镜像如何共同影响服务启动
如何查看容器日志和实际环境变量
如何按照故障链路排查 MySQL 连接失败
如何使用明确镜像版本完成快速回滚
```

一句话理解：

> Docker 不是 Java 语言本身的一部分，也不是后端业务开发的核心，但它已经成为现代后端服务交付、运行和排障的重要基础工具。Java 后端不一定需要达到专业运维或平台工程师的深度，但必须能够把自己开发的服务容器化，并排查常见的启动、配置、网络和依赖问题。

---

## 二、Docker 解决的核心问题

传统 Java 项目的部署流程可能是：

```text
开发电脑编写代码
↓
本地执行 Maven 打包
↓
上传 Jar 到服务器
↓
检查服务器是否安装 JDK
↓
检查 JDK 版本
↓
安装其他运行依赖
↓
修改环境变量和配置文件
↓
执行启动命令
↓
人工检查日志
```

这种方式存在大量环境差异：

```text
开发环境使用 Java 21
生产环境使用 Java 17

开发电脑存在某个系统依赖
生产服务器没有安装

测试环境配置正确
生产环境手工修改配置时写错

不同服务器的目录、权限和启动命令不同

部署过程依赖个人记忆
```

最后容易出现经典问题：

```text
我的电脑能运行
但服务器不能运行
```

Docker 的目标是将以下内容标准化：

```text
应用制品
运行时版本
系统依赖
工作目录
启动命令
```

使同一个镜像可以在不同环境中以相近的方式运行：

```text
测试环境
预发环境
生产环境
云服务器
Kubernetes
```

但需要注意：

> Docker 只能减少环境差异，不能消除所有环境差异。

Docker 镜像通常包含用户态运行环境，不包含一套完全独立的操作系统内核。容器仍然会受到以下因素影响：

```text
宿主机操作系统和内核
CPU 架构
网络环境
文件挂载
权限
外部数据库
配置中心
Secret
资源限制
```

因此更准确的说法是：

```text
Docker 提高了运行环境的一致性和可重复性
```

而不是：

```text
使用 Docker 后，应用在任何机器上都绝对不会出问题
```

---

## 三、Java 后端为什么需要学习 Docker

一个常见疑问是：

```text
Docker 明明和部署有关
这不是运维应该学习的吗
为什么 Java 后端也要学习
```

这个疑问本身是合理的，因为 Docker 确实属于部署和基础设施领域。

但现代后端开发的工作边界已经不再是：

```text
开发只负责写代码
运维负责代码离开 IDE 之后的一切
```

更常见的责任划分是：

```text
后端开发
负责把服务开发成可构建、可配置、可部署、可运行、可排查的交付物

运维或 DevOps
负责服务器、网络、发布平台、容器平台、监控平台和生产环境治理
```

一个 Java 服务最终需要经历：

```text
编写业务代码
↓
执行测试
↓
Maven 打包
↓
构建 Docker 镜像
↓
推送镜像仓库
↓
部署容器
↓
注入生产配置
↓
连接 MySQL、Redis、MQ 等外部依赖
↓
接收真实流量
↓
通过日志和监控排查问题
```

如果开发人员只会在 IDEA 中启动项目，却不知道服务如何在容器中运行，就很难独立判断：

```text
是代码失败
还是配置错误
还是容器网络不通
还是数据库不可用
还是镜像版本不对
```

因此 Docker 对 Java 后端的意义可以类比 Git：

> Git 不是 Java 语言的一部分，但不会 Git 很难正常参与团队开发。Docker 也不是业务开发核心，但完全不懂 Docker，会缺少完整的服务交付和线上排障能力。

---

## 四、后端开发、运维和 DevOps 的职责边界

### 1. Java 后端开发通常负责什么

Java 后端的核心职责仍然是：

```text
业务逻辑
接口设计
数据库设计
缓存
消息队列
事务
并发
幂等
状态机
权限
日志
异常处理
系统设计
```

但在容器化环境中，通常还应掌握：

```text
编写基础 Dockerfile
理解服务需要的 JDK 版本
确认应用监听端口
知道配置如何注入
知道日志输出到哪里
能够构建和运行镜像
能够查看容器日志
能够检查环境变量
能够判断容器是否存活
能够排查常见网络和依赖问题
能够使用指定镜像版本回滚
```

### 2. 运维通常负责什么

运维更偏向：

```text
服务器管理
操作系统维护
网络与防火墙
磁盘和文件系统
数据库基础设施
Nginx 和负载均衡
备份与恢复
权限管理
生产环境稳定性
```

### 3. DevOps 或平台工程通常负责什么

DevOps 或平台工程更偏向：

```text
Jenkins 或 GitLab CI
镜像仓库
自动化发布平台
Kubernetes 集群
发布策略
监控告警平台
日志平台
基础设施自动化
```

### 4. 为什么现实中经常重叠

在大型成熟公司中，职责可能分得比较细：

```text
后端维护代码、Dockerfile 和应用配置
平台团队提供构建与发布系统
SRE 或运维管理生产基础设施
```

在中小公司中，一个后端开发者可能需要同时处理：

```text
Java 业务开发
Linux
Docker
Jenkins
Nginx
日志排查
简单发布
```

因此，当前 Java 后端求职阶段不需要把自己训练成专业运维，但必须具备基本的容器化交付能力。

合理学习边界是：

```text
能够独立把 Spring Boot 项目容器化
能够理解 Jenkins 到 Docker 的发布链路
能够在服务器上启动和检查容器
能够排查常见的启动、配置和网络问题
能够完成基础版本回滚
```

暂时不需要优先深入：

```text
Docker 源码
Namespace 和 Cgroups 的完整内核实现
OverlayFS 深度原理
Harbor 集群高可用
Kubernetes 集群搭建
CNI 和 CSI 源码
大规模容器调度算法
```

---

## 五、Docker 的三个核心概念

## 1. Image：镜像

镜像可以理解为：

```text
应用运行模板
```

对于一个 Java 服务，镜像中通常包含：

```text
JRE
Jar
必要的系统依赖
工作目录
默认启动命令
```

例如：

```text
ai-api:1.2.0
```

镜像本身是只读模板，不是正在运行的进程。

需要注意：

```text
生产数据库密码
Redis 密码
第三方密钥
环境专属地址
```

这些敏感或环境相关配置不应该直接固化到镜像中。

### 2. Container：容器

容器是镜像运行后的实例。

```text
Image
ai-api:1.2.0
↓
运行
↓
Container
```

同一个镜像可以启动多个容器：

```text
ai-api-1
ai-api-2
ai-api-3
```

它们运行相同版本的应用，但可以拥有不同的：

```text
容器名称
IP 地址
端口映射
环境变量
资源限制
```

### 3. Registry：镜像仓库

镜像仓库用于保存和分发镜像，类似于 Git 仓库保存和分发代码。

常见镜像仓库包括：

```text
Docker Hub
Harbor
阿里云 ACR
云厂商私有镜像仓库
```

典型流程：

```text
Jenkins 构建镜像
↓
docker push
↓
镜像仓库
↓
生产服务器 docker pull
↓
启动指定版本容器
```

---

## 六、Java 服务的容器化发布链路

结合 Jenkins，完整流程通常是：

```text
Git Push
↓
Jenkins 拉取代码
↓
Maven 编译和测试
↓
生成 Jar
↓
Docker Build
↓
生成带版本号的镜像
↓
推送镜像仓库
↓
服务器拉取指定版本
↓
启动新容器
↓
检查容器状态
↓
调用健康检查接口
↓
确认服务可用
```

例如广告投放平台可能包含：

```text
Admin
API
Worker
Scheduler
```

每个服务可以分别构建镜像：

```text
ad-admin:1.3.0
ad-api:1.3.0
ad-worker:1.2.8
ad-scheduler:1.1.5
```

这样升级 Worker 时，不需要重新发布 Admin 和 API。

AI 异步任务平台也可以拆分为：

```text
ai-api
ai-worker
ai-scheduler
```

其中 Worker 负责：

```text
OCR
PDF 转换
AI 模型调用
任务执行
```

镜像化后，不同环境使用同一个应用制品，只在启动时注入不同配置。

---

## 七、基础 Dockerfile 的含义

一个最小 Java Dockerfile：

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY target/app.jar app.jar

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 1. FROM

```dockerfile
FROM eclipse-temurin:17-jre
```

表示使用包含 Java 17 JRE 的基础镜像。

需要确认：

```text
项目编译目标版本
基础镜像 JDK 或 JRE 版本
```

如果项目使用 Java 21 编译，却使用 Java 17 运行，可能出现：

```text
UnsupportedClassVersionError
```

### 2. WORKDIR

```dockerfile
WORKDIR /app
```

设置容器内部的工作目录。

后续相对路径都以 `/app` 为基础。

### 3. COPY

```dockerfile
COPY target/app.jar app.jar
```

将构建出的 Jar 复制到镜像中。

常见错误包括：

```text
Jar 文件名写错
多模块项目复制了错误模块
Maven 尚未打包就执行 Docker Build
复制了旧 Jar
```

### 4. ENTRYPOINT

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

定义容器启动时执行的命令。

数组形式通常比字符串形式更适合正确传递信号：

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

而不是：

```dockerfile
ENTRYPOINT java -jar app.jar
```

### 5. 更接近生产的基础版本

可以进一步增加非 root 用户和 JVM 参数注入能力：

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app

RUN useradd --system --uid 10001 appuser

COPY --chown=appuser:appuser target/app.jar app.jar

USER appuser

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

但使用 `sh -c` 时还要考虑信号传递问题。更成熟的方案可以通过启动脚本或直接传递 JVM 参数实现。

当前阶段最重要的是先掌握：

```text
基础镜像
工作目录
应用制品
启动命令
```

---

## 八、配置为什么必须和镜像分离

错误做法：

```text
把生产数据库地址
生产数据库密码
Redis 密码
第三方密钥

直接写进 application.yml
并一起打入镜像
```

风险包括：

```text
镜像泄露导致密钥泄露
测试环境误连生产数据库
不同环境必须重新构建镜像
密码变更后必须重新打包
无法独立回滚配置和代码
```

更合理的方式是：

```text
镜像保存应用和默认启动逻辑
环境配置在启动时外部注入
```

常见配置来源：

```text
环境变量
外部配置文件挂载
Apollo
Nacos
Kubernetes ConfigMap
Kubernetes Secret
云平台 Secret 服务
```

Spring Boot 可以使用环境变量：

```yaml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

启动容器时注入：

```bash
docker run -d \
  --name ai-api \
  -e DB_HOST=mysql \
  -e DB_PORT=3306 \
  -e DB_NAME=ai_platform \
  -e DB_USERNAME=app_user \
  -e DB_PASSWORD='******' \
  ai-api:1.2.0
```

实际生产中不应把密码直接写进公开脚本、命令历史或仓库，应使用受控的 Secret 管理方式。

还需要注意变量名称必须一致。

例如应用读取：

```text
DB_PASSWORD
```

但部署脚本注入：

```text
MYSQL_PASSWORD
```

应用就无法读取到正确值。

---

## 九、为什么不能长期使用 latest

错误做法：

```text
ai-api:latest
```

问题是：

```text
latest 今天可能指向版本 A
明天可能指向版本 B
旧标签可能被覆盖
发布记录无法确定真实内容
出现故障后不知道上一版本是什么
```

更合理的镜像标签包括：

```text
ai-api:1.2.5
ai-api:20260725-001
ai-api:git-fb1e628
```

成熟发布记录至少应该能够关联：

```text
Git Commit ID
Jenkins Build Number
镜像版本
部署环境
配置版本
数据库迁移版本
发布时间
操作人
```

镜像版本的核心目标是：

```text
可定位
可追踪
不可随意覆盖
可回滚
```

---

## 十、容器中的 localhost 是什么

这是 Docker 网络中最容易出现的误区之一。

本地运行 Spring Boot 时：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo
```

这里的 `localhost` 指开发电脑自身。

当 Spring Boot 运行在 Docker 容器中时：

```text
localhost
=
当前 Java 容器自身
```

它不代表：

```text
宿主机
另一个 MySQL 容器
公司数据库服务器
```

如果 MySQL 也运行在 Docker 中，并且两个容器位于同一个 Docker 网络，通常使用：

```text
MySQL 容器名
或 Docker Compose 服务名
```

例如：

```yaml
services:
  mysql:
    image: mysql:8.4

  ai-api:
    image: ai-api:1.2.0
    environment:
      DB_HOST: mysql
```

此时 Java 服务使用：

```text
jdbc:mysql://mysql:3306/ai_platform
```

如果需要访问宿主机上的 MySQL，Windows 和 macOS Docker Desktop 常见地址是：

```text
host.docker.internal
```

Linux 环境的行为不同，可能需要额外配置：

```bash
--add-host=host.docker.internal:host-gateway
```

生产环境中更常见的是使用明确的数据库域名、内网地址或服务发现名称，而不是依赖 `localhost`。

---

## 十一、容器日志和数据为什么不能依赖可写层

容器拥有自己的可写层，但容器被删除后，这一层通常也会消失。

如果应用日志只写到：

```text
/app/logs
```

而没有挂载外部目录或接入日志平台，删除容器后日志可能无法保留。

如果数据库数据直接保存在数据库容器的可写层中，删除容器后也存在数据丢失风险。

正确思路是：

```text
应用容器尽量无状态
日志输出到标准输出或外部日志系统
持久化数据使用 Volume 或外部存储
```

例如查看标准输出日志：

```bash
docker logs ai-api
```

持续跟踪：

```bash
docker logs -f ai-api
```

挂载目录示例：

```bash
docker run -d \
  --name ai-api \
  -v /opt/ai-api/logs:/app/logs \
  ai-api:1.2.0
```

但生产日志管理不能只依赖本地目录，还应考虑：

```text
日志轮转
磁盘容量
集中采集
检索
告警
traceId
保留周期
```

---

## 十二、Docker 不等于隔离一切

Docker 属于操作系统级隔离，不是完整虚拟机。

多个容器仍然共享宿主机资源：

```text
CPU
内存
磁盘 IO
网络
操作系统内核
```

如果不设置资源限制，一个异常容器可能：

```text
占满内存
持续消耗 CPU
写满磁盘
创建大量连接
影响同一宿主机上的其他服务
```

基础限制示例：

```bash
docker run -d \
  --name ai-worker \
  --memory=2g \
  --cpus=2 \
  ai-worker:1.0.0
```

但资源限制不能随意设置。

例如 Java 应用的：

```text
JVM Heap
Direct Memory
Metaspace
线程栈
本地库内存
```

共同占用容器内存。

如果只根据 `-Xmx` 判断内存需求，可能仍然触发容器 OOM。

---

## 十三、真实故障：本地正常，Docker 容器连接 MySQL 失败

故障现象：

```text
本地运行正常
↓
构建 Docker 镜像
↓
容器启动失败
↓
日志显示 Could not connect to MySQL
```

第一个问题是：

```text
这是代码问题还是部署问题
```

不能仅凭一行日志绝对确定，但在以下前提下：

```text
同一版本代码本地正常
容器环境启动失败
```

优先级通常是：

```text
部署配置问题
>
容器网络问题
>
数据库状态或权限问题
>
镜像构建问题
>
代码问题
```

原因是本地已经证明以下内容大概率正常：

```text
数据源配置结构
JDBC 驱动基本可用
Spring Boot 能够创建数据源
项目能够连接某个 MySQL
```

但这不能证明容器实际使用的：

```text
数据库地址
端口
数据库名
用户名
密码
网络
权限
```

与本地完全相同。

---

## 十四、代码问题和配置问题必须区分

本次回答中最需要纠正的判断是：

```text
YAML 中密码错误
或数据库名错误
所以属于代码问题
```

更准确的分类是：

```text
业务代码问题
配置问题
部署问题
基础设施问题
```

### 1. 业务代码问题

例如：

```text
数据源初始化代码被错误修改
错误移除了 JDBC 驱动
动态数据源路由逻辑存在 Bug
程序拼接了错误 JDBC URL
代码读取了错误配置键
```

### 2. 配置问题

例如：

```text
数据库密码错误
数据库名错误
数据库 Host 错误
端口错误
Spring Profile 错误
环境变量没有注入
配置中心读取了错误环境
```

即使配置写在 `application.yml` 中，它仍然属于配置，而不是业务代码。

### 3. 部署问题

例如：

```text
部署脚本没有传入环境变量
挂载了错误配置文件
启动了错误镜像
端口映射错误
容器不在正确网络
```

### 4. 基础设施问题

例如：

```text
MySQL 没有启动
网络不通
防火墙阻断
DNS 解析失败
MySQL 连接数耗尽
磁盘已满
```

因此：

> 本地正常、容器失败时，应先比较环境差异，而不是先怀疑业务代码。

---

## 十五、为什么 SQL 版本不是第一排查项

本次回答中提到：

```text
检查 SQL 版本是否正确
```

但日志是：

```text
Could not connect to MySQL
```

连接失败通常发生在 SQL 执行之前。

典型链路是：

```text
读取数据库配置
↓
解析数据库 Host
↓
建立 TCP 连接
↓
完成 MySQL 认证
↓
选择数据库
↓
执行 SQL
```

如果连接都没有建立，业务 SQL 通常尚未执行。

因此此时优先检查：

```text
Host
Port
网络
用户名
密码
权限
数据库是否存在
MySQL 是否运行
```

只有当日志明确出现以下内容时，才优先检查 SQL：

```text
SQLSyntaxErrorException
Table doesn't exist
Unknown column
Data truncation
Deadlock found
```

---

## 十六、第一步：读取完整异常，而不是只看表面日志

查看容器日志：

```bash
docker logs ai-api
```

查看最后 200 行：

```bash
docker logs --tail 200 ai-api
```

持续跟踪：

```bash
docker logs -f ai-api
```

不能只看到：

```text
Could not connect to MySQL
```

还要继续向下查找最底层异常。

常见错误及含义：

| 异常信息 | 常见原因 |
|---|---|
| `Access denied for user` | 用户名、密码或 MySQL 授权错误 |
| `Unknown database` | 数据库名不存在或环境配错 |
| `Connection refused` | 地址可达，但目标端口没有服务监听，或端口配置错误 |
| `Unknown host` | 数据库域名、容器名或 DNS 解析失败 |
| `Connection timed out` | 网络、防火墙、安全组或路由问题 |
| `Communications link failure` | 泛化连接失败，需要继续查看根因 |
| `Too many connections` | MySQL 连接数已耗尽 |
| `Public Key Retrieval is not allowed` | MySQL 认证插件和 JDBC 参数问题 |

排障的关键不是看到异常名称，而是找到：

```text
最底层 cause
```

---

## 十七、第二步：确认容器实际拿到的配置

不能只查看代码仓库里的 `application.yml`。

原因是容器运行时配置可能来自：

```text
环境变量
外部挂载文件
Spring Profile
配置中心
Secret
启动参数
```

查看容器环境变量：

```bash
docker exec ai-api env
```

筛选数据库变量：

```bash
docker exec ai-api env | grep -i DB_
```

或：

```bash
docker exec ai-api env | grep -i datasource
```

查看容器完整配置：

```bash
docker inspect ai-api
```

重点确认：

```text
DB_HOST
DB_PORT
DB_NAME
DB_USERNAME
Spring Profile
配置中心地址
配置文件挂载位置
镜像版本
启动命令
```

需要注意：

> `docker inspect` 可能直接显示环境变量中的密码和密钥，不应把未经处理的完整输出发布到公开仓库、群聊或工单中。

如果容器已经退出，不能使用：

```bash
docker exec
```

但仍然可以使用：

```bash
docker inspect ai-api
docker logs ai-api
```

还可以用同一个镜像临时启动 shell：

```bash
docker run --rm -it \
  --entrypoint sh \
  ai-api:1.2.0
```

但部分精简镜像可能没有 shell。

---

## 十八、第三步：检查地址、DNS 和端口连通性

进入正在运行的容器：

```bash
docker exec -it ai-api sh
```

检查数据库域名是否能够解析：

```bash
getent hosts mysql
```

如果镜像包含相关工具，可以测试端口：

```bash
nc -zv mysql 3306
```

或：

```bash
telnet mysql 3306
```

需要建立分层判断：

```text
数据库名称能否解析
↓
能否到达数据库 IP
↓
3306 端口是否可访问
↓
MySQL 是否接受认证
↓
账号是否有数据库权限
```

不同结果代表不同故障层：

```text
Unknown host
→ DNS、服务名或容器网络问题

Connection timed out
→ 网络、路由、防火墙或安全组问题

Connection refused
→ 目标端口没有监听，或地址、端口错误

Access denied
→ 网络已经通，重点检查账号、密码和授权
```

---

## 十九、第四步：检查 Docker 网络

查看网络列表：

```bash
docker network ls
```

查看容器所属网络：

```bash
docker inspect ai-api
```

查看某个网络中的容器：

```bash
docker network inspect app-network
```

如果 Java 容器与 MySQL 容器不在同一个用户自定义网络中，Java 容器可能无法通过 MySQL 容器名访问数据库。

Docker Compose 通常会为项目创建默认网络：

```yaml
services:
  mysql:
    image: mysql:8.4

  ai-api:
    image: ai-api:1.2.0
    depends_on:
      - mysql
```

在这个网络中，`ai-api` 可以使用服务名：

```text
mysql
```

但需要注意：

> `depends_on` 通常只能控制启动顺序，不能天然保证 MySQL 已经完成初始化并可以接受连接。

更可靠的方式是：

```text
MySQL 健康检查
+
应用连接重试
+
合理启动超时
```

---

## 二十、第五步：检查 MySQL 自身状态

如果 MySQL 也运行在 Docker 中：

```bash
docker ps -a
```

查看 MySQL 日志：

```bash
docker logs mysql
```

需要确认：

```text
MySQL 容器是否运行
3306 是否监听
初始化是否完成
数据库是否存在
用户是否存在
用户是否允许对应来源登录
密码是否发生变化
连接数是否耗尽
磁盘是否已满
```

如果 MySQL 在远程服务器或云数据库中，还需要检查：

```text
安全组
白名单
内网地址
VPC 网络
账号权限
数据库实例状态
```

不能因为 Java 日志中出现 MySQL，就直接得出：

```text
MySQL 本身坏了
```

更准确的理解是：

```text
Java 容器到 MySQL 的连接链路中某一环失败
```

完整链路包括：

```text
Java 应用
→ 配置读取
→ DNS
→ Docker 网络
→ 宿主机网络
→ 防火墙或安全组
→ MySQL 端口
→ MySQL 认证
→ 数据库权限
```

---

## 二十一、第六步：最后检查镜像和代码差异

当配置、网络和数据库都没有发现问题时，再进一步检查：

```text
是否启动了错误镜像
镜像内是否复制了错误 Jar
是否运行了旧版本 Jar
JDK 版本是否变化
JDBC 驱动是否被排除
Spring Profile 是否变化
数据源初始化逻辑是否修改
Dockerfile 是否修改
```

查看当前容器镜像：

```bash
docker inspect ai-api
```

查看本地镜像：

```bash
docker images
```

更成熟的镜像可以写入构建元数据：

```dockerfile
ARG GIT_COMMIT
LABEL org.opencontainers.image.revision=$GIT_COMMIT
```

这样可以将镜像与 Git Commit 对应起来。

---

## 二十二、推荐的完整排查顺序

当出现：

```text
本地正常
Docker 容器启动失败
Could not connect to MySQL
```

推荐顺序是：

```text
1. 查看完整容器日志和最底层异常
↓
2. 确认当前容器和镜像版本
↓
3. 检查容器实际环境变量、Profile 和配置来源
↓
4. 检查数据库 Host 是否错误使用 localhost
↓
5. 在容器内测试 DNS 和 3306 端口
↓
6. 检查 Docker 网络和端口配置
↓
7. 检查 MySQL 服务、账号、密码、权限和连接数
↓
8. 对比昨天和今天发生的配置、镜像、数据库和基础设施变更
↓
9. 最后检查镜像构建和代码改动
```

这个顺序不是绝对固定的。

真正专业的原则是：

> 先根据异常类型缩小故障层，再根据最近变更确定最高优先级，而不是机械地把所有命令执行一遍。

---

## 二十三、昨天能启动，今天不能启动，先检查什么

本次回答中先后考虑了：

```text
数据库
配置
镜像
```

更好的判断方式不是固定使用：

```text
配置 → 数据库 → 镜像
```

而是先问：

```text
昨天到今天发生了什么变化
```

### 场景一：今天刚发布新镜像

优先检查：

```text
镜像版本
Dockerfile
Jar 内容
启动参数
发布配置
```

### 场景二：今天没有发布，但数据库刚修改密码

优先检查：

```text
Secret
数据库账号
密码同步
MySQL 权限
```

### 场景三：服务器今天重启

优先检查：

```text
MySQL 是否自动启动
Docker 网络是否恢复
容器是否自动启动
挂载目录是否存在
环境变量是否丢失
```

### 场景四：网络或安全组刚调整

优先检查：

```text
端口
路由
安全组
白名单
DNS
```

因此线上排障的核心原则是：

```text
故障时间
+
变更时间
+
异常类型
```

三者结合判断。

---

## 二十四、如何使用明确镜像版本快速回滚

假设当前故障版本是：

```text
registry.example.com/ai-api:20260725-001
```

昨天稳定版本是：

```text
registry.example.com/ai-api:20260724-001
```

停止并删除当前容器：

```bash
docker stop ai-api
docker rm ai-api
```

拉取旧版本：

```bash
docker pull registry.example.com/ai-api:20260724-001
```

使用原有配置重新启动：

```bash
docker run -d \
  --name ai-api \
  --restart=always \
  -p 8080:8080 \
  --env-file /opt/ai-api/.env \
  registry.example.com/ai-api:20260724-001
```

如果使用 Docker Compose：

```yaml
services:
  ai-api:
    image: registry.example.com/ai-api:20260724-001
```

然后执行：

```bash
docker compose up -d
```

回滚后不能立即宣布恢复，还需要检查：

```text
容器是否持续存活
启动日志是否正常
健康接口是否正常
数据库是否可以连接
关键接口是否可用
```

### 回滚的前提

镜像回滚并不天然安全，需要满足：

```text
旧镜像没有被覆盖
旧镜像能够被准确定位
当前配置与旧代码兼容
数据库结构能够向后兼容
消息格式能够兼容
缓存数据结构没有破坏性变化
```

如果新版本已经执行不可逆数据库迁移：

```text
删除字段
修改字段语义
修改枚举值
写入旧版本无法识别的数据
```

仅回滚镜像可能无法恢复系统。

因此成熟发布需要同时管理：

```text
应用版本
配置版本
数据库迁移版本
```

---

## 二十五、基础 Docker 命令清单

Java 后端当前至少应能够理解和使用：

### 构建镜像

```bash
docker build -t ai-api:1.2.0 .
```

### 查看镜像

```bash
docker images
```

### 启动容器

```bash
docker run -d --name ai-api -p 8080:8080 ai-api:1.2.0
```

### 查看运行中的容器

```bash
docker ps
```

### 查看所有容器

```bash
docker ps -a
```

### 查看日志

```bash
docker logs ai-api
```

### 进入容器

```bash
docker exec -it ai-api sh
```

### 查看容器详细信息

```bash
docker inspect ai-api
```

### 停止容器

```bash
docker stop ai-api
```

### 删除容器

```bash
docker rm ai-api
```

### 拉取镜像

```bash
docker pull registry.example.com/ai-api:1.2.0
```

### 推送镜像

```bash
docker push registry.example.com/ai-api:1.2.0
```

掌握命令不是为了背诵，而是为了能够回答：

```text
服务现在运行的是哪个镜像
容器是否还活着
为什么退出
拿到了什么配置
网络是否正常
如何恢复旧版本
```

---

## 二十六、本次思考题回答复盘

### 问题一：是代码问题还是部署问题

原回答认为：

```text
大概是代码问题
可能是 YAML 密码错误或数据库名错误
```

其中具体原因方向有价值，但分类不准确。

正确结论是：

```text
密码错误或数据库名错误
→ 配置问题

在本地正常、容器失败的前提下
→ 优先排查部署环境、配置和网络
```

代码问题仍然存在可能，但优先级通常更低。

### 问题二：按照什么顺序排查

原回答首先想到：

```text
看日志
```

这是正确起点。

但后续只想到：

```text
数据库名
SQL 版本
```

范围过窄。

应扩展为：

```text
完整异常
→ 实际配置
→ localhost
→ DNS 和端口
→ Docker 网络
→ MySQL 状态和权限
→ 镜像与代码差异
```

SQL 版本不是连接失败的第一排查项。

### 问题三：如何确认容器环境变量

原回答是不知道。

需要记住两个命令：

```bash
docker exec 容器名 env
docker inspect 容器名
```

其中：

```text
docker exec
只能用于正在运行的容器

docker inspect
即使容器已经退出也可以查看
```

### 问题四：昨天能启动，今天不能启动

原回答最终排序是：

```text
先配置
再数据库
最后镜像
```

这个顺序可以作为基础思路，但不应该机械固定。

更成熟的方法是：

```text
先检查最近变更
```

谁在故障前发生变化，谁优先级最高。

### 问题五：如何快速回滚

原回答是不知道。

需要形成以下记忆：

```text
找到昨天稳定镜像标签
↓
停止当前容器
↓
拉取旧镜像
↓
使用原配置启动旧版本
↓
检查容器状态和健康接口
```

回滚能够快速执行的前提是：

```text
没有长期只使用 latest
镜像仓库保留历史版本
配置和数据库能够向后兼容
```

---

## 二十七、今天暴露出的认知缺口

今天已经能够想到：

```text
查看日志
检查数据库名
检查密码
检查配置
检查数据库
检查镜像
```

说明已经具备基础故障意识。

但仍然缺少以下关键连接：

```text
密码错误属于配置问题，不是业务代码问题
本地正常、容器失败应优先比较环境差异
容器中的 localhost 指向容器自身
MySQL 连接失败可能发生在 Docker 网络层
需要读取容器实际环境变量，而不是只看源码 YAML
排障顺序应结合最底层异常和最近变更
镜像版本管理是快速回滚的基础
数据库变更可能使镜像回滚失效
```

当前状态可以概括为：

```text
能够想到表面原因
但尚未形成完整的容器化故障链路
```

今天真正应该建立的是：

```text
应用
→ 配置
→ 容器
→ DNS
→ 网络
→ 数据库端口
→ 认证
→ 权限
```

这一整条链路的排查意识。

---

## 二十八、面试表达

可以这样回答：

> 我理解 Docker 对 Java 后端来说不是业务开发核心，但属于服务交付和故障排查的基础能力。我会将 Spring Boot 服务构建成带明确版本号的 Docker 镜像，通过 Jenkins 完成 Maven 构建、镜像构建和推送，部署环境只拉取指定版本镜像并在启动时注入配置。这样可以减少环境差异，也便于追踪和回滚。
>
> 配置方面，我不会把生产数据库密码等敏感信息直接固化到镜像中，而是通过环境变量、配置中心或 Secret 注入。镜像负责保存应用制品和运行时，配置负责区分测试、预发和生产环境。
>
> 如果出现本地正常但 Docker 容器连接 MySQL 失败，我会优先判断为环境、配置或网络问题，而不是直接判断为业务代码问题。首先查看完整容器日志，根据 `Access denied`、`Unknown host`、`Connection refused` 或超时等底层异常判断故障层；然后使用 `docker inspect` 和 `docker exec ... env` 确认实际配置，重点检查容器中是否错误使用了 `localhost`，再测试 DNS、3306 端口、Docker 网络以及 MySQL 服务和账号权限。
>
> 如果故障发生在新版本发布之后，我会结合最近变更优先检查镜像和配置。回滚时使用镜像仓库中昨天的明确版本重新启动，并检查容器状态和健康接口。为了保证回滚可靠，还要保留 Git Commit、Jenkins Build Number、镜像版本、配置版本和数据库迁移版本，避免只使用 `latest`。

---

## 二十九、今天内容的实际价值判断

Docker 基础概念本身并不难：

```text
Image
Container
Registry
Dockerfile
```

仅会解释这些概念，最多属于基础认知。

真正有价值的是能够把 Docker 与以下内容连接起来：

```text
Java 服务交付
配置管理
Jenkins 发布
容器网络
日志排查
外部依赖
版本管理
故障回滚
```

对于当前 Java 后端求职阶段，合理投入是：

```text
学会实际使用 Docker
学会容器化 Spring Boot
学会排查常见问题
```

而不是投入大量时间成为 Docker 内核专家。

当前优先级应该是：

```text
第一层：会构建、启动、查看和停止容器
第二层：理解配置、端口、网络、Volume 和版本
第三层：能够排查容器启动失败和依赖连接失败
第四层：理解 Jenkins 构建、健康检查和回滚链路
```

达到这一层后，应继续把主要时间投入到更核心的 Java 后端能力：

```text
MySQL
Redis
MQ
事务
并发
接口设计
权限
幂等
状态机
分布式系统
系统设计
```

---

## 三十、今天的最终结论

今天最需要避免的错误判断是：

```text
YAML 密码错误
=
代码问题
```

更准确的分类是：

```text
YAML、环境变量、Profile、数据库地址错误
=
配置问题
```

第二个需要避免的错误判断是：

```text
日志出现 MySQL
=
MySQL 本身一定坏了
```

更准确的理解是：

```text
Java 容器到 MySQL 的连接链路中某一环失败
```

第三个需要记住的结论是：

```text
本地正常
+
容器失败

优先比较环境差异
```

第四个需要记住的结论是：

```text
昨天正常
+
今天失败

优先检查故障发生前的最近变更
```

第五个需要记住的结论是：

```text
明确镜像版本
+
保留历史镜像
+
配置和数据库向后兼容

才可能实现可靠快速回滚
```

今天最核心的一句话是：

> Java 后端不需要把 Docker 学到专业运维的深度，但必须理解自己的服务如何被构建、配置、运行和排查。容器化故障不能只盯着 Java 代码或 MySQL，而要沿着应用、配置、容器、网络、认证和数据库的完整链路定位问题。

能力等级：

```text
能够解释 Image、Container、Registry
→ Docker 基础认知

能够编写基础 Dockerfile 并启动 Spring Boot 容器
→ 初级工程实践

能够排查配置、localhost、容器网络和 MySQL 连接问题
→ 初级到中级后端工程能力

能够设计不可变镜像、健康检查、资源限制和可靠回滚
→ 中级容器化交付能力
```
