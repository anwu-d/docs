# 个人 A 股量化研究 · 交易辅助 · 复盘系统 — 最终设计文档（实施蓝图）

> **文档性质**：可直接落地的实施蓝图（Final Design Document），面向部署在 **Mac mini（Apple Silicon）** 上的个人 A 股量化研究 / 交易辅助 / 复盘系统。
> **输入**：4 份调研报告（`docs/research/01-data-layer.md`、`02-backtest-factor.md`、`03-news-events.md`、`04-agent-orchestration.md`，下称"调研 01–04"）。
> **数据核实时间**：**2026-09-22**。文中所有 GitHub stars / License / 价格 / 版本 / 日期均为该日快照（经 GitHub API、PyPI、官方定价页核实，明细见调研报告与本文附录 A）；调研日期之后以原文链接为准。
> **写作约定**：专有名词保留英文；优先级标记 **P0**（MVP 必需）/ **P1**（第一阶段）/ **P2**（第二阶段及以后）；每章标注对应的调研依据，关键取舍给出理由与被放弃的备选。

> **修订记录 v2（2026-09-23，依据审计 A01–A12/§8）**
>
> 依据[《个人 A 股量化系统审计与公开数据优先整改方案》](personal-quant-audit-plan.md)（审计日期 2026-09-23，基线 `anwu-d/docs@afbebb3d`）分两批对主设计定点修订：
>
> - **第 1 批（本次）**：
>   - **§5**：新增可见性时间契约（"市场已公开时间"与"本系统实际获得时间"双口径、逐查询记录口径、无披露时刻取保守下一交易时段规则、`historical_pit_unverified` 阻断，A01）；信号股票池/下单候选/成交结果/持仓四分离，禁止事后删除未成交样本（A03）；数据策略改为"公开数据包初始化（investment_data 固定 release tag + 固定 commit `qlib/validate_archive.py` 校验 + 独立快照目录原子切换）+ 定期同步 + 质量验收 + 按需补缺"，BaoStock/AKShare 降为补充层与抽样对账（A05/审计 §11）；单写入者不可变 Parquet 快照 + manifest、读任务绑定 `snapshot_id`、SQLite 只存任务元数据、独立介质/异地加密备份（目标 RPO 24h/RTO 1d，实测确认，A09）。
>   - **§6**：内存更正——Hermes-4-14B GGUF **Q5_K_M 实际约 10.51GB**（审计 S2，bartowski 量化发布页），删除"Q4 4–6GB"旧写法；权重 ≠ 总内存（含 KV cache/上下文/运行时）；给出实测记录模板；取消首期购买 32/64GB 设备的前置条件，训练/回测/推理错峰；成本改为用量记录口径（输入/输出/缓存/重试/电费/已有订阅分开记录），删除笼统月费数字（A06）。
>   - **§7**：精简依赖；删除占位版本（如 `minio/minio:RELEASE.2026-xx`）；命令拆为"架构示例/待实现接口/已验证操作"三类标注（当前"已验证操作"为空）；同一服务禁止容器与裸机重复启动（A07）。
>   - **§8**：协作模式改为"GPT 负责计划/研究规范/审核，DeepSeek 或 MiMo 执行编码、数据接入、测试、报告"分工表（数值与回测由确定性程序计算、模型不得编造或心算替代；最终样本外由隔离验证任务执行、结果不回流自动修复循环）；任务契约八字段、每任务一个主执行者、模型切换携带上下文；执行边界（工具/目录/网络/并发/时长限制、原始数据与封存集只读、提示注入防护、修复与假设变更分开登记、额度预留-结算）（A04/A12）。
>   - **§11**：出报改为数据就绪驱动（每数据集登记截至时间/预期日期/就绪状态；初版标注缺项 → 补齐 → 次日晨间修订；旧数据标 `stale`；单源延迟/故障展示缺项及版本、恢复后补跑不重复统计，A10）。
> - **第 2 批（2026-09-23 已完成）**：覆盖 **§12–14、§15.3、§16.3、附录 B** ——统一阶段优先级（G0–G4 阶段门）、训练/开发验证/最终封存测试分离与最终测试访问规则、删除"入库因子数量"硬指标（A04）；部署命令速查按三类拆分（A07）；License 改为按许可条款与使用方式判断、允许适合的个人 GPL 工具、四类许可分开记录（A08）；附录 B 优先级索引同步。
> - **修订原则**：与审计冲突的旧结论按审计口径改写；2026-09-22 已核实事实与来源 URL 保留；审计未重核项标注"**待核实**"；运行类验收在实际执行前一律不预填通过。

---

## 0. 阅读指引：16 点覆盖索引

| # | 用户要求 | 所在章节 |
|---|---|---|
| 1 | 整体系统架构（ASCII 分层架构图、数据流、三大闭环 Pipeline） | §1 |
| 2 | 推荐开源项目清单（表格） | §2 |
| 3 | 每个项目负责什么及选型理由（整合 4 份调研） | §3 |
| 4 | 直接复用 vs 需要自研的能力边界 | §4 |
| 5 | 数据层/研究层/回测层/Agent 层/知识库/任务调度层/展示层逐层详细设计（schema、接口、目录约定） | §5 |
| 6 | Mac mini 本地部署方案（内存/磁盘、本地模型 vs 云端 API、月度成本） | §6 |
| 7 | Docker/Python/数据库/对象存储等基础设施（docker-compose 服务清单、arm64 注意事项） | §7 |
| 8 | Agent 协作与任务编排（Orchestrator、9 类 Agent、任务状态机、experiments/<id>/、MLflow、防重复四层防线、MCP） | §8 |
| 9 | 论文→因子→回测自动化 Pipeline（每环节输入输出） | §9 |
| 10 | 新闻→市场事件→持仓分析 Pipeline | §10 |
| 11 | 每日收盘后→自动复盘→次日预案 Pipeline（16:15 触发、持仓逐只输出、输出模板） | §11 |
| 12 | 实验追踪、因子版本管理、结果管理（因子库 schema、血缘、MLflow、入库门槛、衰减监控） | §12 |
| 13 | 量化研究风险控制（数据质量、幸存者偏差、前视偏差、多重检验校正、过拟合、样本外机制、数据可用性陷阱） | §13 |
| 14 | 实施路线图：阶段门 G0–G4（放行证据、工作要点、已删除旧指标） | §14 |
| 15 | 各阶段部署的项目、服务与代码仓库（monorepo 目录树） | §15 |
| 16 | 降低重复开发的总体原则（开源优先决策流程、自研准入标准、License 合规清单） | §16 |

**一句话架构**：`akshare + baostock + tushare` 三源采集 → Parquet/DuckDB/qlib_bin 落地（PIT 表防前视）→ qlib 研究基座（alphalens-reloaded 单因子体检、quantstats 报告、MLflow/Optuna 实验管理、RD-Agent/gplearn 因子挖掘）→ DSH（DeepSeek Harness）Orchestrator 编排 9 类 Agent（Hermes-4-14B 本地 + DeepSeek API 混合推理、MCP 统一工具层）→ 三大闭环 Pipeline（研究闭环 / 事件-持仓闭环 / 复盘-预案闭环）→ Markdown/JSONL 知识库 + Streamlit 展示层。**自研只做薄层**：调度/ETL/对账/校验/PIT 表、事件库、持仓与复盘业务逻辑、3 个 MCP server。

---

## 1. 整体系统架构

> 依据：调研 01 §7/§10（存储与 ETL 边界）、调研 02 §8.1（研究栈分层）、调研 03 §3/§4（事件流水线）、调研 04 §3.2/§4.4（编排拓扑与触发模型）。

### 1.1 分层架构图（ASCII）

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ ⑦ 展示层 (apps/)                                                                     │
│   Streamlit 控制台：研究看板 │ 复盘/预案阅读器 │ 因子库浏览器 │ 实验对比(MLflow UI)      │
│   输出物：晨报 08:30 │ 复盘报告 16:45 │ 次日预案 17:30 │ 周报(周五 20:00)              │
└───────────────▲──────────────────────────────────────────────▲──────────────────────┘
                │ HTTP / 静态文件                                │ 查询
┌───────────────┴──────────────────────────────────────────────┴──────────────────────┐
│ ⑥ 任务调度层 (orchestration/)  ——  DSH Orchestrator（dsh-schedule + dsh-webhook）      │
│   任务状态机 tasks.sqlite：PENDING→RUNNING→SUCCEEDED_EVAL→DONE │ 防重复四层防线         │
│   日级 07:30/16:15 触发 │ 事件触发（回测完成/重大新闻/失败告警） │ 周级周五 20:00        │
└──────┬──────────────────┬──────────────────┬──────────────────┬─────────────────────┘
       │ 任务派发/回收      │                  │                  │
┌──────▼─────────┐ ┌──────▼─────────┐ ┌──────▼─────────┐ ┌──────▼─────────────────────┐
│ ④ Agent 层      │ │ ⑤ 知识库        │ │ ③ 回测层        │ │ ② 研究层                    │
│ (agents/)       │ │ (knowledge/)   │ │ (backtest/)    │ │ (factors/ + 外部框架)       │
│ 9 类 Agent：     │ │                │ │                │ │                             │
│ Orchestrator    │ │ papers/        │ │ qlib Exchange  │ │ qlib Handler/Dataset        │
│ Research Data   │ │ reviews/       │ │  (涨跌停/T+1   │ │ alphalens-reloaded 体检     │
│ Factor Backtest │ │ postmortems/   │ │   费用/容量)   │ │ quantstats 绩效             │
│ Alpha News      │ │ playbook.md    │ │ rqalpha 复核   │ │ Optuna 调参                 │
│ Market Portfolio│ │ 向量库(LanceDB │ │ quantstats 报告│ │ RD-Agent/gplearn 挖掘        │
│ Review          │ │  /sqlite-vec)  │ │ MLflow run     │ │ MLflow(QlibRecorder)        │
│ 执行体：DSH /    │ │ RAG(LlamaIndex)│ │ experiments/   │ │ RollingGen 滚动样本外        │
│ Codex / Hermes-4│ │                │ │  <id>/         │ │                             │
│ 本地 / DeepSeek │ │                │ │                │ │                             │
└──────┬─────────┘ └──────▲─────────┘ └──────▲─────────┘ └──────▲─────────────────────┘
       │ MCP 工具层（统一边界：ashare-data / backtest / mlflow-kb 三个 MCP server）        │
┌──────▼──────────────────▼──────────────────▼──────────────────▼─────────────────────┐
│ ① 数据层 (data/ + etl/)                                                              │
│   采集：baostock(主) + akshare(广度/兜底) + tushare 2000 积分(财务/合规) + yfinance/fredapi│
│   存储：raw(响应包) → staging → clean(Parquet) │ DuckDB 查询 │ SQLite 元数据/水位        │
│         qlib_bin 物化视图(dump_bin) │ PIT 财务版本表 │ universe 快照 │ MinIO 冷备(P2)     │
│   质量：DQ 校验规则集 → dq_issue 表 │ 多源对账 │ 交易日历/停牌日历/复权因子               │
└─────────────────────────────────────────────────────────────────────────────────────┘

数据流向（实线为主流）：
  采集源 ──▶ raw ──▶ staging ──▶ clean Parquet ──▶ DuckDB 视图 ──▶ qlib_bin ──▶ 研究层
                                    │                              │
                                    ├──▶ PIT 表 / universe 快照 ─────┤（防前视输入）
                                    └──▶ 事件库(news/event) ──▶ Agent 层 ──▶ 知识库 ──▶ 展示层
  研究层 ──▶ 回测层 ──▶ MLflow run + experiments/<id>/ ──▶ 评审 ──▶ 因子库/知识库 ──▶ 展示层
