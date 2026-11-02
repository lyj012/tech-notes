# 后端工程化每日训练 Day 68：systemd 服务管理、Java 启动失败与服务依赖排障复盘

## 一、今天学习什么

今天学习 Linux 服务器上一个非常常见、也非常通用的工程化主题：

**systemd 服务管理，以及 Java 服务“启动失败、启动后退出、进程还在但业务不可用”时应该怎么排查。**

在实际工作里，经常会遇到这些现象：

```text
systemctl start xxx.service
→ failed

服务器重启之后
→ Java 服务没有起来

Java 进程存在
→ 但 8082 没有监听

手工 java -jar 能启动
→ systemctl start 却失败

服务每隔几秒
→ 启动、退出、自动重启、再次退出
```

这些现象看起来都像“服务坏了”，但真正的根因可能完全不同。

今天最核心的一句话是：

> **不要把 systemd 显示的状态当成业务最终状态，要沿着“systemd → 进程 → 日志 → 端口 → 外部依赖 → 业务请求”逐层确认。**

---

## 二、systemd 到底在管理什么

可以把 systemd 简单理解为 Linux 上的服务管理器。

一个 Java 服务通常会有类似配置：

```ini
[Unit]
Description=Java Application

[Service]
User=app
WorkingDirectory=/opt/app
ExecStart=/usr/bin/java -jar /opt/app/app.jar
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

当执行：

```bash
systemctl start app.service
```

systemd 大致会做：

```text
读取 service 配置
↓
按照 User 切换运行用户
↓
进入 WorkingDirectory
↓
按照 ExecStart 执行命令
↓
跟踪主进程 PID
↓
根据进程退出状态更新服务状态
```

因此：

```text
systemctl start app
```

并不等价于当前 SSH Shell 中直接执行：

```bash
java -jar app.jar
```

因为两种方式的：

```text
运行用户
工作目录
PATH
JAVA_HOME
启动参数
环境变量
文件权限
```

都可能不同。

---

## 三、`failed` 只是结果，不是根因

执行：

```bash
systemctl status app
```

看到：

```text
Active: failed
status=1/FAILURE
```

只能说明：

> systemd 管理的主进程异常退出了。

它不能直接告诉我们为什么退出。

根因可能是：

```text
配置文件错误
端口被占用
JDK 路径错误
目录或文件权限不足
数据库连接失败
Redis / MQ 等依赖不可用
Spring Bean 初始化失败
磁盘满
内存不足
```

所以不能只看：

```text
Active: failed
```

就开始反复：

```bash
systemctl restart app
```

因为这样没有增加任何新的排障证据。

下一步应该优先看日志：

```bash
journalctl -u app.service -n 100
```

需要实时观察时：

```bash
journalctl -u app.service -f
```

排障时重点不是找最后一句：

```text
service failed
```

而是往前找：

> **第一条真正有意义、能够解释为什么应用退出的异常。**

完整思路：

```text
failed
↓
只是现象
↓
journalctl
↓
找到真实异常
↓
再进入对应层继续排查
```

---

## 四、systemd 显示 failed，但还能看到 Java 进程

场景：

```bash
systemctl status app
```

显示：

```text
failed
```

但执行：

```bash
ps -ef | grep java
```

还能看到 Java。

这里不能直接得出：

```text
“app 启动失败了，但它自己的 JVM 还活着”
```

因为这个 Java 进程可能根本不是 systemd 当前管理的那个主进程。

常见可能包括：

```text
1. 机器上还有另一个 Java 服务
2. 之前手工 java -jar 启动过一个进程
3. 启动脚本派生出了子进程，systemd 跟踪的主进程已经退出
4. service 的 Type / 启动脚本设计导致 systemd 跟踪错进程
5. 存在遗留或孤立进程
```

首先看 systemd 认为的主进程：

```bash
systemctl status app
```

或者：

```bash
systemctl show app -p MainPID
```

再查看对应 PID：

```bash
ps -fp <PID>
```

同时查看 service 实际启动命令：

```bash
systemctl cat app.service
```

重点对比：

```text
MainPID
ExecStart
实际 Java 命令行
jar 路径
```

如果：

```text
MainPID=0
```

但 `ps` 里仍然有 Java，则至少可以确定：

> 当前看到的这个 Java 进程并不是 systemd 正在跟踪的 app 主进程。

---

## 五、手工 `java -jar` 能启动，但 systemd 启动失败

这是非常典型的线上问题。

现象：

```bash
java -jar app.jar
```

可以正常启动。

但是：

```bash
systemctl start app
```

失败。

如果两次操作发生在同一台机器、同一时间，那么第一优先级不应该马上怀疑网络，而应该先比较：

> **手工启动环境和 systemd 运行环境到底哪里不同。**

### 1. 运行用户

手工操作可能使用：

```text
root
```

systemd 配置却可能是：

```ini
User=app
```

这会影响：

```text
jar 文件读取权限
配置文件读取权限
日志目录写入权限
上传目录权限
证书权限
```

### 2. Java 路径和环境变量

当前 Shell 可能存在：

```text
JAVA_HOME
PATH
```

但 systemd 默认不会完全继承当前交互式 Shell 的环境。

所以要确认：

```ini
ExecStart=/usr/bin/java ...
```

实际使用的是不是预期 JDK。

可以检查：

```bash
which java
java -version
```

以及 service 中是否配置：

```ini
Environment=
EnvironmentFile=
```

### 3. WorkingDirectory

例如：

```ini
WorkingDirectory=/opt/app
```

如果程序使用相对路径读取：

```text
./config/application.yml
./logs
./cert
```

工作目录不同就可能导致程序找不到文件。

### 4. ExecStart 启动参数

需要对比：

```text
jar 路径
Spring Profile
配置文件路径
JVM 参数
端口参数
其他启动参数
```

例如：

```ini
ExecStart=/usr/bin/java -jar /opt/app/app.jar --spring.profiles.active=prod
```

与手工执行：

```bash
java -jar app.jar
```

实际上并不是完全相同的启动方式。

因此这一类问题可以先记成：

```text
手工能启动
+
systemd 不能启动
↓
优先比较运行上下文
```

---

## 六、`active (running)` 不等于业务已经正常

这是今天非常重要的一层校准。

假设：

```bash
systemctl status app
```

显示：

```text
Active: active (running)
```

但：

```bash
ss -lntp | grep 8082
```

没有结果。

这时不能认为服务已经正常。

`active (running)` 更准确的含义是：

> systemd 当前认为它管理的主进程还活着。

它并不能证明：

```text
Spring Boot 已经初始化完成
Tomcat 已经成功启动
8082 已经监听
数据库已经连接成功
业务接口已经可以响应
```

可以把服务正常程度分成四层：

```text
systemd active
↓
主进程还活着

