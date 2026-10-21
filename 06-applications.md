# 第 6 章 常用应用安装

上一章装好了桌面，这一章把日常要用的软件装齐：浏览器、编辑器、终端工具链，再加容器、Node.js、数据库这些开发/运维基础设施。

每节统一格式：**安装命令 + 验证命令 + 一句说明**。示例统一基于 Ubuntu 24.04 LTS，命令可直接复制执行；涉及第三方源的部分会特别标注，因为「加源」意味着你信任了源的维护方，要心里有数。

开始前先更新索引：

```bash
sudo apt update
```

## 6.1 浏览器

### 6.1.1 Firefox

```bash
# 24.04 默认方式：snap 版
sudo snap install firefox

# 验证
firefox --version
```

**说明**：Ubuntu 24.04 里 `apt install firefox` 得到的不是传统 deb 包，而是一个过渡包，它会转手去装 snap 版——这是 Canonical 的既定策略，所以直接用 `snap` 命令装反而更直白。

**apt 版之争简介**：snap 版被诟病的主要是早期启动慢、与系统集成（如输入法、企业策略）偶有问题。想要纯 deb 版，社区常用的做法是加 Mozilla 官方维护的 mozillateam PPA 并给 apt 设置优先级（pinning），强制 Firefox 走 PPA 而非 snap。这条路能走通，但多了一个第三方源和优先级配置，新手容易把依赖搞乱。建议：没有明确痛点就用 snap 版；公司策略、特定插件兼容性有硬需求时再考虑 PPA，且操作前先备份 apt 配置。

### 6.1.2 Chromium

```bash
sudo snap install chromium

# 验证
chromium --version
```

**说明**：Ubuntu 仓库里的 Chromium 同样只提供 snap 版，装法与 Firefox 对称，别再去找 deb 包了。

### 6.1.3 Google Chrome

```bash
# 下载官方 .deb（示例地址，请以 Google 官网当前下载页为准）
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb

# 安装（apt 会自动处理依赖，并顺带配好 Google 的 apt 源，后续可正常 apt 升级）
sudo apt install ./google-chrome-stable_current_amd64.deb

# 验证
google-chrome --version
```

**说明**：Chrome 是闭源的，不在 Ubuntu 官方仓库里；用官方 `.deb` 安装的好处是它会自动注册官方 apt 源，以后 `apt upgrade` 就能一起升，不用手动重下。

## 6.2 VS Code

三种装法三选一，**不要混装**，混装会导致「到底哪个在更新」都说不清。

**方式一：官方 .deb（推荐，最简单）**

```bash
# 从 VS Code 官网下载 .deb 后
sudo apt install ./code_*.deb

# 验证
code --version
```

**方式二：微软 apt 源（适合想全程用 apt 管理的人）**

官网 `.deb` 安装时其实也会自动配好微软的 apt 源，所以方式一装完后同样能用 `apt upgrade` 更新——多数人选方式一就够了。只有当你要批量部署、或明确不想走 snap 时，才需要手动按微软官方文档添加源和签名密钥。

**方式三：snap**

```bash
sudo snap install code --classic

# 验证
code --version
```

注意 snap 版必须带 `--classic`，否则沙盒限制会让它访问不了系统目录，开发体验残缺。

**说明**：日常单机开发选 `.deb`；服务器上其实更常用「本地 VS Code + Remote-SSH 连到服务器」的模式，服务器本体可以不装编辑器 GUI。

## 6.3 终端工具箱

这一组是运维和开发的日常家伙，一次装齐：

```bash
sudo apt install tmux htop btop ncdu tree ripgrep fzf \
  curl wget aria2 unzip p7zip-full net-tools iproute2 git
```

逐个验证与说明：

| 工具 | 验证命令 | 一句话说明 |
| --- | --- | --- |
| tmux | `tmux -V` | 终端复用器：会话断线不丢、一个窗口分多屏，远程运维必备 |
| htop | `htop --version` | 交互式进程查看器，比 `top` 直观，能直接排序/结束进程 |
| btop | `btop --version` | 更现代的资源监视器，CPU/内存/磁盘/网络一屏看全，颜值高 |
| ncdu | `ncdu --version` | 交互式磁盘占用分析，找「哪个目录把盘吃满了」最快 |
| tree | `tree --version` | 把目录结构画成树，一眼看清层级 |
| ripgrep | `rg --version` | 极快的全文搜索，默认尊重 `.gitignore`，搜代码首选 |
| fzf | `fzf --version` | 模糊查找器：配 Ctrl+R 搜历史命令、管道里选文件都极顺手 |
| curl / wget | `curl --version` / `wget --version` | 下载与测接口的双子星，脚本里到处都是 |
| aria2 | `aria2c --version` | 多线程下载器，大文件、多源下载明显更快 |
| unzip / p7zip-full | `unzip -v` / `7z | head -1` | 解压 zip 与 7z/rar 等格式，没装迟早被一个压缩包卡住 |
| net-tools | `ifconfig -V` | 老牌网络工具（`ifconfig`、`netstat`），新脚本不推荐，但老教程全是它 |
| iproute2 | `ip -V` | 现代网络工具集（`ip`、`ss`），查端口占用用 `ss -tlnp` |
| git | `git --version` | 版本控制，没什么可解释的，装就完了 |

