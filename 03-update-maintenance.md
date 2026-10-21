# 第 3 章 更新与维护

> 服务器装好只是开始。长期稳定运行靠的是规律的更新与清理：安全补丁及时打、磁盘别被日志撑爆、大版本升级不蛮干。本章讲透 apt 更新机制、自动安全更新、内核管理、空间清理、大版本升级流程和备份思路。

---

## 3.1 apt update 与 apt upgrade 的区别

这是新手最常见的混淆点，两条命令名字像，做的事完全不同：

| 命令 | 做什么 | 会不会改动系统里的软件 |
|------|--------|--------------------------|
| `apt update` | 从软件源下载最新的「软件目录」（包索引），刷新本地缓存 | **不会**。只更新「知道有哪些新版本」，不安装任何东西 |
| `apt upgrade` | 对照索引，把已安装软件升级到新版本并安装 | **会**。真正下载并安装更新 |

打个比方：`update` 是「去书店拿一份最新书目」，`upgrade` 才是「按书目把书买回来」。所以标准动作永远是连着执行：

```bash
sudo apt update
sudo apt upgrade -y
```

只跑 `upgrade` 不跑 `update`，apt 用的是旧目录，可能告诉你「已经是最新」而其实安全补丁早已发布——这是很多服务器长期没打上补丁的原因。

### 3.1.1 查看哪些包可更新

```bash
# 列出可更新的包：名称 / 当前版本 → 新版本
apt list --upgradable

# 只看与安全相关的更新（需先 apt update）
apt list --upgradable 2>/dev/null | grep -i security
```

### 3.1.2 upgrade 与 full-upgrade 的差异

```bash
sudo apt full-upgrade -y
```

| 对比项 | `apt upgrade` | `apt full-upgrade`（旧名 dist-upgrade） |
|--------|---------------|-------------------------------------------|
| 安装新版本 | ✅ | ✅ |
| 需要时安装新依赖 | ❌ 保守处理 | ✅ 会装 |
| 需要时删除冲突的旧包 | ❌ 会保留（held back） | ✅ 会删 |
| 典型场景 | 日常更新 | `upgrade` 后仍有包被「保留」时；大版本升级时 |

日常维护用 `upgrade` 就够；当输出里出现 `The following packages have been kept back`（有包被保留未升），说明升级需要增删依赖，这时再执行 `full-upgrade`，并**仔细看它列出的「将要删除」清单**，确认没有你需要的软件被误伤后再确认。

> 历史注脚：`apt-get dist-upgrade` 是老命令，与 `apt full-upgrade` 等价，网上老教程里很常见，见到时知道是同一回事即可。名字里的 dist 指 distribution（发行版），但它本身并不会把 22.04 变成 24.04——跨大版本要用 3.5 节的 `do-release-upgrade`。

### 3.1.3 更新后要不要重启

```bash
# 存在这个文件 = 有更新要求重启才完全生效（多为内核、libc 等核心组件）
ls /var/run/reboot-required

# 看是哪些包要求的
cat /var/run/reboot-required.pkgs
```

普通包更新不需要重启，服务会自行重载；内核和底层库更新则必须重启。生产机请选业务低峰期重启。

---

## 3.2 unattended-upgrades：自动安全更新

服务器最怕「装好就忘」：几个月不登录，安全补丁积压。`unattended-upgrades` 能自动下载安装**安全更新**，是 Ubuntu Server 的标配能力，建议开启。

### 3.2.1 安装与启用

```bash
# 24.04/22.04 服务器通常已预装，先确认；没有就装
sudo apt install -y unattended-upgrades

# 交互式配置：弹出界面时选 Yes（启用自动更新）
sudo dpkg-reconfigure -plow unattended-upgrades
```

选 Yes 后会生成 `/etc/apt/apt.conf.d/20auto-upgrades`，内容大致如下，也可以手动核对/编辑：

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

两行分别表示：每天自动刷新索引、每天自动执行无人值守升级。值 `"1"` 为开启，`"0"` 为关闭。

### 3.2.2 核心配置文件解读

主配置文件：`/etc/apt/apt.conf.d/50unattended-upgrades`

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

