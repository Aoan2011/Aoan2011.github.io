# 附录 · 速查与考据

本附录是全书的后勤手册：查命令、查配置、查文件、查报错、查术语、考查据。所有内容以 v3（`F:\one-cedric3`，Public Beta I）为准；标注了章节的地方可以直接跳回去看完整上下文。

## A.1 三个版本对照

全书对照的三份代码快照：

| | **v1** | **v2** | **v3** |
| --- | --- | --- | --- |
| 时间 | 2026-09-19 首发 | 2026-10-初（大开发日后） | 2026-10-11（Public Beta I） |
| 工具数 | **49** | **242** | **250** |
| handler 数 | —（无 `_dispatch` 层） | — | **171** |
| `tools/` 模块数 | 17 | 62 | 61 + `office/`(6) + `sysops/`(14) |
| 顶层模块 | `ui.py` 单文件；无 gateway | + `memory` `cron` `dream` `cost_tracker` `pricing` `i18n`；目录 + `gateway/` `hooks/` `mcp/` `ui/` | + `_version.py` `integrations.py` `my_models.py` `winvt.py` `agents_skills.py` `browser_tools.py` |
| 测试 | 无 `tests/` 目录 | 无 `tests/` 目录 | **12 个测试文件 / 249 个测试 + CI + CodeQL** |
| 对应章节 | 第 2–4 章雏形 | 第 5–11 章主体 | 全书终态 |

三句话版本故事：

- **v1**：一个能跑的单包 Agent——单文件 `ui.py`、49 个工具、没有网关也没有记忆（第 2–4 章的"最小系统"放大版）；
- **v2**：大开发日的产物——工具冲到 242，长出 gateway、webui、memory、dream、cron、MCP、hooks（第 5–11 章对应这一段）；
- **v3**：发布形态——拆分文件、补 TUI 输入体验、补安全修复、补测试与 CI，版本号 **0.17.0 / Public Beta I**（第 12 章）。

## A.2 命令行与命令速查

### 启动与全局参数

```text
python -m one_cedric [options]

--chat                 jump straight into the chat REPL
--gateway, --gw        start the WebUI gateway and keep running (Ctrl+C stops)
--bind <addr>          gateway bind address (default from config, 127.0.0.1)
--port <port>          gateway port (default from config, 2043)
--token <secret>       require a Bearer token on the gateway
--host <addr>          model/API endpoint, NOT the gateway address
--license              print the GPL-3.0 license text and exit
--version              print version and project info
--sandbox-terminal     force the shell sandbox on
--no-sandbox-terminal  force the shell sandbox off
-y, --yes              auto-confirm write/shell actions (unattended runs)
-p, --prompt <text>    run one prompt and exit (needs -y to be unattended)
```

三个容易踩的坑（第 12 章的安全修复带来的行为变化）：

- **`--prompt` 不再自动确认**——无人值守必须显式加 `-y`；
- **`--host` 是模型地址**，不是网关地址（网关用 `--bind`）；
- **`--bind 0.0.0.0` 必须配 `--token`**——否则同一局域网里任何人都能用你的 Agent。

常用组合：

```bash
python -m one_cedric --chat                          # 直接进聊天
python -m one_cedric --gateway                       # 启动 WebUI（默认 http://127.0.0.1:2043）
python -m one_cedric --gateway --token <secret>      # 带鉴权的网关
python -m one_cedric -p "总结这个仓库" -y             # 一次性任务
python -m one_cedric --version                       # One Cedric 0.17.0 (Public Beta I)
```

### CLI 斜杠命令

