# 12 · 测试、审计与发布：把"能用"变成"敢发布"

## 12.1 "我觉得测试没什么意义啊"

📅 **开发日志 · 2026-09-18 23:01**（对话 #67）

那天的 One Copilot 有 19 个工具、56 个测试、一个可以跑的 CI。然后用户说了这句话：

> **09-18 23:01 用户**：我觉得测试没什么意义啊。

助手的回答没有硬推"测试很重要"，而是先承认，再划一条线：

> **助手**：你说得对，我在这上面推得有点急了。
>
> 这个阶段的项目，测试的 ROI 确实低：
>
> - **接口天天变**：Beta 1→14，每个版本都在改签名、改返回格式。写完的测试下个版本就得改，纯负担。
> - **单人自用**：没有第三方依赖你的行为，坏了修一下就行，不需要回归保护。
> - **测的东西不重要**：`glob` 返回对不对、`json_query` 路径解析对不对——这些错了一眼就能看出来，写测试是自我安慰。
>
> ……
>
> **但有两处例外**：**沙箱路径解析** 和 **bash 白名单**——这两个一旦出 bug，不是"功能坏了"，是"AI 在你系统上跑了 `rm -rf`"……这种错误的成本远高于测试成本。

用户接受了这个判断，回了一句"继续加工具"——于是测试的工程化被雪藏了整整三周。

📅 **快进到 2026-10-11 下午**。发布前最后一天，git 历史长这样：

```text
ce13ff7 10-11 14:26 fix(webui): allow 'unsafe-eval' so the Vue app can mount (black screen)
af22158 10-11 14:15 chore(lint): make the blocking ruff set actually pass
b22c7d0 10-11 14:09 feat(webui): layer the stylesheets and add a visual/interaction polish pass
3b48993 10-11 14:06 feat(cli): bottom-pinned REPL input box with mouse, editing and mid-turn queue/interject
47c9c19 10-11 14:02 fix(security): close the audit findings that were reachable today
188421a 10-11 13:58 test: add a pytest suite, CI workflow and fix the bugs it found
dfd709d 10-11 13:26 fix(deps): correct distribution names, split dev/optional requirements, unify version
0a0cf2f 10-11 13:23 fix(repo): track winvt.py and my_models.py, add .gitattributes
53266d9 10-11 13:21 chore(repo): stop tracking build artifacts, bytecode caches and vendored node_modules
0376a12 10-11 13:20 chore: checkpoint before optimization (WIP baseline)
```

**一个多小时，九个提交**——其中六个是典型的"发布前整改"：清理仓库、修依赖名、补测试、修安全、加 CI、修 lint。那位说过"测试没什么意义"的用户，这天的提交信息里写着：

> `test: add a pytest suite, CI workflow and fix the bugs it found`

这一章讲的就是**这个转变为什么发生、怎么发生的**——以及你可以直接从哪一步抄起。

## 12.2 什么时候该回头补测试：ROI 曲线的拐点

09-18 的三条判断放到当时全对：接口天天变、单人自用、测的东西不重要。但它们有一个共同的前提——**项目还小**。三周里发生了三件事，把这个前提一条条拆掉了：

**① 工具从 19 涨到 250。**单个工具"错了一眼就能看出来"；250 个工具**之间**的关系（每个 schema 都有实现吗？写集合和只读集合有没有重叠？缓存集合是不是只读子集？）错了，眼睛根本看不出来——你只会在某天发现"plan 模式偷偷改了文件"。

**② 要发布了，别人会 clone。**单人自用可以"坏了修一下"；但 `pip install` 的陌生人遇到的是**启动即崩**——而这恰好是真实发生过的（10-11 13:23 那个提交：`winvt.py` 和 `my_models.py` 从未提交，新克隆的仓库一启动就 ImportError）。

