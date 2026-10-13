# 数据库与表结构设计完整说明

## 1. 数据库基本信息

| 项目 | 值 |
|------|----|
| 文件名 | `overseas_v4_20260929.db` |
| 路径 | `data/overseas_v4_20260929.db` |
| 引擎 | SQLite 3 (WAL模式) |
| 大小 | 135.5 MB (34690页 × 4096字节) |
| 编码 | UTF-8 |
| 表总数 | 30张（含sqlite_sequence） |

## 2. 表清单与四层架构

| # | 表名 | 行数 | 层级 | 中文名 | 说明 |
|---|------|------|------|--------|------|
| 1 | `crawl_error` | 3,472 | 运维监控层 | 爬取错误日志表 | 爬取过程中的错误日志 |
| 2 | `crawl_run` | 842 | 运维监控层 | 爬取运行记录表 | 每次爬取运行的记录 |
| 3 | `dim_product_model` | 6,493 | 维度层 | 型号维度表 | 所有电视型号+海信标记+尺寸 |
| 4 | `dim_product_model_bak` | 6,295 | 其他 | 型号维度备份表 | 备份 |
| 5 | `dim_spec_item_map` | 1,974 | 映射层 | 规格项映射表 | 原始规格项名→标准param_id映射 |
| 6 | `dim_spec_normalization_map` | 9 | 映射层 | 同义归一映射表 | 不同品牌同义术语归一 |
| 7 | `dim_spec_parameter` | 18 | 维度层 | 对标参数定义表 | 18个标准对标参数定义 |
| 8 | `fact_benchmark_comparison` | 4 | 事实层 | 对阵比较结果表 | AI对阵分析结果存储 |
| 9 | `fact_spec_value` | 52,636 | 事实层 | 归一化规格值表 | 每型号每参数的归一化规格值 |
| 10 | `kakaku_task_status` | 807 | 运维监控层 | Kakaku任务状态表 | 价格.com爬取任务状态 |
| 11 | `kakaku_thread_state` | 13 | 运维监控层 | Kakaku线程状态表 | 评测帖追踪状态 |
| 12 | `price_shop_snapshot` | 1,441 | 其他 | price_shop_snapshot |  |
| 13 | `price_snapshot` | 14,971 | 原始爬取层 | 价格快照表 | 每个产品每次抓取的价格记录 |
| 14 | `product` | 14,959 | 原始爬取层 | 零售站产品表 | 爬虫从电商站抓取的产品信息 |
| 15 | `product_spec` | 136,588 | 原始爬取层 | 零售站规格表 | 从电商产品详情页抓取的规格键值对 |
| 16 | `ranking_snapshot` | 0 | 其他 | 排名快照表(预留) | 0行，预留 |
| 17 | `raw_page` | 0 | 其他 | 原始页面表(预留) | 0行，预留 |
| 18 | `retail_task_status` | 8,322 | 运维监控层 | 零售任务状态表 | 零售站爬取任务状态 |
| 19 | `review` | 94,303 | 原始爬取层 | 用户评论表 | 从电商站抓取的用户评价 |
| 20 | `review_summary` | 2,749 | 原始爬取层 | 评论汇总表 | 每产品的评分分布汇总 |
| 21 | `schema_version` | 7 | 其他 | schema_version |  |
| 22 | `selling_point` | 0 | 其他 | 卖点表(预留) | 0行，预留 |
| 23 | `selling_point_status` | 0 | 其他 | 卖点状态表(预留) | 0行，预留 |
| 24 | `site` | 260 | 运维监控层 | 站点配置表 | 爬取站点配置 |
| 25 | `spec_entry_audit` | 92 | 运维监控层 | SPEC入口审计表 | 品牌官网入口页审计 |
| 26 | `spec_row` | 175,218 | 原始爬取层 | 品牌官网规格行表 | 每个系列的规格表，按行存储 |
| 27 | `spec_series` | 1,087 | 原始爬取层 | 品牌官网系列表 | 品牌官网发现的电视系列及型号列表 |
| 28 | `spec_series_status` | 3,123 | 运维监控层 | SPEC系列状态表 | 品牌官网系列抓取状态 |
| 29 | `spec_untranslated` | 2,513 | 运维监控层 | 未翻译规格项表 | 待翻译的日语规格项 |

### 四层架构关系图

```
┌───────────────────────────────────────────────────────────────────────┐
│                        原始爬取层 (8张表)                               │
│  spec_series ──1:N──→ spec_row        product ──1:N──→ product_spec   │
│       │                                    │                    │   │
│       │                                    ├──1:N──→ price_snapshot  │
│       │                                    └──1:N──→ review          │
│                  ↓ 映射                      │         ↓ 映射          │
│         dim_spec_item_map               dim_spec_item_map             │
│         (item_ja → param_id)            (key → param_id)              │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓                                          
┌───────────────────────────────────────────────────────────────────────┐
│                        维度层 (2张表)                                   │
│  dim_product_model (型号维度, our_brand标记海信)                        │
│  dim_spec_parameter (18个标准对标参数)                                   │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓                                          
┌───────────────────────────────────────────────────────────────────────┐
│                        映射层 (2张表)                                   │
│  dim_spec_item_map (1,974条: 原始项名→param_id)                        │
│  dim_spec_normalization_map (9条: 同义术语归一)                        │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓                                          
┌───────────────────────────────────────────────────────────────────────┐
│                        事实层 (2张表)                                   │
│  fact_spec_value (型号×参数=归一化规格值, 含confidence置信度)            │
│  fact_benchmark_comparison (AI对阵比较结果)                             │
└───────────────────────────────────────────────────────────────────────┘
```

## 3. 逐表详细说明

### 3.1 `spec_series` — 品牌官网系列表 (1,087行)

**层级**: 原始爬取层  
**用途**: 品牌官网发现的电视系列及型号列表

**建表语句 (DDL):**
```sql
CREATE TABLE spec_series (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    brand        TEXT NOT NULL,             -- 品牌 code，如 hisense_jp
    brand_name   TEXT,                      -- 展示名
    series       TEXT NOT NULL,             -- 系列，如 U8S
    url          TEXT,
    models       TEXT,                      -- 机型列表 JSON（列顺序）
    first_seen   TEXT NOT NULL,
    last_seen    TEXT NOT NULL,
    UNIQUE (brand, series)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `brand` | TEXT | ✓ |  |  | 品牌+区域标识，如sony_jp/samsung_de/tcl_us |
| 3 | `brand_name` | TEXT |  |  |  | 品牌显示名 |
| 4 | `series` | TEXT | ✓ |  |  | 系列名，如K-XR90M2/Bravia 8 |
| 5 | `url` | TEXT |  |  |  | 官网规格页URL |
| 6 | `models` | TEXT |  |  |  | JSON数组，该系列所有型号列表，如["K-65XR90M2","K-55XR90M2"] |
| 7 | `first_seen` | TEXT | ✓ |  |  | 首次发现时间 |
| 8 | `last_seen` | TEXT | ✓ |  |  | 最后发现时间 |

**索引:**

- `sqlite_autoindex_spec_series_1`: 自动索引

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `brand`: `sony_jp`
- `brand_name`: `Sony BRAVIA（日本）`
- `series`: `K-XR90M2`
- `url`: `https://www.sony.jp/bravia/products/K-XR90M2/spec.html`
- `models`: `["K-115XR90M2", "K-85XR90M2", "K-75XR90M2", "K-65XR90M2"]`
- `first_seen`: `2026-09-07T02:05:14+00:00`
- `last_seen`: `2026-09-28T04:37:58+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `brand`: `sony_jp`
- `brand_name`: `Sony BRAVIA（日本）`
- `series`: `K-XR80M2`
- `url`: `https://www.sony.jp/bravia/products/K-XR80M2/spec.html`
- `models`: `["K-65XR80M2", "K-55XR80M2"]`
- `first_seen`: `2026-09-07T02:05:22+00:00`
- `last_seen`: `2026-09-28T04:38:09+00:00`

</details>

**品牌列表 (39个):**

- `hisense_ca`: 33个系列
- `hisense_jp`: 13个系列
- `hisense_mx`: 23个系列
- `hisense_us`: 35个系列
- `lg_ca`: 32个系列
- `lg_mx`: 51个系列
- `lg_us`: 99个系列
- `panasonic_jp`: 13个系列
- `philips_ca`: 11个系列
- `regza_jp`: 39个系列
- `samsung_bo`: 7个系列
- `samsung_ca`: 66个系列
- `samsung_cl`: 52个系列
- `samsung_co`: 51个系列
- `samsung_de`: 10个系列
- `samsung_ec`: 11个系列
- `samsung_fr`: 9个系列
- `samsung_it`: 11个系列
- `samsung_mx`: 22个系列
- `samsung_pe`: 65个系列
- `samsung_uk`: 32个系列
- `samsung_us`: 66个系列
- `sharp_jp`: 13个系列
- `sony_ca`: 16个系列
- `sony_de`: 19个系列
- `sony_fr`: 20个系列
- `sony_it`: 11个系列
- `sony_jp`: 9个系列
- `sony_uk`: 14个系列
- `sony_us`: 33个系列
- `tcl_bo`: 3个系列
- `tcl_ca`: 16个系列
- `tcl_cl`: 13个系列
- `tcl_de`: 15个系列
- `tcl_fr`: 15个系列
- `tcl_it`: 57个系列
- `tcl_jp`: 41个系列
- `tcl_mx`: 30个系列
- `tcl_us`: 11个系列

