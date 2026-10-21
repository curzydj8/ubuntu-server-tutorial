# 第 9 章 远程控制进阶

上一章解决了「怎么发按键」。这一章解决远程控制的另外四件事：**看得见**（截图）、**找得到**（窗口管理）、**传得出**（剪贴板）、**跑得稳**（脚本化与无人值守）。

场景设定不变：Ubuntu 24.04 LTS + XFCE（X11 会话），操作者通过 SSH 远程连接，桌面会话在服务器本地保持登录。

## 9.1 截图：远程「看一眼」桌面

截图工具都在 X11 会话里工作，SSH 下使用时同样要先带上 `DISPLAY`（原理见 8.6 节）。

### 9.1.1 三个常用工具

```bash
sudo apt update
sudo apt install scrot imagemagick gnome-screenshot
```

| 工具 | 特点 | 适合场景 |
|---|---|---|
| `scrot` | 轻量、参数简单、纯命令行 | 脚本定时截图 |
| ImageMagick `import` | 功能全，可裁剪、指定窗口 | 需要只截某个区域/窗口 |
| `gnome-screenshot` | GNOME 出品，XFCE 下也能用 | 想要延时、含指针等选项 |

### 9.1.2 基本用法

```bash
# 先确保在桌面会话环境下（SSH 里执行必做，本机终端可跳过）
export DISPLAY=:0
export XAUTHORITY=~/.Xauthority

# scrot：全屏截图
scrot ~/Pictures/shot.png

# scrot：延时 3 秒后截图（留时间切换窗口）
scrot -d 3 ~/Pictures/shot-delay.png

# scrot：截当前活动窗口（带边框）
scrot -u -b ~/Pictures/window.png

# scrot：文件名带时间戳，避免覆盖
scrot ~/Pictures/'shot-%Y%m%d-%H%M%S.png'

# ImageMagick import：全屏
import -window root ~/Pictures/full.png

# ImageMagick import：按窗口 ID 截图、裁剪区域
import -window <窗口ID> ~/Pictures/win.png
import -window root -crop 800x600+100+50 ~/Pictures/crop.png

# gnome-screenshot：全屏 / 当前窗口 / 延时
gnome-screenshot -f ~/Pictures/g1.png
gnome-screenshot -w -f ~/Pictures/g2.png
gnome-screenshot -d 3 -f ~/Pictures/g3.png
```

### 9.1.3 完整流程：SSH 截图并取回本地

```bash
# 第 1 步：SSH 登录服务器（示例 IP 为文档保留段，按实际替换）
ssh <用户名>@192.0.2.10

# 第 2 步：在服务器上带 DISPLAY 截图
export DISPLAY=:0
export XAUTHORITY=~/.Xauthority
scrot /tmp/desktop-now.png
ls -lh /tmp/desktop-now.png
```

```bash
# 第 3 步：回到本地终端，用 scp 取回
scp <用户名>@192.0.2.10:/tmp/desktop-now.png ./desktop-now.png
```

一条命令版（在本地执行）：

```bash
ssh <用户名>@192.0.2.10 'DISPLAY=:0 XAUTHORITY=$HOME/.Xauthority scrot /tmp/s.png' \
  && scp <用户名>@192.0.2.10:/tmp/s.png ./s.png
```

排查顺序：截图文件是纯黑图 → 多半是锁屏或会话不是 active；报 `Can't open display` → `DISPLAY` 没带对；文件生成了但打不开 → 传输中断，重新 `scp` 一次即可。

> Wayland 提示：GNOME Wayland 下上述工具多数失效，截图要走 `xdg-desktop-portal`（如 `gnome-screenshot` 新版、Flameshot 的 Wayland 模式），且通常要求本地有人确认授权，不适合无人值守。这也是本教程坚持 X11 会话的原因之一。

## 9.2 窗口管理：先定位，再操作

远程自动化最怕「窗口一多就发错」。原则：**任何发键之前，先用窗口工具点名目标窗口。**

### 9.2.1 wmctrl

