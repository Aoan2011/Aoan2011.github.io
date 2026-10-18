# 02 · 第一个 Agent：500 行起步

> 本章代码全部来自 2026-09-13 的真实开发记录（对话 #57），只做了排版与少量注释补充。你可以完整复制运行。

## 2.1 需求：一句话开始

📅 **开发日志 · 2026-09-13 16:58**（对话 #57）

> **用户**：请你生成一个简易的 Python agent-cli，支持 read-file tool，ollama 模型

十七分钟后，第一版代码完成。又过了五分钟，用户补了一句：

📅 **2026-09-13 17:03**

> **用户**：jiajigetools

AI 把这句话理解成了"加几个 tools"，于是又加上了目录列表、全文搜索、文件信息三个只读工具。这个有点好笑的细节，恰好说明了一件事：**Agent 的能力边界，就是工具列表的长度**。后来 One Cedric 的 240+ 工具，就是从这四个开始的。

本章的目标不是写出最终版，而是写出一个**结构正确**的最小版。它应该：

- 能连接 Ollama（或任意 OpenAI 兼容服务）；
- 有 `read_file` 工具，且被限制在工作目录内；
- 能完成"模型请求工具 → 执行 → 回填 → 再请求"的完整循环；
- 有可交互的 CLI。

## 2.2 五个设计决定

在写代码之前，先看这一版做出的五个决定，它们决定了后面一个月的走向：

| 决定 | 理由 | 后果 |
| --- | --- | --- |
| 用 OpenAI 兼容端点，不用 Ollama 原生 API | `tool_calls` 有标准 `id`，换后端不用改代码 | 后来接 DeepSeek/Kimi 只改配置 |
| 第一版**非流式** | 工具调用的流式解析复杂，先跑通闭环 | 第 6 章才补上流式 |
| `read_file` 带工作目录沙箱 | 模型可能读任意路径，必须限制 | 成为整个安全体系的起点 |
| 工具错误**以字符串返回**，不抛异常 | 让模型看到错误并自我修正 | 后来所有工具都遵守这个约定 |
| 只读工具优先 | 写操作需要确认机制，第一版不做 | 第 7 章补权限层 |

## 2.3 代码骨架

```
agent.py
├── 配置区        DEFAULT_HOST / DEFAULT_MODEL / 限制常量 / SYSTEM_PROMPT
├── 工具定义      TOOLS（JSON Schema，给模型看）
├── 工具实现      read_file() / list_files() / ...
├── 分发器        dispatch_tool()（按名字路由）
├── 模型调用      chat()（HTTP 请求）
├── 消息规整      normalize_assistant()
├── 主循环        agent_turn()（本章的核心）
└── CLI          main()（argparse + REPL）
```

这个骨架后来被完整保留在 One Cedric 里，只是每个部分都长大了：`TOOLS` 变成 120KB 的 `schema.py`，`dispatch_tool` 变成 handler 注册表，`agent_turn` 变成 2000 行的调度器。

## 2.4 工具定义：写给模型看的"说明书"

```python
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": (
                "读取本地文本文件的内容，返回带行号的文本。"
                "可通过 start_line / end_line 读取指定行范围，适合处理大文件。"
            ),
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {
                        "type": "string",
                        "description": "文件路径，可以是相对工作目录的相对路径，也可以是绝对路径。",
                    },
                    "start_line": {
                        "type": "integer",
                        "description": "起始行号，从 1 开始。默认 1。",
                    },
                    "end_line": {
                        "type": "integer",
                        "description": "结束行号（含）。默认读到文件末尾。",
                    },
                },
                "required": ["path"],
            },
        },
    }
]
```

三个容易写错的点：

1. **`description` 是提示词，不是注释**。模型靠它判断"何时用、怎么用"。写"读取文件"和写"读取本地文本文件，返回带行号的内容，可用行范围分页"——后者的调用准确率明显更高；
2. **`required` 只写真正必需的**。`start_line` / `end_line` 有默认值，就不该进 `required`，否则模型每次都得猜；
3. **参数类型要具体**。`integer` 会让模型传 `10`，`string` 会让它传 `"10"`——后者你还要在实现里做类型容错。

## 2.5 路径沙箱：第一道安全线

