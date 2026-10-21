# 第 8 章 模拟键盘鼠标

在服务器上跑桌面，最常见的自动化需求就是「替人按键、替人点鼠标」：自动登录某个程序、定时在窗口里输入命令、给游戏或测试程序发送按键序列……

Linux 下模拟输入不是一件事，而是分两层。先分清层级，后面选工具才不会走弯路。

> 本章主线：Ubuntu 24.04 LTS + XFCE 桌面（X11 / Xorg 会话）。Wayland 的差异会单独对比说明。

## 8.1 原理分层：先看懂两条路线

### 8.1.1 XTEST 协议（显示服务器层）

X11 协议里有一个扩展叫 **XTEST**。客户端（比如 `xdotool`）向 X Server 发送「假装用户按了这个键」的请求，X Server 再把事件分发给当前获得焦点的窗口。

- **是什么**：应用层协议，事件是 X Server「伪造」出来的。
- **为什么方便**：不需要 root，不需要访问设备文件，普通用户就能用。
- **限制在哪**：
  - 只在 X11 会话内有效，Wayland 会话下基本失效（见 8.7 节）。
  - 事件带有 `synthetic`（合成）标记，部分程序（尤其是安全敏感程序、部分游戏的反作弊）会直接忽略合成事件。
  - 必须有正确的 `DISPLAY` 才能连上 X Server，没桌面会话就无从谈起。
  - 默认发给「当前活动窗口」，发错窗口的风险始终存在。

### 8.1.2 /dev/uinput（内核层）

`/dev/uinput` 是 Linux 内核提供的虚拟输入设备接口。程序往这个设备写入事件，内核就凭空多出一台「虚拟键盘 / 虚拟鼠标」，系统和所有上层软件看到的都是**真实硬件事件**。

- **是什么**：内核级虚拟设备，事件走 evdev 子系统，和真键盘插进来没有区别。
- **为什么强**：不依赖 X11 还是 Wayland，不依赖某个窗口有没有焦点，连登录界面（GDM / LightDM）和虚拟控制台都能收到。
- **限制在哪**：
  - 需要 `/dev/uinput` 的访问权限，配置不当就是安全洞（见 8.4.3 节）。
  - 用的是 Linux 原始键码（keycode），不是 X11 的 keysym，写起来更底层、更容易写错。
  - 没有「窗口」概念：事件发给当前焦点，谁有焦点谁收到，发之前必须自己保证焦点正确。

### 8.1.3 对比表

| 对比项 | XTEST（xdotool） | uinput（ydotool / python-uinput） |
|---|---|---|
| 所在层级 | X Server（用户态显示服务器） | Linux 内核 evdev |
| 典型工具 | `xdotool`、pyautogui、python-xlib | `ydotool`、python-uinput、evdev |
| X11 会话 | ✅ 完整可用 | ✅ 可用 |
| Wayland 会话 | ❌ 基本失效 | ✅ 可用 |
| 登录界面 / TTY | ❌ 不可用 | ✅ 可用 |
| 是否需要焦点 | 需要（可配合窗口操作切换） | 需要（内核不区分窗口） |
| 事件是否被标记为合成 | 是，部分程序会拒绝 | 否，与真实硬件一致 |
| 权限要求 | 普通用户，会话内即可 | 需要 uinput 设备权限 |
| 按键表示方式 | keysym 名，如 `Return`、`a` | Linux 键码，如回车 `28` |
| 上手难度 | 低，命令直观 | 中，要查键码表、管守护进程 |

**选型口诀**：XFCE / X11 桌面日常自动化，优先 xdotool；目标程序拒绝合成事件、或跑在 Wayland / 登录界面，才上 uinput 路线。

## 8.2 xdotool：X11 下的主力工具

### 8.2.1 安装与前提

```bash
sudo apt update
sudo apt install xdotool
xdotool version
```

能用的前提只有一个，但最容易翻车：**环境变量 `DISPLAY` 必须指向目标桌面会话**。在本机桌面打开的终端里它天然正确；从 SSH 进来时默认没有这个变量，xdotool 会直接报错 `Can't open display`。SSH 场景的正确设置方法见 8.6.3 节。

先自检一下当前会话：

```bash
echo $DISPLAY          # 桌面会话里通常是 :0
xdotool getdisplaygeometry   # 能打印分辨率说明连接正常
xdotool getactivewindow getwindowname   # 打印当前活动窗口标题
```

