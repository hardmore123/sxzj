# 通过 Hisense HiGPT 使用 Claude Code 技术方案

## 1. 背景

Claude Code 是 Anthropic 推出的 AI 编程助手 CLI 工具，它通过 Anthropic Messages API 与后端大模型通信。Hisense HiGPT 是海信内部的大模型平台，提供 OpenAI 兼容的 API 接口。两者的 API 格式不兼容：

| 项目 | Claude Code (Anthropic) | Hisense HiGPT (OpenAI 兼容) |
|------|------------------------|----------------------------|
| 请求端点 | `/v1/messages` | `/v1/chat/completions` |
| 请求格式 | Anthropic Messages 格式 | OpenAI Chat Completions 格式 |
| 响应格式 | Anthropic Messages 响应 | OpenAI Chat Completions 响应 |
| 工具调用 | `tool_use` / `tool_result` content block | `tool_calls` / `tool` 角色消息 |
| 流式协议 | Anthropic SSE 事件 | OpenAI SSE chunk |

项目中已有的 cc-switch 工具是一个 Claude Code 供应商切换管理器，支持代理转发，但**不支持 Anthropic↔OpenAI 格式转换**，直接转发会导致 404 错误。

## 2. 技术方案

### 2.1 整体架构

在 Claude Code 和 Hisense HiGPT 之间部署一个 **Anthropic→OpenAI 格式转换代理**，负责双向格式转换：

```
Claude Code                    转换代理                     Hisense HiGPT
    │                             │                             │
    │  Anthropic 格式请求          │                             │
    │  POST /v1/messages          │                             │
    │  ──────────────────────────>│                             │
    │                             │  OpenAI 格式请求             │
    │                             │  POST /v1/chat/completions   │
    │                             │  ──────────────────────────>│
    │                             │                             │
    │                             │  OpenAI 格式响应             │
    │                             │  <──────────────────────────│
    │  Anthropic 格式响应          │                             │
    │  <──────────────────────────│                             │
    │                             │                             │
```

### 2.2 技术选型

- **代理框架**：FastAPI + uvicorn（项目已有依赖，无需额外安装）
- **HTTP 客户端**：httpx（项目已有依赖，支持异步和流式）
- **无外部依赖**：因内网环境 pypi.org 不可达，无法安装 LiteLLM 等现成方案，采用自研轻量代理

### 2.3 创建产物清单

本次方案共创建/修改了以下文件，分为三类：**核心服务文件**、**脚本与文档文件**、**修改的已有配置文件**。

#### 2.3.1 核心服务文件（新建）

| 文件 | 绝对路径 | 类型 | 作用 |
|------|---------|------|------|
| 转换代理 | `C:\Users\user\AppData\Local\Temp\opencode\litellm_proxy\anthropic_to_openai_proxy.py` | Python 脚本 | Anthropic↔OpenAI 格式转换代理（FastAPI 服务，端口 8082） |
| LiteLLM 配置（未使用） | `C:\Users\user\AppData\Local\Temp\opencode\litellm_proxy\config.yaml` | YAML | 最初为 LiteLLM proxy 准备的配置，因 pypi 不可达改用自研代理，此文件保留但未使用 |
| 调试代理（临时） | `C:\Users\user\AppData\Local\Temp\opencode\litellm_proxy\debug_proxy.py` | Python 脚本 | 开发期间用于抓取请求/响应的调试工具，非生产文件 |

#### 2.3.2 脚本与文档文件（新建）

| 文件 | 绝对路径 | 类型 | 作用 |
|------|---------|------|------|
| 启动脚本 | `D:\HisenseWork\01_projects\tool_audio_test\bin\start-claude-hisense.ps1` | PowerShell 脚本 | 一键启动转换代理 + Claude Code |
| 使用说明 | `D:\HisenseWork\01_projects\tool_audio_test\bin\README.md` | Markdown | 面向使用者的简要说明 |
| 技术方案 | `D:\HisenseWork\01_projects\tool_audio_test\docs\hisense-claude-code-tech-spec.md` | Markdown | 本文档：完整技术方案与格式转换说明 |

#### 2.3.3 修改的已有配置文件

| 文件 | 绝对路径 | 修改内容 |
|------|---------|---------|
| Claude Code 配置 | `C:\Users\user\.claude\settings.json` | 将 `ANTHROPIC_BASE_URL` 从 cc-switch 代理（`http://127.0.0.1:15721`）改为转换代理（`http://127.0.0.1:8082`）；所有模型映射统一指向 `xinghai-ultra`；移除 `PROXY_MANAGED` 和 `[1M]` 后缀 |

#### 2.3.4 目录结构总览

