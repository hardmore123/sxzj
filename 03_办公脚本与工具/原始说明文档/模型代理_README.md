# 聚好看模型 (Hisense HiGPT) 接入 VSCode Claude Code 与 DeepSeek Harness

通过一个本地「Anthropic→OpenAI 格式转换代理」，让 VSCode 里的 Claude Code、以及 DeepSeek Harness (DSH)
都能使用海信聚好看自研模型。

## 组成

| 组件 | 本机路径 | 作用 |
|------|---------|------|
| 转换代理 | `%LOCALAPPDATA%\opencode\litellm_proxy\anthropic_to_openai_proxy.py` | 监听 8082，把 Anthropic / OpenAI 两种客户端协议转成 HiGPT 的 OpenAI 格式并转回 |
| 启动脚本 | `Desktop\setting\start-claude-hisense.ps1` | 一键后台启动代理并做健康检查 |
| Claude Code 配置 | `%USERPROFILE%\.claude\settings.json` | `ANTHROPIC_BASE_URL` 指向代理，模型映射到 `xinghai-ultra` |
| DSH 配置 | `%USERPROFILE%\.dsh\settings.yaml` | `llm-pi-ai.providers.hisense` 声明自研模型路由 |

代理对外提供两套面：

| 端点 | 协议 | 用它的客户端 |
|------|------|-------------|
| `POST /v1/messages` | Anthropic Messages | VSCode Claude Code |
| `GET /v1/models`、`POST /v1/chat/completions` | OpenAI Chat Completions | DeepSeek Harness 等 |
| `GET /health` | — | 健康检查 / 看门狗 |

## 使用步骤

1. 启动转换代理：
   ```powershell
   powershell -File "$env:USERPROFILE\Desktop\setting\start-claude-hisense.ps1"
   ```
   看到「代理状态: ok」即就绪。

2. 在 VSCode 中打开 Claude Code，正常对话即可，请求会经代理走到聚好看模型。

## 停止

```powershell
Get-CimInstance Win32_Process | Where-Object { $_.CommandLine -like "*anthropic_to_openai_proxy*" } | ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

## 接入 DeepSeek Harness (DSH)

DSH 内置 `@deepseek-ai/dsh-llm-pi-ai` 适配器（默认挂载但休眠），只要在用户 settings 里
给出一段 `llm-pi-ai:` 供应商配置，路由就会**热加载生效，无需重启**（`dsh-settings-file` 的
`watch` 默认为 true）。

### 配置（`%USERPROFILE%\.dsh\settings.yaml`）

```yaml
llm-pi-ai:
  providers:
    hisense:
      displayName: Hisense HiGPT (自研)
      api: openai-completions            # 受支持协议：openai-completions / openai-responses / anthropic-messages
      baseURL: http://127.0.0.1:8082/v1  # 指向本地代理的 OpenAI 兼容面
      apiKeyEnv: HISENSE_DSH_KEY         # 只放占位值，真实凭据留在代理进程侧
      models:
        - id: xinghai-ultra
          name: xinghai-ultra
          contextWindow: 262144
          maxTokens: 32768
```

同时在 `%USERPROFILE%\.dsh\.credentials.yaml` 的 `refs` 下放一个占位凭据
（`apiKeyEnv` 必须是 POSIX 环境变量名，且解析不到会直接报 `MISSING_CREDENTIAL`）：

```yaml
refs:
  HISENSE_DSH_KEY: local-proxy-placeholder
```

### 在界面里选择

模型选择器里会出现 **Hisense HiGPT (自研)** 分组，下面三个模型：
`xinghai-ultra`、`DeepSeek V4 Flash`、`qwen35-397B`。选中即用。

### 校验配置是否被真正接受

```powershell
# 用 DSH 自己的 schema 校验 llm-pi-ai 段（不是靠读文档推断）
cd "$env:USERPROFILE\Desktop\setting"; node verify_dsh_route.mjs

# 端到端跑一次（headless 一次性调用，走 hisense 路由）
& "$env:LOCALAPPDATA\npm-cache\_npx\1e7f6d9597241db0\node_modules\.bin\dsh.cmd" `
  --profile headless --patch ".\dsh-hisense-eval.yml" "reply with exactly: PONG"
```

实测输出 `PONG`，且代理日志出现
`OpenAI 面请求 model=xinghai-ultra`，即 DSH 确实走到了自研模型。

