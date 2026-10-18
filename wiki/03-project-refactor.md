# 03 · 拆包与工程化：从脚本到项目

## 3.1 触发点：一句"拆分"

📅 **开发日志 · 2026-09-19 09:41**（对话 #67）

> **用户**：拆分

此时距离第一个单文件 Agent 诞生还不到六天。单文件已经长到了近千行：工具定义、实现、模型调用、循环、CLI 全挤在 `agent.py` 里。再加两个工具还能忍，但作者已经决定要做成"产品"——产品不能只有一个文件。

这一章讲的就是：**怎么把一个能跑的脚本，拆成一个能长大的项目**。以及拆的过程中，Python 的导入系统会怎么教训你。

## 3.2 拆完之后的目录

下面是一份真实的早期快照（v1，项目已改名为 One Cedric，但结构在 One Copilot 时期就已定型）：

```text
one-cedric/
├── run.py                     # 开发期启动器
├── pyproject.toml             # 打包与入口点
└── one_cedric/
    ├── __init__.py            # 包入口
    ├── __main__.py            # python -m one_cedric
    ├── cli.py                 # argparse + 主菜单 / REPL
    ├── config.py              # 常量、主题、SYSTEM_PROMPT、默认配置
    ├── core.py                # OneCedric 类：agent_turn / 权限 / 会话
    ├── ui.py                  # Rich 渲染、动画、banner（后期拆成 ui/ 包）
    ├── storage.py             # 配置与会话读写
    ├── cache.py               # 工具结果 LRU 缓存
    ├── at_ref.py              # @文件引用展开
    ├── git_snap.py            # git 快照
    ├── tools/
    │   ├── __init__.py        # 工具层总入口（re-export）
    │   ├── schema.py          # 所有工具的 JSON Schema
    │   ├── sandbox.py         # 路径沙箱
    │   ├── readonly.py        # 只读工具：read_file / list_files / grep...
    │   ├── write.py           # 写工具：write_file / edit_file
    │   ├── fileops.py         # 批量文件操作
    │   ├── shell.py           # bash 策略
    │   ├── net.py             # web_fetch / web_search
    │   ├── data.py            # json / csv / sqlite
    │   ├── dev.py             # git / docker / archive / diff
    │   ├── text.py            # find_replace / regex / template
    │   ├── system.py          # env / process / clipboard
    │   └── ...
    ├── computer/              # computer use（截屏/鼠标/键盘）
    └── lsp/                   # LSP 客户端与工具
```

### 拆分的第一原则：按"变化的原因"分文件

不是按"函数数量平均分"，而是按**谁会因为什么原因修改**来分：

- 工具 schema 会频繁增删 → 独立成 `schema.py`；
- 每个工具的实现相对独立 → 按领域分 `readonly/write/net/data/...`；
- 权限、循环、会话总是同时变 → 留在 `core.py`；
- 终端渲染会反复调整 → 独立成 `ui.py`（后来甚至独立成 `ui/` 包）。

这个原则在工具涨到 240+ 时救了大命：新增一个 PDF 工具，只需要动 `schema.py` + `tools/pdf.py` + 一个 handler，循环代码一行不改。

## 3.3 第一个跟头：相对导入

拆包之后，作者改完代码，运行：

📅 **2026-09-19 16:01**（对话 #67，用户粘贴）

```text
PS F:\one-copilot> python -m run.py
Traceback (most recent call last):
  File "<frozen runpy>", line 189, in _run_module_as_main
  ...
  File "F:\one-copilot\run.py", line 1, in <module>
    from cli import main
  File "F:\one-copilot\cli.py", line 8, in <module>
    from .config import DEFAULT_HOST, DEFAULT_MODEL
ImportError: attempted relative import with no known parent package
```

### 为什么会这样

`cli.py` 里写的是：

```python
from .config import DEFAULT_HOST, DEFAULT_MODEL
```

`from .` 的意思是"从**当前包**里导入"。但当你直接运行 `run.py`（或 `python -m run.py`）时，`cli.py` 是被当作一个**顶层脚本**加载的，它没有"当前包"这个概念——于是 Python 报"没有已知的父包"。

三个概念要分清：

| 写法 | 含义 | 何时可用 |
| --- | --- | --- |
| `from .config import X` | 相对导入：同一包内 | 模块必须作为包的一部分被导入 |
| `from one_cedric.config import X` | 绝对导入：从项目根开始 | 只要项目根在 `sys.path` 里 |
| `from config import X` | 顶层导入 | 仅当 `config.py` 与入口在同一目录、且该目录在 `sys.path` 里 |

### 正确的修法

**方案一（推荐）：入口文件用绝对导入，包内模块用相对导入。**

