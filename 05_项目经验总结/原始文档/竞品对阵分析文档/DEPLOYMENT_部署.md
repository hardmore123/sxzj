# 部署与运维详细指南

> 本文档详细说明竞品对阵分析系统的安装、启动、配置、运维和故障排查流程。
> 供接手开发者或运维人员参考。

---

## 一、环境要求

### 1.1 运行环境

| 项目 | 要求 | 说明 |
|------|------|------|
| 操作系统 | Windows 10+ / Linux | 跨平台支持 |
| Python | 3.10+（推荐3.12） | 使用了类型提示和 `from __future__ import annotations` |
| 磁盘空间 | 200 MB+ | 代码1MB + 数据库135MB + 依赖~60MB |
| 内存 | 512 MB+ | SQLite轻量级 |
| 网络 | 本地访问 | 服务默认绑定127.0.0.1 |

### 1.2 端口规划

| 服务 | 端口 | 用途 | 必需 |
|------|------|------|------|
| 后端API | 8090 | 对阵分析服务（型号/规格/AI分析） | ✅ 必需 |
| LLM代理 | 8082 | xinghai-ultra AI代理 | ⬜ 可选（不装走规则兜底） |
| 前端 | 8090 | 后端直接返回前端HTML | ✅ 同后端 |

### 1.3 网络要求

- **本地访问**：无需外网，全部本地运行
- **AI分析**：LLM代理需要访问海信内网 `https://inner-apisix.hisense.com/higpt-new/v1`
- **无AI时**：系统完全可用，AI分析走规则兜底返回结构化结果

---

## 二、依赖安装

### 2.1 后端服务依赖

后端服务（端口8090）**零第三方依赖**，仅用Python标准库：

```
argparse, json, os, sqlite3, sys, traceback,
http.server, urllib.parse, datetime, re, threading
```

无需 `pip install` 任何包即可运行。

### 2.2 LLM代理依赖（可选）

LLM代理（端口8082）需要三个第三方包：

```bash
pip install fastapi uvicorn httpx
```

| 包 | 版本 | 用途 |
|----|------|------|
| fastapi | >=0.104.0 | Web框架（接收请求） |
| uvicorn | >=0.24.0 | ASGI服务器 |
| httpx | >=0.25.0 | HTTP客户端（转发请求到海信API） |

### 2.3 完整安装

```bash
# 一键安装（Windows）
install.bat

# 手动安装
pip install -r requirements.txt

# 验证安装
python -c "import fastapi, uvicorn, httpx; print('LLM代理依赖OK')"
```

### 2.4 验证

```bash
# 验证后端可启动（不需要任何第三方包）
python run_benchmark_server.py --port 8090

# 验证数据库
python -c "import sqlite3; c=sqlite3.connect('data/overseas_v4_20260929.db'); print('OK', c.execute('SELECT COUNT(*) FROM dim_product_model').fetchone()[0])"
```

---

## 三、目录结构说明

```
竞品对阵分析/
│
├── run_benchmark_server.py          # 启动入口（12行，调用benchmark_server.main()）
├── requirements.txt                 # 依赖清单
├── install.bat                      # Windows一键安装
├── start.bat                        # Windows一键启动
├── README.md                         # 主交接文档
│
├── overseas/                         # Python后端代码包
│   ├── __init__.py                   # 包初始化（__version__ = "0.1.0"）
│   ├── benchmark_server.py           # 核心后端服务（2178行，端口8090）
│   ├── config.py                     # 全局配置（环境变量驱动，314行）
│   ├── llm_client.py                 # LLM客户端（258行，多provider支持）
│   ├── db.py                         # SQLite存储层（57KB）
│   ├── backfill.py                   # 参数补全工具（14KB）
│   ├── models.py                     # 数据模型定义（6KB）
│   └── benchmark_static/             # 前端静态文件目录
│       ├── index.html                # 对阵分析前端页面（39KB）
│       └── echarts.min.js            # ECharts图表库（1MB）
│
├── data/                             # 数据库目录
│   └── overseas_v4_20260929.db       # SQLite数据库（135MB）
│
├── llm_proxy/                        # LLM AI代理（可选）
│   └── anthropic_to_openai_proxy.py  # xinghai-ultra代理（1138行，端口8082）
│
└── docs/                             # 技术文档目录
    ├── SYSTEM_DESIGN.md              # 系统设计文档
    ├── API_REFERENCE.md              # API接口文档
    ├── FRONTEND_LLM.md              # 前端与LLM技术说明
    ├── DATABASE_DETAIL.md            # 数据库完整表结构
    ├── FEATURES.md                   # 功能说明
    ├── DATABASE.md                   # 数据库说明
    └── DEPLOYMENT.md                 # 本文档
```

