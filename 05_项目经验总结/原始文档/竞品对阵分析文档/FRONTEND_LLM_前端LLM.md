# 前端交互与LLM代理技术说明

> 本文档基于以下源文件整理：
> - 前端页面：`竞品对阵分析/overseas/benchmark_static/index.html`（773 行）
> - LLM 客户端：`竞品对阵分析/overseas/llm_client.py`（258 行）
> - LLM 代理：`竞品对阵分析/llm_proxy/anthropic_to_openai_proxy.py`（1138 行）
> - 配置系统：`竞品对阵分析/overseas/config.py`（314 行）
> - 参数补全：`竞品对阵分析/overseas/backfill.py`（282 行）

---

## 一、前端页面架构

### 1.1 HTML结构

页面采用单文件 SPA 架构，`<head>` 内联全部 CSS，`<body>` 末尾内联全部 JavaScript，仅外链 `echarts.min.js` 一个库。

整体骨架（`<body>` 内）：

```
<div class="wrap">                         ← 最大宽度 1400px 居中容器
  <header>                                  ← 渐变深色顶栏（标题 + AI 状态指示 #ai-status）
  <div class="card">                        ← 卡片1：对阵条件选择
    .controls                               ← 国家选择器 + 尺寸选择器 + AI推荐按钮 + 刷新按钮
    全库自动溯源条 (#auto-trace-*)           ← 紫色渐变块，选尺寸+执行溯源
    #recommend                              ← AI 推荐结果区（默认 display:none）
  <div class="card">                        ← 卡片2：对阵对象选择
    #selected-bar                           ← 已选型号芯片条（dashed 边框黄底）
    #model-pool                             ← 型号池（flex wrap, max-height:200px 滚动）
    按钮组                                   ← 生成对阵分析 / 调用AI / 回溯补全 / 刷新对比表
  <div id="result-area" style="display:none"> ← 结果区（生成后显示，含4张卡片）
    <div class="card"> 核心指标 #metrics     ← 3 个 .metric 指标条
    <div class="card"> SPEC 归一化对比表 #spec-table  ← 18参数×N型号矩阵
    <div class="card" id="backfill-card">    ← 回溯补全结果（默认隐藏）
    <div class="card"> 能力雷达图 #radar     ← ECharts 容器
    <div class="card"> AI 对阵结论卡 #ai-conclusion-area  ← 优势/劣势/机会三栏
  <footer>
</div>
```

关键 DOM ID 一览：`sel-country`、`sel-size`、`recommend`、`model-pool`、`selected-bar`、`result-area`、`metrics`、`spec-table`、`backfill-card`、`backfill-results`、`radar`、`ai-conclusion-area`、`ai-status`、`auto-trace-status`、`auto-trace-result`。

### 1.2 CSS样式

#### 设计令牌（CSS 变量）

`:root` 定义全部配色，统一 `--bg / --card / --border / --text / --text2 / --text3` 基础色，以及语义色：

| 变量 | 值 | 语义 |
|---|---|---|
| `--our` / `--our-bg` | `#e23d2c` / `#fdf2f0` | 我方（海信）红色系 |
| `--lead` / `--lead-bg` | `#67c23a` / `#f0f9eb` | 领先绿色系 |
| `--weak` / `--weak-bg` | `#e6a23c` / `#fdf6ec` | 劣势黄色系 |
| `--gap` / `--gap-bg` | `#909399` / `#f4f4f5` | 缺失灰色系 |
| `--primary` / `--primary-bg` | `#2c6cf6` / `#ecf2fe` | 主色蓝色系（竞品/按钮） |

#### 关键样式选择器

- **卡片** `.card`：白底圆角 8px + 灰边框，`padding:16px 20px`，`margin-bottom:14px`
- **顶部控制条** `.controls`：`display:flex; flex-wrap:wrap; gap:12px; align-items:flex-end`
- **型号池** `.model-pool`：`display:flex; flex-wrap:wrap; gap:8px; max-height:200px; overflow-y:auto`
- **型号项** `.model-item` / `.model-item.selected` / `.model-item.our`：选中态蓝色高亮，我方型号红色边框
- **对比表** `table.spec`：`width:100%; border-collapse:collapse; font-size:12px`，表头 `position:sticky; top:0`
  - `td.our`（我方红底）、`td.lead`（绿色领先）、`td.weak`（黄色落后）、`td.gap`（灰色缺失）
  - `td.traced`（紫色溯源补全：`background:#f3e8ff; color:#7c3aed; font-style:italic`，`::before` 伪元素加「溯源」标签）
  - `.cat-tag` 按分类着色：画质绿、智能黄、游戏蓝、连接灰、音质红、设计绿
- **AI 结论** `.conclusion`：`display:grid; grid-template-columns:1fr 1fr 1fr; gap:12px`
  - `.concl-col.adv`（绿）、`.concl-col.dis`（黄）、`.concl-col.opp`（蓝），每项 `.concl-item` 白底卡片含置信度与溯源证据
- **雷达图** `#radar`：`width:100%; height:420px`

#### 响应式布局

```css
@media(max-width:768px){
  .conclusion{ grid-template-columns:1fr; }   /* 三栏结论堆叠为单栏 */
  .controls{ flex-direction:column; }          /* 控制条纵向排列 */
}
```

#### Loading / 空态动效

