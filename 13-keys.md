# 第 13 章 各种 Key 的添加与管理

> [第 4 章](04-remote-access.md) 讲过 SSH 密钥登录的流程，这一章把「Key」这件事单独收全：SSH 密钥对怎么生成、公钥怎么加到 GitHub 和服务器、API Key 该放哪、泄露了怎么办。贯穿全章只有一条铁律先说在前面：**Key 是密码的加强版——绝不贴进聊天记录、绝不写进代码仓库、绝不出现在截图里。**

---

## 13.1 SSH 密钥对：生成与权限

### 13.1.1 生成

在**将要发起连接的那台机器**上执行（这点至关重要，13.2 节会用血泪教训展开）：

```bash
ssh-keygen -t ed25519 -C "zhangdawei@zhang-new"
```

- `-t ed25519`：算法选 Ed25519，密钥短、速度快、安全性足够，是目前的默认推荐。
- `-C "备注"`：写进公钥末尾的注释，只是给人看的标签——**写清是哪台机器、哪个用途**，将来在 GitHub 或服务器上看到一串公钥时能认出谁是谁。
- 一路回车会在 `~/.ssh/` 下生成两个文件：
  - `id_ed25519`：**私钥**，相当于保险柜钥匙，永远不离开本机、永远不给任何人。
  - `id_ed25519.pub`：**公钥**，可以随便发，它的作用是「锁」，装到你要登录的目标上。
- 是否给私钥再加一层 passphrase（口令）：加了更安全，但每次用钥匙都要输一遍（或交给 ssh-agent 托管）。无人值守的自动化机器上通常不加口令，改用「私钥文件权限 + 机器网络隔离」来兜底——这是明确的权衡，别默认跟风。

### 13.1.2 权限：错一位就连不上

SSH 对权限的检查非常严格，权限太松会直接拒绝使用密钥（报错见 13.1.4）：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys   # 服务器一侧收公钥的文件，见 13.3
```

- `~/.ssh` 目录 `700`：只有自己能进，防止别人窥探你连过哪些机器（`known_hosts`、`config` 都在里面）。
- 私钥 `600`：只有自己能读。**这条是硬性的**——私钥若是组可读/全局可读，OpenSSH 会判定「私钥可能已泄露」而拒绝使用。
- 公钥 `644`：公钥本来就是公开的，权限宽松无所谓。
- 改了会怎样：权限给多了不是「警告一下照用」，而是直接罢工。所以每次连不上先查权限，别先怀疑密钥本身。

### 13.1.3 把公钥装到目标机：两种方法

方法 A：`ssh-copy-id`（目标机已能密码登录时最省事）

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@<服务器IP>
```

它会用你输的密码登录一次，把公钥追加到目标机的 `~/.ssh/authorized_keys`，并顺手修正权限。

方法 B：手动追加（没有 ssh-copy-id、或只能通过跳板/控制台操作时）

```bash
# 先看一眼公钥内容（公钥可以公开，私钥不行）
cat ~/.ssh/id_ed25519.pub

# 在目标机上执行：
mkdir -p ~/.ssh && chmod 700 ~/.ssh
echo 'ssh-ed25519 AAAAC3...（整行粘贴）... zhangdawei@zhang-new' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

注意是 `>>` 追加不是 `>` 覆盖——覆盖会把目标机上已有的其他公钥全部冲掉。

### 13.1.4 排障：Permissions are too open

典型报错：

```
Permissions 0664 for '/home/you/.ssh/id_ed25519' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

翻译过来就是「私钥权限太松，已忽略这把钥匙」，于是连接退回密码登录或直接失败。解法就是 13.1.2 的 `chmod 600`。同类问题还包括：`~/.ssh` 目录权限不是 700、目标机上 `authorized_keys` 权限不是 600、甚至家目录本身被组写入（sshd 的 `StrictModes` 会连家目录一起查）。排查顺序：**先 `ls -la` 看权限，再看别的**。

---

## 13.2 把公钥加到 GitHub

GitHub 的 SSH 密钥是**按账号**绑定的：加上之后，用这把私钥就能以该账号身份 pull/push，不再需要输账号密码或令牌。

### 13.2.1 步骤

1. 在**要推代码的那台机器**上查看公钥：

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

   复制整行（从 `ssh-ed25519` 到末尾备注，一个字符都别漏）。

2. 浏览器打开 GitHub → 右上角头像 → **Settings** → 左侧 **SSH and GPG keys** → **New SSH key**：
   - **Title**：写清机器名，如 `zhang-new`、`muse`、`sandbox-vm`，将来吊销时认得出。
   - **Key type**：选 **Authentication Key**（见 13.2.3 的区别）。
   - **Key**：粘贴公钥，保存。

