# 11 · 记忆、自动化与生态：让 Agent 长出"长期生活"

## 11.1 从"一次对话"到"一直活着"

📅 **开发日志 · 2026-09-26 / 09-27 / 10-01**

> **09-26 12:59 用户**：增加使用 lsp 来读取智能分析代码文件，支持配置 lsp
>
> **09-26 13:00 用户**：再加 @上传文件
>
> **09-27 11:02 用户**：新增 computer use 功能
>
> **10-01 19:56 用户**：增加梦境功能和定时梦境

到这一天，One Cedric 已经不只是"你问一句、它答一句"。它开始有：

- **长期记忆**：跨会话记住你的偏好和项目背景；
- **梦境**：在后台回顾最近的会话，生成洞察和记忆候选；
- **定时任务**：按 cron 表达式自动执行 prompt；
- **子 Agent**：在主上下文之外跑只读探索，返回摘要；
- **外部生态**：MCP 服务器、Hooks、自定义工具、第三方技能；
- **代码智能**：LSP 语义分析；
- **桌面自动化**：鼠标、键盘、窗口。

这一章逐个讲它们的设计，重点不是 API 细节，而是**"一个本地 Agent 如何安全地扩展自己"**。

## 11.2 长期记忆：一个 TOML 文件就够了

`memory.py` 的文档字符串只有一行：

```python
"""长期记忆：跨会话记住用户偏好、项目背景。

存储: ~/.one-cedric/memory.toml
"""
```

### 数据结构

```python
CATEGORIES = ("preference", "context", "profile", "project", "other")

{
    "id": "mem_1a2b3c4d",
    "key": "回答语言",
    "value": "中文",
    "category": "preference",
    "created_at": 1759...,
    "updated_at": 1759...,
    "source": "user",          # user / dream
    "tags": ["语言", "偏好"],
}
```

五个分类覆盖了大多数场景：

- `preference`：回答语言、代码风格、常用工具；
- `context`：当前项目的背景；
- `profile`：用户身份、角色；
- `project`：项目级事实（"这个仓库用 pytest，不用 unittest"）；
- `other`：兜底。

### 写入是"按 key upsert"

```python
def add(key, value, category="other", tags=None, source="user") -> str:
    ...
    for it in items:
        if it.get("key", "").lower() == key.lower():
            it["value"] = value
            it["category"] = category
            ...
            return it.get("id", "")
    mid = _new_id()
    items.append({...})
    _save(data)
    return mid
```

同一个 key 不会产生两条记录，而是更新——这让"用户改主意"变成覆盖，而不是堆积矛盾信息。

### 读取是"注入 system prompt"

每轮请求前，`_stream_chat` 会调用 `memory.to_prompt_block()`：

```python
def to_prompt_block(max_items: int = 50, max_chars: int = 4000) -> str:
    ...
    lines = ["## 长期记忆", "",
             "以下是你记住的关于用户和项目的持久信息，"
             "对话中优先遵守（除非用户明确改变）：", ""]
    for it in items[:max_items]:
        line = (f"- [{it.get('category', 'other')}] "
                f"**{it.get('key', '')}**: {it.get('value', '')}")
        if total + len(line) > max_chars:
            lines.append(f"- ...（还有 {len(items) - shown} 条未展示）")
            break
        ...
```

三个限制：**最多 50 条、最多 4000 字符、按更新时间倒序**。记忆不是越多越好——它每轮都占上下文，必须设上限。

### 工具接口

| 工具 | 作用 |
| --- | --- |
| `memory_remember` | 新增/更新一条（写操作，需确认） |
| `memory_recall` | 按关键词搜索 |
| `memory_list` | 列出（可按分类） |
| `memory_forget` | 删除（写操作，需确认） |

> ⚠️ **踩坑**：记忆最怕"模型自作主张"。One Cedric 的 `memory_remember` 是写工具，必须用户确认；`dream` 生成的记忆候选**默认不自动保存**（`save_memories=False`），要用户显式开启。**记忆是持久化副作用，不能由模型单方面决定。**

