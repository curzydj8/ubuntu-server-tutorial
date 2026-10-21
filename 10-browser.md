# 第 10 章 浏览器

浏览器是桌面服务器上最重的应用，也是远程自动化里出场率最高的工具。这一章讲四件事：装哪个、怎么从 SSH 打开、调试端口怎么安全地用、无头模式与代理怎么配。

> 环境：Ubuntu 24.04 LTS。Chrome / Chromium 的常规安装在第 6 章已经讲过，本章不重复安装细节，只讲使用与进阶。

## 10.1 Firefox

### 10.1.1 snap 版说明

Ubuntu 24.04 的 Firefox 官方渠道是 **snap 包**（执行 `apt install firefox` 也会被导向 snap）。这意味着：

- 启动稍慢（snap 挂载与沙箱初始化），第一次打开尤其明显，属正常现象。
- 程序与配置都在 snap 的沙箱目录里：个人数据在 `~/snap/firefox/common/.mozilla/firefox/`，不是传统的 `~/.mozilla/`。排障找错目录是最常见的乌龙。
- 更新由 snapd 后台自动完成，可能在你没防备时换版本；介意的话用 `snap refresh --hold firefox` 暂时冻结版本。

查看与管理：

```bash
snap list firefox
which firefox          # 通常是 /snap/bin/firefox
firefox --version
```

### 10.1.2 常用命令行参数

```bash
firefox https://example.com               # 打开指定网址
firefox --new-window https://example.com  # 新窗口
firefox --private-window https://example.com  # 隐私窗口
firefox --profile /path/to/profile        # 指定配置目录启动
firefox --ProfileManager                  # 打开配置管理器（图形界面）
firefox --headless --screenshot=/tmp/a.png --window-size=1280,800 https://example.com
firefox --headless --print-to-pdf=/tmp/a.pdf https://example.com
```

多个用途（办公 / 自动化 / 测试）建议各用一个 `--profile` 隔离，互不污染 Cookie 与登录态。

## 10.2 Chrome / Chromium 安装回顾

安装方式（deb 直装、版本查看、默认浏览器设置）在 **第 6 章** 已完整讲过，这里只给速查：

```bash
google-chrome --version     # Chrome
chromium --version          # Chromium（snap 版为 chromium-browser 过渡包时注意提示）
```

本章后续的调试端口、无头、代理参数，Chrome 与 Chromium 通用，把命令名互换即可。

## 10.3 从 SSH 打开图形浏览器：两种完全不同的路

「SSH 里敲 `firefox`」到底把窗口开到哪儿，有两种模式，混淆它们是远程操作的第一大坑。

### 10.3.1 模式 A：窗口开在服务器自己的桌面上

原理：不转发画面，只把 `DISPLAY` 指向服务器本地的 X Server（`:0`），浏览器就出现在服务器的显示器上，你通过截图 / VNC / 远程桌面看它。

```bash
ssh <用户名>@192.0.2.10
export DISPLAY=:0
export XAUTHORITY=~/.Xauthority
firefox https://example.com &
```

这正是第 8、9 章一直在用的模式：**自动化、截图、发按键全部围绕服务器本地会话展开**，选它。

### 10.3.2 模式 B：ssh -X 把窗口转发到你本地

原理：SSH 的 X11 转发在两端之间架一条加密通道，X 协议流量回传到你本地机器的 X Server 渲染，窗口像本地程序一样出现在你的桌面上。

```bash
# 本地需先有 X Server：Linux 桌面天然有；Windows 需另装（如 VcXsrv），macOS 需 XQuartz
ssh -X <用户名>@192.0.2.10
echo $DISPLAY        # 会变成形如 localhost:10.0 的转发地址，切勿手动改回 :0
firefox &
```

要点与限制：

- 依赖 **`xauth`**：服务器上要装好 `xauth`（包名同名），SSH 才能写入转发用的授权 cookie；缺了它转发直接失败并提示。
- 服务端 `sshd_config` 里 `X11Forwarding` 需为 `yes`（改后重载 sshd）。
- **慢是常态**：X 协议是大量小往返的聊天式协议，跨网络延迟会被成倍放大；浏览器这种重绘频繁的应用尤其卡，只适合偶尔开个设置页，不适合日常浏览与自动化。
- `-X` 是「不受信转发」，部分扩展受限；确需完整扩展可用 `-Y`（受信），但那等于把本地 X Server 完全暴露给远端程序，**只对可信服务器使用**。

一句话选型：**要操作服务器桌面 → 模式 A；只是想在本地弹个远端程序窗口应急 → 模式 B。**

## 10.4 远程调试端口（Chrome DevTools Protocol）

### 10.4.1 启动与用法

```bash
google-chrome \
  --remote-debugging-port=9222 \
  --remote-debugging-address=127.0.0.1 \
  --user-data-dir=/tmp/chrome-debug \
  https://example.com
```

启动后，调试接口只提供 HTTP / WebSocket 访问：

```bash
curl http://127.0.0.1:9222/json/version    # 浏览器版本与 WebSocket 地址
curl http://127.0.0.1:9222/json/list       # 当前所有标签页
```

Puppeteer、Playwright 等工具就是通过这个端口接管浏览器的（连接方式见 10.5 节）。

### 10.4.2 安全警告（务必看完）

> ⚠️ **调试端口 = 浏览器的完全控制权。** 能连上 9222 的任何进程/人，都可以读取所有页面内容（含已登录页面的 Cookie 与表单）、执行任意 JS、以浏览器身份发请求。它没有账号密码这一层防护。

