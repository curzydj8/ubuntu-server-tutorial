# 第 4 章 连接方式总览与具体步骤

> 服务器通常没有长期接显示器，日常运维全靠远程连接。本章先给一张全景对比表，再逐个讲八种连接方式：每种都给出「服务端安装命令 + 客户端连接步骤 + 常见报错」。先掌握 SSH（含密钥与转发），其余按需取用。

---

## 4.0 总览对比表

| 方式 | 协议 / 默认端口 | 是否图形界面 | 穿透内网能力 | 典型适用场景 |
|------|-----------------|--------------|--------------|--------------|
| SSH 密码/密钥登录 | SSH / TCP 22 | 否（命令行） | 无（需公网 IP 或转发） | 日常运维的主力，99% 的管理工作 |
| SSH 端口转发 | 随 SSH 连接 | — | 借 SSH 通道间接访问 | 安全访问内网 Web、数据库、VNC |
| ProxyJump 跳板机 | SSH / TCP 22 | 否 | 经一台公网机中转 | 批量管理内网机器 |
| Tailscale | WireGuard（UDP 41641 等）/ 虚拟网 100.x | 否（提供虚拟局域网） | **强**（NAT 穿透、组网） | 无公网 IP 时远程运维、异地组网 |
| VNC | RFB / TCP 5900+ | 是 | 无 | 远程图形桌面（建议只走 SSH 隧道） |
| xrdp | RDP / TCP 3389 | 是 | 无 | 用 Windows 远程桌面连 Linux 桌面 |
| NoMachine | NX / TCP 4000 | 是 | 无（企业版有云服务） | 流畅的远程桌面，音视频体验好 |
| 串口 / 云厂商网页控制台 | 串口 / HTTPS | 否（文本控制台） | 不依赖网络配置 | 系统起不来、网络改错时的最后救命通道 |

**选型建议（先记住这三条）：**

1. 日常运维只用 **SSH 密钥登录**，这是安全与效率的最优解。
2. 服务器在家里/内网、没有公网 IP：用 **Tailscale** 组网，之后在虚拟网里照常用 SSH。
3. 确实需要图形桌面：内网或隧道里用 **VNC/xrdp/NoMachine**，三者都不要直接暴露到公网。

---

## 4.1 SSH 密码登录与密钥登录

### 4.1.1 服务端：确认 OpenSSH 已安装并运行

第 1 章安装时已勾选 OpenSSH。若当时漏勾，补装：

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
systemctl status ssh
ss -tlnp | grep 22
```

### 4.1.2 客户端：密码登录

Windows 10/11、macOS、Linux 终端都自带 `ssh` 命令，用法一致：

```bash
ssh user1@<服务器IP>
# 示例
ssh user1@192.0.2.10
```

首次连接会显示服务器指纹并询问 `Are you sure you want to continue connecting (yes/no)?`，输入 `yes` 后输密码即可。指纹确认的意义：确认你连的确实是目标机器而不是冒充者；服务器重装系统后指纹会变，客户端会报警告（处理见 4.1.6 常见报错）。

指定端口登录（SSH 改过端口时）：

```bash
ssh -p 2222 user1@192.0.2.10
```

### 4.1.3 密钥登录全流程（强烈推荐）

密钥登录用一对密钥代替密码：私钥留在你电脑上，公钥放到服务器上。好处是：免输密码、可彻底关闭密码登录从根源杜绝爆破。

**第 1 步：在你自己的电脑上生成密钥对**

```bash
# -t ed25519：算法（现代首选）；一路回车即可，passphrase 可设可不设（设了更安全，连私钥被盗也需口令）
ssh-keygen -t ed25519 -C "user1-laptop"