## 11.3 梦境：让 Agent 在后台"反刍"

`dream.py` 的比喻很准确：把最近的会话当作"白天经历"，让模型在后台做一次结构化回顾。

### 收集上下文

```python
def collect_recent_context(copilot, days=3, max_sessions=20,
                           max_chars=30000) -> tuple:
    cutoff = time.time() - days * 86400
    filtered = [s for s in all_sessions if s.get("updated_at", 0) >= cutoff]
    filtered.sort(key=lambda x: x.get("updated_at", 0), reverse=True)
    filtered = filtered[:max_sessions]
```

只取最近 **3 天、最多 20 个会话、最多 3 万字符**。会话内容按角色截断：用户 300 字、助手 400 字、工具只计数不展开。

### 固定四段输出

```python
DREAM_PROMPT = """你正在做「梦境回顾」。分析下面最近的会话摘要，输出结构化洞察。

要求：
1. 用简体中文
2. 严格按以下格式，不要额外解释：

## 回想
（1-2 句总结最近在做什么，最高频的主题）

## 洞察
- 观察 1
- 观察 2
（3-6 条，关于用户的工作模式、习惯、未完成事项）

## 建议记忆
- [preference] key: value
- [project] key: value
（0-5 条，只有确实值得长期记住的才写）

## 明日建议
- 建议 1
- 建议 2
（2-4 条可执行的小建议）

只输出上述四部分。"""
```

为什么要固定格式？因为**输出要被程序解析**。`_extract_memories` 只认 `## 建议记忆` 段里的 `- [category] key: value`：

```python
if s.startswith("## 建议记忆"):
    in_section = True
    continue
if in_section and s.startswith("## "):
    break
if not in_section or not s.startswith("-"):
    continue
body = s.lstrip("- ").strip()
if not body.startswith("["):
    continue
...
```

**自然语言输出 + 固定小标题 + 可解析的行格式**，是"让模型产出结构化数据"的最稳做法——比要求纯 JSON 更宽容（模型可以自由写洞察），又比纯自由文本更可用（记忆候选能被提取）。

### 产物

`~/.one-cedric/dreams/dream_<timestamp>_<hex>.md`：

```markdown
# 🌙 梦境 · 2026-10-01 19:56:52

- 会话数: 6
- 轮数: 42
- 工具调用: 137
- 时长: 8.4s
- focus: （无）

---

## 回想
...
```

工具：`dream_run`（生成）、`dream_list`、`dream_read`、`dream_stats`。定时梦境则是"cron + dream_run"的组合。

> 💡 **设计笔记**：梦境的本质是"**离线反思**"。在线时模型被当前任务牵着走，没空总结；离线后给它一段完整历史，让它找模式。这个模式可以推广到任何 agent：日志分析、失败复盘、偏好提取，都是"dream"的变体。

## 11.4 定时任务：一个 30 秒轮询的守护线程

`cron.py` 的存储与梦境类似：`~/.one-cedric/cron.toml` + `cron_log/<job_id>/<timestamp>.log`。

### 五段式 cron 解析

```python
def _match_field(field: str, value: int) -> bool:
    if field == "*":
        return True
    if field.startswith("*/"):
        try:
            step = int(field[2:])
        except ValueError:
            return False
        return step > 0 and value % step == 0
    ...
```

支持 `*`、`*/n`、`a,b,c`、`a-b`——覆盖绝大多数日常需求（每分钟、每 5 分钟、每天 9 点、工作日 10 点）。不追求完整 cron 语法（`L`、`W`、`#` 这些），因为**用户要的是"每天备份一次"，不是"每月最后一个星期五"**。

### 守护线程

