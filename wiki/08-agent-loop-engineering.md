# 08 · Agent 循环工程：让 250 个工具跑得又快又稳

## 8.1 从"能跑"到"跑得好"

📅 **开发日志 · 2026-09-25 / 09-26**

> **09-25 21:52 用户**：你搜过后需要浏览网页
>
> **09-26 09:12 用户**：修复 websearch 中没有 readability 和语义搜索，优化上下文消耗
>
> **09-27 16:15 用户**：能不能在每次对话时显示当前的 token 数量和速度

三句话对应三个工程问题：

1. **多步任务**："先搜索、再打开网页"不是一次工具调用能完成的，循环必须能连续跑；
2. **上下文预算**：搜索结果、网页正文、工具输出会迅速撑爆窗口，必须有缓存、压缩、截断；
3. **可观测性**：用户要知道这一轮花了多少 token、多快、哪一步最慢。

到 10 月，工具数达到 250，模型一轮可能一口气请求 5 个工具。循环如果不做批处理，用户就要看 5 次串行的加载动画；如果不做去重，模型会把同一个 `read_file` 调用三遍。这一章讲 `agent_turn` 怎么把这些事摆平。

## 8.2 最小循环 vs 工程化循环

第 2 章的循环只有十几行：

```python
for _ in range(max_steps):
    msg = chat(messages, tools)
    messages.append(msg)
    if not msg.tool_calls:
        return msg.content
    for tc in msg.tool_calls:
        result = dispatch(tc)
        messages.append({"role": "tool", ...})
```

工程化之后，同样的骨架膨胀到约 550 行（`core.py:6307-6878`），但**结构没变**。多出来的全是"边界情况"：

```text
agent_turn
├── 准备：用户消息、字符统计、plan 模式换 system prompt
├── for step in range(max_steps):            ← 步数上限（默认 8）
│   ├── 流式请求 + UI 双通道渲染
│   ├── final：DeepSeek 兜底 / 文本工具调用恢复
│   ├── 插队检查（用户中途输入）
│   ├── 分类任务：error / dup / write / readonly
│   ├── 合批：连续 readonly 合成一批
│   ├── 执行：write 串行确认；readonly 并行 + 缓存
│   └── 追加 tool 消息，进入下一轮
├── 达到 max_steps：返回"请简化问题"
└── finally：token/成本统计、标题生成、会话落盘
```

> 💡 **设计笔记**：判断一个 agent 循环是否"工程化"，不看它有多少行，而看它**显式处理了多少种失败**：模型不调工具、调错工具、重复调、参数缺失、工具抛异常、用户中途插话、上下文超限、达到步数上限。每一种失败都对应循环里的一段代码。

## 8.3 循环的骨架

```python
def agent_turn(self, prompt: str) -> str:
    self._snapshot_taken_this_turn = False
    t0 = time.time()
    self.messages.append({"role": "user", "content": prompt})

    turn_input_chars = sum(len(m.get("content") or "") for m in self.messages)
    turn_output_chars = 0
    turn_first_token_at = None
    turn_first_content_at = None

    if self.plan_mode:
        api_messages = ([{"role": "system", "content": PLAN_SYSTEM_PROMPT}]
                        + self.messages[1:])
    else:
        api_messages = list(self.messages)

    try:
        for _step in range(self.max_steps):
            ...
        return "(已达到最大工具调用步数，请简化问题后重试)"
    finally:
        ...  # 统计、标题、保存
```

四个准备动作：

1. **`_snapshot_taken_this_turn = False`**：本轮的 git 快照标记复位（第 7 章）；
2. **字符统计**：`turn_input_chars` 在循环前算一次（上下文大小），`turn_output_chars` 每轮累加；
3. **plan 模式换 system prompt**：`PLAN_SYSTEM_PROMPT` 明确要求"只能使用只读工具"；
4. **`t0`**：整轮耗时，包含所有工具调用，不只是模型生成。

`max_steps` 默认是 **8**（`config.py:323`，CLI `--max-steps` 可改）。注意它限制的是**模型轮次**，不是工具调用数——一轮里并行 4 个只读工具，只算 1 步。