Java 进程存在
↓
JVM 仍然运行

TCP 端口监听
↓
Web Server 已经成功 bind 端口

真实业务请求成功
↓
业务才真正具备可用性
```

因此：

```text
systemd active
≠ Java 应用初始化完成

Java 进程存在
≠ 8082 已经监听

8082 监听
≠ 所有业务功能正常
```

最终仍然要验证：

```bash
ss -lntp | grep 8082
curl http://127.0.0.1:8082
```

必要时还要从真实调用方发起请求。

---

## 七、为什么 Java 进程活着，但 8082 还没有监听

一种很常见的情况是应用还处于启动过程中。

例如：

```text
systemd 启动 Java
↓
JVM 进程创建
↓
Spring Boot 开始初始化
↓
扫描 Bean
↓
初始化数据库连接池
↓
连接 Redis / MQ / 配置中心
↓
启动 Web Server
↓
绑定 8082
```

如果程序卡在：

```text
初始化数据库连接池
```

那么此时可能出现：

```text
Java 进程存在
systemd active
8082 尚未监听
```

所以看到进程存在时，还需要结合：

```text
日志进度
端口监听
依赖状态
```

继续判断程序到底运行到了哪一步。

---

## 八、`Connection refused` 到底说明什么

场景：

```bash
systemctl status app
```

显示：

```text
active (running)
Main PID: 12345
```

并且：

```bash
ps -fp 12345
```

确认：

```text
java -jar /opt/app/app.jar
```

但是：

```bash
curl http://127.0.0.1:8082
```

返回：

```text
Connection refused
```

这里说明：

> TCP 请求已经到达本机，但当前 8082 没有进程监听，因此本机内核直接返回 RST。

不能理解成：

```text
Linux 知道 Java 应用启动失败了，所以主动拒绝
```

Linux 内核并不知道 Spring Boot 业务是否“启动成功”。

它只知道：

```text
8082 当前没人监听
```

所以：

```text
active (running)
+
Connection refused
```

完全可以同时存在。

---

## 九、`Restart=always` 与重启循环

如果 service 配置：

```ini
Restart=always
```

而 Java 每次启动后都会异常退出，就可能形成：

```text
启动
↓
Java 崩溃
↓
systemd 自动重启
↓
再次崩溃
↓
再次重启
```

这可以形成持续的重启循环。

此时第一目标不是继续调整 Restart 参数来“治好服务”，因为：

> **Restart 只决定进程退出以后 systemd 做什么，它不会修复 Java 为什么退出。**

排障时更直接的止血方式是：

```bash
systemctl stop app
```

先停止重启循环，再看日志：

```bash
journalctl -u app -n 100
```

完整顺序：

```text
发现反复重启
↓
先 stop，停止制造新故障和日志噪声
↓
查看 journal
↓
找到第一次真正的 Java 异常
↓
检查配置 / 权限 / 端口 / 外部依赖
↓
修复根因
↓
重新启动
↓
验证进程、端口、业务
```

如果需要长期控制重启频率，再考虑：

```text
RestartSec
StartLimitBurst
StartLimitIntervalSec
```

但它们属于恢复策略，不是应用故障根因。

---

## 十、`Address already in use` 怎么排

日志出现：

```text
Application failed to start