```python
"""定时任务：让 agent 定期自动执行 prompt。

存储: ~/.one-cedric/cron.toml
日志: ~/.one-cedric/cron_log/<job_id>/<timestamp>.log
调度: CronDaemon 后台线程，每 30s 检查一次
"""
```

每 30 秒扫一遍所有 job，比较 `next_run` 与当前时间；到点就起一个线程跑 `copilot.ask_no_ui(prompt)`，结果写日志、更新 `last_status` / `last_error` / `next_run`。

三个工程细节：

1. **30 秒粒度**：够用，且不会因为系统休眠错过的秒级任务而疯狂补跑；
2. **每个 job 独立日志**：出问题能定位是哪个任务、哪一次；
3. **`last_error` 落盘**：WebUI 的定时任务页可以直接展示"上次失败原因"。

工具：`cron_add` / `cron_remove` / `cron_enable` / `cron_list` / `cron_logs` / `cron_next`。

## 11.5 子 Agent：把"探索"扔进另一个上下文

`tools/subagent.py` 的文档字符串把设计要点写得很清楚：

```python
"""Sub agent：在主 agent 派生的独立上下文里跑只读探索任务。

设计要点：
- 子 agent 只持有只读工具（不写文件、不发邮件、不改属性）
- 独立 messages，不污染主上下文
- 独立 console，输出捕获后只返回摘要
- 不能再 spawn 子 agent（_depth 防护在 core 层）
- 支持 3 种模式：explore / code / plan
"""
```

### 三种模式

| 模式 | 用途 | 输出 |
| --- | --- | --- |
| `explore` | 快速探索代码库/资料 | `## 发现` + `## 未确认` |
| `code` | 分析一段代码、找问题 | `## 摘要` + `## 问题` + `## 建议改法` |
| `plan` | 制定可执行计划 | `## 计划`（有序列表） |

上限：`max_steps` 默认 6（最大 20），`timeout` 默认 300 秒（最大 1800）。

### 为什么子 Agent 必须只读

1. **权限不可传递**：主 agent 的确认流程在子 agent 里没有 UI，写操作会变成"静默执行"；
2. **上下文隔离**：子 agent 读了 50 个文件，只返回 20 行摘要——这正是它存在的意义；
3. **失败可丢弃**：子 agent 跑偏了，主 agent 可以直接忽略它的输出，不用收拾副作用。

### `multi_review`：三个子 Agent 并行审查

`multi_review` 固定派 3 个 sub-agent，各看一个维度：

```python
DEFAULT_DIMENSIONS = [
    {"key": "security", "name": "安全",
     "focus": "认证/授权绕过、SQL/命令注入、密钥泄露、XSS/CSRF、路径穿越……"},
    {"key": "performance", "name": "性能",
     "focus": "N+1 查询、循环内 IO、无界循环、内存泄漏、同步阻塞……"},
    {"key": "readability", "name": "可读性",
     "focus": "命名是否达意、函数/类复杂度、魔法数字、死代码、重复逻辑……"},
]
```

实现用 `ThreadPoolExecutor` + `as_completed`，每个维度独立超时（默认 180 秒），结果汇总成一份报告。

> 💡 **设计笔记**：多 Agent 的价值不在"更多智能"，而在"**不同 prompt 强制不同视角**"。一个模型同时想安全、性能、可读性，往往会漏；三次独立调用，每次都只说一个维度，覆盖率明显更高。

## 11.6 @引用：把文件"贴"进对话

`at_ref.py` 支持四种形式：

```text
@README.md                整个文件
@src/main.py#L10-L20      指定行范围
@src/                     目录列表
@src/**                   递归目录列表
@F:\绝对路径.png          绝对路径（若在沙箱内）
```

解析流程：

1. **正则抓取** `@([^\s@]+)`；
2. **剥掉尾部标点**（`，。；：！？)` 等）——中文输入法经常把标点粘在路径后面；
3. **解析 `#L10-L20`** 行范围；
4. **`_resolve_path` 过沙箱**；
5. **读文件/列目录**，包成 Markdown 代码块。