关键段落（默认内容节选，注释已译为中文说明）：

```text
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}";
        "${distro_id}:${distro_codename}-security";
        // "${distro_id}:${distro_codename}-updates";   // 默认注释掉
};
```

- `Allowed-Origins` 决定「自动装哪些来源的更新」。默认只含**发行版本身 + security 安全源**，这是合理保守的默认：只自动打安全补丁，不自动追功能更新。
- 如果你想连普通更新（`-updates`）也自动装，把对应行前面的 `//` 去掉。改了会怎样：更新更及时，但偶尔的功能更新可能改变软件行为，生产环境建议保持默认（只自动安全更新），普通更新人工择期执行。

其他常用可选项（在同一文件里按需取消注释并改值）：

```text
// 自动清理不再需要的旧依赖（建议开启，防磁盘被旧内核/旧包堆满）
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// 更新需要重启时是否自动重启：默认 false
Unattended-Upgrade::Automatic-Reboot "false";

// 若允许自动重启，指定在几点执行（例如凌晨 3:30）
Unattended-Upgrade::Automatic-Reboot-Time "03:30";

// 出问题时发邮件通知（需要本机已配置邮件发送，一般可忽略，用日志检查代替）
// Unattended-Upgrade::Mail "root";
```

> 自动重启怎么设：`Automatic-Reboot "true"` 意味着装完内核等关键更新后机器会自己重启。无人值守的个人/小团队服务器开启它更安全（避免补丁装了却因没重启不生效）；承载在线业务的机器则保持 `false`，人工在低峰期重启。两种选择没有对错，关键是心里有数。

### 3.2.3 验证自动更新真的在工作

```bash
# 查看无人值守升级日志（最近干了什么、装了哪些包）
sudo tail -n 50 /var/log/unattended-upgrades/unattended-upgrades.log

# 只演习不真装（dry-run），检查配置有没有写错
sudo unattended-upgrades --dry-run --debug 2>&1 | head -n 50
```

养成习惯：每月登录服务器时顺手看一眼这个日志，确认自动更新没在报错。

---

## 3.3 内核更新与回滚

### 3.3.1 机制说明

内核是与硬件打交道的核心组件。apt 安装新内核时**不会覆盖旧内核**，新旧内核并存，GRUB 启动菜单里可以选择用哪个启动。正在运行的内核要等重启后才切换到新版本。

```bash
# 查看当前正在运行的内核版本
uname -r

# 查看已安装的全部内核
dpkg -l | grep linux-image

# 查看启动后默认会用的最新内核（下次重启生效）
ls /boot/vmlinuz-*
```

**保留几个内核合适**：保留「当前在用的 + 上一个已知正常的」即可，通常 2–3 个。留太多浪费 `/boot` 空间（每个内核约占数百 MB），只留 1 个则一旦新内核有问题就无处回滚。正常情况下 `apt autoremove`（见 3.4 节）会自动只保留最近的几个，不必手工删。

### 3.3.2 新内核出问题时回滚

症状：升级重启后无法启动、网卡/磁盘驱动异常、服务起不来。处理：

1. 重启机器，在开机时进入 **GRUB 菜单**（物理机在 BIOS 自检后狂按 `Esc` 或按住 `Shift`；云服务器用厂商网页控制台操作，见第 4 章）。
2. 选择 `Advanced options for Ubuntu`（Ubuntu 高级选项）。
3. 在列表里选一个**旧版本号**的内核项（不要选带 `recovery mode` 的，除非要进恢复模式），回车启动。
4. 用旧内核进入系统后，确认服务恢复正常；然后卸载有问题的新内核，防止下次重启又默认进它：

```bash
# 把版本号换成有问题的那一个
sudo apt purge linux-image-6.8.0-XX-generic linux-headers-6.8.0-XX-generic
sudo update-grub
```

> 要点：回滚靠的是「旧内核一直都在」，所以永远不要手动把旧内核删到只剩一个。等官方发布修复版内核后再正常 `apt upgrade` 即可。

---

## 3.4 清理：磁盘空间日常维护

长期运行的服务器，空间主要被四类东西吃掉：旧内核与无用依赖、apt 下载缓存、系统日志、业务/用户的大文件。

