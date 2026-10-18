# 10 · 网关与 WebUI：给终端 Agent 开一扇浏览器窗户

## 10.1 "新增 gateway，Claude 风"

📅 **开发日志 · 2026-10-01**

> **09:14 用户**（十条需求之第 8 条）：新增 gateway，Claude 风
>
> **13:04 用户**：批次 4：Gateway + WebUI（6 个文件）+ 测试清单 + 使用方法
>
> **13:06 用户**：WebUI 要美化一下，功能要完善
>
> **13:07 用户**：ok，然后双语，端口改为 2043

同一天下午还追加了：

> **14:51 用户**：……然后 full access 是基本不用确认的……
>
> **19:56 用户**：搜索框四周白框，左侧有一个小狗，像素级输出就行……

一天之内，一个纯终端 Agent 长出了 HTTP 网关、浏览器界面、中英双语、统计页、设置页。这一章讲它的结构，尤其是**最难的那部分：把同步的 core 接到异步的 FastAPI 上，还要保持流式**。

## 10.2 网关不是一个独立服务

最常见的做法是"CLI 一个进程，Gateway 另一个进程，通过 HTTP/WebSocket 通信"。One Cedric 没这么做——Gateway 是**嵌在 CLI 进程里的一个线程**：

```python
class Gateway:
    ASK_GRACE_SECONDS = 60

    def __init__(self, copilot):
        self.copilot = copilot
        self.host = "127.0.0.1"
        self.port = 2043
        self.token = ""
        ...
        # 反向引用，供 core 里的 ask_user 使用
        copilot._gateway_ref = self

        self.app = self._build_app()
```

启动：

```python
config = uvicorn.Config(self.app, host=self.host, port=self.port,
                        log_level="warning", access_log=False)
self._server = uvicorn.Server(config)
self._server.install_signal_handlers = lambda: None

def _run():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    try:
        loop.run_until_complete(self._server.serve())
    ...

self._thread = threading.Thread(
    target=_run, daemon=True, name="cedric-gateway")
self._thread.start()
```

四个关键决策：

1. **`daemon=True`**：CLI 退出时网关线程自动结束，不会挂住进程；
2. **`install_signal_handlers = lambda: None`**：uvicorn 默认会接管 `SIGINT`，在子线程里注册信号处理器会报错；必须关掉；
3. **独立 event loop**：`asyncio.new_event_loop()`，不干扰主线程；
4. **`copilot._gateway_ref = self`**：core 里的 `ask_user` 通过这个反向引用找到网关，把问题推给浏览器。

好处：**共享同一个 `OneCedric` 实例**。会话历史、工具缓存、权限状态、undo 栈都是同一份——用户在 CLI 里聊到一半，打开浏览器可以接着聊。坏处：**线程安全**要自己保证，`_chat_lock` 就是为此而生。

## 10.3 44 条路由的地图

`server.py` 一共注册了 **44 条路由**（实测统计）。按用途分组：

| 组 | 路由 | 说明 |
| --- | --- | --- |
| 页面 | `/` `/settings` `/stats` `/static/{name}` `/favicon.ico` | WebUI 三页 + 静态资源 |
| 状态 | `/health` `/api/status` `/api/diagnose` `/api/network` | 健康检查、诊断 |
| 模型 | `/api/models` `/api/my-models`（增删切换） | 模型列表与自定义模型 |
| 工具 | `/api/tools` `/api/tools/{name}/toggle` `/api/tools/{name}/call` | 工具列表、启停、直调 |
| 自定义工具 | `/api/custom-tools`（CRUD + generate + toggle） | 用户自定义工具 |
| 定时/梦境 | `/api/cron`（增删启停跑）、`/api/dream*` | 自动化 |
| 对话 | `/api/chat` `/api/chat/stream` `/api/chat/undo` `/api/chat/compact` `/api/chat/clear` | 核心对话 |
| 会话 | `/api/sessions` | 会话列表 |
| 提问 | `/api/ask/pending` `/api/ask/answer` | ask_user 的反向通道 |
| 配置 | `/api/config`（GET/POST） | 设置页 |
| 文件 | `/api/files` `/api/file-preview` `/api/upload` | 文件浏览与上传 |
| 日志 | `/api/logs` | 最近请求日志 |

注意 `/api/chat` 和 `/api/chat/stream` 是**两个**入口：

- `/api/chat`：同步等待完整回答，适合脚本调用；
- `/api/chat/stream`：SSE 流式，WebUI 用这个。

## 10.4 SSE 桥：同步 core 怎么接异步 FastAPI

这是整个网关最精妙的一段。

### 问题

`core.ask_no_ui_stream()` 是一个**同步生成器**——它内部阻塞地读模型流、执行工具，`yield` 事件。FastAPI 的 `StreamingResponse` 需要一个**异步生成器**。两者不能直接对接：

