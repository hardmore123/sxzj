# 已实现功能清单

> 本文档汇总海外电视竞品情报系统已落地的全部功能，分为**数据爬取系统**、**对阵分析系统**、**运维系统**三大板块。
> 数据来源：`海外数据爬取_部署包/CHANGELOG.md`、`compintel-deploy/FUNCTIONALITY.md`、`海外/.kiro/skills/overseas-schema-gen/SKILL.md`。

---

## 一、数据爬取系统

### 1. SPEC 分支 — 品牌官网规格爬取

- **支持 39 个品牌系列**：覆盖全球主要区域的品牌官网 lineup 页
  - Samsung：`samsung_jp` / `samsung_us` / `samsung_ca` / `samsung_de` / `samsung_fr` / `samsung_it` 等
  - Sony：`sony_jp` / `sony_us` / `sony_uk` / `sony_de` / `sony_fr` / `sony_it` 等
  - TCL：`tcl_jp` / `tcl_us` / `tcl_ca` / `tcl_de` / `tcl_fr` / `tcl_it` 等
  - LG：`lg_us` / `lg_ca` / `lg_mx`
  - Hisense：`hisense_jp` / `hisense_us` / `hisense_ca` / `hisense_mx`
  - 其他：`panasonic_jp`、`sharp_jp`、`philips_ca`、`regza_jp`
- **自动发现型号**：从品牌官网 lineup / 总览页（系列 → 型号两级结构）自动提取全部电视型号
- **规格表抓取**：按 `spec_row` 抓取品牌官网规格表，支持多语言项名识别
  - 日语（`item_ja`）、英语（`item_en`）、德语（`item_de`）、法语（`item_fr`）、意大利语（`item_it`）
- **Playwright 突破 403 WAF**：对 Sony / Samsung / LG 等返回 403 的官网，用 Playwright 渲染过 WAF（实测 lg_us / sony_ca / sony_uk 均突破）
- **持久化浏览器 profile**：通过 `OVERSEAS_USE_PROFILE=1` 复用登录态与 Cookie，避免反复触发反爬
- **断点续抓（supervisor）**：长时爬取支持进程级断点续跑，崩溃后自愈，不丢已抓数据

### 2. RETAIL 分支 — 零售站价格评论爬取

- **260 个站点**：`site` 表登记的零售站覆盖全球四区域
- **VTEX Catalog API 直采（14 站）**：南美 VTEX 平台站通过 Catalog API 直接拉取产品+价格，无需 Playwright
  - 14/14 站全部成功入库（产品 +990，价格 +979，0 失败）
- **Magento Playwright（6 站）**：用 Playwright 渲染 Magento 搜索页入库
  - hiraoka_pe / laganga_ec / raenco_pa / panafoto_pa / dismac_bo / coral_ec 等已生成契约
- **北美站点**：Amazon US/CA/MX、BestBuy US/CA、Costco CA 等
  - BestBuy 走 sitemap 确定性发现，绕开 robots 禁搜索
  - Amazon 三站统一委托 `amazon_common.py`（`parse_amazon_product()` / `price_in_region()`）
- **南美站点**：Falabella（Next.js）、Alkosto、Hiraoka、Ktronix、Lacuracao 等
- **日本站点**：Amazon JP、Kakaku 价格.com（含周度价格曲线 `weekly_kakaku_price.py`）
- **欧洲站点**：Amazon UK/DE/FR/IT/ES、Mediamarkt ES、Samsung UK、Sony UK 等
  - Amazon 欧洲五站全部入库（Playwright 原生 API 输邮编 / 搜索页直返本币价格）
- **价格 + 评论双抓**：搜索页定位 → PDP 抓价格 → 评价区抓评论
- **持久化 profile 保持登录态**：Amazon 凭据 Cookie 按国家加后缀（`at-acbca` / `at-main` / `at-acbmx`），登录判定收敛为单一硬标准（必须存在 `Sign Out`）

### 3. CompIntel 认证智能体

- **7 个认证数据源**：
  | 数据源 | 国家/地区 | 抓取方式 | 近1年数据量 |
  |---|---|---|---|
  | FCC | 美国 | Grantee Code 全量扫描 | 310 条 |
  | EPREL | 欧盟 | supplierOrTrademark 搜索 | 53 条 |
  | ENERGY STAR | 美国 | Socrata API | 38 条 |
  | ANATEL | 巴西 | 全量 ZIP-CSV 下载过滤 | 223 条 |
  | IC/ISED | 加拿大 | Spring WebFlow API | 2 条 |
  | TÜV Rheinland | 欧盟 | Certipedia 搜索页批量提取 | 89 条 |
  | VDE/CB | 欧盟 | VDE Solr API 搜索 | 136 条 |