Caused by:
Address already in use
```

这时优先进入：

> Linux 端口 / 进程层。

不是先排 systemd，也不是先排交换机、路由之类的网络链路。

例如应用应该监听 8082：

```bash
ss -lntp | grep 8082
```

可能看到：

```text
LISTEN ... 0.0.0.0:8082 ... users:(("java",pid=1234,fd=56))
```

已经能够得到：

```text
进程名
PID
监听端口
```

然后继续：

```bash
ps -fp 1234
```

确认这个 PID 到底是什么程序。

固定链路：

```text
Address already in use
↓
查监听端口
↓
找到 PID
↓
确认进程身份
↓
判断是旧进程未退出、重复启动还是端口配置冲突
```

相比：

```bash
ps -ef | grep 8082
```

`ss -lntp` 更直接，因为 `ps` 搜索的是进程命令行文本，不是系统真实的端口监听关系。

---

## 十一、数据库 IP 能 Ping，但 3306 Timeout 怎么理解

场景：

```text
Java 主进程还活着
8082 没有监听
日志停在 Connecting to database...
```

数据库服务器：

```bash
ping 10.0.0.20
```

正常。

但是从 Java 所在服务器执行：

```bash
nc -vz 10.0.0.20 3306
```

超时。

这里首先要区分：

```text
Ping 通
```

只能证明：

```text
ICMP 层面存在可达性
```

不能证明：

```text
TCP 3306 可以建立连接
```

当前证据更指向：

> **Java 服务器到数据库服务器的 TCP 3306 链路，或者数据库监听本身存在问题。**

继续可以检查：

```bash
ip addr
ip route
ip neigh
```

再根据网络拓扑继续检查：

```text
路由
ACL
防火墙
VLAN
数据库监听地址
数据库是否真正监听 3306
VPN（前提是该项目架构本来就要求走 VPN）
```

这里还有一个重要排障原则：

> **应该优先从真正发起数据库连接的 Java 服务器测试，而不是先从自己的电脑测试。**

因为：

```text
自己的电脑 → 数据库
```

和：

```text
Java 服务器 → 数据库
```

可能完全不是同一条网络路径。

如果 `nc` 表现为 timeout，则说明 TCP 三次握手没有正常完成。

但不能看到 timeout 就直接认定一定是 VPN，需要结合真实拓扑继续缩小范围。

---

## 十二、`After` 和 `Requires` 的区别

这两个配置通常写在 systemd service 的 `[Unit]` 段。

例如：

```ini
[Unit]
Description=Java Monitor Service
Requires=mysql.service
After=mysql.service
```

### 1. `After`

可以先记成：

> **启动顺序关系。**

```ini
After=mysql.service
```

表示：

```text
如果 app 和 mysql 都要启动
那么 app 的启动动作排在 mysql 后面
```

需要注意：

> `After` 本身主要描述顺序，并不等于“自动建立强依赖”。

也就是说，仅仅写：

```ini
After=mysql.service
```

不能简单理解成：

```text
MySQL 失败，app 就绝对不会启动
```

### 2. `Requires`

可以先记成：

> **强依赖关系。**

```ini
Requires=mysql.service
```

表示 app 对 mysql 存在 requirement dependency。

启动 app 时，systemd 会同时尝试拉起要求的单元。

通常会和 `After` 配合：

```ini
Requires=mysql.service
After=mysql.service
```

可以帮助记忆为：

```text
Requires
→ 我需要它

After
→ 我的启动顺序在它后面
```

---

## 十三、`After=mysql.service` 为什么仍然不能保证数据库已经可用

这是 systemd 服务依赖中一个非常容易误判的地方。

即使：

```text
mysql.service
→ active
```

也不一定代表：

```text
数据库已经完全 ready，可以接受 Java 的业务连接
```

例如数据库启动过程中可能还在：

```text
恢复数据
执行初始化
加载表空间
回放日志
打开监听
```

所以可能发生：

```text
服务器重启
↓
systemd 启动数据库
↓
数据库主进程已经进入 active
↓
systemd 按 After 顺序启动 Java
↓
数据库内部仍未完全 ready
↓
Java 初始化连接池
↓
连接数据库失败
↓
Spring ApplicationContext 启动失败
↓
JVM 退出
↓
systemd 显示 app failed
```

因此：

```text
进程启动顺序正确
```

不等于：

```text
业务依赖已经真正 ready
```

这也是为什么应用层仍然需要合理的：

```text
连接超时
重试
退避
健康检查
容错
```

而不能完全依赖 systemd 的启动顺序解决所有依赖可用性问题。

---

## 十四、今天形成的分层排障方法

以后遇到：

```text
systemctl start xxx
失败
```

可以固定按照下面的层次排。

### 第一层：先确认现象

先区分：

```text
完全没启动？