```python
# 这样写会阻塞整个 event loop
async def event_gen():
    for evt in core.ask_no_ui_stream(...):   # 同步阻塞！
        yield f"data: {json.dumps(evt)}\n\n"
```

### 解法：线程 + 队列 + `run_coroutine_threadsafe`

```python
async def event_gen():
    loop = asyncio.get_running_loop()
    queue: asyncio.Queue = asyncio.Queue()

    def _produce():
        try:
            for evt in gw.copilot.ask_no_ui_stream(
                    req.message, auto_approve=req.auto_approve):
                fut = asyncio.run_coroutine_threadsafe(
                    queue.put(evt), loop)
                fut.result(timeout=30)
        except Exception as exc:
            asyncio.run_coroutine_threadsafe(
                queue.put({"type": "error", "message": str(exc)}), loop)
        finally:
            asyncio.run_coroutine_threadsafe(queue.put(None), loop)

    acquired = gw._chat_lock.acquire(blocking=False)
    if not acquired:
        yield 'data: {"type":"error","message":"另一个请求正在进行，请稍后"}\n\n'
        yield "data: [DONE]\n\n"
        return

    try:
        threading.Thread(target=_produce, daemon=True).start()
        while True:
            evt = await queue.get()
            if evt is None:
                break
            yield f"data: {json.dumps(evt, ensure_ascii=False)}\n\n"
        yield "data: [DONE]\n\n"
    finally:
        gw.mark_orphaned_all()
        gw._chat_lock.release()
```

三段式：

```text
同步 core（工作线程）            asyncio 事件循环（主线程）
     │  yield evt                        │
     ├── run_coroutine_threadsafe ──────→ queue.put(evt)
     │                                   │
     │                              await queue.get()
     │                                   │
     │                              yield "data: {...}"
```

为什么不用 `asyncio.to_thread` 包住整个生成器？因为生成器需要在**每个事件**上往返，而不是等全部结束。`to_thread` 只能返回一个最终值。

为什么 `fut.result(timeout=30)`？如果浏览器断开、event loop 停止，`queue.put` 的 future 永远不会完成；加超时让工作线程能退出，避免线程泄漏。

### 响应头

```python
return StreamingResponse(
    event_gen(),
    media_type="text/event-stream",
    headers={
        "Cache-Control": "no-cache",
        "X-Accel-Buffering": "no",
        "Connection": "keep-alive",
    },
)
```

- **`X-Accel-Buffering: no`**：告诉 Nginx（以及某些反向代理）不要缓冲。没有这个头，SSE 会被攒成一整块再发，流式就废了；
- **`Cache-Control: no-cache`**：SSE 不能被缓存；
- **`Connection: keep-alive`**：保持长连接。

### 并发保护

`_chat_lock.acquire(blocking=False)`：同一时刻只允许一个对话请求。第二个请求立刻收到 `"另一个请求正在进行，请稍后"`，而不是排队——排队会让用户以为界面卡死。

## 10.5 事件协议：core 和 WebUI 的契约

`_stream_event` 是 core 里所有 UI 事件的出口（`core.py:774`）：

```python
def _stream_event(self, event: dict) -> None:
    if self._stream_hook is None:
        return
    try:
        self._stream_hook(event)
    except Exception:
        pass
```

事件类型（WebUI 的 `app.js` 逐类处理）：

| type | 字段 | 用途 |
| --- | --- | --- |
| `reasoning` | `delta` | 思考增量 |
| `content` | `delta` | 回答增量 |
| `tool_call` | `call_id` `name` `target` `summary` `status` `result` `duration` | 工具开始/结束 |
| `ask_user` | `id` `questions` `timeout` `header` | 请求用户选择 |
| `done` | `answer` `duration` | 整轮结束 |
| `error` | `message` | 错误 |

`ask_no_ui_stream` 把同步 core 包成"事件流"（`core.py:7019`）：

```python
def ask_no_ui_stream(self, prompt: str, auto_approve: bool = False):
    q: queue.Queue = queue.Queue()

    def _hook(evt: dict):
        q.put(evt)

    old_hook = self._stream_hook
    self._stream_hook = _hook

    def _runner():
        r = self.ask_no_ui(prompt, auto_approve=auto_approve)
        result_box["answer"] = r.get("answer", "")
        ...
        q.put(None)

    threading.Thread(target=_runner, daemon=True).start()

    try:
        while True:
            evt = q.get()
            if evt is None:
                break
            yield evt
    finally:
        self._stream_hook = old_hook
```

注意 `finally: self._stream_hook = old_hook`：WebUI 请求结束后必须恢复 hook，否则下一次 CLI 对话的事件会跑进已经关闭的队列。

