# 第 2 章 初始设置

> 上一章你已经装好 Ubuntu Server 24.04 LTS 并能用 SSH 连上。本章把一台「刚装好的裸机」调整成「可长期稳定使用的服务器」：用户与权限、主机名、时间、网络静态 IP、防火墙，每一项都讲清是什么、为什么、改错了怎么办。

本章操作默认你已用安装时创建的用户（如 `user1`）登录。该用户拥有 `sudo` 权限。

---

## 2.1 首次登录后先看一眼系统状态

```bash
# 我是谁、我在哪台机器上
whoami
hostname

# 系统版本
lsb_release -a
# 或
cat /etc/os-release

# 内核版本、内存、磁盘概览
uname -r
free -h
df -h
```

这些命令不改任何东西，只是建立「基线认知」：后面任何异常，都可以和刚装好时的状态对比。

---

## 2.2 sudo 与用户管理

### 2.2.1 为什么不直接用 root

Linux 的 `root` 是最高权限账户，能删除任何文件、改任何配置，没有任何拦截。日常直接用 root 有三个实际风险：

1. **手滑无挽回**：一条写错路径的 `rm` 或 `chmod` 就能毁掉系统，普通用户至少还会被权限挡一下。
2. **被爆破的头号目标**：全世界的扫描器都在对每台服务器的 `root` 账户试密码。禁用 root 远程登录，等于直接废掉绝大部分自动化攻击（2.2.3 节与第 4 章都会落实这一点）。
3. **审计困难**：多人共用 root 时，出了事无法区分是谁的操作；每人用自己账户 + `sudo`，`sudo` 的每条命令都会记日志。

**推荐模式**：平时用普通用户登录，需要管理操作时在命令前加 `sudo` 临时提权。这也是 Ubuntu 的默认设计——安装时创建的用户就在 `sudo` 组里，root 账户本身被锁定密码、不能直接登录。

```bash
# 以管理员权限执行单条命令（会要求输入你自己的密码）
sudo apt update

# 临时切到 root shell（排障时偶尔用，用完 exit 退出，不要常驻）
sudo -i
```

### 2.2.2 新建管理员用户（多人使用或备用账户）

如果是多人运维，不要共用 `user1`，给每个人建独立账户：

```bash
# 创建用户（会交互式地让你设密码、填姓名等，姓名可一路回车跳过）
sudo adduser user2

# 把新用户加入 sudo 组，获得管理员权限
sudo usermod -aG sudo user2

# 验证：查看 user2 属于哪些组，输出中应包含 sudo
groups user2
```

> 参数解读：`usermod -aG sudo user2` 中 `-a` 是 append（追加），`-G` 指定附加组。**漏写 `-a` 是经典事故**：`usermod -G sudo user2` 会把用户原有的其他附加组全部冲掉。改了会怎样：轻则用户丢了某些设备/服务的访问组权限，排查起来很隐蔽。所以这条命令请连 `-a` 一起背下来。

普通（无 sudo）用户同理，只建不加组即可：

```bash
sudo adduser user3
```

其他常用管理命令：

```bash
# 修改某用户密码
sudo passwd user2

# 临时锁定/解锁账户（不能登录，但文件保留）
sudo usermod -L user2
sudo usermod -U user2

# 删除用户（-r 连主目录一起删，删前确认数据已备份）
sudo deluser --remove-home user3
```

### 2.2.3 为禁用 root SSH 登录做铺垫

本章先确认状态，具体的 SSH 配置修改放在第 4 章（改 SSH 配置前必须先保证密钥/新账户可用，否则容易把自己锁在门外）。

```bash
# 查看 root 账户状态：密码字段为 ! 或 * 表示已锁定（Ubuntu 默认即如此）
sudo passwd -S root
```

只要保持这个默认状态、并且后续按第 4 章设置 `PermitRootLogin no`，root 就无法从网络登录，本机也只能通过 `sudo` 间接使用。

---

## 2.3 修改主机名

主机名是这台机器在命令行提示符、日志和局域网里的名字，如 `server1`、`web-01`。命名建议：全小写字母+数字+连字符，能表达用途，如 `web-01`、`db-01`，不要用中文、空格和下划线。

