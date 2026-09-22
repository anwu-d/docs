# 个人 A 股量化研究系统 · 开源选型调研 02：回测与因子研究

> **调研对象**：部署于 Mac mini（Apple Silicon）的个人 A 股量化研究系统
> **选型原则**：尽量复用现成开源项目，不重复造轮子
> **调研方法**：所有项目的 stars / License / 维护状态均于 **2026-09-22** 通过 GitHub API（`api.github.com/repos/<owner>/<repo>`）逐一核实；功能描述结合官方文档、源码（如 `microsoft/qlib` 的 `qlib/backtest/exchange.py`）与社区资料交叉验证。文末附全部来源 URL。
> **说明**：stars 数与"最后推送日期（pushed_at）"为调研时点快照，之后会变化。

---

## 0. 执行摘要（TL;DR）

| 层次 | 推荐选型 | 备选 |
| -- | -- | -- |
| **主回测 / 因子研究基座** | **Microsoft qlib**（MIT，48.7k stars，活跃） | rqalpha（注意非商用条款） |
| **单因子分析（IC/分层/换手）** | **Alphalens → alphalens-reloaded** | qlib 内置 `analysis_model` |
| **组合绩效与风险指标** | **quantstats** + qlib 内置 `risk_analysis` | empyrical-reloaded |
| **Alpha 自动挖掘** | **RD-Agent（qlib 生态）**；表达式因子管道用 qlib 表达式引擎 | alphagen（RL）、gplearn（GP） |
| **实验追踪** | **MLflow**（qlib QlibRecorder 原生基于它） | W&B（图形体验更好） |
| **超参搜索** | **Optuna** | W&B Sweeps |
| **数据/流水线版本化** | DVC（可选）+ Kedro（可选，工程化流水线） | Git + 自建脚本 |
| **策略原型快速验证** | backtesting.py（单标的）/ vectorbt（组合向量化） | — |

**结论**：以 **qlib 为研究基座（数据 + 因子 + 模型 + 回测 + 实验记录）**，叠加 **alphalens-reloaded（单因子体检）、quantstats（组合报告）、MLflow + Optuna（实验与调参）、RD-Agent（LLM 因子挖掘）**，即可在 Mac mini（Apple Silicon）上构成完整的个人 A 股量化研究闭环，全部组件开源、无需重复造轮子。

---

## 1. 评估维度说明

对每个项目按以下维度评估：

1. **GitHub 地址 / stars / 维护状态**（GitHub API 核实，含最后推送时间）；
2. **License**（个人研究通常宽松，但 GPL/AGPL/无 License/非商用条款影响后续商用与分发）；
3. **A 股支持**细分为四项：
   - 前/后复权处理；
   - 涨跌停限制；
   - T+1 交收制度；
   - 手续费 / 印花税（及最低佣金、滑点）模型；
4. **"直接复用 vs 需改造"判断**；
5. **推荐理由**。

维护状态约定：`pushed_at` 距调研日 1 个月内 = **活跃**；1–6 个月 = **低频维护**；6–24 个月 = **半停滞**；>24 个月或仓库已归档 = **停滞**。

---

## 2. 因子研究与回测框架

### 2.1 项目逐项评估

#### 2.1.1 Microsoft qlib —— ★ 主基座推荐

- **GitHub**：https://github.com/microsoft/qlib
- **Stars**：48,746（2026-09-22）；Forks 7,709
- **维护状态**：**非常活跃**（最后推送 2026-09-22；持续合并 PR，RD-Agent 联动更新）
- **License**：MIT
- **定位**：AI-Oriented Quant Investment Platform，覆盖"数据→因子/数据集→模型→回测→分析→在线服务"全链路
- **A 股支持**：
  - **复权**：数据以"原始价 + `$factor` 复权因子"表达（`exchange.py` 中 `$factor` 用于复权还原与整手取整；缺 factor 时自动退化为"复权价撮合"并告警）。研究侧可用表达式自行构造复权价；撮合侧按真实价 + factor 还原，天然避免复权价成交的失真。
  - **涨跌停**：`Exchange(limit_threshold=...)` 原生支持——float（如 0.1，按 `$change` 判断）或自定义表达式（区分买入/卖出涨停），命中限制的标的当日不可交易；`C.region=REG_CN/REG_TW` 时未设置会主动告警（源码核实）。
  - **T+1**：日频工作流天然满足"T 日信号 → T+1 日调仓成交"的时序；但撮合层（`exchange.py`）**无显式"当日买入不可当日卖"持仓锁**——对每日一次调仓的 TopK 类策略无实际影响，若做日内/分钟级嵌套执行需在策略/持仓层自行约束。
  - **费用**：`open_cost` / `close_cost`（可分别模拟买入佣金与卖出印花税+佣金）、`min_cost`（最低 5 元）、`impact_cost`（冲击成本/滑点，建议 0.1%）、`volume_threshold`（容量/成交量约束）均原生支持（源码核实）。
  - **其他 A 股细节**：`trade_unit=100`（整手交易，注释明确写 "trade unit, 100 for China A market"）、停牌（`$close` 为 NaN 即停牌不可交易）、批量涨跌停可用 `extra_quote` 补充 ETF 等标的。
- **复用判断**：**直接复用为主 + 少量改造**（数据源接入、label/费用参数按 A 股校准；严格 T+1 需小改）
- **推荐理由**：功能覆盖最全、社区最大（48.7k stars）、MIT 许可、与 RD-Agent/MLflow 原生集成、内置 Alpha158/Alpha360 与 20+ 基准模型，详见第 5 章重点评估。

#### 2.1.2 RQAlpha（Ricequant）

- **GitHub**：https://github.com/ricequant/rqalpha
- **Stars**：6,787；Forks 1,796
- **维护状态**：**活跃**（最后推送 2026-09-22，open issues 仅 31）
- **License**：**自定义双轨**——非商业用途按 Apache-2.0；**任何商业用途需米筐科技（Ricequant）书面授权**（LICENSE 原文核实："未经米筐科技授权，任何个人不得出于任何商业目的使用本软件……任何法人或其他组织不得出于任何目的使用本软件"）
- **A 股支持**（A 股原生，中文文档 rqalpha.readthedocs.io/zh_CN）：
  - 复权：与数据源绑定（RQData/自建 bundle），支持分红除权处理；
  - 涨跌停：撮合引擎（`sys_simulation`）+ 风控（`sys_risk`）支持涨跌停拒绝成交（依赖数据中的涨跌停价）；
  - T+1：股票持仓模型区分"可卖数量（closable）"，**原生 T+1**；
  - 费用：`sys_transaction_cost` mod 实现股票/期货**佣金、印花税**等税费计算；
  - 事件驱动撮合（bar/tick），支持 Mod Hook 扩展。
