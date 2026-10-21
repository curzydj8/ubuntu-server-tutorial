# 第 14 章 实战操作代码

> 本章是「实战篇」的代码卷：六组命令与脚本，都是在 [第 12 章](12-machines.md) 那些机器上真实跑过、并真实踩过坑之后留下的版本。每组都给「可复制代码 + 逐段解释 + 验证方法」，其中的地址、口令类内容一律是占位符，请替换成自己的值（Key 与凭据的规矩见 [第 13 章](13-keys.md)）。

---

## 14.1 SSH 保活配置：治长连接静默断开

**现象**：SSH 连上服务器放着不动，或挂一个长任务，过一会儿终端像冻住一样——不报错、不掉线，就是没反应。这多半不是服务器死了，而是中间链路（NAT、防火墙、组网中继）把「长时间没流量」的连接悄悄丢弃了，本端还以为连接活着。

**解法**：在**客户端**的 `~/.ssh/config` 里给目标主机加心跳，让连接周期性有流量通过：

```sshconfig
Host zhang-new
    HostName 100.114.210.86
    User zhangdawei
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 15
    ServerAliveCountMax 4
    TCPKeepAlive yes

Host muse
    HostName 100.90.18.40
    User zhangdawei
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 15
    ServerAliveCountMax 4
    TCPKeepAlive yes
```

逐项解释：

- `ServerAliveInterval 15`：客户端每 15 秒给服务器发一个保活请求。**是什么**：SSH 协议层的心跳（不是 ping）。**为什么是 15 秒**：足够勤，能赶在大多数 NAT 表项老化之前续命；又不至于频繁到产生可感知流量。
- `ServerAliveCountMax 4`：连续 4 次心跳无回应就判定连接已死、主动断开。效果是真断线时最多 60 秒（15×4）就报错退出，而不是无限期假死——**让失败来得明确，比假装活着强**。
- `TCPKeepAlive yes`：再叠一层 TCP 层保活，与上面的 SSH 层心跳互补。

**验证**：连上后挂一个 `top` 或 `sleep 600` 放着，隔十几分钟回来操作，若立刻有响应即生效；人为断网一分钟左右，终端应明确报错退出而不是永久冻住。

> 注：这组参数解决的是「客户端视角」的假死。若是服务器侧想踢掉僵死会话，对应的是服务端的 `ClientAliveInterval`，别配错边。

---

## 14.2 SSH 经 HTTP CONNECT 隧道：ProxyCommand 脚本

**场景**：有些环境的出站流量被管控，只允许走 HTTP 代理，连 SSH 的 22 端口直连都不行（[第 12 章](12-machines.md) 的沙盒 VM 就是这样）。HTTP 代理有个标准用法叫 **CONNECT 隧道**：让代理先替你和目标主机建一条 TCP 通道，之后的字节流原样转发——SSH 就可以跑在这条通道里。

**用法**：把下面脚本存为 `~/bin/ssh-proxy.py` 并 `chmod +x`，然后在 `~/.ssh/config` 里给需要绕代理的主机加一行：

```sshconfig
Host github.com
    User git
    ProxyCommand ~/bin/ssh-proxy.py %h %p
```

（`%h`、`%p` 由 ssh 自动替换成目标主机和端口。）

**脚本（脱敏版）**：

