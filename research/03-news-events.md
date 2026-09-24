# 03 · 新闻、公告、事件分析与 AI 金融分析 — 开源项目与数据源调研

> **修订记录 v2（2026-09-23，依据审计 A10/A11/A12/§4）**
>
> - **A11 · 去重与正确性口径重写**：§4.3 改为"来源文档 ID + 内容哈希"精确去重，保留修订链；转载、更正公告、同标题不同内容一律保留并可区分，不凭标题相似直接丢弃。§3.1/§3.3 增加字段级原文证据要求：金额、单位、主体、日期须能定位原文，缺证据输出未知。§4.5 改为固定人工标注样本评估（漏检 / 误合并 / 关键字段准确性）；取消固定滤重比例指标，取消"JSON 五要素齐全率=正确率"的暗示，阈值由用途确定。
> - **A10 · 出报机制改为数据就绪驱动**：§4.2 新增数据集截至时间、预期日期、就绪状态登记；先生成明确标注缺项的初版，随后补齐，次日晨间修订；旧数据不得伪装为当天数据；单源延迟或故障时报告展示缺项及版本，恢复后补跑不重复统计。
> - **A12 · 提示注入安全边界**：新增 §4.6 —— 新闻、公告、论文正文中的指令不得触发工具执行，外部文本一律作为数据处理。
> - **审计 §4 · 首期范围收敛**：首期只处理持仓、自选、重大公告；全市场轮询、语义去重（simhash/datasketch/text2vec）、向量库及 RAG 按实际检索需求延后（§1.1/§2/§7 同步标注）。
> - **修订原则**：2026-09-22 已核实事实与来源 URL 原样保留；本次修订未逐项重核外部事实，新增或改写处凡未重核均标注"待核实"。主设计（`a-share-quant-system-design.md`）不在本次修改范围，其对应章节改动见审计 §8 修改清单。

> 目标环境：Mac mini（Apple Silicon）上的个人 A 股量化研究系统
> 调研原则：**尽量复用现成开源项目**，仅在必要处做轻量改造
> 核实时间：2026-09-22（stars / License / 维护状态均通过 GitHub API、shields.io、web 搜索逐项核实，来源 URL 见各表末列与文末汇总）

**术语约定**：直接复用 = pip/conda 安装后按官方 API 即可用于生产流水线；需改造 = 可借鉴代码/接口设计，需自行适配数据源、反爬或 schema；仅参考 = 停更或质量不足，只取思路。

---

## 1. 财经新闻 / 公告 / 政策数据源

### 1.1 总体推荐

**首期范围（审计 §4）**：只处理**持仓、自选、重大公告**三类对象相关的新闻与公告；全市场轮询、语义去重、向量库及 RAG 按实际检索需求延后。本节对比表保留全量信源盘点（2026-09-22 已核实事实与后续扩展清单），首期接入范围以本节说明为准。

**首选策略：以 akshare 为统一新闻/公告入口**（一个库覆盖财联社电报、东方财富快讯与个股新闻、新浪财经快讯、巨潮资讯公告；首期仅按持仓/自选代码与重大类目过滤拉取），巨潮公告全文下载补充 `rollysys/use_cninfo`（PDF→Markdown，直接对接 LLM），tushare pro 作公告/结构化事件数据的第二来源。独立爬虫仓库普遍停更且无 License，只作参考。

### 1.2 对比表

| 项目 | GitHub 地址 | Stars | License | 维护状态 | 中文能力 | 直接复用 vs 需改造 | 推荐理由 |
|---|---|---|---|---|---|---|---|
| **akshare** | https://github.com/akfamily/akshare | 22,692 | MIT | ✅ 活跃（pushed 2026-09-20） | 原生中文 | **直接复用** | 一库覆盖全部四类信源：`stock_info_global_cls`（财联社电报）、`stock_info_global_em`（东方财富全球财经快讯）、`stock_info_global_sina`（新浪财经全球快讯）、`stock_news_em`（个股新闻）、`stock_notice_report` / 巨潮公告披露接口；纯 Python、免 key、文档全中文 |
| **tushare pro**（SDK：waditu/tushare） | https://github.com/waditu/tushare | 15,409 | BSD-3-Clause | ⚠️ SDK 仓库 pushed 2024-03 后停滞，但 tushare.pro 平台接口仍在维护 | 原生中文 | **直接复用**（走 pro API，需注册积分） | 公告/年报/业绩预告等结构化披露数据质量高、字段规整，适合做"事件事实底座"；与 akshare 互为备份 |
| **efinance** | https://github.com/Micro-sheep/efinance | 4,067 | MIT | ✅ 维护中（pushed 2026-07-17） | 原生中文 | 直接复用（补充） | 东方财富基金/股票/债券/期货数据，接口简洁，可作东财源备份 |
| **use_cninfo**（巨潮公告） | https://github.com/rollysys/use_cninfo | 29 | MIT | ✅ 2026-05 新项目 | 原生中文 | **直接复用** | 巨潮公告按需抓取 + PDF 经 PyMuPDF 转全文 Markdown + 公告类目过滤，产出格式天然适配 LLM 事件抽取，是公告全文链路的最佳现成件 |
| **CnInfoReports** | https://github.com/tr1s7an/CnInfoReports | 32 | MIT | ⚠️ pushed 2023-02 | 中文 | 需改造 | 巨潮公告批量下载器，逻辑简单可取；接口旧，需按现行 cninfo `announcement/query` 接口修 |
| **cailianpress-unified**（财联社） | https://github.com/caimao9539/cailianpress-unified | 9 | MIT | ✅ pushed 2026-03 | 原生中文 | 需改造 | 财联社电报/加红/热度/文章详情统一接口封装，逆向了 cls.cn 接口，省去自己抓包；量小但实用 |
| **FinSpider**（韭研公社/东财股吧） | https://github.com/jiaweif3ng/FinSpider | 13 | ⚠️ 无 License | ⚠️ pushed 2025-02 | 中文 | 需改造 | 东财股吧/韭研公社爬虫（散户情绪源），代码量小；无 License，仅可参考实现自行重写 |
| **finance_spider_data_analysis** | https://github.com/Anton-Mu/finance_spider_data_analysis | 65 | ⚠️ 无 License | ❌ 停更（2023-03） | 中文 | 仅参考 | 东财股吧+新浪财经爬虫并带情感分析报告生成的完整教程，流水线设计可借鉴 |
| **wallstreetcnScrapy** | https://github.com/jianzhichun/wallstreetcnScrapy | 35 | ⚠️ 无 License | ❌ 停更（2016） | 中文 | 仅参考 | 华尔街见闻/新浪/同花顺 Scrapy 爬虫，仅取 Scrapy 工程结构 |
| CnInfoHedgeCrawler | https://github.com/Interstellar1217/CNInfoHedgeCrawler | 12 | MIT | ✅ pushed 2026-06 | 中文 | 需改造 | 巨潮公告自动爬取工具，较新，可与 use_cninfo 对比选型 |
| CninfoDistributedSpider | https://github.com/flicck/CninfoDistributedSpider | 19 | Apache-2.0 | ❌ 停更（2019） | 中文 | 仅参考 | scrapy+kafka 分布式公告爬虫，个人场景用不上分布式 |
| cninfo-search-python | https://github.com/Poncirus/cninfo-search-python | 2 | ⚠️ 无 License | 不活跃 | 中文 | 仅参考 | 按关键词搜索/屏蔽/下载巨潮公告，查询逻辑可抄 |
| 新浪财经长尾爬虫（WES6/finance、chenmj11/sina_finance、bhcbhc/finance_sina_crawler 等） | 见来源 | 2–3 | 多数无 License | ❌ 全部停更 | 中文 | 不建议 | 页面结构早已变化；新浪快讯直接用 akshare `stock_info_global_sina` |

