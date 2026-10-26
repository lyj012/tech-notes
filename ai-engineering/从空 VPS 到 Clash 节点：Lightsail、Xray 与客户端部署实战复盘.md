# 从空 VPS 到 Clash 节点：Lightsail、Xray 与客户端部署实战复盘

> **实践日期**：2026-10-26  
> **环境**：AWS Lightsail（东京）/ Ubuntu 24.04.4 LTS / Xray / VLESS + REALITY / Mihomo / Clash Verge / Clash Meta for Android  
> **定位**：基于一次真实部署的技术复盘，侧重部署链路、验证证据与故障处理，不是可直接复制执行的一键安装教程。
>
> 本文依据当次部署中实际执行的命令、服务器回读、电脑配置文件核对和连接测试整理。代码块中的密钥、UUID、Short ID、公网地址等已使用占位符；不能把占位符直接当作可用节点。
>
> 整理文档期间仅对现有状态做只读核对，没有重新安装服务、切换代理或升级套餐。历史测试结果仅对应测试当时，不构成长期在线或稳定性保证。
>
> **公开发布提醒**：仅发布脱敏文档，不应提交实际 PEM 私钥、服务端配置原件、未脱敏 YAML、VLESS 分享链接、完整认证参数或调试日志中的令牌。添加图片、附件或修改历史前同样需要检查敏感信息。

### 结果摘要

| 维度 | 本次证据支持的结果 | 边界 |
|---|---|---|
| 服务端 | Xray 已安装，`systemd` 设为开机启动，TCP 443 监听 | 服务运行不等于任何网络路径都可达 |
| 客户端 | Windows 与 Android 均完成配置，真实 HTTP/HTTPS 请求测试成功 | 未覆盖全部 App、UDP 和长期稳定性 |
| 故障 | 实例曾出现健康检查失败和 OOM 日志；人工重启后恢复 | OOM 不能被直接认定为整次失联的唯一原因 |
| 资源调整 | 启用 2 GiB Swap，已核对当前激活状态及开机配置 | 未在添加 Swap 后再次重启验收；512 MB 物理内存未改变 |

**阅读路线**：第 1～10 节介绍服务器与网络层；第 11～15 节说明客户端配置和验收；第 16～21 节记录故障、Swap 与未完成事项；第 22～24 节提炼可复用方法和参考资料。

**成本边界**：本次实例创建时选择 Lightsail 公网 IPv4 的 5 美元/月套餐，不能将云服务抵扣金等同于“永久免费”。停机、流量超额、其他附加资源和抵扣金到期等情形的计费风险，仍应以 AWS 实际账单与现行条款为准。


## 1. 部署原理：为什么没有自行开发代理程序，也能完成接入？

整个部署并非只创建一台 AWS VPS 和一份 YAML 文件；还包含服务端安装、协议配置、进程托管、防火墙放行、客户端导入与端到端验证。

实际完成的是：

1. 使用授权的 SSH 私钥登录 Ubuntu 服务器。
2. 下载已经编译好的 Xray 程序，校验下载文件并安装。
3. 给 Xray 配置协议、监听端口、允许的客户端和 REALITY 参数。
4. 创建 systemd 服务，让 Xray 在后台运行并开机自动启动。
5. 在 AWS Lightsail 的云防火墙中放行 TCP 443。
6. 生成与服务端参数对应的 Clash YAML。
7. 用独立的 Mihomo 内核验证节点实际能访问外网。
8. 在电脑的 Clash Verge 中创建本地配置、填入实际内容并选中使用。
9. 由使用者将配置导入手机的 Clash Meta，随后通过只读检查与真实请求验证手机连接。
10. 后续排查服务器失联问题，并在取得授权后增加 2 GiB Swap。

**这里主要做的是软件部署、配置和验证，而不是从零开发一个代理程序。**

可以类比 Java 项目：部署已经构建好的 JAR、填写配置并启动服务，不等于重新编写 JAR 的源码。

本次使用的是已经编译好的 Linux 可执行文件 `xray`。它不是 JAR，但“下载现成程序 → 配置 → 启动服务”的思路类似。

YAML 文件本身不能创造服务器，也不能转发流量。如果 VPS 上没有对应的服务，导入 YAML 最多只是让 Clash 显示一个节点名字。

## 2. 这次涉及的东西，各自负责什么？

| 名称 | 可以怎样理解 | 本次实际作用 |
|---|---|---|
| AWS Lightsail | 云服务器产品 | 提供东京区域的一台 Ubuntu 虚拟机 |
| VPS | 按需租用的远程计算实例 | 运行 Xray，并作为访问目标网站的出口 |
| Ubuntu | VPS 的操作系统 | 管理网络、内存、文件、进程和服务 |
| SSH | 加密的远程管理通道 | 从 Windows 终端登录 VPS 并执行管理命令 |
| PEM 私钥 | SSH 登录凭据 | 供客户端向 VPS 证明登录身份；必须妥善保管 |
| Xray | 服务器上的代理程序 | 接收客户端连接并转发网络请求 |
| VLESS | 客户端与服务端约定的代理协议 | 描述代理连接以及客户端身份等 |
| REALITY | 这条连接使用的传输安全机制 | 配合密钥和握手参数验证连接 |
| XTLS Vision | 本次配置的流控模式 | 服务端和客户端均配置 `xtls-rprx-vision` |
| Clash Verge | Windows 上的客户端界面 | 管理配置、选择节点、控制系统代理等 |
| Mihomo | Clash Verge 内部的代理内核 | 实际解析配置、建立连接和转发本地请求 |
| Clash Meta for Android | 手机客户端 | 通过 Android VPN 接口接管手机流量 |
| JSON | 本次服务端配置格式 | 告诉 Xray 如何运行 |
| YAML | 本次客户端配置格式 | 告诉 Clash 如何连接和选择路由 |
| systemd | Linux 的服务管理器 | 让 Xray 在后台运行、开机启动、失败后重启 |
| Swap | 磁盘上的交换空间 | 在内存紧张时提供缓冲，不等于真实 RAM |

“VPN”在日常交流中常被用作通俗称呼。严格区分，本次部署的是基于 Xray 的代理服务；手机通过 Android VPN 接口接管流量，并不代表服务器安装了 OpenVPN 或 WireGuard。

## 3. 整条网络链路到底怎么走？

### 3.1 电脑使用系统代理时

```text
电脑上遵循系统代理设置的应用
        ↓
Windows 系统代理：127.0.0.1:7897
        ↓
Clash Verge 内部的 Mihomo
        ↓ 读取 YAML、选择 Lightsail-Tokyo
VLESS + REALITY，经 TCP 连接 VPS 的 443 端口
        ↓
东京 VPS 上持续运行的 Xray
        ↓
Google / GitHub / ChatGPT 等目标服务
        ↓
响应沿上述连接返回
```

这里有两个完全不同的端口：