**③ 改动在加速。**AI 辅助编码让"一次改动"的规模变大：一晚上重构 REPL 输入、换一套样式表、批量修 461 个 lint 问题。改动越快，**"手工回归一遍"越不可能**。

于是 10-11 采取的立场，恰好是 09-18 那晚结论的**升级版**：

> **不测"功能对不对"（自己用一眼就看出来），测"关系对不对"（契约不变量）和"边界守不守得住"（安全）。**

这套测试最终长成了这样（数字是写作时在干净检出上现跑的）：

| 测试文件 | 数量 | 类别 | 测什么 |
| --- | ---: | --- | --- |
| `test_shell_policy.py` | 51 | 安全 | 三层命令分类 + 沙箱环境清洗 |
| `test_live_input.py` | 43 | 表面 | 行编辑器 / 鼠标 / 按键序列 / 排队与插队 |
| `test_security.py` | 38 | 安全 | 审计结论逐条"钉住" |
| `test_gateway_auth.py` | 28 | 安全 | 每个 API 路由的 token 强制 |
| `test_tools_e2e.py` | 27 | 端到端 | 真调用本地工具 |
| `test_webui_assets.py` | 17 | 契约 | 资源存在性 / CSP 与 Vue 的兼容 |
| `test_packaging.py` | 14 | 契约 | 版本单一来源 / 依赖名单 |
| `test_tool_registry.py` | 12 | 契约 | 250 工具的关系网不变量 |
| `test_storage.py` | 9 | 端到端 | 配置分层、往返、坏文件容错 |
| `test_tool_dispatch.py` | 6 | 契约 | 分发器错误包容与权限路由 |
| `test_cli_smoke.py` | 4 | 表面 | CLI 参数真跑一遍 |
| **合计** | **249** | | 全部通过（Python 3.14.3 / Windows） |

249 这个数字可以逐段对账：`188421a` 的提交信息写着 **151** 个测试；`47c9c19` 补完安全测试后写着 **189** 个；`af22158` 写着 **247** 个；`ce13ff7` 后你在本机 `pytest --collect-only -q` 会看到 **249** 个。**提交信息里的数字和实际数字能对上账，本身就是工程纪律。**

## 12.3 骨架：conftest 与"永不碰真实配置"

整个套件的底座是 `tests/conftest.py`，只有两个 fixture，但立了两条规矩：

```python
REPO_ROOT = Path(__file__).resolve().parents[1]
if str(REPO_ROOT) not in sys.path:
    sys.path.insert(0, str(REPO_ROOT))


@pytest.fixture()
def isolated_home(tmp_path, monkeypatch):
    """Point ``Path.home()`` at a throwaway directory.

    Any code under test that reads or writes ``~/.one-cedric`` must use this
    fixture so the suite never touches the developer's real configuration,
    sessions or API keys.
    """
    home = tmp_path / "home"
    home.mkdir()
    monkeypatch.setattr(Path, "home", classmethod(lambda cls: home))
    return home


@pytest.fixture()
def workspace(tmp_path):
    """A scratch workspace directory with a few sample files."""
    ...
```

两条规矩：

1. **`isolated_home`：测试永不读写真实的 `~/.one-cedric`。**配置、会话、API key 全部重定向到临时目录——否则"跑一遍测试"就可能把你的真实配置改坏（`storage` 测试是第一个受益者）；
2. **`workspace`：每个端到端测试拿到一个全新的临时工作目录**，里面预置了 `notes.txt` / `data.json` / `rows.csv` / `sub/deep.txt`。工具测试（尤其写工具测试）再也不用"挑一个稳妥的系统临时目录不定哪天就踩坑"。

> 💡 **设计笔记**：fixture 不是"减少重复代码"的工具，是**规矩的执行器**。"测试不许碰真实配置"写在文档里没人记得住；写成 `isolated_home`、并在 review 里要求"读写 `~` 的测试必须带这个参数"，它就变成了可执行的规矩。

## 12.4 第一类：契约测试——250 个工具的关系网