### 1.3 各信源接入要点

- **财联社（cls.cn）**：电报流是 A 股事件最快的公开源之一。首期只按持仓/自选代码与重大类目过滤拉取 `ak.stock_info_global_cls()`（轮询电报列表）；需要"加红/热度/文章详情"时用 cailianpress-unified 的接口封装。财联社快讯短，适合做"事件触发器"，正文细节再回源抓文章。全市场轮询延后（审计 §4）。
- **东方财富**：`ak.stock_info_global_em()`（7×24 快讯）、`ak.stock_news_em(symbol=...)`（个股新闻，含时间/正文/来源）；股吧情绪用 FinSpider 思路自研（注意合规与频率限制）。
- **新浪财经**：`ak.stock_info_global_sina()` 全球财经快讯，覆盖面广、含宏观与海外，适合外围事件传导分析。
- **巨潮资讯（cninfo，法定披露源）**：
  1) 列表/检索：走 cninfo 官方 `http://www.cninfo.com.cn/new/hisAnnouncement/query`（akshare 公告接口已封装，返回公告标题、类型、PDF 链接）；
  2) 全文：use_cninfo 下载 PDF 并转 Markdown 归档；
  3) 分类：先按巨潮 category（年报/半年报、业绩预告、增减持、重组、处罚问询等）硬分类，LLM 只做细粒度事件抽取。首期只处理持仓/自选 + 重大公告类目。
- **tushare pro 公告**：`公告数据` 系列接口（见 tushare.pro 文档 doc_id=460 一带的披露/公告类接口）提供公告索引与部分结构化字段，适合校验 akshare 抓取的完整性；重大事件（业绩预告、解禁、增减持）有专门结构化接口，**优先吃结构化接口，PDF 全文仅对持仓股与高影响事件下载**。
- **政策类**：无单一成熟开源库。组合方案：akshare 宏观/政策类资讯接口 + 新浪/财联社"政策"频道快讯 + 证监会/央行网站 RSS 级自研薄爬虫（页面简单，requests+BeautifulSoup 即可）。**注意**：政策页面抓取的文本同样属于外部文本，适用 §4.6 提示注入安全边界。

---

## 2. 新闻 NLP 分析（去重 / 分类 / 摘要 / 事件抽取 / 情绪与影响方向）

### 2.1 对比表