---

### 3.2 `spec_row` — 品牌官网规格行表 (175,218行)

**层级**: 原始爬取层  
**用途**: 每个系列的规格表，按行存储

**建表语句 (DDL):**
```sql
CREATE TABLE spec_row (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    series_id     INTEGER NOT NULL REFERENCES spec_series(id) ON DELETE CASCADE,
    category      TEXT,                     -- 区分
    item_ja       TEXT NOT NULL,            -- 项目（日文）
    item_zh       TEXT,                     -- 项目（中文，查词典）
    row_order     INTEGER DEFAULT 0,
    values_json   TEXT,                     -- {机型: 值} JSON
    captured_week TEXT NOT NULL,            -- ISO 周，如 2026-W36
    captured_at   TEXT NOT NULL,
    UNIQUE (series_id, item_ja, category, captured_week)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `series_id` | INTEGER | ✓ |  |  | 关联spec_series.id |
| 3 | `category` | TEXT |  |  |  | 规格分类，如"画面"/"入力・出力"/"ビデオ" |
| 4 | `item_ja` | TEXT | ✓ |  |  | 规格项名（日文/德文/法文等原文），如"画素数"/"HDR対応" |
| 5 | `item_zh` | TEXT |  |  |  | 中文翻译（如有） |
| 6 | `row_order` | INTEGER |  | 0 |  | 行序号，决定规格表展示顺序 |
| 7 | `values_json` | TEXT |  |  |  | JSON对象，型号→规格值映射，如{"65C728":"●","55C728":"●"} |
| 8 | `captured_week` | TEXT | ✓ |  |  | 抓取周，如2026-W37 |
| 9 | `captured_at` | TEXT | ✓ |  |  | 抓取时间 |

**索引:**

- `sqlite_autoindex_spec_row_1`: 自动索引
- `idx_spec_row_series`: CREATE INDEX idx_spec_row_series ON spec_row(series_id, captured_week)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `7775`
- `series_id`: `82`
- `category`: `入力・出力`
- `item_ja`: `CEC対応`
- `item_zh`: `CEC 支持`
- `row_order`: `1`
- `values_json`: `{"75C728": "Ver2.0a/Ver1.4a対応 HDMIポート共通", "65C728": "Ver2.0a/Ver1.4a対応 HDMIポート共通", "55C728": "Ver2.0a/Ver1.4a対応 HDMIポート共通"}`
- `captured_week`: `2026-W37`
- `captured_at`: `2026-09-10T06:19:43+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `7776`
- `series_id`: `82`
- `category`: `入力・出力`
- `item_ja`: `ビデオ入力端子`
- `item_zh`: `视频输入端子`
- `row_order`: `2`
- `values_json`: `{"75C728": "●", "65C728": "●", "55C728": "●"}`
- `captured_week`: `2026-W37`
- `captured_at`: `2026-09-10T06:19:43+00:00`

</details>

**覆盖系列数:** 1071
**不同规格项名数:** 4655
**规格分类数:** 463

---

### 3.3 `product` — 零售站产品表 (14,959行)

**层级**: 原始爬取层  
**用途**: 爬虫从电商站抓取的产品信息

**建表语句 (DDL):**
```sql
CREATE TABLE product (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    site_id     INTEGER NOT NULL REFERENCES site(id),
    sku         TEXT NOT NULL,                 -- ASIN / 站内商品号
    url         TEXT,
    title       TEXT,
    brand       TEXT,
    model       TEXT,                          -- 厂商型号，跨站比价用
    category    TEXT,
    first_seen  TEXT NOT NULL,
    last_seen   TEXT NOT NULL, size TEXT,
    UNIQUE (site_id, sku)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `site_id` | INTEGER | ✓ |  |  | 关联site.id |
| 3 | `sku` | TEXT | ✓ |  |  | 站内SKU，与site_id组合唯一 |
| 4 | `url` | TEXT |  |  |  | 产品详情页URL |
| 5 | `title` | TEXT |  |  |  | 商品标题 |
| 6 | `brand` | TEXT |  |  |  | 品牌（原始） |
| 7 | `model` | TEXT |  |  |  | 型号编码 |
| 8 | `category` | TEXT |  |  |  | 品类 |
| 9 | `first_seen` | TEXT | ✓ |  |  | 首次发现时间 |
| 10 | `last_seen` | TEXT | ✓ |  |  | 最后发现时间 |
| 11 | `size` | TEXT |  |  |  | 屏幕尺寸（原始文本，如55"） |

**索引:**

- `sqlite_autoindex_product_1`: 自动索引
- `idx_product_model`: CREATE INDEX idx_product_model ON product(model)
- `idx_product_brand`: CREATE INDEX idx_product_brand ON product(brand)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `site_id`: `1`
- `sku`: `B0GPHMTMLC`
- `url`: `https://www.amazon.ca/dp/B0GPHMTMLC`
- `title`: `LG 77-Inch G6 OLED evo AI 4K Smart TV (2026)`
- `brand`: `LG`
- `model`: `OLED77G6WUA`
- `category`: `Portable TVs`
- `first_seen`: `2026-09-04T05:22:22+00:00`
- `last_seen`: `2026-09-27T20:52:55+00:00`
- `size`: `77"`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `site_id`: `1`
- `sku`: `B0GQD6F41Q`
- `url`: `https://www.amazon.ca/dp/B0GQD6F41Q`
- `title`: `LG 83-Inch G6 OLED evo AI 4K Smart TV (2026)`
- `brand`: `LG`
- `model`: `OLED83G6WUA`
- `category`: `Portable TVs`
- `first_seen`: `2026-09-04T05:22:32+00:00`
- `last_seen`: `2026-09-27T20:52:35+00:00`
- `size`: `83"`

</details>


---

### 3.4 `price_snapshot` — 价格快照表 (14,971行)

**层级**: 原始爬取层  
**用途**: 每个产品每次抓取的价格记录

**建表语句 (DDL):**
```sql
CREATE TABLE price_snapshot (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id  INTEGER NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    price       REAL,
    list_price  REAL,
    currency    TEXT,
    in_stock    INTEGER,                       -- 1/0/NULL
    raw_text    TEXT,
    captured_at TEXT NOT NULL
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `product_id` | INTEGER | ✓ |  |  | 关联product.id |
| 3 | `price` | REAL |  |  |  | 实际售价（数字） |
| 4 | `list_price` | REAL |  |  |  | 标价/原价 |
| 5 | `currency` | TEXT |  |  |  | 货币代码：JPY/USD/CAD/EUR/GBP/CHF/SEK等21种 |
| 6 | `in_stock` | INTEGER |  |  |  | 是否有货：1=有货, 0=缺货 |
| 7 | `raw_text` | TEXT |  |  |  | 原始价格文本 |
| 8 | `captured_at` | TEXT | ✓ |  |  | 抓取时间 |

**索引:**

- `idx_price_product_time`: CREATE INDEX idx_price_product_time
    ON price_snapshot(product_id, captured_at)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `2`
- `product_id`: `2`
- `price`: `6297.99`
- `list_price`: `NULL`
- `currency`: `CAD`
- `in_stock`: `1`
- `raw_text`: `$6,297.99`
- `captured_at`: `2026-09-04T05:22:32+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `3`
- `product_id`: `3`
- `price`: `366.68`
- `list_price`: `448.0`
- `currency`: `CAD`
- `in_stock`: `1`
- `raw_text`: `$366.68`
- `captured_at`: `2026-09-04T05:22:57+00:00`

</details>

**按货币(currency)分布:**

| 货币 | 行数 | 对应国家 |
|------|------|----------|
| PEN | 2,856 | 秘鲁 |
| COP | 2,804 | 哥伦比亚 |
| USD | 2,579 | 美国 |
| EUR | 1,490 | 欧洲(德/法/意) |
| CAD | 1,170 | 加拿大 |
| GBP | 429 | 英国 |
| CLP | 368 | 智利 |
| JPY | 316 | 日本 |
| GTQ | 212 | 危地马拉 |
| BOB | 177 | 玻利维亚 |
| MXN | 175 | 墨西哥 |
| CHF | 142 | 瑞士 |
| SEK | 117 | 瑞典 |
| PLN | 90 | 波兰 |
| RSD | 67 | 塞尔维亚 |
| TRY | 50 | 土耳其 |
| RON | 30 | 罗马尼亚 |
| NOK | 30 | 挪威 |
| DKK | 30 | 丹麦 |
| BGN | 23 | 保加利亚 |
| BAM | 6 | 波黑 |

---

### 3.5 `product_spec` — 零售站规格表 (136,588行)

**层级**: 原始爬取层  
**用途**: 从电商产品详情页抓取的规格键值对

**建表语句 (DDL):**
```sql
CREATE TABLE product_spec (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id  INTEGER NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    key         TEXT NOT NULL,
    value       TEXT,
    captured_at TEXT NOT NULL,
    UNIQUE (product_id, key)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `product_id` | INTEGER | ✓ |  |  | 关联product.id |
| 3 | `key` | TEXT | ✓ |  |  | 规格项名（英文），如"Product Dimensions"/"Screen Size"/"HDR" |
| 4 | `value` | TEXT |  |  |  | 规格值（英文），如"144.7 x 83.2 x 5.8 cm" |
| 5 | `captured_at` | TEXT | ✓ |  |  | 抓取时间 |

**索引:**

- `sqlite_autoindex_product_spec_1`: 自动索引

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `product_id`: `1`
- `key`: `Product Dimensions`
- `value`: `2.48 x 171.2 x 98.2 cm; 43.09 kg`
- `captured_at`: `2026-09-27T20:52:55+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `product_id`: `1`
- `key`: `Manufacturer`
- `value`: `LG`
- `captured_at`: `2026-09-27T20:52:55+00:00`