`test_tool_registry.py` 的文档字符串说得很直接：

> The registry is the contract between the model, the permission layer and the implementation. A drift here silently breaks tools, so every relationship between `schema.py` and `_dispatch.py` is asserted.

它不问"`read_file` 能不能读文件"（那是端到端的事），只问"**250 个工具之间的 12 条关系**有没有断"。挑三条看：

```python
def test_write_and_readonly_sets_are_disjoint():
    """A tool must never be both write and read-only."""
    overlap = set(S.WRITE_TOOLS) & set(S.READONLY_TOOL_NAMES)
    assert not overlap, f"tools classified as both write and read-only: {sorted(overlap)}"


def test_every_tool_is_routable(names):
    """Each schema tool must reach an implementation."""
    unrouted = sorted(
        n for n in names
        if n not in _HANDLERS and not is_core_handled(n)
    )
    assert not unrouted, f"tools exposed to the model but not routed anywhere: {unrouted}"


def test_no_orphan_handlers(names):
    """A registered handler that no schema advertises is dead code."""
    orphan = sorted(set(_HANDLERS) - set(names))
    assert not orphan, f"handlers registered but not exposed to the model: {orphan}"
```

这三条各管一堵墙：

- **不重叠**：写集合 ∩ 只读集合 = ∅——它抓到了真实 bug：`image_process` / `pdf_metadata` / `sysop` 三个工具同时挂在两个集合里，导致 plan 模式（只允许只读工具）可以直接跑它们的**写入分支**。修复方式很工程：`WRITE_TOOLS` 永远赢，重叠的记入 `AMBIGUOUS_READONLY_WRITE_TOOLS` 名单留档（第 5 章 5.4）；
- **可达**：模型能看到的工具，必须有实现——要么在 `_HANDLERS` 里，要么 core 认领（`is_core_handled`）；
- **无孤儿**：注册了 handler 但 schema 没曝光 = 死代码——它抓到 `python_check` / `python_libs` 这类"声明了但没注册 handler"的反向漏网（详见 12.6）。

还有两条更细的：

```python
def test_cacheable_tools_are_read_only():
    leaked = sorted(set(S.CACHEABLE_TOOLS) - set(S.READONLY_TOOL_NAMES))
    assert not leaked, (
        f"cacheable tools must be read-only, otherwise a cached result could "
        f"hide a write: {leaked}")
```

**"可缓存的必须是只读的"**——如果缓存里存着一个写工具的结果，第二次调用会直接返回旧结果、**跳过确认与执行**。这类 bug 在运行时表现为"偶尔少改了一个文件"，几乎不可能靠手工复现；但写成不变量测试，它一天都活不下来。

**契约测试的写法特征**：它断言的是**集合之间的关系**，不是具体行为。所以它不关心 `read_file` 的实现改了几次、返回值格式变没变——它只在你**动了分类/注册/暴露关系**时报错。这正是"接口天天变"时代最便宜的保险：**行为测试会被重构冲垮，关系测试不会。**

## 12.5 第二类：安全测试——把审计结论"钉住"

10-11 那天 `SECURITY_AUDIT.md`（1214 行）和测试套件一起进了仓库。审计的方法论值得完整学一遍：

> - 审计对象：`F:\one-cedric3`（One Cedric，CLI + FastAPI WebUI 本地 AI Agent）
> - 审计方式：**只读源码审计**（`grep` 定位 sink → `read` 确认上下文），未修改任何既有文件、未安装依赖、未执行写操作
>
> **威胁模型**（按任务给定：**Agent 由初学者在本机运行，模型不可信**）：
>
> ```text
> 不可信模型输出（或模型读到的恶意文件/网页内容）
>         │
>         ├─→ 拼接进 WebUI DOM（v-html + marked）      → 浏览器上下文任意 JS
>         ├─→ 进入 bash / python_exec 工具参数          → 本机任意代码执行
>         └─→ 进入文件写入工具 path/content             → 工作目录外读写
> ```