### 8.2.2 键盘：key 与 type

```bash
# 发送单个按键 / 组合键
xdotool key Return          # 回车
xdotool key ctrl+c          # Ctrl+C
xdotool key ctrl+shift+t    # 新开终端标签页（视终端而定）
xdotool key alt+F4          # 关闭窗口
xdotool key super           # Super（Win）键

# 连续发送多个按键
xdotool key a b c Return

# 输入一整段文字（自动处理大小写与符号，不用逐键拼）
xdotool type "hello world"

# 控制打字速度：--delay 单位毫秒，模拟真人输入、避开程序丢键
xdotool type --delay 50 "systemctl status ssh"

# 输入后回车执行：type 与 key 串联
xdotool type "ls -la" && xdotool key Return
```

常用 keysym 名速查（大小写敏感，写错了会静默无效或报错）：

| keysym | 按键 | keysym | 按键 |
|---|---|---|---|
| `Return` | 回车 | `Escape` | Esc |
| `Tab` | 制表符 | `BackSpace` | 退格 |
| `space` | 空格 | `Delete` | Delete |
| `ctrl` | Ctrl | `shift` | Shift |
| `alt` | Alt | `super` | Win 键 |
| `F1` ~ `F12` | 功能键 | `Up` / `Down` / `Left` / `Right` | 方向键 |
| `Prior` / `Next` | PageUp / PageDown | `Home` / `End` | 行首 / 行尾 |

完整 keysym 列表可查本机头文件：

```bash
grep -E "XK_(Return|Escape)" /usr/include/X11/keysymdef.h | head
```

### 8.2.3 鼠标：移动、点击、滚轮

```bash
xdotool mousemove 500 300            # 移动到屏幕坐标 (500, 300)
xdotool mousemove_relative -- -10 0  # 相对当前位置左移 10 像素（负数要加 --）
xdotool click 1                      # 左键单击（1 左键，2 中键，3 右键）
xdotool click --repeat 2 1           # 左键双击
xdotool mousedown 1                  # 按下左键不放
xdotool mouseup 1                    # 释放左键（拖拽就是 down → move → up）
xdotool click 4                      # 滚轮上滚（4 上，5 下）
xdotool getmouselocation             # 查看当前鼠标位置与所在窗口
```

### 8.2.4 窗口：查找、激活、给指定窗口发按键

多窗口环境下「先确认焦点，再发按键」是铁律。xdotool 提供全套窗口操作：

```bash
# 列出 / 查找窗口
xdotool search --name "Firefox"          # 按标题关键字找窗口，输出窗口 ID
xdotool search --class "xfce4-terminal"   # 按窗口类名找
xdotool getactivewindow                  # 当前活动窗口 ID
xdotool getactivewindow getwindowname    # 当前活动窗口的标题
xdotool getwindowname <窗口ID>            # 指定窗口的标题

# 激活窗口并确认
xdotool windowfocus <窗口ID>             # 仅切焦点
xdotool windowactivate <窗口ID>          # 激活并置前（推荐）
```

**关键用法：`--window` 直接给指定窗口发按键**，不必先抢焦点：

```bash
WIN_ID=$(xdotool search --name "文本编辑器" | head -n1)
xdotool key --window "$WIN_ID" ctrl+s
```

注意：`--window` 只是把事件投递给该窗口，程序是否接受「无焦点收键」取决于程序本身。很多程序会忽略后台收到的键盘事件，所以稳妥流程仍然是：

```bash
WIN_ID=$(xdotool search --name "文本编辑器" | head -n1)
xdotool windowactivate --sync "$WIN_ID"
sleep 0.3
xdotool key ctrl+s
```

### 8.2.5 用 sleep 串联动作序列

程序需要反应时间：窗口还没弹出来就发按键，按键必丢。用 `sleep` 和命令串联把节奏排好：

```bash
#!/usr/bin/env bash
# 示例：打开终端，执行一条命令
xfce4-terminal &
sleep 1.5                                   # 等窗口出现
xdotool type --delay 30 "htop"
sleep 0.2
xdotool key Return
```

要点：

- `sleep` 支持小数秒，`sleep 0.3` 是自动化脚本的常用档位。
- 等窗口出现不要死等固定时间，可用 `xdotool search --sync --name "标题"` 等窗口就绪后再继续。
- 每个关键步骤后留一拍（0.2~0.5 秒），比堆长延时更可靠。