### 关键路径说明

| 路径 | 来源 | 说明 |
|------|------|------|
| `data/overseas_v4_20260929.db` | `benchmark_server.py` 第122行 `DEFAULT_DB` | 数据库默认路径 |
| `overseas/benchmark_static/` | `benchmark_server.py` 第123行 `STATIC_DIR` | 前端静态文件目录 |

`DEFAULT_DB` 使用相对路径：
```python
DEFAULT_DB = os.path.join(
    os.path.dirname(os.path.dirname(os.path.abspath(__file__))),  # overseas/的父目录
    'data', 'overseas_v4_20260929.db'
)
```
即：`竞品对阵分析/data/overseas_v4_20260929.db`

---

## 四、服务启动

### 4.1 Windows一键启动

```bash
# 方式1：双击 start.bat
start.bat

# 方式2：命令行启动
py -3.12 run_benchmark_server.py --port 8090
```

### 4.2 Linux启动

```bash
# 安装依赖（仅LLM代理需要）
pip install -r requirements.txt

# 启动后端服务
python3 run_benchmark_server.py --port 8090

# 启动LLM代理（可选，另开终端）
python3 llm_proxy/anthropic_to_openai_proxy.py
```

### 4.3 启动参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--host` | `127.0.0.1` | 监听地址（0.0.0.0允许外部访问） |
| `--port` | `8090` | 监听端口 |
| `--db` | `data/overseas_v4_20260929.db` | 数据库路径 |
| `--cors` | `""` | CORS允许的origin |
| `--enable-scheduled-trace` | 关闭 | 启用每天凌晨2点定时溯源 |

### 4.4 LLM代理启动（可选）

```bash
# 单独启动LLM代理
python llm_proxy/anthropic_to_openai_proxy.py

# 代理监听 8082 端口
# 上游：https://inner-apisix.hisense.com/higpt-new/v1
# 模型：xinghai-ultra
```

### 4.5 启动验证

```bash
# 1. 健康检查
curl http://127.0.0.1:8090/api/health
# 期望返回：
# {"status":"ok","ai_ready":true,"ai_model":"xinghai-ultra","ai_base":"http://127.0.0.1:8082/v1"}

# 2. 国家列表
curl http://127.0.0.1:8090/api/countries
# 期望返回21个国家

# 3. 日本线55寸型号
curl http://127.0.0.1:8090/api/models?region=JP&size=55
# 期望返回型号列表

# 4. 浏览器访问
# http://127.0.0.1:8090/
```

---

## 五、环境变量配置

### 5.1 核心环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `OVERSEAS_DATA_DIR` | `data/` | 数据目录 |
| `OVERSEAS_DB_PATH` | `data/overseas.db` | 爬取数据库路径（非对阵分析用） |
| `PYTHONIOENCODING` | - | 建议设为`utf-8` |

### 5.2 LLM相关环境变量

以下变量由 `_setup_xinghai()` 自动注入，通常无需手动设置：

| 变量名 | 自动注入值 | 说明 |
|--------|-----------|------|
| `OVERSEAS_LLM_BASE_URL` | `http://127.0.0.1:8082/v1` | LLM代理地址 |
| `OVERSEAS_LLM_API_KEY` | `local-proxy-placeholder` | API Key（占位值） |
| `OVERSEAS_LLM_MODEL` | `xinghai-ultra` | 模型名 |
| `OVERSEAS_LLM_PROVIDER` | `xinghai` | Provider类型 |
| `OVERSEAS_LLM_STREAM` | `false` | 非流式模式 |
| `OVERSEAS_LLM_SSL_VERIFY` | `false` | 不验证SSL |
| `OVERSEAS_XINGHAI_BASE_URL` | `http://127.0.0.1:8082/v1` | xinghai专用地址 |
| `OVERSEAS_XINGHAI_API_KEY` | `local-proxy-placeholder` | xinghai API Key |
| `OVERSEAS_XINGHAI_USER_KEY` | `local-proxy-placeholder` | xinghai User Key |

### 5.3 自定义LLM配置

如需使用其他LLM（如DeepSeek公网API），设置以下环境变量后启动：

