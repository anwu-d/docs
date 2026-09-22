# A 股量化研究系统 · 数据层调研报告

> 场景：个人 A 股量化研究系统，部署于 Mac mini（Apple Silicon），单机运行。
> 原则：**尽量复用现成开源项目与数据服务，不自研采集系统**。
> 核实时间：2026-09-22（stars / License / 最近提交均通过 GitHub API、PyPI 元数据实时抓取核实；调研日期之后请以链接为准）。

---

## 0. 结论速览（TL;DR）

| 用途 | 首选 | 备选/交叉校验 | 说明 |
|---|---|---|---|
| 日线/分钟线采集 | **baostock**（免费稳定）+ **akshare**（广度） | efinance、tushare（付费后更强） | 双源互校，落库后统一 |
| 财务/基本面 | **tushare**（2000 积分档，200 元/年） | baostock 季频财务（免费）、akshare 财报 | tushare 覆盖三大表+分红+股本，性价比最高 |
| 指数/板块/资金流/两融/龙虎榜 | **akshare**（免费） | tushare（付费后更稳） | 北向数据已停发实时明细，见 §4.4 |
| 宏观/利率/汇率/大宗/港美股 | **akshare** + **tushare** | baostock 宏观（窄） | tushare 宏观成体系（GDP/CPI/PPI/社融/美债） |
| 数据框架/研究层 | **microsoft/qlib** 数据层（PIT + DataLoader） | hikyuu（速度）、rqalpha（事件回测） | 只复用其数据层，不必全盘采用其建模范式 |
| 落地存储 | **Parquet（事实表）+ DuckDB（查询）+ SQLite（元数据）** | PostgreSQL（并发写需求出现时）、MinIO（备份/多机时） | Apple Silicon 全部原生支持 arm64 |

一句话推荐：**akshare（广度）+ baostock（稳定免费日线/分钟/复权因子）+ tushare 小额付费（财务与合规 API）三源采集 → Parquet 落地 → DuckDB 研究查询 → qlib bin 格式服务量化流水线**，自写部分仅限 ETL 调度、质量校验与 PIT 财报表（可参考 qlib PIT 设计）。

---

## 1. 调研范围与评估维度

- 范围：①行情（日线/分钟线）；②财务/基本面与商业数据 API 付费边界；③指数/行业概念板块/资金流/两融/北向南向/龙虎榜；④宏观/利率/汇率/大宗商品/美股港股；⑤开源数据框架（qlib、hikyuu、rqalpha）数据层。
- 每个候选给出：GitHub 地址、stars、维护活跃度、License、数据覆盖、更新频率、历史完整性、收费、稳定性与合规性（是否网页抓取、ToS 风险）、推荐理由。
- 额外：数据落地选型（SQLite/PostgreSQL/DuckDB/Parquet/MinIO）、增量更新与去重、数据质量校验（停牌/复权/除权除息/退市与幸存者偏差）。

---

## 2. 行情数据（日线 / 分钟线）

### 2.1 总对比表

