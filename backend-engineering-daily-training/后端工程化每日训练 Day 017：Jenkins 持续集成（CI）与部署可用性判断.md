# 后端工程化每日训练 Day 17：Jenkins 持续集成（CI）与部署可用性判断

## 一、今天学习的知识点

今天学习的是：

**Jenkins 持续集成（CI）与部署可用性判断**

Jenkins 的基础作用是把以下流程自动化：

```text
拉取代码
↓
编译
↓
测试
↓
打包
↓
构建镜像
↓
部署
```

但对于已经在真实项目中使用过 Jenkins 的开发者，今天真正需要理解的重点不是：

```text
Jenkins 是什么
如何点击构建按钮
Pipeline 有哪些 Stage
```

而是：

```text
Jenkins 页面上的成功到底代表什么
为什么构建成功不等于服务一定可用
为什么以前点击部署时，失败会自动阻止发布
502 出现后应该由开发还是运维排查
开发人员至少要完成哪些基础判断
如何让 Pipeline 验证服务真正启动成功
如何为快速回滚保留可靠的版本信息
```

一句话理解：

> Jenkins 负责执行预先配置好的构建和部署流程。它能够检查到哪一层，取决于 Pipeline 和部署脚本中写了哪些判断。Jenkins 显示成功，表示配置过的检查已经通过，并不天然等于所有业务功能、网关链路和外部依赖都绝对正常。

---

## 二、Jenkins 解决的核心问题

传统 Java 项目可能采用手工发布：

```text
开发者本地执行 mvn package
↓
手动找到 Jar
↓
上传服务器
↓
SSH 登录服务器
↓
停止旧进程
↓
启动新进程
↓
人工检查日志
```

这种方式存在以下问题：

```text
每个人执行步骤不同
可能上传错误版本
可能漏执行某一步
生产配置容易使用错误
无法快速确认发布人和发布时间
失败后难以恢复
多模块发布容易混乱
```

Jenkins 的主要价值是：

```text
流程标准化
+
操作自动化
+
执行结果可追踪
+
失败及时终止
+
版本信息可回溯
```

它将发布流程从：

```text
依赖个人记忆和手工操作
```

变成：

```text
依赖统一配置的 Pipeline 和部署脚本
```

---

## 三、真实工作中使用 Jenkins 的两种角色

在公司中，开发人员使用 Jenkins 时，可能处于两种不同角色。

### 1. Jenkins 使用者

常见操作是：

```text
进入 Jenkins 页面
↓
选择项目或模块
↓
选择环境或分支
↓
点击构建或部署
↓
查看成功或失败
```

例如广告投放平台包含：

```text
Admin
API
Worker
Scheduler
```

开发人员选择需要更新的模块，点击部署即可。

这种情况下，开发人员主要负责：

```text
选择正确模块
选择正确分支
确认构建结果
查看 Console Output
验证自己负责的功能
```

### 2. Pipeline 设计和维护者

这类人员需要负责：

```text
编写 Jenkinsfile
编写 Shell 部署脚本
管理 Jenkins Credentials
设计构建和部署阶段
配置服务器和镜像仓库
增加健康检查
处理失败通知
设计版本回滚
维护多环境发布流程
```

不同公司中，这些工作可能由以下人员负责：

```text
运维
DevOps 工程师
后端负责人
基础架构团队
开发与运维共同维护
```

因此：

> 会在 Jenkins 页面点击部署，说明具备 Jenkins 的实际使用经验；能够设计 Pipeline、分析退出码、验证健康状态并完成回滚，才属于更深入的 CI/CD 工程能力。

---

## 四、为什么以前遇到问题会直接显示构建失败

在实际项目中，开发人员经常看到：

```text
代码有问题
↓
Jenkins 构建失败
↓
新版本不会发布
```

这通常说明 Jenkins 背后的 Pipeline 已经由运维、DevOps 或项目负责人配置完成。

例如 Pipeline 可能执行：