```bash
# 正规改法：一条命令同时改当前主机名并写入 /etc/hostname
sudo hostnamectl set-hostname web-01

# 立即验证（重新登录后提示符也会更新）
hostnamectl
```

改完还要同步 `/etc/hosts`，否则某些服务（如 `sudo` 本身）解析本机名时会变慢或报警告：

```bash
sudo nano /etc/hosts
```

把其中旧主机名对应的行改掉，典型内容如下（`<服务器IP>` 替换为实际地址，本机行通常是 127.0.1.1）：

```text
127.0.0.1 localhost
127.0.1.1 web-01
```

> 是什么：`/etc/hosts` 是本机优先的名字解析表，系统查名字时先查它、再查 DNS。
> 为什么要改：`hostnamectl` 只改了 `/etc/hostname`，`/etc/hosts` 里的旧名不会自动更新，两者不一致时 `sudo` 每次都会多等几秒并打印 `unable to resolve host` 警告。
> 改错会怎样：把 `localhost` 那行删掉会引起一系列诡异问题；只改名字部分、不动 IP，是安全操作。

---

## 2.4 时区与时间同步

服务器时间不准，后果比想象中严重：日志时间对不上没法排障、证书校验会因「证书尚未生效/已过期」失败、定时任务在错误的时间触发。装好后第一件事就该把时区和自动同步确认好。

### 2.4.1 设置时区

```bash
# 查看当前时间与时区设置
timedatectl

# 设置为中国标准时间（在国内的服务器）
sudo timedatectl set-timezone Asia/Shanghai

# 再次确认，输出中 Time zone 应为 Asia/Shanghai (CST, +0800)
timedatectl
date
```

不知道时区名怎么写时可以列出全部可选值：

```bash
timedatectl list-timezones | grep -i shanghai
```

### 2.4.2 时间同步（NTP）

Ubuntu Server 默认使用 `systemd-timesyncd` 做轻量 NTP 同步，对绝大多数场景够用：

```bash
# 确认同步状态：System clock synchronized: yes、NTP service: active
timedatectl status

# 若 NTP service 不是 active，手动开启
sudo timedatectl set-ntp true
```

对时间精度要求高（如数据库集群、金融类应用）时，可改用功能更完整的 `chrony`：

```bash
sudo apt install -y chrony
sudo systemctl enable --now chrony

# 查看同步源与偏差
chronyc tracking
chronyc sources -v
```

> 注意：`chrony` 与 `systemd-timesyncd` 二选一即可，装了 chrony 后应停掉 timesyncd 避免冲突：`sudo timedatectl set-ntp false` 并确认 `systemctl status chrony` 正常。
> 22.04 与 24.04 差异：两者默认均为 timesyncd，本节命令通用，无区别。

---

## 2.5 语言与 locale

locale 决定了命令行报错语言、日期/货币格式、字符排序规则。服务器推荐保持 `en_US.UTF-8` 或按团队习惯设 `zh_CN.UTF-8`，关键是**编码必须是 UTF-8**，否则中文文件名、日志会出现乱码。

```bash
# 查看当前 locale
locale

# 查看系统已生成的 locale
locale -a | grep -i -E "zh_CN|en_US"

# 若缺 zh_CN.UTF-8，安装语言包并生成
sudo apt install -y language-pack-zh-hans
sudo locale-gen zh_CN.UTF-8 en_US.UTF-8

# 设置系统默认 locale（二选一）
sudo update-locale LANG=en_US.UTF-8
# 或
sudo update-locale LANG=zh_CN.UTF-8
```

修改后**重新登录**才生效。注意 `update-locale` 写的是 `/etc/default/locale`，只影响新登录会话；当前窗口不会立刻变化，这是正常现象。

> 常见坑：SSH 客户端会把本地 locale 变量透传到服务器，若服务器没生成对应 locale，会报 `perl: warning: Setting locale failed` 之类警告。解决办法就是在服务器上把客户端用的那个 locale 用 `locale-gen` 生成出来，或者在客户端 `~/.ssh/config` 里不发送 locale 变量。