3. 回终端验证：

   ```bash
   ssh -T git@github.com
   ```

   期望输出：

   ```
   Hi <你的用户名>! You've successfully authenticated, but GitHub does not provide shell access.
   ```

   看到 `Hi` 就说明密钥对上了。如果这里还是 `Permission denied (publickey)`，99% 是下面这个坑。

### 13.2.2 实战大坑：加错机器的 key

真实发生过的乌龙：想在 A 机器上推代码，于是去 GitHub 加了一把 SSH key，加完 A 机器 `ssh -T` 还是 Permission denied。折腾半天才发现——**加的是 B 机器的公钥**（当时人坐在 B 机器前，顺手 `cat` 的就是 B 的公钥）。

原理其实很简单：**每台机器的密钥对都不一样**。公钥是「锁」，私钥是「钥匙」，给哪台机器授权，就必须加哪台机器的公钥、由那台机器用自己口袋里的私钥来开。所以：

- 加 key 前，先在**真正要连接的那台机器**上执行 `cat ~/.ssh/id_ed25519.pub`，复制那一行；
- Title 务必填机器名，否则 Settings 里躺着五把都叫 `id_ed25519` 的 key，出事时根本分不清；
- 有多台机器要推代码，就加多把 key（一机一把），不要把私钥拷来拷去图省事——私钥一旦开始流动，泄露面就失控了。

### 13.2.3 认证密钥 vs 签名密钥

GitHub 的 SSH key 分两种用途，添加时 Key type 要选对：

| 类型 | 用途 | 说明 |
|------|------|------|
| Authentication Key | 登录与代码推送 | 本章讲的就是它，`ssh -T git@github.com` 验证的也是它 |
| Signing Key | 给 git commit 签名 | 提交记录上显示 Verified 标记，证明「这个提交确实是我签的」；可以与认证密钥是同一把，也可以分开 |

只推代码的话，认证密钥就够了；签名是进阶项，用同一把 ed25519 密钥兼任也可以（在同一页面再加一次、类型选 Signing Key）。

---

## 13.3 把公钥加到服务器：免密登录 + 别名

目标：在自己的一台机器上，把公钥装到 zhang-new / muse 这类服务器，之后免密登录。

```bash
# 1) 装公钥（任选 13.1.3 的一种方法），例如：
ssh-copy-id -i ~/.ssh/id_ed25519.pub zhangdawei@100.114.210.86

# 2) 验证：应不再问密码
ssh zhangdawei@100.114.210.86
```

然后在客户端 `~/.ssh/config` 里写别名（完整写法与保活参数见 [第 14 章](14-field-ops.md) 14.1 节），之后直接用别名：

```sshconfig
Host zhang-new
    HostName 100.114.210.86
    User zhangdawei
    IdentityFile ~/.ssh/id_ed25519
```

```bash
ssh zhang-new
```

免密打通并验证成功之后，再考虑关掉服务器的密码登录（改 `/etc/ssh/sshd_config` 的 `PasswordAuthentication no` 后重启 ssh 服务）——**顺序不能反**：先确认密钥能进，再关密码通道，且改的时候保持一个已登录会话别关，防把自己锁在门外。

---

## 13.4 API Key 与令牌：放哪、怎么防泄露

SSH 密钥之外，日常还会遇到各种 API Key、访问令牌（Token）。它们和密码同级敏感，存放方式按「谁需要读」来选：

### 13.4.1 三种放法对比

| 放法 | 做法 | 适合场景 | 注意 |
|------|------|----------|------|
| 环境变量 | `export SOME_API_KEY=<your-key>`；长期生效写进 `~/.bashrc` | 自己交互式用的工具、临时调试 | 写进 `.bashrc` 等于明文落盘在自家目录，可接受；但**别写进脚本再把脚本提交 git** |
| `.env` 文件 | 项目目录放 `.env`，代码启动时加载 | 项目级配置、多人/多环境切换 | 必须 `chmod 600 .env`；**`.gitignore` 里必须有 `.env`**，加完还要 `git status` 确认它没被跟踪 |
| systemd EnvironmentFile | 服务单元里写 `EnvironmentFile=/etc/<服务名>.env` | 常驻服务的密钥（见 [第 7 章](07-python.md) 的 systemd 一节） | 文件放 `/etc` 下、`chmod 600`、属主 root，普通用户读不到 |

共同原则：**密钥与代码分离**。代码可以随便复制、提交、发人；密钥只待在被保护的那一小块地方。

### 13.4.2 GitHub Personal Access Token（PAT）

有些场景 Git 不走 SSH 而走 HTTPS + 令牌（比如某些 CI、网页 API 脚本、git 凭据助手）：

