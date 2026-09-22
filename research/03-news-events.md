# 03 · 新闻、公告、事件分析与 AI 金融分析 — 开源项目与数据源调研

> 目标环境：Mac mini（Apple Silicon）上的个人 A 股量化研究系统
> 调研原则：**尽量复用现成开源项目**，仅在必要处做轻量改造
> 核实时间：2026-09-22（stars / License / 维护状态均通过 GitHub API、shields.io、web 搜索逐项核实，来源 URL 见各表末列与文末汇总）

**术语约定**：直接复用 = pip/conda 安装后按官方 API 即可用于生产流水线；需改造 = 可借鉴代码/接口设计，需自行适配数据源、反爬或 schema；仅参考 = 停更或质量不足，只取思路。

---

## 1. 财经新闻 / 公告 / 政策数据源

### 1.1 总体推荐

**首选策略：以 akshare 为统一新闻/公告入口**（一个库覆盖财联社电报、东方财富快讯与个股新闻、新浪财经快讯、巨潮资讯公告），巨潮公告全文下载补充 `rollysys/use_cninfo`（PDF→Markdown，直接对接 LLM），tushare pro 作公告/结构化事件数据的第二来源。独立爬虫仓库普遍停更且无 License，只作参考。

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

- **财联社（cls.cn）**：电报流是 A 股事件最快的公开源之一。优先 `ak.stock_info_global_cls()`（轮询电报列表）；需要"加红/热度/文章详情"时用 cailianpress-unified 的接口封装。财联社快讯短，适合做"事件触发器"，正文细节再回源抓文章。
- **东方财富**：`ak.stock_info_global_em()`（7×24 快讯）、`ak.stock_news_em(symbol=...)`（个股新闻，含时间/正文/来源）；股吧情绪用 FinSpider 思路自研（注意合规与频率限制）。
- **新浪财经**：`ak.stock_info_global_sina()` 全球财经快讯，覆盖面广、含宏观与海外，适合外围事件传导分析。
- **巨潮资讯（cninfo，法定披露源）**：
  1) 列表/检索：走 cninfo 官方 `http://www.cninfo.com.cn/new/hisAnnouncement/query`（akshare 公告接口已封装，返回公告标题、类型、PDF 链接）；
  2) 全文：use_cninfo 下载 PDF 并转 Markdown 归档；
  3) 分类：先按巨潮 category（年报/半年报、业绩预告、增减持、重组、处罚问询等）硬分类，LLM 只做细粒度事件抽取。
- **tushare pro 公告**：`公告数据` 系列接口（见 tushare.pro 文档 doc_id=460 一带的披露/公告类接口）提供公告索引与部分结构化字段，适合校验 akshare 抓取的完整性；重大事件（业绩预告、解禁、增减持）有专门结构化接口，**优先吃结构化接口，PDF 全文仅对持仓股与高影响事件下载**。
- **政策类**：无单一成熟开源库。组合方案：akshare 宏观/政策类资讯接口 + 新浪/财联社"政策"频道快讯 + 证监会/央行网站 RSS 级自研薄爬虫（页面简单，requests+BeautifulSoup 即可）。

---

## 2. 新闻 NLP 分析（去重 / 分类 / 摘要 / 事件抽取 / 情绪与影响方向）

### 2.1 对比表

