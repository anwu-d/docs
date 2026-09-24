# 个人 A 股量化研究系统 · 开源选型调研 02：回测与因子研究

> **修订记录 v2（2026-09-23，依据审计 A02/A03/A04/A08/§4）**：
>
> - **A02（执行契约）**：新增第 2.1–2.2 节"统一执行契约"与"标签语义"。明确 Qlib 默认标签 `Ref($close,-2)/Ref($close,-1)-1` 表示 **T+1 收盘至 T+2 收盘收益**（审计 S1）；标签一律作为**预测代理**使用，必须声明与最终执行收益的差异，**删除原稿"与执行假设一致"类表述**（原 §7.2、§9.3 已改写）。
> - **A03（成交与样本完整性）**：新增第 2.3 节。信号股票池、下单候选、成交结果、持仓分开保存；禁止事后删除未成交样本美化回测；旧仓未成交继续持有和估值并记录原因；日线无法判定的盘中成交采用公开说明的保守假设（改写原 §2.1.7、§4.2 中"剔除不可交易样本"的口径）。
> - **A04（样本外封存）**：新增第 2.4 节。训练/开发验证/最终封存测试分离；冻结后才开最终测试；实现错误可修、经济表现不佳记失败不得循环"修到通过"；最终测试访问与结果回流留日志；按标签跨度处理切分边界；保存失败/否决/参数变体；**删除"入库因子数量"硬指标，零有效因子允许正常验收**。
> - **A08（许可证口径）**：第 3.1.3、3.1.10、3.2、4.3、10.1 节按 GNU FAQ/AGPLv3 条款重写：GPL 个人本地使用、修改不因使用本身要求公开个人代码（S4），**Backtrader 重新列入替代候选**；AGPL 按条款区分修改、网络交互与源代码提供义务（S5），不简单等同分发；无许可证不等于允许复制；非商业限制须检查具体用途；进程隔离不自动消除许可义务；开源代码许可/模型许可/数据使用条款/服务订阅条款分开记录。
> - **审计 §4（收敛范围）**：回测沿用 Qlib、先简单基线与手算案例；Backtrader 仅作替代候选、**不同时维护两套引擎**；因子分析限少量预先声明的简单因子及对照；RD-Agent、遗传规划（gplearn）、自动因子生产线**延后**（原第 3 章改标为延后项，实施路线重排）。
> - **事实保留与待核实约定**：v1 中经 GitHub API/官方文档/源码核实的事实与来源 URL 全部保留，核实时点为 2026-09-22（stars/维护状态为该日快照，此后变化**待核实**）；本次修订新增的许可条款依据（S4 GNU FAQ、S5 AGPLv3）在审计中仅取得摘要或页面超时，采用前**待核实**完整条款；未逐项重核的旧结论沿用原核实记录。

> **调研对象**：部署于 Mac mini（Apple Silicon）的个人 A 股量化研究系统
> **选型原则**：尽量复用现成开源项目，不重复造轮子
> **调研方法**：所有项目的 stars / License / 维护状态均于 **2026-09-22** 通过 GitHub API（`api.github.com/repos/<owner>/<repo>`）逐一核实；功能描述结合官方文档、源码（如 `microsoft/qlib` 的 `qlib/backtest/exchange.py`）与社区资料交叉验证。文末附全部来源 URL。
> **说明**：stars 数与"最后推送日期（pushed_at）"为调研时点快照，之后会变化（v2 修订未重核，**待核实**当前值）。

---

## 0. 执行摘要（TL;DR）

| 层次 | 首期方案（审计 §4 收敛后） | 备选 / 延后 |
| -- | -- | -- |
| **主回测 / 因子研究基座** | **Microsoft qlib**（MIT，48.7k stars，活跃）——先简单基线 + 手算对齐案例 | **backtrader（GPL-3.0）作为替代候选**（仅当 qlib A 股适配测试不通过时启用，不同时维护两套）；rqalpha（注意非商用条款，按用途检查） |
| **回测正确性契约** | 统一执行契约 + 标签差异声明 + 成交/持仓分层留痕（本文 §2，A02/A03） | — |
| **样本外机制** | 训练/开发验证/最终封存测试三分 + 冻结后开测（本文 §2.4，A04） | qlib `RollingGen` 作为滚动训练机制组件 |
| **单因子分析（IC/分层/换手）** | **Alphalens → alphalens-reloaded**，限少量**预先声明**的简单因子及对照 | qlib 内置 `analysis_model` |
| **组合绩效与风险指标** | **quantstats** + qlib 内置 `risk_analysis` | empyrical-reloaded |
| **Alpha 自动挖掘** | **延后**（审计 §4）：首期不做 RD-Agent / 遗传规划 / 自动因子生产线 | RD-Agent（MIT）、gplearn（BSD-3）、alphagen 思路——延后至 G4 且按 A04 防过拟合约束 |
| **实验追踪** | **MLflow**（qlib QlibRecorder 原生基于它），落盘失败/否决/参数变体 | W&B（图形体验更好） |
| **超参搜索** | **Optuna**（嵌套在开发验证内，禁止触碰封存测试集） | W&B Sweeps |
| **数据/流水线版本化** | Git + 自建脚本（冻结数据版本用 snapshot_id） | DVC（可选）+ Kedro（可选） |
| **策略原型快速验证** | backtesting.py（单标的，AGPL 按条款分场景判断）/ vectorbt（组合向量化） | — |

**结论（v2）**：以 **qlib 为唯一主基座（数据 + 因子 + 模型 + 回测 + 实验记录）**，首期只做**少量预先声明的简单因子 + 简单基线策略 + 手算对齐案例**，把可信度验证（执行契约、成交守恒、封存测试）置于挖掘规模之前；叠加 **alphalens-reloaded（单因子体检）、quantstats（组合报告）、MLflow + Optuna（实验与调参）** 构成个人 A 股量化研究闭环。RD-Agent/gplearn 等自动挖掘、第二套回测引擎（backtrader 仅作替代候选）均延后，避免在执行口径未对齐前放大过拟合与维护成本。

---

## 1. 评估维度说明

对每个项目按以下维度评估：

1. **GitHub 地址 / stars / 维护状态**（GitHub API 核实，含最后推送时间）；
2. **License**——按 A08 口径**分四类分开记录**：①开源代码许可；②模型许可；③数据使用条款；④服务订阅条款。开源代码许可再按**使用方式**分别判断：个人本地自用 / 代码公开发布（分发）/ 对外网络服务。GPL/AGPL 不再整体列为禁区，见 §10.1；
3. **A 股支持**细分为四项：
   - 前/后复权处理；
   - 涨跌停限制；
   - T+1 交收制度；
   - 手续费 / 印花税（及最低佣金、滑点）模型；
4. **"直接复用 vs 需改造"判断**；
5. **推荐理由**。

维护状态约定：`pushed_at` 距调研日 1 个月内 = **活跃**；1–6 个月 = **低频维护**；6–24 个月 = **半停滞**；>24 个月或仓库已归档 = **停滞**。

---

## 2. 回测正确性契约（本次修订核心，落实 A02 / A03 / A04）

> 依据：审计方案 §3 发现 A02/A03/A04 及 §7 验收矩阵。本节约束适用于本文所有回测/因子评估工具，不论底层引擎是 qlib、backtrader 还是其他。

### 2.1 统一执行契约（A02）

任何研究指标与策略回测在运行前必须声明并落盘同一份**执行契约**，至少包含：

| 要素 | 必须明确的内容 | qlib 落点（配置项） |
|---|---|---|
| **信号截至时间** | 信号使用哪些时刻已收盘/已披露的数据；T 日信号的确切截点（如 T 日收盘后） | dataset/label 表达式的时间索引（`Ref` 系列） |
| **最早下单时间** | 信号产生后最早的下单/成交时段（如 T+1 开盘或 T+1 收盘） | `Executor` 交易时段 + `deal_price` |
| **买卖价格** | 买入价、卖出价分别取 `$open/$close/$vwap` 或其他，并说明滑点 | `Exchange.deal_price`（买卖可分别指定）、`impact_cost` |
| **持有期** | 标签/收益度量的持有区间与策略实际持有期一致或给出差异说明 | label 定义 vs 策略调仓频率 |
| **交易日历** | 使用哪个日历（交易所日历，非周一至周五）；跨节假日的"下一交易日"语义 | `calendar/day.txt`（数据包自带） |
| **可卖数量** | T+1 交收下当日买入不可当日卖；可卖数量（closable）如何计算 | 引擎层无原生持仓锁（`exchange.py` 源码核实），须在策略/持仓层显式约束 |
| **费用** | 佣金（含最低佣金）、印花税、滑点/冲击成本的费率与计费边 | `open_cost/close_cost/min_cost/impact_cost` |
| **部分成交规则** | 成交量约束下的部分成交如何记账（成交价、剩余单、撤销规则） | `volume_threshold`（容量约束）+ 自建记账规则，见 §2.3 |