| 命令 | 作用 |
| --- | --- |
| `/help` | 帮助 |
| `/new` `/clear` `/sessions` `/resume` `/save-as` `/export` | 会话管理 |
| `/model` `/think` `/reasoning` `/mode` | 模型与思考档位 |
| `/shell on\|off` | 允许任意 shell 命令 |
| `/sandbox on\|off` | 沙箱终端开关 |
| `/tools [list\|install\|uninstall\|gen]` | 工具管理 / 自定义工具 |
| `/plan` `/todos` | 计划模式与任务清单 |
| `/undo` `/rollback` `/snapshots` | 回退文件修改（git 快照） |
| `/diff [n]` | 彩色 diff |
| `/history` `/tag` `/untag` | 文件历史与标签 |
| `/stats` `/audit` `/cost` | 统计、审计与成本 |
| `/mem` `/cron` `/asks` | 记忆、定时任务、ask_user 线程 |
| `/config` `/save-config` `/profile` `/lang` | 配置与语言 |

> 表中是 README 收录的常用命令；源码里还分发 `/replay` `/cache` `/compact` `/computer` `/log` `/kill` `/email` `/amap-key` `/agent` `/review` 等维护/调试命令（逻辑在 `core.py` 的命令分发段）。

### WebUI 斜杠命令

`/new` `/clear` `/search` `/export` `/theme` `/lang` `/settings` `/sidebar` `/help` `/stats` `/model` `/think` `/sessions` `/tools` `/asks` `/sandbox` `/diagnose`

### 输入框交互（v3 新增）

- **鼠标**：点击行内任意位置移动光标；
- **编辑**：方向键、Home/End、Delete、Ctrl+W 式词移动；
- **回合中插话**：Agent 干活时按 Enter——**queue**（回车，本轮结束后执行）/ **interject**（`i`，打断当前回答立即处理）/ **cancel**（Esc，丢弃）；
- 管道输入（CI、`echo … | one-cedric`）时自动降级为经典行输入（第 12 章 `3b48993`）。

## A.3 配置与数据目录

### `~/.one-cedric/` 全景

| 文件 / 目录 | 内容 | 章节 |
| --- | --- | --- |
| `config.toml` | 全局配置（见下） | 3 / 7 |
| `projects/<key>.toml` | 项目级配置（覆盖全局） | 3 |
| `profiles/` | 工作画像 | 4 |
| `sessions/` | 会话记录（JSON） | 3 |
| `memory.toml` | 长期记忆 | 11 |
| `my_models.toml` | 自定义模型档案（含 api_key，明文——SEC-08） | 6 / 12 |
| `integrations.json` | MCP 服务器 / hooks / 自定义工具 | 11 |
| `custom_tools/<name>.py` | AI 生成的自定义工具脚本 | 5 / 11 |
| `cron.toml` + `cron_log/` | 定时任务与运行日志 | 11 |
| `dreams/dream_<时间戳>.md` | 梦境产出 | 11 |
| `costs.jsonl` + `budget.toml` | 成本流水与预算 | 8 |
| `pricing.toml` | 模型价格表 | 8 |
| `email.toml` / `amap.toml` / `lsp.toml` | 各集成配置 | 9 / 11 |

### `config.toml` 默认值（`config.py:DEFAULT_CONFIG`）

```toml
[default]
model = "…"                 # 默认模型（可用 /model 改）
host = "…"                  # OpenAI 兼容端点
api_key = ""
temperature = 0.2
auto_yes = false            # -y 的配置版本
max_steps = 8               # 单轮最多工具步数
compact_keep_turns = 4      # 压缩时保留最近 N 轮
auto_compact_threshold = 0  # 0 = 按默认阈值触发
show_reasoning = true
think_level = "medium"      # minimal/low/medium/high/xhigh/max/ultra
enable_computer_use = false
enable_vision = false
allow_arbitrary_shell = false
sandbox_terminal = true
mode = "ask"                # allow / ask / assisted
language = "zh"

[bash]
extra_prefixes = []         # 追加 shell 白名单前缀
extra_blacklist = []        # 追加黑名单

[gateway]
enabled = false
host = "127.0.0.1"
port = 2043
token = ""

[amap]
api_key = ""
```

## A.4 源码文件地图（v3）