```groovy
pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

如果 Maven 编译失败：

```text
mvn clean package
↓
返回非 0 退出码
↓
Jenkins 将 Stage 标记为失败
↓
后续 Deploy 不再执行
```

因此，以前看到的：

```text
构建失败后不会发布新内容
```

是正常、合理的发布逻辑。

这说明背后的流程至少实现了：

```text
构建失败阻止部署
```

有些公司还会进一步实现：

```text
部署命令失败阻止流量切换
健康检查失败保留旧版本
部署失败自动回滚
```

但是否具备这些能力，需要查看实际 Pipeline 和部署脚本，不能仅根据 Jenkins 页面推断。

---

## 五、Jenkins 如何判断成功或失败

Jenkins 最基础的判断依据是：

> 它执行的命令是否正常结束，以及命令返回的退出码是否为 0。

Linux 命令通常通过退出码表示结果：

```text
exit code = 0
表示命令执行成功

exit code != 0
表示命令执行失败
```

例如：

```bash
mvn clean package
```

如果代码编译失败、测试失败或 Maven 插件异常，Maven 会返回非 0 退出码。

Jenkins 因此可以识别：

```text
Build Failed
```

再例如：

```bash
scp app.jar server:/opt/app/
```

如果服务器不可访问或文件上传失败，命令通常返回非 0 退出码，Jenkins 会将部署阶段标记为失败。

因此：

```text
Jenkins 并不是自己理解了 Java 代码和系统状态
```

而是：

```text
Jenkins 根据 Pipeline、脚本逻辑和命令退出码判断结果
```

---

## 六、构建成功、部署成功和服务可用不是同一件事

需要区分三个层级。

### 1. 构建成功

通常表示：

```text
代码拉取成功
编译成功
测试通过
Jar 或镜像生成成功
```

但它不能证明：

```text
生产配置正确
数据库能够连接
Redis 能够连接
端口没有冲突
服务能够启动
Nginx 能够连接后端
所有业务接口都没有 Bug
```

### 2. 部署命令成功

可能只是表示：

```text
Jar 上传成功
或
Docker 收到了启动容器的命令
```

例如：

```bash
docker run -d my-service:123
```

这条命令返回成功时，容器可能刚刚被创建。

但几秒后 Spring Boot 可能因为以下原因退出：

```text
数据库连接失败
配置文件缺失
端口冲突
Bean 创建失败
配置中心不可用
环境变量不存在
```

如果 Pipeline 没有继续检查容器状态和健康接口，仍可能将 Deploy Stage 标记成功。

### 3. 服务真正可用

至少需要确认：

```text
进程或容器仍然存活
应用端口已经监听
健康检查接口返回正常
网关能够连接后端
关键依赖正常
必要的核心接口能够访问
```

因此正确关系是：

```text
构建成功
≠
部署命令成功
≠
服务真正可用
```

但在一个已经完善的发布平台中，运维可能已经把后两层检查集成进 Pipeline。

这时开发人员看到的就是：

```text
服务启动失败
↓
Jenkins 直接标记部署失败
↓
新版本不生效或自动恢复旧版本
```

---

## 七、Jenkins 成功后为什么仍然可能出现问题

Jenkins 成功后仍然可能出现两类问题。

### 1. Pipeline 没有检查到的问题

例如 Pipeline 只执行：

```bash
docker compose up -d
```

Docker 接受启动命令后返回 0，Jenkins 认为命令成功。

但随后：

```text
Java 应用连接数据库失败
↓
进程退出
↓
容器变成 Exited
↓
Nginx 无法连接后端
↓
用户访问返回 502
```

这属于：

```text
部署脚本验证不完整
```

### 2. Pipeline 无法覆盖的运行期业务问题

例如代码可以正常：

```text
编译
测试
打包
启动
```

但某个请求传入特殊数据以后触发：

```text
NullPointerException
SQL 异常
参数边界错误
状态机非法流转
```

此时 Jenkins 可以全部显示成功，但接口仍可能返回 500。

所以 Jenkins 成功更准确的含义是：

> 本次流水线中已经配置的构建、部署和验证步骤全部通过。

它不代表：

```text
整个系统不存在任何运行期问题
```

---

## 八、500 和 502 的区别

排查问题前必须先区分 HTTP 状态码。

### 1. 500 Internal Server Error

500 通常表示：

```text
请求已经到达后端应用
↓
后端在处理请求时发生异常
```

常见原因：

```text
空指针异常
SQL 执行失败
业务校验异常没有正确转换
下游调用异常
代码逻辑错误
```

排查重点通常是：

```text
Java 应用日志
异常堆栈
traceId
请求参数
数据库和下游服务
```

### 2. 502 Bad Gateway

502 通常表示：

```text
用户请求到达 Nginx 或网关
↓
Nginx 尝试访问后端服务
↓
连接失败或得到无效响应
```

常见原因：

```text
Java 服务没有启动
容器已经退出
后端端口没有监听
Nginx upstream 地址错误
端口映射错误
容器网络不通
后端连接超时
网关配置没有生效
```

因此：

```text
500 更偏向应用内部异常
502 更偏向网关到后端服务的调用链异常
```

但 502 的根因仍可能是 Java 应用没有正常启动，所以开发和运维都可能需要参与。

---

## 九、浏览器开发者工具能看到什么

出现接口异常时，可以打开浏览器开发者工具：

```text
F12
↓
Network
↓
找到失败请求
↓
查看 Status、URL、Response 和 Timing
```

开发者工具可以帮助确认：

```text
哪个接口失败
返回的是 500、502 还是其他状态码
请求发生的准确时间
请求 URL 和方法
请求参数和响应内容
是否所有接口都失败
```

但它通常不能直接确认：

```text
Nginx 为什么连接失败
后端容器为什么退出
服务器端口是否监听
网关 upstream 配置是否错误
```

因为这些信息位于服务器端，需要查看：

```text
Nginx error.log
应用日志
Docker 日志
服务器进程和端口
网络配置
```

因此更准确的处理流程是：

```text
浏览器确认请求现象和状态码
↓
提供失败接口、时间点和响应信息
↓
开发或运维查看服务器端日志
```

---

## 十、502 出现后应该找开发还是运维

不能简单地回答：

```text
一定找开发
```

或者：

```text
一定找运维
```

需要根据责任边界判断。

### 开发通常负责

```text
Java 应用启动日志
Spring Boot 启动异常
数据库、Redis、MQ 连接问题
配置读取错误
应用监听端口
健康检查接口
Dockerfile 中的应用启动命令
业务代码异常
```

### 运维或 DevOps 通常负责

```text
Jenkins 平台
服务器资源
Nginx 和负载均衡
域名和证书
防火墙和安全组
Docker 或 Kubernetes 基础环境
镜像仓库
发布脚本和服务器权限
节点、网络和磁盘问题
```

### 通常需要共同排查

```text
环境变量错误
容器端口映射错误
Nginx upstream 与应用端口不一致
容器网络问题
配置中心无法访问
健康检查失败
发布脚本缺陷
```

如果开发人员在浏览器中确认所有请求都返回 502，把问题交给运维处理是合理的。

但更高效的交接方式是先提供：

```text
故障发生时间
失败接口地址
HTTP 状态码
Jenkins 构建编号
本次部署模块和分支
是否刚刚发生发布
应用是否有明显启动异常
```

例如：

> Jenkins 构建编号 103 显示发布成功，后端模块为 API，发布后所有接口经 Nginx 访问返回 502。浏览器侧失败时间为 21:15，麻烦协助检查 Nginx upstream、端口映射和后端实例状态。

这比只说：

```text
系统挂了，帮忙看看
```

更容易快速定位。

---

## 十一、实际排查顺序

对于题目中的场景：

```text
Build Success
Deploy Success
用户访问返回 502 Bad Gateway
```

推荐按照以下顺序排查。

### 第一步：查看 Jenkins Console Output

先确认：

```text
部署的是哪个模块
使用的是哪个分支和 Commit
构建了哪个镜像或 Jar
部署到哪个环境和服务器
各 Stage 是否真的执行
是否有警告或被忽略的错误
是否执行了健康检查
```

重点关注脚本中是否存在：

```bash
command || true
```

这种写法会忽略失败：

```text
command 执行失败
↓
true 返回成功
↓
整个脚本退出码仍然是 0
↓
Jenkins 可能显示成功
```

### 第二步：确认后端应用是否存活

非 Docker 部署可以检查：

```bash
ps -ef | grep java
```

确认端口：

```bash
ss -lntp | grep 8080
```

绕过 Nginx 直接访问：

```bash
curl -v http://127.0.0.1:8080/actuator/health
```

如果本机直接访问后端都失败，问题更可能在：

```text
Java 应用启动
配置
依赖连接
应用端口
```

### 第三步：查看应用启动日志

重点寻找：

```text
APPLICATION FAILED TO START
Port already in use
Failed to configure a DataSource
Connection refused
BeanCreationException
配置中心连接失败
环境变量缺失
```

### 第四步：检查 Nginx

检查服务状态：

```bash
systemctl status nginx
```

检查配置语法：

```bash
nginx -t
```

查看错误日志：

```bash
tail -n 200 /var/log/nginx/error.log
```

常见错误：

```text
connection refused
upstream timed out
no live upstreams
host not found in upstream
```

### 第五步：检查服务器和网络

包括：

```text
CPU 和内存是否耗尽
磁盘是否已满
端口是否被防火墙阻断
容器网络是否正常
目标服务器是否可达
```

正确思路不是盲目检查所有内容，而是先确定故障层级：

```text
浏览器到 Nginx 是否正常
Nginx 到后端是否正常
后端进程是否正常
应用依赖是否正常
```

---

## 十二、Docker 部署需要检查哪些内容

### 1. 容器状态

```bash
docker ps -a
```

重点查看：

```text
Up
Exited
Restarting
Unhealthy
```

如果状态是：

```text
Exited (1)
```

通常表示应用启动后异常退出。

### 2. 容器日志

```bash
docker logs --tail 300 容器名
```

实时查看：

```bash
docker logs -f 容器名
```

重点确认：

```text
Spring Boot 是否完成启动
数据库是否连接成功
配置是否读取成功
端口是否冲突
JVM 是否异常退出
```

### 3. 端口映射

```bash
docker port 容器名
```

或者：

```bash
docker inspect 容器名
```

需要确认：

```text
宿主机端口
↓
正确映射到容器应用端口
```

例如：

```text
Nginx 访问宿主机 8080
但容器实际映射为 8081:8080
```

就会出现连接失败。

### 4. 容器内部服务

进入容器：

```bash
docker exec -it 容器名 sh
```

在容器内访问：

```bash
curl http://127.0.0.1:8080/actuator/health
```

如果容器内部正常、宿主机访问失败，重点检查：

```text
端口映射
监听地址
Docker 网络
防火墙
```

### 5. Docker 网络

```bash
docker network ls
docker network inspect 网络名
```

如果 Nginx 和 Java 服务都运行在容器中，Nginx 容器里的：

```text
127.0.0.1
```

表示 Nginx 容器自己，不是 Java 容器。

通常应该通过 Docker 服务名访问：

```text
http://backend:8080
```

### 6. 环境变量和挂载

通过：

```bash
docker inspect 容器名
```

确认：

```text
Spring Profile
数据库地址
Redis 地址
配置中心地址
JVM 参数
配置文件挂载
日志目录挂载
Secret
```

### 7. 镜像版本

确认当前容器运行的镜像：

```bash
docker inspect --format='{{.Config.Image}}' 容器名
```

避免出现：

```text
Jenkins 构建的是新镜像
但服务器仍然启动了旧镜像
```

---

## 十三、如何让 Pipeline 验证服务真正启动成功

最基础的做法是在部署后增加健康检查 Stage。

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Deploy') {
            steps {
                sh 'docker compose up -d'
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    for i in $(seq 1 30); do
                        if curl -fsS http://127.0.0.1:8080/actuator/health \
                            | grep -q '"status":"UP"'; then
                            echo "service started successfully"
                            exit 0
                        fi

                        echo "waiting for service... attempt=$i"
                        sleep 5
                    done

                    echo "service failed to start"
                    docker ps -a
                    docker logs --tail 200 backend-container
                    exit 1
                '''
            }
        }
    }
}
```

