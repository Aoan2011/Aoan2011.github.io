# 04 · 终端 UI 进化：用 Rich 把黑框变产品

## 4.1 "不，我需要你用 Rich 来美化 UI"

📅 **开发日志 · 2026-09-19**（对话 #67）

> **10:02 用户**：我们不能去改一下 ui，你看一下意见
>
> **10:05 用户**：不，我需要你用 rich 来美化 ui
>
> **10:13 用户**：还要再美化一下，执行每一个 tools 都有不同的执行动画

此时项目刚拆完包，功能上已经能跑，但界面还是最朴素的 `print()`。作者的要求很明确：**不是"能看"，而是"好看、生动、有辨识度"**。

这一章讲怎么用 [Rich](https://github.com/Textualize/rich) 做到三件事：

1. 流式输出时，Markdown 能实时重排并且不闪烁；
2. 模型"思考"和"回答"两个通道能被区分展示；
3. 每个工具执行时有自己的动画，最终还有一只像素小狗。

> 本章的 UI 代码在 v3 里已经拆分到 `one_cedric/ui/` 包；但设计思路和核心调用从 v1 起就没变过。文中引用的行号以 v1/v3 快照为准。

## 4.2 为什么是 Rich

终端 UI 的选项并不多：手写 ANSI、curses、Textual、Rich。选择 Rich 的理由：

- **Markdown 渲染开箱即用**：模型输出天然是 Markdown，Rich 的 `Markdown` 组件能渲染标题、列表、代码块、表格；
- **`Live` 组件**：可以在一小块区域反复刷新，而不是每次 `print` 都追加新行——这是流式输出的关键；
- **主题系统**：把颜色收敛成语义化 token（`brand` / `tool` / `ok` / `warn` / `err`），换配色只改一处；
- **纯 Python、无编译依赖**：Windows 上 `pip install rich` 即可。

后来 One Cedric 的 WebUI 走的是另一条路（浏览器 + CSS），但 CLI 始终是 Rich。Textual TUI 是发布前才加的第三条路（第 12 章会提到）。

## 4.3 第一层：语义化主题

不要到处写 `[red]`、`[bold green]`。One Cedric 在 `config.py` 里定义了色板与主题：

```python
BRAND = "#d97757"
ACCENT = "#4ec9b0"
USER_C = "#4a9eff"
TOOL_C = "#7dcfff"
WARN_C = "#e0af68"
OK_C = "#9ece6a"
ERR_C = "#f7768e"
DIM_C = "#565f89"

THEME = Theme({
    "brand":     f"bold {BRAND}",
    "accent":    ACCENT,
    "user":      f"bold {USER_C}",
    "tool":      TOOL_C,
    "tool.name": f"bold {TOOL_C}",
    "ok":        f"bold {OK_C}",
    "warn":      f"bold {WARN_C}",
    "err":       f"bold {ERR_C}",
    "dim":       DIM_C,
    "diff.add":  OK_C,
    "diff.del":  ERR_C,
})
```

使用：

```python
console = Console(theme=THEME)
console.print("[brand]One Cedric[/]  [dim]v0.17.0[/]")
console.print("  [err]✗ 命令被拒绝[/]")
```

好处：暗色/亮色主题、无障碍配色、WebUI 的主题同步，都只需要改这一张表。

## 4.4 第二层：流式渲染不闪烁

模型一边生成，界面一边显示。直觉写法是每收到一个 token 就 `print(token, end="")`，但这样会带来两个问题：

1. Markdown 无法增量渲染（表格、代码块会被打散）；
2. 已经打印的内容无法"重排"，代码块高亮也无法回填。

Rich 的解法是 `Live`：维护一个"可刷新的渲染区域"，每次用完整的累积文本重新渲染。

```python
with Live(ui.get_spinner("_thinking"),
          console=self.console,
          refresh_per_second=20,
          transient=True) as live:
    ...
    content_buf.append(data)
    text = "".join(content_buf)
    now = time.time()
    if now - last_render >= 0.12 and text.strip():
        last_render = now
        live.update(_md_with_cursor(text))
```

三个工程细节：

- **0.12 秒节流**（约 8 帧/秒）。Markdown 重排是 CPU 密集操作；每个 token 都重排会让终端卡顿，甚至拖慢网络读取；
- **`transient=True`**：Live 结束后擦除临时内容，避免"闪一下再被最终答案覆盖"；
- **`refresh_per_second=20`**：Spinner 的刷新率，和 Markdown 刷新解耦。

> ⚠️ **踩坑**：`Live` 活动期间不要直接 `print()`，否则输出会插进渲染区域，造成错位。需要"固定打印"时用 `live.console.print(...)`（Rich 会把这条消息暂时移到 Live 区域上方），或者先退出 `with`。One Cedric 在"思考结束、正文开始"的瞬间，就是用 `live.console.print(Panel(...))` 把思考过程固定到上方的。

## 4.5 第三层：思考过程与回答，两个通道

Reasoning 模型（DeepSeek-R1、DeepSeek v4.1-flash 等）会同时输出两类内容：

- `reasoning_content`：思考过程；
- `content`：最终回答。

如果混在一起显示，用户会看到一大段"内心戏"，找不到答案。One Cedric 的做法是**双通道 + 状态机**：

```python
phase = "reasoning"          # 初始阶段
reasoning_buf: list[str] = []
content_buf: list[str] = []

for kind, data in self._stream_chat(api_messages):
    if kind == "reasoning":
        reasoning_buf.append(data)
        if self.show_reasoning and phase == "reasoning":
            acc = "".join(reasoning_buf)
            preview = acc if len(acc) <= 800 else "…" + acc[-800:]
            live.update(Panel(
                Text(preview, style="dim italic"),
                title=f"[dim]思考中… {len(acc)} 字[/]",
                border_style=DIM_C,
                box=box.ROUNDED,
                expand=False,
            ))

    elif kind == "content":
        if phase == "reasoning":
            phase = "content"
            # 思考结束：把完整过程固定打在 Live 上方
            if self.show_reasoning and reasoning_buf and not reasoning_frozen:
                reasoning_frozen = True
                acc = "".join(reasoning_buf)
                live.console.print(Panel(
                    Text(acc, style="dim italic"),
                    title=f"[dim]思考过程（{len(acc)} 字）[/]",
                    ...
                ))
        content_buf.append(data)
        ...
```

这个设计解决了三个真实问题：

1. **思考太长会刷屏**：预览只显示最后 800 字（"…"+尾部），完整内容在思考结束时一次性固定；
2. **思考结束后不能消失**：早期的实现是思考面板一消失就没了，用户抱怨"显示完思考过程后就隐藏了"（2026-09-27 16:12 的原话）。现在用 `reasoning_frozen` 标记，把完整面板打印到滚动区；
3. **模型只思考不回答**：如果整轮只有 `reasoning` 没有 `content`，界面会显示警告面板："模型只输出了思考过程，未生成最终回答。可能是 max_tokens 太小，或提示词触发了纯推理。"

> 💡 **设计笔记**：流式 UI 的本质是"**用状态机管理一段会变形的文本**"。阶段（reasoning/content）、缓冲区、冻结标记、节流时间戳——这四个变量足以覆盖绝大多数情况。

## 4.6 那个著名的 Spinner 报错

📅 **2026-09-19 16:06**（对话 #67，用户粘贴）

```text
File "F:\one-copilot\one_copilot\ui.py", line 661, in get_spinner
    return Spinner("line", text=f"[dim]{verb}中…[/]", style=DIM_C,
                   frames=[f.strip() for f in cfg["frames"]],
                   interval=cfg["interval"])
TypeError: Spinner.__init__() got an unexpected keyword argument 'frames'
```

### 原因

作者想给"思考中"也做一套自定义动画，于是把工具动画的配置（`frames` / `interval`）直接传给了 Rich 的 `Spinner`。但 Rich 的 `Spinner` 构造函数**不接受** `frames` 参数——它的动画帧是由 `spinner` 名称（如 `"dots"`、`"line"`）在内部查表决定的，自定义帧要走 `Spinner` 之外的机制。

### 修法

v1 的 `ui.py` 最终改成：

```python
def get_spinner(name: str = "_thinking") -> Spinner:
    verb = SPINNER_VERBS.get(name, "思考")
    return Spinner("dots", text=f"[dim]{verb}中…[/]", style=DIM_C)
```

也就是：**思考动画用 Rich 内置的 `dots`**，把"每个工具不同动画"的需求交给另一套机制（下一节）。这是一个很典型的取舍——不是所有东西都要自定义，内置的够好就用内置的。

> ⚠️ **踩坑提醒**：Rich 的 API 在不同版本间有过变化（`Spinner` 的参数、`Live` 的行为）。**锁版本**（`rich>=15.0.0`）和**写一个最小复现**，比对着文档猜快得多。

## 4.7 每个工具都有自己的动画

用户的要求是"执行每一个 tool 都有不同的执行动画"。实现方式不是给每个工具写一个类，而是**一张数据表 + 一个通用渲染器**：

```python
# ui.py（v1）
TOOL_FRAMES = {
    "read_file": {"frames": ["▱▱▱▱▱", "▰▱▱▱▱", "▰▰▱▱▱", "▰▰▰▱▱",
                             "▰▰▰▰▱", "▰▰▰▰▰"], "interval": 0.08},
    "bash":      {"frames": ["▰▱▱▱    ", "▱▰▱▱    ", "▱▱▰▱    ",
                             "▱▱▱▰    ", "▱▱▱▱▰   "], "interval": 0.1},
    "web_search":{"frames": ["◜       ", "◠       ", "◝       ",
                             "◞       ", "◡       "], "interval": 0.12},
    "write_file":{"frames": ["✏       ", " ✏      ", "  ✏     ",
                             "   ✏    ", "    ✏   "], "interval": 0.1},
    # …… 几十个工具
}
```

通用渲染器（`ui.ToolStatus` 的思路）：

```python
class ToolStatus:
    def __init__(self, console, name, target):
        self.frames = TOOL_FRAMES.get(name, DEFAULT_FRAMES)["frames"]
        self.interval = TOOL_FRAMES.get(name, DEFAULT_FRAMES)["interval"]
        ...

    def __enter__(self):
        # 启动一个线程或 Live，按 interval 轮播 self.frames
        ...

    def __exit__(self, *exc):
        # 停止动画，打印结果摘要（耗时、行数、退出码）
        ...
```

这样新增一个工具时，动画是"可选的"：没有配置就走默认帧，配置了就有专属动画。**动画是数据，不是代码**——和第 5 章工具 schema 的思想一脉相承。

## 4.8 让界面有"人味"：光标、状态栏、小狗

### 光标

模型输出末尾会跟一个橙色 `✦`，提示"还在生成"：

```python
CURSOR_MARK = "✦"
CURSOR_STYLE = "bold #ff9e2c"
```

流式结束后去掉光标，换成正常结尾。这是很小的细节，但让"正在打字"和"已经说完"有了明确的视觉区分。

### 状态栏与 token 统计

📅 **2026-09-27 16:15** · 用户："能不能在每次对话时显示当前的 token 数量和速度"

`_print_turn_metrics()` 每轮结束打印一行：

```text
│ 输入 3.2k tok · 缓存 1.1k tok · 输出 412 tok · 28.4 tok/s · 耗时 14.6s · 首字 1.2s · 本轮 $0.00042
```

如果 API 没返回真实 usage，就用字符数除以 2.5 估算，并在数字前加 `~` 标记——**永远不要假装估算是真实值**。

### 像素小狗

📅 **2026-10-01 19:56** · 用户："搜索框四周白框，左侧有一个小狗，像素级输出就行……有任务进行时红色，需要确认时黄色，空闲时绿色。"

`ui/dog.py` 用几行字符画实现了这个需求：

```python
DEFAULT_FRAMES = {
    "idle": {
        "color": "green",
        "frames": [
            "▐•ᴥ•▌ ",
            "▐•ᴥ•▌~",
            "▐-ᴥ-▌~",   # 眨眼
            "▐•ᴥ•▌~",
            "▐•ᴥ•▌ ",
            "~▐•ᴥ•▌",   # 摇尾巴
            "▐-ᴥ-▌~",
            "▐•ᴥ•▌ ",
        ],
        "interval": 0.35,
    },
    "busy":    {"color": "red",    "frames": ["▐●ᴥ●▌ ", ...], "interval": 0.12},
    "confirm": {"color": "yellow", "frames": ["▐◉ᴥ◉▌ ?", ...], "interval": 0.3},
    "error":   {"color": "red",    "frames": ["▐✕ᴥ✕▌ ", ...], "interval": 0.6},
}
```

三个状态对应三种颜色：**绿=空闲、红=忙、黄=等你确认**。这比一行"Loading..."有用得多——用户扫一眼就知道该不该理它。小狗的帧还支持用户通过 `~/.one-cedric/dog.toml` 自定义。

## 4.9 Windows 终端的两件小事

### 中文乱码与框线崩溃

Windows 默认代码页是 GBK，Rich 的框线字符、破折号、emoji 可能触发 `UnicodeEncodeError`。`cli.py` 在启动时把标准输出切到 UTF-8：

```python
def _ensure_utf8_stdio() -> None:
    import sys
    for stream in (sys.stdout, sys.stderr):
        try:
            enc = (getattr(stream, "encoding", "") or "").lower()
            if enc.replace("-", "").replace("_", "") != "utf8":
                stream.reconfigure(encoding="utf-8", errors="replace")
        except Exception:
            pass
```

### 让旧终端支持 ANSI

Windows 10 之前的控制台需要显式开启虚拟终端序列：

```python
from .winvt import enable_vt_terminal
enable_vt_terminal()
```

这两件事都不"优雅"，但没有它们，用户看到的就是乱码和没有颜色的方块。

## 4.10 界面设计的取舍

回头看，这一章的所有实现都遵循同一个原则：**UI 是状态的函数**。

| 状态 | 视觉表现 |
| --- | --- |
| 正在思考 | 灰色 spinner + "思考中… N 字"预览 |
| 思考结束 | 完整思考面板固定到上方（dim italic） |
| 正在回答 | Markdown 流式渲染 + 橙色光标 |
| 正在执行工具 | 工具专属动画 + 参数摘要 |
| 等待确认 | 黄色小狗 + 5 选项菜单 |
| 出错 | 红色 `✗` + 可行动的提示 |
| 空闲 | 绿色小狗 + 底部状态栏 |

只要状态定义清楚，UI 代码就是"状态 → 组件"的映射。反过来，如果你发现 UI 代码里到处是 `if`，通常意味着状态没有定义清楚。

## 🔧 自己动手

1. 用 Rich 写一个最小流式渲染器：模拟每 50ms 到达一个字符，用 `Live` + Markdown 显示；
2. 给 `read_file` / `bash` 各配一组 5 帧的动画，做一个 `ToolStatus` 上下文管理器；
3. 实现"思考中"与"回答中"的双通道显示：思考时显示最后 200 字，正文开始时把完整思考打印到上方；
4. 给状态栏加 token 统计（先用字符数估算），观察长对话下的数字变化。

---

[← 上一章：03 拆包与工程化](03-project-refactor) ｜ [返回目录](Home) ｜ [下一章：05 工具系统 →](05-tool-system)