```

### 1.2 三大闭环 Pipeline（贯穿各层）

| 闭环 | Pipeline 名 | 触发 | 路径（层间流向） | 主要产出 | 章节 |
|---|---|---|---|---|---|
| **闭环 A：研究-因子闭环** | `pipeline_paper2factor`（论文→因子→回测→入库） | 周级定时 + arXiv/RSS 事件 | 知识库(papers) → Agent 层(Research/Factor) → 研究层(体检) → 回测层(验证) → 知识库(入库评审) | 因子库新条目 + 实验记录 + 否决记录 | §9 |
| **闭环 B：事件-持仓闭环** | `pipeline_news2position`（新闻→事件→持仓分析） | 盘中 5–15 分钟轮询 + 盘后批处理 | 数据层(新闻/公告) → Agent 层(News) → 事件库 → Agent 层(Market/Portfolio) → 展示层(晨报/盘中提示) | events.jsonl + 持仓影响分析 | §10 |
| **闭环 C：复盘-预案闭环** | `pipeline_daily_review`（收盘→复盘→次日预案→沉淀） | 每交易日 16:15（cron） | 数据层(EOD) → Agent 层(Market/Portfolio/Review) → 知识库(postmortems/playbook) → 展示层(复盘报告/次日预案) | 复盘报告 + 次日情景预案 + 行为纠偏记录 | §11 |

三大闭环共同依赖一个**长期积累层**（§5.5 知识库 + §12 因子库）：`factors/`（个人因子库）、`knowledge/`（研究知识库）、`trading_journal`（交易行为数据库）。闭环 C 的复盘结论反哺闭环 A 的选题优先级（"哪些类因子在当前市场阶段有效"），闭环 B 的事件命中率（`event_feedback`）反哺事件抽取模型迭代——这是"持续迭代"的落点。

### 1.3 三条设计主线（贯穿全文）

1. **开源优先、自研只做薄层**（§4/§16）：采集/回测/因子分析/实验追踪/Agent 运行时全部复用成熟开源项目；自研收敛为 4 类薄件（ETL-调度-校验-PIT、事件库、持仓/复盘业务逻辑、MCP server）。
2. **证据与情景，而非买卖结论**（§11.4）：所有面向用户的输出是"结构化证据 + 情景分析 + 关键价位 + 风险点"，系统不输出自动下单指令；调仓建议必须人工确认（依据调研 04 §5.2 自动化边界）。
3. **可审计、可重放、防自欺**（§13）：raw 层保留原始响应、PIT 多版本保留、universe 快照防幸存者偏差、多重检验校正防数据窥探、四层防线防重复实验。

---

## 2. 推荐开源项目清单（总表）

> 依据：调研 01 §0/§2/§3/§6、调研 02 §0/§2.2/§3.2/§5.1、调研 03 §1.2/§2.1/§5.1/§7、调研 04 §1/§2。stars / License 为 2026-09-22 快照。

| 项目 | 定位 | 在本系统中负责什么 | 为什么选它（一句话） | 复用 or 自研 | 优先级 |
|---|---|---|---|---|---|
| **baostock** | A 股行情/财务采集（自有 TCP 服务器） | 日线/5-60 分钟线、复权因子、分红送股、季频财务（含 pubDate）、含退市股的证券列表——**核心历史库第一数据源** | 免费、BSD、不抓网页合规风险最低、含退市股与 PIT 所需披露日（调研 01 §2.2） | 直接复用 | P0 |
| **akshare**（22,692★，MIT） | 广度数据采集（东财/新浪/同花顺等公开接口） | 指数/板块概念/资金流/两融/龙虎榜/涨跌停、宏观利率汇率大宗、财联社/东财/新浪快讯与巨潮公告列表、港美行情——**兜底采集层** | 1000+ 接口免费广度之王；作第二数据源与覆盖面上限（调研 01 §2.2、调研 03 §1.3） | 直接复用 | P0 |
| **tushare**（15,409★，BSD-3） | 官方授权商业数据 API（积分制） | 财务三大表/股本/分红/两融/龙虎榜/申万分类/宏观体系/公告与结构化事件——**合规基线 + 三源对账** | 2000 积分=200 元/年补齐全部结构化缺口且无 ToS 风险（调研 01 §2.2/§3.2） | 直接复用 | P0 |
| **yfinance**（25,310★，Apache-2.0） | 外围市场行情 | 美股指数/港股/黄金/原油/美债/美元指数/人民币汇率 | 一个库拿全外围资产、免费稳定（调研 03 §5.1） | 直接复用 | P1 |
| **fredapi**（1,658★，Apache-2.0） | FRED 宏观序列客户端 | DGS10 美债、美元指数、汇率、WTI、伦敦金定盘价 | 宏观口径最权威（调研 03 §5.1） | 直接复用 | P2 |
| **microsoft/qlib**（48,746★，MIT） | AI 量化研究基座 | **数据层**（qlib_bin 格式/DataLoader/PIT 数据库/dump_bin/健康检查）+ **因子表达式引擎** + Handler/Dataset + 模型接口 + **回测**（Exchange/Executor/TopkDropout）+ RollingGen 滚动样本外 + QlibRecorder | 功能覆盖最全、A 股微观结构（涨跌停/停牌/整手/费用/容量）原生建模、混合型（向量化研究+事件化执行）（调研 02 §7） | 直接复用+少量改造 | P0 |
| **alphalens-reloaded**（653★，Apache-2.0；原版 4,449★ 停滞） | 单因子分析事实标准 | IC/RankIC/分层收益/换手/衰减的"因子体检" | 一键 tearsheet；用续维护 fork 避开停滞（调研 02 §2.1.7） | 直接复用 | P0 |
| **quantstats**（7,651★，Apache-2.0） | 组合绩效报告 | Sharpe/Sortino/Calmar/最大回撤/月度热力图/滚动指标 tearsheet | 复盘报告性价比最高（调研 02 §2.1.9） | 直接复用 | P0 |
| **MLflow**（28,095★，Apache-2.0） | 实验追踪/模型登记 | 一回测一 run；params/metrics/artifacts/tags；LLM tracing 记账 | qlib QlibRecorder 原生基于 MLflow，零接入成本（调研 02 §5、调研 04 §3.5） | 直接复用 | P0 |
| **Optuna**（14,832★，MIT） | 超参搜索 | 模型超参与策略参数搜索（嵌套在滚动框架内） | TPE/剪枝/Dashboard，与 MLflow 集成简单（调研 02 §5.1） | 直接复用 | P1 |
| **microsoft/RD-Agent**（14,711★，MIT） | LLM 自动化 R&D | 闭环 A 的 Alpha 挖掘执行体：研报/论文→因子代码→qlib 回测→迭代 | qlib 生态钦定、MIT、"表达式因子挖掘+LLM"当前最优开源实现（调研 02 §3.1.1） | 直接复用 | P1 |
| **gplearn**（1,889★，BSD-3） | 遗传规划/符号回归 | 因子挖掘 GP 对照组（自写 IC 适应度） | 最省事 GP 引擎、BSD 宽松（调研 02 §3.1.3） | 直接复用 | P2 |
| **ricequant/rqalpha**（6,787★，**Apache-2.0+仅非商业**） | 事件驱动回测 | 入围策略的高保真撮合复核（严格 T+1/涨跌停拒单/税费 mod）；公司行为处理逻辑参考 | A 股交易规则建模最贴近实盘；**License 隔离使用**（调研 02 §2.1.2、调研 01 §6.3） | 直接复用（隔离） | P2 |
| **hikyuu**（3,522★，Apache-2.0） | C++ 内核快速回测 | 传统规则型选股/择时的极速验证沙盒 | 日线级组合回测极快、A 股原生、中文文档（调研 02 §2.1.6） | 直接复用（可选） | P2 |
| **vnpy**（45,502★，MIT） | 实盘交易接入平台 | 未来接实盘时的交易执行层 | 国内实盘生态最完整（调研 02 §2.1.4） | 直接复用（远期） | P2 |
| **use_cninfo**（29★，MIT） | 巨潮公告抓取 | 公告列表→PDF→Markdown 全文归档 | 公告全文链路最佳现成件、产出天然适配 LLM（调研 03 §1.2） | 直接复用 | P1 |
| **1e0ng/simhash**（≈1k★，MIT） | 标题近重复去重 | 近似去重层（P1 按需引入；首期仅来源 ID+内容哈希，A11） | 标准实现（调研 03 §4.2） | 直接复用 | P1 |
| **ekzhu/datasketch**（≈3k★，MIT） | MinHash/LSH | 近似去重层批量候选对召回（P1 按需；A11） | 海量快讯近似去重（调研 03 §4.2） | 直接复用 | P1 |
| **shibing624/text2vec**（≈5k★，Apache-2.0） | 中文句向量 | 语义并簇 + 向量库嵌入（P1 按需；首期不启用，A11） | 中文语义相似度开箱即用（调研 03 §2.1） | 直接复用 | P1 |
| **bardsai/finance-sentiment-zh-base**（HF 模型） | 中文金融情感三分类 | 全量新闻情绪打分（本地 CPU 可跑） | 现成、轻量、每秒数百条（调研 03 §2.1/§6.3） | 直接复用 | P0 |
| **ProsusAI/finbert**（2,237★，Apache-2.0） | 英文金融情绪 BERT | 英文快讯（美债/美联储/美股）情绪 | 与中文模型互补（调研 03 §2.1） | 直接复用 | P2 |
| **FinGPT**（21,272★，MIT） | 金融 LLM 配方与数据 | 情绪/事件分类器微调配方（LoRA）+ 中文微调数据集 | 金融 LLM 事实标准；持续改进本地模型的依据（调研 03 §2.1/§6.3） | 复用思路与数据 | P1 |
| **DeepKE**（≈4.5k★，MIT） | NER/RE/事件抽取工具箱 | 事件抽取小模型兜底（低成本批量） | 传统小模型方案最成熟（调研 03 §2.1） | 需改造（schema 适配） | P2 |
| **zjunlp/OneKE**（194★，MIT） | LLM schema 约束抽取 | 事件抽取 JSON 契约输出的 prompt/结构参考 | 直接产出结构化事件 JSON（调研 03 §2.1） | 需改造（取思路） | P1 |
| **run-llama/llama_index**（≈52k★，MIT） | 嵌入式 RAG | 知识库检索（论文/复盘/公告证据链） | 单机轻量最灵活（调研 03 §2.1） | 直接复用 | P1 |
| **infiniflow/ragflow**（≈91k★，Apache-2.0） | 深度文档解析 RAG | 长公告/年报 PDF 深度解析问答（可选增强） | PDF 解析/切分/引用溯源成熟（调研 03 §2.1） | 直接复用（可选） | P2 |
| **TradingAgents**（108,032★，Apache-2.0） | 多 Agent 交易框架 | 角色分工与多空辩论结构的参考范式（数据源换 A 股） | 最火金融多 Agent 实现，整段借鉴其架构（调研 03 §2.1、调研 04 §2.2） | 需改造（取范式） | P1 |
| **FinRobot**（8,049★，Apache-2.0） | 金融 Agent 平台 | 公司情绪分析/研报生成等 prompt 与流程拆用 | AI4Finance 成熟 Agent 库（调研 03 §2.1） | 需改造（拆用） | P2 |
| **deepseek-ai/deepseek-harness（DSH）**（232,442★，MIT） | Coding agent 运行时 + 多 Agent 编排底座 | **Orchestrator**：goal 长程循环、Agent Teams 任务 DAG+质量门、workflow、MCP client、webhook、schedule、token 计量；兼任 Factor/Backtest coding 执行体 | 免费接 DeepSeek API、编排原生、quality gate 对应回测验收流程（调研 04 §2.1.2） | 直接复用（自写配置/插件薄层） | P0 |
| **openai/codex**（125,873★，Apache-2.0） | Coding agent CLI | 第二实现者/互审：`codex exec` 无人值守批量改代码、审 DSH diff | Apache-2.0、`codex exec` 天然适合流水线（调研 04 §2.1.3） | 直接复用 | P1 |
| **XiaomiMiMo/MiMo-Code**（13,338★，MIT） | Coding agent CLI | 低成本批量小改动的备选执行体 | 便宜且开源（调研 04 §2.1.4） | 直接复用（备选） | P2 |
| **Hermes-4-14B**（Apache-2.0，HF） | 本地开源权重推理模型 | 纯推理岗位：News 事件抽取、Research 精读、Market 解读、Review-lite——**隐私零外泄、边际成本≈0** | 函数调用/结构化输出原生、Mac mini 32GB 可跑 GGUF-Q4/Q5（调研 04 §2.1.1） | 直接复用（本地部署） | P0 |
| **modelcontextprotocol/python-sdk**（24,350★，MIT，v2.2.0） | MCP server SDK | 自建 3 个私有 MCP server 的实现框架 | 工具只写一遍、DSH/Codex/Claude Code 全支持（调研 04 §2.3） | 直接复用 + **自研 server** | P1 |
| **langchain-ai/langgraph**（42,122★，MIT） | 编排框架（备选） | 若不用 DSH 内建编排时的状态图 Orchestrator 备选 | checkpoint 断点续跑；但**不与 DSH 叠两层编排**（调研 04 §2.2） | 备选（默认不用） | P2 |
| **DuckDB / Parquet / SQLite / MinIO** | 存储 | DuckDB=研究查询引擎；Parquet=事实表归档；SQLite=元数据/水位/任务状态；MinIO=冷备快照（P2） | 全 arm64 原生、零运维组合（调研 01 §7） | 直接复用 + 自研 schema | P0 |
| **DVC**（15,881★，Apache-2.0）/ **Kedro**（11,006★，Apache-2.0） | 数据版本化 / 流水线框架 | 数据超 10GB 或流程节点 >10 后再引入（qlib_bin、分钟线版本化） | 避免过度工程（调研 02 §5.2） | 直接复用（延后） | P2 |
| **Streamlit** | 本地 Web 展示 | 个人控制台/报告阅读器 | Python 原生、与 pandas/DuckDB 无缝 | 直接复用 + 自研页面 | P1 |

**明确不采用（被放弃的备选与理由）**：

| 项目 | 放弃理由 |
|---|---|
| backtrader（23,302★，GPL-3.0，半停滞） | A 股规则全需自建 + GPL 传染 + 停滞（调研 02 §2.1.3） |
| backtesting.py（8,982★，**AGPL-3.0**） | AGPL 传染性最强（网络服务也算分发），仅单标的；可用但默认不入正式依赖（调研 02 §2.1.10） |
| alphagen（1,242★，**无 License**） | 无 License 默认保留所有权利；只借鉴 RL 挖因子思路（调研 02 §3.1.2） |
| Ashare（3,869★，**无 License**） | 无授权条款；同类能力用 easyquotation（MIT）（调研 01 §2.2） |
| zipline-reloaded（1,941★） | 美股中心设计，A 股需大量改造，不如 qlib 划算（调研 02 §2.1.5） |
| AutoGen / CrewAI / MetaGPT / Agno / smolagents | 与 DSH 叠两层编排违反"一个运行时 + 多个执行体"原则（调研 04 §2.2 补充观察） |
| investpy（1,856★） | 反爬频繁失效，能力被 yfinance/akshare 覆盖（调研 03 §5.1） |
| FinSpider / wallstreetcnScrapy / 新浪长尾爬虫 | 无 License 或停更；情绪源思路可借鉴但自行重写（调研 03 §1.2） |
| vectorbt（9,152★，自定义 NOASSERTION） | 可用作参数扫描，但许可非标准；参数扫描用 Optuna+qlib 已覆盖，P2 需要时再审查引入（调研 02 §2.1.11） |
| 通联 DataAPI / 优矿 | 个人渠道不确定（调研 01 §3.2） |
| Kafka / RabbitMQ | 单机运维成本大于收益；SQLite tasks 表即队列（调研 04 §3.4） |

---

## 3. 各项目职责与选型理由（整合 4 份调研的对比结论）

> 依据：调研 01 §2.3/§3/§6/§10、调研 02 §2.2/§3.2/§4/§6/§7、调研 03 §2.2/§7、调研 04 §2.1/§3.7。

### 3.1 数据采集层：三源采集 + 多源仲裁

**结论（调研 01 §2.3）**：核心双源 **baostock（主）+ akshare（辅/兜底）**，加 **tushare 2000 积分档（200 元/年）** 做三源对账与财务补齐。

| 能力 | 首选 | 备选/交叉校验 | 被放弃的方案与理由 |
|---|---|---|---|
| 日线/分钟线 | baostock（免费稳定、含退市股、复权因子） | akshare（广度）、efinance（东财轻量） | 东财抓取源对退市股覆盖不全，**不可作唯一来源**（调研 01 §9.4） |
| 财务/基本面 | tushare 2000 积分（三大表+分红+股本） | baostock 季频（含 pubDate，天然 PIT）、akshare 财报 | 免费源都无"历史修订多版本"，必须自建 PIT 表（调研 01 §3.1） |
| 指数/板块/资金流/两融/龙虎榜 | akshare（免费） | tushare（THS/DC 双口径校验） | — |
| 宏观/利率/汇率/大宗/港美 | akshare + tushare | yfinance、fredapi | investpy 反爬失效（调研 03 §5.1） |
| 新闻/公告 | akshare（财联社/东财/新浪/巨潮列表） | use_cninfo（PDF→Markdown）、tushare 公告结构化 | 独立爬虫仓普遍停更且无 License（调研 03 §1.2） |
| 深度分钟线（2009 起 1 分钟） | **tushare 历史分钟包（2000 元一次性）**——免费与付费最大分界 | — | 免费方案只有 baostock 5 分钟起（2011 起）与东财浅历史（调研 01 §2.3） |

**选型理由对比结论**：
- **baostock 作压舱石**：自有数据服务器 + 专用 TCP 协议、不抓网页、接口十年不变；自带 `preclose`/`tradestatus`/`isST`、复权因子、除权除息明细、`pubDate`（PIT 关键字段）、含退市股的 `ipoDate/outDate/status`——这些字段直接决定回测数据质量下限（调研 01 §2.2）。缺点（无板块/资金流/两融、字段全字符串、pandas 2.x 需手工迭代）由 akshare/tushare 补齐。
- **akshare 作广度兜底**：22,692★、MIT、近日报活跃、1000+ 接口；但抓取型接口会随上游改版失效（调研 01 列举 issue #6143/#7180/#6574），故固定放 **ETL 第二数据源** 位，核心事实表以 baostock/tushare 双校验。
- **tushare 作合规基线**：唯一可放心用于严肃回测的商业源；200 元/年档（2000 积分、200 次/分）补齐股本变动、分红明细、两融、龙虎榜、申万分类。**注意**：SDK 仓库 2024-03 后停滞但服务端持续运营（ICP 2026 签发），走 pro HTTP API 而非依赖 SDK 更新（调研 01 §2.2、调研 03 §1.2）。
- **efinance/qstock/easyquotation**：仅作交叉校验或实时快照（easyquotation MIT、5,398★，盘中监控场景备选），不入核心依赖。

### 3.2 研究/回测层：qlib 为基座的两段式

**结论（调研 02 §6/§7.9）**：研究段用向量化（Alphalens + qlib 表达式引擎，分钟级证伪想法），验证段用事件化执行（qlib Exchange 全微观结构建模）；qlib 恰好是混合型框架，一套内完成两段——这是选它做主基座的核心理由。

| 对比项 | qlib（选定） | rqalpha | hikyuu | backtrader | zipline-reloaded |
|---|---|---|---|---|---|
| Stars/License | 48,746 / MIT | 6,787 / **仅非商业** | 3,522 / Apache-2.0 | 23,302 / **GPL-3.0** | 1,941 / Apache-2.0 |
| 维护（2026-09-22） | 活跃（日更） | 活跃 | 活跃 | 半停滞（2024-08） | 低频 |
| 复权 | `$factor` 复权因子，撮合按真实价还原 | 数据层支持 | 内建前/后复权 | 数据自理 | 无 |
| 涨跌停 | `limit_threshold` 原生（可按板块表达式） | 撮合+风控原生 | 条件过滤 | 无 | 无 |
| T+1 | 时序天然满足；无持仓锁（日频无影响） | **原生（closable）** | 延迟一日近似 | 无 | 无 |
| 费用/容量 | open/close/min/impact cost + volume_threshold | 佣金+印花税 mod | 可配 | CommissionInfo | 可配 |
| PIT 防前视 | **原生 PIT 数据库** | 无 | 无 | 无 | 无 |
| 实验记录 | QlibRecorder（MLflow 后端） | 无 | 无 | 无 | 无 |
| 复用判断 | **主基座** | 高保真复核（隔离） | 极速验证沙盒（可选） | 放弃 | 放弃 |

**关键细节（调研 02 §7）**：
- Alpha158/Alpha360 开箱即用（US/China 双市场、20+ 基准模型对照表），且本质是 YAML 表达式清单，可复制改造；
- label 默认 `Ref($close,-2)/Ref($close,-1)-1`（T+1 收盘成交假设），自定义执行假设必须同步改 label（调研 02 §7.2，issue #1514）；
- `RollingGen` 免费提供滚动训练/回测（walk-forward），无需自研；
- 需改造仅 6 小项（数据源接入/费用校准/涨跌停参数/label 定制/严格 T+1 持仓锁/分阶段报告胶水），工作量均小（调研 02 §7.9）。

**单因子体检与报告**：**alphalens-reloaded**（IC/RankIC/分层/换手/衰减三件套+）与 **quantstats**（tearsheet）均为事实标准、Apache-2.0、直接复用；A 股使用注意四点——后复权收益、剔除涨跌停不可交易样本、剔除 ST/次新/停牌、年化 252→244（调研 02 §4.2）。**empyrical-reloaded** 作函数级备用（调研 02 §2.1.8）。

**因子挖掘三条路线对照**：RD-Agent（LLM，MIT，14,711★）为主线；gplearn（GP，BSD-3）作对照组；alphagen（RL）只借鉴思路（无 License）。表达式因子是个人研究性价比最高的形态：可解释 + 可增量计算 + 天然进 qlib 管道（调研 02 §3.1.4）。

### 3.3 事件/AI 分析层：本地小模型 + 云端 API 分工

**结论（调研 03 §2.2/§6.3）**：按"量大便宜走本地、复杂推理走云端"分工：

| 环节 | 选定 | 理由 |
|---|---|---|
| 去重 | 首期 `(source, source_doc_id)` + 归一化内容哈希 + 修订链登记；simhash/text2vec 语义层 P1 按需 | 确定性去重与关系登记（转载/更正保留可区分），不设固定滤重比例（A11，调研 03 §4.3 v2） |
| 分类 | 规则硬分类（巨潮 category/来源频道）+ 微调 BERT 15 类事件分类 | 规则免费、BERT 本地 CPU 每秒数百条 |
| 情绪/方向 | **finance-sentiment-zh-base**（中文全量）+ finbert（英文）+ 重点事件 LLM 判方向/期限 | 全量打分成本≈0；重点事件才花 LLM 钱 |
| 摘要 | 短快讯免摘要；长公告本地 Qwen/Hermes 7B–14B"5W+金额"；晨报云端 | 长公告复杂推理本地 14B 易错（调研 03 §6.1） |
| 事件抽取 | LLM + JSON schema 约束（OneKE/DISC-FinLLM prompt 思路）；DeepKE 兜底 | JSON 契约五要素：事件类型/标的/方向/置信度/证据（调研 03 §3.3） |
| RAG/证据检索 | **LlamaIndex**（单机轻量）；RAGFlow 备选（深度 PDF 解析） | Mac mini 单机首选（调研 03 §2.1） |

### 3.4 Agent/编排层：一个运行时 + 多个执行体

**结论（调研 04 §1/§3.7）**：**DSH 做 Orchestrator + coding 主力**，**Hermes-4-14B 本地做纯推理**，**DeepSeek API 做云端兜底与难题**，**Codex CLI 做第二实现者/互审**。不叠 LangGraph 第二层编排。

| 决策点 | 选定 | 被放弃的备选 | 理由 |
|---|---|---|---|
| Orchestrator | DSH goal + Agent Teams / workflow | LangGraph / AutoGen / CrewAI / MetaGPT | DSH 运行时原生带 goal 长程循环、子 Agent、后台任务、MCP client、webhook、schedule（232,442★，MIT）；个人系统不需要重编排抽象（调研 04 §2.2） |
| Coding 执行体 | DSH 自身 + Codex CLI（`codex exec`） | MiMo Code、Claude Code、OpenHands、Aider | DSH 免费接 DeepSeek API 且编排原生；Codex Apache-2.0 适合无人值守互审；Claude Code 无开源 License + 订阅周限额（调研 04 §2.1） |
| 纯推理 | Hermes-4-14B 本地（llama.cpp/Ollama/MLX） | 纯云端 API | Apache-2.0、函数调用/JSON mode 原生、隐私零外泄、边际成本≈0（调研 04 §2.1.1） |
| 工具层 | **MCP**（自建 3 私有 server） | 各框架私有工具协议 | 宿主全覆盖（DSH/Codex/Claude Code/OpenHands），工具只写一遍（调研 04 §2.3） |
| 实验追踪 | MLflow 3.x（SQLite/文件 backend） | W&B | qlib QlibRecorder 原生 MLflow；W&B 图形更好但数据上云需评估隐私（调研 02 §5.2） |

**职责切分判定标准（调研 04 §3.7）**：需要"写文件+跑命令+看报错迭代"的用 coding agent（Factor/Backtest 代码生成与修复）；"数字输入→判断输出"的用纯推理（论文精读/结果解读/事件抽取/复盘质询）；调度本身是确定性逻辑，尽量代码化，LLM 只做异常处置。推荐混合模式：`纯推理产出 spec.json → coding agent 按 spec 写代码 → 确定性引擎跑回测 → 纯推理评审`。

---

## 4. 直接复用 vs 需要自研的能力边界

> 依据：调研 01 §10（结论：采集 100% 复用，自写收敛为"调度+ETL+校验+PIT"四件薄层）、调研 02 §4.2（唯一自研点：~50 行滚动稳定性脚本 + ~30 行分阶段切分）、调研 03 §3.1（事件库 schema 自研）、调研 04 §2.3/§3.6（MCP server 自研、防重复四层）。
>
> **总原则：只开发真正有差异化价值的部分。** 个人量化的差异化价值在：①个人数据资产的正确性（PIT/对账/退市股）；②个人交易行为的闭环（持仓/复盘/预案）；③个人知识积累（因子库/知识库）。其余一律复用。

### 4.1 能力边界总表

| 能力域 | 直接复用（不写） | 自研薄层（只写这些） | 自研代码量级估计 |
|---|---|---|---|
| 行情/财务/新闻采集 | baostock / akshare / tushare / use_cninfo / yfinance 客户端 | 调度（launchd/dsh-schedule）、水位表、增量窗口回看、多源仲裁 | ~500 行 |
| 数据落地 | DuckDB / Parquet / SQLite / MinIO 现成组件 | 表 schema、raw→staging→clean 分区设计、重放脚本 | ~400 行 |
| PIT 与防前视 | qlib PIT 文件格式、PITProvider 查询、`dump_bin` | PIT 版本表写入逻辑（ann_date 多版本保留）、point-in-time 校验器 | ~300 行 |
| 数据质量 | qlib `check_data_health`、rqalpha 撮合规则（参考） | DQ 规则集（结构/数值/对账/除权除息一致性）+ dq_issue 报告 | ~300 行 |
| 因子研究/回测 | qlib（表达式引擎/Handler/Dataset/Exchange/Executor/RollingGen）、alphalens-reloaded、quantstats、hikyuu、rqalpha | 因子注册模板、因子准入卡脚本（IC_IR/分层单调性阈值判定）、滚动稳定性脚本（~50 行）、分市场阶段切分（~30 行） | ~400 行 |
| 实验追踪 | MLflow、QlibRecorder、Optuna | 一回测一 run 的登记封装（params/metrics/tags 规范）、idem_key 生成 | ~200 行 |
| 因子库/版本/血缘 | — | **因子库 schema + 注册/入库评审/衰减监控**（差异化核心） | ~500 行 |
| 事件分析 | simhash / datasketch / text2vec / finance-sentiment-zh-base / FinGPT 配方 / DeepKE / OneKE 思路 / LlamaIndex | **事件库 schema（调研 03 §3.1）、去重与修订链流水线编排（首期 ID+哈希，语义层延后）、事件抽取 JSON 契约校验、event_feedback 回写** | ~600 行 |
| 持仓/复盘/预案 | quantstats（指标）、TradingAgents（辩论范式） | **持仓分析、逐只持仓的情景预案生成编排、复盘模板、交易行为数据库**（差异化核心） | ~800 行 |
| Agent 编排 | DSH（goal/Agent Teams/workflow/schedule/webhook）、Codex CLI、Hermes 本地推理 | Orchestrator 配置、任务状态机持久化（tasks.sqlite）、防重复四层防线、prompt 模板集 | ~600 行 |
| MCP 工具层 | modelcontextprotocol/python-sdk | **3 个私有 MCP server**：`ashare-data` / `backtest` / `mlflow-kb`（差异化核心） | ~800 行 |
| 展示层 | Streamlit、MLflow UI | 3–5 个页面（研究看板/复盘预案阅读器/因子库浏览器） | ~600 行 |

**合计自研约 5,000 行级**（不含测试），全部是"胶水+业务规则+schema"，无重复造轮子成分。任何超出上表右列的开发需求，先走 §16 的自研准入标准。

### 4.2 五条硬边界（写进仓库 CONTRIBUTING）

1. **不写采集器**：一切数据获取走 baostock/akshare/tushare/yfinance/use_cninfo 客户端；只有政策类页面（证监会/央行）允许 `requests+BeautifulSoup` 级薄爬虫（调研 03 §1.3）。
2. **不写回测引擎**：回测只用 qlib Exchange/Executor；rqalpha 仅作复核；如发现 qlib 不满足，优先提配置/表达式解决，其次 `limit_threshold`/Strategy 层小改，**禁止自研撮合**。
3. **不写 ML/训练框架**：模型接入只实现 qlib `Model.fit/predict`；挖掘用 RD-Agent/gplearn；调参用 Optuna。
4. **不写向量库/LLM 网关/RAG 引擎**：向量存储用 LanceDB 或 sqlite-vec，RAG 用 LlamaIndex，模型服务用 Ollama/llama.cpp + OpenAI 兼容 API。
5. **Agent 不直接改量化栈内部**（调研 04 §2.4）：只通过 (a) 仓库内 Python 代码/YAML、(b) MCP 工具、(c) CLI 命令三个边界交互——边界清晰才能幂等与审计。

### 4.3 自研准入标准（与 §16.2 呼应）

满足全部三条才允许自研：①GitHub/成熟开源中**确实不存在**满足需求的项目（附检索证据）；②该能力**构成差异化价值**（个人数据正确性/交易闭环/知识积累三类之一）；③可实现为**薄层**（<1k 行、可整体替换、无框架化倾向）。否则降级为"配置/脚本/上游 PR"。

---

## 5. 逐层详细设计（数据层 → 研究层 → 回测层 → Agent 层 → 知识库 → 任务调度层 → 展示层）

> 依据：调研 01 §7/§8/§9/§10（存储/增量去重/质量校验）、调研 02 §5/§7（研究接口）、调研 03 §3.1（事件库 schema）、调研 04 §3.3/§3.4/§3.5（状态机/文件约定/MLflow）。

### 5.0 顶层目录约定（monorepo，详见 §15.2）

```
quant-lab/                        # git 主仓库（MIT，自研薄层全在此）
├── data/                         # ①数据层代码（不含数据本体）
│   ├── collectors/               #   采集适配器（薄封装 baostock/akshare/tushare/yfinance/use_cninfo）
│   ├── etl/                      #   raw→staging→clean 转换、dump_bin 管道
│   ├── dq/                       #   质量校验规则集与报告生成
│   └── schema/                   #   表 schema DDL、schema_version 迁移
├── factors/                      # ②研究层：个人因子库（注册表 + 因子定义）
├── backtest/                     # ③回测层：策略/回测配置模板、准入卡脚本
├── agents/                       # ④Agent 层：9 类 Agent 定义、prompt 模板、执行体配置
├── pipelines/                    # ⑤三大闭环 Pipeline 编排定义
├── knowledge/                    # ⑥知识库（git 管理的 Markdown；向量索引不入 git）
├── apps/                         # ⑦展示层（Streamlit 页面）
├── orchestration/                #   任务调度层：状态机、防重复防线、MCP server 实现
├── experiments/<experiment_id>/  #   实验工作区（见 §5.4；.gitignore 中排除 worktree 运行态）
├── infra/                        #   docker-compose、launchd plist、备份脚本
└── state/                        #   运行态（不入 git）：tasks.sqlite、cache/、mlruns/、qlib_bin/
```

数据本体（Parquet/DuckDB/qlib_bin/原始响应）放在仓库外的 `~/quant-data/`（见 §6.2 磁盘规划），仓库内只放代码、schema、知识文本与小型注册表。

### 5.1 数据层（data/ + etl/）

#### 5.1.1 存储布局与三层模型

raw → staging → clean 三层，**全部幂等可重放**（调研 01 §8）：

```
~/quant-data/
├── raw/<source>/<dataset>/<ingest_date>/...        # 原始响应包（JSON/CSV/PDF），审计与重放证据
├── staging/                                        # 去重/类型化后的暂存（可随时重建）
├── clean/market/freq=daily/market=CN/year=YYYY/…parquet    # 事实表（按 (market, freq, year) 分区）
├── clean/fundamental/…                              # 财务/股本/分红事实表
├── clean/reference/…                               # 交易日历/证券信息/行业分类/复权因子
├── clean/events/…                                  # 新闻/公告/事件（亦入库 DuckDB）
├── qlib_bin/                                       # qlib 格式物化视图（dump_bin 生成，可随时重建）
├── duckdb/app.duckdb                               # 研究查询库（只读视图：事件库+视图+注册表）
├── sqlite/meta.sqlite                              # 只存任务元数据：水位/日历/DQ/任务（WAL）
└── snapshots/                                      # 不可变 Parquet 快照 + manifest（snapshot_id，见 §5.1.11）
```

#### 5.1.2 核心 schema（DDL 摘要；完整版在 `data/schema/`）

```sql
-- ①行情事实表（Parquet；主键 (code, freq, trade_date)，分钟加 bar_time）
--    原则：只存不复权价 + 复权因子；前/后复权在查询层动态计算（调研 01 §9.3）
CREATE TABLE bar_daily (
  code TEXT, trade_date DATE, freq TEXT DEFAULT 'daily',
  open DOUBLE, high DOUBLE, low DOUBLE, close DOUBLE,
  preclose DOUBLE,                 -- 用于除权除息一致性校验
  volume DOUBLE, amount DOUBLE,
  adj_factor DOUBLE,               -- 后复权因子（除权事件回填重算）
  tradestatus INT, is_st BOOLEAN, is_suspended BOOLEAN,
  turn DOUBLE, pe DOUBLE, pb DOUBLE,
  PRIMARY KEY (code, freq, trade_date)
);

-- ②PIT 财务版本表（防未来函数核心；调研 01 §8.3/§9.5，格式对齐 qlib PIT 四列设计）
CREATE TABLE financial_pit (
  code TEXT, report_period DATE,   -- 报告期 statDate
  ann_date DATE,                   -- 披露日（版本键；同 (code,period,item) 可多行）
  item TEXT, value DOUBLE,
  fetched_at TIMESTAMP,            -- 本次抓取时间（版本证据；补不回历史原始版本，见 §5.1.9）
  _next BIGINT,                    -- 指向同 item 下一版本（复用 qlib 链表设计）
  PRIMARY KEY (code, report_period, item, ann_date)
);
-- 查询"t 时刻可见版本"：WHERE ann_date <= t AND report_period <= t
--   取 max(ann_date) 的行（含业绩预告/快报的 pubDate/statDate 双字段，baostock 天然提供）
--   注意（A01）：fetched_at/_next 本身补不出历史 PIT 版本；ann_date 只有日期无披露时刻时，
--   按 §5.1.9 保守下一交易时段规则取 t，禁止默认当日开盘可用；无法恢复原始版本的字段
--   标 historical_pit_unverified 并阻断进入声称无前视的历史研究

-- ③universe 快照表（防幸存者偏差核心；调研 01 §9.4）
CREATE TABLE universe_snapshot (
  trade_date DATE, code TEXT,
  listed BOOLEAN, delist_date DATE, is_st BOOLEAN, is_new BOOLEAN,  -- 次新标记
  board TEXT,                      -- main/gem/star/bj（涨跌停档位依据）
  suspend BOOLEAN,                 -- 当日停牌（不可成交）
  PRIMARY KEY (trade_date, code)
);
-- 回测选股只能用当日快照，绝不用"当前股票列表"回溯

-- ④行业/概念成分历史（分类体系本身就是数据；调研 01 §4.3）
CREATE TABLE industry_map (
  classification TEXT,             -- SW2021 / CSRC / DC / THS
  code TEXT, name TEXT,
  in_date DATE, out_date DATE,     -- 区间表，绝不只存当前成分
  PRIMARY KEY (classification, code, in_date)
);

-- ⑤公司行为表（分红送转/除权除息；按 ann_date 做 PIT）
CREATE TABLE corporate_action (
  code TEXT, ex_date DATE, ann_date DATE,
  cash_div DOUBLE, stock_div DOUBLE, allot_ratio DOUBLE,
  preclose_check DOUBLE, dq_ok BOOLEAN, PRIMARY KEY (code, ex_date, ann_date)
);

-- ⑥采集水位与增量（SQLite meta.sqlite；调研 01 §8.2）
CREATE TABLE watermark (dataset TEXT, code TEXT, freq TEXT,
  last_date DATE, ingested_at TIMESTAMP,
  PRIMARY KEY (dataset, code, freq));
-- 增量窗口固定回看 5 个交易日重抓覆盖，吸收上游追溯修订

-- ⑦数据质量问题表（多源差异超阈值写入，人工复核，不静默覆盖）
CREATE TABLE dq_issue (
  id BIGINT PRIMARY KEY, found_at TIMESTAMP, dataset TEXT, key TEXT,
  rule TEXT, severity TEXT, detail JSON, status TEXT   -- open/resolved/ignored
);

-- ⑧数据可用性断点字典（结构性断点必须显式记录；调研 01 §4.4）
CREATE TABLE data_regime_change (
  effective_date DATE, dataset TEXT, description TEXT, impact TEXT,
  PRIMARY KEY (effective_date, dataset)
);
-- 首批必录：('2024-05-13','northbound','沪深港通不再实时披露买入/卖出/成交总额',
--            '盘中北向流向因子失效；改用盘后十大活跃股+季度持股变动')
```

#### 5.1.3 采集适配器接口（自研薄层的唯一契约）

```python
# data/collectors/base.py —— 所有采集器实现同一接口，返回标准化 DataFrame
class Collector(Protocol):
    source: Literal["baostock","akshare","tushare","yfinance","cninfo", ...]
    def fetch(self, dataset: str, code: str | None, start: date, end: date) -> pd.DataFrame: ...
    def healthcheck(self) -> HealthReport: ...   # 登录/配额/上游可用性