```python
def _resolve_path(root: Path, path: str) -> tuple[Path | None, str]:
    """解析路径并确保在沙箱内。返回 (绝对路径, 错误信息)。"""
    if not path or not isinstance(path, str):
        return None, "ERROR: 参数 path 不能为空。"

    p = Path(path).expanduser()
    if not p.is_absolute():
        p = root / p

    try:
        p = p.resolve()
    except OSError as exc:
        return None, f"ERROR: 路径解析失败: {exc}"

    try:
        p.relative_to(root)
    except ValueError:
        return None, f"ERROR: 拒绝访问工作目录之外的路径: {path}（工作目录: {root}）"

    return p, ""
```

这段不到 20 行的函数，是 One Cedric 安全体系的原点。三个关键动作：

- **先 `resolve()` 再比较**：`..`、符号链接、Windows 的短路径（`PROGRA~1`）都可能骗过朴素的前缀比较，解析成真实路径后才可靠；
- **用 `relative_to()` 判断包含关系**：`str(p).startswith(str(root))` 是经典错误——`C:\work2` 会以 `C:\work` 开头；
- **失败返回错误字符串**：模型能看到"拒绝访问工作目录之外的路径"，从而调整行为，而不是让整个进程崩掉。

> ⚠️ **一个真实的历史 bug**：早期实现用的是 `root not in p.parents and p != root`。当 `root` 本身是盘符根目录（如 `F:\`）时，`parents` 为空，判断会出错。后来统一改成 `relative_to()`。这类边界条件，只有真正被用户拿奇怪路径试过才会暴露。

## 2.6 工具实现：好工具会"教"模型怎么用

```python
MAX_BYTES = 200_000   # 单次最多读取的字节数
MAX_LINES = 2_000     # 单次最多返回的行数


def read_file(root: Path, path: str, start_line=1, end_line=None) -> str:
    p, err = _resolve_path(root, path)
    if err:
        return err

    rel = str(p.relative_to(root))
    if not p.exists():
        return f"ERROR: 文件不存在: {rel}"
    if not p.is_file():
        return f"ERROR: 不是普通文件: {rel}"

    try:
        raw = p.read_bytes()
    except OSError as exc:
        return f"ERROR: 读取失败: {exc}"

    truncated_bytes = len(raw) > MAX_BYTES
    if truncated_bytes:
        raw = raw[:MAX_BYTES]

    text = raw.decode("utf-8", errors="replace")
    lines = text.splitlines()

    if not lines:
        return f"文件: {rel}\n(空文件，0 行)"

    total = len(lines)
    start = max(1, _as_int(start_line, 1))
    if start > total:
        return f"ERROR: start_line={start} 超出文件总行数 {total}（文件: {rel}）"

    if end_line is None:
        end = total
    else:
        end = min(_as_int(end_line, total), total)
    if end < start:
        return f"ERROR: end_line({end}) 小于 start_line({start})"

    if end - start + 1 > MAX_LINES:
        end = start + MAX_LINES - 1

    header = f"文件: {rel}  (共 {total} 行，显示第 {start}-{end} 行)"
    body = "\n".join(f"{i:>6}| {lines[i - 1]}" for i in range(start, end + 1))

    footer = ""
    if end < total:
        footer += f"\n... (还有 {total - end} 行未显示，可用 start_line/end_line 继续读取)"
    if truncated_bytes:
        footer += f"\n... (文件过大，已截断到前 {MAX_BYTES // 1000}KB)"

    return f"{header}\n{body}{footer}"
```

值得学习的不是代码本身，而是几个"工具设计"细节：

1. **返回带行号的内容**（`f"{i:>6}| {line}"`）。模型后续引用行号、或调用 `edit_file` 时，行号是重要的定位锚点；
2. **截断要"可恢复"**：末尾明确告诉模型"还有多少行没显示，怎么继续读"。否则模型会以为文件就这么多，给出错误结论；
3. **错误信息要可行动**："文件不存在"比"读取失败"有用；"start_line 超出范围（共 120 行）"比"参数错误"有用；
4. **非法字节不崩溃**：`decode("utf-8", errors="replace")` 让二进制文件、GBK 文件也能读出一个可解释的结果（真正的多编码检测是后来才加的）。

## 2.7 调用模型：只有 40 行

```python
def chat(messages: list, model: str, host: str, temperature: float = 0.2) -> dict:
    """调用 Ollama 的 OpenAI 兼容接口，返回 assistant message。"""
    url = f"{host.rstrip('/')}/v1/chat/completions"
    payload = {
        "model": model,
        "messages": messages,
        "tools": TOOLS,
        "tool_choice": "auto",
        "stream": False,
        "temperature": temperature,
    }

    try:
        resp = requests.post(url, json=payload, timeout=(10, 600))
    except requests.exceptions.ConnectionError:
        raise SystemExit(
            f"无法连接到 Ollama（{url}）。\n"
            f"请确认已启动服务：ollama serve"
        )
    except requests.exceptions.Timeout:
        raise SystemExit("请求 Ollama 超时。")

    if resp.status_code >= 400:
        raise SystemExit(
            f"Ollama 返回错误 {resp.status_code}:\n{resp.text[:800]}\n\n"
            f"提示：请确认模型 '{model}' 已拉取（ollama pull {model}），"
            f"且该模型支持工具调用。"
        )

    data = resp.json()
    try:
        return data["choices"][0]["message"]
    except (KeyError, IndexError):
        raise SystemExit(f"无法解析 Ollama 响应:\n{json.dumps(data, ensure_ascii=False)[:800]}")
