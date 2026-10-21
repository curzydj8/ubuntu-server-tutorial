# 第 11 章 FAQ 与排障清单

排障的通用心法只有一条：**先定位故障在哪一层，再动手修**。本章每条都按「现象 → 排查命令 → 常见原因与解法」写，对照执行即可。命令默认在服务器本机或已登录的会话里运行；SSH 本身连不上时，需要先借云厂商控制台 / VNC / 物理显示器进系统再排查。

## 11.1 SSH 连不上

**现象**：`ssh` 卡住超时、或立即报 `Connection refused`、或反复要求输密码进不去。

**排查命令**：

```bash
# 在本地执行：网络是否可达（部分云主机禁 ping，不通不等于机器没了）
ping -c 3 192.0.2.10

# 端口是否开放（22 为默认 SSH 端口，改过端口按实际值）
nc -vz 192.0.2.10 22
# 或：telnet 192.0.2.10 22

# 本地啰嗦模式连接，看卡在哪一步
ssh -vvv <用户名>@192.0.2.10
```

```bash
# 在服务器上（经控制台进入后）执行：
systemctl status ssh
ss -tlnp | grep 22
sudo ufw status
ls -l ~/.ssh/
```

**常见原因与解法**：

| 原因 | 判断依据 | 解法 |
|---|---|---|
| sshd 没在运行 | `systemctl status ssh` 不是 active | `sudo systemctl restart ssh`，失败看 `journalctl -u ssh -e` |
| 防火墙拦截 | 本机 `ufw` 未放行 22 | `sudo ufw allow 22/tcp` |
| 云安全组未放行 | 本机一切正常、外部端口不通 | 去云控制台安全组放行 22（来源 IP 尽量收窄） |
| 端口被改过 | `ss` 显示 sshd 监听在其他端口 | 用实际端口连接：`ssh -p <端口>` |
| 密钥文件权限过宽（本地侧） | `-vvv` 显示 `bad permissions` 后跳过密钥 | 本地私钥改 `chmod 600 ~/.ssh/id_ed25519` |
| IP 被 fail2ban 封禁 | 换个网络就能连 | 服务器上 `sudo fail2ban-client status sshd` 查看并解封 |
| 机器根本没开机/网络断了 | ping 与所有端口都不通 | 查云控制台实例状态与账单/流量状态 |

记住：`Connection refused` 说明网络通、端口没人听（服务问题）；`Connection timed out` 说明包过不去（防火墙/安全组/网络问题）。先分清这两种，再往下查。

## 11.2 SSH 能连，但密钥登录失败

**现象**：服务器有响应，但密钥被拒，最后回落到密码登录或直接 `Permission denied (publickey)`。

**排查命令**：

```bash
# 服务器上检查权限（这是最高频原因）
ls -ld ~ ~/.ssh
ls -l ~/.ssh/authorized_keys

# 服务端日志直接写明拒绝原因
sudo journalctl -u ssh -n 50 --no-pager
sudo tail -n 50 /var/log/auth.log
```