启动后立即退出？

正在反复重启？

systemd active，但端口没有监听？

端口监听，但业务访问失败？
```

这些不是同一种故障。

### 第二层：systemd 层

```bash
systemctl status xxx
systemctl cat xxx
systemctl show xxx -p MainPID
```

重点确认：

```text
Active
MainPID
退出码
ExecStart
User
WorkingDirectory
Environment
Restart
After
Requires
```

### 第三层：日志层

```bash
journalctl -u xxx -n 100
```

核心目标：

> 找到第一条真正解释应用失败原因的异常。

### 第四层：Linux 进程 / 文件 / 权限 / 资源层

根据异常继续检查：

```bash
ps -fp <PID>
ls -l
df -h
free -h
which java
java -version
ss -lntp
```

### 第五层：应用层

检查：

```text
Spring Boot 是否完成初始化
配置文件是否正确
Bean 是否创建成功
端口是否真正绑定
```

### 第六层：外部依赖层

继续确认：

```text
数据库
Redis
MQ
配置中心
远程 API
```

以及相应 TCP 端口是否真正可达。

### 第七层：业务验证

不能只看到：

```text
active (running)
```

就结束排障。

还要继续验证：

```bash
ss -lntp | grep 8082
curl http://127.0.0.1:8082
```

最后最好从真实调用方验证。

最终链路：

```text
现象
↓
systemd 状态
↓
journal 日志
↓
主进程 / 用户 / 文件 / 权限 / 资源
↓
Java 应用启动过程
↓
端口监听
↓
数据库 / Redis / MQ 等依赖
↓
真实业务请求
```

---

## 十五、今天纠正的几个容易混淆的点

### 1. `active (running)` 不是“正在启动”

它更准确表示：

> systemd 当前认为主进程处于运行状态。

但不能直接推出业务已经 ready。

### 2. Java 进程存在不代表 Web 服务成功

可能出现：

```text
JVM 存在
↓
Spring 仍在初始化 / 卡住
↓
8082 没监听
```

### 3. 8082 是 Java 应用端口，不是数据库端口

数据库可能是：

```text
3306
```

Java Web 服务可能是：

```text
8082
```

两者需要分开验证。

### 4. `Connection refused` 不是 Linux 在判断业务成功与否

它表示：

```text
TCP 到达目标主机
但目标端口当前没有监听者
```

### 5. `Restart=always` 不是故障修复方案

它只是进程退出以后的恢复策略。

### 6. `After` 不等于依赖已经 ready

```text
启动顺序正确
≠
数据库业务已经可连接
```

---

## 十六、今天需要记住的命令

### systemd 状态

```bash
systemctl status app.service
```

### 查看 service 配置

```bash
systemctl cat app.service
```

### 查看 systemd 记录的主 PID

```bash
systemctl show app.service -p MainPID
```

### 查看日志

```bash
journalctl -u app.service -n 100
```

### 实时看日志

```bash
journalctl -u app.service -f
```

### 修改 unit 文件后重新加载

```bash
systemctl daemon-reload
```

### 查看 Java 进程

```bash
ps -ef | grep java
ps -fp <PID>
```

### 查看监听端口

```bash
ss -lntp | grep 8082
```

### 测试本机 HTTP 服务

```bash
curl http://127.0.0.1:8082
```

### 测试远程 TCP 端口

```bash
nc -vz 10.0.0.20 3306
```

### 查看本机网络配置

```bash
ip addr
ip route
ip neigh
```

---

## 十七、今天最终形成的排障模型

今天最重要的不是记住几个 systemctl 命令，而是建立一个更完整的服务排障模型：

```text
systemd 管理的是进程
↓
进程活着不等于应用 ready
↓
应用 ready 还需要端口真正监听
↓
端口监听不等于所有依赖正常
↓
最终仍然要用真实业务请求验证
```

以后再看到：

```text
Java 服务启动失败
```

不能只停留在：

```text
重启一下试试
```

而应该继续追问：

```text
systemd 到底执行了什么？
主进程是谁？
为什么退出？
日志第一条关键异常是什么？
端口有没有监听？
依赖有没有真正 ready？
业务请求能不能真正完成？
```

可以压缩成一句话：

> **先确认 systemd 管理的进程状态，再通过日志找到失败位置，最后用端口和真实请求证明服务是否真正恢复。**