- **复用判断**：**可直接复用**（做 A 股事件驱动回测/模拟撮合），但**商用受限**
- **推荐理由**：A 股交易规则建模最贴近国内实盘（T+1/涨跌停/印花税开箱即用）、中文文档与社区（QQ 群）、Mod 机制灵活。**风险**：License 非标准，若未来产品化/对外服务需先取得授权；数据侧 RQData 为商业数据源（可换自建数据）。适合做 qlib 的"高保真撮合验证"补充层。

#### 2.1.3 backtrader

- **GitHub**：https://github.com/mementum/backtrader
- **Stars**：23,302；Forks 5,289
- **维护状态**：**半停滞**（最后推送 2024-08-19；issues 已关闭，仅偶发合并 PR）
- **License**：**GPL-3.0**（传染性：分发衍生作品须开源）
- **A 股支持**：
  - 复权：交给数据源（自备前/后复权数据）；
  - 涨跌停：**无原生支持**（需自定义 Broker/数据过滤）；
  - T+1：**无原生支持**（需自定义 sizer/broker 逻辑）；
  - 费用：`CommissionInfo` 支持佣金/印花税/最低佣金，可自定义。
- **复用判断**：**需中等改造**（A 股交易规则全部自建），且维护停滞 + GPL
- **推荐理由**：经典事件驱动框架、教程与第三方资料极多、指标/分析器（Analyzer）丰富，适合学习与快速搭建事件驱动原型。**不建议作为长期基座**：停滞 + GPL + A 股规则缺失。

#### 2.1.4 vnpy（VeighNa）

- **GitHub**：https://github.com/vnpy/vnpy
- **Stars**：45,502；Forks 12,486
- **维护状态**：**活跃**（最后推送 2026-09-13）
- **License**：MIT（社区版；vnpy_pro 等周边为商业产品）
- **A 股支持**：
  - 定位是**交易/实盘接入平台**（CTA、期权、多因子等应用层在 vnpy_org 各仓库），回测为事件驱动 bar 级（vnpy_ctabacktester / vnpy_factor）；
  - 费率/滑点/合约乘数/保证金可配；**T+1、涨跌停需在策略或子引擎自行处理**（回测引擎偏期货设计）；
  - 复权依赖数据源（RQData/自建数据库）。
- **复用判断**：**部分复用**——研究/回测用 qlib 更合适；vnpy 的价值在**实盘柜台接口**（CTP、券商接口等）与模拟交易
- **推荐理由**：国内实盘生态最完整（45.5k stars），若系统未来要接实盘，用 vnpy 做交易执行层、qlib 做研究层是常见组合。

#### 2.1.5 zipline-reloaded

- **GitHub**：https://github.com/stefan-jansen/zipline-reloaded（quantopian/zipline 的社区续maintained fork）
- **Stars**：1,941（原 quantopian/zipline 20,108 但已停更于 2024-02）
- **维护状态**：**低频维护**（最后推送 2026-01-06）
- **License**：Apache-2.0
- **A 股支持**：
  - **美股中心设计**（交易日历、bundle、佣金/滑点模型如 `Commission`、`VolumeShareSlippage` 可配）；
  - 复权/涨跌停/T+1 **均无原生支持**；A 股需自建 bundle（可参考第三方 https://github.com/azluck/zipline 的 tdx bundle）、自定义交易日历、魔改撮合；
  - Python 3.8+ 兼容性经社区维护恢复（zipline-reloaded 的主要贡献即现代化适配）。
- **复用判断**：**需大量改造**（A 股全套规则 + 数据管道自建）
- **推荐理由**：算法框架经典（`initialize/handle_data` 范式）、与 `zipline-reloaded`/`alphalens-reloaded`/`pyfolio-reloaded`/`empyrical-reloaded`（Stefan Jansen 全家桶）配套，配合其著作《Machine Learning for Algorithmic Trading》使用。对 A 股个人研究**不如 qlib 划算**。

#### 2.1.6 Hikyuu

- **GitHub**：https://github.com/fasiondog/hikyuu
- **Stars**：3,522；Forks 833
- **维护状态**：**非常活跃**（最后推送 2026-09-22，open issues 仅 4）
- **License**：Apache-2.0
- **A 股支持**（A 股原生，**中文文档** hikyuu.readthedocs.io/zh-cn）：
  - 复权：内建前/后复权（复权因子）与除权处理；
  - 交易规则：日线级系统化交易（System/Portfolio/TradeManager），"延迟一日建仓"机制贴近 T+1 信号执行；**涨跌停/ST 需用指标条件自行过滤**；
  - 费用：可配佣金/印花税（TradeManager 费用参数）；
  - C++ 内核 + Python 接口，全市场日线级组合回测速度极快；数据源支持通达信/腾讯/新浪等免费源。
- **复用判断**：**直接复用**（日线系统化/组合级快速回测、传统技术指标选股）
- **推荐理由**：A 股原生 + 中文 + 高性能，非常适合传统规则型选股/择时的快速批量验证；但**机器学习因子研究管线（数据集/模型管理）弱于 qlib**。定位为 qlib 的"极速验证沙盒"或传统策略引擎。

#### 2.1.7 Alphalens / alphalens-reloaded

- **GitHub**：https://github.com/quantopian/alphalens （原版）；https://github.com/stefan-jansen/alphalens-reloaded （续维护 fork）
- **Stars**：alphalens 4,449；alphalens-reloaded 653
- **维护状态**：alphalens **停滞**（最后推送 2024-02-12，Quantopian 已关闭）；alphalens-reloaded **低频维护**（最后推送 2025-12-15）
- **License**：均为 Apache-2.0
- **A 股支持**：**与市场无关**（输入 = 因子值 DataFrame + 收益率序列）；A 股的复权收益、涨跌停过滤需在输入数据层处理（把涨跌停当日的标的收益置 NaN 或剔除）
- **复用判断**：**直接复用**（推荐装 `alphalens-reloaded`）
- **推荐理由**：单因子分析事实标准——IC/分层（Quantile）收益/换手率（Turnover Analysis）/因子衰减（Mean Period Wise Return）/信息系数分析一键出图，是"因子体检"标配。

#### 2.1.8 empyrical / empyrical-reloaded

- **GitHub**：https://github.com/quantopian/empyrical ；https://github.com/stefan-jansen/empyrical-reloaded
- **Stars**：empyrical 1,513；empyrical-reloaded 121（ecosyste.ms 核实）
- **维护状态**：empyrical **半停滞**（最后推送 2024-07-26）；empyrical-reloaded **低频维护**（约 2025 年底仍有推送，发布至 0.5.9）
- **License**：均为 Apache-2.0
- **A 股支持**：纯收益序列→指标计算（Sharpe/Sortino/Max Drawdown/Calmar/Omega/尾部风险等），**与市场无关**；年化因子需按 A 股 244 交易日调整
- **复用判断**：**直接复用**（作为函数库被 zipline/pyfolio/quantstats 引用）
- **推荐理由**：轻量、指标函数齐全，适合嵌入自定义研究代码做滚动/分段指标计算；但单独使用偏底层，报告呈现不如 quantstats。