```bash
# 使用DeepSeek公网API
export OVERSEAS_LLM_PROVIDER=deepseek
export OVERSEAS_LLM_BASE_URL=https://api.deepseek.com
export OVERSEAS_LLM_API_KEY=your_deepseek_key
export OVERSEAS_LLM_MODEL=deepseek-chat

# 使用任意OpenAI兼容端点
export OVERSEAS_LLM_PROVIDER=openai_compatible
export OVERSEAS_LLM_BASE_URL=https://your-api-endpoint.com/v1
export OVERSEAS_LLM_API_KEY=your_key
export OVERSEAS_LLM_MODEL=your-model
```

---

## 六、数据库管理

### 6.1 数据库位置

```
data/overseas_v4_20260929.db  (135 MB)
```

### 6.2 数据库备份

```bash
# 备份
copy data\overseas_v4_20260929.db data\overseas_v4_backup.db

# SQLite在线备份
python -c "import sqlite3; src=sqlite3.connect('data/overseas_v4_20260929.db'); dst=sqlite3.connect('data/backup.db'); src.backup(dst); print('done')"
```

### 6.3 数据库更新

当源数据库有更新时（新数据爬取后），需同步到部署目录：

```bash
# 从源DB复制到部署DB
copy <源DB路径>\overseas_v4_20260929.db data\overseas_v4_20260929.db

# 重启服务使新数据生效
# Ctrl+C停止服务后重新启动
```

### 6.4 数据库查看

```bash
# 命令行查看
sqlite3 data/overseas_v4_20260929.db

# 关键查询
SELECT COUNT(*) FROM fact_spec_value;           -- 归一化规格值总数
SELECT COUNT(DISTINCT model_id) FROM fact_spec_value;  -- 覆盖型号数
SELECT COUNT(*) FROM dim_product_model;         -- 型号总数
SELECT * FROM dim_spec_parameter ORDER BY sort_order;  -- 18个参数
SELECT * FROM dim_spec_normalization_map;       -- 9条归一化映射
```

---

## 七、定时溯源（可选功能）

### 7.1 启用定时溯源

```bash
# 启动时加 --enable-scheduled-trace 参数
python run_benchmark_server.py --port 8090 --enable-scheduled-trace
```

### 7.2 功能说明

- 每天凌晨2点自动执行
- 扫描全库缺失参数的型号
- 尝试从其他来源补全规格
- 日志输出到控制台

### 7.3 手动触发

```bash
# API手动触发
curl -X POST http://127.0.0.1:8090/api/auto-trace
```

---

## 八、故障排查

### 8.1 服务无法启动

**症状**：`python run_benchmark_server.py` 报错

**排查步骤**：

```bash
# 1. 检查Python版本
python --version  # 需要 3.10+

# 2. 检查数据库文件
dir data\overseas_v4_20260929.db  # 确认文件存在

# 3. 检查端口占用
netstat -ano | findstr 8090

# 4. 查看错误日志
python run_benchmark_server.py 2>error.log
type error.log
```

### 8.2 前端页面空白

**症状**：浏览器访问 `http://127.0.0.1:8090/` 返回空白

**排查**：

```bash
# 1. 确认服务在运行
curl http://127.0.0.1:8090/api/health

# 2. 确认静态文件存在
dir overseas\benchmark_static\index.html

# 3. 检查静态目录路径
python -c "import os; print(os.path.join(os.path.dirname(os.path.abspath('overseas/benchmark_server.py')), 'benchmark_static'))"
```

### 8.3 AI分析返回fallback

**症状**：AI分析结果中 `fallback: true`

**原因**：LLM代理（8082）未运行或不可达

**解决**：

```bash
# 1. 检查8082端口
curl http://127.0.0.1:8082/v1/models

# 2. 启动LLM代理
pip install fastapi uvicorn httpx
python llm_proxy/anthropic_to_openai_proxy.py

# 3. 验证
curl http://127.0.0.1:8082/v1/models
# 期望返回模型列表
```

### 8.4 某区域无型号数据

**症状**：选择某国家后型号列表为空

**原因**：
1. 该区域无零售站数据
2. 价格快照货币不匹配
3. 型号名过滤规则误杀

**排查**：

```bash
# 检查该区域货币的price_snapshot
sqlite3 data/overseas_v4_20260929.db "SELECT COUNT(*) FROM price_snapshot WHERE currency='SEK'"

# 检查该区域有效型号
sqlite3 data/overseas_v4_20260929.db "SELECT DISTINCT p.model FROM product p JOIN price_snapshot ps ON ps.product_id=p.id WHERE ps.currency='SEK' AND LENGTH(p.model)>4 AND p.model NOT LIKE '%HTML%' LIMIT 10"
```