```python
raw = path_str.strip().rstrip(",.，。；;:：!！?？)）]】")
...
p, err = _resolve_path(root, raw)
if err:
    return None, f"{raw}: {err}"
```

`@` 引用的产物是一条 user 消息里的附件块：

```markdown
### src/main.py#L10-L20
```
（文件内容）
```
```

这样模型看到的是"带路径标签的代码块"，既能理解内容，也知道出处。

> ⚠️ **踩坑**：09-27 16:36 用户报了一个错：`'tuple' object has no attribute 'encode'`。原因是 `build_attachment` 早期版本返回了 `(block, warning)` 元组，调用方却把它当字符串直接 `encode()`。修复是**统一返回值契约**：要么 `(block, None)`，要么 `(None, warning)`，调用方必须先解包。**多返回值函数最容易在调用点被误用**，类型标注（`-> tuple`）救不了，只有测试能。

## 11.7 MCP：用 JSON-RPC 接第三方工具

MCP（Model Context Protocol）是目前最通用的"工具生态协议"。One Cedric 的实现是 **stdio 传输 + JSON-RPC 2.0**。

### 握手

```python
self._request("initialize", {
    "protocolVersion": "2025-03-26",
    "capabilities": {},
    "clientInfo": {"name": "One Cedric", "version": "0.17.0"},
})
self._notify("notifications/initialized")
```

### 读线程 + 响应队列

MCP 服务器会**主动**发消息（通知、日志），不能"发一个请求就同步读一行"。所以：

```python
def _read_messages(self) -> None:
    for line in self.proc.stdout:
        line = line.strip()
        if not line:
            continue
        try:
            self._responses.put(json.loads(line))
        except json.JSONDecodeError:
            self._responses.put({"_invalid_message": line[:500]})
    self._responses.put({"_closed": True})
```

一个后台线程持续读 stdout，把每条 JSON 塞进 `queue.Queue`；请求方按 `id` 匹配响应。stderr 也单独起线程读，保留最近 20 行——MCP 服务器崩了，错误信息通常在 stderr 里。

### 工具命名与写判定

外部工具会合并进 `TOOLS`，名字冲突时**外部优先**（`_dispatch.dispatch_tool` 先查 `is_external_tool`）。写判定由配置里的 `mutating` 标志决定（`is_mutating_tool`），而不是靠名字猜。

> ⚠️ **踩坑**：审计报告 SEC-13 指出，MCP 子进程默认继承完整环境变量——第三方 MCP 能拿到 `ONE_CEDRIC_API_KEY`。修复方向是**启动子进程时清洗 env**，只传必要的变量。接生态的第一天就要想清楚"我给了它什么权限"。

## 11.8 Hooks 与自定义工具：最小的扩展点

### Hooks

配置在 `~/.one-cedric/integrations.json`：

```json
{
  "hooks": [
    {
      "name": "notify-on-write",
      "events": ["after_tool"],
      "command": ["python", "D:\\scripts\\notify.py"],
      "args": [],
      "enabled": true
    }
  ]
}
```

执行方式（`integrations.py:537`）：

```python
message = json.dumps({"event": event, **payload}, ensure_ascii=False)
result = subprocess.run(
    [*command, *args],
    input=message,
    capture_output=True, text=True,
    encoding="utf-8", errors="replace",
    timeout=15,
)
if result.returncode:
    raise RuntimeError(f"Hook '{hook.get('name', '?')}' 返回 ...")
```

**事件 JSON 从 stdin 进，退出码非 0 视为失败**，超时 15 秒。事件目前有 `after_tool`（工具执行后）等，payload 包含工具名、摘要、状态。

### 自定义工具

命令式工具（command-backed）：

```json
{
  "custom_tools": {
    "deploy_staging": {
      "description": "部署到 staging 环境",
      "parameters": {"type": "object", "properties": {...}},
      "command": ["python", "deploy.py"],
      "args": ["--env", "staging"],
      "write": true,
      "enabled": true
    }
  }
}
```