- `7897`：电脑本机的代理入口。
- `443`：VPS 对外提供服务的入口。

电脑上的浏览器先把请求交给本机 Mihomo；Mihomo 再连接远程 VPS。不是浏览器直接去访问 VPS 上的 YAML 文件。

`127.0.0.1` 是“当前这台设备自己”。电脑上的 `127.0.0.1:7897` 与手机上的 `127.0.0.1:7897` 不是同一个地方。

### 3.2 手机使用 Clash Meta 时

```text
手机应用
   ↓
Android VPN 接口 / Clash Meta 的流量接管与路由
   ↓
手机上的代理内核
   ↓
VLESS + REALITY → 东京 VPS:443 → Xray → 目标网站
```

电脑和手机都可以各自直接连接 VPS，不要求电脑必须开机，也不要求手机必须插着电脑。

当时的手机核对结果显示：USB 功能为 `adb`，底层联网方式是 Wi-Fi，没有证据表明手机通过 USB 借用了电脑的代理。

### 3.3 SSH 在哪里？

SSH 是另一条管理通道：

```text
Windows 上的 ssh 客户端 → VPS:22 → Ubuntu 的 sshd
```

平时上网不需要保持这个 SSH 窗口开着。SSH 断开以后，只要 Xray 的 systemd 服务还在运行，代理就仍可工作。

反过来，如果 VPS 整机不可达，SSH 和代理可能一起失效。因此故障排查时同时检查 TCP 22 与 TCP 443。

## 4. 一开始拿到的服务器是什么状态？

部署前已取得实例公网地址、SSH 用户名 `ubuntu`，以及 Windows 上对应的 Lightsail PEM 私钥文件。

首次成功登录后，实际检查结果如下：

- 操作系统：Ubuntu 24.04.4 LTS。
- 架构：x86_64，因此选择 Linux 64 位的 Xray 发布包。
- 当时服务器刚启动不久，没有现成的网站或代理业务服务。
- SSH 在 TCP 22 监听。
- TCP 443 空闲，没有与已有网站争抢端口。
- 系统可见内存约 412 MiB，Lightsail 套餐标称 512 MB。
- 根磁盘约 19 GiB，可用空间约 17 GiB。
- 当时没有启用 Swap。
- Ubuntu 的 UFW 未启用。

系统可见内存少于套餐标称值，不应直接解读为“少买了内存”；一部分资源会用于系统、保留区域等。

当时用过的检查命令主要是：

```bash
uname -a
cat /etc/os-release
uptime
sudo ss -lntup
systemctl --type=service --state=running --no-pager
sudo ufw status
df -h /
free -m
```

各命令的意义：

| 命令 | 主要回答的问题 |
|---|---|
| `uname -a` | 内核、架构是什么？ |
| `cat /etc/os-release` | Linux 发行版和版本是什么？ |
| `uptime` | 开机多久，负载如何？ |
| `ss -lntup` | 哪些程序监听了哪些 TCP/UDP 端口？ |
| `systemctl ...` | 当前运行哪些服务？ |
| `ufw status` | 主机上是否有 UFW 规则？ |
| `df -h /` | 系统盘剩多少空间？ |
| `free -m` | 内存、缓存、Swap 分别有多少？ |

这一步属于部署前调查，避免未经检查就覆盖已有配置或占用已有端口。

## 5. SSH 登录中实际遇到过什么问题？

第一次 SSH 连接没有直接成功。Windows OpenSSH 提示私钥文件权限过宽，拒绝使用它，类似：

```text
UNPROTECTED PRIVATE KEY FILE!
Permissions ... are too open.
This private key will be ignored.
```

这不是 Ubuntu 用户名错误，也不是必须换一把私钥。问题出在 Windows 上的密钥文件访问权限。

实际处理方式是：

1. 保持原始 PEM 文件内容不变。
2. 复制一个用于本次操作的临时副本。
3. 取消这个副本的权限继承，仅允许当前 Windows 用户读取。
4. 用这个权限收紧后的副本重新 SSH 登录。

副本最终放在 Windows 临时目录下的本次任务文件夹里。原计划放在工作目录，但实际写文件遇到了限制，因此调整了位置。

**没有上传 SSH 私钥给 VPS，也没有把 SSH 私钥写入 Clash YAML。** SSH 客户端使用私钥完成身份验证，与服务器通信；不需要把私钥本身当作配置发送过去。

下面只是登录命令的脱敏结构：

```powershell
ssh -i '<本机私钥副本路径>' `
  -o BatchMode=yes `
  -o ConnectTimeout=15 `
  ubuntu@<VPS_PUBLIC_IPV4>
```

最初连接时还使用了 `StrictHostKeyChecking=accept-new`，接受并记录这个地址首次出现的主机密钥。它与“SSH 登录私钥权限”是两个不同问题；本次没有另外通过 AWS 控制台核对主机指纹。

## 6. VPS 上安装的到底是什么？

### 6.1 安装了现成的 Xray 二进制

实际使用的发布包是：

```text
项目：XTLS/Xray-core
版本：v26.10.21
文件：Xray-linux-64.zip
```

下载来源是项目的 GitHub Release，同时下载了对应的 `.dgst` 校验文件。

实际记录的 ZIP 文件 SHA-256 为：

```text
0eec3ed4163c273d8c8bf821f7d219e257921cf584aa3fb970715e458e631978
```

本次用 Python 标准库计算 ZIP 的 SHA-256，与下载的 `.dgst` 中 `SHA2-256` 值比较，一致后才提取 `xray` 文件。

该校验说明下载文件与同一发布来源提供的摘要一致，不等于已经对项目开展独立安全审计。

以下为执行过的命令片段整理，不应在已经完成部署的实例上不加判断地重复执行：

```bash
curl -fL -o Xray-linux-64.zip \
  https://github.com/XTLS/Xray-core/releases/download/v26.10.21/Xray-linux-64.zip

curl -fsSL -o Xray-linux-64.zip.dgst \
  https://github.com/XTLS/Xray-core/releases/download/v26.10.21/Xray-linux-64.zip.dgst

