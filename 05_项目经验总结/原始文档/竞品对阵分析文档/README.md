# 竞品对阵分析 — 交接文档

> 海外电视产品竞品对阵分析平台。基于规格归一化数据，自动构建我方（海信）与竞品的对阵矩阵，并接入 xinghai-ultra 大模型生成优势/劣势/机会点分析结论，最终在前端以雷达图 + 规格对比表呈现。
>
> **版本**：overseas 0.1.0 ｜ **后端端口**：8090 ｜ **LLM 代理端口**：8082 ｜ **数据库**：overseas_v4_20260929.db（约 135 MB）

---

## 一、系统概述

### 1.1 功能定位

海外电视产品竞品对阵分析平台，面向产品/市场团队，核心解决三类问题：

- **"我和谁打"**：按国家/区域 × 尺寸 × 价格带，自动筛选海信型号并匹配同档竞品。
- **"差在哪"**：把日/英/西/葡等多语言官网规格做归一化，形成同口径的规格对比矩阵。
- **"怎么打"**：接入 xinghai-ultra 大模型，基于规格对比生成"优势点 / 劣势点 / 机会点 / 策略建议"结论，每条可溯源到具体参数或价格。

### 1.2 覆盖范围

覆盖 **21 个国家/区域**，按区域线分组（`benchmark_server.py` 的 `COUNTRIES` 字典定义）：

| 区域线 | 数量 | 国家/区域 |
|--------|------|-----------|
| 日本线 | 1 | JP（日本，JPY） |
| 北美线 | 3 | US（美国，USD）、CA（加拿大，CAD）、MX（墨西哥，MXN） |
| 南美线 | 5 | CO（哥伦比亚，COP）、PE（秘鲁，PEN）、CL（智利，CLP）、GT（危地马拉，GTQ）、BO（玻利维亚，BOB） |
| 欧洲线 | 13 | UK（英国，GBP）、EU（欧元区，EUR）、CH（瑞士，CHF）、SE（瑞典）、PL（波兰）、RS（塞尔维亚）、RO（罗马尼亚）、NO（挪威）、DK（丹麦）、BG（保加利亚）、BA（波黑）、TR（土耳其）等 |

每个国家/区域按尺寸（55/65/75/85 等）定义了四档价格带：**入门 / 中端 / 高端 / 旗舰**（阈值见 `COUNTRIES[code]['bands']`）。

### 1.3 核心能力链路

```
规格归一化 → 对阵矩阵 → AI 优势分析 → 雷达图
   (db)        (benchmark_server)   (xinghai-ultra)   (前端 echarts)
```

1. **规格归一化**：`dim_spec_parameter`（18 个标准参数）+ `dim_spec_item_map`（1974 条映射规则）+ `dim_spec_normalization_map`，把多语言官网规格值映射到统一参数码（如 `REFRESH_RATE`、`HDR`）。
2. **对阵矩阵**：按区域货币严格过滤价格（缺价 > 错价，不跨货币 fallback），按价格带匹配我方型号与同档竞品。
3. **AI 优势分析**：把规格对比摘要 + 雷达得分交给 xinghai-ultra，返回结构化 JSON（优势/劣势/机会/策略），AI 不可用时自动走规则兜底。
4. **雷达图**：六维（画质/音质/智能/游戏/连接/设计）评分，文本特征 + 数值相对评分混合算法。

---

## 二、系统架构

### 2.1 两层架构

```
┌─────────────────────────────────────────────────────────────┐
│  浏览器 (用户)                                                │
│  http://127.0.0.1:8090/                                      │
│  └─ overseas/benchmark_static/index.html (39KB, echarts)    │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTP GET/POST
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  【第一层：对阵分析后端】 端口 8090（必需）                    │
│  overseas/benchmark_server.py  (2178 行)                     │
│  纯 Python 标准库（http.server + sqlite3），零第三方依赖       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  GET  /api/health      健康检查 + AI 就绪状态          │  │
│  │  GET  /api/countries   国家列表                         │  │
│  │  GET  /api/sizes       尺寸列表                         │  │
│  │  GET  /api/models      型号列表（含均价/价格带）        │  │
│  │  GET  /api/ai-recommend AI 推荐对阵组合                 │  │
│  │  GET  /api/spec        规格归一化对比矩阵               │  │
│  │  GET  /api/radar       六维雷达得分                     │  │
│  │  POST /api/ai-analysis AI 优势/劣势/机会分析            │  │
│  │  POST /api/backfill    参数补全（三层回溯）             │  │
│  │  GET  /api/backfill-status 补全状态查询                │  │
│  │  POST /api/auto-trace  全库自动溯源                    │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │ 读取规格/价格数据
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  data/overseas_v4_20260929.db  (135 MB, SQLite)             │
│  6 张核心表 + 价格快照表                                       │
└─────────────────────────────────────────────────────────────┘

           ┌──────────────── 可选（推荐）────────────────┐
           ▼                                             │
┌─────────────────────────────────────────────────────────┐
│  【第二层：LLM AI 代理】 端口 8082（可选但推荐）           │
│  llm_proxy/anthropic_to_openai_proxy.py (1138 行)        │
│  依赖 fastapi/uvicorn/httpx                              │
│  作用：把 xinghai-ultra（内网 HiGPT）封装为 OpenAI 兼容    │
│        /v1 端点，供 benchmark_server 调用                  │
│  不启动 → benchmark_server 走规则兜底，AI 分析降级        │
└─────────────────────────────────────────────────────────┘
```

