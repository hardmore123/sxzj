# 竞品对阵分析 — 功能说明

## 一、功能概览

竞品对阵分析是面向海信电视海外市场的规格对标系统，核心价值在于：

- **核心价值**：将海信（Hisense）电视与各区域竞品电视做规格对阵分析，自动归一化多语言规格项，再由 AI 生成结构化的优势点 / 劣势点 / 机会点 / 策略建议，输出可溯源到具体参数或价格的结论。
- **覆盖范围**：21 个国家 / 区域，按 4 大业务模块组织——日本线、北美线、南美线、欧洲线。
- **数据规模**：6,493 个型号维度、52,636 条归一化规格事实值、1,974 条规格项映射规则，覆盖 8 个品牌（海信 / Sony / Samsung / LG / TCL / REGZA / Sharp / Panasonic）。
- **四大模块**：
  - **规格对阵矩阵**：18 个标准对标参数的归一化对比表
  - **AI 优势分析**：xinghai-ultra 大模型生成结论 + 规则兜底
  - **雷达图**：ECharts 多维能力可视化
  - **参数自动补全（backfill）**：跨区域三层回溯，只取真实数据、AI 不编造

---

## 二、核心功能

### 2.1 国家 / 区域选择

系统支持 21 个国家 / 区域，按 4 大业务模块分组。每个国家配置了独立货币和按尺寸分档的价格档位（入门 / 中端 / 高端 / 旗舰），用于型号筛选与价格带卡位分析。

- **API**：`GET /api/countries` —— 返回全部 21 国的 `code / name / currency / flag / group`。
- **区域过滤策略**（`_region_filter`）：
  - **JP（日本线）**：竞品按 `brand_raw LIKE '%日本%'` 匹配日本线品牌，海信按 `brand LIKE '%isense%' AND currency='JPY'` 匹配有日元价格的我方型号。
  - **非 JP**：按该区域货币严格过滤，只显示有该货币价格的型号（`WHERE ps.currency=?`），**缺价 > 错价**，无该货币价格则不显示，不回退到任意货币。

