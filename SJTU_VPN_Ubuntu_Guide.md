# 上海交通大学 VPN：Ubuntu 配置与使用备忘

> **适用范围**：Ubuntu 24.04 LTS（其他 Ubuntu 版本可参考，但请核对软件包和服务名）；**学生 jAccount**；交大 **IKEv2/IPsec + EAP-PEAP** 接入。  
> **方案**：Ubuntu APT + **strongSwan (`charon-systemd`) + `swanctl`**，不使用 Shell alias、函数、第三方安装脚本或旧式 `ipsec.conf`。  
> **记录日期**：2026-11-06。**实际进度**：已在本次配置中成功执行 `swanctl --initiate --child vpn-student`，完成 VPN 连接。DNS、重启后恢复及 Clash 切换仍应按下方检查项确认。  
> **安全提醒**：本文使用占位符，**不包含真实 jAccount 用户名/密码**。请勿将含明文密码的真实 `/etc/swanctl/conf.d/sjtu.conf` 直接提交到 Git 或公开网盘。

## 0. 日常使用（配置完成后只看这里）

`charon-systemd` 服务已设置为开机启动；**服务运行 ≠ VPN 已连接**。日常只需以下原生命令：

| 操作 | 命令 |
| --- | --- |
| 连接校园 VPN | `sudo swanctl --initiate --child vpn-student` |
| 断开校园 VPN | `sudo swanctl --terminate --ike vpn-student` |
| 查看 VPN 状态和流量计数 | `sudo swanctl --list-sas` |
| 查看已加载的连接定义 | `sudo swanctl --list-conns` |
| 修改配置后重新加载 | `sudo swanctl --load-all` |
| 查看服务状态 | `systemctl status strongswan` |
| 查看最近日志 | `sudo journalctl -u strongswan -b -n 100 --no-pager` |

连接成功时通常能看到 IKE_SA 为 `ESTABLISHED`、CHILD_SA 为 `INSTALLED, TUNNEL`。`--list-sas` 中的流量计数可辅助判断是否确实在传输数据。

断开后再次执行 `sudo swanctl --list-sas`，确认对应 SA 已消失。一般**无需停止** `strongswan` 服务，更无需重启系统。

### 每次使用前后（与 Clash 共存）

- 如果 Clash/Mihomo 开启了 **TUN** 或 **System Proxy**，建议先关闭，再连接校园 VPN；用完断开 VPN 后再恢复 Clash。
- 安装 strongSwan 本身不会自动开启 VPN；但 IPsec 与 Clash TUN **同时运行**可能争用路由、DNS。这里采用“不同时使用”的简单策略，并**没有**声称已验证两者同时运行。
- 如果断开 VPN 后浏览器仍无法上网，检查 Clash 的系统代理是否残留（如 `env | grep -i proxy`；GNOME 可用 `gsettings get org.gnome.system.proxy mode`）。

## 1. 为什么选这套方案？

交大 Linux 指南提供的可确认协议是 **IKEv2/IPsec + EAP-PEAP**；客户端不能任意改用 WireGuard、OpenVPN 等不兼容协议。strongSwan 是成熟的 Linux IKEv2/IPsec 实现：

- **`charon-systemd`**：用户态 IKEv2 协商、证书/EAP 认证、密钥和连接生命周期管理，由 `systemd` 管理。
- **`swanctl`**：通过 VICI 接口加载配置、发起/终止连接、查看状态。
- **Linux XFRM/IPsec**：内核负责 IPsec 安全关联、策略匹配和数据包处理；**不必**出现 `tun0` 一类虚拟网卡。
- **`systemd-resolved`**：Ubuntu 常见的 DNS 管理组件，DNS 是否随 VPN 自动更新需实际检查。

学生网关：`stu.vpn.sjtu.edu.cn`；PEAP 内部认证服务器的预期身份：`@radius.net.sjtu.edu.cn`。

## 2. 一次性安装与配置

以下命令是**恢复/重装时参考**，**不要在已经配置成功的机器上不加区分地重新执行创建文件命令**。

### Step 1：安装软件包

```bash
sudo apt update
sudo apt install \
  charon-systemd \
  strongswan-swanctl \
  libcharon-extra-plugins \
  libcharon-extauth-plugins \
  libstrongswan-extra-plugins \
  ca-certificates
```

**作用**：安装 systemd 集成的 IKE daemon、`swanctl`、EAP-PEAP 等插件和系统 CA 证书集合。使用 APT 管理更新，不另装旧式 `strongswan-starter`。旧装机如已有 starter，先检查，避免多个 daemon 同时监听 IKE 端口。

### Step 2：配置可信 CA（我们实际执行过，且连接日志已验证有效）