### 2.2 数据流

```
SQLite 数据库
   │
   ├─ dim_product_model (6493 型号) ──┐
   ├─ fact_spec_value (52636 规格)  ──┤
   ├─ dim_spec_parameter (18 参数)  ──┤─→ benchmark_server
   ├─ dim_spec_item_map (1974 映射) ──┤     规格归一化 + 对阵矩阵
   ├─ price_snapshot (14971 价格)  ───┘
   │
   ▼
benchmark_server (8090)
   │
   ├─ /api/spec  ────────── 规格对比矩阵 (JSON)
   ├─ /api/radar ────────── 六维雷达得分 (JSON)
   └─ /api/ai-analysis ──── 拼装规格摘要 ──→ LLMClient
                                              │
                          ┌───────────────────┤
                          ▼ (代理可用)         ▼ (代理不可用)
                    xinghai-ultra (8082)    规则兜底 _fallback_analysis
                          │                    （价格/系统/游戏启发式）
                          ▼
                   优势/劣势/机会 JSON
                          │
                          ▼
                  前端 index.html + echarts
```

---

## 三、目录结构

```
竞品对阵分析/                         ← 独立可部署文件夹，根目录即运行目录
├── overseas/                         ← Python 后端代码包
│   ├── __init__.py                   ← 包初始化，__version__ = "0.1.0"
│   ├── benchmark_server.py           ← 核心后端服务（2178 行，端口 8090）
│   │                                    含 COUNTRIES 区域配置 / API 路由 /
│   │                                    规格归一化 / 雷达评分 / AI 分析
│   ├── config.py                     ← 全局配置（314 行，环境变量驱动）
│   │                                    含路径/代理/浏览器/LLM/xinghai 等配置段
│   ├── llm_client.py                 ← LLM Chat 客户端（258 行）
│   │                                    多 provider（xinghai/deepseek/openai_compatible）
│   ├── db.py                         ← SQLite 存储层（58 KB）
│   ├── backfill.py                   ← 参数补全工具（14 KB，三层回溯）
│   ├── models.py                     ← 数据模型（6 KB）
│   └── benchmark_static/             ← 前端静态文件目录
│       ├── index.html                ← 对阵分析前端页面（39 KB，echarts 雷达图）
│       └── echarts.min.js            ← ECharts 图表库（1 MB，离线内联）
├── data/                             ← 数据库目录
│   └── overseas_v4_20260929.db       ← SQLite 数据库（135 MB）
│                                      含 6 张核心表 + 价格/产品/评价等
├── llm_proxy/                        ← LLM AI 代理（可选）
│   └── anthropic_to_openai_proxy.py ← xinghai-ultra 代理（1138 行，端口 8082）
│                                      Anthropic ↔ OpenAI 格式双向转换
├── docs/                             ← 文档目录（本 README 及补充文档存放处）
├── requirements.txt                  ← 依赖清单（fastapi/uvicorn/httpx）
├── install.bat                       ← 一键安装（Windows）
├── start.bat                         ← 一键启动（Windows，先起 8082 再起 8090）
└── run_benchmark_server.py           ← 启动入口脚本（12 行，切工作目录后调 main）
```

### 3.1 各文件职责详解