完整国家配置见 [第四节：21 个国家配置](#四21-个国家配置)。

### 2.2 型号列表查询

按「区域 + 尺寸」筛选可用型号，返回每个型号的均价、价格档位和我方 / 竞品标记。

- **API**：
  - `GET /api/sizes?region=JP` —— 返回该区域可用尺寸列表 + 各尺寸型号数。尺寸白名单：19/24/32/40/42/43/48/50/55/58/65/75/77/85/98/100 英寸。
  - `GET /api/models?region=JP&size=65` —— 返回该区域该尺寸的所有型号。
- **返回字段**：`model_id / brand / model_code / price（均价）/ price_band / our_brand / is_our`。
- **价格计算**：严格按区域货币取 `AVG(price_snapshot.price)`，无该货币价格则跳过该型号。
- **价格档位自动计算**：按 `COUNTRIES[region]['bands'][size]` 的区间把均价映射到「入门 / 中端 / 高端 / 旗舰」之一。
- **脏数据过滤**（`_is_valid_model_code`）：排除纯数字（如 `38751`）、纯规格词（如 `180HZ`、`60LED`、`4KHDR`，匹配 `^\d{2,4}[A-Z]{1,3}$`）和过短（<4 字符）的型号名，只保留真实型号（如 `65U8R`、`65C855`）。
- **排序**：`ORDER BY our_brand DESC, brand` —— 我方型号优先，再按品牌字母序。

### 2.3 规格对阵矩阵

对选定的多个型号生成 18 个标准对标参数的归一化对比表。

- **API**：`GET /api/spec?model_ids=1195,2301,2302&region=JP` —— `model_ids` 为逗号分隔的 `dim_product_model.model_id`。
- **核心逻辑**（`api_spec_compare`）：
  - 从 `fact_spec_value` 取各型号已归一化的规格值，按 18 个参数（`PARAM_ORDER`）逐行排列。
  - 每个单元格含 `display`（归一化显示值）和 `numeric`（数值型参数的数值）。
  - **缺失参数标记为 `"—"`**：无事实值的单元格统一显示为 `—`，不编造。
  - 价格严格按区域货币过滤，不回退。
- **输出**：`spec_matrix`（参数 → 各型号值）+ `models`（型号元信息 + 价格）。

### 2.4 AI 优势分析

接入 xinghai-ultra 大模型，基于规格对阵数据生成结构化的我方 vs 竞品结论。

- **API**：`GET /api/ai-analysis?model_ids=...&region=JP`。
- **输入构造**：把规格矩阵压缩为文本摘要（区域、各型号角色与价格、前 12 个参数的对比值），拼进 prompt。
- **输出 JSON 结构**：
  ```json
  {
    "advantage_points": [{"point":"...","evidence":"...","confidence":0.9}],
    "disadvantage_points": [{"point":"...","evidence":"...","confidence":0.85}],
    "opportunity_points": [{"point":"...","evidence":"...","confidence":0.8}],
    "strategy_suggestion": "一句话策略建议"
  }
  ```
  - **优势点**：我方领先的参数（数值高 / 功能全）。
  - **劣势点**：我方落后之处。
  - **机会点**：可差异化或补位的市场机会。
  - 每条要求可溯源到具体参数或价格，置信度 0–1。
- **规则兜底**（`_fallback_analysis`）：AI 不可用时自动降级为规则结论，保证服务可用——
  - `llm_client` 模块缺失 → 兜底。
  - 未配置 API Key → 兜底并提示 `config_needed`。
  - `LLMUnavailable` / `JSONDecodeError` → 兜底。
  - 兜底逻辑：价格低于竞品均价 85% 记为「价格领先」优势点；`REFRESH_RATE / GAMING / HDR` 等参数检测 `144Hz / FreeSync / 240Hz / Dolby Vision` 等关键词记优势点。

### 2.5 雷达图

基于 ECharts 能力雷达，对选定型号做多维度可视化对比。

- **API**：`GET /api/radar?model_ids=...`。
- **6 个雷达维度**（`RADAR_DIMS`）：
  | 维度 | 包含参数 |
  |------|---------|
  | 画质 | PEAK_BRIGHTNESS, DIMMING_ZONES, REFRESH_RATE, COLOR_GAMUT, HDR, CHIP, RESOLUTION, CONTRAST |
  | 音质 | DOLBY_ATMOS |
  | 智能 | SYSTEM |
  | 游戏 | GAMING, REFRESH_RATE, HDMI |
  | 连接 | HDMI, WIFI, BLUETOOTH |
  | 设计 | POWER_CONSUMPTION, SCREEN_SIZE, WEIGHT, DIMENSIONS |
- **评分逻辑**：
  - 文本型参数按关键词档位评分（如 SYSTEM：VIDAA=55、Google TV/Android=80、Tizen/webOS=78）。
  - 数值型参数按该组型号的 min–max 归一化到 60–95 分，方向由 `higher_is_better` 决定。
  - 无数据时该维度记 50 分（基准线）。
- **输出**：`radar_dims`（维度名列表）+ `scores`（型号 → 各维度分）+ `models`。

### 2.6 参数自动补全（backfill）

对缺失参数做三层递进回溯补全，**核心原则：AI 永远不产出规格值，只查找"去哪里能抓到真实数据"；找不到则诚实标记"待维护"，绝不编造**。

- **API**：
  - `POST /api/backfill`（body: `{"model_ids":"1195,2301","region":"JP","skip_ai":false}`）—— 触发补全。
  - `GET /api/backfill-status?model_id=1195`—— 查某型号已有 / 缺失参数。
- **三层回溯策略**（`backfill_model_params`）：
  | 层 | 来源 | 做什么 | 置信度 |
  |----|------|--------|--------|
  | 第 1 层 | **db** | 在原始 `spec_row` / `product_spec` 里用 18 参数的精确关键词（日英多语言）搜索该型号真实记录的值 | ≥0.9 直接采用 |
  | 第 2 层 | **site** | 调 `spec_crawl` 重抓官网规格页（真实抓取，按 `brand_raw → adapter code` 映射选适配器） | 真实抓到才入库 |
  | 第 3 层 | **ai** | 调 AI 查找该型号的官网规格页 URL，供第 2 层真实抓取（**AI 不产出规格值**） | 找到 URL 后交 spec_crawl |
  | 兜底 | — | 三层均未找到真实数据源 → 标记 `source:'none'`，detail 说明原因 | 0 |
- `skip_ai=True`：只走第 1 层 + 第 2 层（全库批量自动溯源用）；`skip_ai=False`：走全部三层含 AI 查 URL（手动触发用）。
- **跨区域核心码匹配**：通过 `brand_raw → adapter code` 映射（`BRAND_TO_ADAPTER`），支持海信 / Sony / REGZA / Sharp / Panasonic / TCL 各品牌日本线适配器的跨区域补全。

---

## 三、18 个对标参数

系统定义了 18 个标准对标参数（`dim_spec_parameter` 表），覆盖画质 / 智能 / 游戏 / 连接 / 音质 / 设计 6 类。下表按 `sort_order` 排列：

| param_id | param_code | 参数名（中） | 参数名（日） | 单位 | 数值型 | 高优/低优 | 类别 | 雷达维度 |
|----------|-----------|------------|------------|------|:------:|:--------:|------|---------|
| 1 | PEAK_BRIGHTNESS | 峰值亮度 | ピーク輝度 | nits | ✔ | 高优 | 画质 | 画质 |
| 2 | DIMMING_ZONES | 分区控光 | ローカルディミング | zones | ✔ | 高优 | 画质 | 画质 |
| 3 | REFRESH_RATE | 刷新率 | リフレッシュレート | Hz | ✔ | 高优 | 画质 | 画质/游戏 |
| 4 | CHIP | 画质芯片 | 画質エンジン | — | ✘ | — | 画质 | 画质 |
| 5 | SYSTEM | 智能系统 | システム/OS | — | ✘ | — | 智能 | 智能 |
| 6 | GAMING | 游戏特性 | ゲーム機能 | — | ✘ | 高优 | 游戏 | 游戏 |
| 7 | COLOR_GAMUT | 色域 | 色域/広色域 | % | ✔ | 高优 | 画质 | 画质 |
| 8 | HDMI | HDMI 2.1 | HDMI 2.1 | 口 | ✘ | 高优 | 连接 | 连接/游戏 |
| 9 | SCREEN_SIZE | 屏幕尺寸 | 画面サイズ | inch | ✔ | 低优 | 设计 | 设计 |
| 10 | RESOLUTION | 分辨率 | 画素数/解像度 | — | ✘ | 高优 | 画质 | 画质 |
| 11 | HDR | HDR格式 | HDR対応 | — | ✘ | 高优 | 画质 | 画质 |
| 12 | CONTRAST | 对比度 | コントラスト | :1 | ✔ | 高优 | 画质 | 画质 |
| 13 | WIFI | Wi-Fi | Wi-Fi/無線LAN | — | ✘ | 高优 | 连接 | 连接 |
| 14 | BLUETOOTH | 蓝牙 | Bluetooth | — | ✘ | 高优 | 连接 | 连接 |
| 15 | DOLBY_ATMOS | 杜比全景声 | Dolby Atmos | — | ✘ | 高优 | 音质 | 音质 |
| 16 | POWER_CONSUMPTION | 功耗 | 消費電力 | W | ✔ | 低优 | 设计 | 设计 |
| 17 | WEIGHT | 重量 | 本体質量 | kg | ✔ | 低优 | 设计 | 设计 |
| 18 | DIMENSIONS | 尺寸 | 外形寸法 | mm | ✘ | 低优 | 设计 | 设计 |

> 说明：`higher_is_better` 为 1 表示数值越高越好（高优），为 0 表示越低越好（低优），为 NULL 表示文本型参数不适用方向比较。`radar_dim` 标记该参数参与哪个雷达维度评分。

---

## 四、21 个国家配置

下表列出全部 21 国的货币与价格档位。价格档位按尺寸（55/65/75/85 英寸，部分国家无 85 寸档）分四档：入门 / 中端 / 高端 / 旗舰。区间为 `[下限, 上限)`，上限 `9e9` 表示无上限。

### 日本线

| code | 国家 | 货币 | 尺寸档位 |
|------|------|------|--------|
| JP | 日本 🇯🇵 | JPY | 55: 入门(0–100k)/中端(100k–200k)/高端(200k–350k)/旗舰(350k+); 65: 0–150k/150k–300k/300k–500k/500k+; 75: 0–200k/200k–400k/400k–600k/600k+; 85: 0–300k/300k–500k/500k–800k/800k+ |

### 北美线

| code | 国家 | 货币 | 尺寸档位 |
|------|------|------|--------|
| US | 美国 🇺🇸 | USD | 55: 0–400/400–900/900–1800/1800+; 65: 0–600/600–1200/1200–2500/2500+; 75: 0–800/800–1800/1800–3500/3500+; 85: 0–1200/1200–2500/2500–5000/5000+ |
| CA | 加拿大 🇨🇦 | CAD | 55: 0–500/500–1000/1000–2000/2000+; 65: 0–700/700–1400/1400–3000/3000+; 75: 0–1000/1000–2000/2000–4000/4000+; 85: 0–1500/1500–3000/3000–6000/6000+ |
| MX | 墨西哥 🇲🇽 | MXN | 55: 0–8k/8k–16k/16k–32k/32k+; 65: 0–12k/12k–24k/24k–45k/45k+; 75: 0–18k/18k–35k/35k–60k/60k+（无 85 寸） |

### 南美线

| code | 国家 | 货币 | 尺寸档位 |
|------|------|------|--------|
| CO | 哥伦比亚 🇨🇴 | COP | 55: 0–1.5M/1.5M–3M/3M–5M/5M+; 65: 0–2.5M/2.5M–5M/5M–8M/8M+; 75: 0–3.5M/3.5M–7M/7M–12M/12M+ |
| PE | 秘鲁 🇵🇪 | PEN | 55: 0–1200/1200–2500/2500–4500/4500+; 65: 0–1800/1800–3800/3800–6500/6500+; 75: 0–2800/2800–5500/5500–9000/9000+ |
| CL | 智利 🇨🇱 | CLP | 55: 0–350k/350k–700k/700k–1.2M/1.2M+; 65: 0–550k/550k–1.1M/1.1M–1.8M/1.8M+; 75: 0–800k/800k–1.6M/1.6M–2.8M/2.8M+ |
| GT | 危地马拉 🇬🇹 | GTQ | 55: 0–2500/2500–5000/5000–9000/9000+; 65: 0–4000/4000–8000/8000–14k/14k+; 75: 0–6000/6000–12k/12k–20k/20k+ |
| BO | 玻利维亚 🇧🇴 | BOB | 55: 0–2000/2000–4000/4000–7000/7000+; 65: 0–3000/3000–6000/6000–10k/10k+; 75: 0–5000/5000–9000/9000–15k/15k+ |

### 欧洲线

| code | 国家 | 货币 | 尺寸档位 |
|------|------|------|--------|
| UK | 英国 🇬🇧 | GBP | 55: 0–300/300–700/700–1400/1400+; 65: 0–500/500–1000/1000–2000/2000+; 75: 0–700/700–1500/1500–3000/3000+ |
| EU | 欧洲欧元区 🇪🇺 | EUR | 55: 0–400/400–800/800–1600/1600+; 65: 0–600/600–1200/1200–2400/2400+; 75: 0–800/800–1800/1800–3500/3500+ |
| CH | 瑞士 🇨🇭 | CHF | 55: 0–500/500–1000/1000–2000/2000+; 65: 0–800/800–1500/1500–3000/3000+; 75: 0–1200/1200–2500/2500–4500/4500+ |
| SE | 瑞典 🇸🇪 | SEK | 55: 0–5k/5k–10k/10k–18k/18k+; 65: 0–8k/8k–15k/15k–28k/28k+; 75: 0–12k/12k–22k/22k–40k/40k+ |
| PL | 波兰 🇵🇱 | PLN | 55: 0–2500/2500–5000/5000–9000/9000+; 65: 0–4000/4000–8000/8000–14k/14k+; 75: 0–6000/6000–12k/12k–20k/20k+ |
| RS | 塞尔维亚 🇷🇸 | RSD | 55: 0–40k/40k–80k/80k–140k/140k+; 65: 0–60k/60k–120k/120k–210k/210k+; 75: 0–90k/90k–180k/180k–300k/300k+ |
| RO | 罗马尼亚 🇷🇴 | RON | 55: 0–2000/2000–4000/4000–7000/7000+; 65: 0–3000/3000–6000/6000–11k/11k+; 75: 0–5000/5000–9000/9000–16k/16k+ |
| NO | 挪威 🇳🇴 | NOK | 55: 0–5k/5k–10k/10k–18k/18k+; 65: 0–8k/8k–15k/15k–28k/28k+; 75: 0–12k/12k–22k/22k–40k/40k+ |
| DK | 丹麦 🇩🇰 | DKK | 55: 0–4k/4k–8k/8k–14k/14k+; 65: 0–6k/6k–12k/12k–22k/22k+; 75: 0–9k/9k–18k/18k–32k/32k+ |
| BG | 保加利亚 🇧🇬 | BGN | 55: 0–1500/1500–3000/3000–5500/5500+; 65: 0–2500/2500–5000/5000–8500/8500+; 75: 0–4000/4000–7500/7500–13k/13k+ |
| BA | 波黑 🇧🇦 | BAM | 55: 0–1000/1000–2000/2000–3500/3500+; 65: 0–1500/1500–3000/3000–5500/5500+; 75: 0–2500/2500–5000/5000–8500/8500+ |
| TR | 土耳其 🇹🇷 | TRY | 55: 0–15k/15k–30k/30k–55k/55k+; 65: 0–25k/25k–50k/50k–90k/90k+; 75: 0–40k/40k–80k/80k–140k/140k+ |

> 注：多数欧洲国家只配置了 55/65/75 三个尺寸档位（无 85 寸）。MX 同样无 85 寸档。

---

## 五、数据规格归一化流程

竞品规格来自多品牌、多语言、多格式的官网爬取数据，需经过四步归一化才能进入对阵矩阵：

### 5.1 原始规格项名 → 参数 ID 映射

- 原始数据存于 `spec_row`（日本线，`item_ja` 日文项名 + `values_json`）和 `product_spec`（零售线，`key/value` 键值对）。
- `dim_spec_item_map`（1,974 行）记录「来源表 + 来源字段 + 原始术语 → `param_id`」的映射规则，含 `match_keyword`（匹配关键词）、`extract_regex`（抽取正则）、`match_type`（匹配方式）、`brand_scope`（品牌范围）。
- 每条规则带 `status`（如 `ACTIVE`）和 `confidence`，未命中的原始项进入 `spec_untranslated`（2,513 行）待人工补录。

### 5.2 同义术语归一

- `dim_spec_normalization_map`（9 条已确认族）把不同品牌的同义术语归一到同一族：
  | 族 | 海信术语 | 竞品同义术语示例 | 归一 param_id |
  |----|---------|----------------|--------------|
  | 高刷族 | 144Hz高刷 | Refresh Rate(Samsung)、Motion Xcelerator(LG)、PANEL REFRESH RATE(TCL)、倍速対応(Sony/REGZA) | 3 (REFRESH_RATE) |
  | 分区控光族 | Mini-LED分区控光 | Dimming Zones(Samsung)、Local Dimming(LG)、XR Backlight Master Drive(Sony) | 2 (DIMMING_ZONES) |
  | 量子点族 | QLED量子点 | QD(TCL)、Quantum Dot(Samsung)、NanoCell(LG)、TRILUMINOS(Sony) | 7 (COLOR_GAMUT) |
  | 画质芯片族 | 信芯AI画质芯片 | AI Picture(Samsung)、α AI Processor(LG)、XR Processor(Sony)、AIPQ Processor(TCL) | 4 (CHIP) |
  | 智能系统族 | VIDAA | Tizen、webOS、Google TV、Android TV、My Home Screen、Roku、Fire TV | 5 (SYSTEM) |
  | 亮度族 | 峰值亮度 | Peak Brightness(Samsung)、Brightness Booster(LG)、HDR Brightness Optimizer(Sony) | 1 (PEAK_BRIGHTNESS) |
  | HDMI族 | HDMI 2.1 | HDMI 2.1(Samsung)、eARC(LG)、HDMI2.1仕様(Sony/TCL) | 8 (HDMI) |
  | HDR族 | HDR | HDR10+(Samsung)、Dolby Vision(LG)、HLG(Sony)、HDR10(Hisense) | 11 (HDR) |
  | 游戏族 | 游戏模式 | Gaming Hub(Samsung)、Game Optimizer(LG)、144Hz Game Mode Pro(TCL)、FreeSync/G-SYNC | 6 (GAMING) |
- 每条带 `status='CONFIRMED'` 和 `confirmed_by/confirmed_at`，保证可溯源。

### 5.3 跨区域匹配

- 通过 `brand_raw → adapter code`（`BRAND_TO_ADAPTER`）映射，同一品牌不同区域的适配器共享规格归一逻辑。
- backfill 第 1 层在 `spec_row` / `product_spec` 里用 18 参数的多语言精确关键词（`BACKFILL_KEYWORDS`，含日 / 英）搜索真实记录，跨区域复用已抓数据。

### 5.4 置信度分级

归一化结果写入 `fact_spec_value` 时带 `confidence`（0–1）：

| 来源 | 置信度 | 含义 |
|------|:------:|------|
| db 精确回溯命中 | ≥0.9 | 原始记录里直接找到真实值，直接采用 |
| site 重抓官网 | 0.8–0.9 | 调 spec_crawl 真实抓取官网规格页得到 |
| ai 查得 URL 供 site 抓 | 0.7–0.8 | AI 找到官网 URL 后由 spec_crawl 抓取得到 |
| 三层均未命中 | 0 | 诚实标记 `source='none'`，不编造 |

对阵矩阵中缺失参数显示为 `—`，雷达图该维度记 50 分基准线，不因缺失而扭曲对比。

---

## 六、技术亮点

### 6.1 零第三方依赖后端

`benchmark_server.py` 基于 Python 标准库 `http.server` 实现 HTTP 服务（`do_GET / do_POST / do_HEAD / do_OPTIONS`），不依赖 Flask / FastAPI 等框架；数据库用标准库 `sqlite3`；前端静态文件由内置 `_serve_static` 直接分发。整个后端可单文件运行，部署无需 pip 安装第三方包。

### 6.2 规则兜底机制

AI 分析全链路设三道降级保障，确保服务始终可用：
- **配置兜底**：未配置 `OVERSEAS_XINGHAI_API_KEY` → 直接走规则兜底，不报错，并返回 `config_needed` 提示。
- **模块兜底**：`llm_client` 不可用 → 规则兜底，返回 `fallback: True`。
- **运行兜底**：`LLMUnavailable` / `JSONDecodeError` / 任意异常 → 规则兜底。
- 规则兜底逻辑（`_fallback_analysis`）：价格低于竞品均价 85% → 优势点；`144Hz/FreeSync/240Hz/Dolby Vision` 等关键词 → 对应参数优势点；无我方型号 → 返回空数组。保证任何情况下前端都能拿到结构化结论。

### 6.3 跨区域核心码提取

- `brand_raw → adapter code` 映射（`BRAND_TO_ADAPTER`）支持海信 / Sony / REGZA / Sharp / Panasonic / TCL 六大日本线品牌的跨区域适配器复用。
- backfill 第 1 层用 18 参数的多语言关键词在原始表跨区域搜索，同一型号在 A 区域缺失的参数可从 B 区域已抓数据补全，避免重复抓取。
- `dim_product_model.model_aliases` 存储型号别名，支持同型号不同命名（如 `65U8R` vs `65U8RQ`）的跨区域归并。

### 6.4 多语言规格归一

- 18 参数均配置日文（`param_name_ja`）+ 中文（`param_name`）+ 英文（`BACKFILL_KEYWORDS`）三语关键词，适配日本线（日文官网）、欧洲线（多语种）、北美线（英文）的原始规格项名。
- `dim_spec_normalization_map` 的 `competitor_terms` 以 JSON 数组存储多品牌同义术语（如 `"Refresh Rate(Samsung)"`、`"倍速対応(Sony)"`），按 `(术语, 品牌)` 粒度归一。
- 未命中术语进入 `spec_untranslated`（2,513 行）沉淀，人工补录后回流 `dim_spec_item_map`，形成"越用越准"的闭环。