## 8.3 ydotool：Wayland 与通用场景

### 8.3.1 是什么

`ydotool` 走 uinput 路线，把事件直接交给内核，因此不受显示协议限制：X11、Wayland、甚至没有图形界面时都能发。代价是要有一个常驻守护进程 `ydotoold` 负责持有虚拟设备，`ydotool` 命令通过 socket 把请求交给它执行。

### 8.3.2 安装

```bash
sudo apt update
sudo apt install ydotool
ydotool --help
```

> Ubuntu 24.04 仓库里的 ydotool 是 0.1.8 风格的老版本接口：`key` 子命令直接吃 Linux 键码加按下/弹起状态（形如 `28:1 28:0`）。网上很多教程写的是新版 1.x 的 `key Return` 写法，在仓库版上用不了，照本章写法即可。

### 8.3.3 先把守护进程跑起来

```bash
# 以当前用户身份启动（推荐，用户级服务）
systemctl --user start ydotoold
systemctl --user status ydotoold

# 或者由系统服务托管（视打包方式而定）
sudo systemctl start ydotoold
sudo systemctl status ydotoold
```

守护进程没跑时，`ydotool` 会报连接 socket 失败——这是它最常见的报错，先查服务状态再查命令写法。

### 8.3.4 常用键码与发键

Linux 键码定义在 `/usr/include/linux/input-event-codes.h`，常用几个：

| 键码 | 按键 | 键码 | 按键 |
|---|---|---|---|
| `1` | Esc | `28` | 回车 Enter |
| `14` | BackSpace | `15` | Tab |
| `29` | 左 Ctrl | `42` | 左 Shift |
| `56` | 左 Alt | `57` | 空格 |
| `30` | A | `31` | S |
| `32` | D | `17` | W |
| `105` / `106` | 左 / 右方向键 | `103` / `108` | 上 / 下方向键 |

发键格式是 `键码:状态`，`1` 表示按下，`0` 表示弹起。一个完整按键必须有按有弹：

```bash
# 回车：按下 28，再弹起 28
ydotool key 28:1 28:0

# Ctrl+C：依次按下 Ctrl、按下 C、弹起 C、弹起 Ctrl
ydotool key 29:1 46:1 46:0 29:0

# 输入文字（老版本有 type 子命令，走同样的 uinput 通道）
ydotool type "hello"

# 鼠标移动与点击（坐标为绝对位置）
ydotool mousemove --absolute -x 500 -y 300
ydotool click 0xC0    # 左键按下+弹起，具体掩码以 ydotool click --help 为准
```

### 8.3.5 权限：socket 与 uinput

ydotool 用不了，十有八九是权限。两处都要过：

1. **ydotoold 的 socket**：守护进程生成的 socket 文件（通常在 `/run/user/<uid>/.ydotool_socket` 或 `/tmp/.ydotool_socket`）必须对调用用户可读写。用 `systemctl --user` 以同一用户启动守护进程时天然匹配。
2. **/dev/uinput 设备权限**：守护进程要能打开 `/dev/uinput`。默认该设备权限较严，需要 udev 规则放行给 `input` 组：

```bash
# 查看当前权限
ls -l /dev/uinput
```

```bash
# 写入 udev 规则：uinput 设备归 input 组，权限 0660
echo 'KERNEL=="uinput", MODE="0660", GROUP="input", OPTIONS+="static_node=uinput"' \
  | sudo tee /etc/udev/rules.d/80-uinput.rules

# 把当前用户加入 input 组
sudo usermod -aG input "$USER"

# 重新加载规则并触发
sudo udevadm control --reload-rules
sudo udevadm trigger
```

**加组后必须重新登录（注销再登录或重启）才生效**，在旧会话里反复试只会一直失败。用下面命令确认组已生效：

```bash
id    # 输出里应包含 input 组
```

> ⚠️ **安全提醒**：`input` 组权限非常敏感——组内成员不仅能「发送」输入事件，还能**读取**所有输入设备的事件，包括别人敲的键盘（等于全局键盘记录能力）。只把确实需要自动化的账号加进去，不要给日常账号图省事长期挂着；服务器多人共用时尤其要克制。

## 8.4 uinput 原理补课

理解这一段，后面用任何 uinput 工具都不慌：