```bash
sudo install -d -m 755 /etc/swanctl/x509ca
sudo ln -s -t /etc/swanctl/x509ca /etc/ssl/certs/*.pem
```

- `ca-certificates` 提供 Ubuntu 的受信任公共 CA 集合；以上操作建立**符号链接**，并非从外部下载未知 CA。
- `swanctl` 使用的标准 CA 目录是 **`/etc/swanctl/x509ca/`**；学校文档前面的通用说明使用 **`/etc/ipsec.d/cacerts/`**，主要对应另一套传统布局。不要认为这两个目录必然互通。
- **不是每次连接都要执行**；已存在同名链接时不应重复执行。更严格的证书最小信任配置可以将 CA 范围缩减到经学校核实的签发链。
- **不要关闭服务器证书验证**。本次成功连接的日志中，`stu.vpn.sjtu.edu.cn` 的链验证成功，`radius.net.sjtu.edu.cn` 的 PEAP TLS 证书链和吊销状态检查也通过。没有必要修改 `revocation.conf` 禁用检查。

### Step 3：创建单独的 VPN 配置文件

检查 `/etc/swanctl/swanctl.conf` 是否包含：

```text
include conf.d/*.conf
```

这行表示主配置会读取 `/etc/swanctl/conf.d/` 中的 `.conf` 文件。随后**首次创建**：

```bash
sudo install -m 600 /dev/null /etc/swanctl/conf.d/sjtu.conf
sudoedit /etc/swanctl/conf.d/sjtu.conf
```

> **重要**：`install ... /dev/null ...` 会创建/清空目标文件，**仅首次创建时执行**；以后修改文件只执行 `sudoedit`。权限 `600` 使只有 root 可读写，因为文件中有密码。

配置内容（**两个位置的 jAccount 必须全部替换**，且与密码凭据对应）：

```ini
connections {
    vpn-student {
        version = 2
        remote_addrs = stu.vpn.sjtu.edu.cn
        vips = 0.0.0.0, ::
        send_certreq = no
        mobike = no

        local {
            auth = eap-peap
            eap_id = YOUR_JACCOUNT
            aaa_id = @radius.net.sjtu.edu.cn
        }

        remote {
            auth = pubkey
            id = @stu.vpn.sjtu.edu.cn
        }

        children {
            vpn-student {
                remote_ts = 0.0.0.0/0, ::/0
            }
        }
    }
}

secrets {
    eap-jaccount {
        id = YOUR_JACCOUNT
        secret = "YOUR_PASSWORD"
    }
}
```

| 字段 | 含义 |
| --- | --- |
| `vpn-student` | 连接名和 CHILD_SA 名，供 CLI 精确引用 |
| `version = 2` | IKEv2 |
| `remote_addrs` | 交大学生 VPN 网关 |
| `vips` | 请求分配 IPv4、IPv6 虚拟地址 |
| `local.auth = eap-peap` | 学校要求的用户认证协议 |
| `eap_id` | jAccount 身份，**不是**字面量 `YOUR_JACCOUNT` |
| `aaa_id` | PEAP 内部认证服务器的预期身份 |
| `remote.auth` / `remote.id` | 使用公钥证书验证网关，并限定预期服务器身份 |
| `remote_ts` | 向网关请求的远端流量选择器；并不单独保证实际使用全隧道 |
| `send_certreq = no` | 不发送证书请求提示；**不是**关闭证书校验 |
| `mobike = no` | 按交大示例禁用 MOBIKE |
| `secrets.eap-jaccount` | 用于内部 EAP-MSCHAPv2 的本地密码凭据 |

未手动固定旧算法套件（`proposals`/`esp_proposals`），使用 strongSwan 的默认协商。若账号密码包含配置语言特殊字符，必须按 `swanctl.conf` 语法正确转义。

### Step 4：启动 systemd 服务

```bash
sudo systemctl enable --now strongswan
systemctl status strongswan
```

**作用**：令 `charon-systemd` 当前运行且开机自动启动。正常 systemd unit 启动时还执行 `swanctl --load-all --noprompt`，加载连接和凭据；但**后来编辑**的配置不会因此自动被重新读取。

### Step 5：显式加载并核验配置（这一步是本次排障重点）

```bash
sudo swanctl --load-all
sudo swanctl --list-conns
sudo swanctl --list-certs --type x509 --flag ca
```

- `--load-all` 加载 connections、credentials（包括 EAP secret）、authorities 和 pools；因为我们引入了很多系统 CA，它可能打印**大量** `loaded certificate` 信息，属正常现象。
- `--list-conns` 必须能看到 **`vpn-student`**。**仅仅将配置写入文件，不代表 daemon 已加载**：本次第一次 `--list-conns` 没输出，执行 `--load-all` 后才正常。
- `--list-certs` 查看已加载、符合筛选条件的 CA；文件存在和 daemon 已加载是两回事。

