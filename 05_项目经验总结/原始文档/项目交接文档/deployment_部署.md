# 部署与运维指南

> 文档版本：2026-09-29
> 适用范围：海外电视数据爬取与竞品对阵分析平台三大子系统（爬取 / 认证智能 CompIntel / 对阵分析）的全部部署与日常运维操作。
> 配套文档：[README.md](README.md)（系统总览）、`database_schema.md`（表结构）、`配置文件/DEPLOY_crawl.md`（爬取部署原文）。

---

## 一、环境要求

| 依赖 | 版本 / 配置 | 说明 |
|------|------------|------|
| Python | **3.12**（`py -3.12`） | 核心 语言，对阵分析/爬取/CompIntel 三者统一 |
| 操作系统 | Windows（开发）/ Linux（服务器 51） | 开发在 Windows，常驻服务部署在 51 服务器 |
| 浏览器 | Playwright 1.49+ + Edge（Windows）/ Chrome（Linux） | JS 渲染与反爬突破；Linux 须离线安装 Chrome |
| 网络 | **出口代理 `http://127.0.0.1:7877`** | 海外站点必需；不设直连 amazon.ca 等超时 |
| 数据库 | SQLite（内置） | 单文件，无需单独安装 |
| LLM | xinghai-ultra（星海 ultra） | 经本地转换代理 `http://127.0.0.1:8082/v1` 接入 |

**核心运行环境变量**（Windows / Linux 通用，缺失会导致抓取失败或中文乱码）：

```bash
OVERSEAS_PROXY=http://127.0.0.1:7877     # 出口代理，L3 站点必需
OVERSEAS_USE_PROFILE=1                   # 持久化浏览器 profile，缺它拿不到登录态
PYTHONIOENCODING=utf-8                    # Windows 下必需
# Linux 额外：
LANG=C.UTF-8
LC_ALL=C.UTF-8
OVERSEAS_BROWSER_EXECUTABLE=/path/to/chrome  # 服务器离线 Chrome 路径
OVERSEAS_HEADLESS=true                      # 服务器无头；调试登录态时设 false
```

> ⚠️ 51 服务器实测 **7877 代理当前未在监听**（2026-09-24 核实，无 clash/mihomo 进程）。海外采集前需另行确认代理服务已启动。

---

## 二、依赖安装

核心库**零第三方依赖**（纯标准库）。按能力按需安装：

```bash
# 1) 全量安装（推荐首次部署）
pip install -r requirements.txt

# 2) 按能力安装
pip install playwright>=1.49.0           # 浏览器抓取（L2/L3 站点必需）
pip install openpyxl==3.1.5               # 导出 xlsx
pip install argostranslate==1.11.0        # 离线日译中
pip install openai                        # CompIntel LLM 解读（可选）

# 3) 可选：NLLB 本地翻译（较重）
pip install "transformers>=4.40,<5" "torch>=2.0" "sentencepiece>=0.1.99"
```

或用 `pyproject.toml` 可编辑安装：

```bash
pip install -e ".[browser,xlsx]"                # 浏览器 + Excel
pip install -e ".[browser,xlsx,translate]"      # 再加离线翻译
```

### 浏览器（Linux 服务器离线安装）

服务器访问不了 Playwright CDN，须离线安装 Chrome：

```bash
pip install playwright                                          # 只装 Python 包，不下载 chromium
# 解压 chrome-linux64.zip 到 .browser/
export OVERSEAS_BROWSER_EXECUTABLE=/path/.browser/chrome-linux64/chrome
sudo python -m playwright install-deps chromium                 # 缺系统库时
```

Windows 本机直接用已装的 Edge，无需额外下载。

### 注意事项