### 3.4.1 清理软件包残留

```bash
# 删除不再被任何软件依赖的包（主要是旧内核、旧库）
sudo apt autoremove -y

# 清空 apt 下载缓存（/var/cache/apt/archives 里的 .deb 安装包）
sudo apt clean

# 查看缓存当前占用
du -sh /var/cache/apt
```

`autoremove` 执行前会列出将删除的清单，扫一眼确认没有业务软件即可。它很安全——只删「孤儿依赖」，不会动你手动安装的软件。

### 3.4.2 清理日志

systemd 日志（journal）默认会持续增长，设个上限或定期按时间清理：

```bash
# 看日志总占用
journalctl --disk-usage

# 只保留最近 7 天的日志
sudo journalctl --vacuum-time=7d

# 或：只保留最多 500 MB
sudo journalctl --vacuum-size=500M
```

应用自身的日志（如 nginx、业务程序）在 `/var/log/` 下，由 `logrotate` 自动轮转，一般无需手工处理；发现某个日志异常膨胀，先查程序为什么刷屏，而不是只删文件。

### 3.4.3 找出大文件与大目录

```bash
# 看每个一级目录占用（定位空间去向）
sudo du -h --max-depth=1 / 2>/dev/null | sort -hr | head -n 20

# 找出 /var 下大于 500 MB 的文件
sudo find /var -type f -size +500M -exec ls -lh {} \; 2>/dev/null

# 交互式浏览磁盘占用（更好用，需安装）
sudo apt install -y ncdu
sudo ncdu /
```

`ncdu` 里用方向键进入目录、`d` 删除（谨慎）、`q` 退出。删任何不认识的大文件前先确认它属于哪个服务——数据库文件、Docker 镜像层都很大但不能直接删。

> Docker 用户额外注意：镜像与停止的容器是吃盘大户，用 `docker system df` 查看、`docker system prune` 清理，不要手动删 `/var/lib/docker`。

---

## 3.5 大版本升级：do-release-upgrade

把 22.04 升到 24.04 这类跨 LTS 升级，用专用工具 `do-release-upgrade`。它会换源、处理依赖冲突、逐项询问配置冲突，是「有风险但有正规流程」的升级。

### 3.5.1 升级前铁律

1. **先完整备份**（见 3.6 节）：系统盘快照或整机镜像，确认可恢复后再动手。升级过程**没有撤销按钮**。
2. **先在测试机/克隆机上跑一遍**：同样的软件、同样的配置，确认业务在新版本上能跑，再动生产机。
3. **先把当前系统更新到最新**：旧版本内部的小版本补丁没打全时跨版本升级最容易失败。
4. **在 tmux/screen 里跑**：升级耗时几十分钟且会重启服务，SSH 断线可能导致升级中断在危险的半成品状态。用会话工具兜住：

```bash
sudo apt install -y tmux
tmux new -s upgrade
# 在 tmux 里执行下面的升级命令；即使 SSH 断开，重新连上后 tmux attach -t upgrade 即可回到现场
```

5. 移除第三方源与 PPA：非官方源最容易在跨版本时造成依赖地狱，升级前在 `/etc/apt/sources.list.d/` 中暂时注释或删除它们，升级后再按新版本重新添加。

### 3.5.2 标准流程

```bash
# 1）当前版本打满补丁并重启一次
sudo apt update && sudo apt full-upgrade -y
sudo reboot

# 2）确认升级工具存在
sudo apt install -y update-manager-core

# 3）（云服务器/特殊环境）检查是否有新 LTS 可用
sudo do-release-upgrade -c

# 4）正式开始（在 tmux 里执行）
sudo do-release-upgrade
```

过程要点：

- 工具会先给出变更摘要（要装多少、删多少），**认真看删除清单**再确认。
- 中途遇到配置文件冲突询问（`keep / install` 选择），自己改过的配置选 keep（保留）；没印象改过的用新版本默认即可。拿不准时先选查看差异（diff）。
- 全部完成后按提示重启，重启后用 `lsb_release -a` 与 `uname -r` 确认已是新版本，并逐个验证业务服务。

### 3.5.3 风险提示