逻辑是：

```text
启动新版本
↓
每 5 秒检查一次健康接口
↓
最多检查 30 次
↓
接口返回 status=UP
↓
Pipeline 成功
```

如果超过等待时间仍未成功：

```text
输出容器状态和日志
↓
exit 1
↓
Jenkins 标记失败
```

需要注意：

```text
只检查进程存在
```

不够可靠。

因为进程可能存在，但：

```text
数据库连接池不可用
Redis 无法连接
应用仍未完成初始化
关键依赖异常
```

健康检查应该至少验证：

```text
HTTP 能够访问
返回码正确
响应内容符合预期
关键依赖是否可用
```

---

## 十四、健康检查也不能无限等待

错误的 Pipeline 可能这样写：

```bash
until curl -f http://127.0.0.1:8080/actuator/health; do
    sleep 5
done
```

如果服务永远无法启动，Pipeline 就会一直等待。

正确做法需要：

```text
有限次数
+
固定或递增等待
+
整体超时
+
失败日志输出
```

Jenkinsfile 还可以增加：

```groovy
timeout(time: 3, unit: 'MINUTES') {
    sh './health-check.sh'
}
```

这样可以防止：

```text
构建任务永久占用 Executor
```

---

## 十五、新版本失败时为什么要考虑保留旧版本

