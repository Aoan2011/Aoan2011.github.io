# 从 0 写一个 Agent · One Cedric 开发手册

> **一部基于真实开发过程的 Agent 教程。**
>
> 主人公：**One Cedric** —— 一个由高中生开发者 [@Aoan2011](https://github.com/Aoan2011) 从零写起的本地 AI Agent：**240+ 工具、CLI + WebUI 双界面**、Python 3.10+、GPL-3.0 开源。
>
> 教程素材：三份开发快照（v1 / v2 / v3）+ 完整开发对话记录（2026 年 9 月 – 10 月）。

---

## 这是什么

这不是一份"教你调 API"的速查手册，而是一本**开发实录级教程**：

- **从 0 开始**：第 2 章你会亲手写出第一个能用的 Agent（约 500 行 Python，不依赖任何 Agent 框架）；
- **一路走到产品**：之后每一章都对照 One Cedric 的真实仓库代码，拆解一个大项目的每一层——工具系统、流式协议、权限安全、网关、WebUI、记忆与自动化、测试与发布；
- **带日期的一手材料**：每个关键设计都附有当时的开发日志（原话与真实报错）。你能看到"**为什么**这么设计"，而不只是"设计**是**什么"。

## 目录

### 第一部分 · 起步

| 章节 | 内容 |
| --- | --- |
| [00 前言与阅读指南](00-preface) | 教程缘起、素材说明、阅读路线图 |
| [01 概念篇：Agent 到底是什么](01-concepts) | Agent vs 聊天机器人、工具调用协议、环境准备、时间线总览 |
| [02 第一个 Agent：500 行起步](02-minimal-agent) | 从头写一个能读文件、会调工具的 CLI Agent（完整代码逐段讲解） |

### 第二部分 · 把脚本做成软件

| 章节 | 内容 |
| --- | --- |
| [03 拆包与工程化](03-project-refactor) | 从单文件到包结构；相对导入、配置集中、版本号 |
| [04 终端 UI 进化：Rich 改造](04-rich-ui) | 流式渲染、思考面板、工具动画、像素小狗 |
| [05 工具系统：从 4 个到 240 个](05-tool-system) | schema / handler / dispatch 三层设计；只读与写操作的分类 |
| [06 流式与推理](06-streaming-reasoning) | SSE 增量重组、reasoning_content 兜底、think 档位、token 统计 |

### 第三部分 · 一个 Agent 的自我修养

| 章节 | 内容 |
| --- | --- |
| [07 权限与安全](07-permissions-safety) | 5 选项确认、三层 shell 策略、沙箱、安全审计 17 项 |
| [08 Agent 循环工程](08-agent-loop-engineering) | 并行只读、缓存、去重、参数校验、中途插队 |
| [09 十条需求与批量交付](09-mega-batch) | 十条需求 → 八批交付；办公/系统/邮件/位置工具 |
| [10 Gateway 与 WebUI](10-gateway-webui) | FastAPI 网关、SSE 桥、Liquid Glass 界面 |
| [11 记忆、自动化与生态](11-memory-automation-ecosystem) | 记忆、梦境、定时任务、MCP、子 Agent、@引用 |
| [12 测试、审计与发布](12-testing-release) | 250 工具冒烟测试、pytest + CI、GPL-3.0、复盘 |

### 附录

| 章节 | 内容 |
| --- | --- |
| [99 附录](99-appendix) | 术语表、文件地图、报错对照表、延伸阅读 |

## 素材说明

教程中的代码与设计均来自真实开发产物：

- **v1 / v2 / v3 三份快照**：记录了项目从早期单包结构（约 25 个工具文件）→ 大开发日后的完整形态（gateway、office、memory、dream、cron）→ 最终开源版（TUI、测试套件、安全修复、Public Beta I）的演进；
- **开发对话记录（编号 #46–#69）**：包含每一个需求原话、每一次真实报错与修复过程。教程中以「📅 开发日志」形式引用，日期时间均为原始记录；
- 代码引用统一使用仓库相对路径（如 `one_cedric/core.py`），可直接对照 [One-Cedric 仓库](https://github.com/Aoan2011/One-Cedric) 阅读。

> 本项目部分由 AI 辅助生成，教程如实记录了这一协作过程与其中的教训。

## 项目主页

- 仓库：<https://github.com/Aoan2011/One-Cedric>
- 协议：GPL-3.0 ｜ 当前版本：**Public Beta I**