```text
one_cedric/
├── _version.py            # 版本单一来源：0.17.0 / "Public Beta I"（第 12 章）
├── cli.py                 # 入口：参数解析、--gateway 启动、REPL 装配（第 2–4 章）
├── core.py                # 心脏：agent 循环、权限、沙箱、确认、快照（第 6–8 章）
├── config.py              # 默认配置、系统提示词、shell 策略表（第 3 / 7 章）
├── storage.py             # 会话 / 全局与项目配置 / 画像（第 3 章）
├── cache.py               # ToolCache：LRU + TTL（第 8 章）
├── at_ref.py              # @文件引用附件构造（第 11 章）
├── integrations.py        # MCP / hooks / 自定义工具（第 11 章）
├── memory.py              # 长期记忆（第 11 章）
├── dream.py               # 梦境回顾（第 11 章）
├── cron.py                # 定时任务（第 11 章）
├── cost_tracker.py        # 成本记账（第 8 章）
├── pricing.py             # 价格表（第 8 章）
├── git_snap.py            # git 快照 / 回退（第 7 章）
├── i18n.py                # 中英双语（第 4 章）
├── my_models.py           # 模型档案（第 6 章）
├── winvt.py               # Windows 虚拟终端支持（第 4 章）
├── agents_skills.py       # ~/.agents/skills 只读技能（第 11 章）
├── browser_tools.py       # 浏览器自动化（第 11 章）
├── tools/
│   ├── schema.py          # 250 份 schema + 分类集合（第 5 章）
│   ├── _dispatch.py       # 注册表合并与分发（第 5 章）
│   ├── _handlers_*.py     # 5 类 handler（readonly/text/media/agent/extra）（第 5 章）
│   ├── normalize.py       # 工具名/参数归一化与 JSON 修复（第 5 章）
│   ├── sandbox.py         # 路径沙箱（第 7 章）
│   ├── shell.py           # 三层命令分类（第 7 章）
│   ├── office/            # Word/Excel/PPT/PDF（第 9 章）
│   └── sysops/            # 系统操作（第 9 章）
├── gateway/
│   ├── server.py          # FastAPI：44 条路由、SSE、鉴权（第 10 章）
│   └── webui/             # Vue 单页：tokens.css → style.css → polish.css（第 10 章）
├── ui/                    # TUI：主菜单/会话/设置/统计 + live_input.py（第 4 / 12 章）
├── lsp/                   # LSP 客户端（第 11 章）
└── computer/              # 鼠标/键盘/窗口（第 11 章）

tests/                     # 249 个测试（第 12 章）
.github/workflows/         # ci.yml + codeql.yml（第 12 章）
SECURITY_AUDIT.md          # 17 项安全审计（第 7 / 12 章）
TOOL_TEST_REPORT.md        # 250 工具冒烟报告（第 12 章）
```

## A.5 报错对照表

全书出现过的真实报错（按"看到什么 → 去第几章 → 一句话根因"）：