`write: true` 的工具会进写流程（确认后才执行）。名字必须匹配 `^[a-zA-Z][a-zA-Z0-9_-]{0,47}$`——防止用奇怪的名字覆盖内置工具。

## 11.9 第三方技能：只读的 `~/.agents/skills`

`agents_skills.py` 只做两件事：**发现**与**读取**。

```python
AGENTS_HOME = Path.home() / ".agents"
SKILLS_DIR = AGENTS_HOME / "skills"
```

扫描 `~/.agents/skills/*/SKILL.md`，解析 YAML frontmatter 的 `name` / `description` / `version`：

```python
_FRONTMETER_RE = re.compile(r"^---\s*\n(.*?)\n---\s*\n?", re.DOTALL)
```

模型通过 `agents_skill` 工具使用：

- 不带 `name`：列出所有技能（名字 + 描述）；
- 带 `name`：返回该技能 `SKILL.md` 全文，模型按其中的指引执行。

**只读**是刻意的：技能文件来自其它 agent 运行时，可能被篡改；One Cedric 不写、不执行技能里的代码，只把它当"说明书"读给模型听。

## 11.10 LSP：让 Agent 看懂代码结构

LSP 模块（`lsp/`）分四层：

```text
config.py    默认服务器配置（pylsp / typescript-language-server / gopls / rust-analyzer）
manager.py   按扩展名选服务器、惰性启动、生命周期管理
client.py    JSON-RPC + Content-Length 帧、initialize、请求/通知
tools.py     暴露成 6 个工具
```

默认配置（`lsp/config.py:20`）：

```python
"python": {
    "command": "pylsp", "args": [],
    "extensions": [".py", ".pyi"],
    "root_markers": ["pyproject.toml", "setup.py", "setup.cfg", ".git"],
    "init_timeout": 30, "request_timeout": 15,
},
```

六个工具：

| 工具 | 作用 |
| --- | --- |
| `lsp_hover` | 悬停信息（类型、文档） |
| `lsp_definition` | 跳转到定义 |
| `lsp_references` | 查找引用 |
| `lsp_diagnostics` | 语法/类型诊断 |
| `lsp_symbols` | 文件符号列表 |
| `lsp_status` | 服务器状态与错误 |

管理器是**单例 + 惰性启动**：

```python
def get_manager(root: Path) -> LSPManager:
    global _MANAGER_INSTANCE
    with _MANAGER_LOCK:
        if _MANAGER_INSTANCE is None:
            _MANAGER_INSTANCE = LSPManager(root)
        elif _MANAGER_INSTANCE.root != root.resolve():
            _MANAGER_INSTANCE.shutdown_all()
            _MANAGER_INSTANCE = LSPManager(root)
        return _MANAGER_INSTANCE
```

切换工作目录时关掉旧服务器，避免进程泄漏。`pick_client` 按扩展名找候选服务器，全部启动失败才报错，并列出每个服务器的错误。

> 💡 **设计笔记**：LSP 是"**按需启动的重型依赖**"。用户可能没装 `pylsp`，但只要不调用 `lsp_*`，程序就不该受影响。惰性启动 + 单个服务器失败不影响其他语言，是这个模块最重要的健壮性设计。

## 11.11 Computer Use：给鼠标键盘上锁

`computer/` 模块让 agent 能操作桌面：`screen_capture`、`mouse_action`、`keyboard_action`、`window_action`。风险极高，所以默认**关闭**，并且有一整套 `safety.py`：

```python
MAX_ACTIONS_PER_MINUTE = 60
MAX_TYPE_CHARS = 500
MAX_DRAG_PIXELS = 1500
MAX_CLICKS = 3
MAX_SCROLL = 20
```

检查逻辑：

