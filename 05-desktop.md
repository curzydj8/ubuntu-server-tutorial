# 第 5 章 桌面环境

Ubuntu Server 默认没有图形界面，这是好事：少占资源、少开端口、攻击面小。但有些场景确实需要桌面——远程办公、跑图形软件、在服务器上直接调试、给不熟悉命令行的同事一个可视化入口。本章讲清楚主流桌面怎么装、怎么切换、怎么卸，以及服务器场景下必须注意的几个坑。

> 主线版本：Ubuntu 24.04 LTS。文中内存占用为**空闲状态量级**，不同机器、驱动、后台服务差异很大，仅供选型参考，不要当作精确指标。

## 5.1 先想清楚：你要的到底是什么

装桌面之前先分清三件事：

| 需求 | 推荐做法 |
| --- | --- |
| 只是偶尔打开一个图形程序 | 不装完整桌面，用 SSH 的 X11 转发或单独装应用即可 |
| 长期远程桌面办公 | 装轻量桌面（XFCE / LXQt）+ 远程桌面服务 |
| 本地接显示器当工作站用 | 按喜好选 GNOME 或 KDE |
| 纯跑服务、从不看画面 | 不装桌面，保持 headless（见 5.9） |

**是什么**：桌面环境（Desktop Environment，DE）是一整套东西——窗口管理器、面板、文件管理器、设置中心、默认应用集合，不是一个单独的软件。

**为什么**：Ubuntu 用「元包（metapackage）」管理桌面。元包本身几乎不含程序，它只声明依赖，`apt` 靠它把一整套组件拉齐。

**改了会怎样**：装桌面会一次性多装几百个包、多占 2–8 GB 磁盘，并引入显示管理器常驻服务。卸载时如果操作不当，容易把共享库误删（见 5.6）。

统一前置操作：

```bash
sudo apt update
```

## 5.2 六个主流桌面

### 5.2.1 GNOME —— Ubuntu 官方默认

```bash
# 完整版：桌面 + 全套默认应用
sudo apt install ubuntu-desktop

# 精简版：桌面核心 + 基础工具，应用少很多
sudo apt install ubuntu-desktop-minimal
```

- **特点**：界面现代、触控支持好、与 Ubuntu 官方版本最一致，问题最容易搜到答案。`ubuntu-desktop-minimal` 是 22.04 起提供的精简元包，少装办公套件等大块应用，更适合服务器上「只要桌面骨架」的场景。
- **空闲内存占用**：约 900 MB – 1.5 GB，仅供参考。
- **适用场景**：本地显示器使用、追求开箱即用、不想折腾的人。

### 5.2.2 KDE Plasma —— 功能最全、最像传统 Windows

```bash
# 完整版：KDE 全家桶
sudo apt install kubuntu-desktop

# 标准版：Plasma 桌面 + 常用 KDE 应用，体积小一截
sudo apt install kde-standard
```

- **特点**：可定制程度最高，面板、菜单、快捷键几乎都能改；自带应用（Dolphin 文件管理器、Konsole 终端）质量高。
- **空闲内存占用**：约 700 MB – 1.1 GB，仅供参考。别被「功能多」吓住，现代 Plasma 本身并不臃肿。
- **适用场景**：颜值党、重度定制党、从 Windows 迁移过来的用户。

### 5.2.3 XFCE —— 轻量、稳定、远程桌面首选

```bash
# 完整版：Xubuntu 的整套桌面
sudo apt install xubuntu-desktop

# 精简版：只装 XFCE 核心 + 常用插件
sudo apt install xfce4 xfce4-goodies
```

- **特点**：十几年界面逻辑基本没变，稳定、 predictable（行为可预期）；组件模块化，不需要的可以不装。`xfce4-goodies` 是一组官方插件与小工具（截图、剪贴板、系统负载监视等）。
- **空闲内存占用**：约 350 – 600 MB，仅供参考。
- **适用场景**：老机器、低配云主机、远程桌面。

**为什么 XFCE 在远程场景特别省资源？** 三个原因：

1. **没有重型合成特效**：GNOME/KDE 默认开合成器做动画、阴影、模糊，远程传输时每一帧变化都要编码发送；XFCE 默认特效极少，画面变化量小，带宽和编码开销都低。
2. **对远程桌面协议友好**：xrdp、VNC 等方案与 XFCE 的兼容历史最久，黑屏、花屏、会话冲突这类问题明显少于 GNOME。
3. **常驻进程少**：没有索引、搜索、在线账户一堆后台服务抢内存，老机器和 1–2 GB 内存的小主机也能流畅跑。

### 5.2.4 LXQt —— 比 XFCE 还轻

```bash
sudo apt install lubuntu-desktop
```

- **特点**：用 Qt 写成，目标就是「轻」。界面朴素，功能够用但生态和美化程度不如前几位。
- **空闲内存占用**：约 300 – 500 MB，仅供参考。
- **适用场景**：内存 ≤2 GB 的老机器、只求「有个图形界面能点」的服务器。

### 5.2.5 MATE —— 经典 GNOME 2 的延续

```bash
sudo apt install ubuntu-mate-desktop
```

