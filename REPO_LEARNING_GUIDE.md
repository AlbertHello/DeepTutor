# DeepTutor 仓库学习指南

版本：v1.7.3（源码 checkout）
适用：第一次接手这个仓库，需要快速建立全局认识

---

## 1. 定位

DeepTutor 是一个 "agent-native" 的学习工作台：对话、出题、深度研究、可视化、解题、精学路径、沉浸阅读、视频课、语音概览都跑在同一套 turn 运行时上。

三个入口，一条主链路：

```
CLI (Typer)            WebSocket /ws              Python SDK (DeepTutorApp)
    \                       |                          /
     └────────► TurnApplicationService ◄─────────────┘
                  (持久化 + 重放 + 会话)
                          │
                    TurnEngine
                          │
                  ChatOrchestrator
                  (按 active_capability 路由)
                          │
        ┌─────────────────┼──────────────────┐
   CapabilityRegistry  ToolRegistry      StreamBus
        │                 │                  │
   11 个能力          100+ 个工具        事件流回放
```

核心设计：**入口不写业务，能力不写传输，工具不写编排**。想看一个功能怎么跑，从 `capability` 进，不要从 router 进。

---

## 2. 技术栈

### 后端

| 项 | 选型 | 位置 |
| --- | --- | --- |
| 语言 | Python 3.11–3.14（本机 3.14.8） | `pyproject.toml` |
| 包管理 | setuptools + 模块化 wheel | `pyproject.toml`、`scripts/build_modules.py` |
| CLI | Typer + Rich | `deeptutor_cli/main.py` |
| HTTP/WS | FastAPI + Uvicorn + sse-starlette | `deeptutor/api/main.py` |
| 数据模型 | Pydantic v2 + jsonschema | `deeptutor/runtime/request_contracts.py` |
| 存储 | SQLite（aiosqlite）+ JSON/YAML 文件 | `data/user/` |
| 检索 | LlamaIndex + FAISS + BM25 + PageIndex | `deeptutor/services/rag/` |
| 模型接入 | OpenAI SDK 兼容层 + 多家 provider 适配 | `deeptutor/services/llm/`、`provider_registry.py` |
| 工具协议 | 自研 `BaseTool` + MCP 客户端 | `deeptutor/core/tool_protocol.py` |
| 日志 | loguru，JSONL | `deeptutor/logging/` |

### 前端

| 项 | 选型 |
| --- | --- |
| 框架 | Next.js 16.4（App Router）+ React 19 + TypeScript 5 |
| 样式 | Tailwind CSS 3 |
| 国际化 | i18next / react-i18next |
| 渲染 | react-markdown、KaTeX、Mermaid、Chart.js、Cytoscape |
| 文档 | PDF.js、EPUB.js、docx-preview、ExcelJS、BlockNote |
| 测试 | Vitest（单测）+ Playwright（E2E/UI 审计） |

### 测试规模

- 后端：`tests/` 下约 985 个 `test_*.py`
- 前端：`web/tests/`、`web/vitest.config.mts`、`web/playwright.config.ts`

---

## 3. 顶层目录

| 目录 | 职责 |
| --- | --- |
| `deeptutor/` | 后端主包：运行时、能力、工具、服务、存储 |
| `deeptutor_cli/` | Typer CLI 实现，`deeptutor` 命令入口 |
| `deeptutor_web/` | 打包进 wheel 的前端静态产物（发布用） |
| `web/` | 前端源码（Next.js App Router） |
| `packages/chat-engine/` | 独立 TypeScript chat 引擎包 |
| `packaging/` | 兼容发行包 `deeptutor-cli` |
| `apps/desktop/` | Electron 桌面壳（macOS / Windows） |
| `scripts/` | 构建、模块拆分、依赖同步、架构检查 |
| `tests/` | 后端测试 |
| `docs-for-dev/` | 开发者手册（架构、存储、模块、排障） |
| `data/` | 运行时目录（已 gitignore） |
| `examples/` | SDK / API 示例 |

