# 后端工程化每日训练 Day 72：Linux 文件权限、用户身份与 Permission denied 排障复盘

## 一、今天学习什么

今天学习 Linux 线上排障中非常常见的一类问题：

**文件权限、用户/用户组，以及 `Permission denied`、`AccessDeniedException` 怎么排查。**

典型现象包括：

```text
root 手工启动 Java 正常
systemctl 启动失败

Java 进程正常
但是日志写不进去

配置文件明明存在
程序却提示 Permission denied

服务整体可用
但是导出、上传等局部功能失败
```

今天最核心的目标不是背 `chmod` 数字，而是建立这条固定思路：

```text
谁在访问
↓
访问什么
↓
走 owner / group / other 哪一组权限
↓
需要 r / w / x 中哪一个
↓
文件本身够不够
↓
父目录能不能穿过
↓
systemd 实际用什么用户运行
↓
只修真正缺失的权限
```

---

## 二、Linux 权限判断首先要知道“谁在访问”

假设 Java PID 是：

```text
12345
```

查看进程身份：

```bash
ps -o user,group,pid,cmd -p 12345
```

例如：

```text
USER     GROUP    PID
monitor  monitor  12345
```

那么真正访问文件的主体是：

```text
monitor 用户
```

不是：

```text
当前 SSH 登录用户
```

也不是：

```text
root
```

如果服务由 systemd 启动，还要查看：

```bash
systemctl cat app.service
```

重点：

```ini
[Service]
User=monitor
Group=monitor
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

因此权限排障第一问永远应该是：

> **到底是谁在访问这个文件或目录？**

---

## 三、owner、group、other 怎么判断

例如：

```bash
-rw-r----- 1 root app 2048 application.yml
```

拆开：

```text
-          → 普通文件

rw-        → owner 权限
r--        → group 权限
---        → other 权限

root       → owner
app        → group
2048       → 文件大小，不是用户 ID
```

假设 Java 用户是：

```text
monitor
```

执行：

```bash
id monitor
```

得到：

```text
uid=1001(monitor)
gid=1001(monitor)
groups=1001(monitor),1002(app)
```

这表示：

```text
monitor 属于 monitor 组
同时也属于 app 组
```

因此判断：

```text
monitor 是 owner root 吗？
→ 不是

monitor 属于 group app 吗？
→ 是

所以走 group 权限：
r--
```

最终：

```text
monitor 可以读取 application.yml
monitor 不能写 application.yml
```

今天训练中的一个易错点是：

```text
monitor UID=1001
app GID=1002
```

不能据此判断“monitor 和 app 没关系”。

真正要看的是：

```text
groups=...app
```

即用户是否属于该组。

---

## 四、r、w、x 在文件和目录上的含义不同

### 1. 普通文件

```text
r = read
w = write
x = execute
```

例如：

```text
application.yml
```

Java 读取配置需要：

```text
r
```

脚本直接执行：

```text
startup.sh
```

通常需要：

```text
x
```

---

### 2. 目录

目录上的权限含义不同：

```text
r
→ 能读取目录内容、列出名字

w
→ 能创建、删除、重命名目录项

x
→ 能进入 / 穿过目录，并继续访问里面的对象
```

其中最容易忽略的是：

```text
目录的 x
```

---

## 五、为什么文件 644 仍然可能 Permission denied

文件：

```text
/opt/app/config/application.yml
```

权限：

```bash
-rw-r--r--
```

也就是：

```text
644
```

单看文件：

```text
owner = rw-
group = r--
other = r--
```

理论上普通用户也有读取权限。

但是访问完整文件需要依次经过：

```text
/
↓
/opt
↓
/opt/app
↓
/opt/app/config
↓
application.yml
```

假设：

```bash
drwx------ root root /opt/app/config
```

那么 `monitor` 对该目录没有 `x`：

```text
不能穿过 config
↓
根本走不到 application.yml
↓
Permission denied
```

所以：

> **文件本身权限正常，不代表整条路径可访问。**

排查整条路径非常实用的命令：

```bash
namei -l /opt/app/config/application.yml
```

也可以逐层查看：

```bash
ls -ld /opt
ls -ld /opt/app
ls -ld /opt/app/config
```

---

## 六、为什么能进入目录，却不能创建文件

目录：

```bash
drwxr-xr-x root root logs
```

Java 用户：

```text
monitor
```

如果 `monitor` 不属于 root 组，则走：

```text
other = r-x
```

因此：

```text
r
→ 可以读取目录内容

x
→ 可以进入 / 穿过目录

w
→ 没有
```

结果：

```text
cd logs
→ 可以

ls logs
→ 可以

创建 app.log
→ 不可以
```

原因：

> **在目录中创建新文件，需要目录具有写权限，并且通常还需要 x。**

这里已有 `x`，真正缺的是：

```text
w
```

所以：

```text
能进入目录
≠
能在目录中创建文件
```

---

## 七、chmod 数字权限

常见数字：

```text
r = 4
w = 2
x = 1
```

因此：

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
```