契约声明"信号—标签—执行"三者一致或**明确列出差异**；不允许只写其一。

### 2.2 标签语义与"预测代理"声明（A02 / 审计 S1）

- **Qlib 默认标签的准确语义**：Alpha158 默认 label `Ref($close,-2)/Ref($close,-1)-1` 表示 **T+1 收盘至 T+2 收盘收益**（审计外部核对 S1：[Qlib 数据文档](https://github.com/microsoft/qlib/blob/main/docs/component/data.rst)、[Alpha158 实现](https://github.com/microsoft/qlib/blob/main/qlib/contrib/data/handler.py)；社区讨论见 issue #1514）。
- **标签是预测代理，不是执行收益**：标签收益未包含手续费、滑点、部分成交、停牌/涨跌停拒单、可卖数量限制与持有期错位。因此：
  - 标签可用于训练与 IC/分层等**预测能力**评估；
  - 但**必须随结果声明其与最终执行收益的差异**（差异来源逐项列出）；
  - **禁止**在任何报告/配置中将标签口径标注为"与执行假设一致"（v1 此类表述已删除）。
- 自定义执行假设（如 T+1 开盘成交）时，标签须按 §2.1 契约同步重定义，并保留旧/新标签的对照说明。

### 2.3 成交、股票池与持仓的分层记录（A03）

- **四类记录分开保存、各自可追溯**：①信号股票池；②下单候选；③成交结果；④持仓。任何一层不得由后一层事后改写。
- **股票池只用决策时已知条件生成**：上市/退市状态、ST 标记、停牌状态、行业成分等均取"决策时点可见"的历史快照，不得用当前股票列表回填（与主设计 `universe_snapshot` 口径一致）。
- **成交时逐单检查**：交易**方向**（涨/跌停方向限制不同）、**可成交量**（容量约束、整手）、**停牌**状态，以及**对应历史时期的交易制度**（如创业板涨跌停 2020-08 前后 10%→20%、ST 档位、T+1 交收）。
- **旧仓未成交的处理**：卖单失败（跌停/停牌）时旧仓**继续持有并按市价估值**，记录未成交原因（方向受限/可成交量不足/停牌/制度限制）；买单失败则记录现金留存。现金、数量、费用、净值逐日守恒可对账。
- **禁止事后删除未成交样本**：不得将未成交的信号/下单样本从回测样本中剔除来"美化"回测；因子诊断中的样本分层对照见 §5.2，也不得回流到净值计算。
- **日线无法判定的盘中成交**（如日内何时触板、部分成交时点）：采用**公开说明的保守假设**（对结果不利方向），并在报告中注明假设文本；不得默认乐观成交。

### 2.4 训练 / 开发验证 / 最终封存测试分离（A04）

- **三分数据**：训练集、开发验证集、最终封存测试集。开发迭代（含调参、因子筛选、自动修复）只允许使用训练 + 开发验证。
- **冻结后开测**：代码、参数、数据版本（snapshot_id）全部冻结并登记后，才允许运行最终封存测试；测试只运行一次口径下的评估，**结果不得回流当前策略的自动修复循环**。
- **修复边界**：实现错误（计算错误、数据处理 bug）可在开发集修复并记录 diff；**策略经济表现不佳记为实验失败**，不允许以提高最终测试收益为目标循环"修到通过"（修复代码错误与修改研究假设分开登记）。
- **访问日志**：最终测试的每次访问、由谁发起、读取了哪些结果，以及结果是否回流，均留日志；已消费的测试窗口不再称为"未见样本"。
- **切分边界按标签跨度处理**：训练段末端与验证/测试段起点之间须留出≥标签持有期的隔离带（如 T+1→T+2 标签需在边界空出对应跨度），防止跨界标签泄漏。
- **保存全部实验痕迹**：失败实验、评审否决记录、参数变体全部落盘（MLflow run + `experiments/<id>/`），支持"每次实验能定位数据分段与试验家族"。
- **删除"入库因子数量"硬指标**：因子准入以证据质量判断，**零有效因子允许正常验收**。
- **验收思路**：开发 Agent/流程不能读取最终测试结果；每次实验可定位数据分段与试验家族；封存集访问记录完整。

### 2.5 手算对齐验收（A02/A03 共用）

- 用**小型手工行情**（数只股票 × 数十交易日）逐笔手算信号、持仓、费用、净值，与引擎输出逐项对齐；误差超出预定义容差即回测不可信。
- 用例必须覆盖：①**收盘信号**（T 日收盘信号 → T+1 执行）；②**跨节假日**（"下一交易日"语义正确）；③**不可当日卖出**（T+1 交收持仓锁生效）；④买入失败、卖出失败、部分成交、停牌与恢复交易（现金、数量、费用、净值守恒，见审计 §7 验收矩阵）。

---

## 3. 因子研究与回测框架

### 3.1 项目逐项评估

#### 3.1.1 Microsoft qlib —— ★ 主基座（首期唯一引擎）

- **GitHub**：https://github.com/microsoft/qlib
- **Stars**：48,746（2026-09-22）；Forks 7,709
- **维护状态**：**非常活跃**（最后推送 2026-09-22；持续合并 PR，RD-Agent 联动更新）
- **License**：MIT（开源代码许可；个人自用/分发/对外服务三种方式下均无 copyleft 障碍）
- **定位**：AI-Oriented Quant Investment Platform，覆盖"数据→因子/数据集→模型→回测→分析→在线服务"全链路
- **A 股支持**：
  - **复权**：数据以"原始价 + `$factor` 复权因子"表达（`exchange.py` 中 `$factor` 用于复权还原与整手取整；缺 factor 时自动退化为"复权价撮合"并告警）。研究侧可用表达式自行构造复权价；撮合侧按真实价 + factor 还原，天然避免复权价成交的失真。
  - **涨跌停**：`Exchange(limit_threshold=...)` 原生支持——float（如 0.1，按 `$change` 判断）或自定义表达式（区分买入/卖出涨停），命中限制的标的当日不可交易；`C.region=REG_CN/REG_TW` 时未设置会主动告警（源码核实）。
  - **T+1**：日频工作流天然满足"T 日信号 → T+1 日调仓成交"的时序；但撮合层（`exchange.py`）**无显式"当日买入不可当日卖"持仓锁**——对每日一次调仓的 TopK 类策略无实际影响，若做日内/分钟级嵌套执行需在策略/持仓层自行约束（对应 §2.1 契约"可卖数量"要素）。
  - **费用**：`open_cost` / `close_cost`（可分别模拟买入佣金与卖出印花税+佣金）、`min_cost`（最低 5 元）、`impact_cost`（冲击成本/滑点，建议 0.1%）、`volume_threshold`（容量/成交量约束）均原生支持（源码核实）。
  - **其他 A 股细节**：`trade_unit=100`（整手交易，注释明确写 "trade unit, 100 for China A market"）、停牌（`$close` 为 NaN 即停牌不可交易）、批量涨跌停可用 `extra_quote` 补充 ETF 等标的。
- **复用判断**：**直接复用为主 + 少量改造**（数据源接入、label/费用参数按 A 股校准；严格 T+1 需小改）
- **推荐理由**：功能覆盖最全、社区最大（48.7k stars）、MIT 许可、与 MLflow 原生集成、内置 Alpha158/Alpha360 与 20+ 基准模型，详见第 8 章重点评估。首期按审计 §4 沿用 Qlib，先简单基线与手算案例（§2.5）。

#### 3.1.2 RQAlpha（Ricequant）

- **GitHub**：https://github.com/ricequant/rqalpha
- **Stars**：6,787；Forks 1,796
- **维护状态**：**活跃**（最后推送 2026-09-22，open issues 仅 31）
- **License**：**自定义双轨**——非商业用途按 Apache-2.0；**任何商业用途需米筐科技（Ricequant）书面授权**（LICENSE 原文核实："未经米筐科技授权，任何个人不得出于任何商业目的使用本软件……任何法人或其他组织不得出于任何目的使用本软件"）。**A08 口径**：附加非商业限制不等于禁止个人自用，但须**按具体用途逐条检查**（个人本地研究 vs 对外服务/商业）；"进程隔离/不并入代码"不自动消除许可义务，实际使用方式与条款原文对照后判断（条款解释**待核实**完整法律意见，本调研不作合规认证）。
- **A 股支持**（A 股原生，中文文档 rqalpha.readthedocs.io/zh_CN）：
  - 复权：与数据源绑定（RQData/自建 bundle），支持分红除权处理；
  - 涨跌停：撮合引擎（`sys_simulation`）+ 风控（`sys_risk`）支持涨跌停拒绝成交（依赖数据中的涨跌停价）；
  - T+1：股票持仓模型区分"可卖数量（closable）"，**原生 T+1**；
  - 费用：`sys_transaction_cost` mod 实现股票/期货**佣金、印花税**等税费计算；
  - 事件驱动撮合（bar/tick），支持 Mod Hook 扩展。
- **复用判断**：**可直接复用**（做 A 股事件驱动回测/模拟撮合），但用途受非商用条款约束，按上述 A08 口径分场景判断
- **推荐理由**：A 股交易规则建模最贴近国内实盘（T+1/涨跌停/印花税开箱即用）、中文文档与社区（QQ 群）、Mod 机制灵活。数据侧 RQData 为商业数据源（可换自建数据）。可做 qlib 的"高保真撮合验证"补充层，但按审计 §4 **不同时维护两套引擎**，仅在确有验证需求时引入。

#### 3.1.3 backtrader —— 替代候选（A08 修订：重新列入）

- **GitHub**：https://github.com/mementum/backtrader
- **Stars**：23,302；Forks 5,289
- **维护状态**：**半停滞**（最后推送 2024-08-19；issues 已关闭，仅偶发合并 PR）
- **License**：**GPL-3.0**。**A08 口径重写**：GPL 的公开源代码义务因**分发**衍生作品而触发；**个人本地使用与修改不因使用本身要求公开所有个人代码**（GNU FAQ S4，[GNU FAQ](https://www.gnu.org/licenses/gpl-faq.html#GPLRequireSourcePostedPublic)，完整条款**待核实**——审计仅取得搜索索引摘要）。v1 将 GPL 整体列为"传染性禁区"的口径不适用于个人自用场景，故**重新列入候选**。若未来公开发布代码或对外提供服务，再按分发/网络服务情形另行判断。
- **A 股支持**：
  - 复权：交给数据源（自备前/后复权数据）；
  - 涨跌停：**无原生支持**（需自定义 Broker/数据过滤）；
  - T+1：**无原生支持**（需自定义 sizer/broker 逻辑）；
  - 费用：`CommissionInfo` 支持佣金/印花税/最低佣金，可自定义。
- **复用判断**：**需中等改造**（A 股交易规则需自建，须实现 §2.1 执行契约的全部要素）；维护停滞是主要保留意见
- **推荐理由**：经典事件驱动框架、教程与第三方资料极多、指标/分析器（Analyzer）丰富，适合学习与快速搭建事件驱动原型。**定位为 qlib 的替代候选**：仅当 qlib A 股适配测试（审计 §4 放行条件）不通过时评估切换；**不要同时维护 qlib 与 backtrader 两套回测口径**，避免两套结果互相矛盾。

#### 3.1.4 vnpy（VeighNa）

- **GitHub**：https://github.com/vnpy/vnpy
- **Stars**：45,502；Forks 12,486
- **维护状态**：**活跃**（最后推送 2026-09-13）
- **License**：MIT（社区版；vnpy_pro 等周边为商业产品——模型/服务订阅条款与代码许可分开记录）
- **A 股支持**：
  - 定位是**交易/实盘接入平台**（CTA、期权、多因子等应用层在 vnpy_org 各仓库），回测为事件驱动 bar 级（vnpy_ctabacktester / vnpy_factor）；
  - 费率/滑点/合约乘数/保证金可配；**T+1、涨跌停需在策略或子引擎自行处理**（回测引擎偏期货设计）；
  - 复权依赖数据源（RQData/自建数据库）。
- **复用判断**：**部分复用**——研究/回测用 qlib 更合适；vnpy 的价值在**实盘柜台接口**（CTP、券商接口等）与模拟交易
- **推荐理由**：国内实盘生态最完整（45.5k stars），若系统未来要接实盘，用 vnpy 做交易执行层、qlib 做研究层是常见组合。

#### 3.1.5 zipline-reloaded

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

#### 3.1.6 Hikyuu

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
- **推荐理由**：A 股原生 + 中文 + 高性能，非常适合传统规则型选股/择时的快速批量验证；但**机器学习因子研究管线（数据集/模型管理）弱于 qlib**。定位为 qlib 的"极速验证沙盒"或传统策略引擎（P2 按需引入）。

#### 3.1.7 Alphalens / alphalens-reloaded

- **GitHub**：https://github.com/quantopian/alphalens （原版）；https://github.com/stefan-jansen/alphalens-reloaded （续维护 fork）
- **Stars**：alphalens 4,449；alphalens-reloaded 653
- **维护状态**：alphalens **停滞**（最后推送 2024-02-12，Quantopian 已关闭）；alphalens-reloaded **低频维护**（最后推送 2025-12-15）
- **License**：均为 Apache-2.0
- **A 股支持**：**与市场无关**（输入 = 因子值 DataFrame + 收益率序列）；A 股的复权收益、涨跌停/停牌处理需在输入数据层进行。**A03 口径修订**：涨跌停/停牌等不可交易样本的处理仅限**因子诊断**（IC/分层）且必须"含/不含"双口径对照、记录剔除数量与理由；**回测净值层面禁止删除未成交样本**（§2.3），两者不可混用。
- **复用判断**：**直接复用**（推荐装 `alphalens-reloaded`）
- **推荐理由**：单因子分析事实标准——IC/分层（Quantile）收益/换手率（Turnover Analysis）/因子衰减（Mean Period Wise Return）/信息系数分析一键出图，是"因子体检"标配。首期仅对**少量预先声明的简单因子**使用（审计 §4）。

#### 3.1.8 empyrical / empyrical-reloaded

- **GitHub**：https://github.com/quantopian/empyrical ；https://github.com/stefan-jansen/empyrical-reloaded
- **Stars**：empyrical 1,513；empyrical-reloaded 121（ecosyste.ms 核实）
- **维护状态**：empyrical **半停滞**（最后推送 2024-07-26）；empyrical-reloaded **低频维护**（约 2025 年底仍有推送，发布至 0.5.9）
- **License**：均为 Apache-2.0
- **A 股支持**：纯收益序列→指标计算（Sharpe/Sortino/Max Drawdown/Calmar/Omega/尾部风险等），**与市场无关**；年化因子需按 A 股 244 交易日调整
- **复用判断**：**直接复用**（作为函数库被 zipline/pyfolio/quantstats 引用）
- **推荐理由**：轻量、指标函数齐全，适合嵌入自定义研究代码做滚动/分段指标计算；但单独使用偏底层，报告呈现不如 quantstats。

#### 3.1.9 quantstats

- **GitHub**：https://github.com/ranaroussi/quantstats
- **Stars**：7,651；Forks 1,235
- **维护状态**：**活跃**（最后推送 2026-07-20）
- **License**：Apache-2.0
- **A 股支持**：纯收益序列分析，**与市场无关**（基准可用 000300/000905 收益序列）；默认 252 年化交易日需改为 244
- **复用判断**：**直接复用**
- **推荐理由**：`qs.reports.html()` 一键生成 tearsheet（Sharpe/Sortino/Max DD/Calmar/尾部/月度热力图/滚动指标），个人研究"复盘报告"性价比最高；社区活跃，另有 quantstats-reloaded 等分支可换。

#### 3.1.10 backtesting.py

- **GitHub**：https://github.com/kernc/backtesting.py
- **Stars**：8,982；Forks 1,534
- **维护状态**：**活跃**（最后推送 2026-08-05）
- **License**：**AGPL-3.0**。**A08 口径重写**：AGPLv3 的义务须**按条款区分**"修改"、"网络交互"与"源代码提供"三种情形（[AGPLv3 原文](https://www.gnu.org/licenses/agpl-3.0.html)、[GNU 许可使用说明](https://www.gnu.org/licenses/gpl-howto.en.html)，S5；原文页面在审计时超时，**待核实**完整条款）：分发修改版触发 copyleft；仅通过网络与修改版交互的用户享有获取对应源代码的权利（AGPLv3 §13）——**不能简单等同"网络服务也算分发"**。个人本地自用、公开分发、对外网络服务三种情形分别判断；个人本地自用本身不触发公开全部个人代码的义务。
- **A 股支持**：**单标的**回测（一个品种 OHLCV），佣金/滑点可配；**无复权/涨跌停/T+1 概念**（需数据层自理）
- **复用判断**：**直接复用**（仅限单标的择时/技术策略原型；若对外提供网络服务前须按 AGPLv3 §13 评估源代码提供义务）
- **推荐理由**：API 极简、内置指标与热力图/滚动优化（`backtesting.lib`+ optuna 集成示例），几分钟验证一个择时想法。注意单标的局限与 AGPL 条款分场景判断。

#### 3.1.11 vectorbt（补充推荐）

- **GitHub**：https://github.com/polakowo/vectorbt
- **Stars**：9,152
- **维护状态**：**活跃**（最后推送 2026-09-17）
- **License**：**自定义（NOASSERTION）**——非标准开源许可，条款须逐条审查后按用途判断（A08：非标准许可 + 可能的附加限制须检查具体用途；另有商业版 vectorbt PRO，其商业条款与代码许可**分开记录**）
- **A 股支持**：向量化组合回测，费用可自定义；涨跌停/T+1 无原生（信号层过滤）
- **复用判断**：**直接复用**（大规模参数扫描/组合级向量化回测），许可条款审查（**待核实**）完成前不作为核心依赖
- **推荐理由**：NumPy 向量化引擎，秒级跑完上千参数组合，适合策略参数空间粗筛。注意：参数扫描会放大多重检验问题，搜索区间与试验家族须按 §2.4 记录。

### 3.2 框架对比总表

| 项目 | GitHub | Stars | 维护状态 | License（按 A08 分场景判断） | 引擎类型 | 前/后复权 | 涨跌停 | T+1 | 手续费/印花税 | 复用判断 | 推荐度 |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| **qlib** | microsoft/qlib | 48,746 | 活跃 | MIT | 向量化信号 + 半事件执行 | ✅ `$factor` 复权因子 | ✅ `limit_threshold` | ⚠️ 时序满足，无持仓锁（契约层补） | ✅ open/close/min/impact cost | 直接复用+少量改造 | ★★★★★ 主基座（首期唯一） |
| **backtrader** | mementum/backtrader | 23,302 | 半停滞(2024-08) | GPL-3.0（个人自用不触发公开义务，S4；分发另判） | 事件驱动 | ⚠️ 数据自理 | ❌ | ❌ | ✅ CommissionInfo | 需中等改造 | ★★★☆ **替代候选**（不与 qlib 并行维护） |
| **RQAlpha** | ricequant/rqalpha | 6,787 | 活跃 | Apache-2.0（**非商业附加限制**，按用途检查） | 事件驱动 | ✅ 数据层 | ✅ 撮合/风控 | ✅ 原生（closable） | ✅ 佣金+印花税 mod | 直接复用（用途受限） | ★★★（按需复核撮合） |
| **Hikyuu** | fasiondog/hikyuu | 3,522 | 活跃 | Apache-2.0 | 事件驱动（C++ 内核） | ✅ 前/后复权 | ⚠️ 条件过滤 | ⚠️ 延迟一日成交近似 | ✅ 可配 | 直接复用 | ★★★★ |
| **vnpy** | vnpy/vnpy | 45,502 | 活跃 | MIT | 事件驱动 | ✅ 数据层 | ⚠️ 需自处理 | ⚠️ 需自处理 | ✅ 费率/滑点/保证金 | 部分复用（实盘层） | ★★★☆ |
| **zipline-reloaded** | stefan-jansen/zipline-reloaded | 1,941 | 低频维护 | Apache-2.0 | 事件驱动 | ❌ | ❌ | ❌ | ✅ 佣金/滑点模型 | 需大量改造 | ★★☆ |
| **backtesting.py** | kernc/backtesting.py | 8,982 | 活跃 | AGPL-3.0（修改/网络交互/源码提供分情形，S5） | 混合（单标的） | ❌ | ❌ | ❌ | ✅ 基础 | 直接复用（原型） | ★★★ |
| **vectorbt** | polakowo/vectorbt | 9,152 | 活跃 | 自定义(NOASSERTION)，条款待核实 | 向量化 | ⚠️ 数据自理 | ❌ | ❌ | ✅ 可定制 | 直接复用（参数扫描） | ★★★ |
| **Alphalens(-reloaded)** | quantopian/alphalens / stefan-jansen/alphalens-reloaded | 4,449 / 653 | 停滞 / 低频维护 | Apache-2.0 | 因子分析（非回测） | 输入自理 | 输入自理（诊断口径，A03） | 不适用 | 不适用 | 直接复用 | ★★★★★ 单因子标配 |
| **empyrical(-reloaded)** | quantopian/empyrical / stefan-jansen/empyrical-reloaded | 1,513 / 121 | 半停滞 / 低频维护 | Apache-2.0 | 指标库 | 不适用 | 不适用 | 不适用 | 不适用 | 直接复用 | ★★★☆ |
| **quantstats** | ranaroussi/quantstats | 7,651 | 活跃 | Apache-2.0 | 绩效报告 | 不适用 | 不适用 | 不适用 | 不适用 | 直接复用 | ★★★★★ 报告标配 |

图例：✅ 原生支持；⚠️ 部分/需配置；❌ 无，需改造。

---

## 4. Alpha / 因子挖掘路线（**延后项**，依据审计 §4）

> **审计 §4 决定**：首期因子分析**限少量预先声明的简单因子及对照**；**RD-Agent、遗传规划（gplearn）、自动因子生产线延后**。原因：执行契约（A02）与样本完整性（A03）未对齐前，自动挖掘只会放大过拟合与多重检验问题（A04）。本章保留 v1 已核实的调研事实与来源，作为 G4 阶段按需引入时的参考，**不进入首期实施**。

### 4.1 首期因子口径

- 首期因子清单在实验前**预先声明**（表达式、经济含义、预期方向、评估窗口），不做事后海量筛选；
- 每个因子配一个**对照**（如随机因子/朴素基准），判定标准以证据质量而非因子数量；
- **零有效因子允许正常验收**（A04：删除"入库因子数量"硬指标）。

### 4.2 延后候选（事实保留，v1 核实于 2026-09-22）

#### 4.2.1 Microsoft RD-Agent（qlib 官方生态）

- **GitHub**：https://github.com/microsoft/RD-Agent
- **Stars**：14,711；Forks 1,925
- **维护状态**：**非常活跃**（最后推送 2026-09-15；微软持续投入，配套论文 R&D-Agent-Quant, arXiv:2505.15155）
- **License**：MIT
- **能力**：LLM Multi-Agent 自动化 R&D——**因子挖掘**（含"从研报提取因子"）+ **模型优化**两阶段闭环；与 qlib 深度集成（自动生成因子代码→在 qlib 上回测评估→迭代进化）；官方描述"automated factor mining and model optimization in quant investment R&D"（qlib README 核实）。
- **复用判断**：**延后**（原"直接复用"降级）——引入条件：G2 可信回测验收通过、§2.4 封存机制运转正常；且其"回测评估→迭代进化"循环必须圈定在训练+开发验证内，不得触碰最终封存测试。

#### 4.2.2 alphagen

- **GitHub**：https://github.com/ICT-FinD-Lab/alphagen（AlphaGen 论文官方实现，中科院 ICT-FinD-Lab）
- **Stars**：1,242；Forks 331
- **维护状态**：**低频维护**（最后推送 2026-06-04，仍有零星提交）
- **License**：**无 LICENSE 文件（GitHub API license: null）**。**A08 口径**：**无许可证不等于允许复制**——默认保留所有权利，"公开可见"不授予复制/修改/再分发权利；仅可参考思路，直接复用代码存在法律风险，商用/复用前必须联系作者取得授权。
- **复用判断**：**参考/借鉴为主**（延后）；不建议直接依赖。

#### 4.2.3 gplearn（遗传规划 / 符号回归）

- **GitHub**：https://github.com/trevorstephens/gplearn
- **Stars**：1,889
- **维护状态**：**低频维护**（最后推送 2026-08-14；功能稳定，长期无大改）
- **License**：BSD-3-Clause
- **能力**：scikit-learn API 风格的遗传规划（SymbolicTransformer / SymbolicRegressor），自定义 `function_set` + `init_depth` 即可做符号回归式因子挖掘；适应度函数可自定义（如最大化 IC）。
- **复用判断**：**延后**（GP 自动挖掘属"自动因子生产线"范畴，按审计 §4 推迟至 G4）。

#### 4.2.4 表达式因子挖掘（qlib 表达式引擎自建管道）

- **依托**：qlib `qlib.data.ops`（Ref/Mean/Std/Corr/Cov/Rank/Quantile/Resi/Slope/IdxMax/EMA/WMA/Delta 等 50+ 滚动算子，API 文档核实）+ `advanced/alpha.html`（Building Formulaic Alphas 教程）
- **复用判断**：qlib 表达式引擎本身**照常使用**（承载第 4.1 节少量预先声明因子）；自动搜索器（GP/RL/LLM）延后。
- **推荐理由**：表达式因子"可解释 + 可增量计算 + 天然进 qlib 数据管道"，是个人研究性价比最高的因子形态。

### 4.3 因子挖掘对比总表（延后参考）

| 项目 | GitHub | Stars | 维护状态 | License | 挖掘范式 | 复用判断 | 推荐度 |
| -- | -- | -- | -- | -- | -- | -- | -- |
| RD-Agent | microsoft/RD-Agent | 14,711 | 活跃 | MIT | LLM Multi-Agent（因子+模型） | 延后（G4 条件引入） | ★★★★ |
| alphagen | ICT-FinD-Lab/alphagen | 1,242 | 低频维护 | **无（保留所有权利；≠允许复制）** | RL 生成因子集合 | 仅参考思路 | ★★ |
| gplearn | trevorstephens/gplearn | 1,889 | 低频维护 | BSD-3-Clause | 遗传规划/符号回归 | 延后（G4） | ★★★ |
| qlib 表达式引擎 | microsoft/qlib | — | 活跃 | MIT | 表达式算子组合（作为因子载体） | 直接复用（限少量声明因子） | ★★★★★ |

---

## 5. 绩效与风险指标计算（IC / 分层 / 换手 / 风险 / 稳定性 / 样本外）

### 5.1 指标 → 工具映射

| 指标 / 分析 | 含义与要点 | 推荐工具（复用） |
| -- | -- | -- |
| **IC（Pearson）** | 因子值与次期收益（§2.2 标签口径）的截面相关系数；看均值、IC>0 比例、IC 衰减 | Alphalens `factor_information_coefficient`；qlib `analysis_model`（IC 图） |
| **RankIC** | 秩相关（Spearman），对离群值稳健，A 股截面更常用 | qlib 内置 Rank IC（`analysis_model` 报告含 IC/Rank IC）；或 pandas `spearmanr` 自算滚动窗口 |
| **分层收益（Quantile）** | 5/10 分组多空净值、多头-空头价差；检验单调性 | Alphalens `create_returns_tear_sheet`（Quantile Analysis）；qlib 分组累计收益图 |
| **换手率** | 因子/组合换手 → 决定成本敏感性；Alphalens Turnover Analysis | Alphalens `create_turnover_tear_sheet`；qlib `indicator_analysis` |
| **最大回撤 / Sharpe / Sortino / Calmar / Omega** | 组合风险收益指标（基于**执行净值**而非标签收益，A02） | quantstats（tearsheet 全覆盖）/ empyrical-reloaded（函数级）/ qlib `risk_analysis` |
| **因子稳定性** | 滚动 IC/RankIC 序列、IC_IR（IC 均值/IC 标准差）、分段一致性、换手分解 | 滚动窗口自算（pandas）+ quantstats 滚动图；建议自定义 ~50 行脚本（唯一少量自研点） |
| **样本外 / 滚动回测** | Walk-forward：滚动训练+滚动预测+滚动回测，防过拟合——**机制组件**，不替代 §2.4 封存测试 | **qlib `RollingGen`**（Task Management，任务滚动生成）+ `benchmarks_dynamic`（Rolling Retraining 基线、DDG-DA）；Optuna 嵌套外层且不得触碰封存集 |
| **分市场阶段回测** | 牛/熊/震荡/流动性危机分段对比 | 按日期区间切分后分别跑 qlib 回测 + quantstats 分段 tearsheet（无现成"一键"工具，切分脚本 <30 行） |
| **多空/基准超额** | 相对 000300/000905 的超额收益、信息比率、跟踪误差 | qlib `risk_analysis`（excess return with/without cost，`qrun` 直接输出）；quantstats `reports.html(benchmark=...)` |

### 5.2 评估要点

- **IC/RankIC、分层、换手**三件套是单因子准入标准：Alphalens(-reloaded) 一键覆盖，**直接复用**；A 股使用时注意：①收益用后复权收益；②不可交易样本（涨跌停/停牌）的处理**仅限因子诊断**且必须"含/不含"双口径对照、剔除数量与理由入档（A03）——**回测净值层面禁止删除未成交样本**，旧仓未成交继续持有和估值（§2.3）；③ST/次新过滤只用决策时已知标记；④默认 252 年化改 244。
- **组合级指标**用 quantstats（报告）+ qlib `risk_analysis`（研究管道内嵌），empyrical-reloaded 备用；所有绩效数字标注其口径是"标签收益（预测代理）"还是"执行净值"（§2.2）。
- **滚动/样本外**：qlib 原生 `RollingGen` 支持滚动任务生成（训练/验证/测试段滚动），是免费拿到的"滚动回测"能力，无需自研；但滚动测试段属于开发验证迭代的一部分，**最终封存测试仍按 §2.4 单独隔离**，切分边界按标签跨度留隔离带。
- **分市场阶段**属于研究方法而非组件，用日期切分循环调用上述工具即可（约 30 行胶水代码，是本方案中极少数需自写的部分）。

### 5.3 组合优化与权重分配

| 需求 | 推荐 | 说明 |
| -- | -- | -- |
| TopK/固定名额调仓 | qlib `TopkDropoutStrategy`（topk + n_drop） | 个人量化最常用范式，无需优化器；首期基线即用此范式 |
| 按权重/指数增强 | qlib `WeightStrategyBase` / `EnhancedIndexingStrategy` | 按目标权重调仓基类、对标指数主动增强 |
| 显式组合优化（均值方差/风险平价/BL） | **cvxpy**（Apache-2.0）自建目标函数；Riskfolio-LP（GPL-3.0）可作现成实现参考——**A08 口径**：个人本地使用/修改 GPL 库不因使用本身要求公开个人代码（S4）；仅在公开分发或对外服务时按分发条款另行判断 | 个人系统 P2 再引入；按使用方式记录许可判断 |
| 组合风险归因 | quantstats + 自研暴露分解（行业/风格中性化残差） | Portfolio/持仓分析消费 |

> 判断：MVP/第一阶段用 TopK 范式即可覆盖绝大多数选股策略；组合优化属第二阶段增量。

---

## 6. 实验追踪与结果管理（含 A04 记录要求）

### 6.1 项目逐项评估

| 项目 | GitHub | Stars | 维护状态 | License | 定位与能力 | 与 qlib/量化研究结合 | 复用判断 |
| -- | -- | -- | -- | -- | -- | -- | -- |
| **MLflow** | mlflow/mlflow | 28,095 | 活跃（2026-09-22 推送） | Apache-2.0 | 实验追踪（Params/Metrics/Artifacts）、模型注册、部署 | **qlib `QlibRecorder` 基于 MLflow 实现**（Experiment/Recorder/Record Template 体系，文档核实），qrun 的实验记录可直接用 MLflow UI 比较 | **直接复用（首选）** |
| **Weights & Biases** | wandb/wandb | 11,258 | 活跃 | MIT（SDK）；SaaS/私有化服务（服务订阅条款与代码许可分开记录） | 实验追踪、Sweeps 超参搜索、系统监控、报告协作 | 图形与对比体验最好；数据上云需评估隐私 | 直接复用（备选，图形化更好） |
| **Kedro** | kedro-org/kedro | 11,006 | 活跃 | Apache-2.0（LICENSE.md 为标准 Apache-2.0 文本） | 数据科学流水线框架（DataCatalog + Pipeline DAG + 约定式工程结构） | 把"数据→因子→模型→回测"组织成可复现节点图，可包裹 qlib 调用 | 可选复用（工程化阶段再引入） |
| **DVC** | treeverse/dvc（原 iterative/dvc，已迁移组织） | 15,881 | 活跃 | Apache-2.0 | 数据/模型文件版本化（git-like）、管道与实验表格（`dvc exp`） | 版本化 qlib 数据快照、回测产物 | 可选复用（数据 >GB 后引入） |
| **Optuna** | optuna/optuna | 14,832 | 活跃（2026-09-18 推送） | MIT | 超参搜索（TPE/CMA-ES/Grid/Random）、剪枝、分布式、Dashboard | 搜模型超参（LightGBM/NN）与策略参数（TopK/n/drop）；与 MLflow 集成简单；**搜索目标只允许开发验证集指标**（A04） | **直接复用（首选）** |
| **qlib qrun / QlibRecorder** | microsoft/qlib | — | 活跃 | MIT | YAML 驱动"数据集→训练→回测→评估"一键流水线；Recorder 记录 SignalRecord/SigAnaRecord/PortAnaRecord 产物 | 即主基座自带实验记录，底层 MLflow | 直接复用 |

### 6.2 组合建议与记录规则

1. **实验记录**：优先直接吃 qlib 红利——`qrun` + QlibRecorder（MLflow 后端）已经把每次实验的参数、预测、IC 分析、回测报告落盘；配 `mlflow ui` 即得实验对比界面。无需额外接入成本。
2. **调参**：Optuna 包住"训练+回测"目标函数（如最大化开发验证段 IR 或 RankIC），实验记录通过 `mlflow.set_tracking_uri` 复用同一后端；**最终封存测试集不得作为调参目标**（A04）。
3. **W&B**：如果更看重图表与对比 UI，可在模型训练脚本里双写 W&B（几行代码），云端看板体验优于 MLflow UI。
4. **Kedro / DVC**：个人研究初期不必引入；当数据（多版本 qlib_bin、分钟线）超 10GB 或流程节点 >10 个时再上，避免过度工程。
5. **A04 强制记录**（不论用哪个工具）：
   - **失败实验、否决记录、参数变体**全部保留（run 标签区分 `status=failed/rejected/variant`），可定位数据分段（snapshot_id）与试验家族（experiment family id）；
   - **最终封存测试访问日志**：谁在何时访问了封存结果、结果是否回流开发闭环，单独留痕；
   - **不设"入库因子数量"指标**：验收只看证据链完整性，零有效因子允许正常验收。

---

## 7. 回测引擎类型对比：向量化 vs 事件驱动

| 维度 | 向量化回测（Vectorized） | 事件驱动回测（Event-Driven） |
| -- | -- | -- |
| **原理** | 信号/持仓/收益对整段历史用矩阵运算一次算出 | 按时间逐 bar/tick 回放，走"事件→策略→撮合→记账"循环 |
| **速度** | 快 1~3 个数量级（NumPy/Pandas 并行） | 慢（Python 循环 + 撮合逻辑），全市场多年回测分钟级起 |
| **仿真精度** | 粗糙：成交假设简化（当日全成/固定滑点），难精确建模资金占用、部分成交、涨跌停拒单 | 精细：可逐单撮合，建模 T+1、涨跌停、停牌、整手、最小佣金、成交量约束 |
| **适用场景** | 因子研究、大样本统计检验、参数扫描、组合权重回测、想法快速证伪 | 细节敏感策略（打板、隔夜、日内执行）、交易规则约束强的验证、模拟盘/实盘对接 |
| **典型代表** | Alphalens（因子级）、vectorbt、qlib 的信号分析层、quantstats（事后分析） | RQAlpha、backtrader、zipline、vnpy、Hikyuu（近似）、qlib 的 Exchange/Executor 层 |
| **A 股微观结构** | 需在信号层规避（过滤涨跌停/停牌样本） | 可在撮合层精确拦截（qlib `limit_threshold`、RQAlpha `sys_risk`） |

**本系统结论（v2 收敛）**：

1. **研究段用向量化**：少量预先声明因子的 IC/分层初筛天然是截面向量运算——Alphalens + qlib 表达式引擎，把"想法→证据"的迭代压到分钟级（审计 §4：首期不做自动挖掘，故 vectorbt 参数扫描亦非必需）；
2. **验证段用 qlib 半事件撮合**：入围策略进入 qlib Exchange 回测（涨跌停/停牌/整手/费用/容量全建模），并按 §2.1 执行契约声明成交价、持有期与费用口径；
3. qlib 恰好是**混合型**（向量化特征管道 + 事件化执行器 Executor/Exchange，还支持 Nested Executor 嵌套日内执行），一套框架内完成上述两段——这是选它做主基座的重要理由。**只维护这一套回测口径**；backtrader 仅作替代候选（§3.1.3）。

---

## 8. 重点评估：Microsoft qlib 作为主回测基座的可行性

> 依据：GitHub API 元数据、官方 ReadTheDocs（v0.9.8.dev）、`qlib/backtest/exchange.py` 源码、官方 README、社区 issue/教程交叉核实。

### 8.1 数据格式与 Point-in-Time 数据库

- **数据格式**：紧凑二进制"qlib 格式"——按 instrument 分目录的 `.day.bin` 特征文件 + calendar/day.txt + instruments/all.txt + features/<code>/*.day.bin；官方基准显示其数据层带 ExpressionCache/DatasetCache 时比 HDF5/MySQL/MongoDB/InfluxDB 快 4~50 倍（README 性能表：14 特征 × 800 股 × 2007–2020 任务，最优 7.4s vs MySQL 365s）。
- **数据摄入**：CSV/Parquet → qlib 格式转换器（`dump_bin`）；Yahoo 爬虫脚本（`scripts/data_collector/`，含 1d/1min）；**官方 cn_data 数据集因数据合规政策暂时下线**，README 明确推荐社区镜像 [chenditc/investment_data](https://github.com/chenditc/investment_data/releases)（个人 A 股研究的现成数据源）。也可用 AkShare/tushare 数据自建 qlib_bin。
- **Point-in-Time 数据库**：官方支持（PR #343，2022-03 发布），专门解决**财报修订造成的数据泄漏**：PIT 表每特征 4 列（date 披露日 / period 报告期 / value / _next 链表指针），带 `.index` 索引文件，支持"任意历史时点取当时可见版本"；配 `scripts/data_collector/pit/` 爬虫+转换器（支持财报 PIT）。**已知限制**（文档原文）：仅面向季度/年度因子（财报类），PIT 计算未做极致优化。对基本面因子研究这是关键能力。注意审计 A01 的边界：PIT 表**补不出不存在的历史版本**，历史字段无版本证据时按 `historical_pit_unverified` 处理（MVP 可暂不使用财务因子）。
- **判断**：✅ 满足需求；PIT 用于财报因子防前视，价量因子靠表达式时序（`Ref`）天然 point-in-time。

### 8.2 Alpha158 / Alpha360 数据集

- 官方"Quant Dataset Zoo"收录 **Alpha158**（158 个价量统计特征）与 **Alpha360**（360 个近端价量原始特征），US/China 双市场可用（README 核实），定义在 `qlib/contrib/data/handler.py`；
- 20+ 基准模型（LightGBM/XGBoost/CatBoost/LSTM/GRU/ALSTM/Transformer/GATs/TRA/HIST/TFT/TabNet 等）在这两个数据集上有一致的 benchmark 结果表（`examples/benchmarks/README.md`），可作为自研模型的对照基线；
- **Label 注意（A02 修订）**：Alpha158 默认 label 为 `Ref($close,-2)/Ref($close,-1)-1`，其语义是 **T+1 收盘至 T+2 收盘收益**（审计 S1；社区经典疑问见 issue #1514）。该标签可作为"T+1 收盘成交、持有至 T+2 收盘"这一执行假设下的**预测代理**，但**必须声明与最终执行收益的差异**（费用、滑点、部分成交、停牌/涨跌停拒单、可卖数量、持有期错位），**不得标注为"与执行假设一致"**；自定义执行假设时按 §2.1 契约同步重定义 label 并保留对照说明。
- 自定义数据集：复制 handler 配置改 features 表达式即可（Alpha158 本质是一份 YAML 化表达式清单）。
- **判断**：✅ 直接复用，且是可自定义的模板。

### 8.3 Handler / DataLoader

- 分层清晰（文档 `component/data.html` 核实）：
  - **DataLoader**：`QlibDataLoader`（表达式→面板）、`StaticDataLoader`（外部 DataFrame/文件），接口 `load()` 可自定义；
  - **DataHandlerLP**：加载 + 预处理（Processor 链：缺失值填充、标准化、去极值、行业市值中性化等可插拔）+ 磁盘缓存；
  - **Dataset**：`DatasetH` 支持滚动切片（segments: train/valid/test），与 `RollingGen` 配合生成滚动任务——切分边界须按 §2.4 标签跨度留隔离带。
- 自定义因子 = 在 features 表达式里加一行（如 `"Ref($close,1)/$close-1"`），或注册新算子（`qlib.data.ops.register_all_ops`）。
- **判断**：✅ 直接复用 + 配置化扩展，无需造数据管道轮子。

### 8.4 模型接口（自定义因子 / 模型）

- `qlib.model.base.Model`：`fit(dataset)` / `predict(dataset)` 两个方法即接入（官方"Custom Model Integration"文档给出完整步骤与配置文件写法）；`ModelFT` 支持微调；
- 模型无关：sklearn / LightGBM / PyTorch / TF 均可包装；Alpha158+LightGBM 是官方 demo（`qrun` 30 行配置跑通全流程）；
- 内置 20+ 论文模型可直接跑对照；Meta-Learning（DDG-DA）与 RL 框架（订单执行）另有模块。
- **判断**：✅ 接口最小化（fit/predict），自定义成本极低。

### 8.5 回测模块（TopKDropout 等策略、A 股微观结构）

- **策略**（`qlib.contrib.strategy`）：`TopkDropoutStrategy`（TopK 持仓 + 每日最多换 Drop 只，最经典的公募指增/量化选股范式）、`SoftTopkStrategy`、`EnhancedIndexingStrategy`（对标指数的主动增强）、`WeightStrategyBase`（按权重调仓基类）、`TWAPStrategy`、`SBBStrategyEMA` 等；
- **执行**：`Executor`/`NestedExecutor`（支持日内嵌套执行，1min 高频示例）、`Exchange` 撮合（第 3.1.1 节已核实的 A 股能力）：
  - `deal_price` 可指定买卖分别用 `$open/$close/$vwap` 等不同成交价；
  - `limit_threshold` 涨跌停（float 或表达式，区分买/卖方向）；
  - `trade_unit=100` 整手 + `$factor` 复权还原取整；
  - `open_cost/close_cost/min_cost/impact_cost`（费用含最低佣金与冲击成本），`volume_threshold` 成交量/容量约束（含分钟级 DayCumsum 累计成交量约束）；
  - 停牌（NaN close）自动不可交易；REG_CN/REG_TW 场景有专门告警。
- **输出**：`qrun` 直接给出"无成本/有成本超额收益"的 mean/std/annualized_return/information_ratio/max_drawdown（README 示例核实）；`PortAnaRecord` 落盘回测明细——但**股票池/下单候选/成交/持仓四类记录须按 §2.3 分层落盘**，不能只留汇总净值。
- **T+1 补充说明（A02 修订）**：日频 TopK 工作流"T 日收盘出信号 → T+1 日调仓"天然满足时序；**严格"T+1 日买入股份当日锁定"在撮合层未建模**（exchange.py 源码核实无持仓锁），须按 §2.1 契约"可卖数量"要素在策略/持仓层显式约束——这是唯一需要小改造的点。
- **判断**：✅ 直接复用（TopK/指数增强场景）；⚠️ 严格 T+1、日内交易规则需少量改造。

### 8.6 qrun 与实验记录机制

- **qrun**：一条命令跑通"构建数据集→训练→预测→信号分析→组合回测→评估"（README 核实），YAML 即实验定义（`workflow_config_lightgbm_Alpha158.yaml` 为模板）；上游工作流入口为 `qrun`，命令须按锁定版本验证（审计 S3）；
- **QlibRecorder**（文档 `component/recorder.html` 核实）：Experiment/Recorder 分层管理，`log_params/log_metrics/log_artifact/save_objects` 齐全，Record Template（SignalRecord→SigAnaRecord→PortAnaRecord）自动沉淀每次实验的预测值、IC 分析、回测报告；底层为 **MLflow**（`set_uri` 可指向共享 MLflow server），`search_records` 跨实验检索对比；亦提供 `workflow_by_code` Python 化编排；
- 任务管理：`TaskManager` + `RollingGen` 支持滚动回测任务生成与收集（多实验汇总 Collector）。
- **判断**：✅ 自带轻量实验追踪（MLflow 后端），与第 6 章选型无缝衔接；A04 的失败/否决/封存访问日志在其上以 run 标签与独立日志补足。

### 8.7 中文文档与社区活跃度

- **官方文档**：ReadTheDocs 英文（结构完整：Quick Start / Data / Model / Strategy / Recorder / Report / PIT / FAQ / API）；官方 README 有中文媒体解读（微软公众号"微矿Qlib"等），**但无官方中文文档站**；
- **中文社区资料**：丰富——中文教程翻译（如 wuzao.com 的中文文档镜像）、DeepWiki 自动生成的中文解读、vnpy 社区"Alpha158 标签"等专题讨论、大量知乎/公众号实战文章；A 股用户基数大（国内量化圈主流研究框架之一）；
- **社区活跃度**：48.7k stars、7.7k forks、510 watchers、近期日更（2026-09-22 仍有推送）、Gitter 实时群 + GitHub Issues（open 480，响应尚可）、微软研究院持续投入（RD-Agent 联动）。
- **判断**：✅ 社区活跃度顶级；⚠️ 官方文档英文为主，中文靠社区——对本项目影响有限（技术文档可读即可）。

### 8.8 Apple Silicon（Mac mini）安装与性能

- **安装**：
  - PyPI 包 `pyqlib` 支持 macOS（README 平台徽章：linux | windows | macos），Python 3.8–3.12 均有 pip/源码安装支持；
  - **官方 M1 提示**（README "Tips for Mac" 原文）：M1 上构建 LightGBM wheel 可能因缺 OpenMP 失败，`brew install libomp` 后即可构建成功——即 Apple Silicon 可用，但需装系统依赖；
  - 历史兼容性摩擦：官方 issue #1525 "Apple M1 not supported" 记录了早期 M1 支持问题（该 issue 页面调研时多次抓取失败，仅核实到标题）；当前 README 已把 macOS 列为正式平台且给出 M1 方案，社区（如 chenditc/investment_data、各中文教程）在 Apple Silicon 上的使用报告普遍正面；
  - 源码安装含 Cython 扩展编译，需 Xcode Command Line Tools；建议 conda 管理环境（README 明确建议）。
- **性能**：
  - qlib 数据层为紧凑二进制 + 两级缓存（ExpressionCache/DatasetCache），CPU 核数越多数据集构建越快（README 基准：1CPU 147s → 64CPU 8.8s）；Mac mini（M 系 8–20 核）多核收益明显；
  - 模型训练（LightGBM/PyTorch）在 Apple Silicon 上有原生 arm64 加速（PyTorch MPS 可用于 NN 模型）；
  - **注意**：qlib 官方无 Apple Silicon 公开基准，上述为架构推断 + 社区经验，落地后应以本机实测（建议以 Alpha158+LightGBM `qrun` 全流程计时为基准测试）——**待核实（需本机实测）**。
- **判断**：✅ 可安装可用（`brew install libomp` 一行解决主要坑）；性能够个人研究用，需本机实测定标。

### 8.9 可行性结论与改造清单

**结论：qlib 作为主回测基座完全可行**，理由：MIT 许可、48.7k 社区、A 股微观结构（涨跌停/停牌/整手/费用/容量）原生建模、PIT 防前视、Alpha158/360 开箱、fit/predict 极简模型接口、qrun+MLflow 实验记录、Apple Silicon 可用。首期按审计 §4 **沿用 Qlib、先简单基线与手算案例**；backtrader 仅作替代候选，不同时维护两套。

**需要的改造/适配（工作量小）**：

| # | 改造点 | 量级 | 说明 |
| -- | -- | -- | -- |
| 1 | 数据源接入 | 小 | 优先评估社区镜像 chenditc/investment_data 公开 Release 作历史底座（审计 §11 接入路径）；缺口用 AkShare/tushare 补 |
| 2 | 费用参数校准 | 极小 | 按实际券商佣金（约 0.01–0.03%，最低 5 元）+ 卖出印花税（当前 0.05%）设 `open_cost/close_cost/min_cost`，作为 §2.1 契约"费用"要素 |
| 3 | 涨跌停参数 | 极小 | 主板 10%/ST 5%/创业板科创板 20%/北交所 30%——用 `limit_threshold` 表达式形式按板块区分，且按**历史制度生效期**区分（如创业板 2020-08 前 10%） |
| 4 | label 定制 | 小 | 按 §2.1 契约定义 label，并随结果输出 §2.2 的"标签 vs 执行收益"差异声明 |
| 5 | 严格 T+1 持仓锁 | 小-中 | 在 Strategy/Position 层加"当日买入不可卖"约束（契约"可卖数量"要素） |
| 6 | 成交/持仓分层留痕 | 小 | 按 §2.3 落盘信号股票池/下单候选/成交/持仓四类记录与未成交原因 |
| 7 | 封存测试隔离与日志 | 小 | 按 §2.4 实现三分数据、冻结登记、访问日志（验收见审计 §7） |

---

## 9. 推荐技术栈组合与实施路线（v2 收敛）

### 9.1 目标架构（自上而下）

```
因子来源（首期）：少量预先声明的简单因子（qlib 表达式引擎）+ 对照
        ↓                          ［延后：RD-Agent（LLM）｜ gplearn（GP）｜ alphagen 思路（RL）］
因子评估层：Alphalens-reloaded（IC/RankIC/分层/换手，双口径对照）+ 自有滚动稳定性脚本
        ↓ 通过准入的因子（零有效因子允许验收）
研究基座层：Microsoft qlib（qlib_bin 数据 + PIT 财报库 + Handler/Dataset + 模型 + RollingGen 滚动训练）
        ↓ 信号（按 §2.1 执行契约声明口径）
回测验证层：qlib Exchange（涨跌停/T+1 时序/费用/容量 + 契约层持仓锁）［替代候选：backtrader，不并行维护］
        ↓ 执行净值（与标签收益分列，§2.2）
报告分析层：quantstats tearsheet + qlib risk_analysis（超额/IR/回撤）
        ↓ 产物
实验管理层：MLflow（QlibRecorder 原生）+ Optuna（调参，仅开发验证集）［失败/否决/变体全留痕，A04］
隔离层：最终封存测试（冻结后开测，访问留日志，结果不回流）
（未来实盘）：vnpy 交易执行层
```

### 9.2 实施路线（建议顺序）

1. **执行契约与手算案例（先行，A02/A03）**：写定 §2.1 执行契约 → 构造 §2.5 小型手工行情用例（收盘信号、跨节假日、不可当日卖出、买卖失败/部分成交/停牌）→ 逐笔手算与引擎输出对齐；
2. **数据**：Mac mini 上装 pyqlib（`brew install libomp`）→ 按审计 §11 路径落地 chenditc/investment_data 公开 Release 快照（记录 URL/SHA-256/快照 id）→ 财务 PIT 可后置（A01 边界内）；
3. **简单基线跑通（审计 §4）**：`qrun` 跑 Alpha158 + LightGBM 官方基准作对照，实测本机性能并校准费用/涨跌停参数；先简单策略（TopK）+ 手算对齐，不追求挖掘规模；
4. **因子分析闭环（限少量）**：接 alphalens-reloaded + quantstats，对**预先声明**的少数简单因子跑"因子准入卡"（IC、RankIC、IC_IR、分层单调性、换手、最大回撤，含/不含不可交易样本双口径）；
5. **样本外封存机制（A04）**：定义训练/开发验证/最终封存测试三分与标签跨度隔离带 → 冻结登记流程 → 访问日志；`RollingGen` 滚动训练模板在开发验证内使用；
6. **实验管理**：起本地 MLflow tracking（QlibRecorder 默认），Optuna 做模型与策略参数搜索（仅开发验证集目标），失败/否决/参数变体全落盘；
7. **（延后）挖掘增强**：G4 阶段按需评估 RD-Agent/gplearn，且其迭代循环圈定在训练+开发验证内；
8. **（可选）** backtrader 替代评估（仅当 qlib A 股适配测试不通过）/ RQAlpha 高保真复核 / vnpy 实盘 / Kedro-DVC 工程化。

---

## 10. 风险与注意事项

### 10.1 License 风险（A08 口径重写，分四类 + 分使用方式）

**记录框架**：①开源代码许可；②模型许可（如 Hermes-4 的 Apache-2.0 权重许可）；③数据使用条款（如 Tushare 积分协议、investment_data 上游数据条款）；④服务订阅条款（如 API 服务额度）。四类**分开记录**，不互相推定（代码许可不覆盖数据再分发授权，Apache-2.0 代码不等于上游数据可再分发）。

**开源代码许可按使用方式分别判断**：

- **GPL-3.0（backtrader、Riskfolio-LP）**：公开源代码义务因**分发**衍生作品触发；**个人本地使用、修改不因使用本身要求公开所有个人代码**（GNU FAQ S4，[链接](https://www.gnu.org/licenses/gpl-faq.html#GPLRequireSourcePostedPublic)；完整条款**待核实**，审计仅得搜索索引摘要）。个人自用场景不再整体回避 GPL；公开发布代码或对外服务前再按分发情形评估。
- **AGPL-3.0（backtesting.py）**：按条款区分**修改**（分发修改版触发 copyleft）、**网络交互**（AGPLv3 §13：通过网络与修改版交互的用户可获取对应源代码）、**源代码提供义务**三种情形（[AGPLv3](https://www.gnu.org/licenses/agpl-3.0.html)、[GNU 指引](https://www.gnu.org/licenses/gpl-howto.en.html)，S5；原文**待核实**——审计时页面超时）。**不能简单等同"网络服务也算分发"**；本地自用、分发、对外网络服务分别判断。
- **无 License（alphagen；Ashare 同理）**：**无许可证不等于允许复制**——默认保留所有权利，仅参考思路，不复制代码进本系统。
- **附加非商用限制（RQAlpha）**：不等于禁止个人自用，但**须检查具体用途**（个人本地研究 vs 商业/对外服务）对照条款原文；**进程隔离/不并入代码不自动消除许可义务**——隔离是工程措施，法律判断以条款与实际使用方式为准。
- **自定义/非标准许可（vectorbt NOASSERTION）**：逐条审查条款后按用途判断，审查完成前不作核心依赖。
- **MIT/Apache-2.0/BSD（qlib/MLflow/Optuna/Kedro/DVC/vnpy/Hikyuu/gplearn 等）**：三种使用方式下均无 copyleft 障碍。

本调研**不作全依赖法律合规认证**；实际采用的每个依赖须记录版本、许可证原文链接与使用方式（本地自用/公开发布/对外网络服务分列判断）。

### 10.2 数据风险

qlib 官方数据集因合规暂时下线；Yahoo 源数据质量一般（官方原话 "might not be perfect"），严肃研究建议优先走公开数据包质量验收（审计 §11.3 检查表）+ `check_data_health.py`。个人使用不自动获得上游数据抓取或再分发授权（数据使用条款与代码许可分开记录）。

### 10.3 前视偏差与口径一致性

财报因子务必走 PIT 通道（否则用"最新修订版"会泄漏未来；注意 A01：PIT 表补不出不存在的历史版本，无版本证据字段按 `historical_pit_unverified` 阻断）；**标签一律按 §2.2 作为预测代理并声明与执行收益的差异，禁止标注"与执行假设一致"**；Alphalens 因子诊断中对不可交易样本的处理须双口径对照（§5.2），不得回流净值计算（A03）。

### 10.4 过拟合与多重检验

挖掘类工具（RD-Agent/gplearn/alphagen，现均延后）会系统性放大过拟合——即使未来引入，也必须以"封存测试 + 换手成本后执行收益"为最终准绳，且迭代循环圈定在训练+开发验证（A04）；经济表现不佳记为失败，不得循环"修到通过"；保存失败/否决/参数变体；不设因子数量指标，零有效因子正常验收。

### 10.5 平台风险

backtrader/empyrical/alphalens 原版均处于停滞/半停滞：分析类工具一律用续维护 fork（alphalens-reloaded/empyrical-reloaded）；backtrader 虽停滞但因事件驱动生态资料丰富保留为替代候选（§3.1.3），启用前须完成 A 股适配与 §2.5 手算对齐。

---

## 11. 来源 URL 清单

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

**v2 修订新增（审计依据，转引自审计方案 §9）：**

- **S1** Qlib 数据与标签定义：https://github.com/microsoft/qlib/blob/main/docs/component/data.rst 、https://github.com/microsoft/qlib/blob/main/qlib/contrib/data/handler.py （支撑 §2.2/§8.2 标签语义）
- **S3** Qlib 运行方式：https://github.com/microsoft/qlib/blob/main/docs/introduction/quick.rst （支撑 §8.6 `qrun` 口径）
- **S4** GPL 私人修改与公开源代码问题（GNU FAQ）：https://www.gnu.org/licenses/gpl-faq.html#GPLRequireSourcePostedPublic （支撑 §3.1.3/§10.1；审计仅取得搜索索引摘要，完整条款**待核实**）
- **S5** AGPL 使用与网络交互：https://www.gnu.org/licenses/gpl-howto.en.html 、https://www.gnu.org/licenses/agpl-3.0.html （支撑 §3.1.10/§10.1；原文页面审计时超时，**待核实**）
- **S6** Backtrader 项目及 GPL-3.0 标识：https://github.com/mementum/backtrader （支撑 §3.1.3 重新列入候选）
