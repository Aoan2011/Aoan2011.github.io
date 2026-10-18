# 06 · 流式与推理：让打字机和"脑内小剧场"都看得见

## 6.1 "不支持流式输出？"

📅 **开发日志 · 2026-09-19**（对话 #67）

> **16:10 用户**：不支持流式输出？

此时的 One Copilot 已经能对话、能调工具，但回复是"等全部生成完，一次性刷出来"。用户看到的是一段长时间的空白，然后突然出现整段文字——在本地小模型上，这个等待可能长达几十秒。

> 📅 **09-19 16:37 用户**（粘贴模型输出）：
>
> ```text
> 好的，我们开始创建文件并写入一些示例内容。我将先创建一个空文件，然后再写入一些示例代码。
>
> 首先，我将创建 F:\testai.py 文件。
>
>  {"name": "write_file", "arguments": {"path": "F:\\testai.py", "content": ""}}
> ```

同一个下午，两个问题同时暴露：**输出不是流式的**，以及**模型把工具调用"写"成了普通文本**。前者是体验问题，后者是正确性问题。

四天后，Reasoning 模型来了：

> 📅 **09-25 17:32 用户**：额，我们应该增加对于 resoning 模型的支持如 DeepSeek v4.1flash
>
> **17:38 用户**：是 DeepSeek API 的那个
>
> **17:40 用户**：还要支持用户自定义 API 地址和 APIkey

再两天后，界面细节：

> 📅 **09-27 16:10 用户**：ok，能不能显示思考过程
>
> **16:12 用户**：目前的话是他显示完思考过程后就隐藏了
>
> **16:15 用户**：能不能在每次对话时显示当前的 token 数量和速度
>
> **16:20 用户**：ok，再加上思考模式调整，minimal，low，medium，max，xhigh，ultra

> 📅 **10-01 14:55 用户**：行，思考模式也要，到 ultra 时变成蓝紫渐变色，有粒子飘动

这一章按数据流顺序讲：**请求怎么发 → SSE 怎么读 → 工具调用怎么拼 → 思考怎么分离 → 档位怎么调 → token 怎么算**。