## 10.6 ask_user：从浏览器把答案送回来

普通工具是"core 调用、返回结果"，但 `ask_user` 是**反向**的：core 需要暂停，等浏览器里的用户点选。

### core 侧：注册 + 等待

```python
def _ask_user_via_gateway(self, questions, timeout, header) -> dict:
    ask_id = f"ask_{int(time.time())}_{secrets.token_hex(3)}"
    evt = threading.Event()
    holder = {"answers": None, "event": evt}
    with self._ask_lock:
        self._ask_registry[ask_id] = holder

    gw.register_ask(ask_id, {
        "header": header, "timeout": timeout, "questions": questions,
    })
    self._stream_hook({
        "type": "ask_user", "id": ask_id,
        "questions": questions, "timeout": timeout, "header": header,
    })

    answered = evt.wait(None if timeout <= 0 else timeout)
    ...
    if not answered:
        return {q["key"]: (q.get("default") or "") for q in questions}
```

三个设计点：

1. **`ask_id` 用时间戳 + 随机 hex**：避免并发请求撞 id；
2. **`threading.Event.wait(timeout)`**：阻塞等待，但不会永远挂住——超时返回默认值；
3. **超时返回默认值，而不是报错**：用户离开浏览器时，agent 还能继续走"保守路径"。

### 网关侧：待答列表 + 提交答案

```python
@app.get("/api/ask/pending")
async def api_ask_pending(request: Request):
    _auth(request)
    return {"pending": gw.get_pending_asks()}

@app.post("/api/ask/answer")
async def api_ask_answer(req: AskAnswerRequest, request: Request):
    _auth(request)
    payload = req.answers if req.answers else req.answer
    ok = gw.copilot.answer_ask(req.id, payload)
    if not ok:
        return {"ok": False, "error": "提问不存在或已超时"}
    return {"ok": True}
```

`answer_ask` 设置 `holder["answers"]` 并 `event.set()`，core 的 `wait` 立刻返回。

### 断线宽限

如果浏览器在提问过程中刷新/关闭，`finally` 里会调用 `mark_orphaned_all()`：

```python
n_orphaned = gw.mark_orphaned_all()
if n_orphaned:
    gw._log_request("SSE", "/api/chat/stream", 200, 0,
                    f"{n_orphaned} 个 ask 进入宽限期")
```

`ASK_GRACE_SECONDS = 60`：问题不会立刻销毁，而是保留 60 秒——用户刷新页面后还能看到并回答。

> 💡 **设计笔记**：`ask_user` 是 agent 里唯一一个"**必须等待人类**"的工具。它的难点不是 UI，而是**生命周期**：请求可能超时、页面可能刷新、连接可能断。把"等待"建模成 `Event + 注册表 + 宽限期`，比"轮询数据库"简单得多。

## 10.7 WebUI：没有构建步骤的 Vue

`gateway/webui/` 下的文件：

```text
index.html      88 KB   聊天主界面
settings.html   24 KB   设置页
stats.html      16 KB   统计页
app.js         174 KB   Vue 应用逻辑
style.css      112 KB   样式（含 Liquid Glass）
polish.css      24 KB   细节修饰
tokens.css       6 KB   设计 token
vue.min.js     147 KB   Vue 3（本地 vendored）
marked.min.js   35 KB   Markdown 渲染
highlight.min.js 121 KB 代码高亮
```

**没有 npm、没有 vite、没有打包**。`index.html` 直接 `<script src="/static/vue.min.js">`，`app.js` 里写 `Vue.createApp({...})`。理由和第 9 章的 PS1 一样：**降低运行门槛**。用户 `pip install` 完就能用，不需要 Node.js。

静态文件路由有一个安全细节：

```python
@app.get("/static/{name}")
async def static_file(name: str):
    if "/" in name or "\\" in name or ".." in name:
        raise HTTPException(400, "invalid filename")
    f = webui_dir / name
    if not f.exists() or not f.is_file():
        raise HTTPException(404, "not found")
```