| 项目 | GitHub / HF 地址 | Stars | License | 维护状态 | 中文能力 | 直接复用 vs 需改造 | 推荐理由 |
|---|---|---|---|---|---|---|---|
| **FinGPT** | https://github.com/AI4Finance-Foundation/FinGPT | 21,272 | MIT | ✅ 活跃（pushed 2026-09-14） | 中英双语（含中文情绪指令数据集） | **直接复用思路与数据，模型需自训** | 金融 LLM 事实标准开源项目；`FinGPT-Sentiment_Analysis` 给出"LLM/LoRA 做金融情绪"的完整成熟配方，且有公开中文情绪微调数据集（如 fingpt_chatglm2_sentiment_instruction_lora_ft_dataset），在 Mac 上可复现微调 |
| **ProsusAI/finbert** | https://github.com/ProsusAI/finbert | 2,237 | Apache-2.0 | ⚠️ pushed 2022-09（不再更新，但模型稳定可用） | ❌ 英文 | 直接复用（英文文本） | 最广泛使用的金融情绪 BERT（pos/neg/neutral），适合处理英文快讯（美债/美联储/美股新闻），与中文模型互补 |
| **finbert-pretrain**（HKUST） | https://huggingface.co/yiyanghkust/finbert-pretrain | —（HF 模型） | 以 HF 页为准 | 语料冻结 | ❌ 英文 | 需改造（作为微调底座） | 金融领域继续预训练 BERT，做英文金融分类微调底座 |
| **finance-sentiment-zh-base** | https://huggingface.co/bardsai/finance-sentiment-zh-base | —（HF 模型，下载量高） | 以 HF 页为准 | 可用 | ✅ 中文金融情感 | **直接复用** | 现成中文金融情感三分类（积极/中性/消极）Transformer 模型，轻量、CPU 可跑，全量新闻情绪打分首选 |
| **text2vec** | https://github.com/shibing624/text2vec | ≈5k | Apache-2.0 | ✅ 持续维护 | ✅ 中文为一等公民 | **直接复用** | 开箱即用的中文句向量/语义相似度（Word2Vec、Sentence-BERT、CoSENT），是向量去重与新闻聚类的核心件 |
| **simhash** | https://github.com/1e0ng/simhash | ≈1k | MIT | 稳定（小而成熟） | 支持中文（字符 n-gram） | **直接复用** | SimHash 指纹 + 汉明距离，标题级近重复去重的标准实现 |
| **datasketch** | https://github.com/ekzhu/datasketch | ≈3k | MIT | ✅ 维护 | 语言无关 | **直接复用** | MinHash/LSH，海量快讯做大规模近似去重与聚类候选对召回 |
| **DeepKE** | https://github.com/zjunlp/DeepKE | ≈4.5k | MIT | ✅ 维护（EMNLP 2022 工具箱） | ✅ 中文场景丰富 | 需改造（schema 适配） | 浙大知识图谱抽取工具箱：命名实体（NER）、关系（RE）、事件抽取（EE）全链路 pipeline + 中文预训练权重，传统小模型方案里最成熟 |
| **OneKE** | https://github.com/zjunlp/OneKE | 194 | MIT | ✅ 维护 | ✅ 中文 | 需改造 | 基于 LLM 的 schema 约束信息抽取（含事件抽取），可直接产出结构化事件 JSON |
| **DISC-FinLLM** | https://github.com/FudanDISC/DISC-FinLLM | 895 | Apache-2.0 | ❌ pushed 2023-11 停更 | ✅ 原生中文金融 | 参考复用（prompt/数据） | 复旦中文金融 LLM：金融信息抽取、Chain-of-Retrieval RAG 的公开实现与数据配方，抽取/RAG prompt 设计可直接抄 |
| **TradingAgents** | https://github.com/TauricResearch/TradingAgents | 108,032 | Apache-2.0 | ✅ 非常活跃（pushed 2026-09-18） | 中文可用（取决于底层 LLM） | 需改造（数据源换 A 股） | 最火的多智能体 LLM 交易框架，其 News/Social Media Analyst + 情绪聚合 + 多空辩论的"事件→交易观点"架构是现成范式，值得整段借鉴 |
| **FinRobot** | https://github.com/AI4Finance-Foundation/FinRobot | 8,049 | Apache-2.0 | ✅ 活跃（pushed 2026-09-11） | 部分中文 | 需改造 | AI4Finance 的金融 Agent 平台，含公司情绪分析、新闻研究报告生成等现成 Agent，可拆其 prompt 与流程 |
| **FinNLP** | https://github.com/AI4Finance-Foundation/FinNLP | 1,487 | MIT | ⚠️ pushed 2024-07 半停更 | 中英 | 参考复用（数据集） | 金融 NLP 数据管道与数据集汇总（新闻情绪、NER 数据等），省去自标注成本 |
| **RAGFlow** | https://github.com/infiniflow/ragflow | ≈91k | Apache-2.0 | ✅ 非常活跃 | ✅ 中文文档/切分优化 | **直接复用**（或用 llama_index 自建） | 成熟 RAG 引擎：深度文档解析（PDF 年报/公告）、切分、检索、引用溯源，公告问答/证据检索直接套 |
| **LlamaIndex** | https://github.com/run-llama/llama_index | ≈52k | MIT | ✅ 非常活跃 | 中文可用 | **直接复用** | 轻量嵌入式 RAG：新闻/公告事件库 + 向量索引 + 结构化抽取的组合最灵活，Mac mini 单机首选 |
| 词向量：Tencent AI Lab 中文词向量 | https://ai.tencent.com/ailab/nlp/en/embedding.html（下载渠道已不稳定，可搜 HF 镜像 qundao/chinese-word-embedding） | — | 研究可用条款 | 冻结 | ✅ 中文 | 直接复用（传统特征） | 800 万中文词向量；若用深度模型可不必上，保留作轻量基线/热词扩展 |