---

## 2.6 网络：用 netplan 配置静态 IP

### 2.6.1 为什么服务器要用静态 IP

DHCP 分配的地址租约到期后可能变化，IP 一变，所有指向这台服务器的 SSH、网站、脚本全部失效。服务器必须用固定地址。静态 IP 要在**路由器/网络规划允许的地址段**里挑一个未被占用的，并避开 DHCP 自动分配池，避免地址冲突。

### 2.6.2 先确认网卡名

```bash
ip link
```

输出示例（节选）：

```text
1: lo: <LOOPBACK,UP> ...
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```

`lo` 是环回口不用管；`enp3s0` 就是物理网卡名（也可能是 `ens33`、`eth0`、`eno1` 等，以你机器实际输出为准，下文用 `enp3s0` 作示例）。

### 2.6.3 找到并备份 netplan 配置文件

```bash
ls /etc/netplan/
# 常见文件名：50-cloud-init.yaml 或 99-config.yaml

# 改之前先备份，改错了能还原
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
```

> 提示：如果文件头写着由 cloud-init 管理，还需在 `/etc/cloud/cloud.cfg.d/` 里禁用 cloud-init 的网络管理，或直接新建一个编号更大的文件如 `99-config.yaml`（netplan 按文件名字典序合并，后面的覆盖前面的）。最省事的做法是编辑现有文件并保持只有一个网络配置文件。

### 2.6.4 静态 IP 完整 YAML 示例

用编辑器打开配置文件：

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

写入下面内容（地址均为示例，请按你的网络替换）：

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp3s0:
      dhcp4: false
      addresses:
        - 192.0.2.10/24
      routes:
        - to: default
          via: 192.0.2.1
      nameservers:
        addresses:
          - 223.5.5.5
          - 8.8.8.8
```

逐项解释：

| 配置项 | 含义 | 填错的后果 |
|--------|------|------------|
| `dhcp4: false` | 关闭 DHCP，改用下面手写的地址 | 不关的话静态配置可能被 DHCP 覆盖 |
| `addresses: 192.0.2.10/24` | 本机 IP 与子网掩码（/24 即 255.255.255.0） | IP 与别的设备重复会造成双方时通时断 |
| `routes: to default via 192.0.2.1` | 默认网关，一般是路由器地址 | 填错则只能通内网、出不了外网 |
| `nameservers` | DNS 服务器，可填运营商/公共 DNS | 填错表现为「能 ping 通 IP，但打不开域名」 |

**YAML 格式铁律：只能用空格缩进，绝不能用 Tab；同一层级缩进必须对齐。** netplan 对格式极其严格，多数「配置不生效」都是缩进问题。

> 22.04 差异：22.04 老式写法用 `gateway4: 192.0.2.1` 表示默认网关，该字段在 24.04 中已废弃（写了会警告甚至报错），统一改用上面 `routes` 的新写法。新写法在 22.04 上同样可用，所以本教程只教新写法。

### 2.6.5 应用配置与自救方法

**强烈建议：改网络配置时，人最好在服务器跟前，或先用 `netplan try`。**

```bash
# 自救神器：try 模式应用后会倒计时，没按 Enter 确认就自动回滚旧配置
sudo netplan try

# 确认网络正常后，在倒计时内按 Enter 固化
# 若新配置导致断网，什么都不用做，等 120 秒自动恢复

# 确认无误的改法（本地控制台操作时可用）：直接应用
sudo netplan apply

