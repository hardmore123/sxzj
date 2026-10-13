# 依赖项详细说明

> 本文档列出系统**所有依赖项**，包括 Python 版本、标准库模块、项目内部依赖、外部服务依赖。

---

## 1. 运行环境依赖

### 1.1 Python 版本

| 依赖 | 要求 | 说明 |
|------|------|------|
| **Python** | ≥ 3.10（推荐 3.12） | 使用了 `match/case`、`X | Y` 类型联合、`from __future__ import annotations` |

验证命令：
```bash
py -3.12 --version
```

### 1.2 第三方 Python 包

| 包 | 是否必需 | 用途 | 安装命令 |
|----|----------|------|----------|
| **无** | ✅ 核心零依赖 | 全部核心模块仅用 Python 标准库 | — |
| `openai` | ❌ 可选 | LLM 解读功能（聚好看接口兼容 OpenAI 格式） | `pip install openai` |
| `lxml` | ❌ 可选 | `fetchers.py` 的 HTML 解析（不装则用正则降级） | `pip install lxml` |

> **不安装 `openai` 也能运行**：LLM 解读会自动降级为规则式摘要，不影响抓取/预警/看板功能。

### 1.3 Python 标准库模块（无需安装，Python 自带）

以下标准库模块被系统使用，均为 Python 3.10+ 内置：

| 模块 | 使用文件 | 用途 |
|------|----------|------|
| `os` | config.py / fetchers.py / 多个 crawler | 文件路径 / 环境变量 |
| `sys` | cert_crawler.py | 标准错误输出 |
| `json` | compintel_db.py / alert.py / llm_analyze.py / web_api.py | JSON 序列化 |
| `sqlite3` | db.py / compintel_db.py | SQLite 数据库 |
| `urllib.request` | fetchers.py / ic_ised_crawler.py | HTTP 请求 |
| `urllib.parse` | fetchers.py / web_api.py / energystar_crawler.py | URL 编解码 |
| `urllib.error` | fetchers.py | HTTP 异常 |
| `http.server` | web_api.py | Web 服务（HTTPServer + BaseHTTPRequestHandler） |
| `http.cookiejar` | ic_ised_crawler.py | ISED Cookie 管理 |
| `html` | fcc_crawler.py | HTML 转义 |
| `re` | fcc_crawler.py / eprel_crawler.py / cert_crawler.py | 正则匹配 |
| `csv` | anatel_crawler.py / ic_ised_crawler.py | CSV 解析 |
| `io` | anatel_crawler.py / ic_ised_crawler.py | 字符流处理 |
| `zipfile` | anatel_crawler.py | ANATEL ZIP 解压 |
| `hashlib` | db.py / infra.py | 哈希 / 去重 |
| `random` | infra.py | UA 轮换 |
| `socket` | infra.py | 代理检测 |
| `threading` | infra.py / web_api.py | 并发控制 |
| `time` | fetchers.py / infra.py / ic_ised_crawler.py | 限速 / 延迟 |
| `datetime` | db.py / alert.py | 日期时间处理 |
| `dataclasses` | models.py / fcc_crawler.py / eprel_crawler.py / fetchers.py | 数据类 |
| `contextlib` | fetchers.py | 上下文管理器 |
| `typing` | fetchers.py / eprel_crawler.py / orchestrator.py | 类型注解 |
| `traceback` | llm_analyze.py | 异常堆栈 |

---

## 2. 环境变量依赖

| 变量 | 必需 | 默认值 | 说明 |
|------|------|--------|------|
| `OVERSEAS_PROXY` | ✅ 抓取必需 | 无 | 出口代理地址。抓取 FCC/EPREL/ENERGY STAR/ANATEL/IC-ISED 等海外 API 必须走代理。看板浏览不需要 |
| `PYTHONIOENCODING` | ✅ Windows 必需 | 系统默认 | 设为 `utf-8`，解决 Windows 中文输出乱码 |
| `HTTP_PROXY` / `HTTPS_PROXY` | ❌ | 无 | LLM 调用时需临时清空（内网直连不走代理） |

### 设置方式

```bash
# Windows PowerShell
$env:OVERSEAS_PROXY = "http://127.0.0.1:7877"
$env:PYTHONIOENCODING = "utf-8"

# Linux / macOS
export OVERSEAS_PROXY=http://127.0.0.1:7877
export PYTHONIOENCODING=utf-8
```

> ⚠️ 如果代理端口不是 7877，替换为实际端口。无代理环境可不设 `OVERSEAS_PROXY`，但海外 API 可能无法访问。

---

## 3. 外部服务依赖

### 3.1 认证数据源 API（7 个）

| 数据源 | API 地址 | 需要 Key | 代理 | 说明 |
|--------|----------|----------|------|------|
| FCC | `apps.fcc.gov/oetcf/eas/reports/` | ❌ | ✅ | 美国联邦通信委员会，公开搜索 |
| EPREL | `eprel.ec.europa.eu/api/products/` | ❌ | ✅ | 欧盟能效注册，公开 API |
| ENERGY STAR | `data.energystar.gov/resource/` | ❌ | ✅ | Socrata SODA API，公开免 Key |
| ANATEL | `sistemas.anatel.gov.br/...` | ❌ | ✅ | 巴西电信局，全量 ZIP-CSV 下载 |
| IC/ISED | `sms-sgs.ic.gc.ca/equipmentSearch/` | ❌ | ✅ | 加拿大 ISED，表单搜索 |
| TÜV Rheinland | `certipedia.com/quality-marks/` | ❌ | ✅ | Certipedia 搜索页 |
| VDE/CB | `vde.com/solr-search` | ❌ | ✅ | VDE Solr API |