- `.loading::after`：14px 圆形边框 + `@keyframes spin` 旋转动画
- `.ai-loading`：`linear-gradient(90deg,#667eea,#764ba2,#667eea)` + `background-size:200%` + `@keyframes shine` 流光动画（AI 分析专属紫蓝渐变）
- `.empty`：居中灰色提示文字

### 1.3 JavaScript模块

全部 JS 在 `<script>` 内联，无模块化打包。全局状态与函数组织如下：

| 全局变量 | 用途 |
|---|---|
| `COUNTRIES` | 国家列表缓存 |
| `CUR` | 当前选择 `{region:'JP', size:65}` |
| `MODELS` | 当前型号池数据 |
| `SELECTED` | 已选型号 ID 数组 |
| `RADAR_CHART` | ECharts 实例 |
| `_generating` / `_loadingSpec` / `_tracing` | 防重入锁 |

核心函数：`init()`、`loadCountries()`、`loadSizes()`、`loadModels()`、`loadAIRecommend()`、`generateAnalysis()`、`loadSpecCompare()`、`renderSpecTable()`、`loadRadar()`、`callAI()`、`renderConclusion()`、`runAutoTrace()`、`backfillMissing()`、`renderBackfill()`、`triggerCrawl()`、`updateMetrics()`。

启动入口：页面加载后直接调用 `init()`。

---

## 二、前端交互流程

### 2.1 国家选择器

`init()` → `loadCountries()` 流程：

1. `fetch('/api/countries')` 获取国家列表，存入 `COUNTRIES`
2. 按 `c.group` 字段分组（`日本 / 北美 / 南美 / 欧洲 / 其他`）
3. 按固定顺序 `['日本','北美','南美','欧洲','其他']` 渲染 `<optgroup label="组名">`，每个国家渲染为 `<option value="${c.code}">${c.flag} ${c.name}（${c.currency}）</option>`
4. 默认选中日本（`sel-country.value = 'JP'`），触发 `onCountryChange()`
5. 加载失败显示「加载失败，请刷新」并 `throw`（被 `init` 的 catch 捕获，顶栏显示红字错误）

`onCountryChange()` 更新 `CUR.region` 后调用 `loadSizes()`。

### 2.2 尺寸联动

`loadSizes()` 流程：

1. `fetch('/api/sizes?region=${CUR.region}')` 获取该区域的可用尺寸
2. 渲染 `<option value="${s.size}">${s.size}寸（${s.model_count}型号）</option>`
3. 自动选中第一个尺寸（`CUR.size = sizes[0].size`）
4. 联动调用 `loadModels()` 加载型号池

`onSizeChange()` 清空 `SELECTED`、重绘已选条、调用 `loadModels()`。

### 2.3 型号列表渲染

`loadModels()` 流程：

1. 容器先显示 `.loading` 占位
2. `fetch('/api/models?region=${CUR.region}&size=${CUR.size}')` 获取型号池
3. 空数据显示「该区域该尺寸暂无型号数据」
4. 每个型号渲染为 `.model-item`：
   ```html
   <div class="model-item [selected] [our]" data-mid="${model_id}" onclick="toggleModel(id)">
     <input type="checkbox" ${checked}>
     <b>${brand} ${model_code}</b>
     <span class="price">¥${price.toLocaleString()}</span>
     [我方标记]
   </div>
   ```
   - `is_our` 为真的型号加 `our` class（红色边框）
   - 已选型号加 `selected` class（蓝色高亮）+ checkbox 勾选

`toggleModel(id)` 切换选中态，最多 5 个（超出 `alert` 拦截），只更新 CSS 不重载列表。`renderSelected()` 渲染顶部已选芯片条（`.sel-chip.our` 红色 / `.sel-chip.comp` 蓝色，带 × 移除按钮）。

### 2.4 规格对阵矩阵

`generateAnalysis()` 入口（防重入 `_generating`），要求 `SELECTED.length >= 2`：

1. 显示 `#result-area`
2. 先 `loadSpecCompare()` 渲染表格（快）
3. `await requestAnimationFrame()` 等一帧确保 DOM 可见
4. 再 `loadRadar()` 渲染雷达图（需容器已可见）

`loadSpecCompare()` 流程（防重入 `_loadingSpec`）：

1. `fetch('/api/spec?model_ids=${SELECTED.join(',')}')`
2. 返回 `{models, spec_matrix}`，`spec_matrix` 是 `{param_code: {param_name, category, is_numeric, higher_is_better, values: {model_code: {display, numeric}}}}` 结构
3. **缺失参数检测**：遍历每个型号每个参数，`display` 为空 / `—` / 含「待维护」则记入 `missingMap`，存到 `window._pendingTrace`，显示「N个参数待溯源」提示并显示「回溯补全待维护」按钮
4. 调用 `renderSpecTable(models, sm)`

`renderSpecTable()` 渲染逻辑：

- 表头：`参数 | 分类 | 型号1 | 型号2 | ...`，我方型号表头红底
- 18 个参数固定顺序（见下方常量数组）
- 每格着色规则：
  - 有 `trace_source` → `traced` 紫色（溯源补全）
  - 数值型且 `higher_is_better` 明确 → 计算 max/min，最大值标 `lead`（↑领先），最小值标 `weak`（↓落后），`SCREEN_SIZE` 例外不参与比较
  - 我方型号无其他标记时加 `our` 红底
  - 空值或「待维护」且非溯源 → `gap` 灰色

