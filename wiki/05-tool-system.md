# 05 · 工具系统：从 4 个函数到 250 个工具

## 5.1 "现在有什么 tools？"——"都要加"

📅 **开发日志 · 2026-09-18**（对话 #67）

> **22:54 用户**：现在有什么 tools
>
> **22:54 用户**：都要加
>
> **22:56 用户**：加别的工具
>
> **23:02 用户**：继续加工具

这是整个项目里**最高频的一句需求**。此后每隔几天就会出现一次：

> 📅 **09-25 21:57 用户**：再加 10 个 tools
>
> 📅 **10-01 13:16 用户**：再加足够多的 tools
>
> 📅 **10-01 13:22 用户**：增加 wttr.in tools，并在使用后打印所请求 wttr.in 的内容
>
> 📅 **10-01 14:42 用户**：压缩工具要增加更多文件格式，支持 ansi 扫描，增加更多的调节系统功能的 tools
>
> 📅 **10-01 14:45 用户**：再加一些 tools
>
> 📅 **10-01 17:02 用户**：我们先做几个其它 tools，python 代码编写，告诉 AI 可以使用这个 tools 来编写代码，图表，分析数据等等。二维码工具要变得更加更加高级

工具数量一路涨：

| 版本 | 时间 | 工具数 | 说明 |
| --- | --- | --- | --- |
| #57 最小 Agent | 09-13 | **4** | `read_file` / `list_files` / `search_in_files` / `file_info`，全部只读 |
| v1（F:\one-cedric） | 09-19+ | **49** | 加了写文件、shell、git、docker、LSP、computer… |
| v2（F:\one-cedric2） | 10-01/02 交付后 | **242** | 邮件、高德、办公、系统操作、编排…… |
| v3（F:\one-cedric3） | 发布版 | **250** | +浏览器、Compose/K8s 只读等 |

> 以上数字来自 `one_cedric/tools/schema.py` 的实测统计（`len(TOOLS)`），不是估算。

问题来了：**40 个工具能写在一个文件里，250 个怎么管？** 这一章讲 One Cedric 的答案——三层结构、一张注册表、两条契约。

## 5.2 总览：schema → handler → dispatch 三层

```text
┌───────────────────────────────────────────────────────────┐
│ 第一层  schema.py    "工具是什么"（纯数据，给模型看）        │
│   TOOLS / TOOLS_READONLY            → 发给 API 的 JSON      │
│   READONLY_TOOL_NAMES / WRITE_TOOLS → 分类与权限的依据      │
│   CACHEABLE_TOOLS                   → 哪些结果可缓存        │
├───────────────────────────────────────────────────────────┤
│ 第二层  _handlers_*.py  "工具怎么做"（实现，给人改）        │
│   def _weather(root, args) -> str   → 171 个 handler 函数   │
│   每个模块导出 HANDLERS: {名字: 函数}                       │
├───────────────────────────────────────────────────────────┤
│ 第三层  _dispatch.py    "到哪里去找"（路由，机器跑）        │
│   dispatch_tool(name, args, root) -> str                    │
│   写工具不在这里 → core 层先确认再执行（第 7 章）            │
└───────────────────────────────────────────────────────────┘
```

三个数字对照（v3 实测）：

- `TOOLS` = **250** 个：其中只读 **160**、写 **90**（恰好 160 + 90 = 250）；
- `_dispatch` 注册表里 **171** 个 handler（48+27+20+22+54，分布在 5 个模块）；
- 剩下 **79** 个 = 全部是写工具，由 core 层接管（先确认后执行）。

## 5.3 第一层：schema 是纯数据

`tools/schema.py` 的文档字符串第一行就写了："**工具的 JSON schema（OpenAI function calling 格式）。纯数据。**" 这个约束很重要：这个文件三万多行、上百万字符，但**没有一行逻辑**——它只声明。

一个典型条目（`schema.py:16`）：

```python
{"type": "function", "function": {
    "name": "read_file",
    "description": (
        "读取本地文本文件，带行号。\n"
        "自动检测编码（UTF-8 / GBK / Big5 / Shift-JIS 等）。\n"
        "可用 start_line/end_line 读取指定行范围。"
    ),
    "parameters": {"type": "object", "properties": {
        "path": {"type": "string", "description": "文件路径。"},
        "start_line": {"type": "integer", "description": "起始行号，从 1 开始。"},
        "end_line": {"type": "integer", "description": "结束行号（含）。"},
        "encoding": {"type": "string",
                     "description": "强制指定编码（如 gbk），留空则自动检测。"},
    }, "required": ["path"]},
}},
```

注意两个细节：

1. **枚举值要写全**。比如 `mouse_action` 的 `action` 字段直接给出 `enum: ["move", "click", "right_click", "double_click", "drag", "scroll"]`——模型对 enum 的遵守度远高于自然语言描述；
2. **默认值可以引用常量**。`glob` 的描述用了 f-string：

```python
"max_results": {"type": "integer",
                "description": f"最多条数，默认 {MAX_GLOB_RESULTS}。"},
```

`MAX_GLOB_RESULTS` 来自 `config.py`。默认值写在两处（schema 描述 + handler 实现）容易不同步，所以**能引用常量就引用常量**。

> 💡 **设计笔记**：schema 是"发送给模型的说明书"。它的读者不是程序员，是模型。所以描述要写"什么时候用我、参数是什么、限制是什么"，而不是"这个函数内部怎么实现"。One Cedric 里 `web_search` 的描述甚至带使用建议："适合快速扫一遍有什么。需要内容时用 web_research。"——这类**路由提示**比技术描述更能提升工具选择准确率。

## 5.4 工具分类：三个集合，一个曾修过的 bug

schema.py 的文件末尾定义三个集合（行号约 2505–2698）：

```python
READONLY_TOOL_NAMES = { ... }        # 160 个
CACHEABLE_TOOLS = { ... }            # 42 个
WRITE_TOOLS = { ... }                # 90 个

TOOLS_READONLY = [
    t for t in TOOLS
    if t["function"]["name"] in READONLY_TOOL_NAMES
]
```