```
C:\Users\user\
├── .claude\
│   └── settings.json                          ← [修改] Claude Code 配置，指向转换代理
│
├── AppData\Local\Temp\opencode\
│   └── litellm_proxy\                         ← [新建目录] 转换代理工作目录
│       ├── anthropic_to_openai_proxy.py        ← [新建] 核心转换代理（生产）
│       ├── config.yaml                         ← [新建] LiteLLM 配置（未使用）
│       ├── debug_proxy.py                      ← [新建] 调试工具（临时）
│       ├── .venv\                              ← [新建] 独立虚拟环境（未使用，改用项目 venv）
│       ├── proxy.log                           ← [运行时生成] 代理运行日志
│       └── debug.log                           ← [运行时生成] 调试日志
│
D:\HisenseWork\01_projects\tool_audio_test\
├── bin\
│   ├── start-claude-hisense.ps1               ← [新建] 一键启动脚本
│   └── README.md                              ← [新建] 使用说明
│
└── docs\
    └── hisense-claude-code-tech-spec.md       ← [新建] 本技术方案文档
```

## 3. 格式转换实现

### 3.1 请求转换（Anthropic → OpenAI）

#### 3.1.1 system 字段

Anthropic 的 `system` 是顶级字段，可以是字符串或 content block 数组；OpenAI 的 system 是 `messages` 数组中 role 为 `system` 的消息。

```python
# Anthropic 请求
{"system": "You are a helpful assistant", "messages": [...]}

# 转换为 OpenAI 请求
{"messages": [{"role": "system", "content": "You are a helpful assistant"}, ...]}
```

#### 3.1.2 消息内容

Anthropic 的 `content` 可以是字符串或 content block 数组（`text` / `tool_use` / `tool_result`）；OpenAI 的 `content` 是字符串，工具调用用独立的 `tool_calls` 字段。

| Anthropic content block | OpenAI 对应处理 |
|------------------------|----------------|
| `{"type": "text", "text": "..."}` | `content` 字符串 |
| `{"type": "tool_use", "name": "...", "input": {...}}` | `tool_calls` 数组 |
| `{"type": "tool_result", "tool_use_id": "...", "content": "..."}` | role 为 `tool` 的独立消息 |

#### 3.1.3 tools 定义

```python
# Anthropic tools 格式
{
    "name": "Bash",
    "description": "Execute a command",
    "input_schema": {"type": "object", "properties": {...}}
}

# 转换为 OpenAI tools 格式
{
    "type": "function",
    "function": {
        "name": "Bash",
        "description": "Execute a command",
        "parameters": {"type": "object", "properties": {...}}
    }
}
```

#### 3.1.4 tool_choice 转换

| Anthropic tool_choice | OpenAI tool_choice |
|----------------------|-------------------|
| `{"type": "auto"}` | `"auto"` |
| `{"type": "any"}` | `"required"` |
| `{"type": "tool", "name": "Bash"}` | `{"type": "function", "function": {"name": "Bash"}}` |

### 3.2 响应转换（OpenAI → Anthropic）

#### 3.2.1 非流式响应

```python
# OpenAI 响应
{
    "choices": [{
        "message": {
            "content": "Hello",
            "tool_calls": [{"id": "call_xxx", "function": {"name": "Bash", "arguments": '{"command":"ls"}'}}]
        },
        "finish_reason": "tool_calls"
    }],
    "usage": {"prompt_tokens": 100, "completion_tokens": 50}
}

# 转换为 Anthropic 响应
{
    "id": "msg_xxx",
    "type": "message",
    "role": "assistant",
    "content": [
        {"type": "text", "text": "Hello"},
        {"type": "tool_use", "id": "call_xxx", "name": "Bash", "input": {"command": "ls"}}
    ],
    "stop_reason": "tool_use",
    "usage": {"input_tokens": 100, "output_tokens": 50}
}
```

#### 3.2.2 流式响应

OpenAI 的 SSE chunk 和 Anthropic 的 SSE 事件结构完全不同，代理通过 `state` 字典跟踪当前 content block 状态，逐 chunk 转换：

| OpenAI SSE chunk | Anthropic SSE 事件 |
|-----------------|-------------------|
| 首个 chunk（含 role） | `message_start` |
| delta.content 文本 | `content_block_start` (首次) + `content_block_delta` (后续) |
| delta.tool_calls 新 ID | `content_block_stop` (关闭 text block) + `content_block_start` (tool_use block) |
| delta.tool_calls arguments | `content_block_delta` (type=`input_json_delta`) |
| finish_reason | `content_block_stop` + `message_delta` (stop_reason) |
| `[DONE]` | `message_stop` |

### 3.3 finish_reason 映射

| OpenAI finish_reason | Anthropic stop_reason |
|---------------------|----------------------|
| `stop` | `end_turn` |
| `length` | `max_tokens` |
| `tool_calls` | `tool_use` |

## 4. 配置说明

### 4.1 Claude Code settings.json