| 项目 | GitHub | Stars | 最近提交 | License | 日线 | 分钟线 | 实时 | 历史深度 | 收费 | 数据获取方式 | ToS/合规风险 | 推荐度 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **akshare** | [akfamily/akshare](https://github.com/akfamily/akshare) | 22,692 | 2026-09-20（活跃） | MIT | ✅ 全市场含港美 | ✅ 1/5/15/30/60 分 | ✅ | 日线自上市（东财源可至 1991） | 免费 | 抓取东财/新浪/同花顺等公开网页接口 | ⚠️ 灰色（非官方授权 API） | ★★★★★ |
| **efinance** | [Micro-sheep/efinance](https://github.com/Micro-sheep/efinance) | 4,067 | 2026-07-17（较活跃） | MIT | ✅ | ✅ | ✅ | 日线全历史 | 免费 | 抓取东方财富公开接口 | ⚠️ 灰色 | ★★★☆ |
| **baostock** | 官网 [baostock.com](https://www.baostock.com)（无官方 GitHub 主仓） | — | PyPI 0.9.4 @ 2026-09-21（活跃） | BSD | ✅ 含退市股 | ✅ 5/15/30/60 分（指数除外） | ❌ | 日线自 1990 年代末（约 1999，含退市股）、分钟线约 2011 起 | 免费 | 自有数据服务器（TCP API），**非抓取** | ✅ 低 | ★★★★★ |
| **tushare** | [waditu/tushare](https://github.com/waditu/tushare) | 15,409 | 2024-03-13（客户端停滞，服务端持续运营，站点 ICP 2026） | BSD-3-Clause | ✅ | ✅（独立付费，2009 起） | ✅（独立付费） | 日线长历史、分钟 2009 起 | 积分制收费（详见 §3.3） | 官方 HTTP API | ✅ 低（商业数据服务） | ★★★★☆ |
| **qstock** | [tkfy920/qstock](https://github.com/tkfy920/qstock) | 1,940 | 2025-03-16（更新慢） | MIT | ✅ | ✅ | ✅ | 日线全历史 | 免费 | 抓取东财/新浪/同花顺/百度 | ⚠️ 灰色 | ★★★ |
| **Ashare** | [mpquant/Ashare](https://github.com/mpquant/Ashare) | 3,869 | 2025-12-24 | **无 License**（默认保留所有权利） | ✅ | ✅（分时/分钟） | ✅（新浪/腾讯双源） | 短 | 免费 | 抓取新浪/腾讯行情 | ⚠️ 灰色 + 无授权条款 | ★★☆ |
| **easyquotation** | [shidenggui/easyquotation](https://github.com/shidenggui/easyquotation) | 5,398 | 2026-02-28 | MIT | ❌（仅快照） | ❌ | ✅ 全市场快照（新浪/腾讯/集思录） | — | 免费 | 抓取新浪/腾讯 | ⚠️ 灰色 | ★★★（实时快照场景） |

> stars/最近提交来源：GitHub REST API（`api.github.com/repos/...`），抓取于 2026-09-22。

### 2.2 逐项详评

#### akshare —— 广度之王，作为"兜底采集层"
- **GitHub**：[akfamily/akshare](https://github.com/akfamily/akshare)；22,692 stars；MIT；最近提交 2026-09-20，近乎日更，issue 归零（滚动关闭）。
- **覆盖**：A 股/港股/美股行情（日/周/月/分钟 K、实时快照）、指数、行业与概念板块、资金流、融资融券、龙虎榜、宏观（国家统计局/央行/外汇局）、利率（Shibor/LPR 等）、汇率、大宗商品、基金/可转债/期权/期货、三大财务报表等 1000+ 接口（[官方文档](https://akshare.akfamily.xyz/)）。
- **更新频率**：随上游站点（东财/新浪/同花顺等）实时或日度。
- **历史完整性**：个股日线自上市起（东财示例数据可至 1991 年，见 [stock.md](https://github.com/akfamily/akshare/blob/main/docs/data/stock/stock.md)）；**分钟线历史深度有限**（东财分钟接口通常只有近几年，1 分钟更短）。
- **收费**：完全免费、无 token。
- **稳定性/合规**：这是 akshare 的主要短板——它通过解析公开网页接口获取数据，上游改版会导致接口失效（如 [Issue #6143](https://github.com/akfamily/akshare/issues/6143)、[Issue #7180](https://github.com/akfamily/akshare/issues/7180)、[Issue #6574](https://github.com/akfamily/akshare/issues/6574)），修复虽快但属于"追着上游跑"；数据使用处于各网站 ToS 的灰色地带，**个人研究可接受，不可作为对外商业产品的数据来源**。
- **推荐理由**：覆盖广、免费、社区活跃，适合做"数据兜底 + 覆盖面上限"；但应把它放在 ETL 的**第二数据源**位置，核心事实表用 baostock/tushare 双校验。

#### efinance —— 东财数据的干净封装
- **GitHub**：[Micro-sheep/efinance](https://github.com/Micro-sheep/efinance)；4,067 stars；MIT；最近提交 2026-07-17（较活跃，但 open issues 153，响应一般）。
- **覆盖**：股票（日/周/月/分钟 K、实时）、资金流、基金、债券、期货；以东方财富为唯一数据源。
- **收费**：免费。**历史**：日线全历史，分钟线与 akshare 东财源同深度。
- **稳定性/合规**：同 akshare（东财抓取），单源依赖。接口简洁、返回规整 DataFrame，比 akshare 更轻。
- **推荐理由**：作为东财源的轻量替代/交叉校验；不建议作为唯一来源。

#### baostock —— 免费+自有服务器的"压舱石"
- **发布渠道**：官网 [baostock.com](https://www.baostock.com) + [PyPI](https://pypi.org/project/baostock/)（无活跃 GitHub 主仓，社区镜像/工具仓另存）。**License：BSD**（PyPI 元数据 "License :: OSI Approved :: BSD License"）。
- **维护活跃度**：PyPI 发版记录——0.8.9（2024-05）之后沉寂，**2024-04-14 之后 2026 年连续发版：0.9.1（2026-04）、0.9.2（2026-06）、0.9.3（2026-07）、0.9.4（2026-09-21）**，即当前仍在积极维护（数据源自 2026 年示例数据可查）。
- **覆盖**（API 明细见 [baostock-skill API 总结](https://github.com/atompilot/baostock-skill/blob/master/skills/baostock/SKILL.md)）：
  - K 线：日/周/月 + 5/15/30/60 分钟；不复权/前复权/后复权（`adjustflag`）+ **复权因子**（`query_adjust_factor`）+ **分红送股**（`query_dividend_data`，含除权除息日）；
  - 行内字段自带 `preclose`、`tradestatus`（停牌标记）、`isST`、PE/PB 等估值；
  - 财务：季频盈利能力/营运/成长/偿债/现金流/杜邦 + 业绩预告/快报（含 `pubDate`/`statDate`，**天然适合做 PIT**）；
  - 基础：交易日历、全证券列表、股票基本信息（**ipoDate/outDate/status，含退市股**）、行业分类（证监会）、上证50/沪深300/中证500 成分；
  - 宏观：存贷款利率、准备金率、货币供应量。
- **更新频率**：日度（交易日收盘后）。
- **历史完整性**：长历史（社区与教学资料记载日线约自 1999 年，含已退市股票；分钟线约自 2011 年；指数不支持分钟线）——精确起点以官方知识库为准。
- **收费**：免费，无积分无频率硬限制。
- **稳定性/合规**：**自有数据服务器 + 专用 TCP 协议**，不抓网页，合规风险最低；接口十年不变，最稳。缺点：功能面窄（无板块概念/资金流/两融/北向/龙虎榜）、ETF 覆盖有限、所有字段返回字符串、pandas 2.x 下 `get_data()` 需手工迭代规避（README 已注明）、复权采用"涨跌幅复权法"与通达信定点复权略有差异。
- **推荐理由**：核心历史数据库的**第一数据源**——免费、含退市股、有复权因子与除权除息明细，直接决定你的回测数据质量下限。

#### tushare（Pro）—— 合规性最好的官方 API，小额付费性价比极高
- **GitHub**：[waditu/tushare](https://github.com/waditu/tushare)；15,409 stars；BSD-3-Clause。客户端库最近提交 2024-03-13（基本不再更新，但 API 稳定）；服务端 tushare.pro 持续运营（官网页脚 ICP 许可证为 2026 年签发）。
- **收费**（[积分与频次权限对应表](https://tushare.pro/document/2?doc_id=290)，2026-09 核实）：
  - 120 积分（免费注册）：仅非复权日线，50 次/分；
  - **2000 积分 = 200 元/年**：日线/复权/股本/分红/财务/两融/龙虎榜等大部分常规接口，200 次/分；
  - 5000 积分 = 500 元/年：常规数据无总量上限；
  - 10000 积分 = 1000 元/年：特色数据（盈利预测、筹码分布、券商金股等）；
  - **独立付费项**：历史分钟线（2009 起）2000 元一次性；实时分钟 1000 元/月；美股日线 2000 元/年；新闻/公告 1000 元/年等。
- **覆盖**：[接口目录](https://tushare.pro/document/2)涵盖基础/行情/财务（三大表+分红+审计+主营构成）/参考数据（股东、质押、解禁、大宗）/两融与转融通/资金流（THS/DC）/打板专题（龙虎榜、涨跌停、连板天梯、游资明细）/指数专题（申万、中信、国际指数）/公募基金/期货/期权/债券（含可转债、国债收益率曲线）/外汇/港股（含财务）/美股（含财务）/宏观（GDP、CPI/PPI、社融、货币供应、PMI、Shibor、LPR、Libor/Hibor、美债收益率）/公告与新闻。
- **更新频率**：日度为主，部分实时（付费）；**历史完整性**：长历史（分钟自 2009、港股财报自 2000）。
- **稳定性/合规**：官方授权数据服务（HTTP API + 积分体系），无 ToS 风险，字段规范、有文档，是**唯一可放心用于严肃回测基线的商业源**。
- **推荐理由**：200 元/年档位补齐 akshare/baostock 的所有结构化缺口（股本变动、分红明细、两融、龙虎榜、申万分类），且合规。

#### qstock / Ashare / easyquotation —— 轻量补充，不入核心
- **qstock**（[tkfy920/qstock](https://github.com/tkfy920/qstock)，1,940 stars，MIT，最近提交 2025-03，更新慢）：数据+选股+回测一体的"个人投研包"，数据源与 akshare 高度重叠（东财/新浪/同花顺），深度更窄。**推荐理由**：想要"开箱即用的一体化小框架"时用；作为数据层无增量价值。
- **Ashare**（[mpquant/Ashare](https://github.com/mpquant/Ashare)，3,869 stars，最近提交 2025-12）：单文件极简实时行情（新浪/腾讯双源自动切换）。**注意：仓库未声明任何 License**（GitHub API `license: null`），法律上默认保留所有权利，**不建议引入到正式代码库**；如需同类能力用 easyquotation（MIT）。
- **easyquotation**（[shidenggui/easyquotation](https://github.com/shidenggui/easyquotation)，5,398 stars，MIT，最近提交 2026-02）：新浪/腾讯/集思录实时快照批量拉取，适合盘中监控场景。**推荐理由**：MIT + 稳定维护，做实时报价快照首选。

### 2.3 行情层结论
- 核心双源：**baostock（主）+ akshare（辅/兜底）**；预算允许则加 **tushare 2000 积分档**做三源对账。
- 分钟线：免费方案只有 baostock（2011 起、5 分钟粒度起）与东财抓取（深度浅）；**需要 2009 年起 1 分钟级历史只能买 tushare 历史分钟（2000 元一次性）**——这是免费与付费的最大分界。

---

## 3. 财务 / 基本面数据

### 3.1 免费三件套对比

| 能力 | baostock | akshare | tushare（免费 120 积分） | tushare（2000 积分，200 元/年） |
|---|---|---|---|---|
| 利润表/资产负债表/现金流量表 | ❌（仅衍生指标） | ✅（东财/新浪抓取） | ❌ | ✅（标准三大表） |
| 季频财务指标（ROE/成长/偿债/现金流/杜邦） | ✅ 含 pubDate | ✅ | ❌ | ✅ |
| 分红送股/除权除息明细 | ✅（含除权日、税前税后现金分红） | ✅ | ❌ | ✅ |
| 业绩预告/快报 | ✅ | ✅ | ❌ | ✅ |
| 股本变动/每日股本 | 部分（totalShare/liqaShare） | ✅ | ❌ | ✅（含盘前股本，独立 500 元/年） |
| 股东/质押/解禁/大宗 | ❌ | ✅ | ❌ | ✅ |
| 财报披露日期（PIT 关键字段） | ✅（pubDate） | 部分 | ❌ | ✅（财报披露日期表） |
| 历史财务修订（PIT 多版本） | ❌ | ❌ | ❌ | ❌（需自建，见 §8/§9；商业 PIT 见 RQData/JQData） |

### 3.2 商业数据 API 的免费/付费边界

| 服务 | 免费边界 | 付费情况 | 覆盖特点 | 合规 | 建议 |
|---|---|---|---|---|---|
| **tushare Pro** | 注册 120 积分（仅非复权日线） | 积分档 200/500/1000/1500 元/年；分钟、美股、新闻等独立付费（§2.2） | A 股结构化数据最全，含宏观/港美 | ✅ 官方 API | **200 元/年档强烈推荐** |
| **JQData（聚宽）** | 线上研究/回测环境免费；本地 SDK 曾开放公测（[公告](https://www.joinquant.com/view/community/detail/4d575f700a403ed4186eaef6ec57d4e7)） | 本地化 JQData 按年订阅收费（价格以[官方说明](https://joinquant.com/help/api/doc?id=9827&name=logon)为准，页面为动态渲染需登录查看） | 行情/财务/行业概念/资金流/两融齐全，数据清洗质量高，支持 PIT 财报 | ✅ 官方 API | 若预算充足、重基本面研究，值得作为 tushare 的上位替代 |
| **RQData（米筐）** | 试用期免费（[官网](https://www.ricequant.com/welcome/rqdata)） | 付费年订（社区反馈约 3000 元/年级别，见 [vnpy 论坛](https://www.vnpy.com/forum/topic/29498-qing-wen-yi-xia-3000yuan-shou-jie-de-radata-bao-han-gu-piao-shu-ju-yao)，以[官方报价](https://rqopen.ricequant.com/welcome/pricing)为准） | A 股 2005 至今+实时、全历史财务 + **官方 PIT API**、基金/期货/期权/可转债、风格因子、宏观（见 [rqalpha PyPI 说明](https://pypi.org/project/rqalpha/)） | ✅ 官方 API | 需要"财报 Point-in-Time"而不想自建 PIT 表时的付费选项 |
| **通联数据 DataAPI（datayes/优矿）** | 早年开放免费 token API（tushare 曾内置 [datayes 模块](https://github.com/waditu/tushare/blob/master/docs/datayes.rst)，现为历史遗留文档） | 当前主要面向机构/优矿平台销售，个人免费通道有限、公开文档维护少 | 基本面/因子/宏观曾是强项 | ✅ 官方 API | **不推荐**作为个人系统依赖（个人渠道不确定） |

> **PIT（Point-in-Time）提示**：免费源都只给"最新修订值 + 披露日期"，回测要防"未来信息"必须自建 PIT 表（记录每次抓取的版本）；付费捷径是 RQData/JQData 的 PIT 财报 API；开源捷径是复用 qlib 的 PIT 文件格式（§6.1）。

---

## 4. 指数、行业/概念板块、资金流、两融、北向/南向、龙虎榜

### 4.1 覆盖矩阵

| 数据 | akshare | tushare（2000+ 积分） | baostock | qstock |
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
- 免费首选 akshare（东财口径），付费/校验用 tushare（THS 与 DC 双口径，可做交叉验证）。三者均依赖交易所盘后公开信息，合规风险低（交易所披露数据）。

### 4.3 行业/概念板块
- 注意**分类体系本身就是数据**：东财/同花顺概念会改名、合并、回溯调整；申万分类 2021 年改版过。落地时必须保存 `(classification, code, name, in_date, out_date)` 的**成分历史快照**，不能只存当前成分（否则引入未来信息）。

### 4.4 北向/南向资金 —— 重要规则变化（2024）
- 2024-04-12 沪深港三所宣布调整沪深港通交易信息披露机制，**2024-05-13 起（第一阶段）港交所不再实时披露沪深股通买入/卖出/成交总额**；当日额度余额 ≥30% 时仅显示"额度充足"（来源：[证券时报e公司/网易转载](https://www.163.com/dy/article/J1U3GV9N0519D3V1.html)、[HKEX 通告](https://www.hkex.com.hk/chi/prod/dataprod/Documents/24-04-12%20Arrangement%20of%20OMD-C%20and%20MMDH%20for%20the%20Adjustment%20to%20Market%20Data%20Dissemination%20in%20Relation%20To%20Northbound%20Trading%20under%20Stock%20Connect_C.pdf)）。
- 盘后披露新规则：每日收市后披露沪深股通**成交总额及总笔数、ETF 成交额、前十大成交活跃证券**；按月/年汇总；**每季度**第 5 个交易日公布上季度末单只证券北向持股数（原每日披露改为季度）。港股通（南向）同样改为盘后披露买卖金额/笔数/十大活跃股，每日收盘后披露单只证券持股。
- **落地含义**：任何"盘中北向资金流向"因子已无数据来源（akshare 相关接口因此失效）；因子库应转向"盘后十大活跃股 + 季度持股变动"，并在数据字典中记录该结构性断点（2024-05-13）。

---

## 5. 宏观、利率、汇率、大宗商品、美股/港股

| 数据 | akshare（免费） | tushare | baostock | 其他 |
|---|---|---|---|---|
| 国内宏观（GDP/CPI/PPI/PMI/社融/货币供应） | ✅（统计局/央行/金十等多接口） | ✅（体系化，含发布日程） | ✅（仅货币供应/存贷款利率/准备金率） | — |
| 利率（Shibor/LPR/国债收益率曲线） | ✅ | ✅（Shibor/报价/LPR/Libor/Hibor/国债收益率曲线/民间借贷） | 部分 | — |
| 汇率 | ✅（央行/新浪等） | ✅（外汇日线） | ❌ | yfinance 备选 |
| 大宗商品/黄金 | ✅（期货、上海金、现货，接口多） | ✅（期货/现货黄金/南华指数） | ❌ | — |
| 美股/港股行情 | ✅（东财/新浪源，免费） | ✅（日线+复权+财报，**付费独立项**） | ❌ | yfinance（免费，未逐一核实本次） |
| 国际指数 | ✅ | ✅（国际主要指数、VIX） | ❌ | — |

- **推荐**：宏观/利率/汇率/商品用 **akshare 覆盖 + tushare 交叉**；外围股市若只做日线级联动因子，akshare 免费源足够，若需美股全历史+财报则 tushare 美股包（2000+500 元/年）或 yfinance。
- akshare 的宏观接口散落在多个上游（统计局/央行/外汇局/金十），字段不统一——**需要自写标准化层**（见 §10）。

---

## 6. 成熟开源数据框架 / 落地方案

### 6.1 microsoft/qlib —— 数据层是其最可复用的部分
- **GitHub**：[microsoft/qlib](https://github.com/microsoft/qlib)；**48,746 stars**；MIT；最近提交 2026-09-22（微软持续维护，与 RD-Agent 联动，非常活跃）。
- **数据层要点**（[Data Layer 文档](https://qlib.readthedocs.io/en/latest/component/data.html)）：
  - **Qlib 文件格式**：`.calendars` / `.instruments` / `.features/<code>/<field>` 二进制文件，按日历对齐，读取极快；`dump_bin.py` 可把 **CSV/Parquet 转换成 Qlib 格式**（也有社区 dump_mysql/dump_db 扩展）；
  - **DataLoader**：`QlibDataLoader`（表达式引擎算因子）、`StaticDataLoader`（任意静态表），接口可自定义，天然对接 pandas；
  - **PIT 数据库**（[Point-in-Time 文档](https://qlib.readthedocs.io/en/latest/advanced/PIT.html)）：文件式设计，每特征 4 列 `(date=披露日, period=报告期, value, _next=下一版本字节偏移)` + `.index` 索引，**专门解决财务数据被追溯修订导致的未来信息泄漏**；`PITProvider`/`LocalPITProvider` 查询"某历史时刻可见的版本"；
  - **data_collector**：官方爬取/转换脚手架（Yahoo、`baostock_5min` 等，见 [scripts/data_collector](https://github.com/microsoft/qlib/tree/main/scripts/data_collector)），含自动日频更新与**数据健康检查**脚本；
  - 缓存体系（ExpressionCache/DatasetCache）、离线/在线（qlib-server）两种服务模式。
- **Apple Silicon**：纯 Python + C 扩展轮子，arm64 可用（FAQ 提示 `qlib.data._libs.rolling` 需安装对应轮子）；作为数据层不涉及 GPU。
- **收费**：免费（数据包 crowd-sourced，行情数据原点为 Yahoo/公开源）。
- **推荐理由**：**"数据层 + PIT"直接复用**：用 akshare/baostock/tushare 采集 → 转 Parquet/CSV → `dump_bin` 进 Qlib 格式 → 直接获得对齐、缺失处理、表达式因子、PIT 查询。不必采用它的 ML 工作流。

### 6.2 hikyuu —— 速度优先的本地数据+回测一体
- **GitHub**：[fasiondog/hikyuu](https://github.com/fasiondog/hikyuu)；3,522 stars；Apache-2.0；最近提交 2026-09-22（非常活跃）。
- **数据层**：C++ 实现的高速本地数据读写（默认 SQLite/HDF5 等本地存储），支持通达信/pytdx、同花顺客户端导出、新浪/腾讯免费源等多种数据导入方式，支持前/后复权与除权除息处理。
- **收费**：免费；数据获取依赖上述公开源（同 akshare 的灰色地带）或券商客户端导出（合规性较好）。
- **Apple Silicon**：C++/Python 混合，官方提供 mac 支持但 wheel 情况随版本变化，建议在目标机实测编译。
- **推荐理由**：若追求"单机极限速度的指标计算/回测"，hikyuu 的数据层与因子组件值得复用；但其数据接入广度不如 akshare/tushare，**定位为研究计算引擎而非采集层**。

### 6.3 rqalpha —— 数据层可插拔，注意 License 限制
- **GitHub**：[ricequant/rqalpha](https://github.com/ricequant/rqalpha)；6,787 stars；最近提交 2026-09-22（活跃）；PyPI 最新 6.4.0。
- **License（重要）**：PyPI 标注 Apache-2.0，但 README/说明书明确 **"仅限非商业使用。如需商业使用，请联系我们：public@ricequant.com"**（[PyPI 页面](https://pypi.org/project/rqalpha/)）；GitHub 因附加条款将 SPDX 标为 NOASSERTION。**个人研究可用，任何商业用途需先获授权。**
- **数据层**：
  - 官方免费 **data bundle**（`rqalpha download-bundle`）：A 股日线为主的打包数据，直接驱动回测；bundle 为快照式发布，增量需要补充源；
  - `DataProxy`/`DataSource` 可插拔，官方扩展 [rqalpha-mod-rqdata](https://github.com/ricequant/rqalpha-mod-rqdata)（接付费 RQData）、rqalpha-mod-tushare（社区）；
  - 事件驱动回测引擎对停牌、除权除息、涨跌停、税费有成熟的默认处理，可作为**质量校验规则的参考实现**。
- **推荐理由**：复用其"数据源抽象 + 公司行为（分红/除权）处理逻辑"与免费 bundle 做基准对账；若系统可能商业化，把 rqalpha 代码隔离或只借鉴设计。

### 6.4 其他备选（未逐一深入核实）
- **pytdx / mootdx**（通达信行情协议，[mootdx org](https://github.com/orgs/mootdx/repositories)）：券商/通达信行情通道，实时性好；协议逆向性质，合规性同抓取类。
- **sric0880/quantdata**（[链接](https://github.com/sric0880/quantdata)）：对 tiledb/mongodb/tdengine/clickhouse/duckdb 等金融时序存储方案的对比与客户端实现，做选型时值得旁证。

---

## 7. 数据落地选型（Mac mini / Apple Silicon 单机）

| 存储 | 适用场景 | 在本系统中的定位 | Apple Silicon |
|---|---|---|---|
| **SQLite** | 单机嵌入式、低并发写、元数据/配置/任务状态 | 股票基础信息、交易日历、采集任务水位、数据质量报告 | ✅ 原生 |
| **PostgreSQL** | 多进程并发写、事务与 `ON CONFLICT` upsert、复杂 SQL/join、jsonb | 采集层事实表（当并发写入与强一致去重成为瓶颈时启用）；可加 TimescaleDB 做时序分区 | ✅ arm64 官方版 |
| **DuckDB** | 嵌入式 OLAP、列存向量化、**直接查询 Parquet**、单机聚合极快 | **研究查询引擎**：pandas/polars 无缝，SQL 即席分析，无需常驻服务 | ✅ arm64 预编译 |
| **Parquet** | 列存压缩、schema 演进、生态通用（pandas/polars/duckdb/qlib dump 都认） | **行情/财务事实表的最终归档格式**，按 `(market, freq, year)` 分区目录 | ✅ |
| **MinIO** | S3 兼容对象存储、大文件版本化/快照/跨机共享 | 冷备与快照仓库（TimeTravel 式回滚、多机扩展时）；单机阶段可延后 | ✅（Go，arm64 支持） |

**推荐组合（分阶段）**：
1. **起步（推荐）**：`采集 → Parquet（raw + clean 两层）` + `DuckDB 查询` + `SQLite 元数据/水位`。零运维、TimeTravel 简单（Parquet 目录快照）、全 arm64。
2. **当出现多进程并发写/事务需求**：事实表搬 PostgreSQL（按月分区表 + 主键 upsert），Parquet 降级为归档导出，DuckDB 可 `postgres_scanner` 联查。
3. **需要备份/版本化/多机**：MinIO 存 Parquet 快照与原始响应包（raw 层本身就是审计与"未来信息"追溯的证据）。
4. **量化流水线**：在 clean Parquet 之上用 qlib `dump_bin` 生成 Qlib 格式（另一份物化视图，可随时重建），供表达式因子/数据集使用。

**DataFrame / 计算层（任务书⑤补充）**：pandas 为生态事实标准（baostock/tushare/qlib/alphalens 全部原生返回或消费 DataFrame）；**polars**（MIT）可作大数据量扫描的可选加速层（读 Parquet 极快），两者经 Arrow/Parquet 互通。本方案以 pandas 为主、polars 为可选，不引入第二套计算范式。

**容量参考**：全 A 约 5400 股 × 30 年日线 ≈ 数千万行 → Parquet < 2 GB；5 分钟线 15 年 ≈ 数亿行 → 10~30 GB；1 分钟线 5 年 ≈ 数十 GB。Mac mini（16GB+ 内存 / 512GB+ SSD）完全够用；分钟级别冷数据可放 MinIO 或外接盘。

---

## 8. 增量更新与去重策略

**总体架构：raw → staging → clean 三层，全部幂等可重放。**

1. **调度**：macOS `launchd`（或 cron）每日收盘后（如 17:30）跑 EOD 任务；分钟线盘中增量与盘后补全分开；宏观/财务按各自披露节奏（财报季日频轮询）。
2. **水位（watermark）**：SQLite 记录 `(dataset, code, freq, last_date, ingested_at)`；增量窗口固定回看 N 日（如 5 个交易日）**重新抓取并覆盖**，以吸收上游的追溯修订（分红实施、财报更正、复权因子变化）。
3. **去重/主键**：
   - 行情：`(code, freq, trade_date)`（分钟加 `bar_time`）为主键，写入用 `INSERT ... ON CONFLICT DO UPDATE`（PG）或 Parquet 分区内 `MERGE INTO`/pandas concat 后 drop_duplicates(keep='last')；
   - 财务：`(code, report_period, item)` + `ann_date`（披露日）保留**多版本**，不覆盖旧行——这就是 PIT；
   - 板块成分：`(classification, code, in_date, out_date)` 区间表。
4. **多源仲裁**：baostock 为主源，akshare/tushare 为校验源；差异超阈值（如收盘价 >0.5% 或复权因子不一致）时写入 `dq_issue` 表人工复核，不静默覆盖。
5. **事件驱动回填**：除权除息/送转股发生时，**回填复权因子并重算该股全历史复权价**（前复权价是"随时间漂移"的，绝不能只存前复权价——必须存**不复权价 + 复权因子**，查询时动态计算）。
6. **补缺**：新股上市回拉全历史；退市股打 `delist_date` 但**永不删除数据**；停牌日显式写 `tradestatus=0` 行或维护停牌日历（勿简单跳过造成"隐形合并"）。
7. **重放能力**：raw 层保存原始响应（JSON/CSV），ETL 代码版本化，任何 schema 变更可全量重建 clean 层。

---

## 9. 数据质量校验方案（停牌 / 复权 / 除权除息 / 退市与幸存者偏差）

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
- 复权因子双源校验：baostock `query_adjust_factor` vs tushare `adj_factor`（东财的"涨跌幅复权法"与通达信"定点复权法"结果略有差异，baostock 官方已注明口径）；
- **除权除息一致性检验**：对每个除权日验证 `preclose ≈ (前日收盘 - 每股派现) / (1 + 送转比例)`（按交易所公式），偏差超阈值即报 dq_issue；
- 注意分红实施公告晚于预案：分红数据本身也要按 `ann_date` 做 PIT。

### 9.4 退市股票与幸存者偏差
- **必须全量保存退市股票**：baostock 的 `query_stock_basic` 提供 `ipoDate/outDate/status` 且历史 K 线含退市股；tushare 有"股票历史列表/股票曾用名/每日股本"；东财抓取源对退市股覆盖不全，**不可作为唯一来源**；
- 维护 **universe 快照表**：`(trade_date, code)` 的"当日真实可交易股票池"（含当日 ST 状态、上市/退市状态），回测选股只能用当日快照，绝不用"当前股票列表"回溯；
- 退市整理期、重大资产重组导致的长停牌要在 universe 中如实反映（否则回测会买"实际不可买"的股票）；
- IPO 初期（次新股）与退市前期（仙股/流动性枯竭）建议单独设过滤规则并记录在研究笔记中。

### 9.5 财务数据 PIT 校验
- 任何基本面/因子回测，数据集构造必须按"报告期 + 披露日"过滤（`statDate ≤ t` 且 `ann_date ≤ t`）；
- 自建 PIT 表（§8.4 多版本保留）或复用 qlib PIT 格式/付费 PIT API；定期抽查若干股票的财报修订记录验证 `_next` 链条完整性。

---

## 10. 直接复用 vs 需要自己写

| 环节 | 直接复用（不写采集系统） | 需要自己写的薄层 |
|---|---|---|
| 行情采集 | baostock / akshare / tushare / efinance 客户端 | 调度（launchd）、水位、多源仲裁 |
| 财务/基本面 | tushare / akshare 财报接口、baostock 季频指标 | PIT 版本表（或复用 qlib PIT 格式与 dump 工具） |
| 板块/资金流/两融/龙虎榜 | akshare / tushare | 板块成分历史快照表（防未来信息） |
| 北向/南向 | tushare 盘后接口 | 新披露规则下的因子重构（结构性工作，无法复用旧代码） |
| 宏观/利率/汇率/大宗/港美 | akshare / tushare | 字段标准化与统一日历 |
| 研究数据层 | qlib（Qlib 格式、DataLoader、PIT、dump_bin、健康检查）、hikyuu 计算层、rqalpha 公司行为处理 | Parquet→Qlib 的 ETL 管道脚本 |
| 存储 | DuckDB/Parquet/SQLite/PG/MinIO 现成组件 | 表分区与 schema 设计（§7/§8） |
| 质量校验 | rqalpha 撮合规则、qlib check_health 脚本作参考 | DQ 规则集与报告（§9，约数百行代码量级） |

**结论**：采集系统 100% 复用现成项目；自写部分收敛为"调度 + ETL + 校验 + PIT 表"四件薄层，符合"不自己写采集系统"的原则。

---

## 11. 推荐落地路线

1. **第 1 周**：装 baostock + akshare + tushare（先免费积分），跑通日线全量入库（含退市股）→ Parquet；SQLite 建水位/日历/证券信息。
2. **第 2 周**：复权因子、分红送转、停牌日历入库；写 DQ 报告任务（§9.1/9.3）。
3. **第 3 周**：开通 tushare 2000 积分（200 元/年），补财务三大表/股本/两融/龙虎榜/申万分类；建 PIT 财务版本表。
4. **第 4 周**：分钟线增量（baostock 5 分钟起，2011 至今）；DuckDB 视图层；`dump_bin` 生成 Qlib 格式接入研究流水线。
5. 持续：多源对账与 dq_issue 复核；需要时上 PostgreSQL/MinIO；有商业化可能时隔离 rqalpha 代码。

---

## 12. 主要来源

**项目状态（GitHub API / PyPI 实时核实，2026-09-22）**
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

**商业数据服务**
- [JQData 使用说明](https://www.joinquant.com/help/api/doc?name=JQDatadoc&id=10514) · [试用和购买说明](https://joinquant.com/help/api/doc?id=9827&name=logon) · [JQData 本地化公测公告](https://www.joinquant.com/view/community/detail/4d575f700a403ed4186eaef6ec57d4e7)
- [米筐产品价格](https://rqopen.ricequant.com/welcome/pricing) · [RQData 介绍](https://www.ricequant.com/welcome/rqdata) · [vnpy 论坛价格讨论](https://www.vnpy.com/forum/topic/29498-qing-wen-yi-xia-3000yuan-shou-jie-de-radata-bao-han-gu-piao-shu-ju-yao)

**规则与市场事实**
- [沪深港通交易信息披露机制调整（证券时报e公司）](https://www.163.com/dy/article/J1U3GV9N0519D3V1.html) · [HKEX 通告 PDF](https://www.hkex.com.hk/chi/prod/dataprod/Documents/24-04-12%20Arrangement%20of%20OMD-C%20and%20MMDH%20for%20the%20Adjustment%20to%20Market%20Data%20Dissemination%20in%20Relation%20To%20Northbound%20Trading%20under%20Stock%20Connect_C.pdf) · [北向接口失效分析](https://quant.csdn.net/691d70325511483559ebf51f.html)

**存储选型旁证**
- [sric0880/quantdata（金融时序库选型对比）](https://github.com/sric0880/quantdata)

> 免责声明：抓取类项目（akshare/efinance/qstock/Ashare/easyquotation）的数据来自各网站公开接口，可能违反上游 ToS 且随时失效；个人研究使用惯例上被容忍，但**不得用于对外商业产品**。stars 与提交时间为 2026-09-22 快照，会随时间变化。