**常见原因与解法**：权限必须满足：家目录不可被组/他人写入、`.ssh` 为 `700`、`authorized_keys` 为 `600`，且属主都是该用户本人。权限过宽时 sshd 会**静默忽略**密钥（StrictModes），日志里能看到 `Authentication refused: bad ownership or modes`。

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R "$USER:$USER" ~/.ssh
```

其他原因：公钥粘贴时被编辑器折行/改字符（应一整行原样追加）；客户端用错私钥（`ssh -i ~/.ssh/<对应私钥> -v` 指定试一次）；服务端 `sshd_config` 里 `PubkeyAuthentication no` 或 `AuthorizedKeysFile` 被改过。

## 11.3 xrdp 登录黑屏 / 闪退

**现象**：远程桌面连上、输完密码后黑屏几秒断开，或桌面一闪而退。

**排查命令**：

```bash
systemctl status xrdp xrdp-sesman
ls -l ~/.xsession
id   # 确认当前用户是否在 ssl-cert 组
tail -n 50 ~/.xsession-errors
```

**常见原因与解法**：

- **没指定会话类型**：xrdp 默认会话在 XFCE 机器上起不来。写死会话文件（本教程桌面为 XFCE）：

```bash
echo "xfce4-session" > ~/.xsession
chmod +x ~/.xsession
sudo systemctl restart xrdp
```

- **证书权限问题**：xrdp 读不到 `/etc/ssl/private/ssl-cert-snakeoil.key` 时会话协商失败，把用户加入证书组：

```bash
sudo usermod -aG ssl-cert "$USER"
sudo systemctl restart xrdp
```

- **缺 xorgxrdp 驱动**：只有 xrdp 没有 Xorg 后端时必然黑屏，补装 `sudo apt install xorgxrdp` 后重启 xrdp。
- **物理会话冲突**：同一用户已在物理桌面登录，某些配置下 xrdp 新会话会与之冲突。先在物理端注销，或换一个用户走 xrdp；也要理解二者本就是两套独立会话（见 9.5.3）。

## 11.4 VNC 连上是灰屏 / 只有鼠标

**现象**：VNC 客户端连得上，但画面是灰底 + 一个叉形鼠标，没有桌面。

**排查命令**：

```bash
ls -l ~/.vnc/xstartup
cat ~/.vnc/xstartup
ls ~/.vnc/*.log && tail -n 50 ~/.vnc/*.log
```

**常见原因与解法**：`~/.vnc/xstartup` 没写启动桌面的命令，或文件没有可执行权限。VNC 服务端只负责起一个裸 X 会话，桌面要靠 xstartup 拉起来（XFCE 写法）：

```bash
cat > ~/.vnc/xstartup <<'EOF'
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
exec startxfce4
EOF
chmod +x ~/.vnc/xstartup
```

改完后杀掉旧会话再重开（会话号按实际）：

```bash
vncserver -kill :1
vncserver :1
```

## 11.5 桌面装完了，开机不自动进图形界面

**现象**：装了 XFCE 却还是停在字符登录界面，或图形登录器没起来。

**排查命令**：

```bash
systemctl get-default
systemctl status display-manager
ls -l /etc/systemd/system/display-manager.service
```

**常见原因与解法**：

- 默认启动目标还是多用户（字符）模式。应改为 `graphical.target`：

```bash
sudo systemctl set-default graphical.target
systemctl get-default   # 应输出 graphical.target
```

- 显示管理器（LightDM）没装或没启用：`sudo apt install lightdm`，安装时在选择界面选 lightdm；已装则 `sudo systemctl enable --now lightdm`。
- 显卡驱动/会话崩溃导致登录器反复重启：看 `journalctl -u lightdm -e` 与 `/var/log/Xorg.0.log` 再对症处理。

## 11.6 磁盘满

**现象**：服务写不了文件、apt 报错、登录异常，`df` 显示根分区 100%。

**排查命令**：

```bash
df -h
sudo du -sh /var/log /var/cache/apt /snap 2>/dev/null
sudo du -h --max-depth=1 / 2>/dev/null | sort -hr | head
ncdu /            # 交互式找大户，需 apt install ncdu
```

**常见原因与解法**：

- **日志失控**：`sudo journalctl --vacuum-time=7d` 或 `--vacuum-size=500M` 清理 systemd 日志；`/var/log` 下单个日志文件异常巨大时查是哪个服务在刷屏，先治源头再清文件。
- **apt 缓存**：`sudo apt clean`。
- **snap 旧版本堆积**：`snap list --all` 看 disabled 的旧版本，逐个 `sudo snap remove <名称> --revision=<版本号>`；并可设置只保留 2 个版本：`sudo snap set system refresh.retain=2`。
- **旧内核**：`sudo apt autoremove --purge` 清理不再使用的内核与依赖。
- 更系统的磁盘管理见 **第 3 章** 的对应小节，本条只作应急清单。

> 提醒：删除前先看清路径，`du` 定位、`ls` 确认、再删；不要对着 `/var` 无差别 `rm -rf`。

## 11.7 服务起不来

**现象**：`systemctl start` 失败，或服务起来几秒就退出。

**排查命令**：

```bash
systemctl status <服务名>            # 先看状态页底部的报错摘要
journalctl -xeu <服务名>             # 看该服务完整日志并定位错误
journalctl -u <服务名> -n 100 --no-pager
ss -tlnp | grep <端口>               # 端口是否被别的进程占用
```

**常见原因与解法**：

| 原因 | 线索 | 解法 |
|---|---|---|
| 端口被占 | 日志含 `Address already in use`，`ss` 显示占用进程 | 停掉占用进程，或给本服务换端口 |
| 配置写错 | 日志含 `syntax` / `parse error`、行号 | 按行号改配置，先跑该软件自带的配置检查（如 `nginx -t`）再重启 |
| 权限不足 | `Permission denied` 打开某文件/目录 | 查该路径属主与服务运行用户是否匹配 |
| 依赖没就绪 | 偶尔失败、重试能成 | 单元里补 `After=` / 改用重试策略，别靠手动多启几次糊弄过去 |
| 磁盘满 / 内存不足 | 多个服务同时异常 | 先查 `df -h`、`free -h`（见 11.6） |

通用动作序列：`status` 看摘要 → `journalctl` 看细节 → 改一处 → `restart` → 再 `status` 确认 active。一次只改一处，否则不知道是哪步起的作用。

## 11.8 网络不通

**现象**：浏览器打不开、apt 更新失败、ping 不通。

**排查命令**（从内向外逐层测）：

```bash
ip a                     # 网卡是否存在、是否拿到 IP
ip route                 # 默认网关是谁
ping -c 3 <网关地址>      # 通内网
ping -c 3 192.0.2.1      # 通公网 IP（示例地址仅示意格式；可换公共 DNS 的 IP 测试）
resolvectl status        # DNS 配置与当前解析服务器
ping -c 3 example.com    # 测域名解析+连通
cat /etc/netplan/*.yaml  # 检查 netplan 配置
```

**常见原因与解法**（netplan 检查清单）：

- 配置文件 YAML **缩进错位**（必须空格、层级对齐），改完没执行 `sudo netplan apply`。
- 静态 IP 与网段/网关不在同一网段，或 IP 与局域网内其他设备冲突。
- DNS 没配或配错：`ip` 通但域名不通就是它，在 netplan 的 `nameservers` 里补公共 DNS。
- 网卡名写错（用 `ip a` 的实际名字，别照抄教程里的示例名）。
- 云主机还要查安全组出方向与实例网络状态；走代理上网的机器，先用第 10 章的 `--no-proxy-server` 思路排除代理因素再查底层网络。

## 11.9 时间不对导致证书报错

**现象**：浏览器报证书「尚未生效 / 已过期」，`curl` 报 `certificate has expired`，但网站在别处打开正常——多半是**本机时间错了**，不是证书的问题。

**排查命令**：

```bash
timedatectl
date
timedatectl show -p NTPSynchronized
```

**常见原因与解法**：

- NTP 同步没开：`sudo timedatectl set-ntp true`，稍等后复查 `NTPSynchronized=yes`。
- 时区不对只影响显示，不影响证书校验（校验用 UTC）；但顺手校准：`sudo timedatectl set-timezone Asia/Shanghai`。
- 虚拟机/休眠恢复后时钟漂移严重：重启 `systemd-timesyncd`（`sudo systemctl restart systemd-timesyncd`）强制重新同步。

## 11.10 apt 被锁 / dpkg 进程中断

**现象**：装软件时报 `Could not get lock /var/lib/dpkg/lock-frontend`，或上次安装中途断电/断网后 apt 拒绝工作。

**排查命令**：

```bash
ps aux | grep -E "apt|dpkg" | grep -v grep     # 是否真有进程在跑
sudo fuser -v /var/lib/dpkg/lock-frontend      # 谁正持有锁
systemctl status unattended-upgrades           # 后台自动更新也可能持锁
```

**常见原因与解法**：

- **真有进程在装东西**（包括后台自动更新）：等它结束，或确认无用后正常终止。锁存在本身不是故障，是保护机制。
- **上次 dpkg 被打断**（断电、强杀、SSH 断开杀掉前台进程）：修复半成品状态：

```bash
sudo dpkg --configure -a
sudo apt --fix-broken install
sudo apt update
```

> ⚠️ **绝不要直接删除锁文件**（`rm /var/lib/dpkg/lock*`）。锁文件删掉并不会解锁真正的数据库状态，只会让两个包管理进程有机会同时写数据库，结果是 dpkg 数据库损坏，修起来比重装还麻烦。流程永远是：先查谁持锁 → 等待或结束该进程 → `dpkg --configure -a` 收尾。

## 11.11 求助时先收集的信息

自己查不下去、要向人（或 AI）求助时，先把下面这组信息打包好，能省掉一半来回：

```bash
lsb_release -a                 # 发行版与版本
uname -a                       # 内核版本与架构
systemctl status <出问题的服务> # 服务状态（连同底部日志行）
journalctl -u <服务名> -n 50 --no-pager   # 最近日志片段
df -h                          # 磁盘概况（很多怪问题源于磁盘满）
free -h                        # 内存概况
ip a                           # 网络概况（打码处理敏感地址后再外发）
```

附带说明三件事：**什么时候开始坏的、出问题前你改过什么、你已经试过哪些办法**。现象 + 日志 + 变更史，是任何排障者最需要的三样东西。

到这里，《Ubuntu Server 全套教程》正文结束。建议把第 8~11 章当工具箱用：遇到具体问题，先回对应章节看原理，再到本章清单里对号入座。

---

**导航**

- 上一章：[第 10 章 浏览器](10-browser.md)
- 下一章：[第 12 章 机器清单与设置（实战环境）](12-machines.md)
- 返回目录：[README](README.md)