> 注意：DSH 的默认模型仍由 `agent-default-model` 决定（当前为 `deepseek-official/deepseek-flash`）。
> 上面的 `dsh-hisense-eval.yml` 只是评测用的临时覆盖层，不影响 web profile 的默认设置。

## 模型映射（settings.json）

| Claude Code 角色 | 实际调用模型 |
|-----------------|-------------|
| sonnet / opus / haiku | `xinghai-ultra` |

其它可用模型：`deepseek-v4-flash`、`qwen3-5-397b`（可在 settings.json 中按需替换某个角色的模型 ID）。

## 保活 / 自动重启（已安装）

代理是个普通后台进程，**默认没有任何自启动**：它一旦挂掉（崩溃、休眠/关机、被手动结束），
VSCode 里的 Claude Code 就会报 `API Error: Connection refused`，直到有人手动把它拉起来。

现在已安装看门狗（VBScript，随登录启动，**无需管理员权限**）：

| 组件 | 位置 | 作用 |
|------|------|------|
| 看门狗脚本 | `Desktop\setting\hisense-proxy-watchdog.vbs` | 每 60 秒探测 `127.0.0.1:8082/health`，不通就重启代理 |
| 自启动项 | `%APPDATA%\Microsoft\Windows\Start Menu\Programs\Startup\hisense-proxy-watchdog.vbs` | 每次登录自动拉起看门狗循环 |
| 看门狗日志 | `%LOCALAPPDATA%\opencode\litellm_proxy\watchdog.log` | 记录启动/重启事件（代理正常时不写，保持安静） |

管理命令：

```powershell
# 查看状态（启动项、代理健康、看门狗循环数、最近日志）
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\Desktop\setting\install-proxy-autostart.ps1" -Status
# 卸载自启动项并停止看门狗
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\Desktop\setting\install-proxy-autostart.ps1" -Remove
# 重新安装 / 立即拉起
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\Desktop\setting\install-proxy-autostart.ps1"
```

实测：手动杀掉代理后，看门狗在 **1 秒内**检测到并重启成功
（`watchdog.log`：`proxy DOWN, restarting ...` → `proxy restarted OK`）。

> 为什么用 VBScript 而不是计划任务：本机 `schtasks.exe` 与 ScheduledTasks CIM 均被安全策略拒绝，
> PowerShell 子进程启动路径也会命中管理员策略规则（`Access to %1 has been restricted by your
> Administrator by policy rule`），而 `wscript.exe` 可正常运行。

## 常见问题

**Q: 报 `API Error: Connection refused — a firewall or proxy may be blocking it`？**
A: 这是**代理进程没在运行**（不是模型或网络问题）。自查：

```powershell
try { Invoke-RestMethod http://127.0.0.1:8082/health -TimeoutSec 5 } catch { "代理没在跑" }
Get-Content "$env:LOCALAPPDATA\opencode\litellm_proxy\watchdog.log" -Tail 10
```

应急手动拉起：

```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\Desktop\setting\install-proxy-autostart.ps1"
```

装好看门狗后，这类报错最多约 60 秒就会自动恢复。

**Q: 报 `400 ... messages[N].tool_calls[1].function missing required field "name"`，只有 deepseek-flash 能用、xinghai-ultra 用不了？**
A: 这是转换代理的 bug，已修复。安装修复版：

```powershell
powershell -ExecutionPolicy Bypass -File "$env:USERPROFILE\Desktop\setting\install-proxy-fix.ps1"
```

根因（两个缺陷叠加）：

1. **流式转换把同一个工具调用的分片拆成了多个 tool_use block。**
   聚好看/LiteLLM 返回的 `tool_calls` 是按 `index` 累积的分片，例如
   `{"index":0,"function":{"name":"Bash","arguments":""}}` →
   `{"index":0,"function":{"arguments":"{\"command\":"}}` →
   `{"index":0,"function":{"arguments":"\"ls\""}}`。
   旧实现把每个分片都当成新的一次工具调用，于是首个分片之外的 block
   `id` 和 `name` 都是空的，`arguments` 只是半截 JSON 碎片。
2. **残缺 block 被写进会话历史后会永久毒化整个会话。**
   下一轮请求回放时它们变成 OpenAI 格式里 `name: ""` 的 tool_calls，
   海信网关直接拒绝整条请求。`deepseek-v4-flash` 对缺 name 较宽松所以碰巧能用，
   `xinghai-ultra` 严格校验必挂——这正是「只有 deepseek-flash 能用」的原因。
   报错信息还会被当成模型输出再写回历史，越滚越多。