---

## 4. 一次对话的完整链路

以 Web 端发一句话为例：

1. 前端聊天页通过 WebSocket `/ws` 发送 turn 请求（`deeptutor/api/routers/unified_ws.py`）
2. `TurnApplicationService` 落库、分配 `turn_id`、启动执行（`deeptutor/app/service.py`）
3. `TurnEngine.execute()` 拿到 `UnifiedContext`（`deeptutor/runtime/turn_engine.py`）
4. `ChatOrchestrator.handle()` 按 `active_capability` 选能力，默认 `chat`（`deeptutor/runtime/orchestrator.py`）
5. 能力内部跑多阶段流水线；`chat` 是单 agentic loop（exploring → responding），
   每一步工具调用都经过 `ToolRegistry`
6. 事件写入 `StreamBus`，由 turn 运行时持久化；适配器按 `seq` 重放给客户端
7. 能力结束时统一走 `emit_capability_result()`，附带 `UsageTracker` 的 `cost_summary`（`deeptutor/agents/_shared/capability_result.py`）

断线重连不是重新生成，而是从存储按 `after_seq` 回放事件，这是整套设计里最值得先读的部分。

---

## 5. 模块体系

1.7.0 起，仓库拆成"一个源码树 + 多个同版本发行包"。每个模块由 `deeptutor/modules/manifests/<id>.json` 声明：

- `paths`：这个模块拥有哪些代码目录（打包边界）
- `capabilities` / `loops` / `tools`：往运行时注册什么
- `routes` / `commands` / `navigation`：往 API、CLI、前端挂什么
- `services`：随服务启动的后台任务
- `requires` / `optional`：模块依赖
- `requirements`：该模块的 Python 依赖

源码 checkout 里所有模块都在，`deeptutor module enable/disable` 只控制装配；`install/uninstall` 仅限 wheel 环境。

### 模块一览（25 个）

| 模块 | 发行包 | 关键代码 | 作用 |
| --- | --- | --- | --- |
| core | `deeptutor` | `deeptutor/runtime`、`deeptutor/core`、`deeptutor/agents/chat` | turn 运行时、chat / ask_questions、SDK、CLI 基座 |
| server | `deeptutor-server` | `deeptutor/api` | FastAPI 应用、50 个 router、WebSocket |
| web | `deeptutor-web` | `deeptutor_web` | 前端产物装配 |
| knowledge | `deeptutor-knowledge` | `deeptutor/services/rag`、`deeptutor/knowledge` | 知识库、向量/BM25/PageIndex 检索、Obsidian / MarginNote / IMA 接入 |
| parsing | `deeptutor-parsing` | `deeptutor/services/parsing` | 文档解析调度与缓存 |
| parser-docling / markitdown / pymupdf4llm / liteparse | 各自发行包 | 无自有源码，纯依赖 | 可选解析引擎 |
| questions | `deeptutor-questions` | `deeptutor/agents/question` | 出题能力 `deep_question`、题库、练习 |
| research | `deeptutor-research` | `deeptutor/agents/research` | 深度研究 `deep_research`、论文检索 |
| solve | `deeptutor-solve` | `deeptutor/capabilities/solve` | 解题 `deep_solve`、解题计划工具 |
| visualize | `deeptutor-visualize` | `deeptutor/agents/visualize`、`deeptutor/visualizers` | 可视化 `visualize`（SVG / Chart.js / Mermaid / HTML / Manim） |
| renderer-manim | `deeptutor-renderer-manim` | `deeptutor/agents/math_animator` | 数学动画 `math_animator` |
| video-lesson | `deeptutor-video-lesson` | `deeptutor/agents/video_lesson` | Remotion 视频课 `video_lesson` |
| audio-overview | `deeptutor-audio-overview` | `deeptutor/capabilities/audio_overview` | 音频概览 `audio_overview` |
| reading | `deeptutor-reading` | `deeptutor/capabilities/reading`、`deeptutor/reading`、`deeptutor/video_learning` | 沉浸阅读 `immersive_reading`、阅读器、视频学习 |
| mastery | `deeptutor-mastery` | `deeptutor/capabilities/mastery`、`deeptutor/learning` | 精学路径 `mastery_path`、评估、评分 |
| books | `deeptutor-books` | `deeptutor/book` | 生成式教材、章节编辑 |
| co-writer | `deeptutor-co-writer` | `deeptutor/co_writer`、`deeptutor/notes` | 笔记创作、块编辑、Anki 导出 |
| notebook | `deeptutor-notebook` | `deeptutor/services/notebook` | 收藏、笔记本 |
| memory | `deeptutor-memory` | `deeptutor/services/memory` | L1/L2/L3 记忆与记忆图谱 |
| calendar | `deeptutor-calendar` | `deeptutor/services/calendar`、`deeptutor/services/cron` | 日程与定时任务 |
| partners | `deeptutor-partners` | `deeptutor/partners`、`deeptutor/services/partners` | 持久化伙伴、IM 通道、子代理编排 |
| providers | `deeptutor-providers` | 无自有源码，纯依赖 | 模型/语音/搜索 provider 依赖集合 |

