# 09 · 十条需求与批量交付：一个人怎么在一周内长出 200 个工具

## 9.1 那个上午的十条需求

📅 **开发日志 · 2026-10-01 09:14**（对话 #69）

> 好了，我们现在增加一下需求，请你根据我给你提供的内容自己思考一下，给我一个具体的方案。
>
> 1. 我们需要能够操作系统功能的 tools，包含调节音量，深浅色模式，检查更新，调节亮度，打开关闭 wlan，蓝牙，连接网络，连接设备，打印东西等等 等等功能
> 2. 我们需要借助 Pythondocx，win32com，mailmerge 库来实现对 word 文档的编辑，读取，加密，合并，打印，提取图片等等功能
> 3. 我们需要使用 pypdf 对于 pdf 文件的读取，编辑，加密，合并，打印，OCR，提取图片等等功能
> 4. 我们需要使用 pythonpptx 库，来实现对 pptx 的编辑读取加密合并，打印，提取内容，添加删除动画等等功能
> 5. 我们需要使用 openpyxl 库，来实现对 XLSX 的编辑读取加密合并，公式，工作簿等等功能
> 6. 优化用户界面……确认界面为 1.yes 换行 2.yes and allow 什么 tools in this session 换行 3.no 4.edit 换行 5.help
> 7. 优化 shell tool，支持任意命令，未知命令需要用户确认
> 8. 新增 gateway，Claude 风
> 9. 支持模型 api 地址和 key 在内部配置
> 10. 再新增 tool 管理界面
>
> 本次改动较大，请你先生成一个合理的方案给我审核

两分钟后：

> **09:16 用户**：ok

方案通过，开始施工。这一天结束时，工具数从 49 涨到 200 多，还多了 Gateway、WebUI、系统操作、Office 四件套。这一章讲**这套"批量交付"工作流**——它不是工程规范，而是"一个人 + 一个 AI"在有限时间里把事做完的现实方法。

## 9.2 先方案，后代码

用户的第一条指令里有一句关键的话："**请你先生成一个合理的方案给我审核**"。这决定了后面所有工作的节奏：

```text
需求（10 条） → 方案（分批、依赖、风险） → 用户审核 → 逐批交付 → 用户集成测试 → 反馈修 bug
```

为什么不能直接开写？因为 10 条需求之间有依赖：

- Office 四件套依赖 `office/common.py` 的路径与格式校验；
- `sysop` 依赖 `sysops/_platform.py` 的平台判断；
- Gateway 依赖 Core 的 `_stream_hook` 与确认回调；
- WebUI 依赖 Gateway 的 SSE 接口。

先写 Gateway 再写 Core 的 hook，等于返工。方案的价值不是"文档好看"，而是**把返工提前到纸面上**。

## 9.3 交付载体：PowerShell 安装脚本

这个项目最特殊的一点：代码不是通过 git push 交付的，而是通过**一段段 PowerShell 脚本**。每批交付一个 `install_*.ps1`，用户双击运行，文件就落盘了。

脚本的骨架（`install_05de2_tools_misc2.ps1`）：

```powershell
# ============================================================
# One Cedric - Step 5de2: 工具实现
# dev / files / pdf / linters / archive
# 用法: .\install_05de2_tools_misc2.ps1
# ============================================================

$ErrorActionPreference = "Stop"
Write-Host ""
Write-Host "=== Step 5de2: dev/files/pdf/linters/archive ===" -ForegroundColor Cyan

function Write-Utf8File {
    param([string]$Path, [string]$Content)
    $dir = Split-Path $Path -Parent
    if ($dir -and !(Test-Path $dir)) {
        New-Item -ItemType Directory -Force -Path $dir | Out-Null
    }
    $full = Join-Path (Get-Location) $Path
    $utf8 = New-Object System.Text.UTF8Encoding $false
    [System.IO.File]::WriteAllText($full, $Content, $utf8)
    Write-Host "  + $Path"
}

# ─── tools/dev.py ───────────────────────────────
Write-Utf8File "one_cedric\tools\dev.py" @'
"""开发辅助：diff_files。"""
from __future__ import annotations
...
'@
```

四个关键细节，每一个都来自真实的 Windows 踩坑：

### ① `UTF8Encoding $false`：不要 BOM

PowerShell 5 的 `Out-File -Encoding utf8` 会写 **BOM**（`EF BB BF`）。Python 解释器能容忍 BOM 的 `.py` 文件，但有些工具（`ast.parse`、部分 linter、拼接脚本）会把 BOM 当成非法字符。`New-Object System.Text.UTF8Encoding $false` 的 `$false` 就是"不要 BOM"。

### ② `@' ... '@`：字面 here-string

单引号 here-string **不做变量替换**。这很重要：Python 代码里有大量 `$`（正则、f-string 不太有，但 shell 脚本、模板里有），双引号 here-string 会把 `$var` 展开成空字符串。用 `@'` 才能原样写入。

### ③ `$ErrorActionPreference = "Stop"`