| 报错 / 现象 | 章节 | 根因一句话 |
| --- | --- | --- |
| `TypeError: Spinner.__init__() got an unexpected keyword argument 'frames'` | 00 / 04 | Rich 版本差异：旧版 Spinner 不接受 `frames`——UI 依赖要锁版本 |
| `ImportError: attempted relative import with no known parent package` | 03 | 单文件脚本直接跑，相对导入没有"包"语境 |
| `SyntaxError: (unicode error) 'unicodeescape' codec can't decode bytes … truncated \xXX` | 03 / 06 | Windows 路径反斜杠写进了普通字符串——写 `r"F:\..."` 或用 `/` |
| `ModuleNotFoundError: No module named 'one_cedric.sandbox'` | 03 | 模块搬家后调用方没跟——拆分包时的经典 | 
| `ImportError: cannot import name 'set_policy' from 'one_cedric.computer'` | 03 | 公共出口改名/迁移，旧导入路径失效 |
| `ModuleNotFoundError: No module named 'one_cedric.tools.backend'` | 03 / 05 | `tools/__init__.py` 里残留坏 import——包出口的一行坏代码是全局故障 |
| 模型把工具调用**写成普通文本**（`{"name": "write_file", …}` 出现在回复里） | 06 | 模型不守工具协议——需要"文本工具调用恢复"兜底 |
| 只思考不回答 / 思考过程显示后消失 | 06 | `reasoning_content` 与 `content` 分离处理；纯推理轮要有兜底提示 |
| `'tuple' object has no attribute 'encode'` | 11 | `build_attachment` 返回 `(block, warning)` 元组，调用方当字符串用——多返回值契约 |
| 新克隆仓库**一启动就 ImportError** | 12 | `winvt.py` / `my_models.py` 从未提交（`0a0cf2f` 修复） |
| WebUI **黑屏**停在启动画面 | 12 | 严格 CSP 与 Vue 全局构建的 `new Function` 冲突（`ce13ff7` 修复） |
| 终端**中文乱码 / 框线崩溃** | 04 | 字体与渲染宽度问题；v3 提供图标降级开关 |
| `pip install` 装不上某个依赖 | 12 | 分发名写错（`bs4`→`beautifulsoup4` 等，`dfd709d` 修复） |
| 30 个工具"跑不通" | 12 | 全是环境依赖（缺库/缺外部程序），不是代码缺陷——看报告里的修复命令 |
| 调用工具报 `ERROR: 未知工具` | 05 | 三种可能：schema 里没有 / handler 没注册 / core 没接（用 12.4 的测试一秒定位） |

## A.6 术语表

| 术语 | 一句话解释 | 章节 |
| --- | --- | --- |
| **Agent 循环** | "模型 → 工具 → 结果 → 模型"的反复循环，直到模型不再调工具 | 1 / 8 |
| **schema** | 给模型看的工具声明（name/description/parameters），纯数据 | 5 |
| **handler** | 工具的真正实现：`fn(root: Path, args: dict) -> str` | 5 |
| **`_dispatch`** | schema 名字 → handler 的查表分发 | 5 |
| **preview / apply** | 写工具的两次调用约定：先算"将发生什么"，确认后才真正落盘 | 5 / 7 |
| **`WRITE_TOOLS` / `READONLY_TOOL_NAMES`** | 工具只读/写的集中分类；交集记入 `AMBIGUOUS_…` 留档 | 5 |
| **SSE** | Server-Sent Events，`data: {...}` 逐行推送的流式协议 | 6 |
| **`delta`** | SSE 增量块：`content` 要拼接，`tool_calls` 更要拼接 | 6 |
| **`reasoning_content`** | DeepSeek 等推理模型单独返回的"思考"字段 | 6 |
| **think 档位** | minimal→ultra 共 7 档，映射到不同的思考预算与提示词 | 6 |
| **`confirm_5`** | 写操作确认的 5 选项菜单（是/总是/编辑/否/…） | 7 |
| **快照（snapshot）** | 写前用 `git stash push --include-untracked` 记录的撤销点 | 7 |
| **三层 shell 策略** | 白名单直跑 / 已知一次确认 / 未知强警告（+ 黑名单永久拒绝） | 7 |
| **沙箱环境** | 传给子进程的清洗后 env——剥掉 `*_KEY`/`*_TOKEN`/`*_PASS` 等 | 7 |
| **plan 模式** | 只许只读工具的"计划阶段" | 7 / 8 |
| **ToolCache** | 只读工具结果的 LRU+TTL 缓存；写操作后清空 | 8 |
| **合批（batching）** | 一轮里多个工具调用按"读/写/错误"分组处理，只读并行、写串行 | 8 |
| **interject** | 回合进行中插话：打断当前回答，立即处理新消息 | 8 / 12 |
| **compact** | 上下文压缩：保留最近 N 轮，旧对话总结 | 8 |
| **Gateway** | FastAPI 本地服务器，为 WebUI 提供 REST + SSE | 10 |
| **ask_user 往返** | Agent 通过网关向浏览器提问、等回答（`Event` + 注册表 + 超时） | 10 |
| **MCP** | 外部工具协议：stdio 启动服务器、列工具、调用 | 11 |
| **hooks** | 生命周期事件钩子：事件 JSON 进 stdin，15 秒超时 | 11 |
| **dream（梦境）** | 后台回顾最近会话，产出洞察与记忆候选 | 11 |
| **@引用** | 用户消息里 `@file`，构造附件块附在消息末尾 | 11 |
| **冒烟测试** | 250 个工具每个都"碰一下"：OK/SKIP/RUN-ERR/REG-ONLY 四桶 | 12 |
| **CodeQL** | GitHub 的静态安全扫描（python + javascript 两个语言） | 12 |