sha256sum Xray-linux-64.zip
```

提取后安装到：

```text
/usr/local/bin/xray
```

使用的安装操作是：

```bash
sudo install -m 755 unpacked/xray /usr/local/bin/xray
```

其中 `755` 允许该程序被执行。接着用以下命令核对版本：

```bash
/usr/local/bin/xray version
```

本次没有安装可视化代理管理面板，也没有编写一个新的代理服务器。主要依赖官方预编译程序、配置文件和 systemd。

### 6.2 写过脚本，但脚本不是代理服务本身

执行过程中使用过 PowerShell、Bash、Python 和少量 Node.js：

- PowerShell：调用 SSH、处理本地路径和 Windows 文件权限。
- Bash：在 Ubuntu 上执行部署与检查命令。
- Python：校验下载、生成配置、生成随机身份参数、编辑 JSON。
- Node.js：处理本地文件、比较配置内容、保存截图等。

它们相当于“安装和检查的工具”。真正长期处理流量的是 `xray`，以及客户端内部的 Mihomo。

有些远程脚本使用 Base64 编码后通过 SSH 传输，是为了解决 Windows 换行、引号和管道问题。Base64 是编码，不是加密；脚本传输的保密性依赖 SSH。

## 7. 服务端怎么知道哪些客户端能连？

### 7.1 生成了三组关键参数

| 参数 | 如何产生 | 放在哪里 | 用途 |
|---|---|---|---|
| 客户端 UUID | Python `uuid.uuid4()` | 服务端 clients 和客户端 YAML | 标识允许连接的客户端身份 |
| REALITY 密钥对 | `xray x25519` | 私钥在服务端，对应值在客户端 | REALITY 握手验证所需 |
| Short ID | Python `secrets.token_hex(8)` | 服务端 shortIds 和客户端 short-id | REALITY 握手中的识别参数 |

本次 8 个随机字节生成了 16 个十六进制字符的 Short ID。

实际执行过：

```bash
/usr/local/bin/xray x25519
```

这个版本的输出字段是：

```text
PrivateKey: <服务端私钥>
Password (PublicKey): <客户端需要的对应值>
Hash32: <另一个派生输出，本次未用于客户端配置>
```

这里实际遇到过一个小问题：最初脚本按照其他字段名称提取公钥值，触发断言失败。随后只查看输出字段名称，确认当前版本使用 `Password (PublicKey)`，调整脚本后才继续生成配置。

**断言失败后停止了该步骤，没有误判为部署成功；私钥原文没有被写入学习文档。**

Xray 的文档在较新版本中把客户端对应参数称为 `password`；Mihomo 配置仍使用 `reality-opts.public-key`。它们需要对应，不是凭字段名字相似就任意填写。[REALITY 参数文档](https://xtls.github.io/en/config/transports/reality.html)

### 7.2 这些参数为什么要保密？

不要把“公钥”两个字理解成“这个节点的全部配置都可以公开”。在本次 REALITY 配置中，UUID、公钥参数、Short ID 等一起组成可用的连接资料。

为了让本文适合发给网页版聊天继续学习，所有示例都已脱敏。原始 YAML 和 VLESS 分享链接仍应自行保管。

SSH PEM 私钥更不能上传给别人，它具有管理服务器的用途；它和代理客户端 UUID 的权限范围不一样。

## 8. 服务端最终配置长什么样？

实际配置位置：

```text
/etc/xray/config.json
```

下面依据文档编写时重新回读的配置整理，隐藏身份和密钥参数；结构与实际部署对应：

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "tag": "vless-reality",
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<CLIENT_UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "raw",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "target": "www.icloud.com:443",
          "xver": 0,
          "serverNames": ["www.icloud.com"],
          "privateKey": "<SERVER_REALITY_PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID>"],
          "minClientVer": "1.8.2"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": [
          "10.0.0.0/8",
          "172.16.0.0/12",
          "192.168.0.0/16",
          "127.0.0.0/8",
          "169.254.0.0/16",
          "::1/128",
          "fc00::/7",
          "fe80::/10"
        ],
        "outboundTag": "block"
      }
    ]
  }
}
```

### 8.1 逐块解释

- `inbounds`：Xray 从哪里接收代理连接。
- `listen: 0.0.0.0`：允许从 IPv4 网络接口接收连接，仍受云防火墙等控制。
- `port: 443`：远程客户端连接的 TCP 端口。
- `protocol: vless`：服务端和客户端要使用同一个代理协议。
- `clients[].id`：这里允许的 UUID 必须与客户端相同。
- `flow`：本次双方都填写 `xtls-rprx-vision`。
- `decryption: none`：VLESS 的该字段为 none，不表示整条连接是明文；外层安全机制由 `security: reality` 指定。
- `network: raw`：本次 Xray 服务端采用基于 TCP 的 RAW 传输；客户端字段写作 `network: tcp`。
- `target`、`serverNames`：REALITY 握手相关的目标和允许的服务器名称。
- `privateKey`：REALITY 服务端私钥，只放在服务端。
- `shortIds`：服务端接受的 Short ID 列表。
- `minClientVer`：这里显式设置了兼容条件，后文说明实际原因。
- `outbounds`：通过验证的连接出站时使用的处理方式。
- `freedom`：从 VPS 直接访问目标服务。
- `blackhole`：丢弃命中的流量。
- `routing`：阻止代理去访问列出的环回、私网和链路本地地址段。这不是一套完整的全网访问控制策略。

### 8.2 `www.icloud.com` 在配置中的作用

不是。

正常通过认证后，请求由 VPS 上的 Xray 转发至实际目标网站。这里的域名服务于 REALITY 握手与回落行为，不代表所有代理流量都发送至该网站。

本次先测试过 `www.microsoft.com`，后来更换为 `www.icloud.com`，使用 `xray tls ping` 检查过 TLS 握手。Xray 对这两个选择都输出过目标选择风险警告。

所以不能把本文的目标域名理解成“已经完成最佳线路和安全选择评估”。本次验证的是连接可用，没有完成 REALITY 目标选择的全面优化，也没有验证长期可达性。