| 文件 | 职责 | 关键点 |
|------|------|--------|
| `benchmark_server.py` | 后端全部逻辑 | 21 国区域配置 + 价格带阈值 + 18 参数归一化 + 6 维雷达评分 + AI 分析 + HTTP 路由 |
| `config.py` | 环境变量配置中心 | `OVERSEAS_` 前缀统一；`llm_settings()` 按 provider 汇总生效配置 |
| `llm_client.py` | LLM 调用封装 | `LLMClient.configured` 检查；`chat()` 方法；`LLMUnavailable` 异常 |
| `db.py` | SQLite 存储层 | 价格快照按时间只增不改；spec 值写入 |
| `backfill.py` | 参数补全 | 三层：DB 已有 → 站点标记查找 → AI 推断官网 URL |
| `models.py` | 数据模型 | 型号/规格/价格的数据类 |
| `anthropic_to_openai_proxy.py` | LLM 代理 | 修复了流式分片累积 + 残缺 tool_use 两个关键缺陷 |
| `run_benchmark_server.py` | 启动入口 | `chdir` 到脚本目录，确保相对路径正确 |

---

## 四、依赖说明

### 4.1 Python 环境

- **要求**：Python >= 3.10（使用了 `from __future__ import annotations` 和类型提示 `dict | None`）
- **推荐**：Python 3.12
- **验证**：`python --version`

### 4.2 后端依赖（第一层：对阵分析后端 + 前端）

**零第三方依赖，纯 Python 标准库即可运行。**

`benchmark_server.py` 仅用以下标准库：

```
argparse / json / os / sqlite3 / sys / traceback
http.server (BaseHTTPRequestHandler, ThreadingHTTPServer)
urllib.parse (parse_qs, urlparse)
```

- 不需要 `pip install` 任何包。
- `db.py` / `backfill.py` / `models.py` 同样只依赖标准库。

### 4.3 LLM 代理依赖（第二层，可选但推荐）

仅当需要 AI 优势分析功能时安装。依赖列表见 `requirements.txt`：

```
fastapi>=0.104.0
uvicorn>=0.24.0
httpx>=0.25.0
```

- `anthropic_to_openai_proxy.py` 用 `fastapi` 起服务、`httpx` 转发请求、`uvicorn` 运行 ASGI。
- 如果跳过 AI，系统会走规则兜底（见 §6.2）。

### 4.4 可选依赖

- **Excel 报表导出**（CLI `export --format xlsx`）：`openpyxl==3.1.5`（已在 `requirements.txt` 中注释）

### 4.5 安装命令

```bash
# 完整安装（含 AI 代理）
pip install -r requirements.txt

# 验证 LLM 代理依赖
python -c "import fastapi, uvicorn, httpx; print('LLM代理依赖OK')"

# 仅后端（无 AI）—— 无需任何 pip install，直接运行即可
```

---

## 五、快速部署

### 5.1 Windows 部署

**一键方式（推荐）**：

```cmd
1. 双击 install.bat    ← 检查 Python、安装 LLM 代理依赖、校验数据库/前端
2. 双击 start.bat      ← 先后台起 LLM 代理(8082)，再前台起后端(8090)
3. 浏览器打开 http://127.0.0.1:8090/
```

`install.bat` 执行流程：
1. 查找 Python（优先 `py -3.12`，回退 `python`）
2. `pip install fastapi uvicorn httpx`
3. 校验 `data\overseas_v4_20260929.db` 存在（不存在则报错退出）
4. 校验 `overseas\benchmark_static\index.html` 存在

`start.bat` 执行流程：
1. 后台启动 LLM 代理（`start /min`，端口 8082）
2. 等待 3 秒
3. 前台启动后端（`python run_benchmark_server.py --port 8090`）

**手动方式**：

```cmd
pip install fastapi uvicorn httpx
start /min python llm_proxy\anthropic_to_openai_proxy.py
python run_benchmark_server.py --port 8090
```

### 5.2 Linux 部署

```bash
# 1. 安装依赖（可选 AI 代理依赖）
pip install -r requirements.txt

# 2. 后台启动 LLM 代理（可选但推荐）
nohup python llm_proxy/anthropic_to_openai_proxy.py > llm_proxy.log 2>&1 &

# 3. 启动后端
python run_benchmark_server.py --port 8090

# 或直接用模块方式启动
python -m overseas.benchmark_server --port 8090
```

**生产环境 systemd 示例**（`config.py` 注释提及）：
- 用 `EnvironmentFile` 注入环境变量（`OVERSEAS_*` 系列）
- `--host 0.0.0.0 --port 8090 --cors "*"`

### 5.3 验证