1. GitHub → Settings → Developer settings → **Personal access tokens** → 按需要选 Fine-grained（可限定到单个仓库、权限更细，推荐）或 Classic。
2. 权限**按最小需要勾**：只推代码就别勾删除仓库、管理账号之类的大权限；有效期也要设，别图省事选永不过期。
3. 生成后**只显示一次**，立刻存进密码管理器或上面说的受保护位置，别随手贴进文档和聊天。
4. git 侧存令牌：用凭据助手避免每次输——

   ```bash
   git config --global credential.helper store   # 明文存 ~/.git-credentials，文件权限确保 600
   # 或仅缓存一段时间：
   git config --global credential.helper 'cache --timeout=3600'
   ```

   `store` 是明文落盘（文件 `chmod 600` 是底线），安全性弱于 SSH 密钥；能走 SSH 的场景优先 SSH（13.2 节），PAT 留给非用不可的场合。

### 13.4.3 红线清单

1. **绝不贴聊天**：不要把 Key 粘进任何聊天窗口、工单、评论区——聊天记录会同步、会被截图、会被转发，贴出去的那一刻就应视为已泄露（见 13.4.4）。
2. **绝不进 git 仓库**：包括写在代码、注释、`.env`、配置文件、提交信息里；git 历史有记忆，事后删文件也洗不掉那次提交。
3. **推送公开仓库前先扫描**：养成习惯，推之前对整个目录跑一遍关键词与常见密钥格式的搜索：

   ```bash
   grep -RInE "ghp_[A-Za-z0-9]{20,}|github_pat_[A-Za-z0-9_]{20,}|sk-[A-Za-z0-9]{20,}|AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY-----" . --exclude-dir=.git
   grep -RInE "(api[_-]?key|secret|token|password)[[:space:]]*[:=]" . --exclude-dir=.git --exclude='*.md'
   ```

   第一条抓「长得像密钥」的字符串，第二条抓「把密钥赋值进配置」的写法。两条都无输出才放心推。注意本教程自身的代码块里会出现这些模式字符串（那是扫描命令本身），真实项目里看到命中要逐条核对。
4. **示例一律占位符**：写文档、给人发命令时，Key 的位置写 `<your-key>`，永不填真值——包括「反正这是测试 key」的情况。

### 13.4.4 泄露处置：发现即吊销

一条真实教训式的说明：曾经有一把 API Key 被顺手贴进了聊天窗口。虽然对方是可信的协作方、聊天也很快删了，但正确的处置只有一个——**立刻去签发方后台把这把 key 吊销、重新生成新的，旧的从此视为作废**。原因很朴素：你无法追踪一条聊天记录被同步到了多少设备、多少备份、多少截图里，「对方说删了」不构成安全保证。

记住这个动作顺序：

1. **先吊销，后善后**：第一时间在签发方禁用/删除旧 key，这一步完成前别做任何其他事。
2. **生成新 key 并更新所有引用处**（环境变量、`.env`、systemd 文件改完重启服务）。
3. **复盘泄露路径**：是怎么出去的（贴聊天？进仓库？截图带出来的？），把那个路径堵上，而不是只换一把了事。
4. 若已进 git 历史：吊销仍然是第一位的——代码可以慢慢清理历史，**已泄露的 key 永远救不回来**，只能换。

---

## 13.5 Tailscale 的 auth key（一句带过）

把新设备批量拉进 Tailscale 网络时，可以在 Tailscale 管理后台的 Settings → Keys 里预生成 **auth key**，装机时用 `tailscale up --authkey <key>` 免交互入网。注意两种区别：**一次性 key** 用完即废，适合单台装机；**可复用 key** 能反复用，方便但泄露后等于敞开组网大门，要设短有效期并当密钥同等保管（存放规则同 13.4 节）。日常一两台机器手动登录网页授权即可，不必动 auth key。

---

## 本章小结

- SSH 密钥：一机一对，私钥不出门；权限三件套（`.ssh` 700、私钥 600、公钥 644）错一位就罢工，报错先查权限。
- 加 GitHub key 的最大坑是**加错机器**：在哪台机器上 `cat` 的公钥，就只能给那台机器开门；Title 写机器名。
- API Key 与代码分离存放（环境变量 / `.env` / systemd EnvironmentFile），推送公开仓库前必须 grep 扫描。
- 泄露处置没有「等等看」：**立即吊销重签，旧 key 视为作废**。

---

**导航**

- 上一章：[第 12 章 机器清单与设置（实战环境）](12-machines.md)
- 下一章：[第 14 章 实战操作代码](14-field-ops.md)
- 返回目录：[README](README.md)