### 2.2 各处理环节的推荐方案

| 环节 | 推荐做法（复用优先） |
|---|---|
| 去重 | 见 §4.2 三层去重：精确 hash → simhash（标题）→ text2vec 向量 + datasketch LSH |
| 分类 | 两段式：巨潮 category / 来源频道做规则硬分类；新闻用微调 BERT（FinGPT 数据 + 中文底座）做 15 类事件类型分类（见 §3.2 枚举） |
| 摘要 | 短快讯免摘要；长公告用本地 Qwen-7B/14B 做"5W+金额"结构化摘要；每日晨报聚合用云端 LLM |
| 事件抽取 | 优先 LLM + JSON schema 约束输出（OneKE 思路 / DISC-FinLLM prompt 配方）；批量低成本场景用 DeepKE 小模型 pipeline 兜底 |
| 情绪/影响方向 | 全量：finance-sentiment-zh-base（中文）+ ProsusAI/finbert（英文快讯）打分；重点事件：LLM 输出"影响方向 + 置信度 + 作用期限"；FinGPT Sentiment 配方做增量微调 |

---

## 3. 事件数据库设计（事件 ↔ 个股 / 行业 / 持仓关联）

### 3.1 推荐 schema（SQLite/DuckDB/PostgreSQL 均可，Mac mini 建议 DuckDB + Parquet 冷备）

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

-- ③ 向量层（供 RAG 与向量去重；可托管于 sqlite-vec / LanceDB / Qdrant 单机模式）
CREATE TABLE news_embedding (
  news_id BIGINT, model TEXT, vector BLOB, PRIMARY KEY(news_id, model)
);