默认情况下 PowerShell 遇到错误会继续执行，结果是"脚本跑完了，但只写了一半文件"。设成 `Stop` 让第一个错误就中断——**宁可停在半路，也不要留下一个看似完整、实际缺文件的工程**。

### ④ 幂等覆盖

`WriteAllText` 直接覆盖同名文件。重复运行同一批脚本不会产生副作用——这对"用户跑一半失败了再跑一遍"的场景是必需的。

> 💡 **设计笔记**：为什么不用 git？因为目标用户是"刚学 Python、在 Windows 上照着聊天窗口操作"的人。让他装 git、配 SSH、理解 branch，成本远高于双击一个 `.ps1`。**交付方式要匹配用户的能力，而不是匹配工程师的审美**。

## 9.4 交付顺序：依赖驱动的批次

`install_05de2` 的结尾写着下一步：

```text
下一步: .\install_05f_tools_media.ps1

之后：05g（email / amap / weather / geolocate / wechat / notify /
            apps / orchestration）
   → 05h（subagent / multi_review / memory / cron / dream /
            database / fileattrs）
   → 06（office + sysops）
   → 07（gateway + webui）
   → 08（lsp + computer + 收尾）
```

完整顺序与内容：

| 批次 | 文件 | 内容 | 依赖 |
| --- | --- | --- | --- |
| `05de2` | dev / files / pdf / linters / archive | 基础工具补齐 | 无 |
| `05f` | python_exec / video / audio / pdf_advanced / sysinfo / network_diag | 执行与媒体 | 05de2 的 sandbox |
| `05g` | email / amap / weather / geolocate / wechat / notify / apps / orchestration | 外部服务与系统 | 配置系统 |
| `05h` | subagent / multi_review / memory / cron / dream / database / fileattrs | Agent 生态 | core 的 state |
| `06` | office + sysops | 办公四件套、系统操作 | 05g 的 apps |
| `07` | gateway + webui | HTTP 网关、浏览器 UI | core 的 hook |
| `08` | lsp + computer | 代码智能、桌面自动化 | 05f 的 python_exec |

顺序背后的原则：**先底层后上层，先无状态后有状态**。`sandbox` 在最前，因为所有文件工具都要用它；`gateway` 在最后，因为它要调用前面所有东西。

## 9.5 案例 A：一个 `sysop` 工具，二十个操作

十条需求的第一条是"操作系统功能"。直觉做法是加 20 个工具：`volume_set`、`volume_get`、`brightness_set`……但 One Cedric 只加了**一个** `sysop`，用 `op` 参数区分：

```python
READONLY_OPS = {
    "volume_get", "brightness_get", "theme_get",
    "wifi_status", "wifi_list",
    "bt_status", "bt_list",
    "printer_list", "printer_default",
    "check_update",
}

def sysop_op(root: Path, args: dict):
    """统一入口。只读 op 返回 str，写 op 返回 (preview, is_write, err)。"""
    op = str(args.get("op", "")).lower()

    if op == "volume_get":
        return volume_get()
    ...
    if op == "volume_set":
        v = args.get("value")
        return f"将设置音量为 {v}%", True, ""
    if op == "wifi_connect":
        ssid = args.get("ssid", "")
        return f"将连接到 WLAN: {ssid}", True, ""
    ...
    return "", False, f"ERROR: 未知 sysop op: {op}"
```

这个设计的精髓在返回类型：

- **只读 op 返回 `str`**：直接作为工具结果；
- **写 op 返回 `(preview, is_write, err)` 三元组**：`preview` 是"将要做什么"的中文描述，`is_write=True` 告诉 core 必须走确认流程；
- **未知 op 返回错误**，不抛异常。

一个工具名、一份 schema、一个 `op` 枚举，换来 20 个能力。好处：

1. **模型只需要记住一个工具**，选择成本低；
2. **确认界面统一**：所有系统操作都显示"将设置音量为 50%"这样的预览；
3. **权限粒度清晰**：`READONLY_OPS` 里的 op 不需要确认，其余都要。

代价是 schema 的 `op` 枚举很长，描述要写清楚每个 op 的参数。但相比 20 个工具名的认知负担，这笔交易划算。

> ⚠️ **踩坑**：`sysop` 同时出现在 `READONLY_TOOL_NAMES` 和 `WRITE_TOOLS` 里（第 5 章的歧义集合）。修复方式是"写集合获胜"，再在 `_classify_is_write` 里按 `op` 判断——`volume_get` 只读，`volume_set` 写。**单工具多 op 的模式，必须配套运行时的 op 级判定**，否则权限会一刀切。

## 9.6 案例 B：Office 四件套的"薄壳"设计

十条需求里的 2–5 条是 Word / PDF / PPT / Excel。实现没有把 `python-docx`、`pypdf`、`python-pptx`、`openpyxl` 的 API 直接暴露给模型，而是每个库一个"薄壳"模块：

```text
tools/office/
├── common.py    路径校验、格式检查、输出目录
├── word.py      docx：读 / 写 / 加密 / 合并 / 打印 / 提图
├── pdf.py       pdf：读 / 编辑 / 加密 / 合并 / OCR / 提图
├── pptx.py      pptx：读 / 编辑 / 加密 / 合并 / 打印 / 动画
└── excel.py     xlsx：读 / 编辑 / 加密 / 合并 / 公式 / 工作簿
```

