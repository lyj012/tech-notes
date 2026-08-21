# 任务：修复 Clash TUN 与公司 VPN 的内网访问冲突

我的电脑需要同时使用：

- Clash Verge Rev，开启 TUN 模式，用于普通公网代理
- 公司 / 客户 VPN，用于访问对方内网
- 内网中可能存在 GitLab、MySQL、Redis、Web 服务等资源

当前目标是：

> 普通公网流量继续走 Clash，公司内网流量必须绕过 Clash，并通过公司 VPN 访问。

请直接检查当前电脑的网络、Clash 和 VPN 配置，并在确认原因后完成修改。

---

## 一、当前目标资源

内网资源信息：

```text
内网 IP：<填写，例如 10.10.20.30>
端口：<填写，例如 3306>
域名：<例如 db.example.internal；没有则留空>
预期用途：<例如 MySQL / GitLab / Redis / Web 服务>
VPN：<填写 VPN 名称；如果不知道写未知>
```

如果我只提供了一个具体 IP，不要自行假设整个私有地址段都属于公司内网。

优先通过当前 VPN 路由表判断实际内网 CIDR。

---

## 二、首先诊断，不要直接修改

先判断当前操作系统和网络环境，并检查：

1. Clash Verge Rev 是否运行
2. TUN 模式是否开启
3. 公司 VPN 是否已经连接
4. 当前系统路由表
5. 目标 IP 实际经过哪个网卡
6. VPN 建立了哪些私有网段路由
7. Clash 是否抢占了目标内网流量
8. 如果使用域名，检查域名解析结果
9. 如果使用浏览器访问，再检查系统 HTTP/HTTPS Proxy
10. 如果是 JDBC、MySQL、Redis 等 TCP 客户端，不要把 HTTP 系统代理当成主要问题

必要时分别测试：

```text
Clash 开 + VPN 开
Clash 关 + VPN 开
Clash 开 + VPN 关
```

重点确认：

> 如果 VPN 开启、Clash 关闭时目标可以访问，而 Clash TUN 开启后无法访问，则优先判断为 Clash TUN 与 VPN 路由冲突。

---

## 三、测试目标连通性

根据操作系统选择合适命令。

### 通用 TCP 测试

```bash
ping <目标IP>
nc -vz <目标IP> <端口>
```

### Windows

```powershell
Test-NetConnection <目标IP> -Port <端口>
```

### macOS

```bash
route get <目标IP>
netstat -rn
```

### Linux

```bash
ip route get <目标IP>
ip route
```

### Windows 路由

```powershell
route print
Get-NetRoute
```

需要明确告诉我：

```text
目标 IP
→ 当前走哪个接口
→ 是否经过 Clash TUN
→ 是否应该经过 VPN
```

---

## 四、确定应该 DIRECT 的最小内网网段

优先从 VPN 路由表中找到目标 IP 所属的实际内网 CIDR。

例如：

```text
目标：
10.10.20.30

如果 VPN 实际路由为：
10.10.20.0/24

则使用：
10.10.20.0/24
```

不要因为目标属于私有 IP，就直接扩大成：

```text
10.0.0.0/8
```

原则：

> 使用能够覆盖目标资源的最小合理网段，避免影响其他网络。

如果无法可靠判断 CIDR，可以先只针对单个 IP 使用：

```text
10.10.20.30/32
```

不要猜测。

---

## 五、修改 Clash Verge Rev 配置

首先找到当前实际生效的 Clash Verge Rev 配置、Merge 或 Override 文件。

不要假设固定配置路径。

修改之前：

1. 备份原配置
2. 保留所有现有配置
3. 只追加必要规则
4. 不覆盖用户原有代理规则

针对内网 IP 添加最高优先级 DIRECT，例如：

```yaml
prepend-rules:
  - IP-CIDR,<实际内网CIDR>,DIRECT,no-resolve
```

例如：