18 参数固定顺序常量：
```javascript
['PEAK_BRIGHTNESS','DIMMING_ZONES','REFRESH_RATE','CHIP','SYSTEM','GAMING',
 'COLOR_GAMUT','HDMI','SCREEN_SIZE','RESOLUTION','HDR','CONTRAST',
 'WIFI','BLUETOOTH','DOLBY_ATMOS','POWER_CONSUMPTION','WEIGHT','DIMENSIONS']
```

### 2.5 AI分析展示

`callAI()` 流程（需 `SELECTED.length >= 2`）：

1. `#ai-conclusion-area` 显示 `.ai-loading` 紫蓝流光动画 + 「xinghai-ultra 分析中…预计10-20秒」
2. `AbortController` 设 60 秒超时
3. `POST /api/ai-analysis`，body=`{model_ids: SELECTED, region: CUR.region}`
4. 响应解析：
   - `r.error && !r.fallback` → 红字显示错误 + `config_needed` 配置提示
   - 否则 `renderConclusion(r.ai_conclusion, r.model, r.fallback)` + `updateMetrics(r.ai_conclusion)`
5. 超时（`AbortError`）显示「AI 分析超时（60秒），请检查 xinghai-ultra 代理状态后重试」

`renderConclusion(c, model, isFallback)` 渲染：

- 顶部标签：兜底时显示黄色「（规则兜底，AI 未就绪）」，正常显示灰色「（${model}）」
- 三栏 grid：`优势点（▲ 绿）` / `劣势点（▼ 黄）` / `机会点（◆ 蓝）`，每项 `.concl-item` 含 `point` + 置信度百分比 + 溯源证据 `evidence`
- 底部 `strategy_suggestion` 策略建议（蓝色加粗「策略建议：」前缀）

`updateMetrics(c)` 更新核心指标条：领先参数数（绿）/ 落后参数数（黄）/ 机会点数（蓝）。

### 2.6 雷达图配置

`loadRadar()` 流程：

1. `fetch('/api/radar?model_ids=${SELECTED.join(',')}')`
2. 返回 `{radar_dims, scores, models}`，默认维度 `['画质','音质','智能','游戏','连接','设计']`
3. 清空容器、设宽 100% 高 400px
4. 旧实例 `RADAR_CHART.dispose()` 后 `echarts.init(radarDiv)`

ECharts 配置要点：

```javascript
{
  tooltip: { trigger: 'item' },
  legend: { bottom: 0, data: models.map(m => `${m.brand} ${m.model_code}`) },
  radar: {
    indicator: dims.map(d => ({ name: d, max: 100 })),  // 六维，满分100
    shape: 'polygon', splitNumber: 5,
    axisName: { color: '#606266', fontSize: 12 },
    splitLine / splitArea / axisLine: 浅灰样式
  },
  series: [{
    type: 'radar',
    data: models.map((m, i) => ({
      name: `${m.brand} ${m.model_code}`,
      value: dims.map(d => scores[m.model_code][d] || 50),  // 缺分默认50
      areaStyle: { opacity: m.is_our ? 0.3 : 0.1 },          // 我方面积更深
      lineStyle: { width: m.is_our ? 3 : 1.5, color: 我方红/调色板色 },
      itemStyle: { color: 同上 }
    }))
  }]
}
```

调色板 `['#e23d2c','#2c6cf6','#67c23a','#e6a23c','#909399','#9b59b6']`，我方强制红色 `#e23d2c`。监听 `window resize` 调 `RADAR_CHART.resize()`。

---

## 三、前端API调用清单

| 函数 | API URL | 方法 | 参数 | 回调处理 |
|---|---|---|---|---|
| `loadCountries` | `/api/countries` | GET | 无 | 按 group 分 optgroup 渲染国家下拉 |
| `checkAI` | `/api/health` | GET | 无 | `ai_ready`→顶栏显示「AI 就绪：${ai_model}」否则「AI 未配置（规则兜底）」 |
| `loadSizes` | `/api/sizes?region=${CUR.region}` | GET | region | 渲染尺寸下拉，默认选第一个 |
| `loadModels` | `/api/models?region=${CUR.region}&size=${CUR.size}` | GET | region, size | 渲染型号池 `.model-item` |
| `loadAIRecommend` | `/api/ai-recommend?region=${CUR.region}&size=${CUR.size}` | GET | region, size | 渲染推荐对阵卡片组，每张含我方+竞品 chip + reason + 采纳按钮 |
| `loadSpecCompare` | `/api/spec?model_ids=${SELECTED.join(',')}` | GET | model_ids | 渲染 18 参数对比表 + 检测缺失参数 |
| `autoTraceAndRefresh` | `/api/backfill` | POST | `{model_ids, region, skip_ai:true}` | 回溯后重载 spec 注入溯源值，渲染 backfill 卡片 |
| `callAI` | `/api/ai-analysis` | POST | `{model_ids, region}` | 渲染 AI 结论三栏 + 策略建议 + 指标条，60s 超时 |
| `runAutoTrace` | `/api/auto-trace?max_models=50&size=${size}` | GET | max_models, size(可选) | 全库溯源，展示结果表格（型号/品牌/缺失/补全/仍缺） |
| `backfillMissing` | `/api/backfill` | POST | `{model_ids, region, skip_ai:true}` | 渲染回溯补全结果卡片 + 重载 spec 表 |
| `triggerCrawl` | `/api/backfill-crawl` | POST | `{adapter_code, series}` | 触发 spec_crawl 重抓官网，完成后刷新 spec |