schema 里只有四个工具（`docx` / `pdf` / `pptx` / `excel`），每个都带 `op` 参数——和 `sysop` 同一模式。`common.py` 负责三件事：

1. **路径必须落在工作目录内**（调用 `_resolve_path`）；
2. **扩展名与工具匹配**（`docx` 工具不接受 `.pdf`）；
3. **输出文件默认写到源文件旁边**，并返回相对路径。

薄壳的边界很清楚：**只做"模型能理解的操作"，不做完整 API 映射**。比如 Word 的"添加删除动画"这种需求，`python-docx` 根本不支持，薄壳就直接返回"不支持"，而不是硬凑一个半成品。

## 9.7 案例 C：配置驱动的外部服务

邮件、高德、微信、通知这些工具有一个共同点：需要用户自己的凭证。它们的实现模式是**配置驱动 + 优雅降级**：

```text
未配置 API key → 工具返回 "ERROR: 未配置 amap key，请在设置中填写"
已配置         → 正常请求
请求失败       → 返回 HTTP 状态码 + 响应片段，不抛异常
```

这样模型不会因为"没配 key"而崩溃，而是会把错误转述给用户："需要先在设置里配置高德 API key"。用户看到的是**可行动的提示**，不是堆栈。

高德工具有 14 个（地理编码、POI 搜索、路径规划、天气、行政区……），全部走同一个 `amap.py` 客户端，共享 key、超时、重试逻辑。

## 9.8 "不要分批"：一次给全

📅 **10-02 09:45**：

> **用户**：ok，不要分批

此前的交付是"一批脚本 → 用户跑 → 反馈 → 下一批"。到 10-02，剩余工作量已经清晰，用户要求**一次性给出全部文件**。这带来两个变化：

1. **单条消息变得极长**：几十个文件、上万行代码，全部用 here-string 嵌在脚本里；
2. **测试责任转移**：用户拿到的是完整工程，需要自己按"测试清单"逐项验证。

这也解释了 v3 仓库里为什么有 `TOOL_TEST_REPORT.md`（250 个工具的冒烟结果）和 `SECURITY_AUDIT.md`——当交付变成"一次性全量"，**验证就必须从"边交付边测"变成"交付后系统测"**。

> 💡 **设计笔记**：分批与全量没有绝对优劣。分批的优点是每批可验证、错误早发现；全量的优点是依赖一次到位、不会出现"半成品工程"。选择标准是：**变更之间是否强耦合**。十条需求彼此依赖，分批反而容易留下"import 一个还不存在的模块"的中间态——这也是 09-27 那三次导入错误的根源。

## 9.9 批量交付的七条经验

1. **脚本用 `$ErrorActionPreference = "Stop"`**：错就停，不留半成品；
2. **UTF-8 无 BOM**：`UTF8Encoding $false`；
3. **here-string 用单引号**：避免 `$` 被展开；
4. **文件路径用相对路径**：脚本在项目根目录运行，`Join-Path (Get-Location)`；
5. **每批结尾打印下一步**：用户不需要记顺序；
6. **每批都自带测试清单**：至少列出"运行后应该看到什么"；
7. **覆盖式写入**：允许重复执行，失败后可以重跑。

## 9.10 这一周的得与失

**得**：

- 工具从 49 到 242，覆盖办公、系统、邮件、地图、编排；
- 形成了"schema → handler → dispatch"的稳定扩展模式，新增工具的成本降到"三处修改"；
- Gateway/WebUI 让 agent 从终端走进浏览器。

**失**：

- 一次性交付大量文件，导致多次"模块不存在 / 循环导入"（第 3、5 章的三次翻车）；
- 测试滞后，250 个工具里 111 个只能 SKIP、30 个 RUN-ERR（第 12 章）；
- 安全审计在功能完成后才做，发现了 4 个严重问题（第 7 章）。

这些"失"不是流程错误，而是**一个人赶进度的必然代价**。教程把它们写出来，不是让你复制错误，而是让你知道：**当你选择"先做出来"时，要预留"再收拾"的时间**。

## 🔧 自己动手

1. **写一个 `install.ps1`**：用 `Write-Utf8File` + 单引号 here-string，把三个 Python 文件写到指定目录，验证无 BOM、可重复执行；
2. **设计一个多 op 工具**：比如 `image`，只读 op 有 `info`，写 op 有 `resize` / `crop` / `convert`，按 `sysop` 的模式返回 `(preview, is_write, err)`；
3. **做一次"十条需求"拆解**：任选一个你熟悉的产品，列 10 条需求，画出依赖图，排出 5 个交付批次；
4. **模拟一次失败重跑**：在脚本中途故意写一个不存在的路径，确认 `Stop` 生效，再修好重跑，验证覆盖式写入。

---

[← 上一章：08 Agent 循环工程](08-agent-loop-engineering) ｜ [返回目录](Home) ｜ [下一章：10 网关与 WebUI →](10-gateway-webui)