```python
# run.py（开发期启动器）
from one_cedric.cli import main

if __name__ == "__main__":
    main()
```

```python
# one_cedric/cli.py（包内）
from .config import DEFAULT_HOST, DEFAULT_MODEL   # ✅ 作为包的一部分被导入
```

然后在项目根目录运行：

```bash
python run.py            # ✅ run.py 能 import 到 one_cedric 包
python -m one_cedric     # ✅ 更标准：让 Python 自己把包当模块执行
```

**方案二：把包安装成可编辑模式，使用命令行入口。**

```toml
# pyproject.toml
[project.scripts]
one-cedric = "one_cedric.cli:main"
```

```bash
pip install -e .
one-cedric --help
```

One Cedric 最终两种方式都支持：`run.py` 给"不想安装、clone 下来就能跑"的用户；`python -m one_cedric` 和 `one-cedric` 命令给正式安装的用户。v3 的 `run.py` 还额外做了一件事——把项目根和本地依赖目录 `.pylibs` 插入 `sys.path`：

```python
# run.py（v3）
sys.path.insert(0, str(Path(__file__).parent))
_libs = Path(__file__).parent / ".pylibs"
if _libs.is_dir():
    sys.path.insert(0, str(_libs))

from one_cedric.cli import main
```

> 💡 **设计笔记**：包的"入口"和"实现"要分开。入口（`run.py` / `__main__.py` / console script）只负责找到包并调用 `main()`；实现内部一律用相对导入。这样无论从哪个入口进来，行为都一致。

## 3.4 第二个跟头：`\x` 转义

📅 **2026-09-19 16:40**（对话 #67，用户粘贴）

```text
PS F:\> python one-copilot/run.py
Traceback (most recent call last):
  ...
  File "F:\one-copilot\one_copilot\config.py", line 153
    SYSTEM_PROMPT = """你是 One Copilot，一个可以操作本地文件的命令行助手。
                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
SyntaxError: (unicode error) 'unicodeescape' codec can't decode bytes in position 7895-7896: truncated \xXX escape
```

### 原因

`config.py` 里有一段普通的三引号字符串，里面写了 Windows 路径，比如：

```python
SYSTEM_PROMPT = """...
所有文件路径都用相对工作目录的形式（如 C:\Users\...）
..."""
```

Python 会把 `\U`、`\x` 当作转义序列解析。`C:\Users` 里的 `\U` 后面不是 8 位十六进制，于是报 `truncated \xXX escape` 或 `invalid \UXXXXXXXX escape`。

### 修法

```python
SYSTEM_PROMPT = r"""...
所有文件路径都用相对工作目录的形式...
..."""
```

加一个 `r` 前缀，字符串里的反斜杠就不再被当作转义。v3 的 `config.py` 至今保留着这个写法：

```python
SYSTEM_PROMPT = r"""你是 One Cedric，...
```

> ⚠️ **踩坑提醒**：Windows 上的 Agent 项目，**所有包含路径的提示词和正则都用 raw string**。这个坑会在你每次改 prompt 时反复出现。

## 3.5 配置集中：一个 `config.py` 管什么

拆包后，`config.py` 成了"全局唯一真相源"，它通常包含五类东西：

```python
# 1) 默认连接参数
DEFAULT_HOST = os.environ.get("OLLAMA_HOST", "http://localhost:11434")
DEFAULT_MODEL = os.environ.get("OLLAMA_MODEL", "qwen2.5:7b")

# 2) 行为限制
MAX_BYTES = 200_000
MAX_LINES = 2_000
MAX_SEARCH_RESULTS = 50
DEFAULT_SHELL_TIMEOUT = 30

# 3) 主题色（Rich 用）
BRAND = "#d97757"
ACCENT = "#4ec9b0"
THEME = Theme({...})

# 4) 系统提示词
SYSTEM_PROMPT = r"""..."""

# 5) 用户配置的默认值
DEFAULT_CONFIG = {
    "default": {
        "model": DEFAULT_MODEL,
        "max_steps": 8,
        "show_reasoning": True,
        ...
    }
}
```

这个文件后来长到了 478 行（v3），但结构没变。经验是：**凡是"你可能想改"的东西，都不要散落在业务代码里**。

## 3.6 版本号：从 Beta 15 到 Public Beta I

📅 **2026-09-18 22:32**（对话 #67）

> **用户**：ok，现在以 beta 加数字

于是界面上的版本号变成了 `One Copilot · Beta 15` 这样的形式。早期它是硬编码在 banner 里的字符串；到了 v3，版本号收敛到一个**无依赖**的小模块：

```python
# one_cedric/_version.py
__version__ = "0.17.0"
__release_name__ = "Public Beta I"
```

```python
# one_cedric/config.py
from ._version import __release_name__, __version__

VERSION_DISPLAY = f"{__version__} · {__release_name__}"
```