### 8.5 Win32 EIO 1175 错误

**症状**：编辑 `benchmark_server.py` 后服务无法启动

**原因**：Windows文件锁定，`__pycache__` 中的 `.pyc` 被占用

**解决**：

```bash
# 1. 停止所有Python进程
taskkill /F /IM python.exe

# 2. 删除 __pycache__
rmdir /s /q overseas\__pycache__

# 3. 重新启动
python run_benchmark_server.py
```

### 8.6 数据库锁定

**症状**：`sqlite3.OperationalError: database is locked`

**原因**：多个进程同时写入，或服务未正常退出

**解决**：

```bash
# 1. 停止所有Python进程
taskkill /F /IM python.exe

# 2. 检查-WAL文件
dir data\overseas_v4_20260929.db*

# 3. 删除-WAL和-SHM文件（会丢失未提交事务）
del data\overseas_v4_20260929.db-wal
del data\overseas_v4_20260929.db-shm

# 4. 重新启动
```

---

## 九、监控指标

### 9.1 服务存活检查

```bash
# 健康检查脚本
curl -s http://127.0.0.1:8090/api/health | python -c "import sys,json; d=json.load(sys.stdin); print('OK' if d['status']=='ok' else 'FAIL')"
```

### 9.2 数据量监控

| 指标 | 查询 | 当前值 |
|------|------|--------|
| 型号总数 | `SELECT COUNT(*) FROM dim_product_model` | 6,493 |
| 规格值总数 | `SELECT COUNT(*) FROM fact_spec_value` | 52,636 |
| 覆盖型号数 | `SELECT COUNT(DISTINCT model_id) FROM fact_spec_value` | 3,451 |
| 覆盖率 | 覆盖型号/型号总数 | 53.1% |
| 价格快照 | `SELECT COUNT(*) FROM price_snapshot` | 14,971 |

### 9.3 各区域覆盖率

| 区域 | 覆盖率 | | 区域 | 覆盖率 |
|------|--------|---|------|--------|
| JP | 100% | | UK | 74.2% |
| US | 50.1% | | EU | 52.4% |
| CA | 84.3% | | CH | 50.5% |
| MX | 91.9% | | PL | 65.6% |
| CO | 41.7% | | SE | 36.8% |
| PE | 45.5% | | RO | 46.7% |
| CL | 38.7% | | NO | 46.4% |
| GT | 37.5% | | DK | 66.7% |
| BO | 31.6% | | BG | 56.5% |
| | | | RS | 41.0% |

---

## 十、开机自启配置（Linux服务器）

### 10.1 systemd服务配置

创建 `/etc/systemd/system/benchmark-analysis.service`：

```ini
[Unit]
Description=竞品对阵分析服务
After=network.target

[Service]
Type=simple
User=www
WorkingDirectory=/opt/benchmark-analysis
ExecStart=/usr/bin/python3 run_benchmark_server.py --port 8090 --host 0.0.0.0
Restart=always
RestartSec=10
Environment=PYTHONIOENCODING=utf-8

[Install]
WantedBy=multi-user.target
```

### 10.2 启用服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable benchmark-analysis
sudo systemctl start benchmark-analysis
sudo systemctl status benchmark-analysis
```

### 10.3 LLM代理自启

创建 `/etc/systemd/system/llm-proxy.service`：

```ini
[Unit]
Description=LLM AI Proxy (xinghai-ultra)
After=network.target

[Service]
Type=simple
User=www
WorkingDirectory=/opt/benchmark-analysis
ExecStart=/usr/bin/python3 llm_proxy/anthropic_to_openai_proxy.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## 十一、日常运维清单

### 11.1 部署前检查

- [ ] Python 3.10+ 已安装
- [ ] `data/overseas_v4_20260929.db` 存在且可读
- [ ] `overseas/benchmark_static/index.html` 存在
- [ ] 端口8090未被占用
- [ ] LLM代理依赖已安装（如需AI功能）

### 11.2 启动后验证

- [ ] `curl http://127.0.0.1:8090/api/health` 返回 `{"status":"ok"}`
- [ ] `curl http://127.0.0.1:8090/api/countries` 返回21个国家
- [ ] 浏览器访问 `http://127.0.0.1:8090/` 页面正常
- [ ] 日本线55寸有型号返回
- [ ] AI分析正常返回（非fallback）

### 11.3 定期维护

- [ ] 每周检查服务存活
- [ ] 每月检查数据库大小
- [ ] 数据更新后重启服务
- [ ] 定期备份数据库