```

工程上做对的三件事：

- **超时分成两段** `(10, 600)`：连接 10 秒，读取 600 秒。本地模型首次加载可能很慢，读取超时不能太短；
- **错误信息带修复建议**：不只是抛 401/404，而是提示"模型是否已 pull、是否支持工具调用"；
- **响应解析兜底**：结构不符时打印原始 JSON 片段，方便定位。

## 2.8 消息规整：为什么不能把响应原样塞回去

```python
def normalize_assistant(msg: dict) -> dict:
    """整理 assistant 消息，只保留回传给 API 所需的字段。"""
    out: dict = {"role": "assistant", "content": msg.get("content") or ""}
    tool_calls = msg.get("tool_calls") or []
    if tool_calls:
        normalized = []
        for i, tc in enumerate(tool_calls):
            fn = tc.get("function") or {}
            args = fn.get("arguments", "{}")
            if not isinstance(args, str):
                args = json.dumps(args, ensure_ascii=False)
            normalized.append(
                {
                    "id": tc.get("id") or f"call_{i}",
                    "type": "function",
                    "function": {"name": fn.get("name", ""), "arguments": args},
                }
            )
        out["tool_calls"] = normalized
    return out
```

这件事看起来多此一举，实际上是**兼容不同后端的必需品**：

- 有的后端会给 `tool_calls` 加 `index`、`extra_content` 等字段，原样回传可能被下一轮拒绝；
- 有的后端不返回 `id`（早期 Ollama 原生接口就是这样），需要自己补一个；
- `arguments` 有时是对象、有时是字符串，统一成字符串最稳。

> 💡 **设计笔记**：Agent 代码里，**进出模型的每一份数据都要"过一遍标准化"**。模型供应商五花八门，你的循环必须只依赖协议里保证的字段。

## 2.9 主循环：整章的心脏

```python
def agent_turn(messages: list, model: str, host: str, root: Path,
               max_steps: int = 8) -> str:
    """执行一轮对话（可能包含多次工具调用），返回最终文本回答。"""
    for _step in range(max_steps):
        msg = chat(messages, model, host)
        messages.append(normalize_assistant(msg))

        tool_calls = msg.get("tool_calls") or []
        if not tool_calls:
            content = (msg.get("content") or "").strip()
            return content or "(模型返回了空回复)"

        for tc in messages[-1].get("tool_calls", []):
            name = tc["function"]["name"]
            raw_args = tc["function"]["arguments"]

            try:
                args = json.loads(raw_args) if isinstance(raw_args, str) else (raw_args or {})
            except json.JSONDecodeError:
                result = f"ERROR: 参数不是合法 JSON: {raw_args!r}"
                args = {}
            else:
                preview = ", ".join(f"{k}={v!r}" for k, v in args.items())
                print(f"\033[36m  ⚙ {name}({preview})\033[0m", file=sys.stderr)
                result = dispatch_tool(name, args, root)

            messages.append(
                {"role": "tool", "tool_call_id": tc["id"], "content": result}
            )

    return "(已达到最大工具调用步数，请简化问题后重试)"
