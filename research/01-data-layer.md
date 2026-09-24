# A 股量化研究系统 · 数据层调研报告

> 场景：个人 A 股量化研究系统，部署于 Mac mini（Apple Silicon），单机运行。
> 原则：**尽量复用现成开源项目与数据服务，不自研采集系统**。
> 核实时间：2026-09-22（stars / License / 最近提交均通过 GitHub API、PyPI 元数据实时抓取核实；调研日期之后请以链接为准）。

> **修订记录 v2（2026-09-23，依据审计 A01/A05/A09/A10/A11/§11）**
>
> 本次修订依据[《个人 A 股量化系统审计与公开数据优先整改方案》](../personal-quant-audit-plan.md)（审计日期 2026-09-23，基线 `anwu-d/docs@afbebb3d`）就地改写，落实以下审计发现：
>
> - **A01（P0）**：PIT 历史版本不可补造。引入"市场已公开时间 vs 本系统实际获得时间"双可见性口径并记录所用口径；规定来源文档编号/内容哈希/原始披露时间/修订披露时间/首次采集时间/版本/时区字段要求；无披露时刻采用保守下一交易时段规则，不允许默认当日开盘可用；无法恢复原始版本的字段标记 `historical_pit_unverified` 并阻止进入声称无前视的历史研究；MVP 可暂不使用财务因子。
> - **A05 + 审计 §11**：数据策略由"三源采集自建全量"改为**"公开数据包初始化 + 定期同步 + 质量验收 + 按需补缺"**；investment_data 采用固定 release tag + 固定 commit 的 `qlib/validate_archive.py` 校验（`--expected-tag` / `--require-publishable`）+ 独立快照目录原子切换；接入路径与放行检查整理为可执行清单（G1 执行项，本次未执行）；BaoStock/AKShare 降为补充层与抽样对账。
> - **A09（P1）**：单写入者产生不可变 Parquet 快照 + manifest，读任务绑定 `snapshot_id`，SQLite 只存任务元数据；仅在证明需要多进程写服务后才考虑数据库升级；独立介质或异地加密备份，目标 RPO 24 小时 / RTO 1 天并以实测确认；不以同机 MinIO 代灾难恢复。
> - **A10（P1）**：每个数据集记录截至时间、预期日期与就绪状态；旧数据不得伪装为当天数据。
> - **A11（P1）**：首期按来源文档 ID + 内容哈希去重，保留修订链，不凭标题相似直接丢弃。
> - **成本口径（审计 v2 修正）**：统计下载/磁盘/备份/维护成本；Tushare 等付费源仅在公开渠道无法满足**已确认**需求时评估，**不预定购买档位**——原"2000 积分档强烈推荐 / 200 元/年必买"结论按审计口径撤销。
>
> 修订原则：保留 2026-09-22 已核实事实与来源 URL；审计未重核项标注"**待核实**"；**stars 数不作为可用性证据**；与审计冲突的结论一律按审计口径改写。

---

## 0. 结论速览（TL;DR，v2 修订版）

| 用途 | 首选 | 备选/交叉校验 | 说明 |
|---|---|---|---|
| 历史日线底座 | **investment_data 公开 Qlib 数据包**（固定 release tag 初始化 + 定期同步） | baostock / akshare 抽样对账 | 公开数据包优先，不从零重建全量采集（A05/§11） |
| 数据入库流程 | **公开包初始化 → 定期同步 → 质量验收 → 按需补缺** | — | 验收不过不放行；缺口逐项建清单再决定补充来源 |
| 补充与对账层 | **baostock**（稳定免费、含退市股、复权因子） | akshare（广度兜底）、efinance | 降为补充层与抽样对账；明确优先级、单位与复权口径，**不静默拼接进第三方二进制包** |
| 缺口补充（财务/两融/龙虎榜/申万等） | 先建**缺口清单** | 公开源逐项补齐 | Tushare 等付费源**仅在公开渠道无法满足已确认需求时评估**，不预定购买档位 |
| PIT / 防前视 | 双可见性口径（市场已公开时间 / 本系统实际获得时间）+ 版本字段要求 | qlib PIT 文件格式可复用 | 历史版本**不可补造**；无法恢复的字段标记 `historical_pit_unverified` 并阻断；**MVP 暂不使用财务因子**（A01） |
| 数据就绪与出报 | 每数据集记录截至时间、预期日期、就绪状态 | 初版标注缺项 → 补齐 → 次日晨间修订 | 固定出报时点 ≠ 数据齐备（A10） |
| 落地存储 | **单写入者 → 不可变 Parquet 快照 + manifest**；读任务绑定 `snapshot_id`；DuckDB 只读查询；SQLite 只存任务元数据 | 数据库升级仅在证明需要多进程写服务后考虑 | 不引入服务数据库（A09） |
| 备份 | **独立介质或异地加密备份 + 恢复验证** | 同机 MinIO 仅作本地快照仓库 | 目标 RPO 24 小时 / RTO 1 天，以实测确认；同机 MinIO **不代**灾难恢复（A09） |

一句话推荐（v2）：**investment_data 公开 Qlib 数据包（固定 tag + 固定 commit 校验）初始化历史日线底座、定期同步新包并原子切换快照 → 质量验收（§10 清单）→ baostock/akshare 作补充层与抽样对账 → 缺口清单驱动按需补缺**；PIT 采用双可见性口径并阻断 `historical_pit_unverified` 字段；存储为单写入者不可变 Parquet 快照 + manifest，独立介质加密备份。自写部分仅限快照/manifest 管理、ETL 调度、就绪状态登记、来源 ID+内容哈希去重（保留修订链）与质量校验。

---

## 1. 调研范围与评估维度

- 范围：①行情（日线/分钟线）；②财务/基本面与商业数据 API 付费边界；③指数/行业概念板块/资金流/两融/北向南向/龙虎榜；④宏观/利率/汇率/大宗商品/美股港股；⑤开源数据框架（qlib、hikyuu、rqalpha）数据层；⑥（v2 新增）公开数据包（investment_data）接入与验收。
- 每个候选给出：GitHub 地址、stars、维护活跃度、License、数据覆盖、更新频率、历史完整性、收费、稳定性与合规性（是否网页抓取、ToS 风险）、推荐理由。
  - **口径声明（v2）**：stars / 最近提交仅为 2026-09-22 的参考快照（审计未重核，**待核实**），**不作为可用性证据**；可用性以接口实测与数据质量验收为准。
- 额外：数据落地选型（Parquet 快照/DuckDB/SQLite/MinIO）、增量更新与去重、数据质量校验（停牌/复权/除权除息/退市与幸存者偏差）、数据就绪状态（A10）。
- **成本口径（v2）**：免费数据同样统计**下载、磁盘、备份、维护**成本；付费数据只在公开渠道无法满足已确认需求时评估，不预定购买档位。

---

## 2. 行情数据（日线 / 分钟线）

> **v2 定位调整**：本节候选的**角色**按审计 A05/§11 下调——历史日线底座优先用 investment_data 公开数据包初始化，下列采集库降为**补充层与抽样对账**（补字段、多源校验、缺口补齐）。表格中的 stars / License / 价格为 2026-09-22 快照，审计未重核，**待核实**。

### 2.1 总对比表