最后一句是整个审计的轴心：**模型不可信**。在"单机自用、方便优先"的系统里，这一条经常被忘掉——审计就是要沿着"不可信输入到危险动作"的路径走一遍。

17 项发现，4 项"严重"，抽两条最典型的看：

### SEC-01：WebUI 的"自动确认"默认开着

```js
// one_cedric/gateway/webui/app.js:754 （修复前）
const autoApprove = ref(true);
```

CLI 里好容易建立的"写操作逐次确认"，在 Web 路径被一个默认 `true` 整个绕过去了。修复就一行：

```js
const autoApprove = ref(false);
```

同时修掉服务端版本（SEC-03：`/api/chat` 请求体里的 `auto_approve` 服务端不再无条件采信）。

### SEC-04：`python -c "..."` 属于"已知命令"，一次确认

沙箱终端对命令分四级（第 7 章）：白名单直跑、已知一次确认、未知强警告、黑名单永久拒绝。问题在于 `python -c "..."` 被判为 **known**——一次确认放行任意代码；更糟的是 `--prompt` 模式会隐式把 `auto_yes` 置为 True，确认次数变成**零**：

```python
# cli.py:173-174 （修复前）
if args.prompt:
    auto_yes = True
```

修复方案是一个"解释器内联执行"清单，把这类命令**强制降级为 unknown**：

```python
# 修复后：python -c / node -e / pwsh -Command / cmd /c / bash -c …
INLINE_EXEC_FLAGS = ...
# 判定到内联执行 → 直接返回 "unknown"，即使命令前缀在别的规则里"看起来"很乖
```

然后测试把它钉死（`tests/test_security.py`）：

```python
@pytest.mark.parametrize("command", [
    'python -c "import os; os.system(\'calc\')"',
    'node -e "require(\'child_process\').execSync(\'calc\')"',
    'powershell -Command "Start-Process calc"',
    'cmd /c "calc"',
    'bash -c "calc"',
    ...
])
def test_interpreter_inline_execution_is_never_whitelisted(command):
    level, reason = classify_command(command)
    assert level == "unknown", (
        f"{command!r} was classified {level!r} ({reason}); inline interpreter "
        f"execution must require a forced confirmation")
```

注意它旁边的"反向测试"：

```python
def test_ordinary_module_invocation_stays_whitelisted():
    """The fix must not break the legitimate `python -m pytest` shortcut."""
    assert classify_command("python -m pytest")[0] == "whitelist"
```

**修安全 bug 的测试必须成对**：一个钉"危险的一定拦住"，一个钉"合法的一定放行"。只测前者，你下次"加固"时很容易把 `python -m pytest` 一起关掉，然后某个下午发现"模型再也不能帮你跑测试了"。

### 修复清单：修了哪些、没修哪些

`47c9c19` 修掉了"当天可达"的 10 项：

| 编号 | 修复 |
| --- | --- |
| SEC-01 / 03 | WebUI 不自动确认（`autoApprove=false`）+ 服务端限制 |
| SEC-02 / 10 / 11 | `marked.parse` 结果全部过 `sanitizeHtml()` 白名单；stats 页改 `textContent` |
| SEC-16 | 内联脚本移入 `/static/boot.js`，换来严格 CSP + nosniff + DENY |
| SEC-04 | 解释器内联执行强制 unknown；`--prompt` 不再隐式 `auto_yes` |
| SEC-06 | 沙箱环境变量清洗扩展（`*_PASS` / `*_WEBHOOK*` / `*_DSN` / `SMTP*` …） |
| SEC-07 | `bash_bg` 与前台 shell 一样用清洗后的环境 |
| SEC-12 | `/api/tools/{name}/call` 拒绝 core 管的写工具（403） |

