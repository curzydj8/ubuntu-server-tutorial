# 第 7 章 Python 使用

Python 是 Ubuntu 上最容易「装着装着把系统搞坏」的语言——不是 Python 本身危险，而是系统自己就靠 Python 干活。这一章先讲清规矩，再讲 venv、镜像、pipx、pyenv 的正确用法，最后把脚本变成 systemd 服务和 cron 定时任务，让程序长期稳定地跑。

版本基线：Ubuntu 24.04 自带 Python 3.12，Ubuntu 22.04 自带 Python 3.10。下文以 24.04 为例。

## 7.1 系统 Python 不能乱动

```bash
python3 --version
# Ubuntu 24.04 应输出类似：Python 3.12.x
```

**是什么**：`/usr/bin/python3` 是系统组件。桌面设置、软件中心、`apt` 的部分工具、各种系统脚本都在调用它。

**为什么不能乱动**：

- 不要删它、不要改它的软链接指向、不要用第三方 Python 覆盖它——系统工具按发行版自带版本写死预期，换版本轻则某个设置界面打不开，重则包管理相关工具集体报错。
- 不要往系统 Python 里 `sudo pip install` 乱装包：包版本一旦和系统组件的要求冲突，坏的是系统，不是你的项目。

**改了会怎样**：记住一条原则就够——**系统 Python 只归系统用；你的项目，永远用项目自己的环境（见 7.3）。** 需要别的 Python 版本，用 pyenv 另装（见 7.6），与系统互不干扰。

### 基础组件安装

```bash
sudo apt install python3-pip python3-venv

# 验证
pip3 --version
python3 -m venv --help | head -3
```

- `python3-pip`：包管理器 pip。注意：装上它不等于可以对系统环境随便装包，规矩见下一节。
- `python3-venv`：创建虚拟环境的支持包。**不装它，`python3 -m venv` 会直接报错**，这是新手最常撞的第一个坑。

## 7.2 PEP 668：externally-managed-environment 报错

在较新的 Ubuntu 上直接执行：

```bash
pip3 install requests
```

很可能得到报错：`error: externally-managed-environment`。

**成因**：这是 PEP 668 机制。发行版给系统 Python 环境打了「外部管理」标记，pip 一旦发现目标是系统环境就主动拒绝安装，防止 pip 和 apt 互相踩坏对方的包。

**三条正道**（按推荐顺序）：

1. **项目依赖 → 用 venv**（7.3）：每个项目一个隔离环境，这是日常开发的标准做法。
2. **独立命令行工具 → 用 pipx**（7.5）：装的是拿来用的工具，不是给项目 import 的库。
3. **`--break-system-packages` → 仅限一次性、你明确知道后果的场合**：

```bash
# ⚠️ 危险用法示例，不要养成习惯
pip3 install --break-system-packages 某个包
```

**风险警告**：这个参数等于亲手关掉保护。装进去的包可能覆盖系统组件依赖的版本，后果从「某个系统工具报错」到「升级系统时依赖地狱」不等。容器里、一次性虚拟机里用用无妨；长期服役的服务器上，永远走前两条路。

## 7.3 venv 全流程

```bash
# 1. 建项目目录并创建虚拟环境（.venv 是约定俗成的目录名）
mkdir -p ~/demo && cd ~/demo
python3 -m venv .venv

# 2. 激活
source .venv/bin/activate

# 激活成功后，命令行提示符前会出现 (.venv)，此时：
which python    # 应指向 ~/demo/.venv/bin/python
pip --version

# 3. 在环境里放心装包（只影响本项目）
pip install requests

# 4. 退出环境
deactivate
```

**是什么**：venv 在项目目录里建一个自带 python 和 pip 的隔离小环境，激活后 `python`/`pip` 都指向它。

**为什么**：项目 A 要 requests 2.x、项目 B 要另一个版本，互不打架；删项目时连 `.venv` 一起删，系统干干净净。

**改了会怎样**：激活只影响当前终端会话；关掉终端就失效，下次进来重新 `source` 一次即可。

### requirements.txt：导出与恢复

```bash
# 导出当前环境的依赖清单
pip freeze > requirements.txt

# 在新机器/新环境里按清单恢复
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

把 `requirements.txt` 和代码一起纳入版本管理，是 Python 项目可复现的最低要求。

## 7.4 pip 国内镜像

官方 PyPI 在国内下载常慢或超时，换国内镜像是最有效的解法。常用两个：

- 清华 TUNA：`https://pypi.tuna.tsinghua.edu.cn/simple`
- 阿里云：`https://mirrors.aliyun.com/pypi/simple/`