- **未启用** → 直接拒绝，并提示如何开启（`--enable-computer-use` 或 `/computer on`）；
- **频率**：60 秒内最多 60 个动作，超出拒绝；
- **拖拽**：超过 1500 像素 → 额外确认；
- **点击**：超过 3 次 → 额外确认；
- **滚动**：超过 20 格 → 额外确认；
- **输入**：超过 500 字符 → 额外确认；
- **快捷键黑名单**：`Ctrl+Alt+Delete`、`Ctrl+Shift+Esc`、`Alt+F4`、`Win+L`、`Cmd+Q` 直接拒绝。

```python
forbidden = {
    ("ctrl", "alt", "delete"),
    ("ctrl", "shift", "escape"),
    ("alt", "f4"),
    ("win", "l"),
    ("cmd", "q"),
}
combo = tuple(sorted(str(k).lower() for k in keys))
if combo in forbidden:
    return SafetyCheck(ok=False, reason=f"快捷键 {keys} 被禁用（涉及系统级操作）。")
```

三个设计原则：

1. **默认关闭**：不是"默认开启 + 提示风险"，而是"你不开就用不了"；
2. **限速 + 限量**：防止模型陷入循环，把用户桌面点成筛子；
3. **系统级快捷键硬禁**：这些组合一旦误触，可能锁屏、注销、关闭程序，不能靠确认框兜底。

## 11.12 生态扩展的通用模式

把这一章的七个模块放在一起看，会发现它们是同一个模式的变体：

| 模块 | 存储/来源 | 进入模型的方式 | 安全边界 |
| --- | --- | --- | --- |
| memory | `memory.toml` | system prompt 注入 | 写操作需确认 |
| dream | `dreams/*.md` | 工具读取 | 记忆候选默认不保存 |
| cron | `cron.toml` | 定时触发 prompt | 独立日志 + 错误落盘 |
| subagent | 独立上下文 | 返回摘要 | 只读工具 + 深度防护 |
| MCP | 子进程 JSON-RPC | 动态 schema | 写判定 + env 清洗 |
| hooks | `integrations.json` | 事件 JSON | 15s 超时 + 退出码 |
| skills | `~/.agents/skills` | 只读文档 | 不执行、不写入 |

共同点：

1. **配置与代码分离**：能放 TOML/JSON 的，不放 Python；
2. **默认保守**：默认关闭、默认只读、默认要确认；
3. **失败隔离**：一个 MCP 服务器崩了、一个 hook 超时了，不能影响主对话；
4. **可观测**：日志、状态、`last_error`，出问题能查。

> 💡 **设计笔记**：Agent 的"生态"不是工具数量的堆砌，而是**一组稳定的扩展契约**。只要契约清楚（schema 怎么声明、写操作怎么标记、失败怎么返回），第三方就能安全地长进来；契约不清楚，每加一个集成就多一处特判。

## 🔧 自己动手

1. **实现一个最小记忆**：TOML 存储 + `add/remove/search` + `to_prompt_block`，限制 20 条 / 2000 字符；
2. **写一个"梦境"**：读取最近 7 天的聊天记录，让模型输出"洞察 / 建议记忆 / 明日建议"三段，保存为 Markdown；
3. **实现 cron 解析**：支持 `*`、`*/5`、`9-18`、`1,15`，写一个每 10 秒轮询的守护线程；
4. **做一个只读子 Agent**：独立 messages + 只读工具 + 步数上限，输入"找出这个项目的入口文件"，输出摘要；
5. **接一个 MCP 服务器**：用 stdio 启动，完成 `initialize` 握手，列出工具并调用一个只读工具；
6. **给 computer use 加一条规则**：禁止在 1 秒内连续点击同一坐标超过 2 次，防止"死循环点击"。

---

[← 上一章：10 网关与 WebUI](10-gateway-webui) ｜ [返回目录](Home) ｜ [下一章：12 测试、审计与发布 →](12-testing-release)