Git 装完先做初始配置，否则第一次提交就会报错或记错作者：

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# 验证
git config --global --list
```

**是什么**：把作者名和邮箱写进 `~/.gitconfig`，每次提交都会带上。

**为什么**：提交记录靠这两个字段署名；用占位符示例即可，换成你自己的真实信息（公司项目注意用公司邮箱）。

**改了会怎样**：只影响之后的新提交，历史提交不变。

## 6.4 Docker

### 方式一：官方一键脚本（快，适合个人机/测试机）

```bash
curl -fsSL https://get.docker.com | sudo sh

# 验证
sudo docker run hello-world
docker --version
docker compose version
```

**说明**：脚本自动识别发行版、配源、装 `docker-ce` 和 Compose 插件。优点是快；缺点是你把一段远程脚本直接交给了 root 执行——生产环境介意这点的话，用方式二。

### 方式二：手动加官方 apt 源（可控，适合生产）

思路是四步：装证书工具 → 添加 Docker 官方 GPG 密钥 → 写入 apt 源 → 安装。完整命令以 Docker 官方文档「Install using the apt repository」为准（源地址和密钥指纹以官方页面当前版本为准，不要抄来历不明的博客版本）。装完验证同上：

```bash
sudo docker run hello-world
```

看到 `Hello from Docker!` 就说明守护进程、网络、镜像拉取三件事都通了。

### 把用户加入 docker 组（重要，含权限含义）

```bash
sudo usermod -aG docker "$USER"
# 注销重新登录（或重启）后生效，验证：
docker run hello-world
```

**是什么**：`docker` 组的成员可以免 `sudo` 直接操作 Docker 守护进程。

**为什么**：每次敲 `sudo docker ...` 很烦，加组后日常操作顺滑很多。

**改了会怎样（必须知道）**：docker 组权限**约等于 root**——组员可以用容器把宿主机目录挂进去读写。个人开发机无所谓；多人共用、生产服务器上，只给可信的人加组，这不是普通「方便权限」。

### Compose 插件

上面两种方式都会装上 `docker compose` 插件（注意是空格写法 `docker compose`，老式的独立命令 `docker-compose` 已淘汰）。验证：

```bash
docker compose version
```

**说明**：用一个 `compose.yaml` 描述多容器应用（应用 + 数据库 + 缓存）的启动方式，是本地开发和小型部署的事实标准。

## 6.5 Node.js

两种装法，先讲清区别再选：

| 方式 | 适合谁 | 代价 |
| --- | --- | --- |
| NodeSource apt 源 | 只需要一个固定大版本（如当前 LTS）、想用 apt 统一管理 | 切换版本要改源 |
| nvm | 同一台机器要跑多个 Node 版本（老项目 16、新项目 20 并存） | 只对当前用户、当前 shell 生效，系统服务里用要额外处理 |

**方式一：NodeSource 源**

按 NodeSource 官方说明添加对应大版本的源后：

```bash
sudo apt install nodejs

# 验证（npm 会随 nodejs 一起装上）
node -v
npm -v
```

**方式二：nvm**

按 nvm 官方仓库说明用其安装脚本安装后，重新打开终端：

```bash
nvm install --lts
nvm use --lts

# 验证
node -v
```

**说明**：拿不准就用 nvm——前端/全栈开发几乎必然遇到多版本问题，早装早省事；纯跑一个后端服务的服务器，NodeSource 更干净。

## 6.6 数据库：只作入口

本节只把软件装上、服务跑起来，建库、用户权限、备份、调优是后面的专题，不要在这一步贪多。

```bash
# MySQL
sudo apt install mysql-server
sudo systemctl status mysql

# PostgreSQL
sudo apt install postgresql
sudo systemctl status postgresql

# Redis
sudo apt install redis-server
sudo systemctl status redis-server
```

验证通用法则：`systemctl status <服务名>` 看到 `active (running)` 就是起来了；退出状态界面按 `q`。进一步的功能验证（能登录、能执行一句查询）留到专题章节。

**后续学习方向**（按顺序）：① 本地登录与改初始密码 → ② 建库建用户与最小权限 → ③ 只监听本机还是开远程访问（开远程前先想清防火墙和弱口令风险）→ ④ 备份与恢复演练。数据库客户端（如只在服务器上连别的库）则装对应的 client 包即可，不必装整个 server。

## 6.7 中文输入法

前提：输入法是桌面环境的东西，headless 服务器不需要本节（语境见第 5 章）。

Ubuntu 中文输入主流两条线：

```bash
# 路线一：IBus + 拼音（GNOME 默认框架，集成最好）
sudo apt install ibus-libpinyin

# 路线二：Fcitx5 + 拼音（轻量桌面、远程桌面下常更跟手）
sudo apt install fcitx5 fcitx5-chinese-addons
```

**说明**：`ibus-libpinyin` 胜在与 GNOME 原生集成、设置里直接添加；Fcitx5 在 XFCE 等轻量桌面和某些远程场景下延迟更低、候选框更稳定。二选一即可，装完要**注销重新登录**，再到系统设置的「区域与语言 / 键盘」里把拼音加为输入源。两个框架不要同时自启，会互相打架。

## 小结

- 浏览器：Firefox/Chromium 用 snap 是 24.04 的常态，Chrome 用官方 `.deb` 自带更新源；
- 工具链一次装齐，Git 先配署名，Docker 加组前想清「约等于 root」这句话；
- Node 多版本用 nvm、单版本用 NodeSource；数据库本节只到「服务在跑」为止；
- 输入法是桌面的事，和第 5 章的桌面选型配套考虑。

---

上一章：[05-desktop.md](05-desktop.md) ｜ 下一章：[07-python.md](07-python.md) ｜ 返回目录：[README](README.md)