```bash
sudo apt install wmctrl

wmctrl -l              # 列出所有窗口：ID、桌面号、机器名、标题
wmctrl -l -x           # 附带窗口类名（WM_CLASS），同名窗口靠它区分
wmctrl -a "Firefox"    # 激活标题包含 Firefox 的窗口
wmctrl -c "Firefox"    # 关闭标题匹配的窗口（发关闭消息，不是 kill）
wmctrl -r "终端" -e 0,100,50,1200,800   # 移动/缩放：桌面,x,y,宽,高
wmctrl -r :ACTIVE: -b toggle,maximized_vert,maximized_horz  # 当前窗口最大化切换
```

### 9.2.2 xdotool 的窗口检索

```bash
# 按标题正则搜索，输出窗口 ID（可多个）
xdotool search --name "设置"

# 只看可见窗口、限定当前桌面
xdotool search --onlyvisible --name "设置"

# 查指定窗口的标题 / 几何信息
xdotool getwindowname <窗口ID>
xdotool getwindowgeometry <窗口ID>

# 当前活动窗口是谁
xdotool getactivewindow getwindowname
```

### 9.2.3 多窗口定位套路

同名窗口（比如开了三个终端）是最常见的坑，组合打法：

```bash
# 1) 用类名+标题缩小范围
wmctrl -l -x

# 2) 逐个取 ID 并核对标题与位置，选对再激活
for id in $(xdotool search --class "xfce4-terminal"); do
  echo "$id  $(xdotool getwindowname $id)  $(xdotool getwindowgeometry $id)"
done

# 3) 激活并二次确认，然后才发按键
xdotool windowactivate --sync <窗口ID>
test "$(xdotool getactivewindow)" = "<窗口ID>" && xdotool key ctrl+l
```

记住：`windowactivate` 之后加一次校验，比直接发键多一行代码，但能挡住绝大多数「发错窗口」事故。

## 9.3 剪贴板：SSH 与桌面之间传文本

X11 剪贴板属于桌面会话，SSH 终端的复制粘贴和它互不相通。用 `xclip` / `xsel` 可以在命令行直接读写桌面剪贴板，长命令、配置片段不用再靠手敲。

```bash
sudo apt install xclip xsel
export DISPLAY=:0
export XAUTHORITY=~/.Xauthority

# 写入剪贴板（之后在桌面里 Ctrl+V 即可粘贴）
echo -n "一串很长的命令 --with-many-flags" | xclip -selection clipboard
cat config片段.txt | xclip -selection clipboard

# 读出剪贴板内容
xclip -selection clipboard -o
xsel --clipboard --output        # xsel 等价写法

# 查看剪贴板里有哪些格式（图片、富文本等）
xclip -selection clipboard -t TARGETS -o
```

实用技巧：

- **把本地文本送进远程桌面剪贴板**：本地执行 `ssh <用户名>@192.0.2.10 'DISPLAY=:0 xclip -selection clipboard' < 本地文件.txt`，到远程桌面直接粘贴。
- 注意 X11 有两个选择区：鼠标划选的是 `PRIMARY`（`-selection primary`），Ctrl+C/V 用的是 `CLIPBOARD`。读写对不上时先检查是不是选错了区。
- X11 剪贴板由「拥有它的程序」保管，那个程序退出后内容即消失——粘贴前别把源程序关掉。

## 9.4 批量动作脚本化

把前几节拼起来，就是可重复执行的自动化脚本。写脚本的三条纪律：**每步可校验、失败要重试、过程留日志。**

```bash
#!/usr/bin/env bash
# 示例：打开终端，查看服务状态，截图留档
set -u
export DISPLAY=:0
export XAUTHORITY="$HOME/.Xauthority"

LOG="$HOME/automation.log"
log() { echo "[$(date '+%F %T')] $*" | tee -a "$LOG"; }

# 带重试的窗口等待：最多等 10 秒
wait_window() {
  local name="$1" id=""
  for i in $(seq 1 20); do
    id=$(xdotool search --onlyvisible --name "$name" | head -n1)
    [ -n "$id" ] && { echo "$id"; return 0; }
    sleep 0.5
  done
  return 1
}

log "开始执行"

xfce4-terminal --title auto-term &
if ! WIN_ID=$(wait_window "auto-term"); then
  log "错误：终端窗口未出现，退出"
  exit 1
fi

xdotool windowactivate --sync "$WIN_ID"
sleep 0.3
xdotool type --delay 20 "systemctl status ssh --no-pager"
xdotool key Return
sleep 1

scrot "$HOME/Pictures/status-$(date +%H%M%S).png"
log "完成，截图已保存"
```