- **不要把填好的 `overseas.env` 提交 Git**（含代理与 API 凭据），仅提交 `deploy/overseas.env.example` 模板。
- xinghai-ultra 的 API 凭据（`OVERSEAS_XINGHAI_API_KEY` / `OVERSEAS_XINGHAI_USER_KEY`）放在环境变量或 `.dsh/settings.yaml.imported`，不入库。
- LLM 依赖不通时系统会自动**降级为规则兜底**，不影响规格对比核心功能，仅 AI 对阵结论缺失。

---

## 三、数据库部署

### 三个数据库副本

| 数据库 | 路径 | 大小 | 用途 |
|--------|------|------|------|
| **源 DB** | `海外\overseas_v4_20260929.db` | 129.5 MB | 开发主库，含全部 25 张表 |
| **爬取 DB** | `海外数据爬取_部署包\data\overseas.db` | 124.7 MB | 爬虫写入库（**无对阵分析表**） |
| **部署 DB** | `竞品对阵分析_部署包\data\overseas_v4_20260929.db` | 129.5 MB | 部署副本（从源 DB 复制） |

### 部署 DB 复制方式

部署 DB 是源 DB 的副本，**必须用 SQLite backup API 复制，绝不直接 `cp`**：

```bash
python - <<'PY'
import sqlite3
src = '海外/overseas_v4_20260929.db'
dst = '竞品对阵分析_部署包/data/overseas_v4_20260929.db'
s = sqlite3.connect(src); d = sqlite3.connect(dst)
with d: s.backup(d)
d.close(); s.close(); print('ok', dst)
PY
```

> 直接 `cp` 在 WAL 模式下会漏掉未落盘事务，得到损坏副本。

### 爬取 DB → 源 DB 同步流程

爬取 DB 与源 DB **结构有差异**（爬取 DB 无对阵分析的事实层/映射层表），新增数据需手动同步：

1. 在源 DB 侧执行归一化与对阵分析表重建（生成 `fact_spec_value` / `fact_benchmark_comparison` 等）。
2. 用 SQLite backup API 把源 DB 整库复制到部署 DB。
3. 同步完成后验证完整性：
   ```bash
   python -c "import sqlite3;print(sqlite3.connect('竞品对阵分析_部署包/data/overseas_v4_20260929.db').execute('pragma integrity_check').fetchone())"
   # 期望 ('ok',)
   ```

