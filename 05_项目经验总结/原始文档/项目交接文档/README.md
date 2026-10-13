# 海外电视数据爬取与竞品对阵分析平台 — 项目交接文档

> 文档版本: 2026-09-29  
> 项目路径: `C:\Users\likunyuan\Desktop\爬取项目`  
> 服务器部署: 51服务器 (192.168.x.x)

---

## 1. 项目概述

本项目是一个**海外电视产品数据爬取与竞品对阵分析平台**，覆盖日本、北美、南美、欧洲4大模块共21个国家/区域。系统从品牌官网和零售电商站自动爬取电视产品规格、价格、用户评论，经归一化映射后生成海信 vs 竞品的规格对阵分析，并通过AI生成优势/劣势/策略建议。

### 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                     前端 GUI (端口 3080)                       │
│                   DSH Web GUI / Vue.js                       │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP
┌──────────────────────────┴──────────────────────────────────┐
│                  后端 API (端口 8090)                         │
│            benchmark_server.py (Python/FastAPI-like)          │
│  - /api/models    型号列表+价格                                │
│  - /api/spec      规格对阵矩阵                                 │
│  - /api/ai-analysis AI优势分析                                │
│  - /api/sizes     尺寸列表                                    │
│  - /api/countries 国家选择器                                   │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP (urllib)
┌──────────────────────────┴──────────────────────────────────┐
│                 LLM API (端口 8082)                           │
│              xinghai-ultra 大模型服务                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    数据爬取层 (定时运行)                        │
│  SPEC分支: 品牌官网 lineup → 发现型号 → 规格                   │
│  RETAIL分支: 电商零售站 → 给定型号 → 抓价+网评                   │
│  COMPINTEL: 认证数据库(FCC/ANATEL/EPREL/IC-ISED/TUV/UL)       │
└──────────────────────────┬──────────────────────────────────┘
                           │ SQLite
┌──────────────────────────┴──────────────────────────────────┐
│              SQLite 数据库 (overseas_v4_20260929.db)          │
│                    129.5 MB, 25张表                            │
│  原始爬取层(12表) → 维度层(2表) → 映射层(2表) → 事实层(2表)      │
└─────────────────────────────────────────────────────────────┘
```

### 三个核心子系统

| 子系统 | 职责 | 代码位置 |
|--------|------|---------|
| **数据爬取** | 爬品牌官网规格+零售站价格评论 | `海外数据爬取_部署包\overseas\` |
| **认证智能(CompIntel)** | 爬7个认证数据库的竞品新品预警 | `compintel-deploy\overseas\compintel\` |
| **对阵分析** | 规格归一化+AI对阵分析+前端展示 | `竞品对阵分析_部署包\overseas\benchmark_server.py` |

---

## 2. 目录结构

```
项目交接文档/
├── README.md                    ← 本文件（主交接文档）
├── code_inventory.md             ← 代码清单与功能说明
├── database_schema.md             ← 数据库表结构详细说明
├── database_ddl.md                ← 数据库DDL+数据统计（自动导出）
├── features_implemented.md        ← 已实现功能清单
├── features_pending.md            ← 设计未实现功能+存在问题
├── deployment.md                  ← 部署与运维指南
├── 配置文件/
│   ├── requirements_crawl.txt     ← 爬取项目依赖
│   ├── requirements_benchmark.txt ← 对阵分析依赖
│   ├── CHANGELOG.md               ← 变更记录
│   ├── DEPLOY_crawl.md            ← 爬取部署说明
│   ├── MASTER_INVENTORY.md        ← 主清单
│   └── CLAUDE.md                  ← 项目规范
└── 数据库/
    └── (overseas_v4_20260929.db 需从源项目复制)