# 生成两个文件（Linux/macOS 在 ~/.ssh/，Windows 在 C:\Users\<你>\.ssh\）
# id_ed25519      —— 私钥，绝不外传、不上传任何服务器
# id_ed25519.pub  —— 公钥，可以公开，要上传的就是它
```

**第 2 步：把公钥传到服务器**

```bash
# Linux/macOS 客户端（会让你输一次服务器密码，之后就不再需要）
ssh-copy-id user1@192.0.2.10

# Windows PowerShell 没有 ssh-copy-id 时的替代做法（把公钥内容追加到服务器）
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh user1@192.0.2.10 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**第 3 步：测试并检查权限**

```bash
# 应直接登录成功，不再问密码（问的是 passphrase 除外）
ssh user1@192.0.2.10
```

若还是要密码，多半是服务器端权限不对。SSH 对家目录与 `.ssh` 权限要求严格（权限太宽会被拒绝使用密钥）：

```bash
# 在服务器上执行
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R user1:user1 ~/.ssh
```

**第 4 步：客户端 `~/.ssh/config` 写别名**

在**客户端**的 `~/.ssh/config`（Windows 为 `C:\Users\<你>\.ssh\config`）写入：

```text
Host myserver
    HostName 192.0.2.10
    User user1
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

之后一条短命令即可登录：

```bash
ssh myserver
```

### 4.1.4 SSH 加固：改端口与关键配置

配置文件在服务器的 `/etc/ssh/sshd_config`，改前先备份。**改 SSH 配置的铁律：不要关掉当前连接，开第二个终端验证新配置能登录成功后，再关旧连接。**

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

常用加固项：

```text
# 改默认端口（减少自动化扫描与爆破噪音；改后防火墙要同步放行新端口）
Port 2222

# 禁止 root 直接登录（第 2 章已铺垫）
PermitRootLogin no

# 只允许密钥登录（确认密钥登录已成功后再关密码！）
PasswordAuthentication no

# 只允许指定用户登录（可选）
AllowUsers user1
```

改完检查语法并重启服务：

```bash
sudo sshd -t
sudo systemctl restart ssh
```

> 每项改了会怎样，逐个说清：
> - `Port`：只是换门牌号，能挡掉绝大多数无差别扫描，但挡不住定向攻击者扫全端口；价值在「清净」，不在「安全本身」。
> - `PermitRootLogin no`：root 无法远程登录，攻击者必须先拿下普通用户再提权，多一道门槛。
> - `PasswordAuthentication no`：**影响最大也最危险的一项**。关掉后密码再也登不进来，爆破彻底失效；但若你的密钥丢了/没配对，你自己也被锁在外面，只能走 4.8 节的控制台自救。所以顺序必须是：配好密钥 → 亲测密钥能登录 → 再改这一项 → 再开新终端复测。
> - 改了端口记得同步：`sudo ufw allow 2222/tcp`，并在云厂商安全组里放行新端口（云服务器有两层防火墙，漏一层都连不上）。

### 4.1.5 fail2ban 简介（防爆破）

fail2ban 监视 SSH 登录日志，同一 IP 多次失败就自动封禁一段时间。对仍开着密码登录的服务器很有价值；已只允许密钥登录的服务器，它是锦上添花。

```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban

# 查看 SSH 监狱状态：被封 IP 数、失败次数
sudo fail2ban-client status sshd

# 手动解封某个 IP（误封自己时用，需从控制台或其他网络操作）
sudo fail2ban-client set sshd unbanip 198.51.100.99
```

默认策略一般为失败 5 次封 10 分钟，在 `/etc/fail2ban/jail.local` 中可调，新手保持默认即可。

### 4.1.6 常见报错

| 报错 | 原因 | 处理 |
|------|------|------|
| `Connection refused` | SSH 服务没起、端口错、防火墙/安全组没放行 | 服务器端查 `systemctl status ssh`、`ss -tlnp`、ufw 与云安全组 |
| `Connection timed out` | IP 不通、路由/防火墙丢弃、服务器关机 | 先 `ping`，再查网络与防火墙（超时多是包被悄悄丢掉） |
| `Permission denied (publickey,password)` | 密码错、密钥没配对、权限过宽、已关密码登录 | 核对用户名密码；查 `.ssh` 权限（700/600）；`ssh -v` 看调试过程 |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED` | 服务器重装/换机导致指纹变化（也可能是中间人攻击，先核实） | 确认是自家机器重装后，在客户端执行 `ssh-keygen -R 192.0.2.10` 删旧指纹再重连 |
| `Host key verification failed` | 同上，未确认指纹 | 同上处理 |
| 能连但立刻断开 | 家目录权限、磁盘满、PAM/账户锁定 | 控制台登录查 `df -h`、`journalctl -u ssh` |