| 项目 | GitHub / HF 地址 | Stars | License | 维护状态 | 中文能力 | 直接复用 vs 需改造 | 推荐理由 |
|---|---|---|---|---|---|---|---|
| **FinGPT** | https://github.com/AI4Finance-Foundation/FinGPT | 21,272 | MIT | ✅ 活跃（pushed 2026-09-14） | 中英双语（含中文情绪指令数据集） | **直接复用思路与数据，模型需自训** | 金融 LLM 事实标准开源项目；`FinGPT-Sentiment_Analysis` 给出"LLM/LoRA 做金融情绪"的完整成熟配方，且有公开中文情绪微调数据集（如 fingpt_chatglm2_sentiment_instruction_lora_ft_dataset），在 Mac 上可复现微调 |
| **ProsusAI/finbert** | https://github.com/ProsusAI/finbert | 2,237 | Apache-2.0 | ⚠️ pushed 2022-09（不再更新，但模型稳定可用） | ❌ 英文 | 直接复用（英文文本） | 最广泛使用的金融情绪 BERT（pos/neg/neutral），适合处理英文快讯（美债/美联储/美股新闻），与中文模型互补 |
| **finbert-pretrain**（HKUST） | https://huggingface.co/yiyanghkust/finbert-pretrain | —（HF 模型） | 以 HF 页为准 | 语料冻结 | ❌ 英文 | 需改造（作为微调底座） | 金融领域继续预训练 BERT，做英文金融分类微调底座 |
| **finance-sentiment-zh-base** | https://huggingface.co/bardsai/finance-sentiment-zh-base | —（HF 模型，下载量高） | 以 HF 页为准 | 可用 | ✅ 中文金融情感 | **直接复用** | 现成中文金融情感三分类（积极/中性/消极）Transformer 模型，轻量、CPU 可跑，全量新闻情绪打分首选 |
| **text2vec** | https://github.com/shibing624/text2vec | ≈5k | Apache-2.0 | ✅ 持续维护 | ✅ 中文为一等公民 | **延后引入（审计 §4）** | 中文句向量/语义相似度（Word2Vec、Sentence-BERT、CoSENT）；语义去重与新闻聚类属全市场场景，按实际检索需求延后 |
| **simhash** | https://github.com/1e0ng/simhash | ≈1k | MIT | 稳定（小而成熟） | 支持中文（字符 n-gram） | **延后引入（审计 §4）** | SimHash 指纹 + 汉明距离的标题级近重复去重标准实现；首期不按标题相似丢弃（A11），仅在确需大规模近似召回时引入 |
| **datasketch** | https://github.com/ekzhu/datasketch | ≈3k | MIT | ✅ 维护 | 语言无关 | **延后引入（审计 §4）** | MinHash/LSH 大规模近似去重与聚类候选对召回；随全市场轮询一起延后 |
| **DeepKE** | https://github.com/zjunlp/DeepKE | ≈4.5k | MIT | ✅ 维护（EMNLP 2022 工具箱） | ✅ 中文场景丰富 | 需改造（schema 适配） | 浙大知识图谱抽取工具箱：命名实体（NER）、关系（RE）、事件抽取（EE）全链路 pipeline + 中文预训练权重，传统小模型方案里最成熟 |
| **OneKE** | https://github.com/zjunlp/OneKE | 194 | MIT | ✅ 维护 | ✅ 中文 | 需改造 | 基于 LLM 的 schema 约束信息抽取（含事件抽取），可直接产出结构化事件 JSON |
| **DISC-FinLLM** | https://github.com/FudanDISC/DISC-FinLLM | 895 | Apache-2.0 | ❌ pushed 2023-11 停更 | ✅ 原生中文金融 | 参考复用（prompt/数据） | 复旦中文金融 LLM：金融信息抽取、Chain-of-Retrieval RAG 的公开实现与数据配方，抽取/RAG prompt 设计可直接抄 |
| **TradingAgents** | https://github.com/TauricResearch/TradingAgents | 108,032 | Apache-2.0 | ✅ 非常活跃（pushed 2026-09-18） | 中文可用（取决于底层 LLM） | 需改造（数据源换 A 股） | 最火的多智能体 LLM 交易框架，其 News/Social Media Analyst + 情绪聚合 + 多空辩论的"事件→交易观点"架构是现成范式，值得整段借鉴 |
| **FinRobot** | https://github.com/AI4Finance-Foundation/FinRobot | 8,049 | Apache-2.0 | ✅ 活跃（pushed 2026-09-11） | 部分中文 | 需改造 | AI4Finance 的金融 Agent 平台，含公司情绪分析、新闻研究报告生成等现成 Agent，可拆其 prompt 与流程 |
| **FinNLP** | https://github.com/AI4Finance-Foundation/FinNLP | 1,487 | MIT | ⚠️ pushed 2024-07 半停更 | 中英 | 参考复用（数据集） | 金融 NLP 数据管道与数据集汇总（新闻情绪、NER 数据等），省去自标注成本 |
| **RAGFlow** | https://github.com/infiniflow/ragflow | ≈91k | Apache-2.0 | ✅ 非常活跃 | ✅ 中文文档/切分优化 | **延后引入（审计 §4）** | 成熟 RAG 引擎（深度文档解析、切分、检索、引用溯源）；公告问答/证据检索属按需能力，首期不建 |
| **LlamaIndex** | https://github.com/run-llama/llama_index | ≈52k | MIT | ✅ 非常活跃 | 中文可用 | **延后引入（审计 §4）** | 轻量嵌入式 RAG；向量库及 RAG 按实际检索需求延后（审计 §4），首期证据追溯走 §3.3 字段级原文定位 |
| 词向量：Tencent AI Lab 中文词向量 | https://ai.tencent.com/ailab/nlp/en/embedding.html（下载渠道已不稳定，可搜 HF 镜像 qundao/chinese-word-embedding） | — | 研究可用条款 | 冻结 | ✅ 中文 | 直接复用（传统特征） | 800 万中文词向量；若用深度模型可不必上，保留作轻量基线/热词扩展 |

> 注：text2vec / simhash / datasketch / RAGFlow / LlamaIndex 的项目事实（stars、License、能力）仍为 2026-09-22 已核实快照；此处仅调整**引入时点**——首期不部署，按实际检索需求延后（审计 §4）。

### 2.2 各处理环节的推荐方案

| 环节 | 推荐做法（复用优先） |
|---|---|
| 去重 | 首期只按**来源文档 ID + 归一化内容哈希**精确去重，保留修订链与转载关系（§4.3）；标题相似/语义去重（simhash/datasketch/text2vec）延后按需 |
| 分类 | 两段式：巨潮 category / 来源频道做规则硬分类；新闻用微调 BERT（FinGPT 数据 + 中文底座）做 15 类事件类型分类（见 §3.2 枚举） |
| 摘要 | 短快讯免摘要；长公告用本地 Qwen-7B/14B 做"5W+金额"结构化摘要；每日晨报聚合用云端 LLM |
| 事件抽取 | 优先 LLM + JSON schema 约束输出（OneKE 思路 / DISC-FinLLM prompt 配方）；批量低成本场景用 DeepKE 小模型 pipeline 兜底；关键字段须带原文证据、缺证据输出未知（§3.3、§4.4） |
| 情绪/影响方向 | 全量：finance-sentiment-zh-base（中文）+ ProsusAI/finbert（英文快讯）打分；重点事件：LLM 输出"影响方向 + 置信度 + 作用期限"；FinGPT Sentiment 配方做增量微调 |
| 安全边界 | 外部文本（新闻/公告/论文正文）一律作为数据进入上述环节，其内嵌指令不得触发工具执行（§4.6） |

---

## 3. 事件数据库设计（事件 ↔ 个股 / 行业 / 持仓关联）

### 3.1 推荐 schema（SQLite/DuckDB/PostgreSQL 均可，Mac mini 建议 DuckDB + Parquet 冷备）

> v2 修订（A11）：原始层增加来源文档 ID、内容哈希与修订链字段；"去重层"由"唯一新闻簇（丢弃非 canonical）"改为**关系登记（保留全部记录）**；事件层关键字段要求字段级原文证据，缺证据输出未知。