> 所有数据源均**无需 API Key**，仅需出口代理即可访问。

### 3.2 LLM 服务（可选）

| 服务 | 地址 | 需要 Key | 说明 |
|------|------|----------|------|
| 聚好看 deepseek-v4-flash | `inner-apisix.hisense.com/higpt-new/v1` | ✅ 内置 | 海信内网，不走代理直连 |

> LLM 不可用时系统自动降级为规则式摘要，不影响核心功能。
> 外部部署需替换 `llm_analyze.py` 中 `MODEL_CONFIGS` 的 `base_url` 和 `api_key`。

---

## 4. 项目内部依赖关系

### 4.1 依赖链（从入口到底层）

```
compintel_cli.py (入口)
  └── overseas.config          (配置/目录)
  └── overseas.db              (数据库管理)
  │     └── overseas.config
  │     └── overseas.models    (数据模型)
  └── overseas.compintel.web_api (Web API)
  │     └── overseas.db
  │     └── overseas.compintel.compintel_db
  └── overseas.compintel.orchestrator (编排器)
  │     └── overseas.db
  │     └── overseas.fetchers  (HTTP 抓取器)
  │     │     └── overseas.config
  │     │     └── overseas.infra (限速/缓存/封锁检测)
  │     │           └── overseas.config
  │     └── overseas.compintel.compintel_db
  │     └── overseas.compintel.fcc_crawler
  │     │     └── overseas.fetchers
  │     │     └── overseas.infra
  │     └── overseas.compintel.eprel_crawler
  │     │     └── overseas.fetchers
  │     └── overseas.compintel.energystar_crawler
  │     │     └── overseas.fetchers
  │     └── overseas.compintel.anatel_crawler
  │     │     └── overseas.fetchers
  │     └── overseas.compintel.ic_ised_crawler
  │     │     └── overseas.fetchers
  │     └── overseas.compintel.cert_crawler
  │     └── overseas.compintel.alert (信号检测)
  │           └── overseas.compintel.compintel_db
  │     └── overseas.compintel.llm_analyze (LLM 解读，可选 openai)
  └── overseas.dict/spec_ja_zh.json  (日文词典，db.py 间接依赖)
  └── overseas.dict/spec_es_zh.json  (西语词典，db.py 间接依赖)
```

### 4.2 各文件职责与被依赖关系

| 文件 | 职责 | 被谁依赖 |
|------|------|----------|
| `config.py` | 配置管理（目录/路径/代理） | db / fetchers / infra / CLI |
| `models.py` | 数据模型定义 | db |
| `infra.py` | 限速/缓存/封锁检测 | fetchers / fcc_crawler |
| `db.py` | SQLite CRUD + schema 管理 | compintel_db / web_api / orchestrator / CLI |
| `fetchers.py` | HTTP GET/POST（代理/重试/UA轮换） | 所有 crawler / orchestrator |
| `compintel_db.py` | 竞品情报DB操作 | alert / web_api / orchestrator |
| `alert.py` | 信号检测引擎 | orchestrator |
| `fcc_crawler.py` | FCC 爬虫 | orchestrator |
| `eprel_crawler.py` | EPREL 爬虫 | orchestrator |
| `energystar_crawler.py` | ENERGY STAR 爬虫 | orchestrator |
| `anatel_crawler.py` | ANATEL 爬虫 | orchestrator |
| `ic_ised_crawler.py` | IC/ISED 爬虫 | orchestrator |
| `cert_crawler.py` | TÜV + VDE/CB 爬虫 | orchestrator |
| `orchestrator.py` | 抓取→落库→信号 编排 | CLI |
| `llm_analyze.py` | LLM 解读 | web_api (通过 API 调用) |
| `web_api.py` | Web API 服务 | CLI |
| `dashboard.html` | 前端看板 | web_api 返回 |
| `schema.sql` | 建表 SQL | db.init_schema() |

### 4.3 可移除的文件（非核心）

| 文件 | 说明 | 是否可删 |
|------|------|----------|
| `dict/spec_ja_zh.json` | 日文规格翻译词典 | ⚠️ db.py 间接引用目录，删除需确认无 import |
| `dict/spec_es_zh.json` | 西语翻译词典 | 同上 |
| `llm_analyze.py` | LLM 解读 | ✅ 可删（看板 AI 简报功能不可用，其他正常） |

---

## 5. 数据库依赖

| 依赖 | 说明 |
|------|------|
| **SQLite** | Python 内置 `sqlite3` 模块，无需单独安装 |
| 数据库文件 | `data/overseas.db`（25MB），含全部已抓取数据 |
| Schema | `overseas/schema.sql`（建表 SQL），`compintel_db.py` 动态创建 compintel 相关表 |

数据库表结构详见 [HANDOVER.md](HANDOVER.md) 第 3 章。

---

## 6. 部署检查清单

部署前逐项检查：

- [ ] Python 3.10+ 已安装（`py -3.12 --version`）
- [ ] `code/overseas/` 目录完整（24 个 .py 文件 + 1 个 .sql + 1 个 .html + 2 个 .json）
- [ ] `data/overseas.db` 存在（25MB）
- [ ] `code/data/overseas.db` 软链接或副本已建立
- [ ] `OVERSEAS_PROXY` 环境变量已设置（如需抓取）
- [ ] `PYTHONIOENCODING=utf-8` 已设置（Windows）
- [ ] `pip install openai` 已执行（如需 LLM 功能）
- [ ] 代理服务可用（如需抓取，`curl -x http://127.0.0.1:7877 https://www.google.com`）

验证命令：
```bash
cd code
py -3.12 compintel_cli.py compintel-alert --limit 3
# 输出 3 条信号即部署成功
```