| 项目 | GitHub | Stars | 最近提交 | License | 日线 | 分钟线 | 实时 | 历史深度 | 收费 | 数据获取方式 | ToS/合规风险 | v2 定位 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **akshare** | [akfamily/akshare](https://github.com/akfamily/akshare) | 22,692 | 2026-09-20（活跃） | MIT | ✅ 全市场含港美 | ✅ 1/5/15/30/60 分 | ✅ | 日线自上市（东财源可至 1991） | 免费 | 抓取东财/新浪/同花顺等公开网页接口 | ⚠️ 灰色（非官方授权 API） | 补充层/兜底与抽样对账 |
| **efinance** | [Micro-sheep/efinance](https://github.com/Micro-sheep/efinance) | 4,067 | 2026-07-17（较活跃） | MIT | ✅ | ✅ | ✅ | 日线全历史 | 免费 | 抓取东方财富公开接口 | ⚠️ 灰色 | 轻量对账备选 |
| **baostock** | 官网 [baostock.com](https://www.baostock.com)（无官方 GitHub 主仓） | — | PyPI 0.9.4 @ 2026-09-21（活跃） | BSD | ✅ 含退市股 | ✅ 5/15/30/60 分（指数除外） | ❌ | 日线自 1990 年代末（约 1999，含退市股）、分钟线约 2011 起 | 免费 | 自有数据服务器（TCP API），**非抓取** | ✅ 低 | 补充层第一对账源 |
| **tushare** | [waditu/tushare](https://github.com/waditu/tushare) | 15,409 | 2024-03-13（客户端停滞，服务端持续运营，站点 ICP 2026） | BSD-3-Clause | ✅ | ✅（独立付费，2009 起） | ✅（独立付费） | 日线长历史、分钟 2009 起 | 积分制收费（详见 §3.3） | 官方 HTTP API | ✅ 低（商业数据服务） | **付费候补**：仅缺口确认后评估（§3.3/§11） |
| **qstock** | [tkfy920/qstock](https://github.com/tkfy920/qstock) | 1,940 | 2025-03-16（更新慢） | MIT | ✅ | ✅ | ✅ | 日线全历史 | 免费 | 抓取东财/新浪/同花顺/百度 | ⚠️ 灰色 | 不入核心 |
| **Ashare** | [mpquant/Ashare](https://github.com/mpquant/Ashare) | 3,869 | 2025-12-24 | **无 License**（默认保留所有权利） | ✅ | ✅（分时/分钟） | ✅（新浪/腾讯双源） | 短 | 免费 | 抓取新浪/腾讯行情 | ⚠️ 灰色 + 无授权条款 | 不建议引入 |
| **easyquotation** | [shidenggui/easyquotation](https://github.com/shidenggui/easyquotation) | 5,398 | 2026-02-28 | MIT | ❌（仅快照） | ❌ | ✅ 全市场快照（新浪/腾讯/集思录） | — | 免费 | 抓取新浪/腾讯 | ⚠️ 灰色 | 实时快照场景可选 |

> stars/最近提交来源：GitHub REST API（`api.github.com/repos/...`），抓取于 2026-09-22；审计（2026-09-23）未重核，**待核实**。

### 2.2 逐项详评

#### akshare —— 广度兜底，补充层
- **GitHub**：[akfamily/akshare](https://github.com/akfamily/akshare)；22,692 stars；MIT；最近提交 2026-09-20，近乎日更（**待核实**）。
- **覆盖**：A 股/港股/美股行情（日/周/月/分钟 K、实时快照）、指数、行业与概念板块、资金流、融资融券、龙虎榜、宏观（国家统计局/央行/外汇局）、利率（Shibor/LPR 等）、汇率、大宗商品、基金/可转债/期权/期货、三大财务报表等 1000+ 接口（[官方文档](https://akshare.akfamily.xyz/)）。
- **更新频率**：随上游站点（东财/新浪/同花顺等）实时或日度。
- **历史完整性**：个股日线自上市起（东财示例数据可至 1991 年，见 [stock.md](https://github.com/akfamily/akshare/blob/main/docs/data/stock/stock.md)）；**分钟线历史深度有限**（东财分钟接口通常只有近几年，1 分钟更短）。
- **收费**：完全免费、无 token。
- **稳定性/合规**：这是 akshare 的主要短板——它通过解析公开网页接口获取数据，上游改版会导致接口失效（如 [Issue #6143](https://github.com/akfamily/akshare/issues/6143)、[Issue #7180](https://github.com/akfamily/akshare/issues/7180)、[Issue #6574](https://github.com/akfamily/akshare/issues/6574)），修复虽快但属于"追着上游跑"；数据使用处于各网站 ToS 的灰色地带，**个人研究可接受，不可作为对外商业产品的数据来源**。
- **v2 推荐理由**：作为**补充层与抽样对账**——补板块/资金流/两融/龙虎榜等非行情字段，以及对 investment_data 快照做独立抽样对账；不承担历史底座。

#### efinance —— 东财数据的干净封装
- **GitHub**：[Micro-sheep/efinance](https://github.com/Micro-sheep/efinance)；4,067 stars；MIT；最近提交 2026-07-17（较活跃，但 open issues 153，响应一般；**待核实**）。
- **覆盖**：股票（日/周/月/分钟 K、实时）、资金流、基金、债券、期货；以东方财富为唯一数据源。
- **收费**：免费。**历史**：日线全历史，分钟线与 akshare 东财源同深度。
- **稳定性/合规**：同 akshare（东财抓取），单源依赖。接口简洁、返回规整 DataFrame，比 akshare 更轻。
- **推荐理由**：作为东财源的轻量替代/交叉校验；不建议作为唯一来源。

#### baostock —— 免费+自有服务器的对账"压舱石"
- **发布渠道**：官网 [baostock.com](https://www.baostock.com) + [PyPI](https://pypi.org/project/baostock/)（无活跃 GitHub 主仓，社区镜像/工具仓另存）。**License：BSD**（PyPI 元数据 "License :: OSI Approved :: BSD License"）。
- **维护活跃度**：PyPI 发版记录——0.8.9（2024-05）之后沉寂，**2024-04-14 之后 2026 年连续发版：0.9.1（2026-04）、0.9.2（2026-06）、0.9.3（2026-07）、0.9.4（2026-09-21）**，即当前仍在积极维护（数据源自 2026 年示例数据可查；**待核实**）。
- **覆盖**（API 明细见 [baostock-skill API 总结](https://github.com/atompilot/baostock-skill/blob/master/skills/baostock/SKILL.md)）：
  - K 线：日/周/月 + 5/15/30/60 分钟；不复权/前复权/后复权（`adjustflag`）+ **复权因子**（`query_adjust_factor`）+ **分红送股**（`query_dividend_data`，含除权除息日）；
  - 行内字段自带 `preclose`、`tradestatus`（停牌标记）、`isST`、PE/PB 等估值；
  - 财务：季频盈利能力/营运/成长/偿债/现金流/杜邦 + 业绩预告/快报（含 `pubDate`/`statDate`）；
  - 基础：交易日历、全证券列表、股票基本信息（**ipoDate/outDate/status，含退市股**）、行业分类（证监会）、上证50/沪深300/中证500 成分；
  - 宏观：存贷款利率、准备金率、货币供应量。
- **更新频率**：日度（交易日收盘后）。
- **历史完整性**：长历史（社区与教学资料记载日线约自 1999 年，含已退市股票；分钟线约自 2011 年；指数不支持分钟线）——精确起点以官方知识库为准。
- **收费**：免费，无积分无频率硬限制。
- **稳定性/合规**：**自有数据服务器 + 专用 TCP 协议**，不抓网页，合规风险最低；接口十年不变，最稳。缺点：功能面窄（无板块概念/资金流/两融/北向/龙虎榜）、ETF 覆盖有限、所有字段返回字符串、pandas 2.x 下 `get_data()` 需手工迭代规避（README 已注明）、复权采用"涨跌幅复权法"与通达信定点复权略有差异。
- **v2 推荐理由**：**补充层第一对账源**——含退市股、有复权因子与除权除息明细，适合对公开数据包做独立抽样对账并补交易状态/复权因子字段。注意：其财务字段只含 `pubDate`/`statDate`，**不等于**历史 PIT 多版本（见 §9.5 / A01）。

#### tushare（Pro）—— 付费候补，不预定购买档位（v2 口径修订）
- **GitHub**：[waditu/tushare](https://github.com/waditu/tushare)；15,409 stars；BSD-3-Clause。客户端库最近提交 2024-03-13（基本不再更新，但 API 稳定）；服务端 tushare.pro 持续运营（官网页脚 ICP 许可证为 2026 年签发）（**待核实**）。
- **收费事实（2026-09-22 核实，审计未重核，待核实）**（[积分与频次权限对应表](https://tushare.pro/document/2?doc_id=290)）：
  - 120 积分（免费注册）：仅非复权日线，50 次/分；
  - 2000 积分档：日线/复权/股本/分红/财务/两融/龙虎榜等大部分常规接口，200 次/分（原文记录为 200 元/年，以[官方页面](https://tushare.pro/document/1?doc_id=290)为准）；
  - 5000 积分档：常规数据无总量上限；10000 积分档：特色数据（盈利预测、筹码分布、券商金股等）；
  - **独立付费项**：历史分钟线（2009 起）、实时分钟、美股日线、新闻/公告等（各计费方式见官网）。
- **覆盖**：[接口目录](https://tushare.pro/document/2)涵盖基础/行情/财务（三大表+分红+审计+主营构成）/参考数据（股东、质押、解禁、大宗）/两融与转融通/资金流/打板专题（龙虎榜、涨跌停等）/指数专题（申万、中信、国际指数）/公募基金/期货/期权/债券/外汇/港股/美股/宏观/公告与新闻。
- **更新频率**：日度为主，部分实时（付费）；**历史完整性**：长历史（分钟自 2009、港股财报自 2000）。
- **稳定性/合规**：官方授权数据服务（HTTP API + 积分体系），无 ToS 风险，字段规范、有文档。
- **v2 推荐口径（取代原"200 元/年档强烈推荐"结论）**：按审计 §5.1 成本原则，**Tushare 等付费来源只在公开渠道无法满足已确认需求时评估；不预定某周购买，也不承诺某积分档覆盖所有接口**。行动顺序：先用公开数据包 + 免费源建立**缺口清单**（缺失的未复权价格、交易状态、行业成分历史、财务披露及事件逐项登记），仅对确认无法补齐的缺口才评估付费方案与档位。原第 3 周"开通 2000 积分"的计划**撤销**。

#### qstock / Ashare / easyquotation —— 轻量补充，不入核心
- **qstock**（[tkfy920/qstock](https://github.com/tkfy920/qstock)，1,940 stars，MIT，最近提交 2025-03，更新慢）：数据+选股+回测一体的"个人投研包"，数据源与 akshare 高度重叠（东财/新浪/同花顺），深度更窄。作为数据层无增量价值。
- **Ashare**（[mpquant/Ashare](https://github.com/mpquant/Ashare)，3,869 stars，最近提交 2025-12）：单文件极简实时行情（新浪/腾讯双源自动切换）。**注意：仓库未声明任何 License**（GitHub API `license: null`），法律上默认保留所有权利，**不建议引入到正式代码库**；如需同类能力用 easyquotation（MIT）。审计 A08 补充：无许可证不等于允许复制，引入前须按使用方式逐项判断。
- **easyquotation**（[shidenggui/easyquotation](https://github.com/shidenggui/easyquotation)，5,398 stars，MIT，最近提交 2026-02）：新浪/腾讯/集思录实时快照批量拉取，适合盘中监控场景。

### 2.3 行情层结论（v2 修订）
- **历史日线底座**：investment_data 公开 Qlib 数据包（§10），固定 release tag 初始化、定期同步、质量验收后原子切换。
- **补充层与抽样对账**：**baostock（主对账）+ akshare（广度兜底）**；用于补交易状态/复权因子/非行情字段，并对公开包做独立多源抽样对账（§10 放行检查第 5 项）。
- **优先级与口径规则**：公开包 > baostock > akshare/efinance；任何补充数据入库必须显式记录**单位**（价格/成交量/金额口径）与**复权口径**（不复权/前复权/后复权/复权因子），且**不静默拼接进第三方二进制包**——补充数据放独立层（Parquet/DuckDB 视图），需要合并时生成新的自有快照并保留来源标识。
- **分钟线**：MVP 不需要分钟线；免费方案只有 baostock（2011 起、5 分钟粒度起）与东财抓取（深度浅）；2009 年起 1 分钟级历史属付费缺口，按 §3.3 口径待需求确认后评估，不预定购买。

---

## 3. 财务 / 基本面数据

### 3.1 免费三件套对比（事实保留；结论见 §3.3 v2 口径）

| 能力 | baostock | akshare | tushare（免费 120 积分） | tushare（积分付费档） |
|---|---|---|---|---|
| 利润表/资产负债表/现金流量表 | ❌（仅衍生指标） | ✅（东财/新浪抓取） | ❌ | ✅（标准三大表） |
| 季频财务指标（ROE/成长/偿债/现金流/杜邦） | ✅ 含 pubDate | ✅ | ❌ | ✅ |
| 分红送股/除权除息明细 | ✅（含除权日、税前税后现金分红） | ✅ | ❌ | ✅ |
| 业绩预告/快报 | ✅ | ✅ | ❌ | ✅ |
| 股本变动/每日股本 | 部分（totalShare/liqaShare） | ✅ | ❌ | ✅（盘前股本或为独立付费项） |
| 股东/质押/解禁/大宗 | ❌ | ✅ | ❌ | ✅ |
| 财报披露日期（PIT 关键字段） | ✅（pubDate） | 部分 | ❌ | ✅（财报披露日期表） |
| 历史财务修订（PIT 多版本） | ❌ | ❌ | ❌ | ❌（**任何源都不能补造历史版本，见 §9.5/A01**；商业 PIT 见 RQData/JQData，亦属付费候补） |

### 3.2 商业数据 API 的免费/付费边界（事实保留，2026-09-22 快照，待核实）

| 服务 | 免费边界 | 付费情况 | 覆盖特点 | 合规 | v2 建议 |
|---|---|---|---|---|---|
| **tushare Pro** | 注册 120 积分（仅非复权日线） | 积分档与独立付费项（§2.2，以[官方说明](https://tushare.pro/document/1?doc_id=290)为准） | A 股结构化数据最全，含宏观/港美 | ✅ 官方 API | 付费候补：仅缺口确认后评估档位 |
| **JQData（聚宽）** | 线上研究/回测环境免费；本地 SDK 曾开放公测（[公告](https://www.joinquant.com/view/community/detail/4d575f700a403ed4186eaef6ec57d4e7)） | 本地化 JQData 按年订阅收费（价格以[官方说明](https://joinquant.com/help/api/doc?id=9827&name=logon)为准，页面为动态渲染需登录查看） | 行情/财务/行业概念/资金流/两融齐全，数据清洗质量高，支持 PIT 财报 | ✅ 官方 API | PIT 财报付费选项之一，仅确认缺口后评估 |
| **RQData（米筐）** | 试用期免费（[官网](https://www.ricequant.com/welcome/rqdata)） | 付费年订（社区反馈价格见 [vnpy 论坛](https://www.vnpy.com/forum/topic/29498-qing-wen-yi-xia-3000yuan-shou-jie-de-radata-bao-han-gu-piao-shu-ju-yao)，以[官方报价](https://rqopen.ricequant.com/welcome/pricing)为准） | A 股 2005 至今+实时、全历史财务 + **官方 PIT API**、基金/期货/期权/可转债、风格因子、宏观（见 [rqalpha PyPI 说明](https://pypi.org/project/rqalpha/)） | ✅ 官方 API | 同上：需要"财报 Point-in-Time"且确认无法自建覆盖时的付费选项 |
| **通联数据 DataAPI（datayes/优矿）** | 早年开放免费 token API（tushare 曾内置 [datayes 模块](https://github.com/waditu/tushare/blob/master/docs/datayes.rst)，现为历史遗留文档） | 当前主要面向机构/优矿平台销售，个人免费通道有限、公开文档维护少 | 基本面/因子/宏观曾是强项 | ✅ 官方 API | **不推荐**作为个人系统依赖（个人渠道不确定） |

> **PIT（Point-in-Time）提示（v2 按审计 A01 改写）**：原"免费源只给最新修订值 + 披露日期，自建 PIT 表（记录每次抓取的版本）即可防未来信息"的说法**不成立**——**历史 PIT 版本不可补造**：现在抓到的修订值若被放回早期披露日期，即构成前视；仅新增 `fetched_at` 或 `_next` 链**本身无法修复**历史缺失版本。正确做法见 §9.5：双可见性口径 + 版本字段要求 + 保守下一交易时段规则 + `historical_pit_unverified` 阻断；MVP 可暂不使用财务因子。付费 PIT API（RQData/JQData）也只解决其覆盖起点之后的版本，须按缺口清单评估。

---

## 4. 指数、行业/概念板块、资金流、两融、北向/南向、龙虎榜

> v2 定位：本节数据属**补充层**（公开数据包以行情为主），由 akshare/baostock 免费源优先补齐；付费源按 §3.3 口径仅缺口确认后评估。

### 4.1 覆盖矩阵

| 数据 | akshare | tushare（积分付费档） | baostock | qstock |
|---|---|---|---|---|
| 指数行情（宽基/全市场） | ✅ | ✅（含实时、分钟，部分独立付费） | ✅（日/周/月） | ✅ |
| 指数成分与权重 | ✅ | ✅ | ✅（上证50/沪深300/中证500 成分） | ❌ |
| 行业分类 | ✅（东财/申万概念） | ✅（申万分级、中信） | ✅（证监会） | ✅ |
| 概念板块分类/成分/行情 | ✅（东财/同花顺） | ✅（THS/DC/TDX 三套） | ❌ | ✅（同花顺） |
| 个股/板块资金流 | ✅（东财） | ✅（THS/DC 双源） | ❌ | ✅ |
| 融资融券（汇总+明细+标的） | ✅ | ✅（含转融通） | ❌ | ✅ |
| 北向/南向资金 | ⚠️ 部分接口已失效（[问题分析](https://quant.csdn.net/691d70325511483559ebf51f.html)） | ✅ 沪深股通十大成交股、港股通数据 | ❌ | ⚠️ |
| 龙虎榜 | ✅ | ✅（每日统计单+机构交易单+游资明细） | ❌ | ✅ |
| 涨跌停/连板/炸板 | ✅ | ✅（打板专题） | ❌ | ✅ |

### 4.2 资金流/两融/龙虎榜
- 免费首选 akshare（东财口径），抽样对账用 baostock/第二源；付费校验源待缺口确认后评估。三者均依赖交易所盘后公开信息，合规风险低（交易所披露数据）。

### 4.3 行业/概念板块
- 注意**分类体系本身就是数据**：东财/同花顺概念会改名、合并、回溯调整；申万分类 2021 年改版过。落地时必须保存 `(classification, code, name, in_date, out_date)` 的**成分历史快照**，不能只存当前成分（否则引入未来信息）。审计 §11.3 放行检查第 6 项同理：历史股票池不得是当前成分的回填。

### 4.4 北向/南向资金 —— 重要规则变化（2024）
- 2024-04-12 沪深港三所宣布调整沪深港通交易信息披露机制，**2024-05-13 起（第一阶段）港交所不再实时披露沪深股通买入/卖出/成交总额**；当日额度余额 ≥30% 时仅显示"额度充足"（来源：[证券时报e公司/网易转载](https://www.163.com/dy/article/J1U3GV9N0519D3V1.html)、[HKEX 通告](https://www.hkex.com.hk/chi/prod/dataprod/Documents/24-04-12%20Arrangement%20of%20OMD-C%20and%20MMDH%20for%20the%20Adjustment%20to%20Market%20Data%20Dissemination%20in%20Relation%20To%20Northbound%20Trading%20under%20Stock%20Connect_C.pdf)）。
- 盘后披露新规则：每日收市后披露沪深股通**成交总额及总笔数、ETF 成交额、前十大成交活跃证券**；按月/年汇总；**每季度**第 5 个交易日公布上季度末单只证券北向持股数（原每日披露改为季度）。港股通（南向）同样改为盘后披露买卖金额/笔数/十大活跃股，每日收盘后披露单只证券持股。
- **落地含义**：任何"盘中北向资金流向"因子已无数据来源（akshare 相关接口因此失效）；因子库应转向"盘后十大活跃股 + 季度持股变动"，并在数据字典中记录该结构性断点（2024-05-13）。

---

## 5. 宏观、利率、汇率、大宗商品、美股/港股

> v2 定位：宏观类不在公开数据包范围内，由 akshare 免费源覆盖优先；付费源仅缺口确认后评估。

| 数据 | akshare（免费） | tushare | baostock | 其他 |
|---|---|---|---|---|
| 国内宏观（GDP/CPI/PPI/PMI/社融/货币供应） | ✅（统计局/央行/金十等多接口） | ✅（体系化，含发布日程） | ✅（仅货币供应/存贷款利率/准备金率） | — |
| 利率（Shibor/LPR/国债收益率曲线） | ✅ | ✅（Shibor/报价/LPR/Libor/Hibor/国债收益率曲线/民间借贷） | 部分 | — |
| 汇率 | ✅（央行/新浪等） | ✅（外汇日线） | ❌ | yfinance 备选 |
| 大宗商品/黄金 | ✅（期货、上海金、现货，接口多） | ✅（期货/现货黄金/南华指数） | ❌ | — |
| 美股/港股行情 | ✅（东财/新浪源，免费） | ✅（日线+复权+财报，**付费独立项**） | ❌ | yfinance（免费，未逐一核实本次） |
| 国际指数 | ✅ | ✅（国际主要指数、VIX） | ❌ | — |

- **推荐（v2）**：宏观/利率/汇率/商品用 **akshare 覆盖 + 抽样对账**；外围股市若只做日线级联动因子，akshare 免费源足够。美股全历史+财报属**确认后**的潜在缺口，按 §3.3 口径评估付费或 yfinance，不预定购买。
- akshare 的宏观接口散落在多个上游（统计局/央行/外汇局/金十），字段不统一——**需要自写标准化层**（见 §12）。
- 宏观数据的披露/修订也有 PIT 属性（初值/修正值），按 §9.5 双可见性口径登记；无法恢复历史版本的宏观修订序列同样标记 `historical_pit_unverified`。

---

## 6. 成熟开源数据框架 / 落地方案

### 6.1 microsoft/qlib —— 数据层是其最可复用的部分
- **GitHub**：[microsoft/qlib](https://github.com/microsoft/qlib)；**48,746 stars**；MIT；最近提交 2026-09-22（微软持续维护，与 RD-Agent 联动，非常活跃；**待核实**）。
- **数据层要点**（[Data Layer 文档](https://qlib.readthedocs.io/en/latest/component/data.html)）：
  - **Qlib 文件格式**：`.calendars` / `.instruments` / `.features/<code>/<field>` 二进制文件，按日历对齐，读取极快；`dump_bin.py` 可把 **CSV/Parquet 转换成 Qlib 格式**（也有社区 dump_mysql/dump_db 扩展）；
  - **DataLoader**：`QlibDataLoader`（表达式引擎算因子）、`StaticDataLoader`（任意静态表），接口可自定义，天然对接 pandas；
  - **PIT 数据库**（[Point-in-Time 文档](https://qlib.readthedocs.io/en/latest/advanced/PIT.html)）：文件式设计，每特征 4 列 `(date=披露日, period=报告期, value, _next=下一版本字节偏移)` + `.index` 索引，**提供"某历史时刻可见的版本"查询机制**（`PITProvider`/`LocalPITProvider`）；
    - **v2 必要警示（A01）**：qlib PIT **格式**解决的是"查询端按可见性取版本"，**不能凭空生成历史上不存在的版本**。用当前抓到的修订值填充 `_next` 链或补 `fetched_at`，无法恢复"更正披露之前"的原始值；审计验收要求"构造先披露、后更正的两版本案例，更正前查询不出现更正值"（§9.5）。
  - **data_collector**：官方爬取/转换脚手架（Yahoo、`baostock_5min` 等，见 [scripts/data_collector](https://github.com/microsoft/qlib/tree/main/scripts/data_collector)），含自动日频更新与**数据健康检查**脚本；
  - 缓存体系（ExpressionCache/DatasetCache）、离线/在线（qlib-server）两种服务模式。
- **Apple Silicon**：纯 Python + C 扩展轮子，arm64 可用（FAQ 提示 `qlib.data._libs.rolling` 需安装对应轮子）；作为数据层不涉及 GPU。
- **收费**：免费（数据包 crowd-sourced，行情数据原点为 Yahoo/公开源）。
- **v2 推荐理由**：**"数据层 + PIT 格式"直接复用**，且 investment_data 公开包已是 Qlib 格式（§10），可免去"采集 → Parquet/CSV → dump_bin"的全量转换；自行补充的数据走 Parquet → `dump_bin` 物化视图。PIT 查询机制复用，但**历史版本数据的完整性由 §9.5 的可见性口径与阻断规则保证**，不由格式保证。不必采用它的 ML 工作流。

### 6.2 hikyuu —— 速度优先的本地数据+回测一体
- **GitHub**：[fasiondog/hikyuu](https://github.com/fasiondog/hikyuu)；3,522 stars；Apache-2.0；最近提交 2026-09-22（非常活跃；**待核实**）。
- **数据层**：C++ 实现的高速本地数据读写（默认 SQLite/HDF5 等本地存储），支持通达信/pytdx、同花顺客户端导出、新浪/腾讯免费源等多种数据导入方式，支持前/后复权与除权除息处理。
- **收费**：免费；数据获取依赖上述公开源（同 akshare 的灰色地带）或券商客户端导出（合规性较好）。
- **Apple Silicon**：C++/Python 混合，官方提供 mac 支持但 wheel 情况随版本变化，建议在目标机实测编译。
- **推荐理由**：若追求"单机极限速度的指标计算/回测"，hikyuu 的数据层与因子组件值得复用；但其数据接入广度不如 akshare/tushare，**定位为研究计算引擎而非采集层**。

### 6.3 rqalpha —— 数据层可插拔，注意 License 限制
- **GitHub**：[ricequant/rqalpha](https://github.com/ricequant/rqalpha)；6,787 stars；最近提交 2026-09-22（活跃）；PyPI 最新 6.4.0（**待核实**）。
- **License（重要）**：PyPI 标注 Apache-2.0，但 README/说明书明确 **"仅限非商业使用。如需商业使用，请联系我们：public@ricequant.com"**（[PyPI 页面](https://pypi.org/project/rqalpha/)）；GitHub 因附加条款将 SPDX 标为 NOASSERTION。**个人研究可用，任何商业用途需先获授权。**（审计 A08：开源代码许可、模型许可、数据使用条款、服务订阅条款应分开记录。）
- **数据层**：
  - 官方免费 **data bundle**（`rqalpha download-bundle`）：A 股日线为主的打包数据，直接驱动回测；bundle 为快照式发布，增量需要补充源；
  - `DataProxy`/`DataSource` 可插拔，官方扩展 [rqalpha-mod-rqdata](https://github.com/ricequant/rqalpha-mod-rqdata)（接付费 RQData）、rqalpha-mod-tushare（社区）；
  - 事件驱动回测引擎对停牌、除权除息、涨跌停、税费有成熟的默认处理，可作为**质量校验规则的参考实现**。
- **推荐理由**：复用其"数据源抽象 + 公司行为（分红/除权）处理逻辑"与免费 bundle 做基准对账；若系统可能商业化，把 rqalpha 代码隔离或只借鉴设计。

### 6.4 其他备选（未逐一深入核实）
- **pytdx / mootdx**（通达信行情协议，[mootdx org](https://github.com/orgs/mootdx/repositories)）：券商/通达信行情通道，实时性好；协议逆向性质，合规性同抓取类。
- **sric0880/quantdata**（[链接](https://github.com/sric0880/quantdata)）：对 tiledb/mongodb/tdengine/clickhouse/duckdb 等金融时序存储方案的对比与客户端实现，做选型时值得旁证。

---

## 7. 数据落地与并发/备份边界（v2 按审计 A09 改写）

**核心口径（A09）**：

1. **单写入者**：全部事实数据由**唯一写入者**（ETL 写入进程）产生**不可变 Parquet 快照**及配套 **manifest**（记录 snapshot_id、生成时间、数据截至时间、来源标识、内容哈希、行数/文件清单）。
2. **读任务绑定 `snapshot_id`**：一切研究/回测/报告任务在任务契约中固定其所读快照，不读"当前最新目录"，保证半发布数据不可见、失败重跑不重复入库、旧实验可复现。
3. **SQLite 只存任务元数据**：任务状态、幂等键、水位、就绪状态、dq_issue 等元数据；**不承担事实数据写入**，避免多 Agent 直接写同一数据库文件的并发风险（[DuckDB 嵌入式并发文档](https://duckdb.org/docs/lts/connect/concurrency)同样限定单写入者多读者）。
4. **数据库升级门槛**：仅在**实测证明**单写入者无法满足需求、确需多进程写服务后，才考虑 PostgreSQL/服务型数据库；不预先部署。
5. **DuckDB 只读查询**：研究查询引擎直接查 Parquet 快照，无需常驻服务。
6. **备份与灾难恢复（A09）**：
   - 从首期开始做**独立介质或异地加密备份**（外接盘/异地对象存储，加密保存），并做**恢复验证**；
   - 目标 **RPO 24 小时 / RTO 1 天**，以实测恢复演练确认，不以纸面目标代替；
   - **同机 MinIO 不代灾难恢复**：MinIO 最多作本地快照仓库/版本化用途，不能抵御整机或磁盘损坏；原"MinIO 冷备"定位修正为"本地快照仓库（可选）"。

| 存储 | 适用场景 | 在本系统中的定位（v2） | Apple Silicon |
|---|---|---|---|
| **Parquet** | 列存压缩、schema 演进、生态通用（pandas/polars/duckdb/qlib dump 都认） | **事实数据最终归档格式**：单写入者产出的**不可变快照** + manifest，按 `(market, freq, snapshot_id)` 组织 | ✅ |
| **DuckDB** | 嵌入式 OLAP、列存向量化、**直接查询 Parquet**、单机聚合极快 | **研究查询引擎（只读）**：SQL 即席分析，无需常驻服务 | ✅ arm64 预编译 |
| **SQLite** | 单机嵌入式、低并发写 | **仅任务元数据**：任务状态/幂等键/水位/就绪状态/dq_issue；不存事实数据 | ✅ 原生 |
| **PostgreSQL** | 多进程并发写、事务与 upsert | **暂不引入**；仅实测证明单写入者不够时再评估 | ✅ arm64 官方版 |
| **MinIO** | S3 兼容对象存储 | 可选的**本地**快照仓库/版本化；**不算备份、不代灾难恢复** | ✅（Go，arm64 支持） |

**推荐组合（分阶段，v2）**：
1. **起步**：`investment_data 快照 + 补充层 Parquet` → 不可变快照 + manifest → `DuckDB 只读查询` + `SQLite 任务元数据` + `独立介质/异地加密备份`（RPO 24h / RTO 1d，实测确认）。
2. **当出现多进程并发写/事务需求**（须先有实测证据）：评估 PostgreSQL，Parquet 保持不可变快照归档，DuckDB 可 `postgres_scanner` 联查。
3. **量化流水线**：在干净快照之上用 qlib 格式（investment_data 包原生即 Qlib 格式；补充数据 `dump_bin` 生成）供表达式因子/数据集使用，物化视图可随时重建但须绑定 snapshot_id。

**DataFrame / 计算层**：pandas 为生态事实标准（baostock/tushare/qlib/alphalens 全部原生返回或消费 DataFrame）；**polars**（MIT）可作大数据量扫描的可选加速层（读 Parquet 极快），两者经 Arrow/Parquet 互通。本方案以 pandas 为主、polars 为可选，不引入第二套计算范式。

**容量参考**：全 A 约 5400 股 × 30 年日线 ≈ 数千万行 → Parquet < 2 GB；5 分钟线 15 年 ≈ 数亿行 → 10~30 GB；1 分钟线 5 年 ≈ 数十 GB。Mac mini（16GB+ 内存 / 512GB+ SSD）完全够用；investment_data 归档约 566MB/包（见 §10），快照保留份数与备份容量计入 §11 成本口径。

---

## 8. 增量更新、去重与数据就绪（v2 按 A10/A11 改写）

**总体架构：来源层 → 快照（不可变）→ 查询视图，全部幂等可重放；单写入者。**

1. **调度**：macOS `launchd`（或 cron）每日收盘后跑 EOD 任务；先检查 investment_data 是否有新发布（§10），补充层增量与盘后补全分开；宏观/财务按各自披露节奏。
2. **数据就绪登记（A10，新增）**：每个数据集记录 **截至时间（as_of）、预期日期（expected_date）与就绪状态（ready / delayed / missing）**。固定出报时点（如 16:15）不等于数据齐备：先生成**明确标注缺项**的初版，随后补齐，次日晨间修订；**旧数据不得伪装为当天数据**。单源延迟/故障时报告展示缺项与数据版本，恢复后补跑不重复统计。
3. **水位（watermark）**：SQLite 记录 `(dataset, code, freq, last_date, ingested_at, as_of, expected_date, ready_state)`；增量窗口固定回看 N 日重新抓取并进入新快照，以吸收上游的追溯修订（分红实施、财报更正、复权因子变化）。快照为不可变，回看重抓**不覆盖旧快照**，而是产出新 snapshot_id。
4. **去重/主键（A11 修订）**：
   - **公告/新闻/文档类**：首期按**来源文档 ID + 内容哈希**去重；**保留修订链**（同一来源文档 ID 的不同内容哈希按披露时间链接为修订序列），**不凭标题相似直接丢弃**——转载、更正、同标题不同内容必须可区分；金额、单位、主体、日期须能定位原文，缺证据时输出"未知"而非猜测。原"滤重比例"指标取消：去重率不代表信息正确率。
   - 行情：`(code, freq, trade_date)`（分钟加 `bar_time`）为主键，写入采用快照重建（单写入者）而非共享表 upsert；
   - 财务：`(code, report_period, item)` + 版本记录保留**多版本**，不覆盖旧行——但注意 §9.5：多版本机制只能保存"本系统实际获得过"的版本，不能补造历史；
   - 板块成分：`(classification, code, in_date, out_date)` 区间表。
5. **多源仲裁**：公开数据包为准，baostock 为主对账源，akshare/efinance 为辅；差异超阈值（如收盘价 >0.5% 或复权因子不一致）时写入 `dq_issue` 表人工复核，不静默覆盖。补充数据入库显式记录**单位与复权口径**，放独立层，**不静默拼接进第三方二进制包**（§2.3）。
6. **事件驱动回填**：除权除息/送转股发生时，**回填复权因子并重算该股全历史复权价**（前复权价是"随时间漂移"的，绝不能只存前复权价——必须存**不复权价 + 复权因子**，查询时动态计算）。
7. **补缺**：新股上市回拉全历史；退市股打 `delist_date` 但**永不删除数据**；停牌日显式写 `tradestatus=0` 行或维护停牌日历（勿简单跳过造成"隐形合并"）。
8. **重放能力**：raw/来源层保存原始响应（JSON/CSV/原始包），ETL 代码版本化，任何 schema 变更可全量重建查询视图；同一 snapshot_id 下的结果可复现。

---

## 9. 数据质量校验方案（停牌 / 复权 / 除权除息 / 退市与幸存者偏差 / PIT）

### 9.1 通用校验（每日自动跑，出报告）
- **结构**：主键唯一、无重复 bar、交易日历对齐（A 股不同时期有不同节假日安排，用 baostock/tushare 交易日历而非周一至周五）。
- **数值**：`open/high/low/close > 0`；`low ≤ min(open,close) ≤ max(open,close) ≤ high`；`volume ≥ 0`；`amount/volume` 落在合理价格区间；日涨跌幅不超涨跌停限制（±10%/±20%/±5% ST，注意 2020-08 创业板改 20%、北交所 30%）。
- **对账**：抽样个股与第二数据源比对收盘价；组合收益与对应指数（沪深300/中证500）做相关性 sanity check。

### 9.2 停牌处理
- 用 `tradestatus`（baostock）/停复牌接口（tushare `suspend_d`、akshare）构建停牌日历；
- 停牌日**保留显式记录**（价格沿用最后成交价、成交量=0、标记 `is_suspended`），避免把停牌段"前向填充后当连续交易"用于收益率计算；
- 回测撮合逻辑遇停牌禁止成交（rqalpha 的处理逻辑可参考）。

### 9.3 复权与除权除息
- **存储原则：只存不复权价（raw）+ 复权因子 + 分红送转明细**；前复权（动态）/后复权（静态）在查询层计算。后复权适合收益率计算，前复权适合展示。
- 复权因子双源校验：baostock `query_adjust_factor` vs 第二源（东财的"涨跌幅复权法"与通达信"定点复权法"结果略有差异，baostock 官方已注明口径）；与 investment_data 包的 normalize 口径对照见 §10 放行检查第 4 项；
- **除权除息一致性检验**：对每个除权日验证 `preclose ≈ (前日收盘 - 每股派现) / (1 + 送转比例)`（按交易所公式），偏差超阈值即报 dq_issue；
- 注意分红实施公告晚于预案：分红数据本身也要按披露时间做版本登记（§9.5）。

### 9.4 退市股票与幸存者偏差
- **必须全量保存退市股票**：baostock 的 `query_stock_basic` 提供 `ipoDate/outDate/status` 且历史 K 线含退市股；tushare 有"股票历史列表/股票曾用名/每日股本"；东财抓取源对退市股覆盖不全，**不可作为唯一来源**；
- 维护 **universe 快照表**：`(trade_date, code)` 的"当日真实可交易股票池"（含当日 ST 状态、上市/退市状态），回测选股只能用当日快照，绝不用"当前股票列表"回溯；与 §10 放行检查第 3/6 项联动（退市/停牌案例核对、历史股票池不得回填）；
- 退市整理期、重大资产重组导致的长停牌要在 universe 中如实反映（否则回测会买"实际不可买"的股票）；
- IPO 初期（次新股）与退市前期（仙股/流动性枯竭）建议单独设过滤规则并记录在研究笔记中。

### 9.5 财务与披露数据 PIT 校验（v2 按审计 A01 改写）

**核心结论：历史 PIT 表不能补出不存在的历史版本。** 免费源大多只提供"最新修订值"；把现在抓到的修订值放回早期披露日期，会让回测使用当时未知的信息。新增 `fetched_at` 或 `_next` 链**本身无法修复**这一问题。

**1. 双可见性口径（必须分别定义并记录所用口径）**

| 口径 | 定义 | 适用 |
|---|---|---|
| **市场已公开时间**（market_public_at） | 该版本信息对市场公开可得的时刻 | 历史研究（声称无前视的回测） |
| **本系统实际获得时间**（system_acquired_at） | 本系统首次采集/入库该版本的时刻 | 实时重放、模拟当时系统实际可见的信息集 |

历史研究与实时重放使用**不同**可见性口径；每一次数据集构造、回测与报告都必须**显式记录所采用的口径**。

**2. 版本记录字段要求（每个披露版本至少保存）**

| 字段 | 含义 |
|---|---|
| 来源文档编号 | 披露文件/公告的来源唯一 ID |
| 内容哈希 | 该版本内容的哈希（与来源 ID 共同构成 A11 去重与修订链依据） |
| 原始披露时间 | 首次披露时刻（含时区） |
| 修订披露时间 | 本次修订/更正的披露时刻（含时区） |
| 首次采集时间 | 本系统首次获得该版本的时刻 |
| 版本 | 版本序号或修订链标识 |
| 时区 | 所有时间字段统一标注时区（A 股场景以北京时间为准，显式记录） |

**3. 无披露时刻的保守规则**：只有日期、没有披露时刻时，一律采用**保守的下一交易时段规则**（该数据自其披露日的**下一交易时段**起才可见）；**不允许默认当日开盘可用**。同日多版本、收盘后公告、披露时间未知等情形均须有确定结果（按保守规则取最晚可见时点）。

**4. `historical_pit_unverified` 阻断规则**：无法恢复原始版本的字段（只有当前修订值、无法证明其早期形态）一律标记 `historical_pit_unverified`；被标记的字段**阻止进入声称无前视的历史研究**（数据集构造与回测加载器校验该标记并拒绝）。**MVP 可暂不使用财务因子**，从而不依赖不可验证的历史财务版本。

**5. 验收（A01）**：构造"先披露、后更正"的两版本案例——**更正前查询不出现更正值**；同日多版本、收盘后公告、时间未知均有确定结果。本验收为 **G1 执行项，本次未执行**。

**6. 其他 PIT 注意事项**：任何基本面/因子回测，数据集构造必须按"报告期 + 披露时间"过滤（`statDate ≤ t` 且 `market_public_at ≤ t`，并按所记录口径选择可见性字段）；复用 qlib PIT 文件格式或自建版本表只能保存实际采集到的版本链，抽查 `_next` 链条完整性不能替代上述历史版本证据。

---

## 10. investment_data 公开数据包接入（A05/审计 §11）

**数据策略（v2，取代"从零采集全市场多年日线"）：公开数据包初始化 + 定期同步 + 质量验收 + 按需补缺。**

### 10.1 已核查事实（审计 2026-09-23 核查；数据包本身未下载，质量验收未执行）

来源：[chenditc/investment_data Releases](https://github.com/chenditc/investment_data/releases)；核查仓库 commit `b8c129b4d9b838f050eac8b1135111b3b79fc894`。

| 项目 | 核查结果 | 结论边界 |
|---|---|---|
| 最近发布 | Release API 返回 `2026-09-22`，发布时间 `2026-09-22T08:17:39Z` | 发布日期不是实际最后交易日，须读取 manifest 和数据 |
| 正式资产 | `qlib_bin.tar.gz`（566,183,981 bytes）与 `qlib_bin.manifest.json`（511 bytes） | 已有 Qlib 数据包，可优先评估为历史初始化底座；不是已证实的逐日增量包 |
| 归档 API digest | `sha256:7a5e563f8f3bd696f8455e42a1eb36b07ac1272a7d0c43db5f0c099755c820a3` | 来源于 API 元数据，**未对下载字节独立重算**（待核实） |
| 校验能力 | 校验器要求归档哈希、长度、来源 commit、目标交易日；检查交易日历和股票池文件 | 能检查发布完整性，不能证明每只股票每个字段正确 |
| 历史起点 | 当前标准化代码设置交易日历起点 `2000-01-04` | 不能据此声称从 1990 年以来全部行情；各证券实际起止日期待包内核验 |
| 价格处理 | 导出后进入 Qlib 标准化，包含价格、成交量、VWAP 等调整逻辑 | 不应把 Qlib 数值直接当作券商报价、股数或金额（单位/复权口径见放行检查第 4 项） |
| 来源 | 项目说明初始导入涉及 Wind、财汇、Tushare、Yahoo；每日更新说明使用 Tushare | 不是官方交易所原始数据库，来源和历史口径要记录 |
| 使用方式 | README 提供直接下载发布资产；自行跑更新脚本需要 Tushare token | 消费公开发布物与自行复制上游生产流水线分开设计；**首期只消费公开发布物**，不部署其整套 Dolt/生产发布流水线 |
| 许可 | 仓库显示 Apache-2.0 代码许可 | 不据此推定所有上游数据都获得同等再分发授权；具体数据条款**待核实** |

### 10.2 接入路径（可执行清单；**G1 执行项，本次未执行**）

> 以下 7 条整理自审计 §11.2。全部为待执行项，本次修订仅固化清单，不代表已完成。

- [ ] **1. 固定 release tag 并下载**：固定明确的 release tag，下载同一 Release 的 archive 与 manifest；记录 URL、asset ID、发布时间、下载时间、大小及 SHA-256。
- [ ] **2. 固定 commit 校验器校验**：使用审阅过且**固定 commit** 的 `qlib/validate_archive.py`，按 README 的 `--expected-tag` 与 `--require-publishable` 方式校验；脚本本身及依赖也纳入版本记录。
- [ ] **3. 独立快照目录 + 原子切换**：解压到**新的独立快照目录**，不直接覆盖现有工作数据；核查成员路径、日历、股票池及目标日期，完成 §10.3 检查后**原子切换**活动快照。
- [ ] **4. Qlib 直读快照**：Qlib 直接读取该快照；Parquet/DuckDB 只保存补充数据、审计索引与统一查询视图，避免重复格式转换。
- [ ] **5. 每日检查新发布**：每日先检查是否有新发布；相同 digest 不重复下载；有新包则验证后切换。**暂按全包快照处理**，不假定只追加一天就能吸收上游历史修订。
- [ ] **6. 失败降级**：Release 延迟或失败时继续保留旧快照并标明截至日期（对接 §8.2 就绪状态登记）。BaoStock/AKShare 补充数据放**独立层**，明确优先级、单位和复权口径，**不静默拼接进第三方二进制包**。
- [ ] **7. 来源标识与实验可追溯**：记录 manifest 中数据和代码来源标识；固定实验使用的实际快照（snapshot_id），即使未来同 tag 内容变化也不影响已完成实验的可追溯性。

### 10.3 放行前必须执行的检查（可执行清单；**G1 执行项，本次未执行**）

> 以下 9 项整理自审计 §11.3。任一项不满足则不放行研究区间。

- [ ] **1. 起止日期与覆盖**：统计交易日历、每只证券首末有效值、交易所/板块/上市退市状态及空值比例 → 形成实际覆盖报告，研究区间不超出已验收范围。
- [ ] **2. 最新日期**：对照 manifest 的 `target_trade_date`、`day.txt` 和股票有效行情最大日期 → 日历末日不能掩盖大量个股陈旧行情。
- [ ] **3. 历史与退市股**：选取已知退市、暂停上市、长停牌和新股案例，核对历史范围 → 不仅仅是"all.txt 存在"；缺失显式登记，**不能声称无幸存者偏差**。
- [ ] **4. 复权及单位**：对照 normalize.py 和所固定 Qlib 实现，抽样除权前后价格、量、额及 VWAP → 研究价格与交易价格可区分，量价单位可解释，不重复复权。
- [ ] **5. 多源口径**：抽样对照独立公开源（baostock/akshare 等），重点看历史拼接、除权和异常日期 → 差异解释或隔离，不能按一个固定百分比忽略所有差异。
- [ ] **6. 指数成分历史**：检查 instruments 区间及上游权重转成分逻辑，核对历史样例 → 历史股票池不是当前成分的回填。
- [ ] **7. 缺失与停牌**：验证缺失值、零成交与停牌标记的区别 → 缺失不默认为零收益或可成交。
- [ ] **8. 财务 PIT 与事件**：检查是否实际包含原始版本、公告时间和事件证据 → 没有就列为缺口，**不以行情包替代 PIT 财务与公告库**；历史版本缺失按 §9.5 标记 `historical_pit_unverified`。
- [ ] **9. 修订与重放**：比较两次快照的历史交集，对变动生成差异报告 → 历史修订可追踪；旧实验使用旧快照可复现。

### 10.4 补充层（BaoStock/AKShare）使用规则（v2）

- **优先级**：公开数据包（底座）> baostock（主对账/补字段）> akshare/efinance（广度兜底）。
- **单位口径**：入库显式记录价格（元）、成交量（股/手）、成交额（元）等单位，禁止隐式换算。
- **复权口径**：显式记录不复权/前复权/后复权/复权因子来源；与公开包 normalize 口径对照后**不重复复权**（放行检查第 4 项）。
- **不静默拼接**：补充数据一律放独立层（Parquet/DuckDB 视图）；如需合并，生成新的自有快照并保留来源标识与审计索引；**绝不修改第三方二进制包内容**。

### 10.5 边界说明

公开日更快照能减少采集成本，也能积累"此快照发布时可见的数据"证据；但**不能还原项目归档开始前每一天的财务原始披露版本**（A01）。真实成交价格、交易限制、财务 PIT、公告及行业历史单独建覆盖矩阵（缺口清单），付费只解决确认存在的缺口。股票真实成交价格、交易限制、财务 PIT、公告及行业历史的防前视、执行契约、样本外封存和备份要求由研究正确性决定，与数据是否免费无关。

---

## 11. 成本口径（v2 按审计修订）

1. **免费数据也计成本**：统计**下载（流量/时间）、磁盘（含快照保留份数）、备份（独立介质/异地加密存储）、维护（校验、对账、人工复核 dq_issue）** 四类成本，按数据集记录。
2. **付费源评估原则**：Tushare 等付费来源**只在公开渠道无法满足已确认需求时评估**；**不预定购买档位、不预定购买时间、不承诺某积分档覆盖所有接口**。评估前置条件：缺口清单中该缺口（a）已确认真实存在（§10.3 检查结论），（b）公开源确实无法补齐（含抽样验证），（c）该缺口对应已确认的研究/复盘需求。
3. **AI/模型费用单列**：AI 为已有能力，费用单列；不将数据省钱目标误解为降低模型能力。模型调用保留用量记录、可配置重试和并发控制，具体额度按已有服务配置（审计 A06/A12 口径）。
4. **硬件不作前置条件**：先测已有设备（Mac mini），训练/回测/推理错峰；取消首期购买 32/64GB 设备的前置条件（审计 A06）。

---

## 12. 直接复用 vs 需要自己写（v2 更新）

| 环节 | 直接复用（不写采集系统） | 需要自己写的薄层 |
|---|---|---|
| 历史日线底座 | investment_data 公开 Qlib 数据包 + `qlib/validate_archive.py`（固定 commit） | 快照目录管理、manifest 登记、原子切换、release 同步检查（§10.2） |
| 补充行情采集 | baostock / akshare / efinance 客户端 | 调度（launchd）、水位、多源仲裁、单位/复权口径登记 |
| 财务/基本面 | baostock 季频指标 / akshare 财报（MVP 不用财务因子） | PIT 版本登记（双可见性口径 + §9.5 字段 + `historical_pit_unverified` 阻断）；缺口清单 |
| 板块/资金流/两融/龙虎榜 | akshare | 板块成分历史快照表（防未来信息） |
| 北向/南向 | akshare 盘后接口 | 新披露规则下的因子重构（结构性工作，无法复用旧代码） |
| 宏观/利率/汇率/大宗/港美 | akshare | 字段标准化与统一日历 |
| 新闻/公告类文档 | akshare 巨潮/财联社等接口 | **来源文档 ID + 内容哈希去重、修订链保留**（A11） |
| 研究数据层 | qlib（Qlib 格式、DataLoader、PIT 格式、dump_bin、健康检查）、hikyuu 计算层、rqalpha 公司行为处理 | Parquet→Qlib 的 ETL 管道脚本（补充数据） |
| 存储 | Parquet/DuckDB/SQLite 现成组件 | 单写入者快照生成 + manifest + snapshot_id 绑定（A09） |
| 备份 | 现成加密备份工具/异地对象存储 | 备份调度 + **恢复验证脚本**（RPO 24h / RTO 1d 实测，A09） |
| 数据就绪与质量 | rqalpha 撮合规则、qlib check_health 脚本作参考 | 就绪状态登记（A10）、DQ 规则集与报告（§9）、放行检查清单执行（§10.3） |

**结论**：采集系统仍 100% 复用现成项目/公开数据包；自写部分收敛为"快照/manifest 管理 + 调度 + ETL + 校验/放行 + 就绪登记 + 来源哈希去重/修订链 + PIT 版本登记"七件薄层，符合"不自己写采集系统"的原则。

---

## 13. 推荐落地路线（v2 —— G1 执行清单，本次未执行）

> 按审计阶段门：以下属 **G1（复盘 MVP）执行项**，放行证据为"连续 10 个交易日有报告；延迟数据明确标记；至少一次恢复成功"。原按周排期与"第 3 周开通 tushare 2000 积分"计划**撤销**（见 §3.3）。

1. **公开包初始化与校验**：执行 §10.2 接入路径 1–4（固定 release tag + 固定 commit `validate_archive.py`（`--expected-tag`/`--require-publishable`）+ 独立快照目录原子切换）。
2. **放行检查**：执行 §10.3 全部 9 项，形成实际覆盖报告与缺口清单；未通过项显式登记，不放行对应研究区间。
3. **快照与备份**：建立不可变 Parquet 快照 + manifest、读任务绑定 snapshot_id、SQLite 任务元数据；配置独立介质/异地加密备份并做**恢复演练**（目标 RPO 24h / RTO 1d，实测确认）。
4. **补充层与抽样对账**：接入 baostock/akshare 补充层（§10.4 规则），对公开包做多源抽样对账；差异入 `dq_issue`。
5. **就绪与出报**：每数据集登记截至时间/预期日期/就绪状态（A10），先出标注缺项的初版报告，补齐后次日晨间修订；连续 10 个交易日出报告。
6. **按需补缺**：依据缺口清单逐项评估补充来源；付费源（Tushare 等）仅对确认无法从公开渠道补齐的缺口评估（§11），不预定档位。
7. **PIT 与因子**：MVP 暂不使用财务因子；PIT 登记按 §9.5 双可见性口径与字段要求实施，`historical_pit_unverified` 阻断生效后才开声称无前视的历史研究（G2 前置）。

---

## 14. 主要来源

**项目状态（GitHub API / PyPI 实时核实，2026-09-22；审计未重核，待核实——stars 不作为可用性证据）**
- [akfamily/akshare](https://github.com/akfamily/akshare)（22,692 stars, MIT, pushed 2026-09-20）· [文档](https://akshare.akfamily.xyz/) · 接口失效案例：[#6143](https://github.com/akfamily/akshare/issues/6143)、[#7180](https://github.com/akfamily/akshare/issues/7180)、[#6574](https://github.com/akfamily/akshare/issues/6574)
- [Micro-sheep/efinance](https://github.com/Micro-sheep/efinance)（4,067 stars, MIT, pushed 2026-07-17）
- [baostock PyPI](https://pypi.org/project/baostock/)（0.9.4 @ 2026-09-21, BSD）· [官网知识库](https://www.baostock.com/mainContent?file=) · [API 总结（第三方）](https://github.com/atompilot/baostock-skill/blob/master/skills/baostock/SKILL.md) · [复权因子说明 PDF](http://www.baostock.com/baostock/images/2/20/BaoStock%E5%A4%8D%E6%9D%83%E5%9B%A0%E5%AD%90%E7%AE%80%E4%BB%8B.pdf)
- [waditu/tushare](https://github.com/waditu/tushare)（15,409 stars, BSD-3-Clause, pushed 2024-03-13）· [积分与频次权限表](https://tushare.pro/document/2?doc_id=290) · [接口目录](https://tushare.pro/document/2) · [datayes 历史文档](https://github.com/waditu/tushare/blob/master/docs/datayes.rst)
- [tkfy920/qstock](https://github.com/tkfy920/qstock)（1,940 stars, MIT, pushed 2025-03-16）
- [mpquant/Ashare](https://github.com/mpquant/Ashare)（3,869 stars, **无 License**, pushed 2025-12-24）
- [shidenggui/easyquotation](https://github.com/shidenggui/easyquotation)（5,398 stars, MIT, pushed 2026-02-28）
- [microsoft/qlib](https://github.com/microsoft/qlib)（48,746 stars, MIT, pushed 2026-09-22）· [Data Layer](https://qlib.readthedocs.io/en/latest/component/data.html) · [PIT](https://qlib.readthedocs.io/en/latest/advanced/PIT.html) · [data_collector](https://github.com/microsoft/qlib/tree/main/scripts/data_collector)
- [fasiondog/hikyuu](https://github.com/fasiondog/hikyuu)（3,522 stars, Apache-2.0, pushed 2026-09-22）
- [ricequant/rqalpha](https://github.com/ricequant/rqalpha)（6,787 stars, Apache-2.0+仅限非商业, pushed 2026-09-22）· [PyPI 6.4.0（含 License 声明与 RQData 覆盖）](https://pypi.org/project/rqalpha/) · [rqalpha-mod-rqdata](https://github.com/ricequant/rqalpha-mod-rqdata)

**investment_data 公开数据包（审计 §11 核查，固定 commit `b8c129b4d9b838f050eac8b1135111b3b79fc894`）**
- [S9：固定版本 README](https://github.com/chenditc/investment_data/blob/b8c129b4d9b838f050eac8b1135111b3b79fc894/README.md)
- [S10：2026-09-22 Release](https://github.com/chenditc/investment_data/releases/tag/2026-09-22)
- [S11：标准化与日历起点](https://github.com/chenditc/investment_data/blob/b8c129b4d9b838f050eac8b1135111b3b79fc894/qlib/normalize.py)
- [S12：归档校验器](https://github.com/chenditc/investment_data/blob/b8c129b4d9b838f050eac8b1135111b3b79fc894/qlib/validate_archive.py)
- [S13：日线表来源与拼接说明](https://github.com/chenditc/investment_data/blob/b8c129b4d9b838f050eac8b1135111b3b79fc894/docs/final_a_stock_eod_price.ch.md)
- [S14：行情导出查询](https://github.com/chenditc/investment_data/blob/b8c129b4d9b838f050eac8b1135111b3b79fc894/qlib/dump_all_to_qlib_source.py)

**商业数据服务**
- [JQData 使用说明](https://www.joinquant.com/help/api/doc?name=JQDatadoc&id=10514) · [试用和购买说明](https://joinquant.com/help/api/doc?id=9827&name=logon) · [JQData 本地化公测公告](https://www.joinquant.com/view/community/detail/4d575f700a403ed4186eaef6ec57d4e7)
- [米筐产品价格](https://rqopen.ricequant.com/welcome/pricing) · [RQData 介绍](https://www.ricequant.com/welcome/rqdata) · [vnpy 论坛价格讨论](https://www.vnpy.com/forum/topic/29498-qing-wen-yi-xia-3000yuan-shou-jie-de-radata-bao-han-gu-piao-shu-ju-yao)

**规则与市场事实**
- [沪深港通交易信息披露机制调整（证券时报e公司）](https://www.163.com/dy/article/J1U3GV9N0519D3V1.html) · [HKEX 通告 PDF](https://www.hkex.com.hk/chi/prod/dataprod/Documents/24-04-12%20Arrangement%20of%20OMD-C%20and%20MMDH%20for%20the%20Adjustment%20to%20Market%20Data%20Dissemination%20in%20Relation%20To%20Northbound%20Trading%20under%20Stock%20Connect_C.pdf) · [北向接口失效分析](https://quant.csdn.net/691d70325511483559ebf51f.html)

**存储与并发边界**
- [DuckDB 嵌入式并发（LTS）](https://duckdb.org/docs/lts/connect/concurrency)（审计 S7）
- [sric0880/quantdata（金融时序库选型对比）](https://github.com/sric0880/quantdata)

**审计依据**
- [《个人 A 股量化系统审计与公开数据优先整改方案》](../personal-quant-audit-plan.md)（2026-09-23）：A01/A05/A09/A10/A11、§11 investment_data 公开数据包接入审计及 S1–S14 证据链接。

> 免责声明：抓取类项目（akshare/efinance/qstock/Ashare/easyquotation）的数据来自各网站公开接口，可能违反上游 ToS 且随时失效；个人研究使用惯例上被容忍，但**不得用于对外商业产品**。stars 与提交时间为 2026-09-22 快照且审计未重核（待核实），会随时间变化；**不以 stars 数代替可用性判断**。