## A.7 素材考据

### 对话索引（#46–#69）

| # | 日期 | 主题 | 与教程的关系 |
| ---: | --- | --- | --- |
| 46 | 09-05 | 放飞理想体操 emoji 版 | 前史 |
| 47 | 09-05 | One-Editor 项目介绍 | Predecessor 项目 |
| 48 | 09-06 | One-Editor 网站总结 | Predecessor 项目 |
| 49 | 09-06 | DeepSeek 粒子光效分析 | 前史 |
| 50 | 09-11 | Python 开源 CLI 推荐 | 调研起点（第 1 章） |
| 51–53 | 09-12 | claw-agent：对接 / ASCII 界面 | 竞品观察（第 1 章） |
| 54–56 | 09-13 | GitHub 镜像 / 主页游戏 / jiter 403 | 环境准备 |
| 57 | 09-13 | **简易 Python Agent CLI** | 最小 Agent（第 2 章） |
| 58 | 09-18 22:59 | **pytest 介绍** | 两分钟后的"测试没意义"（第 12 章） |
| 59 | 09-25 | Claude Code tools count | 工具调研（第 5 章） |
| 60–62 | 09-25~26 | xueersi 解析 / 高德路线 / 邮件代码 | 部分工具来源（第 9 章） |
| 63 | 09-26 | Claude Code 内置工具 | 工具调研 |
| 64 | 09-26 | Win7 Aero 按钮网页 | UI 灵感 |
| 65 | 09-27 | 泄露 API 密钥提醒 | 安全警觉 |
| 66 | 09-27 | Textual 打包 APK | TUI 探索（第 4 章） |
| 67 | 09-18 → 10-02 | **One Copilot Rich 改造**（导出 3.75 MB） | 全书主干：从 19 工具到 250 |
| 68 | 10-01 → 10-02 | **One Cedric项目完成**（导出 4.38 MB） | 大开发日与收尾 |
| 69 | 10-02 | One Cedric项目继续 | 批量交付续章（第 9 章） |

### v3 git 里程碑

```text
3a8349c 09-19 13:30 Initial commit
cf1d3ce 10-04 18:35 beta17                    ← 版本号 0.17.0 的来历
6d806cb 10-04 18:41 Merge branch 'main'       ┐ beta17 之后半小时内，
8e0620f 10-04 18:48 dependabot: pip           ├ 仓库的"门禁"设施一次性
8210436 10-04 18:50 SECURITY.md               ├ 补齐：依赖机器人、代码
1a79af5 10-04 18:50 CodeQL workflow           │  扫描、安全政策、issue
12dfb7f 10-04 18:52 issue templates           ┘ 模板
6111867 10-05 22:30 "20261005"                ← 这一天只有这一条提交，信息就是日期
835388f 10-06 21:17 Public Beta 1             ┐
b4961f0 10-06 21:43 Public Beta I             ├ 一晚三连：发布命名定型
6adf3dd 10-06 22:01 Public Beta I             ┘
7708fcb 10-07 15:13 Add some tools and improved WebUI
0376a12 10-11 13:20 chore: checkpoint before optimization (WIP baseline)
53266d9 10-11 13:21 chore(repo): stop tracking build artifacts, bytecode caches and vendored node_modules
0a0cf2f 10-11 13:23 fix(repo): track winvt.py and my_models.py, add .gitattributes
dfd709d 10-11 13:26 fix(deps): correct distribution names, split dev/optional requirements, unify version
188421a 10-11 13:58 test: add a pytest suite, CI workflow and fix the bugs it found
47c9c19 10-11 14:02 fix(security): close the audit findings that were reachable today
3b48993 10-11 14:06 feat(cli): bottom-pinned REPL input box with mouse, editing and mid-turn queue/interject
b22c7d0 10-11 14:09 feat(webui): layer the stylesheets and add a visual/interaction polish pass
af22158 10-11 14:15 chore(lint): make the blocking ruff set actually pass
ce13ff7 10-11 14:26 fix(webui): allow 'unsafe-eval' so the Vue app can mount (black screen)
```