铁律：

1. **只绑定 `127.0.0.1`**（如上 `--remote-debugging-address=127.0.0.1`），永远不要绑 `0.0.0.0`，更不要在云安全组/防火墙里放行 9222。
2. 需要从本地访问时，走 SSH 端口转发，把流量裹在 SSH 隧道里：

```bash
# 在本地执行：把服务器的 9222 映射到本地 9222
ssh -L 9222:127.0.0.1:9222 <用户名>@192.0.2.10

# 然后本地浏览器打开 http://127.0.0.1:9222 即可访问调试页
```

3. 用完即关：调试进程不要长期挂着，更不要开机自启一个带调试端口的浏览器。
4. 调试实例用独立的 `--user-data-dir`（见 10.6 节），别让它碰你日常登录着账号的主配置目录。

## 10.5 无头模式：没有窗口的浏览器

无头（headless）模式不需要图形会话，适合服务器上做截图、转 PDF、抓取渲染后内容。

```bash
# 输出渲染后的 DOM（JS 执行完的最终 HTML）
google-chrome --headless=new --disable-gpu --dump-dom https://example.com

# 整页截图
google-chrome --headless=new --disable-gpu \
  --screenshot=/tmp/page.png --window-size=1280,1200 https://example.com

# 打印为 PDF
google-chrome --headless=new --disable-gpu \
  --print-to-pdf=/tmp/page.pdf https://example.com

# 无头也要指定独立数据目录，避免与桌面实例抢锁
google-chrome --headless=new --user-data-dir=/tmp/chrome-headless --dump-dom https://example.com
```

注意：

- `--headless=new` 是当前新版无头实现；老教程里的 `--headless`（旧实现）在新版 Chrome 已逐步移除，照新写法走。
- 一次性命令（dump-dom / screenshot / print-to-pdf）执行完自动退出，适合脚本调用；超时页面记得在脚本层加 `timeout` 兜底。
- **与 Puppeteer / Playwright 的关系**：它们是上层自动化框架，底层就是启动（或连接）一个带调试端口的 Chrome/Firefox 并说 CDP 协议。需要点击、填表、等待元素等复杂交互时直接用框架，别手搓 `curl` 调试接口。

## 10.6 多用户数据目录隔离

Chrome 系的铁规则：**一个 `--user-data-dir` 同一时刻只能被一个浏览器进程使用**，第二个进程会直接接管已有窗口或报配置锁定。所以多开、多用途必须隔离目录：

```bash
# 日常使用（默认目录 ~/.config/google-chrome，一般不用显式写）
google-chrome

# 自动化专用，与日常登录态完全隔离
google-chrome --user-data-dir=$HOME/.chrome-automation https://example.com

# 临时一次性环境，用完可删
google-chrome --user-data-dir=/tmp/chrome-tmp https://example.com
```

隔离的好处：自动化脚本不会动到你登录着网银、邮箱的主配置；不同脚本之间也不会互相抢锁。Firefox 的对应机制是 `--profile`（见 10.1.2）。

## 10.7 浏览器代理设置

给浏览器单独挂代理有三条路，优先级从高到低说：

### 10.7.1 命令行参数（Chrome 系，最直接）

```bash
# SOCKS5 代理（<端口> 按你的代理程序实际监听端口填写）
google-chrome --proxy-server="socks5://127.0.0.1:<端口>" https://example.com

# HTTP 代理
google-chrome --proxy-server="http://127.0.0.1:<端口>" https://example.com

# 明确不走代理（排障时用来对比验证）
google-chrome --no-proxy-server https://example.com
```

此参数只作用于这一次启动的这个实例，配合独立 `--user-data-dir` 使用最干净。

### 10.7.2 环境变量

```bash
export http_proxy="http://127.0.0.1:<端口>"
export https_proxy="http://127.0.0.1:<端口>"
export all_proxy="socks5://127.0.0.1:<端口>"
export no_proxy="localhost,127.0.0.1"
```

说明：环境变量对 `curl` 等命令行工具普遍有效；**Chrome 基本不读这些变量**（它认自己的参数与系统设置），Firefox 则需在设置里选「使用系统代理设置」才可能跟随——给浏览器配代理，优先用 10.7.1 或图形设置，别指望环境变量。

### 10.7.3 与系统代理 / v2rayN 的关系

- 在本教程的机器上，代理程序（如 v2rayN）跑在服务器本机，向本机回环地址提供一个本地端口（如 SOCKS5 / HTTP 混合端口）。浏览器代理要填的就是 `127.0.0.1` 加那个端口，**不是**远程节点的地址。
- 浏览器填的端口必须与代理程序当前实际监听的端口一致；在 v2rayN 界面里确认端口号后再填，节点切换一般不影响本地监听端口，程序重启/改设置才会变。
- 系统级代理（GNOME/XFCE 网络设置或 `gsettings` 的 proxy）会影响所有遵循系统设置的程序；只想让某一个浏览器实例走代理时，用命令行参数，别动系统设置殃及全体。
- 验证是否生效：开代理后访问显示出口 IP 的页面，与不开代理时对比，出口不同才算真的走了代理。

---

**导航**

- 上一章：[第 9 章 远程控制进阶](09-remote-automation.md)
- 下一章：[第 11 章 FAQ 与排障清单](11-faq.md)
- 返回目录：[README](README.md)