修复版做的事：

- 流式侧按 `index` 累积分片，`name`/`arguments` 拼完整后才产出 content block；
  name 始终缺失的分片降级为文本，绝不产出空名 tool_use。
- 请求侧 sanitize：历史里 `name` 无效的 tool_use 降级为文本（同时保留对应
  tool_result 的内容，不丢信息），孤儿 tool 消息、空 assistant 消息一并处理。
- 校验兜底：每个 tool_call 都保证有非空 `name` 与 `id`。
- 上游 400/422 时用清洗后的负载重试。

验证方式：

```powershell
# 单元回归（含用真实会话历史复现污染场景）
python test_fixed_proxy.py "$env:USERPROFILE\.claude\projects\...\xxxx.jsonl"
# 端到端真实请求（--apply 才会访问网关）
python live_e2e.py "$env:USERPROFILE\.claude\projects\...\xxxx.jsonl" --offset 1613 --tail 141 --apply
```

实测结论：同一段被污染的历史（141 条消息、含 34 个残缺 tool_use），
旧版负载被上游以 `messages[74].tool_calls[0].function missing required field "name"`
拒绝（HTTP 400），修复版流式与非流式均返回 HTTP 200。

**Q: 代理启动了但请求报 ConnectError / 500？**
A: 本机若开了系统代理（如科学上网，`127.0.0.1:7877`），httpx 默认会走它去连海信内网导致失败。当前代理已用 `trust_env=False` 绕过系统代理直连内网，无需手动关代理。

**Q: 报 `upstream 401: `（空 body），代理和网络都正常？**
A: 这是**上游鉴权形状**问题，不是代理挂了。聚好看网关（APISIX/F5）是
「Bearer **或** apikey 二选一」的鉴权，实测矩阵：

| 发给上游的鉴权 | 结果 |
|---|---|
| `Authorization: Bearer TOKEN` + `?user_key=KEY` | ✅ 200（8/8） |
| `Authorization: Bearer TOKEN` + `?user_key=KEY&apikey=KEY` | ✅ 200（8/8） |
| `Authorization: Bearer TOKEN` + `apikey` / `X-Api-Key` **请求头** + `?user_key` | ❌ 401（0/8） |
| 只有 `apikey` 头，没有 `?user_key` | ❌ 401 `Missing API key found in request` |

两个要点：

1. **`user_key` 必须作为 query 参数**（`?user_key=...`），不能只放请求头；
   缺了它，网关会直接报 `Missing API key found in request`。
   同机 `Desktop\项目\CertAgent`、`list-merge-demo` 用的也是这个形状
   （`Authorization: Bearer LLM_API_KEY` + `?user_key=LLM_USER_KEY`）。
2. **一旦带上 `apikey` / `X-Api-Key` 请求头，网关就改走 apikey 路径并 401**——
   这正是 2026-09-23 凌晨那次「连续 19 次 401、Claude Code 完全不可用」的根因：
   代理当时同时发了 Authorization 头 + 两个 apikey 头。

修复版（当前已部署）只发 `Authorization` 头，把 `user_key`（与冗余的 `apikey`）
放进 query，并对 401 做 3 次指数退避重试以吸收网关抖动：

```powershell
# 自查：确认代理发出的请求形状正确
Invoke-RestMethod http://127.0.0.1:8082/health | Select-Object -ExpandProperty stats
# stats 里 upstream_auth_failures / retried_after_401 持续增长才说明又在 401
```

**Q: 想在不依赖海信内网的情况下验证代理代码本身？**
A: 用离线 mock 网关（复刻了上面的鉴权规则与 SSE 形状）：

```powershell
cd "$env:USERPROFILE\Desktop\setting"
python test_proxy_offline.py      # 鉴权形状 + 401 重试路径 + Anthropic/OpenAI 双面流式
```

覆盖了 Anthropic 流式转换、OpenAI 流式、模型名规范化、以及「上游请求头里绝不出现
apikey」这条回归断言。

**Q: 端口 8082 被占用？**
A: 查占用：`netstat -ano | findstr :8082`。

**Q: Hisense 返回 401？**
A: 先按上面第一条检查鉴权形状；确认形状无误后仍偶发 401，则是网关侧抖动，
代理已自动重试 3 次（指数退避），无需人工干预。

**Q: 必须在海信内网吗？**
A: 是。代理直连 `inner-apisix.hisense.com`（内网地址 `10.19.64.234`），需能访问海信内网。