## 8.4 分类：把模型的一轮请求拆成四类

模型返回的 `tool_calls` 是一个数组，但它们的"性质"不同。`core.py:6549` 开始做分类：

```python
seen_calls: set = set()
tasks: list = []

for tc in assistant_msg["tool_calls"]:
    name, args, nerr = prepare_tool_call(tc)

    if nerr or not name:
        tasks.append({... "kind": "error"})
        continue

    call_sig = (name, json.dumps(args, sort_keys=True, ensure_ascii=False))
    if call_sig in seen_calls:
        tasks.append({... "kind": "dup"})
        continue
    seen_calls.add(call_sig)

    raw_name = (tc.get("function") or {}).get("name", "")
    if raw_name and raw_name != name:
        self.messages.append({
            "role": "system",
            "content": (f"[One Cedric] 你刚才调用了 '{raw_name}'，"
                        f"标准工具名是 '{name}'。请下次直接使用标准名。"),
        })

    has_missing = False
    for t in self._current_tool_schemas:
        if t["function"]["name"] == name:
            req = t["function"].get("parameters", {}).get("required", [])
            miss = [r for r in req if r not in args or args[r] in ("", None)]
            if miss:
                has_missing = True
                tasks.append({... "err": f"缺少必需参数 {', '.join(miss)}。"})
            break
    if has_missing:
        continue

    tc["function"]["name"] = name
    tc["function"]["arguments"] = json.dumps(args, ensure_ascii=False)

    is_write = self._classify_is_write(name, args)
    tasks.append({... "kind": "write" if is_write else "readonly"})
```

四类任务：

| kind | 触发条件 | 处理方式 |
| --- | --- | --- |
| `error` | 工具名无法识别 / 缺必需参数 | 直接生成 `ERROR:` tool 消息 |
| `dup` | 本轮内同名 + 同参数已出现 | 直接生成"已跳过重复调用" |
| `write` | 写工具 / 有副作用 | 串行，走确认流程（第 7 章） |
| `readonly` | 其余 | 合批，可并行，可缓存 |

三个容易被忽略的细节：

**① 去重签名用规范化后的参数**，不是原始 JSON 字符串：

```python
call_sig = (name, json.dumps(args, sort_keys=True, ensure_ascii=False))
```

`{"a":1,"b":2}` 和 `{"b":2,"a":1}` 是同一个调用——`sort_keys=True` 才能识别。如果没有这一步，模型换个键序就能绕过去重。

**② 标准名纠正会"教育"模型**：当模型调用别名（`readfile`）时，除了执行，还追加一条 system 消息告诉它标准名。这不是为了这一次，而是为了**下一轮**——小模型会记住上下文里的纠正。

**③ 缺参数不执行**：返回 `ERROR: 缺少必需参数 path。请重新调用 read_file 并补全。` 错误信息**明确告诉模型下一步做什么**，比"invalid arguments"有效得多。

## 8.5 合批：连续只读的合成一批

分类之后，按"顺序"合批：

```python
batches: list = []
for t in tasks:
    if t["kind"] == "readonly":
        if batches and batches[-1]["kind"] == "readonly":
            batches[-1]["items"].append(t)
        else:
            batches.append({"kind": "readonly", "items": [t]})
    else:
        batches.append({"kind": t["kind"], "items": [t]})
```

规则只有一条：**连续的只读任务合并，遇到写任务就断开**。

为什么写任务不能和只读并行？

1. 写操作需要用户确认（可能弹出 5 选项菜单），并行弹多个确认框会互相打架；
2. 写操作会改变文件，后续只读可能读到旧内容——顺序必须严格；
3. 写操作失败时，后面的只读可能基于错误的前提。

例：

```text
模型请求：[read A, read B, write C, read D, read E]
合批结果：[[read A, read B], [write C], [read D, read E]]
       并行 2 个         串行确认      并行 2 个
```

