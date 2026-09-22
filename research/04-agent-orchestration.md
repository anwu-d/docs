# 04 — Agent 工具与多 Agent 协作编排调研报告

> 面向：部署在 Mac mini 上的个人 A 股量化研究系统（论文研究 → 因子生成 → 回测 → 新闻分析 → 持仓分析 → 交易复盘 全流程）。
> 调研日期：2026-09-22。所有 GitHub stars / License / 维护状态均于当日通过 GitHub REST API（api.github.com）或 repos.ecosyste.ms 镜像核实；模型与价格信息来自 Hugging Face、官方定价页与公开报道（见文末「来源与核实记录」）。

---

## 目录

1. [结论速览（TL;DR）](#1-结论速览tldr)
2. [调研对象逐项分析](#2-调研对象逐项分析)
   - 2.1 用户点名工具：Hermes / DSH / Codex / MiMo Code
   - 2.2 通用 Agent 框架
   - 2.3 MCP 与金融 MCP server 生态
   - 2.4 量化栈衔接件（qlib / MLflow / pandas / RD-Agent / TradingAgents）
3. [多 Agent 协作与任务编排方案（重点）](#3-多-agent-协作与任务编排方案重点)
   - 3.1 九类 Agent 角色定义
   - 3.2 Orchestrator：日级 / 周级流水线（定时 + 事件触发）
   - 3.3 任务状态机设计
   - 3.4 中间结果保存：文件约定 / 数据库 / 消息队列
   - 3.5 实验追踪：MLflow 接入
   - 3.6 防重复劳动：幂等键 / 结果缓存 / 去重 / 工作目录隔离
   - 3.7 职责切分：coding agent 自动改代码 vs 纯推理模型
4. [推荐组合与月度成本估算框架（Mac mini + 云端 API 混合）](#4-推荐组合与月度成本估算框架mac-mini--云端-api-混合)
5. [风险与注意事项](#5-风险与注意事项)
6. [来源与核实记录](#6-来源与核实记录)

---

## 1. 结论速览（TL;DR）

| 决策点 | 推荐 | 一句话理由 |
| --- | --- | --- |
| Orchestrator | **DSH（DeepSeek Harness）goal + Agent Teams / workflow** | 运行时原生带 goal 长程循环、子 Agent、后台任务、MCP client、webhook、schedule，插件化（MIT） |
| Coding agent（改策略/因子代码 + 跑回测） | **DSH 主力 + Codex CLI（codex-rs）做第二实现者/互审** | DSH 免费接 DeepSeek API 且编排原生；Codex 的 `codex exec` 适合无人值守批量改代码，Apache-2.0 |
| 纯推理（论文阅读、新闻归因、复盘质询） | **Hermes-4-14B/70B 本地（llama.cpp / Ollama / vLLM-MLX）+ DeepSeek-V4 系列云端兜底** | Hermes-4 Apache-2.0、函数调用/结构化输出原生，14B 在 Mac mini 可跑，隐私零外泄 |
| 备选 coding agent | MiMo Code（MIT、13.3k★）、Claude Code（订阅制）、OpenHands | MiMo Code 便宜且开源；Claude Code 交互式重构最强但订阅+周限额 |
| 多 Agent 框架（若不用 DSH 内建） | **LangGraph**（状态机/断点续跑）；TradingAgents 参考角色分工 | 个人系统不需要 AutoGen/CrewAI/MetaGPT 那套重抽象 |
| 工具接入 | **MCP 为统一工具层**（自建 qlib/backtest/MLflow/news MCP server） | 官方 MCP Registry 已上线，DSH/Codex/Claude Code/OpenHands 全支持 |
| 实验追踪 | **MLflow 3.x（本地 SQLite/文件 backend）** | qlib/pandas 无缝；每次回测 = 一个 MLflow run |
| 成本 | 纯本地推理 + DeepSeek API 轻量编排：**约 ¥150–600/月**；重度云端 coding：**¥500–1,500/月** | 见 §4 估算框架 |

---

## 2. 调研对象逐项分析

### 2.1 用户点名工具

#### 2.1.1 Hermes（Nous Research）

| 项目 | 内容 |
| --- | --- |
| **定位** | 开源权重**模型系列**（+ 函数调用规范与示例代码），不是编排框架 |
| **仓库/主页** | [NousResearch/Hermes-Function-Calling](https://github.com/NousResearch/Hermes-Function-Calling)（1,473★，MIT，Jupyter Notebook，最后 push 2025-12-22——示例性质、半休眠）；模型在 [HF NousResearch](https://huggingface.co/NousResearch)；官方发布列表 [nousresearch.com/releases](https://nousresearch.com/releases) |
| **模型现状** | Hermes-4 家族 2025-08-26 发布（[Hermes-4-14B](https://huggingface.co/NousResearch/Hermes-4-14B) / 70B / Llama-3.1-405B，[技术报告 arXiv:2508.18255](https://arxiv.org/abs/2508.18255)）；后续 Hermes-4.3-Seed-36B（2025-12）、NousCoder-14B（2026-01，编程模型）、Hermes Agent（2026-02，服务器自治 Agent 产品） |
| **License** | Hermes-4-14B：**Apache-2.0**（HF model card `license: apache-2.0`，基于 Qwen/Qwen3-14B）；Hermes-Function-Calling 代码 MIT。⚠️ Hermes-4-70B/405B 基于 Llama-3.1 底座，发布集合里各模型 License 以各自 HF card 为准，商用前逐个核对 |
| **维护状态** | 模型线活跃（2025-12~2026-02 仍有新品）；`hermes-function-calling` 仓库基本定型，只作参考实现 |
| **Agent/函数调用能力** | ChatML 提示格式；`<tool_call>{...}</tool_call>` 专用 token（流式可解析）；`<scratch_pad>`（Hermes-3 GOAP 规划）、`<tools>` schema 注入、JSON mode / structured outputs（Pydantic schema）；Hermes-4 混合推理（`<think>`）且**推理后单轮内出 tool call**；vLLM `tool_parser=hermes`、SGLang `qwen25` 内置解析器 |
| **本地部署（Mac mini）** | 14B：BF16 约 28GB / FP8 约 14GB / GGUF-Q4 约 4–6GB（CPU 也能跑）；**推荐 Mac mini 32GB 跑 Hermes-4-14B GGUF-Q4/Q5（llama.cpp/Ollama/MLX），64GB 可跑 70B Q4（约 40GB+）**。llm.co 给出的显存参考与 GGUF 量化档位见文末「来源与核实记录」。采样建议 `temperature=0.6, top_p=0.95, top_k=20` |
| **API 成本** | 自托管边际成本≈电费；不想本地跑可走 Nous Portal / Chutes / Nebius / Featherless 等第三方托管（价格各异，多为订阅制/按 token） |
| **适合的 Agent 角色** | **Research Agent（论文精读/摘要）、News Agent（事件抽取/影响归因）、Review Agent（交易复盘质询）、Market Agent（盘面解读）**——即一切「纯推理 + 结构化输出」岗位；不建议让它裸写大量代码（见 §3.7） |
| **与 Python 量化栈衔接** | 以 OpenAI 兼容 API（vLLM/SGLang/LM Studio 均提供）暴露给 LangGraph/Agno/smolagents/DSH；函数工具直接定义为 `get_factor_ic`、`run_backtest`、`query_qlib` 等 Python 函数（`hermes-function-calling` 的 `functions.py` 即此模式，其示例工具本身就是 yfinance 股票基本面查询）；JSON mode 可稳定产出因子规格/信号 JSON |

#### 2.1.2 DSH（DeepSeek Harness）

| 项目 | 内容 |
| --- | --- |
| **定位** | **Coding agent 运行时 + 多 Agent 编排底座**（"Everything is a Plugin"）。DeepSeek 官方出品、对标 Claude Code 的开源 harness |
| **仓库/包** | [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)（**232,442★**，MIT，TypeScript，2026-08 创建后极速增长，2026-09 仍每日活跃）；npm：[`@deepseek-ai/dsh`](https://www.npmjs.com/package/@deepseek-ai/dsh)（最新 0.1.5-rc.2，MIT）；文档 [deepseek-harness.github.io](https://deepseek-harness.github.io/deepseek-harness/)（DeepSeek API 官网「Agent Integrations」直接链接） |
| **License** | MIT（仓库与 npm 包均标注） |
| **维护状态** | 极活跃；版本线 0.1.x（alpha/rc 交替），社区已有第三方插件（如 dsh-agent-conductor、dsh-multi-model-orchestrator） |
| **与本任务相关的运行时能力**（据 npm 包依赖清单逐项核实） | `dsh-goal` + `dsh-goal-round-driver`（**goal 长程目标/多轮自动续跑**）、`dsh-tool-subagent` / `dsh-tool-subagent-control` / `dsh-subagent-fork-in-process`（**子 Agent 派生/继承上下文 fork**）、`dsh-tool-jobs` / `dsh-jobs-local`（**后台任务**）、`dsh-tool-workflow` / `dsh-workflow-worker-thread`（**JS workflow 脚本批量编排子 Agent**）、`dsh-experimental-agent-team`（**多 Agent 团队：任务 DAG、成员分工、审查/修复质量门**）、`dsh-mcp-client`（**MCP 客户端**）、`dsh-webhook` + `dsh-webhook-github`（**事件触发**）、`dsh-schedule`（**定时任务**）、`dsh-token-meter`（token 计量）、`dsh-hooks-codex` / `dsh-hooks-claude-code`（**可挂接 Codex / Claude Code 作为外部执行器**）、`dsh-tool-ralph`（fresh-agent 迭代循环）、`dsh-pwsh-sandbox`/`dsh-fs-sandbox`（文件与命令沙箱）、`dsh-session-persistence-jsonl`（会话持久化） |
| **本地部署要求** | Node.js 24.x（npm 包 `_nodeVersion: 24.20.0`），`npm i -g @deepseek-ai/dsh`；Mac mini 上轻量，纯 CLI + 本地 Web UI |
| **API 成本** | harness 本身免费（MIT）；模型走 DeepSeek API（见 §4 价格表）或任意 OpenAI 兼容端点 |
| **适合的 Agent 角色** | **Orchestrator（总编排）**，同时胜任 **Factor/Backtest Agent 的 coding 执行体**（生成/修改因子代码、跑 pytest/回测命令、读结果、迭代）。其 Agent Teams 的 quality gate（requirements→implementation→verification→review→integration）正好对应回测验收流程 |
| **与 Python 量化栈衔接** | 通过 `dsh-tool-bash`/`dsh-tool-pwsh` 直接调用 `python -m qlib.run...`、`pytest`、`mlflow`；通过 `dsh-mcp-client` 挂 qlib-mcp / 自建 backtest MCP；`dsh-experimental-code-runtime-python` 可内嵌 Python 执行 |

#### 2.1.3 Codex（OpenAI Codex CLI / codex-rs）

| 项目 | 内容 |
| --- | --- |
| **定位** | **Coding agent**（终端 CLI + 云端后台任务），Rust 实现 |
| **仓库** | [openai/codex](https://github.com/openai/codex)（**125,873★**，Apache-2.0，Rust；`codex-rs/` 即 Rust 主体、约 60 个 crate；2026-09-22 仍在每日 push） |
| **License** | Apache-2.0（可商用、可二次开发） |
| **维护状态** | 极活跃；OpenAI 官方主线产品 |
| **Agent 能力** | `codex` 交互 TUI / `codex exec` 无人值守单发任务（天然适合流水线）；MCP 支持（config.toml `[mcp_servers]`）；沙箱执行与审批分级（`--sandbox`、`--full-auto`）；云端任务（Cloud Tasks）与 GitHub PR 集成；模型侧有 GPT-5.x-Codex 系列与 Luna/Terra/Sol/Astra 档位 |
| **本地部署要求** | macOS 原生二进制（`npm i -g @openai/codex` 或 brew），Apple Silicon 良好支持；本身不跑模型（本地推理需另接） |
| **API 成本** | 订阅：Codex 随 ChatGPT 计划捆绑——Free $0、Go $8（仅本地）、**Plus $20、Pro 5x $100、Pro 20x $200**、Business $20–25/人；或 API 按 token 计费（GPT-5.6 Luna $0.20/$1.20、Terra $2/$12、Sol $4/$20、GPT-6 Astra $10/$50 每百万输入/输出 token）。独立测评（Artificial Analysis Coding Agent Index v1.5，2026-09）：Codex+GPT-6 Astra 约 **$7.47/任务**、Codex+DeepSeek V4 Pro 约 $0.24/任务（第三方路由） |
| **适合的 Agent 角色** | **Factor Agent / Backtest Agent 的 coding 执行体**（生成因子库代码、改造回测引擎、修复失败实验）、**Review Agent 的代码互审**（审 DSH 产出的 diff）；`codex exec` 可被 DSH 的 `dsh-hooks-codex` 直接调用 |
| **与 Python 量化栈衔接** | 在 qlib/pandas 工程目录内直接工作，读写 Python 文件、跑命令；MCP 可挂行情/回测 server；配合 git worktree 做实验隔离（§3.6） |

#### 2.1.4 Mimo Code（小米 MiMo）

| 项目 | 内容 |
| --- | --- |
| **定位** | **Coding agent CLI**（对标 Claude Code）+ 小米 MiMo 模型/托管 API 生态 |
| **仓库** | [XiaomiMiMo/MiMo-Code](https://github.com/XiaomiMiMo/MiMo-Code)（**13,338★**，MIT，TypeScript，2026-06-10 创建，2026-09-22 仍每日活跃；官网 [mimo.xiaomi.com/mimocode](https://mimo.xiaomi.com/mimocode)）；模型仓库 [XiaomiMiMo/MiMo](https://github.com/XiaomiMiMo/MiMo)（2,339★，Apache-2.0，MiMo-7B 论文代码）、[MiMo-VL](https://github.com/XiaomiMiMo/MiMo-VL)（643★，Apache-2.0）；另有 mimoagent（mini-swe-agent fork，MIT）、uni-agent（长程 Agent 训练框架，Apache-2.0）等 |
| **License** | MiMo Code：MIT；MiMo 模型权重/代码：Apache-2.0 |
| **维护状态** | 活跃（V0.1.0 于 2026-06-11 开源发布，此后持续迭代） |
| **Agent 能力** | 终端 AI 编程助手；持久记忆、Compose 模式、Dream 自进化、语音输入（官网描述）；内置多模态模型 **MiMo-V2.5（现役 V2.6 系列）**，并可接入 DeepSeek / Kimi / GLM 等第三方模型与 Token Plan |
| **本地部署要求** | Node/终端应用，Mac 可用；模型侧若走小米托管 API 则无本地要求；本地开源权重可用 vLLM 等自服务 |
| **API 成本** | 2026-07-26 18:00 起结束免费期，需订阅 **Xiaomi MiMo API Token Plan**（首订 88 折；另有 Team Plan、Batch API 半价）。MiMo-V2.5 按量价曾「最高降 99%」（2026-05-27 生效），属于低价档；V2.5 系列将于 2026-10-21 下线，需迁移到 V2.6 |
| **适合的 Agent 角色** | **Factor/Backtest coding 的低成本执行体**（配合其 Token Plan 跑批量小改动）；MiMo Responses API 兼容 OpenAI Responses API，官方提供 Codex / Claude Code 接入指南——即 MiMo 模型可以反向给 Codex/Claude Code 当后端 |
| **与 Python 量化栈衔接** | 同任意 coding agent：直接改仓库、跑 qlib/pytest；其 API 的 Tool Calling / Structured Output / Batch API 也可被 LangGraph/Agno 直接消费 |

> 注：网络上另有第三方 `dsh-agent-conductor`（在 DSH 会话内调度 Codex、Claude Code、Gemini 等 11 种外部 CLI）等社区插件，说明「DSH 做编排 + 各家 coding CLI 做执行」已是有生态验证的组合模式。

### 2.2 通用 Agent 框架

| 框架 | 定位 | GitHub | Stars（2026-09-22） | License | 维护状态 | 适合角色 | 量化栈衔接 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **LangGraph** | 编排框架（状态图/持久 checkpoint/人在环） | [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) | 42,122 | MIT | 活跃（2026-09-21 push） | **Orchestrator 备选**：把 9 类 Agent 建模为 StateGraph 节点，checkpoint 断点续跑 | 纯 Python 库，pandas/qlib 直接 import；LangSmith/MLflow 双记录 |
| **AutoGen** | 编排框架（会话式多 Agent） | [microsoft/autogen](https://github.com/microsoft/autogen) | 61,105 | CC-BY-4.0（仓库级；PyPI 包历史标注 MIT，商用前核对 LICENSE） | **放缓**（最后 push 2026-04，约 5 个月前） | Research/Review 辩论式小组讨论 | Python；可调任意函数工具。注意 .NET 线与 magentic-one 子项目 |
| **CrewAI** | 编排框架（角色扮演 Crew/Flows） | [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) | 58,892 | MIT | 活跃（OSS 1.0 GA，2026-09 push） | 快速搭「研究员+分析师+评审」流水线原型 | Python；CrewAI Enterprise 平台另计 |
| **MetaGPT** | 多 Agent 框架（SOP 驱动、软件公司隐喻） | [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT) | 70,551 | MIT | 一般（最后 push 2026-01） | 论文→研报的 SOP 流水线参考 | Python；与 qlib 集成需自写工具 |
| **Agno（原 phidata）** | Agent 平台框架（Agent/Team/Workflow + UI） | [agno-agi/agno](https://github.com/agno-agi/agno) | 42,295 | Apache-2.0 | 活跃（2026-09-22 push） | 想要自带 Web 控制台的轻量 Orchestrator | Python；内置 20+ 模型接入与工具生态 |
| **smolagents** | 极简 Agent 库（CodeAgent，模型写 Python 执行） | [huggingface/smolagents](https://github.com/huggingface/smolagents) | 29,438 | Apache-2.0 | 活跃（2026-08 push） | **Factor Agent 原型**：CodeAgent 直接生成 pandas 因子计算代码并执行 | 与 HF 生态/本地模型无缝；适合跑在本地 Hermes 上 |
| **OpenHands**（原 OpenDevin） | Coding agent 平台（Web UI + 沙箱） | [OpenHands/OpenHands](https://github.com/OpenHands/OpenHands) | 88,790 | MIT | 极活跃（2026-09-22 push） | 无人值守改 bug/迁移代码的执行体（SDK/headless 可编排） | Docker 沙箱内跑 qlib 回测；REST API 触发 |
| **Aider** | Coding agent（终端结对编程） | [Aider-AI/aider](https://github.com/Aider-AI/aider) | 49,112 | Apache-2.0 | 放缓（最后 push 2026-05） | git 原生小步修改（自动 commit）、benchmark 常客 | 直接编辑 pandas/qlib 代码；`aider --message` 可脚本化 |
| **Claude Code** | Coding agent（终端/桌面/Web，闭源 CLI） | [anthropics/claude-code](https://github.com/anthropics/claude-code)（issue/文档仓库） | 147,557 | **无开源 License**（专有，npm 分发） | 极活跃 | 难题交互式重构、疑难 bug 排查；可被 DSH 以 hook 调度 | 直接操作仓库；MCP 支持好 |

补充观察：

- **Coding agent 阵营**（Codex / Claude Code / OpenHands / MiMo Code / Aider / DSH）与**编排框架阵营**（LangGraph / AutoGen / CrewAI / MetaGPT / Agno / smolagents）正在合流：DSH、Codex 内建子 Agent/任务编排，LangGraph 也提供 agent runtime。个人单机系统建议「一个运行时 + 多个执行体」，不要叠两层编排。
- **TradingAgents**（[TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents)，**106,834★**，Apache-2.0，活跃）是金融多 Agent 的事实参考实现（基本面/情绪/新闻/技术面多角色 + 交易员/风控辩论结构），其角色划分可直接借鉴到 §3.1，但其本身偏美股且以 LangGraph 为核心。
- **microsoft/RD-Agent**（[链接](https://github.com/microsoft/RD-Agent)，14,650★，MIT，活跃）是 qlib 官方 README 点名的自动化 R&D Agent（因子/模型研究闭环），可作为 Factor/Alpha Agent 的「研究自动化」参考甚至直接组件。
- 任务书提及的 **OpenClaw 等其他 coding CLI** 未逐一核实（同类能力已被上表 Codex/Claude Code/OpenHands/Aider/MiMo Code/DSH 覆盖）；如需引入，按 §3.7 判定标准与维护/License 四问评估，不新增第二层编排。

### 2.3 MCP（Model Context Protocol）与金融 MCP server 生态

**可行性结论：非常可行，且应当作为本系统的统一工具层。** 理由：

1. **宿主全覆盖**：DSH（`dsh-mcp-client`）、Codex（`[mcp_servers]`）、Claude Code、OpenHands、LangGraph、smolagents 等全部支持 MCP client；工具只写一遍（Python `mcp` SDK），所有 Agent 共享。
2. **官方生态已成型**：[MCP Registry](https://modelcontextprotocol.io/registry/about)（官方中心化元数据仓库，Anthropic/GitHub/Microsoft/PulseMCP 共建，含 DNS 命名空间验证与 REST API；目前 preview）；[modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)（90,363★，TypeScript，2026-01 仍有版本 tag，活跃）；[modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)（24,350★，MIT，v2.2.0，活跃）。
3. **金融垂直生态存在但不成熟**——见下表，多为个人项目，需审查后使用（或自建）：

| MCP server | 覆盖 | 状态（2026-09 核实） | 建议 |
| --- | --- | --- | --- |
| [lsj210001/qlib-mcp](https://github.com/lsj210001/qlib-mcp) | Qlib 数据查询/因子分析/策略回测，A 股 | 2★、无 License、7 个月未更新 | 思路可抄（工具面设计），代码不宜直接依赖 |
| [Eternity714/finance-mcp](https://github.com/Eternity714/finance-mcp)（fork 自 huweihua123/stock-mcp） | A/港/美股数据 API、原生 MCP、对接 trading-agent | 0★、无 License、11 个月未更新 | 同上 |
| PyPI `finance-mcp-server` / `mcp-markets` / `infoway-mcp-server` | 美股行情/财报/新闻 | 小型个人包 | 仅美股参考 |
| [lijinly/akshare_mcp_server](https://github.com/lijinly/akshare_mcp_server)、qiupo/marketMcp 等 | A 股（akshare/行情） | 个人项目 | 可做 akshare 包装参考 |

**推荐做法**：用 `modelcontextprotocol/python-sdk` 自建 3 个私有 MCP server（不公开、不经 Registry）：

- `ashare-data`：封装 akshare/tushare/qlib 数据层（行情、财务、行业、停牌、涨跌停）；
- `backtest`：`run_backtest(config_hash) -> metrics`、`get_run_status`、`factor_ic(experiment_id)` 等幂等工具（§3.6 的执行入口）；
- `mlflow-kb`：实验查询 + 论文/研报知识库检索（配合本地向量库）。

### 2.4 量化栈衔接件

| 组件 | GitHub | Stars / License / 状态 | 在系统中的位置 |
| --- | --- | --- | --- |
| **qlib** | [microsoft/qlib](https://github.com/microsoft/qlib) | 48,698★ / MIT / 活跃（2026-09 push；v0.9.7） | 数据层 + 因子/模型/回测执行核心（Data/Backtest/Factor Agent 的被操作对象） |
| **MLflow** | [mlflow/mlflow](https://github.com/mlflow/mlflow) | 28,036★ / Apache-2.0 / 活跃（v3.16.x，3 天前 push；已含 GenAI/agentops 能力） | 实验追踪 + 模型/产物登记（§3.5） |
| **RD-Agent** | [microsoft/RD-Agent](https://github.com/microsoft/RD-Agent) | 14,650★ / MIT / 活跃 | 因子-假设-实验自动闭环的参考/组件（Alpha Agent） |
| **TradingAgents** | [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents) | 106,834★ / Apache-2.0 / 活跃 | 多 Agent 角色设计与辩论结构参考 |
| **pandas** | — | 事实标准 | 所有 Agent 输出最终落 pandas/DataFrame/Parquet |

衔接原则：**Agent 永远不直接改 qlib 内部**，只通过 (a) 项目仓库内的 Python 代码（因子定义、策略类、配置 YAML），(b) MCP 工具，(c) CLI 命令 三个边界与量化栈交互——边界清晰才能做幂等与审计。

---

## 3. 多 Agent 协作与任务编排方案（重点）

### 3.1 九类 Agent 角色定义

| 角色 | 职责 | 首选执行体 | 模型档位 | 输入 → 输出 |
| --- | --- | --- | --- | --- |
| **Research** | arXiv/研报精读、方法提炼、可实现性判断 | 纯推理（Hermes-4 本地） | 14B/70B 本地 | PDF/URL → `papers/<id>/summary.md` + 结构化因子假设 JSON |
| **Data** | 行情/财务/新闻数据获取、清洗、质量校验 | 确定性 Python（无 LLM）+ coding agent 维护 | — | 数据源 → `data/` Parquet + 校验报告 |
| **Factor** | 因子代码生成/修改、单因子测试（IC/IR/换手） | **DSH / Codex**（coding agent） | 云端大模型档 | 因子假设 JSON → `factors/<name>.py` + 单测 + IC 报告 |
| **Backtest** | 组合回测执行、参数扫描、结果解读 | 确定性回测引擎 + coding agent 修配置 | — | 策略配置 → MLflow run + `reports/<run_id>/metrics.json` |
| **Alpha** | 假设提出、因子挖掘规划、过拟合质控（对照 RD-Agent 思路） | 纯推理（Hermes / DeepSeek-V4-Pro） | 中高档 | 因子库状态 + 研究笔记 → 下一批实验计划（带优先级） |
| **News** | 盘前新闻/公告/政策抓取、事件抽取、相关性打分 | 纯推理（Hermes-4 本地，隐私/成本友好） | 14B 本地 | RSS/爬虫原文 → `news/<date>/events.jsonl` + 摘要 |
| **Market** | 盘面快照、风格/行业轮动、情绪面 | 纯推理 + 确定性指标 | 14B 本地 | 行情数据 + events → `market/<date>/snapshot.md` |
| **Portfolio** | 持仓分析、风险暴露、调仓建议（不自动下单） | 确定性计算 + 纯推理解读 | 中档 | 持仓 + 信号 → `portfolio/<date>/review.md` |
| **Review** | 交易/实验复盘、归因、失败质询、改进清单 | 纯推理（Hermes-70B 或 DeepSeek-V4-Pro） | 高档 | 交易日志 + 回测记录 → `reviews/<week>/postmortem.md` |
| **Orchestrator** | 调度全部、状态机推进、去重、告警 | **DSH goal/Agent Teams/workflow** | 编排用小档 | 触发器 → 任务 DAG → 各 Agent 产物 |

### 3.2 Orchestrator：日级 / 周级流水线（定时 + 事件触发）

采用 **「时间驱动为主、事件驱动为辅」** 的双触发模型。A 股的节奏天然按交易日历切分：

```
触发层（DSH: dsh-schedule 定时 + dsh-webhook 事件）
│
├─ 每日 07:30（cron） pre-market 日级流水线
│   ├─ Data      拉取隔夜数据/公告（幂等：data_date=当日）
│   ├─ News      事件抽取 → events.jsonl（事件触发入口的数据源）
│   ├─ Market    生成盘前快照
│   └─ Portfolio 基于昨日持仓 + 今日信号生成盘前提示（人工确认，不自动交易）
│
├─ 事件触发（webhook / 文件 watcher / 消息队列）
│   ├─ 新论文入池（RSS/arXiv 收藏）→ Research Agent
│   ├─ 回测任务完成（Backtest 写 result.json 触发）→ Alpha Agent 复核 + Review 抽查
│   ├─ 回测失败/异常指标（回撤>阈值）→ Review Agent 紧急质询 + 告警
│   └─ 盘中重大新闻（可选）→ News Agent 增量 + Portfolio 提示
│
├─ 每日 16:15（cron） post-market 日级流水线
│   ├─ Data      当日收盘数据落库（增量、幂等）
│   ├─ Market    盘后快照
│   ├─ Portfolio 当日持仓表现归因（确定性计算为主）
│   └─ Review-lite 当日交易/信号执行偏差检查（5 分钟级别小结）
│
└─ 每周五 20:00（cron） 周级流水线
    ├─ Review    全周复盘（交易 + 实验），产出改进清单
    ├─ Alpha     基于本周实验结果提出下周研究计划（回测队列）
    ├─ Factor+Backtest（coding agent 循环）夜间批量执行周计划实验
    └─ Orchestrator 汇总周报（含成本/token 计量，来自 dsh-token-meter）
```

编排实现建议（以 DSH 为宿主）：

1. **日/周固定流水线** → `dsh-schedule` 触发 `goal`（每条流水线一个长期 goal，round 间自动续跑）或 `workflow` 脚本（顺序 DAG，纯调度不开模型）。
2. **实验批处理**（Factor→Backtest→评审）→ DSH **Agent Teams**：Orchestrator 作 captain 建任务 DAG（依赖图天然表达「因子代码→单测→回测→评审」），quality gate（implementation→verification→review）对应「改代码→跑回测→验收评审」，review 失败自动进入 repair 轮次——这正是实验迭代需要的闭环。
3. **事件** → `dsh-webhook-github`/通用 `dsh-webhook` 接收外部系统回调；文件事件用 watcher 脚本转 webhook。
4. **人工闸门**：调仓建议、真实下单、以及「删除已有实验」类破坏性操作必须人工确认（Orchestrator 只输出建议）。

### 3.3 任务状态机设计

任务（Task）是最小调度单元，实验（Experiment）是任务的业务包装（1 实验 = 因子/策略改动 + 回测 + 评审的任务组）。统一状态机：

```
                 ┌────────────────────────────────────────────┐
                 │                  (事件/定时触发)              │
                 ▼                                           │
 [PENDING] ──依赖满足/调度──▶ [RUNNING] ──正常完成──▶ [SUCCEEDED_EVAL] ──验收通过──▶ [DONE]
     │                        │   │                        │  │
     │                        │   └─失败/超时─▶ [FAILED_RETRY]─┤（重试≤N，指数退避）
     │                        │                              └─验收不过─▶ [NEEDS_REVISION]
     │                        │                                        │（回到 RUNNING，round+1）
     ├─手动/策略取消─▶ [CANCELLED]                                    └─round>上限─▶ [ESCALATED]（转人工）
     └─同幂等键已有 DONE─▶ [SKIPPED_DUPLICATE]

 [DONE]/[CANCELLED]/[ESCALATED]/[SKIPPED_DUPLICATE] 为终态，结果不可变（immutable terminal result）。
 [HALTED]（人工暂停）可从 RUNNING/PENDING 进入，仅允许显式 resume 退出。
```

关键规则：

- **幂等键 = 状态机的准入条件**（见 §3.6）：进入 PENDING 前先查同键任务，`DONE` 直接 `SKIPPED_DUPLICATE`，`RUNNING` 则合并（不重复派发）。
- **验收（acceptance）挂在 SUCCEEDED_EVAL 之前**：回测任务的验收不是「命令退出码 0」，而是「metrics 产出 + 阈值检查」（如 Sharpe/回撤/换手在合理区间、无 look-ahead 报警）。可直接借用 DSH Agent Teams 的 acceptance criteria / verify commands 契约。
- **NEEDS_REVISION 必须带结构化 findings**（severity/problem/requiredFix），否则不允许流转——防止评审 Agent 含糊放行。
- **超时即失败**：每类任务带 timeout（如 Data 10min、单因子测试 30min、全量回测 4h），防止僵尸任务占位。
- 状态持久化：单机系统用 **SQLite 一张 `tasks` 表** 即可（字段：`task_id, idem_key, type, state, attempt, round, assignee, inputs_hash, started_at, finished_at, result_uri`）；若用 DSH Agent Teams，其任务图/attempt 机制就是此状态机的现成实现。

### 3.4 中间结果保存：文件约定 / 数据库 / 消息队列

**分层原则：文件为事实来源（git 可审计），数据库为索引与状态，消息为触发信号。**

#### 文件约定（工作区布局）

```
quant-lab/
├── code/                     # 策略/因子/工具代码（git 主仓库）
├── experiments/
│   └── <experiment_id>/      # 每个实验独立目录（或 git worktree，见 §3.6）
│       ├── spec.yaml         # 实验规格：假设、因子参数、回测配置、inputs_hash
│       ├── factors/          # 本实验新增/修改的因子代码（diff 到主干）
│       ├── logs/             # 任务日志（stdout/stderr/agent transcript）
│       ├── results/
│       │   ├── metrics.json  # 标准化指标（sharpe, ann_ret, max_dd, turnover, ic_mean...）
│       │   ├── equity.parquet / positions.parquet
│       │   └── report.md     # 人读报告
│       └── review.md         # Review Agent 结论（pass/needs_revision + findings）
├── runs/
│   ├── data/<data_date>/manifest.json     # 数据落位清单 + 校验和
│   ├── news/<date>/events.jsonl           # 事件流（append-only）
│   ├── market/<date>/snapshot.md
│   ├── portfolio/<date>/review.md
│   └── reviews/<ISO-week>/postmortem.md
├── papers/<paper_id>/{paper.pdf, summary.md, hypotheses.json}
└── state/                    # 不入 git（.gitignore）
    ├── tasks.sqlite          # 任务状态机（§3.3）
    ├── cache/                # 结果缓存（§3.6）
    └── mlruns/               # MLflow backend（或用独立 SQLite+对象目录）
```

命名规范：`<type>_<yyyymmdd>_<short_hash>`；所有产物 JSON 内嵌 `schema_version`、`created_by`（agent 名+模型）、`idem_key`、`inputs_hash`，保证可追溯与向后兼容。

#### 数据库

| 存什么 | 选型 | 理由 |
| --- | --- | --- |
| 任务状态/幂等键/审计 | **SQLite**（WAL 模式） | 单机、事务够用、零运维；并发要求上来后平移到 Postgres |
| 行情/因子值/回测明细 | **Parquet 文件集**（分区按日期）+ qlib 自身数据层 | 列存高效，pandas 直读；大结果不必进 DB |
| 实验元数据/指标 | **MLflow**（自带 SQLite/文件 backend） | 见 §3.5 |
| 论文/研报/复盘长文本 | Markdown 文件 + 本地向量库（如 LanceDB/Chroma） | Agent 可 grep/read，也可 RAG |

#### 消息队列

- **最小方案（推荐起步）**：SQLite `tasks` 表即队列（状态机字段就是队列语义）+ 文件 watcher/webhook 把事件写成 `runs/**/events.jsonl`。单机场景足够，天然幂等。
- **升级方案**：Redis Streams 或 NATS（Mac mini 上 Docker 单容器）承载事件流（`event.type` + `event_id` 兑费去重）；消费侧仍是「写 tasks 表」。**不建议**上 Kafka/RabbitMQ——运维成本大于收益。
- DSH 侧对应机制：`dsh-webhook`（外部事件入口）、`dsh-tool-jobs`（后台长任务）、Agent Teams 任务图（内部派发）。

### 3.5 实验追踪：MLflow 接入

- **部署**：Mac mini 上 `mlflow server --backend-store-uri sqlite:///state/mlflow.db --artifacts-uri ./state/mlartifacts`，纯本地零成本。
- **映射规则**：**一个回测实验 = 一个 MLflow run**：
  - `mlflow.set_experiment("factor/<factor_family>")`；
  - `run` 命名 = `experiment_id`（与文件目录一致）；
  - params = `spec.yaml` 全量展开（因子参数、回测区间、费率、基准）+ `code_git_sha` + `inputs_hash`；
  - metrics = `metrics.json` 全量（Sharpe、年化、最大回撤、换手、IC/IR、命中率……）；
  - artifacts = `equity.parquet`、`positions.parquet`、`report.md`、`review.md`；
  - tags = `agent`（谁改的代码）、`model`、`idem_key`、`round`、`verdict`（pass/needs_revision）。
- **LLM 调用本身也记录**：用 `mlflow.tracing`（MLflow 3.x 自带 GenAI tracing/eval 能力）或自建 `llm_calls` 表记录 prompt/completion/token 数——复盘「哪个 Agent 烧钱」以及对齐 §4 成本。
- **贯通**：Alpha Agent 选题前先 `mlflow.search_runs()` 过滤「同 inputs_hash 已跑过」的实验（与 §3.6 联动）；Review Agent 复盘直接读 run 列表做周对比。
- 可选：`dsh-mcp-client` 挂 `mlflow-kb` MCP server，让纯推理 Agent 不写 Python 也能查询实验。

### 3.6 防重复劳动：幂等键 / 结果缓存 / 去重 / 工作目录隔离

这是多 Agent 系统最容易翻车的地方，四层防线：

**① 任务幂等键（idem_key）**

```
idem_key = sha256( task_type ‖ normalized(inputs_hash) ‖ code_version_constraint )
```

- `task_type`：如 `factor_unittest` / `backtest` / `news_extract`；
- `normalized(inputs_hash)`：对输入（因子 spec、回测配置、数据日期）做**键排序、浮点归一化**后哈希——语义相同的任务必得相同键；
- `code_version_constraint`：是否允许复用旧代码版本的结果（数据类任务可以，代码类任务必须绑 `git_sha`）。
- 调度器准入逻辑：`INSERT INTO tasks(idem_key,...) ON CONFLICT DO NOTHING`——冲突即不派发；配合 §3.3 的 `SKIPPED_DUPLICATE` 状态。

**② 结果缓存**

- `state/cache/<idem_key>/` 存上一次成功结果；命中即直接返回（读 `result.json`）并标注 `cache_hit=true`；
- **缓存失效条件**（写进 spec）：数据日期推进、代码 sha 变化、qlib 数据校验和变化、缓存 TTL（如数据类 24h、代码类 7d）；
- LLM 调用层做**语义缓存**（embedding 相似度 > 阈值复用）只用于 News/Research 摘要类，禁止用于任何数字计算。

**③ 去重（内容级）**

- News：`event_key = hash(canonical(title) + source + date)`，抽取前先查当日 events.jsonl；
- Research：`paper_id = arXiv id / DOI`，已入 `papers/` 即跳过；
- Alpha 实验计划：新假设与既有实验做 embedding 查重（同一因子换个名字反复试是过拟合温床，Review Agent 有责任打回）；
- 跨 Agent 广播「谁正在做什么」：Orchestrator 维护 `claim(task_id, agent)` 租约（带 TTL），避免两个 Agent 同时改同一文件。

**④ 工作目录隔离**

- **每个实验独立 git worktree**（`git worktree add experiments/<id> -b exp/<id>`）：coding agent（DSH/Codex）只允许在自己的 worktree 内写文件，主干永远干净；合并回主干必须经 Review Agent 通过 + 人工确认。
- 重活（回测）在独立进程/容器跑，工作目录 = 实验目录，环境用锁版本的 venv/uv（`uv.lock` 提交进 spec），保证可复现。
- 并发上限：Mac mini 资源有限（回测吃 CPU），Orchestrator 用信号量限制「同时最多 K 个 backtest 进程」（建议 K=2~3），coding agent 会话不受限（不占 CPU）。

### 3.7 职责切分：coding agent 自动改代码+跑回测 vs 纯推理模型

| 环节 | 用 coding agent（DSH / Codex / MiMo Code） | 用纯推理模型（Hermes / DeepSeek-V4） | 判定标准 |
| --- | --- | --- | --- |
| 论文精读、假设提炼 | | ✅ | 无副作用的阅读/写作任务 |
| 因子规格设计（数学定义、伪代码） | | ✅（产出结构化 JSON） | 创造性+判断力优先 |
| **因子/策略代码生成与修改** | ✅ | | 必须写文件、跑单测、看报错迭代 |
| **回测执行与失败修复** | ✅（写配置、跑命令、读 traceback、修 bug） | | 需要工具循环（observe→act） |
| 回测结果解读、过拟合质控 | | ✅（也可让 DSH 做，但纯推理更便宜） | 数字输入→判断输出 |
| 新闻事件抽取/打分 | | ✅（本地 Hermes，量大、隐私敏感） | 高频、可结构化输出 |
| 盘面/持仓解读 | | ✅ | 解释性任务 |
| 交易复盘质询 | | ✅（70B/云端大模型，要求批判性） | 需要长上下文综合 |
| Orchestrator 调度本身 | ✅（DSH goal/Agent Teams，**尽量用代码逻辑而非 LLM**） | 例外时用小模型 | 调度是确定性逻辑，LLM 只做异常处置 |

**混合模式（推荐）**：`纯推理模型产出 spec.json → coding agent 按 spec 写代码 → 确定性引擎跑回测 → 纯推理模型评审结果`。这样 coding agent 的每次调用都有明确验收标准（spec），纯推理模型不接触文件系统（减小风险面），两边成本都可控。

**自动改代码 + 跑回测的循环**（夜间实验批处理，推荐 DSH Agent Teams 的 quality gate 或 `codex exec` 脚本化循环）：

```
loop (round = 1..MAX=3):
  1. coding agent 在 exp/<id> worktree 内实现 spec（含单测）
  2. verify: pytest 单测 + 单因子 IC 测试（快速回测）必须过
  3. 全量回测 → metrics.json
  4. Review Agent 按验收标准评审（过拟合检查、换手/费率现实性、与 spec 一致性）
     ├─ verdict=pass        → 合并候选，登记 MLflow，通知人工
     ├─ verdict=needs_revision → 带 findings 回到 1（round+1）
     └─ round > MAX          → ESCALATED，转人工
```

---

## 4. 推荐组合与月度成本估算框架（Mac mini + 云端 API 混合）

### 4.1 硬件与本地推理层

| Mac mini 配置 | 可跑本地模型 | 用途 |
| --- | --- | --- |
| 16GB | Hermes-4-14B GGUF-Q4（约 4–6GB）勉强 | News/Research 轻量任务 |
| **32GB（推荐起步）** | Hermes-4-14B Q5/Q6（流畅）；MiMo-7B 级模型 | News + Research + Market + Review-lite 全部本地化 |
| 64–128GB（推荐进阶） | Hermes-4-70B Q4（约 40GB+）或 14B 高精度 + 并行多实例 | Review/Alpha 高质量本地推理，云端只做 coding |

本地推理栈建议：**Ollama 或 llama.cpp（GGUF）起步**，追求吞吐换 MLX / vLLM（若转 Linux GPU 机器）。模型价（按 Nous 官方尺寸）：Hermes-4-14B 约 28GB BF16 / 14B FP8 约 14GB / GGUF-Q4 4–6GB。

### 4.2 云端 API 价格底账（2026-09 核实，用于代入估算）

| 服务 | 价格要点 | 来源 |
| --- | --- | --- |
| **DeepSeek API** | deepseek-flash：输入 $0.15/M（缓存未命中，off-peak）、$0.003（命中），输出 $0.6/M；deepseek-v4-pro：$0.66 / $0.022 / $1.98；高峰时段翻倍；off-peak 为峰时 5 折；1M 上下文 | [api-docs.deepseek.com](https://api-docs.deepseek.com/quick_start/pricing/) |
| **OpenAI Codex** | 订阅：Plus $20 / Pro 5x $100 / Pro 20x $200（5 小时滚动窗口限额，超出买 credit）；API：Luna $0.20/$1.20、Terra $2/$12、Sol $4/$20、Astra $10/$50（每 M 输入/输出） | [nops 成本对比（2026-09）](https://www.nops.io/blog/codex-vs-claude-code/)、[Codex rate card](https://help.openai.com/en/articles/20001106-codex-rate-card) |
| **Anthropic Claude Code** | 订阅：Pro $20 / Max 5x $100 / Max 20x $200（与 Claude 聊天共享 5 小时窗口 + 周上限）；API：Sonnet 5 $2/$10、Opus 5 $5/$25 | 同上 nops 对比 |
| **Xiaomi MiMo** | MiMo Code 需 Token Plan（首订 88 折）；MiMo-V2.6 系列按量/Token Plan；Batch API 半价；V2.5 曾大降价、2026-10-21 下线 | [mimo.mi.com](https://mimo.mi.com/docs/en-US/quick-start/summary/welcome)、[ithome](https://www.ithome.com/0/980/799.htm) |
| **Hermes 本地** | 一次性硬件 + 电费（≈0） | — |
| **独立成本/任务参考**（Artificial Analysis v1.5） | Codex+GPT-6 Astra ≈ $7.47/任务；Codex+DeepSeek V4 Pro ≈ $0.24/任务；Claude Code+Fable 5.1 ≈ $12.39/任务（API 计价、基准任务集） | [nops 引述](https://www.nops.io/blog/codex-vs-claude-code/) |

### 4.3 月度成本估算框架

按「任务量 × 单价」建一张自己的表，月成本：

```
月成本 = Σ_t (月任务量_t × 单次 token_t × 单价档_t)     ← API 路线
       或 Σ_i 订阅费_i + 超额 credit                      ← 订阅路线
       + 数据源费用（tushare pro 等）
       + 电费（本地推理）
```

**用量基线假设**（个人系统，可按实际替换）：

| 用量项 | 估价 | 月成本（保守 → 重度） |
| --- | --- | --- |
| News（本地 Hermes-4-14B，每日 300–1000 条事件抽取） | 本地 ≈ ¥0 | ¥0 |
| Research（每周 3–5 篇论文，云端大模型精读可选） | DeepSeek-V4-Pro，约 2–5M token/月 | ¥10–40 |
| Factor/Backtest coding（每周 10–30 个实验循环，每循环 3–8 个 coding 会话） | 路线 A：DeepSeek API（flash/pro 混合）约 30–120M token/月；路线 B：Codex Plus 订阅 + credit | 路线 A：**¥80–350**；路线 B：**$20 + credit ≈ ¥200–500** |
| Review/Alpha（每周 5–10 次长上下文评审，本地 70B 或云端） | 本地 ¥0；云端约 5–15M token/月 | ¥0–80 |
| MiMo Code Token Plan（可选，替代/补充 coding 路线） | 按 Token Plan 档位 | ¥100–300 |
| 数据源（tushare pro 等） | 积分制 | ¥0–200 |
| **合计** | | **轻量纯本地方案：¥150–600/月；重度云端 coding（双订阅+API）：¥800–1,500/月** |

三条省钱纪律：

1. **订阅 vs API 的套利**：交互式、稳定高频用订阅（Max/Pro 档封顶可控）；无人值守夜间批处理用 API（可断点重试、可精确核算）。nops 引用的第三方测算称 Claude Code Max 20x 重度使用相当于 $600–1,500/月 API 量——反过来说夜间流水线别烧订阅额度，订阅额度留给白天交互。
2. **off-peak 红利**：DeepSeek off-peak 半价——把回测批处理和 coding 循环安排在 00:00–04:00 / 06:00–10:00 UTC 之外（即北京时间白天/晚间错峰，按官方峰谷表排程）。
3. **缓存纪律**：prompt 前缀稳定（系统提示+工具定义放前面）以命中 DeepSeek 上下文缓存（$0.003 vs $0.15，差 50 倍）；§3.6 的结果缓存避免重复实验是最贵浪费的解药。

### 4.4 最终推荐拓扑

```
                    ┌────────────────────────────────────────────┐
   cron/webhook ──▶ │  DSH（Orchestrator，Mac mini，MIT 免费）      │
                    │  goal / Agent Teams / workflow / MCP client │
                    └───────┬───────────────┬───────────────┬─────┘
                            │               │               │
             coding 执行（改代码+跑回测）    │        纯推理执行（读/写文档）
              ┌─────────────┴──────┐        │        ┌──────┴──────────────┐
              ▼                    ▼        │        ▼                     ▼
        DSH 自身 coding        Codex CLI     │   Hermes-4 本地          DeepSeek-V4-Pro
        （DeepSeek API）      （codex exec，  │  （News/Research/        （Review/Alpha 难题，
              │                可互审/备份）   │   Market/Review-lite）    云端兜底）
              │                    │        │        │                     │
              └─────────┬──────────┘        │        └──────────┬──────────┘
                        ▼                   ▼                   ▼
              git worktree 实验隔离      MCP 工具层        Markdown/JSONL 产物
                        │          (ashare-data/backtest/mlflow-kb)   │
                        └──────────────► qlib + pandas + MLflow ◄─────┘
```

---

## 5. 风险与注意事项

1. **过拟合是头号敌人**：多 Agent 一夜能跑几百个实验 = 一夜能挖出几百个假因子。必须由 Alpha/Review Agent 做实验去重（§3.6③）、样本外强制验证、并对「同一族因子反复微调参数」打回。RD-Agent 论文同款问题，可参考其评审设计。
2. **自动化边界**：Agent 只产出**调仓建议**，不接实盘下单（若未来接，必须独立风控进程 + 人工确认闸门 + 熔断）。
3. **License 尽调**：AutoGen 仓库为 CC-BY-4.0（商用需评估）；Claude Code 无开源 License；Hermes 各尺寸随底座（Llama 系列）License 不同，逐模型核对 HF card。
4. **版本快速演进**：DSH（0.1.x rc）、MiMo Code（V0.x）、DeepSeek/小米模型迭代极快（MiMo V2.5 即将于 2026-10-21 下线）——把模型名/版本写进配置与 MLflow tags，别硬编码。
5. **单机单点**：Mac mini 需要备份（Time Machine/异地同步 `experiments/` 与 `state/`）；SQLite 记得开 WAL 并定期 `VACUUM INTO` 备份。
6. **数据合规**：行情/财务数据遵守数据源协议；新闻抓取注意版权与频率限制。
7. **成本失控**（订阅制尤其隐蔽）：用 `dsh-token-meter` + MLflow LLM tracing 记账，设月预算告警；nops 文中 Microsoft 内部 Claude Code 成本失控案例是典型教训。

---

## 6. 来源与核实记录

**仓库元数据（stars / license / 维护状态，2026-09-22 通过 GitHub API 或 repos.ecosyste.ms 核实）**

- [openai/codex](https://github.com/openai/codex) — 125,873★，Apache-2.0，Rust，push 2026-09-22（GitHub API）
- [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) — 232,442★，MIT，TypeScript（repos.ecosyste.ms）；npm 包 [`@deepseek-ai/dsh`](https://www.npmjs.com/package/@deepseek-ai/dsh) 0.1.5-rc.2 元数据（npm registry，含全部插件依赖清单）
- [XiaomiMiMo/MiMo-Code](https://github.com/XiaomiMiMo/MiMo-Code) — 13,338★，MIT，TypeScript，push 2026-09-22；[XiaomiMiMo/MiMo](https://github.com/XiaomiMiMo/MiMo) — 2,339★，Apache-2.0；[MiMo-VL](https://github.com/XiaomiMiMo/MiMo-VL) — 643★（GitHub API，org repos 列表）
- [NousResearch/Hermes-Function-Calling](https://github.com/NousResearch/Hermes-Function-Calling) — 1,473★，MIT，最后 push 2025-12-22（GitHub API）；[README 原文](https://raw.githubusercontent.com/NousResearch/Hermes-Function-Calling/main/README.md)（ChatML/`<tool_call>`/GOAP 格式与 yfinance 工具示例）
- [langchain-ai/langgraph](https://github.com/langchain-ai/langgraph) — 42,122★，MIT（GitHub API）
- [microsoft/autogen](https://github.com/microsoft/autogen) — 61,105★，CC-BY-4.0，最后 push 2026-04（GitHub API + [ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/microsoft%2Fautogen)）
- [crewAIInc/crewAI](https://github.com/crewAIInc/crewAI) — 58,892★，MIT，push 2026-09-22（GitHub API）；[CrewAI OSS 1.0 GA 公告](https://crewai.com/blog/crewai-oss-1-0---we-are-going-ga)
- [FoundationAgents/MetaGPT](https://github.com/FoundationAgents/MetaGPT) — 70,551★，MIT，最后 push 2026-01（GitHub API）
- [agno-agi/agno](https://github.com/agno-agi/agno) — 42,295★，Apache-2.0（GitHub API）
- [huggingface/smolagents](https://github.com/huggingface/smolagents) — 29,438★，Apache-2.0（GitHub API）
- [OpenHands/OpenHands](https://github.com/OpenHands/OpenHands) — 88,790★，MIT，push 2026-09-22（GitHub API，All-Hands-AI 组织已迁移）
- [Aider-AI/aider](https://github.com/Aider-AI/aider) — 49,112★，Apache-2.0，最后 push 2026-05（GitHub API）
- [anthropics/claude-code](https://github.com/anthropics/claude-code) — 147,557★，无 License（专有），push 2026-09-21（GitHub API）
- [microsoft/qlib](https://github.com/microsoft/qlib) — 48,698★，MIT（[ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/microsoft%2Fqlib)）
- [mlflow/mlflow](https://github.com/mlflow/mlflow) — 28,036★，Apache-2.0，v3.16.x（[ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/mlflow%2Fmlflow)）
- [microsoft/RD-Agent](https://github.com/microsoft/RD-Agent) — 14,650★，MIT（[ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/microsoft%2FRD-Agent)）
- [TauricResearch/TradingAgents](https://github.com/TauricResearch/TradingAgents) — 106,834★，Apache-2.0（[ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/TauricResearch%2FTradingAgents)）

**MCP 生态**

- [The MCP Registry — 官方说明](https://modelcontextprotocol.io/registry/about)（preview；Anthropic/GitHub/Microsoft/PulseMCP 共建）
- [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) — 90,363★（[ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/modelcontextprotocol%2Fservers)）
- [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) — 24,350★，MIT，v2.2.0（[ecosyste.ms](https://repos.ecosyste.ms/hosts/GitHub/repositories/modelcontextprotocol%2Fpython-sdk)）
- 金融 MCP：[lsj210001/qlib-mcp](https://github.com/lsj210001/qlib-mcp)、[Eternity714/finance-mcp](https://github.com/Eternity714/finance-mcp)、[lijinly/akshare_mcp_server](https://github.com/lijinly/akshare_mcp_server)、PyPI `finance-mcp-server` / `mcp-markets` / `infoway-mcp-server`（均经 ecosyste.ms / PyPI 核实为小型个人项目）

**模型与产品信息**

- [NousResearch/Hermes-4-14B model card（经 hf-mirror 读取原始 README）](https://hf-mirror.com/NousResearch/Hermes-4-14B/raw/main/README.md) — `license: apache-2.0`、base_model Qwen3-14B、ChatML/`<tool_call>`/vLLM `tool_parser=hermes`、FP8/GGUF 变体
- [Hermes 4 Technical Report（arXiv:2508.18255）](https://arxiv.org/abs/2508.18255)
- [Nous Research Releases](https://nousresearch.com/releases) — Hermes-4 家族/4.3-36B/NousCoder-14B/Hermes Agent 发布时间线与模型尺寸
- [llm.co Hermes-4-14B](https://llm.co/llms/hermes-4-14b) — 部署显存档位（BF16 28GB / FP8 14GB / GGUF-Q4 4–6GB）
- [Xiaomi MiMo 开放平台文档](https://mimo.mi.com/docs/en-US/quick-start/summary/welcome) — Token Plan/Team Plan/Batch API/MiMo Code/Responses API/V2.5 下线公告
- [IT之家：MiMo Code 7 月 26 日结束免费](https://www.ithome.com/0/980/799.htm)、[MiMo Code V0.1.0 发布](https://www.ithome.com/0/962/693.htm)
- [DeepSeek API 定价（官方）](https://api-docs.deepseek.com/quick_start/pricing/) — deepseek-flash / deepseek-v4-pro 峰谷价
- [nops：Codex vs Claude Code 成本对比（2026-09）](https://www.nops.io/blog/codex-vs-claude-code/) — 订阅档位、API 单价、Artificial Analysis 每任务成本
- [OpenAI Codex rate card](https://help.openai.com/en/articles/20001106-codex-rate-card)（页面反爬，仅索引核实）
- [DeepSeek Harness 解读（orcarouter）](https://www.orcarouter.ai/blog/deepseek-harness-explained)、[搜狐：DeepSeek Harness 公测与 NPM 插件生态](https://m.sohu.com/a/1062499707_114760)（背景佐证）

> 核实方法说明：GitHub 数据优先取自 api.github.com REST（匿名限额 60 次/时，超限后改用 repos.ecosyste.ms 镜像，其字段与 GitHub 官方一致且每日同步）；模型卡经 hf-mirror.com 读取 Hugging Face 原始 README；价格均取官方定价页或注明第三方出处。所有数值为 2026-09-22 快照，重定价频繁发生，落地前请复核。