```bash
# 健康检查（返回 status=ok 即正常）
curl http://127.0.0.1:8090/api/health

# 预期返回：
# {
#   "status": "ok",
#   "version": "0.1.0",
#   "ai_ready": true|false,
#   "ai_model": "xinghai-ultra",
#   "ai_base": "http://127.0.0.1:8082/v1"
# }

# 验证国家列表
curl http://127.0.0.1:8090/api/countries

# 验证前端页面
curl http://127.0.0.1:8090/ | head -5
```

启动时控制台会打印：
```
竞品对阵分析服务 v0.1.0
数据库: .../data/overseas_v4_20260929.db
静态目录: .../overseas/benchmark_static
xinghai-ultra: http://127.0.0.1:8082/v1 model=xinghai-ultra
AI 代理: [OK] 可用    ← 或 [不可用] ...，将走规则兜底
访问: http://127.0.0.1:8090/
```

---

## 六、服务启动

### 6.1 后端服务（端口 8090，必需）

**启动命令**：

```bash
# 方式一：入口脚本（推荐，自动 chdir）
python run_benchmark_server.py --port 8090

# 方式二：模块方式
python -m overseas.benchmark_server --port 8090

# 完整参数
python run_benchmark_server.py \
  --host 127.0.0.1 \
  --port 8090 \
  --db data/overseas_v4_20260929.db \
  --cors "*" \
  --enable-scheduled-trace
```

**启动参数**（`argparse`）：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--host` | `127.0.0.1` | 监听地址，外网访问用 `0.0.0.0` |
| `--port` | `8090` | 监听端口 |
| `--db` | `data/overseas_v4_20260929.db` | SQLite 数据库路径（`DEFAULT_DB`） |
| `--cors` | 空 | CORS 允许源，如 `*` |
| `--enable-scheduled-trace` | 关 | 启用定时溯源（每天凌晨 2 点扫描全库缺失参数并补全） |

**健康检查**：

```bash
curl http://127.0.0.1:8090/api/health
# status=ok 表示后端存活；ai_ready 表示 LLM 凭据是否就绪
```

**API 端点列表**：

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/health` | 健康检查 + AI 就绪状态 |
| GET | `/api/countries` | 21 国列表（含分组/货币/旗帜） |
| GET | `/api/sizes?region=JP` | 某区域可用尺寸 + 型号数 |
| GET | `/api/models?region=JP&size=65` | 某区域某尺寸型号列表（含均价/价格带） |
| GET | `/api/ai-recommend?region=JP&size=65` | AI 推荐对阵组合 |
| GET | `/api/spec?model_ids=1,2,3` | 规格归一化对比矩阵 |
| GET | `/api/radar?model_ids=1,2,3` | 六维雷达得分 |
| GET/POST | `/api/ai-analysis` | AI 优势/劣势/机会分析 |
| POST | `/api/backfill` | 参数补全（三层回溯） |
| GET | `/api/backfill-status?model_id=1` | 补全状态查询 |
| POST | `/api/backfill-crawl` | 触发站点爬取 |
| GET/POST | `/api/auto-trace` | 全库自动溯源 |

### 6.2 LLM 代理（端口 8082，可选但推荐）

**启动命令**：

```bash
# 后台启动
python llm_proxy/anthropic_to_openai_proxy.py
# 监听 127.0.0.1:8082，暴露 /v1（OpenAI 兼容）端点
```

**作用说明**：

- 把 xinghai-ultra（海信内网 HiGPT，`https://inner-apisix.hisense.com/higpt-new/v1`）封装为本地 OpenAI 兼容端点 `http://127.0.0.1:8082/v1`。
- `benchmark_server.py` 的 `_setup_xinghai()` 在启动时自动把 LLM 配置注入为该本地代理地址（`proxy_url = "http://127.0.0.1:8082/v1"`，apiKey 用占位值 `local-proxy-placeholder`，provider 设为 `xinghai`，流式关闭，SSL 校验关闭）。
- 代理脚本修复了两个关键缺陷（详见脚本头部注释）：
  1. 流式转换把同一工具调用的分片拆成多个残缺 `tool_use` block（导致 xinghai-ultra 报 400）；
  2. 历史里残缺 `tool_use` 永久毒化会话（请求侧 sanitize 降级为文本）。

**不启动的影响**：

- 后端正常启动，`/api/health` 返回 `ai_ready: true`（凭据已注入）但代理不可达。
- `/api/ai-analysis` 调用 xinghai-ultra 时连接失败，**自动走规则兜底** `_fallback_analysis()`：
  - 仍返回结构化 JSON（`advantage_points` / `disadvantage_points` / `opportunity_points` / `strategy_suggestion`）；
  - 返回中带 `"fallback": true` 和 `"model": "xinghai-ultra（未配置，规则兜底）"`；
  - 兜底逻辑基于价格对比 + 关键参数（144Hz/FreeSync/HDR/智能系统）启发式判断。
