# 竞品对阵分析 — 系统设计文档

> 本文档面向下一位接手本系统的开发者，目标是让其在不读源码的情况下也能理解系统全貌、核心设计决策与扩展方式。
> 数据库版本：`overseas_v4_20260929.db`（2026-09-29 快照）。后端服务版本：`overseas v0.1.0`。

---

## 一、系统定位与目标

### 1.1 系统做什么

本系统是**海外电视产品竞品对阵分析平台**，把海信（Hisense）电视与同价位段竞品做规格对阵，并交由 AI 生成优劣势分析。

输入：用户选一个国家/区域 → 选一个屏幕尺寸 → 选一组海信+竞品型号。
输出：18 项核心规格的对标矩阵 → 六维雷达图 → AI 生成的优势/劣势/机会/策略结论。

### 1.2 核心价值

- **规格对阵可视化**：把各家品牌用不同语言（日文/英文/西语/德文…）描述的同一规格归一化成统一中文显示，形成可对比矩阵。
- **AI 优劣势解读**：规格表只是数据，AI 负责"读表说话"——指出海信领先/落后之处、可差异化的市场机会，每条结论可溯源到具体参数。
- **降级可用**：即使 AI 不可用（离线部署、代理宕机），规格对比矩阵 + 雷达图 + 规则兜底结论依然可用，核心功能永不因 AI 断线而空白。

### 1.3 覆盖范围

- **21 个国家/区域**，分 4 大模块：
  - 日本线：JP（1 国）
  - 北美线：US / CA / MX（3 国）
  - 南美线：CO / PE / CL / GT / BO（5 国）
  - 欧洲线：UK / EU / CH / SE / PL / RS / RO / NO / DK / BG / BA / TR（12 国）
- **18 个对标参数**，归入 6 个雷达维度（画质/音质/智能/游戏/连接/设计）。
- **数据库规模**（`overseas_v4_20260929.db` 实测）：
  - `dim_product_model`：6,493 个型号
  - `fact_spec_value`：52,636 条规格值，覆盖 3,451 个型号
  - `dim_spec_item_map`：1,974 条规格项名映射
  - `dim_spec_normalization_map`：9 条同义归一
  - `product`：14,959 条；`price_snapshot`：14,971 条；`spec_row`：175,218 条；`product_spec`：136,588 条

### 1.4 用户场景

```
产品经理打开浏览器
  → /api/countries        选国家（按 日本/北美/南美/欧洲 分组展示）
  → /api/sizes            选尺寸（55/65/75/85 等，按该国型号数排序）
  → /api/ai-recommend     一键"AI 智能推荐"：自动选海信型号 + 同价格带竞品 3 组
  → /api/spec             看海信 vs 竞品 18 项规格对比矩阵
  → /api/radar            看六维雷达图（画质/音质/智能/游戏/连接/设计）
  → /api/ai-analysis      AI 生成优势/劣势/机会/策略结论（可溯源到参数）
```

全流程均在一个浏览器页面内完成，后端零第三方依赖、单进程多线程。

---

## 二、系统架构

### 2.1 两层架构

| 层 | 端口 | 实现 | 依赖 | 职责 |
|---|---|---|---|---|
| 第一层：后端 API 服务 | **8090** | `overseas/benchmark_server.py` | **仅 Python 标准库**（http.server + sqlite3 + urllib） | 国家/尺寸/型号/规格/雷达/AI 入口的 HTTP 路由 + SQLite 查询 + 静态前端托管 |
| 第二层：LLM AI 代理 | **8082** | `anthropic_to_openai_proxy.py`（DSH settings 注入） | fastapi + uvicorn | 把 xinghai-ultra 的 Anthropic 协议转成 OpenAI 兼容协议，供后端调用 |

第一层是"重逻辑、轻依赖"的设计核心：只用标准库，便于离线部署到无外网的 Linux 服务器。第二层是可选增强，只在需要 AI 结论时才被调用。