**没修**的也值得逐条知道（不是忘了，是成本/风险排序后的决定）：SEC-05（网关鉴权仍是逐路由手写调用——审计建议改成全局依赖）、SEC-08（API key 仍明文落盘；`/api/my-models` 仍把档案含 key 返回给浏览器）、SEC-09（解压校验）、SEC-13（MCP 子进程环境）、SEC-14（`open_app` 拼接）、SEC-15（`email_tools` 的 `shell=True`）、SEC-17（未接线模块，当前不可达但"接线前必须先修"）。

> ⚠️ **踩坑：修完安全，界面黑屏了。**（`ce13ff7`）SEC-16 把 CSP 收紧成 `script-src 'self'` 之后，WebUI 打开就停在启动画面——黑屏。排查过程值得全文看：先 `node --check app.js`（语法没问题）；再**用 jsdom 加载真实页面**（jsdom 不执行 CSP）——挂载完美、零错误。两头一夹，问题被隔离到 CSP：**WebUI 用的 Vue 是全局构建版，会在运行时用 `new Function` 编译模板**，`script-src` 没有 `'unsafe-eval'` 就抛异常、`mount()` 不完成、启动画面撤不掉。
>
> 最终修复：`script-src 'self' 'unsafe-eval'`。**XSS 的真正缓解不是 CSP 里的 `unsafe-eval` 挡的那部分，而是 `sanitizeHtml()` 白名单**——CSP 继续挡住内联脚本（`'unsafe-inline'` 依然没有），这才是注入场景的实弹。并且把"CSP 和 Vue 构建必须互相兼容"也写成了测试（`test_csp_allows_the_vue_build_that_is_shipped`），防止未来某天有人只改一边。

## 12.6 第三类：260 个工具的冒烟——把"全量"真的跑到

契约测试管"关系"，安全测试管"边界"，剩下的问题靠第三种测试兜底：**把每个工具都真的碰一下**。10-10 的 `TOOL_TEST_REPORT.md` 就是这件事的产物：

- 工具总数：**250**；
- 统计：`{"SKIP": 111, "OK": 85, "RUN-ERR": 30, "REG-ONLY": 24}`；
- 结论：发现并修复 **9 处真实 Bug**；其余未通过项均为**环境依赖**（缺第三方库/未安装外部工具/未配置服务），并非代码缺陷。

四个桶的分法就是这套冒烟测试的精华，**先分桶，再把能跑的真跑**：

| 桶 | 数量 | 规则 |
| --- | ---: | --- |
| ✅ `OK` | 85 | 无副作用、可在本机直跑 → **真的执行，看结果** |
| ⚠️ `RUN-ERR` | 30 | 真执行了但报错 → 逐个归因（缺库/缺服务/缺配置） |
| ⏭️ `SKIP` | 111 | 有副作用或依赖真实外部服务 → 只验证注册与参数解析 |
| 🔒 `REG-ONLY` | 24 | 写工具 → 除白名单外不实跑，只验证注册 |

它抓到的 9 个 bug 长这样（完整表在原报告）：

| # | 工具 | Bug | 性质 |
| --- | --- | --- | --- |
| 1 | `pdf_extract` | ImportError：`MAX_PDF_CHARS/MAX_PDF_PAGES` 在 config.py 不存在 | 常量缺失 |
| 2 | pdf 全家 | `_pypdf()` 失败返回 `(None, err)`，调用处解包 `None` 抛 TypeError | 错误路径本身是错的 |
| 3 | `archive` | core 引用了不存在的 `archive_op_v2` | 接线到不存在的函数 |
| 4 | `git` | 导入不存在的 `git_tool/_format_git_result` | 同上 |
| 5 | `file_attrs` | `from .tools import file_attrs_preview` 导入失败 | 相对导入写错 |
| 6 | qrcode 全家 | core 从 `one_cedric.tools` 导入不存在的 `qrcode_styled` 等 | re-export 缺失 |
| 7 | video/audio/pdf/compose/k8s 等 | 同上——`tools/__init__.py` 没补齐 re-export | re-export 缺失 |
| 8 | `python_check` / `python_libs` | schema 声明了但**没有注册任何 handler**，调用必报"未知工具" | 声明与实现脱节 |
| 9 | `file_attrs` `op=get/bulk` | handler 未实现这两个 op，掉进兜底被拒 | 分支缺失 |