- 程序打开 `/dev/uinput`，通过 `ioctl` 声明「我要造一台键盘，支持这些按键」，内核随即在 `/dev/input/` 下注册一个新的 `eventX` 设备，`lsusb` 看不到它，但 `libinput list-devices` / `evtest` 能看到。
- 之后程序往 uinput 写入 `input_event` 结构体（类型 + 键码 + 值），内核把它当作真实硬件中断上报，经 evdev 分发给显示服务器，再到当前焦点窗口。
- **为什么 Wayland 下也能工作**：Wayland 出于安全禁止普通客户端互相注入事件，但 uinput 事件在 Wayland 合成器（compositor）眼里就是真键盘输入，走的是硬件通道，绕过了协议层的限制。
- **为什么登录界面也能工作**：登录界面同样从 evdev 读输入，它不在乎事件来自真键盘还是虚拟设备。
- **代价**：内核只管「把事件交给当前会话」，不知道也不关心哪个窗口该收——焦点管理永远是使用者自己的事。

## 8.5 Python 方案

### 8.5.1 pyautogui（X11 友好，Wayland 受限）

```bash
pip install pyautogui
```

```python
import time
import pyautogui

pyautogui.FAILSAFE = True   # 鼠标猛甩到左上角即抛异常中止，防止脚本失控
pyautogui.PAUSE = 0.2       # 每个动作后自动暂停 0.2 秒

print(pyautogui.size())                 # 屏幕分辨率
pyautogui.moveTo(500, 300, duration=0.3)
pyautogui.click()
pyautogui.write("hello", interval=0.05) # 逐字输入
pyautogui.press("enter")
pyautogui.hotkey("ctrl", "s")
pos = pyautogui.position()              # 当前鼠标位置
```

注意：

- pyautogui 在 Linux 上底层走 Xlib / XTEST，**Wayland 会话下移动和按键会失效或报错**——这是协议限制，不是库的 bug。
- `FAILSAFE` 默认开启，别关。脚本失控时把鼠标甩到屏幕左上角是最后的刹车。
- 截图、图像定位（`locateOnScreen`）在 X11 下可用，常用来「找到按钮再点」。

### 8.5.2 python-xlib：直接发 X 事件

需要更精细的控制（指定窗口、监听窗口事件）时，用 `python-xlib` 直接和 X Server 对话：

```bash
pip install python-xlib
```

```python
from Xlib import X, display
from Xlib.ext import xtest

d = display.Display()            # 连接当前 DISPLAY
root = d.screen().root

# 用 XTEST 伪造一次回车
# （keysym 到 keycode 需经 d.keysym_to_keycode 转换，此处只示意结构）
xtest.fake_input(d, X.KeyPress, 36)    # 36 是某种键盘布局下的回车 keycode，仅示意
xtest.fake_input(d, X.KeyRelease, 36)
d.sync()
```

keycode 随键盘布局变化，不要硬编码；正式代码先用 `d.keysym_to_keycode()` 把 keysym（如 `XK_Return`）换算成当前布局的 keycode。日常任务优先 xdotool，xlib 留给需要编程控制的场景。

### 8.5.3 uinput 的 Python 库

- **python-uinput**：`pip install python-uinput`，API 直接对应内核概念，声明设备时列出支持的按键，之后 `emit_click` / `emit_combo` 发事件。需要 uinput 权限（见 8.3.5）。
- **evdev**：`pip install evdev`，更底层，既能向 uinput 设备写事件，也能**读取**真实设备事件（做按键监听、录制宏时用它）。

两者都不依赖显示服务器，适合写常驻服务；但同样没有窗口概念，焦点要靠 xdotool / wmctrl 等工具配合（下一章会讲窗口管理）。

## 8.6 SSH 远程操作桌面会话

这是本章最实用的场景：人不在机器跟前，SSH 进去给桌面发按键。

### 8.6.1 核心要点

> **xdotool 必须在目标桌面会话的 `DISPLAY` 环境下运行。**
> SSH 会话默认没有 `DISPLAY`，不是工具坏了，是它根本没连上 X Server。

### 8.6.2 找到正确的 DISPLAY 与 XAUTHORITY

桌面会话的 X Server 默认是 `:0`。连接它还需要 X 授权文件 `XAUTHORITY`，常见位置：