```sql
-- ① 原始新闻/公告层：只追加、不修改
CREATE TABLE news_raw (
  id            BIGINT PRIMARY KEY,
  source        TEXT NOT NULL,        -- cls / em / sina / cninfo / tushare ...
  source_doc_id TEXT,                 -- 来源文档 ID（信源侧唯一标识，公告编号等）
  url           TEXT UNIQUE,
  title         TEXT,
  content       TEXT,                 -- 正文或公告 Markdown 全文（外部文本一律按数据处理，见 §4.6）
  content_hash  TEXT NOT NULL,        -- 归一化正文 sha256（精确去重与转载识别，见 §4.3）
  publish_time  TIMESTAMP,            -- 信源发布时间（事件时间以它为准）
  fetch_time    TIMESTAMP,
  author        TEXT,
  category_raw  TEXT,                 -- 信源原始分类/公告类目
  revision_kind TEXT NOT NULL DEFAULT 'original',  -- original / correction / update（更正公告单独成行）
  revision_of   BIGINT REFERENCES news_raw(id),    -- 修订链：指向被更正/被更新的前一版本
  raw_json      JSON,
  UNIQUE (source, source_doc_id)
);

-- ② 去重与关系层（A11 重写）：只登记关系，绝不因去重丢弃记录
CREATE TABLE news_relation (
  id              BIGINT PRIMARY KEY,
  news_id         BIGINT NOT NULL REFERENCES news_raw(id),
  related_news_id BIGINT NOT NULL REFERENCES news_raw(id),
  relation        TEXT NOT NULL,      -- exact_duplicate(同源同文档重复抓取) / repost(跨源同内容转载)
                                      --   / revision(同源修订链) / same_title_diff_content(同标题不同内容，提示人工区分)
  method          TEXT NOT NULL,      -- source_doc_id / content_hash（首期仅此两种，见 §4.3）
  note            TEXT,
  UNIQUE (news_id, related_news_id, relation)
);

-- ③ 向量层（延后：向量库与 RAG 按实际检索需求引入，审计 §4；首期不建）
-- CREATE TABLE news_embedding (
--   news_id BIGINT, model TEXT, vector BLOB, PRIMARY KEY(news_id, model)
-- );

-- ④ 事件层：LLM 结构化抽取产出
CREATE TABLE event (
  id            BIGINT PRIMARY KEY,
  event_type    TEXT NOT NULL,        -- 见 §3.2 枚举
  sub_type      TEXT,
  occur_time    TIMESTAMP,            -- 事件实际发生时间（无原文证据时为 unknown）
  publish_time  TIMESTAMP,
  title         TEXT,
  summary       TEXT,                 -- 5W+金额 结构化摘要
  direction     TEXT,                 -- positive / negative / neutral（对市场总体）
  confidence    REAL,                 -- 0~1
  importance    REAL,                 -- 0~1 影响力预估
  horizon       TEXT,                 -- intraday / days / weeks
  evidence_news_ids JSON,             -- 证据来源 news_raw.id 列表（保留修订链与转载，见 §4.3）
  key_facts     JSON,                 -- 关键事实：金额/单位/主体/日期须逐字段带原文证据；缺证据值为 "unknown"（§3.3）
  extract_model TEXT,                 -- 抽取所用模型/版本
  extract_prompt_version TEXT,        -- 提示版本（可复现抽取结果）
  status        TEXT                  -- new / validated / consumed / invalidated
);

-- ⑤ 事件-实体关联（个股/行业/指数/宏观标的；多对多带权重与方向）
CREATE TABLE event_entity (
  event_id   BIGINT REFERENCES event(id),
  entity_type TEXT,                   -- stock / industry / index / macro / commodity
  entity_code TEXT,                   -- 600519.SH / SW.801080 / ^GSPC / DGS10
  entity_name TEXT,
  role        TEXT,                   -- target(标的自身) / peer(同业) / supply_chain / competitor
  direction   TEXT,                   -- 该实体视角的影响方向（同一事件对上下游可相反）
  weight      REAL,                   -- 影响权重 0~1
  evidence_quote TEXT,                -- 主体认定的原文片段（主体须能定位原文，见 §4.4）
  PRIMARY KEY (event_id, entity_type, entity_code)
);

-- ⑥ 持仓快照（事件与持仓关联就靠这三张表 JOIN）
CREATE TABLE position (
  date DATE, account TEXT, code TEXT, name TEXT, qty DOUBLE, weight DOUBLE,
  PRIMARY KEY(date, account, code)
);
CREATE TABLE watchlist (code TEXT PRIMARY KEY, name TEXT, tags JSON);
CREATE TABLE industry_map (code TEXT, industry_code TEXT, source TEXT, valid_from DATE, valid_to DATE);

-- ⑦ 事后复盘（事件信号质量闭环）
CREATE TABLE event_feedback (
  event_id BIGINT PRIMARY KEY,
  ret_1d DOUBLE, ret_5d DOUBLE, ret_20d DOUBLE,   -- 事件后超额收益
  hit    BOOLEAN,                                  -- 方向判断是否兑现
  note   TEXT
);
```

**关联查询示例**："今日影响我的持仓的事件"：

```sql
SELECT e.*, ee.entity_code, ee.direction, p.weight
FROM event e
JOIN event_entity ee ON ee.event_id = e.id
JOIN position p ON p.code = ee.entity_code AND p.date = CURRENT_DATE
WHERE e.publish_time >= CURRENT_DATE AND ee.weight >= 0.5
ORDER BY e.importance DESC;
```

行业传导：`event_entity.role IN ('peer','supply_chain')` + `industry_map` 扩展到持仓的同业/上下游，实现"事件 → 行业 → 持仓"的两跳归因。

### 3.2 事件类型枚举（分类器标签体系）

`业绩预告 / 定期财报 / 增减持 / 股票回购 / 并购重组 / 股权激励 / 定增配股 / 解禁 / 监管处罚问询 / 诉讼仲裁 / 中标订单合同 / 产品调价 / 产能扩产 / 股东质押冻结 / 高管变动 / ST与退市风险 / 行业政策 / 宏观数据 / 货币政策 / 海外市场传导 / 大宗商品 / 汇率 / 舆情传闻 / 其他`

### 3.3 结构化事件抽取输出格式（LLM JSON 输出契约）

> v2 修订（A11）：`key_facts` 改为**逐字段证据对象**（value / unit / evidence_quote / evidence_pos）；金额、单位、主体、日期须能定位原文，缺证据一律输出 `"unknown"`，禁止推断或补全。"五要素齐全"只是格式契约，**不代表内容正确**；正确性按 §4.5 的固定样本评估度量。