为什么单独一个文件？注释里写得很清楚：**避免循环导入**（`config` 和 `__init__` 都要读版本号），同时让 `setuptools` 能在构建时静态读取版本（`[tool.setuptools.dynamic]`）。

> 💡 **设计笔记**：版本号是典型的"到处都要用、但不该有依赖"的数据。给它一个零依赖的模块，比塞进 `config.py` 更干净。

## 3.7 第三个跟头：模块搬家，导入没跟上

这是整个开发过程中最典型的一类事故。2026-09-27，为了接入 computer use，作者移动/新增了一批模块，然后连续三次启动失败：

📅 **2026-09-27 16:04**（对话 #67）

```text
File "F:\one-cedric\one_cedric\lsp\tools.py", line 8, in <module>
    from ..sandbox import _as_int, _resolve_path
ModuleNotFoundError: No module named 'one_cedric.sandbox'
```

📅 **16:05**

```text
File "F:\one-cedric\one_cedric\core.py", line 63, in <module>
    from .computer import SafetyPolicy, set_policy as _set_computer_policy
ImportError: cannot import name 'set_policy' from 'one_cedric.computer'
```

📅 **16:06**

```text
File "F:\one-cedric\one_cedric\tools\__init__.py", line 2, in <module>
    from .backend import ComputerBackend, BackendError, get_backend, reset_backend
ModuleNotFoundError: No module named 'one_cedric.tools.backend'
```

三个错误，三种原因：

1. `lsp/tools.py` 以为 `sandbox.py` 在包根（`..sandbox`），实际在 `tools/sandbox.py`。修法：改成 `from ..tools.sandbox import ...`（v1 现码即如此）；
2. `computer/__init__.py` 没有把 `tools.py` 里的 `set_policy` 再导出。修法：在 `__init__.py` 里补上 `set_policy, get_policy` 并加入 `__all__`；
3. `tools/__init__.py` 还在导入一个已经搬到 `computer/backend.py` 的模块。修法：删掉这条导入，backend 归 computer 包管。

### 教训

**移动文件 = 一次全局重构。** 正确的流程是：

1. 移动/重命名文件；
2. 全仓库搜索旧导入路径（`grep -rn "from ..sandbox"`）；
3. 跑一次"导入冒烟测试"：`python -c "import one_cedric"`；
4. 再跑 CLI 的 `--help`，确保入口链路完整。

One Cedric 后来把这条写进了开发习惯：**任何结构性改动，先 `python -c "import one_cedric"`，再谈功能。**

## 3.8 包结构演进：ui.py → ui/ 包

同样的"长大"也发生在 UI 上。v1 还是一个 56KB 的 `ui.py`；v3 已经变成：

```text
one_cedric/ui/
├── __init__.py
├── legacy.py        # 早期 ui.py 的兼容层
├── live_input.py    # 底部固定输入框（鼠标、编辑、中途插话）
├── confirm.py       # 5 选项确认界面
├── progress.py      # 进度矩阵 / 多任务动画
├── theme.py         # 主题
├── dog.py           # 像素小狗
├── icons.py         # 图标
├── menu.py          # 主菜单
├── screens/         # models / tools / integrations 等管理界面
└── ...
```

`legacy.py` 这个命名很诚实：**重构不必一次到位**。把旧实现保留为兼容层，新代码逐步迁移，比"停业整顿式重写"安全得多。

## 3.9 工程化的检查清单

拆包完成后，一个 Python Agent 项目至少应该具备：

- [x] `pyproject.toml`：依赖、入口点、版本来源；
- [x] `run.py` 与 `python -m package` 两种启动方式；
- [x] 配置集中（`config.py` + 用户配置文件）；
- [x] 包内相对导入、入口绝对导入；
- [x] 一个"导入冒烟测试"（`python -c "import pkg"`）；
- [x] 版本号单一来源；
- [x] `requirements.txt` 分运行时 / 可选 / 开发三份（v3 的做法）。

## 🔧 自己动手

1. 把你第 2 章的 `agent.py` 拆成 `myagent/` 包：`config.py`、`tools/`、`core.py`、`cli.py`、`__main__.py`；
2. 故意制造一次 `ImportError`，然后按 3.7 的流程修复；
3. 给 `pyproject.toml` 加一个 `[project.scripts]` 入口，`pip install -e .` 后直接用命令启动；
4. 把 `SYSTEM_PROMPT` 改成 `r"""`，并在里面写一个 Windows 路径，验证不再报转义错误。

---

[← 上一章：02 第一个 Agent](02-minimal-agent) ｜ [返回目录](Home) ｜ [下一章：04 终端 UI 进化 →](04-rich-ui)