- 耗时：视机器与网络，30 分钟到 2 小时不等，期间服务会中断，**不要在业务高峰做**。
- PHP、Python、数据库等运行时大版本会随系统一起升，代码兼容性要在测试机先验。
- LTS 之间官方只支持逐级升（22.04 → 24.04 → 26.04），不能跳级；且一般要等新 LTS 的第一个小版本（如 24.04.1）发布后，`do-release-upgrade` 才会向旧 LTS 用户开放升级通道——刚发布时查不到是正常现象。
- 如果机器太老、改动太多，「备份数据 → 全新安装新版本 → 恢复数据」往往比原地升级更干净可靠，这也是完全正当的选择（第 1 章流程重走一遍即可）。

---

## 3.6 备份与快照思路

更新维护的最后一块拼图是退路。按成本从低到高，常见方案：

| 方案 | 备份什么 | 恢复粒度 | 适合场景 |
|------|----------|----------|----------|
| 云厂商磁盘快照 | 整块系统盘 | 整盘回滚 | 云服务器大改动/升级前，一键最省事 |
| LVM 快照 | 某个逻辑卷在某时刻的状态 | 卷级回滚 | 本地机用了 LVM（第 1 章）时，升级前打一个 |
| Timeshift | 系统文件（不含用户数据） | 系统级还原 | 桌面及轻量服务器的系统自救 |
| rsync 定时同步 | 指定的数据目录 | 文件级 | 重要数据异地/异盘留副本，成本最低 |

### 3.6.1 LVM 快照（升级前快速留退路）

前提是系统用了 LVM 且卷组有空闲空间：

```bash
# 给根卷打一个 10 GB 的快照（名字自取）
sudo lvcreate -L 10G -s -n root-snap /dev/ubuntu-vg/ubuntu-lv

# 出问题时合并回滚（需要卸载/重启配合，按提示操作）
# sudo lvconvert --merge /dev/ubuntu-vg/root-snap

# 确认无事后删除快照（快照长期留着会拖慢写入并占空间）
sudo lvremove /dev/ubuntu-vg/root-snap
```

### 3.6.2 rsync 同步重要数据

```bash
# 把 /data 同步到备份盘（首次全量，之后只传增量；末尾斜杠不要漏）
sudo rsync -aH --info=progress2 /data/ /mnt/backup/data/

# 同步到另一台服务器（经 SSH，<备份服务器IP> 替换为实际地址）
sudo rsync -aH -e ssh /data/ user1@198.51.100.20:/backup/data/
```

配合第 2 章思路，用 cron 每天凌晨跑一次即可实现简易自动备份。注意：rsync 同步是「镜像」，源端误删会同步掉副本，重要数据还应保留带日期的多份副本或开启备份端的快照。

### 3.6.3 Timeshift 简介

```bash
sudo apt install -y timeshift
```

Timeshift 底层用 rsync 或 btrfs 快照，强项是「把系统还原到某个时间点」，默认**不备份用户主目录数据**——它救系统，不救数据。定位与云快照/LVM 快照类似，都是升级前的保险，三者选顺手的一个即可，不要叠加互斥的方案把问题搞复杂。

> 备份的验收标准只有一个：**真的恢复过一次**。从没验证过恢复流程的备份，只能算「有份拷贝」。新方案上线时，务必做一次演练恢复。

---

## 本章小结

- `apt update` 只刷新目录，`upgrade` 才真正安装；`full-upgrade` 能增删依赖，处理被保留的包。
- 开启 `unattended-upgrades` 自动打安全补丁；自动重启与否按业务性质决定，并定期看日志。
- 内核新旧并存是回滚的资本，保留 2–3 个；GRUB 选旧内核即可自救。
- 清理三板斧：`autoremove`、`apt clean`、`journalctl --vacuum-time`；找大文件用 `ncdu`。
- 大版本升级：备份 → 测试机先行 → tmux 里跑 `do-release-upgrade` → 逐项验证。

---

*导航：上一章：[第 2 章 初始设置](02-initial-setup.md) | 下一章：[第 4 章 连接方式总览与具体步骤](04-remote-access.md) | 返回目录：[README](README.md)*