排障利器：客户端加 `-v`（可叠三层 `-vvv`）看握手全过程；服务端看日志 `sudo journalctl -u ssh -e`。

---

## 4.2 SSH 端口转发（隧道）

端口转发是把 SSH 连接当加密隧道，让你像在本地一样访问服务器内网的服务，而**无需把那些服务暴露到公网**。这是运维必备技能。

### 4.2.1 本地转发 -L：访问服务器内网的 Web 服务

场景：服务器内网有一台 `192.0.2.20` 跑着只对内网开放的管理页面（8080 端口），你想在自己电脑浏览器里打开它。

在**客户端**执行（保持这个终端开着）：

```bash
# 把本机 18080 端口 → 经服务器 → 转发到 192.0.2.20:8080
ssh -L 18080:192.0.2.20:8080 user1@192.0.2.10
```

然后浏览器访问 `http://localhost:18080`，看到的就是内网页面。目标也可以是服务器本机自身（如本机数据库 `localhost:3306`）：`ssh -L 13306:localhost:3306 user1@192.0.2.10`，再用数据库客户端连本机 13306 即可。加 `-N` 表示只转发不打开 shell，适合挂后台。

### 4.2.2 远程转发 -R：把内网服务临时给别人看

场景：你在内网机器上跑了个 8000 端口的测试网页，想让外网同事临时访问（经一台有公网 IP 的服务器中转）。

```bash
# 在内网机器上执行：把公网服务器的 18000 端口 → 转发回本机 8000
ssh -R 18000:localhost:8000 user1@198.51.100.10
```

同事访问 `http://198.51.100.10:18000` 即可（服务器端需在 `sshd_config` 开启 `GatewayPorts yes` 才能监听 0.0.0.0，且注意防火墙放行；仅本机回环监听则不需要）。这是临时演示的捷径，长期对外服务请正经部署，不要长期靠 -R 挂着。

### 4.2.3 动态转发 -D：SOCKS 代理

```bash
# 在本机 1080 端口开一个 SOCKS5 代理，流量经服务器出去
ssh -D 1080 -N user1@192.0.2.10
```

把浏览器/系统的代理设为 `SOCKS5 127.0.0.1:1080`，你的网页流量就以服务器为出口。适合在不可信网络下安全上网、调试「从服务器视角看网站」等问题。

---

## 4.3 跳板机 ProxyJump

场景：内网机器（如 `192.0.2.30`）不能直连，必须先登录一台有公网地址的跳板机再二次跳转。手动跳两次很烦，`ProxyJump` 一条命令直达：

```bash
# -J 跳板机地址，目标写内网地址
ssh -J user1@198.51.100.10 user1@192.0.2.30
```

写进客户端 `~/.ssh/config` 后更省事：

```text
Host bastion
    HostName 198.51.100.10
    User user1
    IdentityFile ~/.ssh/id_ed25519

Host db-internal
    HostName 192.0.2.30
    User user1
    ProxyJump bastion
```

之后 `ssh db-internal` 自动经跳板机到达内网机器，`scp` 等命令同样适用。前提：跳板机上要能通内网目标的 22 端口，且你的密钥在两台机上都已配置（或用 `ssh-agent` 转发，本教程不展开）。