# etl 管道（幂等）：fetch → raw 落盘 → staging 清洗 → clean upsert → 水位推进
# 补充层对账（v2，§5.1.10）：历史底座 = investment_data 公开数据包；BaoStock/AKShare 仅补字段
#   与抽样对账；|Δclose|/close > 0.5% 或复权因子不一致 → 写 dq_issue（调研 01 §8.4），不静默覆盖
```

**调度接口**：每数据集一个 `etl run --dataset bar_daily --date <d>` CLI；由任务调度层（§5.6）以 idem_key=`sha256("etl"+dataset+date)` 派发，保证幂等。

#### 5.1.4 数据质量校验（每日自动，出报告）

按调研 01 §9 实现为规则集（`data/dq/rules.py`），四组：**结构**（主键唯一、交易日历对齐——用 baostock/tushare 日历而非周一至周五）、**数值**（OHLC 关系、volume≥0、涨跌幅不超板块档位 ±10%/±5% ST/±20% 创业板科创板/±30% 北交所，注意 2020-08 创业板改 20%）、**对账**（抽样第二源比对收盘价、组合收益与沪深300/中证500 相关性 sanity check）、**除权除息一致性**（`preclose ≈ (前收-每股派现)/(1+送转)` 偏差超阈值报 dq_issue）。停牌日显式写 `is_suspended` 行，禁止前向填充后当连续交易。每日产出 `runs/data/<date>/dq_report.md`。

**DQ 规则目录（首版 24 条；每条含 severity：blocker 阻断闭环 C / warn 出报告）**：

| # | 规则 | 组 | 阈值/判定 | severity |
|---|---|---|---|---|
| D01 | 主键唯一（code,freq,trade_date） | 结构 | 0 重复 | blocker |
| D02 | 交易日历对齐 | 结构 | 无日历外日期、无缺失交易日 | blocker |
| D03 | bar 时间连续性（分钟） | 结构 | 缺 bar 显式标记 is_suspended | warn |
| D04 | OHLC>0 | 数值 | 0 违例 | blocker |
| D05 | low≤min(open,close)≤max(open,close)≤high | 数值 | 0 违例 | blocker |
| D06 | volume≥0、amount≥0 | 数值 | 0 违例 | blocker |
| D07 | amount/volume 落在 [low,high] 合理区间 | 数值 | 偏差 <5% | warn |
| D08 | 日涨跌幅≤板块档位 | 数值 | 除权日豁免（用 preclose 校验） | warn |
| D09 | 复权因子单调不减（后复权） | 数值 | 非递减 | blocker |
| D10 | 除权除息一致性 `preclose≈(前收-派现)/(1+送转)` | 数值 | 偏差 >0.5% 报 issue | warn |
| D11 | 收盘价双源对账 | 对账 | \|Δ\|/close ≤0.5% | warn→issue |
| D12 | adj_factor 双源对账 | 对账 | 完全一致（口径差异记录文档） | warn→issue |
| D13 | 组合收益 vs 基准相关性 sanity | 对账 | 相关系数 ∈ [0.3, 0.95] | warn |
| D14 | 财务 pubDate ≤ 当前日期 | PIT | 0 违例 | blocker |
| D15 | financial_pit `_next` 链完整性 | PIT | 抽 20 股 100% | blocker |
| D16 | universe 快照覆盖当日全部上市股 | 结构 | 缺失 0 | blocker |
| D17 | 退市股历史完整性 | 结构 | delist 前 K 线无断档（停牌除外） | warn |
| D18 | industry_map 区间无重叠 | 结构 | 同 (classification,code) 区间不重叠 | blocker |
| D19 | 停牌日显式记录 | 结构 | 与停牌日历 100% 一致 | warn |
| D20 | 新股回拉全历史 | 结构 | 上市日至首 bar 无缺 | warn |
| D21 | news_raw.url 唯一 | 结构 | 0 重复 | warn |
| L01 | raw 层落盘率 | 重放 | 100% 请求有 raw 记录 | warn |
| L02 | 水位回看 5 日覆盖 | 重放 | 每任务执行 | warn |
| L03 | schema_version 与 manifest 一致 | 重放 | 一致 | blocker |

#### 5.1.5 数据集 → 数据源 → API 映射表（采集适配器配置即此表）

> **v2 口径（A05/审计 §11）**：历史日线底座由 investment_data 公开数据包初始化（§5.1.10），不从零采集全量；本表转为**补充采集层**映射（补字段与抽样对账用）。BaoStock/AKShare 为补充层；Tushare 等付费源仅在公开渠道无法满足**已确认**需求时评估，不预定购买档位。表中 PIT 要求的实现一律受 §5.1.9 可见性时间契约约束。

| 数据集 | 主源（baostock/tushare/akshare/…） | 具体 API | 备源 | 频率 | PIT 要求 |
|---|---|---|---|---|---|
| bar_daily（日线） | baostock | `query_history_k_data_plus(code, fields, adjustflag=1)` | akshare `stock_zh_a_hist`；tushare `daily`/`daily_adj` | 日度，17:30 | 否 |
| bar_5min…60min | baostock | `query_history_k_data_plus(freq=5/15/30/60)` | tushare `stk_mins`（独立付费） | 盘中增量+盘后补全 | 否 |
| adjust_factor | baostock | `query_adjust_factor` | tushare `adj_factor` | 日度 | 否 |
| corporate_action（分红送转） | baostock | `query_dividend_data`（含除权除息日） | tushare `dividend` | 日度轮询 | **是（ann_date）** |
| financial_pit（季频指标） | baostock | `query_profit_data/query_growth_data/...`（含 pubDate/statDate） | tushare `fina_indicator` | 财报季日频轮询 | **是** |
| financial_statements（三大表） | tushare | `income/balancesheet/cashflow` | akshare 财报接口 | 财报季日频 | **是（披露日表）** |
| share_float（股本变动） | tushare | `daily_basic`/`share_float` | akshare | 日度 | **是** |
| universe_snapshot | baostock+tushare | `query_stock_basic`（ipoDate/outDate/status）+ `stock_basic`/`namechange` | akshare | 日度 | 否（快照即 PIT） |
| industry_map | tushare | `index_classify/index_member`（申万） | baostock 行业分类（证监会）；akshare 东财/同花顺概念 | 周度+变更事件 | **是（in/out_date）** |
| margin_detail（两融） | akshare | `stock_margin_detail_sse/szse` | tushare `margin_detail` | 日度盘后 | 否 |
| money_flow（资金流） | akshare | `stock_individual_fund_flow`/板块资金流 | tushare `moneyflow`（THS/DC 双口径） | 日度盘后 | 否 |
| dragon_tiger（龙虎榜） | akshare | `stock_lhb_detail_em` | tushare `top_list/top_inst` | 日度盘后 | 否 |
| northbound（**2024-05-13 断点**） | tushare | `hk_hold`（季度持股）+ 十大活跃股 | akshare（部分接口已失效） | 盘后+季度 | 是（regime 变化） |
| suspend（停牌） | tushare | `suspend_d` | baostock tradestatus | 日度 | 否 |
| macro_cn | tushare | `cn_gdp/cn_cpi/cn_pmi/shibor/lpr`（体系化） | akshare 统计局/央行接口 | 按披露节奏 | **是（发布日）** |
| macro_us / fx / commodity | yfinance+FRED | `^TNX/DGS10/DTWEXBGS/DCOILWTICO`（fredapi） | akshare 外盘接口 | 日度+周度 | 否 |
| news_flash（快讯） | akshare | `stock_info_global_cls/_em/_sina` | cailianpress-unified（财联社细节） | 盘中 1–5/5–15 分钟 | 否 |
| news_stock（个股新闻） | akshare | `stock_news_em(symbol)` | tushare 新闻（1000 元/年） | 5–15 分钟 | 否 |
| announcements（公告列表） | akshare | 巨潮 `announcement/query` 封装 | tushare 公告/披露接口 | 15:00–23:00 每 30 分钟+08:00 补漏 | **是（ann_date）** |
| announcements_pdf（全文） | use_cninfo | PDF→Markdown（PyMuPDF） | — | 随列表增量（仅持仓股+重要类目） | 是 |
| events_structured（业绩预告/解禁/增减持） | tushare | 对应结构化接口 | akshare | 日度盘后 | **是** |
| calendar | baostock | `query_trade_dates` | tushare `trade_cal` | 每年+变更 | 否 |

#### 5.1.6 ETL 任务目录（调度层派发的最小单元；每行一个 `etl run --task <id>`）

| task_id | 数据集 | 频率/触发 | idem_key 构成 | 超时 | 失败策略 |
|---|---|---|---|---|---|
| `etl.eod.bars` | bar_daily + adjust_factor | 每交易日 17:30 | `etl‖bar_daily‖<date>` | 10min | 重试 2 次（指数退避）→ dq_issue 告警 |
| `etl.eod.reference` | suspend/universe/margin/flow/dragon | 每交易日 17:30 | `etl‖<dataset>‖<date>` | 10min | 同上 |
| `etl.finance.roll` | financial_pit/statements/share_float | 财报季每交易日 19:00 | `etl‖finance‖<date>` | 20min | 重试 1 次；缺报不阻塞（PIT 补录） |
| `etl.corpaction.backfill` | corporate_action + 复权因子回填 | 除权事件驱动 + 周日全量 | `etl‖corpaction‖<code>‖<ex_date>` | 30min | 失败即告警（影响全历史复权价） |
| `etl.minute.intraday` | bar_5min | 盘中每 5 分钟 | `etl‖bar_5min‖<date>‖<slot>` | 5min | 丢弃该 slot，盘后补全 |
| `etl.minute.eod_fill` | bar_5min/15min | 每交易日 18:30 | `etl‖minute_fill‖<date>` | 60min | 重试 2 次 |
| `etl.macro.weekly` | macro_cn/us/fx/commodity | 每周一 09:00 | `etl‖macro‖<iso_week>` | 15min | 重试 |
| `etl.news.flash` | news_flash/news_stock | 盘中 5–15 分钟 | `etl‖news‖<date>‖<slot>` | 5min | 丢弃 slot（下轮增量补） |
| `etl.news.announce` | announcements | 15:00–23:00 每 30 分钟 | `etl‖announce‖<date>‖<slot>` | 10min | 08:00 补漏兜底 |
| `etl.dq.daily` | 全部 clean 表 | 每交易日 17:50（EOD 后） | `dq‖<date>` | 15min | 出报告不阻塞；blocker 级阻断闭环 C |
| `etl.qlib.dump` | clean→qlib_bin | 每交易日 18:00 | `dump‖<date>` | 30min | 失败告警（研究层用旧 bin 不受影响） |
| `etl.snapshot.weekly` | 全量快照（P2 MinIO） | 周日 02:00 | `snap‖<iso_week>` | 120min | 重试 1 次 |

#### 5.1.7 事件库完整 DDL（采用调研 03 §3.1 原设计；DuckDB/PostgreSQL 均可）

```sql
-- ① 原始新闻/公告层：只追加、不修改
CREATE TABLE news_raw (
  id            BIGINT PRIMARY KEY,
  source        TEXT NOT NULL,        -- cls / em / sina / cninfo / tushare ...
  url           TEXT UNIQUE,
  title         TEXT,
  content       TEXT,                 -- 正文或公告 Markdown 全文
  publish_time  TIMESTAMP,            -- 信源发布时间（事件时间以它为准）
  fetch_time    TIMESTAMP,
  author        TEXT,
  category_raw  TEXT,                 -- 信源原始分类/公告类目
  raw_json      JSON
);

-- ② 去重层：唯一新闻簇（canonical）
CREATE TABLE news_cluster (
  cluster_id    BIGINT PRIMARY KEY,
  canonical_news_id BIGINT REFERENCES news_raw(id),  -- 最早/最权威的一条
  dedup_method  TEXT,                 -- exact_hash / simhash / vector
  simhash       BIGINT,
  title_norm    TEXT,
  members       JSON,                 -- 该簇全部 news_raw.id
  first_publish_time TIMESTAMP
);

-- ③ 向量层（供 RAG 与向量去重；托管于 LanceDB / sqlite-vec / Qdrant 单机模式）
CREATE TABLE news_embedding (
  news_id BIGINT, model TEXT, vector BLOB, PRIMARY KEY(news_id, model)
);

-- ④ 事件层：LLM 结构化抽取产出（五要素契约 §10.2）
CREATE TABLE event (
  id            BIGINT PRIMARY KEY,
  event_type    TEXT NOT NULL,        -- 枚举见 §10.2
  sub_type      TEXT,
  occur_time    TIMESTAMP,
  publish_time    TIMESTAMP,
  title         TEXT,
  summary       TEXT,                 -- 5W+金额 结构化摘要
  direction     TEXT,                 -- positive / negative / neutral
  confidence    REAL, importance REAL,
  horizon       TEXT,                 -- intraday / days / weeks
  evidence_news_ids JSON,
  extract_model TEXT,
  status        TEXT                  -- new / validated / consumed / invalidated
);

-- ⑤ 事件-实体关联（多对多带权重与方向）
CREATE TABLE event_entity (
  event_id BIGINT REFERENCES event(id),
  entity_type TEXT,                   -- stock / industry / index / macro / commodity
  entity_code TEXT,                   -- 600519.SH / SW.801080 / ^GSPC / DGS10
  entity_name TEXT,
  role        TEXT,                   -- target / peer / supply_chain / competitor
  direction   TEXT,
  weight      REAL,
  PRIMARY KEY (event_id, entity_type, entity_code)
);

-- ⑥ 持仓快照（事件↔持仓 JOIN 的三张表）
CREATE TABLE position (
  date DATE, account TEXT, code TEXT, name TEXT, qty DOUBLE, weight DOUBLE,
  PRIMARY KEY(date, account, code)
);
CREATE TABLE watchlist (code TEXT PRIMARY KEY, name TEXT, tags JSON);

-- ⑦ 事后复盘（事件信号质量闭环）
CREATE TABLE event_feedback (
  event_id BIGINT PRIMARY KEY,
  ret_1d DOUBLE, ret_5d DOUBLE, ret_20d DOUBLE,
  hit    BOOLEAN,
  note   TEXT
);
```

#### 5.1.8 对账、回填与重放细则（etl/ 内实现要点）

| 环节 | 细则 |
|---|---|
| 增量水位 | `watermark` 表推进前固定**回看 5 个交易日重抓覆盖**，吸收上游追溯修订（分红实施/财报更正/复权因子变化） |
| 多源仲裁 | 主源 baostock；\|Δclose\|/close>0.5% 或 adj_factor 不一致 → `dq_issue` 人工复核；**不静默覆盖**，修复走显式补丁记录 |
| 除权回填 | 除权事件发生 → 回填复权因子 → **重算该股全历史复权价（查询层动态）**；后复权用于收益计算、前复权用于展示 |
| 停牌 | 显式写 `is_suspended` 行（价格沿用最后成交价、volume=0）；禁止"跳过日期造成隐形合并" |
| 退市 | 打 `delist_date`，**永不删除**；退市整理期在 universe_snapshot 标 suspend/风险段 |
| 新股 | 上市日回拉全历史；IPO<60 日打 `is_new` |
| 重放 | raw 层保留原始响应 + ETL 代码 git 版本化；任何 schema 变更可全量重建 clean 层（`etl rebuild --from raw`） |
| schema 演进 | Parquet schema 演进向后兼容；`data/schema/migrations/` 顺序迁移脚本 + `schema_version` 记录在 manifest.json |

#### 5.1.9 可见性时间契约（防前视核心，v2 新增，审计 A01）

> 历史 PIT 表**补不出不存在的历史版本**：现在抓到的修订值不能放回早期披露日期使用；新增 `fetched_at` 或 `_next` 本身无法修复该问题。

1. **双可见性口径**：分别定义并存储
   - `market_public_time`（**市场已公开时间**）：来源文档的原始披露时间；
   - `system_acquired_time`（**本系统实际获得时间**）：首次采集/入库时间。
   历史研究与实时重放可使用不同口径，但**每处查询必须记录所用口径**（写入 `spec.yaml`、manifest 与 MLflow params）。
2. **来源证据字段**：每条 PIT 记录保存来源文档编号、内容哈希、原始披露时间、修订披露时间、首次采集时间、版本、时区。
3. **无披露时刻规则**：只有日期、没有披露时刻时，采用明确的**保守下一交易时段**规则（自下一交易时段起方视为可见），**禁止默认当日开盘可用**；同日多版本、收盘后公告、披露时间未知三种情形均须有确定查询结果。
4. **`historical_pit_unverified` 标记**：无法恢复原始版本的字段一律标记 `historical_pit_unverified`，并**阻断进入声称无前视的历史研究**（查询层拒绝加载或强制降级声明）；MVP 可暂不使用财务因子。
5. **验收（待执行，不预填通过）**：构造"先披露、后更正"两版本案例，更正前查询不出现更正值；同日多版本/收盘后公告/时间未知均有确定结果。

#### 5.1.10 数据底座与采集策略（v2 改写，A05 / 审计 §11）

> 由"从零采集全市场多年日线"改为：**公开数据包初始化 + 定期同步 + 质量验收 + 按需补缺**。

1. **历史底座（investment_data 公开 Qlib 数据包）**：
   - 固定明确的 **release tag**：同一 Release 下载 `qlib_bin.tar.gz` + `qlib_bin.manifest.json`，记录 URL、asset ID、发布时间、下载时间、大小及 SHA-256；
   - 使用**审阅过且固定 commit 的 `qlib/validate_archive.py`**（`--expected-tag` / `--require-publishable`）校验，脚本及依赖纳入版本记录；
   - 解压到**独立快照目录**（不直接覆盖现有工作数据），核查成员路径、日历、股票池与目标日期、通过质量验收后**原子切换**活动快照；
   - Qlib 直接读取该快照；Parquet/DuckDB 只保存补充数据、审计索引与统一查询视图，避免重复格式转换。
2. **定期同步**：每日先检查是否有新 Release；相同 digest 不重复下载；有新包则验证后切换。暂按**全包快照**处理，不假定"只追加一天"能吸收上游历史修订。Release 延迟或失败时继续保留旧快照并标明截至日期。
3. **质量验收（放行前必查，G1 执行项，本次未执行）**：起止日期与覆盖、最新日期（日历末日不得掩盖个股陈旧行情）、历史与退市股、复权及单位、多源口径抽样对照、指数成分历史、缺失与停牌区分、财务 PIT 与事件缺口、修订与重放差异报告（明细见审计 §11.3）。
4. **补充层（BaoStock/AKShare）**：仅补字段与**抽样对账**，放独立层，明确优先级、单位和复权口径，**不静默拼接进第三方二进制包**；§5.1.5 映射表按此定位使用。
5. **按需补缺**：未复权价格、交易状态、行业成分历史、财务披露及事件逐项建缺口清单后再决定补充来源；Tushare 等付费源只在公开渠道无法满足**已确认**需求时评估（不预定购买档位，档位覆盖能力待核实）。
6. **边界**：数据包归档校验只能证明发布完整性，不能证明每只股票每个字段正确；Qlib 标准化数值不得直接当作券商报价、股数或金额；公开快照不能还原归档开始前每一天的财务原始披露版本（该缺口按 §5.1.9 标记处理）。来源与许可边界（Apache-2.0 代码许可 ≠ 上游数据再分发授权，数据条款待核实）见审计 §11.1。

#### 5.1.11 存储并发与备份边界（v2 新增，审计 A09）

1. **单写入者 + 不可变快照**：ETL 写入由**单一写入者**执行，产出**不可变 Parquet 快照 + manifest**（含 `snapshot_id`、`schema_version`、校验和）；**读任务绑定 `snapshot_id`**，不会读到半发布数据；失败重跑不重复入库（幂等键保证）。
2. **SQLite 只存任务元数据**（水位/tasks/因子注册表）；DuckDB 为只读查询视图；仅在实测证明单写入者不足后，才考虑多进程写数据库升级（审计 S7：DuckDB 嵌入式并发边界）。
3. **备份从首期开始**：**独立介质或异地加密备份**，目标 **RPO 24 小时 / RTO 1 天（以实测确认）**；不以同机 MinIO/Time Machine 单副本替代灾难恢复（同机快照不能抵御整机或磁盘损坏）。
4. **验收（待执行）**：读任务不读到半发布数据；中断重放无重复事实；从独立备份恢复原始数据、元数据、配置与报告至少成功一次。

### 5.2 研究层（factors/ + 外部框架）

**职责**：因子定义、单因子体检、数据集构建、模型训练/调参。复用 qlib Handler/Dataset/表达式引擎 + alphalens-reloaded + quantstats + Optuna（调研 02 §7.3/§8.1）。

#### 5.2.1 因子定义接口（自研注册表，落到 qlib 表达式或 Python）

```yaml
# factors/registry/<factor_id>.yaml —— 因子规格（纯推理 Agent 产出 spec.json → coding agent 落码）
factor_id: F0001.mom20.v1          # ID/版本规则见 §12.1
family: momentum
expression: "Ref($close,-20)/$close-1"        # 表达式因子（qlib ops 50+ 算子）
# 或 python_impl: factors/impl/F0001_mom20.py  # 代码因子（实现 qlib DatasetH 兼容接口）
universe_filter: "universe_snapshot, exclude ST/次新(上市<60日)/停牌/涨跌停（仅用决策时已知条件；被过滤样本保留记录，禁做事后删除，A03）"
neutralize: [industry_SW2021, log_mktcap]      # 可选中性化（DataHandlerLP Processor 链）
hypothesis_ref: papers/2406.xxxxx/hypotheses.json#h2   # 假设血缘
created_by: {agent: research-agent, model: hermes-4-14b}
```

#### 5.2.2 研究层接口

| 接口 | 实现 | 说明 |
|---|---|---|
| `build_dataset(factor_ids, segments)` | qlib `DataHandlerLP` + `DatasetH` | 复制 Alpha158 handler 配置改 features 表达式（调研 02 §7.2） |
| `factor_tear_sheet(factor_id)` | alphalens-reloaded | IC/RankIC/分层/换手/衰减；输入前按**决策时已知条件**过滤不可交易样本（过滤清单随产物保存，禁止事后删除美化，A03）、用后复权收益、年化 244（调研 02 §4.2） |
| `stability_report(factor_id)` | 自研 ~50 行 | 滚动 IC/RankIC 序列、IC_IR、分段一致性（调研 02 §4.1 唯一自研点） |
| `admission_check(factor_id)` | 自研准入卡 | 阈值判定（§12.3 入库门槛），输出 pass/fail + evidence |
| `train_model(cfg)` | qlib `Model.fit/predict` | LightGBM/PyTorch 均可包装；Optuna 包住训练+回测目标函数 |
| `rolling_tasks(segments)` | qlib `RollingGen` | 滚动训练/验证/测试任务生成（walk-forward，调研 02 §4.1） |
| `tearsheet(equity, bench)` | quantstats | 组合绩效报告（Sharpe/回撤/月度热力图） |

### 5.3 回测层（backtest/）

**职责**：策略实现 + 撮合验证 + 绩效产物。复用 qlib Exchange/Executor + TopkDropoutStrategy 等策略族（调研 02 §7.5）。

#### 5.3.1 标准回测配置（YAML = MLflow params 的来源）

```yaml
# backtest/configs/topk_dropout_default.yaml
strategy: TopkDropoutStrategy        # topk=50, n_drop=5（公募指增强经典范式）
period: {start: 2018-01-01, end: 2026-09-30}
benchmark: "000300.SH"               # 或 000905.SH
exec: {deal_price_buy: "$open", deal_price_sell: "$close", trade_unit: 100}
exchange:
  limit_threshold: "board_aware"     # 主板10%/ST5%/创业板科创板20%/北交所30%（表达式按 universe_snapshot.board）
  open_cost: 0.0002                  # 佣金约 0.01–0.03%
  close_cost: 0.0007                 # 印花税 0.05% + 佣金
  min_cost: 5.0                      # 最低 5 元
  impact_cost: 0.001                 # 滑点 0.1%
  volume_threshold: 0.05             # 单日成交占比上限（容量约束）
label: "Ref($close,-2)/Ref($close,-1)-1"   # 仅作预测代理（= T+1 收盘→T+2 收盘收益，审计 S1）；
                                          # 必须声明与最终执行收益的差异，禁止标"与执行假设一致"（A02，见 §5.3.3）
pit: true                            # 财务数据强制走 PIT 通道（口径见 §5.1.9）
```

#### 5.3.2 回测执行接口与产物

```
backtest run --config <yaml> --experiment-id <id>
  → 1 回测 = 1 MLflow run（§12.2）
  → experiments/<id>/results/{metrics.json, equity.parquet, positions.parquet, report.md}
metrics.json 标准字段（准入与对比的唯一口径）：
  {sharpe, ann_ret, max_dd, calmar, turnover, excess_ret, ir,
   ic_mean, rank_ic_mean, ic_ir, hit_rate,
   stability_score, oos_sharpe, oos_ratio, regime_table{bull,bear,shock,range}}
```

- **两段式验证**（调研 02 §6）：入围因子/策略先向量化粗筛（Alphalens+qlib 信号分析，分钟级），再 qlib Exchange 精撮合；需要严格 T+1/事件撮合复核时用 rqalpha（**隔离进程/目录**，见 §16.3）。
- **验证命令**（Agent Teams verify 契约）：`pytest factors/impl/`（单测）→ `backtest run --quick`（单因子 IC 快测）→ `backtest run --full`（全量）。
- 分市场阶段测试：自研 ~30 行日期切分脚本循环调用 quantstats（牛/熊/震荡/流动性危机四段，分段表进 metrics.json `regime_table`）。

#### 5.3.3 统一执行契约与四分离记录（v2 新增，审计 A02 / A03）

**统一执行契约（A02）**：每个回测/策略 spec 必须显式定义——信号截至时间、最早下单时间、买卖价格（如买 `$open`/卖 `$close`）、持有期、交易日历、可卖数量（T+1 等）、费用与部分成交规则。标签（如 Qlib 默认 `Ref($close,-2)/Ref($close,-1)-1` = T+1 收盘→T+2 收盘）只能作为**预测代理**，必须声明其与最终执行收益衡量的交易区间差异，**禁止标"与执行假设一致"**。

**四分离记录（A03）**：**信号股票池 / 下单候选 / 成交结果 / 持仓**四类分开保存，禁止混用：

1. **信号股票池**：只用**决策时已知条件**生成（当日 `universe_snapshot` 快照），绝不用当前股票列表回溯；
2. **下单候选**：由信号按执行契约生成，与股票池分表保存（含未入选原因）；
3. **成交结果**：**逐单检查**方向、可成交量（容量约束/部分成交）、停牌状态及对应**历史交易制度**（各期涨跌停档位）；
4. **持仓**：旧仓当日未成交（如跌停卖不出）**继续持有并估值**，记录未成交原因；**禁止事后删除未成交样本**（事后剔除涨跌停/不可成交样本即选择偏差）；日线无法判定的盘中成交采用**公开说明的保守假设**。

**验收（待执行，不预填通过）**：①小型手工行情逐笔计算信号、持仓、费用与净值，与引擎输出对齐，覆盖收盘信号、跨节假日与不可当日卖出持仓；②买入失败、卖出失败、部分成交、停牌及恢复交易案例均守恒，现金、数量、费用与净值可对账。

### 5.4 Agent 层（agents/）与中间结果约定

> 详细编排设计见 §8；本节只定义"产物文件契约"（调研 04 §3.4）。

```
experiments/<experiment_id>/          # 每实验独立目录（或 git worktree exp/<id>）
├── spec.yaml                         # 实验规格：假设、因子参数、回测配置、inputs_hash
├── factors/                          # 本实验新增/修改的因子代码（diff 到主干）
├── logs/                             # 任务日志（stdout/stderr/agent transcript）
├── results/
│   ├── metrics.json                  # 标准化指标（§5.3.2 字段）
│   ├── equity.parquet / positions.parquet
│   └── report.md                     # 人读报告
└── review.md                         # Review Agent 结论（verdict + 结构化 findings）