### Step 6：发起连接并验证

```bash
sudo swanctl --initiate --child vpn-student
sudo swanctl --list-sas
```

本次第一次连接失败，日志中出现：

```text
no EAP key found for hosts 'radius.net.sjtu.edu.cn' - 'YOUR_JACCOUNT'
EAP_MSCHAPV2 method failed
```

**根因**：配置中的用户名/密码占位符没有全部替换，造成 EAP 身份与 secret 不匹配。修复、重新 `--load-all` 后，再执行 `--initiate` **已连接成功**。这不是 CA 或 IKE 算法问题；当时服务器证书链和吊销状态检查已经通过。

连接建立后建议检查实际校园访问：

```bash
sudo swanctl --list-sas
resolvectl status
curl --noproxy '*' --max-time 10 http://whatismyip.sjtu.edu.cn
```

`curl --noproxy '*'` 可绕过 HTTP/SOCKS 代理环境变量，但**无法绕过仍在运行的 Clash TUN**。IP 是否变化并非判断 VPN 成功的唯一条件；更可靠的是访问一个确实需要校园网权限的资源，结合 SA 和流量计数判断。

### Step 7：断开

```bash
sudo swanctl --terminate --ike vpn-student
sudo swanctl --list-sas
```

确认相应 SA 消失。连接/断开不必停止 daemon；没有活动 VPN SA 时，后台服务运行不等于路由经过 VPN。

## 3. 常见问题与排查

| 现象 | 先检查什么 | 应对 |
| --- | --- | --- |
| `--list-conns` 没有输出 | 是否改完文件但没重新加载 | `sudo swanctl --load-all` 后再查看 |
| `--load-all` 刷屏显示很多证书 | 引入了系统 CA 集合 | 正常，重点找 `loaded connection 'vpn-student'` 及错误日志 |
| `no EAP key found` | `eap_id` 与 `secrets.eap-jaccount.id` 是否都正确、secret 是否已加载 | 修改后 `sudo swanctl --load-all`；不要直接归因密码错误 |
| `AUTHENTICATION_FAILED` | 用户身份、密码、PEAP 内部认证及证书日志 | 查看日志，逐项定位 |
| `no trusted certificate found` | `/etc/swanctl/x509ca/` 和加载结果 | 核对 CA、服务器证书及身份；不要跳过验证 |
| IKE 超时/IPv6 网络问题 | 基础网络、防火墙、IPv4/IPv6 | 学校提供 IPv4 专用学生网关 `stuv4.vpn.sjtu.edu.cn`，必要时调整 `remote_addrs` 并重载 |
| 连得上 IP，但域名打不开 | `resolvectl status`、`resolvectl dns` | 先确认 DNS 是否随 VPN 配置；不要默认永久修改 `/etc/resolv.conf` |
| `systemctl` 显示 active 但校园资源不可访问 | `sudo swanctl --list-sas` | daemon 活着 ≠ IKE/CHILD_SA 已建立 |
| 与 Clash TUN 同时运行时异常 | 路由、DNS 和代理设置 | 优先采用先关闭 Clash、再连 VPN 的简单流程 |

进一步诊断：

```bash
sudo journalctl -u strongswan -b -n 100 --no-pager
ip rule show
ip route show table 220
ip xfrm policy
resolvectl status
```

**注意**：policy-based IPsec 不一定创建 `tun0`；`table 220` 是否有记录取决于当前配置，不能单独作为成败依据。`ip xfrm state` 可能泄漏会话密钥等敏感材料，**不要公开粘贴完整输出**。

### DNS 处理原则

当前记录只确认 **VPN 连接成功**，尚不把“所有校内域名能解析”和“断开后 DNS 自动恢复”当作已实测结论。如果访问正常，**不增加任何 DNS 修改**。如确需手动 DNS，先记录原状态、明确网卡和恢复方式；优先考虑 `systemd-resolved` 的可逆配置，而不是永久改写系统 DNS。学校官方给出过校内 DNS `202.120.2.101` 和 `202.112.26.40`，但是否适用于你的网络应实测确认。

## 4. 重装系统时恢复与安全备份

1. 保存**这份 Markdown**（不包含密码），备份当前 VPN 配置的结构和你采用的参数。
2. 在新机器上重新执行 Step 1、Step 2；首次创建 Step 3 文件并填写**当前**的 jAccount 凭据。
3. 执行 `sudo systemctl enable --now strongswan`、`sudo swanctl --load-all`、`sudo swanctl --list-conns`，确认 `vpn-student` 后连接。
4. **切勿直接公开或提交真实 `sjtu.conf`**；如需单独备份实际配置，必须使用可信的加密存储并保护文件权限。系统 CA 可以通过 apt 重新建立，无需把整套 CA 链接目录拷贝进备份。
5. 修改密码后，更新 `secrets.eap-jaccount.secret` 并运行 `sudo swanctl --load-all`。