**临时用法（只对这一次命令生效）：**

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple requests
```

**持久配置（一劳永逸，推荐）：**

```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 验证
pip config get global.index-url
```

换阿里云就把地址替换为阿里云那个。持久配置写在 `~/.config/pip/pip.conf`，想取消执行 `pip config unset global.index-url`。

> 注意：在 venv 里执行的 `pip config set` 影响的仍是当前用户全局配置；团队机器上改源前，确认没有统一的内网源要求。

## 7.5 pipx：安装独立 CLI 工具

有些 Python 包是「拿来用的工具」而不是「给代码 import 的库」，比如 `httpie`、`poetry`、`yt-dlp`。给每个工具单独建 venv 太麻烦，pipx 就是干这个的：

```bash
sudo apt install pipx
pipx ensurepath

# 安装一个工具（自动建隔离环境并把命令链到 ~/.local/bin）
pipx install httpie

# 验证 / 管理
httpie --version
pipx list
pipx upgrade httpie
pipx uninstall httpie
```

**分工记住**：库进项目的 venv，工具进 pipx。两者隔离方式相同，区别只在用途。

## 7.6 pyenv：多版本 Python

**什么时候才需要它**：系统是 3.12，但老项目只兼容 3.10、或要测试 3.13 新特性时。**只跑一个版本就别装**，徒增复杂度。

```bash
# 1. 先装编译依赖（pyenv 是从源码编译 Python 的）
sudo apt install build-essential libssl-dev zlib1g-dev libbz2-dev \
  libreadline-dev libsqlite3-dev libncursesw5-dev xz-utils tk-dev \
  libxml2-dev libxmlsec1-dev libffi-dev liblzma-dev curl git

# 2. 安装 pyenv（按 pyenv 官方仓库的安装说明执行，并把初始化行加进 ~/.bashrc）
# 重新打开终端后验证：
pyenv --version

# 3. 查看可装版本并安装某个版本（编译需要几分钟）
pyenv versions
pyenv install --list | grep " 3.11"
pyenv install 3.11.9

# 4. 设置版本
pyenv global 3.12.4     # 全局默认（写进 ~/.pyenv/version）
pyenv local 3.11.9      # 只对当前目录（写进 ./.python-version，cd 进来自动切换）
python --version        # 验证当前生效版本
```

**global 与 local 的区别**：`global` 是「没特别说明时用哪个」的兜底；`local` 在项目目录放一个 `.python-version` 文件，进目录自动用它——这个文件适合提交到 Git，队友用 pyenv 时自动对齐版本。

与 venv 的关系：pyenv 管「用哪个 Python」，venv 管「包隔离」，两者常配合：先用 pyenv 切到项目版本，再在该版本下建 venv。

## 7.7 conda / miniforge 简介

一句话定位：**conda 是跨语言的环境 + 包管理器**，除了 Python 包，还能管 C 库、CUDA、R 等二进制依赖。

- **何时用**：数据科学/AI 场景，依赖里有大量编译好的科学计算库（numpy、pytorch 的特定 CUDA 版本等），pip 装起来费劲时，conda 往往一把过。推荐装轻量的 **Miniforge**（默认 conda-forge 源）而非臃肿的 Anaconda 发行版。
- **与 venv 的分工**：普通 Web 后端、脚本、自动化——venv + pip 足够，别上重武器；科学计算、复杂二进制依赖——conda。两套环境不要混用（不要在 conda 环境里再套 venv），出了问题极难排查。

## 7.8 运行脚本的几种方式

```bash
# 方式一：直接运行（关终端就结束，适合调试）
python3 app.py
./app.py        # 文件首行有 #!/usr/bin/env python3 且 chmod +x 时

# 方式二：后台运行 + & 与作业控制
python3 app.py &     # 放后台
jobs                 # 看后台作业
fg %1                # 把 1 号作业切回前台
# Ctrl+Z 暂停当前前台任务，bg 让它在后台继续