#### 2.1.9 quantstats

- **GitHub**：https://github.com/ranaroussi/quantstats
- **Stars**：7,651；Forks 1,235
- **维护状态**：**活跃**（最后推送 2026-07-20）
- **License**：Apache-2.0
- **A 股支持**：纯收益序列分析，**与市场无关**（基准可用 000300/000905 收益序列）；默认 252 年化交易日需改为 244
- **复用判断**：**直接复用**
- **推荐理由**：`qs.reports.html()` 一键生成 tearsheet（Sharpe/Sortino/Max DD/Calmar/尾部/月度热力图/滚动指标），个人研究"复盘报告"性价比最高；社区活跃，另有 quantstats-reloaded 等分支可换。

#### 2.1.10 backtesting.py

- **GitHub**：https://github.com/kernc/backtesting.py
- **Stars**：8,982；Forks 1,534
- **维护状态**：**活跃**（最后推送 2026-08-05）
- **License**：**AGPL-3.0**（传染性最强，网络服务也算分发）
- **A 股支持**：**单标的**回测（一个品种 OHLCV），佣金/滑点可配；**无复权/涨跌停/T+1 概念**（需数据层自理）
- **复用判断**：**直接复用**（仅限单标的择时/技术策略原型）
- **推荐理由**：API 极简、内置指标与热力图/滚动优化（`backtesting.lib`+ optuna 集成示例），几分钟验证一个择时想法。注意 AGPL 与单标的局限。

#### 2.1.11 vectorbt（补充推荐）

- **GitHub**：https://github.com/polakowo/vectorbt
- **Stars**：9,152
- **维护状态**：**活跃**（最后推送 2026-09-17）
- **License**：**自定义（NOASSERTION）**——非标准开源许可，商用前需审查条款；另有商业版 vectorbt PRO
- **A 股支持**：向量化组合回测，费用可自定义；涨跌停/T+1 无原生（信号层过滤）
- **复用判断**：**直接复用**（大规模参数扫描/组合级向量化回测）
- **推荐理由**：NumPy 向量化引擎，秒级跑完上千参数组合，适合策略参数空间粗筛；许可与 A 股规则需注意。

### 2.2 框架对比总表

| 项目 | GitHub | Stars | 维护状态 | License | 引擎类型 | 前/后复权 | 涨跌停 | T+1 | 手续费/印花税 | 复用判断 | 推荐度 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| **qlib** | microsoft/qlib | 48,746 | 活跃 | MIT | 向量化信号 + 半事件执行 | ✅ `$factor` 复权因子 | ✅ `limit_threshold` | ⚠️ 时序满足，无持仓锁 | ✅ open/close/min/impact cost | 直接复用+少量改造 | ★★★★★ 主基座 |
| **RQAlpha** | ricequant/rqalpha | 6,787 | 活跃 | Apache-2.0（**仅非商业**） | 事件驱动 | ✅ 数据层 | ✅ 撮合/风控 | ✅ 原生（closable） | ✅ 佣金+印花税 mod | 直接复用（商用受限） | ★★★★ |
| **Hikyuu** | fasiondog/hikyuu | 3,522 | 活跃 | Apache-2.0 | 事件驱动（C++ 内核） | ✅ 前/后复权 | ⚠️ 条件过滤 | ⚠️ 延迟一日成交近似 | ✅ 可配 | 直接复用 | ★★★★ |
| **vnpy** | vnpy/vnpy | 45,502 | 活跃 | MIT | 事件驱动 | ✅ 数据层 | ⚠️ 需自处理 | ⚠️ 需自处理 | ✅ 费率/滑点/保证金 | 部分复用（实盘层） | ★★★☆ |
| **backtrader** | mementum/backtrader | 23,302 | 半停滞(2024-08) | **GPL-3.0** | 事件驱动 | ⚠️ 数据自理 | ❌ | ❌ | ✅ CommissionInfo | 需中等改造 | ★★★ |
| **zipline-reloaded** | stefan-jansen/zipline-reloaded | 1,941 | 低频维护 | Apache-2.0 | 事件驱动 | ❌ | ❌ | ❌ | ✅ 佣金/滑点模型 | 需大量改造 | ★★☆ |
| **backtesting.py** | kernc/backtesting.py | 8,982 | 活跃 | **AGPL-3.0** | 混合（单标的） | ❌ | ❌ | ❌ | ✅ 基础 | 直接复用（原型） | ★★★ |
| **vectorbt** | polakowo/vectorbt | 9,152 | 活跃 | 自定义(NOASSERTION) | 向量化 | ⚠️ 数据自理 | ❌ | ❌ | ✅ 可定制 | 直接复用（参数扫描） | ★★★ |
| **Alphalens(-reloaded)** | quantopian/alphalens / stefan-jansen/alphalens-reloaded | 4,449 / 653 | 停滞 / 低频维护 | Apache-2.0 | 因子分析（非回测） | 输入自理 | 输入自理 | 不适用 | 不适用 | 直接复用 | ★★★★★ 单因子标配 |
| **empyrical(-reloaded)** | quantopian/empyrical / stefan-jansen/empyrical-reloaded | 1,513 / 121 | 半停滞 / 低频维护 | Apache-2.0 | 指标库 | 不适用 | 不适用 | 不适用 | 不适用 | 直接复用 | ★★★☆ |
| **quantstats** | ranaroussi/quantstats | 7,651 | 活跃 | Apache-2.0 | 绩效报告 | 不适用 | 不适用 | 不适用 | 不适用 | 直接复用 | ★★★★★ 报告标配 |

图例：✅ 原生支持；⚠️ 部分/需配置；❌ 无，需改造。

---

## 3. Alpha / 因子挖掘

### 3.1 项目逐项评估

#### 3.1.1 Microsoft RD-Agent（qlib 官方生态）

