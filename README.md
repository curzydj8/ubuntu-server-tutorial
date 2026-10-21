# Ubuntu Server 安装、连接与使用全套教程

一套从零开始的 Ubuntu Server 中文教程：从做启动盘、装系统、连上去，到装桌面、装软件、写 Python 服务、再到用脚本模拟键盘鼠标做远程自动化。每章的命令都可以直接复制执行，关键设置都会讲清「是什么、为什么、改了会怎样」。第 12~14 章是**实战篇**，记录笔者自己几台机器的真实设置、用过的真实代码和踩过的坑。

- **主线版本**：Ubuntu Server 24.04 LTS（兼顾 22.04，并注明差异）
- **典型场景**：一台放在家里/机房/云上的 Ubuntu 服务器，用 SSH、Tailscale、远程桌面等方式连接和使用
- **阅读方式**：新手按章节顺序读；老手直接跳到需要的那一章，每章末尾都有导航

## 章节目录

| 章 | 标题 | 内容概要 |
|----|------|----------|
| 01 | [系统安装](01-install.md) | LTS 版本选择、下载校验、U 盘启动盘、Subiquity 安装全步骤、分区与 LVM 建议、务必勾选 OpenSSH、装后验收 |
| 02 | [初始设置](02-initial-setup.md) | 用户与 sudo、主机名、时区与语言、netplan 静态 IP、ufw 防火墙、确认 SSH 服务 |
| 03 | [更新与维护](03-update-maintenance.md) | apt update/upgrade 的区别、自动安全更新、内核更新与回滚、清理磁盘、do-release-upgrade、备份与快照思路 |
| 04 | [连接方式总览与具体步骤](04-remote-access.md) | SSH（密码/密钥/配置文件/端口转发/跳板机）、Tailscale、VNC、xrdp、NoMachine、串口与云控制台的对比和完整步骤 |
| 05 | [桌面环境](05-desktop.md) | GNOME / KDE / XFCE / LXQt / MATE / Cinnamon 的特点与安装、显示管理器切换、多桌面共存与卸载、自动登录与关闭锁屏 |
| 06 | [常用应用安装](06-applications.md) | Firefox / Chrome / VS Code、终端工具箱、Git、Docker、Node.js、数据库的安装代码与验证命令 |
| 07 | [Python 使用](07-python.md) | 系统 Python 红线、venv 虚拟环境、pip 国内镜像、pipx / pyenv / conda、systemd 常驻服务、cron 定时任务、PEP 668 报错解法 |
| 08 | [模拟键盘鼠标](08-input-simulation.md) | xdotool（X11）与 ydotool（uinput/Wayland）、uinput 原理与权限、pyautogui 等 Python 方案、SSH 下操作桌面会话 |
| 09 | [远程控制进阶](09-remote-automation.md) | 远程截图、窗口管理、剪贴板、批量动作脚本化、无人值守自动化的注意事项 |
| 10 | [浏览器](10-browser.md) | 从 SSH 打开图形浏览器、远程调试端口、无头模式、代理设置、多数据目录隔离 |
| 11 | [FAQ 与排障清单](11-faq.md) | SSH 连不上、xrdp 黑屏、磁盘满、服务起不来、网络不通等常见问题的排查命令清单 |
| 12 | [机器清单与设置（实战环境）](12-machines.md) | 笔者在用的几台机器逐台摊开：主力机 zhang-new、服务器 muse、手机控制端与沙盒 VM 的角色、关键设置清单，含自动登录与永久关锁屏的做法 |
| 13 | [各种 Key 的添加与管理](13-keys.md) | SSH 密钥生成与权限、公钥加到 GitHub 与服务器（含「加错机器的 key」实战坑）、API Key 的三种放法、泄露红线与吊销处置 |
| 14 | [实战操作代码](14-field-ops.md) | 真实跑过的六组代码：SSH 保活、HTTP CONNECT 隧道脚本、远程启动 GUI 程序与验证三件套、xdotool 定向发键、GitHub API 全量推送、v2rayN 配置坑 |

## 推荐阅读路径

**第一次装机（从零到能用）**

01 系统安装 → 02 初始设置 → 04 连接方式（先只看 SSH 与 Tailscale 两节）→ 03 更新与维护

**需要图形界面时**

05 桌面环境 → 06 常用应用安装 → 04 连接方式（VNC / xrdp / NoMachine 三节）

**跑服务与写代码时**

07 Python 使用 → 06 常用应用安装（Docker 一节）→ 03 更新与维护（备份一节）

**做远程自动化时**

05 桌面环境（先关掉锁屏）→ 08 模拟键盘鼠标 → 09 远程控制进阶 → 10 浏览器

**实战复盘（我们实际跑过的环境与代码）**

12 机器清单与设置 → 13 各种 Key 的添加与管理 → 14 实战操作代码

**出问题时**

直接翻 11 FAQ 与排障清单，按现象对号入座。

## 约定

- 命令默认在 Ubuntu Server 上以普通用户执行，需要管理员权限的命令都带 `sudo`。
- 示例中的 IP 地址使用文档保留段 `192.0.2.x`、`198.51.100.x` 或 `<服务器IP>` 占位符，请替换成你自己的地址。
- 涉及密码、密钥的内容只讲生成与存放方法，不出现任何真实凭据。