```

### 源项目目录结构

```
C:\Users\likunyuan\Desktop\爬取项目\
├── 海外/                          ← 源代码+源数据库(开发主目录)
│   ├── overseas/                   ← Python代码包
│   │   ├── api/                    ← 爬虫API服务
│   │   ├── compintel/              ← 认证智能体代码
│   │   ├── dict/                   ← 字典/配置
│   │   ├── sites/                  ← 站点适配器
│   │   └── fetchers.py             ← 核心抓取器(45KB)
│   ├── .kiro/skills/overseas-schema-gen/  ← Skill定义(活文档)
│   │   ├── SKILL.md                ← 统一入口
│   │   ├── spec/                   ← SPEC分支(品牌官网规格)
│   │   ├── retail/                 ← RETAIL分支(零售站价格评论)
│   │   └── ops/                    ← 运维分支
│   ├── overseas_v4_20260929.db     ← 源数据库(129.5MB)
│   └── data/                       ← 爬取数据/profile/cache
│
├── 海外数据爬取_部署包/             ← 爬取项目部署包(15GB)
│   ├── overseas/                   ← 部署代码副本
│   │   ├── benchmark_server.py      ← 对阵分析服务器(开发版)
│   │   └── benchmark_static/        ← 前端静态文件
│   ├── data/overseas.db            ← 爬取数据库(124.7MB)
│   ├── CRAWL_CONFIG.json           ← 爬取配置(136KB)
│   └── scripts/                    ← 运维脚本
│
├── 竞品对阵分析_部署包/             ← 对阵分析部署包(131MB)
│   ├── overseas/
│   │   ├── benchmark_server.py      ← 对阵分析服务器(部署版)
│   │   └── benchmark_static/
│   │       └── index.html           ← 前端页面(39KB)
│   ├── data/overseas_v4_20260929.db ← 部署数据库(129.5MB)
│   ├── run_benchmark_server.py     ← 启动脚本
│   └── 启动竞品对阵分析.bat          ← Windows启动批处理
│
├── compintel-deploy/               ← 认证智能部署包
│   ├── overseas/compintel/          ← 认证爬虫代码
│   └── compintel_cli.py            ← CLI入口
│
└── 海外电视爬取项目/                ← Git仓库(历史版本)
```

---

## 3. 数据库概览

### 三个数据库副本

| 数据库 | 路径 | 大小 | 用途 |
|--------|------|------|------|
| **源DB** | `海外\overseas_v4_20260929.db` | 129.5 MB | 开发主库，含全部25张表 |
| **爬取DB** | `海外数据爬取_部署包\data\overseas.db` | 124.7 MB | 爬虫写入库（无对阵分析表） |
| **部署DB** | `竞品对阵分析_部署包\data\overseas_v4_20260929.db` | 129.5 MB | 部署副本（从源DB复制） |

### 25张表分4层

| 层 | 表数 | 核心表 | 职责 |
|----|------|--------|------|
| **原始爬取层** | 12 | product, price_snapshot, spec_row, product_spec, review | 爬虫直接写入 |
| **维度层** | 2 | dim_product_model, dim_spec_parameter | 型号+参数维度 |
| **映射层** | 2 | dim_spec_item_map, dim_spec_normalization_map | 规格项映射+同义归一 |
| **事实层** | 2 | fact_spec_value, fact_benchmark_comparison | 归一化规格值+对阵结果 |

详见 `database_schema.md` 和 `database_ddl.md`。

---

## 4. 核心数据指标（截至2026-09-29）

| 指标 | 数值 |
|------|------|
| 覆盖国家/区域 | 21个（4大模块） |
| 型号总数(dim_product_model) | 6,493个 |
| 海信型号(our_brand=1) | 740个 |
| 竞品型号(our_brand=0) | 5,753个 |
| 价格快照(price_snapshot) | 14,971条 |
| 品牌官网规格行(spec_row) | 175,218行 |
| 零售站规格(product_spec) | 136,588行 |
| 归一化规格值(fact_spec_value) | 52,636条 |
| 规格项映射(dim_spec_item_map) | 1,974条 |
| 同义归一(dim_spec_normalization_map) | 9条 |
| 用户评论(review) | 94,303条 |
| 已接入站点(site) | 260个 |
| 可对阵分析国家 | 16/19（SE/NO/DK海信无零售型号） |

### 各区域规格覆盖率

| 区域 | 覆盖率 | | 区域 | 覆盖率 |
|------|--------|---|------|--------|
| JP 日本 | 100% | | UK 英国 | 74.2% |
| US 美国 | 50.1% | | EU 欧元区 | 52.4% |
| CA 加拿大 | 84.3% | | CH 瑞士 | 50.5% |
| MX 墨西哥 | 91.9% | | PL 波兰 | 65.6% |
| CO 哥伦比亚 | 41.7% | | SE 瑞典 | 36.8% |
| PE 秘鲁 | 45.5% | | RO 罗马尼亚 | 46.7% |
| CL 智利 | 38.7% | | NO 挪威 | 46.4% |
| GT 危地马拉 | 37.5% | | DK 丹麦 | 66.7% |
| BO 玻利维亚 | 31.6% | | BG 保加利亚 | 56.5% |
| | | | RS 塞尔维亚 | 41.0% |

---

## 5. 技术栈

| 组件 | 技术 | 说明 |
|------|------|------|
| 语言 | Python 3.12 | 核心语言 |
| 数据库 | SQLite | 无服务器，单文件 |
| 爬虫框架 | Playwright 1.49+ | JS渲染+反爬突破 |
| 前端 | HTML/JS + ECharts | 无框架，原生JS |
| 后端 | Python http.server | 无框架，纯标准库 |
| LLM | xinghai-ultra (星海ultra) | HTTP API调用 |
| 翻译 | argostranslate / NLLB | 离线日译中 |
| 导出 | openpyxl | Excel报表 |
| 浏览器 | Edge (Windows) / Chrome (Linux) | Playwright驱动 |

### 依赖安装

```bash
# 核心依赖（零第三方依赖，纯标准库）
pip install playwright>=1.49.0
pip install argostranslate==1.11.0
pip install openpyxl==3.1.5