- 前端页面、规格对比、雷达图**完全不受影响**，仅 AI 结论降级为规则启发式。

---

## 七、API 接口文档

### 7.1 GET `/api/health`

健康检查 + AI 就绪状态。

**参数**：无

**返回**：

```json
{
  "status": "ok",
  "version": "0.1.0",
  "ai_ready": true,
  "ai_model": "xinghai-ultra",
  "ai_base": "http://127.0.0.1:8082/v1"
}
```

### 7.2 GET `/api/countries`

返回 21 个国家/区域列表。

**参数**：无

**返回**：

```json
{
  "countries": [
    {"code": "JP", "name": "日本", "currency": "JPY", "flag": "🇯🇵", "group": "日本"},
    {"code": "US", "name": "美国", "currency": "USD", "flag": "🇺🇸", "group": "北美"},
    ...
  ]
}
```

### 7.3 GET `/api/sizes`

返回某区域可用尺寸列表 + 各尺寸型号数。

**参数**：

| 参数 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `region` | string | 否 | `JP` | 区域码（JP/US/CA/EU 等） |

**返回**：

```json
{
  "region": "JP",
  "currency": "JPY",
  "sizes": [
    {"size": 55, "model_count": 42},
    {"size": 65, "model_count": 38},
    {"size": 75, "model_count": 25}
  ]
}
```

> 仅返回标准尺寸：19/24/32/40/42/43/48/50/55/58/65/75/77/85/98/100。

### 7.4 GET `/api/models`

返回某区域某尺寸的所有型号 + 均价 + 价格带。

**参数**：

| 参数 | 类型 | 必填 | 默认 | 说明 |
|------|------|------|------|------|
| `region` | string | 否 | `JP` | 区域码 |
| `size` | int | 否 | `65` | 尺寸（英寸） |

**返回**：

```json
{
  "region": "JP",
  "currency": "JPY",
  "size": 65,
  "models": [
    {
      "model_id": 101,
      "brand": "Hisense",
      "model_code": "65U8R",
      "price": 198000,
      "price_band": "中端",
      "our_brand": 1,
      "is_our": true
    },
    ...
  ]
}
```

> 价格严格按区域货币过滤（`price_snapshot.currency = COUNTRIES[region].currency`），无该货币价格的型号直接跳过（缺价 > 错价）。脏数据型号名（纯数字、规格词如 `180HZ`）会被 `_is_valid_model_code()` 过滤。

### 7.5 GET `/api/spec`

返回多个型号的规格归一化对比矩阵。

**参数**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `model_ids` | string | 是 | 型号 ID 逗号分隔，如 `1,2,3` |
| `region` | string | 否 | 区域码（默认 `JP`，用于货币过滤） |

**返回**：

```json
{
  "spec_matrix": {
    "PEAK_BRIGHTNESS": {
      "param_code": "PEAK_BRIGHTNESS",
      "param_name": "...",
      "param_name_zh": "峰值亮度",
      "param_unit": "nits",
      "is_numeric": 1,
      "higher_is_better": 1,
      "category": "画质",
      "values": {
        "65U8R": {"display": "1000nits", "numeric": 1000.0, "raw": "1000"},
        "XR-65A95L": {"display": "—", "raw": null}
      }
    },
    ...
  },
  "models": [...]
}
```

> 覆盖 18 个参数（`PARAM_ORDER`）：峰值亮度/分区控光/刷新率/画质芯片/智能系统/游戏特性/色域/HDMI/屏幕尺寸/分辨率/HDR/对比度/Wi-Fi/蓝牙/杜比全景声/功耗/重量/尺寸。●/○ 符号值用 `source_term` 做归一化输入。

### 7.6 GET `/api/radar`

返回多个型号的六维雷达图得分。

**参数**：同 `/api/spec`（`model_ids` 逗号分隔）。

**返回**：

```json
{
  "radar_dims": ["画质", "音质", "智能", "游戏", "连接", "设计"],
  "scores": {
    "65U8R": {"画质": 78, "音质": 80, "智能": 55, "游戏": 85, "连接": 72, "设计": 70},
    ...
  },
  "models": [...]
}
```