分类的用途：

| 集合 | 谁在用 | 用途 |
| --- | --- | --- |
| `READONLY_TOOL_NAMES` | 计划模式 / 子 Agent | 只给"不会改动东西"的工具 |
| `WRITE_TOOLS` | core 层 | 必须走确认流程的工具 |
| `CACHEABLE_TOOLS` | core 层 `ToolCache` | 同参数重复调用直接读缓存 |
| `TOOLS_READONLY` | 子 Agent / 受限会话 | 直接可发给 API 的 schema 子集 |

**这里修过一个真实的分类 bug**。有些工具同时有只读和写两种 op——比如 `image_process`（`op=info` 只读、`op=resize` 写文件）、`pdf_metadata`（get 只读、set 写）、`sysop`（查询类只读、设置类写）。这些名字被同时放进了两个集合。修复方式是**让写集合永远获胜**（`schema.py:2684`）：

```python
# Tools listed in both sets on purpose: they expose a read-only "preview"/"get"
# op *and* a writing "apply"/"set" op (image_process, pdf_metadata, sysop).
# WRITE_TOOLS must always win, otherwise plan mode and sub-agents would be
# allowed to run a tool that can modify the user's files.
AMBIGUOUS_READONLY_WRITE_TOOLS = READONLY_TOOL_NAMES & WRITE_TOOLS

# Invariant enforced from here on: a tool name is never both write and
# read-only. Everything downstream (plan mode, sub-agent tools, cacheability)
# consumes this corrected set.
READONLY_TOOL_NAMES = READONLY_TOOL_NAMES - WRITE_TOOLS
```

> ⚠️ **踩坑**：这是一个**安全相关**的 bug，而且是靠安全审计（第 7 章讲的 `SECURITY_AUDIT.md`）找出来的——不是靠报错，因为它根本不报错。分类"多算"的后果是：计划模式和子 Agent 能调用会改用户文件的工具。教训：**当同一个名字出现在两个互斥集合里，默认必须是"更严格"的那个**。宁可让只读会话少一个工具，也不能多一个。

## 5.5 第二层：handler 长什么样

handler 模块共 5 个，按领域分工：

| 模块 | handler 数 | 内容 |
| --- | --- | --- |
| `_handlers_readonly.py` | 48 | 文件读 / 网络 / 数据 / 系统 |
| `_handlers_text.py` | 27 | 文本 / 编码 / 转换 / AST |
| `_handlers_media.py` | 20 | 图片 / 视频 / 音频 / PDF / 二维码 |
| `_handlers_agent.py` | 22 | 记忆 / 定时 / 成本 / 通知 / 技能 |
| `_handlers_extra.py` | 54 | 应用 / 天气 / 高德 / 邮件 / 微信 / Compose / K8s |

每个 handler 的签名是统一的 **`(root: Path, args: dict) -> str`**。文件开头有三个共用小工具（`_handlers_readonly.py:10`）：

```python
def _imp(module_name: str):
    """导入同级模块。失败抛 ImportError。"""
    import importlib
    return importlib.import_module(f".{module_name}", package=__package__)

def _require(args: dict, *keys: str) -> str:
    for k in keys:
        if k not in args or args[k] in ("", None):
            return f"ERROR: 缺少必需参数 {k}。"
    return ""

def _err_wrap(fn, *a, **kw):
    try:
        return fn(*a, **kw)
    except Exception as exc:
        return f"ERROR: {type(exc).__name__}: {exc}"
```

三个设计点，逐个说：

### 惰性导入（`_imp`）

250 个工具背后是 60 多个实现模块（`email_tools.py` 一个文件就 46KB），其中大量依赖第三方库（`python-docx`、`openpyxl`、`av`、`pytesseract`……）。如果 handler 模块顶部全部 `import`，那么**少装一个库，整个程序都起不来**。

`_imp` 把导入推迟到工具真正被调用的那一刻：没装 `openpyxl`？`excel` 工具报 `ImportError`，其他 249 个照常工作。这是"工具多"场景下最实用的容错模式。

### 必备参数校验（`_require`）

模型可能忘传参数。handler 自己先检查一遍，返回**人类和模型都能读懂的错误**：

```python
def _weather(root: Path, args: dict) -> str:
    err = _require(args, "location")
    if err:
        return err
    return _imp("weather").get_weather(
        location=args["location"],
        format=args.get("format", "full"),
        lang=args.get("lang", ""))
```

注意这是**双重防线**：agent 主循环里还会根据 schema 的 `required` 检查一次（第 8 章），handler 里再查一次。为什么？因为 handler 可能被别的路径调用（子 Agent、WebUI、测试），不能假设调用方一定守规矩。

### 错误即返回值（`_err_wrap` 与"ERROR: "前缀）

所有 handler **永远返回字符串，绝不把异常抛给上层**。错误统一格式：

```text
ERROR: 缺少必需参数 location。
ERROR: 请求超时
URL: https://wttr.in/Beijing?format=3
ERROR: 拒绝访问工作目录之外的路径: C:\Windows\system.ini（工作目录: ...）
```

好处：错误文本会作为 tool 消息回给模型，模型读到 `ERROR: ...` 就能自我纠正（补参数、换路径、换工具）。"拒绝访问……（工作目录: ...）"这种错误还带了**行动建议**（提示可以用 `/mode allow` 放开），模型会把建议转述给用户。

## 5.6 案例：一个工具的全链路（weather）

以 `weather` 为例，从 schema 到输出共 4 步：

**第 1 步：schema 声明**（`schema.py:1978`）

```python
{"type": "function", "function": {
    "name": "weather",
    "description": (
        "查询全球天气（wttr.in）。\n"
        "默认返回原生 ASCII 艺术天气图，包含详细天气。"
        "format='full' 或 '' 用原生大图（推荐）；"
        "'3' 一行简版；'j1' JSON。"
        "中文用 lang=zh。"
    ),
    "parameters": {"type": "object", "properties": {
        "location": {"type": "string"},
        "format": {"type": "string"},
        "lang": {"type": "string"},
    }, "required": ["location"]},
}},
```

