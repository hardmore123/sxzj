# 预警相关抓取功能 — 全部执行命令手册

> **文档说明**：本手册列出系统的**全部可执行命令**，含环境变量、参数说明、实际示例和预期输出。
> 所有命令在 `预警相关抓取功能/code/` 目录下执行，数据库默认 `data/overseas.db`。

---

## 目录

1. [环境准备](#1-环境准备)
2. [Web 看板命令](#2-web-看板命令)
3. [单数据源抓取命令](#3-单数据源抓取命令)
4. [全量更新命令](#4-全量更新命令)
5. [信号查看命令](#5-信号查看命令)
6. [API 接口命令（curl）](#6-api-接口命令curl)
7. [数据库操作命令](#7-数据库操作命令)
8. [LLM 分析命令](#8-llm-分析命令)
9. [完整运维流程示例](#9-完整运维流程示例)
10. [命令速查表](#10-命令速查表)

---

## 1. 环境准备

### 1.1 必须设置的环境变量

```powershell
# Windows PowerShell
$env:PYTHONIOENCODING = "utf-8"                    # 必须：Windows 中文编码
$env:OVERSEAS_PROXY = "http://127.0.0.1:7877"       # 必须：出口代理（访问海外 API）
```

```bash
# Linux / macOS
export PYTHONIOENCODING=utf-8
export OVERSEAS_PROXY=http://127.0.0.1:7877
```

### 1.2 进入工作目录

```powershell
cd "C:\Users\likunyuan\Desktop\爬取项目\预警相关抓取功能\code"
```

### 1.3 数据库（已内置，开箱即用）

> 数据库已内置在 `code/data/overseas.db`（25MB，含 851 条已爬取记录），无需额外准备。

```powershell
# 验证数据库已就位
py -3.12 db_stats.py
```
**预期输出**：
```
============================================================
数据库统计
============================================================
  总记录: 851
  总信号: 980
  上市预警: 129
  竞品品牌: 6
  ...
```

### 1.4 验证环境

```powershell
# 检查 Python 版本
py -3.12 --version
```
**预期输出**：`Python 3.12.x`

```powershell
# 检查模块可加载
py -3.12 -c "from overseas.compintel.compintel_db import CompIntelDB; print('OK')"
```
**预期输出**：`OK`

```powershell
# 检查代理可用性（如需抓取）
curl.exe -x http://127.0.0.1:7877 -s -o NUL -w "%{http_code}" https://www.google.com
```
**预期输出**：`200`

### 1.5 可选依赖安装

```powershell
# LLM 解读功能（可选）
pip install openai
```

---

## 2. Web 看板命令

### 2.1 启动看板（前台运行）

```powershell
py -3.12 compintel_cli.py compintel-web --port 8088
```
**预期输出**：
```
启动竞品情报看板 Web 服务...
启动竞品情报看板: http://127.0.0.1:8088
```
**访问地址**：`http://127.0.0.1:8088`

### 2.2 启动看板（后台运行，不阻塞终端）

```powershell
# Windows — 独立进程
Start-Process -FilePath "py" -ArgumentList "-3.12","compintel_cli.py","compintel-web","--port","8088" -WindowStyle Hidden
```

```bash
# Linux / macOS — 后台运行
nohup py -3.12 compintel_cli.py compintel-web --port 8088 > web.log 2>&1 &
```

### 2.3 指定数据库路径

```powershell
py -3.12 compintel_cli.py compintel-web --db ..\data\overseas.db --port 8088
```

### 2.4 指定其他端口

```powershell
py -3.12 compintel_cli.py compintel-web --port 9090
```

### 2.5 停止看板

```powershell
# Windows — 按端口杀进程
Get-NetTCPConnection -LocalPort 8088 -State Listen | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

```bash
# Linux — 按端口杀进程
kill $(lsof -t -i:8088)
```

### 2.6 验证看板已启动

```powershell
# 检查 API
Invoke-WebRequest -Uri "http://127.0.0.1:8088/api/stats" -UseBasicParsing | Select-Object -ExpandProperty Content
```
**预期输出**：
```json
{"total_records": 851, "alerts": 980, "total_brands": 6, ...}
```

```powershell
# 检查首页
Invoke-WebRequest -Uri "http://127.0.0.1:8088/" -UseBasicParsing | Select-Object StatusCode, @{N='Length';E={$_.Content.Length}}
```
**预期输出**：`200` / `17224`

---

## 3. 单数据源抓取命令

### 3.1 FCC（美国无线电认证）

```powershell
# 默认抓取近1年（365天）
py -3.12 compintel_cli.py compintel-fcc
```
**预期输出**：
```
FCC: ok=310 fail=0 信号=320
```

```powershell
# 指定品牌（逗号分隔）
py -3.12 compintel_cli.py compintel-fcc --brands Samsung,LG

# 指定起始日期
py -3.12 compintel_cli.py compintel-fcc --since 2025-06-01

# 全量历史抓取
py -3.12 compintel_cli.py compintel-fcc --since 1900-01-01

# 全量模式（按 Grantee Code 分组翻页）
py -3.12 compintel_cli.py compintel-fcc --full --max-pages 5

# 指定授权码
py -3.12 compintel_cli.py compintel-fcc --full --gc-codes A3L BEJ
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--brands` | 空（全6品牌） | 逗号分隔的品牌名 |
| `--since` | 近365天 | 起始日期 YYYY-MM-DD |
| `--full` | False | 全量模式（按 Grantee Code 翻页） |
| `--max-pages` | 0（不限） | 每个授权码最多翻几页 |
| `--gc-codes` | 全部 | 指定授权码列表 |

### 3.2 EPREL（欧盟能效注册）

```powershell
# 方式1：供应商搜索（⭐ 推荐，近1年新品）
py -3.12 compintel_cli.py compintel-eprel-supplier
```
**预期输出**：
```
EPREL供应商搜索: ok=53 fail=0 信号=106 (since=2025-10-27)
```

```powershell
# 指定品牌
py -3.12 compintel_cli.py compintel-eprel-supplier --brands Samsung,LG

# 指定起始日期
py -3.12 compintel_cli.py compintel-eprel-supplier --since 2026-01-01

# 方式2：列表抓取（最新25条/品牌）
py -3.12 compintel_cli.py compintel-eprel --brands Sony

# 方式3：全量扫描（按注册号区间，慢）
py -3.12 compintel_cli.py compintel-eprel-scan --start 380000 --end 420000 --step 5
```

| 命令 | 参数 | 说明 |
|------|------|------|
| `compintel-eprel-supplier` | `--brands`, `--since` | ⭐ 推荐：按供应商名搜索，含子品牌多关键词 |
| `compintel-eprel` | `--brands`, `--details` | 列表抓取（最新25条/品牌） |
| `compintel-eprel-scan` | `--start`, `--end`, `--step`, `--since` | 按注册号区间遍历扫描（慢） |

**EPREL 供应商搜索关键词**（内置 `orchestrator.py` 第485行 `EPREL_SUPPLIER_SEARCH`）：
```
Samsung → ["Samsung"]
LG      → ["LG", "HELGI", "elgato"]     # 子品牌
Hisense → ["Hisense"]
TCL     → ["TCL", "THOMSON"]            # 子品牌
Sony    → ["Sony", "SONY"]
Panasonic → ["Panasonic"]
```

### 3.3 ENERGY STAR（美国能效认证）

```powershell
# 默认近1年
py -3.12 compintel_cli.py compintel-energystar
```
**预期输出**：
```
ENERGY STAR: ok=38 fail=0 信号=55 (since=2025-10-27)
```

```powershell
# 指定品牌
py -3.12 compintel_cli.py compintel-energystar --brands Samsung,Sony

# 指定起始日期
py -3.12 compintel_cli.py compintel-energystar --since 2025-01-01

# 全量历史
py -3.12 compintel_cli.py compintel-energystar --since 1900-01-01
```

### 3.4 ANATEL（巴西电信认证）

```powershell
# 默认近1年（下载53MB ZIP-CSV后本地过滤）
py -3.12 compintel_cli.py compintel-anatel
```
**预期输出**：
```
ANATEL: ok=223 fail=0 信号=224 (since=2025-10-27)
```

```powershell
# 指定品牌
py -3.12 compintel_cli.py compintel-anatel --brands Samsung,LG

# 指定起始日期
py -3.12 compintel_cli.py compintel-anatel --since 2025-06-01
```

> **注意**：ANATEL 首次会下载 53MB ZIP-CSV 文件，网络慢时需几分钟。

### 3.5 IC/ISED（加拿大无线电认证）

```powershell
# 默认近1年
py -3.12 compintel_cli.py compintel-ic-ised
```
**预期输出**：
```
IC/ISED: ok=2 fail=0 信号=2 (since=2025-10-27)
```

```powershell
# 指定品牌
py -3.12 compintel_cli.py compintel-ic-ised --brands Panasonic

# 指定起始日期
py -3.12 compintel_cli.py compintel-ic-ised --since 2025-01-01
```

> **注意**：ISED API 不稳定，频繁返回 500。品牌间有 15s 延迟。

### 3.6 TÜV + VDE/CB（欧盟安全认证）

```powershell
# 同时抓 TÜV 和 VDE/CB
py -3.12 compintel_cli.py compintel-certs
```
**预期输出**：
```
TÜV+VDE: ok=225 fail=0 信号=273
```

```powershell
# 指定品牌
py -3.12 compintel_cli.py compintel-certs --brands Samsung,Panasonic
```

> **注意**：TÜV 和 VDE/CB 的 model/auth_date 全空（数据源 API 不返回）。

---

## 4. 全量更新命令

### 4.1 一键全量更新（⭐ 日常推荐）

```powershell
# 默认抓取近1年，覆盖全部5个扩展源（ENERGY STAR + ANATEL + IC-ISED + TÜV/CB + EPREL）
py -3.12 compintel_cli.py compintel-run-extended
```
**预期输出**：
```
============================================================
竞品情报全量更新完成
============================================================
  energystar   ok=38   fail=0    信号=55
  anatel       ok=223  fail=0    信号=224
  ic_ised      ok=2    fail=0    信号=2
  tuv_ul       ok=225  fail=0    信号=273
  eprel        ok=53   fail=0    信号=106
  总信号: 660 条
```

```powershell
# 指定品牌
py -3.12 compintel_cli.py compintel-run-extended --brands Samsung,LG,Sony

# 指定数据源子集（逗号分隔）
py -3.12 compintel_cli.py compintel-run-extended --sources energystar,anatel,eprel

# 指定起始日期
py -3.12 compintel_cli.py compintel-run-extended --since 2026-01-01
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--brands` | 全6品牌 | 逗号分隔 |
| `--sources` | 全5源 | 可选：energystar/anatel/ic_ised/tuv_ul/eprel |
| `--since` | 近365天 | YYYY-MM-DD |

### 4.2 完整全量抓取（首次部署）

```powershell
# 第1步：FCC（单独抓，需翻页）
py -3.12 compintel_cli.py compintel-fcc --since 1900-01-01

# 第2步：其余5源一键更新
py -3.12 compintel_cli.py compintel-run-extended --since 1900-01-01
```

### 4.3 分源全量抓取

```powershell
py -3.12 compintel_cli.py compintel-energystar --since 1900-01-01
py -3.12 compintel_cli.py compintel-anatel --since 1900-01-01
py -3.12 compintel_cli.py compintel-ic-ised --since 1900-01-01
py -3.12 compintel_cli.py compintel-certs
py -3.12 compintel_cli.py compintel-eprel-supplier --since 1900-01-01
```

---

## 5. 信号查看命令

### 5.1 查看全部信号

```powershell
# 默认显示50条
py -3.12 compintel_cli.py compintel-alert
```
**预期输出**：
```
共 50 条信号 (since=全部, type=全部)
  [eprel_new_reg  ] Sony       2422459         EPREL 新注册: #2422459 | SONY | FW-85BZ30P | 能效 E
  [eprel_new_reg  ] Sony       2422462         EPREL 新注册: #2422462 | SONY | FW-43BZ35P | 能效 F
  ...
```

```powershell
# 显示100条
py -3.12 compintel_cli.py compintel-alert --limit 100
```

### 5.2 按信号类型查看

```powershell
# 只看上市预警
py -3.12 compintel_cli.py compintel-alert --type launch_warning --limit 50
```
**预期输出**：
```
共 50 条信号 (since=全部, type=launch_warning)
  [launch_warning ] Sony       2422459         [上市预警] Sony 新电视/显示器 #2422459 | EPREL | ...
  [launch_warning ] Samsung    40053022        [上市预警] Samsung 新电视/显示器 #40053022 | ul | ...
  ...
```

```powershell
# 只看 FCC 新认证
py -3.12 compintel_cli.py compintel-alert --type new_fcc_id --limit 50

# 只看 EPREL 新注册
py -3.12 compintel_cli.py compintel-alert --type eprel_new_reg --limit 50

# 只看其他新认证
py -3.12 compintel_cli.py compintel-alert --type new_cert --limit 50
```

### 5.3 按时间过滤

```powershell
# 2026年以来的信号
py -3.12 compintel_cli.py compintel-alert --since 2026-01-01

# 近1月
py -3.12 compintel_cli.py compintel-alert --since 2026-09-27
```

### 5.4 参数汇总

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--since` | 空（全部） | 起始日期 YYYY-MM-DD |
| `--type` | 空（全部） | launch_warning/new_fcc_id/new_cert/eprel_new_reg |
| `--limit` | 50 | 最多显示条数（前20条详细打印） |

---

## 6. API 接口命令（curl）

> 启动 Web 服务后（`compintel-web`），可用 HTTP 请求查询数据。

### 6.1 库统计

```powershell
curl.exe http://127.0.0.1:8088/api/stats
```
**预期输出**：
```json
{
  "total_records": 851,
  "alerts": 980,
  "total_brands": 6,
  "alerts_by_type": {"launch_warning": 129, "new_fcc_id": 310, "new_cert": 488, "eprel_new_reg": 53},
  "alerts_by_source": {"fcc": 320, "eprel": 106, "energystar": 55, "anatel": 224, "ic_ised": 2, "tuv": 89, "ul": 184}
}
```

### 6.2 信号列表（5维过滤）

```powershell
# 全部信号
curl.exe "http://127.0.0.1:8088/api/alerts?limit=5000"

# 按类型过滤
curl.exe "http://127.0.0.1:8088/api/alerts?type=launch_warning&limit=5000"

# 按数据源过滤
curl.exe "http://127.0.0.1:8088/api/alerts?source=eprel&limit=100"

# 按品牌过滤
curl.exe "http://127.0.0.1:8088/api/alerts?brand=Samsung&limit=100"

# 按时间过滤（近365天）
curl.exe "http://127.0.0.1:8088/api/alerts?days=365&limit=5000"

# 组合过滤
curl.exe "http://127.0.0.1:8088/api/alerts?type=launch_warning&source=eprel&brand=Sony&days=365&limit=50"
```

| 参数 | 说明 |
|------|------|
| `type` | launch_warning / new_fcc_id / new_cert / eprel_new_reg |
| `source` | fcc / eprel / energystar / anatel / ic_ised / tuv / ul |
| `brand` | Samsung / LG / Hisense / TCL / Sony / Panasonic |
| `days` | 近 N 天（与 since 互斥） |
| `since` | 起始日期时间戳（与 days 互斥） |
| `limit` | 返回条数（默认 5000） |

### 6.3 认证记录

```powershell
# 按数据源查记录
curl.exe "http://127.0.0.1:8088/api/records?source=fcc&limit=10"

# 按品牌查记录
curl.exe "http://127.0.0.1:8088/api/records?brand=Samsung&limit=10"
```

### 6.4 历史快照

```powershell
# 某个 source_id 的历史变化记录
curl.exe "http://127.0.0.1:8088/api/history?source=fcc&source_id=2A3LJ-RVU"
```

### 6.5 触发抓取（通过 API）

```powershell
# 触发 FCC + EPREL 抓取
curl.exe "http://127.0.0.1:8088/api/run?applicants=Samsung,LG&eprel_brands=Samsung,LG&details=0"
```

### 6.6 LLM 解读

```powershell
# 单条信号解读（默认 launch_warning 类型）
curl.exe "http://127.0.0.1:8088/api/analyze?mode=single&type=launch_warning&limit=5"

# 品牌趋势简报
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&brand=Samsung&limit=20"
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&brand=LG&limit=20"
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&brand=Sony&limit=20"

# 全部品牌趋势
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&limit=129"
```

---

## 7. 数据库操作命令

### 7.1 查看数据库统计（推荐方式）

系统内置了 `db_stats.py` 脚本，可一键查看全部统计：

```powershell
py -3.12 db_stats.py
```
**预期输出**：
```
============================================================
数据库统计
============================================================
  总记录: 851
  总信号: 980
  上市预警: 129
  竞品品牌: 6

各数据源记录数:
  fcc            310 条
  anatel         223 条
  ul             136 条
  tuv             89 条
  eprel           53 条
  energystar      38 条
  ic_ised          2 条

各信号类型:
  eprel_new_reg           53 条
  launch_warning         129 条
  new_cert               488 条
  new_fcc_id             310 条

上市预警按品牌:
  Samsung     45 条
  Sony        33 条
  Panasonic   30 条
  LG          18 条
  TCL          2 条
  Hisense      1 条
```

```powershell
# 指定其他数据库路径
py -3.12 db_stats.py ..\data\overseas.db
```

### 7.2 查看上市预警详情

```powershell
# 显示前 10 条上市预警的品牌/数据源/产品类型/提前天数
py -3.12 alert_details.py
```
**预期输出**：
```
上市预警（前 5 条）：
品牌         数据源          认证ID            产品类型                          提前天数
---------------------------------------------------------------------------
Hisense    fcc          2A4A3-HL7602    display                        94d
TCL        fcc          2ACCJB247       projector                      53d
TCL        fcc          2ACCJB253       projector                      67d
LG         fcc          2BO3LCCRCSLIMH  rear seat entertainment        95d
...
```

```powershell
# 显示前 20 条
py -3.12 alert_details.py 20

# 指定其他数据库路径
py -3.12 alert_details.py 10 ..\data\overseas.db
```

### 7.3 查看抓取批次日志

```powershell
# 显示最近 20 次抓取批次
py -3.12 crawl_log.py
```
**预期输出**：
```
抓取批次日志（最近 5 次）：
 #ID 场景                     状态             成功     失败 开始时间
---------------------------------------------------------------------------
#212 eprel_signal_fix       completed     106      0 2026-10-09T07:52:44+00:00
#211 eprel_signal_fix       completed      53      0 2026-10-09T07:50:54+00:00
#210 eprel_recent           completed      53      0 2026-10-09T07:48:29+00:00
...
```

```powershell
# 显示最近 10 次
py -3.12 crawl_log.py 10
```

### 7.4 备份数据库

```powershell
# Windows
Copy-Item data\overseas.db "data\overseas_backup_$(Get-Date -Format yyyyMMdd).db"
```

```bash
# Linux
cp data/overseas.db data/overseas_backup_$(date +%Y%m%d).db
```

### 7.5 清理重复数据

```powershell
# 删除 compintel_record 表中同 source+source_id 的重复记录（保留最新一条）
py -3.12 dedup_records.py
```
**预期输出**（当前数据无重复）：
```
发现 0 个重复 source_id
清理完成：851 → 851 条（删除 0 条重复）
```

---

## 8. LLM 分析命令

### 8.1 通过 Web API 调用

```powershell
# 单条信号解读
curl.exe "http://127.0.0.1:8088/api/analyze?mode=single&type=launch_warning&limit=5"
```

```powershell
# 品牌趋势简报
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&brand=Samsung&limit=20"
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&brand=LG&limit=20"
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&brand=Sony&limit=20"
```

```powershell
# 全部上市预警趋势
curl.exe "http://127.0.0.1:8088/api/analyze?mode=trend&type=launch_warning&limit=129"
```

### 8.2 通过 Python 直接调用

```powershell
py -3.12 -c "import sys; sys.path.insert(0,'.'); from overseas.db import Database; from overseas.compintel.compintel_db import CompIntelDB; from overseas.compintel.llm_analyze import analyze_launch_warning; db=Database('data/overseas.db'); cidb=CompIntelDB(db); alerts=cidb.alerts_since(''); lw=[a for a in alerts if a['signal_type']=='launch_warning'][:3]; [print('---', analyze_launch_warning(a)) for a in lw]; db.close()"
```

> **注意**：LLM 功能需要 `pip install openai` 且能访问海信内网。不通时自动降级为规则式摘要。

---

## 9. 完整运维流程示例

### 9.1 首次部署

```powershell
# 1. 进入目录
cd "C:\Users\likunyuan\Desktop\爬取项目\预警相关抓取功能\code"

# 2. 设置环境变量
$env:PYTHONIOENCODING = "utf-8"
$env:OVERSEAS_PROXY = "http://127.0.0.1:7877"

# 3. 验证数据（数据库已内置，无需复制）
py -3.12 db_stats.py

# 4. 启动看板
py -3.12 compintel_cli.py compintel-web --port 8088

# 5. 浏览器访问 http://127.0.0.1:8088
```

### 9.2 日常更新（每日/每周）

```powershell
# 1. 设置环境变量
$env:PYTHONIOENCODING = "utf-8"
$env:OVERSEAS_PROXY = "http://127.0.0.1:7877"

# 2. 备份数据库
Copy-Item data\overseas.db "data\overseas_backup_$(Get-Date -Format yyyyMMdd).db"

# 3. 一键全量更新（近1年增量）
py -3.12 compintel_cli.py compintel-run-extended

# 4. 补抓 FCC（如果需要最新 FCC 数据）
py -3.12 compintel_cli.py compintel-fcc

# 5. 查看更新后的信号
py -3.12 compintel_cli.py compintel-alert --type launch_warning --limit 20

# 6. 重启看板
Get-NetTCPConnection -LocalPort 8088 -State Listen | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
Start-Process -FilePath "py" -ArgumentList "-3.12","compintel_cli.py","compintel-web","--port","8088" -WindowStyle Hidden
```

### 9.3 全量历史抓取（首次填充）

```powershell
# 全量抓取所有数据源的历史数据
py -3.12 compintel_cli.py compintel-fcc --since 1900-01-01
py -3.12 compintel_cli.py compintel-energystar --since 1900-01-01
py -3.12 compintel_cli.py compintel-anatel --since 1900-01-01
py -3.12 compintel_cli.py compintel-ic-ised --since 1900-01-01
py -3.12 compintel_cli.py compintel-certs
py -3.12 compintel_cli.py compintel-eprel-supplier --since 1900-01-01
```

### 9.4 只抓特定品牌

```powershell
# 只抓 Samsung 和 LG 的近1年数据
py -3.12 compintel_cli.py compintel-run-extended --brands Samsung,LG
py -3.12 compintel_cli.py compintel-fcc --brands Samsung,LG
```

### 9.5 只抓特定数据源

```powershell
# 只抓 EPREL + ENERGY STAR
py -3.12 compintel_cli.py compintel-run-extended --sources eprel,energystar

# 只抓 ANATEL
py -3.12 compintel_cli.py compintel-anatel
```

---

## 10. 命令速查表

| 场景 | 命令 |
|------|------|
| 启动看板 | `compintel-web --port 8088` |
| 一键更新近1年 | `compintel-run-extended` |
| 抓 FCC | `compintel-fcc` |
| 抓 EPREL | `compintel-eprel-supplier` |
| 抓 ENERGY STAR | `compintel-energystar` |
| 抓 ANATEL | `compintel-anatel` |
| 抓 IC/ISED | `compintel-ic-ised` |
| 抓 TÜV+VDE | `compintel-certs` |
| 查上市预警 | `compintel-alert --type launch_warning` |
| 查全部信号 | `compintel-alert --limit 100` |
| 全量历史 | 加 `--since 1900-01-01` |
| 指定品牌 | 加 `--brands Samsung,LG` |
| 库统计 API | `curl http://127.0.0.1:8088/api/stats` |
| 信号 API | `curl http://127.0.0.1:8088/api/alerts?type=launch_warning` |
| LLM 解读 | `curl http://127.0.0.1:8088/api/analyze?mode=trend&brand=Samsung` |