例如：

```bash
chmod 640 application.yml
```

表示：

```text
owner = rw-
group = r--
other = ---
```

但线上排障重点不是背数字，而是：

> **根据实际访问需求给最小必要权限。**

---

## 八、为什么 chmod -R 777 很差

遇到：

```text
Permission denied
```

直接：

```bash
chmod -R 777 /opt/app
```

有时确实可能立即恢复服务。

但它只是说明：

```text
故障很可能与权限有关
```

不能说明真正根因已经修复。

### 1. 安全问题

```text
777
→ owner / group / other 全部 rwx
```

意味着大量本机用户都可能：

```text
读取
修改
执行
```

配置、脚本、JAR、敏感文件可能都被赋予过大的权限。

---

### 2. 影响范围问题

```text
-R = recursive
```

会递归修改整个目录树。

原本不同文件可能应该是：

```text
配置文件 → 640
JAR      → 640 / 644
脚本     → 750
日志目录 → 750 / 770
敏感文件 → 600
```

一个：

```text
chmod -R 777
```

会把原本合理的权限体系全部抹平。

---

### 3. 根因被掩盖

真正根因可能是：

```text
User=monitor 配错
文件 owner/group 错
monitor 没加入 app 组
父目录缺 x
日志目录缺 w
新发布 JAR 变成 root:root 600
```

直接 777 后现象消失，但根因没有被识别。

下次发布时还可能再次出现。

---

## 九、发布后 root:root 600 导致 systemd 启动失败

真实场景：

发布人员使用 root 上传：

```text
/opt/app/app.jar
```

新文件：

```bash
-rw------- root root app.jar
```

也就是：

```text
600 root:root
```

service：

```ini
[Service]
User=monitor
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

重启：

```bash
systemctl restart app
```

失败。

完整故障链：

```text
发布前服务正常
↓
root 上传新 JAR
↓
新文件 owner=root、group=root
↓
权限变成 600
↓
systemd 仍用 monitor 启动 Java
↓
monitor 不是 owner
↓
group / other 都没有 r
↓
monitor 无法读取 app.jar
↓
Java 无法启动
↓
systemd failed
```

这里需要特别区分：

```text
文件没有读取权限
```

和：

```text
父目录没有 x
```

这两个都可能造成 Permission denied，但证据不同。

本场景已知 `app.jar` 为 `600 root:root`，首先能直接确认的是：

> **monitor 没有读取 JAR 的权限。**

不能没有证据就直接说“目录进不去”。

---

## 十、完整权限排障模型

以后看到：

```text
Permission denied
AccessDeniedException
Unable to access jarfile
```

可以按下面顺序排查。

### 1. 现象

例如：

```text
systemctl restart app
→ failed
```

日志：

```text
Permission denied
```

---

### 2. 建立假设

```text
1. systemd 运行用户不对
2. 文件 owner/group 不对
3. 文件 rwx 不够
4. 父目录没有 x
5. 手工启动和 systemd 用户不同
```

---

### 3. 确认谁在运行

```bash
systemctl cat app.service
```

或者：

```bash
ps -o user,group,pid,cmd -p PID
```

---

### 4. 检查目标文件

```bash
ls -l /opt/app/app.jar
```

判断：

```text
进程是 owner？
属于 group？
还是走 other？
```

---

### 5. 检查用户组

```bash
id monitor
```

---

### 6. 检查完整路径

```bash
namei -l /opt/app/app.jar
```

特别检查父目录的：

```text
x
```

---

### 7. 找到真正缺失权限

最终应该定位成类似：

```text
monitor 对 app.jar 缺 r
```

或者：

```text
monitor 对 /opt/app/config 缺 x
```

或者：

```text
monitor 对 /opt/app/export 缺 w
```

---

### 8. 最小权限修复

目标不是：

```text
让所有用户都能 rwx
```

而是：

```text
正确用户
+
正确用户组
+
最小必要权限
```

---

### 9. 修复后验证

例如：

```bash
systemctl restart app
systemctl status app
journalctl -u app -n 100
ss -lntp
curl http://127.0.0.1:8082/...
```

不能只看到：

```text
进程起来
```

就结束。

还要确认：

```text
端口
接口
相关功能
日志
```

真正恢复。

---

## 十一、服务可用不代表所有权限都没问题

场景：

```text
systemctl status app
→ active

8082
→ 正常监听

普通业务接口
→ 正常返回
```

但日志持续：

```text
AccessDeniedException: /opt/app/export
```

这并不矛盾。

一个 Java 服务可能同时包含：

```text
查询接口
导入功能
导出功能
文件上传
日志写入
临时文件生成
配置读取
```

可能出现：

```text
启动服务
→ 不需要写 export
→ 正常

普通查询
→ 不需要写 export
→ 正常