依赖关系（`requires`）值得记住的几条：`knowledge → parsing`、`reading → knowledge + parsing + mastery + notebook`、`books → knowledge + parsing + questions + visualize + mastery`、`partners → memory`、`web → server`。

---

## 6. 能力（Capability）

能力是"接管一轮对话"的流水线，统一由 `CapabilityRegistry` 注册，`ChatOrchestrator` 路由。

| 能力 | 代码 | 阶段 |
| --- | --- | --- |
| `chat` | `deeptutor/agents/chat` | exploring → responding |
| `ask_questions` | `deeptutor/capabilities/ask_questions` | responding（强制走 ask_user） |
| `deep_solve` | `deeptutor/capabilities/solve` | responding（chat loop + 解题规划工具） |
| `deep_question` | `deeptutor/agents/question` | ideation → generation |
| `deep_research` | `deeptutor/agents/research` | rephrasing → decomposing → researching → reporting |
| `visualize` | `deeptutor/agents/visualize`、`deeptutor/visualizers` | analyzing → generating → reviewing；Manim 走 concept_analysis → … → render_output |
| `math_animator` | `deeptutor/agents/math_animator` | concept_analysis → concept_design → code_generation → code_retry → summary → render_output |
| `video_lesson` | `deeptutor/agents/video_lesson` | code_generation → render_check → render_output |
| `mastery_path` | `deeptutor/capabilities/mastery`、`deeptutor/learning` | responding（chat loop + mastery 工具） |
| `immersive_reading` | `deeptutor/capabilities/reading` | responding（文档/媒体锚定） |
| `audio_overview` | `deeptutor/capabilities/audio_overview` | retrieving → script_writing → voice_generation → publishing |

另外有 `loops` 概念：不单独接管 turn，而是往普通 chat loop 里注入策略或工具的组合。当前包括 `subagent`、`explore_context`、`setup`、`creator`、`notes`、`obsidian`、`marginnote4`、`ima`、`partner_authoring`、`partner_group`、`solve`、`mastery`、`immersive_reading`、`visualization_generation`、`ask_questions`。

---

## 7. 工具（Tool）

工具是 LLM 在一次 loop 里按需调用的单函数，定义在 `deeptutor/core/tool_protocol.py`。

### 用户可开关（8 个）

`USER_TOGGLEABLE_TOOL_NAMES`（`deeptutor/tools/builtin/__init__.py`）：

| 工具 | 用途 |
| --- | --- |
| `brainstorm` | 广度发散，带理由 |
| `web_search` | 联网搜索 + 引用 |
| `paper_search` | arXiv 检索 |
| `zotero_search` | Zotero 文库检索 |
| `reason` | 独立深度推理调用 |
| `geogebra_analysis` | 数学题图 → GeoGebra 指令 |
| `imagegen` | 生成图片 |
| `videogen` | 生成视频 |