runs/                                 # 日期驱动的日常流水线产物
├── data/<data_date>/manifest.json    # 数据落位清单 + 校验和 + dq_report.md
├── news/<date>/events.jsonl          # 事件流（append-only）
├── market/<date>/snapshot.md
├── portfolio/<date>/review.md        # 持仓逐只分析（闭环 C 输入）
├── plan/<date>/next_day_plan.md      # 次日预案（闭环 C 输出）
└── reviews/<ISO-week>/postmortem.md  # 周复盘
```

**命名规范**：`<type>_<yyyymmdd>_<short_hash>`；所有产物 JSON 内嵌 `schema_version`、`created_by`（agent 名+模型版本）、`idem_key`、`inputs_hash`、`created_at`——保证可追溯与向后兼容（模型名/版本写入而非硬编码，调研 04 §5.4）。

### 5.5 知识库（knowledge/）

**分层原则（调研 04 §3.4）：文件为事实来源（git 可审计），数据库为索引与状态，消息为触发信号。**

| 子库 | 内容 | 形态 | 写入者 | 读取者 |
|---|---|---|---|---|
| `knowledge/papers/<paper_id>/` | paper.pdf、summary.md、hypotheses.json（结构化因子假设） | 文件 + 向量索引 | Research Agent | Factor/Alpha Agent |
| `knowledge/postmortems/<ISO-week>/` | 周复盘、失败实验质询、改进清单 | Markdown | Review Agent | Alpha Agent（选题输入） |
| `knowledge/playbook.md` | **交易行为准则**（持续修订的行为纠偏清单） | Markdown | Review Agent | Portfolio/Market Agent（预案约束） |
| `knowledge/reviews/<run_id>/` | 实验评审记录（pass/needs_revision + findings） | Markdown | Review Agent | 全部 |
| `knowledge/events_kb/` | 事件复盘沉淀（event_feedback 高价值样本） | DuckDB 表 + MD | News/Review Agent | News Agent（few-shot 库） |
| 向量索引 | 上述全部文本的 embedding | **LanceDB**（或 sqlite-vec；调研 03 §3.1 注） | 索引任务 | LlamaIndex RAG |

**接口**：`mlflow-kb` MCP server 提供 `search_knowledge(query)`、`get_paper(id)`、`list_hypotheses(status)`；Research/Alpha/Review Agent 全部经 MCP 读写，禁止直接改他人负责的子库（写权限按 §8.1 角色表划分）。

**三大长期资产的落点**：个人因子库 = `factors/registry/` + DuckDB `factor_registry`（§12.1）；研究知识库 = `knowledge/papers/`+`reviews/`+向量索引；交易行为数据库 = `trading_journal` 表（§11.5）+ `playbook.md`。

### 5.6 任务调度层（orchestration/）

> 依据：调研 04 §3.2/§3.3/§3.6。DSH 为宿主（dsh-schedule 定时 + dsh-webhook 事件 + goal/Agent Teams/workflow 执行），自研持久化与防重复。

**组成**：
1. **触发**：macOS `launchd` 拉起 DSH schedule（07:30 / 16:15 / 周五 20:00）；`dsh-webhook` 接收事件（回测完成、重大新闻、失败告警）。
2. **任务状态机持久化**：SQLite `tasks` 表（WAL）：

```sql
CREATE TABLE tasks (
  task_id TEXT PRIMARY KEY, idem_key TEXT UNIQUE, type TEXT,   -- etl/factor_unittest/backtest/news_extract/...
  state TEXT,            -- PENDING/RUNNING/SUCCEEDED_EVAL/DONE/NEEDS_REVISION/FAILED_RETRY/
                         -- CANCELLED/SKIPPED_DUPLICATE/ESCALATED/HALTED（状态机图见 §8.3）
  attempt INT, round INT, assignee TEXT,     -- agent 名
  inputs_hash TEXT, spec_uri TEXT, result_uri TEXT,
  timeout_sec INT, started_at TIMESTAMP, finished_at TIMESTAMP
);
INSERT INTO tasks(idem_key, ...) ON CONFLICT DO NOTHING;   -- 幂等准入（§8.6 四层防线①）
```

3. **并发控制**：信号量限制"同时最多 2–3 个 backtest 进程"（Mac mini CPU 有限）；coding agent 会话**同样受限**——受任务契约 `timeout`/`retry_policy` 与并发上限约束（旧稿"coding 会话不受限"作废，A12，见 §8.11）。
4. **超时即失败**：Data 10min、单因子测试 30min、全量回测 4h、新闻批处理 30min——防止僵尸任务占位。
5. **人工闸门**：调仓建议、真实下单、删除实验等破坏性操作必须人工确认（调研 04 §3.2）。

### 5.7 展示层（apps/）

| 页面 | 内容 | 数据来源 | 优先级 |
|---|---|---|---|
| `apps/daily/` 复盘与预案阅读器 | 16:45 复盘报告 + 17:30 次日预案（逐只持仓卡片：状态/影响因素/多空风险/关键价位/情景） | `runs/plan/<date>/next_day_plan.md` | P0 |
| `apps/market/` 盘前快照 + 晨报 | 08:30 晨报（昨夜事件簇 + 持仓影响 + 外围市场） | `runs/market/` + `runs/news/` | P1 |
| `apps/factors/` 因子库浏览器 | 因子列表/版本血缘/IC 曲线/衰减监控/准入状态 | DuckDB `factor_registry` + MLflow | P1 |
| `apps/research/` 研究看板 | 实验对比、回测曲线、评审状态、运行中的 Agent 任务 | MLflow UI（嵌入）+ tasks.sqlite | P1 |
| MLflow UI | 实验对比原生界面 | `mlflow server`（§7.2） | P0 |

技术选型：**Streamlit**（Python 原生、与 pandas/DuckDB/MLflow 无缝）；报告同时输出 Markdown 静态文件（可直接在编辑器/终端阅读，展示层故障不影响产出）。被放弃的备选：Grafana（偏监控、做研究报告不顺手）、自研 React 前端（违反 §4.2 边界 5）。

---

## 6. Mac mini 本地部署方案（Apple Silicon）

> 依据：调研 01 §7（容量参考）、调研 02 §7.8（Apple Silicon 安装与性能）、调研 03 §6（本地小模型 vs 云端 API）、调研 04 §4（硬件档位 + 成本估算框架）。

### 6.1 硬件与内存规划（v2 按审计 A06 更正）

| Mac mini 配置 | 本地模型能力 | 系统内存分配建议 | 适用阶段 |
|---|---|---|---|
| 16GB / 512GB（已有设备起步） | Hermes-4-14B 量化档仅小上下文试验（权重见下） | OS 4G + DuckDB/qlib 4G + 推理 6G + 余量 2G | 仅 MVP 试验；回测并行受限 |
| 32GB / 1TB | Hermes-4-14B GGUF Q5_K_M 实用；MiMo-7B 级并行 | OS 4G + 研究进程 8G + 推理（Q5_K_M 权重 10.51GB + KV cache/运行时）12G+ + 缓存 | MVP + 第一阶段（**按实测决定是否购入**） |
| 64GB / 2TB | 14B 高精度/多实例，或更大模型（如 70B Q4 约 40GB+，待核实） | OS 4G + 研究 12G + 推理 + 缓存，按实测峰值分配 | 第二阶段（Review/Alpha 高质量本地推理，按需） |
| 128GB（远期） | 70B Q5/Q6 + 多实例并行 | 按需 | 完整系统 |

- 本地推理栈：**Ollama 或 llama.cpp（GGUF）起步**，追求吞吐换 MLX/vLLM（调研 04 §4.1）。**Hermes-4-14B GGUF 量化文件实际大小以具体发布为准：Q5_K_M 约 10.51GB**（审计 S2，bartowski 量化发布页；不是所有量化格式的统一内存承诺）。**旧稿"GGUF-Q4 4–6GB"写法作废**。采样建议 `temperature=0.6, top_p=0.95, top_k=20`（待核实）。
- **权重 ≠ 总内存**：运行内存 = 模型权重 + KV cache + 上下文长度开销 + 运行时开销；容量规划必须按**实际峰值**测算，不得把模型权重当作总内存。
- **实测记录模板**（每模型/每批次一行；连续运行验证不因资源不足中断——A06 验收项，待执行）：

  | 设备 | 模型版本（文件名+量化格式+哈希） | 上下文长度 | 任务耗时 | 内存峰值 | 交换内存（swap） |
  |---|---|---|---|---|---|

- **取消首期购买 32/64GB 设备的前置条件**：先在已有设备按上表实测定标；**训练、回测与模型推理错峰**安排（分时运行，避免叠加峰值），以实测数据决定是否升级硬件。进入 G2 前不得以回测收益作为购买硬件的依据（审计 §6）。
- CPU 注意：qlib 数据集构建吃多核（官方基准 1CPU 147s → 64CPU 8.8s），Mac mini 8–20 核收益明显；模型训练 LightGBM/PyTorch 有原生 arm64 加速、NN 可用 MPS（调研 02 §7.8）。

### 6.2 磁盘规划（1TB 机型为例）

| 路径 | 内容 | 容量估算 | 备份策略 |
|---|---|---|---|
| `~/quant-data/clean/` + `staging/` | Parquet 事实表 | 日线 30 年全 A **<2GB**；5 分钟 15 年 **10–30GB**；1 分钟 5 年数十 GB（调研 01 §7） | MinIO 快照（P2）+ 外接盘 |
| `~/quant-data/raw/` | 原始响应包（审计/重放证据） | 20–50GB（公告 PDF 控量下载：仅持仓股+重要类目，调研 03 §4.1） | 冷备 |
| `~/quant-data/qlib_bin/` | qlib 物化视图 | 日线 ≈2–3GB；分钟线视需求 | 可重建，不备份 |
| `state/mlruns/` + `experiments/` | 实验产物（equity/positions Parquet） | 5–20GB/年 | git + 异地同步 |
| `state/cache/` + DuckDB/SQLite | 缓存与状态 | 5–10GB | VACUUM INTO 定期备份 |
| Ollama 模型 | GGUF 权重 | 14B Q5 ≈10GB；70B Q4 ≈40GB | 不备份（可重下） |
| **合计（第一阶段）** | | **约 150–250GB**（1TB 充裕；分钟级冷数据放 MinIO 或外接盘） | |

### 6.3 本地模型 vs 云端 API 取舍

**结论（调研 03 §6.3 + 调研 04 §4.4）：混合架构——"全量便宜走本地、复杂推理走云端、coding 走 API 订阅套利"。**

| 工作负载 | 走本地（Hermes-4-14B / BERT） | 走云端（DeepSeek API 等） | 理由 |
|---|---|---|---|
| 全量新闻情绪/事件分类（每日 800 条级） | ✅ finance-sentiment-zh-base + 微调 BERT，每秒数百条，成本≈0 | ❌ 质量过剩不划算 | 调研 03 §6.2 |
| 短快讯事件抽取（结构化 JSON） | ✅ 14B-Instruct Q4 夜间批 2–4h，边际成本≈0 | 可选兜底 | 长公告复杂金额推理才见差距 |
| 长公告全文抽取/多实体归因 | ⚠️ 需分段抽取再合并 | ✅ 强项（8k tok 级抽样重要篇目） | 调研 03 §6.1 |
| 晨报/摘要生成 | 一般（14B 以上才顺） | ✅ 质量最好 | 调研 03 §6.1 |
| 论文精读/复盘质询 | ✅ 14B/70B（隐私友好） | ✅ DeepSeek-V4-Pro 兜底难题 | 调研 04 §2.1.1 |
| Factor/Backtest coding | ❌（不建议本地模型裸写代码，调研 04 §3.7） | ✅ DSH+DeepSeek API / Codex 订阅 | coding 质量优先 |
| 持仓/调仓相关推理 | 持仓数据不出本机优先 | 仅发脱敏摘要 | 合规/隐私 |

**持续改进闭环**：云端抽取出的高质量样本回流 LoRA 微调本地 7B/14B（FinGPT 配方），逐步压降云端调用量；`event_feedback` 命中率驱动模型/阈值迭代（调研 03 §6.3）。

### 6.4 成本与用量记录口径（v2 按审计 A06 重写）

> **口径原则**：不再给出笼统月费估算——旧稿"轻量 ¥150–600/月、重度 ¥800–1,500/月"等月费数字**作废**；模型与服务额度按**已有服务配置**，不自设月费上限与每日处理量上限（用户已准备 AI 模型）。费用按下列口径**分开记录**、按实际用量核算：

| 记录项 | 口径 |
|---|---|
| 输入 token | 每次调用记录（含实际模型标识） |
| 输出 token | 与输入分开记账（单价档不同） |
| 缓存 | 缓存命中/未命中 token 分开记账（价差可达数十倍） |
| 重试 | 重试次数与重试消耗单独归集，不与首轮混合 |
| 耗时/并发 | 任务耗时、并发峰值、限流/退避次数 |
| 电费 | 本地推理/回测批次耗电估算单列 |
| 已有订阅 | 订阅费与订阅内用量单列，不与 API 混算 |
| 数据源费用 | 只记录**已确认购买**项；免费源也统计下载、磁盘、备份与维护成本；付费源仅在公开渠道无法满足已确认需求时评估（不预定档位） |

- 若已有服务配置**硬额度**：调用前预留费用、完成后结算（A12，§8.11）；额度或服务限流触发时停止新增调用、保留检查点，确定性数字报告继续生成。
- 记账工具：`dsh-token-meter` + MLflow LLM tracing（§8.5；调研 04 §5.7 成本失控教训）。具体调用端点、上下文与账户额度沿用用户已准备配置，本文未验证这些账户的运行权限。

**云端 API 价格底账（2026-09-22 快照，本次审计未重核，待核实；仅供代入实测用量估算，不代表月费承诺）**：

| 服务 | 价格要点 |
|---|---|
| DeepSeek API | deepseek-flash 输入 $0.15/M（缓存未命中、off-peak）/ $0.003（缓存命中）/ 输出 $0.6/M；deepseek-v4-pro $0.66 / $0.022 / $1.98；高峰翻倍、off-peak 5 折；1M 上下文 |
| OpenAI Codex | 订阅：Plus $20 / Pro 5x $100 / Pro 20x $200（5 小时滚动窗口限额）；API：Luna $0.20/$1.20、Terra $2/$12、Sol $4/$20、Astra $10/$50（每 M 输入/输出） |
| Anthropic Claude Code | 订阅 Pro $20 / Max 5x $100 / Max 20x $200；API Sonnet 5 $2/$10、Opus 5 $5/$25 |
| Xiaomi MiMo | MiMo Code 需 Token Plan（首订 88 折）；Batch API 半价；V2.5 于 2026-10-21 下线需迁 V2.6 |
| 独立测算（Artificial Analysis v1.5） | Codex+GPT-6 Astra ≈ $7.47/任务；Codex+DeepSeek V4 Pro ≈ $0.24/任务 |

**用量登记基线（按工作负载登记实际用量，不折算月费）**：

| 用量项 | 登记口径 |
|---|---|
| News（本地 Hermes-4-14B，条数按实际） | 本地推理：耗时/内存峰值/电费；如走云端则记输入/输出 token |
| Research（论文精读） | 云端精读按次登记 token（输入/输出/缓存分开） |
| Factor/Backtest coding（实验 × 会话） | 每会话登记 token、重试、耗时；订阅路线与 API 路线分开归集 |
| Review/Alpha（长评审） | 按次登记；本地/云端分开 |
| MiMo Code（可选） | 已有订阅档位与订阅内用量单列 |
| 数据源 | 只登记已确认购买项（旧稿"tushare 2000 积分 ¥17/月"结论撤销——不预定购买档位，待核实） |

**三条用量纪律（调研 04 §4.3，v2 改写）**：①订阅 vs API 分流——白天交互走订阅、夜间无人值守批处理走 API（可断点重试、精确核算），两线分开记账；②off-peak 红利——批处理与 coding 循环可排到低价时段（按官方峰谷表排程，待核实）；③缓存纪律——系统提示+工具定义固定前缀以命中上下文缓存，结果缓存防重复实验。记账与限额执行按上文"记录项/硬额度预留-结算"口径，不自设月预算数字。

---

## 7. 基础设施建议（Docker / Python / 数据库 / 对象存储）

### 7.1 docker-compose 服务清单

> 原则：**能裸机（uv/venv + launchd）不容器化的**：Python 研究栈（qlib/LightGBM/MPS）裸机跑性能与调试最顺；**需要常驻服务或隔离的**进 Docker。arm64 镜像优先官方多架构镜像。**同一服务二选一**：要么容器、要么裸机，**禁止同一服务同时以容器和裸机重复启动**（避免端口/数据目录冲突；如 mlflow、ollama 二选一）。
>
> **命令分类标注（v2，A07）**：本文与 §15.3（第 2 批修订）中的命令/配置按三类标注；未经空环境执行验证一律不得标"已验证操作"：
>
> | 类别 | 含义 | 现状 |
> |---|---|---|
> | **架构示例** | 说明形态与拓扑，版本/参数未锁定，不可直接照抄运行 | 本节 docker-compose 清单与服务表 |
> | **待实现接口** | 依赖尚未交付的模块/配置（如 `data.etl`、完整项目文件、锁定版本号） | `etl run` 系列、`mlflow server` 参数、安装/初始化命令 |
> | **已验证操作** | 在目标环境从空目录执行通过并留有日志（含依赖锁定、所需文件、版本、预期输出、失败处理与恢复步骤） | **无（待 G1 空环境验证；当前不得标记 A07 通过）** |

```yaml
# infra/docker-compose.yml —— 【架构示例·待验证】服务清单；版本须在落地时锁定具体 release，
# 禁止占位版本（如旧稿 minio/minio:RELEASE.2026-xx 已删除）
services:
  postgres:        # [P1] 事件库/事实表并发写启用；MVP 阶段用 DuckDB+SQLite 即可
    image: postgres:16-alpine          # arm64 官方多架构
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    environment: {POSTGRES_DB: quantlab}
  mlflow:          # [P0] 实验追踪（backend SQLite + artifacts 本地目录）
    image: ghcr.io/mlflow/mlflow:v3.16   # Apache-2.0；与裸机 `mlflow server` 二选一，禁止重复启动
    command: >
      mlflow server --host 0.0.0.0 --port 5000
      --backend-store-uri sqlite:////state/mlflow.db
      --artifacts-uri /state/mlartifacts
    volumes: ["./state:/state"]
  # minio：已从清单移除——同机 MinIO 不能替代灾难恢复（备份走独立介质/异地加密，§5.1.11）；
  #        确有多机共享对象存储需求时再引入并锁定具体 release（禁止占位版本）
  qdrant:          # [P1 可选] 向量库（单机模式）；默认用 LanceDB/sqlite-vec 嵌入式则不启
    image: qdrant/qdrant:latest
    ports: ["6333:6333"]
  ollama:          # [P0] 本地推理（Hermes-4-14B GGUF）；容器或裸机 llama.cpp/Ollama 二选一
    image: ollama/ollama:latest
    ports: ["11434:11434"]
    volumes: ["ollamamodels:/root/.ollama"]
    # Apple Silicon：容器内 Metal 加速受限，追求吞吐建议裸机 Ollama/MLX（见 §6.1）；禁止两者同时启动
  streamlit:       # [P1] 展示层
    build: ../apps    # Python 3.12 + Streamlit
    ports: ["8501:8501"]
volumes: {pgdata:, ollamamodels:}
```

| 服务 | 版本线 | 优先级 | 用途 | 替代/备注 |
|---|---|---|---|---|
| PostgreSQL | 16-alpine（arm64 官方） | P1 | 事实表并发写 + `ON CONFLICT` upsert + jsonb 事件库；可加 TimescaleDB 时序分区 | MVP 用 DuckDB/SQLite；出现并发写瓶颈再启用（调研 01 §7） |
| DuckDB | 1.x（pip，arm64 预编译） | P0 | 研究查询引擎（直接查 Parquet） | 嵌入式，无容器 |
| SQLite | 3.x（WAL 模式） | P0 | 元数据/水位/tasks/因子注册表 | 定期 `VACUUM INTO` 备份 |
| MinIO | 待锁定具体 release（禁止占位版本） | P2 | 仅多机共享对象存储场景引入 | 首期不部署；同机 MinIO 不作灾难恢复，备份见 §5.1.11 |
| MLflow | v3.16.x（Apache-2.0） | P0 | 实验追踪 + LLM tracing | backend SQLite 起步 |
| 向量库 | LanceDB 或 sqlite-vec（嵌入式） | P1 | 知识库 RAG + 语义去重 | Qdrant 单机模式备选（调研 03 §3.1 注） |
| Ollama/llama.cpp | 待锁定具体版本 | P0 | 本地 Hermes-4-14B / 金融 BERT | 裸机优先（Metal）；容器/裸机二选一，禁止重复启动 |
| Streamlit | latest | P1 | 展示层 | — |
| Redis Streams / NATS | — | P2 | 事件流升级方案 | **不上 Kafka/RabbitMQ**（调研 04 §3.4） |

### 7.2 Python 环境与工具链

> 本节命令均为**待实现接口/架构示例**（"已验证操作"为空，见 §7.1 命令分类表）；A07 空环境验证前不得当作运行手册直接执行。

- **Python 3.11 或 3.12**（qlib 支持 3.8–3.12；建议 3.11 兼容 LightGBM/torch 生态最稳，调研 02 §7.8）。
- **包管理用 `uv`**（锁文件 `uv.lock` 提交进仓库；实验环境可复现，调研 04 §3.6④），研究环境用 conda 亦可（qlib README 建议 conda）。
- **Apple Silicon 必装**：`xcode-select --install`（qlib Cython 扩展编译）+ **`brew install libomp`**（LightGBM/OpenMP；调研 02 §7.8 官方 M1 提示——缺 OpenMP 会导致 LightGBM wheel 构建失败，这是 Apple Silicon 最大的坑，一行解决）。
- qlib 安装：`pip install pyqlib`；若源码安装注意 Cython 扩展编译。Qlib 上游工作流入口为 `qrun`，本文命令须按**锁定版本**逐一验证后才可归入"已验证操作"（审计 S3，待核实）。落地后以 Alpha158+LightGBM `qrun` 全流程计时做本机基准测试（官方无 Apple Silicon 公开基准，需自测定标，调研 02 §7.8）。
- 定时任务：macOS **launchd**（比 cron 可靠，支持漏跑补执行）拉起 DSH schedule / pipeline CLI；调研 01 §8.1 同此建议。

### 7.3 版本与配置管理纪律

1. 所有服务版本写死在 `infra/docker-compose.yml` 与 `uv.lock`，**模型名/版本写入配置与 MLflow tags 而非硬编码**（DeepSeek/MiMo 迭代极快，MiMo V2.5 即将下线是前车之鉴，调研 04 §5.4）。
2. 配置分层：`config/defaults.yaml`（git）< `config/local.yaml`（不入 git，含 API key）< 环境变量；密钥一律不入库。
3. 备份：Time Machine + 异地同步 `experiments/`、`knowledge/`、`state/`（单机单点风险，调研 04 §5.5）；SQLite 每周 `VACUUM INTO` 快照。

---

## 8. Agent 协作与任务编排方案

> 依据：调研 04 §3（九类角色/Orchestrator/状态机/中间结果/MLflow/防重复/职责切分）+ §2.3（MCP）。本章是调研 04 结论在本系统中的落地设计。

### 8.1 Orchestrator 设计

**宿主**：DSH（DeepSeek Harness，232,442★，MIT）——goal 长程循环、Agent Teams 任务 DAG+质量门、workflow 脚本、`dsh-schedule` 定时、`dsh-webhook` 事件、`dsh-mcp-client` 工具、`dsh-token-meter` 记账（调研 04 §2.1.2）。**不叠 LangGraph 第二层编排**（一个运行时 + 多个执行体原则）。

**Orchestrator 三条实现原则**：
1. **调度是确定性逻辑**：DAG 推进、幂等准入、超时、告警全用代码（workflow 脚本/tasks 表），LLM 只做异常处置与汇总（调研 04 §3.7）。
2. **双触发模型**：时间驱动为主（交易日历切分的 07:30 / 16:15 / 周五 20:00），事件驱动为辅（新论文入池、回测完成、回撤越阈、盘中重大新闻）。
3. **人工闸门**：调仓建议/真实下单/破坏性操作必须人工确认；Agent 只产出建议（调研 04 §5.2）。

**实验批处理用 Agent Teams quality gate**：captain（Orchestrator）建任务 DAG `实现 → verify(pytest+IC 快测) → 全量回测 → review`，quality gate 契约（acceptance criteria + verify commands）即回测验收标准；review 失败自动进入 repair 轮次（round+1，上限 3）——正是实验迭代需要的闭环（调研 04 §3.2）。

### 8.2 协作模式与任务契约（v2 改写，审计 §5.2 / A04 / A12）

**分工表（GPT 计划/研究规范/审核；DeepSeek 或 MiMo 执行）**：

| 环节 | 责任 | 交付与边界 |
|---|---|---|
| 计划与研究规范 | **GPT** | 明确目标、数据快照、允许数据分段、任务依赖、验收命令和成功条件 |
| 编码与运行 | **DeepSeek 或 MiMo** | 执行计划，提交代码 diff、日志、测试、指标及失败原因 |
| 数值与回测 | **确定性程序** | 计算收益、费用、指标；**模型不得编造或心算替代** |
| 审核与修订 | **GPT** | 对照固定验收标准检查证据；重大策略变更形成新实验版本 |
| 最终样本外 | **隔离验证任务** | 冻结方案后评估；**结果不得回流当前策略的自动修复循环**（A04） |

**任务契约（八字段，缺一不可）**：`task_id`、`plan_version`、`data_snapshot_id`、`code_commit`、`allowed_paths`、`acceptance_commands`、`timeout`、`retry_policy`。

- **每个任务指定一个主执行者**（DeepSeek 或 MiMo），提交物含代码 diff、日志、测试、指标及失败原因；
- **模型切换必须携带**：已完成步骤、失败证据、剩余工作——不从头重复试验；GPT/DeepSeek/MiMo 均记录实际模型标识；
- 可在一个运行时中实现两个模型角色，无需为每个角色独立部署服务（调研 04 §3.7）；具体调用端点、上下文与账户额度沿用用户已准备配置（未验证运行权限）。

#### 8.2.1 执行角色细分表（原"九类 Agent 职责表"，按 v2 分工解释）

| # | 角色 | 职责 | 首选执行体 | 模型档位 | 输入 → 输出 | 写权限 |
|---|---|---|---|---|---|---|
| 1 | **Orchestrator** | 调度全部、状态机推进、去重、告警、周报汇总 | DSH goal/Agent Teams/workflow | 编排小档（代码逻辑为主） | 触发器 → 任务 DAG → 各 Agent 产物 | orchestration/、tasks.sqlite |
| 2 | **Research** | arXiv/SSRN/研报精读、方法提炼、可实现性判断 | 纯推理（Hermes-4-14B 本地） | 14B/70B 本地 | PDF/URL → `papers/<id>/summary.md` + `hypotheses.json` | knowledge/papers/ |
| 3 | **Data** | 行情/财务/新闻采集、清洗、对账、质量校验 | **确定性 Python（无 LLM）** + coding agent 维护 | — | 数据源 → clean Parquet + dq 报告 | data/（代码）、~/quant-data/ |
| 4 | **Factor** | 因子代码生成/修改、单测、单因子测试（IC/IR/换手） | **DSH / Codex**（coding agent） | 云端大模型档 | 因子假设 JSON → `factors/impl/<name>.py` + 单测 + IC 报告 | factors/、experiments/<id>/ |
| 5 | **Backtest** | 组合回测执行、参数扫描、失败修复、结果整理 | 确定性引擎 + coding agent 修配置/bug | — | 策略配置 → MLflow run + `results/metrics.json` | backtest/configs/、experiments/<id>/ |
| 6 | **Alpha** | 假设提出、挖掘规划、实验去重、过拟合质控 | 纯推理（Hermes-4 / DeepSeek-V4-Pro） | 中高档 | 因子库状态+研究笔记 → 下批实验计划（带优先级） | experiments/（spec.yaml） |
| 7 | **News** | 新闻/公告/政策抓取编排、去重（首期 ID+哈希+修订链）、事件抽取、情绪打分 | 纯推理（Hermes-4 本地）+ 本地 BERT | 14B 本地 | 原文 → `news/<date>/events.jsonl` + 事件库 | runs/news/、事件库 |
| 8 | **Market** | 盘面快照、风格/行业轮动、情绪面、盘前晨报 | 纯推理 + 确定性指标 | 14B 本地 | 行情+events → `market/<date>/snapshot.md` | runs/market/ |
| 9 | **Portfolio** | 持仓分析、风险暴露、情景预案（不自动下单） | 确定性计算 + 纯推理解读 | 中档 | 持仓+信号+事件 → `portfolio/<date>/review.md` + 次日预案 | runs/portfolio/、runs/plan/ |
| 10 | **Review** | 交易/实验复盘、归因、失败质询、改进清单 | 纯推理（Hermes-4-70B 或 DeepSeek-V4-Pro） | 高档（批判性） | 交易日志+回测记录 → `postmortem.md` + review.md + playbook 修订 | knowledge/postmortems/、playbook.md |

> 注：表列 10 行（Orchestrator + 9 类工作 Agent），对应调研 04 §3.1 的九类角色 + Orchestrator。TradingAgents 的"多空辩论"结构用于 Portfolio 的情景分析（正反方各给证据，不给结论）。
>
> **v2 解释（按 §8.2 分工表）**：表中"coding agent"执行体为 **DeepSeek 或 MiMo**；计划/研究规范/审核类产出由 **GPT** 负责；数值与回测一律由**确定性程序**计算（模型不得编造或心算替代）；最终样本外评估由**隔离验证任务**执行，结果不回流自动修复循环。表中"模型档位"列为旧稿表述，本次审计未逐项重核，**待核实**。

**执行体资源分配**（调研 04 §3.7 混合模式）：`纯推理产出 spec.json → coding agent 按 spec 写代码 → 确定性引擎跑回测 → 纯推理评审结果`。纯推理模型不接触文件系统（减小风险面），coding agent 每次调用都有 spec 验收标准。

### 8.3 任务状态机

> 依据：调研 04 §3.3（状态机原文设计，本系统直接采用）。

```
                 ┌────────────────────────────────────────────┐
                 │              (事件/定时触发)                  │
                 ▼                                            │
 [PENDING] ──依赖满足/调度──▶ [RUNNING] ──正常完成──▶ [SUCCEEDED_EVAL] ──验收通过──▶ [DONE]
     │                        │   │                        │  │
     │                        │   └─失败/超时─▶ [FAILED_RETRY]─┤（重试≤N，指数退避）
     │                        │                              └─验收不过─▶ [NEEDS_REVISION]
     │                        │                                        │（回到 RUNNING，round+1）
     ├─手动/策略取消─▶ [CANCELLED]                                     └─round>上限─▶ [ESCALATED]（转人工）
     └─同幂等键已有 DONE─▶ [SKIPPED_DUPLICATE]

 [DONE]/[CANCELLED]/[ESCALATED]/[SKIPPED_DUPLICATE] 为终态，结果不可变（immutable terminal result）。
 [HALTED]（人工暂停）可从 RUNNING/PENDING 进入，仅允许显式 resume 退出。