- **GitHub**：https://github.com/microsoft/RD-Agent
- **Stars**：14,711；Forks 1,925
- **维护状态**：**非常活跃**（最后推送 2026-09-15；微软持续投入，配套论文 R&D-Agent-Quant, arXiv:2505.15155）
- **License**：MIT
- **能力**：LLM Multi-Agent 自动化 R&D——**因子挖掘**（含"从研报提取因子"）+ **模型优化**两阶段闭环；与 qlib 深度集成（自动生成因子代码→在 qlib 上回测评估→迭代进化），有中英文演示视频；官方描述"automated factor mining and model optimization in quant investment R&D"（qlib README 核实）。
- **A 股支持**：依托 qlib 数据/回测层，继承 qlib 的 A 股能力；产出为**表达式/代码因子**，可落入 qlib 表达式因子管道。
- **复用判断**：**直接复用**（需要配置 LLM API Key，属于研究加速器而非基座）
- **推荐理由**：qlib 生态"钦定"的 Alpha 自动挖掘方案、MIT、社区热度高（一年内 14.7k stars）；是"表达式因子挖掘 + LLM"的当前最优开源实现。注意：LLM 调用产生费用，因子有效性仍需传统因子分析（第 4 章）把关。

#### 3.1.2 alphagen

- **GitHub**：https://github.com/ICT-FinD-Lab/alphagen（AlphaGen 论文官方实现，中科院 ICT-FinD-Lab）
- **Stars**：1,242；Forks 331
- **维护状态**：**低频维护**（最后推送 2026-06-04，仍有零星提交）
- **License**：**无 LICENSE 文件（GitHub API license: null）——默认保留所有权利**，个人研究可看代码自用，但复制/修改/分发存在法律风险，商用前必须联系作者
- **能力**：用**强化学习生成公式化因子集合**（"Generating sets of formulaic alpha (predictive) stock factors via reinforcement learning"），以"因子集合的复合 IC/RankIC"为奖励，解决因子冗余问题；算子集可自定义（表达式因子挖掘）；数据接口面向 A 股（论文实验即 A 股），但数据需自备（原用 RQData）。
- **复用判断**：**参考/借鉴为主**，或私有化改造（注意许可风险）
- **推荐理由**：学术前沿（RL 挖因子、集合奖励），思路可移植到自研引擎；由于无 License + 依赖商业数据源，**不建议直接依赖**，优先 RD-Agent（MIT）。

#### 3.1.3 gplearn（遗传规划 / 符号回归）

- **GitHub**：https://github.com/trevorstephens/gplearn
- **Stars**：1,889
- **维护状态**：**低频维护**（最后推送 2026-08-14；功能稳定，长期无大改）
- **License**：BSD-3-Clause
- **能力**：scikit-learn API 风格的遗传规划（SymbolicTransformer / SymbolicRegressor），自定义 `function_set`（add2/sub2/mul2/div2/sqrt/exp/log/abs/min/max/sin/cos 等）+ `init_depth` 即可做**符号回归式因子挖掘**；适应度函数可自定义（如最大化 IC）。
- **A 股支持**：与市场无关（输入面板数据/特征矩阵）；需自行把 A 股特征矩阵喂入并定义 IC 适应度。
- **复用判断**：**直接复用**（作为 GP 引擎，外层自写 IC 适应度 + 表达式去重）
- **推荐理由**：最省事的 GP/符号回归引擎，BSD 宽松，适合与 alphagen/RD-Agent 形成"GP 系 vs RL 系 vs LLM 系"三条挖掘路线对照实验。

#### 3.1.4 表达式因子挖掘（qlib 表达式引擎自建管道）

- **依托**：qlib `qlib.data.ops`（Ref/Mean/Std/Corr/Cov/Rank/Quantile/Resi/Slope/IdxMax/EMA/WMA/Delta 等 50+ 滚动算子，API 文档核实）+ `advanced/alpha.html`（Building Formulaic Alphas 教程）
- **复用判断**：**组合复用**：算子库与因子求值直接用 qlib；搜索器可用 gplearn（GP）/ alphagen（RL）/ RD-Agent（LLM）产出表达式，再注册为 qlib 自定义算子（`register_all_ops` 扩展）
- **推荐理由**：表达式因子的优势是"可解释 + 可增量计算 + 天然进 qlib 数据管道"，是个人研究性价比最高的因子形态。

### 3.2 因子挖掘对比总表

| 项目 | GitHub | Stars | 维护状态 | License | 挖掘范式 | 复用判断 | 推荐度 |
| -- | -- | -- | -- | -- | -- | -- | -- |
| RD-Agent | microsoft/RD-Agent | 14,711 | 活跃 | MIT | LLM Multi-Agent（因子+模型） | 直接复用 | ★★★★★ |
| alphagen | ICT-FinD-Lab/alphagen | 1,242 | 低频维护 | **无（保留所有权利）** | RL 生成因子集合 | 参考/改造 | ★★★ |
| gplearn | trevorstephens/gplearn | 1,889 | 低频维护 | BSD-3-Clause | 遗传规划/符号回归 | 直接复用 | ★★★★ |
| qlib 表达式引擎 | microsoft/qlib | — | 活跃 | MIT | 表达式算子组合（作为因子载体） | 直接复用 | ★★★★★ |

---

## 4. 绩效与风险指标计算（IC / 分层 / 换手 / 风险 / 稳定性 / 样本外）

### 4.1 指标 → 工具映射

| 指标 / 分析 | 含义与要点 | 推荐工具（复用） |
| -- | -- | -- |
| **IC（Pearson）** | 因子值与次期收益的截面相关系数；看均值、IC>0 比例、IC 衰减 | Alphalens `factor_information_coefficient`；qlib `analysis_model`（IC 图） |
| **RankIC** | 秩相关（Spearman），对离群值稳健，A 股截面更常用 | qlib 内置 Rank IC（`analysis_model` 报告含 IC/Rank IC）；或 pandas `spearmanr` 自算滚动窗口 |
| **分层收益（Quantile）** | 5/10 分组多空净值、多头-空头价差；检验单调性 | Alphalens `create_returns_tear_sheet`（Quantile Analysis）；qlib 分组累计收益图 |
| **换手率** | 因子/组合换手 → 决定成本敏感性；Alphalens Turnover Analysis | Alphalens `create_turnover_tear_sheet`；qlib `indicator_analysis` |
| **最大回撤 / Sharpe / Sortino / Calmar / Omega** | 组合风险收益指标 | quantstats（tearsheet 全覆盖）/ empyrical-reloaded（函数级）/ qlib `risk_analysis` |
| **因子稳定性** | 滚动 IC/RankIC 序列、IC_IR（IC 均值/IC 标准差）、分段一致性、换手分解 | 滚动窗口自算（pandas）+ quantstats 滚动图；建议自定义 ~50 行脚本（唯一少量自研点） |
| **样本外 / 滚动回测** | Walk-forward：滚动训练+滚动预测+滚动回测，防过拟合 | **qlib `RollingGen`**（Task Management，任务滚动生成）+ `benchmarks_dynamic`（Rolling Retraining 基线、DDG-DA）；Optuna 嵌套外层 |
| **分市场阶段回测** | 牛/熊/震荡/流动性危机分段对比 | 按日期区间切分后分别跑 qlib 回测 + quantstats 分段 tearsheet（无现成"一键"工具，切分脚本 <30 行） |
| **多空/基准超额** | 相对 000300/000905 的超额收益、信息比率、跟踪误差 | qlib `risk_analysis`（excess return with/without cost，`qrun` 直接输出）；quantstats `reports.html(benchmark=...)` |