## 5. 官方与上游参考资料

1. [上海交通大学网络信息中心：Linux 系统交大 VPN 使用说明](https://net.sjtu.edu.cn/info/1200/3296.htm) — 校方 IKEv2/EAP-PEAP 参数、学生/教职工网关、DNS 排障。
2. [strongSwan：swanctl.conf](https://docs.strongswan.org/docs/latest/swanctl/swanctlConf.html) — `connections`、`children`、`secrets` 配置字段。
3. [strongSwan：swanctl Directory](https://docs.strongswan.org/docs/latest/swanctl/swanctlDir.html) — `conf.d`、`x509ca`、证书和凭据目录。
4. [strongSwan：swanctl Tool](https://docs.strongswan.org/docs/latest/swanctl/swanctl.html) — 连接、断开、状态等 CLI 命令入口。
5. [strongSwan：swanctl --load-all](https://docs.strongswan.org/docs/latest/swanctl/swanctlLoadAll.html) — 重新加载连接和凭据的准确作用。
6. [strongSwan：charon-systemd](https://docs.strongswan.org/docs/latest/daemons/charon-systemd.html) — systemd 服务及自动加载机制。
7. [strongSwan 官方 strongswan.service 定义](https://github.com/strongswan/strongswan/blob/master/init/systemd/strongswan.service.in) — `ExecStart` 与 `ExecStartPost`。

---

**一句话记忆**：正常日常操作只需要 `sudo swanctl --initiate --child vpn-student`、`sudo swanctl --list-sas`、`sudo swanctl --terminate --ike vpn-student`；**修改配置后记得 `sudo swanctl --load-all`**。

## 6. 与 Tailscale 共存：已验证的路由排除

场景：本机（Ubuntu 24.04，本文档的配置目标机）开启 SJTU VPN 的同时
运行 Tailscale；Mac 通过官方 GUI Tailscale 客户端经 SSH/ping 访问本机。
2026-11-08 已验证：本机连接 SJTU VPN 后，Mac 的 Tailscale 仍能发现本机、
MagicDNS 也能解析，但 VPN 默认路由（table 220）优先于 Tailscale
（table 52），导致本机发往 Mac（Tailscale 网段）的回应包被送入 VPN
隧道、走错路径，普通 ping 和 SSH 不通。仅 discovery/TSMP 成功不足以
证明 SSH 可用。

解决方式是从 VPN 路由表排除 Tailscale 的固定 IPv4/IPv6 地址段；不固定
某台机器 IP、设备网口或 UDP socket。已安装的唯一持久化补充文件为：

`/etc/systemd/system/strongswan.service.d/tailscale-routing.conf`

```ini
[Service]
ExecStartPost=/usr/sbin/ip route replace throw 100.64.0.0/10 table 220
ExecStartPost=/usr/sbin/ip -6 route replace throw fd7a:115c:a1e0::/48 table 220
```

它追加到原 strongSwan 服务启动命令之后；`throw` 让目标包继续查询后续
规则，由 Tailscale 表 52 处理。其他流量仍可走 VPN 默认路由。
配置已加载进 systemd，两条命令已在当前内核执行并验证。未实际重启机器验证。

日常连接、查看、断开仍使用第 0 节原有 swanctl 命令，不需要额外脚本、
定时器或每次手工重复添加路由。当前服务启用开机启动；VPN 本身仍显式连接。

本次结果（自 Mac 发起）：普通 IPv4 ping 10/10 回复，7.451–97.937 ms，
平均 32.066 ms；IPv4 与 IPv6 的普通 SSH 均能执行 hostname。延迟仍会抖动，
未达到稳定低于 30 ms。实测时 Mac 连接 SJTU Wi-Fi、自身 VPN 未连接，
使用官方 GUI Tailscale 客户端。

重装时可创建上述目录并用 sudoedit 写入此文件，然后执行：

```sh
sudo systemctl daemon-reload
sudo ip route replace throw 100.64.0.0/10 table 220
sudo ip -6 route replace throw fd7a:115c:a1e0::/48 table 220
```

若需撤销，请在本机终端先按原文档断开 VPN，再删除此补充配置和排除：

```sh
sudo swanctl --terminate --ike vpn-student
sudo rm /etc/systemd/system/strongswan.service.d/tailscale-routing.conf
sudo systemctl daemon-reload
sudo ip route del throw 100.64.0.0/10 table 220
sudo ip -6 route del throw fd7a:115c:a1e0::/48 table 220
```

不再使用测试定时器 sjtu-vpn-test-rollback；该 timer/service 已自动回收。