# 方式三：nohup（退出登录也不死，适合临时挂一个任务）
nohup python3 app.py > app.log 2>&1 &
```

**说明**：`nohup` + 重定向是「临时常驻」的土办法，能用但不专业：开机不自启、挂了不拉活、日志不轮转。**长期运行的程序请直接上 systemd（下一节）**，`nohup` 只适合几小时量级的临时任务。

## 7.9 用 systemd 把程序做成常驻服务

假设项目在 `/home/你的用户名/demo`，环境是其下的 `.venv`，入口脚本是 `app.py`。

新建服务文件 `/etc/systemd/system/demo.service`：

```ini
[Unit]
Description=Demo Python Service
After=network.target

[Service]
Type=simple
User=你的用户名
WorkingDirectory=/home/你的用户名/demo
ExecStart=/home/你的用户名/demo/.venv/bin/python /home/你的用户名/demo/app.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

逐行理解关键项：

- `WorkingDirectory`：程序的工作目录。**是什么**：相当于先 `cd` 到这里再启动。**为什么**：脚本里用相对路径读配置文件、写日志时，不写它就会以 `/` 为基准，各种「文件找不到」。**改了会怎样**：填错目录，程序行为就像在别的目录被双击运行一样诡异。
- `ExecStart`：必须用**绝对路径**，且直接指向 venv 里的 python——systemd 环境里没有你 shell 的激活状态，写 `python app.py` 必翻车。
- `Restart=on-failure`：进程异常退出时自动重启；`RestartSec=5` 是重启间隔，防止崩溃循环刷屏。正常退出（返回 0）则不重启，想无条件重启用 `Restart=always`。
- `User=`：用普通用户跑，别默认 root，权限最小化。

启用与管理：

```bash
sudo systemctl daemon-reload          # 每次改完 unit 文件必做
sudo systemctl enable --now demo      # enable=开机自启，--now=顺便立即启动
sudo systemctl status demo            # 看状态
journalctl -u demo -f                 # 实时看日志（程序 print 的输出都在这）
sudo systemctl restart demo           # 改代码后重启
sudo systemctl stop demo              # 停止
```

**是什么**：把你的脚本注册成和 MySQL、SSH 同级的系统服务。**为什么**：崩溃自拉活、开机自启、日志统一进 journal，这才是「常驻」的完整含义。验证是否真的自启，最硬核的办法是重启一次机器再看 `systemctl status`。

## 7.10 cron 定时任务

```bash
crontab -e    # 编辑当前用户的定时任务（首次会让你选编辑器）
crontab -l    # 查看已设的任务
```

### 五字段格式

```text
分  时  日  月  周  命令
*   *   *   *   *   要执行的东西
```

示例：

```bash
# 每天凌晨 3:30 跑备份脚本，并把输出追加进日志
30 3 * * * /home/你的用户名/demo/.venv/bin/python /home/你的用户名/demo/backup.py >> /home/你的用户名/demo/cron.log 2>&1
```

### 两个必踩的坑

1. **绝对路径**：cron 的环境极简，`PATH` 和你登录终端完全不同。命令、python、脚本、日志路径全部写绝对路径；python 用 venv 里的那个（同 systemd 的道理）。
2. **环境变量缺失**：脚本依赖的变量（如 API 地址、时区）在 cron 里默认不存在。要么在 crontab 顶部显式写 `KEY=value`，要么让脚本自己读 `.env`/配置文件，不要指望 `~/.bashrc`——cron 不读它。

**日志重定向**：cron 本身不帮你存输出，不加 `>> 日志 2>&1` 的话，输出或被发去本地邮件或直接丢失，出错时你将一无所知。每条任务都带上重定向，排查时先看这个日志。

**验证方法**：先把时间设成两分钟后跑一次，确认日志里有预期输出，再改回正式时间——不要设完就等明天，错了白等一天。

## 小结

- 系统 Python 只归系统；项目用 venv、工具用 pipx，这是 PEP 668 时代的铁律，`--break-system-packages` 不是日常选项；
- 慢就换镜像（清华/阿里云），持久配置一次搞定；
- pyenv 只在多版本时上，conda 只在重型科学计算依赖时上，别为用不上的能力买单；
- 临时任务 `nohup`，长期常驻 systemd，周期执行 cron——三者各就其位，路径都写绝对。

---

上一章：[06-applications.md](06-applications.md) ｜ 下一章：[08-input-simulation.md](08-input-simulation.md) ｜ 返回目录：[README](README.md)