> 六维（`RADAR_DIMS`）：画质（亮度/分区/刷新率/色域/HDR/芯片/分辨率/对比度）、音质（杜比全景声）、智能（系统）、游戏（游戏特性+刷新率+HDMI）、连接（HDMI+Wi-Fi+蓝牙）、设计（功耗/尺寸/重量/重量/尺寸）。评分算法：文本特征打分（如 `Google TV` → 80）+ 数值型相对打分（按最大最小值归一化到 25~95）。

### 7.7 POST `/api/ai-analysis`

接入 xinghai-ultra 生成 AI 对阵结论。

**请求体**（JSON）：

```json
{
  "model_ids": [1, 2, 3],
  "region": "JP"
}
```

**返回（AI 可用）**：

```json
{
  "ai_conclusion": {
    "advantage_points": [
      {"point": "刷新率领先：144Hz", "evidence": "spec REFRESH_RATE", "confidence": 0.9}
    ],
    "disadvantage_points": [...],
    "opportunity_points": [...],
    "strategy_suggestion": "以画质+游戏+性价比组合应对，补强智能生态"
  },
  "model": "xinghai-ultra via http://127.0.0.1:8082/v1",
  "summary": "区域: JP\n我方(海信) Hisense 65U8R 价格198000JPY\n..."
}
```

**返回（AI 不可用，规则兜底）**：

```json
{
  "ai_conclusion": {
    "advantage_points": [...],
    "disadvantage_points": [...],
    "opportunity_points": [...],
    "strategy_suggestion": "以画质+游戏+性价比组合应对，补强智能生态"
  },
  "model": "xinghai-ultra（未配置，规则兜底）",
  "fallback": true,
  "summary": "...",
  "config_needed": "设置 OVERSEAS_XINGHAI_API_KEY / ..."
}
```

> 也支持 GET 方式（`/api/ai-analysis?model_ids=1,2,3&region=JP`）。AI prompt 要求输出 JSON（优势/劣势/机会 + 置信度 0-1），每条可溯源到参数或价格。

---

## 八、数据库说明

**文件位置**：`data/overseas_v4_20260929.db`（135 MB，SQLite）

**连接方式**：`sqlite3.connect(SETTINGS['db'])`，`row_factory = sqlite3.Row`

**6 张核心表**：

| 表名 | 行数 | 说明 |
|------|------|------|
| `dim_spec_parameter` | 18 | 规格参数维度表（param_code → 名称/单位/是否数值/越高越好/雷达维度） |
| `dim_spec_item_map` | 1974 | 规格项映射规则（source_term → param_id，含匹配关键词/正则/品牌范围） |
| `dim_product_model` | 6493 | 产品型号维度表（model_code → 品牌/区域/尺寸/是否我方） |
| `fact_spec_value` | 52636 | 规格值事实表（model_id × param_id → value_text/value_numeric/source_term） |
| `dim_spec_normalization_map` | 9 | 海信 vs 竞品术语归一化映射（hisense_term ↔ competitor_terms） |
| `fact_benchmark_comparison` | 4 | 对阵分析结果存档（我方型号 + 竞品型号 + 优势/劣势/机会 + 置信度 + 策略） |

**其他关键表**：

| 表名 | 行数 | 说明 |
|------|------|------|
| `product` | 14959 | 产品表（site_id/sku/url/brand/model） |
| `price_snapshot` | 14971 | 价格快照表（按时间只增不改，含 currency/in_stock） |
| `price_shop_snapshot` | - | 店铺级价格快照 |
| `review` / `review_summary` | - | 用户评价 |
| `spec_row` / `spec_series` | - | 原始规格行 |
| `crawl_run` / `crawl_error` | - | 抓取运行记录 |
| `site` | - | 站点配置 |

> 详见 `docs/DATABASE.md`（待补充）。

---

## 九、配置项

所有配置通过环境变量覆盖，命名统一 `OVERSEAS_` 前缀（见 `config.py`）。未显式设置时用默认值，可直接本地跑通。

### 9.1 路径配置

| 环境变量 | 默认值 | 说明 |
|----------|--------|------|
| `OVERSEAS_DATA_DIR` | `<BASE_DIR>/data` | 数据目录 |
| `OVERSEAS_DB_PATH` | `<DATA_DIR>/overseas.db` | 通用 DB 路径（对阵分析用 `--db` 覆盖） |
| `OVERSEAS_CACHE_DIR` | `<DATA_DIR>/cache` | 缓存目录 |
| `OVERSEAS_RAW_DIR` | `<DATA_DIR>/raw` | 原始 HTML 存档 |
| `OVERSEAS_LOG_DIR` | `<DATA_DIR>/logs` | 日志目录 |
| `OVERSEAS_SESSION_DIR` | `<DATA_DIR>/session` | 登录态（cookie，敏感） |
| `OVERSEAS_PROFILE_DIR` | `<DATA_DIR>/browser_profile` | 持久化浏览器 profile |