```json
{
  "event_type": "增减持",
  "sub_type": "高管拟减持",
  "occur_time": {"value": "2026-09-22", "evidence_quote": "……本公告披露日 2026 年 9 月 22 日……", "evidence_pos": "第1段"},
  "entities": [
    {"code": "300750.SZ", "name": "宁德时代", "role": "target",
     "direction": "negative", "weight": 0.9,
     "evidence_quote": "……宁德时代（300750.SZ）……", "evidence_pos": "第1段"}
  ],
  "direction": "negative",
  "horizon": "days",
  "confidence": 0.85,
  "importance": 0.6,
  "summary": "副总经理计划减持不超过 120 万股（约占总股本 0.03%），窗口期 3 个月。",
  "key_facts": {
    "amount_cny": {"value": 3.2e8, "unit": "CNY", "evidence_quote": "……减持金额不超过 3.2 亿元……", "evidence_pos": "第2段"},
    "share_pct": {"value": 0.03, "unit": "%", "evidence_quote": "……约占总股本 0.03%……", "evidence_pos": "第2段"},
    "duration":   {"value": "3个月", "unit": null, "evidence_quote": "……窗口期 3 个月……", "evidence_pos": "第2段"},
    "change_of_control": {"value": "unknown", "unit": null, "evidence_quote": null, "evidence_pos": null}
  },
  "evidence": {"source": "cninfo", "source_doc_id": "2026-045", "content_hash": "sha256:…", "quote": "……公告编号 2026-045……"}
}
```

字段即事件契约：事件类型、涉及标的、影响方向、置信度、证据来源五要素定义**输出格式**；`evidence` + 各字段 `evidence_quote` 保证可追溯到 `news_raw.content` 原文，`key_facts` 支撑后续量化回测。**校验规则**：
1. 每个 key_fact 必须能在 `news_raw.content` 中按 `evidence_quote` 定位复核；定位失败即判该字段错误（不是"齐全即正确"）。
2. 原文没有的信息输出 `"unknown"`，summary 中也不得出现无证据数字。
3. 金额必须带单位（`unit`），单位换算规则写入 note，不做静默换算。
4. 同一公告被更正时（`revision_kind='correction'`），抽取结果挂在对应版本上，旧版本结论标记 superseded，不覆盖（见 §4.3）。

---

## 4. 每日信息流水线建议

### 4.1 首期采集范围与频率（范围收敛，审计 §4）

首期只处理**持仓、自选、重大公告**。下表按首期范围给出频率；全市场轮询、语义去重、向量库与 RAG 延后到出现实际检索需求时再评估。

| 数据 | 首期范围 | 频率 | 说明 |
|---|---|---|---|
| 财联社电报 | 仅按持仓/自选代码与重大类目过滤 | 盘前盘中 5–15 分钟轮询；盘后 30 分钟 | 事件触发器，增量拉取按 `(source, source_doc_id)` 幂等 |
| 东财/新浪 7×24 快讯 + 个股新闻 | 仅持仓/自选股 | 15–30 分钟 | 交易时段可加密；不拉全市场流 |
| 巨潮公告列表 | 持仓/自选 + 重大类目（业绩预告、定期财报、增减持、并购重组、处罚问询、退市风险、解禁等） | 交易日 15:00–23:00 每 60 分钟对账；次日 08:00 补漏 | 公告多在盘后与深夜披露，晨间补一次 |
| 公告 PDF 全文 | **仅持仓股 + 重大类目 + 高影响事件** | 随列表增量 | 控磁盘与解析成本 |
| tushare 结构化事件（业绩预告/解禁/增减持） | 持仓/自选 + 重大事件 | 每日 1 次（盘后） | 作事实校验层 |
| 外围市场（yfinance/akshare） | 重大事件相关外围资产（美债/美元/黄金/原油/美股/港股） | 每日收盘后 1 次 + 次日 08:00 前补一次 | 覆盖外围传导 |
| FRED 宏观序列 | 宏观事件背景 | 每周 1 次 | 频率低，无需每日 |
| 事件抽取（LLM） | 上述范围内增量 | 盘中批量（15–30 分钟一批），公告当晚批处理 | 与抓取解耦，走队列；抽取模型无工具权限（§4.6） |
| 晨报生成 | 持仓/自选事件聚合 | 每交易日 08:30 | 同时是前一日初版报告的**修订版**（见 §4.2） |

**延后项（审计 §4）**：全市场快讯轮询、股吧/社媒全量情绪、语义去重（simhash/datasketch/text2vec）、向量库与 RAG（LlamaIndex/RAGFlow）。升级条件：出现"按语义检索历史事件/论文"的实际需求并能说明收益后，单独评估引入。

### 4.2 数据就绪驱动出报（A10 修订）

固定出报时点不等于数据齐备。出报由**数据就绪状态**驱动，不由时钟单独决定：

1. **就绪登记**：每个数据集记录 `cutoff_time`（数据截至时间）、`expected_date`（预期日期）、`ready_status`（ready / partial / missing / stale）、`missing_items`、`report_version`。示例：

   | 数据集 | cutoff_time | expected_date | ready_status | missing_items |
   |---|---|---|---|---|
   | 日线行情 | 2026-09-23 16:10 | 2026-09-23 | ready | — |
   | 两融 | 2026-09-22 20:00 | 2026-09-23 | missing | 全部 |
   | 巨潮公告列表 | 2026-09-23 22:40 | 2026-09-23 | partial | 23:00 后批次 |

2. **三段式出报**：
   - **初版**：达到最低就绪门槛即生成（如持仓行情 + 公告列表），报告头部显著标注**缺项清单与各数据集截至时间**，缺项板块输出"数据未就绪"，不留空白假象；
   - **补齐**：各源陆续就绪后增量补齐，生成新 `report_version`，注明相对初版的差异；
   - **次日晨间修订**：08:30 晨报合并夜间补漏（公告凌晨批次等），形成前一交易日的修订版，修订记录入报告尾部。
3. **旧数据不得伪装为当天数据**：任何沿用旧值的板块必须标注"截至 YYYY-MM-DD"（stale 标记）；预期日期与实际 cutoff 不一致时以 cutoff 为准展示。
4. **单源延迟/故障演练（验收项）**：模拟任一单源延迟或故障，报告必须展示该源的缺项状态与报告版本号，不得静默用旧数据顶替或省略。
5. **恢复后补跑不重复统计**：补跑以 `(dataset, expected_date, 幂等键)` 写入（新闻按 `(source, source_doc_id)`，公告按公告编号），事件计数、条数统计、费用统计均按幂等键去重；同一 `expected_date` 的补跑只更新对应数据集分区并 bump 报告版本，不重复计入当日已统计口径。