```python
#!/usr/bin/env python3
"""ssh-proxy.py —— 经 HTTP CONNECT 隧道转发 SSH 流量。
代理地址从环境变量 https_proxy 读取（形如 http://[user:pass@]host:port），
口令会轮换、由平台托管，所以绝不写死在本文件里。
用法（由 ssh 的 ProxyCommand 调用）：ssh-proxy.py <host> <port>
"""
import base64
import os
import socket
import sys
import threading
from urllib.parse import urlparse


def main() -> int:
    host, port = sys.argv[1], int(sys.argv[2])

    # 1) 从环境变量解析代理地址与（可能存在的）凭据
    proxy = urlparse(os.environ["https_proxy"])
    auth = ""
    if proxy.username:
        token = base64.b64encode(
            f"{proxy.username}:{proxy.password or ''}".encode()
        ).decode()
        auth = f"Proxy-Authorization: Basic {token}\r\n"

    # 2) 连上代理，发出 CONNECT，请它代连目标
    s = socket.create_connection((proxy.hostname, proxy.port or 3128), timeout=30)
    s.sendall(
        f"CONNECT {host}:{port} HTTP/1.1\r\n"
        f"Host: {host}:{port}\r\n{auth}\r\n".encode()
    )
    resp = b""
    while b"\r\n\r\n" not in resp:
        chunk = s.recv(4096)
        if not chunk:
            print("proxy closed during handshake", file=sys.stderr)
            return 1
        resp += chunk
    if not resp.startswith(b"HTTP/1.1 200"):
        print(f"proxy refused: {resp[:120]!r}", file=sys.stderr)
        return 1

    # 3) 双向转发：socket <-> 本进程的 stdin/stdout
    def fwd(src, dst):
        read = src.recv if hasattr(src, "recv") else (
            lambda n: os.read(src.fileno(), n))

        def write(data):
            if hasattr(dst, "sendall"):
                dst.sendall(data)
            else:
                dst.write(data)
                dst.flush()

        try:
            while True:
                data = read(65536)
                if not data:
                    break
                write(data)
        except OSError:
            pass
        finally:
            try:
                dst.shutdown(socket.SHUT_WR)
            except (OSError, AttributeError):
                pass

    t = threading.Thread(target=fwd, args=(s, sys.stdout.buffer), daemon=True)
    t.start()
    fwd(sys.stdin.buffer, s)
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

关键点逐个说：

- **凭据从环境变量现读**：代理口令会轮换。写死在脚本里的做法能跑通一时，轮换后立刻全断，而且脚本进仓库还会把口令送出去——两个毛病一起犯。
- **`fwd` 里的 `hasattr` 判断不是炫技，是真踩过的坑**：在 OpenSSH 的 ProxyCommand 场景下，本进程的 stdin 是**管道**不是 socket，`sys.stdin.buffer` 根本没有 `.recv()` 方法，直接调用会在转发第一步抛 `AttributeError`，表现是「隧道握手成功、SSH 却立刻断开」，极难联想。正确做法是对文件对象用 `os.read(fd)` 读、对 socket 用 `recv` 收，上面的 `fwd` 用能力检测把两种情况都兼容了；stdout 一侧同理用 `write + flush`。
- **先验证隧道、再怀疑 SSH**：单独执行 `ssh -T git@github.com` 若输出 `Permission denied (publickey)`，说明隧道是通的（GitHub 已经应答了），问题只剩密钥——一步就能把「网络问题」和「认证问题」分开。密钥怎么加见 [第 13 章](13-keys.md) 13.2 节。

---

## 14.3 远程启动桌面 GUI 程序：一条命令 + 验证三件套

**场景**：人在外面，只有 SSH，想把某台机器桌面上的图形程序拉起来（比如代理客户端、模拟器、浏览器）。难点不在「启动」，在于 SSH 会话和桌面会话是两套环境：不补齐变量，程序要么报 `cannot open display`，要么以 root 身份跑出一堆权限问题。

**命令模板**：

```bash
sudo -u <user> env DISPLAY=:0 \
  XAUTHORITY=/run/user/1000/gdm/Xauthority \
  XDG_RUNTIME_DIR=/run/user/1000 \
  setsid nohup <程序路径> >/tmp/app-start.log 2>&1 &
```

逐参数解释：

- `sudo -u <user>`：切到**桌面登录用户**的身份运行。GUI 程序要读写该用户的配置目录，用 root 跑会生成一堆 root 属主的文件，之后用户自己打开反而权限错误。
- `DISPLAY=:0`：告诉程序「画面画到第 0 号显示器」，也就是那台机器物理桌面的会话。没有它，图形程序不知道往哪儿显示。
- `XAUTHORITY=...`：X 会话的「门票」文件，没有它会被 X Server 拒绝连接。**路径随显示管理器不同**：上面是 GDM 的路径（`/run/user/1000/gdm/Xauthority`，其中 1000 是用户 UID）；LightDM 机器上通常是 `/home/<user>/.Xauthority`，用前先 `ls` 确认。
- `XDG_RUNTIME_DIR=/run/user/1000`：用户级运行时目录（总线、套接字都在里面），缺了它程序可能连不上会话总线，表现为启动后静默退出或找不到系统托盘。
- `setsid nohup ... &`：脱离当前 SSH 会话独立运行，SSH 断开也不会被带走；输出重定向到日志文件，出问题有地方查。

**验证三件套（缺一不可）**。启动命令返回 0 只代表「发出去了」，在我们的实操纪律里，**已启动 ≠ 已生效**，必须三件都过：

```bash
# 1) 进程在不在
pgrep -af '<程序名>'