-- ④ 事件层：LLM 结构化抽取产出
CREATE TABLE event (
  id            BIGINT PRIMARY KEY,
  event_type    TEXT NOT NULL,        -- 见 §3.2 枚举
  sub_type      TEXT,
  occur_time    TIMESTAMP,            -- 事件实际发生时间
  publish_time  TIMESTAMP,
  title         TEXT,
  summary       TEXT,                 -- 5W+金额 结构化摘要
  direction     TEXT,                 -- positive / negative / neutral（对市场总体）
  confidence    REAL,                 -- 0~1
  importance    REAL,                 -- 0~1 影响力预估
  horizon       TEXT,                 -- intraday / days / weeks
  evidence_news_ids JSON,             -- 证据来源 cluster_id 列表（可追溯）
  extract_model TEXT,                 -- 抽取所用模型/版本
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

```json
{
  "event_type": "增减持",
  "sub_type": "高管拟减持",
  "occur_time": "2026-09-22",
  "entities": [
    {"code": "300750.SZ", "name": "宁德时代", "role": "target",
     "direction": "negative", "weight": 0.9}
  ],
  "direction": "negative",
  "horizon": "days",
  "confidence": 0.85,
  "importance": 0.6,
  "summary": "副总经理计划减持不超过 120 万股（约占总股本 0.03%），窗口期 3 个月。",
  "key_facts": {"amount_cny": 3.2e8, "share_pct": 0.03, "duration": "3个月"},
  "evidence": {"news_cluster_id": 10231, "quote": "……公告编号 2026-045……"}
}
```

字段即事件契约：**事件类型、涉及标的、影响方向、置信度、证据来源** 五要素齐全，`evidence.quote` 保证可追溯，`key_facts` 支撑后续量化回测。

---

## 4. 每日信息流水线建议

### 4.1 抓取频率

| 数据 | 频率 | 说明 |
|---|---|---|
| 财联社电报 | 盘前盘中 1–5 分钟轮询；盘后 15–30 分钟 | 事件触发器，增量拉取按 id 去重 |
| 东财/新浪 7×24 快讯 + 个股新闻 | 5–15 分钟 | 交易时段加密 |
| 巨潮公告列表 | 交易日 15:00–23:00 每 30 分钟全量对账；08:00 补漏 | 公告多在盘后与深夜披露，凌晨补一次 |
| 公告 PDF 全文 | 随列表增量，**仅持仓股 + 重要类目 + 高影响事件**下载 | 控磁盘与解析成本 |
| tushare 结构化事件（业绩预告/解禁/增减持） | 每日 1 次（盘后） | 作事实校验层 |
| 外围市场（yfinance/akshare） | 每日收盘后 1 次 + 美股收盘后（次日 08:00 前）补一次 | 覆盖美债/美元/黄金/原油/美股/港股 |
| FRED 宏观序列 | 每周 1 次 | 频率低，无需每日 |
| 事件抽取（LLM） | 盘中快讯批量（5–15 分钟一批），公告当晚批处理 | 与抓取解耦，走队列 |
| 晨报生成 | 每交易日 08:30 | 聚合昨日事件簇 + 持仓影响 |

### 4.2 去重策略（三层漏斗，复用 simhash + datasketch + text2vec）

1. **精确层**：`sha256(title_norm)`（去空白/全半角/标点归一）或 `source+doc_id` 命中即丢弃，成本≈0，滤掉 40–60% 转载。
2. **近似层（标题）**：标题分词/字符 2-gram → SimHash 64bit，汉明距离 ≤3 判为同一事件标题（1e0ng/simhash）。快讯类加 datasketch MinHash-LSH 做批量候选对召回。
3. **语义层（正文）**：text2vec（或 bge-m3）句向量，余弦相似度 ≥0.92 且时间窗 72h 内并簇；同一事件簇保留**最早+最权威**（信源优先级：巨潮 > 财联社 > 东财 > 新浪 > 自媒体）为 canonical，其余进 `news_cluster.members`，后续统计"报道热度 = 簇内条数/信源数"。
4. **事件级合并**：抽取后同 `event_type + entities + 3 日窗` 的事件做第二次归并（防止跨日跟踪报道生成重复事件），合并证据链而非删除。

### 4.3 结构化事件抽取输出格式

见 §3.3 JSON 契约（事件类型 / 涉及标的 / 影响方向 / 置信度 / 证据来源 + 时间窗 + 关键事实）。落库走 §3.1 `event` + `event_entity` 两表。

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

### 6.2 成本/质量估算（按每日去重后 800 条新闻 + 50 篇公告全文）

- **全量情绪**（本地 BERT）：800 条 × ~1ms/条 → 分钟级，电费可忽略。
- **事件抽取走云端**（每条 input ≈ 800 tok、output ≈ 300 tok；公告按 8k tok 抽样重要篇目）：约 1.5–2.5M tok/日。按国产旗舰 API 量级（输入约 ¥1–4/百万 tok、输出约 ¥4–16/百万 tok）估算 **每月约 ¥30–150**，个人完全可承受；质量上 14B 本地模型与其差距主要在长公告与多实体归因。
- **事件抽取走本地**（14B-Instruct Q4，Mac mini 夜间批处理）：800 条 × 300 output tok ≈ 24 万 tok，2–4 小时可完成，**边际成本≈0**；但对长公告需先做"分段抽取再合并"，建议只对短快讯走本地。

### 6.3 推荐混合架构（性价比最优）

1. **全量、低延迟、便宜**：本地小模型 —— 情绪分类（finance-sentiment-zh-base）、事件类型分类（微调 BERT，FinGPT/FinNLP 数据）、去重（simhash+text2vec 全本地）。
2. **高价值、复杂推理**：云端 API —— 仅处理（a）持仓股与 watchlist 相关事件、（b）本地分类器标为高重要性的事件、（c）长公告全文的结构化抽取与晨报生成；并强制 JSON schema + 证据引用，失败降级到本地模型。
3. **持续改进**：云端抽取出的高质量样本回流微调本地 7B/14B（LoRA，FinGPT 配方），逐步把云端调用量压下来；`event_feedback` 表评估方向判断命中率，驱动模型/阈值迭代。

---

## 7. 结论速览（选型清单）

| 环节 | 选定项目 |
|---|---|
| 新闻/快讯 | akshare（财联社/东财/新浪一库打通）+ cailianpress-unified 补财联社细节 |
| 公告 | akshare 巨潮列表 + use_cninfo（PDF→Markdown 全文）+ tushare pro 结构化事件 |
| 去重 | 1e0ng/simhash + ekzhu/datasketch + shibing624/text2vec |
| 分类/情绪 | finance-sentiment-zh-base + ProsusAI/finbert（英文）+ FinGPT 微调配方 |
| 事件抽取 | LLM + JSON schema（OneKE/DISC-FinLLM prompt 思路），DeepKE 小模型兜底 |
| RAG/证据检索 | LlamaIndex（单机轻量）或 RAGFlow（要深度 PDF 解析时） |
| 事件库 | DuckDB/PostgreSQL + §3.1 schema（事件↔实体↔持仓三段关联） |
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