### 9.2 DEFAULT_DB 与端口配置

- **DEFAULT_DB**（`benchmark_server.py` 硬编码）：
  `<竞品对阵分析>/data/overseas_v4_20260929.db`
  可用 `--db` 启动参数或 `OVERSEAS_DB_PATH` 环境变量覆盖。
- **后端端口**：默认 `8090`，`--port` 参数覆盖。
- **LLM 代理端口**：`8082`（`anthropic_to_openai_proxy.py` 中 `PROXY_PORT = 8082`）。
- **CORS**：默认空（不允许跨域），`--cors "*"` 开放。

### 9.3 LLM / xinghai 配置

`benchmark_server.py` 的 `_setup_xinghai()` 在启动时自动注入（环境变量未显式设置时）：

| 环境变量 | 注入默认值 | 说明 |
|----------|------------|------|
| `OVERSEAS_LLM_BASE_URL` | `http://127.0.0.1:8082/v1` | LLM 端点（本地代理） |
| `OVERSEAS_LLM_API_KEY` | `local-proxy-placeholder` | API Key（占位） |
| `OVERSEAS_LLM_MODEL` | `xinghai-ultra` | 模型名 |
| `OVERSEAS_LLM_PROVIDER` | `xinghai` | provider |
| `OVERSEAS_LLM_STREAM` | `false` | 流式（本地代理用非流式更稳） |
| `OVERSEAS_LLM_SSL_VERIFY` | `false` | SSL 校验（本地代理关闭） |
| `OVERSEAS_XINGHAI_BASE_URL` | `http://127.0.0.1:8082/v1` | xinghai 端点 |
| `OVERSEAS_XINGHAI_API_KEY` | `local-proxy-placeholder` | xinghai API Key |
| `OVERSEAS_XINGHAI_USER_KEY` | `local-proxy-placeholder` | xinghai user key |

> 切换到公网 DeepSeek：`OVERSEAS_LLM_PROVIDER=deepseek`，默认端点 `https://api.deepseek.com`，模型 `deepseek-chat`。

### 9.4 抓取相关配置（对阵分析通常不涉及）

`config.py` 还含代理/浏览器/节流/翻译等配置段（`OVERSEAS_PROXY` / `OVERSEAS_MIN_INTERVAL` / `OVERSEAS_BROWSER_EXECUTABLE` 等），对阵分析后端不依赖这些，详见 `config.py` 注释。

---

## 十、常见问题

### Q1：AI 不可用怎么办？

**现象**：`/api/health` 返回 `ai_ready: true` 但 `/api/ai-analysis` 返回 `fallback: true`；启动日志显示 `AI 代理: [不可用] ...，将走规则兜底`。

**排查**：

1. 确认 LLM 代理是否启动：`curl http://127.0.0.1:8082/v1/models`，应返回模型列表。
2. 确认 8082 端口未被占用：`netstat -ano | findstr :8082`。
3. 确认 `anthropic_to_openai_proxy.py` 内网可达 `https://inner-apisix.hisense.com/higpt-new/v1`（内网环境，公网不可达）。
4. 若代理脚本报 400，检查是否触发了"残缺 tool_use"问题（脚本已修复，确认用的是最新版）。

**影响**：不影响后端、前端、规格对比、雷达图；AI 结论降级为规则兜底（价格 + 关键参数启发式）。

### Q2：数据库路径不对？

**现象**：启动报 `sqlite3.OperationalError: unable to open database file` 或 `api_models` 返回空。

**排查**：

1. 确认 `data/overseas_v4_20260929.db` 存在（135 MB）。
2. 用 `--db` 显式指定绝对路径：`python run_benchmark_server.py --db "C:\...\data\overseas_v4_20260929.db"`。
3. 确认 `run_benchmark_server.py` 已 `chdir` 到项目根目录（相对路径基准）。
4. 检查数据库完整性：`sqlite3 data/overseas_v4_20260929.db "PRAGMA integrity_check;"`。

### Q3：端口被占用？

**现象**：启动报 `OSError: [Errno 10048/98] Address already in use`。

**排查**：

```cmd
# Windows
netstat -ano | findstr :8090
taskkill /PID <占用进程PID> /F

# Linux
lsof -i :8090
kill -9 <PID>
```

