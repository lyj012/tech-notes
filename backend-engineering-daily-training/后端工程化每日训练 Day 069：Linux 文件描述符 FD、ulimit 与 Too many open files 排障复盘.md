# 后端工程化每日训练 Day 69：Linux 文件描述符 FD、ulimit 与 Too many open files 排障复盘

## 一、今天学习什么

今天学习 Linux 服务器中非常常见的一类资源耗尽问题：

**文件描述符（File Descriptor，FD）耗尽，以及 Java 服务出现 `Too many open files` 时如何排查。**

线上经常出现这样的情况：

```text
Java 进程还存在
CPU 正常
内存正常
磁盘正常

但是：
HTTP 请求失败
数据库连接失败
日志文件无法打开

错误：
Too many open files
```

核心问题：

> Linux 进程能够同时打开的资源数量存在限制，当 FD 耗尽时，进程无法继续创建新的资源。

---

## 二、核心概念

### 1. 什么是 FD

FD（File Descriptor，文件描述符）是 Linux 中进程访问已打开资源的编号。

例如 Java 服务：

```text
FD 0  → 标准输入
FD 1  → 标准输出
FD 2  → 标准错误
FD 3  → 日志文件
FD 4  → TCP Socket
FD 5  → MySQL连接
FD 6  → Redis连接
```

重点：

```text
FD 不只是磁盘文件
```

Socket、数据库连接、日志文件等都会占用 FD。

---

## 三、为什么会出现 Too many open files

类似内存泄漏：

```text
申请内存
↓
没有释放
↓
越来越多
↓
OOM
```

FD 泄漏：

```text
打开资源
↓
没有关闭
↓
FD越来越多
↓
达到限制
↓
Too many open files
```

例如：

```text
正常：
2000 FD

异常：
5000
10000
30000
60000
```

最终：

```text
无法创建新的 Socket
无法打开文件
数据库连接失败
```

---

## 四、排查命令

## 1. 查看当前 Shell 限制

```bash
ulimit -n
```

注意：

如果 Java 是 systemd 启动：

```text
SSH Shell 的限制
≠
Java进程的限制
```

---

## 2. 查看 Java 进程真实限制

找到 PID：

```bash
ps -ef | grep java
```

查看：

```bash
cat /proc/PID/limits
```

重点：

```text
Max open files
```

---

## 3. 查看当前 FD 使用量

```bash
ls /proc/PID/fd | wc -l
```

例如：

```text
Max open files = 65535
当前 FD = 65000
```

说明：

```text
FD 已经接近耗尽
```

但是不能直接证明：

```text
一定是代码泄漏
```

---

## 4. 查看 FD 类型

```bash
lsof -p PID
```

lsof：

```text
list open files
```

用于查看进程打开了哪些资源。

如果大量出现：

```text
socket
```

说明方向转向网络连接。

---

## 5. 查看 TCP 状态

```bash
ss -antp
```

重点关注：

```text
ESTABLISHED
CLOSE_WAIT
TIME_WAIT
```

特别是：

```text
CLOSE_WAIT持续增长
```

需要重点排查应用是否没有关闭 Socket。

---

## 五、CLOSE_WAIT 与 FD 泄漏链路

完整链路：

```text
客户端关闭连接
↓
发送 FIN
↓
服务器进入 CLOSE_WAIT
↓
等待 Java 应用调用 close()
↓
Java没有释放Socket
↓
Socket持续占用FD
↓
FD数量持续增长
↓
达到Max open files限制
↓
新连接创建失败
↓
Too many open files
```

注意：

```text
CLOSE_WAIT不是为了等待客户端关闭
```

它表示：

> 对端已经关闭连接，但是本地应用还没有关闭 Socket。

---

## 六、容量不足 vs FD泄漏

这是排障中最重要的判断。

### 情况一：正常业务容量超过限制

例如：

```text
业务正常需要 120000 FD
限制只有 65535
```

处理：

```text
评估真实容量
↓
合理提高 LimitNOFILE
↓
继续监控
```

systemd 示例：

```ini
[Service]
LimitNOFILE=200000
```

---

### 情况二：程序存在 FD 泄漏

表现：

```text
业务量没有明显变化

FD持续上涨

1000
5000
20000
50000
```

处理：

```text
定位未关闭资源
↓
修复代码
↓
增加监控
```

单纯调整：

```text
65535 → 200000
```

只能延迟故障，不解决根因。

---

## 七、真实故障排查案例

现象：

```text
systemctl status app
→ active (running)

Java PID
→ 存在

8082请求失败

FD
→ 99%

CLOSE_WAIT
→ 30000+
```

分析：

```text
进程存在
≠
服务可用
```

排查过程：

```text
现象
↓
Java服务请求失败
↓
查看FD使用量
↓
发现接近限制
↓
查看lsof
↓
发现大量socket
↓
查看ss
↓
发现CLOSE_WAIT持续增长
↓
判断Java没有及时释放Socket
↓
定位代码资源关闭问题
```

---

## 八、今天核心总结

### 1.

```text
Too many open files
```

不代表磁盘文件太多，而是：

```text
进程FD资源耗尽
```

---

### 2.

```text
systemctl active(running)
```

只能说明：

```text
进程仍然存在
```

不能说明：

```text
服务一定可用
```

---

### 3.

FD排障链路：

```text
错误现象
↓
FD耗尽假设
↓
查看限制
↓
查看使用量
↓
查看FD类型
↓
分析Socket状态
↓
区分容量不足和资源泄漏
↓
修复根因
```

---

## 九、今日训练问题复盘

1. 为什么 CPU、内存、磁盘正常，Java 仍然可能无法建立连接？

因为 FD 是独立资源，FD耗尽不会直接表现为 CPU、内存异常。

2. 为什么不能直接用 `ulimit -n` 判断 Java 限制？

因为 systemd 启动的 Java 与 SSH Shell 属于不同进程环境。

3. FD接近上限能证明什么？

能证明资源接近耗尽，但不能直接证明一定存在代码泄漏。

4. socket很多是否一定代表泄漏？

不是，需要结合 TCP 状态、业务量和增长趋势判断。

5. 为什么进程存在但服务不可用？

因为进程生命周期和业务健康状态不是同一个概念。