> ⚠️ **踩坑**：合批的依据是**模型给出的顺序**，不是工具依赖图。模型如果先 `read_file` 再 `write_file` 再 `read_file`，第二批读会读到新内容——这是正确行为。反过来，如果模型先写后读，读到的也是新内容，同样正确。**不要自作聪明重排顺序**：模型可能依赖"写之前的旧内容"做对比。

## 8.6 并行只读：4 个线程、结果保序

一批只读任务用 `ThreadPoolExecutor` 并行（`core.py:6805`）：

```python
with ThreadPoolExecutor(max_workers=min(4, len(items))) as pool:
    futures = {}
    for t in items:
        tc_id = t["tc"]["id"]
        tid, _ = tid_map[tc_id]
        fut = pool.submit(_run_one, t, tid)
        futures[fut] = tc_id

    for fut in as_completed(futures):
        tc_id = futures[fut]
        result = fut.result()
        results_by_id[tc_id] = (result, t)
```

要点：

1. **`max_workers=min(4, len(items))`**：上限 4。工具大多受 I/O 限制（HTTP、磁盘），4 个足够；再多会打爆某些 API 的速率限制；
2. **`as_completed` 乱序完成**：谁先完成先更新进度条；
3. **结果按 `tool_call_id` 存起来**：最后仍按模型请求的顺序写回 `messages`：

```python
for t in items:
    tc_id = t["tc"]["id"]
    result, task = results_by_id[tc_id]
    self.messages.append({
        "role": "tool", "tool_call_id": tc_id, "content": result,
    })
```

**消息顺序必须与 tool_calls 顺序一致**——这是 OpenAI 协议的要求，乱序会导致 400。并行的是执行，不是消息。

4. **单个工具失败不影响整批**：`_run_one` 内部 `try/except`，异常变成 `ERROR: ...` 字符串；`as_completed` 里还有一层 `fut.result()` 的兜底；
5. **UI 降级**：如果 `ProgressMatrix` 导入失败，整批退化为串行执行（`if ProgressMatrix is None`）——花哨的 UI 不能成为功能的前置依赖。

> 💡 **设计笔记**：为什么用线程而不是 asyncio？因为 250 个工具的实现全是同步阻塞代码（`requests`、`subprocess`、`PIL`）。改成 async 要重写所有工具，收益只有"省几个线程"。**并发模型要跟着既有代码走**，不是越新越好。

## 8.7 缓存：只读工具的结果复用

### ToolCache 的结构

`cache.py` 是一个 **LRU + TTL** 的内存缓存：

```python
class ToolCache:
    """LRU + TTL 内存缓存。"""

    def __init__(self, max_entries: int = 200, ttl: int = 300):
        ...
        self._data: dict = {}
        self._order: list = []
        self._lock = threading.Lock()
        self._hits = 0
        self._misses = 0

    def _key(self, root: Path, name: str, args: dict) -> str:
        payload = {"root": str(root), "name": name, "args": args}
        s = json.dumps(payload, sort_keys=True, ensure_ascii=False)
        return hashlib.sha1(s.encode("utf-8")).hexdigest()
```

三个关键设计：

- **key 包含 `root`**：同一个 `read_file` 在 `F:\a` 和 `F:\b` 下是不同的结果；
- **key 用 `sort_keys=True`**：参数顺序不影响命中；
- **`threading.Lock`**：并行批次里多个线程同时读写，必须加锁。

`get` 里检查 TTL 并更新 LRU 顺序；`put` 里淘汰最旧的条目。`stats()` 给出命中率，`/stats` 命令会展示。

### 哪些工具可缓存

只有 `CACHEABLE_TOOLS`（42 个）会走缓存，原则是：**纯查询、结果不随时间变化、参数决定结果**。

```python
CACHEABLE_TOOLS = {
    "read_file", "list_files", "file_info", "glob",
    "json_query", "env_info", "csv_query", "calculate",
    "hash_file", "markdown_toc",
    "lsp_hover", "lsp_definition", "lsp_references", "lsp_symbols",
    "hash_text", "hash_data", "random_bytes",
    ...
}
```

`web_search` **不在**缓存里：同一关键词十分钟后结果可能不同。`read_file` 在，但写入后必须失效——所以 `_execute_write` 里有一行：