**第 2 步：handler 接住**（`_handlers_extra.py:73`）

```python
def _weather(root: Path, args: dict) -> str:
    err = _require(args, "location")
    if err:
        return err
    return _imp("weather").get_weather(
        location=args["location"],
        format=args.get("format", "full"),
        lang=args.get("lang", ""))
```

**第 3 步：注册进表**（`_handlers_extra.py:426`）

```python
HANDLERS = {
    **_BROWSER_HANDLERS,
    # 版本信息
    "github_latest_version": _github_latest_version,
    # 通知
    "notify_check":         _notify_check,
    ...
    # 天气 / 定位
    "weather":              _weather,
    "locate_rough":         _locate_rough,
    ...
}
```

**第 4 步：实现**（`weather.py`，71 行）。这个实现里有一个只有"真去调了 API"才会发现的细节——**User-Agent 决定返回格式**：

```python
USER_AGENT = "curl/8.0.0"        # wttr.in 对 curl UA 返回纯文本
BASE = "https://wttr.in"
```

wttr.in 会按 User-Agent 猜客户端：浏览器 UA 返回 HTML 页面，curl UA 返回纯文本。作者在开发中还遇到过一次"返回了 HTML（可能限流）"的情况，于是加了一道防御：

```python
low = text[:200].lower()
if low.startswith("<!doctype") or low.startswith("<html"):
    return (
        f"ERROR: wttr.in 返回了 HTML（可能限流，稍后再试）\n"
        f"URL: {url}\n---\n{text[:500]}"
    )
```

返回值的格式也值得一提——**"URL + 分隔线 + 内容"**：

```python
return f"URL: {url}\n---\n{text}"
```

这是全项目的惯例：工具返回里带上 URL/路径/参数回显，出问题时用户和模型都能自己看到"到底请求了什么"。

## 5.7 第三层：`_dispatch.py` 注册表

`_dispatch.py` 全文只有 83 行，是整个工具系统的枢纽。核心是"合并五张表"：

```python
_HANDLERS: dict[str, Callable[[Path, dict], str]] = {}

def _merge(module_name: str) -> None:
    try:
        mod = importlib.import_module(f".{module_name}", package=__package__)
    except ImportError as exc:
        # 未实现不致命：只影响该模块的工具
        print(f"[tools] 跳过 {module_name}（{exc}）")
        return
    h = getattr(mod, "HANDLERS", None)
    if not isinstance(h, dict):
        return
    for k, v in h.items():
        _HANDLERS[k] = v

_merge("_handlers_readonly")
_merge("_handlers_text")
_merge("_handlers_media")
_merge("_handlers_agent")
_merge("_handlers_extra")
```

注意 `except ImportError` 的处理：**一个 handler 模块导入失败，只是跳过它**（打印一行提示），而不是让整个程序崩溃。这个设计来自惨痛经历——某个版本里 `_handlers_readonly` 引用了还不存在的模块，导致所有工具全部不可用。工具系统的可用性应该是"尽量多"，而不是"全有或全无"。

分发的入口是 `dispatch_tool`：

```python
def dispatch_tool(name: str, args: dict, root: Path) -> str:
    """执行工具。写操作不经过这里（core 层先确认）。"""
    if not isinstance(args, dict):
        args = {}

    from ..integrations import call_external_tool, is_external_tool
    if is_external_tool(name):
        return call_external_tool(name, args, root)

    fn = _HANDLERS.get(name)
    if fn is not None:
        try:
            return fn(root, args)
        except TypeError as exc:
            return f"ERROR: 工具 '{name}' 参数不匹配: {exc}"
        except Exception as exc:
            return (f"ERROR: 工具 '{name}' 执行失败: "
                    f"{type(exc).__name__}: {exc}")

    if is_core_handled(name):
        return (f"ERROR: 工具 '{name}' 是写操作或需要 core state，"
                f"不应直接走 dispatch_tool。")

    return f"ERROR: 未知工具 '{name}'（未在 dispatch 表里注册）。"
```

四件事按顺序发生：

1. **外部工具优先**：MCP 服务器或用户自定义工具（`integrations`，第 11 章）可以覆盖同名工具；
2. **查表执行**：GIL 之下的 `try/except` 兜底——`TypeError` 单独归类（多半是参数问题），其他异常归为"执行失败"；
3. **核心写工具**：不在这里执行，返回明确错误提示调用方"流程错误"——这条防护抓过真实 bug（有代码路径忘了先确认就调 `bash`）；
4. **未知工具**：`ERROR: 未知工具`。

配套的 `is_core_handled` 定义"哪些工具不归我管"：

```python
def is_core_handled(name: str) -> bool:
    """工具是否由 core 层处理（写操作 / 需要确认 / 需要 state）。"""
    if name in WRITE_TOOLS:
        return True
    # 少部分"只读但需要 core state"的工具
    if name in ("spawn_agent", "multi_review", "ask_user",
                "dream_run", "cost_export"):
        return True
    return False
```

> 💡 **设计笔记**：为什么 `ask_user` 这种"只是问用户一句"的工具要归 core？因为它需要**访问会话的输入输出状态**——它要暂停 agent 循环、渲染交互式选项、把答案写回消息流。判断标准不是"只读还是写"，而是"**需不需要 core 的 state**"。第 8 章会看到 `agent_turn` 如何利用这两个函数把 250 个工具分成两股流。

## 5.8 写工具的"两次调用"约定

写工具不进 `_dispatch`，那它们长什么样？答案是 **preview / apply 双函数**。以 PDF 水印为例（`tools/pdf_advanced.py`）：

```python
def pdf_watermark(...) -> str:        # preview：只计算，不动文件
    """生成水印操作的预览（不落盘）。"""

def pdf_watermark_apply(...) -> str:  # apply：真正写文件
    """按预览参数执行水印写入。"""
```

约定：