### 数字考据（写作时实测）

| 数字 | 值 | 来源 |
| --- | ---: | --- |
| v1 / v2 / v3 工具数 | 49 / 242 / 250 | 各仓库 `schema.py` 导入实算 |
| v3 handler 数 | 171 | `_dispatch._HANDLERS` 实算 |
| v3 只读 / 写 / 缓存集合 | 160 / 90 / 42 | `schema.py` 三集合实算（分类见第 5 章） |
| pytest 测试数（干净 HEAD） | 249（全绿） | `pytest --collect-only -q`（Python 3.14.3 / Windows） |
| 提交信息里的测试数轨迹 | 151 → 189 → 247 → 249 | `188421a` / `47c9c19` / `af22158` 提交信息 vs 现状 |
| 工具冒烟 | SKIP 111 / OK 85 / RUN-ERR 30 / REG-ONLY 24 | `TOOL_TEST_REPORT.md` |
| 安全审计 | 17 项（4 严重），当天修 10 项 | `SECURITY_AUDIT.md` + `47c9c19` |
| ruff 发现 / 自动修 | 629 / 461 | `af22158` 提交信息 |

## A.8 延伸阅读

**本项目资料**

- 仓库：<https://github.com/Aoan2011/One-Cedric>（GPL-3.0）
- 本仓库内一手材料：`README.md`、`SECURITY_AUDIT.md`、`TOOL_TEST_REPORT.md`、`.github/workflows/ci.yml`
- 官方 Issue / Discussions 入口（见 `SECURITY.md`）

**协议与 API 文档**（第 6 章要求对照阅读）

- OpenAI Chat Completions：<https://platform.openai.com/docs/api-reference/chat>
- DeepSeek API：<https://api-docs.deepseek.com/>
- MCP（Model Context Protocol）：<https://modelcontextprotocol.io/>
- SSE 规范：<https://html.spec.whatwg.org/multipage/server-sent-events.html>

**工具与库**

- pytest：<https://docs.pytest.org/>｜ruff：<https://docs.astral.sh/ruff/>
- FastAPI：<https://fastapi.tiangolo.com/>｜Starlette TestClient：<https://www.starlette.io/testclient/>
- Rich：<https://rich.readthedocs.io/>｜Textual：<https://textual.textualize.io/>
- Vue 3：<https://vuejs.org/>｜marked：<https://marked.js.org/>｜DOMPurify：<https://github.com/cure53/DOMPurify>
- CodeQL：<https://codeql.github.com/>

**继续学习的方向**

- 把第 12 章的"自己动手"清单做完（契约测试 → CI → 只读安全审计 → 修一条真实的安全发现）；
- 回到第 5 章 5.13，把三个工具的"可选步骤"补全（动画帧、冒烟报告）；
- 拿你的项目跑一遍 A.5 的报错对照表，把属于你的那几行加上。

---

[← 上一章：12 测试、审计与发布](12-testing-release) ｜ [返回目录](Home)