最危险的简单发布流程是：

```text
先停止旧版本
↓
再启动新版本
↓
新版本启动失败
↓
系统完全不可用
```

更稳妥的流程是：

```text
旧版本继续提供服务
↓
启动新版本
↓
检查新版本健康状态
↓
健康后切换流量
↓
再次验证
↓
停止旧版本
```

如果新版本失败：

```text
不切换流量
↓
旧版本继续运行
```

这种思想可以通过以下方式实现：

```text
蓝绿发布
滚动发布
Kubernetes Deployment
双容器切换
Nginx upstream 切换
```

因此题目中提到：

```text
新版本启动失败如何回滚
```

并不是说所有 Jenkins 都会自动做到。

而是考察：

> 发布流程是否有能力在新版本异常时恢复到最近一个已知可用版本。

---

## 十六、快速回滚至少需要保留哪些信息

### 1. Git Commit ID

用于确认源码版本：

```text
a82f6c1
```

### 2. Git 分支或 Tag

例如：

```text
main
release/1.3.0
v1.3.0
```

### 3. Jenkins Build Number

例如：

```text
Build #103
```

用于关联：

```text
构建日志
执行时间
发布环境
镜像版本
发布结果
```

### 4. Jar 或 Docker 镜像版本

不要只使用：

```text
backend:latest
```