```python
self.tool_cache.clear()
```

> ⚠️ **踩坑**：缓存失效比缓存命中更难写对。任何写操作（包括用户手动改文件、git checkout）都可能让 `read_file` 的缓存过期。One Cedric 的取舍是**写操作后全清**——简单、保守、不会读到脏数据。200 条缓存清空的代价，远小于"模型基于旧文件内容改代码"的代价。

## 8.8 插队：用户不等模型说完

📅 **10-01 20:58 用户**：不要让用输入 y 或什么什么的，让他们用上下左右选择

插队（interject）是另一个交互细节：模型正在流式输出或执行工具时，用户可以直接输入新消息，**不打断当前轮次的数据结构**，而是在下一轮之前插入。

实现（`core.py:6349`）：

```python
if self.repl_ui is not None:
    self.repl_ui.poll_during_turn()
    if self.repl_ui.interject:
        break
```

流式循环里每次收到 chunk 都非阻塞地检查键盘；用户确认插队就 `break` 出流。之后：

```python
if self.repl_ui is not None and self.repl_ui.interject:
    text = self.repl_ui.take_interject()
    partial = "".join(content_buf).strip()
    if partial:
        # 保留已经输出的部分回答，避免上下文断裂
        self.messages.append({"role": "assistant", "content": partial})
    self.messages.append({"role": "user", "content": text})
    api_messages = list(self.messages)
    self.console.print(f"\n[accent]⤵ 已插队，模型将立刻处理：[/]{text}\n")
    continue
```

三个决策：

1. **保留半截回答**：把已经生成的部分作为 assistant 消息写入历史，否则模型会"失忆"；
2. **作为 user 消息插入**：模型下一轮会把它当作新指令；
3. **`continue` 而不是 `break`**：不结束 `agent_turn`，只是重新开始下一轮请求——用户的插队不会浪费已经执行的工具结果。

> 💡 **设计笔记**：插队的本质是"**在流式输出和工具执行之间找一个安全点**"。不能在工具执行到一半时插——那会留下半完成的副作用；也不能等整轮结束——那用户要等很久。One Cedric 选择在"每个 chunk 到达时"检查，因为这是唯一一个"没有副作用正在进行"的时刻。

## 8.9 计划模式：先看地图再开车

计划模式（`/plan`）把"调研"和"执行"分成两个阶段：

```python
PLAN_SYSTEM_PROMPT = r"""你现在处于「计划模式」。只能使用只读工具，不能修改任何文件。

## 任务

1. 用只读工具调研清楚现状。
2. 调研完成后，输出一份可执行的计划。
3. 计划必须是 Markdown 有序列表（1. 2. 3.），每条一个原子动作。
4. 每条包含：做什么 + 涉及哪个文件。
5. 输出计划后停下，等用户批准。
"""
```

技术实现只有两处：

- **工具集换成 `TOOLS_READONLY`**（`_stream_chat` 里 `TOOLS_READONLY if self.plan_mode else _ALL_TOOLS`）；
- **system prompt 换成 `PLAN_SYSTEM_PROMPT`**（`agent_turn` 开头）。

批准后的执行阶段，把计划项变成 checklist 塞回 system 消息：

```python
checklist = "\n".join(f"- [ ] {it}" for it in items)
self.messages.append({
    "role": "system",
    "content": ("[One Cedric] 用户已确认以下计划，开始执行。"
                "每完成一项后，在回复里用 `- [x]` 标注该项。\n\n"
                + checklist),
})
```

这样模型在执行时有一个"进度条"，不会做着做着跑偏。

## 8.10 上下文压缩：`/compact`

当对话变长，历史会挤占窗口。`_compact` 的做法是：**保留最近 N 轮原文，把更早的历史压成一条结构化摘要**。