拒绝 `/`、`\`、`..`——否则 `/static/../../config.toml` 就能读到配置（里面有 API key）。这是路径穿越的经典防御。

缓存用查询参数控制：`style.css?v=20261012a`。改样式时更新 `v`，浏览器就会重新拉取，不需要配置 ETag。

## 10.8 Liquid Glass：用 CSS 变量做"液态玻璃"

用户要"Claude 风"，最终落地成一套 **Liquid Glass** 皮肤（`style.css:2300`）：

```css
:root {
  --glass-bg:          rgba(255, 255, 255, 0.075);
  --glass-bg-soft:     rgba(14, 14, 17, 0.46);
  --glass-btn-bg:      rgba(255, 255, 255, 0.055);
  --glass-btn-hover:   rgba(255, 255, 255, 0.12);
  --glass-border:      rgba(255, 255, 255, 0.13);
  --glass-border-strong: rgba(255, 255, 255, 0.24);
  --glass-shadow:      0 10px 36px rgba(0, 0, 0, 0.38),
                       0 2px 8px rgba(0, 0, 0, 0.22);
  --glass-inner:       inset 0 1px 0 rgba(255, 255, 255, 0.09);
  --glass-blur:        blur(24px) saturate(1.5);
}
```

组件只引用变量：

```css
.topbar {
  height: 52px;
  margin: 10px 14px 0;
  border-radius: 20px;
  border: 1px solid var(--glass-border);
  background: var(--glass-bg);
  -webkit-backdrop-filter: var(--glass-blur);
  backdrop-filter: var(--glass-blur);
  box-shadow: var(--glass-shadow), var(--glass-inner);
}
```

三个"玻璃感"的来源：

1. **`backdrop-filter: blur(24px) saturate(1.5)`**：模糊背后的内容并提高饱和度；
2. **`inset 0 1px 0 rgba(255,255,255,.09)`**：顶部一条内高光，模拟玻璃边缘的反光；
3. **彩色径向渐变背景**：玻璃需要"有东西可折射"。纯黑背景上的模糊玻璃看起来只是灰色方块：

```css
.app-bg {
  background:
    radial-gradient(900px 600px at 12% -8%, rgba(217,119,87,.16), transparent 60%),
    radial-gradient(1000px 700px at 108% 12%, rgba(125,207,255,.13), transparent 62%),
    radial-gradient(800px 800px at 50% 118%, rgba(186,154,247,.10), transparent 55%),
    var(--bg);
}
```

亮色主题只覆盖变量（`.theme-light { --glass-bg: rgba(255,255,255,.52); ... }`），组件 CSS 一行不改。

> ⚠️ **踩坑**：`backdrop-filter` 在旧版 Chromium / 部分集成显卡上性能很差，滚动时会掉帧。One Cedric 的取舍是：**只给固定区域（顶栏、侧栏、输入框、弹层）加玻璃，聊天消息列表不加**。大面积模糊 + 长列表滚动 = 卡顿。

## 10.9 网关的安全边界

网关把 agent 暴露到了 HTTP，风险比 CLI 高一个量级。三道防线：

### ① 默认只监听本机

```python
self.host = "127.0.0.1"
```

要局域网访问必须显式传 `--host 0.0.0.0`，并且最好设 token。

### ② 可选的 Bearer token

`_auth(request)` 在每个敏感路由上调用；token 为空时放行（本机场景），设置后校验 `Authorization`。审计报告指出旧版是"逐路由手写、非恒定时间比较"（SEC-05），修复方向是统一中间件 + `secrets.compare_digest`。

### ③ 自动确认默认关闭

审计报告 SEC-01 的修复直接体现在前端：

```js
// app.js:754
const autoApprove = ref(false);
```

服务端也把默认值改成 `False`：

```python
auto_approve: bool = False
```

**即使前端被篡改，服务端默认也不放行**——这就是"纵深防御"：不把安全寄托在任何一个界面上。

## 10.10 网关开发的六个坑

1. **SSE 被代理缓冲**：加 `X-Accel-Buffering: no`，否则流式变批量；
2. **同步生成器阻塞 event loop**：必须用线程 + `run_coroutine_threadsafe`；
3. **并发对话**：`_chat_lock` 非阻塞获取，第二个请求直接拒绝；
4. **浏览器断开**：`mark_orphaned_all()` + 60 秒宽限，避免 ask 永久挂起；
5. **静态路径穿越**：拒绝 `..`、`/`、`\`；
6. **信号处理器**：uvicorn 在子线程里不能 `install_signal_handlers`，要显式禁用。

## 🔧 自己动手

1. **写一个最小 SSE 服务**：FastAPI + `StreamingResponse`，每 0.5 秒推一条 `data: {...}`，用浏览器 `EventSource` 接收；
2. **复现同步→异步桥**：用一个阻塞生成器 + `queue.Queue` + `run_coroutine_threadsafe`，把它接到 SSE 上；
3. **实现 ask_user 往返**：`GET /pending` + `POST /answer` + `threading.Event`，测试超时返回默认值；
4. **做一套 CSS 变量主题**：定义 `--glass-*`，实现暗/亮两套，组件只引用变量；
5. **攻击自己的静态路由**：尝试 `/static/../config.toml`、`/static/%2e%2e/config.toml`，确认都被拒绝。

---

[← 上一章：09 十条需求与批量交付](09-mega-batch) ｜ [返回目录](Home) ｜ [下一章：11 记忆、自动化与生态 →](11-memory-automation-ecosystem)