详见 [README.md 第 9 节注意事项](README.md#9-注意事项)：源 DB → 部署 DB 需手动复制；爬取 DB 与源 DB 结构不同，新增数据须手动同步。

---

## 四、服务启动

三个服务端口分工：

| 服务 | 端口 | 启动方式 |
|------|------|---------|
| 后端 API（对阵分析） | 8090 | `run_benchmark_server.py` |
| LLM API（xinghai-ultra） | 8082 | 本地转换代理（DSH 注入配置） |
| 前端 GUI（DSH Web GUI） | 3080 | DSH web 启动 |

### 4.1 对阵分析服务（8090）

**启动命令**（在 `竞品对阵分析_部署包\` 下）：

```bash
py -3.12 run_benchmark_server.py --port 8090
# 或直接调模块
python -m overseas.benchmark_server --port 8090 --cors "*"
```

**启动批处理**（双击即可）：`竞品对阵分析_部署包\启动竞品对阵分析.bat`

```bat
@echo off
chcp 65001 >nul
cd /d "%~dp0"
echo   访问地址: http://127.0.0.1:8090/
echo   xinghai-ultra 经本地转换代理 http://127.0.0.1:8082/v1 接入
echo   无AI代理时自动走规则兜底，不影响规格对比功能
py -3.12 run_benchmark_server.py --port 8090
pause
```

`run_benchmark_server.py` 本身仅 12 行，负责切工作目录并调用 `overseas.benchmark_server.main()`：

```python
HERE = os.path.dirname(os.path.abspath(__file__))
os.chdir(HERE); sys.path.insert(0, HERE)
from overseas.benchmark_server import main; main()
```

`main()` 启动时自动执行 `_setup_xinghai()`：从 DSH settings（`.dsh/settings.yaml.imported`）读取 xinghai-ultra 配置，注入 `OVERSEAS_LLM_BASE_URL=http://127.0.0.1:8082/v1`、`model=xinghai-ultra`、`provider=xinghai`，并把 `uses_user_key` 置为 False（代理忽略客户端鉴权头，apiKey 用占位值 `local-proxy-placeholder` 即可）。

**健康检查**：

```bash
curl http://127.0.0.1:8090/api/health
# 期望：{"status":"ok","ai_ready":true,"ai_model":"xinghai-ultra","ai_base":"http://127.0.0.1:8082/v1"}
```

`ai_ready` 反映 `XINGHAI_API_KEY` 是否已注入；为 `false` 时 AI 走规则兜底。

### 4.2 LLM API 服务（8082）

xinghai-ultra **不单独部署服务**，而是经 DSH 本地转换代理 `http://127.0.0.1:8082/v1` 接入（OpenAI 兼容协议）。配置来源是 DSH settings（`.dsh/settings.yaml.imported`）。

**启动方式**：

- 由 DSH Harness 在启动前端 GUI（3080）时一并拉起本地转换代理（8082）。
- 若只跑对阵分析服务而不需前端 GUI，仍需先启动 DSH（`dsh web`）以激活 8082 转换代理，否则 AI 分析降级为规则兜底。
- 转换代理忽略客户端鉴权头，apiKey 用占位值即可。

**验证 8082 可达**：

```bash
curl http://127.0.0.1:8082/v1/models
# 期望返回模型列表，含 xinghai-ultra
```

### 4.3 前端 GUI（3080）

前端是 **DSH Web GUI / Vue.js**，通过 DSH Harness 启动：

```bash
# 在 DSH checkout 目录下
dsh web
# 或
pnpm run dev:web
```

启动后浏览器访问 `http://127.0.0.1:3080`。DSH Web GUI 是整个交互入口，其 `window.__DSH_BOOT__` 由 `dsh web` 注入；**不要用其他 Vite/服务器替换它**，否则前端拿不到启动上下文。

> ⚠️ `apps/web` 的 Vite entry 不是独立应用——只构建 shell，`dsh web` 负责注入 `window.__DSH_BOOT__`。client-plugin 的 HMR 热更新仅在 `pnpm run dev:web` 同时运行时才生效。

### 4.4 三服务联动验证

按顺序启动并做端到端验证：

1. **启动 8082 + 3080**：在 DSH checkout 下执行 `dsh web`（或 `pnpm run dev:web`），确认 8082 与 3080 均监听。
2. **启动 8090**：双击 `启动竞品对阵分析.bat`，或 `py -3.12 run_benchmark_server.py --port 8090`。
3. **健康检查**：
   ```bash
   curl http://127.0.0.1:8090/api/health         # 后端 ok
   curl http://127.0.0.1:8082/v1/models          # LLM 代理可达
   curl http://127.0.0.1:3080                    # 前端 200
   ```
4. **端到端验证**（日本线 55 寸）：
   ```bash
   curl "http://127.0.0.1:8090/api/models?region=JP&size=55"          # 型号列表
   curl "http://127.0.0.1:8090/api/spec?model_ids=1,2&region=JP"       # 规格对阵
   curl "http://127.0.0.1:8090/api/ai-analysis?model_ids=1&region=JP"  # AI 分析
   ```
5. **前端联调**：浏览器打开 `http://127.0.0.1:3080`，选择区域/尺寸 → 选型号 → 查看规格对阵 + AI 优势分析。若 AI 结论缺失但规格正常，说明 8082 未起，走规则兜底（非 bug）。

---

## 五、爬虫部署

爬虫统一入口是 `overseas.cli` 与 `overseas-schema-gen` skill（见 `海外\.kiro\skills\overseas-schema-gen\SKILL.md`），内部分三个分支自动分派：**SPEC**（品牌官网规格）、**RETAIL**（零售站价格评论）、**OPS**（全量运维）。

### 5.1 SPEC 分支

品牌官网 lineup → 发现型号 → 规格（日本六品牌：sharp_jp 等）：

```bash
python -m overseas.cli spec-crawl  --site sharp_jp
python -m overseas.cli spec-export --site sharp_jp --format xlsx --out SPEC_夏普日本.xlsx
python scripts/refresh_spec.py                  # 六品牌统一刷新 + 校验
python scripts/refresh_spec.py --export-only    # 只重导出（补词典后用）
```

Windows 包装：`scripts/refresh_spec.ps1`。中文翻译来自离线词典 `overseas/dict/spec_ja_zh.json`，未命中项记入 `spec_untranslated` 表；刷新脚本**默认把未翻译视为失败**（避免周报静默降级）。

### 5.2 RETAIL 分支

电商零售站 → 给定型号 → 抓价 + 网评。**亚马逊三站必须带 `--save-db`**，否则只出 CSV 不落库：

```bash
export OVERSEAS_PROXY=http://127.0.0.1:7877
export OVERSEAS_USE_PROFILE=1
export PYTHONIOENCODING=utf-8

# 小样验证
python scripts/crawl_ca_retail.py --site amazon_us --limit 3 --save-db

# 全量（预计 ~10.5 小时：US 2.7h -> MX 1.9h -> CA 5.9h）
python scripts/crawl_ca_retail.py --site amazon_us --save-db
python scripts/crawl_ca_retail.py --site amazon_mx --save-db
python scripts/crawl_ca_retail.py --site amazon_ca --save-db
```

> ⚠️ **串行执行，不要并行**。实测三站并行 11.2 分钟 vs 串行 7.3 分钟（9 型号）——并行更慢，且同站双进程会使浏览器 profile 加锁失败，并三倍抬高出口 IP 请求速率。

### 5.3 CompIntel 认证爬取

认证数据库（FCC / ANATEL / EPREL / IC-ISED / TÜV / ENERGY STAR / UL）的竞品新品预警：

```bash
# 一键全量更新（推荐日常）
py -3.12 compintel_cli.py compintel-run-extended

# 单数据源
py -3.12 compintel_cli.py compintel-fcc
py -3.12 compintel_cli.py compintel-eprel-supplier       # 推荐，近 1 年新品
py -3.12 compintel_cli.py compintel-energystar
py -3.12 compintel_cli.py compintel-anatel
py -3.12 compintel_cli.py compintel-ic-ised
py -3.12 compintel_cli.py compintel-certs               # TÜV + VDE/CB

# 查看信号
py -3.12 compintel_cli.py compintel-alert --limit 50
py -3.12 compintel_cli.py compintel-alert --type launch_warning --limit 20

# 启动 Web 看板
py -3.12 compintel_cli.py compintel-web --port 8088      # http://127.0.0.1:8088
```

默认抓近 1 年、覆盖 6 家竞品（Samsung, LG, Hisense, TCL, Sony, Panasonic）。`add_record` 已内置防重检查，重复抓取会跳过已存在的 source_id。CompIntel 看板里的"AI 趋势简报"调用聚好看 deepseek-v4-flash（需 `pip install openai` 且能访问海信内网 `inner-apisix.hisense.com`）。

---

## 六、定时任务与开机自启

### 51 服务器 supervisor 配置

51 服务器（10.18.87.51）上的常驻服务由 **systemd** 管理（非 supervisor）；supervisor 仅负责少数 uv 进程（如 `feishu-bot-server`，占用 8000 端口）。海外采集 `overseas-*` 实测**未部署为 systemd 服务**。

长跑爬取用守护脚本（支持重启与断点续跑）：

```bash
python monitor/supervisor_amazon_full.py
python monitor/watchdog_rebuild_amazon_full.py     # 另开终端
# 日志：monitor/logs/supervisor_amazon_full.log、watchdog_rebuild_amazon_full.log
```

Linux 定时任务（systemd timer）：

```bash
sudo cp deploy/overseas-*.service deploy/overseas-*.timer /etc/systemd/system/
sudo cp deploy/overseas.env.example /etc/overseas.env      # 填真实值
sudo systemctl daemon-reload
sudo systemctl enable --now overseas-spec.timer overseas-monitor.timer
systemctl list-timers --all | grep overseas
```

| 单元文件 | 作用 |
|---|---|
| `overseas-spec.service` + `.timer` | 每周刷新 SPEC 六品牌 |
| `overseas-monitor.service` + `.timer` | 站点健康监控 |
| `overseas-api.service` | 常驻后端 API |
| `kakaku_weekly_task.xml` | Windows 任务计划（周度 kakaku 价格） |
| `setup_linux.sh` | 一键安装脚本 |

Windows 周度价格趋势：导入 `deploy/kakaku_weekly_task.xml` 到任务计划程序。

### 开机自启服务清单（51 服务器实测 2026-09-24）

| 项目 | systemd 单元 | 端口 | 开机自启 | 断电自恢复 |
|------|-------------|------|---------|-----------|
| **CertAgent** | `certagent.service` | `127.0.0.1:8125`（nginx 8123 反代） | ✅ enable | ✅ `Restart=always`+`RestartSec=8`+`StartLimit*=0`+ready 探测 |
| **证书核对** | `certverify` | `127.0.0.1:8758`（nginx `/craw/`） | ✅ enable | ✅ `Restart=on-failure`+`RestartSec=5`+`StartLimit*=0`+ready 探测 |
| 解密服务 | `decryptor.service` | `0.0.0.0:8500` | ✅ enable+active | ⚠️ 未做 StartLimit 加固 |
| MySQL 8 | `mysql.service` | `0.0.0.0:3306` | ✅ enable | ✅ 系统默认 |
| Docker 容器 | `docker.service` | hisense-ams(:7000)、ams-mysql(:3307) | ✅ enable | ⚠️ 容器 restart policy 待核 |
| product-data-tools | `product-data-tools*.service` | `127.0.0.1:8000` | ✅ enable | ⚠️ 未确认 Restart |
| 产品平台 dashboard | `productagentplatform-dashboard.service` + `java-collector.timer` | `:8001` | ✅ enable | ⚠️ 未确认 |
| 进程守护 | `supervisor.service` | — | ✅ enable | ✅ 负责 uv 进程 |
| 飞书机器人 | `feishu-bot-server`（supervisor/uv，非 systemd） | `127.0.0.1:8000` | ⚠️ 待确认 | ⚠️ |
| 统一入口 | `nginx` | `:80` `:8123` | ✅ enable | ✅ 默认 |

> `overseas-*`（海外采集）实测**未部署为 systemd 服务**，`8000` 端口实际被 feishu-bot-server 占用。

### 重启保护机制

**certagent.service 加固**（防开机竞态）：

```ini
After=network-online.target
Wants=network-online.target
Restart=always
RestartSec=8
StartLimitIntervalSec=0          # 关闭「10s 内失败 5 次即永久 failed」
```

**certverify.service 加固**：

```bash
# ExecStartPre → cert_verify_ready.sh（依次等路由→解密8500→CERT_PROXY，各≤60s，超时仅告警，永远 exit 0）
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=0
TimeoutStartSec=240              # 默认 90s 会误杀
After=network-online.target
Wants=decryptor.service          # 弱依赖，缺失不连锁 failed
```

**断电恢复顺序**：`nginx` 与 `decryptor` 与各后端各自 `After=network`，抢跑风险由 `Restart` + `StartLimitIntervalSec=0` 兜住，无限重试直至拉起。

验证：

```bash
sudo systemctl is-active certagent certverify          # 期望 active
sudo systemctl show certverify -p NRestarts
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8758/   # 200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8123/   # 200
```

---

## 七、常用运维操作

### 7.1 数据库同步

源 DB → 部署 DB（必须用 SQLite backup API，见 [三、数据库部署](#三数据库部署)）：

```bash
python - <<'PY'
import sqlite3
s = sqlite3.connect('海外/overseas_v4_20260929.db')
d = sqlite3.connect('竞品对阵分析_部署包/data/overseas_v4_20260929.db')
with d: s.backup(d)
d.close(); s.close(); print('ok')
PY
```

备份（每日 / 每周）：

```bash
python - <<'PY'
import sqlite3, datetime
stamp = datetime.datetime.now().strftime('%Y%m%d')
for src, dst in [('data/overseas.db', f'data/db/backup/overseas_{stamp}.bak'),
                 ('data/mx/overseas_mx.db', f'data/db/backup/overseas_mx_{stamp}.bak')]:
    s = sqlite3.connect(src); d = sqlite3.connect(dst)
    with d: s.backup(d)
    d.close(); s.close(); print('ok', dst)
PY
```

### 7.2 代码同步（开发 → 部署）

- 部署包是按日期整体快照的，回滚 = 用上一版 zip 解压覆盖，然后重跑自检。
- **DEFAULT_DB 路径差异**：部署版 `benchmark_server.py` 用相对路径，开发版用绝对路径，同步代码时需修改。
- 同步 `benchmark_server.py` 后必须先停服务 + 删 `__pycache__` 再重启（见 [7.3](#73-清理缓存__pycache__)）。

### 7.3 清理缓存（`__pycache__`）

**编辑 `benchmark_server.py` 前必须停 Python 进程 + 删 `__pycache__`**，否则触发 Win32 EIO 1175 错误（见 [8.1](#81-win32-eio-1175错误)）：

```powershell
# Windows
Get-Process python -ErrorAction SilentlyContinue | Stop-Process -Force
Get-ChildItem -Path . -Recurse -Directory -Filter '__pycache__' | Remove-Item -Recurse -Force
```

```bash
# Linux
pkill -f benchmark_server || true
find . -type d -name '__pycache__' -exec rm -rf {} +
```

### 7.4 全量重抓

长跑守护式全量抓取（支持重启与断点续跑）：

```bash
python monitor/supervisor_amazon_full.py
python monitor/watchdog_rebuild_amazon_full.py     # 另开终端
```

> ⚠️ 长时爬取**必须** `--save-db`、**串行**（单出口 IP）、先备份 DB、用 supervisor 断点续抓。失败型号（failed/no_item/no_price/no_reviews）直接跳，不无限重试。原则："缺价 > 错价"，宁留空不造假价。

### 7.5 导出报告

```bash
# SPEC 导出 xlsx
python -m overseas.cli spec-export --site sharp_jp --format xlsx --out SPEC_夏普日本.xlsx

# 日本线周度价格趋势
python scripts/weekly_kakaku_price.py             # 采集本周快照
python scripts/gen_jp_price_trend_xlsx.py         # 生成 xlsx（含折线图）
python scripts/gen_jp_price_trend_html.py         # 生成 html
```

> `price_shop_snapshot` 需要**连续多周运行**才能形成曲线。

---

## 八、故障排查

### 8.1 Win32 EIO 1175 错误

**现象**：编辑 `benchmark_server.py` 后服务起不来，Windows 报 `[WinError 1175] 没有可用的锁定冲突处理程序` 或 EIO。

**原因**：Python 进程仍占用 `__pycache__` 中的 `.pyc`，文件被锁。

**解决**（见 [README.md 第 9 节](README.md#9-注意事项)）：

```powershell
Get-Process python -ErrorAction SilentlyContinue | Stop-Process -Force
Get-ChildItem -Path . -Recurse -Directory -Filter '__pycache__' | Remove-Item -Recurse -Force
# 再重启服务
```

### 8.2 Playwright 403 WAF

**现象**：采集全部 `blocked`。

**排查**：

- 代理不通 / 出口 IP 被风控 → 先 `python -m overseas.cli retail-health` 探针。
- 登录态丢失 → 检查 `OVERSEAS_USE_PROFILE=1`；`data/browser_profile/` 是否被清。
- DOM 改版 → 先 `python -m overseas.cli probe --site amazon_ca --url https://www.amazon.ca/dp/<ASIN>` 定位坏掉的选择器，再改 JSON 适配器 —— **不要改 Python**。

**已知正常行为**（非 bug）：

- 价格全为 `None`：页面 buybox 不存在（"See All Buying Options"）。
- 评价 0 条：检查 `data-hook` 是否又改版（历史从 `review-title` 改成 `reviewTitle`）。

### 8.3 代理连接失败

**现象**：海外站点超时，`OVERSEAS_PROXY` 相关报错。

**排查**：

1. 确认 7877 在监听：`curl -x http://127.0.0.1:7877 https://www.amazon.ca`。
2. 51 服务器实测 7877 当前未监听（2026-09-24），需先启动 clash/mihomo 等代理进程。
3. IC/ISED 的 500 是正常的（API 不稳定），多试几次。
4. EPREL 抓取 0 条：API 限制只能取最新 25 条/品牌，确认 `--since` 参数近 1 年。
5. ANATEL 抓取很慢：下载 53MB ZIP-CSV，网络慢时需几分钟，本地过滤很快。

### 8.4 LLM 超时

**现象**：AI 对阵分析返回 `xinghai-ultra 调用失败` 或 `model: xinghai-ultra（未配置，规则兜底）`。

**排查**：

1. 确认 8082 转换代理可达：`curl http://127.0.0.1:8082/v1/models`。
2. 确认 DSH 已启动（`dsh web` 激活 8082）。
3. 确认 `.dsh/settings.yaml.imported` 中 xinghai-ultra 配置完整。
4. CompIntel 的 AI 趋势简报：确认 `pip install openai` 且能访问 `inner-apisix.hisense.com`（海信内网）。
5. LLM 不通时系统自动降级为规则式摘要，**不影响核心功能**。

### 8.5 数据库锁定

**现象**：`database is locked` 或 `database is malformed`。

**排查**：

1. 完整性检查：`python -c "import sqlite3;print(sqlite3.connect('data/overseas.db').execute('pragma integrity_check').fetchone())"`，期望 `('ok',)`。
2. 同站不要跑两个进程（profile 加锁失败 + DB 写锁竞争）。
3. 恢复备份：
   ```bash
   systemctl stop overseas-api overseas-monitor.timer
   cp data/db/backup/overseas_YYYYMMDD.bak data/overseas.db
   rm -f data/overseas.db-wal data/overseas.db-shm     # 关键：清残留 WAL
   python -c "import sqlite3;print(sqlite3.connect('data/overseas.db').execute('pragma integrity_check').fetchone())"
   systemctl start overseas-api overseas-monitor.timer
   ```

**其他常见现象**：

| 现象 | 排查 |
|---|---|
| 中文乱码 | `LANG=C.UTF-8`、`LC_ALL=C.UTF-8`、`PYTHONIOENCODING=utf-8` |
| 浏览器启动失败 | `OVERSEAS_BROWSER_EXECUTABLE` 是否指向存在的 chrome |
| profile 加锁失败 | 同站不要跑两个进程 |
| 启动 Web 服务 `Connection refused` | 端口被占用，换 `--port` |

---

## 九、监控指标

### 服务存活检查

```bash
# 三服务健康
curl -s http://127.0.0.1:8090/api/health | grep -o '"status":"[^"]*"'    # ok
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/v1/models # 200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3080           # 200

# 51 服务器 systemd 服务
sudo systemctl is-active certagent certverify decryptor mysql docker nginx
systemctl --failed                                                    # 应无本项目的 failed
sudo systemctl show certverify -p NRestarts                          # 重启次数
```

### 数据量监控

```bash
python - <<'PY'
import sqlite3
db = sqlite3.connect('竞品对阵分析_部署包/data/overseas_v4_20260929.db')
for t,sql in [
  ('dim_product_model','select count(*),sum(our_brand) from dim_product_model'),
  ('price_snapshot','select count(*) from price_snapshot'),
  ('spec_row','select count(*) from spec_row'),
  ('product_spec','select count(*) from product_spec'),
  ('fact_spec_value','select count(*) from fact_spec_value'),
  ('review','select count(*) from review'),
  ('site','select count(*) from site'),
]:
    print(t, db.execute(sql).fetchone())
PY
```

基准指标（截至 2026-09-29，见 [README.md 第 4 节](README.md#4-核心数据指标截至2026-09-29)）：

| 指标 | 基准值 |
|------|--------|
| 型号总数（dim_product_model） | 6,493 |
| 海信型号（our_brand=1） | 740 |
| 价格快照（price_snapshot） | 14,971 |
| 品牌官网规格行（spec_row） | 175,218 |
| 零售站规格（product_spec） | 136,588 |
| 归一化规格值（fact_spec_value） | 52,636 |
| 用户评论（review） | 94,303 |
| 已接入站点（site） | 260 |

数据量显著低于基准 → 触发全量重抓（见 [7.4](#74-全量重抓)）。

### 覆盖率监控

```bash
# 站点健康探针（爬取）
python -m overseas.cli retail-health

# 各区域规格覆盖率（对阵分析可对阵国家 16/19）
# SE/NO/DK 海信无零售型号，需补抓
```

各区域规格覆盖率基准（见 [README.md 第 4 节](README.md#4-核心数据指标截至2026-09-29)）：

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

覆盖率持续偏低 → 走 `overseas-schema-gen` skill 的 SPEC/RETAIL 分支补抓，并在 `cases.jsonl` 追加经验记录。

---

## 十、首次部署自检清单

按顺序执行，任一步失败先解决再继续：

```bash
python -m overseas.cli init-db              # 建表
python -m overseas.cli sites                # 期望输出 80 个站点
python SELFCHECK.py                         # 期望 36/36 PASS
python -m overseas.cli retail-health        # 站点健康探针
python -m overseas.cli probe --site amazon_ca --url https://www.amazon.ca/dp/<ASIN>   # 抽取链路验证
python -c "import sqlite3;print(sqlite3.connect('data/overseas.db').execute('pragma integrity_check').fetchone())"  # ('ok',)
```

---

## 十一、十一节前检查清单

> 移植自 51 服务器清单（2026-09-24 实测核实）。

- [ ] `certverify.service` + `cert_verify_ready.sh` 部署并 active
- [ ] `certagent.service` 加固（`network-online`+`StartLimit=0`+ready 探测）并 active
- [ ] `decryptor.service`、`nginx` 均 enabled / active
- [ ] `mysql`、`product-data-tools`、`productagentplatform-*` 自启恢复
- [ ] `systemctl --failed` 清理（当前有 `hi-ai-assistant` not-found 残留）
- [ ] curl 自检：8123=200、8125/health=200、8758=200
- [ ] 全站模拟一次断电后开机，确认关键服务自动拉起、无孤进程
- [ ] 7877 出海代理是否在监听（十一期间需海外采集时确认）

---

*本指南依据 `海外数据爬取_部署包/DEPLOY.md`、`compintel-deploy/DEPLOY.md`、`51服务器开机自启服务与重启保护清单_两项目汇总.md`、`竞品对阵分析_部署包/run_benchmark_server.py` 与 `启动竞品对阵分析.bat` 整合编写，并与 `README.md` 架构交叉核对。*