- **特点**：传统菜单 + 面板布局，上手零学习成本；对老硬件友好，社区维护稳定。
- **空闲内存占用**：约 500 – 800 MB，仅供参考。
- **适用场景**：怀旧用户、给长辈或新手用的机器、老机器焕新。

### 5.2.6 Cinnamon —— Linux Mint 的当家桌面

```bash
sudo apt install cinnamon-desktop-environment
```

- **特点**：布局像 Windows 7：底部面板、开始菜单、系统托盘一应俱全，过渡成本最低；颜值在传统布局里算好看的。
- **空闲内存占用**：约 700 MB – 1.0 GB，仅供参考。
- **适用场景**：Windows 用户第一次上 Linux、颜值与习惯兼顾。

### 5.2.7 速览对照表

| 桌面 | 元包 | 空闲内存（参考） | 一句话定位 |
| --- | --- | --- | --- |
| GNOME | `ubuntu-desktop` / `ubuntu-desktop-minimal` | 900 MB – 1.5 GB | 官方默认，省心 |
| KDE Plasma | `kubuntu-desktop` / `kde-standard` | 700 MB – 1.1 GB | 最能定制 |
| XFCE | `xubuntu-desktop` / `xfce4` | 350 – 600 MB | 轻量稳定，远程首选 |
| LXQt | `lubuntu-desktop` | 300 – 500 MB | 最轻 |
| MATE | `ubuntu-mate-desktop` | 500 – 800 MB | 经典布局 |
| Cinnamon | `cinnamon-desktop-environment` | 700 MB – 1.0 GB | 最像 Windows |

## 5.3 用 tasksel 一次勾选

不想记元包名字，可以用 `tasksel`——Debian/Ubuntu 传统的任务选择工具，交互式勾选整套软件组：

```bash
sudo apt install tasksel
sudo tasksel
```

**是什么**：一个文本界面工具，列出「Ubuntu desktop」「Xubuntu desktop」「SSH server」等预定义任务组，空格勾选、Tab 确认后统一安装。

**为什么**：安装系统时见过的那个选择界面就是它，适合一次装多组软件。

**改了会怎样**：效果与手动 `apt install` 元包基本一致。注意：**不要用 tasksel 卸载**——它卸载时依赖计算比较粗暴，有误删共享组件的案例，卸载老老实实用 `apt purge`（见 5.6）。

## 5.4 显示管理器：登录界面谁来管

显示管理器（Display Manager，DM）就是开机后那个图形登录界面，它负责验证密码并拉起桌面会话。

| 显示管理器 | 默认搭配 | 特点 |
| --- | --- | --- |
| `gdm3` | GNOME | 与 GNOME 集成最深，支持 Wayland |
| `lightdm` | XFCE、MATE、Cinnamon、LXQt 等多数轻量桌面 | 轻、快，配置就是改一个文本文件 |
| `sddm` | KDE Plasma | Qt 写的，与 Plasma 主题统一 |

### 安装时怎么选

安装桌面元包时，如果依赖里带了显示管理器，`apt` 会弹出一个 debconf 蓝色界面让你选默认 DM，用方向键选、回车确认即可。选错了没关系，事后可以改。

### 事后切换

```bash
# 把 <dm> 换成 gdm3 / lightdm / sddm 中已安装的那个
sudo dpkg-reconfigure gdm3

# 查看当前实际在跑的显示管理器
systemctl status display-manager
```

**是什么**：`display-manager.service` 是一个链接，指向当前生效的 DM。`dpkg-reconfigure` 会重新走一遍选择流程并改掉这个链接。

**为什么**：一台机器同一时间只能有一个 DM 占着登录界面，多个 DM 装着没问题，但生效的只有一个。

**改了会怎样**：切换后需要重启（或 `sudo systemctl restart display-manager`，会断开当前图形会话）才生效。

> 经验法则：桌面和 DM 尽量配对使用——GNOME 配 gdm3、KDE 配 sddm、其他轻量桌面配 lightdm。混搭能跑，但主题、锁屏、电源管理容易出现小毛病。

## 5.5 登录后切换桌面、多桌面共存的代价

多个桌面可以共存。在登录界面：

1. 先点用户名（或输入用户名）；
2. 点密码框附近 / 屏幕角落的**齿轮图标**；
3. 在列表里选要进入的会话（如 `Ubuntu`、`Xfce Session`、`Plasma`）；
4. 再输入密码登录。

系统会记住上次的选择，下次默认进同一个。

**共存的代价，要提前知道：**

- **应用重复**：每个桌面自带一套文件管理器、终端、文本编辑器、图片查看器，应用菜单里会出现两三个功能一样的东西，新手容易点错。
- **占磁盘**：GNOME + KDE 共存在 6–10 GB 量级并不稀奇。
- **配置互相污染**：GTK/Qt 主题、默认应用关联是共享的，在 A 桌面改了设置，B 桌面可能跟着变。
- **建议**：服务器上只装一个桌面。真要体验，先装轻量的试，不满意卸载后再换。

## 5.6 卸载桌面