---

## 4.4 Tailscale：无公网 IP 也能组网远程

### 4.4.1 是什么、与普通 SSH 的关系

Tailscale 基于 WireGuard，把你的多台设备组成一个虚拟局域网（tailnet），每台设备得到一个 `100.x.x.x` 的固定虚拟 IP。它自动做 NAT 穿透，所以服务器哪怕在家庭宽带、公司内网、没有公网 IP，也能被你的笔记本直接连上。

它与 SSH 的关系：Tailscale 解决的是「网络通不通」，通了之后你仍然用普通 `ssh user1@100.x.x.x` 登录（它还自带一个 `tailscale ssh` 免密登录功能，见下）。可以理解为：Tailscale 负责铺路，SSH 负责开车。

### 4.4.2 服务端安装与加入网络

```bash
# 官方一键安装脚本（Ubuntu Server 24.04/22.04 通用）
curl -fsSL https://tailscale.com/install.sh | sh

# 加入你的 tailnet（会输出一个链接，浏览器打开登录授权）
sudo tailscale up

# 查看本机获得的 100.x 虚拟 IP 与设备名
tailscale ip -4
tailscale status
```

在你自己的电脑上也装 Tailscale 并登录**同一账号**，两台设备即互通：

```bash
# 从笔记本直接 SSH 到服务器的虚拟 IP
ssh user1@100.100.100.10
# 或用 Tailscale 分配的设备名（MagicDNS 开启时）
ssh user1@web-01
```

### 4.4.3 tailscale ssh 与常用命令

```bash
# 服务端允许 Tailscale SSH（基于 tailnet 身份免密登录）
sudo tailscale up --ssh

# 客户端连接（由 Tailscale 接管认证，不再需要服务器密码/密钥）
tailscale ssh user1@web-01
```

常用运维命令：

```bash
tailscale status          # 看组网内所有设备与在线状态
tailscale ping web-01     # 测与另一台设备的连通性（直连还是中继 DERP）
tailscale netcheck        # 诊断本机网络穿透条件
tailscale down            # 断开组网（IP 保留，下次 up 恢复）
tailscale logout          # 退出登录并从网络移除本机身份
```

> 注意：走中继（DERP）时也能连，但延迟高于直连，`tailscale ping` 输出会告诉你是 `direct` 还是 `via DERP`。另外，Tailscale 的访问控制（ACL）在管理后台配置，重要服务器建议只允许自己的设备访问其 22 端口。

### 4.4.4 常见报错

| 现象 | 原因 | 处理 |
|------|------|------|
| `tailscale up` 卡住/超时 | 服务器出网受限、DNS 问题 | 查外网连通性，`tailscale netcheck` 看报告 |
| 能 ping 通但 SSH 不通 | SSH 服务未启、ufw 未放行虚拟网卡流量 | 服务器查 `systemctl status ssh`；ufw 需允许 `tailscale0` 接口或 100.x 网段 |
| 显示离线 | 设备休眠/断网、客户端版本过旧 | 两端 `tailscale version` 对比并升级后重连 |

---

## 4.5 VNC：图形桌面（务必走隧道）

Ubuntu Server 默认无桌面。需要图形界面时，先装一个轻量桌面（如 Xfce），再装 VNC 服务端。**VNC 协议本身加密很弱，绝不要把 590x 端口直接开到公网**——正确姿势是 VNC 只监听本机，外面用 SSH 隧道进来（4.2.1 节的方法）。

### 4.5.1 服务端安装与配置（TigerVNC）

```bash
# 安装轻量桌面与 VNC 服务端（已有桌面的机器可跳过桌面安装）
sudo apt install -y xfce4 xfce4-goodies tigervnc-standalone-server

# 首次运行设置 VNC 密码（最多 8 位有效字符；只用于连 VNC，与系统密码无关）
vncpasswd

# 启动一个桌面会话：:1 表示编号 1，对应端口 5901
vncserver :1 -geometry 1440x900 -localhost yes

# 查看已启动的会话
vncserver -list

# 关闭会话
vncserver -kill :1
```