# 2) 它该监听的端口在不在（以本地服务型程序为例）
ss -tlnp | grep <端口>

# 3) 功能实测：真发一个请求过去
curl -s -o /dev/null -w '%{http_code}\n' <它的本地地址>
```

第 3 步最重要：进程活着、端口开着，都仍可能功能是坏的（配置指错节点、证书过期……）。只有真实请求拿到预期响应，才能说「启动成功」。

---

## 14.4 xdotool 实战片段：向指定窗口发按键

原理与工具对比在 [第 8 章](08-input-simulation.md)，这里只贴真实用过的片段：给街机模拟器（MAME）的窗口发按键，驱动自动打牌流程。

```bash
# 0) 当前桌面会话里先导出显示变量（SSH 里操作桌面程序必备，同 14.3 节）
export DISPLAY=:0
export XAUTHORITY=/run/user/1000/gdm/Xauthority

# 1) 按窗口标题找到目标窗口 ID
WID=$(xdotool search --name 'MAME' | head -1)
echo "$WID"

# 2) 向「这个窗口」发按键，而不是发给当前焦点窗口
xdotool key --window "$WID" --clearmodifiers Return
```

实战要点：

- **`--window <ID>` 是关键**：按键默认发给「当前活动窗口」，桌面上一旦有别的窗口抢焦点（弹窗、通知），按键就打到错误的地方。用窗口 ID 定向发送后，焦点在哪都不影响。
- **`--clearmodifiers`**：发送前清掉残留的修饰键状态（比如上一步按住的 Shift 没弹起），防止按键被组合成快捷键，引发完全意外的行为。
- **先 Return 跳过警告弹窗，是这套流程里真实的第一步**：某个游戏 ROM 每次启动都会先弹一个版本警告画面（大意是转储不完整、按任意键继续），自动流程若不先发一个 `Return` 把它消掉，后面所有按键都会打在警告画面上、流程全错位。**先清场、再干活**，是 GUI 自动化的通用规矩。
- 常用按键名速记：投币 `5`、开始 `1`、确认 `Return`、方向 `Up/Down/Left/Right`，组合键用 `ctrl+c` 这种写法；发完一轮后**截图验收**：

  ```bash
  scrot /tmp/check.png   # 截图后把文件取回来肉眼核对画面状态
  ```

  按键「发出去了」不等于「游戏响应了」，以截图里画面真的变化为准（还是那条验证纪律，见 14.3 节）。

---

## 14.5 GitHub API 全量推送：没有 SSH key 时的备选路线

**场景**：要推送的机器上没有能被 GitHub 认的 SSH key（或 SSH 端口被网络挡住，见 14.2 节连隧道都不行时），但手上有 GitHub 的 API 访问凭据。这时可以不走 git 协议，直接用 **Git Database API** 在服务器端拼出一次提交。真实用过一次：把一个 699 文件的完整仓库树推上 GitHub 的新分支。

**四步**（接口基址 `https://api.github.com/repos/<owner>/<repo>`，请求带 API 凭据的 Authorization 头）：

```text
1. 逐文件建 blob
   POST /git/blobs
   body: {"content": "<文件内容的 base64>", "encoding": "base64"}
   返回该文件的 blob SHA —— 与本地 `git hash-object` 的结果应当一致，可逐一断言。

2. 建 tree（注意下面的大坑，要增量建）
   POST /git/trees
   body: {"tree": [{"path": "a/b.txt", "mode": "100644", "type": "blob", "sha": "<上一步的 SHA>"}, ...]}

3. 建 commit
   POST /git/commits
   body: {"message": "...", "tree": "<tree SHA>", "parents": ["<父提交 SHA>"]}

4. 建/更新分支引用
   POST /git/refs        body: {"ref": "refs/heads/<新分支>", "sha": "<commit SHA>"}
   （分支已存在则改用 PATCH /git/refs/heads/<新分支>）
```

**关键限制（真实踩过）**：第 2 步把近 700 个条目塞进**一次**建树请求，会被 GitHub 以 **422** 拒绝（提示输入过大/超时，建议增量建树）。正确做法是**按目录自底向上增量建**：先给每个子目录建子树拿到 SHA，再在上层 tree 里用 `{"path": "<目录名>", "mode": "040000", "type": "tree", "sha": "<子树 SHA>"}` 引用它；单个目录条目仍然过多时，用 `base_tree` 参数分批往同一棵树上追加。根目录的直接条目数通常远小于文件总数，这样每步请求都不超限。