> 📖 **阅读提示（请认真对待）**：这一章是全书中**最依赖外部文档**的一章——下面几乎每一行代码，都是在"翻译"服务商的 API 文档。请在继续之前，先把对应文档打开，对照着读：
>
> - **OpenAI 兼容接口**：[Chat Completions API 文档](https://platform.openai.com/docs/api-reference/chat)（重点：`stream`、`stream_options.include_usage`，以及响应里的 `delta` / `tool_calls` / `finish_reason` / `usage` 各是什么形状）；
> - **DeepSeek**：[api-docs.deepseek.com](https://api-docs.deepseek.com/)（重点：`reasoning_content` 与 `deepseek-reasoner` 的用法和限制）；
> - **你实际用的服务商**（通义、Kimi、GLM、Ollama……）：找它的"OpenAI 兼容 / chat completions / 流式"章节，逐字段对照。
>
> 为什么必须看文档：**字段名与服务商强相关，而且一直在变**。本章代码是"照着文档抄 + 处理现实世界的脏数据"的产物。看懂文档，出错时你才能判断"是代码错了，还是文档变了，还是这个服务商就是没实现这个字段"；只背代码，换个服务商就会立刻卡住。

## 6.2 SSE：流式输出的底层协议

OpenAI 兼容接口的流式响应是 [Server-Sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)。一次请求的响应体长这样（典型形状）：

```text
data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{"role":"assistant","content":""},"finish_reason":null}]}

data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{"content":"你"},"finish_reason":null}]}

data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{"content":"好"},"finish_reason":null}]}

data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{"tool_calls":[{"index":0,"id":"call_abc","type":"function","function":{"name":"read_file","arguments":""}}]},"finish_reason":null}]}

data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{"tool_calls":[{"index":0,"function":{"arguments":"{\"path\":"}}]},"finish_reason":null}]}

data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{"tool_calls":[{"index":0,"function":{"arguments":"\"a.txt\"}"}}]},"finish_reason":null}]}

data: {"id":"chatcmpl-1","choices":[{"index":0,"delta":{},"finish_reason":"tool_calls"}]}

data: [DONE]
```

几个必须知道的点：

1. 每一行以 `data: ` 开头，空行是分隔符（`requests.iter_lines()` 会处理）；
2. 最后一个数据是字面量 `[DONE]`，**不是 JSON**；
3. `delta` 是"增量"，不是完整消息——`content` 要拼接，`tool_calls` 更要拼接；
4. `usage` 默认不返回。要拿 token 统计，得在请求里加 `stream_options: {"include_usage": true}`。

## 6.3 `_stream_chat`：一个生成器搞定请求与解析

核心函数在 `core.py:863`（v3）。它是个**生成器**，`yield (kind, data)` 三元组：

- `("reasoning", text)`：思考增量；
- `("content", text)`：回答增量；
- `("final", {...})`：一轮结束，给出完整结果。

### 请求部分

```python
payload = {
    "model": self.model, "messages": patched_messages,
    "tools": tools_list,
    "tool_choice": "auto", "stream": True,
    "temperature": ov.get("temperature", self.temperature),
    "stream_options": {"include_usage": True},
}
payload.update(self._think_params())
```

四个细节：

1. **`stream: True`**：这是"流式"的开关；
2. **`stream_options.include_usage`**：让服务端在最后一个 chunk 里带 `usage`；
3. **`tool_choice: "auto"`**：让模型自己决定调不调工具。设成 `"required"` 会强迫它每轮都调，设成具体函数则锁死——都不适合通用 agent；
4. **`patched_messages`**：system prompt 被"二次加工"——注入思考模式提示 + 记忆块（`memory.to_prompt_block()`）。注意只改发给 API 的副本，不改 `self.messages` 里的原始历史。

### 解析部分

```python
for raw in resp.iter_lines():
    if not raw:
        continue
    line = raw.decode("utf-8", errors="replace").strip()
    if not line.startswith("data:"):
        continue
    data = line[5:].strip()
    if data == "[DONE]":
        break
    try:
        chunk = json.loads(data)
    except json.JSONDecodeError:
        continue
```

`iter_lines()` 是阻塞迭代器：服务端发一行，这里读一行——这就是"流式"在客户端的样子。三个防御：

- `decode(..., errors="replace")`：某些代理会切断多字节 UTF-8 字符，宁可出现一个 `�`，也不要抛异常；
- `startswith("data:")` 之外的行走 `continue`：SSE 允许 `event:`、`id:`、注释行（`:` 开头的心跳）；
- `json.loads` 失败也 `continue`：有的服务端会插入非 JSON 的保活内容。

> ⚠️ **踩坑**：不要用 `resp.text` 或 `resp.json()` 读流式响应——它们会等整个响应结束，等于把流式退化成非流式。必须 `stream=True` + `iter_lines()`（或 `iter_content`）。

### usage 的捕获

```python
usage = chunk.get("usage")
if usage:
    self._last_usage = usage
```

`usage` 通常在最后一个 chunk 里，此时 `choices` 是空数组——所以**必须在 `if not choices: continue` 之前处理 usage**。这个顺序错过一次：早期版本先 `continue` 再看 usage，结果 token 统计永远是估算值。

### 结束时的汇总

```python
finally:
    resp.close()

yield ("final", {
    "content": "".join(content_parts),
    "reasoning": "".join(reasoning_parts),
    "tool_calls": [tool_calls[i] for i in sorted(tool_calls)],
})
```

`finally: resp.close()` 很重要：用户按 Ctrl+C 中断、或者上层提前 `break` 时，连接必须释放，否则连接池会泄漏。

## 6.4 最难的一段：tool_calls 增量重组

非流式响应里，工具调用是一个完整对象：

```json
{"id": "call_abc", "type": "function",
 "function": {"name": "read_file", "arguments": "{\"path\": \"a.txt\"}"}}
```

流式响应里，它被**切成任意碎片**，而且：

- `id` 只在第一片出现；
- `name` 可能分几片到达；
- `arguments` 是 JSON 字符串，被切成任意位置（可能切在转义符中间）；
- 多个工具调用用 `index` 区分，**交错到达**。

所以不能"收到一片就 append"，必须按 `index` 聚合：

```python
tool_calls: dict = {}

for tc in delta.get("tool_calls") or []:
    idx = tc.get("index", 0)
    entry = tool_calls.setdefault(idx, {
        "id": "", "type": "function",
        "function": {"name": "", "arguments": ""},
    })
    if tc.get("id"):
        entry["id"] = tc["id"]
    fn = tc.get("function") or {}
    if fn.get("name"):
        entry["function"]["name"] += fn["name"]
    if fn.get("arguments"):
        entry["function"]["arguments"] += fn["arguments"]
```

图示（两个工具调用交错）：

```text
chunk 1: index=0 id=call_a name="read_"      → {0: {id:call_a, name:"read_"}}
chunk 2: index=1 id=call_b name="list_files" → {0:..., 1:{id:call_b, name:"list_files"}}
chunk 3: index=0 arguments='{"path":'        → {0: {args:'{"path":'}}
chunk 4: index=1 arguments='{}'              → {1: {args:'{}'}}
chunk 5: index=0 arguments='"a.txt"}'        → {0: {args:'{"path":"a.txt"}'}}
```

最后按 key 排序输出：

```python
[entry for _, entry in sorted(tool_calls.items())]
```

> 💡 **设计笔记**：这是整个流式实现里**唯一不能靠"边收边显示"偷懒**的地方。文本可以逐字渲染，工具调用必须**先完整聚合再解析**——因为在 JSON 闭合之前，任何解析尝试都会失败。协议设计如此，没有捷径。

## 6.5 三个字段名的兼容：reasoning_content / reasoning / thinking

Reasoning 模型的"思考过程"没有统一字段名：

| 提供方 | 字段 |
| --- | --- |
| DeepSeek（v4.1 系列） | `reasoning_content` |
| OpenRouter / 部分兼容层 | `reasoning` |
| 一些本地推理框架 | `thinking` |

One Cedric 的兼容写法：

```python
reasoning = (
    delta.get("reasoning_content")
    or delta.get("reasoning")
    or delta.get("thinking")
)
if reasoning:
    reasoning_parts.append(reasoning)
    ...
    yield ("reasoning", reasoning)
```

用 `or` 链而不是 `if/elif`：只要任意一个字段有值就取，字段为空字符串时自动落到下一个。代价是"同时返回两个字段"时会丢一个——实践中没遇到过，真遇到也该去重而不是拼接。

### DeepSeek 的"答案落在 reasoning_content"bug

📅 **09-27 前后**发现：某些 DeepSeek 模型（`deepseek-v4.1-flash` 等）在**带工具调用**的轮次里，会把最终答案写进 `reasoning_content`，而 `content` 是空的。表现是：界面上只有一大段"思考"，然后 agent 停了。

core 里的兜底（`core.py:6399`）：

```python
if (not final_content.strip()
        and final_reasoning.strip()
        and not tool_calls
        and self._is_deepseek_buggy_model()):
    final_content = final_reasoning
    self.console.print(
        "  [warn]⚠ 检测到 DeepSeek 答案"
        "落在 reasoning_content，"
        "已自动恢复为回答[/]")
```

四个条件缺一不可：

1. `content` 为空；
2. `reasoning` 非空；
3. **没有工具调用**（有工具调用时，reasoning 里可能只是"我要调工具"的思考，不能当答案）；
4. 模型名命中已知问题列表（`DEEPSEEK_BUGGY_MODEL_KEYWORDS`）——**不要对没问题的模型做这个猜测**。

对应的另一个方向：带工具调用时，把 reasoning 回填进 assistant 消息（`core.py:6462`）：

```python
if (final_reasoning and tool_calls
        and self._is_deepseek_buggy_model()):
    assistant_msg["reasoning_content"] = final_reasoning
```

原因：DeepSeek 要求多轮对话中**原样回传**上一轮的 `reasoning_content`，否则会报 400。这是"兼容层"必须记住的：**修一个 provider 的 bug，往往要在请求侧也补一刀**。

## 6.6 当模型把工具调用写成文本

回到 09-19 16:37 那段输出。qwen2.5:7b 没有走 `tool_calls` 字段，而是直接在正文里写了：

```text
 {"name": "write_file", "arguments": {"path": "F:\\testai.py", "content": ""}}
```

对 API 来说，这是一段普通文本，`tool_calls` 是空的——agent 会把它当最终回答展示给用户，然后结束。用户看到的是"模型说它要写文件，但什么都没发生"。

修复分两步：**扫描恢复** + **清理正文**。

### 扫描：`_scan_text_tool_calls`

`core.py:248`，一个手写的**花括号配对扫描器**，不是正则：

```python
while i < n:
    if text[i] != "{":
        i += 1
        continue
    depth = 0
    in_str = False
    esc = False
    j = i
    while j < n:
        c = text[j]
        if esc:
            esc = False
        elif c == "\\":
            esc = True
        elif c == '"' and not esc:
            in_str = not in_str
        elif not in_str:
            if c == "{":
                depth += 1
            elif c == "}":
                depth -= 1
                if depth == 0:
                    break
        j += 1
    ...
```

为什么不用正则？因为 JSON 里允许出现 `{`、`}`、`"`、`\\`：

```json
{"content": "def f(): return {\"a\": 1}"}
```

正则会被字符串里的花括号骗到。扫描器维护三个状态（深度 / 是否在字符串内 / 是否转义），才能正确找到**配对的右括号**。这是"解析结构化文本"的经典写法，值得背下来。

找到候选片段后：

```python
snippet = text[i:j + 1]
try:
    obj = json.loads(snippet)
except json.JSONDecodeError:
    i += 1
    continue
if not isinstance(obj, dict):
    i += 1
    continue
name = obj.get("name")
args = obj.get("arguments")
if isinstance(name, str) and name and isinstance(args, dict):
    results.append({
        "id": f"recover_{len(results)}",
        "type": "function",
        "function": {
            "name": name,
            "arguments": json.dumps(args, ensure_ascii=False),
        },
    })
```

只认 **`{"name": str, "arguments": dict}`** 这一种形状——足够覆盖模型的"文本工具调用"，又不会把普通 JSON 误判成工具。

### 清理：`_strip_text_tool_calls`

恢复之后，正文里还留着那些 JSON 片段，直接展示会很奇怪。`_strip_text_tool_calls` 用同样的扫描逻辑，把**确认是工具调用的片段**从文本里剪掉，再压缩多余空行：

```python
cleaned = "".join(out_parts)
cleaned = re.sub(r"\n{3,}", "\n\n", cleaned).strip()
```

接入点在 `agent_turn`（`core.py:6480`）：

```python
if not tool_calls and final_content:
    recovered = _scan_text_tool_calls(final_content)
    if recovered:
        self.console.print(
            f"  [warn]⚠ 模型把 {len(recovered)} 个"
            f"工具调用写成了文本，已自动恢复[/]")
        tool_calls = recovered
        assistant_msg["tool_calls"] = recovered
        cleaned = _strip_text_tool_calls(final_content)
        assistant_msg["content"] = cleaned
        final_content = cleaned
```

> ⚠️ **踩坑**：恢复出来的工具调用**没有真实 `id`**，这里用 `recover_0`、`recover_1` 伪造。绝大多数 API 接受任意字符串 id（只要 tool 消息里回传一致）；但个别严格实现要求 `call_` 前缀。如果遇到 400，先检查这里。

> 💡 **设计笔记**：这个功能本质上是"**给不听话的模型擦屁股**"。它不是优雅的架构，但极大提升了小模型的可用性——qwen2.5:7b、gpt-oss:20b 这类模型经常这么干。判断要不要做这类兼容，标准是"**这类模型有多少用户真的在用**"，而不是"协议是否纯粹"。

## 6.7 思考档位：从 minimal 到 ultra

用户最初要的档位是 `minimal / low / medium / max / xhigh / ultra`，实现时补了一个 `high`（因为 `max` 的语义和 `xhigh` 重叠，需要一个中间的"增强"档）：

```python
THINK_LEVELS = ("minimal", "low", "medium", "high", "xhigh", "max", "ultra")

THINK_TO_BUDGET = {
    "minimal": 512,
    "low":     2048,
    "medium":  8192,
    "high":    12288,
    "xhigh":   32768,
    "max":     16384,
    "ultra":   65536,
}
```

注意 `max`（16384）比 `xhigh`（32768）小——这是历史遗留：`max` 是早期档位名，后来插入了 `xhigh`，数值没跟着重排。**兼容旧配置比"数值单调"重要**。

### 三套映射，因为 provider 的 API 不一样

`_think_params()`（`core.py:831`）根据模型名决定发什么参数：

```python
m = self.model.lower()
tl = ov.get("think_level", self.think_level)
if any(k in m for k in ("o1", "o3", "o4", "gpt-5")):
    out["reasoning_effort"] = THINK_TO_REASONING_EFFORT.get(tl, "medium")
elif "claude" in m or "anthropic" in m:
    if tl != "minimal":
        out["thinking"] = {
            "type": "enabled",
            "budget_tokens": THINK_TO_BUDGET.get(tl, 8192),
        }
```

| 模型族 | 参数 | 取值 |
| --- | --- | --- |
| OpenAI o 系列 / gpt-5 | `reasoning_effort` | 档位名本身 |
| Claude / Anthropic | `thinking.budget_tokens` | 512–65536 |
| 其他 | **不发参数** | 靠 system prompt 提示 |

某些模型 的 API 不接受 `reasoning_effort` 或 `thinking`，强行发会 400。所以第三类模型走"提示词档位"：

```python
THINK_PROMPT_HINTS = {
    "minimal": "\n\n[思考模式: minimal] 直接给出答案，不要展示推理过程。",
    "low":     "\n\n[思考模式: low] 简要推理后给出答案。",
    "medium":  "",
    "high":    "\n\n[思考模式: high] 增强推理：深入分析后再回答。",
    "xhigh":   ("\n\n[思考模式: xhigh] 深度推理，从多个角度验证结论，"
                "主动指出潜在风险与替代方案。"),
    "max":     "\n\n[思考模式: max] 深入分析问题，考虑多种可能性后再回答。",
    "ultra":   ("\n\n[思考模式: ultra] 穷尽式推理：列出所有可能方案，"
                "逐一评估优缺点，正反论证，分析边界条件与失败场景，"
                "最后给出最优建议。必要时明确说明假设与不确定性。"),
}
```

`_system_with_think()` 把这些提示**追加到 system prompt 末尾**（只对发出去的副本生效）。`medium` 是空字符串——默认档位不加任何噪音。

> 💡 **设计笔记**：同一件事（"想多深"）在三种 API 里是三种参数。抽象层不该假装它们一样，而应该**按模型族分派**。判断依据是模型名子串（`"o1" in m`），粗糙但足够——比维护一张永远过时的模型清单可靠。

### 档位还影响 UI

📅 **10-01 14:55** 用户要求 "到 ultra 时变成蓝紫渐变色，有粒子飘动"。这条需求落在第 4 章的 UI 层：档位不只是 API 参数，也是**用户能看见的状态**。minimal 是安静的，ultra 是"引擎全开"的——视觉反馈让等待变得可解释。

## 6.8 token、速度、成本：把账算清楚

### 优先用真实 usage

```python
real_usage = self._last_usage or {}
real_in = _extract_input_tokens(real_usage)
real_out = _extract_output_tokens(real_usage)
real_cached = _extract_cached_tokens(real_usage)
if real_in or real_out:
    in_tok, out_tok, cached_tok = real_in, real_out, real_cached
    tag = ""
else:
    in_tok = int(input_chars / 2.5)
    out_tok = int(output_chars / 2.5)
    cached_tok = 0
    tag = "~"
```

估算系数 **2.5 字符/token** 是中英混合文本的经验值：英文约 4 字符/token，中文约 1.5–2 字符/token。估算值前面加 `~`，**明确告诉用户这不是真实值**。

三个提取函数兼容不同 provider 的字段名：

```python
# 输入：prompt_tokens / input_tokens
# 输出：completion_tokens / output_tokens
# 缓存：prompt_tokens_details.cached_tokens
#       / cache_read_input_tokens（Anthropic）
#       / prompt_cache_hit_tokens（DeepSeek）
```

### 首字延迟与生成速度

```python
speed_base = t_start
if first_content_at:
    speed_base = first_content_at
elif first_token_at:
    speed_base = first_token_at
gen_time = max(0.001, elapsed - (speed_base - t_start))
speed = out_tok / gen_time if gen_time > 0 else 0
ttfb = (first_token_at - t_start) if first_token_at else 0
```

两个时间点要区分：

- **TTFB（首字）**：从发请求到第一个 token（含思考 token）到达；
- **生成速度**：从**第一个正文 token** 到结束。用正文起点算，才不会被漫长的思考时间拉低。

最终输出一行：

```text
│ 输入 3.2k tok · 缓存 1.1k tok · 输出 412 tok · 28.4 tok/s · 耗时 14.6s · 首字 1.2s · 本轮 $0.00042
```

### 成本与预算

`pricing.compute_cost(model, in_tok, out_tok, cached_tok)` 按模型单价计算，缓存 token 通常打 1–5 折。之后：

- `cost_tracker.record(...)` 落盘；
- `check_budget()` 检查日/月预算，超了打红字；
- `check_turn_anomaly()` 用 z-score 检测"这一轮贵得离谱"（比如上下文泄漏导致 10 倍输入）。

> ⚠️ **踩坑**：`usage` 里的 `prompt_tokens` **已经包含缓存命中部分**，不要重复相加。One Cedric 的展示是"输入 3.2k（其中缓存 1.1k）"，计算成本时按 `(in - cached) * 单价 + cached * 缓存单价` 拆分。

## 6.9 思考过程为什么"显示完就没了"

📅 **09-27 16:12** 用户原话："目前的话是他显示完思考过程后就隐藏了"。

原因：Rich 的 `Live` 区域是 `transient=True` 的，思考结束后整个区域被擦除；而正文开始后，代码只把正文重新渲染进 Live——思考面板从未被"固定"到滚动区。

修复（第 4 章讲过 UI 侧，这里补数据侧）：引入 `reasoning_frozen` 标记，在**第一次收到正文**时把完整思考打印到 `live.console.print(...)`（固定输出，不随 Live 擦除）：

```python
if (self.show_reasoning and reasoning_buf
        and not reasoning_frozen and phase == "reasoning"):
    reasoning_frozen = True
    acc = "".join(reasoning_buf)
    self.console.print(Panel(
        Text(acc, style="dim italic"),
        title=f"[dim]思考过程（{len(acc)} 字）[/]",
        ...
    ))
```

另一个相关兜底：**只思考、不回答**。如果整轮只有 `reasoning` 没有 `content`（max_tokens 太小、或提示词触发纯推理），界面显示警告面板而不是空白：

```python
elif final_reasoning.strip():
    live.update(Panel(
        Text("模型只输出了思考过程，未生成最终回答。\n"
             "可能是 max_tokens 太小，或提示词触发了纯推理。",
             style="warn"),
        title="[warn]警告[/]", ...))
```

这条提示语是给用户看的**诊断建议**，不是给模型的。好的错误提示应该告诉用户"下一步能改什么"。

## 6.10 流式实现的取舍清单

| 决策 | 选择 | 理由 |
| --- | --- | --- |
| 解析方式 | `iter_lines()` 逐行 | 内存恒定，能处理任意长回复 |
| 解码容错 | `errors="replace"` | 代理截断 UTF-8 时不崩 |
| 渲染节流 | 0.12s | Markdown 重排昂贵（第 4 章） |
| 工具调用 | 聚合后再解析 | JSON 未闭合前无法解析 |
| usage | `include_usage` + 估算兜底 | 不是所有 provider 都返回 |
| 思考字段 | 三字段 `or` 链 | 兼容 DeepSeek/OpenRouter/本地框架 |
| 档位 | 按模型族分派三套参数 | API 不统一，硬统一会 400 |
| 文本工具调用 | 扫描 + 清理 | 小模型常见的现实问题 |

## 🔧 自己动手

1. **手写一个 SSE 客户端**：用 `requests` + `stream=True` 请求任意 OpenAI 兼容接口，逐行打印 `delta.content`，观察 `[DONE]` 的位置；
2. **模拟分片工具调用**：构造两个交错的 `tool_calls` 分片序列，验证你的聚合逻辑能还原出完整的 `name` 和 `arguments`；
3. **实现三字段兼容**：把 `reasoning_content` / `reasoning` / `thinking` 都映射到统一变量，写一个只有 `reasoning` 的假响应测试；
4. **测一次"文本工具调用"**：让本地小模型（qwen2.5:7b 之类）写文件，观察它是否把 JSON 写进正文；用本章的扫描器把它恢复成真正的工具调用；
5. **算一次账**：用真实 usage 和"字符数/2.5"估算各算一遍，对比误差；再故意把 `max_tokens` 设得很小，触发"只思考不回答"的警告。

---

[← 上一章：05 工具系统](05-tool-system) ｜ [返回目录](Home) ｜ [下一章：07 权限与安全 →](07-permissions-safety)