# 可选（NLLB本地翻译）
pip install "transformers>=4.40,<5" "torch>=2.0" "sentencepiece>=0.1.99"
```

详见 `配置文件/requirements_crawl.txt`。

---

## 6. 环境变量

```bash
# 出口代理（不设直连失败）
OVERSEAS_PROXY=http://127.0.0.1:7877

# 持久化浏览器profile（缺它拿不到登录态）
OVERSEAS_USE_PROFILE=1

# Python IO编码
PYTHONIOENCODING=utf-8

# xinghai-ultra LLM凭据
OVERSEAS_XINGHAI_API_KEY=xxx
OVERSEAS_XINGHAI_USER_KEY=xxx

# Linux离线Chrome
OVERSEAS_BROWSER_EXECUTABLE=/path/to/chrome
```

---

## 7. 快速启动

### 对阵分析服务

```bash
cd 竞品对阵分析_部署包
py -3.12 run_benchmark_server.py --port 8090
# 或双击: 启动竞品对阵分析.bat
```

### 验证

```bash
# 健康检查
curl http://127.0.0.1:8090/api/health
# → {"status":"ok","ai_ready":true,"ai_model":"xinghai-ultra"}

# 日本线55寸型号
curl http://127.0.0.1:8090/api/models?region=JP&size=55
```

### 三服务启动

| 服务 | 端口 | 启动命令 |
|------|------|---------|
| 后端API | 8090 | `py -3.12 run_benchmark_server.py --port 8090` |
| LLM API | 8082 | xinghai-ultra服务（独立启动） |
| 前端GUI | 3080 | DSH Web GUI（独立启动） |

详见 `deployment.md`。

---

## 8. 文档索引

| 文档 | 内容 |
|------|------|
| `code_inventory.md` | 所有代码文件清单+功能说明 |
| `database_schema.md` | 25张表结构+字段说明+数据流转 |
| `database_ddl.md` | 完整建表DDL+数据统计（自动生成） |
| `features_implemented.md` | 已实现功能清单 |
| `features_pending.md` | 设计未实现功能+已知问题 |
| `deployment.md` | 部署运维指南 |
| `配置文件/` | 依赖、配置、变更记录等 |

---

## 9. 注意事项

1. **数据库同步**：源DB(`海外\`) → 部署DB(`竞品对阵分析_部署包\data\`) 需手动复制
2. **爬取DB不同步**：`海外数据爬取_部署包\data\overseas.db` 是爬虫写入库，与源DB结构有差异，新增数据需手动同步
3. **编辑前停服务**：编辑 `benchmark_server.py` 前必须 `Stop-Process python` + 删 `__pycache__`，否则Win32 EIO 1175错误
4. **DEFAULT_DB路径**：部署版用相对路径，开发版用绝对路径，同步代码时需修改
5. **经验沉淀**：每次重大操作后须在 `海外\.kiro\skills\overseas-schema-gen\spec\cases.jsonl` 追加记录
6. **SE/NO/DK未覆盖**：海信在瑞典/挪威/丹麦零售站无型号数据，需补抓
