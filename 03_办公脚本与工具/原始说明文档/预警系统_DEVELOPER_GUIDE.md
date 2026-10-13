# 开发者对接指南

> **文档目的**：给下一位接手开发的人，详细说明已实现的功能设计、数据库 schema、代码逻辑、部署方式，以及未实现/待改进的部分。
> **适用对象**：需要修改/扩展/维护本系统的后端开发者。

---

## 目录

1. [系统架构总览](#1-系统架构总览)
2. [数据库设计与 Schema](#2-数据库设计与-schema)
3. [核心数据流：从抓取到预警](#3-核心数据流从抓取到预警)
4. [信号检测引擎详解（alert.py）](#4-信号检测引擎详解alertpy)
5. [7 个爬虫实现详解](#5-7-个爬虫实现详解)
6. [编排器详解（orchestrator.py）](#6-编排器详解orchestratorpy)
7. [Web API 与前端设计](#7-web-api-与前端设计)
8. [LLM 解读模块](#8-llm-解读模块)
9. [部署架构](#9-部署架构)
10. [设计但未实现的功能](#10-设计但未实现的功能)
11. [已知问题与技术债](#11-已知问题与技术债)
12. [扩展指南](#12-扩展指南)

---

## 1. 系统架构总览

### 1.1 分层架构

```
┌─────────────────────────────────────────────────────────┐
│  前端看板（dashboard.html）                              │
│  统计卡片 + 4维筛选 + 信号表格 + LLM简报按钮             │
├─────────────────────────────────────────────────────────┤
│  Web API 层（web_api.py）                                │
│  HTTPServer + 7 个 REST 端点 + 5维服务端过滤            │
├─────────────────────────────────────────────────────────┤
│  信号检测层（alert.py）                                 │
│  detect_new_records() + detect_launch_warning()         │
│  + 5个未启用检测函数                                     │
├─────────────────────────────────────────────────────────┤
│  编排层（orchestrator.py）                               │
│  7个 crawl_* 函数 + crawl_all_extended()                 │
│  职责：调爬虫→落库→调信号检测→返回结果                   │
├─────────────────────────────────────────────────────────┤
│  爬虫层（7 个 *_crawler.py）                             │
│  FCC / EPREL / ENERGY STAR / ANATEL / IC-ISED / TÜV / VDE│
├─────────────────────────────────────────────────────────┤
│  数据层（compintel_db.py + db.py）                       │
│  CompIntelDB 类：add_record / add_alert / stats 等       │
│  Database 类：SQLite 连接管理 / schema 迁移              │
├─────────────────────────────────────────────────────────┤
│  基础设施层（fetchers.py + infra.py + config.py）         │
│  HttpFetcher（代理/限速/重试） + RateLimiter + DiskCache │
├─────────────────────────────────────────────────────────┤
│  SQLite 数据库（overseas.db）                           │
│  compintel_record / compintel_alert / crawl_run 等 10 表 │
└─────────────────────────────────────────────────────────┘
```

### 1.2 文件依赖关系

```
compintel_cli.py（CLI 入口）
  ├── overseas.config            → 目录/路径配置
  ├── overseas.db.Database        → SQLite 连接 + init_schema()
  │     ├── overseas.config
  │     └── overseas.models       → Product/PriceSnapshot 等数据类
  ├── overseas.compintel.web_api  → Web 服务
  │     ├── overseas.db
  │     └── overseas.compintel.compintel_db.CompIntelDB
  └── overseas.compintel.orchestrator → 编排器
        ├── overseas.db
        ├── overseas.fetchers.HttpFetcher → HTTP 抓取
        │     └── overseas.infra → RateLimiter/DiskCache/BlockDetector
        ├── overseas.compintel.compintel_db
        ├── overseas.compintel.fcc_crawler
        ├── overseas.compintel.eprel_crawler
        ├── overseas.compintel.energystar_crawler
        ├── overseas.compintel.anatel_crawler
        ├── overseas.compintel.ic_ised_crawler
        ├── overseas.compintel.cert_crawler
        ├── overseas.compintel.alert → 信号检测
        └── overseas.compintel.llm_analyze → LLM 解读（可选）
```

### 1.3 技术选型

| 选型 | 理由 |
|------|------|
| **SQLite** | 零部署、单文件、25MB 数据量足够 |
| **Python 标准库 http.server** | Web 服务零依赖，不引入 Flask/FastAPI |
| **urllib.request**（非 requests） | HTTP 抓取零依赖，通过 OVERSEAS_PROXY 走代理 |
| **argparse**（非 Click/Typer） | CLI 零依赖 |
| **openai 库**（可选） | LLM 解读，不通时降级为规则式摘要 |
| **时序快照设计** | 每次 INSERT 新行而非 UPDATE，保留变化历史 |

---

## 2. 数据库设计与 Schema

### 2.1 数据库文件

- 路径：`code/data/overseas.db`（25MB，已内置 851 条记录）
- 类型：SQLite，启用 WAL 模式
- schema 管理：`overseas/schema.sql`（建表 SQL）+ `db.py` 的 `init_schema()` 方法（幂等执行 + 迁移）

### 2.2 表关系图（ER）

```
crawl_run（抓取批次）
  │ 1:N
  ├──→ compintel_record（认证记录）
  │      │ 1:N
  │      └──→ compintel_doc（认证文档）
  │
  ├──→ compintel_alert（预警信号）
  │
  └──→ crawl_error（错误日志）

compintel_crawl_state（抓取断点状态）  ← 独立，无外键
compintel_skip（跳过黑名单）            ← 独立，无外键

product（产品表，海外采集框架）  ← detect_launch_warning 查此表判断在售
  └──→ product_spec（产品规格）
```

### 2.3 核心表 DDL

#### `compintel_record` — 认证记录表（851 行）

```sql
CREATE TABLE IF NOT EXISTS compintel_record (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id          INTEGER REFERENCES crawl_run(id) ON DELETE SET NULL,
    source          TEXT NOT NULL CHECK (source IN
                    ('fcc','eprel','energystar','ul','tuv','ic_ised','anatel')),
    source_id       TEXT NOT NULL,            -- FCC ID / EPREL 注册号 / UL 文件号
    product_group   TEXT,                     -- EPREL 产品类 / ES 品类
    applicant       TEXT,                      -- 申请人/供应商(公司名)
    applicant_code  TEXT,                      -- FCC Grantee Code
    brand            TEXT,                     -- 标准化后的品牌名
    model            TEXT,                     -- 型号标识
    description      TEXT,                      -- 产品描述
    auth_date       TEXT,                      -- 认证/授权/注册日期
    market_start    TEXT,                      -- 上市开始日（EPREL 专用）
    market_end      TEXT,                      -- 上市结束日（EPREL 专用）
    frequency       TEXT,                      -- FCC 频段
    energy_class    TEXT,                      -- 能效等级（A~G）
    extra_json      TEXT,                      -- 其余技术规格 JSON
    confidentiality TEXT,                      -- 保密标记（FCC 专用）
    captured_at     TEXT NOT NULL,
    UNIQUE (source, source_id, captured_at)    -- 同一记录每次抓取一行快照
);
-- 索引：
--   idx_ci_record_source_applicant(source, applicant)
--   idx_ci_record_source_id(source, source_id)
--   idx_ci_record_brand_model(brand, model)
--   idx_ci_record_auth_date(source, auth_date)
```

**设计要点**：
- **时序快照**：UNIQUE 含 `captured_at`，同一 source_id 每次抓取 INSERT 新行，保留字段变化历史
- **source CHECK 约束**：限制 7 个数据源，v9 迁移时扩展（原 v8 只有 fcc/eprel）
- **brand 字段**：存储标准化后的品牌名（如 "TCL" 而非 "TCL Electronics"），方便按竞品聚合

#### `compintel_alert` — 预警信号表（980 行）

```sql
CREATE TABLE IF NOT EXISTS compintel_alert (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id          INTEGER REFERENCES crawl_run(id) ON DELETE SET NULL,
    signal_type     TEXT NOT NULL,  -- new_fcc_id/new_cert/eprel_new_reg/launch_warning 等
    source          TEXT NOT NULL CHECK (source IN
                    ('fcc','eprel','energystar','ul','tuv','ic_ised','anatel')),
    applicant       TEXT,
    brand            TEXT,
    model            TEXT,
    source_id       TEXT,
    level           TEXT NOT NULL DEFAULT 'watch',  -- info/watch/alert/critical
    message         TEXT NOT NULL,            -- 人读消息
    details_json    TEXT,                      -- 结构化补充 JSON
    detected_at     TEXT NOT NULL,
    UNIQUE (signal_type, source, source_id, detected_at)
);
-- 索引：
--   idx_ci_alert_type(signal_type, detected_at)
--   idx_ci_alert_applicant(applicant, detected_at)
```

**details_json 结构**（launch_warning 类型示例）：
```json
{
  "product_type": "electronic_display",  // 匹配的电视/显示器关键词
  "lead_days": 219,                       // 认证日期距今天数（提前量）
  "on_market": false,                     // 是否已在售
  "auth_date": "2026-03-04",              // 认证日期
  "description": "...",                   // 产品描述
  "source": "eprel"                       // 数据源
}
```

#### `compintel_crawl_state` — 抓取断点状态（85 行）

```sql
CREATE TABLE IF NOT EXISTS compintel_crawl_state (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    source          TEXT NOT NULL,            -- fcc/eprel/energystar/...
    state_key       TEXT NOT NULL,            -- fcc=授权码 / eprel=scan:from-to
    last_crawled_at TEXT NOT NULL,            -- 上次抓取完成时间(UTC)
    last_record_date TEXT,                    -- 该单位最新记录日期
    pages_crawled   INTEGER DEFAULT 0,
    records_seen    INTEGER DEFAULT 0,
    records_new     INTEGER DEFAULT 0,
    meta_json       TEXT,
    UNIQUE (source, state_key)
);
```

**用途**：增量抓取的断点续抓依据。FCC 每个授权码一行，记录上次抓到哪页、已知哪些 source_id，下次跳过已知的。

#### `crawl_run` — 抓取批次（206 行）

```sql
CREATE TABLE IF NOT EXISTS crawl_run (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    scenario        TEXT NOT NULL,    -- fcc_crawl / eprel_crawl / energystar_crawl 等
    site_code       TEXT,
    params          TEXT,             -- JSON 参数
    status          TEXT NOT NULL,    -- running/completed/failed
    started_at      TEXT NOT NULL,
    finished_at     TEXT,
    ok_count        INTEGER,
    fail_count      INTEGER,
    message         TEXT
);
```

### 2.4 防重复机制

**问题**：UNIQUE 约束是 `(source, source_id, captured_at)`，含 `captured_at`，所以同一 source_id 在不同时间可以多次 INSERT，导致重复。

**修复**（`compintel_db.py` 第54-61行）：`add_record()` 在 INSERT 前先查 `SELECT id FROM compintel_record WHERE source=? AND source_id=? ORDER BY captured_at DESC LIMIT 1`，如果已存在且不是强制更新（`_force_update=False`），直接返回已有 id，不再 INSERT。

```python
def add_record(self, run_id, source, source_id, **fields) -> int:
    if not fields.get("_force_update"):
        existing = self.conn.execute(
            "SELECT id FROM compintel_record WHERE source=? AND source_id=? "
            "ORDER BY captured_at DESC LIMIT 1",
            (source, source_id)).fetchone()
        if existing:
            return int(existing["id"])  # 已存在，跳过
    # ... INSERT 新记录
```

---

## 3. 核心数据流：从抓取到预警

### 3.1 完整数据流（以 ENERGY STAR 为例）

```
用户执行: compintel_cli.py compintel-energystar
    │
    ├── 1. CLI 入口: cmd_compintel_energystar(args)
    │     ├── _parse_since(args, default_days=365)  → since="2025-10-27"
    │     ├── Database(args.db) + db.init_schema()
    │     └── 调 orchestrator.crawl_energystar(db, brands, since)
    │
    ├── 2. 编排器: crawl_energystar(db, brands, since)
    │     ├── HttpFetcher(proxy=OVERSEAS_PROXY)
    │     ├── EnergyStarCrawler(fetcher)
    │     ├── cidb = CompIntelDB(db)
    │     ├── run_id = cidb.start_run("energystar_crawl", {...})
    │     ├── records = crawler.fetch_all_brands(brands, since, categories)
    │     │     └── 对每个 brand: GET data.energystar.gov/resource/8wj2-sec8.json
    │     │         ?$where=upper(brand_name)='SAMSUNG' AND date_certified > '2025-10-27'
    │     │         → 返回 JSON 记录列表
    │     │
    │     ├── for r in records:
    │     │     ├── is_new = not cidb.seen_source_id("energystar", r["source_id"])
    │     │     ├── r["is_new"] = is_new      ← 关键：标记 is_new 供信号检测用
    │     │     ├── sid = r.pop("source_id")
    │     │     └── cidb.add_record(run_id, "energystar", sid, **r)
    │     │         └── INSERT 前查重，已存在则跳过
    │     │
    │     ├── sigs = alert.detect_new_records(cidb, run_id, "energystar", records)
    │     │     └── 对每条 is_new=True 的记录生成 new_cert 信号
    │     │
    │     ├── all_signals += alert.detect_launch_warning(cidb, run_id, records, "energystar")
    │     │     └── 对每条 is_new=True 的记录：
    │     │         1. _match_product_type(desc, group, "energystar", model)
    │     │            → 匹配 TV/display 关键词，排除 remote/phone 等
    │     │         2. _is_on_market(cidb, "energystar", sid, brand, model)
    │     │            → 查 product 表是否已有此型号
    │     │         3. if on_market and source != "eprel": skip（已上市不预警）
    │     │         4. 计算 lead_days = (now - auth_date).days
    │     │         5. cidb.add_alert(run_id, "launch_warning", "energystar", ...)
    │     │
    │     ├── cidb.finish_run(run_id, "completed", ok, fail)
    │     └── return {"records": ok, "fail": fail, "signals": all_signals}
    │
    └── 3. CLI 输出: "ENERGY STAR: ok=38 fail=0 信号=55 (since=2025-10-27)"
```

### 3.2 is_new 标志的关键作用

信号检测依赖 `is_new` 标志判断"本次新出现的记录"：
- **入库前**：`is_new = not cidb.seen_source_id(source, sid)` 查库判断
- **入库后**：`seen_source_id` 恒为 True，所以必须在 `add_record` 之前计算
- **传递方式**：`r["is_new"] = is_new`，记录 dict 带入 `detect_new_records` 和 `detect_launch_warning`

如果 `is_new` 为 None（未设），`detect_new_records` 会回退到查库（第104-105行），但此时已入库，恒为 False。

---

## 4. 信号检测引擎详解（alert.py）

### 4.1 信号类型体系

```python
SIGNAL_TYPES = [
    "new_fcc_id",          # ✅ 启用：FCC 新认证 ID
    "new_cert",            # ✅ 启用：其他源新认证（ES/ANATEL/ISED/TÜV/VDE）
    "eprel_new_reg",       # ✅ 启用：EPREL 新能效注册
    "launch_warning",      # ✅ 启用：上市预警（核心信号）
    "confidential_expired",# ⚠️ 未启用：FCC 保密期到期
    "energy_class_jump",   # ⚠️ 未启用：能效等级跳变
    "eprel_delisted",      # ⚠️ 未启用：产品下架
    "supplier_change",     # ⚠️ 未启用：供应商变更
    "field_change",        # ⚠️ 未启用：字段变化
]
```

### 4.2 `detect_new_records()` — 新记录检测

**位置**：`alert.py` 第84行

**逻辑**：
1. 根据 source 确定信号类型：fcc→`new_fcc_id`，eprel→`eprel_new_reg`，其他→`new_cert`
2. 遍历 records，取 `is_new` 标志（入库前预设）
3. 对 `is_new=True` 的记录，调 `cidb.add_alert()` 生成信号
4. 信号 message 格式：`"FCC 新认证: {fcc_id} | {applicant} | {description}"`
5. details_json 含：`lead_days`、`auth_date`、`description`、`source`

### 4.3 `detect_launch_warning()` — 上市预警

**位置**：`alert.py` 第397行

**这是系统最核心的函数**，在 `new_*` 信号基础上叠加业务过滤，只留真正"新产品即将上市"的信号。

**四步过滤逻辑**：

```
对每条 is_new=True 的记录：
  │
  ├── 步骤1: 产品类型过滤
  │   ptype = _match_product_type(description, product_group, source, model)
  │   if not ptype: skip  ← 非电视/显示器，跳过
  │
  ├── 步骤2: 在售状态比对
  │   on_market = _is_on_market(cidb, source, sid, brand, model)
  │   if on_market and source != "eprel": skip  ← 已上市不预警
  │   （EPREL 例外：近1年新品即使已上市也生成预警）
  │
  ├── 步骤3: 提前量计算
  │   date_field = "market_start" if source=="eprel" else "auth_date"
  │   ref_date = _parse_date(r[date_field])
  │   lead_days = (now - ref_date).days
  │
  └── 步骤4: 产出 launch_warning 信号
      cidb.add_alert(run_id, "launch_warning", source, level="alert",
          brand=brand, model=model, source_id=sid,
          message=msg, details={product_type, lead_days, on_market, ...})
```

### 4.4 `_match_product_type()` — 产品类型匹配

**位置**：`alert.py` 第328行

**策略**：
- **EPREL**：`product_group` 含 "display" 直接通过（EPREL 只抓显示器组）
- **其他源**：合并 `description + model` 小写后做关键词匹配
  1. 先排除：含 `remote`/`charger`/`phone`/`speaker` 等排除词 → return None
  2. 再匹配：含 `tv`/`display`/`monitor`/`oled`/`4k`/`projector` 等 → return 匹配关键词

```python
TV_DISPLAY_KEYWORDS = [
    "tv", "television", "display", "monitor", "tuner",
    "oled", "qled", "led tv", "lcd tv", "smart tv",
    "uhd", "4k", "8k", "set-top", "stb", "media player",
    "rear seat entertainment", "digital signage", "projector",
]

EXCLUDE_KEYWORDS = [
    "remote", "remote control", "charger", "headset", "speaker",
    "phone", "handset", "wearable", "watch", "glasses",
    "lock", "thermostat", "appliance", "oven", "cooktop",
    "baby monitor", "module", "transceiver",
]
```

### 4.5 `_is_on_market()` — 在售判断

**位置**：`alert.py` 第368行

**策略**：
- **EPREL**：查 `market_start` 字段，如果已过当前日期 → 已上市
- **其他源**：查 `product` 表是否有 `model = source_id` 的记录（极少见）
  - FCC 认证 ≠ 在售，不能靠"品牌在 product 表有记录"判定（会误判所有新认证为已售）
  - 只有 FCC ID 本身出现在 product 表的 model 字段才算在售

### 4.6 未启用的检测函数

以下 5 个函数已在 `alert.py` 中完整实现，但 `orchestrator.py` 未调用：

| 函数 | 行号 | 功能 | 启用方法 |
|------|------|------|----------|
| `detect_confidential_expired()` | ~100 | FCC 保密期到期检测 | 在 crawl_fcc 中调用 |
| `detect_supplier_change()` | ~140 | 供应商变更检测 | 在各 crawl_* 后调用 |
| `detect_field_changes()` | ~175 | 字段变化检测 | 在各 crawl_* 后调用 |
| `detect_energy_class_jump()` | ~210 | 能效等级跳变检测 | 在 crawl_eprel 后调用 |
| `detect_delisted()` | ~250 | 退市检测 | 在 crawl_eprel 后调用 |

**启用方法**：在 orchestrator 的对应 `crawl_*` 函数中，在 `detect_launch_warning` 后追加调用，例如：
```python
all_signals += alert.detect_energy_class_jump(cidb, run_id, records, "eprel")
```

---

## 5. 7 个爬虫实现详解

### 5.1 FCC 爬虫（fcc_crawler.py，291 行）

**抓取流程**：
1. GET `apps.fcc.gov/oetcf/eas/reports/GraSearch.cfm` 获取搜索表单
2. POST 搜索（按申请人名或 Grantee Code）
3. 解析搜索结果页 HTML（`parse_search_results`）
4. 翻页遍历（`parse_grantee_page` 获取总页数）
5. 对每条记录可选抓详情页（`fetch_detail`）

**核心数据结构**：
```python
@dataclass
class FccRecord:
    fcc_id: str           # 完整 FCC ID（如 "A3LUN65DU7200"）
    grantee_code: str     # 授权码（如 "A3L"）
    product_code: str     # 产品码（如 "UN65DU7200"）
    company: str          # 公司名
    description: str      # 产品描述
    latest_filing: str    # 最新认证日期
    frequencies: str      # 频段
```

### 5.2 EPREL 爬虫（eprel_crawler.py，221 行）

**API 端点**：`eprel.ec.europa.eu/api/products/{group}`

**三种抓取模式**：
1. **list_recent()**：列表 API，返回最新 25 条 + 总数
2. **search_by_supplier()**（⭐ 推荐）：用 `?supplierOrTrademark={supplier}` 过滤器搜索
3. **scan_range()**：按注册号区间遍历（慢，用于补数据）

**API 限制**：`offset` 翻页不工作（不同 offset 返回相同 25 条），只能用 `search_by_supplier` 按品牌搜索。

**供应商搜索关键词**（`orchestrator.py` 第485行）：
```python
EPREL_SUPPLIER_SEARCH = {
    "Samsung": ["Samsung"],
    "LG": ["LG", "HELGI", "elgato"],      # 子品牌
    "Hisense": ["Hisense"],
    "TCL": ["TCL", "THOMSON"],            # 子品牌
    "Sony": ["Sony", "SONY"],
    "Panasonic": ["Panasonic"],
}
```

### 5.3 ENERGY STAR 爬虫（energystar_crawler.py，170 行）

**API**：Socrata SODA `data.energystar.gov/resource/8wj2-sec8.json`

**关键查询**（第115行）：
```python
where_parts = [f"upper(brand_name)='{brand.upper()}'"]  # 大小写不敏感
if since:
    where_parts.append(f"date_certified > '{since}'")
```

**品类过滤**：`TV_DISPLAY_CATEGORIES` 只取 Televisions/Displays 等。

### 5.4 ANATEL 爬虫（anatel_crawler.py，199 行）

**抓取方式**：全量 ZIP-CSV 下载（53MB，19万行），本地按制造商名过滤竞品。

```python
# 下载 ZIP
zip_data = fetcher.get(ANATEL_ZIP_URL).content
# 解压 CSV
with zipfile.ZipFile(io.BytesIO(zip_data)) as zf:
    csv_file = zf.read(zf.namelist()[0])
# 解析 CSV，按品牌过滤 + since 日期过滤
for row in csv.reader(io.StringIO(csv_file)):
    if _match_brand(row["Fabricante"]):
        if not since or _parse_date_br(row["DataAprovacao"]) >= since:
            records.append(...)
```

### 5.5 IC/ISED 爬虫（ic_ised_crawler.py，383 行）

**抓取方式**：Spring WebFlow 表单搜索，需维护 `execution` token + Cookie。

**关键修复**：
- 删除 `licenceDateFrom` API 参数（导致 HTTP 500）
- 改为本地按 `auth_date >= since` 过滤
- `max_pages_per_company` 2→10，`max_companies_per_brand` 1→2
- 品牌间 15s 延迟避免 ISED 返回 500

### 5.6 TÜV + VDE/CB 爬虫（cert_crawler.py，223 行）

**两个爬虫合一**：
- `TuvCertCrawler`：Certipedia 搜索页批量提取（只取证书号）
- `VdeCertCrawler`：VDE Solr API 搜索

**已知限制**：TÜV model/auth_date 全空（搜索页无结构化摘要）；VDE auth_date 空。

---

## 6. 编排器详解（orchestrator.py）

### 6.1 编排器职责

每个 `crawl_*` 函数遵循统一模式：
1. 创建 `HttpFetcher`（走代理）
2. 创建对应 `Crawler` 实例
3. 创建 `CompIntelDB` + `start_run()` 记录批次
4. 调 `crawler.fetch_*()` 获取记录
5. 对每条记录：计算 `is_new` → `add_record()` 入库
6. 调 `alert.detect_new_records()` 生成基础信号
7. 调 `alert.detect_launch_warning()` 生成上市预警
8. `finish_run()` 结束批次
9. 返回 `{"records": ok, "fail": fail, "signals": all_signals}`

### 6.2 品牌标准化（第828行）

```python
_BRAND_NORMALIZE = {
    "tcl electronics": "TCL",
    "tcl technology": "TCL",
    "tcl overseas": "TCL",
    # ... 更多变体
    "samsung electronics": "Samsung",
    "samsung": "Samsung",
    "lg electronics": "LG",
    "lg": "LG",
    # ...
}

def _normalize_brand(applicant: str) -> str:
    low = (applicant or "").lower().strip()
    for key, val in _BRAND_NORMALIZE.items():
        if low.startswith(key):  # 从长到短匹配
            return val
    return applicant  # 未匹配返回原名
```

**EPREL 特殊处理**（`_eprel_to_db_dict` 第885行）：supplier_name 可能是子品牌（如 THOMSON），organisation_name 含集团名（如 TCL Operations）。两级匹配：先用 supplier_name，未匹配再用 organisation_name。

### 6.3 `crawl_all_extended()` — 一键全量更新（第779行）

```python
def crawl_all_extended(db, brands=None, sources=None, since="") -> dict:
    sources = sources or ["energystar", "anatel", "ic_ised", "certs", "eprel"]
    results = {}
    total_signals = 0

    if "energystar" in sources:
        r = crawl_energystar(db, brands, since)
        results["energystar"] = r
        total_signals += len(r.get("signals", []))
    # ... anatel / ic_ised / certs / eprel

    results["signals_total"] = total_signals
    return results
```

**注意**：`sources` 参数中的 "certs" 对应 TÜV+VDE（非 "tuv_ul"）。

---

## 7. Web API 与前端设计

### 7.1 Web API（web_api.py，220 行）

**技术**：Python 标准库 `http.server.HTTPServer + BaseHTTPRequestHandler`，无框架。

**路由表**：

| 路由 | 方法 | 参数 | 功能 |
|------|------|------|------|
| `/` | GET | — | 返回 dashboard.html |
| `/api/stats` | GET | — | 库统计（记录数/信号数/按类型/按源分组） |
| `/api/alerts` | GET | type,source,brand,days,since,limit | 信号列表（5维服务端过滤） |
| `/api/records` | GET | source,applicant,brand,limit | 最新状态记录 |
| `/api/history` | GET | source,source_id | 某 source_id 历史快照 |
| `/api/run` | GET | applicants,eprel_brands,details | 触发 FCC+EPREL 抓取（后台线程） |
| `/api/analyze` | GET | mode,type,brand,limit | LLM 解读预警 |

**`/api/alerts` 5维过滤逻辑**（第91-115行）：
```python
since = qs.get("since", [""])[0]
stype = qs.get("type", [""])[0]
source = qs.get("source", [""])[0]
brand = qs.get("brand", [""])[0]
days = qs.get("days", [""])[0]
limit = int(qs.get("limit", ["5000"])[0])

# days → since 转换
if days and not since:
    since = (datetime.utcnow() - timedelta(days=int(days))).isoformat()[:19]

alerts = self.cidb.alerts_since(since)
if stype:
    alerts = [a for a in alerts if a["signal_type"] == stype]
if source:
    alerts = [a for a in alerts if (a.get("source") or "") == source]
if brand:
    bl = brand.lower()
    alerts = [a for a in alerts
              if (a.get("brand") or "").lower() == bl
              or (a.get("applicant") or "").lower().find(bl) >= 0]
alerts = alerts[:limit]  # 服务端过滤后再截断
```

### 7.2 前端看板（dashboard.html，347 行）

**单文件 SPA**，无框架，纯 HTML + CSS + JavaScript。

**核心函数**：
- `loadStats()`：GET `/api/stats`，更新统计卡片 + 下拉计数
- `loadAlerts()`：GET `/api/alerts?...`，渲染信号表格
- `triggerCrawl()`：GET `/api/run`，触发抓取
- `analyzeTrend(brand)`：GET `/api/analyze?mode=trend&brand=...`

**4 个筛选下拉**：
- `filterType`：信号类型（全部/上市预警/FCC新认证/新认证/EPREL新注册）
- `filterSource`：数据源（全部/7个源）
- `filterTime`：时间范围（全部/近1月/近半年/近1年/近2年）
- `filterApplicant`：竞品品牌（全部/6个品牌）

---

## 8. LLM 解读模块（llm_analyze.py）

### 8.1 模型配置

```python
MODEL_CONFIGS = {
    "deepseek-v4-flash": {
        "base_url": "https://inner-apisix.hisense.com/higpt-new/v1",
        "api_key": "YiVcZbGaxXCEL49qL1KEiITOzANgVVzq",
        "user_key": "5z93eqr6995k8f8iikarjxghic2felpr",
    },
}
```

**关键**：聚好看接口是海信内网，不走 `OVERSEAS_PROXY`。调用时临时清空 `HTTP_PROXY/HTTPS_PROXY`。

### 8.2 三个解读函数

| 函数 | 功能 | 提示词策略 |
|------|------|------------|
| `analyze_cert_signal(alert, cidb)` | 解读单条认证信号 | 根据 source 填充 SOURCE_META（名称/国家/认证类型）|
| `analyze_launch_warning(alert)` | 解读单条上市预警 | 强调提前量 + 产品类型 + 在售状态 |
| `analyze_competitor_trend(alerts, brand, cidb)` | 品牌趋势简报 | 按品牌聚合信号，分析产品线方向 |

### 8.3 降级策略

LLM 不通时（网络错误/内网不可达），自动降级为规则式摘要：
```python
except Exception as e:
    return f"（LLM 暂不可用）{alert['brand']} {alert['source']} 新认证: {alert['message']}"
```

---

## 9. 部署架构

### 9.1 部署目录

```
预警相关抓取功能/
├── code/
│   ├── compintel_cli.py     ← CLI 入口（py -3.12 compintel_cli.py ...）
│   ├── db_stats.py          ← 数据库统计工具
│   ├── alert_details.py     ← 预警详情工具
│   ├── crawl_log.py         ← 批次日志工具
│   ├── dedup_records.py     ← 去重工具
│   ├── data/overseas.db     ← SQLite 数据库（已内置 851 条记录）
│   └── overseas/            ← Python 包
│       ├── db.py / fetchers.py / infra.py / config.py / models.py / schema.sql
│       └── compintel/       ← 竞品情报模块（12 个文件）
```

### 9.2 启动流程

```powershell
# 1. 环境变量
$env:PYTHONIOENCODING = "utf-8"
$env:OVERSEAS_PROXY = "http://127.0.0.1:7877"

# 2. 进入目录
cd "...\预警相关抓取功能\code"

# 3. 启动看板（数据库已内置，无需额外准备）
py -3.12 compintel_cli.py compintel-web --port 8088

# 4. 访问 http://127.0.0.1:8088
```

### 9.3 生产环境建议

| 组件 | 当前 | 建议 |
|------|------|------|
| Web 服务 | 标准库 HTTPServer（单线程） | 配 systemd/supervisor 守护 + nginx 反代 |
| 进程管理 | 手动启动 | 用 systemd / Windows Task Scheduler |
| 数据库备份 | 手动 Copy-Item | 定时任务每日备份 |
| 定时抓取 | 手动 CLI | cron / Task Scheduler 每日执行 `compintel-run-extended` |
| 日志 | stdout | 重定向到文件或 journalctl |

---

## 10. 设计但未实现的功能

### 10.1 已写代码但未启用的信号检测（alert.py）

| 函数 | 行号 | 信号类型 | 启用方法 |
|------|------|----------|----------|
| `detect_confidential_expired()` | ~100 | `confidential_expired` | 在 `crawl_fcc` 后调用 |
| `detect_supplier_change()` | ~140 | `supplier_change` | 在各 `crawl_*` 后调用 |
| `detect_field_changes()` | ~175 | `field_change` | 在各 `crawl_*` 后调用 |
| `detect_energy_class_jump()` | ~210 | `energy_class_jump` | 在 `crawl_eprel` 后调用 |
| `detect_delisted()` | ~250 | `eprel_delisted` | 在 `crawl_eprel` 后调用 |

**启用示例**（在 orchestrator.py 的 crawl_eprel_supplier 函数中）：
```python
all_signals += alert.detect_new_records(cidb, run_id, "eprel", all_records)
all_signals += alert.detect_launch_warning(cidb, run_id, all_records, "eprel")
# 新增以下调用：
all_signals += alert.detect_energy_class_jump(cidb, run_id, all_records, "eprel")
all_signals += alert.detect_delisted(cidb, run_id, all_records)
```

### 10.2 完全未实现的功能

| 功能 | 优先级 | 说明 |
|------|--------|------|
| 定时调度 | P0 | 无定时任务，需手动运行 CLI 或配 cron/Task Scheduler |
| 邮件告警 | P0 | 新预警无法自动通知，需加 SMTP 发送 |
| 信号详情弹窗 | P0 | 前端点击信号行无法展开完整详情 |
| 信号趋势图 | P1 | 无时间趋势可视化，需加 Chart.js |
| 跨源关联信号 | P1 | 同一型号多源出现未做关联匹配 |
| 置信度评分 | P1 | 信号无优先级/置信度评分 |
| 增量抓取 | P1 | 每次全量抓取近1年，`compintel_crawl_state` 表已建但未用于增量 |
| 导出 Excel/CSV | P2 | 看板无法导出信号列表 |
| Docker 容器化 | P2 | 无 Dockerfile |
| 新增数据源 | P2 | SRRC(中国)/KCs(韩国)/TELEC(日本) 未接入 |

---

## 11. 已知问题与技术债

### 11.1 数据源 API 限制（不可修复）

| 数据源 | 限制 | 影响 | 代码位置 |
|--------|------|------|----------|
| EPREL | offset 翻页不工作 | 只能取最新 25 条/品牌 | `eprel_crawler.py` |
| TÜV | 搜索页无结构化 model/auth_date | 89 条记录字段全空 | `cert_crawler.py` |
| VDE | Solr API 不返回认证日期 | 136 条记录 auth_date 空 | `cert_crawler.py` |
| IC/ISED | API 不稳定，频繁 500 | 近1年仅 2 条 | `ic_ised_crawler.py` |
| ANATEL | 全量 ZIP CSV，电视占比极低 | 223 条仅 1 条 launch_warning | `anatel_crawler.py` |

### 11.2 代码问题

| 问题 | 位置 | 影响 | 修复建议 |
|------|------|------|----------|
| UNIQUE 含 captured_at | schema.sql 第371行 | 同 source_id 可重复入库 | `add_record` 已加查重修复 |
| 5 个信号函数未启用 | orchestrator.py | 信号类型只有 4 种 | 在 crawl_* 中追加调用 |
| TÜV launch_warning 0 | alert.py `_match_product_type` | TÜV description 无 TV 关键词 | 无法修复（数据源限制） |
| Web 单线程 | web_api.py | 高并发时阻塞 | 生产环境用 gunicorn/waitress |
| LLM 依赖内网 | llm_analyze.py | 外部部署不可用 | 替换 MODEL_CONFIGS |
| `--since` 默认陷阱 | compintel_cli.py | 默认近365天，非空 | 设 1900-01-01 抓全量 |

### 11.3 数据覆盖缺口

| 品牌 × 数据源 | 缺失原因 |
|---------------|----------|
| Samsung/Hisense/TCL × EPREL | 近1年在 EPREL 无新电子显示器注册（API 确认） |
| Hisense/TCL/Sony × IC-ISED | ISED API 不稳定，公司号抓取失败 |
| IC/ISED × launch_warning | 近1年仅 2 条非电视类记录 |
| TÜV × launch_warning | description 无电视关键词可匹配 |

---

## 12. 扩展指南

### 12.1 新增数据源

以新增 SRRC（中国无线电认证）为例：

1. **创建爬虫** `srrc_crawler.py`：
```python
class SrrcCrawler:
    def search_by_company(self, company: str) -> list[dict]:
        # 实现 HTTP 请求 + HTML/JSON 解析
        # 返回统一格式的记录 dict: {source_id, brand, model, description, auth_date, ...}
```

2. **在 orchestrator.py 添加编排函数**：
```python
def crawl_srrc(db, brands=None, since="") -> dict:
    fetcher = HttpFetcher(proxy=os.environ.get("OVERSEAS_PROXY", ""))
    crawler = SrrcCrawler(fetcher=fetcher)
    cidb = CompIntelDB(db)
    run_id = cidb.start_run("srrc_crawl", {"brands": brands, "since": since})
    # ... 抓取 → 落库 → 信号检测
    all_signals += alert.detect_new_records(cidb, run_id, "srrc", records)
    all_signals += alert.detect_launch_warning(cidb, run_id, records, "srrc")
    return {"records": ok, "fail": fail, "signals": all_signals}
```

3. **更新 schema.sql 的 CHECK 约束**：
```sql
-- compintel_record 和 compintel_alert 的 source CHECK 加 'srrc'
source TEXT NOT NULL CHECK (source IN
    ('fcc','eprel','energystar','ul','tuv','ic_ised','anatel','srrc'))
```

4. **在 compintel_cli.py 添加命令**：
```python
s = sub.add_parser("compintel-srrc", help="SRRC 中国无线电认证")
s.add_argument("--brands", default="")
s.add_argument("--since", default="")
s.set_defaults(func=cmd_compintel_srrc)
```

5. **在 alert.py 的 SOURCE_PRODUCT_MATCH 添加策略**：
```python
SOURCE_PRODUCT_MATCH["srrc"] = "keywords"  # 用 description 关键词匹配
```

6. **在 dashboard.html 的数据源下拉添加选项**：
```javascript
const labels = {..., 'srrc': 'SRRC(中国)'};
```

### 12.2 新增信号类型

以新增"跨源关联信号"为例：

1. 在 `alert.py` 添加检测函数：
```python
def detect_cross_source_match(cidb, run_id, records, source) -> list[dict]:
    """检测同一型号在多个数据源同时出现。"""
    for r in records:
        if not r.get("is_new"):
            continue
        model = r.get("model", "")
        if not model:
            continue
        # 查其他数据源是否有相同 model
        other_sources = cidb.conn.execute(
            "SELECT DISTINCT source FROM compintel_record "
            "WHERE model=? AND source!=?", (model, source)).fetchall()
        if len(other_sources) >= 1:
            cidb.add_alert(run_id, "cross_source_match", source, ...)
```

2. 在 `SIGNAL_TYPES` 列表添加 `"cross_source_match"`
3. 在 orchestrator 的 crawl_* 函数中调用

### 12.3 启用定时抓取

**Windows Task Scheduler**：
```powershell
# 创建定时任务，每天 09:00 执行
$action = New-ScheduledTaskAction -Execute "py" -Argument "-3.12 compintel_cli.py compintel-run-extended" -WorkingDirectory "C:\...\code"
$trigger = New-ScheduledTaskTrigger -Daily -At 9am
Register-ScheduledTask -TaskName "CompIntelDaily" -Action $action -Trigger $trigger
```

**Linux cron**：
```bash
# 每天凌晨 2 点执行
0 2 * * * cd /path/to/code && py -3.12 compintel_cli.py compintel-run-extended >> /var/log/compintel.log 2>&1
```

### 12.4 启用未使用的信号检测函数

在 `orchestrator.py` 的对应 `crawl_*` 函数中，在 `detect_launch_warning` 后追加：

```python
# crawl_fcc 中追加：
all_signals += alert.detect_confidential_expired(cidb, run_id, records, "fcc")

# crawl_eprel_supplier 中追加：
all_signals += alert.detect_energy_class_jump(cidb, run_id, all_records, "eprel")
all_signals += alert.detect_delisted(cidb, run_id, all_records)

# 所有 crawl_* 中追加（变化检测）：
all_signals += alert.detect_supplier_change(cidb, run_id, records, source)
all_signals += alert.detect_field_changes(cidb, run_id, records, source)
```

---

> **最后提醒**：
> 1. 所有爬虫的 `--since` 默认近 1 年（365天），设为 `1900-01-01` 可抓全量历史
> 2. 数据库 `code/data/overseas.db` 已内置 851 条记录，开箱即用
> 3. LLM 功能需要海信内网，外部部署需替换 `llm_analyze.py` 的 `MODEL_CONFIGS`
> 4. 详细的执行命令见 `COMMANDS.md`，依赖说明见 `DEPENDENCIES.md`
> 5. 扩展新数据源时，务必更新 `schema.sql` 的 `source CHECK` 约束