- `xxx`（无后缀）= 预览：解析参数、检查文件存在、算出"将要发生什么"，**绝不产生副作用**；
- `xxx_apply` = 执行：真正落盘；
- schema 里**只注册 `xxx`**，模型不知道 `apply` 的存在；
- core 层收到 `xxx` 调用后：先调预览函数拿到描述 → 展示给用户确认（第 7 章的 5 选项菜单）→ 用户同意后调 `_apply`。

这样做的收益：确认界面显示的"将要改什么"不是模型说的，而是**代码算出来的**——模型说"我会删除 3 个文件"，预览函数实际数出来是 17 个，用户在确认界面上看到的就是 17。**永远不要让模型自己描述它将要造成的副作用。**

这套约定也是"写工具巨多"的原因：90 个写工具，背后是 45 组左右的 preview/apply 对，`tools/__init__.py` 再把它们 re-export 给 core 使用（第 7 章详述确认流程）。

## 5.9 模型给什么就执行什么？不——参数归一化

本地模型和廉价 API 模型有个共同点：**它们不太遵守工具协议**。真实观察到的问题：

- 工具名写成 `readfile`、`ls`、`cat`、`shell`（训练数据里见得多）；
- `arguments` 是坏 JSON：带 ```` ```json ```` 围栏、单引号、尾逗号、缺右括号；
- 参数名写成 `file` / `filepath` / `target`（schema 里叫 `path`）；
- 类型错：`"start_line": "5"`（字符串）、`"recursive": "yes"`。

`tools/normalize.py`（285 行）专门消化这些。核心是五个函数，串成一条流水线（`prepare_tool_call`）：

```python
def prepare_tool_call(tc: dict) -> tuple:
    fn = tc.get("function") or {}
    raw_name = fn.get("name", "")
    name = normalize_tool_name(raw_name)          # ① 工具名归一化
    raw_args_str = fn.get("arguments", "{}")
    if isinstance(raw_args_str, dict):
        args = raw_args_str
    else:
        args = repair_json(str(raw_args_str))     # ② JSON 修复
    args = normalize_args(name, args)             # ③ 参数别名 + 类型矫正
    err = ""
    if not name:
        err = "无法识别工具名。"
    return name, args, err
```

**① 工具名归一化**：一张 70 多行的别名表（`TOOL_ALIASES`）：

```python
TOOL_ALIASES: dict = {
    "readfile": "read_file", "read": "read_file", "cat": "read_file",
    "ls": "list_files", "listdir": "list_files", "dir": "list_files",
    "grep": "search_in_files", "search": "search_in_files",
    "edit": "edit_file", "replace": "edit_file", "patch_file": "edit_file",
    "write": "write_file", "create_file": "write_file",
    "shell": "bash", "run": "bash", "exec": "bash", "terminal": "bash",
    ...
}
```

注意 `normalize_tool_name` 的顺序：**先查真实工具名，再查别名表**。否则像 `list`（既是别名又可能是真名）就会误伤。

**② JSON 修复** `repair_json`：剥代码围栏 → 直接 `json.loads` → 单引号转双引号 → 去尾逗号 → **补齐括号**：

```python
for _ in range(5):
    opens = fixed.count("{") - fixed.count("}")
    brackets = fixed.count("[") - fixed.count("]")
    if opens <= 0 and brackets <= 0:
        break
    fixed += "]" * max(0, brackets) + "}" * max(0, opens)
    try:
        return json.loads(fixed)
    except json.JSONDecodeError:
        continue
```

最后还有一档兜底：正则扫 `"key": value` 逐个抠出来。**宁可少几个参数，也不要整次调用失败**——缺的参数会在主循环里触发"缺少必需参数"错误，让模型重试。

**③ 参数别名 + 类型矫正**：`COMMON_PARAM_ALIASES` 给每个标准参数名列出常见别名，`normalize_args` 按 schema 支持的参数集做映射；`coerce_types` 把 `"5"` 转成 `5`、`"true"/"yes"/"1"/"on"` 转成 `True`。

这套东西的价值用一句话概括：**归一化是"宽容输入、严格输出"**。工具的 250 份 schema 是严格契约，但入口处对模型的不规范输出尽量宽容。

> ⚠️ **踩坑**：别让归一化"聪明过头"。`normalize_args` 里有一条规则：参数名先转小写再查表（`key_low`），但对**不在别名表里的参数**保留原名。曾经有一个版本把全部参数名强转小写，结果 `Python_exec` 工具里大小写敏感的 `cwd` 变 `CWD` 直接报错。**归一化只该作用于"你知道语义"的字段**。

## 5.10 运行时怎么判定只读 / 写？`_classify_is_write`

除了静态集合，core 里还有一层**运行时判定**（`core.py:6880`）：

```python
def _classify_is_write(self, name: str, args: dict) -> bool:
    """判断工具调用是否写操作。"""
    from .integrations import is_external_tool, is_mutating_tool
    if is_external_tool(name):
        return is_mutating_tool(name)
    WRITE_NAMES = set(WRITE_TOOLS) | { ... }
    if name not in WRITE_NAMES:
        return False
    # 只读 op 例外
    if name == "image_process" and str(args.get("op", "")).lower() == "info":
        ...