### 2.2 架构图（ASCII）

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户浏览器 (http://127.0.0.1:8090)            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTP (静态页面 + /api/* JSON)
┌──────────────────────────────▼──────────────────────────────────────┐
│  第一层：benchmark_server.py (ThreadingHTTPServer, 多线程, 端口 8090) │
│                                                                      │
│  路由层:  Handler.do_GET/do_POST → _route → api_* 函数                 │
│  业务层:  api_countries / api_sizes / api_models / api_ai_recommend  │
│           api_spec_compare / api_radar / api_ai_analysis / backfill  │
│  归一化:  _DISPLAY_NORMALIZE (≈280 条正则) + _normalize_display       │
│  数据层:  sqlite3 → overseas_v4_20260929.db (18 张表)                │
│                                                                      │
│  ┌─────────────── 仅 AI 分析路径 ───────────────┐                      │
│  │ LLMClient.chat()  ──urllib──►  8082 代理      │                      │
│  │   (温度0.2, max_tokens 2048, timeout 120s)    │                      │
│  └───────────────────────┬──────────────────────┘                      │
└──────────────────────────┼────────────────────────────────────────────┘
                           │ HTTP (OpenAI 兼容 /v1)
┌──────────────────────────▼────────────────────────────────────────────┐
│  第二层：anthropic_to_openai_proxy.py (端口 8082, fastapi+uvicorn)    │
│  职责：Anthropic 协议 → OpenAI 协议转换；忽略客户端鉴权头(占位 key)    │
└──────────────────────────┬────────────────────────────────────────────┘
                           │
                           ▼
                  xinghai-ultra (大模型)
```

### 2.3 数据流

```
用户浏览器
   │ GET /api/countries                      → COUNTRIES 字典直出
   │ GET /api/sizes?region=JP                 → _region_filter → SQLite → 尺寸列表
   │ GET /api/models?region=JP&size=65       → _region_filter → SQLite → 型号+均价+价格带
   │ GET /api/ai-recommend?region=JP&size=65 → api_models → 按价格带分组推荐
   │ GET /api/spec?model_ids=1,2,3           → dim_product_model + fact_spec_value → _normalize_display → 规格矩阵
   │ GET /api/radar?model_ids=1,2,3          → api_spec_compare + 文本/数值评分 → 六维得分
   │ POST /api/ai-analysis {model_ids,region}→ api_spec_compare + api_radar → 构建prompt
   │
   │  ┌── AI 可用 ──► LLMClient.chat() → 8082 代理 → xinghai-ultra → JSON 解析
   │  └── AI 不可用 ► _fallback_analysis() 规则兜底 → fallback=true 标记
```

关键原则：**规格对比矩阵是数据真相源**。`api_radar` 复用 `api_spec_compare`，`api_ai_analysis` 同时复用 `api_spec_compare` + `api_radar`，三者共用同一份归一化后的 `spec_matrix`，保证结论可溯源。

### 2.4 代码包结构

`overseas/` 包内文件职责（本系统实际使用的）：

| 文件 | 行数级 | 职责 |
|---|---|---|
| `__init__.py` | 17 行 | 包声明，`__version__ = "0.1.0"`，分层说明 docstring |
| `benchmark_server.py` | 2178 行 | **本系统主文件**：HTTP 服务 + 全部业务 API + 规格归一化 + AI 调用 + 三层回溯入口 |
| `config.py` | ~320 行 | 配置中枢：环境变量读取 + LLM provider 默认值 + xinghai 凭据 + 超时/重试参数 |
| `llm_client.py` | ~260 行 | LLM 客户端：SDK 优先，失败回退 urllib；`configured`/`chat`/`describe`；抛 `LLMUnavailable` |
| `backfill.py` | 282 行 | 三层回溯补全：第1层(db精确搜索) → 第2层(site标记重抓) → 第3层(AI查官网URL) |
| `db.py` | — | 存储层封装（SQLite，价格/评分按时序快照只增不改），被 `backfill` 等场景调用 |
| `models.py` | — | 数据模型定义（型号/价格/评价等 ORM-like 结构） |

> 注：`overseas/` 同时承载着"海外电商采集框架"的其余能力（`scenarios/`、`sites/`、`extract/`、`fetchers/`、`infra/` 等子包），它们是数据来源侧，不在本系统运行时路径上。本系统只读 `overseas_v4_20260929.db`。

启动入口：`run_benchmark_server.py`（12 行）—— `cd` 到脚本目录、插入 `sys.path`、调 `overseas.benchmark_server.main()`。

---

## 三、核心设计决策

### 3.1 零第三方依赖（第一层）

- **为什么**：第一层后端要能离线部署到无外网的内网/机房服务器（客户环境常禁外网）。依赖 `pip install` 在离线环境不可行。
- **怎么做**：全部用 Python 3.12 标准库：
  - HTTP 服务：`http.server.ThreadingHTTPServer` + `BaseHTTPRequestHandler`（手写路由）
  - 数据库：`sqlite3`（标准库内置，单文件 DB）
  - HTTP 客户端：`urllib.request`（调 8082 代理、探活 `/models`）
  - 解析：`urllib.parse`、`json`、`re`、`mimetypes`
- **代价**：
  - 没有 ORM，所有 SQL 手写（见 `api_models` / `api_spec_compare`）。
  - 没有 Web 框架，路由在 `Handler._route` / `_route_post` 里 `if/elif` 串接。
  - 没有 async，靠 `ThreadingHTTPServer` 多线程并发。
- **收益**：单文件部署，`py -3.12 run_benchmark_server.py` 即起，无需 `requirements.txt`。

### 3.2 规格归一化引擎

这是本系统最核心、最复杂的部分。**为什么要归一化**：不同品牌用不同语言、不同格式、不同术语描述同一规格——

- 海信叫"144Hz高刷"，三星叫"Refresh Rate"，LG 叫"Motion Xcelerator"，TCL 叫"PANEL REFRESH RATE"，索尼叫"倍速対応"，REGZA 叫"倍速駆動"——它们都是"刷新率"这一族。
- 夏普的 HDMI 值是 `4（3,840×2,160 eARC/ARC対応）入力3 4K120 VRR/ALLM`，要归一成 `HDMI 2.1×4口 (4K120Hz/eARC/VRR/ALLM)`。

**两层映射设计**（解决"规格项名"不一致）：

1. `dim_spec_item_map`：**项名 → param_id**。解决"画素数"（夏普日文）vs "Resolution"（零售站英文）vs "分辨率"（中文）都是 `RESOLUTION` 参数的问题。
   - `source_table=spec_row, source_field=item_ja`：品牌官网抓到的规格项名（日文为主）→ 802 条 CONFIRMED + 425 条 AUTO
   - `source_table=product_spec, source_field=key`：零售站抓到的规格项名 → 690 条 EXACT + 57 条 LIKE
   - `match_type`：EXACT（精确匹配）/ LIKE（模糊匹配，子串命中）
   - `status`：AUTO（爬取时自动生成）/ CONFIRMED（人工确认）
2. `dim_spec_normalization_map`：**同义归一**（9 条，覆盖 9 个核心参数）。每条含海信术语 + 竞品术语 JSON 数组 + 归一化族名。用于前端显示时统一对标族名（如"144Hz高刷族"、"Mini-LED分区控光族"）。详见第七章。

**`_DISPLAY_NORMALIZE` 正则规则引擎**：约 280 条正则的有序字典，**首匹配胜出**（`for pattern, replacement in _DISPLAY_NORMALIZE.items()` + `re.search`，命中即 `re.sub` 返回）。设计原则：**最具体的规则放前面，避免被宽泛规则先匹配**（例：Sharp HDMI 规则含 `3,840×2,160` 字样，必须排在通用分辨率规则前；REGZA 专用分区控光规则必须排在通用 Mini LED 规则前）。详见第七章 7.1。

**跨区域核心码匹配**：同品牌同系列的不同区域型号（如日本 `4T-C65HN2` 与海外 `65C655`），通过提取"核心码"（去除品牌前缀和区域后缀）匹配，从一个区域复制规格到另一个区域。详见第七章 7.4。

### 3.3 价格档位设计

- **21 国 × 3-4 尺寸 × 4 档位**。每国独立配置价格区间，按当地货币：
  - 4 档位：`入门` / `中端` / `高端` / `旗舰`
  - 区间为 `(下限, 上限)` 元组，`旗舰` 上限为 `9e9`（正无穷）
- **各国尺寸覆盖不同**：JP/US/CA 有 55/65/75/85 四尺寸；多数国家只有 55/65/75 三尺寸（见 `COUNTRIES` 字典各 `bands` 键）。
- **为什么不自动计算**：各国消费水平、汇率、税制不同，自动用 USD 换算会失真（如哥伦比亚 COP 100 万、智利 CLP 50 万、塞尔维亚 RSD 4 万，都是"入门"但数值天差地别）。人工配置当地实际零售价区间更准。
- **价格档位用法**：`api_models` 为每个型号算均价后，查 `bands[size]` 找落入哪档，写入 `price_band`。`api_ai_recommend` 用 `price_band` 做同档竞品匹配。

### 3.4 AI 分析降级策略

AI 分析有三层降级，保证核心功能始终可用：

| 情况 | 触发条件 | 行为 | 响应标记 |
|---|---|---|---|
| **正常** | `LLMClient.configured == True` 且调用成功 | xinghai-ultra 生成 JSON，解析出优势/劣势/机会/策略 | 无 `fallback` 字段 |
| **未配置/模块缺失** | `import llm_client` 失败，或 `client.configured == False` | 走 `_fallback_analysis()` 规则兜底 | `fallback: true` + `config_needed` 提示 |
| **调用失败/解析失败** | 抛 `LLMUnavailable` 或 `JSONDecodeError` | 走 `_fallback_analysis()` + 附 `error` 原因 | `fallback: true` + `error` 字段 |

- **为什么不直接报错**：保证核心功能（规格对比矩阵 + 雷达图）始终可用。AI 只是增强，规格对比本身不依赖 AI。
- **超时/重试**：`LLM_TIMEOUT=120s`（`config.py` 默认），`LLM_MAX_RETRIES=2`。`llm_client` 对 408/425/429/500/502/503/504 状态码重试。
- **`_fallback_analysis` 逻辑**：价格对比（我方低于均价 85% → 价格领先优势点）+ 关键参数领先（144Hz/FreeSync/240Hz）+ 系统劣势（VIDAA 生态弱于 Google TV/Tizen）+ 固定策略建议。详见第八章 8.4。

### 3.5 区域过滤策略

`_region_filter(region)` 根据 region 返回 `dim_product_model` 的 WHERE 条件片段：

- **日本线（JP）**：`brand_raw LIKE '%日本%' OR (海信型号且有 JPY 价格)`。
  - 为什么特殊：日本数据来源是 Kakaku（价格比价站）+ 各品牌日本官网（spec），竞品的 `brand_raw` 带后缀 `_jp` 标记"日本线"，海信日本线型号靠"有 JPY 价格"识别。
- **非日本线**：`model_code IN (SELECT DISTINCT p.model FROM product JOIN price_snapshot WHERE ps.currency = 该国货币)`。
  - 即"该型号有该国货币的零售价才显示"。严格按货币过滤，**无该货币价格则跳过**（`缺价 > 错价` 原则，见 `api_models` 中 `if price is None: continue`）。
- **为什么不同**：日本数据来源特殊（Kakaku + 品牌官网），非日本线数据来自各国零售站，按货币过滤最可靠。早期版本曾对日本线 fallback 到任意货币，会把 COP/PEN/USD 当 JPY 显示，已修正为严格只取 JPY。

---

## 四、18 个对标参数设计

18 个参数定义在 `dim_spec_parameter` 表（实测 param_id 1-18），并在 `benchmark_server.py` 的 `PARAM_ORDER` / `PARAM_NAMES_ZH` / `PARAM_CATEGORY` / `RADAR_DIMS` 中维护。

### 4.1 参数总表

| param_id | param_code | 中文名 | 日文/英文名 | 单位 | 数值型 | 高优(1)/低优(0) | 分类 | 雷达维度 |
|---|---|---|---|---|---|---|---|---|
| 1 | PEAK_BRIGHTNESS | 峰值亮度 | ピーク輝度 / Peak Brightness | nits | 1 | 1 高优 | 画质 | 画质 |
| 2 | DIMMING_ZONES | 分区控光 | ローカルディミング / Local Dimming | zones | 1 | 1 高优 | 画质 | 画质 |
| 3 | REFRESH_RATE | 刷新率 | 倍速対応 / Refresh Rate | Hz | 1 | 1 高优 | 画质 | 画质/游戏 |
| 4 | CHIP | 画质芯片 | 画像処理エンジン | (无) | 0 | (None) | 画质 | 画质 |
| 5 | SYSTEM | 智能系统 | スマートTV / Smart OS | (无) | 0 | (None) | 智能 | 智能 |
| 6 | GAMING | 游戏特性 | ゲーム機能 / Gaming | (无) | 0 | 1 高优 | 游戏 | 游戏 |
| 7 | COLOR_GAMUT | 色域 | 色域 / Color Gamut | % | 1 | 1 高优 | 画质 | 画质 |
| 8 | HDMI | HDMI 2.1 | HDMI端子 | 口 | 0 | 1 高优 | 连接 | 连接/游戏 |
| 9 | SCREEN_SIZE | 屏幕尺寸 | 画面サイズ / Screen Size | inch | 1 | **0 低优** | 设计 | 设计 |
| 10 | RESOLUTION | 分辨率 | 画素数 / Resolution | (无) | 0 | 1 高优 | 画质 | 画质 |
| 11 | HDR | HDR格式 | HDR対応 / HDR Format | (无) | 0 | 1 高优 | 画质 | 画质 |
| 12 | CONTRAST | 对比度 | コントラスト比 | :1 | 1 | 1 高优 | 画质 | 画质 |
| 13 | WIFI | Wi-Fi | 無線LAN / Wi-Fi | (无) | 0 | 1 高优 | 连接 | 连接 |
| 14 | BLUETOOTH | 蓝牙 | Bluetooth | (无) | 0 | 1 高优 | 连接 | 连接 |
| 15 | DOLBY_ATMOS | 杜比全景声 | ドルビーアトモス / Dolby Atmos | (无) | 0 | 1 高优 | 音质 | 音质 |
| 16 | POWER_CONSUMPTION | 功耗 | 消費電力 / Power | W | 1 | **0 低优** | 设计 | 设计 |
| 17 | WEIGHT | 重量 | 質量 / Weight | kg | 1 | **0 低优** | 设计 | 设计 |
| 18 | DIMENSIONS | 尺寸 | 外形寸法 / Dimensions | mm | 0 | **0 低优** | 设计 | 设计 |

> `higher_is_better`：`1` = 越大越好（如亮度、分区数）；`0` = 越小越好（如功耗、重量、尺寸）；`None` = 文本型无方向（如芯片、系统）。雷达图评分时，低优参数用 `(mx-num)/(mx-mn)` 反向归一。

### 4.2 雷达图 6 维度分组

`RADAR_DIMS` 定义（注意同一参数可属多维度，雷达评分时会被多次计入）：

| 雷达维度 | 包含参数 | 评分方式 |
|---|---|---|
| **画质** | PEAK_BRIGHTNESS, DIMMING_ZONES, REFRESH_RATE, COLOR_GAMUT, HDR, CHIP, RESOLUTION, CONTRAST | 数值型相对评分 + 文本特征评分 |
| **音质** | DOLBY_ATMOS | 文本特征评分（Dolby Atmos=80，有值=60） |
| **智能** | SYSTEM | 文本特征评分（Google/Android TV=80，Tizen/webOS=78，VIDAA=55） |
| **游戏** | GAMING, REFRESH_RATE, HDMI | FreeSync/240Hz=90，144Hz=85，Game=72 |
| **连接** | HDMI, WIFI, BLUETOOTH | 主要靠数值型 + 文本 |
| **设计** | POWER_CONSUMPTION, SCREEN_SIZE, WEIGHT, DIMENSIONS | 数值型相对评分 |

每个维度的得分 = 该维度内各参数得分的均值（`round(total/cnt)`，无值时记 50 分基准）。数值型参数用**相对评分**：`60 + 35 * (归一化位置)`，其中高优参数 `(num-mn)/(mx-mn)`、低优参数 `(mx-num)/(mx-mn)`，基准 60、最高 95。

### 4.3 各参数归一化逻辑说明

- **PEAK_BRIGHTNESS（峰值亮度）**：Sony 的 `ルミナンスブースター` 系列规则提取；纯数字加 ` nits`。海信"峰值亮度族"（normalization map #6）。
- **DIMMING_ZONES（分区控光）**：REGZA 专用规则（`新開発 高輝度 ファイン Mini LED` 等）必须在通用 Mini LED 前；`Mini-LED分区控光族`（#2）。
- **REFRESH_RATE（刷新率）**：REGZA/Sony 的 `●120Hz倍速` 系列规则在前；TCL 的 `倍速120Hz/4K VRR 288Hz` 提取双值；纯数字加 `Hz`。"高刷族"（#1，144Hz高刷 vs Motion Xcelerator 等）。
- **CHIP（画质芯片）**：HI-VIEW AI引擎 / REGZA引擎 / Medalist S6 / AiPQ / XR 处理器等，按品牌映射。"画质芯片族"（#4）。
- **SYSTEM（智能系统）**：VIDAA / Google TV / Android TV / Fire TV / Tizen / webOS / VIERA Link，单值映射。"智能系统族"（#5）。
- **GAMING（游戏特性）**：Game Master + FreeSync Premium Pro + DLG 多特征组合规则；TCL HDMI 规则必须在 VRR/ALLM 通用规则前。"游戏族"（#9）。
- **COLOR_GAMUT（色域）**：量子点/QLED/NanoCell/TRILUMINOS/Rich Color 等。"量子点族"（#3）。
- **HDMI（HDMI 2.1）**：最复杂。Sharp 的长串 `（3,840×2,160 eARC/ARC）入力3 4K120 VRR/ALLM` 多规则拆解；TCL 的 `4個(VRR/ALLM...)` 系规则；Sony 的 `4ea(...)` / `Yes(eARC/ARC)`；LG 的 `eARC (HDMI 2)`。纯数字加 `口`。"HDMI族"（#7）。
- **SCREEN_SIZE（屏幕尺寸）**：`65V型`/`65型` → `65寸`；纯数字加 `寸`。低优（越大越占空间）。
- **RESOLUTION（分辨率）**：`3,840×2,160` → `4K (3840×2160)`；`3840x2160(4K UHD)` → `4K (3840×2160)`。Sharp HDMI 含分辨率字样的规则已在前面处理，这里只匹配纯分辨率值。
- **HDR（HDR格式）**：HDR10/Dolby Vision/HLG 的多种组合归一到 `HDR10+HLG+Dolby Vision` 或带 `IQ` 后缀。多规则按具体度排序。"HDR族"（#8）。
- **CONTRAST（对比度）**：RGB独立分区控光 / Mini LED分区控光 / 真黑控光等 REGZA 专用；纯数字加 `:1`。
- **WIFI（Wi-Fi）**：`IEEE802.11ax` → `Wi-Fi 802.11ax`；REGZA 带 `●` 前缀的规则在前；`双频 5G+2.4G`。
- **BLUETOOTH（蓝牙）**：`Bluetooth内蔵` → `支持`；版本号 `x.y` → `Bluetooth x.y`。
- **DOLBY_ATMOS（杜比全景声）**：Dolby Atmos / ドルビーアトモス / Dolby Audio / DTS:X 统一。
- **POWER_CONSUMPTION（功耗）**：`約239W（0.5W）` → `239W`；`260 W / ...` → `260W`。低优。
- **WEIGHT（重量）**：`①約28.5kg ②約23.5kg` → `28.5kg`；纯数字加 `kg`。低优。
- **DIMENSIONS（尺寸）**：最特殊。`mm_to_cm` 标记触发 `_normalize_display` 里的特殊处理——提取 mm 数值除以 10 转 cm。支持幅×高さ×対角、幅×高さ×奥行、三维 mm、二维 mm 多种格式。Sony 系列共享值 `【K-65XR50】144.7 x 81.7 x 5.7[...]` 按当前型号提取。低优。

---

## 五、21 个国家/区域配置

`COUNTRIES` 字典定义在 `benchmark_server.py` 顶部（28-120 行）。每国：`code / name / currency / flag / group / bands`。

### 5.1 国家总表

| 代码 | 中文名 | 货币 | 旗帜 | 所属模块 | 可用尺寸 |
|---|---|---|---|---|---|
| JP | 日本 | JPY | 🇯🇵 | 日本 | 55/65/75/85 |
| US | 美国 | USD | 🇺🇸 | 北美 | 55/65/75/85 |
| CA | 加拿大 | CAD | 🇨🇦 | 北美 | 55/65/75/85 |
| MX | 墨西哥 | MXN | 🇲🇽 | 北美 | 55/65/75 |
| CO | 哥伦比亚 | COP | 🇨🇴 | 南美 | 55/65/75 |
| PE | 秘鲁 | PEN | 🇵🇪 | 南美 | 55/65/75 |
| CL | 智利 | CLP | 🇨🇱 | 南美 | 55/65/75 |
| GT | 危地马拉 | GTQ | 🇬🇹 | 南美 | 55/65/75 |
| BO | 玻利维亚 | BOB | 🇧🇴 | 南美 | 55/65/75 |
| UK | 英国 | GBP | 🇬🇧 | 欧洲 | 55/65/75 |
| EU | 欧洲(欧元区) | EUR | 🇪🇺 | 欧洲 | 55/65/75 |
| CH | 瑞士 | CHF | 🇨🇭 | 欧洲 | 55/65/75 |
| SE | 瑞典 | SEK | 🇸🇪 | 欧洲 | 55/65/75 |
| PL | 波兰 | PLN | 🇵🇱 | 欧洲 | 55/65/75 |
| RS | 塞尔维亚 | RSD | 🇷🇸 | 欧洲 | 55/65/75 |
| RO | 罗马尼亚 | RON | 🇷🇴 | 欧洲 | 55/65/75 |
| NO | 挪威 | NOK | 🇳🇴 | 欧洲 | 55/65/75 |
| DK | 丹麦 | DKK | 🇩🇰 | 欧洲 | 55/65/75 |
| BG | 保加利亚 | BGN | 🇧🇬 | 欧洲 | 55/65/75 |
| BA | 波黑 | BAM | 🇧🇦 | 欧洲 | 55/65/75 |
| TR | 土耳其 | TRY | 🇹🇷 | 欧洲 | 55/65/75 |

### 5.2 价格档位配置（示例）

每国 `bands` 字典：`{尺寸: {档位: (下限, 上限)}}`，货币单位即该国 currency。

**日本线（JP，JPY）**——四尺寸：

| 尺寸 | 入门 | 中端 | 高端 | 旗舰 |
|---|---|---|---|---|
| 55 | 0–100,000 | 100,000–200,000 | 200,000–350,000 | 350,000+ |
| 65 | 0–150,000 | 150,000–300,000 | 300,000–500,000 | 500,000+ |
| 75 | 0–200,000 | 200,000–400,000 | 400,000–600,000 | 600,000+ |
| 85 | 0–300,000 | 300,000–500,000 | 500,000–800,000 | 800,000+ |

**北美线（US，USD）**——四尺寸：

| 尺寸 | 入门 | 中端 | 高端 | 旗舰 |
|---|---|---|---|---|
| 55 | 0–400 | 400–900 | 900–1,800 | 1,800+ |
| 65 | 0–600 | 600–1,200 | 1,200–2,500 | 2,500+ |
| 75 | 0–800 | 800–1,800 | 1,800–3,500 | 3,500+ |
| 85 | 0–1,200 | 1,200–2,500 | 2,500–5,000 | 5,000+ |

**南美线（CO，COP）**——三尺寸：

| 尺寸 | 入门 | 中端 | 高端 | 旗舰 |
|---|---|---|---|---|
| 55 | 0–1,500,000 | 1,500,000–3,000,000 | 3,000,000–5,000,000 | 5,000,000+ |
| 65 | 0–2,500,000 | 2,500,000–5,000,000 | 5,000,000–8,000,000 | 8,000,000+ |
| 75 | 0–3,500,000 | 3,500,000–7,000,000 | 7,000,000–12,000,000 | 12,000,000+ |

**欧洲线（UK，GBP / EU，EUR）**——三尺寸（以 EU 为例）：

| 尺寸 | 入门 | 中端 | 高端 | 旗舰 |
|---|---|---|---|---|
| 55 | 0–400 | 400–800 | 800–1,600 | 1,600+ |
| 65 | 0–600 | 600–1,200 | 1,200–2,400 | 2,400+ |
| 75 | 0–800 | 800–1,800 | 1,800–3,500 | 3,500+ |

> 其余国家档位详见 `COUNTRIES` 字典源码（`benchmark_server.py:28-120`），结构同上，数值按当地消费水平调整（如 RS 用 RSD 4 万–30 万、DK 用 DKK 4 千–3.2 万、TR 用 TRY 1.5 万–14 万）。

### 5.3 分组说明

- **日本线**：1 国。数据来源 Kakaku（价格）+ 各品牌日本官网（spec），按 `brand_raw LIKE '%日本%'` + JPY 价格过滤。
- **北美线**：3 国。零售站数据，按货币（USD/CAD/MXN）过滤。
- **南美线**：5 国。零售站数据，按货币（COP/PEN/CLP/GTQ/BOB）过滤。
- **欧洲线**：12 国。零售站数据，按货币（GBP/EUR/CHF/SEK/PLN/RSD/RON/NOK/DKK/BGN/BAM/TRY）过滤。其中 TR 在数据上归"欧洲"模块但实际为亚洲国家。

---

## 六、数据库设计概览

数据库文件：`data/overseas_v4_20260929.db`（SQLite，单文件）。共 18 张业务表 + `sqlite_sequence` + 一张 `dim_product_model_bak` 备份。

### 6.1 四层架构

| 层 | 表数 | 表名 | 职责 |
|---|---|---|---|
| **原始爬取层** | 12 | `product` / `price_snapshot` / `price_shop_snapshot` / `product_spec` / `spec_row` / `spec_series` / `spec_series_status` / `review` / `review_summary` / `ranking_snapshot` / `selling_point` / `selling_point_status` | 各国零售站与品牌官网抓到的原始数据，价格按时序快照只增不改 |
| **维度层** | 2 | `dim_product_model` / `dim_spec_parameter` | 型号维度（brand/size/our_brand）+ 参数维度（param_id/code/unit/数值型/高优低优） |
| **映射层** | 2 | `dim_spec_item_map` / `dim_spec_normalization_map` | 规格项名→param_id 映射 + 同义归一族 |
| **事实层** | 2 | `fact_spec_value` / `fact_benchmark_comparison` | 归一化后的规格值事实表 + 对比结论表 |

辅助表：`crawl_run` / `crawl_error` / `raw_page` / `spec_entry_audit` / `spec_untranslated` / `site` / `kakaku_task_status` / `kakaku_thread_state` / `retail_task_status` / `schema_version`（运维/审计用）。

### 6.2 核心表关系（ASCII）

```
                      dim_spec_parameter (18 行, param_id 主键)
                              │ param_id
                              │
   dim_product_model ─────────┼──────────► fact_spec_value
   (6493 行, model_id)         │           (52636 行)
        │ model_id              │           model_id + param_id 联合
        │                       │           value_text/value_numeric/source_term/confidence
        │                       │
        │ model_code            ▼
        │              dim_spec_item_map (1974 行)
        │              source_table+source_field+item → param_id
        │
        │ model_code
        ▼
   product (14959 行) ──► price_snapshot (14971 行)   product_spec (136588 行)
        │ id                    │ product_id              │ product_id
        │                       │ currency/price/date     │ key/value
        │
        └─► spec_row (175218 行, series_id) ──► spec_series (系列/品牌)

   dim_spec_normalization_map (9 行) ── 独立，按 param_id 关联，前端显示归一族名用
   fact_benchmark_comparison ── 对比结论存储（当前主走 API 实时生成，此表为预留/历史）
```

**关键关系链**（规格对比查询路径）：
`api_spec_compare(model_ids)` → 对每个 `model_id` 查 `dim_product_model` 得型号信息 → 对每个 `param_code` 查 `dim_spec_parameter` 得 `param_id` → 用 `model_id + param_id` 查 `fact_spec_value` 得原始值 → `_normalize_display` 归一化显示。

### 6.3 数据量（实测 `overseas_v4_20260929.db`）

| 表 | 行数 | 说明 |
|---|---|---|
| `dim_product_model` | 6,493 | 维度型号表（含 brand/size/our_brand/brand_raw） |
| `dim_spec_parameter` | 18 | 18 个对标参数定义 |
| `dim_spec_item_map` | 1,974 | 规格项名→param_id（spec_row 1227 + product_spec 747） |
| `dim_spec_normalization_map` | 9 | 9 个归一族（高刷/分区/量子点/芯片/系统/亮度/HDMI/HDR/游戏） |
| `fact_spec_value` | 52,636 | 归一化规格值事实，覆盖 3,451 个型号 |
| `fact_benchmark_comparison` | — | 对比结论（预留） |
| `product` | 14,959 | 原始产品 |
| `price_snapshot` | 14,971 | 价格快照（按时序只增不改） |
| `product_spec` | 136,588 | 零售站规格键值 |
| `spec_row` | 175,218 | 品牌官网规格行（按 series_id 分组） |
| `spec_series` | — | 系列（品牌×系列名） |

> 覆盖率：`fact_spec_value` 覆盖 3,451 / 6,493 ≈ **53%** 型号有规格数据；其余约 47% 型号只有价格无规格（可通过 `/api/backfill` 三层回溯补全）。

---

## 七、归一化引擎详细设计

### 7.1 `_DISPLAY_NORMALIZE` 正则规则

定义在 `benchmark_server.py:169-446+`，是一个**有序字典**（Python 3.7+ dict 保序），约 280 条 `(正则模式, 替换文本)` 对。匹配算法在 `_normalize_display()`（1247 行）中：

```python
for pattern, replacement in _DISPLAY_NORMALIZE.items():
    if re.search(pattern, text, flags=re.I):
        if replacement == 'mm_to_cm':
            # 特殊处理：提取 mm 数值除以 10 转 cm（多种子格式）
            ...
        new = re.sub(pattern, replacement, text, flags=re.I)
        return new if len(new) <= 60 else new[:58] + '…'
```

**首匹配胜出**，所以规则顺序极其关键。

**设计原则：最具体的规则放前面，避免被宽泛规则先匹配。**

按品牌分组的专用规则区块（顺序即优先级）：

1. **Sharp HDMI**（含 `3,840×2,160` 字样的多条规则，必须排在通用分辨率规则前）：把夏普的长串 HDMI 描述拆解成 `HDMI 2.1×N口 (4K120Hz/eARC/VRR/ALLM)` 等。
2. **海信/TCL HDMI**：`HDMI 2.1×N / HDMI 2.0×M` 双频格式；TCL 的 `N個(VRR/ALLM...)` 系列规则。
3. **分辨率**：`3,840×2,160` → `4K (3840×2160)`（Sharp HDMI 已在前面处理，这里只匹配纯分辨率值）。
4. **功耗/重量/尺寸/HDR/刷新率/音响/屏幕尺寸/背光分区/画质芯片/色域/游戏/智能系统/杜比/Wi-Fi/蓝牙/对比度**：各参数的专用规则块。
5. **REGZA 分区控光**（`新開発 高輝度 ファイン Mini LED` 等）必须在通用 Mini LED 规则前。
6. **Sony 专用规则**：画质芯片（`プロセッサー「XR」` → `XR处理器`）、色域（`トリルミナス` → `特丽魅彩`）、峰值亮度（`ルミナンスブースター`）、HDMI（`4ea(...)` / `Yes(eARC/ARC)`）。
7. **通用兜底规则**：放在最后，处理未命中的宽泛情况。

**特殊标记 `mm_to_cm`**：当 replacement 为 `mm_to_cm` 时，不走 `re.sub`，而是进 `_normalize_display` 里的特殊分支——提取 mm 数值除以 10 转 cm。支持子格式：
- `幅1428 x 高さ804 対角 1639` → `142.8×80.4cm`
- `幅1453mm / 高さ892mm / 奥行303mm` → `145.3×30.3×89.2cm`
- 通用三维/二维 `NNN x NNN x NNN mm` 格式

**纯数字兜底**（在所有正则未命中后）：按 `param_code` 加单位——`WEIGHT` 加 `kg`、`HDMI` 加 `口`、`PEAK_BRIGHTNESS` 加 ` nits`、`BLUETOOTH` 加 `Bluetooth ` 前缀、`CONTRAST` 加 `:1`、`SCREEN_SIZE` 加 `寸`、`REFRESH_RATE` 加 `Hz`。

**Sony 系列共享值提取**：`model_code` 非空且文本含 `【` 且含当前型号时，先用正则提取当前型号的值（`【K-65XR50】144.7 x 81.7 x 5.7` → `144.7 x 81.7 x 5.7`），再走归一化。

### 7.2 `dim_spec_item_map` 映射

1,974 条，解决"规格项名 → param_id"的问题。两种来源：

| source_table | source_field | 含义 | match_type 分布 | status 分布 |
|---|---|---|---|---|
| `spec_row` | `item_ja` | 品牌官网抓到的规格项名（日文为主，如"画素数"、"消費電力"） | EXACT 1227 | AUTO 425 / CONFIRMED 802 |
| `product_spec` | `key` | 零售站抓到的规格项名（如"Resolution"、"Peak Brightness"） | EXACT 690 / LIKE 57 | CONFIRMED 747 |

- **`match_type`**：
  - `EXACT`：精确匹配，`item == key` 才命中。
  - `LIKE`：模糊匹配，`key` 含某子串即命中（处理零售站规格项名多变体，如"Peak Brightness (nits)" 含 "Peak Brightness"）。
- **`status`**：
  - `AUTO`：爬取时自动生成（爬虫遇到新规格项名，按关键词猜 param_id 后写入，待人工确认）。
  - `CONFIRMED`：人工确认映射正确，可信。
- **用途**：爬虫把 `spec_row.values_json` / `product_spec` 的原始项名，经此表映射成 `param_id`，写入 `fact_spec_value`。本系统运行时**直接读 `fact_spec_value`**，不实时走此表；此表是数据生产侧的映射资产。

### 7.3 `dim_spec_normalization_map` 归一化

9 条记录，覆盖 9 个核心参数。每条结构：

```
map_id | param_id | hisense_term（海信术语/族名） | competitor_terms（竞品术语 JSON 数组） | normalized_category（归一族名） | status
```

| map_id | param_id | 参数 | hisense_term | normalized_category | 竞品术语示例 |
|---|---|---|---|---|---|
| 1 | 3 | REFRESH_RATE | 144Hz高刷 | 高刷族 | Refresh Rate(Samsung)/Motion Xcelerator(LG)/倍速対応(Sony)/倍速駆動(TCL) |
| 2 | 2 | DIMMING_ZONES | Mini-LED分区控光 | 分区控光族 | Local Dimming(LG)/XR Backlight Master Drive(Sony)/ローカルディミング(TCL) |
| 3 | 7 | COLOR_GAMUT | QLED量子点 | 量子点族 | Quantum Dot(Samsung)/NanoCell(LG)/TRILUMINOS(Sony)/量子ドット(REGZA) |
| 4 | 4 | CHIP | 信芯AI画质芯片 | 画质芯片族 | AI Picture(Samsung)/α AI Processor(LG)/XR Processor(Sony)/AIPQ Processor(TCL) |
| 5 | 5 | SYSTEM | VIDAA | 智能系统族 | Tizen(Samsung)/webOS(LG)/Google TV(Sony/TCL)/Fire TV(Amazon)/Roku(Hisense US) |
| 6 | 1 | PEAK_BRIGHTNESS | 峰值亮度 | 亮度族 | Peak Brightness(nits)(Samsung)/Brightness Booster(LG)/高輝度技術(Sony) |
| 7 | 8 | HDMI | HDMI 2.1 | HDMI族 | HDMI 2.1(Samsung)/eARC(LG)/HDMI2.1仕様(Sony)/HDMI®2.1規格対応機能(REGZA) |
| 8 | 11 | HDR | HDR | HDR族 | HDR10+(Samsung)/Dolby Vision(LG)/HLG(Sony)/HDR10/HLG対応(Panasonic) |
| 9 | 6 | GAMING | 游戏模式 | 游戏族 | Gaming Hub(Samsung)/Game Optimizer(LG)/Game Mode(Sony)/FreeSync(Samsung)/瞬速ゲームモード(REGZA) |

全部 `status=CONFIRMED`，`confirmed_by=benchmark_maintain`，`confirmed_at=2026-10-10`。

**用途**：前端显示规格对比矩阵时，把各家术语统一显示为"族名"（如把 `Motion Xcelerator` 和 `144Hz高刷` 都归到"高刷族"），让用户一眼看出这是同一维度的对标。运行时 AI prompt 也用族名让 AI 理解对标关系。

### 7.4 跨区域核心码匹配

用于同品牌同系列、不同区域型号之间复制规格数据（如日本 `4T-C65HN2` 与海外某型号同属夏普 AQUOS 某 系列）。

- **核心码提取**：品牌前缀去除（`K-`/`4T-`/`TV-` 等）+ 区域后缀去除（`_jp`/`_us`/`_uk` 等），保留中间型号核心（如 `65XR50`、`65HN2`）。
- **4 轮渐进匹配**（在 `backfill._backfill_from_db` 及关联逻辑中，配合 `infer_region` / `_infer_adapter_code`）：
  1. **精确匹配**：同 model_code 直接命中（confidence 最高）。
  2. **品牌 + 尺寸 + 核心码**：同品牌、同尺寸、核心码一致的跨区域型号复制规格。
  3. **同尺寸**：同品牌同尺寸的近似型号（兜底，置信度低）。
  4. **同品牌**：同品牌的任意型号（最后兜底）。
- **confidence 置信度**（写回 `fact_spec_value.confidence`）：
  - `0.95`：第1层 db 精确搜索命中（`_backfill_from_db`，原始 spec_row/product_spec 命中关键词）
  - `0.5`：第2层 site 标记可重抓（`_backfill_from_site`，返回 `(待重抓)`，需触发 `spec_crawl`）
  - `0.4`：跨区域复制 / 模糊匹配（核心码匹配回溯）
  - `1.0`：直接抓取（官网规格页实时抓取并归一，最高可信）
  - `0`：AI 未找到可确认 URL（不写回，避免编造）

> 第3层 `_backfill_from_ai` 的 AI **只查官网 URL，不产出规格值**——找到 URL 后由 `spec_crawl` 真实抓取，避免 AI 编造数据（严格规则：不确定则返回 `未找到|0|...`，confidence=0）。

---

## 八、AI 分析设计

入口：`api_ai_analysis(model_ids, region=None)`（1644 行）。

### 8.1 请求构建

1. 调 `api_spec_compare(model_ids, region)` 拿规格矩阵，调 `api_radar(model_ids)` 拿雷达得分。
2. 识别"我方"型号：`next((m for m in models if m['is_our']), models[0])`。
3. 构建 `summary` 文本：
   - 区域行：`区域: JP`
   - 每型号一行：`我方(海信) Hisense 65U8KQ 价格280000JPY` / `竞品 Sony K-65XR50 价格...`
   - 规格对比行：取 `PARAM_ORDER[:12]`（前12个核心参数），每行 `参数名: 型号A=值 | 型号B=值`。
4. 拼 prompt（见下节）。

### 8.2 xinghai-ultra 调用

prompt 模板（`benchmark_server.py:1670`）：

```
你是电视产品竞品分析专家。基于以下规格对比数据，生成"我方(海信)"对阵竞品的结论。
要求输出 JSON（不要 markdown 代码块），格式:
{
  "advantage_points": [{"point":"...","evidence":"...","confidence":0.9}],
  "disadvantage_points": [{"point":"...","evidence":"...","confidence":0.85}],
  "opportunity_points": [{"point":"...","evidence":"...","confidence":0.8}],
  "strategy_suggestion": "一句话策略建议"
}
优势点：我方领先的参数（数值高/功能全）；劣势点：我方落后之处；机会点：可差异化或补位的市场机会。
每条要可溯源到具体参数或价格。置信度0-1。

规格对比数据:
{summary}
```

调用参数：`client.chat([{"role":"user","content":prompt}], temperature=0.2, max_tokens=2048)`。
- 超时：`LLM_TIMEOUT=120s`（`config.py`）。
- 重试：`LLM_MAX_RETRIES=2`，对 408/425/429/500/502/503/504 重试（`llm_client._RETRYABLE_STATUS`）。
- 非流式：`_setup_xinghai` 强制 `LLM_STREAM=False`（本地代理非流式更稳）。
- SSL 关闭：`LLM_SSL_VERIFY=False`（本地 HTTP 代理无需校验）。
- 客户端实现：`LLMClient` 优先用 SDK（若装了 openai/httpx），失败回退 `urllib`（`_chat_urllib`）。

### 8.3 响应解析

```python
text = result_text.strip()
if text.startswith('```'):  # 容错：去掉可能的 ```json 包裹
    text = text.split('\n', 1)[-1].rsplit('```', 1)[0].strip()
ai_result = json.loads(text)
return {'ai_conclusion': ai_result, 'model': client.describe(), 'summary': summary}
```

返回结构：
- `ai_conclusion`：解析后的 JSON，含 `advantage_points` / `disadvantage_points` / `opportunity_points`（每条 `{point, evidence, confidence}`）+ `strategy_suggestion`。
- `model`：`client.describe()`（如 `xinghai-ultra @ http://127.0.0.1:8082/v1`）。
- `summary`：喂给 AI 的规格对比摘要（前端可展示，便于核对）。

### 8.4 规则兜底（`_fallback_analysis`）

AI 不可用时调用（1717 行）。逻辑：

1. **价格对比**：算竞品均价，我方价格 < 均价×0.85 → 生成"价格领先"优势点（confidence 0.95）+ "价格带卡位"机会点（0.85）。
2. **关键参数领先**：遍历 `REFRESH_RATE/GAMING/HDR`，若我方显示含 `144Hz`/`FreeSync`/`240Hz` → 生成该参数领先优势点（0.9）。
3. **系统劣势**：若我方系统是 `VIDAA` → 生成"智能系统生态弱于 Google TV/Tizen 系"劣势点（0.85）+ "VIDAA 本地化机会"机会点（0.75）。
4. **固定策略建议**：`以画质+游戏+性价比组合应对，补强智能生态`。

返回带 `fallback: true` 标记，前端据此提示"规则兜底结论，建议配置 AI 后重试"。

---

## 九、部署架构

### 9.1 开发环境

```
工作目录: C:\Users\likunyuan\Desktop\爬取项目\竞品对阵分析\
Python: 3.12
启动:   py -3.12 run_benchmark_server.py    （或 python -m overseas.benchmark_server）
访问:   http://127.0.0.1:8090/
```

依赖：
- 第一层（后端）：**无第三方依赖**，Python 3.12 标准库即可。
- 第二层（AI 代理）：`fastapi` + `uvicorn`（由 DSH settings 注入的 `anthropic_to_openai_proxy.py` 提供，本仓库不含）。
- `llm_client.py`：若装了 `openai` SDK 优先用，否则回退 `urllib`（标准库，永远可用）。

### 9.2 生产部署

- **离线部署**：后端只需 `overseas/` 包 + `data/overseas_v4_20260929.db` + `benchmark_static/` 前端目录，拷到目标机 `py -3.12 run_benchmark_server.py` 即起。
- **AI 可选**：若目标机无外网/无 8082 代理，AI 分析自动走规则兜底，规格对比+雷达图正常可用。
- **CORS**：`--cors "*"` 开启跨域（供其他前端调用）；默认空（仅本机访问）。
- **定时溯源**：`--enable-scheduled-trace` 启用每天凌晨 2 点自动溯源补全缺失参数（`api_auto_trace`，`with_ai=False` 只走第1+2层，不调慢的 AI 第3层）。

### 9.3 三个端口说明

| 端口 | 服务 | 协议 | 必要性 |
|---|---|---|---|
| **8090** | `benchmark_server.py`（后端 API + 静态前端） | HTTP（标准库 http.server） | **必需**，核心服务 |
| **8082** | `anthropic_to_openai_proxy.py`（AI 转换代理） | HTTP（fastapi，OpenAI 兼容 /v1） | 可选，AI 分析时需要 |
| 8080 | DSH Web GUI（开发用，非本系统） | — | 无关 |

8082 代理特点：**忽略客户端鉴权头**（apiKey 用占位值 `local-proxy-placeholder`），由 DSH settings 注入真实 xinghai-ultra 凭据。`_setup_xinghai()` 在 `main()` 启动时把这套配置注入 `config` 模块（详见 `benchmark_server.py:2049-2098`）。

启动时探活：`urllib.request.urlopen(f"{XINGHAI_BASE_URL}/models", timeout=3)`，打印 `AI 代理: [OK] 可用` 或 `[不可用] ...，将走规则兜底`。

### 9.4 数据库同步流程

数据生产侧（`overseas` 框架的 `scenarios/`、`sites/`）负责爬取与归一化，写入 `data/overseas_v4_20260929.db`。本系统**只读**此 DB，不写。同步流程：

1. 爬取侧按 `overseas` 框架跑 `spec_crawl`（品牌官网）+ 零售站抓取，写原始层（product/price_snapshot/spec_row/product_spec）。
2. 归一侧用 `dim_spec_item_map` 把原始项名映射成 `param_id`，写 `fact_spec_value`（含 `confidence`）。
3. 缺失参数经 `/api/backfill` 三层回溯补全，或 `--enable-scheduled-trace` 定时补全。
4. 本系统启动时指向最新 DB 文件（`--db` 参数或默认 `data/overseas_v4_20260929.db`）。

> DB 文件名带日期版本（`_v4_20260929`），更新版本时需同步改 `DEFAULT_DB` 常量（`benchmark_server.py:122`）。

---

## 十、已知限制与扩展方向

### 10.1 当前限制

| 限制 | 详情 | 影响 |
|---|---|---|
| **SE/NO/DK 海信无零售型号** | 这 3 个欧洲国家目前无海信零售型号数据 | `api_ai_recommend` 返回 `该区域该尺寸暂无海信型号`，无法生成对阵 |
| **TR 只有脏数据** | 土耳其数据为脏数据 | 型号名过滤（`_is_valid_model_code`）后可能所剩无几 |
| **实时价格未实现** | `price_snapshot` 是快照，非实时 | 价格滞后于实际零售价，长期未更新国家价格可能失真 |
| **规格覆盖率约 53%** | 3,451/6,493 型号有规格 | 约 47% 型号只有价格无规格，需经 backfill 补全 |
| **归一化规则维护成本** | 约 280 条正则，新品牌/新格式需手加规则 | 接入新品牌或遇到新规格描述格式时，归一化可能不命中，显示原始文本 |
| **AI prompt 用前 12 参数** | `PARAM_ORDER[:12]`，后 6 参数（WIFI/蓝牙/杜比/功耗/重量/尺寸）不进 AI 分析 | AI 结论可能遗漏连接性/功耗/尺寸维度 |
| **单人/单进程** | `ThreadingHTTPServer` 多线程但单进程，无水平扩展 | 高并发需多实例 + 负载均衡（当前内部工具场景够用） |

### 10.2 扩展方向

| 扩展 | 操作步骤 | 注意事项 |
|---|---|---|
| **新增国家/区域** | 在 `COUNTRIES` 字典加一项（code/name/currency/flag/group/bands），确保该货币的 `price_snapshot` 有数据 | 需同步在数据生产侧爬取该国零售数据；`_region_filter` 自动按货币过滤 |
| **新增品牌** | 数据生产侧新增站点适配器（`overseas/sites/`），跑 `spec_crawl`；归一侧在 `_DISPLAY_NORMALIZE` 加该品牌的专用正则规则 | 规则按品牌分组，最具体的放前面；必要时在 `dim_spec_normalization_map` 加该品牌术语到对应族 |
| **新增参数** | 在 `dim_spec_parameter` 插一行（param_id/code/name/unit/is_numeric/higher_is_better）；在 `PARAM_ORDER`/`PARAM_NAMES_ZH`/`PARAM_CATEGORY` 同步加；若要进雷达图，更新 `RADAR_DIMS` | `fact_spec_value` 需回填该参数；归一化规则在 `_DISPLAY_NORMALIZE` 加对应块 |
| **更新归一化规则** | 直接编辑 `benchmark_server.py` 的 `_DISPLAY_NORMALIZE` 字典，重启服务即生效 | 遵循"最具体规则放前面"原则；新增规则后建议用具体型号的原始值回归测试 |
| **更新归一族映射** | 在 `dim_spec_normalization_map` 加行或更新 `competitor_terms` JSON 数组，`status` 设 CONFIRMED | 全部 9 条已 CONFIRMED，新增族按相同结构 |
| **提升规格覆盖率** | 跑 `/api/auto-trace?with_ai=1`（慢，单型号）或 `--enable-scheduled-trace`（每天凌晨批量，with_ai=False） | AI 第3层慢且可能找不到 URL，批量任务默认跳过 |
| **AI 分析后 6 参数** | 把 `PARAM_ORDER[:12]` 改成全 18，或单独把 WIFI/蓝牙/杜比/功耗/重量/尺寸纳入 prompt | prompt 变长，max_tokens 需适当调大 |
| **实时价格** | 接入零售站实时价 API 或定时刷新 `price_snapshot` | 遵循"缺价 > 错价"原则，宁留空不造假价 |

---

## 附录：关键文件速查

| 路径 | 行数 | 用途 |
|---|---|---|
| `overseas/benchmark_server.py` | 2178 | 主服务（HTTP + 业务 + 归一化 + AI + 回溯） |
| `overseas/config.py` | ~320 | 配置中枢（环境变量 + LLM 默认值） |
| `overseas/llm_client.py` | ~260 | LLM 客户端（SDK + urllib 双路） |
| `overseas/backfill.py` | 282 | 三层回溯补全 |
| `overseas/__init__.py` | 17 | 包声明 + 分层 docstring |
| `run_benchmark_server.py` | 12 | 启动入口 |
| `data/overseas_v4_20260929.db` | — | SQLite 数据库（18 张业务表） |
| `overseas/benchmark_static/` | — | 前端静态资源（index.html + JS + CSS） |

**关键 API 路由速查**：

| 路由 | 方法 | 功能 |
|---|---|---|
| `/api/health` | GET | 健康检查 + AI 就绪状态 |
| `/api/countries` | GET | 21 国列表（含分组） |
| `/api/sizes?region=` | GET | 某区域可用尺寸 |
| `/api/models?region=&size=` | GET | 某区域某尺寸型号 + 均价 + 价格带 |
| `/api/ai-recommend?region=&size=` | GET | AI 智能推荐对阵组合 |
| `/api/spec?model_ids=` | GET/POST | 18 项规格归一化对比矩阵 |
| `/api/radar?model_ids=` | GET | 六维雷达图得分 |
| `/api/ai-analysis` | POST | AI 优劣势结论（body: model_ids, region） |
| `/api/backfill` | GET/POST | 三层回溯补全缺失参数 |
| `/api/backfill-status?model_id=` | GET | 某型号回溯状态 |
| `/api/auto-trace` | GET/POST | 全库自动溯源 |

---

*文档生成于基于 `overseas v0.1.0` + `overseas_v4_20260929.db` 实测数据。后续修改源码或数据库结构后，请同步更新本文档对应章节。*