```

逐行看几个关键点：

- **`max_steps` 必须有**。模型可能陷入"读文件 → 读文件 → 读文件"的死循环。给一个上限，并在超限时返回可读的提示，是最低成本的保险；
- **assistant 消息必须先入历史**。工具结果（`role: tool`）必须跟在一个带 `tool_calls` 的 assistant 消息后面，否则 API 会报"tool_call_id 找不到对应的调用"；
- **`tool_call_id` 一一对应**。当模型一轮返回多个工具调用时，每个结果都要用对应的 id 回填，顺序错了就会张冠李戴；
- **参数解析失败也要回填**。注意 `except` 分支里给了一个 `ERROR: 参数不是合法 JSON` 的 result——让模型看到自己传错了，下一轮就有机会改；
- **工具调用日志打到 stderr**。stdout 留给最终回答，这样 `python agent.py -p "..." > answer.txt` 时文件里不会混入调试信息。

## 2.10 CLI：把循环包成产品

```python
def main() -> None:
    parser = argparse.ArgumentParser(
        description="简易 Ollama Agent CLI（支持 read_file 工具）",
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
    )
    parser.add_argument("-m", "--model", default=DEFAULT_MODEL)
    parser.add_argument("--host", default=DEFAULT_HOST)
    parser.add_argument("-r", "--root", default=".", help="允许读取的工作目录（沙箱根）")
    parser.add_argument("-p", "--prompt", help="单次提问后退出")
    parser.add_argument("--max-steps", type=int, default=8)
    parser.add_argument("-t", "--temperature", type=float, default=0.2)
    args = parser.parse_args()

    root = Path(args.root).expanduser().resolve()
    if not root.is_dir():
        raise SystemExit(f"工作目录不存在或不是目录: {root}")

    model = args.model
    system_msg = {"role": "system", "content": SYSTEM_PROMPT}
    messages: list = [system_msg]

    def run(prompt: str) -> str:
        messages.append({"role": "user", "content": prompt})
        return agent_turn(messages, model, args.host, root, args.max_steps)

    if args.prompt:
        print(run(args.prompt))
        return

    while True:
        try:
            line = input("you> ").strip()
        except (EOFError, KeyboardInterrupt):
            print("\n再见！")
            break
        # /help /clear /model /exit 略
        try:
            answer = run(line)
        except SystemExit as exc:
            print(f"\n{exc}\n", file=sys.stderr)
            messages.pop()  # 移除刚才压入的 user 消息
            continue
        except KeyboardInterrupt:
            print("\n(已中断本轮对话)", file=sys.stderr)
            continue
        print(f"\nassistant> {answer}\n")
```

这段 CLI 里有两个细节后来被保留了很多年：

- **失败时 `messages.pop()`**。网络错误导致这一轮没完成，把用户消息撤回，避免历史里留下一个"用户说了话但助手没回"的悬空轮次；
- **`/model` 切换后清空历史**。不同模型的上下文格式、能力不同，混用历史容易出怪问题。

## 2.11 跑起来是什么样

```text
$ python agent.py -m qwen2.5:7b -r ./my_project
Ollama Agent CLI  |  model=qwen2.5:7b  |  root=F:\my_project
输入 /help 查看命令，/exit 退出。

you> README.md 里写了什么？
  ⚙ read_file(path='README.md')

assistant> 这个项目是……
```

`⚙ read_file(path='README.md')` 这行来自 stderr，是开发时最有用的"透明度"：你能立刻看出模型**想干什么**，而不是对着一个转圈等待。

## 2.12 加几个 tools：从 1 到 4

📅 **2026-09-13 17:03** · 用户："jiajigetools"

第二版加了三个只读工具：

| 工具 | 作用 | 设计要点 |
| --- | --- | --- |
| `list_files` | 列目录（可递归） | 返回 `[DIR]` / `[FILE]` 前缀，模型一眼能分辨 |
| `search_in_files` | 全文搜索关键词 | 限制单文件大小（500KB）与结果数（50 条），防止把上下文撑爆 |
| `file_info` | 大小、行数、修改时间 | 让模型先"看目录"再决定读哪个 |

分发器也相应扩展：

```python
def dispatch_tool(name: str, args: dict, root: Path) -> str:
    if name == "read_file":
        allowed = {"path", "start_line", "end_line"}
        kwargs = {k: v for k, v in args.items() if k in allowed}
        if "path" not in kwargs:
            return "ERROR: 缺少必需参数 path。"
        return read_file(root=root, **kwargs)

    if name == "list_files":
        allowed = {"path", "recursive"}
        kwargs = {k: v for k, v in args.items() if k in allowed}
        return list_files(root=root, **kwargs)

    # search_in_files / file_info 同理……

    return f"ERROR: 未知工具 '{name}'。可用工具: read_file, list_files, search_in_files, file_info"