### 4.3 去重与修订链（A11 重写）

首期**只按来源文档 ID 和内容哈希去重**，且去重只做"关系登记"，不丢弃任何记录：

1. **来源文档 ID 精确去重**：同一 `(source, source_doc_id)` 的重复抓取只保留一条 `news_raw`（幂等），重复抓取记 `fetch_time` 追加日志。
2. **内容哈希**：`content_hash = sha256(归一化正文)`（去空白、全半角与标点归一后）。跨源 `content_hash` 相同 → 标记 `relation='repost'`（转载）：**全部保留**，以 `news_relation` 可区分来源与时间，不合并、不丢弃。
3. **修订链**：同一 `(source, source_doc_id)` 的更正/更新公告以 `revision_kind='correction'/'update'` + `revision_of` 单独成行，多版本全部保留；抽取结果绑定具体版本，旧版本结论标记 superseded。
4. **同标题不同内容必须保留并可区分**：标题相同/相似但 `content_hash` 不同的记录一律各自保留；可另记 `relation='same_title_diff_content'` 提示人工区分。**不凭标题相似直接丢弃或合并**——标题相似既可能是转载，也可能是更正公告、同题不同内容（如每日同名日报、同名更正稿），首期不做该判定。
5. **明确不做**：标题 SimHash / MinHash 近重复丢弃、正文语义并簇、"保留最早+最权威"的 canonical 合并——这些会误合并金额、对象或披露状态变化的公告（审计 A11），且属全市场规模才需要的能力，延后（审计 §4）。语义聚类如引入，只允许作为**附加标签**（报道热度统计用），不允许作为丢弃或合并依据。

**阈值由用途确定**：取消"滤重比例 ≥40%"一类固定指标（审计 A11）；去重只保证"同一文档不重复计数、不同文档不被误并"，不做数量比例考核。

### 4.4 关键字段证据与正确性口径（A11）

1. **金额、单位、主体、日期**四类关键字段必须携带 `evidence_quote` + `evidence_pos`，能定位到 `news_raw.content` 原文复核；校验器按 quote 回查原文，定位失败判该字段错误。
2. **缺证据输出未知**：原文未写明的字段一律 `"unknown"`（含 summary 内文字），禁止用常识、历史公告或模型知识补全。金额无单位、日期无时点同样输出 unknown 并注明。
3. **口径记录**：金额单位换算、日期口径（披露日/发生日/计划窗口）写入 note；同一事件多次披露数值不一致时，各版本数值并列保留，以修订链确定当前有效值。
4. **"JSON 五要素齐全率 ≠ 正确率"**：五要素齐全只是格式校验（缺字段拒收），不代表内容正确；抽取正确性按 §4.5 度量。不再使用"五要素齐全率"作为质量指标。

### 4.5 评估口径（A11 验收：固定人工标注样本）

评估改为**固定一批人工标注样本**，覆盖三类必含情形：**转载、更正公告、同标题不同内容**；样本一经固定即冻结（记录样本清单与标注版本），跨轮次可比。分别统计：

| 指标 | 定义 | 度量对象 |
|---|---|---|
| **漏检** | 应当识别而未识别：真实的转载/修订关系未被标记；真实事件未被抽取 | news_relation 召回、event 召回 |
| **误合并** | 不应合并而合并/丢弃：转载、更正、同标题不同内容被误判为同一文档而丢弃或覆盖 | news_relation 精确率（误合并条数 / 合并条数） |
| **关键字段准确性** | 金额、单位、主体、日期与标注原文一致（含正确输出 unknown） | key_facts 逐字段准确率 |

- 三类指标分开报告，不合并成单一"正确率"。
- **阈值由用途确定**：例如进入复盘报告的关键数字按字段准确率单独定门槛，事件触发器按漏检/误合并权衡取舍；门槛在使用场景处声明，不由本文统一设定。
- 评估在每次抽取 prompt/模型/去重规则变更后重跑同一固定样本，结果与 `extract_model`、`extract_prompt_version` 一同登记。

### 4.6 提示注入安全边界（A12 新增）

新闻、公告、研报/论文正文（含 PDF 转写文本、评论、社交媒体内容）均为**不可信外部文本**：

1. **指令不触发工具**：外部文本中出现的任何指令（"忽略以上提示""调用 XX 工具""执行命令""读取密钥"等）一律不得触发工具执行、命令运行、文件写入或网络请求；LLM 抽取以**无工具权限**的纯推理模式运行，工具调用只能由确定性代码按白名单发起。
2. **外部文本一律作为数据处理**：进入 prompt 时放在明确分隔的数据区（如 `<document>…</document>`），系统提示声明其为待抽取数据而非指令；输出只允许写入 schema 规定的结构化字段（§3.3），不回流为新指令、任务或 prompt。
3. **执行边界（与审计 A12 一致）**：处理外部文本的任务限制可写目录与网络出口；原始数据（raw 层）只读；密钥与封存测试集不可见；模型输出的"建议操作"只作为文本记录，任何动作须经确定性代码校验与人工确认。
4. **审计与告警**：检出疑似注入语句（指令样式命中规则/分类器）时记录 `dq_issue` 级告警并保留原文证据，样本进入 §4.5 评估集；被拒绝的指令不因重试或换模型而执行。
5. **范围**：本边界覆盖闭环 A（论文→因子）与闭环 B（新闻→事件）的一切外部文本入口，包括本调研 §1.3 中政策类页面抓取的文本。

### 4.7 结构化事件抽取输出格式

见 §3.3 JSON 契约（事件类型 / 涉及标的 / 影响方向 / 置信度 / 证据来源 + 时间窗 + 逐字段原文证据的 key_facts）。落库走 §3.1 `event` + `event_entity` 两表。

---

## 5. 外围市场数据获取

### 5.1 对比表