看这个分布：**9 个里有 6 个是"接线问题"**（导入错、re-export 缺、函数名对不上），1 个是"声明了没实现"，只有 2 个是真的领域逻辑缺陷。这不是巧合——250 个工具×每个都要在 5 个地方登记（schema / 实现 / handler / re-export / 分类），**接线错误的概率远大于逻辑错误**。冒烟测试的第一价值不是查质量，是查**接线**。

> 💡 **设计笔记**：冒烟报告的价值在**归因**，不在通过率。30 个 `RUN-ERR` 里没有一个是代码缺陷——每一条都写了根因和修复命令（`pip install pypdf`、`winget install ffmpeg`、按提示配置 `lsp.toml`……）。**"测试失败"和"环境不满足"是两种完全不同的结论**，把它们混在一起报的测试报告没有用。

## 12.7 lint 与 CI：把"约定"变成"机器执行的规矩"

### 先清账：629 个 ruff 发现

`af22158` 的提交信息第一行就很诚实：

> The project had 629 ruff findings across 201 files and no way to gate on them.

处理方式是**分级**，而不是"全修完"：

- **阻断集（blocking）**：`E4,E7,E9,F,W`（pycodestyle 的语法级错误 + pyflakes 的"用了不存在的名字/导入"）。这些是**真 bug 探测器**，修完，并在 CI 里强制；
- **建议集（advisory）**：`I,UP,B,C4,SIM`（导入排序、语法现代化、bugbear 等风格的），单独跑、`continue-on-error`——保持可见，不阻断 PR；
- **自动修**：461 个安全/不安全 autofix（EOF 换行、无用导入、多余 f 前缀、空白）。

最能说明"lint 能抓真 bug"的是这一条：`notify_tool` 里计算了 `sound_ps`（根据 `sound` 参数）**却从未使用**——修掉形式是：

```python
-    sound_ps = "1" if sound else "0"
+    sound_line = ("[System.Media.SystemSounds]::Asterisk.Play()"
+                  if sound else "Start-Sleep -Milliseconds 1")
```

也就是说在 Windows Forms 降级路径上，`sound=False` 被**静默忽略**——用户点了一个"通知"，声音该关的没关。死代码检查抓到的不是美学问题，是行为问题。

### CI 全貌

`.github/workflows/ci.yml` 的逻辑值得逐段读：

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest]
        python-version: ["3.10", "3.11", "3.12", "3.13"]
        include:
          # The project is primarily developed on Windows; cover it once.
          - os: windows-latest
            python-version: "3.12"
```

**五个作业组合**：3.10–3.13 在 Ubuntu 全测，Windows 上补一个 3.12（"主开发平台至少过一次 CI"）。`fail-fast: false` 让四组独立并行、互不取消；顶上还有：

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

**同一个分支的新推送自动取消旧的一轮**——省 CI 分钟，也让"最新提交的结论"永远是队列里唯一在跑的。

测试作业的步骤顺序是刻意的：

1. `compileall -q one_cedric one_editor_lite`——**先验证"每个文件都能编译"**。语法错误一秒钟出结果，不必等依赖装好、不必进 pytest；
2. `ruff check .`（阻断）+ `--select I,UP,B,C4,SIM`（建议）+ 格式检查（建议）；
3. `pytest -q`；
4. 只在 3.12×Ubuntu 上额外跑一次 `--cov`，把覆盖率丢进 job summary（`continue-on-error`，不挡发布）。

还有一个专门防"版本漂移"的打包作业——构建 wheel，然后**断言 wheel 元数据里的版本号与代码里的 `_version.py` 一致**：

```yaml
- name: Assert the version comes from the package
  run: |
    python - <<'PY'
    import pathlib, subprocess, sys, tarfile, tomllib, zipfile
    expected = subprocess.run(
        [sys.executable, "-c",
         "import one_cedric._version as v; print(v.__version__)"],
        capture_output=True, text=True, check=True).stdout.strip()
    wheels = list(pathlib.Path("dist").glob("*.whl"))
    assert wheels, "no wheel produced"
    meta = tomllib.loads(
        zipfile.ZipFile(wheels[0]).read(
            [n for n in zipfile.ZipFile(wheels[0]).namelist()
             if n.endswith("METADATA")][0]).decode())
    assert meta["version"] == expected, (meta["version"], expected)
    print("version ok:", expected)
    PY