- XFCE / LightDM：`~/.Xauthority`（用户家目录）
- GNOME / GDM：`/run/user/<uid>/gdm/Xauthority`

SSH 进去后这样设置（把 `<用户名>`、`<uid>` 换成实际值）：

```bash
export DISPLAY=:0
export XAUTHORITY=/home/<用户名>/.Xauthority
# GDM 环境则用：
# export XAUTHORITY=/run/user/<uid>/gdm/Xauthority

xdotool getactivewindow getwindowname   # 能输出窗口名说明打通了
```

查 uid 与当前有哪些图形会话：

```bash
id -u <用户名>
loginctl list-sessions
loginctl show-session <会话ID> -p Type -p Display -p Active
```

`loginctl` 能确认目标会话是 `x11` 还是 `wayland`、是否处于 active 状态——会话被锁屏或切到别的用户时，事件可能发过去但没人理。

### 8.6.3 常见坑

| 现象 | 原因 | 解法 |
|---|---|---|
| `Can't open display: (null)` | SSH 会话没有 `DISPLAY` | `export DISPLAY=:0` |
| `Authorization required` | `XAUTHORITY` 路径不对或文件不属于当前用户 | 指向桌面用户自己的 `.Xauthority`，并以该用户身份运行 |
| 命令无报错但窗口没反应 | 发给了错误会话 / 窗口无焦点 | 先 `xdotool windowactivate` 目标窗口再发键；用 `loginctl` 确认 active 会话 |
| 锁屏状态下无效 | 锁屏程序独占输入 | 先解锁（或避免锁屏，见第 9 章无人值守部分） |

还有一个身份问题：SSH 登录用户与桌面登录用户**不是同一人**时，即使设了变量也会被 X 授权挡住。最稳妥的做法是 SSH 直接登桌面那个用户，或用 `sudo -u <桌面用户>` 切换身份再执行。

## 8.7 X11 vs Wayland 差异与限制小结

| 事项 | X11（Xorg） | Wayland |
|---|---|---|
| 全局模拟输入 | XTEST 开放给所有客户端 | 协议层禁止跨应用注入 |
| xdotool | ✅ 完整可用 | ❌ `key` / `mousemove` 基本无效（查窗口名等少数功能在 XWayland 应用上偶有例外，不可依赖） |
| pyautogui | ✅ 可用 | ❌ 失效 |
| ydotool / uinput | ✅ 可用 | ✅ 可用（走内核通道） |
| 截图 / 录屏 | 任意客户端可截全屏 | 必须经 xdg-desktop-portal 授权（见第 9 章） |
| 本教程桌面 | XFCE 目前只有 X11 会话 | — |

**GNOME Wayland 下 xdotool 为何失效**：Wayland 的设计原则是应用互相隔离，一个客户端无权向另一个客户端注入事件，也无权全局监听键盘。xdotool 依赖的 XTEST 在 Wayland 下没有对应物；对 XWayland 里跑的老应用发事件，也只能影响该应用自身，够不到原生 Wayland 窗口。

**何时切回 Xorg 会话**：如果你的自动化链条（截图、发键、窗口管理）整套都建立在 X11 工具上，且机器用 GNOME，最省事的办法是登录时选 Xorg 会话：注销后在登录界面点齿轮图标，选「Ubuntu on Xorg」再登录。XFCE 用户无此烦恼——XFCE 在 Ubuntu 24.04 上仍是 X11 会话，直接按本章主线走即可。长期方向则是把输入层迁移到 uinput（ydotool），显示协议再变也不受影响。

## 8.8 安全与礼貌提醒

- 模拟输入是「以你的身份操作你的桌面」，权限等同你本人坐在键盘前。**只在自己的机器、正当的自动化目的下使用。**
- 不要把发键脚本对着别人的会话、别人的机器跑；共享服务器上先确认会话归属。
- 脚本里避免盲发：在发关键按键（如确认删除、回车提交）前，先校验活动窗口标题，宁可中止也不要发错地方。
- `input` 组、uinput 权限按最小需要授予（见 8.3.5），用完可回收。
- 自动化不等于可以绕过目标软件的使用条款：给游戏、考试系统、他人服务自动发输入前，先想清楚合规性。

---

**导航**

- 上一章：[第 7 章 Python 环境](07-python.md)
- 下一章：[第 9 章 远程控制进阶](09-remote-automation.md)
- 返回目录：[README](README.md)