- **EPREL supplierOrTrademark 搜索**：API 支持 `supplierOrTrademark` 过滤器，每次返回 25 条
- **ANATEL CSV 全量过滤**：53MB / 19 万行 CSV 下载后按制造商名过滤竞品
- **FCC Socrata API**：按 Grantee Code 分组翻页扫描
- **IC-ISED Spring WebFlow API**：维护 execution token + cookie，已缓存 6 品牌公司号
- **自动品牌子品牌识别**：
  - LG → HELGI、elgato
  - TCL → THOMSON
  - Sony → SONY
- **launch_warning 上市预警**：竞品新品在认证阶段即被发现（FCC/EPREL/ANATEL/ENERGY STAR/UL 五源可生成），共 129 条预警信号
- **LLM 智能解读**：单条解读 + 趋势简报，模型为 deepseek-v4-flash，LLM 不通时降级返回规则式摘要

---

## 二、对阵分析系统

### 1. 规格归一化

- **18 个标准对标参数**：定义跨品牌可比的规格维度
- **`dim_spec_item_map` 1974 条映射**：多语言规格项名 → 参数 ID
- **`dim_spec_normalization_map` 9 条同义归一**：同义规格值合并
- **`_normalize_display` 函数**：有序正则匹配，首匹配胜出（避免歧义）
- **跨区域核心码匹配**：4 轮渐进匹配，解决同型号跨区域命名差异
- **品牌核心码提取**：支持 Samsung / LG / Sony / Philips / Panasonic / TCL 六大品牌

### 2. 对阵分析 API

- **21 个国家/区域**：日本 / 北美 3 国 / 南美 5 国 / 欧洲 13 国
- **价格档位配置（bands）**：按价格区间分档对比
- **规格矩阵生成（spec_matrix）**：海信 vs 竞品的规格对阵表
- **AI 优势分析**：调用 xinghai-ultra 大模型
  - 输出优势点 / 劣势点 / 机会点 / 策略建议四类结论

### 3. 前端展示

- **国家选择器**：`optgroup` 按区域分组
- **尺寸选择器**：按屏幕尺寸筛选
- **型号列表**：海信 vs 竞品对照展示
- **规格对阵矩阵**：参数逐行对比
- **AI 分析结论展示**：自然语言呈现优势/劣势/机会/策略
- **雷达图（ECharts）**：多维规格可视化对比

---

## 三、运维系统

### 1. 监控脚本（supervisor 断点续抓）

- `monitor/supervisor_amazon_full.py`：长时全量爬取的断点续抓守护进程
- `monitor/watchdog_rebuild_amazon_full.py`：僵尸进程检测 + 自愈
- `monitor/retail_health.py`：零售站健康度巡检
- 僵尸检测判定：1h 无新 DB 写入 + 日志尾 `socket.send() raised exception` = 卡死，自动杀子进程自愈

### 2. 导出功能（Excel / JSON）

- `gen_jp_price_trend_xlsx.py`：日本线价格趋势 Excel 导出（实测 258.4 KB）
- `gen_jp_price_trend_html.py`：日本线价格趋势 HTML 报告
- `weekly_kakaku_price.py`：Kakaku 周度价格曲线
- 支持全量数据 Excel / JSON 导出

### 3. 全量重抓修复

- `rebuild_amazon_csvs.py --region`：按区域重建 CSV
- 污染修复流程：先备份受影响行 → DELETE（仅 `task='price'` 且 `price IS NOT NULL`）→ 重建
- 原则："缺价 > 错价"——宁留空不造假价

### 4. 僵尸 / 污染检测

- 僵尸进程检测：进程活着 ≠ 在干活，1h 无 DB 写入即判定卡死
- 污染检测：价格脏数据识别（配件推荐价污染主机价等，如 32A4NV 曾错为 $9.49）

### 5. 经验沉淀（cases.jsonl）

- SPEC 经验库：`spec/cases.jsonl`（52+ 条）+ `reference.md` + `playbook/`
- RETAIL 经验库：`retail/cases.jsonl`（548+ 条）+ `reference.md`（1245+ 行）+ `playbook/`
- 每接入新站强制追加一条 case，skill 越用越强
- `selfcheck.py` 自检验证沉淀完整性
