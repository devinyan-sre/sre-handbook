# ccgram 部署完整指南

> **ccgram**([alexei-led/ccgram](https://github.com/alexei-led/ccgram))是一个 **Telegram ↔ tmux 桥接工具**,让你用手机上的 Telegram 机器人远程控制服务器上运行的 Claude Code(也支持 Codex / Gemini / Shell)。
>
> 本文档基于 2026-07-23 在本机(RHEL 9,用户 `devops`)的实际部署过程整理,按步骤操作即可独立完成部署。

---

## 目录

1. [工作原理](#1-工作原理)
2. [环境要求与检查](#2-环境要求与检查)
3. [Telegram 侧准备(3 样东西)](#3-telegram-侧准备3-样东西)
4. [安装 ccgram](#4-安装-ccgram)
5. [写配置文件](#5-写配置文件)
6. [自检与安装 Claude hooks](#6-自检与安装-claude-hooks)
7. [配置 systemd 开机自启](#7-配置-systemd-开机自启)
8. [使用方法](#8-使用方法)
9. [问题处理实录](#9-问题处理实录)
10. [运维命令速查](#10-运维命令速查)
11. [安全注意事项](#11-安全注意事项)
12. [Telegram 命令手册](#12-telegram-命令手册)
13. [使用技巧](#13-使用技巧)
14. [已发出内容如何撤销/中止](#14-已发出内容如何撤销中止)
15. [多人协作与权限治理](#15-多人协作与权限治理)

---

## 1. 工作原理

理解了这张图,后面每一步"为什么这么做"就都清楚了:

```
手机 Telegram「话题群组」                服务器
┌─────────────────────┐        ┌──────────────────────────┐
│ 话题1 "项目A"        │ ◄────► │ tmux 会话 ccgram          │
│ 话题2 "修bug"        │ ◄────► │   ├─ 窗口1: claude (项目A) │
│ 话题3 "..."          │ ◄────► │   ├─ 窗口2: claude (修bug) │
│                     │        │   └─ 窗口3: ...           │
│ General(常规) ✗ 不可用│        │                          │
└─────────────────────┘        │ ccgram 守护进程            │
        ▲                      │  (systemd user service)   │
        │  Bot API 长轮询        │  · 轮询 Telegram 消息      │
        └──────────────────────│  · 把消息注入 tmux 终端     │
                               │  · 定期把终端画面截图回传    │
                               └──────────────────────────┘
```

核心概念:

- **一个话题(Topic)= 一个 tmux 窗口 = 一个独立的 Claude 会话**。所以群组必须开启"话题"功能,且不能在 General 里直接用。
- ccgram 以**守护进程**方式跑在服务器上,通过 Telegram Bot API **长轮询(getUpdates)** 收消息。⚠️ 这意味着**同一个 bot token 只能被一个程序使用**,两个程序抢同一个 token 会互相冲突丢消息。
- ccgram 会往 Claude Code 的 `~/.claude/settings.json` 里安装 **hooks**(会话跟踪钩子),这样它能感知 Claude 会话的启动/结束/状态,用于在 Telegram 里展示会话状态、自动关闭已结束的话题。

---

## 2. 环境要求与检查

| 依赖 | 要求 | 检查命令 |
|---|---|---|
| Python | 3.14+(uv 可自动下载,系统自带旧版没关系) | `python3 --version` |
| uv | 任意近期版本 | `which uv` |
| tmux | 有即可(实测 3.2a 可用) | `tmux -V` |
| claude CLI | 已安装且已登录 | `which claude` |
| systemd 用户实例 | 用于开机自启 | `systemctl --user status` |

本机实测环境:Node v22(无关)、系统 Python 3.9(太旧,由 uv 解决)、tmux 3.2a ✓、claude 在 `~/.local/bin/claude` ✓。

如果没有 uv,先安装:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

---

## 3. Telegram 侧准备(3 样东西)

### 3.1 Bot Token

1. Telegram 搜索 **@BotFather** → 发送 `/newbot`
2. 按提示起显示名和用户名(用户名必须以 `bot` 结尾,如 `your_bot`)
3. 得到形如 `1234567890:AAxxxxxxxx...` 的 **token**,保存好

> ⚠️ 不要复用其它程序(比如已有的通知 bot)正在用的 token,原因见第 1 节的长轮询冲突。**为 ccgram 单独建一个新 bot。**

### 3.2 你的 User ID(白名单用)

- 搜索 **@userinfobot**,随便发条消息,它回复你的数字 ID(如 `<你的UserID>`)
- 这个 ID 会写进 `ALLOWED_USERS`,只有白名单里的人能操作 bot——**这是唯一的鉴权手段,务必配置**

### 3.3 群组 ID

1. 新建一个 Telegram **群组**(不是频道)
2. 群组设置 → 开启 **Topics(话题)** 功能 ← **关键步骤,不开后面用不了**
3. 把你的 bot 拉进群,**设为管理员**,勾选 **管理话题(Manage Topics)** 权限
4. 把 **@RawDataBot** 拉进群,它会打印一段 JSON,找到 `chat.id`,形如 `<你的群组ID>`(话题群组的 ID 都带 `-100` 前缀)
5. 记下 ID 后把 @RawDataBot 踢出群

验证 token 是否有效(可选):

```bash
curl -s "https://api.telegram.org/bot<你的TOKEN>/getMe"
# 返回 {"ok":true,"result":{...,"username":"你的bot用户名",...}} 即有效
```

---

## 4. 安装 ccgram

ccgram 要求 Python 3.14+,用 uv 安装时指定 `--python 3.14`,uv 会自动下载对应的 Python,**不影响系统 Python**:

```bash
uv tool install ccgram --python 3.14
```

装完验证:

```bash
ccgram --version     # 实测装到的是 4.3.11
ccgram --help
```

可执行文件位于 `~/.local/bin/ccgram`(uv tool 的默认位置)。如果 `ccgram` 命令找不到,确认 `~/.local/bin` 在 `$PATH` 里。

子命令一览:

| 命令 | 作用 |
|---|---|
| `ccgram run` | 启动 bot 守护进程 |
| `ccgram doctor` | 自检环境和配置 |
| `ccgram status` | 查看运行状态 |
| `ccgram hook --install` | 往 Claude 配置里装会话跟踪 hook |

---

## 5. 写配置文件

配置放在 `~/.ccgram/.env`:

```bash
mkdir -p ~/.ccgram && chmod 700 ~/.ccgram
cat > ~/.ccgram/.env <<'EOF'
TELEGRAM_BOT_TOKEN=<BotFather给的token>
ALLOWED_USERS=<你的UserID>
CCGRAM_GROUP_ID=<群组ID,含-100前缀>
EOF
chmod 600 ~/.ccgram/.env
```

三个变量的含义:

| 变量 | 说明 | 本次部署的值 |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | bot 身份凭证 | (保密,bot 为 @your_bot(你的bot用户名)) |
| `ALLOWED_USERS` | 允许操作的用户 ID,逗号分隔多个 | `<你的UserID>` |
| `CCGRAM_GROUP_ID` | 限定只在这个群工作 | `<你的群组ID>` |

> `chmod 600` 很重要——token 泄露等于任何人都能冒充这个 bot。

---

## 6. 自检与安装 Claude hooks

### 6.1 跑 doctor

```bash
ccgram doctor
```

首次运行的实际输出(节选)及解读:

```
✓ multiplexer backend: tmux
✓ claude found at /home/devops/.local/bin/claude
✓ tmux 3.2a found
✗ tmux session "ccgram" not found     ← 正常,服务启动后自动创建
✗ no hook events installed            ← 需要执行 6.2 安装
✓ TELEGRAM_BOT_TOKEN set
✓ ALLOWED_USERS: 1 user(s)
```

前几项 ✓ 必须满足;两个 ✗ 按下面步骤解决。

### 6.2 安装 hooks

```bash
ccgram hook --install --provider claude
# 输出: Hooks installed in /home/devops/.claude/settings.json: 9 new, 0 already present
```

这一步把 9 个会话跟踪事件写进 `~/.claude/settings.json` 的 `hooks` 段。作用是让 ccgram 知道每个 Claude 会话的生命周期(启动、空闲、结束),从而在 Telegram 里正确显示状态并自动清理死会话。卸载用 `--uninstall`,检查用 `--status`。

---

## 7. 配置 systemd 开机自启

### 7.1 写 unit 文件

`~/.config/systemd/user/ccgram.service`:

```ini
[Unit]
Description=ccgram — Telegram <-> tmux bridge for Claude Code
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/devops/.local/bin/ccgram run
Environment=PATH=/home/devops/.local/bin:/home/devops/.bun/bin:/usr/local/bin:/usr/bin:/bin
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

**为什么要显式写 `Environment=PATH=...`**:systemd 用户服务默认的 PATH 非常精简(通常只有 `/usr/bin:/bin`),而 `claude` 装在 `~/.local/bin`。不写这行的话,ccgram 起来了,但它在 tmux 里拉起 `claude` 时会报 command not found——这类问题**日志里往往不明显,极难排查**(本机之前的 bun 版 telegram 插件就踩过同样的坑)。

### 7.2 启用并启动

```bash
systemctl --user daemon-reload
systemctl --user enable --now ccgram.service
systemctl --user status ccgram.service    # 应显示 active (running)
```

### 7.3 确认 linger(否则 SSH 断开/重启后服务不会自动跑)

```bash
loginctl show-user $USER --property=Linger
# 输出 Linger=yes 即可;若是 no,执行:
sudo loginctl enable-linger $USER
```

linger 的意思是:即使该用户没有登录会话,systemd 也保持其用户实例运行——服务器重启后 ccgram 会自动拉起。

### 7.4 验证

```bash
ccgram status
# ccgram 4.3.11
# Provider: claude (hook, resume, continue)
# Tmux session: ccgram (1 windows)      ← tmux 会话已自动创建
# Monitored sessions: 0
```

---

## 8. 使用方法

### 8.1 开始一个会话(重点!)

1. 打开群组 → 点右上角群名(或 ☰)进入**话题列表**
2. **✏️ 新建话题**,起个名字(如 `test`、项目名)
3. **进入该话题**发任意消息(如 `hi`)
4. bot 弹出**目录浏览器** → 选项目目录 → 选 `claude` → 会话启动
5. 之后在话题里发的消息都会注入到这个 Claude 会话;bot 会定期回传终端画面

### 8.2 常用操作

| 操作 | 方法 |
|---|---|
| 发指令给 Claude | 直接在话题里发文字 |
| 语音输入 | 直接发语音消息 |
| 从服务器取文件 | `/send <文件路径>` |
| 开第二个并行会话 | 再建一个新话题 |
| 回到服务器终端接管 | `tmux attach -t ccgram`,切到对应窗口 |

会话结束后,话题默认 30 分钟自动关闭;死掉的会话 10 分钟自动清理(可用 `ccgram run --autoclose-done N --autoclose-dead N` 调整)。

---

## 9. 问题处理实录

本次部署实际遇到/预判的问题:

### 问题 1:bot 提示 "Please use a named topic. Create a new topic to start a session."

- **现象**:在群里 @ 机器人或直接发消息,bot 回复上面这句话。
- **原因**:消息发在了 **General(常规)** 里。ccgram 的设计是"一个话题 = 一个会话",General 不映射任何会话,所以拒绝服务。
- **解决**:按 8.1 新建一个**命名话题**,在话题里发消息。若找不到"新建话题"入口,说明群组没开 Topics 功能(见 3.3 第 2 步)。

### 问题 2:系统 Python 版本太旧(3.9,ccgram 要求 3.14+)

- **解决**:不动系统 Python,用 `uv tool install ccgram --python 3.14`,uv 自动下载独立的 Python 3.14 运行时,隔离在 `~/.local/share/uv/` 下。

### 问题 3(预判规避):token 复用导致消息互抢

- **背景**:本机已有另一个 Telegram bot 服务(`claude-ops.service`)。Telegram 长轮询模式下,两个进程用同一个 token 调 `getUpdates` 会互相"抢"消息,表现为随机丢消息、bot 时灵时不灵,极难定位。
- **解决**:为 ccgram **新建独立 bot**,一个 token 只给一个程序用。

### 问题 4(预判规避):systemd 用户服务 PATH 缺失

- 见 7.1 的说明。症状是服务 active 但功能异常(拉不起 claude),写全 `Environment=PATH` 一劳永逸。

### 通用排查路径

```bash
ccgram doctor                                  # 第一步永远是这个
systemctl --user status ccgram.service         # 服务活着吗
journalctl --user -u ccgram.service -n 50      # 看日志(需在 systemd-journal 组,否则 sudo)
tmux attach -t ccgram                          # 直接看终端里到底发生了什么
curl -s "https://api.telegram.org/bot<TOKEN>/getMe"   # token 还有效吗
```

> 注意:排查时**不要**手动 `curl .../getUpdates`——这会把消息从正在运行的 ccgram 手里抢走(又是问题 3 的坑)。

---

## 10. 运维命令速查

```bash
# 服务
systemctl --user restart ccgram.service   # 重启(改完 .env 后必须执行)
systemctl --user stop ccgram.service      # 停止
journalctl --user -u ccgram.service -f    # 实时日志

# ccgram 自身
ccgram doctor        # 自检
ccgram status        # 运行状态、会话数
ccgram hook --status --provider claude    # hook 是否在位

# 升级
uv tool upgrade ccgram && systemctl --user restart ccgram.service

# 彻底卸载
systemctl --user disable --now ccgram.service
rm ~/.config/systemd/user/ccgram.service && systemctl --user daemon-reload
ccgram hook --uninstall --provider claude
uv tool uninstall ccgram
rm -rf ~/.ccgram
```

---

## 11. 安全注意事项

1. **token 即身份**:泄露 token = 任何人可控制你的 bot。若 token 曾出现在聊天记录/截图中,去 @BotFather 用 `/revoke` 作废旧 token,把新 token 写入 `~/.ccgram/.env` 后 `systemctl --user restart ccgram.service`。
2. **`ALLOWED_USERS` 是唯一门禁**:它决定谁能通过 bot 在你服务器上执行 Claude(而 Claude 能执行 shell)。只加自己的 ID,不要图方便留空。
3. **群组要私密**:虽然有白名单,群成员仍能看到会话内容(终端截图、文件),不要拉无关人员。
4. **文件权限**:`~/.ccgram` 700、`.env` 600,已在第 5 步设置。
5. **风险认知**:这套东西本质是"从手机远程在服务器上执行任意命令"的通道,便利与风险并存——上面几条是底线,不要放松。

---

## 附:本次部署最终状态

| 项 | 值 |
|---|---|
| ccgram 版本 | 4.3.11(uv + Python 3.14) |
| Bot | @your_bot(你的bot用户名) |
| 群组 ID | `<你的群组ID>` |
| 白名单 | `<你的UserID>` |
| 配置 | `~/.ccgram/.env`(600) |
| Hooks | `~/.claude/settings.json`,9 个事件 |
| 服务 | `ccgram.service`(user unit,enabled,linger=yes) |
| tmux 会话 | `ccgram` |

---

## 12. Telegram 命令手册

> 基于 ccgram 4.3.11 实际注册的命令列表 + 官方 [docs/guides.md](https://github.com/alexei-led/ccgram/blob/main/docs/guides.md) 整理。

### 12.1 会话创建与管理

| 命令 | 作用 | 说明 |
|---|---|---|
| `/new` | 新建会话 | 弹目录浏览器 → (git 仓库还会问是否建 worktree 隔离分支)→ 选 provider → 启动 |
| `/sessions` | 会话仪表盘 | 所有话题/窗口的状态总览 |
| `/resume` | 恢复历史会话 | 浏览 Claude 历史对话,选一个接着聊(对应 `claude --resume`) |
| `/restore` | 救活死话题 | 崩溃后弹三个按钮:**Fresh**(原目录重开)/ **Continue**(继续上次对话)/ **Resume**(挑历史会话) |
| `/unbind` | 解绑话题 | 话题与 tmux 窗口解除关联(话题保留,窗口不再受控) |
| `/sync` | 状态审计修复 | 话题↔窗口映射乱了(如手动杀过 tmux 窗口)时对账修复 |

### 12.2 查看终端

| 命令 | 作用 | 说明 |
|---|---|---|
| `/screenshot` | 截一张终端图 | 带 ANSI 颜色的 PNG |
| `/live` | 实时监视 | 每 5 秒自动刷新,默认 5 分钟自动停;内容无变化不重发(省流量) |
| `/last` | 重发最近一条回复 | 超 4096 字符自动转 .txt 附件 |
| `/history` | 本话题消息历史 | 回看话题内往来记录 |
| `/panes` | 窗格列表/通知开关 | 查看 tmux pane;控制 pane 创建/关闭是否通知 |

### 12.3 文件与输入

| 命令 | 作用 | 说明 |
|---|---|---|
| `/send` | 从服务器取文件 | `/send docs/a.png`(精确)、`/send *.png`(glob)、`/send arch`(模糊);**不带参数=弹目录浏览器**。安全限制:不出项目目录、自动屏蔽 `.env`/`.pem`/`.key` 等、上限 50MB |
| `/recall` | 最近命令回放 | 快速重发之前的输入 |
| 🎤 语音消息 | 语音转文字 | 需配置 Whisper API,转写后弹「✓发送 / ✗丢弃」确认按钮 |

### 12.4 终端按键控制(重点)

`/toolbar` 弹出按键工具栏(Claude 模式布局):

```
[📷 截屏] [⏹ Ctrl-C] [📺 Live]
[Mode]   [Think]    [Esc]
[↑ 上]   [↵ 回车]   [↓ 下]
[📄 Last] [Get File] [关闭]
```

Claude 在终端弹出选择菜单(`/model` 选型号、权限确认等)时,用 ↑↓+回车远程操作;`Esc` 打断当前输出;`Ctrl-C` 强制中断。可通过 `~/.ccgram/toolbar.toml` 自定义按钮。

### 12.5 杂项

| 命令 | 作用 |
|---|---|
| `/commands` | 列出当前 provider 可用命令(带 ↗ 的如 `/clear`、`/model`、`/usage` 会直接注入 Claude 终端) |
| `/verbose` | 切换工具调用消息批量/详细显示 |
| `/agent` | 手动切换/纠正 provider(自动识别错了才用) |
| `/upgrade` | 远程升级 ccgram 并自动重启 |

---

## 13. 使用技巧

1. **话题标题 emoji 就是状态灯**:🟢 = Claude 正在干活,🟡 = 空闲等你。扫一眼话题列表即可知道哪个会话需要处理。(`CCGRAM_STATUS_MODE=user` 可反转语义)
2. **并行开工**:每个话题完全独立,可同时开多个话题跑不同任务,状态灯变色就知道该看哪个。
3. **git 项目用 worktree 隔离**:`/new` 时选择创建 worktree,自动建 `ccg/<话题名>` 分支,多会话改同一仓库互不冲突。
4. **开启语音输入**:`.env` 追加(Groq 有免费额度):
   ```ini
   CCGRAM_WHISPER_PROVIDER=groq
   GROQ_API_KEY=gsk_xxxx
   ```
   重启服务后直接发语音即可。
5. **`/send` 别背语法**:发不带参数的 `/send`,目录浏览器点选比记 glob 快。
6. **会话死了优先 Continue**:`/restore` → **Continue** 比重开快,Claude 带着上下文继续。
7. **随时回终端接管**:`tmux attach -t ccgram`,终端才是"本体",手机和终端看到的是同一个会话。
8. **消息太吵**:发 `/verbose` 切批量模式,或 `.env` 加 `CCGRAM_HIDE_TOOL_CALLS=true`。

---

## 14. 已发出内容如何撤销/中止

**关键认知:文字消息一发出就立即注入终端,在 Telegram 里删除消息无法撤回已注入的内容。** 补救手段按场景:

| 场景 | 操作 |
|---|---|
| 指令发出去了,Claude 正在执行,想停 | `/toolbar` → **Esc**(温和打断当前回合,上下文保留) |
| Esc 不管用/程序卡死 | `/toolbar` → **⏹ Ctrl-C**(强制中断) |
| 语音消息 | 转写后有「✓发送 / ✗丢弃」确认步骤,**这是唯一有"发送前反悔"机会的输入方式**,点 ✗ 即取消 |
| Claude 要执行危险操作(删文件、改配置) | 只要**没开** `--dangerously-skip-permissions`,Claude Code 自身会弹权限确认,此时选拒绝即可——这是最后一道闸门 |
| 整个会话不要了 | `/unbind` 解绑话题,或 tmux 里直接杀窗口后 `/sync` 对账 |

> 建议:重要环境下不要给 Claude 开 yolo/skip-permissions 模式,保留权限确认这道闸,"撤销"就永远来得及。

---

## 15. 多人协作与权限治理

### 15.1 ccgram 的权限模型(事实)

ccgram **只有一层鉴权**:`ALLOWED_USERS` 白名单(源码中每个操作入口都检查 `is_user_allowed(user.id)`)。特点:

- **全有或全无**:白名单内 = 完全控制(发指令、取文件、升级、解绑);白名单外 = bot 完全无视其消息
- **没有**角色分级(管理员/普通用户)、**没有**内置审批流、**没有**操作审计报表

### 15.2 "开发能看、领导审批"的实现方案

虽然 ccgram 没有内置审批,但用现有机制可以组合出等效效果:

**方案 A:白名单只放审批人(推荐,零开发)**

```ini
# .env 中只写领导/管理员的 User ID
ALLOWED_USERS=<领导UserID>
```

- 开发们仍在群里:**能看到**所有终端截图、Claude 输出、文件——透明可见
- 但他们发的任何消息 bot 一律无视——**想执行操作必须让白名单内的人代发** = 天然的人工审批
- 多个审批人用逗号分隔:`ALLOWED_USERS=111,222`

**方案 B:保留 Claude Code 自身的权限确认闸门**

启动 Claude 时不要用 skip-permissions 模式。危险操作(写文件、执行命令)会在终端弹确认框,而**只有白名单用户能通过 /toolbar 按下确认键**。即使把开发加进白名单让他们能发指令,配合 Claude Code 的 `PermissionRequest` hook 还能把审批请求单独路由给领导(私聊/单独频道),实现"开发发指令、领导批权限"的两级模式。

**方案 C:按项目拆分实例**

跑多个 ccgram 实例(不同 bot token + 不同群 + 不同 `TMUX_SESSION_NAME` + 不同系统账号),生产环境群白名单只放领导,测试环境群放开发——用"环境隔离"代替"角色分级"。

### 15.3 防篡改/防恶意删除清单

| 风险 | 防线 |
|---|---|
| 普通成员冒充操作 | 不加入 `ALLOWED_USERS` 即可,bot 无视非白名单消息(默认已具备) |
| 普通成员删话题/删消息捣乱 | **Telegram 群权限**:群设置 → 权限,关闭普通成员的「管理话题」;删话题只是断了 Telegram 侧入口,tmux 会话仍在,`/restore` 或终端可恢复 |
| 白名单成员误操作/越权 | 方案 B 的权限确认 + 服务器上 `~/.claude/settings.json` 配置 Claude Code 的 permissions deny 规则(如禁止 `rm -rf`、禁止读密钥文件) |
| 恶意改 ccgram 配置 | `.env` 在服务器上,只有有 SSH 权限的人能改;保持 600 权限 |
| 事后追责 | tmux 会话滚动缓冲 + `/history` + Claude Code 的会话记录(`~/.claude/projects/`)都是操作留痕;群消息本身也是审计日志 |
| 爆炸半径控制 | 用**专用低权限系统账号**跑 ccgram 服务,Claude 能干的坏事 ≤ 该账号的权限;绝不用 root 跑 |

> 一句话总结:**ccgram 管"谁能说话"(白名单),Claude Code 管"哪些操作要确认"(权限模式+hooks),Telegram 群权限管"谁能动话题",系统账号管"最坏能坏到哪"。四层叠加,才是完整的治理。**