```

两层分类的关系：**静态集合回答"这个工具能不能改东西"，运行时判定回答"这一次调用会不会改东西"**。`image_process` 传 `op=info` 时不需要确认，传 `op=resize` 时需要——只看工具名无法区分。外部工具（MCP）的语义由集成层声明（`is_mutating_tool`）。

这份 `WRITE_NAMES` 是 `WRITE_TOOLS` 的超集：里面还有 `sqlite`、`clipboard`、`todo`、`screen_capture`、`screen_info`、`ask_user`、`spawn_agent`、`multi_review` 这类"**字面上不写文件、但会改变会话/系统状态**"的工具。它们都要走确认或特殊流程。

> 💡 **设计笔记**：注意 `sqlite` 出现在写名单里——因为 SQL 可能是 `DELETE FROM`；`clipboard` 同理——读剪贴板是隐私（可能读到密码），写剪贴板会改变用户环境。**"是不是写操作"的判定标准是"会不会产生用户可见的副作用"，不是"会不会落盘"。**

## 5.11 新增一个工具的完整清单

从 4 涨到 250 的过程里，每个新工具都要过这份 checklist（后版本固化下来的）：

1. **schema 条目**：加进 `TOOLS`，写清 description（给模型的路由提示）、所有参数、`required`；
2. **实现函数**：写进对应领域模块（`weather.py`、`email_tools.py`…），签名返回 `str`，错误一律 `ERROR: ` 开头；
3. **handler 包装**：在对应的 `_handlers_*.py` 里写 `def _xxx(root, args)`，做 `_require` 校验、……装进 `HANDLERS` 表；
4. **分类**：只读进 `READONLY_TOOL_NAMES`，写进 `WRITE_TOOLS`（写工具的 preview/apply 对要 re-export 到 `tools/__init__.py`）；
5. **缓存决策**：纯函数、结果稳定的（hash、计算、查询）考虑进 `CACHEABLE_TOOLS`；
6. **动画**：在 `ui` 的 `TOOL_FRAMES` 里加一组帧（可选，默认帧兜底，第 4 章）；
7. **测试**：进 `TOOL_TEST_REPORT.md` 的冒烟清单（第 12 章），保证"工具名 → 不崩"；
8. **审计**：涉及文件/网络/系统的过一遍 `SECURITY_AUDIT.md` 的分类（第 7 章）。

第 12 章会讲这份清单怎么被 250 个工具的冒烟测试监督：一次测试跑出 `{"SKIP":111,"OK":85,"RUN-ERR":30,"REG-ONLY":24}`，修出 9 个真实 bug——**其中一半是参数校验缺失**，正对应第 3 步。

## 5.12 踩坑记录：工具系统的三次翻车

**翻车一：`tools/__init__.py` 成了 import 链的瓶颈**。v1 时 `tools/__init__.py` 直接导入所有实现模块，结果 `lsp/tools.py` 想 `from ..sandbox import ...`——sandbox 早已搬进 `tools/` 包，正确写法是 `from ..tools.sandbox import ...`。一个错误路径让**整个包 import 失败**，全部工具不可用。修复：`lsp/tools.py` 改回 `from ..tools.sandbox`。

**翻车二：`ModuleNotFoundError: No module named 'one_cedric.tools.backend'`**（用户 09-27 16:06 粘贴的报错）。`tools/__init__.py` 里有一行残留的 `from .backend import ...`，backend 模块早就删了。教训同上：**包的 `__init__` 是公共出口，任何一行坏 import 都是全局故障**。修复方式是把它改成"薄转发"（v2 起 `__init__.py` 只有 811 字节的 re-export，v3 加上 `__all__` 明确出口）。

**翻车三：`_handlers_readonly` 导入失败导致全线罢工**。这个促成了 `_dispatch._merge` 的"跳过而不崩"设计（见 5.7）——那行 `print(f"[tools] 跳过 {module_name}（{exc}）")` 就是那次事故的纪念碑。

三次翻车指向同一条守则：**工具系统的可靠性目标不是"永不失败"，而是"局部失败，整体存活"**。

## 5.13 手把手：从零写三个工具

前面 12 节都在"读代码"。这一节换你来写——三个例子，难度递进，全部按 v3 的真实结构走：

| # | 工具 | 类型 | 你会学到 |
| --- | --- | --- | --- |
| 1 | `text_slug` | 只读、纯函数 | 最小闭环：schema → 实现 → handler → 分类 → 测试 |
| 2 | `batch_rename` | 写工具 | preview/apply 约定；确认、快照、审计全部"免费"获得 |
| 3 | `image_ascii` | 可选依赖 | 惰性导入 + 缺依赖友好报错 + 输出规模上限 |

每写完一步都有**验证手段**——不要攒到最后一起跑。

### 工具一：`text_slug`——只读、纯函数

**要做什么**：把任意文本变成 URL slug（`Hello World! 你好，世界` → `hello-world-你好世界`）。

**第 1 步：schema（`one_cedric/tools/schema.py`）**

在 `TOOLS` 列表里、与 `word_frequency`/`dedupe_lines` 相邻的位置插入：

```python
{"type": "function", "function": {
    "name": "text_slug",
    "description": "把文本转成 URL slug：小写、空白转连字符、去标点，保留中文。",
    "parameters": {"type": "object", "properties": {
        "text": {"type": "string"},
        "file": {"type": "string", "description": "从文件读取（与 text 二选一）。"},
        "max_len": {"type": "integer", "description": "最大长度，默认 80。"},
    }},
}},
```

两个细节：description 是给模型的**路由信息**（"什么时候该用它"）；`text` / `file` 都留在 `properties` 而不是 `required`——"两个里至少给一个"这种约束 schema 表达不了，交给实现层报错。**schema 只说"参数长什么样"，不说业务规则。**

**第 2 步：实现（`one_cedric/tools/text_advanced.py`）**

这个模块已经有 `_read(text, file, root)` helper（统一处理"读文件 / 直接用文本 / 两个都没有就报错"），直接复用：

```python
def text_slug(text: str = "", file: str = "",
              max_len: int = 80, root: Path | None = None) -> str:
    raw, err = _read(text, file, root)
    if err:
        return err
    s = raw.strip().lower()
    s = re.sub(r"\s+", "-", s)        # 空白 → 连字符
    s = re.sub(r"[^\w-]+", "", s)     # 去标点（\w 在 Python 3 里含中文）
    s = re.sub(r"-{2,}", "-", s).strip("-")   # 折叠重复连字符、去首尾
    if max_len > 0 and len(s) > max_len:
        s = s[:max_len].rstrip("-")
    if not s:
        return "ERROR: 文本为空，或去掉标点后什么都不剩。"
    return f"slug: {s}"