```json
{
    "env": {
        "ANTHROPIC_AUTH_TOKEN": "test",
        "ANTHROPIC_BASE_URL": "http://127.0.0.1:8082",
        "ANTHROPIC_DEFAULT_SONNET_MODEL": "xinghai-ultra",
        "ANTHROPIC_DEFAULT_SONNET_MODEL_NAME": "xinghai-ultra",
        "ANTHROPIC_DEFAULT_OPUS_MODEL": "xinghai-ultra",
        "ANTHROPIC_DEFAULT_OPUS_MODEL_NAME": "xinghai-ultra",
        "ANTHROPIC_DEFAULT_HAIKU_MODEL": "xinghai-ultra",
        "ANTHROPIC_DEFAULT_HAIKU_MODEL_NAME": "xinghai-ultra"
    }
}
```

| 环境变量 | 说明 |
|---------|------|
| `ANTHROPIC_BASE_URL` | 指向转换代理 `http://127.0.0.1:8082` |
| `ANTHROPIC_AUTH_TOKEN` | 代理不校验 token，任意值即可 |
| `*_MODEL` | 实际发给 API 的模型 ID（Hisense 端的模型名） |
| `*_MODEL_NAME` | Claude Code 界面显示名称 |

### 4.2 代理配置（anthropic_to_openai_proxy.py）

```python
HISENSE_BASE_URL = "https://inner-apisix.hisense.com/higpt-new/v1"
HISENSE_AUTH_TOKEN = "YiVcZbGaxXCEL49qL1KEiITOzANgVVzq"
HISENSE_API_KEY = "5z93eqr6995k8f8iikarjxghic2felpr"
DEFAULT_MODEL = "xinghai-ultra"
PROXY_PORT = 8082
```

### 4.3 可用模型

| 模型 ID | 显示名 |
|--------|--------|
| `xinghai-ultra` | xinghai-ultra |
| `deepseek-v4-flash` | DeepSeek V4 Flash |
| `qwen3-5-397b` | qwen35-397B |

## 5. 使用方式

### 5.1 一键启动

```powershell
# 先停 cc-switch（它会覆盖 settings.json）
Get-Process cc-switch -ErrorAction SilentlyContinue | Stop-Process -Force

# 运行启动脚本
powershell -File D:\HisenseWork\01_projects\tool_audio_test\bin\start-claude-hisense.ps1
```

启动脚本执行流程：

1. 检查端口 8082 是否已有代理在运行
2. 若未运行，后台启动转换代理
3. 验证代理健康状态（`/health` 端点）
4. 启动 Claude Code

### 5.2 手动启动

```powershell
# 1. 停 cc-switch
Get-Process cc-switch -ErrorAction SilentlyContinue | Stop-Process -Force

# 2. 启动转换代理
Start-Process -FilePath "D:\HisenseWork\01_projects\tool_audio_test\.venv\Scripts\python.exe" `
    -ArgumentList "-u", "C:\Users\user\AppData\Local\Temp\opencode\litellm_proxy\anthropic_to_openai_proxy.py" `
    -WindowStyle Hidden `
    -WorkingDirectory "D:\HisenseWork\01_projects\tool_audio_test"

# 3. 启动 Claude Code
claude
```

### 5.3 停止

```powershell
# 停止转换代理
Get-Process python | Where-Object { $_.CommandLine -like "*anthropic_to_openai_proxy*" } | Stop-Process -Force
```

## 6. 已知问题与限制

### 6.1 cc-switch 冲突

cc-switch 启动后会进入"代理接管"模式，覆盖 `~/.claude/settings.json`，把 `ANTHROPIC_BASE_URL` 改为 `http://127.0.0.1:15721`（cc-switch 自己的代理）。cc-switch 代理只做透明转发，不支持 Anthropic→OpenAI 格式转换。因此使用本方案前必须停止 cc-switch。

### 6.2 后台进程 401 问题

`Start-Process` 后台启动的 Python 进程调用 Hisense 端点时会间歇性返回 401，而前台运行同样代码正常。原因未完全定位，可能与进程启动方式有关。当前方案通过 `-WorkingDirectory` 参数指定工作目录规避。

### 6.3 Hisense 端点间歇性 401

Hisense 网关偶尔返回 401（可能与 F5 BIG-IP 负载均衡的 cookie 粘滞有关），重试即可。

### 6.4 不支持的功能

- 图片输入（Anthropic `image` content block）
- 缓存控制（`cache_control`）
- 批量请求（`/v1/messages/batches`）
- token 计数（`/v1/messages/count_tokens`）

## 7. 测试验证

### 7.1 非流式文本请求

```
请求: "你好，请回复一个字：好"
结果: HTTP 200, content="好", usage={input_tokens: 14, output_tokens: 1} ✅
```

### 7.2 流式文本请求

```
请求: 同上，stream=true
结果: 流式返回"好" ✅
```

### 7.3 工具调用（tool_use）

```
请求: "List files in the current directory"，带 Bash 工具定义
结果: stop_reason=tool_use, content=[{type: tool_use, name: Bash, input: {command: "ls"}}] ✅
```

### 7.4 工具结果回传（tool_result）

```
请求: 在 tool_use 后追加 tool_result，内容为 "file1.py\nfile2.py\nREADME.md\ntest_dir/"
结果: stop_reason=end_turn, 模型正确解读文件列表并回复 ✅
```