```yaml
prepend-rules:
  - IP-CIDR,10.10.20.0/24,DIRECT,no-resolve
```

如果只允许单个目标 IP：

```yaml
prepend-rules:
  - IP-CIDR,10.10.20.30/32,DIRECT,no-resolve
```

如果配置中已经存在 `prepend-rules`：

> 合并进去，不要创建重复 YAML key。

---

## 六、如果存在公司内网域名，再处理 DNS

只有实际需要通过类似以下域名访问时才处理：

```text
db.example.internal
git.example.internal
service.example.internal
```

首先确认：

```text
域名当前解析成什么 IP
VPN DNS 能否正确解析
Clash DNS 是否返回 fake-ip
```

如果存在明确的域名与内网 IP 映射，可以考虑 hosts，例如：

```text
10.10.20.30 db.example.internal
```

Clash DNS 根据当前实际配置考虑：

```yaml
dns:
  use-hosts: true
  fake-ip-filter:
    - '*.example.internal'
```

注意：

- 保留现有 `dns` 配置
- 只合并必要字段
- 不重复创建 `dns`
- 不随意关闭 Clash DNS
- 不随意关闭 fake-ip 模式

如果 VPN 自带 DNS 已经可以正确解析内部域名，优先判断是否可以直接使用 VPN DNS，不要无必要写死 hosts。

---

## 七、浏览器和数据库客户端区别处理

### Web / GitLab

如果目标通过浏览器访问，例如：

```text
https://git.example.internal
```

除了 Clash DIRECT，还需要检查系统 HTTP/HTTPS Proxy。

必要时将：

```text
*.example.internal
```

加入系统代理 Bypass。

修改前必须保留原有 bypass 内容。

---

### MySQL / JDBC / Redis / SSH

例如：

```text
jdbc:mysql://10.10.20.30:3306/example_db
```

或者：

```text
10.10.20.30:6379
10.10.20.30:22
```

这些属于 TCP 连接。

主要排查：

```text
VPN
↓
系统路由
↓
Clash TUN
↓
目标 TCP 服务
```

不要因为浏览器方案中存在 System Proxy Bypass，就机械修改系统 HTTP Proxy。

---

## 八、修改后验证

修改完成后重新加载 Clash 配置。

必要时重启 Clash Verge Rev，但不要无意义修改 VPN。

再次验证：

```text
目标域名解析
目标 IP 路由
目标端口连通性
实际业务连接
```

最终需要满足：

```text
普通公网
    ↓
Clash
    ↓
代理节点

公司内网
    ↓
DIRECT
    ↓
系统路由
    ↓
公司 VPN
    ↓
内网目标
```

目标内网流量不得被发送到公网代理服务器。

---

## 九、安全要求

执行过程中严格遵守：

1. 修改任何配置前先备份
2. 不删除已有 Clash 规则
3. 不覆盖整个 Clash 配置文件
4. 不随意使用过大的私有网段规则
5. 不确定 CIDR 时优先使用单 IP `/32`
6. 不修改与本问题无关的网络配置
7. 不关闭防火墙
8. 不关闭安全软件
9. 不修改数据库账号密码
10. 不修改公司 VPN 服务端配置
11. 不把内网流量发送给公网代理节点
12. 每一项修改都说明原因
13. 不在日志、提交记录或输出中泄露数据库密码、VPN 密钥、Token、Cookie 等凭据

---

## 十、最终输出

处理完成后给出一份简洁报告：

```text
问题原因：
实际 VPN 内网网段：
目标 IP：
目标端口：
修改的 Clash 配置：
修改的 DNS / hosts：
修改的系统代理：
修改前路由：
修改后路由：
连通性测试：
最终结果：
备份文件位置：
```

如果没有完全修复，不要猜测成功。

明确告诉我当前卡在哪一层：

```text
DNS
Clash 规则
TUN 路由
VPN 路由
TCP 端口
服务端防火墙
数据库权限
数据库服务本身
```