```

**关键规则**：
1. **幂等键是状态机准入条件**：进 PENDING 前查同键任务，DONE→`SKIPPED_DUPLICATE`，RUNNING→合并不重复派发。
2. **验收挂在 SUCCEEDED_EVAL 之前**：回测任务的验收不是"命令退出码 0"，而是"metrics 产出 + 阈值检查"（Sharpe/回撤/换手在合理区间、无 look-ahead 报警）。直接借用 DSH Agent Teams 的 acceptance criteria / verify commands 契约。
3. **NEEDS_REVISION 必须带结构化 findings**（id/severity/problem/requiredFix），否则不允许流转——防止评审 Agent 含糊放行。
4. **超时即失败**：Data 10min、单因子测试 30min、全量回测 4h、新闻批处理 30min。
5. 持久化：SQLite `tasks` 表（§5.6 DDL）；用 DSH Agent Teams 时其任务图/attempt 机制即此状态机的现成实现。

### 8.4 中间结果存储：experiments/<id>/ 约定

> 依据：调研 04 §3.4 + 本设计 §5.4。**文件为事实来源（git 可审计），数据库为索引与状态，消息为触发信号。**

- 每实验独立目录（或 git worktree）：`spec.yaml / factors/ / logs/ / results/{metrics.json,equity.parquet,positions.parquet,report.md} / review.md`。
- 命名 `<type>_<yyyymmdd>_<short_hash>`；产物 JSON 内嵌 `schema_version/created_by/idem_key/inputs_hash`。
- 长文本（论文/复盘）用 Markdown + 向量库（LanceDB/sqlite-vec），Agent 可 grep/read 也可 RAG；大结果（回测明细）进 Parquet 不进 DB。
- 消息/事件：起步用 **SQLite tasks 表即队列 + `runs/**/events.jsonl`**（天然幂等）；升级才考虑 Redis Streams/NATS；**不上 Kafka/RabbitMQ**（调研 04 §3.4）。

### 8.5 MLflow 实验追踪接入

> 依据：调研 04 §3.5 + 调研 02 §5.2。映射规则：**一个回测实验 = 一个 MLflow run**。

| MLflow 元素 | 内容 |
|---|---|
| experiment | `factor/<factor_family>` 或 `strategy/<strategy_name>` |
| run 名 | = `experiment_id`（与 experiments/<id>/ 目录一致） |
| params | `spec.yaml` 全量展开（因子参数、回测区间、费率、基准）+ `code_git_sha` + `inputs_hash` |
| metrics | `metrics.json` 全量（Sharpe、年化、最大回撤、换手、IC/RankIC/IC_IR、命中率、oos_sharpe、regime_table） |
| artifacts | equity.parquet、positions.parquet、report.md、review.md |
| tags | `agent`（谁改的代码）、`model`（模型名+版本）、`idem_key`、`round`、`verdict`（pass/needs_revision） |

- **LLM 调用也记录**：MLflow 3.x 自带 GenAI tracing/eval（`mlflow.tracing`）或自建 `llm_calls` 表（prompt/completion/token 数）——复盘"哪个 Agent 烧钱"并对齐 §6.4 成本。
- **贯通**：Alpha Agent 选题前 `mlflow.search_runs()` 过滤"同 inputs_hash 已跑过"的实验（与 §8.6 联动）；Review Agent 复盘读 run 列表做周对比；qlib QlibRecorder 原生 MLflow 后端，`qrun` 产物直接进同一后端（调研 02 §7.6）。
- 纯推理 Agent 经 `mlflow-kb` MCP server 查询实验，不写 Python（可选）。

### 8.6 防重复劳动四层防线

> 依据：调研 04 §3.6（多 Agent 系统最容易翻车的地方）。

**① 任务幂等键（idem_key）——状态机准入**

```
idem_key = sha256( task_type ‖ normalized(inputs_hash) ‖ code_version_constraint )
```
- `task_type`：`factor_unittest` / `backtest` / `news_extract` / `etl` …；
- `normalized(inputs_hash)`：输入（因子 spec、回测配置、数据日期）**键排序 + 浮点归一化**后哈希——语义相同的任务必得相同键；
- `code_version_constraint`：数据类任务可复用旧代码版本结果；代码类任务必须绑 `git_sha`；
- 准入逻辑：`INSERT INTO tasks(idem_key,...) ON CONFLICT DO NOTHING`——冲突即不派发，配合 `SKIPPED_DUPLICATE` 状态。

**② 结果缓存**

- `state/cache/<idem_key>/` 存上次成功结果，命中直接返回 `result.json` 并标注 `cache_hit=true`；
- **失效条件**（写进 spec）：数据日期推进、代码 sha 变化、qlib 数据校验和变化、TTL（数据类 24h、代码类 7d）；
- LLM 语义缓存（embedding 相似度复用）**只用于 News/Research 摘要类，禁止用于任何数字计算**。

**③ 内容级去重**

- News：`event_key = hash(canonical(title)+source+date)`，抽取前查当日 events.jsonl；
- Research：`paper_id = arXiv id / DOI`，已入 `papers/` 即跳过；
- Alpha 实验计划：新假设与既有实验 embedding 查重——**同一因子换个名字反复试是过拟合温床，Review Agent 有责任打回**；
- 跨 Agent 广播"谁正在做什么"：Orchestrator 维护 `claim(task_id, agent)` 租约（带 TTL），避免两个 Agent 同时改同一文件。

**④ git worktree 隔离（v2：只隔离目录，需进程级权限控制补齐——A12）**

- 每实验 `git worktree add experiments/<id> -b exp/<id>`：coding agent 只在自己 worktree 写文件，**主干永远干净**；合并回主干必须 Review 通过 + 人工确认；
- **git worktree 只隔离目录**，不能限制进程访问密钥、封存测试集或原始数据——必须以**进程级权限控制**补齐（限制工具/可写目录/网络/并发/运行时长，见 §8.11）；
- 重活（回测）独立进程/容器跑，工作目录 = 实验目录，环境锁版本（`uv.lock` 提交进 spec）保证可复现；
- 并发上限：同时最多 2–3 个 backtest 进程（Mac mini CPU 信号量）；coding agent 会话**同样受限**（受任务契约 `timeout`/`retry_policy` 与并发上限约束；旧稿"coding 会话不受限"作废）。

### 8.7 MCP 统一工具层

> 依据：调研 04 §2.3（结论：非常可行且应当作为统一工具层；金融垂直 MCP 生态不成熟，自建 3 个私有 server）。

| MCP server | 工具面（示例） | 实现 | 使用者 |
|---|---|---|---|
| **`ashare-data`** | `query_bars(code, start, end, freq)`、`query_financial_pit(code, as_of)`、`get_universe(date)`、`get_events(date, codes)` | python-sdk + DuckDB/Parquet 封装 akshare/tushare/qlib 数据层 | News/Market/Portfolio/Research Agent |
| **`backtest`** | `run_backtest(config_hash) -> metrics`、`get_run_status(run_id)`、`factor_ic(experiment_id)` | python-sdk + qlib CLI 封装（**幂等工具即 §8.6① 的执行入口**） | Factor/Backtest/Alpha Agent |
| **`mlflow-kb`** | `search_experiments(filters)`、`search_knowledge(query)`、`get_paper(id)`、`list_hypotheses(status)` | python-sdk + MLflow client + LlamaIndex | Research/Alpha/Review Agent |

原则：
- 3 个 server 均**私有部署、不公开、不经 MCP Registry**（调研 04 §2.3 建议）；工具只写一遍，DSH/Codex/Claude Code 全支持（DSH 走 `dsh-mcp-client`，Codex 走 `config.toml [mcp_servers]`）。
- 被放弃的备选：lsj210001/qlib-mcp、finance-mcp 等个人项目（2★/0★、无 License、停更）——**思路可抄（工具面设计），代码不直接依赖**（调研 04 §2.3）。
- 所有 MCP 工具强制幂等语义：写操作带 `idem_key` 参数，读操作无副作用——这是 Agent 与量化栈交互的审计边界。

### 8.8 自动改代码 + 跑回测的夜间循环（P1）

> 依据：调研 04 §3.7 循环原文。用 DSH Agent Teams quality gate 或 `codex exec` 脚本化：

```
loop (round = 1..MAX=3):
  1. coding agent（DeepSeek/MiMo）在 exp/<id> worktree 内实现 spec（含单测）
  2. verify: pytest 单测 + 单因子 IC 快测必须过
  3. 全量回测 → metrics.json（1 回测 = 1 MLflow run；数值一律确定性程序计算）
  4. Review（GPT）按验收标准评审（过拟合检查、换手/费率现实性、与 spec 一致性）
     ├─ verdict=pass          → 合并候选，登记 MLflow，通知人工
     ├─ verdict=needs_revision → 带 findings 回到 1（round+1）
     └─ round > MAX            → ESCALATED，转人工
```

**循环约束（v2，A04/A12）**：本循环只覆盖训练/开发验证集上的实验迭代；**最终封存样本外**由隔离验证任务在代码/参数/数据版本冻结后执行，**结果不回流本循环**；**修复代码错误与修改研究假设分开登记**（fix_log vs hypothesis_log），**不得以提高最终测试收益为修复目标**；实现错误可在开发集修复，策略经济表现不佳记为失败，不循环"修到通过"；每轮遵守 §8.11 执行边界。

被放弃的备选：RD-Agent 内建闭环直接跑（P1 先用自定质量门掌握验收口径，P2 再把 RD-Agent 接入为挖掘执行体，见 §9.3）。

### 8.9 Prompt 模板与契约管理（agents/prompts/）

所有 Agent prompt 入 git、带 `schema_version`；**系统提示+工具定义固定前缀**（命中 DeepSeek 上下文缓存，§6.4 纪律③），可变上下文放尾部。核心模板骨架：

```text
# prompts/factor_spec_extract.md（Research Agent，Hermes-4 本地）
[system] 你是量化研究助理。从论文中提取可回测的因子假设，只输出符合 hypotheses.schema.json 的 JSON。
         每个假设必须给出：表达式草案（qlib 语法）、universe、预期方向、持有期、rationale、known_risks。
         禁止编造论文中不存在的数值；不确定处填 null 并在 rationale 说明。
[user]   <paper 摘要段落 / 关键表格>   ……（可变部分在尾部）

# prompts/event_extract.md（News Agent，本地 14B / 云端兜底）
[system] 你是 A 股事件抽取器。输入财经短讯或公告段落，输出 event.schema.json（五要素：
         event_type/entities/direction/confidence/evidence）。evidence.quote 必须逐字摘录原文。
         无法确定 direction 时输出 neutral + confidence<0.5。
[user]   {source: ..., publish_time: ..., text: ...}

# prompts/next_day_plan_card.md（Portfolio Agent）
[system] 你输出持仓次日情景卡片（六节模板 §11.3）。规则：只给情景/概率/关注点/关键价位，
         禁止输出买卖指令；所有数字标注来源（事件 id / 行情字段）；多空双方各给证据后列出分歧点。
[user]   {position: ..., events: ..., market: ..., playbook_rules: ...}

# prompts/review_challenge.md（Review Agent，70B/云端）
[system] 你是苛刻的评审。必须：①列出 3 个该因子/决策失效的理由；②检查换手与费率现实性；
         ③检查是否与历史被否决因子同签名（rejection_log）；④verdict 只能 pass/needs_revision/reject，
         needs_revision 必须附 findings（id/severity/problem/requiredFix）。