```

（顺带：这个作业还跑了 `python -m twine check dist/*`，检查包的元数据能不能过 PyPI 的门。）

### 另外三件"仓库级设施"

- **CodeQL**（`codeql.yml`）：对 `python` 和 `javascript-typescript` 两个语言跑安全扫描，push/PR 触发之外还有每周三 05:19 的定时任务——**没人改代码的星期也要扫**；
- **Dependabot**（`dependabot.yml`）：`pip`、`weekly`——依赖漏洞不靠人记得住;
- **Issue 模板 + SECURITY.md**：bug/feature 两个模板把"环境、复现、预期"的问句固化进表单；`SECURITY.md` 声明支持版本与报告渠道（它的版本表当时还是空的——你看，连官方仓库也有"小尾巴"，见本章练习）。

> 💡 **设计笔记**：CI 的第一价值不是"跑测试"，而是**让项目说的规矩变成机器执行的规矩**。"提交前跑一下 ruff"是愿望；`ruff check` 红了就合不进去是规矩。一处例外：**建议集故意不阻断**——把"能抓 bug 的规则"和"纯风格偏好"分开，前者硬拦、后者软提示，团队才不会被 lint 训练成"无脑 `--fix` 党"。

## 12.8 发布：0.17.0 / Public Beta I

### 版本的故事

版本号不是拍脑袋定的，git 历史里能读出它的来历：

```text
835388f 10-06 21:17 Public Beta 1      ← 一晚之内连打三枪
b4961f0 10-06 21:43 Public Beta I
6adf3dd 10-06 22:01 Public Beta I
7708fcb 10-07 15:13 Add some tools and improved WebUI
```

而更早的 `cf1d3ce 10-04 18:35` 提交信息只有三个字：`beta17`。于是版本号落在 **0.17.0**——数字是"beta 17"，名字升级为 **Public Beta I**（罗马数字，看起来正式一点——这个细节改了两次提交才定下来）。

### 单一版本来源

发布工程的第一课：**版本号只能有一个地方能改**。v3 把它放在一个刻意"没有任何 import"的文件里：

```python
# one_cedric/_version.py
"""Single source of truth for the project version. ..."""
__version__ = "0.17.0"

#: Human-facing release name shown in the UI and the banner.
__release_name__ = "Public Beta I"
```

为什么强调"没有 import"？因为它要同时被三方读取：`config.py`（不能有循环导入）、`pyproject.toml`（setuptools 构建时静态解析）、CLI 的 `--version`。实测输出：

```text
$ python -m one_cedric --version
One Cedric 0.17.0 (Public Beta I)
```

`pyproject.toml` 不写死版本，而是动态读取它：

```toml
[project]
dynamic = ["version"]
...
[tool.setuptools.dynamic]
version = { attr = "one_cedric._version.__version__" }
```

`test_packaging.py` 把这条链钉住——其中包括"用 AST 扫描 `_version.py` 不许出现任何 import"：

```python
def test_version_has_a_single_source():
    data = _load_pyproject()
    assert "version" not in data["project"], (
        "pyproject.toml declares a static version; it must use "
        "[tool.setuptools.dynamic] so _version.py stays the single source")
    ...
    assert pkg_version == leaf_version
    assert leaf_version == config.PROJECT_VERSION
```

同一屋檐下的还有一批"打包卫生"检查：声明的包目录必须真实存在（`test_declared_packages_exist_on_disk`）、运行时依赖里不许出现开发工具（`pytest`/`ruff`/`black` 在 `requirements-dev.txt`）、**不许写错分发名**（`bs4`→`beautifulsoup4`、`docx`→`python-docx`、`msoffcrypto`→`msoffcrypto-tool`——这类错在本机装了旧包的环境里"看起来能用"，一到新机器上就装不上）。

### 文档里的数字会腐烂

发布时 `README.md` 写着：

```bash
python -m pytest -q              # 247 tests
```

而写作本章时在干净检出上实测：**249 个**。两个测试在之后两次修复中悄悄加进来，README 没跟上。这不是谁的错——**是精确数字写进文档的必然结局**。

处理原则：**要么自动生成（不要把数字写进文档），要么接受它会过期**。本章所有数字之所以可信，是因为它们是写作时现跑的——你读到这里时，它们可能也已经过期了；**方法（怎么数）比数字（数出来多少）活得久**。

### 一份可复制的发布清单

按 v3 最后一天的提交顺序整理：

1. **清仓库**：构建产物、字节码缓存、vendored `node_modules` 全部 untrack；`.gitattributes` 统一换行（`53266d9`）；
2. **查"新用户路径"**：真的 `git clone` 到一个新目录跑一遍——10-11 那次就是这样发现两个从未提交的文件（`0a0cf2f`）；
3. **依赖体检**：运行时 / 可选 / 开发三份 requirements 分离，分发名逐个核对（`dfd709d`）；
4. **版本单一来源**：`_version.py` + dynamic version + `--version` 输出核对；
5. **补测试并把它抓到的 bug 全修**（`188421a`；9 个 bug 的账在 12.5 和 12.6 里对过了）；
6. **跑一遍安全审计**，能修的当天修、不能修的写成清单并注明理由（`47c9c19`）；
7. **CI 门禁**：compileall → 阻断 lint → 测试 → 打包版本断言；
8. **文档诚实**：README 的功能清单、测试数、已知问题（那份没修的安全清单就是最好的"已知问题"）；
9. **提交信息写清"为什么"**——v3 的提交信息是这本书很多章节的第一手材料，未来的你也会感谢现在的你。

## 🔧 自己动手

1. **写第一个契约测试**：当作 250 个工具不存在，为你自己的小项目写一条"关系测试"——比如"每个注册的命令都出现在帮助文本里""每个配置项都出现在默认值表里"。体会它和"测某个函数返回什么"的区别；
2. **把手工回归表变成 parametrize**：找一张你写过的"发版前手工检查清单"，把它翻译成 `@pytest.mark.parametrize` 的输入/期望对；
3. **搭最小 CI**：只做三件事——`compileall`、`pytest`、阻断集 lint。跑通之后再加打包版本断言；
4. **做一次只读安全审计**：照 12.5 的格式——先写威胁模型（"谁不可信、他能到哪"），再沿三条链路 grep，每条发现写"证据（文件:行）+ 建议修复 + 是否可达"；**审计过程中不要改任何代码**；
5. **修一条本章留下的尾巴**：从 12.5 的未修清单里挑一条真修（比如 SEC-08：`/api/my-models` 返回时把 `api_key` 剥掉），**然后补一个成对的测试**——一个钉"Key 不出现在响应里"，一个钉"合法的模型切换仍然工作"。

---

[← 上一章：11 记忆、自动化与生态](11-memory-automation-ecosystem) ｜ [返回目录](Home) ｜ [下一章：附录 →](99-appendix)