```

三条贯穿全项目的约定在这里全齐了：**错误以 `"ERROR: "` 开头返回**（不是抛异常）；**返回 `str`**（模型能直接读懂的自然语言 + 数据）；**参数全带默认值**（因为你收到的永远是 `args.get()` 出来的"可能缺"的东西）。

> 💡 Python 3 的 `\s` 是 Unicode 语义——全角空格 `\u3000`、不换行空格 `\u00a0` 都会被匹配到。自己验证一下：
>
> ```bash
> python -c "import re; print(bool(re.match(r'\s', '\u3000')))"   # True
> ```
>
> 所以中文输入法最常见的全角空格，一条 `re.sub(r"\s+", "-", s)` 就顺带处理了，不用再手写字符类。

**第 3 步：handler（`one_cedric/tools/_handlers_text.py`）**

```python
def _text_slug(root: Path, args: dict) -> str:
    mod = _imp("text_advanced")
    return mod.text_slug(
        text=args.get("text", ""), file=args.get("file", ""),
        max_len=int(args.get("max_len", 80) or 80),
        root=root)
```

再把它登记进文件底部的 `HANDLERS` 表：

```python
    "dedupe_lines":       _dedupe_lines,
    "text_slug":          _text_slug,        # ← 新增
    "text_similarity":    _text_similarity,
```

注意 `int(args.get("max_len", 80) or 80)`：模型可能传 `"40"`（字符串）甚至 `null`。归一化层（5.9）会尽力修，但**handler 是最后一道门**——自己再防一次，成本一行。

**第 4 步：分类（`schema.py`）**

只读，所以进 `READONLY_TOOL_NAMES` 的"文本"分组：

```python
    # 文本
    "regex_extract", "code_stats", "markdown_toc", "template_render",
    "word_frequency", "tokenize", "simplify_chinese",
    "dedupe_lines", "text_similarity", "strip_invisible",
    "text_slug",                                # ← 新增
```

结果只由参数决定，所以也可以进缓存（`CACHEABLE_TOOLS`）：

```python
    "dedupe_lines", "text_similarity", "strip_invisible",
    "text_slug",                                # ← 新增
```

> ⚠️ 进缓存前问两个问题：**结果只由参数决定吗？**（同样的 `text` 永远同样的 slug → 是）**写路径会清掉它吗？**（会——core 里每次写操作后都 `self.tool_cache.clear()`）。两个都是"是"，才加。反例：`disk_tree` 的结果依赖磁盘当下的状态，加缓存就可能给出过时的目录树（v3 里它加了，靠"写后清空 + 300 秒 TTL"兜底——**这是取舍，不是免费的**）。

**第 5 步：测试（`tests/test_tools_e2e.py`）**

```python
def test_text_slug(workspace):
    out = _call("text_slug", {"text": "Hello World! 你好，世界"}, workspace)
    assert "slug: hello-world-你好世界" in out
```

**验证三步**（每步都别跳过）：

```bash
python -m pytest tests/test_tool_registry.py -q                    # 契约不变量
python -m pytest tests/test_tools_e2e.py -k text_slug -q           # 真调用一次
python -c "from pathlib import Path; from one_cedric.tools import dispatch_tool; \
print(dispatch_tool('text_slug', {'text': 'One Cedric Wiki!'}, Path('.')))"
```

第一条最重要——`test_tool_registry.py` 会替你检查：schema 里的工具**有没有实现**（`test_every_tool_is_routable`）、handler **有没有被 schema 曝光**（`test_no_orphan_handlers`）、缓存集合**是不是只读子集**（`test_cacheable_tools_are_read_only`）。漏了任何一步，它会指名道姓报出来。

**最后**：启动 CLI，对模型说"用 text_slug 处理一下 'Hello World 2026'"。它自己选中了新工具，说明 description 写得合格；它绕路去调 `python_exec`，说明描述里还得补一句使用场景。

### 工具二：`batch_rename`——写工具，走确认流

**要做什么**：对工作目录里匹配 `glob` 的文件批量改文件名（正则替换），**改名前必须先看到预览**。

写工具不写 handler，而是写 **preview / apply 一对**（5.8 的约定），然后接进 core 的"简单写工具"通道 `_handle_simple_write`——走这条通道，**确认菜单、git 快照、审计日志、缓存清理全部免费获得**。

**第 1 步：schema + 分类**

```python
{"type": "function", "function": {
    "name": "batch_rename",
    "description": "批量重命名工作目录里的文件：glob 选文件，pattern 是正则，"
                    "replacement 是替换文本（支持 \\1 分组引用）。写操作，"
                    "会先展示改名预览再执行。",
    "parameters": {"type": "object", "properties": {
        "glob": {"type": "string", "description": "匹配模式，如 *.md。"},
        "pattern": {"type": "string", "description": "要匹配的正则。"},
        "replacement": {"type": "string", "description": "替换文本。"},
        "dir": {"type": "string", "description": "相对子目录，默认 '.'。"},
    }, "required": ["glob", "pattern", "replacement"]},
}},
```

然后加进 `WRITE_TOOLS`：

```python
WRITE_TOOLS = {
    "edit_file", "write_file", "apply_patch", "file_ops",
    ...
    "batch_rename",                                # ← 新增
}
```

**第 2 步：preview / apply（`one_cedric/tools/fileops.py`）**

```python
import re