> 全部 API 均为相对路径（`/api/...`），由同源后端服务提供。`callAI` 与 `backfill` 系列为 POST JSON，其余为 GET。`callAI` 是唯一带超时控制（`AbortController` 60s）的调用。

---

## 四、LLM客户端设计

### 4.1 LLMClient类

`overseas/llm_client.py` 定义通用 Chat Completions 客户端，核心设计：

```python
class LLMClient:
    def __init__(self, settings: dict | None = None, **overrides):
        cfg = dict(settings or config.llm_settings())  # 默认取 config.llm_settings()
        cfg.update({k: v for k, v in overrides.items() if v is not None})  # 关键字参数覆盖
        # 解析全部字段：
        self.provider / base_url / api_key / user_key / model_name
        self.uses_user_key / timeout / max_retries / max_tokens / temperature
        self.ssl_verify / stream
```

异常类 `LLMUnavailable(RuntimeError)`：未配置或请求失败时抛出。

### 4.2 三种Provider

| Provider | base_url 默认 | model 默认 | uses_user_key | 说明 |
|---|---|---|---|---|
| `xinghai` | `https://inner-apisix.hisense.com/higpt-new/v1` | `xinghai-ultra` | **True** | 内网 HiGPT，user_key 走查询参数（extra_query）；未单独配 `LLM_*` 时复用 `XINGHAI_*` 凭据 |
| `deepseek` | `https://api.deepseek.com` | `deepseek-chat` | False | 公网 DeepSeek 官方 API |
| `openai_compatible` | （空，必须显式配） | （空，必须显式配） | False | 任意 OpenAI 兼容端点 |

`configured` 属性逻辑：
```python
ok = bool(self.base_url and self.api_key and self.model_name)
if self.uses_user_key:
    ok = ok and bool(self.user_key)   # xinghai 额外要求 user_key
return ok
```

`_ensure()` 在 `configured` 为假时抛 `LLMUnavailable`，提示需设置的环境变量名。

### 4.3 chat()方法流程

```python
def chat(self, messages, temperature=None, max_tokens=None) -> str:
    self._ensure()
    temp = self.temperature if temperature is None else float(temperature)
    maxt = self.max_tokens if max_tokens is None else int(max_tokens)
    sdk_error = None
    # 优先路径：openai SDK
    try:
        result = self._chat_openai_sdk(messages, temp, maxt)
        if result: return result
    except Exception as exc:
        sdk_error = exc
    # 回退路径：urllib
    try:
        return self._chat_urllib(messages, temp, maxt)
    except LLMUnavailable:
        # 两路都失败，合并报错
        raise LLMUnavailable(f"LLM 调用失败（SDK: {type(sdk_error).__name__}，urllib 亦失败）")
```

**优先路径 `_chat_openai_sdk`**（实测内网/公网均以此方式调通）：

1. `from openai import OpenAI`，ImportError 返回空串交给 urllib 回退
2. `ssl_verify=False` 时构造 `httpx.Client(verify=False)`
3. `client = OpenAI(base_url, api_key, timeout, max_retries, **kwargs)`
4. `create_kwargs` 含 model/messages/temperature/max_tokens/stream
5. `uses_user_key` 时加 `extra_query={"user_key": self.user_key}`（xinghai 专属）
6. 非流式取 `resp.choices[0].message.content`；流式遍历 chunk 拼 `delta.content`

**回退路径 `_chat_urllib`**（无 user_key 的标准端点可靠）：

1. 构造 payload（强制 `stream=True`）
2. `_endpoint()` 拼接 `/chat/completions`，xinghai 的 user_key 作查询参数 `?user_key=...`
3. `Authorization: Bearer ${api_key}` + `Accept: text/event-stream, application/json`
4. `urlopen` 请求，`_parse_response` 兼容解析 SSE（`data:` 行）与普通 JSON
5. `_choice_content` 多路径兜底取 content：delta/message/output_text/text/content

### 4.4 重试与降级

**urllib 路径重试**：

```python
_RETRYABLE_STATUS = frozenset({408, 425, 429, 500, 502, 503, 504})  # 可重试状态码
```

- `max_retries + 1` 次尝试
- `HTTPError` 且状态码在 `_RETRYABLE_STATUS` 且未达上限 → 指数退避 `min(2.0**attempt, 8.0)` 秒后重试
- `URLError / TimeoutError / OSError` 同样退避重试
- 超出重试或不可重试状态码 → 抛 `LLMUnavailable(f"LLM HTTP {status}")`

**SDK 路径降级**：SDK 抛任何异常 → 记录 `sdk_error` → 转 urllib；urllib 也失败 → 合并两边错误抛 `LLMUnavailable`。

**流式策略**：默认非流式更简单可靠；xinghai provider 默认流式（网关强制），可由 `OVERSEAS_LLM_STREAM` 环境变量覆盖。

---

## 五、LLM代理服务

### 5.1 代理作用

`llm_proxy/anthropic_to_openai_proxy.py` 是一个双向格式转换代理，解决 **Claude Code（Anthropic Messages API）与海信 HiGPT（OpenAI 兼容 API）之间的协议差异**：

- 监听本地端口，同时暴露 **Anthropic 风格 `/v1/messages`** 与 **OpenAI 风格 `/v1/chat/completions`** 两套端点
- 将 Anthropic 请求转换为 OpenAI `/v1/chat/completions` 格式转发给海信 HiGPT 上游
- 将 OpenAI 响应（含流式 SSE）转换回 Anthropic 格式返回客户端