### 4.2 评估要点

- **IC/RankIC、分层、换手**三件套是单因子准入标准：Alphalens(-reloaded) 一键覆盖，**直接复用**；A 股使用时注意：①收益用后复权收益；②剔除涨跌停不可交易样本（否则多头分层虚高/虚低）；③剔除 ST/次新/停牌；④默认 252 年化改 244。
- **组合级指标**用 quantstats（报告）+ qlib `risk_analysis`（研究管道内嵌），empyrical-reloaded 备用。
- **滚动/样本外**：qlib 原生 `RollingGen` 支持滚动任务生成（训练/验证/测试段滚动），是免费拿到的"滚动回测"能力，无需自研。
- **分市场阶段**属于研究方法而非组件，用日期切分循环调用上述工具即可（约 30 行胶水代码，是本方案中极少数需自写的部分）。

---

### 4.3 组合优化与权重分配（任务书⑤补充）

| 需求 | 推荐 | 说明 |
| -- | -- | -- |
| TopK/固定名额调仓 | qlib `TopkDropoutStrategy`（topk + n_drop） | 个人量化最常用范式，无需优化器 |
| 按权重/指数增强 | qlib `WeightStrategyBase` / `EnhancedIndexingStrategy` | 按目标权重调仓基类、对标指数主动增强 |
| 显式组合优化（均值方差/风险平价/BL） | **cvxpy**（Apache-2.0）自建目标函数；Riskfolio-LP（**GPL-3.0**）仅作现成实现参考 | 个人系统 P2 再引入；GPL 库不并入分发代码（见设计文档 §16.3） |
| 组合风险归因 | quantstats + 自研暴露分解（行业/风格中性化残差） | Portfolio/持仓分析消费 |

> 判断：MVP/第一阶段用 TopK 范式即可覆盖绝大多数选股策略；组合优化属第二阶段增量，且优先 cvxpy（License 干净）而非 GPL 库。

---

## 5. 实验追踪与结果管理

### 5.1 项目逐项评估

| 项目 | GitHub | Stars | 维护状态 | License | 定位与能力 | 与 qlib/量化研究结合 | 复用判断 |
| -- | -- | -- | -- | -- | -- | -- | -- |
| **MLflow** | mlflow/mlflow | 28,095 | 活跃（2026-09-22 推送） | Apache-2.0 | 实验追踪（Params/Metrics/Artifacts）、模型注册、部署 | **qlib `QlibRecorder` 基于 MLflow 实现**（Experiment/Recorder/Record Template 体系，文档核实），qrun 的实验记录可直接用 MLflow UI 比较 | **直接复用（首选）** |
| **Weights & Biases** | wandb/wandb | 11,258 | 活跃 | MIT（SDK）；SaaS/私有化服务 | 实验追踪、Sweeps 超参搜索、系统监控、报告协作 | 图形与对比体验最好；免费额度够个人用；数据上云需评估隐私 | 直接复用（备选，图形化更好） |
| **Kedro** | kedro-org/kedro | 11,006 | 活跃 | Apache-2.0（LICENSE.md 为标准 Apache-2.0 文本） | 数据科学流水线框架（DataCatalog + Pipeline DAG + 约定式工程结构） | 把"数据→因子→模型→回测"组织成可复现节点图，可包裹 qlib 调用 | 可选复用（工程化阶段再引入） |
| **DVC** | treeverse/dvc（原 iterative/dvc，已迁移组织） | 15,881 | 活跃 | Apache-2.0 | 数据/模型文件版本化（git-like）、管道与实验表格（`dvc exp`） | 版本化 qlib 数据快照、回测产物 | 可选复用（数据 >GB 后引入） |
| **Optuna** | optuna/optuna | 14,832 | 活跃（2026-09-18 推送） | MIT | 超参搜索（TPE/CMA-ES/Grid/Random）、剪枝、分布式、Dashboard | 搜模型超参（LightGBM/NN）与策略参数（TopK/n/drop）；与 MLflow 集成简单 | **直接复用（首选）** |
| **qlib qrun / QlibRecorder** | microsoft/qlib | — | 活跃 | MIT | YAML 驱动"数据集→训练→回测→评估"一键流水线；Recorder 记录 SignalRecord/SigAnaRecord/PortAnaRecord 产物 | 即主基座自带实验记录，底层 MLflow | 直接复用 |

### 5.2 组合建议

1. **实验记录**：优先直接吃 qlib 红利——`qrun` + QlibRecorder（MLflow 后端）已经把每次实验的参数、预测、IC 分析、回测报告落盘；配 `mlflow ui` 即得实验对比界面。无需额外接入成本。
2. **调参**：Optuna 包住"训练+回测"目标函数（如最大化回测 IR 或 RankIC），实验记录通过 `mlflow.set_tracking_uri` 复用同一后端。
3. **W&B**：如果更看重图表与对比 UI，可在模型训练脚本里双写 W&B（几行代码），云端看板体验优于 MLflow UI。
4. **Kedro / DVC**：个人研究初期不必引入；当数据（多版本 qlib_bin、分钟线）超 10GB 或流程节点 >10 个时再上，避免过度工程。

---

## 6. 回测引擎类型对比：向量化 vs 事件驱动

| 维度 | 向量化回测（Vectorized） | 事件驱动回测（Event-Driven） |
| -- | -- | -- |
| **原理** | 信号/持仓/收益对整段历史用矩阵运算一次算出 | 按时间逐 bar/tick 回放，走"事件→策略→撮合→记账"循环 |
| **速度** | 快 1~3 个数量级（NumPy/Pandas 并行） | 慢（Python 循环 + 撮合逻辑），全市场多年回测分钟级起 |
| **仿真精度** | 粗糙：成交假设简化（当日全成/固定滑点），难精确建模资金占用、部分成交、涨跌停拒单 | 精细：可逐单撮合，建模 T+1、涨跌停、停牌、整手、最小佣金、成交量约束 |
| **适用场景** | 因子研究、大样本统计检验、参数扫描、组合权重回测、想法快速证伪 | 细节敏感策略（打板、隔夜、日内执行）、交易规则约束强的验证、模拟盘/实盘对接 |
| **典型代表** | Alphalens（因子级）、vectorbt、qlib 的信号分析层、quantstats（事后分析） | RQAlpha、backtrader、zipline、vnpy、Hikyuu（近似）、qlib 的 Exchange/Executor 层 |
| **A 股微观结构** | 需在信号层规避（过滤涨跌停/停牌样本） | 可在撮合层精确拦截（qlib `limit_threshold`、RQAlpha `sys_risk`） |