因为 `latest` 会不断变化，无法准确回到历史版本。

更合理的 Tag：

```text
backend:build-103
backend:a82f6c1
backend:20260724-103
```

更严格的场景还可以记录镜像 Digest。

### 5. 部署环境

例如：

```text
test
staging
production
```

### 6. 配置版本

代码版本正确，但配置错误，仍然可能启动失败。

因此需要记录：

```text
配置中心版本
环境变量版本
Secret 版本
配置变更时间
```

### 7. 数据库变更版本

例如 Flyway 或 Liquibase 版本：

```text
V103__add_task_retry_column.sql
```

代码可以回滚，但数据库结构不一定能够直接回滚。

所以发布前需要考虑：

```text
新旧代码是否都兼容当前表结构
数据库变更是否可逆
是否采用先加字段再切换代码
```

### 8. 发布时间和操作人

用于事故定位和审计：

```text
发布时间
发布人
审批人
变更说明
```

最少需要形成以下关联：

```text
Git Commit
↕
Jenkins Build Number
↕
Jar 或镜像版本
↕
部署环境
↕
配置和数据库版本
```

---

## 十七、一个更可靠的部署脚本思路

```bash
#!/usr/bin/env bash

set -euo pipefail

CONTAINER_NAME="backend-service"
HEALTH_URL="http://127.0.0.1:8080/actuator/health"

# 启动或更新容器
docker compose up -d

# 等待健康检查
for i in $(seq 1 30); do
    if curl -fsS "$HEALTH_URL" | grep -q '"status":"UP"'; then
        echo "deployment succeeded"
        exit 0
    fi

    echo "waiting for application, attempt=$i"
    sleep 5
done

echo "deployment failed"
docker ps -a
docker logs --tail 200 "$CONTAINER_NAME" || true
exit 1
```