或换端口：`python run_benchmark_server.py --port 8091`。LLM 代理端口 8082 同理（需改 `anthropic_to_openai_proxy.py` 的 `PROXY_PORT`）。

### Q4：前端页面空白？

**现象**：打开 `http://127.0.0.1:8090/` 显示空白或 404。

**排查**：

1. 确认 `overseas/benchmark_static/index.html` 存在（39 KB）。
2. 确认 `echarts.min.js` 存在（1 MB，离线内联，不依赖 CDN）。
3. 浏览器 F12 控制台看是否有 JS 报错（通常是 API 404 或 CORS 问题）。
4. 若跨域调用，启动时加 `--cors "*"`。
5. 静态文件由 `_serve_static()` 处理，根路径 `/` 自动映射到 `index.html`。

### Q5：某区域型号为空？

**现象**：`/api/models?region=XX&size=YY` 返回空 `models`。

**原因**：该区域货币在 `price_snapshot` 中无价格记录（缺价 > 错价，不跨货币 fallback）。日本线严格只取 JPY，之前曾误把 COP/PEN/USD/EUR 当 JPY 显示，现已修复。

### Q6：定时溯源如何启用？

启动时加 `--enable-scheduled-trace`，每天凌晨 2 点自动扫描全库缺失参数并补全（`api_auto_trace(max_models=100, with_ai=False)`）。默认不启动。

---

## 十一、交接清单

### 11.1 需确认的项

| # | 确认项 | 现状 | 需动作 |
|---|--------|------|--------|
| 1 | 数据库文件 `data/overseas_v4_20260929.db` 是否最新 | 135 MB，含 6493 型号 | 确认数据截止日期，必要时重新生成 |
| 2 | LLM 代理内网可达性 | 脚本写死 `https://inner-apisix.hisense.com/higpt-new/v1` | 确认部署环境能访问海信内网 |
| 3 | LLM 代理凭据 | `anthropic_to_openai_proxy.py` 内含 `HISENSE_AUTH_TOKEN` / `HISENSE_API_KEY` | 确认凭据仍有效，失效则更新 |
| 4 | Python 环境 | 要求 3.10+，推荐 3.12 | 确认服务器 Python 版本 |
| 5 | 前端静态文件版本 | `index.html` 39 KB + `echarts.min.js` 1 MB | 确认 echarts 版本与 index.html 兼容 |
| 6 | `docs/` 目录 | 当前为空 | 需补充 `DATABASE.md`（数据库表结构详解） |
| 7 | 21 国价格带阈值 | 在 `COUNTRIES[code]['bands']` 硬编码 | 确认阈值是否需要按市场更新 |
| 8 | 18 个规格参数 | `PARAM_ORDER` 硬编码 | 确认是否需增减参数 |
| 9 | 规格归一化规则 | `dim_spec_item_map` 1974 条 + `_DISPLAY_NORMALIZE` 正则 | 确认新品牌/新型号的规则覆盖 |
| 10 | 对阵结果存档 | `fact_benchmark_comparison` 仅 4 条 | 确认是否需定期快照 |

### 11.2 运维关注点

- **数据库备份**：135 MB SQLite，变更前先备份（`copy data\overseas_v4_20260929.db data\backup_YYYYMMDD.db`）。
- **日志位置**：后端日志输出到 stderr（`log_message` 写 `sys.stderr`），生产环境重定向到文件。
- **进程管理**：Windows 用 `start.bat`（前台）；Linux 用 `nohup` 或 systemd。
- **AI 降级**：LLM 代理挂掉不影响核心功能，无需告警熔断；但 AI 结论质量下降需知晓。

### 11.3 后续可扩展

- 增加国家/区域：编辑 `COUNTRIES` 字典（含货币 + 价格带阈值）。
- 增加规格参数：在 `dim_spec_parameter` 插入 + `PARAM_ORDER` / `PARAM_NAMES_ZH` / `PARAM_CATEGORY` / `RADAR_DIMS` 同步。
- 切换 LLM provider：`OVERSEAS_LLM_PROVIDER=deepseek`（公网 DeepSeek）或任意 OpenAI 兼容端点。
- 迁移 PostgreSQL：`config.py` 预留 `OVERSEAS_DB_URL`（目前实现 SQLite）。

---

> **文档版本**：1.0 ｜ **对应代码版本**：overseas 0.1.0 ｜ **最后更新**：2025-09
>
> 如有疑问，参考源码：`overseas/benchmark_server.py`（核心逻辑）、`overseas/config.py`（配置项）、`llm_proxy/anthropic_to_openai_proxy.py`（AI 代理）。