def batch_rename(glob: str, pattern: str, replacement: str,
                 dir: str = ".", root: Path | None = None):
    """预览：算出将发生的改名。返回 (preview, is_write, err)。"""
    if root is None:
        return "", False, "ERROR: 需要 root"
    base, err = _resolve_path(root, dir)
    if err:
        return "", False, err
    if not base.is_dir():
        return "", False, f"ERROR: 目录不存在: {dir}"
    try:
        rx = re.compile(pattern)
    except re.error as exc:
        return "", False, f"ERROR: 正则不合法: {exc}"

    files = [p for p in sorted(base.glob(glob)) if p.is_file()]
    plan = [(p, rx.sub(replacement, p.name)) for p in files]
    plan = [(p, n) for p, n in plan if n != p.name]
    if not plan:
        return f"没有需要改名的文件（匹配 {len(files)} 个）。", False, ""

    names = [n for _, n in plan]
    if any("/" in n or "\\" in n or n in ("", ".", "..") for n in names):
        return "", False, "ERROR: 替换结果不是合法文件名，拒绝执行。"
    dupes = {n for n in names if names.count(n) > 1}
    lines = [f"计划重命名 {len(plan)} 个文件："]
    for p, n in plan:
        flag = "  ⚠ 目标重名" if n in dupes else ""
        lines.append(f"  {p.name}  →  {n}{flag}")
    if dupes:
        lines.append("注意：目标名有冲突，执行时会拒绝。")
    return "\n".join(lines), True, ""


def batch_rename_apply(glob: str, pattern: str, replacement: str,
                       dir: str = ".", root: Path | None = None) -> str:
    """执行：按同一份参数重新算一遍，再真正改名。"""
    if root is None:
        return "ERROR: 需要 root"
    base, err = _resolve_path(root, dir)
    if err:
        return err
    rx = re.compile(pattern)
    plan = []
    for p in sorted(base.glob(glob)):
        if not p.is_file():
            continue
        n = rx.sub(replacement, p.name)
        if n == p.name:
            continue
        if "/" in n or "\\" in n or n in ("", ".", ".."):
            return f"ERROR: 非法目标名: {n}"
        plan.append((p, n))
    names = [n for _, n in plan]
    if len(names) != len(set(names)):
        return "ERROR: 目标文件名冲突，已放弃。"
    done = 0
    for p, n in plan:
        dst = p.with_name(n)
        if dst.exists():
            return f"ERROR: 目标已存在: {n}（已改名 {done} 个）"
        p.rename(dst)
        done += 1
    return f"已重命名 {done} 个文件。"
```

`preview` 返回**三元组**是这套约定的核心：

- `(preview, True, "")` → core 打印预览 → 弹确认菜单（第 7 章的 5 选项）→ 用户按 y 才调 `apply`；
- `(preview, False, "")` → "查了一遍，什么都不用做"——core 把 preview 直接当结果返回，**不弹确认**（这就是"没有需要改名的文件"分支存在的意义）；
- `("", False, "ERROR: ...")` → 出错，原样返回给模型。

> ⚠️ **踩坑**：`apply` 不能接收 `preview` 算好的结果再执行——**两个函数吃同一份参数、各自重算一遍**。中间隔着用户确认的几十秒，磁盘状态随时可能变（用户手动改了、别的工具在跑）。上面 `batch_rename_apply` 里"把 plan 重新算一遍"不是重复，是防御。v3 里所有 preview/apply 对都遵守这一条。

**第 3 步：接入 core（三处小改）**

(1) `_SIMPLE_WRITE_DISPATCH`（`core.py`）——告诉 core 这对函数叫什么、吃哪些参数：

```python
    _SIMPLE_WRITE_DISPATCH = {
        ...
        "batch_rename": ("batch_rename", "batch_rename_apply",
                         ("glob", "pattern", "replacement", "dir")),
    }
```

(2) `_dispatch_write_single` 里路由到简单写工具的那组名字（`core.py`）：

```python
        if name in ("video_convert", ..., "k8s_exec",
                    "batch_rename"):                  # ← 新增
            self._bump_tool(name)
            self._handle_simple_write(tc)
            return
```

(3) re-export（`tools/__init__.py`）——`_handle_simple_write` 是用 `getattr(one_cedric.tools, 名字)` 找函数的：

```python
from .fileops import batch_rename, batch_rename_apply
```

**第 4 步：测试**

```python
def test_batch_rename_preview_and_apply(workspace):
    for n in ("a1.txt", "a2.txt"):
        (workspace / n).write_text("x")
    from one_cedric.tools import batch_rename, batch_rename_apply

    preview, is_write, err = batch_rename(
        "*.txt", r"a(\d)", r"b\1", ".", root=workspace)
    assert is_write and not err
    assert "a1.txt  →  b1.txt" in preview

    out = batch_rename_apply("*.txt", r"a(\d)", r"b\1", ".", root=workspace)
    assert not out.startswith("ERROR")
    assert (workspace / "b1.txt").exists()
    assert not (workspace / "a1.txt").exists()
```

**验证**：

```bash
python -m pytest tests/test_tool_registry.py tests/test_tools_e2e.py -q
python -m one_cedric --chat     # 说："把 demo 目录下的 *.md 文件名里的 demo- 前缀去掉"
```

在 CLI 里你会依次看到：**预览面板 → 确认菜单 → git 快照 → 审计日志多一条 `batch_rename ok`**。这四件事你一行都没写，是 core 的通道送的——这就是"加入一个成熟的工具系统"和"从零搭一个"的差别。

### 工具三：`image_ascii`——可选依赖 + 输出规模上限

**要做什么**：把一张图转成字符画（ASCII art），只读。

**第 1 步：schema + 分类**

```python
{"type": "function", "function": {
    "name": "image_ascii",
    "description": "把图片转成字符画（ASCII art），按亮度映射到一组字符。",
    "parameters": {"type": "object", "properties": {
        "path": {"type": "string", "description": "图片路径。"},
        "width": {"type": "integer", "description": "字符画宽度，默认 80，最大 200。"},
        "invert": {"type": "boolean", "description": "反色（深色背景用）。"},
    }, "required": ["path"]},
}},
```

加进 `READONLY_TOOL_NAMES` 的"图片"分组：

```python
    # 图片
    "image_info", "image_process",
    "image_ascii",                                 # ← 新增
    "ocr_image", "ocr_status",
```

**第 2 步：实现（`one_cedric/tools/image_ops.py`）**

这个模块已经有一个 `_pil()`：惰性获取 Pillow，缺依赖时返回带修复命令的错误。直接用：

```python
ASCII_CHARS = " .:-=+*#%@"