| 项目 | GitHub 地址 | Stars | License | 维护状态 | 中文能力 | 直接复用 vs 需改造 | 推荐理由 |
|---|---|---|---|---|---|---|---|
| **yfinance** | https://github.com/ranaroussi/yfinance | 25,310 | Apache-2.0 | ✅ 活跃（pushed 2026-09-17） | 文档英文 | **直接复用** | 一个库拿全外围资产：美股指数（^GSPC/^DJI/^IXIC）、港股（^HSI/^HSCE）、黄金期货 GC=F、WTI 原油 CL=F、美债收益率 ^TNX（10Y）、美元指数 DX-Y.NYB、人民币汇率 CNY=X / USDCNY，免费稳定 |
| **akshare** | https://github.com/akfamily/akshare | 22,692 | MIT | ✅ 活跃 | 原生中文 | **直接复用** | 外盘期货/外汇/全球指数接口全（新浪/东财源），与 yfinance 互备；伦敦金、美元兑人民币中间价等国内口径也能拿 |
| **fredapi**（FRED） | https://github.com/mortada/fredapi | 1,658 | Apache-2.0 | ✅ 维护（pushed 2026-01） | 文档英文 | **直接复用** | FRED 官方数据的 Python 客户端；宏观口径最权威：DGS10（10Y 美债）、DTWEXBGS（美元指数）、DEXCHUS（人民币汇率）、DCOILWTICO（WTI）、GOLDAMGBD228NLBM（伦敦金定盘价）等，注册免费 API key |
| **investpy** | https://github.com/alvarobartt/investpy | 1,856 | MIT | ⚠️ 低频维护（pushed 2026-04，244 个 open issues） | 文档英文 | 不推荐主力 | Investing.com 抓取，历史上多次被反爬打断、接口随页面变动而失效；数据 yfinance/akshare 均已覆盖 |
| efinance | https://github.com/Micro-sheep/efinance | 4,067 | MIT | ✅ 维护 | 中文 | 直接复用（补充） | 港股/商品补充源 |

### 5.2 外围资产 → 数据源映射（建议落库字段）

| 资产 | 主源 | 备源 | 常用标的/序列 |
|---|---|---|---|
| 美股主要指数 | yfinance | akshare | ^GSPC、^DJI、^IXIC、^RUT、VIX |
| 港股 | yfinance | akshare | ^HSI、^HSCE、00700.HK |
| 黄金 | yfinance GC=F | akshare 现货/上金所 + FRED GOLDAMGBD228NLBM | COMEX 金、Au9999 |
| 原油 | yfinance CL=F、BZ=F | FRED DCOILWTICO/DCOILBRENTEU | WTI、Brent |
| 美债收益率 | FRED DGS2/DGS10/DGS30 | yfinance ^TNX、^FVX、^TYX | 10Y 为主，2s10s 利差自算 |
| 美元指数 | FRED DTWEXBGS | yfinance DX-Y.NYB | 两口径并存 |
| 人民币汇率 | FRED DEXCHUS（中间价口径） | yfinance CNY=X、USDCNY=X + akshare 中间价/在岸 | USDCNY、CFETS 指数 |
| 全球主要指数 | yfinance | akshare | ^GSPC、^STOXX50E、^GDAXI、^N225、^KS11、^BSESN |

---

## 6. 本地小模型 vs 云端 API（Mac mini / Apple Silicon 场景）

### 6.1 两种路线对比

| 维度 | 本地小模型（Mac mini 统一内存） | 云端 API |
|---|---|---|
| 硬件/费用 | 一次性投入；16GB 可跑 7B–8B Q4，32–64GB 可跑 14B–32B（Ollama / llama.cpp / MLX） | 按 token 计费；国产大模型 API（DeepSeek、Qwen、GLM 等）价格很低 |
| 分类/情绪打分 | ✅ 强项：微调 BERT（finance-sentiment-zh-base / FinGPT 配方）CPU/ANE 每秒数百条，成本≈0 | 质量过剩，不划算 |
| 严格 schema 事件抽取 | ⚠️ 7B–14B 可用（JSON 约束 + few-shot），长公告、复杂金额推理易出错 | ✅ 强项：复杂事件、跨段金额、影响推断明显更好 |
| 摘要/晨报 | 一般（14B 以上才顺） | ✅ 质量最好 |
| 数据合规 | ✅ 公告/持仓不出本机 | ⚠️ 投研数据发第三方需自担合规 |
| 时延/吞吐 | 7B Q4 约 20–40 tok/s（M 系列），夜间批量可行 | 并发高、秒级返回 |
| 断网可用 | ✅ | ❌ |

### 6.2 成本/质量估算（示例性推算，待核实）

> 注（v2）：审计 v2 已取消自行设定的 AI 月费与每日处理量上限——额度按用户已有服务配置，保留输入/输出/重试/并发用量记录，具体限额不由本文设定。下述规模假设（每日去重后 800 条新闻 + 50 篇公告全文）来自全市场口径的示例推算，**首期只处理持仓、自选、重大公告，实际量级更小**；数字未经实测，标注**待核实**，实施时按实际用量重新测算。

- **全量情绪**（本地 BERT）：800 条 × ~1ms/条 → 分钟级，电费可忽略。
- **事件抽取走云端**（每条 input ≈ 800 tok、output ≈ 300 tok；公告按 8k tok 抽样重要篇目）：约 1.5–2.5M tok/日。按国产旗舰 API 量级（输入约 ¥1–4/百万 tok、输出约 ¥4–16/百万 tok）估算**每月约 ¥30–150（待核实）**；质量上 14B 本地模型与其差距主要在长公告与多实体归因。
- **事件抽取走本地**（14B-Instruct Q4，Mac mini 夜间批处理）：800 条 × 300 output tok ≈ 24 万 tok，2–4 小时可完成，**边际成本≈0**；但对长公告需先做"分段抽取再合并"，建议只对短快讯走本地。

### 6.3 推荐混合架构（性价比最优）

1. **全量、低延迟、便宜**：本地小模型 —— 情绪分类（finance-sentiment-zh-base）、事件类型分类（微调 BERT，FinGPT/FinNLP 数据）、精确去重（来源文档 ID + 内容哈希，全本地）。
2. **高价值、复杂推理**：云端 API —— 仅处理（a）持仓股与 watchlist 相关事件、（b）本地分类器标为高重要性的事件、（c）长公告全文的结构化抽取与晨报生成；并强制 JSON schema + 字段级原文证据（§3.3），缺证据输出 unknown，失败降级到本地模型。发送到云端的文本同样适用 §4.6：模型输出不得触发工具执行。
3. **持续改进**：云端抽取出的高质量样本回流微调本地 7B/14B（LoRA，FinGPT 配方），逐步把云端调用量压下来；`event_feedback` 表评估方向判断命中率，抽取质量迭代以 §4.5 固定样本的漏检/误合并/关键字段准确性为准。