```

注意 **`allowed` 白名单过滤**：模型可能传 `{"path": "x", "recursive": "yes", "foo": 1}`，直接把 `**args` 展开会抛 `TypeError`。过滤一遍，既能容错，也能防止模型用奇怪参数搞事。

> 💡 **设计笔记**：这个 `if name == ...` 的链式分发，在工具有 4 个时很清晰；到 40 个时会变成灾难。第 5 章会讲 One Cedric 怎么把它换成"注册表 + 合并"的架构。

## 2.13 真实踩坑记录

这一版代码看起来简单，但上线第一天就遇到了不少问题。以下都是后续对话里真实出现过的：

### 坑 1：模型根本不支持工具调用

```
Ollama 返回错误 400:
{"error":"... does not support tools"}
```

**原因**：不是所有 Ollama 模型都支持 function calling。
**修复**：换 `qwen2.5` / `llama3.1` 等支持工具的模型；错误提示里直接给出 `ollama pull` 建议。

### 坑 2：云端模型的 502

📅 **2026-09-19 16:19**（对话 #67，用户粘贴）

```
Ollama 返回错误 502:
{"error":"Post \"https://ollama.com:443/v1/chat/completions?ts=...\": connection error: PROTOCOL_ERROR"}
```

**原因**：`gpt-oss:20b-cloud` 走的是 Ollama 云端代理，网络不稳时网关直接 502；本地 `ollama run` 却正常。
**修复**：切回本地模型 `qwen2.5:7b`；同时在代码里把 502 与"模型不存在"区分提示。

### 坑 3：模型把工具调用"写"在正文里

📅 **2026-09-19 16:37**（对话 #67，用户粘贴的真实输出）

```
好的，我们开始创建文件并写入一些示例内容。
首先，我将创建 F:\testai.py 文件。

 {"name": "write_file", "arguments": {"path": "F:\\testai.py", "content": ""}}
```

**原因**：一些模型（尤其量化小模型）没走原生 `tool_calls` 通道，而是把调用意图写成了普通文本。你的循环收不到 `tool_calls`，就会把这堆 JSON 当最终回答。
**修复**：后来 One Cedric 加了"文本工具调用恢复"（第 6 章）：扫描正文里的 `{"name": ..., "arguments": ...}`，还原成真正的调用。

### 坑 4：Windows 上没有 `ls`

📅 **2026-09-19 16:16**（对话 #67）

```
✗ bash  ls -la  ERROR: 找不到命令 'ls'。
```

**原因**：模型按 Linux 习惯发命令，宿主却是 Windows。
**修复**：工具描述里明确"当前是 Windows"；后续的 shell 层做命令映射/白名单（第 7 章）。

### 坑 5：上下文越聊越长

多轮对话后，`messages` 会包含所有历史工具结果。一次 `read_file` 可能塞进 200KB 文本，几轮之后就会超出模型上下文。
**修复**：限制单次读取大小、结果截断（这一版已有）；后续加"自动压缩历史"（第 8 章）。

## 2.14 这一版已经具备了什么

回头看，这 500 行已经包含了 Agent 的全部核心机制：

- ✅ 工具 schema 与模型通信
- ✅ 工具执行与结果回填
- ✅ 多步循环与步数上限
- ✅ 路径沙箱
- ✅ 错误可恢复（返回字符串而不是崩溃）
- ✅ 单次 / 交互两种模式

它缺少的是"产品化"所需的一切：

- ❌ 流式输出（用户只能干等）
- ❌ 写操作与权限确认（只能读，不能改）
- ❌ 工具注册表（4 个工具还能忍，40 个就不行了）
- ❌ 会话持久化（关掉就忘）
- ❌ UI（纯文本，没有 Rich、没有动画）

接下来的十章，就是把这些"❌"一个个变成"✅"的过程。

## 🔧 自己动手

1. **跑通它**：把本章代码保存为 `agent.py`，用 `qwen2.5:7b` 问它一个关于本地文件的问题；
2. **加一个工具**：实现 `file_info`（大小、行数、修改时间），记得同时改三处：`TOOLS`、实现函数、`dispatch_tool`；
3. **制造一个错误**：让 `read_file` 读一个不存在的文件，观察模型收到 `ERROR` 后是否会换路径重试；
4. **观察上下文**：在 `agent_turn` 里打印 `len(messages)` 和每条消息的长度，聊十轮后看看增长曲线。

---

[← 上一章：01 概念篇](01-concepts) ｜ [返回目录](Home) ｜ [下一章：03 拆包与工程化 →](03-project-refactor)