要点拆解：

- **失败重试**：窗口出现、程序启动这类异步事件，一律用「循环等待 + 超时退出」，不要赌固定 `sleep`。
- **日志**：每步打时间戳写日志文件；无人值守跑出问题的唯一线索就是日志。关键节点顺手截图，事后复盘有图有真相。
- **幂等**：脚本重跑一遍不应出乱子（比如先检测窗口已存在就复用，而不是再开一个）。
- **退出码**：失败就 `exit 1`，交给 cron / systemd 时能触发告警，而不是悄悄「假成功」。

## 9.5 无人值守注意事项

脚本写得再好，会话环境一变就全盘失效。长期跑之前，把这几项处理掉：

### 9.5.1 锁屏与屏保

- 锁屏后，截图变黑图，合成按键多半被锁屏程序吞掉，自动化等于停工。
- XFCE 下关闭自动锁屏：设置 → 电源管理里关掉「锁屏」，并在「屏幕保护程序」设置里禁用自动启动；或命令行检查当前状态后再决定：

```bash
xfce4-screensaver-command -q   # 查看屏保状态（若在用 xfce4-screensaver）
xset -display :0 q             # 查看 X11 屏幕空白/DPMS 设置
xset -display :0 s off -dpms   # 关闭 X 层屏保与显示器省电（当前会话生效）
```

- 注意区分「显示器休眠」和「锁屏」：只关显示不锁屏，截图工具照样能截到帧缓冲内容；一旦锁屏，输入通道就断了。

### 9.5.2 保持会话登录与禁止休眠

- 桌面会话必须**一直有人登录着**：注销 = X Server 关闭，所有 DISPLAY 工具全部失效。远程维护时别手滑点注销。
- 屏蔽系统休眠/挂起目标，避免机器自己睡过去：

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
# 恢复：sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
systemctl status sleep.target   # 显示 masked 即已屏蔽
```

- 笔记本合盖、电源计划里的「无操作自动挂起」也要一并关掉（XFCE 电源管理设置内）。

### 9.5.3 远程桌面断开后，会话还在吗

这是最容易混淆的一点，分两种情况：

| 方式 | 会话本质 | 断开后 | 与物理桌面共享画面吗 |
|---|---|---|---|
| xrdp | 独立的虚拟会话，与物理登录互不相同 | 会话与程序继续运行，重连可回到原会话（取决于配置） | ❌ 不共享，是另一套桌面 |
| VNC 接管物理会话（如 x11vnc 连 `:0`） | 物理会话的镜像 | 物理会话在，会话就在 | ✅ 同一块屏幕 |
| SSH + `DISPLAY=:0` | 借用物理会话 | 物理会话在就有效 | ✅ 操作的就是物理桌面 |

所以：用 xrdp 远程时，你在 xrdp 里开的程序，走到机器跟前是看不到的；反过来，物理桌面上开的程序，xrdp 里也看不到。自动化脚本要发给哪个会话，先想清楚它的 `DISPLAY` 是多少（xrdp 会话通常是 `:10` 起）。

## 9.6 安全提醒

- **自动化脚本里不要硬编码密码**：不要把 `echo "密码" | sudo -S`、明文 token 写进脚本。脚本会进备份、进 Git、被日志带出来，泄露面远超你的预期。需要提权就给具体命令配 sudoers 白名单，或改用密钥文件并设 `chmod 600`。
- 截图可能截到敏感内容（聊天、密钥、账单）：截图目录按私有目录管理，定期清理，别顺手同步到公开位置。
- 剪贴板里常驻着密码管理器复制的内容：脚本读剪贴板前先确认内容来源，别把剪贴板原文写进日志。
- 无人值守 ≠ 无人负责：长期任务至少保留日志轮转与磁盘占用检查（见第 11 章磁盘满条目）。

---

**导航**

- 上一章：[第 8 章 模拟键盘鼠标](08-input-simulation.md)
- 下一章：[第 10 章 浏览器](10-browser.md)
- 返回目录：[README](README.md)