其中：

```bash
set -euo pipefail
```

用于降低 Shell 脚本静默失败的概率。

含义包括：

```text
-e：命令失败时终止脚本
-u：使用未定义变量时报错
-o pipefail：管道中任意命令失败时返回失败
```

但需要注意：

```text
set -e
```

也不能代替完整的业务判断。

例如：

```bash
docker compose up -d
```

本身成功，并不意味着应用健康。

所以仍然需要显式健康检查。

---

## 十八、开发人员如何正确使用 AI 协助排查

遇到线上故障时，直接截图发给 AI 可以作为辅助，但不能只提供：

```text
系统打不开了，为什么
```

更有效的方式是先收集证据：

```text
HTTP 状态码
失败接口和时间点
Jenkins Build Number
Console Output 中的异常
容器状态
应用日志
Nginx 错误日志
端口监听情况
```

然后向 AI 提供：

```text
现象
+
时间线
+
部署版本
+
关键日志
+
已经排除的方向
```

例如：

```text
Jenkins Build #103 显示成功。
部署后所有接口返回 502。
docker ps 显示 backend-container 为 Restarting。
docker logs 中出现数据库连接失败。
请分析根因和修复顺序。
```

这种情况下，AI 能基于证据进行分析。

正确协作方式是：

```text
自己先确定故障层级
↓
收集日志和状态
↓
让 AI 帮助解释和形成排查路径
↓
由开发或运维执行验证
```

而不是：

```text
不收集信息
↓
让 AI 猜测所有可能性
```

---

## 十九、今天思考题复盘

### 问题一：为什么 Jenkins 显示成功，但系统仍不可用

原因可能是：

```text
Jenkins 只验证了构建和部署命令
部署脚本没有检查服务健康状态
容器启动后立即退出
Nginx upstream 配置错误
应用运行期依赖异常
发布后出现未被自动化测试覆盖的业务问题
```

最核心结论是：

> Jenkins 成功表示 Pipeline 中已配置的步骤成功，不代表 Pipeline 没有验证的内容也一定正常。

但如果公司已经由运维配置了完善的健康检查，那么服务启动失败通常会被 Jenkins 识别为部署失败。

### 问题二：应该按照什么顺序排查

推荐顺序：

```text
确认 HTTP 状态码和失败范围
↓
查看 Jenkins Console Output
↓
确认发布模块、Commit 和镜像版本
↓
检查 Java 进程或 Docker 容器状态
↓
查看应用启动日志
↓
绕过 Nginx 访问健康接口
↓
检查 Nginx upstream 和 error.log
↓
检查服务器资源、网络和防火墙
```

### 问题三：Docker 部署需要检查哪些内容

至少包括：

```text
容器状态
容器日志
端口映射
容器内部健康接口
Docker 网络
环境变量
配置文件挂载
镜像版本
资源限制
```

### 问题四：如何让服务真正启动前不标记成功

Pipeline 中增加：

```text
部署后等待
健康检查重试
整体超时
响应内容校验
失败日志输出
exit 1
```

更成熟的流程还应：

```text
新版本健康后再切换流量
新版本失败时保留旧版本
```

### 问题五：快速回滚至少保留哪些信息

最少包括：