```python
split = _find_split_index(self.messages, keep_turns)
old_msgs = self.messages[1:split]          # 保留 system[0]
...
summary_messages = [
    {"role": "system", "content": COMPACT_PROMPT},
    {"role": "user", "content": "对话历史如下：\n\n"
                                + self._render_for_summary(old_msgs)},
]
summary = self._once_chat(summary_messages, temperature=0.2)
...
summary_msg = {
    "role": "system",
    "content": "[历史摘要 · 由 /compact 生成]\n\n" + summary,
}
kept_msgs = self.messages[split:]
new_messages = [self.system_msg, summary_msg] + kept_msgs
```

摘要提示词要求保留五类信息（`config.py:421`）：

```text
1. 用户的原始目标、约束和偏好
2. 已完成的所有文件改动：文件路径 + 关键变化
3. 关键决策及其原因
4. 尚未完成的待办事项
5. 通过工具发现的重要事实
```

输出固定为五段：目标 / 已完成 / 关键决策 / 关键上下文 / 待办。

`_maybe_auto_compact` 在每轮结束后估算 token，超过阈值时**提示**用户运行 `/compact`（不自动压缩——压缩会丢信息，应该由用户决定）。

> ⚠️ **踩坑**：压缩的难点不是"总结"，是"**不丢关键事实**"。文件路径、函数名、错误信息这类"硬事实"一旦被摘要成"修改了一些文件"，模型后续就无法继续工作。所以 `COMPACT_PROMPT` 用列举式指令，而不是"请总结一下"。

## 8.11 收尾：finally 里的事，一件都不能少

```python
finally:
    elapsed = time.time() - t0
    self._bump("turns")
    self.stats.setdefault("turn_durations", []).append(round(elapsed, 2))
    if len(self.stats["turn_durations"]) > 100:
        self.stats["turn_durations"] = self.stats["turn_durations"][-100:]
    try:
        self._print_turn_metrics(...)
    except Exception:
        pass
    self._maybe_generate_title()
    self._save_session()
```

即使 `agent_turn` 里抛异常（API 断开、用户 Ctrl+C），这些也会执行：

- **统计**：轮次、耗时（只留最近 100 轮）；
- **token/成本**：`_print_turn_metrics` 自己还有一层 `try`，统计失败不能影响主流程；
- **标题**：首轮结束后用 `TITLE_PROMPT` 生成会话标题（失败也不影响）；
- **落盘**：`_save_session` 写会话文件，用户下次可以 `/resume`。

> 💡 **设计笔记**：把"收尾"放在 `finally` 里，是 agent 循环最重要的一条工程习惯。用户按 Ctrl+C 是常态，不是异常——**不能因为用户中断，就丢掉已经产生的对话和统计**。

## 8.12 循环工程的十条经验

1. **步数限制按"模型轮次"算**，不按工具数——一轮并行 4 个工具是高效的，不该被惩罚；
2. **先去重再执行**：签要用规范化参数；
3. **连续只读合批**，写操作严格串行；
4. **并行执行、保序写回**：消息顺序是协议要求；
5. **单个工具失败不影响整批**：异常一律转成 `ERROR:` 字符串；
6. **缓存 key 要含 root**，写后全清；
7. **错误信息要可行动**："缺少参数 path" 比 "invalid request" 有用；
8. **插队只在安全点发生**：chunk 边界，不是工具执行中；
9. **计划模式靠工具集 + prompt 双保险**，不能只靠提示词；
10. **收尾放 finally**：统计、标题、落盘，一个都不能少。

## 🔧 自己动手

1. **给最小 agent 加合批**：把 `[read A, read B, write C]` 拆成两批，只读并行、写串行；
2. **实现 `ToolCache`**：LRU + TTL + 线程锁，写一个 100 次随机读写的测试，打印命中率；
3. **做一个"插队"原型**：用 `input()` 的非阻塞版本（Windows 可用 `msvcrt.kbhit`）在流式输出中检测按键；
4. **写一个 `/compact`**：保留最近 3 轮，把更早的历史用一次非流式请求总结成五段式摘要；
5. **故意让工具抛异常**：验证并行批次里一个失败、其他仍能完成，且消息顺序正确。

---

[← 上一章：07 权限与安全](07-permissions-safety) ｜ [返回目录](Home) ｜ [下一章：09 十条需求与批量交付 →](09-mega-batch)