# 应用后验证
ip a
ping -c 3 192.0.2.1        # 通网关：内网链路 OK
ping -c 3 223.5.5.5        # 通外网 IP：路由 OK
ping -c 3 ubuntu.com       # 通域名：DNS 也 OK
```

> 为什么强调 `netplan try`：如果你正通过 SSH 远程改 IP，一旦新配置有误，连接立即断开且再也连不回来，只能去现场接显示器。`netplan try` 的自动回滚就是为这个场景设计的。远程改网络前，先用 `netplan try`，是老运维的肌肉记忆。
> 如果连 `try` 都来不及、已经失联：在本地控制台把备份文件还原（`sudo cp ...yaml.bak ...yaml`）后 `sudo netplan apply` 即可恢复。

配置文件权限还要留意：新版 netplan 会警告配置文件权限过宽，执行下面命令收紧：

```bash
sudo chmod 600 /etc/netplan/*.yaml
```

---

## 2.7 ufw 防火墙基础

ufw（Uncomplicated Firewall）是 Ubuntu 对底层防火墙规则的简化封装。服务器开防火墙的意义：只开放真正需要的端口，其余一律拒绝，把攻击面降到最低。

### 2.7.1 顺序警告：先放行 SSH，再启用防火墙

> ⚠️ **这是本章第二重要的警告**：如果你通过 SSH 远程操作，**必须先执行 `sudo ufw allow ssh`，再执行 `sudo ufw enable`**。顺序反了，防火墙启用瞬间就会切断你的 SSH 连接并拒绝重连，只能去现场处理。

```bash
# 1）先放行 SSH（22 端口）
sudo ufw allow ssh
# 等价写法：sudo ufw allow 22/tcp

# 2）再启用防火墙（会提示确认，输入 y）
sudo ufw enable

# 3）查看状态与带序号的规则列表
sudo ufw status numbered
```

输出示例：

```text
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
```

### 2.7.2 常用规则速查

```bash
# 放行网站端口
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 只允许某个内网段访问某端口（例如数据库只给内网用）
sudo ufw allow from 192.0.2.0/24 to any port 3306 proto tcp

# 按序号删除规则（序号来自 status numbered）
sudo ufw delete 2

# 直接按规则内容删除
sudo ufw delete allow 80/tcp

# 拒绝（明确 drop）某个端口
sudo ufw deny 23/tcp

# 查看完整状态
sudo ufw status verbose

# 关闭 / 重置（重置会清空所有规则，慎用）
sudo ufw disable
sudo ufw reset
```

规则说明：ufw 默认策略是「入站拒绝、出站允许」，这正是服务器需要的：外面主动连进来默认不行（除放行名单外），服务器自己往外访问（下载更新等）不受影响。

### 2.7.3 换了 SSH 端口别忘了同步改防火墙

如果你按第 4 章把 SSH 改到非 22 端口（如 2222），记得：

```bash
sudo ufw allow 2222/tcp
# 确认新端口能连上后，再删掉旧的 22 放行
sudo ufw delete allow 22/tcp
```

---

## 2.8 确认 SSH 服务状态（本章验收）

最后做一遍总检查，确认本章成果：

```bash
# SSH 服务是否在运行、是否开机自启
systemctl status ssh
systemctl is-enabled ssh

# 22 端口是否在监听（应看到 sshd 监听 0.0.0.0:22）
ss -tlnp | grep 22

# 若服务异常，重启它
sudo systemctl restart ssh
```

从你自己的电脑再 SSH 登录一次，并核对这几件事：

- [ ] 能用 `user1`（及新建的管理员账户）登录、能 `sudo`
- [ ] `hostname` 显示新主机名，`sudo` 不再报解析警告
- [ ] `timedatectl` 显示时区为 Asia/Shanghai 且时间已同步
- [ ] 服务器 IP 已是静态地址，重启后不变（可 `sudo reboot` 验证一次）
- [ ] `sudo ufw status` 为 active，且包含 SSH 放行规则

全部打勾，初始设置完成。下一章进入日常更新与维护。

---

## 本章小结

- 日常不用 root：普通用户 + `sudo`，新建管理员务必 `usermod -aG sudo`（别漏 `-a`）。
- 主机名用 `hostnamectl` 改，并同步 `/etc/hosts`。
- 时区 `Asia/Shanghai` + NTP 同步是排障和证书正常工作的前提。
- netplan 静态 IP：YAML 缩进用空格、网关用 `routes` 新写法、远程修改先 `netplan try`。
- ufw 顺序铁律：**先 `allow ssh`，后 `enable`**。

---

*导航：上一章：[第 1 章 系统安装](01-install.md) | 下一章：[第 3 章 更新与维护](03-update-maintenance.md) | 返回目录：[README](README.md)*