**修复的两个关键缺陷**（导致 `xinghai-ultra` 报 400 的根因）：

1. **流式 tool_calls 分片拆分错误**：聚好看/LiteLLM 的 tool_calls 分片按 index 累积，旧实现把每个分片当新 tool_use，导致残缺 block（id 空、name 空、arguments 半截）毒化会话。修复：按 index 累积分片，拼完整再落地。
2. **历史残缺 tool_use 永久毒化**：请求侧 sanitize——name 无效的 tool_use 降级为普通文本（保留对应 tool_result），保证上游每条 `tool_calls[i].function.name` 合法；对 400/422 自动用清洗后负载重试一次。

### 5.2 配置项

```python
HISENSE_BASE_URL = "https://inner-apisix.hisense.com/higpt-new/v1"   # 上游 API 地址
PROXY_PORT = 8082                                                      # 监听端口
REQUEST_TIMEOUT = 300.0                                                # 请求超时（秒）
DEFAULT_MODEL = "xinghai-ultra"                                        # 默认模型
RETRY_STATUS = {400, 422}                                              # 用清洗负载重试的状态码
AUTH_RETRY_STATUS = {401}                                              # 鉴权抖动重试状态码
AUTH_RETRY_ATTEMPTS = 4                                                # 1 次首发 + 3 次重试
AUTH_RETRY_BASE_DELAY = 0.6                                            # 指数退避基数：0.6/1.2/2.4s
```

启动方式：
```python
if __name__ == "__main__":
    uvicorn.run(app, host="127.0.0.1", port=PROXY_PORT,
                log_level="warning", log_config=None)  # 本机回环，仅 8082 端口
```

`log_config=None` 禁用 uvicorn 默认日志，避免无控制台（pythonw）下 `ColourizedFormatter` 调 `sys.stdout.isatty()` 触发 `AttributeError`。

### 5.3 认证机制

上游鉴权采用「**Bearer 或 apikey 二选一**」策略（实测形状）：

```python
HISENSE_AUTH_TOKEN = "YiVcZbGaxXCEL49qL1KEiITOzANgVVzq"   # Authorization Bearer 凭据
HISENSE_API_KEY     = "5z93eqr6995k8f8iikarjxghic2felpr"  # apikey / user_key
HISENSE_USER_KEY = os.environ.get("HISENSE_USER_KEY") or HISENSE_API_KEY  # 可被环境变量覆盖
```

**关键约束**（实测致命坑）：
- ✅ 可用：`Authorization: Bearer <TOKEN>` 头 + `?user_key=<API_KEY>` 查询参数
- ❌ 致命：额外再带 `apikey` / `X-Api-Key` **请求头** → 网关改走 apikey 路径并以 401 拒绝（实测同一凭据带头 0/8 成功，去掉头 8/8 成功）
- 因此代理只保留 `Authorization` 头，把 `user_key` 放 query，`apikey` 作为冗余 query 参数一并带上：

```python
def _upstream_params() -> dict:
    return {"user_key": HISENSE_USER_KEY, "apikey": HISENSE_API_KEY}
```

401 抖动处理：网关在 F5 BIG-IP 下偶发抖动，`_post_upstream` / `_stream_upstream` 对 401 按指数退避（0.6/1.2/2.4s）自动重试至多 4 次，重试成功计入 `STATS["upstream_auth_recovered"]`。

### 5.4 模型列表

**聚好看合法模型 ID**（`VALID_MODELS`）：
```python
{"xinghai-ultra", "deepseek-v4-flash", "qwen3-5-397b"}
```

**模型别名规范化**（`MODEL_ALIASES` + `normalize_model()`）：

| 客户端传入 | 规范化为 |
|---|---|
| `sonnet` / `opus` / `haiku` | `xinghai-ultra` |
| `deepseek-flash` / `deepseek` / `deepseek-v4-flash` | `deepseek-v4-flash` |
| `qwen` / `qwen35-397b` / `qwen3-5-397b` | `qwen3-5-397b` |
| 含 `deepseek` 关键字 | `deepseek-v4-flash` |
| 含 `qwen` 关键字 | `qwen3-5-397b` |
| 其余未知（含 `claude-3-5-sonnet-*` 等长名） | `xinghai-ultra`（DEFAULT_MODEL） |

**OpenAI 模型目录**（`/v1/models` 端点返回，`OPENAI_MODEL_CATALOG`）：

| id | name | context_window | max_tokens |
|---|---|---|---|
| `xinghai-ultra` | xinghai-ultra | 262144 | 32768 |
| `deepseek-v4-flash` | DeepSeek V4 Flash | 262144 | 32768 |
| `qwen3-5-397b` | qwen35-397B | 131072 | 32768 |