```bash
# 1. 先卸元包本身（把 <元包名> 换成你装的那个）
sudo apt purge ubuntu-desktop-minimal

# 2. 再清理它拉进来、且无人再用的依赖
sudo apt autoremove --purge
```

注意事项：

- **先 purge 元包，再 autoremove**，顺序反了清不干净。
- 执行 `autoremove` 前**认真看一遍它列出的待删清单**。如果清单里出现你其他软件在用的库（比如某个服务依赖的 Qt 库），先按 `n` 取消，查清再动手。这是桌面卸载翻车的高发区。
- 卸载前最好切回命令行或至少退出图形会话，避免删到一半会话崩掉。
- 用户目录下的配置（`~/.config` 等）不会被删，重装同一桌面时旧设置会回来；想彻底重置要手动清家目录配置，这一步不可逆，先备份。

## 5.7 自动登录与关闭锁屏（服务器场景）

### 自动登录

远程服务器重启后如果卡在登录界面，远程桌面就进不去，所以很多人会开自动登录：指定用户开机直接进桌面，不用输密码。

**LightDM（XFCE 等轻量桌面常用）：**

编辑 `/etc/lightdm/lightdm.conf`，在 `[Seat:*]` 段下加：

```ini
[Seat:*]
autologin-user=你的用户名
autologin-user-timeout=0
```

没有这个文件就新建，段名不能写错。

**GDM3（GNOME）：**

编辑 `/etc/gdm3/custom.conf`，在 `[daemon]` 段下加：

```ini
[daemon]
AutomaticLoginEnable=true
AutomaticLogin=你的用户名
```

改完重启生效：

```bash
sudo reboot
```

> ⚠️ **风险提示**：自动登录等于「任何人能碰到这台机器（或它的控制台/远程会话），就能直接拿到已登录桌面」。只在物理环境可信、或虚机控制台本身有密码保护时开启；公网可达、机房托管、多人共用的机器不要开。另外，开了全盘加密的机器开机仍需先解密，自动登录帮不了这一步。

### 关闭锁屏与屏保

服务器 + 远程桌面的组合里，锁屏和屏保是纯麻烦：远程连进来一片黑、输密码解屏保偶尔还会和远程会话冲突，而一台 7×24 跑服务的机器根本不需要靠屏保「保护屏幕」。所以这类机器通常关掉它们。

**XFCE：**

设置 → 电源管理 → 显示，把「显示器休眠/关闭」设为从不；再打开「会话与启动」或屏幕保护程序设置，禁用锁屏。也可以命令行处理 screensaver：

```bash
# XFCE 若装了 xscreensaver，直接禁用其自启并杀掉进程
xfconf-query -c xfce4-session -p /general/LockCommand -s ""
pkill xscreensaver 2>/dev/null || true
```

**GNOME（gsettings）：**

```bash
# 关闭自动锁屏
gsettings set org.gnome.desktop.screensaver lock-enabled false
# 屏幕永不熄灭（0 = 从不）
gsettings set org.gnome.desktop.session idle-delay 0
# 合盖/闲置不挂起（可选，按需）
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
```

**xscreensaver 注意事项**：一些轻量桌面默认带 `xscreensaver`，它有自己独立的配置和锁屏逻辑，系统设置里关了它可能还在锁。确认它在不在：`ps aux | grep xscreensaver`；不需要就直接卸载，一了百了：

```bash
sudo apt purge xscreensaver
```

## 5.8 headless 机器到底需不需要桌面

不需要，是默认答案。

- 纯跑 Web、数据库、脚本、Docker 的服务器，桌面是纯开销：多占内存磁盘、多一堆要打补丁的软件、多一个攻击面。
- 「我想看状态」用监控面板（如命令行 `htop`、Web 面板）解决；「我想传文件」用 SFTP/SCP 解决；「我不会命令行」才是装桌面的正当理由——那就装最轻的 XFCE/LXQt，用完还能卸。
- 判断标准一句话：**桌面是给人坐的，不是给服务跑的。** 没有真人要长期看这块屏幕，就别装。

## 5.9 startx：不用显示管理器，手动进桌面

装了桌面组件但没装/没启用 DM 时，可以在本地 TTY 登录后手动拉起图形界面：

```bash
# 前提：已装 xorg 和某个桌面（如 xfce4）
sudo apt install xorg xfce4
startx
```

**是什么**：`startx` 读 `~/.xinitrc`（没有就用系统默认）启动 X 会话，退出桌面就回到字符终端。

**适用场景**：临时用一下图形、调试 DM 起不来的问题、极简机器。远程服务器日常运维用不上它，了解即可。

## 小结

- 选型口诀：省心选 GNOME、定制选 KDE、远程/老机器选 XFCE、极致轻量选 LXQt；
- DM 与桌面配对，切换用 `dpkg-reconfigure`，查状态看 `display-manager`；
- 服务器桌面两件套：按需开自动登录（先评估物理风险）、关掉锁屏屏保；
- 没人长期看屏幕的机器，保持 headless 是最优解。

---

上一章：[04-remote-access.md](04-remote-access.md) ｜ 下一章：[06-applications.md](06-applications.md) ｜ 返回目录：[README](README.md)