**完成标准**：推送后 `GET /git/trees/<根 tree SHA>?recursive=1` 把远端整棵树拉回来，与本地逐文件比对 SHA，**全部一致**才算推完——接口返回 200 只说明「收到了」，不说明「内容对」。

**与正常 git push 的关系**：这是应急路线。长期还是应该给机器配好 SSH key（[第 13 章](13-keys.md) 13.2 节），日常推送用原生 git（有完整提交历史、增量传输），API 路线适合「一次性全量同步到新分支」这种场合。

---

## 14.6 v2rayN 配置实战坑（V7.24.1）

代理客户端 v2rayN 在两台机器上都装了（见 [第 12 章](12-machines.md)）。它的界面操作没什么可讲的，但**直接改配置文件**时有几个坑，每一个都真实地烧掉过排查时间：

**坑一：先找到数据目录**

- 程序本体在 `/opt/v2rayN`，但配置与节点数据**不在程序目录**，而在用户目录：`~/.local/share/v2rayN/guiConfigs/`，核心两件套是 `guiNConfig.json`（全局配置）与 `guiNDB.db`（SQLite，节点库）。
- 只盯着程序目录找配置，永远找不到——先 `find ~ -name guiNDB.db` 确认位置再动手。

**坑二：`guiNConfig.json` 里的 `IndexId` 是字符串**

- 它表示「当前选中节点的 ID」，在 JSON 里是**带引号的字符串**，例如 `"IndexId": "5517684816683470291"`。
- 若用脚本（如 Python 的 json 模块不经意地）把它改写成**数字**，程序读配置时会失败并**静默回退**到另一个节点——界面和日志都不报错，只是连出去发现节点不对，极具迷惑性。
- 正确改法：只对那一行做最小文本替换（`sed` 定点替换原字符串），不要整文件解析重写；格式与缩进都原样保留。

**坑三：节点库里 UUID/密码在 `Password` 列**

- `guiNDB.db` 的 `ProfileItem` 表存每个节点，其中 **UUID（VLESS/VMess）和密码（Trojan/SS）都存在 `Password` 列**；同表的 `Id` 列在这一版里是空的。
- 按文档直觉去读 `Id` 列组装配置，会得到空 UUID，核心直接报 `invalid UUID` 起不来。
- 排查这类问题先 `sqlite3 guiNDB.db ".schema ProfileItem"` 看真实表结构，再 `select` 两行真实数据核对字段含义，别凭字段名猜。

**改配置的固定流程（把上面三个坑串起来的纪律）**

1. **先完全退出 GUI**（含托盘），确认进程不在了再改——程序运行中改文件，退出时它会用内存里的旧配置把你的修改覆盖回去。
2. **改前备份**：`guiNConfig.json` 与 `guiNDB.db` 各复制一份带时间戳的副本。
3. **只做最小替换**：定点改目标字段，不整文件重写。
4. **改完启动验证**：启动后重新读一遍配置文件确认值还在，再按 14.3 节的验证三件套确认代理功能真实可用（进程、端口、实际请求）。

---

## 本章小结

- SSH 保活靠客户端心跳（15 秒 × 4 次），本质是让假死变明断。
- 代理隧道脚本的两个要点：凭据从环境变量读、不硬编码；stdin 是管道不是 socket，转发要用 fd 读。
- 远程拉 GUI 程序：身份、DISPLAY、XAUTHORITY、XDG_RUNTIME_DIR 四样补齐，然后用「进程 + 端口 + 功能实测」三件套验收。
- xdotool 按窗口 ID 定向发键 + 先清场（警告弹窗）+ 截图验收，是 GUI 自动化的稳定打法。
- GitHub API 推送是无 key 时的备选路线：blobs → 按目录增量建树 → commit → ref，最后全量拉树比对 SHA 才算完。
- 改 v2rayN 配置：GUI 先退出、文件先备份、只做最小替换，警惕 `IndexId` 的字符串类型和 `Password` 列这两个反直觉点。

---

**导航**

- 上一章：[第 13 章 各种 Key 的添加与管理](13-keys.md)
- 下一章：无（本章为当前最后一章）
- 返回目录：[README](README.md)
