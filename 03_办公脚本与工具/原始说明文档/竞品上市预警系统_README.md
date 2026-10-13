# 预警相关抓取功能

> **竞品上市预警系统** — 自动抓取全球 7 个权威认证数据库，监控 6 家电视/显示器竞品品牌的新品认证动态，提前生成上市预警信号。

---

## 快速部署（3 步启动）

### 第 1 步：环境准备

```bash
# 进入功能目录
cd "C:\Users\likunyuan\Desktop\爬取项目\预警相关抓取功能\code"

# 设置环境变量（Windows PowerShell）
$env:PYTHONIOENCODING = "utf-8"
$env:OVERSEAS_PROXY = "http://127.0.0.1:7877"    # 出口代理（访问海外 API 必需）
```

> 数据库已内置在 `code/data/overseas.db`（25MB，851条记录），无需额外准备。

### 第 2 步：验证数据

```bash
py -3.12 db_stats.py
```

### 第 3 步：启动看板

```bash
py -3.12 compintel_cli.py compintel-web --port 8088
```

浏览器访问 **http://127.0.0.1:8088**

---

## 系统总览

| 指标 | 数值 |
|------|------|
| 数据源 | 7 个（FCC/EPREL/ENERGY STAR/ANATEL/IC-ISED/TÜV/VDE-CB） |
| 竞品品牌 | 6 家（Samsung/LG/Hisense/TCL/Sony/Panasonic） |
| 总记录 | 851 条（全部近1年认证） |
| 总信号 | 980 条 |
| 上市预警 | 129 条 |
| 代码量 | ~5000 行 Python + 347 行 HTML |
| 数据库 | SQLite 25MB |

---

## 目录结构

```
预警相关抓取功能/
│
├── README.md                ← 本文档（快速部署 + 总览）
├── DEVELOPER_GUIDE.md       ← ⭐ 开发者对接指南（逐函数代码逻辑+数据库设计+扩展指南）
├── PITFALLS_AND_DESIGN.md   ← ⭐ 误区/设计思路/注意事项（踩坑总结+设计决策+避坑清单）
├── DEPENDENCIES.md          ← 所有依赖项详细说明
├── HANDOVER.md              ← 详细交接文档（代码逐文件说明 + 数据库 + 未实现 + 问题）
├── COMMANDS.md              ← 全部执行命令手册
│
├── code/                    ← 全部源代码
│   ├── compintel_cli.py     ← 独立 CLI 入口（零外部依赖）
│   ├── db_stats.py         ← 数据库统计脚本
│   ├── alert_details.py    ← 上市预警详情查看
│   ├── crawl_log.py        ← 抓取批次日志查看
│   ├── dedup_records.py    ← 清理重复记录
│   └── overseas/            ← Python 包
│       ├── db.py            ← 数据库管理
│       ├── fetchers.py      ← HTTP 抓取器
│       ├── infra.py         ← 基础设施
│       ├── config.py        ← 配置管理
│       ├── models.py        ← 数据模型
│       ├── schema.sql       ← 建表 SQL
│       └── compintel/       ← 竞品情报模块（核心）
│           ├── alert.py          ← 信号检测引擎
│           ├── compintel_db.py   ← DB 操作
│           ├── fcc_crawler.py    ← FCC 爬虫
│           ├── eprel_crawler.py  ← EPREL 爬虫
│           ├── energystar_crawler.py
│           ├── anatel_crawler.py
│           ├── ic_ised_crawler.py
│           ├── cert_crawler.py   ← TÜV + VDE/CB
│           ├── orchestrator.py  ← 编排器
│           ├── llm_analyze.py    ← LLM 解读
│           ├── web_api.py        ← Web API
│           └── dashboard.html    ← 前端看板
│
├── data/
│   └── overseas.db         ← SQLite 数据库
│
└── docs/                   ← 辅助文档
    ├── FUNCTIONALITY.md    ← 功能介绍
    ├── DEPLOY.md           ← 部署指南
    ├── REPORT.md           ← 实现报告
    └── TODO.md             ← 待开发功能
```

---

## 核心功能

1. **7 数据源抓取**：FCC(美)/EPREL(欧盟)/ENERGY STAR(美)/ANATEL(巴西)/IC-ISED(加)/TÜV(欧盟)/VDE-CB(欧盟)
2. **上市预警生成**：产品类型关键词过滤 + 在售比对 + 提前量计算（FCC 最提前 211 天）
3. **子品牌识别**：LG→HELGI/elgato，TCL→THOMSON，Sony→SONY
4. **Web 看板**：5 维筛选（类型/源/时间/品牌/条数）+ 统计卡片 + 信号表格
5. **LLM 解读**：单条信号 + 品牌趋势简报（聚好看 deepseek-v4-flash）
6. **防重复入库**：INSERT 前查重，已存在则跳过

---

## 常用命令速查

| 场景 | 命令 |
|------|------|
| 启动看板 | `compintel-web --port 8088` |
| 一键更新近1年 | `compintel-run-extended` |
| 抓 FCC | `compintel-fcc` |
| 抓 EPREL | `compintel-eprel-supplier` |
| 查上市预警 | `compintel-alert --type launch_warning` |

> 开发者对接见 [DEVELOPER_GUIDE.md](DEVELOPER_GUIDE.md)
> 误区与设计见 [PITFALLS_AND_DESIGN.md](PITFALLS_AND_DESIGN.md)
> 详细命令见 [COMMANDS.md](COMMANDS.md)
> 依赖说明见 [DEPENDENCIES.md](DEPENDENCIES.md)
> 交接文档见 [HANDOVER.md](HANDOVER.md)