```

契约文件：`agents/prompts/contracts/{hypotheses.schema.json, event.schema.json, plan_card.schema.json, review.schema.json}`——MCP 工具与 Agent 输出共用同一 schema 校验（Pydantic/JSON Schema），校验失败即任务失败（进 NEEDS_REVISION）。

### 8.10 MCP 工具契约明细（orchestration/mcp_servers/）

| server | 工具 | 签名 | 幂等语义 | 超时 |
|---|---|---|---|---|
| ashare-data | `query_bars` | `(codes[], start, end, freq, adjust) -> DataFrame(parquet)` | 只读 | 30s |
| ashare-data | `query_financial_pit` | `(code, as_of, items[]) -> rows`（**as_of 强制**，防前视） | 只读 | 30s |
| ashare-data | `get_universe` | `(date, filters) -> codes[]` | 只读 | 10s |
| ashare-data | `get_events` | `(date_from, date_to, codes[], min_importance) -> event[]` | 只读 | 30s |
| backtest | `run_backtest` | `(config_hash, idem_key) -> run_id` | **写：同 idem_key 返回既有 run_id** | 4h |
| backtest | `get_run_status` | `(run_id) -> state, metrics_uri` | 只读 | 5s |
| backtest | `factor_ic` | `(experiment_id, quick: bool) -> ic_report` | **写：结果缓存 24h** | 30min/5min |
| mlflow-kb | `search_experiments` | `(filters, limit) -> runs[]` | 只读 | 15s |
| mlflow-kb | `search_knowledge` | `(query, scope) -> hits[]`（LlamaIndex） | 只读 | 15s |
| mlflow-kb | `get_paper` / `list_hypotheses` | `(id) -> summary` / `(status) -> list` | 只读 | 10s |
| mlflow-kb | `log_llm_call` | `(agent, model, tokens, cost)` | **写：idem_key** | 5s |

> 纯推理 Agent（不接触文件系统）与 coding Agent 共用这套工具面；工具实现本身是自研薄件中代码量最集中的一块（约 800 行），但每个工具只做"封装既有开源能力 + 幂等语义"，无业务框架化。

### 8.11 执行边界与运行控制（v2 新增，审计 A12）

1. **权限与资源限制**：每个任务按契约限制**工具面、可写目录（`allowed_paths`）、网络访问、并发数与运行时长（`timeout`）**；超限即失败（进 NEEDS_REVISION/HALTED）。"coding 会话不受限"类表述作废。
2. **原始数据与封存集只读/不可见**：原始数据层（raw/、不可变快照）与**最终封存测试集**对执行任务只读或不可见；受限任务不能修改原始数据、读取未授权密钥或封存结果（A12 验收项，待执行）。git worktree 仅隔离目录，须以进程级权限控制补齐（§8.6④）。
3. **提示注入边界**：新闻、论文、公告**正文中的指令不得触发工具执行**；外部文本一律作为数据处理（与调研 03 §4.6 一致）。
4. **修复与假设变更分开登记（A04）**：修复代码错误（fix_log）与修改研究假设（hypothesis_log）分开登记，**不得以提高最终测试收益为修复目标**；保存失败、否决及参数变体，删除"入库因子数量"硬指标（零有效因子允许正常验收，细则见 §12–13 第 2 批修订）。
5. **调用与额度**：记录全部模型调用与重试（`llm_calls`/`dsh-token-meter`，§8.5/§6.4）；若已有服务配置**硬额度**，**调用前预留费用、完成后结算**；额度或服务限流触发时停止新增调用、保留检查点，确定性数字报告继续生成（涉及新付费数据或硬件采购时先说明缺口和费用）。

---

## 9. 闭环 A：论文→因子→回测自动化 Pipeline（`pipeline_paper2factor`）

> 依据：调研 02 §3（挖掘路线）、§4（指标映射）、§8（路线）；调研 04 §3.1（Research/Factor/Alpha/Review 角色）、§3.7（夜间循环）。触发：**每周五 20:00 周级流水线 + arXiv/RSS 新论文事件**。

### 9.1 流水线总览（每环节输入 → 处理 → 输出）

| # | 环节 | 输入 | 处理（执行体） | 输出 | 落点 |
|---|---|---|---|---|---|
| 1 | **获取** | arXiv RSS（q-fin）、SSRN/研报收藏、`papers/inbox/` | 增量拉取 PDF（Research 前置脚本，无 LLM） | `papers/<id>/paper.pdf` + 元数据 | knowledge/papers/ |
| 2 | **筛选去重** | 新 PDF 元数据 | `paper_id = arXiv id/DOI` 查重（§8.6③）；主题相关性打分（Hermes-4 本地） | keep/drop 决策 + 理由 | tasks 表（SKIPPED_DUPLICATE） |
| 3 | **精读总结** | paper.pdf | Research Agent（Hermes-4-14B 本地；难题走 DeepSeek-V4-Pro）：方法/数据/结论/局限提炼 | `summary.md`（含可实现性判断） | knowledge/papers/<id>/ |
| 4 | **提取因子/假设** | summary.md | Research Agent 结构化抽取：表达式草案、适用 universe、预期方向、持有期 | `hypotheses.json`（JSON 契约见 §9.2） | knowledge/papers/<id>/ |
| 5 | **假设去重与排期** | hypotheses.json + 因子库状态 | Alpha Agent：与既有因子 embedding 查重（换名重试打回）；按预期价值×实现成本排优先级 | `experiments/queue.yaml`（下批实验计划） | experiments/ |
| 6 | **代码生成** | hypothesis（spec.json） | Factor Agent（DSH/Codex，git worktree `exp/<id>`）：因子实现 + 单测 + spec.yaml | `factors/impl/<name>.py` + 单测 | experiments/<id>/ |
| 7 | **单因子体检** | 因子代码 | 因定性引擎：alphalens-reloaded tearsheet + 自研稳定性脚本（滚动 IC/IC_IR） | IC/RankIC/分层/换手/衰减报告 | results/ + MLflow |
| 8 | **回测与统计检验** | 通过体检的因子 | Backtest Agent：qlib 回测（TopK/组合），统计显著性 + 多重检验校正（§13.4） | metrics.json（含 Deflated Sharpe/White RC 校正项） | MLflow run |
| 9 | **稳健性测试** | metrics.json | 固定脚本：样本外/滚动（RollingGen）/分市场阶段（牛/熊/震荡/危机四段） | oos_sharpe、regime_table | results/ |
| 10 | **入库评审** | 全部产物 | Review Agent 按准入门槛（§12.3）评审，verdict + findings | review.md（pass/needs_revision/reject） | knowledge/reviews/<id>/ |
| 11 | **入库/否决** | review.md | 因子库登记（版本/血缘/门槛证据）或否决记录（防止重蹈） | `factor_registry` 新条目 / `rejection_log` | DuckDB + factors/registry/ |

### 9.2 假设 JSON 契约（环节 4 → 6 的交接格式）

```json
{
  "hypothesis_id": "H2026W39-002",
  "source_paper": "arXiv:2506.xxxxx#h2",
  "name_draft": "overnight_reversal_5d",
  "family": "reversal",
  "expression_draft": "-Ref($close,-5)/Ref($open,-5)-1",
  "universe": "all_A_ex_st_ex_new(60d)",
  "expected_direction": "negative",
  "holding_period": "5d",
  "rationale": "论文称隔夜过度反应在 A 股小市值显著；样本 2015–2023",
  "known_risks": ["与价量反转族相关性可能>0.7", "论文未披露交易成本"],
  "priority": 2, "effort": "S"
}
```

### 9.3 各环节验收与工具映射

| 环节 | 关键指标 | 工具（复用，调研 02 §4.1） | 验收口径 |
|---|---|---|---|
| 7 单因子体检 | IC、RankIC、IC_IR、分层单调性、换手 | alphalens-reloaded + 自研滚动稳定性（~50 行） | 进入 §12.3 准入卡初筛 |
| 8 回测 | Sharpe、年化、最大回撤、换手、成本后超额 | qlib risk_analysis + quantstats | 成本后超额 >0 且回撤在策略预算内 |
| 8 统计检验 | t-stat、**Deflated Sharpe、White Reality Check** | 自研封装（§13.4） | 校正后仍显著才进环节 9 |
| 9 稳健性 | oos_sharpe、regime_table 四段一致性 | qlib RollingGen + quantstats 分段 tearsheet（~30 行切分） | 样本外不崩塌、≥3/4 市场阶段为正 |
| 10 评审 | 与 spec 一致性、换手/费率现实性、过拟合质控 | Review Agent（Hermes-70B 或 DeepSeek-V4-Pro） | verdict=pass 才入库 |

**RD-Agent 接入点（P2）**：环节 5–8 用 microsoft/RD-Agent（MIT，14,711★）的"研报提取因子→生成代码→qlib 回测→迭代进化"闭环作为挖掘执行体（其论文 R&D-Agent-Quant, arXiv:2505.15155）；产出的表达式因子注册为 qlib 自定义算子（`register_all_ops` 扩展）。**gplearn（GP）作对照组**（自写 IC 适应度 + 表达式去重）；**alphagen 只借鉴 RL 思路不引代码**（无 License）。

**防过拟合纪律（Pipeline 级，调研 04 §5.1）**：挖掘类工具会系统性放大过拟合——①同一族因子反复微调参数直接打回；②最终准绳 = 滚动样本外 + 分市场阶段 + **成本后**收益；③Optuna 搜索嵌套在滚动框架内；④每周实验数量与通过数量进周报，通过率异常高（>30%）触发 Review 专项质询。

---

## 10. 闭环 B：新闻→市场事件→持仓分析 Pipeline（`pipeline_news2position`）

> 依据：调研 03 §3（schema/契约）、§4（频率/去重）、§1.3（信源）、§5（外围市场）。触发：**盘中 5–15 分钟轮询 + 盘后批处理 + 晨报 08:30**。

### 10.1 流水线总览

| # | 环节 | 输入 | 处理（工具链） | 输出 |
|---|---|---|---|---|
| 1 | **抓取** | 财联社电报（1–5 分钟轮询，`ak.stock_info_global_cls`）、东财 7×24/个股新闻（`stock_info_global_em`/`stock_news_em`）、新浪快讯（`stock_info_global_sina`）、巨潮公告列表（15:00–23:00 每 30 分钟 + 08:00 补漏）、tushare 结构化事件（盘后 1 次）、外围（yfinance/akshare 盘后 + 美股收盘后补） | 采集适配器（§5.1.3）→ `news_raw` 表 + raw 落盘 |
| 2 | **去重与修订链** | news_raw | 首期只按 `(source, source_doc_id)` 唯一 + 归一化内容哈希；`news_relation` 关系登记（转载/更正/同标题不同内容不丢弃、可区分，`revision_of`/`revision_kind` 保留修订链）；金额/单位/主体/日期逐字段带 evidence_quote，缺证据输出 unknown（A11）。simhash/datasketch/text2vec 语义层为 P1 按需引入；**不设固定滤重比例**（信源优先级：巨潮>财联社>东财>新浪>自媒体取 canonical） | `news_cluster` + `news_relation`（报道热度=簇内条数/信源数） |
| 3 | **分类摘要** | news_cluster | 规则硬分类（巨潮 category/频道）+ 微调 BERT 15 类事件分类；长公告本地 7B–14B"5W+金额"摘要（持仓股/高影响事件用云端） | category + summary |
| 4 | **事件抽取** | 分类后文本 | LLM + JSON schema 约束（OneKE/DISC-FinLLM prompt 思路；持仓股/高重要性走云端，短快讯走本地 Hermes-4）；DeepKE 小模型兜底；事件级二次归并（同 `event_type+entities+3 日窗`） | `event` + `event_entity` 表（契约见 §10.2） |
| 5 | **情绪/方向** | 全量簇 + 重点事件 | 全量：finance-sentiment-zh-base（中文）+ finbert（英文）打分；重点事件：LLM 输出"影响方向 + 置信度 + 作用期限（intraday/days/weeks）" | direction/confidence/horizon |
| 6 | **关联分析** | event + 持仓/行业映射 | "事件→个股→行业→持仓"两跳归因（`event_entity.role IN ('peer','supply_chain')` + `industry_map` 历史区间表）；输出"今日影响我的持仓的事件"（§10.3 查询） | 持仓影响清单（喂给闭环 C） |
| 7 | **沉淀** | 全部 | events.jsonl（append-only）→ 事件库 → 高价值样本进 `events_kb` few-shot 库；`event_feedback` 事后回写（ret_1d/5d/20d、hit） | 事件库 + 反馈闭环 |

### 10.2 事件抽取 JSON 契约（调研 03 §3.3 原文采用）

```json
{
  "event_type": "增减持", "sub_type": "高管拟减持",
  "occur_time": "2026-09-22",
  "entities": [{"code": "300750.SZ", "name": "宁德时代", "role": "target",
                "direction": "negative", "weight": 0.9}],
  "direction": "negative", "horizon": "days", "confidence": 0.85, "importance": 0.6,
  "summary": "副总经理计划减持不超过 120 万股（约占总股本 0.03%），窗口期 3 个月。",
  "key_facts": {"amount_cny": 3.2e8, "share_pct": 0.03, "duration": "3个月"},
  "evidence": {"news_cluster_id": 10231, "quote": "……公告编号 2026-045……"}
}
```

**五要素齐全才落库**：事件类型、涉及标的、影响方向、置信度、证据来源；`evidence.quote` 保证可追溯，`key_facts` 支撑后续量化回测（事件因子研究的原料）。事件类型枚举（15+ 类）沿用调研 03 §3.2：`业绩预告/定期财报/增减持/股票回购/并购重组/股权激励/定增配股/解禁/监管处罚问询/诉讼仲裁/中标订单合同/产品调价/产能扩产/股东质押冻结/高管变动/ST与退市风险/行业政策/宏观数据/货币政策/海外市场传导/大宗商品/汇率/舆情传闻/其他`。

**事件类型分类器标签体系（15+ 类细化表；分类=规则硬分类 + 微调 BERT 二级）**：

| event_type | 典型子类 | 默认方向 | 默认 horizon | 关键 key_facts | 结构化优先源（tushare/巨潮 category） |
|---|---|---|---|---|---|
| 业绩预告 | 预增/预减/扭亏/首亏 | 按幅度 | days | 净利区间、同比% | tushare forecast |
| 定期财报 | 年报/半年报/季报 | 按超预期 | weeks | 营收/净利/毛利率 | tushare fina |
| 增减持 | 高管/大股东/回购 | 减持 negative | days–weeks | 股数、占比、窗口期 | tushare stk_holdertrade |
| 股票回购 | 回购预案/实施 | positive | weeks | 金额、价格上限 | 巨潮 category |
| 并购重组 | 收购/借壳/资产注入 | 视对价 | weeks | 对价、标的、稀释 | 巨潮 category |
| 股权激励 | 授予/行权 | 中性偏正 | weeks | 行权价、考核目标 | 巨潮 category |
| 定增配股 | 预案/获批/实施 | 稀释 negative | weeks | 规模、定价基准 | 巨潮 category |
| 解禁 | 首发/定增解禁 | negative | intraday–days | 解禁市值、股东 | tushare share_float |
| 监管处罚问询 | 问询函/立案/处罚 | negative | days | 事由、金额 | 巨潮 category |
| 诉讼仲裁 | 立案/判决 | 视角色 | weeks | 金额、进度 | 巨潮 category |
| 中标订单合同 | 大单/框架协议 | positive | days–weeks | 金额、占营收% | 财联社/东财快讯 |
| 产品调价 | 提价/降价 | 视方向 | days | 幅度、品类 | 快讯 |
| 产能扩产 | 投产/扩产/停产 | 视方向 | weeks | 投资额、产能 | 快讯 |
| 股东质押冻结 | 质押/冻结/平仓风险 | negative | days | 比例、警戒线 | 巨潮 category |
| 高管变动 | 辞任/被查/新任 | 视情形 | days | 职位、任期 | 巨潮 category |
| ST 与退市风险 | ST/退市整理/恢复 | negative | weeks | 触发条件 | 公告+universe |
| 行业政策 | 监管/补贴/规划 | 视条款 | weeks | 受益/受损板块 | 政策频道+证监会 |
| 宏观数据 | GDP/CPI/社融/PMI | 视预期差 | intraday–days | 实际 vs 预期 | tushare macro |
| 货币政策 | 降准/降息/MLF/LPR | 视方向 | days | 幅度 | 央行+快讯 |
| 海外市场传导 | 美联储/美债/VIX | 视方向 | intraday | 数值变动 | yfinance/FRED |
| 大宗商品 | 原油/黄金/铜 | 传导至产业链 | days | 涨跌幅 | yfinance/akshare |
| 汇率 | USDCNY/CFETS | 传导至进出口 | days | 变动 | FRED/akshare |
| 舆情传闻 | 未证实消息 | confidence<0.5 | intraday | 来源可信度 | 财联社/股吧（从严） |
| 其他 | — | neutral | — | — | — |

### 10.3 事件库 schema 与关联查询

schema 采用调研 03 §3.1 七表设计（`news_raw / news_cluster / news_embedding / event / event_entity / position+watchlist+industry_map / event_feedback`），落地在 DuckDB（MVP）/PostgreSQL（P1 并发需要时）。核心查询"今日影响我的持仓的事件"（原文示例）：

```sql
SELECT e.*, ee.entity_code, ee.direction, p.weight
FROM event e
JOIN event_entity ee ON ee.event_id = e.id
JOIN position p ON p.code = ee.entity_code AND p.date = CURRENT_DATE
WHERE e.publish_time >= CURRENT_DATE AND ee.weight >= 0.5
ORDER BY e.importance DESC;
```

行业传导：`event_entity.role` + `industry_map`（注意用 `valid_from/valid_to` 区间过滤，防分类未来信息）扩展到持仓的同业/上下游，实现"事件 → 行业 → 持仓"两跳归因。该查询输出即闭环 C 的"关键新闻事件"输入（§11.3）。

### 10.4 落地要点

- **公告全文控量**：PDF 只对持仓股 + 重要类目 + 高影响事件下载（控磁盘与解析成本，调研 03 §4.1）；全文用 use_cninfo（PDF→Markdown，MIT）归档到 `news_raw.content`。
- **政策类**：无单一成熟开源库——akshare 政策资讯 + 财联社"政策"频道 + 证监会/央行 `requests+BeautifulSoup` 薄爬虫（§4.2 边界 1 的例外）。
- **情绪源（股吧/韭研公社）**：FinSpider 思路自研（无 License 不引代码），合规与频率限制从严，P2 再考虑。
- **外围资产映射表**（调研 03 §5.2）作为 ETL 配置：美股指数/港股/黄金/原油/美债（FRED DGS10）/美元指数/人民币汇率的主源（yfinance/FRED）+ 备源（akshare），两口径并存入 `clean/reference/`。
- **晨报（08:30）**：聚合昨日事件簇 + 持仓影响 + 外围市场（云端 LLM 生成），属闭环 B 输出、闭环 C 输入。

---

## 11. 闭环 C：每日收盘后→自动复盘→次日预案 Pipeline（`pipeline_daily_review`）

> 依据：调研 04 §3.2（16:15 post-market 流水线 + 07:30 pre-market）、调研 02 §2.1.9（quantstats 复盘报告）、调研 03 §3.1（持仓关联）。**系统输出结构化证据与情景分析，不输出买卖结论**（§1.3 主线 2）。

### 11.1 触发时间表与依赖数据（v2：数据就绪驱动出报，审计 A10）

> **出报由数据就绪驱动，不以固定时点假定数据齐备**：下表时间仅为**目标时点**——不同来源更新时间不同，16:15 不能天然保证日线、两融、龙虎榜和公告齐全，实际出报以数据集就绪状态为准。

| 时间 | 动作 | 执行体 | 依赖数据（就绪检查） |
|---|---|---|---|
| 15:30–16:10 | EOD 数据落库 + DQ 校验（`etl run --dataset bar_daily ...`） | Data（确定性） | 交易所收盘数据、baostock/tushare 日线（约 16:00 后可得） |
| **16:15** | **Pipeline 触发**（cron/launchd → DSH goal） | Orchestrator | 当日 clean Parquet + `dq_report.md` 无 blocker |
| 16:15–16:30 | 盘后快照：指数/行业/资金流/涨跌停统计/风格轮动 | Market Agent（Hermes-4 本地） | 行情 + akshare 板块资金流 + 两融 |
| 16:20–16:45 | 持仓逐只分析（确定性计算 + 解读） | Portfolio Agent | `position` 当日快照 + events 当日 + universe_snapshot |
| 16:30–17:00 | 当日复盘：执行偏差、归因、失败质询 | Review Agent（Review-lite 日级） | 交易日志 + 信号 vs 实际执行 + metrics |
| 17:00–17:30 | **次日预案生成**（逐只持仓情景卡片） | Portfolio + Market Agent（多空辩论结构） | 上述全部 + 隔夜外围（美盘前值） |
| 17:30 | 输出复盘报告 + 次日预案 → 展示层 + 知识库 | Orchestrator | — |
| 次日 07:30–08:30 | 隔夜补数据（美股收盘/公告）→ 晨报 → 预案修订标记 | Data + News + Market | yfinance/FRED 隔夜数据 |

**依赖数据清单**：①当日 EOD 行情（含复权因子增量、停牌/涨跌停标记）；②当日 universe_snapshot（含 ST/上市状态）；③当日持仓快照（手工录入或券商导出，`position` 表）；④当日事件（events.jsonl，闭环 B 产出）；⑤当日资金流/两融/龙虎榜（akshare/tushare 盘后）；⑥组合净值与基准（quantstats 输入）。

**数据就绪登记（A10；每数据集一行，写入 `runs/data/<date>/manifest.json`）**：

| 字段 | 含义 |
|---|---|
| dataset | 数据集名（bar_daily / margin_detail / dragon_tiger / announcements / events …） |
| as_of（截至时间） | 数据实际覆盖到的日期/时刻 |
| expected_date（预期日期） | 预期应到达的交易日/批次 |
| ready_state（就绪状态） | ready / delayed（超预期未到） / failed / stale（沿用旧数据） |

**出报规则（v2，A10）**：
1. **初版 → 补齐 → 晨间修订**：目标时点先生成**明确标注缺项的初版**报告；缺失数据集到达后补齐并重算相关章节；次日 08:30 晨报合并夜间补漏（公告凌晨批次等），对前一交易日形成**修订版**，修订记录写入报告尾部。
2. **旧数据标 `stale`**：沿用旧批次的数据必须标注 stale 及实际 as_of，**不得伪装为当天数据**。
3. **单源延迟或故障**：报告直接展示缺项清单及所用数据版本；恢复后补跑相关任务，**补跑结果不重复统计**（幂等键保证，§8.6）。

### 11.2 复盘报告（目标 16:45；就绪驱动：初版标注缺项 → 补齐 → 次日晨间修订）内容规格

1. **市场概览**（Market）：指数表现、行业轮动、涨跌停家数、情绪温度、风格（大小盘/成长价值）。
2. **组合表现**（确定性）：当日/本周净值、vs 基准超额、归因（行业/个股贡献 Top5）、换手与成本。
3. **执行偏差检查**（Review-lite）：昨日预案 vs 实际走势复核（情景命中情况）、信号执行滑点、纪律违反（playbook 违例）标记。
4. **事件复核**（News→Review）：今日事件方向判断与盘面对照，写 `event_feedback.ret_1d/hit`。
5. **改进清单**（Review）：行为纠偏项（写入 playbook.md 修订建议）。

**复盘报告输出模板**（`runs/reviews/<date>/daily_review.md`，展示层与 git 归档双落点）：

```markdown
# 每日复盘 · <date>（目标 16:45 by orchestrator + market/portfolio/review-lite agents；就绪驱动出报）
## 0. 数据就绪与版本（A10）
- 数据版本：data_snapshot_id=<id>；各数据集 as_of / expected_date / ready_state（见 §11.1 登记表）
- 缺项清单：未就绪数据集及影响章节；stale 数据标注（旧数据不伪装为当天）
## 1. 市场概览
- 指数：上证 +x% / 深成 +x% / 创业板 +x% / 沪深300 +x%；两市成交 x 万亿（环比 +x%）
- 行业轮动：领涨 Top3（附资金流口径）/ 领跌 Top3；风格：大盘/小盘 x:y，成长/价值 x:y
- 情绪面：涨停 x 家 / 跌停 x 家 / 炸板率 x%；连板高度 x；北向（季度口径数据更新标记）
## 2. 组合表现（确定性计算）
- 净值：当日 +x% / 本周 +x% / 本月 +x%；vs 000300 超额 +x bps
- 归因：行业贡献 Top5、个股贡献 Top5（附权重与涨跌）
- 交易成本：今日换手 x%、费用 x 元、滑点估计 x bps
## 3. 昨日预案复核（plan_score 回写）
- 情景命中：逐持仓标注 实际走的情景 A/B/C + score；情景概率校准偏差记录
- 预案失效参考位是否触发、触发后 playbook#N 流程执行情况
## 4. 事件复核（event_feedback 回写）
- 今日重要事件方向判断 vs 盘面对照表（event_id / 预判方向 / 实际 / hit）
## 5. 纪律与改进（playbook 修订候选）
- playbook_violation 列表（若有）；本周改进项（写入 postmortem 累积）
- 明日关注（进入次日预案 §6 的"明日必看"）
```

### 11.3 次日预案（目标 17:30，就绪驱动，缺项标注同 §11.1）：逐只持仓结构化卡片（输出模板）

> 设计原则：**结构化情景分析而非买卖结论**。每张卡片回答"处于什么状态、受什么影响、风险在哪、关键价位在哪、明天可能怎么走、该关注什么"，行动项只到"关注/验证/提示"级，不下单指令。

```markdown
## 次日预案 · 2026-09-23（生成于 2026-09-22 17:30 by portfolio-agent/hermes-4-14b）
## 持仓卡片 — 600519.SH 贵州茅台（权重 8.2%，成本 1,4xx，现价 1,5xx，浮盈 x%）

### 1. 当前状态
- 技术状态：20 日均线上方 / 下方；量能环比 +x%；处于近 60 日区间 [a, b] 的 % 分位
- 持仓状态：浮盈 x%、持有 x 日、上次操作 yyyy-mm-dd（依据 playbook#3）
- 基本面/估值快照：PE(TTM) xx（行业中位 xx）；最新报告期 2026H1，披露日 2026-08-2x（PIT）

### 2. 主要影响因素（按权重排序，附证据）
- [0.9] 事件：Q3 中秋动销超预期传闻（cluster#10231，财联社 09-22 14:xx；confidence 0.7）
- [0.6] 资金面：北向季度持股变动 +x%（2024-05-13 后改季度披露口径，见 data_regime_change）
- [0.4] 行业：白酒板块资金流连续 3 日净流入（akshare 东财口径）

### 3. 多空风险点（辩论结构：正反方各给证据，不给结论）
- 多方证据：动销传闻 + 估值回到 x 年中位以下 + 机构调研频次上升
- 空方证据：传闻未经证实（evidence 缺官方来源）；消费板块拥挤度处 90% 分位；解禁日 2026-11-xx 临近
- 关键分歧点：传闻证实/证伪的时点与力度

### 4. 关键新闻事件（今日已发生，来自事件库查询 §10.3）
- 09-22 20:00 [公告] 股东大会决议（中性，importance 0.3）
- 09-22 14:2x [传闻] 中秋动销（positive，importance 0.6，待验证）

### 5. 关键价位
- 压力位：1,5xx（前高/成交密集区上沿）；支撑位：1,4xx（20 日线/缺口）
- 预案失效参考位：跌破 1,4xx 且放量 → 触发 playbook#5 风险复核流程（非交易指令）

### 6. 次日情景与关注点（三情景 + 触发条件）
- 情景 A（基准，p≈0.55）：传闻无进一步信息，跟随板块波动 ±1%。关注：开盘竞价量能、板块联动
- 情景 B（上行，p≈0.25）：动销数据/研报证实。触发：盘前官方口径或龙头同业跟涨。关注：冲高兑现压力位 1,5xx
- 情景 C（下行，p≈0.20）：传闻证伪或消费数据走弱。触发：板块跳空低破支撑。关注：1,4xx 支撑有效性、是否进入 playbook#5 流程
- 明日必看：10:00 社零数据（宏观）、盘后龙虎榜、北向季度口径数据有无更新
```

**输出物落点**：`runs/plan/<date>/next_day_plan.md`（展示层首页）+ 卡片结构化字段同步进 `trading_journal`（§11.5）；次日 15:00 由 Review-lite 反查"情景命中"并回写 score。

### 11.4 复盘沉淀到知识库（闭环的"持续迭代"落点）

| 沉淀物 | 写入 | 消费方 |
|---|---|---|
| 周复盘 `postmortem.md`（周五 20:00） | knowledge/postmortems/<ISO-week>/ | Alpha Agent（选题：哪类因子当前市场阶段有效） |
| 行为纠偏清单 | `knowledge/playbook.md`（Review Agent 修订，git diff 可审计） | Portfolio/Market Agent（预案约束，卡片"预案失效参考位"引用条款） |
| 事件方向命中统计 | `event_feedback` 表 | News Agent（few-shot 库 + 阈值迭代） |
| 情景命中率（预案 score） | `trading_journal.plan_score` | Review Agent（月度归因：情景概率校准改进） |
| 失败实验质询记录 | knowledge/reviews/ | Alpha Agent（防换名重试） |

### 11.5 交易行为数据库（`trading_journal` schema）

```sql
CREATE TABLE trading_journal (
  trade_id TEXT PRIMARY KEY, trade_date DATE, code TEXT,
  action TEXT,                    -- buy/sell/hold_decision（人工确认后的实际动作）
  signal_source TEXT,             -- 哪个策略/因子/预案情景触发
  plan_id TEXT,                   -- 关联次日预案（runs/plan/<date>）
  plan_scenario TEXT,             -- 实际兑现的情景 A/B/C
  plan_score DOUBLE,              -- 情景命中评分（事后回写）
  execution_price DOUBLE, planned_price DOUBLE, slippage DOUBLE,
  discipline_check TEXT,          -- ok / playbook_violation:<rule_id>
  postmortem_ref TEXT,            -- 关联周复盘小节
  note TEXT
);
```

**自动化边界（红线）**：Agent 只产出建议与情景，**不接实盘下单**；未来若接实盘（P2+），必须独立风控进程 + 人工确认闸门 + 熔断（调研 04 §5.2）。

**次日预案卡片结构化契约**（`plan_card.schema.json`，卡片 Markdown 由该 JSON 渲染；字段进 trading_journal 关联）：

```json
{
  "schema_version": "1.0",
  "plan_date": "2026-09-23", "generated_at": "2026-09-22T17:30:00+08:00",
  "created_by": {"agent": "portfolio-agent", "model": "hermes-4-14b"},
  "cards": [{
    "code": "600519.SH", "weight": 0.082,
    "state": {"tech": "above_ma20", "pct_in_range_60d": 0.72, "pnl_pct": 0.061, "held_days": 23},
    "factors_influence": [{"weight": 0.9, "type": "event", "ref": "cluster#10231"},
                          {"weight": 0.6, "type": "flow", "ref": "northbound_quarterly"}],
    "bull_bear": {"bull": ["...evidence..."], "bear": ["...evidence..."], "key_debate": "..."},
    "key_events_today": [{"event_id": 8801, "direction": "positive", "importance": 0.6}],
    "key_levels": {"resistance": [1580], "support": [1455], "invalidation": 1455},
    "scenarios": [
      {"name": "A", "prob": 0.55, "desc": "...", "triggers": [], "watch": ["竞价量能"]},
      {"name": "B", "prob": 0.25, "desc": "...", "triggers": ["官方动销口径"], "watch": ["1580"]},
      {"name": "C", "prob": 0.20, "desc": "...", "triggers": ["板块跳空低破支撑"], "watch": ["playbook#5"]}
    ],
    "must_watch_tomorrow": ["10:00 社零数据", "盘后龙虎榜"]
  }]
}
```

**playbook.md 初始条款示例**（交易行为数据库的文字侧；Review Agent 周度修订、git diff 审计、卡片"预案失效参考位"按 `#编号` 引用）：

| # | 条款示例（可按个人风格改写） | 触发场景 | 违例标记 |
|---|---|---|---|
| #1 | 单票权重上限 10%，行业集中度上限 30% | 预案生成时检查 | playbook_violation:#1 |
| #2 | 传闻类事件（confidence<0.5）不得作为加仓依据 | 情景 B 触发时 | #2 |
| #3 | 同一标的两次调仓间隔 ≥5 交易日（防手痒） | 交易前检查 | #3 |
| #4 | 单日止损纪律：浮亏 >x% 强制进入复核（非自动卖出） | 盘中/预案 | #4 |
| #5 | 支撑位跌破且放量 → 触发风险复核流程（输出复核清单，不下指令） | 情景 C | —（流程条款） |
| #6 | 事件驱动持仓须记录 evidence 与验证期限，到期未验证降权 | 周复盘 | #6 |
| #7 | 新因子未过准入卡不得进入组合参考 | 预案引用因子时 | #7 |
| #8 | 周换手上限 x%（成本预算约束） | 周复盘 | #8 |
| #9 | 情景概率校准偏差 >20% 连续 2 周 → 修订概率模型 | 周复盘 | —（自校准条款） |
| #10 | 禁止在开盘竞价与尾盘最后 5 分钟做冲动决策（记录例外理由） | 交易日志 | #10 |

---

## 12. 实验追踪、因子版本管理与结果管理

> 依据：调研 02 §5（MLflow/QlibRecorder/Optuna）、§7.6（qrun 实验记录）；调研 04 §3.5（一回测一 run 映射）、§3.6（idem_key/缓存）；调研 01 §8.3（PIT 多版本思想）。

### 12.1 因子库 schema（个人因子库的资产化设计）

```sql
-- DuckDB factor_registry（factors/registry/<factor_id>.yaml 为人类可编辑源，DB 为索引）
CREATE TABLE factor_registry (
  factor_id     TEXT PRIMARY KEY,      -- F<major><3位序号>.<family>.v<version>，见 §12.1.1
  name          TEXT, family TEXT,     -- momentum/reversal/value/quality/growth/event/flow...
  version       INT,  status TEXT,     -- draft / testing / admitted / decayed / deprecated / rejected
  expression    TEXT,                  -- 表达式因子（qlib ops）；代码因子则为 NULL
  python_impl   TEXT,                  -- factors/impl/<file>.py + git_sha
  universe_filter TEXT, neutralize TEXT,
  hypothesis_id TEXT,                  -- 血缘①：来源假设 → papers/<id>/hypotheses.json#h2
  derived_from  TEXT,                  -- 血缘②：派生自哪个因子（迭代/改进关系）
  inputs_hash   TEXT,                  -- 血缘③：输入数据口径（数据集+区间+处理链哈希）
  created_by    JSON,                  -- {agent, model, experiment_id}
  created_at    TIMESTAMP, admitted_at TIMESTAMP,
  admit_metrics JSON,                  -- 入库时的门槛证据（IC_IR/分层/换手/oos/校正后显著性）
  decay_monitor JSON                   -- 衰减监控状态（§12.4）
);
CREATE TABLE factor_version_diff (     -- 版本血缘可追溯
  factor_id TEXT, version INT, git_sha TEXT, diff_summary TEXT,
  reason TEXT, changed_at TIMESTAMP, PRIMARY KEY (factor_id, version)
);
CREATE TABLE rejection_log (           -- 否决记录：防止换名重试（§8.6③）
  reject_id BIGINT PRIMARY KEY, hypothesis_id TEXT, factor_signature TEXT,  -- 表达式归一化哈希
  verdict TEXT, findings JSON, rejected_by TEXT, rejected_at TIMESTAMP
);
```