**暴露端点**：
| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/health` | 健康检查 + STATS 统计 |
| POST | `/v1/messages` | Anthropic 风格入口（Claude Code 用） |
| GET | `/v1/models` | OpenAI 模型发现 |
| POST | `/v1/chat/completions` | OpenAI 风格入口（透传上游 SSE） |

**依赖**：`fastapi`、`uvicorn`、`httpx`（随本机全局 Python 安装）。

---

## 六、配置系统

### 6.1 路径配置

`overseas/config.py` 全部路径支持环境变量覆盖，默认值可直接本地跑通：

| 常量 | 环境变量 | 默认值 | 说明 |
|---|---|---|---|
| `BASE_DIR` | — | `overseas` 包父目录 | 海外目录根 |
| `DATA_DIR` | `OVERSEAS_DATA_DIR` | `{BASE_DIR}/data` | 数据根目录 |
| `DB_PATH` | `OVERSEAS_DB_PATH` | `{DATA_DIR}/overseas.db` | SQLite 数据库 |
| `CACHE_DIR` | `OVERSEAS_CACHE_DIR` | `{DATA_DIR}/cache` | 缓存目录 |
| `RAW_DIR` | `OVERSEAS_RAW_DIR` | `{DATA_DIR}/raw` | 原始 HTML 存档 |
| `LOG_DIR` | `OVERSEAS_LOG_DIR` | `{DATA_DIR}/logs` | 日志目录 |
| `SESSION_DIR` | `OVERSEAS_SESSION_DIR` | `{DATA_DIR}/session` | Playwright storage_state（含 cookie，敏感） |
| `PROFILE_DIR` | `OVERSEAS_PROFILE_DIR` | `{DATA_DIR}/browser_profile` | 持久化浏览器 profile |

辅助函数：`session_path(site_code)` 按站点隔离登录态文件、`profile_dir(site_code)` 按站点隔离 profile、`ensure_dirs()` 建全部目录、`storage_enabled()` 控制登录态持久化开关。

### 6.2 LLM配置

`llm_settings()` 函数按当前 provider 汇总生效配置（环境变量覆盖 provider 默认值）：

```python
def llm_settings() -> dict:
    provider = LLM_PROVIDER if LLM_PROVIDER in _LLM_PROVIDER_DEFAULTS else "xinghai"
    defaults = _LLM_PROVIDER_DEFAULTS[provider]
    base_url = LLM_BASE_URL or defaults["base_url"]
    model = LLM_MODEL or defaults["model"]
    api_key = LLM_API_KEY
    user_key = LLM_USER_KEY
    if provider == "xinghai":
        # 未单独配 LLM_* 时复用网评线已有的 XINGHAI_* 凭据，避免重复配置
        api_key = api_key or XINGHAI_API_KEY
        user_key = user_key or XINGHAI_USER_KEY
        base_url = base_url or XINGHAI_BASE_URL
        model = model or XINGHAI_MODEL
    return {
        "provider", "base_url", "api_key", "user_key", "model",
        "uses_user_key": bool(defaults["uses_user_key"]),
        "stream": _flag("OVERSEAS_LLM_STREAM", provider == "xinghai"),  # xinghai 默认流式
        "timeout", "max_retries", "max_tokens", "temperature", "ssl_verify"
    }
```

关键逻辑：
- `provider` 不在 `_LLM_PROVIDER_DEFAULTS` 时回落 `xinghai`
- `xinghai` provider 未单独配 `LLM_*` 时**复用 `XINGHAI_*` 网评线凭据**，避免重复配置
- `stream` 默认值：xinghai 强制流式（网关要求），其余 provider 非流式更稳

### 6.3 环境变量完整列表

#### LLM 专属（阶段三适配器自动生成）

| 环境变量 | 默认值 | 说明 |
|---|---|---|
| `OVERSEAS_LLM_PROVIDER` | `xinghai` | provider：xinghai/deepseek/openai_compatible |
| `OVERSEAS_LLM_BASE_URL` | （按 provider 默认） | 端点地址 |
| `OVERSEAS_LLM_API_KEY` | （xinghai 复用 XINGHAI_API_KEY） | API Key |
| `OVERSEAS_LLM_USER_KEY` | （xinghai 复用 XINGHAI_USER_KEY） | user_key（仅 xinghai 需要） |
| `OVERSEAS_LLM_MODEL` | （按 provider 默认） | 模型名 |
| `OVERSEAS_LLM_TIMEOUT` | `120.0` | 超时秒 |
| `OVERSEAS_LLM_MAX_RETRIES` | `2` | 重试次数 |
| `OVERSEAS_LLM_MAX_TOKENS` | `4096` | 最大输出 token |
| `OVERSEAS_LLM_TEMPERATURE` | `0.1` | 温度 |
| `OVERSEAS_LLM_SSL_VERIFY` | `True` | SSL 校验 |
| `OVERSEAS_LLM_STREAM` | （xinghai=True，其余=False） | 流式开关 |

#### 网评 AI（HiGPT / xinghai-ultra，与 LLM 共用上游）

| 环境变量 | 默认值 | 说明 |
|---|---|---|
| `OVERSEAS_XINGHAI_BASE_URL` | `https://inner-apisix.hisense.com/higpt-new/v1` | HiGPT 端点 |
| `OVERSEAS_XINGHAI_API_KEY` | （空） | API Key |
| `OVERSEAS_XINGHAI_USER_KEY` | （空） | user_key |
| `OVERSEAS_XINGHAI_MODEL` | `xinghai-ultra` | 模型 |
| `OVERSEAS_XINGHAI_TIMEOUT` | `90.0` | 超时 |
| `OVERSEAS_XINGHAI_MAX_RETRIES` | `2` | 重试 |
| `OVERSEAS_XINGHAI_MAX_WORKERS` | `2` | 并发 |
| `OVERSEAS_XINGHAI_BATCH_RECORDS` | `8` | 批量记录数 |
| `OVERSEAS_XINGHAI_MAX_TOKENS` | `1024` | 最大 token |
| `OVERSEAS_XINGHAI_MAX_INPUT_CHARS` | `16000` | 最大输入字符 |
| `OVERSEAS_XINGHAI_TEMPERATURE` | `0.1` | 温度 |
| `OVERSEAS_XINGHAI_SSL_VERIFY` | `True` | SSL |
| `OVERSEAS_XINGHAI_FALLBACK_ENGINE` | `nllb` | 翻译兜底引擎（nllb/argos） |