```text
Git Commit ID
Git 分支或 Tag
Jenkins Build Number
Jar 或镜像版本
部署环境
配置版本
数据库变更版本
发布时间和操作人
```

---

## 二十、今天内容的实际价值判断

今天的 Jenkins 基础内容本身偏简单。

原因是已经具备以下实际经验：

```text
进入 Jenkins
选择模块
点击部署
查看成功或失败
出现 502 时交给运维处理
```

因此以下内容的边际价值较低：

```text
Jenkins 可以自动拉取代码
Jenkins 可以自动打包
Pipeline 包含 Build 和 Deploy
```

今天真正补充的认知是：

```text
以前使用的是已经配置好的发布流程
Jenkins 能识别哪些失败取决于 Pipeline
502 需要区分应用层和基础设施层
浏览器只能确认现象，不能代替服务器日志
开发交给运维前应提供基本故障信息
健康检查和版本回滚属于更深入的 CI/CD 能力
```

因此今天的学习不能算高难度 Jenkins 实战，但完成了从：

```text
只会使用 Jenkins 页面
```

到：

```text
理解按钮背后 Pipeline 的判断边界
```

这一层认知补充。

---

## 二十一、面试表达

可以这样回答：

> 我之前在项目中实际使用过 Jenkins 进行模块化发布。发布平台已经由运维或 DevOps 配置完成，我通常会选择对应模块和分支，触发构建并根据 Console Output 判断执行结果。编译、打包或部署脚本失败时，Pipeline 会直接终止，不会继续发布新版本。
>
> 我理解 Jenkins 的成功状态取决于 Pipeline 中配置了哪些检查。构建成功只能证明代码完成了编译、测试和打包，部署命令成功也不一定等于服务真正可用。如果脚本只启动容器，没有检查容器状态和健康接口，就可能出现 Pipeline 成功但服务不可用。因此部署后应该增加带超时和重试的健康检查，确认应用端口和 `/actuator/health` 正常后再标记发布成功。
>
> 如果用户访问返回 502，我会先确认 Jenkins 的构建编号、部署模块和 Console Output，然后检查应用或容器是否存活、端口是否监听以及应用启动日志。如果应用正常，再结合 Nginx upstream 和错误日志判断是否属于网关、端口映射或网络问题。真实团队中这类问题通常需要开发和运维协同，而不是简单归属于某一方。
>
> 为了支持快速回滚，我会保留 Git Commit ID、Jenkins Build Number、不可变的 Jar 或 Docker 镜像版本、配置版本和数据库变更版本，避免只使用无法定位历史内容的 `latest` 标签。

---

## 二十二、今天的最终结论

今天最需要避免的错误判断是：

```text
Jenkins 显示成功
=
整个系统绝对没有问题
```

更准确的理解是：

```text
Jenkins 显示成功
=
Pipeline 中已经配置的步骤和检查全部通过
```

是否能够证明服务真正可用，取决于 Pipeline 是否包含：

```text
进程或容器状态检查
端口检查
健康接口检查
关键依赖检查
流量切换后的验证
```

同时需要明确：

```text
500 通常更偏向应用内部异常
502 通常更偏向网关无法连接后端
```

出现 502 时，把问题交给运维处理可能完全合理，但开发人员最好先提供：

```text
失败接口
故障时间
HTTP 状态码
Jenkins 构建编号
部署模块和版本
应用是否正常启动
```

今天需要记住的核心结论是：

> Jenkins 不是自动理解系统是否正确，而是执行别人设计好的流程。成熟的 Pipeline 不仅要保证代码能够构建和发布，还要验证新版本真正健康、保留可回滚制品，并在失败时输出足够的日志，让开发和运维能够快速确定故障发生在哪一层。

能力等级：

```text
使用 Jenkins 页面完成模块发布
→ 基础工程实践

能够阅读 Console Output 并判断失败 Stage
→ 初级到中级工程能力

能够设计健康检查、版本管理、流量切换和回滚
→ 中级 CI/CD 工程能力
```
