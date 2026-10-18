# 01 · 概念篇：Agent 到底是什么

## 1.1 从"聊天"到"干活"

先做一个思想实验。

你对ChatGPT说："帮我看看 `README.md` 里写了什么。"

它只能回答："请把文件内容粘贴给我。"

这不是它笨，而是它的世界只有**文本**：你给它一段文字，它预测下一段文字。它看不见你的硬盘，也碰不到你的键盘。

现在换成一个 Agent：

```
你：帮我看看 README.md 里写了什么。
Agent：（内部）我需要读文件 → 调用 read_file(path="README.md")
运行时：（真的去读文件，把内容返回）
Agent：（拿到内容）这个项目是……
```

区别在哪？多了一个**动作**，以及执行动作后**把结果喂回模型**的闭环。

> **一句话定义**：Agent = 会调用工具的语言模型 + 一个反复"思考 → 行动 → 观察"的循环。

注意到关键点：**模型本身仍然没有执行任何东西**。它只是输出了一段结构化文本：

```json
{
  "name": "read_file",
  "arguments": {"path": "README.md"}
}
```

真正打开文件的是**你的代码**。这是理解一切 Agent 框架的第一把钥匙——所谓"工具调用"，本质是：

1. 你告诉模型"我有这些工具，参数长这样"；
2. 模型回复"那请帮我调用这个工具，参数是这些"；
3. 你的程序执行，把结果作为新消息追加；
4. 模型看到结果，继续下一步，直到不再请求工具。

## 1.2 Agent 的五个零件

把上面这个过程拆开，一个最小 Agent 只有五个零件：

| 零件 | 作用 | 在 One Cedric 里对应的位置 |
| --- | --- | --- |
| **模型（Brain）** | 决策：下一步说什么、调什么工具 | 任意 OpenAI 兼容 API（Ollama / DeepSeek / …） |
| **工具（Tools）** | 能力：读写文件、跑命令、搜网页…… | `one_cedric/tools/`（240+ 个） |
| **循环（Loop）** | 调度：发请求 → 执行工具 → 回填结果 → 再发请求 | `core.py` 的 `agent_turn()` |
| **上下文（Context）** | 记忆：system prompt、历史消息、工具结果、记忆块 | `messages` 列表 + `memory.py` |
| **边界（Guardrails）** | 权限：哪些能自动做、哪些要问人、哪些永远不行 | `core.py` 权限层 + `tools/shell.py` |

初学者最容易只关注前两个（选什么模型、写多少工具），而忽略后三个。但一个 Agent 从"玩具"变成"能日常用"的分水岭，几乎全在后三个零件上——这也是本教程后半部分的重点。

## 1.3 工具调用协议：一次完整的往返

Agent 生态里事实上的通用语言，是 OpenAI 的 Chat Completions 协议。One Cedric 从头到尾只实现这一套协议，因此可以无缝切换 Ollama（本地）、DeepSeek、Kimi、OpenAI 等后端。

### 请求

```http
POST http://localhost:11434/v1/chat/completions
Content-Type: application/json
```

```json
{
  "model": "qwen2.5:7b",
  "messages": [
    {"role": "system", "content": "你是一个可以读取本地文件的助手。"},
    {"role": "user", "content": "README.md 里写了什么？"}
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "read_file",
        "description": "读取本地文本文件，返回带行号的内容。",
        "parameters": {
          "type": "object",
          "properties": {
            "path": {"type": "string", "description": "文件路径"}
          },
          "required": ["path"]
        }
      }
    }
  ],
  "tool_choice": "auto",
  "stream": false
}
```

三个字段值得记住：

- `tools`：工具清单。注意它是 **JSON Schema**，模型靠 `name` / `description` / `parameters` 来决定"什么时候用、怎么传参"。写得含糊，模型就用得乱七八糟；
- `tool_choice: "auto"`：让模型自己决定要不要调用；
- `stream`：是否流式返回（第 6 章详细讲）。