#### 12.1.1 因子 ID / 版本 / 血缘规则

| 规则 | 内容 |
|---|---|
| ID 格式 | `F<major><seq3>.<family>.v<ver>`，如 `F1023.momentum.v2`；家族编号段 1xxx=momentum、2xxx=reversal、3xxx=value、4xxx=quality、5xxx=event、6xxx=flow… |
| 版本语义 | 表达式/算法变更 → `v+1`（新条目 + `derived_from` 指向旧版）；仅参数/中性化/过滤调整 → `version_diff` 行；口径修复（数据错误）→ 原版本标注 `superseded` 并全量重算 |
| 血缘三元组 | `hypothesis_id`（研究来源）+ `derived_from`（因子谱系）+ `inputs_hash`（数据口径）——任何一者缺失不许入库 |
| 命名稳定性 | name 一经 admitted 不再变；换名 = 新因子 + rejection_log 关联（防"同一因子换个名字反复试"） |

### 12.2 实验追踪：一回测一 run（MLflow）

映射规则见 §8.5。补充约定：

1. **qrun/QlibRecorder 产物进同一 MLflow 后端**（qlib `set_uri` 指向共享 server），Alpha158 基准、RD-Agent 挖掘、自研实验全部可对比（调研 02 §5.2/§7.6）。
2. **实验目录 ↔ run 双向可寻址**：run tag `experiment_id` ↔ `experiments/<id>/results/` 的 `mlflow_run_id` 记录文件；artifacts 全量落 `experiments/<id>/results/`（文件为事实来源）。
3. **LLM tracing**：所有 Agent 调用记 prompt/completion/token/成本（MLflow 3.x GenAI tracing 或 `llm_calls` 表），按 agent/模型聚合出用量归属（对齐 §6.4 用量记录与已有服务额度口径，A06）。
4. **Optuna 集成**：`mlflow.set_tracking_uri` 复用同一后端；每个 trial = 一个 run（tag `optuna_study/trial`），搜索嵌套在 RollingGen 滚动框架内（§13.5）。

### 12.3 因子入库门槛（准入卡，`admission_check` 自动判定）

> 全部通过才 `status=admitted`；门槛数值为首版建议，按个人风险偏好在 `backtest/thresholds.yaml` 调整并版本化。

| # | 门槛 | 建议阈值 | 数据来源 |
|---|---|---|---|
| 1 | RankIC 显著 | \|mean RankIC\| ≥ 0.03 且 IC_IR ≥ 0.3（全样本） | alphalens-reloaded |
| 2 | 分层单调性 | 5 分组收益单调（Spearman ≥ 0.9），多空价差为正 | alphalens tear sheet |
| 3 | 成本后有效 | 按换手×费率（§5.3.1）扣减后多头超额仍 >0 | quantstats + 回测 |
| 4 | 样本外不崩塌 | oos_sharpe ≥ 0.5 × 全样本 Sharpe 且 >0 | RollingGen 滚动 |
| 5 | 分阶段稳健 | 牛/熊/震荡/危机四段中 ≥3 段 RankIC 同号 | 分阶段切分脚本 |
| 6 | 多重检验校正 | Deflated Sharpe >0（p<0.05）且 White Reality Check p<0.10 | §13.4 封装 |
| 7 | 低冗余 | 与已入库因子 \|RankIC 相关\| < 0.7（超限要求提供增量证据：组合 IC 提升） | 因子相关矩阵 |
| 8 | 评审通过 | Review Agent verdict=pass（spec 一致性/费率现实性/无换名重试） | review.md |
| 9 | 血缘齐全 | hypothesis_id / derived_from / inputs_hash 完整 | factor_registry 非空约束 |

**未达标因子不删**：进 `status=testing/rejected` + rejection_log（否决理由是知识资产，防止重复劳动）。

**零有效因子允许正常验收（A04）**：本门槛判定的是**证据质量**，不设"入库因子数量"硬指标（原"因子库 admitted ≥30"等数量验收删除）；一轮研究零个因子通过准入即为正常结果，如实记录失败与否决即可。

**门槛配置示例**（`backtest/thresholds.yaml`——阈值版本化，调整即新 `thresholds_version` 进 inputs_hash）：

```yaml
thresholds_version: "2026-09.v1"
admission:
  rank_ic_abs_min: 0.03
  ic_ir_min: 0.3
  quantile_monotonic_spearman: 0.9
  cost_adjusted_excess_min: 0.0        # 成本后多头超额 > 0
  oos_sharpe_ratio_min: 0.5            # oos / full-sample
  regime_same_sign_min: 3              # 4 段中 ≥3 段同号
  deflated_sharpe_p: 0.05
  white_rc_p: 0.10
  max_corr_with_admitted: 0.7
cost_model: {open_cost: 0.0002, close_cost: 0.0007, min_cost: 5.0, impact_cost: 0.001}
universe_default: "all_A_ex_st_ex_new(60d)_ex_suspended_ex_limit"
```

**因子家族定义表**（`families` 编号段与准入侧重）：

| family | 编号段 | 定义 | 持有期基准 | 准入侧重 |
|---|---|---|---|---|
| momentum | 1xxx | 价格/盈利动量、行业动量 | 5–20d | 分阶段稳健（动量在熊市易崩） |
| reversal | 2xxx | 短期反转、隔夜反转 | 1–5d | 换手与成本敏感性 |
| value | 3xxx | EP/BP/股息率等估值 | 月级 | **PIT 强制**（财报修订） |
| quality | 4xxx | ROE/现金流/应计质量 | 月级 | **PIT 强制** |
| growth | 5xxx | 营收/利润增速、预期差 | 月级 | **PIT 强制**+预告口径 |
| event | 6xxx | 事件驱动（解禁/回购/增减持…） | intraday–weeks | event_feedback.hit≥55% |
| flow | 7xxx | 资金流/两融/北向（新口径） | 1–10d | **regime 声明强制**（北向断点） |
| sentiment | 8xxx | 新闻情绪/舆情热度 | 1–5d | 去重质量 + 证据链 |
| vol/risk | 9xxx | 波动率、特质波动、流动性 | 5–20d | 市场阶段一致性 |

### 12.4 因子衰减监控（admitted 后的持续体检）

| 机制 | 频率 | 判定 → 动作 |
|---|---|---|
| 滚动 IC 漂移 | 每月（月度数据点） | 近 6 月 mean RankIC < 全样本 50% → `decay_monitor.alert=amber`；连续 3 月 <30% 或变号 → `status=decayed`，组合禁用待评审 |
| 换手/容量漂移 | 每月 | 换手超准入值 1.5× → 成本敏感性重测 |
| 相关性漂移 | 每季度 | 与其他 admitted 因子 \|相关\|突破 0.7 → 组合层面去冗余评审 |
| 事件命中率（事件族因子） | 每周 | `event_feedback.hit` 滚动 20 次 <55% → 事件因子专项复核 |
| 全量重检 | 每半年 | 数据口径变化（data_regime_change 新增）→ 触发受影响因子全量重算 + 版本 diff |

监控任务为确定性脚本（无 LLM），告警走 Orchestrator → Review Agent 专项质询；衰减判定写回 `factor_registry.decay_monitor` 并在因子库浏览器标色。

### 12.5 结果管理与保留策略

| 产物 | 位置 | 保留 | 备份 |
|---|---|---|---|
| metrics/report/review | experiments/<id>/ + MLflow artifacts | 永久（git + 文件） | 异地同步 |
| equity/positions Parquet | experiments/<id>/results/ | 永久（小文件） | 独立介质/异地加密备份（RPO 24h/RTO 1d，A09） |
| 数据集中间缓存 | state/cache/ | TTL 7d，可重建 | 不备份 |
| raw 响应包 | ~/quant-data/raw/ | ≥2 年（审计/重放证据） | 冷备 |
| LLM tracing | mlflow/llm_calls | 12 个月滚动 | 不备份 |
| tasks.sqlite / meta.sqlite | state/ | 永久（小） | 每周 VACUUM INTO |

---

## 13. 量化研究风险控制方案

> 依据：调研 01 §8/§9（数据质量/PIT/幸存者偏差/§4.4 数据可用性陷阱）、调研 02 §9（前视/过拟合/License/平台风险）、调研 04 §5.1（挖掘工具放大过拟合）。本章是"防自欺"的制度化。

### 13.1 数据质量校验（P0）

**每日自动跑、出报告**（`runs/data/<date>/dq_report.md`），规则集见 §5.1.4：
- **结构**：主键唯一、无重复 bar、交易日历对齐（用 baostock/tushare 交易日历，**A 股不同时期节假日安排不同**，不能用周一至周五假设）；
- **数值**：OHLC 关系、volume/amount 合理性、涨跌幅不超板块档位（±10%/±5% ST/±20% 创业板科创板/±30% 北交所；**2020-08 创业板改 20%** 为已知结构变化）；
- **对账**：多源仲裁（baostock 主 vs akshare/tushare），差异 >0.5% 或复权因子不一致 → `dq_issue` 人工复核，**不静默覆盖**；
- **除权除息一致性**：逐除权日验证 `preclose ≈ (前收-每股派现)/(1+送转)`；
- **组合 sanity**：组合收益与基准指数相关性检查（异常低相关 = 数据或复权问题的信号）。

### 13.2 幸存者偏差（P0）

| 机制 | 实现 | 依据 |
|---|---|---|
| 退市股全量保存 | baostock `query_stock_basic`（ipoDate/outDate/status，含退市股）+ tushare 股票历史列表/曾用名；**东财抓取源退市股覆盖不全，不可作唯一来源** | 调研 01 §9.4 |
| universe 快照 | `universe_snapshot(trade_date, code)` 当日真实可交易池（含 ST/上市退市状态/停牌/板块）；回测选股**只用当日快照** | 调研 01 §9.4 |
| 特殊阶段如实反映 | 退市整理期、重组长停牌在 universe 中标记 `suspend`（否则回测会买"实际不可买"的股票） | 调研 01 §9.4 |
| 次新/仙股过滤 | IPO<60 日、退市前期流动性枯竭单独设过滤规则并记录在研究笔记（过滤参数进 inputs_hash） | 调研 01 §9.4 |
| 数据永不删除 | 退市股打 `delist_date` 但保留全部历史 | 调研 01 §8.6 |

### 13.3 未来函数 / 前视偏差（P0）

1. **可见性时间契约（A01 重写，详见 §5.1.9）**：历史 PIT 表**不能补出不存在的历史版本**（只给"最新修订值+披露日"的免费源，新增 `fetched_at`/`_next` 本身无法修复）——按"**市场已公开时间 / 本系统实际获得时间**"双可见性口径登记版本证据（来源文档编号、内容哈希、原始披露时间、修订披露时间、首次采集时间、版本、时区）；只有日期没有披露时刻时采用**保守下一交易时段规则**，不允许默认当日开盘可用；无法恢复原始版本的字段标记 `historical_pit_unverified` 并**阻断**进入声称无前视的历史研究；MVP 可暂不使用财务因子（调研 01 §8.3/§9.5、审计 A01）。
2. **point-in-time 检查器**（`data/dq/pit_check.py`）：对任何数据集构造做静态检查——①所有基本面列声明**可见性口径**（市场已公开/系统获得）并逐查询记录所用口径；②随机抽 20 股验证财报修订链完整性与 `historical_pit_unverified` 阻断生效；③**执行契约检查（A02）**：label 语义与执行假设的差异必须显式声明——Qlib 默认 label `Ref($close,-2)/Ref($close,-1)-1` 是 **T+1 收盘→T+2 收盘收益的预测代理**（审计 S1），禁止标注"与执行假设一致"；改执行假设必须按 §5.3.1 统一执行契约同步改 label 并保留对照说明（调研 02 §2.1/§2.2）。
3. **分类体系历史化**：行业/概念成分只允许查 `industry_map` 区间版本（东财/同花顺概念会改名回溯、申万 2021 改版——分类即数据，调研 01 §4.3）。
4. **Alphalens 输入净化（仅限因子诊断，A03）**：涨跌停/停牌等不可交易样本的剔除**只允许用于 IC/分层等因子诊断**，须以"含/不含"双口径对照并记录剔除数量与理由（否则分层收益虚高/虚低，调研 02 §4.2/§5.2）；**回测净值层面禁止删除未成交样本**美化回测（成交留痕与四分离见本节第 7 条）。
5. **复权价漂移**：只存不复权价+复权因子，查询时动态计算；除权事件回填复权因子并重算全历史（前复权价随时间漂移，绝不能只存前复权价，调研 01 §8.5/§9.3）。
6. **qlib `check_health` 脚本**每日对 qlib_bin 跑健康检查（调研 01 §6.1）。
7. **成交留痕与股票池四分离（A03）**：**信号股票池 / 下单候选 / 成交结果 / 持仓**分开保存（§5.3.2 四类产物），任何一层不得由后一层事后改写；股票池只用**决策时已知条件**生成；成交时逐单检查交易**方向**、**可成交量**、**停牌**状态及**对应历史时期的交易制度**（如创业板 2020-08 前后 10%→20%、ST 档位、T+1 交收）；旧仓未成交（跌停/停牌卖不出）**继续持有并按市价估值**，记录未成交原因（方向受限/可成交量不足/停牌/制度限制），买单失败记录现金留存；**禁止事后删除未成交样本**美化回测；日线无法判定的盘中成交采用**公开说明的保守假设**（对结果不利方向）。现金、数量、费用、净值逐日守恒可对账。

### 13.4 数据窥探与多重检验校正（P0——挖掘时代的头号统计风险）

> 动机：多 Agent 一夜跑几百个实验 = 一夜挖出几百个假因子（调研 04 §5.1）。必须把"试了多少次"计入显著性。

| 机制 | 做法 | 位置 |
|---|---|---|
| **试验计数台账** | 每个 hypothesis/实验进 MLflow（含失败/否决）；`rejection_log` 记录因子签名哈希——"有效因子比例 π₀"估计的分母 | §12.1 |
| **Deflated Sharpe Ratio** | 按试验次数 N、样本偏度/峰度、回测长度对 Sharpe 去膨胀；`dsr > 0 (p<0.05)` 是入库门槛 6 | `backtest/stats/dsr.py`（~100 行自研，Bailey & López de Prado 公式） |
| **White Reality Check / SPA** | 对"最优策略 vs 基准池"做 Bootstrap 数据窥探检验，p<0.10 才认 | `backtest/stats/white_rc.py`（~100 行，pandas+numpy） |
| **预注册（pre-registration）** | 实验在跑之前先写死 spec.yaml（假设/区间/指标/门槛）——事后改指标 = 新实验重新计数 | §8.4 spec.yaml 不可变（git） |
| **族级控制** | 同 family 一周内新实验 >5 个触发 Review 专项质询（防止参数微调刷显著性） | Orchestrator 规则 |
| **OOS 封存** | 训练/开发验证/最终封存测试**三分**；最近 2 年数据默认不进挖掘训练/验证段，只做最终样本外仲裁——冻结代码/参数/数据版本（snapshot_id）登记后由**隔离验证任务**一次性开箱，**访问与结果回流留日志**（A04，§13.5） | DatasetH segments 配置 + 封存访问日志 |

### 13.5 过拟合控制（P0/P1）

- **训练 / 开发验证 / 最终封存测试三分（A04）**：开发迭代（含调参、因子筛选、自动修复）只允许使用训练 + 开发验证；**冻结代码、参数与数据版本（`snapshot_id`）并登记后**才允许运行最终封存测试，由**隔离验证任务**按一次性口径评估，结果**不得回流当前策略的自动修复循环**；**最终测试访问规则与访问日志**：每次访问（谁、何时、读取了哪些结果、结果是否回流）均留日志，开发 Agent/流程不能读取最终测试结果，**已消费的测试窗口不再称为"未见样本"**；切分边界按**标签跨度**留 ≥标签持有期的隔离带（如 T+1→T+2 标签），防止跨界标签泄漏。
- **修复边界（A04/A12）**：实现错误（计算错误、数据处理 bug）可在开发集修复并记录 diff；**策略经济表现不佳记为实验失败**，不允许循环"修到通过"；修复代码错误与修改研究假设**分开登记**；**不能以提高最终测试收益为修复目标（修复 ≠ 提收益）**；失败实验、评审否决记录与参数变体全部落盘，保证每次实验能定位数据分段与试验家族。
- **滚动样本外强制**：一切入库因子必须过 qlib `RollingGen` walk-forward（训练/验证/测试段滚动），oos_sharpe ≥ 0.5×全样本（门槛 4）；
- **Optuna 嵌套在滚动框架内**：超参搜索的适应度只用滚动验证段均值，测试段永不参与搜索（调研 02 §9.4）；
- **分市场阶段回测**：牛/熊/震荡/流动性危机四段分开验证（~30 行切分脚本），≥3/4 段同号（门槛 5）——单阶段有效通常是 regime 拟合；
- **低冗余门槛**：与已入库因子 |相关|<0.7（门槛 7），挖掘工具（RD-Agent/gplearn）产出默认带"挖掘原罪"折扣，通过阈值从严；
- **成本后口径为唯一准绳**：所有对比看扣费扣滑点后的数字（调研 02 §9.4）；
- **Review Agent 制度性唱反调**：评审 prompt 强制包含"列出 3 个该因子失效的理由"（多空辩论结构复用，调研 03 §2.1 TradingAgents 范式）。

### 13.6 数据可用性陷阱（P0——调研 01 的实证教训）

**案例：北向资金 2024-05-13 停发**（调研 01 §4.4）：2024-04-12 沪深港三所宣布调整沪深港通信息披露机制，**2024-05-13 起港交所不再实时披露沪深股通买入/卖出/成交总额**（额度余额 ≥30% 仅显示"额度充足"）；盘后改为披露成交总额/笔数/ETF 成交额/前十大活跃证券，单只持股改为**每季度**披露。

**制度化应对**：
1. `data_regime_change` 表（§5.1.2⑧）**首批必录该断点**及其影响（"盘中北向流向因子失效；改用盘后十大活跃股+季度持股变动"）；
2. **每个数据集 ETL 配置必须声明 `availability_regime`**（可用区间 + 粒度变化点）；因子 inputs_hash 包含该声明——口径变化自动使旧实验失效；
3. **新数据源接入检查单**：①字段是否结构性断发（监管规则/披露机制变化）；②抓取类接口失效预案（akshare 上游改版是常态，调研 01 issue #6143/#7180/#6574）；③口径文档（如 baostock"涨跌幅复权法" vs 通达信"定点复权法"差异，调研 01 §9.3）；
4. 季度性复查 `data_regime_change`（交易所/监管规则跟踪），新断点触发受影响因子重检（§12.4 全量重检）。

### 13.7 平台与合规风险（P1）

| 风险 | 应对 |
|---|---|
| 抓取类数据源 ToS 灰色（akshare/efinance 等） | 个人研究可接受、**不得用于对外商业产品**；核心事实表以 baostock/tushare 双校验（调研 01 §12 免责） |
| 开源项目停滞/换 fork | 一律用续维护 fork（alphalens-reloaded/empyrical-reloaded/zipline-reloaded）或避开；模型名/版本写 tags 不硬编码（调研 02 §9.5、调研 04 §5.4） |
| 单机单点 | **独立介质或异地加密备份 + 恢复验证**（A09，目标 **RPO 24 小时 / RTO 1 天**，以实测确认；同机 MinIO/Time Machine 单副本不代灾难恢复）；experiments/、knowledge/、state/ 全覆盖；SQLite 每周 VACUUM INTO；qlib_bin 可重建（调研 04 §5.5） |
| LLM 幻觉进决策链 | 事件抽取强制 evidence.quote 可追溯；预案卡片所有数字标注来源；LLM 不接触交易执行（§11.5 红线） |
| 成本失控 | dsh-token-meter + MLflow tracing 记账 + 月预算告警（调研 04 §5.7） |
| License 传染 | 见 §16.3 合规清单与隔离建议 |

### 13.8 风险登记表（Risk Register，随实施滚动更新）

| ID | 风险 | 类别 | 可能性 | 影响 | 缓解措施（章节） | 触发监控 |
|---|---|---|---|---|---|---|
| R1 | 数据源接口失效（akshare 上游改版） | 数据 | 高 | 中 | 三源仲裁+兜底（§3.1）；healthcheck 告警（§5.1.3） | 每日 DQ 对账失败率 |
| R2 | 数据结构性断发（如 2024-05-13 北向） | 数据 | 中 | 高 | data_regime_change + availability_regime 声明（§13.6） | 季度 regime 复查 |
| R3 | 前视偏差混入研究 | 研究 | 中 | 极高 | PIT 表 + pit_check + pre-registration（§13.3/§13.4） | 入库门槛 9 + 抽查 |
| R4 | 幸存者偏差（退市股缺失/名单回溯） | 研究 | 中 | 极高 | universe 快照 + 退市股全量（§13.2） | 每季度 universe 完整性审计 |
| R5 | 多重检验/挖掘过拟合 | 研究 | **极高** | 高 | DSR/White RC/试验台账/OOS 封存/族级控制（§13.4/§13.5） | 周通过率异常（>30% 触发质询） |
| R6 | LLM 幻觉进决策链 | Agent | 中 | 高 | evidence.quote 强制 + 数字标来源 + 不接执行（§10.2/§11.5） | Review 抽查 + event_feedback.hit |
| R7 | 重复实验浪费成本 | Agent | 中 | 中 | 四层防线（§8.6） | idem_key 拦截计数 / token 计量 |
| R8 | API/订阅额度超限 | 成本 | 中 | 中 | 用量记录（输入/输出/缓存/重试/电费/订阅分列）+ 额度预留-结算 + 触发即停止新增调用、保留检查点（§6.4/§8，A06/A12） | 月度用量归属报表 |
| R9 | 单机单点（磁盘损坏/误删） | 运维 | 低 | 极高 | 独立介质/异地加密备份 + 恢复演练（RPO 24h/RTO 1d，A09）（§5.1.1/§7.3/§12.5） | 每季度恢复演练 |
| R10 | 依赖停滞/License 变更 | 合规 | 中 | 中 | 续维护 fork 策略 + 年度复审（§13.7/§16.3） | 年度依赖复审 |
| R11 | 模型版本下线（如 MiMo V2.5 2026-10-21） | 平台 | 高 | 低 | 模型名进配置/tags 不硬编码（§7.3） | 供应商公告跟踪 |
| R12 | 自研范围蔓延（重复造轮子） | 流程 | 中 | 高 | 五条硬边界 + 自研准入三条（§4.2/§16.2） | PR 审查（CONTRIBUTING） |

---

## 14. 实施路线图：阶段门 G0–G4（审计 §6 统一定义）

> 依据：审计 §6"分阶段整改与放行条件"。采用**阶段门**：时间仅为计划参考（个人业余投入约 15–25 小时/周），**不按日期自动升级**；**G0–G4 替代原 MVP/P0–P2 阶段定义（已删除，见 §14.3）**。进入 G2 前不得以回测收益作为购买硬件或扩大自动化的依据；进入 G4 前**不设有效因子数量、论文数量或 Agent 数量指标**，**零有效因子允许正常验收**（A04，§12.3）。

### 14.1 阶段门总览（放行证据即工程验收依据，以审计 §7 验收矩阵实测为准）

| 阶段 | 工作 | 放行证据 |
|---|---|---|
| **G0：设计修订** | GPT 制定计划，DeepSeek/MiMo 执行文档与接入任务；完成 A01–A04、A07 的设计决策；统一执行契约与数据可见性契约；删除冲突阶段定义 | 修订后的主设计（本文 v2）、决策记录（`docs/g0-decision-records.md`）、"待实现/已验证"清单 |
| **G1：复盘 MVP** | 持仓导入、日线更新（investment_data 公开数据包初始化 + 定期同步 + 质量验收 + 按需补缺）、质量检查、确定性报告（数据就绪驱动出报）、独立备份 | 连续 10 个交易日有报告；延迟数据明确标记；至少一次恢复成功 |
| **G2：可信回测** | 简单基线、持仓现金账本、手算案例、不可成交案例、数据版本与费用模型（统一执行契约 + 成交四分离） | 固定输入可复现；手算结果在预定义误差内；无已知前视路径 |
| **G3：研究与事件闭环** | 扩展 GPT 计划与 DeepSeek/MiMo 执行协作到公告、事件、因子和复盘 | 固定样本评估、计划可追溯、额度配置和失败恢复通过 |
| **G4：按需扩展** | 因子研究、更多数据、可选编排（含 3 个 MCP server、PostgreSQL 等） | 证明新增组件解决具体瓶颈；给出新增费用、维护工作及替换方法 |

### 14.2 各阶段工作要点（替代原逐周排期；工作量与时间估算待实施时按实际投入重估）

- **G1（复盘 MVP）**：①公开数据包接入（investment_data 固定 release tag + 固定 commit `qlib/validate_archive.py` 校验 + 独立快照目录原子切换，审计 §11.2）；②执行放行检查清单 9 项并形成实际覆盖报告与缺口清单（审计 §11.3）；③补充层（baostock/akshare）抽样对账；④持仓导入 + 日线更新 + DQ 报告；⑤数据就绪驱动的确定性复盘报告（初版标注缺项 → 补齐 → 次日晨间修订，§11.1）；⑥独立介质/异地加密备份 + 恢复演练（RPO 24h / RTO 1d）。**刻意不做**：Agent 自动化、事件抽取 LLM、分钟线、自动因子挖掘。
- **G2（可信回测）**：统一执行契约（§5.3.1）+ 手算对齐案例（收盘信号、跨节假日、不可当日卖出、买卖失败、部分成交、停牌与恢复交易）+ 成交留痕四分离守恒对账（§13.3 第 7 条）+ 训练/开发验证/最终封存测试三分与访问日志（§13.5）+ 少量**预先声明**的简单因子及对照（**零有效因子允许验收**）。
- **G3（研究与事件闭环）**：GPT 计划/研究规范/审核 + DeepSeek/MiMo 执行的协作扩展到公告、事件、因子和复盘；固定人工标注样本评估（漏检/误合并/关键字段准确性，A11）；额度配置、失败恢复、提示注入防护与执行边界验收（A12）。
- **G4（按需扩展）**：因子研究扩展、更多数据（分钟线/付费缺口补全）、可选编排（MCP 三 server、PostgreSQL——出现多客户端复用/多进程写实测需求才引入）、RD-Agent/gplearn 自动因子生产线（受 §13.4/§13.5 防过拟合约束）、rqalpha 隔离复核、LoRA 微调回流。