`-localhost yes` 的意思是只允许本机连接，这是配合 SSH 隧道的关键设置——公网扫描器连 5901 会被直接拒绝。

### 4.5.2 客户端连接（经 SSH 隧道）

第 1 步，在客户端开隧道（保持终端开着）：

```bash
# 本机 5901 → 服务器本机 5901
ssh -L 5901:localhost:5901 user1@192.0.2.10
```

第 2 步，打开 VNC Viewer（RealVNC Viewer、TigerVNC Viewer 均可），连接地址填：

```text
localhost:5901
```

输入 `vncpasswd` 设的密码，即可看到 Xfce 桌面。

### 4.5.3 x11vnc：接管已有桌面

如果机器本身接了显示器、已有桌面在跑，想远程看到**同一个**画面（而不是新开虚拟桌面），用 x11vnc：

```bash
sudo apt install -y x11vnc
# 在已有桌面会话中启动（:0 是物理显示器那个桌面），同样建议只走隧道访问
x11vnc -display :0 -auth guess -rfbport 5900 -localhost
```

### 4.5.4 常见报错

| 现象 | 原因 | 处理 |
|------|------|------|
| 连上是灰屏/黑屏 | 桌面环境没装全、启动脚本缺失 | 确认 xfce4 已装；检查 `~/.vnc/xstartup` 是否调用 `startxfce4`，改后重启会话 |
| `Connection refused` | vncserver 没启动或端口算错 | `vncserver -list` 确认；端口 = 5900 + 会话编号 |
| 密码总不对 | VNC 密码只取前 8 位；或连错了会话 | 重设 `vncpasswd` 后重启会话再试 |
| 能连但极卡 | 分辨率太高、走公网直连绕路 | 降分辨率/色深；确保走隧道或改用 NoMachine |

---

## 4.6 xrdp：用 Windows 远程桌面连 Linux

xrdp 让 Linux 充当 RDP 服务端，Windows 自带的「远程桌面连接」就能直接连，免装客户端。

### 4.6.1 服务端安装

```bash
# 前置：已装桌面环境（如 Xfce，见 4.5.1）
sudo apt install -y xrdp

# 把 xrdp 用户加入 ssl-cert 组（否则会因读不到 TLS 证书报错）
sudo adduser xrdp ssl-cert

sudo systemctl enable --now xrdp
```

### 4.6.2 防火墙放行

```bash
# 只建议对内网放行（示例为内网段）；有条件优先限制来源
sudo ufw allow from 192.0.2.0/24 to any port 3389 proto tcp
# 若必须全网放行（不推荐长期如此）：sudo ufw allow 3389/tcp
```

### 4.6.3 Windows 客户端连接步骤

1. 按 `Win + R`，输入 `mstsc` 回车，打开远程桌面连接。
2. 「计算机」填服务器 IP（如 `192.0.2.10`），点「连接」。
3. 出现 xrdp 登录界面后，Session 选 `Xorg`，输入服务器的**系统用户名和密码**（如 `user1` 及其密码，不是 VNC 密码）。
4. 登录后即进入桌面。

### 4.6.4 常见报错（黑屏预告）

最经典的问题是**输完密码后黑屏或闪退**，常见原因：同一用户已在本地物理桌面登录（会话冲突）、桌面环境与 xrdp 兼容问题、权限/证书未配好。先做两个快速自查：退出服务器本地的桌面登录再远程连；确认 `xrdp` 已在 `ssl-cert` 组。更系统的排查清单见本教程 FAQ 章节的「xrdp 黑屏排查」一节。

---

## 4.7 NoMachine：流畅型远程桌面

NoMachine 使用 NX 协议，画面流畅度、低延迟和音视频支持都明显好于 VNC/RDP，个人使用免费，需要桌面与图形需求时是很实用的选择。