#### 路径（见 6.1）

`OVERSEAS_DATA_DIR` / `OVERSEAS_DB_PATH` / `OVERSEAS_CACHE_DIR` / `OVERSEAS_RAW_DIR` / `OVERSEAS_LOG_DIR` / `OVERSEAS_SESSION_DIR` / `OVERSEAS_PROFILE_DIR`。

#### 抓取节流与熔断

`OVERSEAS_MIN_INTERVAL`(6.0) / `OVERSEAS_JITTER`(3.0) / `OVERSEAS_MAX_RETRIES`(2) / `OVERSEAS_MAX_RETRIES_PER_REQUEST`(2) / `OVERSEAS_MAX_CONSECUTIVE_MODEL_FAILURES`(3) / `OVERSEAS_MAX_CONSECUTIVE_SERIES_FAILURES`(5) / `OVERSEAS_MAX_CONSECUTIVE_SEARCH_BLOCKED`(3) / `OVERSEAS_MAX_CONSECUTIVE_PDP_BLOCKED`(5) / `OVERSEAS_MAX_CONSECUTIVE_NO_PRICE`(10) / `OVERSEAS_MAX_CONSECUTIVE_REVIEW_BLOCKED`(3) / `OVERSEAS_MAX_REVIEW_PAGES`(20) / `OVERSEAS_MAX_REVIEW_THREADS`(60)。

#### HTTP / 浏览器 / 代理 / 缓存 / 数据库

`OVERSEAS_HTTP_TIMEOUT`(30) / `OVERSEAS_USER_AGENT` / `OVERSEAS_BROWSER_EXECUTABLE` / `OVERSEAS_BROWSER_CHANNELS`(msedge,chrome,chromium) / `OVERSEAS_HEADLESS`(True) / `OVERSEAS_NAV_TIMEOUT`(30) / `OVERSEAS_SETTLE`(4.0) / `OVERSEAS_SCROLL_PASSES`(3) / `OVERSEAS_SCROLL_WAIT`(0.8) / `OVERSEAS_LOCALE`(en-CA) / `OVERSEAS_SPEC_TIMEOUT_SECONDS`(1800) / `OVERSEAS_REVIEW_TIMEOUT_SECONDS`(1800) / `OVERSEAS_PROXY` / `OVERSEAS_PROXY_PROBE`(True) / `OVERSEAS_PROXY_REQUIRED`(False) / `OVERSEAS_PROXY_WAIT`(20.0) / `OVERSEAS_CACHE_ENABLED`(True) / `OVERSEAS_CACHE_TTL`(86400) / `OVERSEAS_SAVE_RAW`(False) / `OVERSEAS_DB_URL`。

#### 在线翻译兜底

`OVERSEAS_TRANSLATE_ENABLED`(False) / `OVERSEAS_TRANSLATE_PROVIDER`(deepl) / `OVERSEAS_TRANSLATE_API_KEY` / `OVERSEAS_TRANSLATE_CUSTOM_URL` / `OVERSEAS_TRANSLATE_CUSTOM_RESULT_PATH`(translatedText) / `OVERSEAS_TRANSLATE_SOURCE`(JA) / `OVERSEAS_TRANSLATE_TARGET`(ZH) / `OVERSEAS_TRANSLATE_TIMEOUT`(10.0) / `OVERSEAS_TRANSLATE_CACHE_PATH`。

生产环境用 systemd `EnvironmentFile` 注入（见 `deploy/overseas.service`）。

---

## 七、参数补全（Backfill）

### 7.1 三层回溯策略

`overseas/backfill.py` 实现待维护参数的三层递进回溯，**核心原则：AI 永远不产出规格值，只查找"去哪里能抓到真实数据"，抓到的才入库，找不到则诚实标记"待维护"，绝不编造**。

```
backfill_model_params(model_id, param_codes, db_path, skip_ai)
  ├─ 第1层 db   _backfill_from_db()    原始 spec_row/product_spec 精确关键词搜索真实值
  ├─ 第2层 site _backfill_from_site()   标记可调 spec_crawl 重抓官网（真实抓取）
  └─ 第3层 ai   _backfill_from_ai()     AI 查找官网规格页 URL（不产出值），找到后交 spec_crawl 真实抓取
```

**三层判定的优先级**：

1. 第1层 db 命中且 `confidence >= 0.9` → 直接采用，跳过后续
2. 第2层 site 标记（`confidence=0.5`，`value='(待重抓)'`，`needs_crawl=True`）
3. 第3层 ai（`skip_ai=True` 时跳过）：
   - AI 找到 URL → `value='(待重抓)'`，`source='ai_url'`，`needs_crawl=True`，仍需 spec_crawl 真实抓取
   - AI 未找到 → `confidence=0`，诚实标记「AI未找到可确认的官网URL，需人工查证」
   - 三层均未找到 → `source='none'`，`detail='三层回溯均未找到真实数据源'`

**18 参数精确搜索关键词**（`BACKFILL_KEYWORDS`，用于第1层 db 回溯）：