导出报表
→ 需要写 /opt/app/export
→ 权限不足
→ 导出功能失败
```

因此：

```text
服务整体可运行
≠
所有功能完全正常
```

判断是整体故障还是局部故障，要继续问：

> **这个报错资源到底被哪个功能使用？**

例如堆栈：

```text
ExportService.exportExcel()
↓
Files.createFile()
↓
AccessDeniedException: /opt/app/export
```

再结合接口验证：

```text
普通查询接口 → 正常
设备接口     → 正常
导出接口     → 500
```

即可把范围缩小为：

```text
Java 服务整体可用
但是 export 局部功能失败
```

---

## 十二、sudo 为什么不能自动解决 systemd 服务权限问题

今天最后一个关键疑问是：

> “我平时没权限，不是直接 sudo 就行了吗？”

这里必须区分：

```text
执行命令的人临时提权
```

和：

```text
服务进程长期以什么用户运行
```

例如：

```bash
sudo systemctl start app
```

这里 `sudo` 提权的是：

```text
systemctl 这个管理命令
```

执行链：

```text
当前用户
↓ sudo
以 root 权限调用 systemctl
↓
systemd 收到“启动 app”命令
↓
读取 app.service
↓
发现 User=monitor
↓
最终仍以 monitor 启动 Java
```

所以：

```text
sudo systemctl start app
```

不等于：

```text
Java 进程以 root 运行
```

如果文件是：

```text
600 root:root
```

service：

```ini
User=monitor
```

那么：

```bash
sudo systemctl start app
```

仍然可能失败。

因为：

```text
systemctl 有权限“发出启动命令”
≠
monitor 有权限“读取 app.jar”
```

---

### 与 sudo java -jar 的区别

如果直接：

```bash
sudo java -jar app.jar
```

那么被 sudo 的就是：

```text
java 进程本身
```

所以：

```text
Java 直接以 root 身份运行
```

这也是为什么会出现：

```text
sudo / root 手工启动成功
但是 systemd 启动失败
```

根本原因通常不是：

```text
JAR 一定坏了
```

而要优先比较：

```text
启动用户
文件权限
用户组
工作目录
环境变量
路径
```

其中出现 `Permission denied` 时，用户身份和文件权限优先级最高。

---

## 十三、今天需要记住的命令

查看文件：

```bash
ls -l /path/to/file
```

查看目录本身：

```bash
ls -ld /path/to/dir
```

查看用户所属组：

```bash
id monitor
```

查看 Java 进程身份：

```bash
ps -o user,group,pid,cmd -p PID
```

查看 systemd 配置：

```bash
systemctl cat app.service
```

检查完整路径权限：

```bash
namei -l /opt/app/config/application.yml
```

查看服务日志：

```bash
journalctl -u app.service -n 100
```

---

## 十四、今日训练问题复盘

### 1. 为什么 root 手工启动成功，不能证明 systemd 一定能启动？

因为 root 和 systemd 的实际运行用户可能完全不同。

```text
root 手工启动成功
→ 只能证明 root 有权限

systemd User=monitor
→ 还必须单独验证 monitor 的权限
```

---

### 2. monitor 属于 app 组时，文件 root:app 640 能不能读取？

可以。

因为：

```text
monitor 不是 owner root
但是属于 group app
所以走 group 权限 r--
```

---

### 3. 文件 644 为什么仍然可能 Permission denied？

因为访问文件还需要穿过整条父目录路径。

父目录缺少 `x`：

```text
文件即使有 r
也可能无法访问
```

---

### 4. drwxr-xr-x 为什么能进入却不能创建文件？

普通用户走：

```text
other = r-x
```

有 `x` 可以进入，但没有 `w`，所以不能创建新的目录项。

---

### 5. 为什么 chmod -R 777 不是正确修复？

因为：

```text
安全范围过大
递归影响过大
真正根因被掩盖
```

---

### 6. root 上传新 JAR 后 systemd failed 的典型根因是什么？

如果：

```text
app.jar = 600 root:root
User=monitor
```

那么：

```text
monitor 无法读取 JAR
→ Java 无法启动
```

---

### 7. 为什么服务可用仍然可能存在权限问题？

因为：

```text
服务是多个功能组成的
```

某个局部功能可能单独依赖：

```text
上传目录
导出目录
日志目录
临时目录
```

因此：

```text
核心接口正常
≠
所有局部功能都正常
```

---

## 十五、今天最终形成的排障模型

看到：

```text
Permission denied
AccessDeniedException
```

以后不要第一反应：

```text
chmod 777
```

应该固定按下面这条链路：

```text
谁在访问？
↓
访问什么？
↓
是 owner / group / other 哪一类？
↓
需要 r / w / x 哪个权限？
↓
文件权限够不够？
↓
父目录有没有 x？
↓
systemd User= 谁？
↓
用户组是否正确？
↓
只修真正缺失的权限
↓
重新验证服务和对应功能
```

今天可以压缩成三句话：

> **先确认“谁”，再判断“权限”。**
>
> **文件能读，不代表路径能走；目录能进，不代表目录能写。**
>
> **sudo systemctl 只是让你有权管理服务，不代表 Java 服务本身会以 root 身份运行。**