### 4.7.1 服务端安装（官网 .deb）

到官网 <https://www.nomachine.com/download> 下载 Linux 版（DEB，amd64）安装包。假设下载的文件名为 `nomachine_*.deb`（以官网当前版本为准）：

```bash
# 在下载目录执行安装（文件名按实际替换）
sudo dpkg -i nomachine_*.deb

# 服务默认开机自启，查看状态
sudo systemctl status nxserver
ss -tlnp | grep 4000
```

### 4.7.2 防火墙与客户端连接

```bash
# 放行 NX 默认端口 4000（TCP+UDP）；同样建议限制来源为内网/自有 IP
sudo ufw allow from 192.0.2.0/24 to any port 4000
```

客户端（你的电脑）同样从官网安装 NoMachine，打开后：

1. 点 `Add` 新建连接，协议选 `NX`；
2. Host 填服务器 IP（如 `192.0.2.10`），Port 保持 `4000`；
3. 双击连接，输入服务器**系统账号**（如 `user1`）与密码；
4. 选择/新建桌面会话，进入后按向导调一次分辨率与画质即可。

常见报错：连不上先查 4000 端口放行与 `nxserver` 状态；提示无可用桌面时，确认服务器已安装桌面环境且目标用户没有被限制登录。

---

## 4.8 串口与云厂商网页控制台：最后的救命通道

前面所有方式都依赖「系统能启动、网络配置正确」。当你改错 netplan 断网、改错 SSH 配置把自己锁在外面、或者新内核起不来时，就需要不依赖系统内网络的通道：

### 4.8.1 云厂商网页控制台（VNC/终端）

阿里云、腾讯云、AWS、Azure 等都在管理后台提供「远程连接 / VNC 登录 / 控制台」入口，本质是把云主机的虚拟显示器搬到网页里。**何时用**：SSH 怎么都连不上时，先上网页控制台登进去，检查 `systemctl status ssh`、`ip a`、`sudo netplan apply` 前的配置错误，修好后再回 SSH。购买云服务器后请先找到这个入口在哪，不要等出事才找。

### 4.8.2 物理机串口控制台（简介）

物理服务器/嵌入式设备常用串口（Serial）输出启动全过程日志。Ubuntu 侧开启串口登录（getty）示例：

```bash
# 为第一个串口启用登录服务（波特率等参数由硬件手册决定）
sudo systemctl enable --now serial-getty@ttyS0.service
```

客户端用串口线连接后，以终端软件（如 PuTTY 串口模式、Linux 的 `screen /dev/ttyUSB0 115200`）打开，即可在无网络情况下登录。数据中心里还有 IPMI/iDRAC/iLO 等带外管理口，作用类似：远程开关机、看屏幕、挂载镜像，是物理服务器的「网页控制台」。

### 4.8.3 使用原则

- 控制台是**兜底手段**，不是日常工具：画质差、延迟高、无复制粘贴便利。
- 但每次做高风险操作（改网络、改 SSH、升级内核）前，先确认自己知道控制台入口在哪——这是敢动手的前提。

---

## 本章小结

- 对比表先行：命令行用 SSH，组网用 Tailscale，图形用 VNC/xrdp/NoMachine（不裸奔公网），救命用控制台。
- SSH 密钥登录是基本功：`ssh-keygen` → `ssh-copy-id` → `~/.ssh/config` 别名 → 验证成功后再关密码登录。
- 端口转发三件套：`-L` 进内网、`-R` 临时外放、`-D` 做代理；跳板机用 `ProxyJump`。
- 改 SSH/网络配置的保命习惯：备份配置、保持旧连接、开新终端验证、熟悉控制台入口。

---

*导航：上一章：[第 3 章 更新与维护](03-update-maintenance.md) | 下一章：无（本章为当前最后一章） | 返回目录：[README](README.md)*