</details>

**不同规格键名数:** 2973

**最常见键名TOP10:**

| key | 出现次数 |
|------|------|
| 消費電力｜年耗电量 | 335 |
| Screen Size | 311 |
| Works with AI Speaker | 288 |
| Wi-Fi | 288 |
| User Manual | 288 |
| USB | 288 |
| Speaker Type | 288 |
| Samsung TV Plus | 288 |
| Resolution | 288 |
| Refresh Rate | 288 |

---

### 3.6 `review` — 用户评论表 (94,303行)

**层级**: 原始爬取层  
**用途**: 从电商站抓取的用户评价

**建表语句 (DDL):**
```sql
CREATE TABLE review (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id    INTEGER NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    review_key    TEXT NOT NULL,               -- 站内唯一 id，增量去重
    rating        REAL,
    title         TEXT,
    body          TEXT,
    author        TEXT,
    review_date   TEXT,
    verified      INTEGER,
    helpful_count INTEGER,
    captured_at   TEXT NOT NULL, review_url TEXT, image_urls TEXT, image_count INTEGER DEFAULT 0, video_urls TEXT, has_video INTEGER DEFAULT 0,
    UNIQUE (product_id, review_key)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `product_id` | INTEGER | ✓ |  |  | 关联product.id |
| 3 | `review_key` | TEXT | ✓ |  |  | 评论唯一标识 |
| 4 | `rating` | REAL |  |  |  | 评分(1-5) |
| 5 | `title` | TEXT |  |  |  | 评论标题 |
| 6 | `body` | TEXT |  |  |  | 评论正文 |
| 7 | `author` | TEXT |  |  |  | 作者 |
| 8 | `review_date` | TEXT |  |  |  | 评论日期 |
| 9 | `verified` | INTEGER |  |  |  | 是否认证购买 |
| 10 | `helpful_count` | INTEGER |  |  |  | 有用数 |
| 11 | `captured_at` | TEXT | ✓ |  |  |  |
| 12 | `review_url` | TEXT |  |  |  |  |
| 13 | `image_urls` | TEXT |  |  |  | 图片URL JSON数组 |
| 14 | `image_count` | INTEGER |  | 0 |  |  |
| 15 | `video_urls` | TEXT |  |  |  | 视频URL JSON数组 |
| 16 | `has_video` | INTEGER |  | 0 |  |  |

**索引:**

- `sqlite_autoindex_review_1`: 自动索引
- `idx_review_product`: CREATE INDEX idx_review_product ON review(product_id)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `product_id`: `1`
- `review_key`: `R2FD6FHIS4UT3A`
- `rating`: `5.0`
- `title`: `No regrets… (so far)`
- `body`: `Improved reflection handling, brightness levels, and contrast. If you are in a market for a high end TV I highly recommend getting a G6 model.`
- `author`: `Adelya`
- `review_date`: `Reviewed in Canada on August 5, 2026`
- `verified`: `1`
- `helpful_count`: `NULL`
- `captured_at`: `2026-09-04T05:22:22+00:00`
- `review_url`: `NULL`
- `image_urls`: `NULL`
- `image_count`: `0`
- `video_urls`: `NULL`
- `has_video`: `0`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `product_id`: `1`
- `review_key`: `R2JF4AGGBPPHEX`
- `rating`: `5.0`
- `title`: `Best out there !`
- `body`: `An amazing TV, completely changed my whole viewing experience!!`
- `author`: `Aniket T`
- `review_date`: `Reviewed in Canada on July 13, 2026`
- `verified`: `1`
- `helpful_count`: `NULL`
- `captured_at`: `2026-09-04T05:22:22+00:00`
- `review_url`: `NULL`
- `image_urls`: `NULL`
- `image_count`: `0`
- `video_urls`: `NULL`
- `has_video`: `0`

</details>


---

### 3.7 `review_summary` — 评论汇总表 (2,749行)

**层级**: 原始爬取层  
**用途**: 每产品的评分分布汇总

**建表语句 (DDL):**
```sql
CREATE TABLE review_summary (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id  INTEGER NOT NULL REFERENCES product(id) ON DELETE CASCADE,
    avg_rating  REAL,
    total_count INTEGER,
    star1       INTEGER,
    star2       INTEGER,
    star3       INTEGER,
    star4       INTEGER,
    star5       INTEGER,
    captured_at TEXT NOT NULL
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `product_id` | INTEGER | ✓ |  |  | 关联product.id |
| 3 | `avg_rating` | REAL |  |  |  | 平均评分 |
| 4 | `total_count` | INTEGER |  |  |  | 评论总数 |
| 5 | `star1` | INTEGER |  |  |  | 1星数量 |
| 6 | `star2` | INTEGER |  |  |  | 2星数量 |
| 7 | `star3` | INTEGER |  |  |  | 3星数量 |
| 8 | `star4` | INTEGER |  |  |  | 4星数量 |
| 9 | `star5` | INTEGER |  |  |  | 5星数量 |
| 10 | `captured_at` | TEXT | ✓ |  |  |  |

**索引:**

- `idx_summary_product_time`: CREATE INDEX idx_summary_product_time
    ON review_summary(product_id, captured_at)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `product_id`: `1`
- `avg_rating`: `4.3`
- `total_count`: `15`
- `star1`: `NULL`
- `star2`: `NULL`
- `star3`: `NULL`
- `star4`: `NULL`
- `star5`: `NULL`
- `captured_at`: `2026-09-04T05:22:22+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `product_id`: `2`
- `avg_rating`: `4.3`
- `total_count`: `15`
- `star1`: `NULL`
- `star2`: `NULL`
- `star3`: `NULL`
- `star4`: `NULL`
- `star5`: `NULL`
- `captured_at`: `2026-09-04T05:22:32+00:00`

</details>


---

### 3.8 `dim_product_model` — 型号维度表 (6,493行)

**层级**: 维度层  
**用途**: 所有电视型号+海信标记+尺寸

**建表语句 (DDL):**
```sql
CREATE TABLE dim_product_model (
    model_id         INTEGER PRIMARY KEY AUTOINCREMENT,
    brand            VARCHAR(50)  NOT NULL,    -- 归一化品牌，如 'Hisense'/'Samsung'
    brand_raw        VARCHAR(50),              -- 原始品牌，如 'SAMSUNG'/'samsung_co'
    model_code       VARCHAR(100) NOT NULL,    -- 规范型号，如 '65U8R'
    model_aliases    TEXT,                     -- JSON别名数组，如 ["116U9R","RGB 116UXS"]
    size_inch        INTEGER,                  -- 屏幕尺寸英寸数
    size_raw         VARCHAR(20),              -- 原始尺寸，如 '65"'/'65V型'
    series           VARCHAR(100),             -- 系列名
    region           VARCHAR(20),              -- 区域，如 'JP'/'CA'/'US'
    price_tier       VARCHAR(20),              -- 价格带 入门/中端/高端/旗舰
    our_brand        TINYINT      NOT NULL DEFAULT 0, -- 是否我方(海信) 1=是
    UNIQUE(brand, model_code)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `model_id` | INTEGER |  |  | 🔑 | 自增主键，全局唯一型号ID |
| 2 | `brand` | VARCHAR(50) | ✓ |  |  | 标准化品牌名（Samsung/LG/HIsense/Sony/TCL等），不含区域后缀 |
| 3 | `brand_raw` | VARCHAR(50) |  |  |  | 原始品牌名，含区域标识（如samsung_jp/sony_de/hisense_us） |
| 4 | `model_code` | VARCHAR(100) | ✓ |  |  | 型号编码，如55QNED70BSA/65XR90M2 |
| 5 | `model_aliases` | TEXT |  |  |  | JSON数组，型号别名 |
| 6 | `size_inch` | INTEGER |  |  |  | 屏幕尺寸（整数英寸），如55/65/75/85 |
| 7 | `size_raw` | VARCHAR(20) |  |  |  | 原始尺寸文本 |
| 8 | `series` | VARCHAR(100) |  |  |  | 系列名 |
| 9 | `region` | VARCHAR(20) |  |  |  | 区域代码 |
| 10 | `price_tier` | VARCHAR(20) |  |  |  | 价格档位 |
| 11 | `our_brand` | TINYINT | ✓ | 0 |  | 是否海信：1=海信(我方), 0=竞品。核心字段，决定对阵分析中谁是"我方" |

**索引:**

- `sqlite_autoindex_dim_product_model_1`: 自动索引
- `idx_model_brand`: CREATE INDEX idx_model_brand  ON dim_product_model(brand)
- `idx_model_size`: CREATE INDEX idx_model_size   ON dim_product_model(size_inch)
- `idx_model_tier`: CREATE INDEX idx_model_tier   ON dim_product_model(price_tier)
- `idx_model_our`: CREATE INDEX idx_model_our    ON dim_product_model(our_brand)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `model_id`: `1`
- `brand`: `United`
- `brand_raw`: `United`
- `model_code`: `0GHZ`
- `model_aliases`: `NULL`
- `size_inch`: `40`
- `size_raw`: `40`
- `series`: `NULL`
- `region`: `NULL`
- `price_tier`: `NULL`
- `our_brand`: `0`

</details>

<details>
<summary>行 2</summary>

- `model_id`: `2`
- `brand`: `LG`
- `brand_raw`: `LG`
- `model_code`: `100`
- `model_aliases`: `NULL`
- `size_inch`: `100`
- `size_raw`: `100"`
- `series`: `NULL`
- `region`: `NULL`
- `price_tier`: `NULL`
- `our_brand`: `0`

</details>

**我方/竞品分布:**

| our_brand | 行数 | 含义 |
|-----------|------|------|
| 0 | 5,753 | 竞品 |
| 1 | 740 | 海信(我方) |

**竞品品牌TOP15:**

| 品牌 | 型号数 |
|------|--------|
| Samsung | 1,884 |
| LG | 1,258 |
| TCL | 704 |
| Sony | 332 |
| Philips | 245 |
| REGZA | 149 |
|  | 138 |
| HYUNDAI | 120 |
| CHALLENGER | 80 |
| Sharp | 68 |
| KALLEY | 65 |
| Panasonic | 52 |
| JVC | 42 |
| Xiaomi | 34 |
| 松下 VIERA（日本） | 31 |

**尺寸分布:**

| 尺寸(英寸) | 型号数 |
|-----------|--------|
| 0" | 1 |
| 10" | 25 |
| 11" | 9 |
| 12" | 12 |
| 13" | 3 |
| 14" | 3 |
| 15" | 8 |
| 16" | 4 |
| 17" | 4 |
| 18" | 4 |
| 19" | 3 |
| 20" | 4 |
| 21" | 1 |
| 22" | 5 |
| 23" | 2 |
| 24" | 45 |
| 25" | 1 |
| 26" | 4 |
| 27" | 13 |
| 28" | 9 |
| 29" | 1 |
| 30" | 3 |
| 31" | 1 |
| 32" | 344 |
| 35" | 1 |
| 37" | 1 |
| 38" | 1 |
| 39" | 3 |
| 40" | 147 |
| 42" | 37 |
| 43" | 531 |
| 45" | 1 |
| 48" | 65 |
| 49" | 27 |
| 50" | 555 |
| 51" | 1 |
| 55" | 922 |
| 58" | 74 |
| 60" | 35 |
| 65" | 883 |
| 66" | 2 |
| 68" | 1 |
| 70" | 64 |
| 73" | 1 |
| 75" | 549 |
| 76" | 1 |
| 77" | 89 |
| 78" | 3 |
| 80" | 1 |
| 81" | 1 |
| 82" | 4 |
| 83" | 41 |
| 85" | 337 |
| 86" | 48 |
| 90" | 1 |
| 91" | 1 |
| 92" | 1 |
| 97" | 8 |
| 98" | 103 |
| 99" | 2 |
| 100" | 76 |
| 110" | 3 |
| 115" | 16 |
| 116" | 5 |
| 125" | 1 |
| 136" | 1 |
| 164" | 1 |
| 165" | 1 |
| 200" | 1 |
| 215" | 1 |
| 300" | 1 |
| 400" | 2 |

---

### 3.9 `dim_spec_parameter` — 对标参数定义表 (18行)

**层级**: 维度层  
**用途**: 18个标准对标参数定义

**建表语句 (DDL):**
```sql
CREATE TABLE dim_spec_parameter (
    param_id         INTEGER PRIMARY KEY,
    param_code       VARCHAR(30)  NOT NULL UNIQUE,
    param_name       VARCHAR(50)  NOT NULL,
    param_name_ja    VARCHAR(100),
    param_unit       VARCHAR(20),
    is_numeric       TINYINT      NOT NULL DEFAULT 1,
    higher_is_better TINYINT,
    category         VARCHAR(20)  NOT NULL,
    sort_order       INTEGER      NOT NULL DEFAULT 0,
    radar_dim        VARCHAR(20)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `param_id` | INTEGER |  |  | 🔑 | 参数ID，1-18 |
| 2 | `param_code` | VARCHAR(30) | ✓ |  |  | 参数代码（英文大写），如PEAK_BRIGHTNESS |
| 3 | `param_name` | VARCHAR(50) | ✓ |  |  | 参数中文名，如峰值亮度 |
| 4 | `param_name_ja` | VARCHAR(100) |  |  |  | 参数日文名 |
| 5 | `param_unit` | VARCHAR(20) |  |  |  | 单位，如nits/Hz/kg/寸。无单位为NULL |
| 6 | `is_numeric` | TINYINT | ✓ | 1 |  | 是否数值型参数：1=数值(可比较大小), 0=文本型(仅展示) |
| 7 | `higher_is_better` | TINYINT |  |  |  | 数值越大越好？1=高优(如亮度), 0=低优(如功耗), NULL=不适用 |
| 8 | `category` | VARCHAR(20) | ✓ |  |  | 分类：画质/智能/游戏/连接/设计/音质 |
| 9 | `sort_order` | INTEGER | ✓ | 0 |  | 排序号，决定前端展示顺序 |
| 10 | `radar_dim` | VARCHAR(20) |  |  |  | 雷达图维度归属 |

**索引:**

- `sqlite_autoindex_dim_spec_parameter_1`: 自动索引
- `idx_spec_param_category`: CREATE INDEX idx_spec_param_category ON dim_spec_parameter(category)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `param_id`: `1`
- `param_code`: `PEAK_BRIGHTNESS`
- `param_name`: `峰值亮度`
- `param_name_ja`: `ピーク輝度`
- `param_unit`: `nits`
- `is_numeric`: `1`
- `higher_is_better`: `1`
- `category`: `画质`
- `sort_order`: `1`
- `radar_dim`: `画质`

</details>

<details>
<summary>行 2</summary>

- `param_id`: `2`
- `param_code`: `DIMMING_ZONES`
- `param_name`: `分区控光`
- `param_name_ja`: `ローカルディミング`
- `param_unit`: `zones`
- `is_numeric`: `1`
- `higher_is_better`: `1`
- `category`: `画质`
- `sort_order`: `2`
- `radar_dim`: `画质`

</details>

**18个对标参数完整定义:**

| param_id | param_code | 中文名 | 日文名 | 单位 | 数值型 | 高优 | 分类 | 排序 | 雷达维度 |
|----------|-----------|--------|--------|------|--------|------|------|------|----------|
| 1 | PEAK_BRIGHTNESS | 峰值亮度 | ピーク輝度 | nits | ✓ | ↑ | 画质 | 1 | 画质 |
| 2 | DIMMING_ZONES | 分区控光 | ローカルディミング | zones | ✓ | ↑ | 画质 | 2 | 画质 |
| 3 | REFRESH_RATE | 刷新率 | リフレッシュレート | Hz | ✓ | ↑ | 画质 | 3 | 画质 |
| 4 | CHIP | 画质芯片 | 画質エンジン | — | ✗ | — | 画质 | 4 | 画质 |
| 5 | SYSTEM | 智能系统 | システム/OS | — | ✗ | — | 智能 | 5 | 智能 |
| 6 | GAMING | 游戏特性 | ゲーム機能 | — | ✗ | ↑ | 游戏 | 6 | 游戏 |
| 7 | COLOR_GAMUT | 色域 | 色域/広色域 | % | ✓ | ↑ | 画质 | 7 | 画质 |
| 8 | HDMI | HDMI 2.1 | HDMI 2.1 | 口 | ✗ | ↑ | 连接 | 8 | 连接 |
| 9 | SCREEN_SIZE | 屏幕尺寸 | 画面サイズ | inch | ✓ | ↓ | 设计 | 9 | — |
| 10 | RESOLUTION | 分辨率 | 画素数/解像度 | — | ✗ | ↑ | 画质 | 10 | — |
| 11 | HDR | HDR格式 | HDR対応 | — | ✗ | ↑ | 画质 | 11 | 画质 |
| 12 | CONTRAST | 对比度 | コントラスト | :1 | ✓ | ↑ | 画质 | 12 | — |
| 13 | WIFI | Wi-Fi | Wi-Fi/無線LAN | — | ✗ | ↑ | 连接 | 13 | 连接 |
| 14 | BLUETOOTH | 蓝牙 | Bluetooth | — | ✗ | ↑ | 连接 | 14 | 连接 |
| 15 | DOLBY_ATMOS | 杜比全景声 | Dolby Atmos | — | ✗ | ↑ | 音质 | 15 | 音质 |
| 16 | POWER_CONSUMPTION | 功耗 | 消費電力 | W | ✓ | ↓ | 设计 | 16 | 设计 |
| 17 | WEIGHT | 重量 | 本体質量 | kg | ✓ | ↓ | 设计 | 17 | — |
| 18 | DIMENSIONS | 尺寸 | 外形寸法 | mm | ✗ | ↓ | 设计 | 18 | — |

**雷达图6维度分组:**

| 雷达维度 | 包含参数 |
|----------|----------|
| 智能 | 智能系统 |
| 游戏 | 游戏特性 |
| 画质 | 峰值亮度 + 分区控光 + 刷新率 + 画质芯片 + 色域 + HDR格式 |
| 设计 | 功耗 |
| 连接 | HDMI 2.1 + Wi-Fi + 蓝牙 |
| 音质 | 杜比全景声 |

---

### 3.10 `dim_spec_item_map` — 规格项映射表 (1,974行)

**层级**: 映射层  
**用途**: 原始规格项名→标准param_id映射

**建表语句 (DDL):**
```sql
CREATE TABLE dim_spec_item_map (
    item_map_id      INTEGER PRIMARY KEY AUTOINCREMENT,
    source_table     VARCHAR(20)  NOT NULL,   -- 'spec_row' / 'product_spec'
    source_field     VARCHAR(20)  NOT NULL,   -- 'item_ja' / 'key'
    source_term      VARCHAR(200) NOT NULL,   -- 原始术语，如 'ピーク輝度' / 'Refresh Rate'
    param_id         INTEGER      NOT NULL,   -- → dim_spec_parameter
    match_keyword    VARCHAR(100),            -- 匹配关键词，如 'Refresh'
    extract_regex    VARCHAR(200),            -- 值提取正则，如 '(\d+)\s*Hz'
    match_type       VARCHAR(20)  NOT NULL DEFAULT 'LIKE', -- LIKE / EXACT / REGEX
    status           VARCHAR(20)  NOT NULL DEFAULT 'AUTO', -- AUTO(自动匹配) / CONFIRMED(人工确认) / REJECTED
    brand_scope      VARCHAR(50),            -- 适用品牌范围，NULL=全品牌，'hisense_jp'=仅该品牌
    created_at       DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    note             VARCHAR(200),
    FOREIGN KEY (param_id) REFERENCES dim_spec_parameter(param_id)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `item_map_id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `source_table` | VARCHAR(20) | ✓ |  |  | 来源表名：spec_row(品牌官网) 或 product_spec(零售站) |
| 3 | `source_field` | VARCHAR(20) | ✓ |  |  | 来源字段名：item_ja(spec_row的日文规格项名) 或 key(product_spec的英文键名) |
| 4 | `source_term` | VARCHAR(200) | ✓ |  |  | 原始规格项名，如"画素数"/"Peak Brightness"/"Product Dimensions" |
| 5 | `param_id` | INTEGER | ✓ |  |  | 映射到的标准参数ID(1-18) |
| 6 | `match_keyword` | VARCHAR(100) |  |  |  | 匹配关键词，用于自动映射时的模糊匹配 |
| 7 | `extract_regex` | VARCHAR(200) |  |  |  | 提取正则，从原始值中提取数值部分 |
| 8 | `match_type` | VARCHAR(20) | ✓ | 'LIKE' |  | 匹配类型：EXACT(精确匹配) 或 LIKE(模糊匹配) |
| 9 | `status` | VARCHAR(20) | ✓ | 'AUTO' |  | 状态：AUTO(自动生成) 或 CONFIRMED(人工确认) |
| 10 | `brand_scope` | VARCHAR(50) |  |  |  | 品牌范围，限定该映射只对某品牌生效 |
| 11 | `created_at` | DATETIME | ✓ | CURRENT_TIMESTAMP |  | 创建时间 |
| 12 | `note` | VARCHAR(200) |  |  |  | 备注 |

**索引:**

- `idx_item_map_term`: CREATE INDEX idx_item_map_term   ON dim_spec_item_map(source_term)
- `idx_item_map_param`: CREATE INDEX idx_item_map_param   ON dim_spec_item_map(param_id)
- `idx_item_map_source`: CREATE INDEX idx_item_map_source  ON dim_spec_item_map(source_table)
- `idx_item_map_status`: CREATE INDEX idx_item_map_status  ON dim_spec_item_map(status)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `item_map_id`: `4406`
- `source_table`: `spec_row`
- `source_field`: `item_ja`
- `source_term`: `画素数`
- `param_id`: `10`
- `match_keyword`: `NULL`
- `extract_regex`: `(\d+)\s*[×x]\s*(\d+)`
- `match_type`: `EXACT`
- `status`: `CONFIRMED`
- `brand_scope`: `NULL`
- `created_at`: `2026-10-10 06:24:49`
- `note`: `NULL`

</details>

<details>
<summary>行 2</summary>

- `item_map_id`: `4407`
- `source_table`: `spec_row`
- `source_field`: `item_ja`
- `source_term`: `画素数(水平×垂直)`
- `param_id`: `10`
- `match_keyword`: `NULL`
- `extract_regex`: `([\d,]+)\s*[×x]\s*([\d,]+)`
- `match_type`: `EXACT`
- `status`: `CONFIRMED`
- `brand_scope`: `NULL`
- `created_at`: `2026-10-10 06:24:49`
- `note`: `NULL`

</details>

**按来源表分布:**

| source_table | source_field | 行数 | 含义 |
|------|------|------|------|
| spec_row | item_ja | 1227 | 品牌官网规格项名(日/德/法文) |
| product_spec | key | 747 | 零售站规格键名(英文) |

**按参数(param_id)分布:**

| param_id | 参数名 | 映射条数 | 示例source_term |
|----------|--------|---------|----------------|
| 1 | 峰值亮度 | 80 | Peak Brightness / Brightness |
| 2 | 分区控光 | 87 | バックライト / Dimming Zones |
| 3 | 刷新率 | 73 | 倍速対応 / 倍速駆動(120Hz) |
| 4 | 画质芯片 | 63 | エンジン / 映像処理エンジン |
| 5 | 智能系统 | 121 | システム / 搭載OS |
| 6 | 游戏特性 | 124 | ゲームモード / HDMI2.1仕様 |
| 7 | 色域 | 49 | 広色域 / 広色域(WCG) |
| 8 | HDMI 2.1 | 161 | HDMI / HDMI端子 |
| 9 | 屏幕尺寸 | 126 | 画面サイズ / パネルサイズ |
| 10 | 分辨率 | 111 | 画素数 / 画素数(水平×垂直) |
| 11 | HDR格式 | 108 | HDR機能 / HDR |
| 12 | 对比度 | 57 | コントラスト比 / パネルコントラスト比 |
| 13 | Wi-Fi | 76 | 無線LAN内蔵 / Wi-Fi内蔵 |
| 14 | 蓝牙 | 66 | Bluetoothイヤホン//ヘッドホン/サウンドバー / Bluetooth内蔵 |
| 15 | 杜比全景声 | 75 | 音声フォーマット / Dolby Atmos |
| 16 | 功耗 | 143 | 定格消費電力 / 定格消費電力(待機時消費電力) |
| 17 | 重量 | 207 | 梱包質量(kg) / 本体質量 |
| 18 | 尺寸 | 247 | スタンド含まない(cm) / スタンド含む(cm) |

**按状态(status)分布:**

| status | 行数 | 含义 |
|--------|------|------|
| AUTO | 425 | 自动生成 |
| CONFIRMED | 1549 | 人工确认 |

**按匹配类型(match_type)分布:**

| match_type | 行数 | 含义 |
|-----------|------|------|
| EXACT | 1917 | 精确匹配(项名完全相等) |
| LIKE | 57 | 模糊匹配(LIKE关键词包含) |

---

### 3.11 `dim_spec_normalization_map` — 同义归一映射表 (9行)

**层级**: 映射层  
**用途**: 不同品牌同义术语归一

**建表语句 (DDL):**
```sql
CREATE TABLE dim_spec_normalization_map (
    map_id              INTEGER PRIMARY KEY AUTOINCREMENT,
    hisense_term        VARCHAR(100) NOT NULL,
    competitor_terms    TEXT         NOT NULL,
    param_id            INTEGER,
    normalized_category VARCHAR(50)  NOT NULL,
    status              VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    confirmed_by        INTEGER,
    confirmed_at        DATETIME,
    created_at          DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    note                VARCHAR(200),
    FOREIGN KEY (param_id) REFERENCES dim_spec_parameter(param_id)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `map_id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `hisense_term` | VARCHAR(100) | ✓ |  |  | 海信的标准术语，如"144Hz高刷" |
| 3 | `competitor_terms` | TEXT | ✓ |  |  | 竞品同义术语JSON数组，如["Refresh Rate(Samsung)","Motion Xcelerator(LG)"] |
| 4 | `param_id` | INTEGER |  |  |  | 关联的参数ID |
| 5 | `normalized_category` | VARCHAR(50) | ✓ |  |  | 归一化分类名，如"高刷族"/"分区控光族" |
| 6 | `status` | VARCHAR(20) | ✓ | 'PENDING' |  | 状态：PENDING/CONFIRMED |
| 7 | `confirmed_by` | INTEGER |  |  |  | 确认人ID |
| 8 | `confirmed_at` | DATETIME |  |  |  | 确认时间 |
| 9 | `created_at` | DATETIME | ✓ | CURRENT_TIMESTAMP |  | 创建时间 |
| 10 | `note` | VARCHAR(200) |  |  |  | 备注 |

**索引:**

- `idx_norm_map_hisense`: CREATE INDEX idx_norm_map_hisense ON dim_spec_normalization_map(hisense_term)
- `idx_norm_map_param`: CREATE INDEX idx_norm_map_param   ON dim_spec_normalization_map(param_id)
- `idx_norm_map_status`: CREATE INDEX idx_norm_map_status  ON dim_spec_normalization_map(status)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `map_id`: `1`
- `hisense_term`: `144Hz高刷`
- `competitor_terms`: `["Refresh Rate(Samsung)","Motion Xcelerator(LG)","PANEL REFRESH RATE(TCL)","Hz(Panasonic)","倍速対応(Sony)","倍速対応(REGZA)","倍速駆動(TCL)"]`
- `param_id`: `3`
- `normalized_category`: `高刷族`
- `status`: `CONFIRMED`
- `confirmed_by`: `benchmark_maintain`
- `confirmed_at`: `2026-10-10 06:24:54`
- `created_at`: `2026-10-10 02:04:19`
- `note`: `NULL`

</details>

<details>
<summary>行 2</summary>

- `map_id`: `2`
- `hisense_term`: `Mini-LED分区控光`
- `competitor_terms`: `["Dimming Zones(Samsung)","Local Dimming(LG)","XR Backlight Master Drive(Sony)","Mini-LED/Local Dimming(TCL)","ローカルディミング(TCL)","バックライト(Hisense)","Mini LEDバックライト(REGZA)","VA×Mini LED(Panasonic)"]`
- `param_id`: `2`
- `normalized_category`: `分区控光族`
- `status`: `CONFIRMED`
- `confirmed_by`: `benchmark_maintain`
- `confirmed_at`: `2026-10-10 06:24:54`
- `created_at`: `2026-10-10 02:04:19`
- `note`: `NULL`

</details>

**9条归一化映射完整内容:**

| map_id | 海信术语 | param_id | 参数名 | 归一分类 | 竞品术语数 | 竞品术语列表 |
|--------|---------|----------|--------|---------|-----------|-------------|
| 1 | 144Hz高刷 | 3 | 刷新率 | 高刷族 | 7 | • Refresh Rate(Samsung)<br>• Motion Xcelerator(LG)<br>• PANEL REFRESH RATE(TCL)<br>• Hz(Panasonic)<br>• 倍速対応(Sony)<br>• 倍速対応(REGZA)<br>...等共7个 |
| 2 | Mini-LED分区控光 | 2 | 分区控光 | 分区控光族 | 8 | • Dimming Zones(Samsung)<br>• Local Dimming(LG)<br>• XR Backlight Master Drive(Sony)<br>• Mini-LED/Local Dimming(TCL)<br>• ローカルディミング(TCL)<br>• バックライト(Hisense)<br>...等共8个 |
| 3 | QLED量子点 | 7 | 色域 | 量子点族 | 8 | • QD(TCL)<br>• Quantum Dot(Samsung)<br>• Quantum Matrix(Samsung)<br>• NanoCell(LG)<br>• TRILUMINOS(Sony)<br>• 量子ドット(TCL)<br>...等共8个 |
| 4 | 信芯AI画质芯片 | 4 | 画质芯片 | 画质芯片族 | 8 | • AI Picture(Samsung)<br>• α AI Processor(LG)<br>• XR Processor(Sony)<br>• AIPQ Processor(TCL)<br>• Neural Quantum(Samsung)<br>• HI-VIEW AIエンジン(Hisense)<br>...等共8个 |
| 5 | VIDAA | 5 | 智能系统 | 智能系统族 | 11 | • Tizen(Samsung)<br>• webOS(LG)<br>• Google TV(Sony)<br>• Google TV(TCL)<br>• My Home Screen(Panasonic)<br>• Android TV(Sony)<br>...等共11个 |
| 6 | 峰值亮度 | 1 | 峰值亮度 | 亮度族 | 7 | • Peak Brightness(nits)(Samsung)<br>• Brightness Booster(LG)<br>• HDR Brightness Optimizer(Sony)<br>• Display Brightness(Hisense)<br>• Brightness(Hisense)<br>• 高輝度技術(Sony)<br>...等共7个 |
| 7 | HDMI 2.1 | 8 | HDMI 2.1 | HDMI族 | 8 | • HDMI 2.1(Samsung)<br>• eARC(LG)<br>• HDMI2.1仕様(Sony)<br>• HDMI 2.1(TCL)<br>• ARC(Panasonic)<br>• HDMI®2.1規格対応機能(Sony)<br>...等共8个 |
| 8 | HDR | 11 | HDR格式 | HDR族 | 9 | • HDR10+(Samsung)<br>• HDR(High Dynamic Range)(Samsung)<br>• Dolby Vision(LG)<br>• HLG(Sony)<br>• HDR10/HLG対応(Panasonic)<br>• HDR10(Hisense)<br>...等共9个 |
| 9 | 游戏模式 | 6 | 游戏特性 | 游戏族 | 11 | • Gaming Hub(Samsung)<br>• Game Optimizer(LG)<br>• Game Mode(Sony)<br>• 144Hz Game Mode Pro(TCL)<br>• Game Motion Plus(Samsung)<br>• Auto Game Mode(ALLM)(Samsung)<br>...等共11个 |

---

### 3.12 `fact_spec_value` — 归一化规格值表 (52,636行)

**层级**: 事实层  
**用途**: 每型号每参数的归一化规格值

**建表语句 (DDL):**
```sql
CREATE TABLE fact_spec_value (
    value_id         INTEGER PRIMARY KEY AUTOINCREMENT,
    model_id         INTEGER      NOT NULL,   -- → dim_product_model
    param_id         INTEGER      NOT NULL,   -- → dim_spec_parameter
    value_text       TEXT,                    -- 原始文本，如 '120Hz倍速（144Hz VRR）'
    value_numeric    DECIMAL(12,2),           -- 抽出的数值，如 144
    value_unit       VARCHAR(20),             -- 单位，如 'Hz'
    source_table     VARCHAR(20),              -- 'spec_row' / 'product_spec'
    source_term      VARCHAR(200),             -- 命中的源术语，如 '倍速対応'
    normalized_term  VARCHAR(200),             -- 归一化后的术语，如 '144Hz高刷'
    norm_map_id      INTEGER,                  -- 命中的归一化映射组 → dim_spec_normalization_map
    confidence       DECIMAL(5,2) DEFAULT 1.00,-- 抽值置信度
    captured_at      DATETIME,                 -- 抓取时间
    UNIQUE(model_id, param_id, source_table),
    FOREIGN KEY (model_id) REFERENCES dim_product_model(model_id),
    FOREIGN KEY (param_id) REFERENCES dim_spec_parameter(param_id)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `value_id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `model_id` | INTEGER | ✓ |  |  | 关联dim_product_model.model_id |
| 3 | `param_id` | INTEGER | ✓ |  |  | 关联dim_spec_parameter.param_id (1-18) |
| 4 | `value_text` | TEXT |  |  |  | 归一化后的规格值文本，如"4K (3840×2160)"/"120Hz"/"Dolby Atmos" |
| 5 | `value_numeric` | DECIMAL(12,2) |  |  |  | 数值型参数的数值，如400(nits)/120(Hz)。文本型为NULL |
| 6 | `value_unit` | VARCHAR(20) |  |  |  | 单位，如nits/Hz/kg |
| 7 | `source_table` | VARCHAR(20) |  |  |  | 数据来源表：spec_row/product_spec/cross_region_spec_row/cross_region_product_spec/manual_* |
| 8 | `source_term` | VARCHAR(200) |  |  |  | 原始规格项名（从哪个项名映射来的） |
| 9 | `normalized_term` | VARCHAR(200) |  |  |  | 归一化后的术语名 |
| 10 | `norm_map_id` | INTEGER |  |  |  |  |
| 11 | `confidence` | DECIMAL(5,2) |  | 1.00 |  | 置信度：1.0=直接抓取, 0.5=跨区域复制, 0.4=模糊匹配 |
| 12 | `captured_at` | DATETIME |  |  |  | 抓取时间 |

**索引:**

- `sqlite_autoindex_fact_spec_value_1`: 自动索引
- `idx_spec_value_model`: CREATE INDEX idx_spec_value_model  ON fact_spec_value(model_id)
- `idx_spec_value_param`: CREATE INDEX idx_spec_value_param   ON fact_spec_value(param_id)
- `idx_spec_value_num`: CREATE INDEX idx_spec_value_num     ON fact_spec_value(param_id, value_numeric)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `value_id`: `330261`
- `model_id`: `5277`
- `param_id`: `8`
- `value_text`: `4`
- `value_numeric`: `NULL`
- `value_unit`: `NULL`
- `source_table`: `spec_row`
- `source_term`: `HDMI`
- `normalized_term`: `NULL`
- `norm_map_id`: `NULL`
- `confidence`: `1`
- `captured_at`: `2026-10-10 06:24:50`

</details>

<details>
<summary>行 2</summary>

- `value_id`: `330794`
- `model_id`: `6172`
- `param_id`: `2`
- `value_text`: `VA×LED`
- `value_numeric`: `NULL`
- `value_unit`: `NULL`
- `source_table`: `spec_row`
- `source_term`: `パネル`
- `normalized_term`: `NULL`
- `norm_map_id`: `NULL`
- `confidence`: `1`
- `captured_at`: `2026-10-10 06:24:50`

</details>

**来源分布 (source_table):**

| source_table | 行数 | 含义 |
|------|------|------|
| spec_row | 27,583 | 品牌官网直接抓取(confidence=1.0) |
| product_spec | 11,084 | 零售站直接抓取(confidence=1.0) |
| cross_region_spec_row | 5,194 | 跨区域复制-品牌官网(confidence=0.5) |
| cross_region_product_spec | 2,703 | 跨区域复制-零售站(confidence=0.5) |
| cross_region_cross_region_spec_row | 1,808 | cross_region_cross_region_spec_row |
| cross_region_comp | 1,380 | cross_region_comp |
| cross_region_hisense | 1,146 | cross_region_hisense |
| cross_region_na_fuzzy | 514 | 北美模糊匹配(confidence=0.4) |
| cross_region_cross_region_product_spec | 326 | cross_region_cross_region_product_spec |
| cross_region_fuzzy | 290 | cross_region_fuzzy |
| cross_region_na | 277 | 北美跨区域匹配(confidence=0.5) |
| cross_region_bo_hisense | 127 | cross_region_bo_hisense |
| cross_region_final | 91 | cross_region_final |
| cross_region_cross_region_cross_region_spec_row | 49 | cross_region_cross_region_cross_region_spec_row |
| manual_kakaku_sharp_official | 28 | 手动补充Sharp官网(confidence=1.0) |
| cross_region_samsung_65 | 16 | cross_region_samsung_65 |
| cross_region_samsung_55 | 16 | cross_region_samsung_55 |
| cross_region_na_anybrand | 4 | 北美跨品牌匹配(confidence=0.4) |

**按参数分布 (param_id):**

| param_id | 参数名 | 行数 |
|----------|--------|------|
| 1 | 峰值亮度 | 1,128 |
| 2 | 分区控光 | 2,301 |
| 3 | 刷新率 | 3,001 |
| 4 | 画质芯片 | 2,703 |
| 5 | 智能系统 | 3,826 |
| 6 | 游戏特性 | 3,214 |
| 7 | 色域 | 1,372 |
| 8 | HDMI 2.1 | 3,797 |
| 9 | 屏幕尺寸 | 3,033 |
| 10 | 分辨率 | 4,057 |
| 11 | HDR格式 | 3,649 |
| 12 | 对比度 | 2,235 |
| 13 | Wi-Fi | 3,650 |
| 14 | 蓝牙 | 3,159 |
| 15 | 杜比全景声 | 2,581 |
| 16 | 功耗 | 3,291 |
| 17 | 重量 | 3,050 |
| 18 | 尺寸 | 2,589 |

**覆盖率:** 3451/6493 (53.1%)

**置信度分布:**

| confidence | 行数 | 含义 |
|-----------|------|------|
| 1 | 8,530 | 直接抓取 |
| 0.9 | 22,871 | 跨区域复制 |
| 0.8 | 7,294 | 跨区域复制 |
| 0.6 | 5,346 | 跨区域复制 |
| 0.55 | 972 | 跨区域复制 |
| 0.5 | 6,815 | 跨区域复制 |
| 0.4 | 808 | 模糊匹配 |

---

### 3.13 `fact_benchmark_comparison` — 对阵比较结果表 (4行)

**层级**: 事实层  
**用途**: AI对阵分析结果存储

**建表语句 (DDL):**
```sql
CREATE TABLE fact_benchmark_comparison (
    comparison_id           INTEGER PRIMARY KEY AUTOINCREMENT,
    comparison_name         VARCHAR(100) NOT NULL,
    size_inch               INTEGER      NOT NULL,
    price_band              VARCHAR(30)  NOT NULL,
    our_model_id            INTEGER      NOT NULL,
    competitor_model_ids    TEXT         NOT NULL,  -- JSON数组: [model_id, ...]
    advantage_points        TEXT,                    -- JSON数组
    disadvantage_points     TEXT,                    -- JSON数组
    opportunity_points      TEXT,                    -- JSON数组
    advantage_confidence    DECIMAL(5,2),
    disadvantage_confidence DECIMAL(5,2),
    opportunity_confidence  DECIMAL(5,2),
    strategy_suggestion     TEXT,
    created_by              INTEGER,
    created_at              DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    snapshot_date           DATE         NOT NULL,
    FOREIGN KEY (our_model_id) REFERENCES dim_product_model(model_id)
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `comparison_id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `comparison_name` | VARCHAR(100) | ✓ |  |  | 对阵名称，如"65寸中端对阵" |
| 3 | `size_inch` | INTEGER | ✓ |  |  | 尺寸 |
| 4 | `price_band` | VARCHAR(30) | ✓ |  |  | 价格档位：入门/中端/高端/旗舰 |
| 5 | `our_model_id` | INTEGER | ✓ |  |  | 海信型号ID |
| 6 | `competitor_model_ids` | TEXT | ✓ |  |  | 竞品型号ID列表JSON数组 |
| 7 | `advantage_points` | TEXT |  |  |  | 优势点JSON |
| 8 | `disadvantage_points` | TEXT |  |  |  | 劣势点JSON |
| 9 | `opportunity_points` | TEXT |  |  |  | 机会点JSON |
| 10 | `advantage_confidence` | DECIMAL(5,2) |  |  |  |  |
| 11 | `disadvantage_confidence` | DECIMAL(5,2) |  |  |  |  |
| 12 | `opportunity_confidence` | DECIMAL(5,2) |  |  |  |  |
| 13 | `strategy_suggestion` | TEXT |  |  |  | 策略建议文本 |
| 14 | `created_by` | INTEGER |  |  |  |  |
| 15 | `created_at` | DATETIME | ✓ | CURRENT_TIMESTAMP |  |  |
| 16 | `snapshot_date` | DATE | ✓ |  |  | 快照日期 |

**索引:**

- `idx_benchmark_size`: CREATE INDEX idx_benchmark_size    ON fact_benchmark_comparison(size_inch)
- `idx_benchmark_band`: CREATE INDEX idx_benchmark_band    ON fact_benchmark_comparison(price_band)
- `idx_benchmark_our`: CREATE INDEX idx_benchmark_our     ON fact_benchmark_comparison(our_model_id)
- `idx_benchmark_created`: CREATE INDEX idx_benchmark_created ON fact_benchmark_comparison(created_at)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `comparison_id`: `18`
- `comparison_name`: `65寸入门对阵`
- `size_inch`: `65`
- `price_band`: `入门`
- `our_model_id`: `2258`
- `competitor_model_ids`: `[1195, 2301, 2302, 2347]`
- `advantage_points`: `["[vs Sharp 4T-C65GL1] 功耗 142W 领先 193W"]`
- `disadvantage_points`: `[]`
- `opportunity_points`: `[]`
- `advantage_confidence`: `0.9`
- `disadvantage_confidence`: `0.85`
- `opportunity_confidence`: `0.82`
- `strategy_suggestion`: `NULL`
- `created_by`: `NULL`
- `created_at`: `2026-10-10 02:13:48`
- `snapshot_date`: `2026-10-10`

</details>

<details>
<summary>行 2</summary>

- `comparison_id`: `19`
- `comparison_name`: `65寸中端对阵`
- `size_inch`: `65`
- `price_band`: `中端`
- `our_model_id`: `2246`
- `competitor_model_ids`: `[1198, 1200, 1202, 1204]`
- `advantage_points`: `["[vs Sharp 4T-C65HN2] 功耗 142W 领先 239W", "[vs Sharp 4T-C65HP1] 功耗 142W 领先 253W", "[vs Sharp 4T-C65HQ1] 功耗 142W 领先 469W", "[vs Sharp 4T-C65HQ2] 功耗 142W 领先 384W"]`
- `disadvantage_points`: `[]`
- `opportunity_points`: `[]`
- `advantage_confidence`: `0.9`
- `disadvantage_confidence`: `0.85`
- `opportunity_confidence`: `0.82`
- `strategy_suggestion`: `NULL`
- `created_by`: `NULL`
- `created_at`: `2026-10-10 02:13:48`
- `snapshot_date`: `2026-10-10`

</details>


---

### 3.14 `site` — 站点配置表 (260行)

**层级**: 运维监控层  
**用途**: 爬取站点配置

**建表语句 (DDL):**
```sql
CREATE TABLE site (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    code        TEXT NOT NULL UNIQUE,          -- amazon_ca
    name        TEXT NOT NULL,
    base_url    TEXT,
    protection  TEXT,                          -- L1 / L2 / L3
    created_at  TEXT NOT NULL
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `code` | TEXT | ✓ |  |  | 站点代码，如amazon_jp/bestbuy_ca |
| 3 | `name` | TEXT | ✓ |  |  | 站点名称 |
| 4 | `base_url` | TEXT |  |  |  | 站点基础URL |
| 5 | `protection` | TEXT |  |  |  | 防护等级 |
| 6 | `created_at` | TEXT | ✓ |  |  |  |

**索引:**

- `sqlite_autoindex_site_1`: 自动索引

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `code`: `amazon_ca`
- `name`: `Amazon Canada`
- `base_url`: `https://www.amazon.ca`
- `protection`: `L3`
- `created_at`: `2026-09-04T05:21:57+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `4`
- `code`: `kakaku_jp`
- `name`: `価格.com（日本）`
- `base_url`: `https://kakaku.com`
- `protection`: `L2`
- `created_at`: `2026-09-09T07:30:41+00:00`

</details>


---

### 3.15 `crawl_run` — 爬取运行记录表 (842行)

**层级**: 运维监控层  
**用途**: 每次爬取运行的记录

**建表语句 (DDL):**
```sql
CREATE TABLE crawl_run (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    scenario    TEXT NOT NULL,                 -- s1_search / s2_monitor / s4_reviews
    site_code   TEXT,
    params      TEXT,                          -- JSON
    status      TEXT NOT NULL,                 -- running/completed/failed/stopped
    started_at  TEXT NOT NULL,
    finished_at TEXT,
    ok_count    INTEGER DEFAULT 0,
    fail_count  INTEGER DEFAULT 0,
    message     TEXT
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `scenario` | TEXT | ✓ |  |  | 场景：s1_search/s2_monitor/s3_detail/s4_reviews |
| 3 | `site_code` | TEXT |  |  |  | 站点代码 |
| 4 | `params` | TEXT |  |  |  | 运行参数JSON |
| 5 | `status` | TEXT | ✓ |  |  | 状态：running/completed/failed/stopped |
| 6 | `started_at` | TEXT | ✓ |  |  | 开始时间 |
| 7 | `finished_at` | TEXT |  |  |  | 结束时间 |
| 8 | `ok_count` | INTEGER |  | 0 |  | 成功数 |
| 9 | `fail_count` | INTEGER |  | 0 |  | 失败数 |
| 10 | `message` | TEXT |  |  |  | 结果消息 |

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `scenario`: `s1_search`
- `site_code`: `amazon_ca`
- `params`: `{"keyword": "hisense tv", "limit": 3, "pages": 1}`
- `status`: `completed`
- `started_at`: `2026-09-04T05:21:57+00:00`
- `finished_at`: `2026-09-04T05:22:57+00:00`
- `ok_count`: `3`
- `fail_count`: `0`
- `message`: `ok=3 fail=0 blocked=0 products=3 prices=3 reviews_new=29`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `scenario`: `s3_detail`
- `site_code`: `amazon_ca`
- `params`: `{"skus": ["B0GPHMTMLC"]}`
- `status`: `completed`
- `started_at`: `2026-09-04T05:26:31+00:00`
- `finished_at`: `2026-09-04T05:26:42+00:00`
- `ok_count`: `1`
- `fail_count`: `0`
- `message`: `ok=1 fail=0 blocked=0 products=1 prices=1 reviews_new=0`

</details>


---

### 3.16 `crawl_error` — 爬取错误日志表 (3,472行)

**层级**: 运维监控层  
**用途**: 爬取过程中的错误日志

**建表语句 (DDL):**
```sql
CREATE TABLE crawl_error (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    run_id      INTEGER REFERENCES crawl_run(id),
    url         TEXT,
    stage       TEXT,                          -- search/detail/reviews
    kind        TEXT,                          -- blocked/timeout/http/parse
    message     TEXT,
    created_at  TEXT NOT NULL
)
```

**字段说明:**

| # | 列名 | 类型 | NOT NULL | 默认值 | PK | 说明 |
|---|------|------|----------|--------|----|------|
| 1 | `id` | INTEGER |  |  | 🔑 | 自增主键 |
| 2 | `run_id` | INTEGER |  |  |  | 关联crawl_run.id |
| 3 | `url` | TEXT |  |  |  | 出错URL |
| 4 | `stage` | TEXT |  |  |  | 阶段：search/detail/reviews |
| 5 | `kind` | TEXT |  |  |  | 错误类型：blocked/timeout/http/parse |
| 6 | `message` | TEXT |  |  |  | 错误消息 |
| 7 | `created_at` | TEXT | ✓ |  |  | 创建时间 |

**索引:**

- `idx_error_run`: CREATE INDEX idx_error_run ON crawl_error(run_id)

**样本数据 (前2行，完整字段):**

<details>
<summary>行 1</summary>

- `id`: `1`
- `run_id`: `10`
- `url`: `https://www.regza.com/tv/lineup/zx1r/spec`
- `stage`: `spec`
- `kind`: `parse`
- `message`: `无行或无机型`
- `created_at`: `2026-09-07T02:45:16+00:00`

</details>

<details>
<summary>行 2</summary>

- `id`: `2`
- `run_id`: `11`
- `url`: `https://www.regza.com/tv/lineup/zx1r/spec`
- `stage`: `spec`
- `kind`: `parse`
- `message`: `无行或无机型`
- `created_at`: `2026-09-07T03:00:37+00:00`

</details>


---

## 4. 表间关系详解

### 4.1 外键关系图

```
site ──1:N──→ product ──1:N──→ price_snapshot
                │            
                ├──1:N──→ product_spec ──→ dim_spec_item_map(key→param_id)
                └──1:N──→ review ──→ review_summary

spec_series ──1:N──→ spec_row ──→ dim_spec_item_map(item_ja→param_id)

dim_product_model ──1:N──→ fact_spec_value ←── dim_spec_parameter
      │                    (model_id+param_id)
      └──→ fact_benchmark_comparison (our_model_id)

dim_spec_normalization_map (独立表, 通过param_id关联dim_spec_parameter)
```

### 4.2 核心JOIN路径

**路径1: 查某型号的所有规格值**
```sql
SELECT p.param_name, f.value_text, f.value_numeric, f.confidence, f.source_table
FROM fact_spec_value f
JOIN dim_spec_parameter p ON p.param_id = f.param_id
WHERE f.model_id = ?
ORDER BY p.sort_order;
```

**路径2: 查某区域有规格的型号列表**
```sql
SELECT DISTINCT m.model_code, m.brand, m.size_inch, m.our_brand
FROM dim_product_model m
WHERE m.model_code IN (
    SELECT DISTINCT p.model FROM product p
    JOIN price_snapshot ps ON ps.product_id = p.id
    WHERE ps.currency = ?  -- 如'USD'
    AND LENGTH(p.model) > 4
)
AND m.model_id IN (SELECT DISTINCT model_id FROM fact_spec_value)
ORDER BY m.our_brand DESC, m.brand;
```

**路径3: 规格项名映射路径**
```sql
-- 从spec_row到fact_spec_value的完整路径:
SELECT sr.item_ja AS 原始规格项名,
       m.param_id AS 映射到的参数ID,
       p.param_name AS 参数中文名,
       f.value_text AS 归一化后的值
FROM spec_row sr
JOIN dim_spec_item_map m ON m.source_table='spec_row' 
    AND m.source_term = sr.item_ja  -- 或 LIKE匹配
JOIN dim_spec_parameter p ON p.param_id = m.param_id
-- fact_spec_value通过model_id+param_id关联
```

**路径4: 价格查询路径**
```sql
SELECT m.model_code, m.brand, m.our_brand, ps.price, ps.currency
FROM dim_product_model m
JOIN product p ON p.model = m.model_code
JOIN price_snapshot ps ON ps.product_id = p.id
WHERE ps.currency = ?  -- 如'JPY'
ORDER BY m.our_brand DESC, ps.price;
```

## 5. 数据流转完整流程

### 5.1 从爬虫到对阵分析的数据流

```
步骤1: 爬虫抓取
  SPEC分支: 品牌官网lineup页 → spec_series(系列+型号列表) → spec_row(规格表行)
  RETAIL分支: 电商搜索页 → product(产品) → price_snapshot(价格) → product_spec(零售规格) → review(评论)

步骤2: 型号入维度表
  product.model / spec_series.models → dim_product_model (统一型号, our_brand标记)

步骤3: 规格项名映射
  spec_row.item_ja → dim_spec_item_map(source_table='spec_row') → param_id
  product_spec.key → dim_spec_item_map(source_table='product_spec') → param_id

步骤4: 规格值归一化
  spec_row.values_json[model] → _normalize_display()正则归一化 → fact_spec_value.value_text
  product_spec.value → _normalize_display()正则归一化 → fact_spec_value.value_text

步骤5: 跨区域补全
  无fact_spec_value的型号 → 核心码匹配同品牌同系列有规格的型号 → 复制fact_spec_value(confidence=0.5)

步骤6: 对阵分析查询
  用户选国家+尺寸 → api_models()查型号 → api_spec()查fact_spec_value → 18参数矩阵 → AI分析
```

## 6. 数据库关键设计决策

### 6.1 为什么用SQLite而不是MySQL/PostgreSQL?
- **零部署成本**: SQLite是单文件，无需安装数据库服务
- **离线部署**: 服务器无需外网即可运行
- **读取性能**: 读多写少场景下SQLite性能足够
- **便携性**: 整个数据库(135MB)可U盘拷贝

### 6.2 为什么fact_spec_value不直接存原始值?
- **多源差异**: 同一参数从spec_row(日文)和product_spec(英文)抓取，格式完全不同
- **归一化需求**: 前端展示需统一格式(如'120Hz'而非'倍速120Hz対応')
- **可追溯**: source_table+source_term保留原始来源, confidence标记可信度

### 6.3 为什么dim_spec_item_map有两类(source_table)?
- **数据源差异**: spec_row的item_ja是多语言(日/德/法), product_spec的key是英文
- **映射独立**: 同一参数可能从两种来源都有映射, 需要分别配置
- **示例**: param_id=10(分辨率) ← '画素数'(spec_row) + 'Resolution'(product_spec)

### 6.4 为什么confidence字段重要?
- **数据可信度**: 直接抓取(1.0) > 跨区域复制(0.5) > 模糊匹配(0.4)
- **前端展示**: 可选择只展示高置信度数据
- **溯源**: 知道某条数据是从哪个区域的哪个型号复制来的(source_term含'from XXX')

### 6.5 为什么our_brand是TINYINT而不是BOOLEAN?
- SQLite没有原生BOOLEAN类型, 用0/1表示
- 1=海信(我方), 0=竞品, 决定对阵分析中谁是'我方'
- 前端通过is_our字段返回给前端