| 参数码 | 关键词（部分） |
|---|---|
| `PEAK_BRIGHTNESS` | ピーク輝度, Peak Brightness, 最大輝度 |
| `DOLBY_ATMOS` | ドルビーアトモス, Dolby Atmos, 音声フォーマット |
| `CONTRAST` | コントラスト比, Contrast Ratio, パネルコントラスト比 |
| `REFRESH_RATE` | 倍速対応, リフレッシュレート, Refresh Rate, Motion Rate |
| `COLOR_GAMUT` | 広色域, Color Gamut, 量子ドット, QLED, Wide Color Gamut |
| `HDMI` | HDMI, HDMI端子, 端子数 |
| `HDR` | HDR対応, HDR10, HDR Support, High Dynamic Range |
| `RESOLUTION` | 画素数, Resolution, 像素数 |
| `POWER_CONSUMPTION` | 定格消費電力, 消費電力, Power Consumption |
| `CHIP` | エンジン, 映像エンジン, Processor, IPQエンジン |
| `SYSTEM` | 搭載OS, Operating System, Smart TV OS, Google TV |
| `GAMING` | ゲームモード, HDMI2.1仕様, VRR, ALLM, ゲーム機能 |
| `DIMMING_ZONES` | バックライト, ローカルディミング, Local Dimming, Mini LED |
| `WIFI` | 無線LAN内蔵, Wi-Fi内蔵, Wi-Fi Built in |
| `WEIGHT` | 梱包質量, 本体質量, スタンドあり(kg) |
| `DIMENSIONS` | 外形寸法, Dimensions, 画面寸法(mm) |
| `BLUETOOTH` | Bluetooth内蔵, Bluetoothイヤホン |
| `SCREEN_SIZE` | 画面サイズ, パネルサイズ, Screen Size |

### 7.2 核心函数

#### `backfill_model_params(model_id, param_codes, db_path, skip_ai)`

主入口，返回 `{param_code: {value, source, confidence, detail, [adapter_code], [needs_crawl], [url]}}`。

- `source` 取值：`db` / `site` / `ai_url` / `none`
- `skip_ai=True` 时跳过第3层（全库批量溯源用，避免 AI 产出值注入面板）
- 先从 `dim_product_model` 查 `model_code` 与 `brand_raw`，型号不存在则全部返回 `none`

#### `_load_model_raw_specs(conn, model_code, brand_raw)`

一次性取该型号所有原始规格文本（带函数级缓存 `cache_key=f'_raw_{model_code}'`）：
- `spec_row` 表：`values_json LIKE '%{model_code}%'`，解析 JSON 后按 model_code 精确匹配，过滤空值/`-`/`—`/`ー`
- `product_spec` 表：JOIN `product` ON `p.model=?`，过滤空值/`-`/`—`

#### `_backfill_from_db(conn, model_id, model_code, param_code, brand_raw)`

第1层：对 `spec_row` 的 `item_ja` 与 `product_spec` 的 `key` 用 `BACKFILL_KEYWORDS[param_code]` 关键词做 `in` 子串匹配，命中即返回 `source='db', confidence=0.95`。

#### `_backfill_from_site(conn, model_id, model_code, brand_raw, param_code)`

第2层：
1. `_infer_adapter_code(brand_raw, model_code)` 推断适配器（如 `hisense_jp`/`sony_jp`/`sharp_jp` 等）
2. `SiteRegistry.get(adapter_code)` 取适配器，检查 `supports_spec`（结果缓存在 `_ADAPTER_CACHE`）
3. 查 `spec_row` 该适配器上次抓取时间 `MAX(captured_at)`
4. 返回 `value='(待重抓)', source='site', confidence=0.5, needs_crawl=True, adapter_code=...`

#### `_backfill_from_ai(conn, model_id, model_code, param_code, brand_raw)`

第3层：调 `LLMClient` 查找官网规格页 URL：
1. 构造 prompt（严格规则：只返回确信存在的官网 URL，不编造，不确定则回「未找到|0|...」，不要电商/评测页）
2. `client.chat([...], temperature=0, max_tokens=200)`
3. 解析 `URL|置信度|来源说明` 格式
4. URL 不以 `http` 开头或含「未找到」或置信度 0 → 返回 `confidence=0`
5. 找到 URL → `value='(待重抓)', source='ai_url', needs_crawl=True, url=...`（**仍需 spec_crawl 真实抓取**）

#### `_infer_adapter_code(brand_raw, model_code)`

按 `BRAND_TO_ADAPTER` 映射推断适配器，未命中时按 model_code 前缀兜底：`K-`→sony_jp、`4T-`→sharp_jp、`TV-`→panasonic_jp、含 `REGZA`→regza_jp。

#### `trigger_site_crawl(adapter_code, series_name, db_path)`

触发第2层实际抓取：
```python
from overseas.scenarios import spec_crawl
from overseas.db import Database
db = Database(db_path)
stats = spec_crawl(adapter_code, only_series=[series_name], db=db, verbose=True)
return {'success': True, 'stats': {'pass': stats.pass_, 'fail': stats.fail}}
```

前端 `triggerCrawl(adapterCode, series)` 通过 `POST /api/backfill-crawl` 调用此函数，完成后自动 `loadSpecCompare()` 刷新对比表。

> 前端 `backfillMissing()` 与 `autoTraceAndRefresh()` 调用 `POST /api/backfill` 时均传 `skip_ai: true`，确保自动溯源只走第1层 db 真实值 + 第2层 site 标记，不调 AI 产出值注入面板。AI 查找 URL 的第3层仅在显式「触发重抓」流程中按需触发。