### 14.3 已删除的旧阶段定义与旧指标（A04/A05/A11）

- ~~MVP（P0，6–8 周）→ 第一阶段（P1）→ 第二阶段（P2）逐周排期~~、~~"第 3 周开通 tushare 2000 积分"~~：与 G0–G4 阶段门冲突，删除；付费源按缺口清单逐项评估（§3.1）。
- ~~"入库因子数量"硬指标~~（"因子库 admitted ≥30"、"admitted ≥10"、"知识库论文 ≥100 篇"等）：删除；**零有效因子允许正常验收**（§12.3）。
- ~~"三层去重滤重率≥40%"~~：删除固定滤重比例指标（A11）；去重与抽取正确性按固定人工标注样本的漏检/误合并/关键字段准确性度量，阈值由用途确定。
- ~~vnpy 实盘执行层~~：不属任何阶段门交付物；若未来考虑，**部署需用户明确确认**并另行放行（调研 04 §5.2）。

---

## 15. 各阶段部署清单、服务与代码仓库（monorepo 目录树）

### 15.1 分阶段部署/服务/仓库总表

| 阶段 | 部署的开源项目 | 创建的服务 | 代码仓库/目录 |
|---|---|---|---|
| **MVP** | baostock、akshare、tushare SDK、pyqlib、alphalens-reloaded、quantstats、MLflow 3.x、DuckDB/SQLite、Ollama+Hermes-4-14B、finance-sentiment-zh-base | `etl`（launchd 定时）、`dq-report`、`mlflow server`、`pipeline_daily_review`（手动触发版）、`apps/daily` | `quant-lab`（monorepo）：data/、factors/、backtest/、apps/daily/、knowledge/ 骨架 |
| **第一阶段** | + DSH、Codex CLI、use_cninfo、simhash、datasketch、text2vec、LlamaIndex、LanceDB/sqlite-vec、mcp python-sdk、Optuna、Streamlit、yfinance、（PostgreSQL） | + `orchestrator`（DSH schedule/webhook）、`tasks-worker`、`ashare-data`/`backtest`/`mlflow-kb` MCP server、`pipeline_news2position`、`pipeline_paper2factor`、`pipeline_daily_review`（无人值守）、`apps/market`/`apps/factors`/`apps/research` | + agents/、pipelines/、orchestration/（含 mcp_servers/）、experiments/ 工作区约定 |
| **第二阶段** | + RD-Agent、gplearn、rqalpha（隔离）、MinIO、DeepKE、RAGFlow（可选）、hikyuu（可选）、DVC（可选） | + `factor-mining-batch`（夜间质量门循环）、`decay-monitor`、`stats-gate`（DSR/White RC）、`snapshot-backup`、PostgreSQL 事实表迁移 | + backtest/stats/、factors/mining/、infra/minio/、`vendor/rqalpha-bridge/`（License 隔离目录） |
| **完整系统** | + vnpy（可选实盘）、微调训练脚本（FinGPT 配方） | + `execution-bridge`（独立风控进程，人工闸门）、`lora-finetune` | + execution/（独立子包，明确红线）、training/ |

**外部账号/付费项**：tushare 积分（120 免费注册 → 2000 积分 200 元/年，P0 第 3 周）；DeepSeek API key（P0）；可选 Codex 订阅（P1）、MiMo Token Plan（P2）；FRED API key（免费，P2）。历史分钟线 2000 元一次性（按需，调研 01 §2.3）。

### 15.2 monorepo 目录树建议（`quant-lab/`，自研薄层全在一个 MIT 仓库）

```
quant-lab/
├── README.md  CONTRIBUTING.md（含 §4.2 五条硬边界）  LICENSE（MIT）
├── pyproject.toml  uv.lock  .gitignore（state/、experiments/ 运行态、config/local.yaml）
├── config/
│   ├── defaults.yaml             # 全局配置（费率/涨跌停档位/门槛阈值引用）
│   └── local.yaml                # API keys（不入 git）
├── data/                         # ①数据层
│   ├── collectors/               #   base.py(Collector 协议) + baostock_/akshare_/tushare_/yfinance_/cninfo_.py
│   ├── etl/                      #   raw→staging→clean 管道、dump_bin 管道、水位管理、多源仲裁
│   ├── dq/                       #   rules.py + pit_check.py + report.py
│   └── schema/                   #   DDL：bar_daily/financial_pit/universe_snapshot/industry_map/
│                                 #         corporate_action/watermark/dq_issue/data_regime_change
├── factors/
│   ├── registry/                 #   <factor_id>.yaml 因子规格（人类可编辑源）
│   ├── impl/                     #   代码因子实现（qlib 兼容接口）+ 单测
│   └── mining/                   #   RD-Agent/gplearn 配置与包装（P2）
├── backtest/
│   ├── configs/                  #   策略 YAML（TopkDropout 等）+ thresholds.yaml（准入门槛）
│   ├── strategies/               #   自定义策略类（qlib StrategyBase 子类）
│   └── stats/                    #   dsr.py / white_rc.py / stability.py / regime_split.py（自研统计薄层）
├── agents/                       # ④Agent 层
│   ├── roles/                    #   9 类角色定义（prompt 模板、写权限、模型档位）
│   ├── prompts/                  #   spec 提取/事件抽取/预案卡片/复盘质询模板（含 JSON 契约）
│   └── executors/                #   DSH/Codex/Hermes/DeepSeek 执行体配置（模型名进配置不硬编码）
├── pipelines/                    # ⑤三大闭环
│   ├── paper2factor.py           #   闭环 A（11 环节 DAG）
│   ├── news2position.py          #   闭环 B（7 环节 DAG）
│   └── daily_review.py           #   闭环 C（16:15 触发；§11.1 时间表）
├── knowledge/                    # ⑥知识库（git）
│   ├── papers/<paper_id>/        #   paper.pdf + summary.md + hypotheses.json
│   ├── postmortems/<ISO-week>/   #   周复盘
│   ├── reviews/<run_id>/         #   评审记录
│   ├── events_kb/                #   事件复盘高价值样本
│   └── playbook.md               #   交易行为准则（git diff 审计）
├── apps/                         # ⑦展示层（Streamlit）
│   ├── daily/  market/  factors/  research/
├── orchestration/                #   任务调度层
│   ├── state_machine.py          #   tasks.sqlite 状态机（§8.3）
│   ├── dedup.py                  #   防重复四层防线（idem_key/缓存/内容去重/租约）
│   └── mcp_servers/              #   ashare-data/ backtest/ mlflow-kb（python-sdk 实现）
├── infra/
│   ├── docker-compose.yml        #   §7.1 服务清单
│   ├── launchd/                  #   com.quantlab.etl.plist / .daily-review.plist / .weekly.plist
│   └── backup.sh                 #   VACUUM INTO + MinIO 快照 + 异地同步
├── experiments/<experiment_id>/  #   §5.4 实验工作区（git worktree exp/<id>）
├── vendor/rqalpha-bridge/        #   rqalpha 隔离桥（§16.3；独立依赖、可整体删除）
└── state/                        #   运行态（不入 git）：tasks.sqlite、cache/、mlruns/、meta.sqlite
```

数据仓库（不入 git）：`~/quant-data/{raw,staging,clean,qlib_bin,duckdb,snapshots}`（§5.1.1）。**git 只存代码、schema、注册表 YAML、知识文本**——这是"文件为事实来源、git 可审计"的前提（调研 04 §3.4）。

### 15.3 部署命令速查（A07 三分类：架构示例 / 待实现接口 / 已验证操作）

> **A07 整改**：命令按三类拆分，与 §7.1 服务清单口径一致。**已验证操作**必须包含依赖锁定、所需文件、版本、预期输出、失败处理及恢复步骤，并从空目录在目标环境执行通过留下日志后才能列入——**目前为空**。同一服务不以容器和裸机重复启动（mlflow、Ollama 各选一种方式，勿同时拉起）。**禁止占位版本号**（如 `minio/minio:RELEASE.2026-xx` 已删除；引入时锁定真实 tag）。

**A. 架构示例（形态示意，未按锁定版本验证，不可直接照抄执行）**：

```bash
# 系统依赖（Apple Silicon 两大坑：Xcode CLT + libomp，调研 02 §7.8）
xcode-select --install
brew install libomp python@3.11 node@24        # LightGBM/OpenMP + Python + DSH(Node 24.x)

# Python 研究栈（示意安装清单；精简为 G1–G2 核心依赖，版本以 uv.lock 为准）
curl -LsSf https://astral.sh/uv/install.sh | sh
uv venv --python 3.11 && uv pip install pyqlib lightgbm pandas duckdb \
   alphalens-reloaded quantstats mlflow optuna baostock akshare
uv lock                                          # uv.lock 入 git
# （tushare/yfinance/fredapi 等不在首期清单：按缺口清单确认后另加）

# 本地推理（裸机 Ollama 与容器 Ollama 二选一，勿同时启动）
brew install ollama && ollama pull <固定模型标签>   # 模型名/量化档写 config
#   Hermes-4-14B GGUF Q5_K_M 文件约 10.51GB（审计 S2）；权重≠总内存（含 KV cache/上下文/运行时）

# mlflow 二选一（勿同时启动）：
mlflow server --backend-store-uri sqlite:///state/mlflow.db \
              --artifacts-uri ./state/mlartifacts --port 5000 &
# 或 docker compose -f infra/docker-compose.yml up -d mlflow
```

**B. 待实现接口（模块/命令尚未交付；实现并锁定版本前不得执行）**：

```bash
uv run python -m data.snapshot fetch --tag <release-tag>      # investment_data archive+manifest 下载登记（URL/asset/SHA-256）
uv run python -m data.snapshot validate --expected-tag <tag>   # 固定 commit 的 validate_archive.py（--expected-tag/--require-publishable）
uv run python -m data.snapshot switch --snapshot <id>          # 完成放行检查后原子切换活动快照
uv run python -m data.etl run --task etl.dq.daily              # DQ 报告 + 放行检查清单（调研 01 §10.3）
uv run python -m reports.daily run --date <d>                  # 数据就绪驱动的复盘报告（§11.1）
uv run python -m backup.run --verify                           # 独立介质/异地加密备份 + 恢复演练（RPO 24h/RTO 1d）
uv run qlib check_health                                       # qlib 健康检查（调研 01 §6.1）
uv run qlib run --config backtest/configs/bench_alpha158_lgbm.yaml   # 本机基准计时（结果记入 knowledge/reviews/benchmarks.md）
cp infra/launchd/com.quantlab.*.plist ~/Library/LaunchAgents/  # 定时任务（launchd；漏跑补执行需验证一次）
launchctl load ~/Library/LaunchAgents/com.quantlab.etl.plist
npm i -g @deepseek-ai/dsh && dsh --version                      # G3 编排运行时（MIT）
```

（Qlib 上游工作流入口为 `qrun`，命令须按锁定版本验证（审计 S3）。）

**C. 已验证操作（工程验证清单）**：

```
（空 —— 截至 2026-09-23，尚无任何命令在目标环境从空目录执行验证并留下日志；
  审计 A07 明确"当前不能标记此项已通过"。
  工程验收以审计 §7 验收矩阵实测为准，详见 docs/g0-decision-records.md §3。）
```

---

## 16. 降低重复开发的总体原则

### 16.1 开源优先决策流程（新需求的标准走法）

```
需求出现
  │
  ├─ 1. 检索：GitHub/awesome-quant/调研 01–04 结论 → 找到候选？
  │        │ 否 → 走 §16.2 自研准入标准（三条全过才自研，否则降级为配置/脚本/放弃）
  │        │ 是 ↓
  ├─ 2. 体检四问：①维护活跃（1 月内 push=活跃；>6 月慎用）？②License 合规（§16.3 绿区）？
  │        │           ③stars/社区（>500 或官方背书）？④A 股/中文适配成本可控？
  │        │ 任一否 → 换 fork/备选/走自研准入
  │        │ 全是 ↓
  ├─ 3. 复用形态选择：直接复用（pip 安装）＞ 配置化扩展（YAML/子类）＞ 薄封装（适配器）＞ 取思路重写
  │        ↓
  ├─ 4. 集成纪律：锁版本（uv.lock）、包一层自己的接口（可整体替换）、写集成测试、License 登记
  │        ↓
  └─ 5. 复审：每年一次依赖复审（维护状态/License 变更/替代品出现）
```

### 16.2 自研准入标准（三条全过才允许；与 §4.3 一致）

1. **不存在性**：附检索证据（关键词/对比表）说明成熟开源确实不满足；
2. **差异化价值**：属于三类之一——个人数据资产正确性（PIT/对账/退市股）、个人交易闭环（持仓/复盘/预案）、个人知识积累（因子库/知识库）；
3. **薄层性**：<1k 行、无框架化倾向、可被未来开源方案整体替换。

**降级选项优先**：改配置 ＞ 写一次性脚本 ＞ 给上游提 PR/issue ＞ 最后才自研模块。自研模块必须先写"替换预案"（哪天开源方案成熟时怎么换掉）。

### 16.3 License 合规清单（A08 重写：按条款 + 使用方式判断；条款原文未逐条重核者**待核实**）

**判断框架（A08）**：

1. **按使用方式分别判断**：①个人本地自用；②代码公开发布（分发/衍生作品）；③对外网络服务——同一 License 在三种方式下义务不同，逐个依赖登记**版本、许可证原文链接、实际使用方式**后再定结论。
2. **GPL**：个人本地使用、修改**不因使用本身**要求公开个人代码（GNU FAQ S4，**待核实**完整条款）；分发衍生作品才触发开源义务——**GPL 个人本地使用工具可回列候选**（如 backtrader）。
3. **AGPL**：按条款区分**修改、网络交互与源代码提供义务**（AGPLv3 §13 等，S5，**待核实**），不能简单等同"分发"；对外网络服务前必须评估向用户提供源代码的义务。
4. **无 License ≠ 可复制**：默认保留所有权利，不复制代码，只读思路后自行重写。
5. **附加非商业限制**（如 rqalpha）须检查**具体用途**；**进程隔离不自动消除许可义务**（仅便于替换/删除）。
6. **四类许可分开记录**：①开源代码许可；②模型许可；③数据使用条款；④服务订阅条款（Claude Code 无开源 License、抓取数据 ToS、tushare 订阅条款等均单独记录）。
7. **个人使用不自动获得上游数据抓取或再分发授权。**

| 区 | License / 条款 | 项目 | 使用规则（按使用方式） |
|---|---|---|---|
| 🟢 宽松 | MIT / BSD-3 / BSD | qlib、RD-Agent、DSH、Codex、MiMo Code、akshare、efinance、easyquotation、Optuna、LangGraph、CrewAI、MetaGPT、OpenHands、simhash、datasketch、text2vec(部分 Apache)、DeepKE、OneKE、use_cninfo、FinGPT、FinNLP、LlamaIndex、mcp python-sdk、vnpy、tushare(BSD-3)、gplearn(BSD-3)、baostock(BSD) | 本地自用/公开发布/对外服务三种方式均无 copyleft 障碍；保留版权声明 |
| 🟢 宽松 | Apache-2.0 | MLflow、quantstats、alphalens-reloaded、empyrical-reloaded、hikyuu、yfinance、fredapi、FinRobot、RAGFlow、TradingAgents、Agno、smolagents、Aider、Kedro、DVC、ProsusAI/finbert、DISC-FinLLM | 三种方式均可；公开发布时保留 NOTICE |
| 🟡 有条件 | **GPL-3.0**：backtrader | backtrader | **重新列为 qlib 替代候选（A08/S4/S6）**：个人本地自用与修改不要求公开个人代码；仅当 qlib A 股适配测试不通过时启用、不并行维护两套引擎；**公开发布/分发衍生作品时**须按 GPL 开源或隔离停用。A 股规则全需自建 + 半停滞（2024-08）为保留意见 |
| 🟡 有条件 | **AGPL-3.0**：backtesting.py | backtesting.py | **分情形（A08/S5）**：本地一次性试验、不分发、不提供网络服务 → 可用；**修改** → 承担对应条款义务；**对外网络交互** → 按 AGPLv3 §13 评估向用户提供源代码义务。默认不入正式依赖；使用前人工核对条款原文（**待核实**） |
| 🟡 有条件 | rqalpha：Apache-2.0 + **仅非商业**（GitHub SPDX 为 NOASSERTION） | rqalpha | 按**具体用途**判断（个人研究属非商业，**待核实**条款原文）；隔离在 `vendor/rqalpha-bridge/`（独立依赖、进程级调用、可整体删除）；一旦商业化先取得书面授权或移除（调研 02 §2.1.2） |
| 🟡 有条件 | vectorbt 自定义（NOASSERTION） | vectorbt | 默认不引入；需要时先人工审查条款 |
| 🟡 有条件 | AutoGen 仓库 CC-BY-4.0（PyPI 历史标注 MIT，口径不一） | AutoGen | 默认不用（亦违反单编排原则）；若用先核对 LICENSE |
| 🟡 有条件 | Hermes 系列随底座：Hermes-4-14B=Apache-2.0（Qwen3-14B 底座）；70B/405B 基于 Llama-3.1 | Hermes-4 家族 | **模型许可单独记录**（四类之②）：14B 可用；更大尺寸逐模型核对 HF card（调研 04 §2.1.1） |
| 🔴 不复制 | **无 License（默认保留所有权利）**：alphagen、Ashare、FinSpider、qlib-mcp、finance-mcp 等 | alphagen 等 | **无 License ≠ 可复制**：零代码复制，只读论文/思路后自行重写（调研 02 §3.1.2） |
| 🔴 不依赖 | 专有：Claude Code（无开源 License，npm 分发） | Claude Code | **服务/订阅条款单独记录**（四类之④）：个人工具可用；不作为系统依赖、不内嵌其产物 |
| ⚪ 数据条款 | 抓取类（akshare/efinance/qstock/Ashare/easyquotation）灰色 ToS；tushare/baostock 官方服务条款 | — | **数据使用条款单独记录**（四类之③）：个人研究惯例可容忍；个人使用不自动获得再分发授权，**不得用于对外商业产品**（调研 01 §12 免责声明） |

**隔离与登记建议**：①🟡需隔离代码进 `vendor/` + 独立依赖文件，主干仅经 CLI/子进程桥接（进程隔离不消除许可义务，仅便于替换/删除）；②🔴代码零复制，思路借鉴写进 `knowledge/reviews/` 并注明"重写实现"；③`CONTRIBUTING.md` 要求每个新依赖 PR 附 License 登记行（**版本、许可证原文链接、使用方式**三项齐全，四类许可分开登记）；④每年复审一次（License 变更/项目迁移导致条款变化，如 dvc 组织迁移、AutoGen 口径不一均为先例）；⑤未来若商业化：先过 rqalpha/抓取数据两道闸，再评估 vectorbt/AutoGen，以及 backtrader 等 GPL 工具的分发义务。

---

## 附录 A：关键数据核实记录（2026-09-22 快照）

> 完整来源 URL 见调研 01 §12、调研 02 §10、调研 03 §8、调研 04 §6。此表仅摘录本文引用的关键数值。

| 对象 | 核实值 | 来源 |
|---|---|---|
| akshare | 22,692★ / MIT / push 2026-09-20 | GitHub API |
| baostock | PyPI 0.9.4 @ 2026-09-21 / BSD / 自有 TCP 服务器 | PyPI |
| tushare | 15,409★ / BSD-3 / SDK push 2024-03-13（服务端运营中，ICP 2026） | GitHub API / tushare.pro |
| tushare 价格 | 2000 积分=200 元/年（200 次/分）；5000=500 元/年；10000=1000 元/年；历史分钟线 2000 元一次性；实时分钟 1000 元/月；美股 2000 元/年；新闻/公告 1000 元/年 | tushare.pro/document/2?doc_id=290 |
| qlib | 48,746★ / MIT / push 2026-09-22（活跃） | GitHub API |
| RD-Agent | 14,711★ / MIT（论文 arXiv:2505.15155） | GitHub API |
| MLflow | 28,095★ / Apache-2.0 / v3.16.x | GitHub API |
| Optuna | 14,832★ / MIT | GitHub API |
| alphalens / -reloaded | 4,449★（停滞 2024-02）/ 653★（低频）/ Apache-2.0 | GitHub API |
| quantstats | 7,651★ / Apache-2.0 / push 2026-07-20 | GitHub API |
| rqalpha | 6,787★ / Apache-2.0+**仅非商业** / push 2026-09-22 | GitHub API + LICENSE 原文 |
| backtrader / backtesting.py | 23,302★ GPL-3.0 半停滞 / 8,982★ **AGPL-3.0** | GitHub API |
| alphagen / Ashare | 1,242★ **无 License** / 3,869★ **无 License** | GitHub API（license: null） |
| TradingAgents | 108,032★（调研 03）/ 106,834★（调研 04，同日不同镜像）/ Apache-2.0 | GitHub API / ecosyste.ms |
| DSH | 232,442★ / MIT / npm @deepseek-ai/dsh 0.1.5-rc.2 / Node 24.x | repos.ecosyste.ms + npm |
| Codex | 125,873★ / Apache-2.0 / Rust / push 2026-09-22 | GitHub API |
| MiMo Code | 13,338★ / MIT / V2.5 于 2026-10-21 下线 | GitHub API + mimo.mi.com |
| Claude Code | 147,557★ / **无开源 License** | GitHub API |
| Hermes-4-14B | Apache-2.0（base Qwen3-14B）；BF16 28GB / FP8 14GB / GGUF Q5_K_M 约 10.51GB（审计 S2，bartowski 量化发布页） | HF model card + llm.co + 审计 S2 |
| mcp python-sdk | 24,350★ / MIT / v2.2.0 | repos.ecosyste.ms |
| FinGPT / FinRobot / FinNLP | 21,272★ MIT / 8,049★ Apache-2.0 / 1,487★ MIT | GitHub API |
| DeepSeek API | flash：$0.15（未命中）/$0.003（命中）/$0.6（输出）每 M；v4-pro：$0.66/$0.022/$1.98；高峰翻倍、off-peak 5 折 | api-docs.deepseek.com |
| Codex 订阅/API | Plus $20 / Pro $100 / Pro 20x $200；API Luna $0.20/$1.20 → Astra $10/$50 每 M | help.openai.com + nops（2026-09） |
| 北向资金规则变化 | 2024-04-12 宣布、**2024-05-13 起**不再实时披露沪深股通买卖总额；持股改季度披露 | HKEX 通告 + 证券时报e公司 |
| 数据容量 | 全 A 30 年日线 Parquet <2GB；5 分钟 15 年 10–30GB；1 分钟 5 年数十 GB | 调研 01 §7 估算 |
| 成本口径 | 不给笼统月费估算；按输入/输出/缓存/重试/电费/已有订阅分项记录，额度按已有服务配置 | 主设计 §6.4（审计 A06） |

> **免责与快照声明**：stars / 价格 / 版本为 2026-09-22 快照，重定价与版本迭代频繁（MiMo V2.5 下线即先例），落地前请按原文链接复核；抓取类数据源存在 ToS 灰色地带，仅限个人研究使用。

## 附录 B：阶段门索引（G0–G4 速查，替代原 P0/P1/P2 优先级索引）

> 依据审计 §6。正文遗留的 P0/P1/P2 标记仅表示组件**引入顺序**（P0≈G1–G2、P1≈G3、P2≈G4），不再是阶段定义。

| 阶段门 | 事项（章节） |
|---|---|
| **G0：设计修订** | 主设计 v2 修订记录（文首）；A01–A12 整改落实（§5.1.9/§5.3/§6/§7/§8/§11/§12–14/§15.3/§16.3）；决策记录与"待实现/已验证"清单（`docs/g0-decision-records.md`） |
| **G1：复盘 MVP** | investment_data 公开数据包接入 + 放行检查（§5.1.10）；补充层抽样对账（§5.1）；DQ 报告（§13.1）；数据就绪驱动复盘报告（§11）；独立介质/异地加密备份 + 恢复演练（§5.1.1/§7.3）；tasks.sqlite 状态机（§5.6/§8.3） |
| **G2：可信回测** | qlib 单引擎基座 + 费用/涨跌停校准（§5.3）；统一执行契约（§5.3.1）；手算对齐与成交守恒/四分离（§5.3.2/§13.3）；少量预先声明简单因子 + 准入卡（§5.2/§12.3，零有效因子允许）；MLflow 一回测一 run（§12.2）；样本外封存三分 + 访问日志（§13.4/§13.5） |
| **G3：研究与事件闭环** | GPT 计划/研究规范/审核 + DeepSeek/MiMo 执行协作（§8.2/§8.11）；三大闭环全自动化（§9/§10/§11）；事件库 + 事件契约（§10.2）；因子库 schema + 血缘（§12.1）；衰减监控（§12.4）；trading_journal（§11.5）；Streamlit 多页（§5.7）；Optuna（§5.2）；use_cninfo/LlamaIndex/LanceDB（§10）；多重检验校正（§13.4）；Hermes-4-14B 本地（§6.1/§6.3） |
| **G4：按需扩展** | RD-Agent/gplearn 挖掘批处理（§9.3，受 A04 约束）；3 个 MCP server（§8.7）；PostgreSQL（§7.1，证明需要多进程写服务后）；MinIO/DVC（§7.1）；rqalpha 隔离复核（§5.3）；DeepKE/RAGFlow/hikyuu（§2）；事件因子研究线；LoRA 微调回流（§6.3）；vnpy 实盘（需用户明确确认，另行放行） |

---

*本文档基于 `docs/research/01-data-layer.md`、`02-backtest-factor.md`、`03-news-events.md`、`04-agent-orchestration.md` 整合撰写；所有技术数据核实于 2026-09-22。文档随实施进度修订，修订记录走 git。*