def image_ascii(path: str, width: int = 80, invert: bool = False,
                root: Path | None = None) -> str:
    if root is None:
        return "ERROR: 需要 root"
    Image, err = _pil()
    if err:
        return err
    p, perr = _resolve_path(root, path)
    if perr:
        return perr
    if not p.exists():
        return f"ERROR: 文件不存在: {path}"
    width = max(20, min(int(width or 80), 200))
    try:
        img = Image.open(p).convert("L")
    except Exception as exc:
        return f"ERROR: 打开图片失败: {exc}"

    height = max(1, int(width * (img.height / img.width) * 0.5))
    img = img.resize((width, height))
    px = img.load()
    chars = ASCII_CHARS[::-1] if invert else ASCII_CHARS
    n = len(chars) - 1
    lines = ["".join(chars[px[x, y] * n // 255] for x in range(width))
             for y in range(height)]
    return "\n".join(lines)
```

三个值得展开的决定：

- **惰性导入**：Pillow 在 `requirements-optional.txt` 里，不是每台机器都有。`_pil()` 在函数体内尝试导入，缺了就返回 `"ERROR: 需要 Pillow（pip install Pillow）"`——**错误信息里带修复命令**，模型会把这句话原样念给用户听；
- **宽度钳位** `max(20, min(width, 200))`：模型可能传 `width: 5000`。不做上限，一次渲染就是 5000 字符 × 数千行，灌爆上下文和终端。**凡是有"输出规模"参数的工具，都必须有钳位**；
- **纵横比修正 ×0.5**：终端字符格是"高比宽长"的矩形，不修正的话圆脸会被拉成长脸。**输出媒介的形状也要在代码里补偿**。

**第 3 步：handler（`_handlers_media.py`）**

```python
def _image_ascii(root: Path, args: dict) -> str:
    err = _require(args, "path")
    if err:
        return err
    mod = _imp("image_ops")
    return mod.image_ascii(
        args["path"],
        width=args.get("width", 80),
        invert=bool(args.get("invert", False)),
        root=root)
```

登记进 `HANDLERS`：

```python
    "image_info":         _image_info,
    "image_ascii":        _image_ascii,            # ← 新增
    "image_process":      _image_process,
```

**第 4 步：测试（缺依赖就跳过）**

```python
def test_image_ascii(workspace):
    Image = pytest.importorskip("PIL.Image")      # 没装 Pillow → skip，不算失败
    Image.new("L", (40, 20), color=255).save(workspace / "white.png")
    out = _call("image_ascii", {"path": "white.png", "width": 40}, workspace)
    assert len(out.splitlines()) >= 5
```

> 💡 `pytest.importorskip` 是可选依赖工具的标准测试姿势：**有依赖就真跑，没依赖就显式跳过**。比"假装通过"诚实，比"整个套件报错"友好——这也是第 12 章里那 30 个"环境依赖"工具不算失败的原因。

**验证**：

```bash
python -m pytest -q -k "image_ascii or registry"
python -c "from pathlib import Path; from one_cedric.tools import dispatch_tool; \
print(dispatch_tool('image_ascii', {'path': 'demo.png', 'width': 60}, Path('.')))"
```

### 三个写完了，回头对一遍 5.11 的清单

| 5.11 的步骤 | `text_slug` | `batch_rename` | `image_ascii` |
| --- | --- | --- | --- |
| ① schema 条目 | ✅ | ✅ | ✅ |
| ② 实现函数 | `text_advanced.py` | `fileops.py`（preview/apply 对） | `image_ops.py` |
| ③ handler | ✅ | —（走 core 通道） | ✅ |
| ④ 分类 | `READONLY` | `WRITE` | `READONLY` |
| ⑤ 缓存决策 | 进缓存 | —（写工具永不缓存） | 想清楚再加 |
| ⑥ 动画帧 | 可选 | 可选 | 可选 |
| ⑦ 测试 | e2e | preview/apply 各一 | skip-if-missing |
| ⑧ 审计 | 无风险面 | 文件写入，走确认 | 读文件，无风险面 |

注意最后一列里的"可选"和"—"：**"显式地不做"和"忘了做"是两回事**。动画帧没画（默认帧会兜底）、冒烟报告没写（发布前统一补）——这些都是"知道自己在跳什么"。

> 💡 **设计笔记**：三个工具恰好对应工具系统要打交道的三种外部世界——**纯函数**（无状态、可缓存）、**副作用**（必须预览 + 确认 + 快照）、**环境依赖**（必须优雅降级）。250 个工具的复杂度，本质就是这三种原型 × 不同领域。以后每加一个工具，先问它属于哪种，把对应通道接对，剩下的都是领域逻辑。

## 🔧 自己动手

1. **合上 5.13 复现一遍**：不看上面的代码，自己写一个只读工具 `disk_usage`——返回工作目录下各子目录大小（前 10 名）。写完再对照 5.13 检查：schema、实现、handler、分类、测试，你到底漏了哪一步；
2. **故意搞坏 JSON**：让模型（或手写）发一个 `arguments` 为 `{'path': 'x.txt',}` 的坏调用，验证 `repair_json` 能修复；再试试缺右括号的情况；
3. **验证分类的效果**：把 `weather` 临时加进 `WRITE_TOOLS`，观察调用它时是否开始要求确认——理解分类对行为的影响；
4. **给 `_merge` 加一个假模块**：`_merge("_handlers_nonexistent")`，确认程序照常启动、其他工具照常工作；
5. **改一个已写好的工具**：给 `batch_rename` 去掉 `dir` 参数、改让 `glob` 直接写 `sub/*.txt`。你会立刻撞上三处假设：`_SIMPLE_WRITE_DISPATCH` 里还列着 `dir`；`apply` 里的 `p.with_name()` 需要对每个文件重算 base；预览里的显示路径要跟着变。**改工具比写工具更容易漏——这就是测试存在的意义。**

---

[← 上一章：04 终端 UI 进化](04-rich-ui) ｜ [返回目录](Home) ｜ [下一章：06 流式与推理 →](06-streaming-reasoning)