### 响应（模型请求调用工具）

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "",
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "read_file",
          "arguments": "{\"path\": \"README.md\"}"
        }
      }]
    }
  }]
}
```

两个细节，后面会反复出现：

1. `arguments` 是**字符串**，不是对象。需要自己 `json.loads()`；
2. `id` 必须原样带回。下一轮你要用它把"工具结果"和"这次调用"对应起来。

### 回填结果（你的程序执行完之后）

```json
{
  "messages": [
    {"role": "system", "content": "……"},
    {"role": "user", "content": "README.md 里写了什么？"},
    {"role": "assistant", "content": "", "tool_calls": [ ... ]},
    {"role": "tool", "tool_call_id": "call_abc123", "content": "文件: README.md\n     1| # One Cedric\n……"}
  ]
}
```

再发一次请求，模型看到工具结果，就会生成最终回答。若它还想调工具，就继续循环。

> **心智模型**：把 `messages` 想成一份不断变长的会议纪要。模型每轮都重读整份纪要，然后决定"接着说"还是"先去做点什么"。所谓 Agent 循环，就是不断往纪要里追加"助手的决定"和"工具的回执"。

### 流式（SSE）长什么样

当 `stream: true` 时，服务端不再返回一个大 JSON，而是一串事件流：

```
data: {"choices":[{"delta":{"content":"这个"}}]}
data: {"choices":[{"delta":{"content":"项目"}}]}
data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"name":"read_file","arguments":"{\"pa"}}]}}]}
data: {"choices":[{"delta":{"tool_calls":[{"index":0,"function":{"arguments":"th\":\"RE"}}]}}]}
data: [DONE]
```

工具调用在流式下是**碎片**：`name` 和 `arguments` 会分多次到达，需要按 `index` 拼接。这是第 6 章的核心内容，也是很多人自己写 Agent 时翻车的地方。

## 1.4 伪代码：循环只有十行

抛开所有工程细节，Agent 循环可以写成：

```python
messages = [system_prompt, user_message]

while True:
    msg = call_model(messages, tools=TOOLS)

    if not msg.tool_calls:
        return msg.content          # 模型不再需要工具，结束

    messages.append(msg)            # 助手的"决定"也要进历史

    for tc in msg.tool_calls:
        result = execute(tc.function.name, json.loads(tc.function.arguments))
        messages.append({
            "role": "tool",
            "tool_call_id": tc.id,
            "content": result,
        })
```

第 2 章会把这十行扩展成 500 行可运行的代码。之后的所有章节，本质上都在回答一个问题：

> **这十行里，每一行在生产环境会出什么幺蛾子？**

## 1.5 为什么选 OpenAI 兼容协议

📅 **开发日志 · 2026-09-13**（对话 #57，模型思考记录）

> "用原生 `/api/chat` 也行……为了稳妥，我用 OpenAI 兼容端点，这样 tool_calls 有 id，格式标准。"

这个决定后来被证明极其关键：

- **一套代码，多个后端**：Ollama 本地模型、DeepSeek、Kimi、OpenAI 都讲同一种语言；换后端只需改 `host` 和 `api_key`；
- **工具调用有标准结构**：`tool_calls[].id` 让多工具、并行工具的结果回填不会串线；
- **生态兼容**：MCP、各类 SDK、各家 API 聚合服务，默认都支持这套协议。

所以教程从第 2 章起就使用 `/v1/chat/completions`，而不是 Ollama 的原生 `/api/chat`。你本地没有 Ollama 也没关系——任何 OpenAI 兼容服务（包括云端）都能跟着跑。

## 1.6 环境准备

### 最低配置

```bash
python --version          # 需要 3.10+
pip install requests      # 第 2 章只需要这一个依赖
```

### 使用本地模型（推荐起步）

1. 安装 [Ollama](https://ollama.com)；
2. 启动服务：`ollama serve`；
3. 拉一个支持工具调用的模型：

```bash
ollama pull qwen2.5:7b
```

> 为什么是 qwen2.5:7b？它对中文友好、支持 function calling、7B 量化后普通笔记本能跑。后续你也可以换 `qwen3`、`deepseek-r1` 等，但注意：**不是所有模型都支持工具调用**，这是新手第一个大坑（第 2 章会展示真实报错）。

### 验证服务可用

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d "{\"model\":\"qwen2.5:7b\",\"messages\":[{\"role\":\"user\",\"content\":\"你好\"}]}"
```