还要注意：REALITY 对不合法连接的回落行为与普通代理请求不同；此配置未额外加入回落限速或前置 SNI 过滤层。相关行为应结合官方文档理解，不应把它宣传成“绝对不会被探测”。[REALITY 目标与回落说明](https://xtls.github.io/en/config/transports/reality.html)

## 9. 为什么 SSH 退出后代理还能继续运行？

因为创建了一个 systemd 服务，而不是只在 SSH 窗口里手工运行一下程序。

实际服务文件：

```text
/etc/systemd/system/xray.service
```

实际内容：

```ini
[Unit]
Description=Xray VLESS REALITY proxy
After=network-online.target
Wants=network-online.target

[Service]
User=xray
Group=xray
ExecStart=/usr/local/bin/xray run -config /etc/xray/config.json
Restart=on-failure
RestartSec=5
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

本次创建了专用的系统用户 `xray`，不让长期运行的代理进程直接使用 root 身份。

配置目录权限为 root 所有、xray 组可访问；JSON 文件权限为 `640 root:xray`。这样服务可以读取配置，但不是任意普通用户都能读到服务端私钥。

`CAP_NET_BIND_SERVICE` 允许这个非 root 进程绑定 443 这类低端口。几个保护项用于限制服务对系统文件、用户目录和临时目录的访问。

实际使用过的管理命令：

```bash
# 先验证配置能否被 Xray 接受
sudo -u xray /usr/local/bin/xray run -test -config /etc/xray/config.json

# 让 systemd 读入新增的服务定义
sudo systemctl daemon-reload

# 设置开机启动，并立即启动
sudo systemctl enable --now xray

# 检查服务和监听状态
systemctl is-active xray
systemctl is-enabled xray
sudo ss -lntp 'sport = :443'
```

这里必须区分三个状态：

| 检查结果 | 能证明什么 | 不能证明什么 |
|---|---|---|
| `Configuration OK` | 配置能解析并通过程序检查 | 不能证明公网端口可达 |
| `active`、443 正在监听 | 进程正在运行并监听端口 | 不能证明云防火墙已放行 |
| 客户端实际代理请求成功 | 当时端到端链路可用 | 不能证明所有应用、UDP 或长期稳定性 |

后来服务器重启后，Xray 自动恢复运行，就是开机启动配置发挥了作用。

## 10. 为什么服务器上明明监听 443，外面还是连不上？

这次真实发生过：

- VPS 本机显示 Xray 已监听 443。
- Ubuntu UFW 没有拦截。
- 后续读取 iptables/ip6tables 的默认策略也没有发现对应主机侧拦截。
- 但电脑连接公网 TCP 443 超时。

随后由使用者在 Lightsail 控制台手动添加：

```text
IPv4 firewall
协议：TCP
端口：443
来源：所有 IPv4 地址
保留原来的 SSH TCP 22 规则
```

规则放行后，重新测试得到 `TCP443 reachable`。

**这条 AWS 云防火墙规则由使用者在控制台手动添加，不属于通过 SSH 在 Ubuntu 内执行的操作。** 登录 Ubuntu 并不自动取得修改 AWS 控制台网络策略的权限。

可以把这几层理解为：

```text
远程客户端
   ↓
AWS 云防火墙是否允许进入？
   ↓
Ubuntu 本机防火墙是否允许进入？
   ↓
443 上是否真的有程序监听？
   ↓
程序是否接受这个客户端的握手与身份参数？
```

任何一层失败，节点都可能显示超时或不可用。

本次实际用于代理的入口是 IPv4 TCP 443。虽然实例还有 IPv6 地址，但没有完成一条独立的 IPv6 代理部署和验收链路。

## 11. 客户端 YAML 是怎么生成的？

### 11.1 它是“服务端参数的对应说明”

生成服务端 JSON 时，程序已经拿到了 UUID、REALITY 密钥对应值和 Short ID。

同一次配置生成过程中，再把客户端需要的值填进 YAML；并不是凭空猜出一份文件，也不是从 AWS 自动获取一个现成的 Clash 节点。

本次最终交付的 YAML，脱敏后如下：

```yaml
mixed-port: 7897
allow-lan: false
mode: rule
log-level: info
ipv6: false

proxies:
  - name: "Lightsail-Tokyo"
    type: vless
    server: <VPS_PUBLIC_IPV4>
    port: 443
    uuid: <CLIENT_UUID>
    network: tcp
    udp: true
    tls: true
    flow: xtls-rprx-vision
    servername: www.icloud.com
    client-fingerprint: chrome
    reality-opts:
      support-x25519mlkem768: true
      public-key: <REALITY_CLIENT_PUBLIC_KEY>
      short-id: "<SHORT_ID>"

proxy-groups:
  - name: "PROXY"
    type: select
    proxies:
      - "Lightsail-Tokyo"
      - DIRECT

rules:
  - IP-CIDR,127.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,10.0.0.0/8,DIRECT,no-resolve
  - IP-CIDR,172.16.0.0/12,DIRECT,no-resolve
  - IP-CIDR,192.168.0.0/16,DIRECT,no-resolve
  - MATCH,PROXY
```

这份文件有三个核心部分：

1. `proxies`：定义真正的远程节点。
2. `proxy-groups`：定义可以选择节点的策略组。
3. `rules`：决定请求交给哪个策略或节点。

`DIRECT` 是直接连接，不是第二台 VPS。只有一个 `Lightsail-Tokyo` 节点，再加一个 DIRECT 选项，不代表拥有两台代理服务器。

### 11.2 最重要的参数对应关系

| 客户端 YAML | 服务端 JSON / 实际状态 | 必须满足的关系 |
|---|---|---|
| `server` | VPS 公网地址 | 地址要指向这台 VPS |
| `port: 443` | `inbounds[].port: 443` | 端口一致，且云防火墙放行 |
| `type: vless` | `protocol: vless` | 协议一致 |
| `uuid` | `settings.clients[].id` | 完全相同 |
| `flow` | 客户端允许项的 `flow` | 本次双方均为 Vision |
| `network: tcp` | `streamSettings.network: raw` | 本次兼容的 TCP/RAW 传输组合 |
| `tls: true` 和 `reality-opts` | `security: reality` | 使用对应的 REALITY 安全连接 |
| `servername` | `realitySettings.serverNames` | 必须是服务端允许的名称 |
| `public-key` | 服务端 `privateKey` 的对应值 | 是一对参数，不能随便更换 |
| `short-id` | `shortIds` 中的一个值 | 客户端值必须被服务端接受 |

官方的字段定义可以参考 [Mihomo VLESS 配置](https://wiki.metacubex.one/config/proxies/vless/) 和 [Mihomo TLS / REALITY 配置](https://wiki.metacubex.one/config/proxies/tls/)。

### 11.3 为什么只改 YAML 的 server 不一定能换一台服务器？

因为新的服务器必须也安装并运行兼容程序，而且要有匹配的 UUID、REALITY 密钥、Short ID、端口和名称等。

地址正确只是第一关。其余参数不一致时，常见现象是 TCP 端口能连接，但 REALITY 认证失败。

### 11.4 这份 YAML 没有做什么？

- 没有建立公开的订阅网站。
- 没有部署订阅转换服务。
- 没有给每台手机单独签发一份身份。
- 没有自动把服务器 IP 变化同步到所有设备。
- 没有完整的国内外域名分流规则集。
- 没有仅凭 `udp: true` 就证明 UDP 游戏、QUIC 等业务正常。

本次规则只有部分本地/私网地址直连，其余交给 PROXY。主机名解析、DNS 请求如何出站，还受客户端 DNS、系统代理或 TUN 配置影响，不能把“规则经过代理”扩大为“已经验证所有 DNS 都走代理”。

## 12. 为什么第一次端口通了，代理请求仍然失败？

这次遇到的真实报错是：

```text
REALITY authentication failed
```

端口可达后，初次用 Mihomo 连接仍失败。对服务端密钥派生值、Short ID 及客户端服务器名称进行核对后，未发现这些参数不一致。

之后做了两项实际兼容调整：

### 12.1 服务端显式配置 minClientVer

最终服务端包含：

```json
"minClientVer": "1.8.2"
```

当时查询到 Mihomo 项目中的兼容性问题：部分新版 Xray 默认的最低客户端版本条件，与 Mihomo REALITY 握手中报告的版本不匹配。

本次先加入这个字段，重新测试仍未成功。因此不能写成“只加 minClientVer 就解决了全部问题”。[项目中的相关问题记录](https://github.com/MetaCubeX/mihomo/issues/2967)

这是为兼容客户端而显式调整服务端版本接受条件；它不是修复服务器物理内存，也不是增加带宽。选择允许旧客户端条件存在兼容与协议演进的取舍，不能把它当成所有环境都必须套用的通用默认值。

### 12.2 客户端启用相应密钥交换支持

最终 YAML 又加入：

```yaml
reality-opts:
  support-x25519mlkem768: true
```

本次调查了 REALITY 实现中的握手检查，并核对了 Mihomo 的相关配置字段。加入这一客户端选项、重启独立测试内核后，真实代理请求才成功。

**能够证实的是：最终这组配置与本次安装的软件版本搭配，通过了实际测试。** 没有对每一种参数组合做完整兼容矩阵，所以不要把这两个字段概括成对所有 Xray/Mihomo 版本都充分或必要。

本次电脑上的内核版本实际核对为：

```text
Mihomo Meta v1.19.32 windows amd64
```

本次手机上的 Clash Meta 版本核对为：

```text
2.11.31.Meta
```

后续照教程操作时，应核对自己真实使用的内核版本和当前文档，不能只看客户端图标是不是叫 Clash。

## 13. 实际怎么接入电脑 Clash Verge？这里需要特别如实说明

### 13.1 最初确实还没有接入

最初完成了：

- 服务端安装与配置。
- YAML 和 VLESS 分享链接生成。
- 独立测试内核的实际代理请求测试。
- 将客户端文件保存到约定的交付目录。

但这一阶段并未将配置导入正在运行的 Clash Verge，也未切换当前配置。因此客户端界面中暂时看不到新节点，与实际完成范围一致。

**文件生成成功与客户端导入完成是两个独立环节。** 初次交付仅完成服务端和配置文件验证，不能表述为客户端接入已全部完成。

### 13.2 后来创建了本地配置卡片

在追加客户端集成任务后，实际打开 Windows 上的 Clash Verge：

```text
订阅 → 新建 → 类型选择“本地”
名称填写“我的 VPS · 东京”
选择配置文件 → 保存
```

这次界面写的是“订阅”，但本地配置也在这个页面管理。它并不意味着这份文件就是一个在线订阅。

### 13.3 导入过程并不是一次顺畅完成

文件选择窗口的自动操作不稳定。界面虽已出现“我的 VPS · 东京”卡片，但随后的文件核对发现：新建配置尚未包含实际的 `Lightsail-Tokyo` 节点内容。

随后做了下面的补救：

1. 找到刚刚由 Clash Verge 创建的那一个本地配置文件。
2. 在临时目录备份它当时的内容。
3. 将已经测试通过的 YAML 写入这一个新建配置文件。
4. 保留已有订阅及它们的文件，没有覆盖原有机场订阅。

本次新建的配置文件采用客户端自动生成的标识符；为便于公开发布，下面改用占位符表示：

```text
<PROFILE_ID>.yaml
```

对应文件位于 Clash Verge 的用户数据目录下（路径中的 `<PROFILE_ID>` 为脱敏占位符）：

```text
%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles\<PROFILE_ID>.yaml
```

这里的 `%APPDATA%` 是 Windows 用户应用数据目录。配置文件的名字由客户端生成，不需要与交付文件的 `lightsail-clash.yaml` 相同。

**上述步骤保留了导入不顺畅及补救过程，不能简化为“一次点击完成全部接入”。**

正常手动操作时，优先使用客户端自己的导入界面；在不了解文件格式和运行状态时，不要随意直接编辑 `profiles.yaml` 或覆盖其他配置文件。

### 13.4 Clash 怎样记住自己当前在用哪份配置？

Clash Verge 的另一个文件是：

```text
%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles.yaml
```

它保存配置索引，包括每份配置的内部标识、名称和文件名等。

文档编写时重新只读核对到了：

```yaml
current: <PROFILE_ID>
```

并且核对了内部文件与交付文件的完整字节内容：

```text
Imported profile equals delivered YAML = true
```

这比仅在界面看到配置名称更有意义：说明实际导入文件确实包含目标节点配置，且当时选中的配置标识对应这一份文件。

在收到停止自动切换的要求后，自动化操作已停止；使用者后续自主选择东京配置。复盘仅记录能够回读或明确确认的步骤，不将人工操作归入自动化执行范围。

### 13.5 为什么有时文件里的 mode 是 rule，界面却是全局？

客户端可以通过自身设置或运行时覆写改变实际模式。当时交付 YAML 中 `mode: rule`，而 Clash Verge 的运行配置为 `global`。

因此，判断实际行为不能只看原始 YAML：还要看客户端当时选择的运行模式和策略组选择。

- 规则模式：按 `rules` 决定路径。
- 全局模式：由 GLOBAL 的选择决定走哪个策略或节点。
- 直连模式：直接连接，不会因为 YAML 中有节点就自动使用它。

界面中的 PROXY 是策略组，Lightsail-Tokyo 才是远程节点；如果选择的是 PROXY，还要看这个组内部选中了什么。

### 13.6 系统代理和 TUN 也不是同一个开关

本次电脑检查过：

```text
系统代理开启
代理地址 127.0.0.1:7897
TUN 未开启
```

这个状态只代表当时的读数，不是永久设置保证。

系统代理主要影响遵循该设置的应用；不遵循系统代理的应用未必被接管。TUN 是另一种接管方式，本次没有完成 Windows TUN 模式的验收。

## 14. 为什么先启动“独立测试内核”？

由于 Windows 客户端原本已有可用订阅，直接覆盖当前配置可能影响现有联网。

本次先启动另一个 Mihomo 测试进程：

```text
平时 Clash 的入口：127.0.0.1:7897
独立测试入口：    127.0.0.1:17897
```

测试 YAML 的主要区别是把本地端口改成 17897，节点参数仍对应同一台 VPS。启动测试进程本身不等于把 Windows 系统代理改到 17897。

示意命令：

```powershell
& 'D:\Clash Verge\verge-mihomo.exe' -t `
  -d '<测试目录>' -f '<测试目录>\test.yaml'
```

`-t` 只验证配置语法。随后另外启动测试进程，再明确指定代理发送请求：

```powershell
curl.exe --proxy http://127.0.0.1:17897 `
  --connect-timeout 8 --max-time 15 `
  https://api.ipify.org

curl.exe --proxy http://127.0.0.1:17897 `
  --connect-timeout 8 --max-time 15 `
  -o NUL -w 'HTTP=%{http_code}\n' `
  https://www.google.com/generate_204
```

本次还使用了 `--noproxy ''`，避免环境中的 no-proxy 设置让明确要经过代理的请求绕开本地测试入口。

这些测试的目的：

- Google 返回 204：验证一次小的 HTTPS 请求完成。
- GitHub 返回 200：验证另一个站点的 HTTPS 请求。
- IP 查询返回 VPS 的公网 IPv4：确认该次请求确实从 VPS 出口出去。

测试结束后，停止的是本次自己启动的独立测试进程，没有用“关闭所有 Mihomo 进程”的方式误伤正在工作的客户端。

## 15. 手机接入时实际做了什么、没做什么？

### 15.1 手机端手动导入

本次手机为 Redmi K70，客户端为 Clash Meta for Android；YAML 由使用者手动导入。手机端不需要安装服务端 Xray，也不需要导入 SSH PEM 私钥。

概念上需要完成：

```text
文件传到手机 → Clash Meta 导入本地文件 → 激活配置
→ 选择正确节点 → 启动 Android VPN 连接
```

Android 的 VPN 授权是允许客户端接管本机网络，不是给这个 App 授权管理 AWS 服务器。

### 15.2 通过 ADB 进行只读检查

手机通过 USB 数据线连接电脑后，`adb devices -l` 显示设备为 `device`，表明 ADB 连接已建立并获得授权。

随后检查过：

- 当前前台应用。
- Clash Meta 的包名与版本。
- Android VPN 是否建立及底层 Wi-Fi 网络。
- Clash 的相关日志。
- 手机上的当前节点画面。
- 从手机发出的真实 HTTP/HTTPS 请求。

读取到手机已选择 `Lightsail-Tokyo`，日志有 REALITY 认证成功记录，出口 IP 与 VPS 一致。

没有通过 ADB 给手机 root，没有安装额外 CA 证书，也没有修改手机账号或清空聊天记录。

### 15.3 为什么 3000 ms 不能直接等同于“所有请求都要三秒”？

测速卡片确实曾显示 3000 ms，这是当时的界面读数，需要与真实请求耗时分别讨论。

但同一轮调查中，手机实际请求结果包括：

| 测试 | 当时实测 |
|---|---|
| 手机明确使用 Clash 本地代理访问 Google，连续 4 次 | 约 0.45～0.97 秒，均返回 204 |
| 手机正常 VPN 路径访问 Google | 约 0.727 秒，返回 204 |
| 手机正常 VPN 路径访问 gstatic 测速地址 | 约 0.578 秒，返回 204 |
| 手机正常 VPN 路径访问 IP 查询 | 约 0.921 秒，返回 200 |
| 上述部分请求的 DNS 时间 | 约 32～41 毫秒 |

这些是具体 HTTP 请求的完成时间，不是纯网络 RTT，也不是下载带宽测试。

因此，当时能得出的结论是：**测速卡片的 3000 ms 与那一轮真实请求不一致。**

没有查清该 3000 ms 数值的精确来源，也没有证据证明它一定由缓存、固定超时值或客户端显示缺陷导致；本次没有人为改写测试数字，也没有宣称该问题已永久修复。

### 15.4 ChatGPT 的 403 又是什么？

用命令行从手机经代理请求 ChatGPT 网页时，曾看到：

```text
HTTP 403
Cf-Mitigated: challenge
Server: cloudflare
```

这说明该命令行请求收到验证挑战。它不能单独证明整个节点失效，也不能证明手机 App 的先前错误一定由同一原因造成。

后续手机 App 当时未再次显示同类错误，因此未能完整复现该问题。既不能据此认定账号被封，也不能宣称 ChatGPT 访问问题已经彻底修复。

## 16. 后来为什么 VPS 节点和 SSH 都不能用了？

### 16.1 当时的证据链

1. 电脑连接 VPS TCP 22 超时。
2. 连接 TCP 443 同样超时。
3. 电脑通过原有订阅访问其他站点仍能成功。
4. Lightsail 截图显示实例仍为“正在运行”，公网 IP 未变，但实例健康检查失败。
5. AWS 浏览器 SSH 也显示 `UPSTREAM_ERROR [515]`。

这时故障范围已经超出“某一个 Clash 节点字段填错”。但这些证据仍不能单独区分服务器内部问题、宿主机问题或所有网络故障类型。

排查过程中没有仅因超时就更换 SSH 私钥，也没有将其直接归因于服务器 IP 被封锁。

### 16.2 谁执行了恢复？

**实例重启由使用者在 AWS 控制台手动执行。** 当时自动化浏览器无法可靠识别控制台页面，因此停止了自动操作，没有代为点击重启。

SSH 恢复后，再次连接服务器并核对：

- Xray 为 active、enabled。
- 443 重新处于监听状态。
- 独立节点测试成功。
- 出口仍是原 VPS 公网 IPv4。

### 16.3 日志确实发现了 OOM

重启后读取较早启动周期的内核日志，发现北京时间 14:09 左右有：

```text
Out of memory: Killed process ... (fwupd) ...
```

具体记录中，被杀进程为 `fwupd`，匿名常驻内存约 145656 KiB，即约 142 MiB。

这能确认：

- 那个时刻系统真实发生过内存耗尽。
- 内核采取了结束进程的处理。
- 当时服务器没有 Swap，缺少这层磁盘缓冲。

它不能确认：

- Xray 是内存耗尽的制造者。
- 从手机测速到服务器失联的所有问题都只有这个原因。
- VPS 宿主机绝对没有问题。
- 加 Swap 后永远不会再断联。

AWS 也区分系统状态检查和实例状态检查。前者与底层基础设施有关，后者可能包括实例内存或网络配置等问题；本次仅凭截图没有完成故障时两种检查的独立回读。[AWS 状态检查说明](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-troubleshooting-instances.html)

## 17. 后来添加 2 GiB Swap，是怎么做的？

### 17.1 这不是升级真实 RAM

完成后的资源关系是：

```text
Lightsail 套餐：仍为标称 512 MB RAM
操作系统可见 RAM：约 412 MiB
新增磁盘交换空间：2 GiB Swap
```

没有升级付费套餐，没有把这台虚拟机变成 2 GB 真实内存实例。

Swap 把部分暂时不用的内存页放到磁盘，给内存紧张提供余地。它比真实内存慢；如果长期大量换入换出，系统仍可能卡顿。

### 17.2 修改前做过的检查与备份

当时确认：

- `swapon --show` 没有现有条目。
- `/swapfile` 不存在。
- 根文件系统是 ext4。
- 剩余磁盘超过所需空间。
- Xray 在运行。

实际备份目录是：

```text
/var/backups/codex-swap-20261026T071329Z/
```

目录名使用 UTC 时间；这里对应北京时间 15:13:29。

保存了：

```text
fstab.before
memory.before.txt
swap.before.txt
```

### 17.3 实际核心操作

以下是实际操作的整理。**现有服务器已经完成，不要照抄再创建一遍。**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

对应含义：

- `fallocate`：分配一个 2 GiB 的文件。
- `chmod 600`：只允许文件所有者读取和写入；本次为 root。
- `mkswap`：写入 Swap 文件格式。
- `swapon`：立即启用。

随后在 `/etc/fstab` 增加：

```fstab
/swapfile none swap sw,nofail 0 0
```

它用于让系统开机识别并启用这个交换文件。`nofail` 的用途是在这个条目无法激活时避免把它当作必须成功的启动依赖；它不是忽略所有系统故障。

### 17.4 如何确认成功？

实际执行了：

```bash
findmnt --verify --tab-file /etc/fstab
sudo systemctl daemon-reload
swapon --show --bytes
systemctl show swapfile.swap -p LoadState -p ActiveState -p FragmentPath
stat -c '%a %U:%G %s %n' /swapfile
```

检查结果包括：

```text
LoadState=loaded
ActiveState=active
FragmentPath=/run/systemd/generator/swapfile.swap
600 root:root 2147483648 /swapfile
```

`swapon --show --bytes` 的可用 Swap 大小略小于文件大小，是文件格式本身需要空间等原因造成的，不代表只分配了一半。

校验 fstab 时曾输出两条警告，但没有解析错误或错误项；其中一条提醒 systemd 仍使用旧版本，随后执行了 daemon-reload。读取到系统已生成并激活对应 swap 单元，说明配置已经被 systemd 识别。

本次没有调整 `vm.swappiness`，保留了读取到的默认值 60，没有顺便优化或停用 fwupd 等其他系统服务。

没有为验证 Swap 而重启服务器；已经验证的是当前激活与开机配置生成，尚未通过添加 Swap 后的下一次真实重启来再次确认。

### 17.5 修改后有没有再测代理？

有。独立测试进程得到：

```text
出口 IP：与 VPS 一致
Google：HTTP 204，约 0.52 秒
GitHub：HTTP 200，约 1.05 秒
```

服务保持运行，没有切换 Windows 当前代理，也没有升级套餐。

文档编写时再次回读，Swap 仍为 active，并已有一小部分空间被使用。这说明系统确实可以使用它，不代表当前又发生了 OOM。

## 18. 现在服务器和电脑上分别有哪些关键文件？

### 18.1 VPS 上

| 路径 | 内容 |
|---|---|
| `/usr/local/bin/xray` | 真正执行的代理程序 |
| `/etc/xray/config.json` | 服务端配置，含服务端私钥等 |
| `/etc/systemd/system/xray.service` | 后台服务与开机启动定义 |
| `/etc/xray/config.before-clash-compat.json` | 兼容性调整前保存的服务端配置副本 |
| `/home/ubuntu/vpn-setup/` | 本次下载包、提取文件、客户端文件等工作材料 |
| `/swapfile` | 新增的 2 GiB Swap 文件 |
| `/etc/fstab` | 包含 Swap 的开机激活条目 |
| `/var/backups/codex-swap-20261026T071329Z/` | 添加 Swap 前的备份与状态记录 |

Ubuntu 的 `/home/ubuntu/vpn-setup/` 与 Windows 的工作目录不是同一个目录，不能混用路径。

### 18.2 Windows 上

交付文件位于本次聊天工作区的 `outputs` 目录：

```text
lightsail-clash.yaml
lightsail-vless.txt
使用说明.txt
本学习文档
```

Clash Verge 自己另外保存了本地导入配置和索引，位置见第 13 节。

临时 SSH 副本、测试配置、测试日志与截图位于 Windows 临时任务目录。本次没有确认这些临时文件已完成清理；原始 PEM 保存在既有位置。

本次没有用 Git 提交这些配置，没有把实际密钥发布到公共仓库，也没有创建公开订阅链接。

## 19. 日常维护时，哪些检查最有价值？

### 19.1 第一步：不要急着改配置，先判断故障范围

```text
只是一个应用失败？
   ↓
多个网站都失败？
   ↓
电脑和手机同时失败？
   ↓
VPS 的 443 能连接吗？22 能连接吗？
   ↓
AWS 实例状态检查是否失败？
```

| 现象 | 优先检查 | 不宜直接下的结论 |
|---|---|---|
| YAML 语法检查失败 | 缩进、字段、客户端版本 | VPS 硬件坏了 |
| 443 TCP 超时 | IP、云防火墙、网络路径、实例状态、监听 | UUID 一定错了 |
| TCP 可达，但 REALITY 认证失败 | 密钥、Short ID、名称、版本兼容、握手配置 | 升级套餐就能修 |
| Google 可访问，某一个应用失败 | 那个应用的响应、登录状态、挑战及路由 | 整条代理必然断了 |
| 22 和 443 同时超时，AWS 状态检查失败 | 实例与基础设施状态 | 只需重装 Clash |
| 日志明确有 OOM | 系统内存压力、进程用量、Swap | 一定是 Xray 泄漏 |
| 只是测速卡片数值很高 | 重新测速与真实请求对照 | 所有业务都要等同样的时间 |

### 19.2 SSH 能连接时，可以读取的状态

```bash
systemctl is-active xray
systemctl is-enabled xray
sudo ss -lntp 'sport = :443'
sudo journalctl -u xray -n 30 --no-pager
free -h
swapon --show
df -h /
```

`free` 中要结合 `available` 理解内存余量，不能只看到 `free` 很小就认定马上耗尽；Linux 会使用内存做缓存。

Xray 内存也不是一个永远固定的数字。本次在不同阶段读到过约 9 MB、26 MB、33 MB 等不同量级，连接、缓存和统计口径都会影响结果。

### 19.3 需要看前一次启动记录时

```bash
sudo journalctl --list-boots --no-pager
sudo journalctl -b -1 -k --no-pager
```

`-1` 表示上一次启动周期。假如连续重启了两次，最初故障可能已经在 `-2`，不能机械地只读 `-1`。

本次就是查看启动列表后读取更早周期，才找到对应的 OOM 记录。启动日志是否保留还取决于系统日志配置。

### 19.4 哪些命令会真正改变运行状态？

```bash
sudo systemctl restart xray
sudo systemctl stop xray
sudo systemctl disable --now xray
```

这些不是只读检查。执行会影响正在使用该节点的设备，不能与 `status`、`is-active` 混为一谈。

不应仅为试探故障而反复重启服务或服务器；如果当前网络连接依赖该 VPS，重启操作可能直接中断访问。

## 20. 如果将来想增加真实内存，应该怎样理解？

本次已做的是 Swap，没有购买更多 RAM。

Lightsail 增加真实内存通常需要：先为原实例创建快照，再用快照创建更大规格的新实例，验证后迁移公网访问配置。

不能通过在 Ubuntu 中执行一条命令，就把实例可用的真实 RAM 从 512 MB 增加到 2 GB。

迁移时要考虑新实例的套餐费用、公网地址、云防火墙和客户端配置。停止再启动没有绑定静态 IP 的实例也可能改变动态公网 IPv4；不能假设这次重启没变，今后任何操作都不会变。

本次没有创建更大套餐实例，也没有购买或绑定静态 IP。[AWS 从快照创建更大实例说明](https://docs.aws.amazon.com/en_en/lightsail/latest/userguide/how-to-create-larger-instance-from-snapshot-using-console.html)

## 21. 这次真正完成了哪些验证，哪些仍然没有？

### 21.1 有直接证据的事项

- SSH 登录成功，并读取了真实 Ubuntu 状态。
- 下载的 Xray ZIP 与对应摘要一致。
- Xray 服务端配置校验通过。
- Xray 服务 active、enabled，443 监听。
- 人工放行 AWS 云防火墙后，公网 TCP 443 可达。
- 最终参数组合在电脑独立 Mihomo 测试中能够访问 Google、GitHub，出口与 VPS 一致。
- 电脑本地导入配置文件的内容，与交付 YAML 完整一致。
- 文档编写时 Clash Verge 的配置索引指向“我的 VPS · 东京”对应文件。
- 手机 Clash Meta 建立 VPN，日志出现 REALITY 认证成功，并有手机路径下的实际请求成功。
- 故障后确实发现过 OOM 日志。
- 人工重启后，SSH、代理恢复，并重新完成独立连接测试。
- 2 GiB Swap 已启用，fstab 和 systemd 生成单元已检查。

### 21.2 未完成或不能据此保证的事项

- 没有验证长期在线稳定性或做全天监控。
- 没有完整复现并修复手机先前那次 ChatGPT App 错误。
- 没有确定测速卡片始终显示 3000 ms 的精确原因。
- 没有完成所有客户端版本的兼容性测试。
- 没有完成 UDP、QUIC、游戏流量和 Windows TUN 的端到端验收。
- 没有测试多用户并发、最大吞吐或不同运营商线路。
- 没有证明 OOM 是整次失联的唯一原因。
- 没有进行目标站点访问策略、DNS 泄漏、全设备流量覆盖等全面审计。
- 没有在添加 Swap 后通过一次真实重启重新验收持久化。
- 没有升级真实 RAM，也没有完成完整的灾备迁移方案。

这些边界不是说“部署没有用”，而是避免把“几个请求成功”扩大解释成“任何设备、任何应用、任何时间都保证正常”。

## 22. 可复用的工程方法

### 22.1 先分清四种东西

```text
资源：VPS、RAM、磁盘、网络
程序：Xray、Mihomo
配置：服务端 JSON、客户端 YAML
运行状态：服务是否在跑、端口是否通、实际请求是否成功
```

只买 VPS，没有代理程序；只生成 YAML，没有服务器；只看到服务 active，没有完整网络验证。这四层要连起来理解。

### 22.2 服务端和客户端必须遵守同一份连接约定

地址、端口、协议、身份、密钥和握手条件都要对应。它很像接口联调：双方配置与行为一致，连接才可能成功。

### 22.3 YAML 是可读的数据，不是“神奇的连接程序”

YAML 的作用是让已有程序知道“连谁、怎么连、哪些请求走这条路”。解析和执行它的，是 Mihomo。

服务端 JSON 同理；不是 JSON 自己监听 443，而是 Xray 根据 JSON 去监听。

### 22.4 自动化不等于没有操作，也不等于没有错误

即使没有长时间操作代码编辑器，部署工作仍可能在终端、远程命令和配置生成过程中完成。

自动化也会遇到私钥权限、换行、文件保存、导入窗口、字段名称和版本兼容等问题。重要的是失败后核对事实并纠正，而不是仅凭“命令运行过”宣称结果完成。

### 22.5 运行证据比节点名称更有价值

记住这条验证链：

```text
配置能解析
  → 程序能启动
  → 端口在监听
  → 公网能到达
  → 握手成功
  → 代理请求成功
  → 出口确实是预期 VPS
```

每一步只能覆盖一部分问题。同样的分层验证方法也适用于服务器、数据库、后端接口等工程场景。

## 23. 附录：基于本记录的学习提示词

下面的提示词不含实际连接凭据，可用于按主题继续学习；该附录不是部署验收证据：

> 请面向初次接触云服务器代理的学习者，基于本次真实操作记录开展教学。不要把历史测试结果当成永久在线保证，也不要索取原始 SSH 私钥、未脱敏 YAML 或 VLESS 分享链接。
>
> 请先用简单语言解释 VPS、SSH、Xray、Mihomo 和 Clash Verge 各自的角色，再按数据流说明浏览器请求如何经过本机 7897、远程 443 和 Xray 到达目标网站。
>
> 然后逐项讲解文档中的服务端 JSON、客户端 YAML 和 systemd 单元，尤其说明 UUID、REALITY 密钥、Short ID、serverName、TCP/RAW、Vision、minClientVer 和 support-x25519mlkem768 的关系，并区分本次版本兼容修复与通用协议要求。
>
> 最后解释怎样区分配置错误、端口不通、握手失败、单站点验证挑战、实例状态检查失败和内存耗尽。先讲概念和只读检查，不要直接修改正在使用的服务器。
>
> 每次只讲一个主题，说明“本次实际操作”“为什么这样做”“能证明什么”“还不能证明什么”，最后提出一道简短理解题。

## 24. 参考资料

这些是本次核对概念或排查时参考的项目/厂商资料。本文中的实际执行记录以本次工具输出和文件回读为准；官方资料只用于说明参数和产品行为。

- [Xray-core 官方项目与发布](https://github.com/XTLS/Xray-core/releases/tag/v26.10.21)：本次程序来源和固定版本。
- [Project X：REALITY](https://xtls.github.io/en/config/transports/reality.html)：服务端与客户端参数、target 和回落行为。
- [Mihomo：VLESS 配置](https://wiki.metacubex.one/config/proxies/vless/)：客户端 VLESS 字段。
- [Mihomo：TLS 与 REALITY 配置](https://wiki.metacubex.one/config/proxies/tls/)：public-key、short-id 和密钥交换选项。
- [Mihomo 项目问题记录 #2967](https://github.com/MetaCubeX/mihomo/issues/2967)：本次调查到的客户端版本兼容问题；issue 是具体问题报告，不是对所有版本的通用保证。
- [Clash Meta for Android 的代理页面实现](https://github.com/MetaCubeX/ClashMetaForAndroid/blob/main/app/src/main/java/com/github/kr328/clash/ProxyActivity.kt)：本次调查过界面测速如何触发健康检查；不能只凭这份源码推断本次 3000 ms 的确切成因。
- [AWS：常见实例问题与状态检查](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-troubleshooting-instances.html)：系统检查与实例检查的区别。
- [AWS：通过快照创建更大实例](https://docs.aws.amazon.com/en_en/lightsail/latest/userguide/how-to-create-larger-instance-from-snapshot-using-console.html)：增加真实 RAM 时的产品升级方式。

---

**本次实践完成的核心工作是：在 Ubuntu VPS 上部署现成的 Xray 服务，配置客户端与服务端匹配的 VLESS + REALITY 参数，并由电脑和手机上的 Clash/Mihomo 客户端读取配置建立连接。YAML 用于描述连接关系，实际网络转发由 Xray 与 Mihomo 执行。**