---

## 7. 结论速览（选型清单）

| 环节 | 选定项目 |
|---|---|
| 范围（审计 §4） | **首期只处理持仓、自选、重大公告**；全市场轮询、语义去重、向量库及 RAG 按实际检索需求延后 |
| 新闻/快讯 | akshare（财联社/东财/新浪一库打通，按持仓/自选过滤）+ cailianpress-unified 补财联社细节 |
| 公告 | akshare 巨潮列表 + use_cninfo（PDF→Markdown 全文）+ tushare pro 结构化事件 |
| 去重（A11） | **首期：`(source, source_doc_id)` 唯一 + 归一化内容哈希**；转载/更正/同标题不同内容保留修订链与关系（§4.3）；simhash/datasketch/text2vec 语义去重延后 |
| 正确性口径（A11） | 金额/单位/主体/日期逐字段原文证据，缺证据输出 unknown（§3.3/§4.4）；评估用固定人工标注样本，统计漏检/误合并/关键字段准确性，阈值由用途确定（§4.5） |
| 出报（A10） | 数据就绪驱动：初版标注缺项 → 补齐 → 次日晨间修订；旧数据不伪装为当天数据；补跑幂等不重复统计（§4.2） |
| 安全边界（A12） | 外部文本一律作为数据，正文指令不得触发工具执行（§4.6） |
| 分类/情绪 | finance-sentiment-zh-base + ProsusAI/finbert（英文）+ FinGPT 微调配方 |
| 事件抽取 | LLM + JSON schema（OneKE/DISC-FinLLM prompt 思路），DeepKE 小模型兜底 |
| RAG/证据检索 | 延后：按实际检索需求引入 LlamaIndex 或 RAGFlow（审计 §4）；首期证据走字段级原文定位 |
| 事件库 | DuckDB/PostgreSQL + §3.1 schema（事件↔实体↔持仓三段关联 + news_relation 关系登记） |
| 外围市场 | yfinance + akshare + fredapi，investpy 不作主力 |
| 架构范式 | TradingAgents / FinRobot 的"新闻分析 Agent→多空辩论→决策"流程借鉴 |

---

## 8. 来源 URL 汇总

**GitHub API / 仓库核实**（stars、License、pushed_at 于 2026-09-22 取数）：
- akshare — https://github.com/akfamily/akshare （API：https://api.github.com/repos/akfamily/akshare ）
- FinGPT — https://github.com/AI4Finance-Foundation/FinGPT
- ProsusAI/finbert — https://github.com/ProsusAI/finbert
- yfinance — https://github.com/ranaroussi/yfinance
- investpy — https://github.com/alvarobartt/investpy
- tushare — https://github.com/waditu/tushare ；tushare 公告/披露文档 — https://tushare.pro/document/2?doc_id=460
- FinNLP — https://github.com/AI4Finance-Foundation/FinNLP
- DISC-FinLLM — https://github.com/FudanDISC/DISC-FinLLM
- TradingAgents — https://github.com/TauricResearch/TradingAgents
- FinRobot — https://github.com/AI4Finance-Foundation/FinRobot
- fredapi — https://github.com/mortada/fredapi
- efinance — https://github.com/Micro-sheep/efinance
- use_cninfo — https://github.com/rollysys/use_cninfo
- CnInfoReports — https://github.com/tr1s7an/CnInfoReports
- CnInfoHedgeCrawler — https://github.com/Interstellar1217/CNInfoHedgeCrawler
- CninfoDistributedSpider — https://github.com/flicck/CninfoDistributedSpider
- cninfo-search-python — https://github.com/Poncirus/cninfo-search-python
- cailianpress-unified — https://github.com/caimao9539/cailianpress-unified
- FinSpider — https://github.com/jiaweif3ng/FinSpider
- finance_spider_data_analysis — https://github.com/Anton-Mu/finance_spider_data_analysis
- wallstreetcnScrapy — https://github.com/jianzhichun/wallstreetcnScrapy
- simhash（经 shields.io 核实 stars/License）— https://github.com/1e0ng/simhash
- datasketch — https://github.com/ekzhu/datasketch
- text2vec — https://github.com/shibing624/text2vec
- DeepKE — https://github.com/zjunlp/DeepKE
- OneKE — https://github.com/zjunlp/OneKE
- RAGFlow — https://github.com/infiniflow/ragflow
- LlamaIndex — https://github.com/run-llama/llama_index

**HuggingFace / 文档**：
- bardsai/finance-sentiment-zh-base — https://huggingface.co/bardsai/finance-sentiment-zh-base
- yiyanghkust/finbert-pretrain — https://huggingface.co/yiyanghkust/finbert-pretrain
- FinGPT 中文情绪微调数据集（示例）— https://huggingface.co/datasets/oliverwang15/fingpt_chatglm2_sentiment_instruction_lora_ft_dataset
- 腾讯 AI Lab 中文词向量 — https://ai.tencent.com/ailab/nlp/en/embedding.html
- akshare 新闻/公告接口源码（stock_info.py 等）— https://raw.githubusercontent.com/akfamily/akshare/main/akshare/stock_feature/stock_info.py
- FRED 数据（美债/汇率/美元指数/原油/黄金）— https://fred.stlouisfed.org

> 注：GitHub stars 为 2026-09-22 快照；simhash/datasketch/text2vec/DeepKE/RAGFlow/LlamaIndex/OneKE/cninfo-search-python 的 stars 与 License 经 shields.io 元数据接口核实（约数以 "≈" 标注）。HuggingFace 模型页在本次调研网络环境下无法直接抓取，模型信息经 web 搜索结果核实，license 与下载量以 HF 页面为准。
>
> v2 修订（2026-09-23）未重核上述外部事实；采用前对项目当前维护状态、接口行为与模型页面**待核实**。新增的去重/出报/安全/评估口径为设计约定，非外部事实。