返回里能看到 `choices[0].message.content`，就说明通了。

### 如果没有本地模型

把 `host` 换成任意 OpenAI 兼容地址（例如 `https://api.deepseek.com`），并在请求头加 `Authorization: Bearer <key>`。第 2 章的代码已经把 `--host` 做成了参数。

## 1.7 开发时间线：这个项目是怎么长出来的

在进入代码之前，先看一眼全局。下面这些日期都来自真实对话记录，后面的章节会反复回到这些节点：

| 日期 | 事件 |
| --- | --- |
| 09-11 | 调研 Python 开源 CLI（对话 #50），确定"做一个终端 Agent"的方向 |
| 09-12 | 研究 claw-agent 的交互与 ASCII 界面（#51–#53） |
| 09-13 | 16:26 完成一个 jiter 安装问题（#56）；**16:58 提出"生成一个简易的 Python agent-cli，支持 read-file tool，ollama 模型"（#57）**；17:03 追加"jiajigetools"（加几个 tools） |
| 09-18 | 22:24 带着 4 工具版本进入新对话（#67），开始 One Copilot 时期；提出"以 beta 加数字"；23:01 说"我觉得测试没什么意义啊"（后来被现实教育） |
| 09-19 | "拆分"（拆包）；"用 rich 来美化 ui"；"每个 tool 都要有不同执行动画"；下午连续遭遇相对导入、Spinner、流式、云端模型 502 等问题 |
| 09-25 | 要求支持 reasoning 模型（DeepSeek v4.1-flash）、自定义 API 地址与 key；查 Claude Code 工具有多少（#59）后说"再加 10 个 tools" |
| 09-26 | 优化 websearch（readability / 语义搜索）、接入 LSP、支持 `@` 文件引用 |
| 09-27 | 全部改名 One Cedric；新增 computer use；修三连模块导入错误；做思考过程展示、token 速度、think 档位（minimal→ultra） |
| 10-01 | 09:14 提出十条大需求；白天分阶段交付；晚上进入 UI 打磨：像素小狗、主菜单、模型/会话管理、梦境 |
| 10-02 | 继续补齐功能；09:45 说"不要分批"；项目主体完成 |
| 10-05 – 10-11 | 发布前收尾：pytest + CI、安全审计 17 项修复、WebUI 打磨、TUI、Public Beta I |

> 一个有意思的细节：09-13 那个下午，作者原本只是想"生成一个简易的 CLI"；25 天后，它变成了一个有 240+ 工具、双界面、带安全审计的开源项目。教程要记录的，就是这段路是怎么一步步走出来的。

## 1.8 设计笔记：一开始就要想清楚的三件事

从 One Cedric 的早期代码里，可以提炼出三个"如果重来一次仍会这么做"的决定：

1. **协议优先于框架**：不引入 Agent 框架，直接实现 OpenAI 协议。代价是要自己处理流式与工具调用，收益是每一行都可控、可调试、可迁移。
2. **工具是数据，不是代码**：每个工具由"schema（给模型看）+ handler（给人跑）"组成，新增工具不需要改循环。这让工具数量从 4 涨到 240+ 而没有重构核心。
3. **默认不信任模型**：从第一版起，`read_file` 就带工作目录沙箱；后来所有写操作都要确认。这个选择在安全审计时救了项目（第 7 章）。

## 🔧 自己动手

1. 用 `curl` 完成一次带 `tools` 的请求，观察模型返回的 `tool_calls` 结构；
2. 把 `stream` 改成 `true`，用 `curl -N` 看事件流长什么样；
3. 故意把 `parameters.required` 写错（比如要求一个不存在的字段），观察模型的行为——你会直观感受到 schema 对模型的影响。

---

[← 上一章：00 前言](00-preface) ｜ [返回目录](Home) ｜ [下一章：02 第一个 Agent →](02-minimal-agent)