### 上下文自动挂载（不暴露开关）

`rag`、`kb_files`、`read_source`、`read_skill`、`load_tools`、`web_fetch`、`github`、`exec`、`ask_user`、`memory` 读写、notebook 收藏、workspace 读写/导出、learning journal、MCP / CLI 应用等。挂载规则集中在 `deeptutor/agents/_shared/tool_composition.py` 的 `ToolMountFlags`。

### 能力专属

`mastery_*`（20 个）、`solve_plan/solve_finish_step/solve_replan`、`reading_*`（8 个）、`obsidian_*`（9 个）、`marginnote_*`（7 个）、`ima_*`（5 个）、`research_*`、`partner_*`、`write_note` 等，由能力激活时带出，不受用户开关控制。

---

## 8. deeptutor/ 子包职责

| 子包 | 职责 |
| --- | --- |
| `runtime/` | turn 引擎、编排器、StreamBus、能力/工具注册表、launcher、worker 隔离 |
| `core/` | 协议层：`UnifiedContext`、`StreamEvent`、`BaseTool`、`TurnCapability` |
| `agents/` | 各能力的 agent 实现；`_shared/` 是公共编排与结果封装 |
| `capabilities/` | 非 agent 形态的能力与 loop 策略（ask_questions、mastery、reading、subagent、setup、creator、obsidian…） |
| `api/` | FastAPI 应用、50 个 router、鉴权、路由装配 |
| `app/` | SDK 门面 `DeepTutorApp` 与 `TurnApplicationService` |
| `services/` | 领域服务：llm、rag、embedding、search、memory、session、config、parsing、mcp、voice、sandbox 等 |
| `tools/` | 内置工具实现、工作区工具、exec、媒体生成 |
| `knowledge/` | 知识库类型、manifest 读取 |
| `reading/`、`video_learning/` | 阅读器与视频学习的数据层 |
| `learning/` | 精学路径的评估、评分、身份、证据 |
| `book/`、`co_writer/`、`notes/` | 教材生成、协作写作、笔记块 |
| `partners/` | 持久化伙伴与 IM 通道 |
| `multi_user/` | 本地多用户：账号、授权、审计、书籍权限 |
| `plugins/` | 插件目录与 entry point 发现 |
| `skills/` | 内置技能（SKILL.md） |
| `visualizers/` | 可视化渲染器注册与存储 |
| `events/` | 事件总线 |
| `i18n/` | 状态文案与元数据国际化 |
| `logging/` | 日志、用量统计 |
| `modules/` | 模块 manifest 与装配目录 |
| `config/`、`utils/` | 配置兼容层与杂项工具 |

---

## 9. 前端结构

`web/` 是标准 Next.js App Router 工程：

```
web/app/
  (workspace)/        主工作区：chat、learning、calendar、co-writer、kanban、partners、whisper
  (settings)/         设置
  (auth)/             登录（启用多用户时）
  (admin)/            管理
  (utility)/          命令面板、侧边聊天等
  api/                Next 侧接口（代理/上传）
web/features/         按领域切分：chat、knowledge、settings、tasks、calendar、capabilities…
web/components/       通用组件
web/context/          React Context
web/contracts/        后端契约生成的 TypeScript 类型
web/i18n/、locales/   国际化资源
web/shared/、lib/、hooks/、types/
```

两条约定：

- 跨端类型不手写，`npm run contracts:generate` 从后端 schema 生成
- 路由预算、依赖方向都有检查：`npm run architecture:check`、`perf:check`

---

## 10. 数据存储

默认不需要任何数据库服务，全部落在一个 runtime home（本仓库根目录，或 `DEEPTUTOR_HOME`）下：