**本系统结论（两段式）**：

1. **研究段用向量化**：因子挖掘与初筛（IC/分层）天然是截面向量运算——Alphalens + qlib 表达式引擎 + vectorbt 参数扫描，把"想法→证据"的迭代压到分钟级；
2. **验证段用事件驱动/半事件撮合**：入围因子/策略进入 qlib Exchange 回测（涨跌停/停牌/整手/费用/容量全建模）或 RQAlpha 撮合复核；两者结论偏差本身就是"执行摩擦"的度量。
3. qlib 恰好是**混合型**（向量化特征管道 + 事件化执行器 Executor/Exchange，还支持 Nested Executor 嵌套日内执行），一套框架内完成上述两段，这是选它做主基座的重要理由。

---

## 7. 重点评估：Microsoft qlib 作为主回测基座的可行性

> 依据：GitHub API 元数据、官方 ReadTheDocs（v0.9.8.dev）、`qlib/backtest/exchange.py` 源码、官方 README、社区 issue/教程交叉核实。

### 7.1 数据格式与 Point-in-Time 数据库

- **数据格式**：紧凑二进制"qlib 格式"——按 instrument 分目录的 `.day.bin` 特征文件 + calendar/day.txt + instruments/all.txt + features/<code>/*.day.bin；官方基准显示其数据层带 ExpressionCache/DatasetCache 时比 HDF5/MySQL/MongoDB/InfluxDB 快 4~50 倍（README 性能表：14 特征 × 800 股 × 2007–2020 任务，最优 7.4s vs MySQL 365s）。
- **数据摄入**：CSV/Parquet → qlib 格式转换器（`dump_bin`）；Yahoo 爬虫脚本（`scripts/data_collector/`，含 1d/1min）；**官方 cn_data 数据集因数据合规政策暂时下线**，README 明确推荐社区镜像 [chenditc/investment_data](https://github.com/chenditc/investment_data/releases)（个人 A 股研究的现成数据源）。也可用 AkShare/tushare 数据自建 qlib_bin。
- **Point-in-Time 数据库**：官方支持（PR #343，2022-03 发布），专门解决**财报修订造成的数据泄漏**：PIT 表每特征 4 列（date 披露日 / period 报告期 / value / _next 链表指针），带 `.index` 索引文件，支持"任意历史时点取当时可见版本"；配 `scripts/data_collector/pit/` 爬虫+转换器（支持财报 PIT）。**已知限制**（文档原文）：仅面向季度/年度因子（财报类），PIT 计算未做极致优化。对基本面因子研究这是关键能力。
- **判断**：✅ 满足需求；PIT 用于财报因子防前视，价量因子靠表达式时序（`Ref`）天然 point-in-time。

### 7.2 Alpha158 / Alpha360 数据集

- 官方"Quant Dataset Zoo"收录 **Alpha158**（158 个价量统计特征）与 **Alpha360**（360 个近端价量原始特征），US/China 双市场可用（README 核实），定义在 `qlib/contrib/data/handler.py`；
- 20+ 基准模型（LightGBM/XGBoost/CatBoost/LSTM/GRU/ALSTM/Transformer/GATs/TRA/HIST/TFT/TabNet 等）在这两个数据集上有一致的 benchmark 结果表（`examples/benchmarks/README.md`），可作为自研模型的对照基线；
- **Label 注意**：Alpha158 默认 label 为 `Ref($close,-2)/Ref($close,-1)-1`（T+1 收盘→T+2 收盘收益，对应"T 日信号、T+1 收盘成交"的执行假设；社区经典疑问见 issue #1514），自定义执行假设时需同步改 label；
- 自定义数据集：复制 handler 配置改 features 表达式即可（Alpha158 本质是一份 YAML 化表达式清单）。
- **判断**：✅ 直接复用，且是可自定义的模板。

### 7.3 Handler / DataLoader

- 分层清晰（文档 `component/data.html` 核实）：
  - **DataLoader**：`QlibDataLoader`（表达式→面板）、`StaticDataLoader`（外部 DataFrame/文件），接口 `load()` 可自定义；
  - **DataHandlerLP**：加载 + 预处理（Processor 链：缺失值填充、标准化、去极值、行业市值中性化等可插拔）+ 磁盘缓存；
  - **Dataset**：`DatasetH` 支持滚动切片（segments: train/valid/test），与 `RollingGen` 配合生成滚动任务。
- 自定义因子 = 在 features 表达式里加一行（如 `"Ref($close,1)/$close-1"`），或注册新算子（`qlib.data.ops.register_all_ops`）。
- **判断**：✅ 直接复用 + 配置化扩展，无需造数据管道轮子。

### 7.4 模型接口（自定义因子 / 模型）

- `qlib.model.base.Model`：`fit(dataset)` / `predict(dataset)` 两个方法即接入（官方"Custom Model Integration"文档给出完整步骤与配置文件写法）；`ModelFT` 支持微调；
- 模型无关：sklearn / LightGBM / PyTorch / TF 均可包装；Alpha158+LightGBM 是官方 demo（`qrun` 30 行配置跑通全流程）；
- 内置 20+ 论文模型可直接跑对照；Meta-Learning（DDG-DA）与 RL 框架（订单执行）另有模块。
- **判断**：✅ 接口最小化（fit/predict），自定义成本极低。

### 7.5 回测模块（TopKDropout 等策略、A 股微观结构）

- **策略**（`qlib.contrib.strategy`）：`TopkDropoutStrategy`（TopK 持仓 + 每日最多换 Drop 只，最经典的公募指增/量化选股范式）、`SoftTopkStrategy`、`EnhancedIndexingStrategy`（对标指数的主动增强）、`WeightStrategyBase`（按权重调仓基类）、`TWAPStrategy`、`SBBStrategyEMA` 等；
- **执行**：`Executor`/`NestedExecutor`（支持日内嵌套执行，1min 高频示例）、`Exchange` 撮合（第 2.1.1 节已核实的 A 股能力）：
  - `deal_price` 可指定买卖分别用 `$open/$close/$vwap` 等不同成交价；
  - `limit_threshold` 涨跌停（float 或表达式，区分买/卖方向）；
  - `trade_unit=100` 整手 + `$factor` 复权还原取整；
  - `open_cost/close_cost/min_cost/impact_cost`（费用含最低佣金与冲击成本），`volume_threshold` 成交量/容量约束（含分钟级 DayCumsum 累计成交量约束）；
  - 停牌（NaN close）自动不可交易；REG_CN/REG_TW 场景有专门告警。
- **输出**：`qrun` 直接给出"无成本/有成本超额收益"的 mean/std/annualized_return/information_ratio/max_drawdown（README 示例核实）；`PortAnaRecord` 落盘回测明细。
- **T+1 补充说明**：日频 TopK 工作流"T 日收盘出信号 → T+1 日调仓"天然合规；**严格"T+1 日买入股份当日锁定"在撮合层未建模**（exchange.py 源码核实无持仓锁），日频单次调仓不受影响，做日内多级执行时需自行加锁——这是唯一需要小改造的点。
- **判断**：✅ 直接复用（TopK/指数增强场景）；⚠️ 严格 T+1、日内交易规则需少量改造。

### 7.6 qrun 与实验记录机制

- **qrun**：一条命令跑通"构建数据集→训练→预测→信号分析→组合回测→评估"（README 核实），YAML 即实验定义（`workflow_config_lightgbm_Alpha158.yaml` 为模板）；
- **QlibRecorder**（文档 `component/recorder.html` 核实）：Experiment/Recorder 分层管理，`log_params/log_metrics/log_artifact/save_objects` 齐全，Record Template（SignalRecord→SigAnaRecord→PortAnaRecord）自动沉淀每次实验的预测值、IC 分析、回测报告；底层为 **MLflow**（`set_uri` 可指向共享 MLflow server），`search_records` 跨实验检索对比；亦提供 `workflow_by_code` Python 化编排；
- 任务管理：`TaskManager` + `RollingGen` 支持滚动回测任务生成与收集（多实验汇总 Collector）。
- **判断**：✅ 自带轻量实验追踪（MLflow 后端），与第 5 章选型无缝衔接。

### 7.7 中文文档与社区活跃度

- **官方文档**：ReadTheDocs 英文（结构完整：Quick Start / Data / Model / Strategy / Recorder / Report / PIT / FAQ / API）；官方 README 有中文媒体解读（微软公众号"微矿Qlib"等），**但无官方中文文档站**；
- **中文社区资料**：丰富——中文教程翻译（如 wuzao.com 的中文文档镜像）、DeepWiki 自动生成的中文解读、vnpy 社区"Alpha158 标签"等专题讨论、大量知乎/公众号实战文章；A 股用户基数大（国内量化圈主流研究框架之一）；
- **社区活跃度**：48.7k stars、7.7k forks、510 watchers、近期日更（2026-09-22 仍有推送）、Gitter 实时群 + GitHub Issues（open 480，响应尚可）、微软研究院持续投入（RD-Agent 联动）。
- **判断**：✅ 社区活跃度顶级；⚠️ 官方文档英文为主，中文靠社区——对本项目影响有限（技术文档可读即可）。

### 7.8 Apple Silicon（Mac mini）安装与性能

- **安装**：
  - PyPI 包 `pyqlib` 支持 macOS（README 平台徽章：linux | windows | macos），Python 3.8–3.12 均有 pip/源码安装支持；
  - **官方 M1 提示**（README "Tips for Mac" 原文）：M1 上构建 LightGBM wheel 可能因缺 OpenMP 失败，`brew install libomp` 后即可构建成功——即 Apple Silicon 可用，但需装系统依赖；
  - 历史兼容性摩擦：官方 issue #1525 "Apple M1 not supported" 记录了早期 M1 支持问题（该 issue 页面调研时多次抓取失败，仅核实到标题）；当前 README 已把 macOS 列为正式平台且给出 M1 方案，社区（如 chenditc/investment_data、各中文教程）在 Apple Silicon 上的使用报告普遍正面；
  - 源码安装含 Cython 扩展编译，需 Xcode Command Line Tools；建议 conda 管理环境（README 明确建议）。
- **性能**：
  - qlib 数据层为紧凑二进制 + 两级缓存（ExpressionCache/DatasetCache），CPU 核数越多数据集构建越快（README 基准：1CPU 147s → 64CPU 8.8s）；Mac mini（M 系 8–20 核）多核收益明显；
  - 模型训练（LightGBM/PyTorch）在 Apple Silicon 上有原生 arm64 加速（PyTorch MPS 可用于 NN 模型）；
  - **注意**：qlib 官方无 Apple Silicon 公开基准，上述为架构推断 + 社区经验，落地后应以本机实测（建议以 Alpha158+LightGBM `qrun` 全流程计时为基准测试）。
- **判断**：✅ 可安装可用（`brew install libomp` 一行解决主要坑）；性能够个人研究用，需本机实测定标。

### 7.9 可行性结论与改造清单

**结论：qlib 作为主回测基座完全可行**，理由：MIT 许可、48.7k 社区、A 股微观结构（涨跌停/停牌/整手/费用/容量）原生建模、PIT 防前视、Alpha158/360 开箱、fit/predict 极简模型接口、qrun+MLflow 实验记录、Apple Silicon 可用。

**需要的改造/适配（工作量小）**：

| # | 改造点 | 量级 | 说明 |
| -- | -- | -- | -- |
| 1 | 数据源接入 | 小 | 用社区镜像 chenditc/investment_data 起步；长期用 AkShare/tushare 自建 qlib_bin + `dump_bin` |
| 2 | 费用参数校准 | 极小 | 按实际券商佣金（约 0.01–0.03%，最低 5 元）+ 卖出印花税（当前 0.05%）设 `open_cost/close_cost/min_cost` |
| 3 | 涨跌停参数 | 极小 | 主板 10%/ST 5%/创业板科创板 20%/北交所 30%——用 `limit_threshold` 表达式形式按板块区分 |
| 4 | label 定制 | 小 | 按执行假设（T+1 开盘/收盘成交）改 Alpha158 label 配置 |
| 5 | 严格 T+1 持仓锁 | 小-中 | 仅当日回转交易场景需要；在 Strategy/Position 层加"当日买入不可卖"约束 |
| 6 | 分市场阶段报告 | 小 | 约 30 行日期切分 + quantstats 胶水脚本 |

---

## 8. 推荐技术栈组合与实施路线

### 8.1 目标架构（自上而下）

```
因子挖掘层：RD-Agent（LLM）｜ gplearn（GP）｜ alphagen 思路（RL，可选）
        ↓ 产出表达式/代码因子
因子评估层：Alphalens-reloaded（IC/RankIC/分层/换手）+ 自有滚动稳定性脚本
        ↓ 通过准入的因子
研究基座层：Microsoft qlib（qlib_bin 数据 + PIT 财报库 + Handler/Dataset + 模型 + RollingGen 滚动样本外）
        ↓ 信号
回测验证层：qlib Exchange（涨跌停/T+1 时序/费用/容量）→ RQAlpha 复核（严格 T+1/事件撮合，可选）
        ↓ 绩效
报告分析层：quantstats tearsheet + qlib risk_analysis（超额/IR/回撤）
        ↓ 产物
实验管理层：MLflow（QlibRecorder 原生）+ Optuna（调参）［W&B 可双写］［Kedro/DVC 后期可选］
（未来实盘）：vnpy 交易执行层
```

### 8.2 实施路线（建议顺序）

1. **数据**：Mac mini 上装 pyqlib（`brew install libomp`）→ 落地社区 qlib_bin（chenditc/investment_data）→ 自建财报 PIT（可后置）；
2. **基线跑通**：`qrun` 跑 Alpha158 + LightGBM 官方基准，实测本机性能并校准费用/涨跌停参数；
3. **因子分析闭环**：接 alphalens-reloaded + quantstats，固定一套"因子准入卡"（IC、RankIC、IC_IR、分层单调性、换手、最大回撤）；
4. **滚动样本外**：用 `RollingGen` 搭滚动训练/回测模板，固化"分市场阶段"切分脚本；
5. **实验管理**：起本地 MLflow tracking（QlibRecorder 默认），Optuna 做模型与策略参数搜索；
6. **挖掘增强**：接 RD-Agent 做 LLM 因子挖掘；gplearn 做 GP 对照组；
7. **（可选）** RQAlpha 高保真复核 / vnpy 实盘 / Kedro-DVC 工程化。

---

## 9. 风险与注意事项

1. **License 风险**：backtrader（GPL-3.0）、backtesting.py（AGPL-3.0）有传染性；**alphagen 无 License（保留所有权利）**——仅参考思路，勿直接复制代码进产品；**RQAlpha 仅限非商业用途**（商用需米筐授权）；vectorbt 为自定义许可，商用前审查。qlib/RD-Agent/MLflow/Optuna/Kedro/DVC/vnpy/Hikyuu 均为 MIT/Apache-2.0，无此类风险。
2. **数据风险**：qlib 官方数据集因合规暂时下线；Yahoo 源数据质量一般（官方原话 "might not be perfect"），严肃研究建议自建高质量数据源并跑 `check_data_health.py`。
3. **前视偏差**：财报因子务必走 PIT 通道（否则用"最新修订版"会泄漏未来）；Alpha158 label 的成交时点假设要与回测执行假设一致；Alphalens 分析时剔除涨跌停/停牌样本。
4. **过拟合**：挖掘类工具（RD-Agent/gplearn/alphagen）会系统性放大过拟合——必须以"滚动样本外 + 分市场阶段 + 换手成本后收益"为最终准绳，Optuna 搜索嵌套在滚动框架内。
5. **平台风险**：backtrader/empyrical/alphalens 原版均处于停滞/半停滞，选型时一律用续维护 fork（zipline-reloaded/alphalens-reloaded/empyrical-reloaded）或避开。

---

## 10. 来源 URL 清单

**GitHub API 元数据核实（stars/License/pushed_at，2026-09-22）：**

- https://api.github.com/repos/microsoft/qlib
- https://api.github.com/repos/ricequant/rqalpha
- https://api.github.com/repos/mementum/backtrader
- https://api.github.com/repos/vnpy/vnpy
- https://api.github.com/repos/stefan-jansen/zipline-reloaded
- https://api.github.com/repos/fasiondog/hikyuu
- https://api.github.com/repos/quantopian/alphalens
- https://api.github.com/repos/stefan-jansen/alphalens-reloaded
- https://api.github.com/repos/quantopian/empyrical
- https://api.github.com/repos/ranaroussi/quantstats
- https://api.github.com/repos/kernc/backtesting.py
- https://api.github.com/repos/microsoft/RD-Agent
- https://api.github.com/repos/ICT-FinD-Lab/alphagen
- https://api.github.com/repos/trevorstephens/gplearn
- https://api.github.com/repos/polakowo/vectorbt
- https://api.github.com/repos/mlflow/mlflow
- https://api.github.com/repos/wandb/wandb
- https://api.github.com/repos/kedro-org/kedro
- https://api.github.com/repositories/83878269 （treeverse/dvc，原 iterative/dvc）
- https://api.github.com/repos/optuna/optuna

**源码 / License 原文：**

- https://raw.githubusercontent.com/microsoft/qlib/main/qlib/backtest/exchange.py （涨跌停/费用/整手/容量约束源码）
- https://raw.githubusercontent.com/microsoft/qlib/main/README.md （qrun、Alpha158/360、M1 提示、数据集状态、性能基准）
- https://raw.githubusercontent.com/ricequant/rqalpha/master/LICENSE （非商业条款原文）
- https://raw.githubusercontent.com/ricequant/rqalpha/master/README.rst （Mod 体系/中文文档/交易税费）
- https://raw.githubusercontent.com/kedro-org/kedro/main/LICENSE.md （Apache-2.0 原文）

**官方文档：**

- https://qlib.readthedocs.io/en/latest/ （qlib 文档总目录：Data/Model/Strategy/Recorder/Report/Task Management）
- https://qlib.readthedocs.io/en/latest/_sources/advanced/PIT.rst.txt （Point-in-Time 数据库设计与限制）
- https://qlib.readthedocs.io/en/latest/component/workflow.html （qrun/Workflow 配置）
- https://qlib.readthedocs.io/en/latest/component/recorder.html （QlibRecorder 实验管理）
- https://qlib.readthedocs.io/en/latest/component/data.html （DataLoader/DataHandlerLP/Processor）
- https://qlib.readthedocs.io/en/latest/component/strategy.html （TopkDropoutStrategy 等）
- https://qlib.readthedocs.io/en/latest/advanced/alpha.html （表达式因子构建教程）
- http://rqalpha.readthedocs.io/zh_CN/latest/ （RQAlpha 中文文档）
- https://hikyuu.readthedocs.io/zh-cn/latest/ （Hikyuu 中文文档）

**社区 / 补充核实：**

- https://github.com/microsoft/qlib/issues/1525 （Apple M1 支持问题）
- https://github.com/microsoft/qlib/issues/1514 （Alpha158 label 计算讨论）
- https://repos.ecosyste.ms/hosts/GitHub/repositories/stefan-jansen%2Fempyrical-reloaded （empyrical-reloaded 元数据）
- https://github.com/chenditc/investment_data （qlib 社区 A 股数据镜像，README 官方推荐）
- https://github.com/azluck/zipline （第三方 A 股 zipline bundle 参考）
- https://www.interactivebrokers.com/campus/ibkr-quant-news/a-practical-breakdown-of-vector-based-vs-event-based-backtesting/ （向量化 vs 事件驱动对比）
- https://arxiv.org/abs/2505.15155 （R&D-Agent-Quant 论文）
- https://www.vnpy.com/forum/topic/33656 （Alpha158 标签中文讨论）
- https://deepwiki.com/vnpy/vnpy/5-backtesting-and-analysis （vnpy 回测引擎说明）