| 路径 | 内容 |
| --- | --- |
| `data/user/settings/` | JSON/YAML 设置与 provider 凭据 |
| `data/user/chat_history.db` | 会话、消息、turn、题库、练习 |
| `data/user/.runtime/` | 私有运行时状态（workspaces.sqlite3 等） |
| `data/user/workspace/` | 内容工作区与生成产物 |
| `data/user/logs/` | 应用日志 |
| `data/knowledge_bases/` | 知识库索引与版本化源文件 |
| `data/parse_cache/` | 文档解析缓存 |
| `data/users/<id>/` | 多用户模式下的用户工作区 |
| `data/partners/<id>/` | 伙伴的虚拟工作区 |
| `data/system/` | 账号、授权、审计、模块启用状态 |

内容工作区可以放在 `data/user` 之外；此时学习状态在该工作区的 `.deeptutor/data/` 下，账号设置仍在 runtime home。

---

## 11. 推荐阅读路线

### 第一天：入口与主链路

1. `AGENTS.md` —— 先建立两层插件模型（Tool / Capability）的印象
2. `deeptutor/core/context.py`、`deeptutor/core/stream.py` —— 看懂 `UnifiedContext` 与 `StreamEvent`
3. `deeptutor/runtime/orchestrator.py` —— 路由与 StreamBus 生命周期
4. `deeptutor/agents/chat/capability.py` + `deeptutor/agents/loop/` —— 单 agentic loop 怎么跑
5. `deeptutor/api/routers/unified_ws.py` —— 前端到底发了什么

### 第二天：能力与工具

1. `deeptutor/runtime/registry/` —— 注册表如何装配
2. `deeptutor/modules/manifests/*.json` —— 模块边界与装配声明
3. 挑一个能力精读：`deep_research` 最适合看多阶段编排（`deeptutor/agents/research`）
4. `deeptutor/tools/builtin/__init__.py` —— 工具包装与开关
5. `deeptutor/agents/_shared/tool_composition.py` —— 上下文挂载规则

### 第三天：存储、前端与扩展

1. `deeptutor/app/service.py` —— 持久化 turn、重放、恢复
2. `docs-for-dev/DATA_STORAGE.md` —— 存储边界
3. `web/app/(workspace)/chat` + `web/features/chat` —— 前端如何消费事件
4. `docs-for-dev/PLUGINS.md`、`docs-for-dev/MODULES.md` —— 扩展方式

---

## 12. 常用命令

```bash
# 环境
source .venv/bin/activate
export DEEPTUTOR_HOME="$PWD"

# 运行
deeptutor start --detach --no-browser     # 生产构建启动
deeptutor start --dev                     # 前端 HMR
deeptutor stop                            # 停止
deeptutor serve --port 8001               # 只起 API

# 单轮能力
deeptutor run chat "解释傅里叶变换"
deeptutor run deep_solve "解 x^2=4" --kb my-kb
deeptutor run visualize "正弦波动画" --config render_mode=manim_video

# 诊断
deeptutor doctor --format json
deeptutor doctor --online --format json
deeptutor module list
deeptutor config show

# 测试
pytest tests/ -q
cd web && npm run test:unit
```

---

## 13. 二次开发注意点

1. **不要用 feature wheel 覆盖源码目录**。editable 安装后，`import deeptutor` 指向工作树；再装 `deeptutor-server` 之类的包会产生同名文件，导入顺序说不清。
2. **依赖改动走 manifest**。模块依赖写在 `deeptutor/modules/manifests/*.json`，再由 `scripts/sync_module_requirements.py` 生成 `requirements/*.txt`，不要手改生成文件。
3. **能力要发事件，不要直接写库**。持久化由 turn 运行时统一做；能力只负责产出 `StreamEvent` 和最终 result。
4. **文案与状态走 i18n**。能力提示词在 `deeptutor/capabilities/prompts/{en,zh}/`，状态文案在 `deeptutor/i18n/`。
5. **前端类型从契约生成**。改后端 schema 后跑 `npm run contracts:generate`，否则 `contracts:check` 会挂。
6. **注意架构约束**。`.importlinter` 和 `scripts/check_architecture.py` 会限制层间依赖方向，跨层 import 可能直接报错。
