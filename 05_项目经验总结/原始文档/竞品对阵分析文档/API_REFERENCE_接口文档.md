# 竞品对阵分析后端 API 参考文档

> 源文件：`overseas/benchmark_server.py`（2178 行）
> 文档版本：基于源码逐行提取，覆盖所有 API 端点与核心函数。

本服务实现"国家选择 → AI 阵容推荐 → SPEC 对比 → 雷达图 → AI 结论"的竞品对阵分析流程。
依赖数据库 `overseas_v4_20260929.db` 中的 6 张表：
`dim_spec_parameter` / `dim_spec_item_map` / `dim_product_model` / `fact_spec_value` / `dim_spec_normalization_map` / `fact_benchmark_comparison`，外加运行时引用的 `product` 与 `price_snapshot` 两张价格表。
AI 分析接入 `xinghai-ultra`（`overseas.llm_client.LLMClient`）。

---

## 目录

- [一、全局配置与常量](#一全局配置与常量)
  - [COUNTRIES 区域配置](#countries-区域配置)
  - [PARAM_ORDER / PARAM_NAMES_ZH / PARAM_CATEGORY](#param_order--param_names_zh--param_category)
  - [RADAR_DIMS 雷达图维度](#radar_dims-雷达图维度)
- [二、核心函数清单](#二核心函数清单)
- [三、API 端点清单](#三api-端点清单)
- [四、HTTP Handler 与路由分发](#四http-handler-与路由分发)
- [五、启动入口与 LLM 配置注入](#五启动入口与-llm-配置注入)

---

## 一、全局配置与常量

### 文件头与全局变量（L1–L124）

```python
DEFAULT_DB = os.path.join(..., 'data', 'overseas_v4_20260929.db')   # L122
STATIC_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'benchmark_static')  # L123
SETTINGS = {'db': DEFAULT_DB, 'cors': ''}                          # L124
```

- `DEFAULT_DB`：默认 SQLite 数据库路径，位于项目根目录 `data/` 下。
- `STATIC_DIR`：前端静态文件目录，与 `benchmark_server.py` 同级的 `benchmark_static/`。
- `SETTINGS`：全局运行时设置，在 `main()` 中由命令行参数覆盖。

依赖导入（L12–L23）：`argparse / json / os / sqlite3 / sys / traceback`，`http.server.BaseHTTPRequestHandler / ThreadingHTTPServer`，`urllib.parse.parse_qs / urlparse`，以及本包 `__version__`、`config`。

### COUNTRIES 区域配置（L28–L120）

`COUNTRIES` 是一个字典，key 为国家/区域代码，value 含字段：

| 字段 | 含义 |
|------|------|
| `name` | 中文国家名 |
| `currency` | 货币代码（JPY/USD/CAD/MXN/COP/PEN/CLP/GTQ/BOB/GBP/EUR/CHF/SEK/PLN/RSD/RON/NOK/DKK/BGN/BAM/TRY） |
| `flag` | 国旗 emoji |
| `group` | 大分组：日本 / 北美 / 南美 / 欧洲 |
| `bands` | 价格带阈值：`{size_inch: {'入门':(lo,hi), '中端':(lo,hi), '高端':(lo,hi), '旗舰':(lo,hi)}}` |

覆盖区域：JP / US / CA / MX / CO / PE / CL / GT / BO / UK / EU / CH / SE / PL / RS / RO / NO / DK / BG / BA / TR（共 21 个）。
- 每个区域为不同尺寸（55/65/75/85 等，部分区域无 85）定义"入门/中端/高端/旗舰"4 档价格区间，旗舰上限统一为 `9e9`。
- `infer_region()`（见下）依赖品牌后缀（如 `_jp`）反推区域。

### PARAM_ORDER / PARAM_NAMES_ZH / PARAM_CATEGORY（L126–L145）

```python
PARAM_ORDER = ['PEAK_BRIGHTNESS','DIMMING_ZONES','REFRESH_RATE','CHIP','SYSTEM','GAMING',
               'COLOR_GAMUT','HDMI','SCREEN_SIZE','RESOLUTION','HDR','CONTRAST',
               'WIFI','BLUETOOTH','DOLBY_ATMOS','POWER_CONSUMPTION','WEIGHT','DIMENSIONS']
```

- `PARAM_ORDER`（L126）：18 个规格参数的标准展示顺序，SPEC 对比矩阵按此顺序输出。
- `PARAM_NAMES_ZH`（L129）：参数代码 → 中文名映射，如 `'PEAK_BRIGHTNESS':'峰值亮度'`。
- `PARAM_CATEGORY`（L136）：参数代码 → 类别映射，类别为 `画质/智能/游戏/连接/设计/音质`。
- `RADAR_DIMS`（L143）：雷达图六维 → 参数代码列表：
  - 画质：PEAK_BRIGHTNESS / DIMMING_ZONES / REFRESH_RATE / COLOR_GAMUT / HDR / CHIP / RESOLUTION / CONTRAST
  - 音质：DOLBY_ATMOS
  - 智能：SYSTEM
  - 游戏：GAMING / REFRESH_RATE / HDMI
  - 连接：HDMI / WIFI / BLUETOOTH
  - 设计：POWER_CONSUMPTION / SCREEN_SIZE / WEIGHT / DIMENSIONS

### _DISPLAY_NORMALIZE 归一化规则表（L169–L1245）

`_DISPLAY_NORMALIZE` 是一个**有序字典**（实际为 Python 3.7+ 保留插入顺序的 dict），把原始规格值（多为日文长串/英文/西语）归一化为简洁中文显示。匹配规则：**按插入顺序遍历，第一个命中正则的生效**（`re.search` + `re.I` 忽略大小写），因此"最具体的规则放前面"。

规则按品牌/参数分组（以下为大类，完整规则见源码 L169–L1245）：

| 行号区间 | 规则分组 | 说明 |
|----------|----------|------|
| L173–L195 | Sharp / TCL / 海信 HDMI + 分辨率 | 含 4K120Hz/eARC/VRR/ALLM 等组合，必须先于通用分辨率规则匹配 |
| L197–L216 | 功耗 / 重量 / 尺寸（mm→cm 转换） | `mm_to_cm` 标记在 `_normalize_display` 内特殊处理 |
| L218–L249 | HDR / 刷新率 / 音响功率 / 屏幕尺寸 | 日文 HDR 组合、REGZA/Sony 120Hz 倍速规则 |
| L259–L287 | 背光/分区控光（Mini LED/OLED/直下式等） | REGZA 专用规则先于通用 Mini LED |
| L289–L311 | 画质芯片（Hi-View/Algo/AiPQ/REGZA/Medalist） | |
| L313–L322 | 色域（量子点/广色域/Rich Color） | |
| L324–L342 | 游戏模式（Game Master/FreeSync/VRR/ALLM） | |
| L344–L362 | 智能系统 / 杜比全景声 | |
| L363–L378 | Wi-Fi / 蓝牙 | |
| L380–L402 | 对比度 / 直下式 / 侧入式 | |
| L403–L459 | Sony 专用规则（XR处理器/特丽魅彩/峰值亮度/eARC） | |
| L461–L544 | REGZA 专用规则（引擎/HDMI/HDR/色域/对比度/尺寸） | |
| L546–L551 | TCL 尺寸 mm→cm | |
| L553–L624 | 补全归一化规则（审计后追加） | |
| L625–L685 | ●/○ 符号归一化 + source_term 归一化 | 当 value_text 是符号时用 source_term |
| L686–L841 | 多区域扩展（英文/西语/葡语）— SCREEN_SIZE/RESOLUTION/HDR/HDMI/REFRESH_RATE | |
| L845–L900 | PEAK_BRIGHTNESS / CHIP（英文） | nits/cd/m²、Cognitive Processor XR 等 |
| L916–L953 | DIMMING_ZONES（英文 Mini LED/Direct/Edge） | |
| L955–L982 | SYSTEM（Google TV/Android/Fire TV/webOS/Tizen/VIDAA/Roku） | |
| L983–L1022 | GAMING（Game Mode PRO/FreeSync/G-Sync/ALLM/VRR） | |
| L1023–L1054 | COLOR_GAMUT（BT.2020/DCI-P3/Quantum Dot） | |
| L1055–L1097 | WIFI（802.11ax/ac/n、双频） | |
| L1098–L1128 | BLUETOOTH（版本号提取） | |
| L1129–L1145 | DOLBY_ATMOS | |
| L1146–L1168 | POWER_CONSUMPTION | |
| L1169–L1191 | WEIGHT（kg/lbs） | |
| L1192–L1220 | DIMENSIONS（mm/inch/cm） | |
| L1221–L1245 | CONTRAST（X:1） | |

> 注意：本表条目极多（数百条），核心是"先具体后通用"的优先级排序。新增品牌或新格式时，需在对应参数的通用规则**之前**插入更具体的规则。

---

## 二、核心函数清单

### `_db()` — 数据库连接（L148–L151）

```python
def _db():
    conn = sqlite3.connect(SETTINGS['db'])
    conn.row_factory = sqlite3.Row
    return conn
```

- 返回一个 `sqlite3.Connection`，`row_factory=sqlite3.Row` 使结果行可用列名访问（`r['model_code']`）。
- 连接路径取自全局 `SETTINGS['db']`，由 `main()` 的 `--db` 参数控制。
- 调用方负责 `conn.close()`（各 `api_*` 函数均显式关闭）。

### `_json(payload)` — JSON 序列化（L154–L155）

```python
def _json(payload) -> bytes:
    return json.dumps(payload, ensure_ascii=False, default=str).encode('utf-8')
```

- `ensure_ascii=False`：保留中文不转义。
- `default=str`：对非原生可序列化对象（如 datetime、Decimal）调用 `str()` 兜底。
- 返回 `bytes`，直接写入 HTTP 响应体。

### `infer_region(brand_raw)` — 从 brand_raw 推断区域（L158–L164）

```python
def infer_region(brand_raw):
    if not brand_raw: return None
    m = {'_jp':'JP','_ca':'CA','_us':'US','_mx':'MX','_pe':'PE','_co':'CO',
         '_cl':'CL','_uk':'UK','_ec':'EC','_bo':'BO'}
    for suf, reg in m.items():
        if brand_raw.endswith(suf): return reg
    return None
```

- 依据 `brand_raw` 的后缀（如 `_jp`）反推区域代码。
- 支持映射：JP/CA/US/MX/PE/CO/CL/UK/EC/BO；无法识别返回 `None`。
- 注意：此函数在本文件中**定义但未被直接调用**（区域由请求参数 `region` 显式传入），是给外部脚本/溯源流程用的工具。

### `_normalize_display(raw, param_code, model_code=None)` — 规格值归一化（L1247–L1362）

**函数签名**：`_normalize_display(raw, param_code, model_code=None) -> str`

**逻辑**：
1. 空值 → 返回 `'—'`（L1252）。
2. `text = str(raw).strip()`，把全角横线 `－` 统一为半角 `—`（L1257）。
3. **Sony 系列共享值提取**（L1259–L1272）：当 `model_code` 存在且 text 含 `【型号】` 前缀时，用正则提取当前型号对应的尺寸/重量/功耗值（因 Sony 系列多个型号共用一段规格文本）。
4. 遍历 `_DISPLAY_NORMALIZE` 规则表（L1273），第一个 `re.search(pattern, text, flags=re.I)` 命中的生效：
   - 特殊值 `mm_to_cm`：触发 mm→cm 转换分支（L1276–L1334），支持 TCL/松下/通用二维/三维 mm 格式，先去逗号再 `/10`，返回 `WxHcm` 或 `WxHxDcm`（可含对角线）。
   - 普通替换：`re.sub(pattern, replacement, text, flags=re.I)`，结果超 60 字符截断为 `前58…`（L1336）。
5. 规则表无命中时，按 `param_code` 做纯数字兜底加单位（L1340–L1359）：
   - `WEIGHT`：纯数字 → `+ 'kg'`
   - `HDMI`：纯整数 → `+ '口'`
   - `PEAK_BRIGHTNESS`：纯整数 → `+ ' nits'`
   - `BLUETOOTH`：`x.y` → `'Bluetooth x.y'`
   - `CONTRAST`：纯数字（含逗号）→ `+ ':1'`
   - `SCREEN_SIZE`：纯数字 → `+ '寸'`
   - `REFRESH_RATE`：纯整数 → `+ 'Hz'`
6. 最终超 60 字符截断（L1360），返回 `text`。

**行号**：L1247–L1362。

### `_region_filter(region)` — 区域过滤 SQL（L1376–L1399）

**函数签名**：`_region_filter(region) -> tuple[str, list]`

**逻辑**：返回 `(where_sql, params)` 片段，用于过滤 `dim_product_model`。

- **JP 分支**（L1383–L1390）：
  ```sql
  (brand_raw LIKE '%日本%' OR (
      model_code IN (
          SELECT DISTINCT p.model FROM product p
          JOIN price_snapshot ps ON ps.product_id=p.id
          WHERE p.brand LIKE '%isense%' AND ps.currency='JPY'
      )
  ))
  ```
  含义：竞品品牌名含"日本"，或海信日本线型号（brand 含 `isense` 且有 JPY 价格）。
- **非 JP 分支**（L1392–L1398）：按货币过滤，只显示有该货币价格的型号：
  ```sql
  model_code IN (
      SELECT DISTINCT p.model FROM product p
      JOIN price_snapshot ps ON ps.product_id=p.id
      WHERE ps.currency=?
  )
  ```
  参数为 `[currency]`（如 USD/CAD/EUR）。
- 无法识别区域且无货币 → 返回 `"1=1", []`（不过滤）。

**行号**：L1376–L1399。

### `api_countries()` — 国家列表（L1369–L1373）

```python
def api_countries():
    return [{'code': k, 'name': v['name'], 'currency': v['currency'], 'flag': v['flag'],
             'group': v.get('group', '')}
            for k, v in COUNTRIES.items()]
```

- 直接从 `COUNTRIES` 导出，**不查数据库**。
- 返回列表，每项含 `code/name/currency/flag/group`。

### `api_sizes(region)` — 尺寸列表（L1402–L1417）

**函数签名**：`api_sizes(region) -> dict`

**核心 SQL**：
```sql
SELECT size_inch, COUNT(*)
FROM dim_product_model
WHERE size_inch IS NOT NULL AND {where}
GROUP BY size_inch
ORDER BY size_inch
```
- `{where}` 来自 `_region_filter(region)`，参数 `params` 一并传入。
- 结果按 `size_inch` 分组计数，但**只保留**白名单尺寸（L1414）：
  `(19,24,32,40,42,43,48,50,55,58,65,75,77,85,98,100)`。
- 返回 `{'region': region, 'currency': currency, 'sizes': [{'size': int, 'model_count': int}, ...]}`。

### `api_models(region, size)` — 型号列表（L1420–L1454）

**函数签名**：`api_models(region, size) -> dict`

**型号主查询 SQL**（L1428–L1431）：
```sql
SELECT model_id, brand, model_code, brand_raw, our_brand
FROM dim_product_model
WHERE size_inch=? AND model_code IS NOT NULL AND {where}
ORDER BY our_brand DESC, brand
```
- 参数：`(int(size),) + tuple(params)`（先尺寸，后区域过滤参数）。

**均价查询 SQL**（L1436–L1438，对每个型号逐个查）：
```sql
SELECT AVG(ps.price)
FROM price_snapshot ps
JOIN product p ON p.id=ps.product_id
WHERE p.model=? AND ps.currency=?
```
- 严格按 region 货币过滤，**不 fallback 到任意货币**（L1441 注释：曾误把 COP/PEN/USD/EUR 当 JPY 显示）。
- `price is None` → 跳过该型号（"缺价 > 错价"）。
- 脏数据型号名用 `_is_valid_model_code()` 过滤（L1445）。
- 按价格带 `bands[size]` 判定 `price_band`（L1448–L1449）。

**返回**：`{'region', 'currency', 'size': int(size), 'models': [{'model_id','brand','model_code','price': int,'price_band','our_brand','is_our': bool}, ...]}`

### `_is_valid_model_code(code)` — 脏数据型号过滤（L1457–L1469）

```python
def _is_valid_model_code(code):
    if not code: return False
    code = str(code).strip()
    if len(code) < 4: return False
    if re.match(r'^\d+$', code): return False            # 纯数字
    if re.match(r'^\d{2,4}[A-Z]{1,3}$', code): return False  # 规格词如 180HZ/60LED
    if not any(c.isalpha() for c in code): return False  # 至少含1字母
    return True
```

- 排除纯数字、纯规格词（2-4 位数字 + 1-3 大写字母，如 `180HZ`/`4KHDR`）、无字母串。
- 保留真实型号名（如 `65U8R`/`65C855`）。

### `api_ai_recommend(region, size)` — AI 智能推荐对阵（L1472–L1504）

**函数签名**：`api_ai_recommend(region, size) -> dict`

**逻辑**：
1. 调用 `api_models(region, size)` 拿到型号列表，再用 `_is_valid_model_code` 过滤脏数据（L1474–L1477）。
2. 拆分我方（`is_our=True`）与竞品（L1478–L1479）。
3. 无我方型号 → `{'error': '该区域该尺寸暂无海信型号', 'recommendations': []}`。
4. 我方型号按 `price_band` 排序，**每个价格带只选 1 个**，最多选 3 个（L1484–L1493）。
5. 每个我方型号匹配竞品：
   - 优先同价格带且价差 < 2 倍（`abs(c.price-o.price)/max(o.price,1) < 2.0`）的竞品（L1496）。
   - 无同带 → 取价格最接近的 4 个竞品（L1498）。
   - 每组最多 4 个竞品（L1499）。
6. 每组含 `our`（我方型号）、`competitors`（竞品列表）、`price_band`、`reason`（中文说明）。

**返回**：`{'region', 'size': int, 'recommendations': [{'our', 'competitors', 'price_band', 'reason'}, ...]}`

### `api_spec_compare(model_ids, region='JP')` — 规格对比矩阵（L1507–L1584）

**函数签名**：`api_spec_compare(model_ids, region='JP') -> dict`

**核心逻辑**：
1. `model_ids` 为空 → `{'error': '未指定型号', 'spec_matrix': {}, 'models': []}`。
2. 取型号信息（每个 mid 单查，L1517）：
   ```sql
   SELECT model_id, brand, model_code, our_brand, size_inch
   FROM dim_product_model
   WHERE model_id=?
   ```
3. 取价（严格按 region 货币，L1522–L1524）：
   ```sql
   SELECT AVG(ps.price), ps.currency
   FROM price_snapshot ps
   JOIN product p ON p.id=ps.product_id
   WHERE p.model=? AND ps.currency=?
   GROUP BY ps.currency
   ```
4. 遍历 `PARAM_ORDER` 的每个参数 code（L1533）：
   - 取参数元信息（L1534）：
     ```sql
     SELECT param_id, param_name, param_unit, is_numeric, higher_is_better
     FROM dim_spec_parameter
     WHERE param_code=?
     ```
   - 对每个型号取规格值（L1541–L1551），**排序优先级**（排除误匹配值 > 非●值 > ●值 > 长度降序）：
     ```sql
     SELECT value_text, value_numeric, source_term
     FROM fact_spec_value
     WHERE model_id=? AND param_id=?
     ORDER BY CASE WHEN value_text LIKE '%VIDAA Voice%' THEN 2
                   WHEN value_text LIKE '%附属品%' THEN 2
                   WHEN value_text LIKE '%取扱説明書%' THEN 2
                   WHEN value_text LIKE '%音声操作%' THEN 2
                   WHEN value_text LIKE '%语音操作%' THEN 2
                   WHEN value_text LIKE '%ハンズフリー%' THEN 2
                   WHEN value_text LIKE '%语音助手%' THEN 2
                   WHEN value_text='●' THEN 1 ELSE 0 END,
              LENGTH(COALESCE(value_text,'')) DESC
     ```
   - 取首行 `v`：当 `value_text` 是 `●/○/◯/〇` 符号时，用 `source_term` 作为归一化输入（L1556–L1562）；否则直接归一化。
   - 调 `_normalize_display(raw_text, code, model_code)` 得 `display`（L1560/L1563/L1565）。
   - 数值型且 display 为纯数字 → 追加 `param_unit`（L1567–L1570）。
   - `values[model_code] = {'display', 'numeric', 'raw'}`；无值 → `{'display':'—','raw':None}`。
5. `spec_matrix[code] = {'param_code','param_name','param_name_zh','param_unit','is_numeric','higher_is_better','category','values'}`。

**返回**：`{'spec_matrix': {...}, 'models': [{'model_id','brand','model_code','is_our','size_inch','price': int|None,'currency': str|None}, ...]}`

### `api_radar(model_ids)` — 六维雷达图得分（L1587–L1641）

**函数签名**：`api_radar(model_ids) -> dict`

**逻辑**：
1. 调 `api_spec_compare(model_ids)` 拿规格矩阵（无 region，默认 JP）。
2. 对每个型号、每个雷达维度（`RADAR_DIMS`），累加该维度各参数的得分，取平均（L1597–L1640）。
3. **文本特征评分**（按 param_code 分支，默认 50）：
   - `SYSTEM`：VIDAA=55；Google TV/Android=80；Tizen/webOS=78；其他非空=70。
   - `GAMING`：FreeSync/240Hz=90；144Hz=85；Game/ゲーム=72；其他非空=65。
   - `HDR`：Dolby Vision + HDR10+ =88；任一=78；含 HDR=68。
   - `CHIP`：含 XR/AI/PRO/Pro =82；其他非空=72。
   - `COLOR_GAMUT`：量子/QLED/Quantum=85；色域/%=72。
   - `DOLBY_ATMOS`：Dolby Atmos/支持=80；其他非空=60。
4. **数值型相对评分**（L1630–L1638）：当参数 `is_numeric=1` 且 `higher_is_better` 非空，跨所有型号取 max/min，归一化到 `[60, 95]`：
   - `higher_is_better=1`：`score = 60 + 35 * (num-mn)/(mx-mn)`
   - `higher_is_better=0`：`score = 60 + 35 * (mx-num)/(mx-mn)`
   - 仅当 `len(all_nums) > 1` 才生效。
5. 每维得分 = `round(total/cnt)`，无参数则 50。

**返回**：`{'radar_dims': [...6维...], 'scores': {model_code: {dim: int}}, 'models': [...]}`

### `api_ai_analysis(model_ids, region=None)` — AI 对阵结论（L1644–L1714）

**函数签名**：`api_ai_analysis(model_ids, region=None) -> dict`

**逻辑**：
1. 调 `api_spec_compare(model_ids, region or 'JP')` 和 `api_radar(model_ids)`（L1646–L1647）。
2. 构造规格对比摘要 `summary`（L1654–L1668）：区域、每个型号的角色（我方/竞品）+ 品牌 + 型号 + 价格，以及前 12 个参数（`PARAM_ORDER[:12]`）的逐型号 display 值。
3. 构造 prompt（L1670–L1683）：要求 AI 输出 JSON，含 `advantage_points` / `disadvantage_points` / `opportunity_points`（每条含 `point/evidence/confidence`）+ `strategy_suggestion`。
4. 调用 LLM（L1684–L1714）：
   - 导入失败 `ImportError` → 走 `_fallback_analysis`，返回 `{'fallback': True, 'model': 'xinghai-ultra（llm_client 不可用，规则兜底）', ...}`。
   - `client.configured=False`（未配置） → 走兜底，返回 `config_needed` 提示设置 `OVERSEAS_XINGHAI_*` 环境变量。
   - 调 `client.chat([{"role":"user","content":prompt}], temperature=0.2, max_tokens=2048)`。
   - 解析返回 JSON：先 strip，去掉 ` ```json ` 包裹（L1703–L1704），再 `json.loads`。
   - 成功 → `{'ai_conclusion': ai_result, 'model': client.describe(), 'summary': summary}`。
   - `LLMUnavailable` / `JSONDecodeError` / 其他异常 → 走兜底，返回 `{'error': ..., 'fallback': True, 'ai_conclusion': _fallback_analysis(...)}`。

**返回结构**（成功）：`{'ai_conclusion': {advantage_points:[], disadvantage_points:[], opportunity_points:[], strategy_suggestion}, 'model': str, 'summary': str}`
**返回结构**（兜底）：含 `fallback: True` + `ai_conclusion`（规则生成）。

### `_fallback_analysis(spec, radar, our_model)` — 规则兜底结论（L1717–L1753）

**函数签名**：`_fallback_analysis(spec, radar, our_model) -> dict`

**逻辑**：AI 不可用时生成规则结论。
1. 无我方型号 → 空三列表（L1719–L1720）。
2. **价格对比**（L1725–L1733）：我方价 < 竞品均价 × 0.85 → 优势点（价格领先）；始终加机会点（价格带卡位）。
3. **参数对比**（L1735–L1742）：检查 `REFRESH_RATE/GAMING/HDR` 的我方 display，含 `144Hz/FreeSync/240Hz` → 优势点。
4. **系统劣势**（L1744–L1751）：我方系统是 VIDAA → 劣势点（生态弱于 Google TV/Tizen）+ 机会点（VIDAA 本地化）。
5. 返回 `{'advantage_points': adv, 'disadvantage_points': dis, 'opportunity_points': opp, 'strategy_suggestion': '以画质+游戏+性价比组合应对，补强智能生态'}`。

### `api_backfill(model_ids_str, region=None, skip_ai=False)` — 三层回溯补全（L1908–L1953）

**函数签名**：`api_backfill(model_ids_str, region=None, skip_ai=False) -> dict`

**逻辑**：对指定型号缺失参数做三层回溯补全。
1. 入参支持逗号分隔字符串或 list（L1916–L1919）。
2. 查已有参数（L1938–L1939）：
   ```sql
   SELECT p.param_code
   FROM fact_spec_value f
   JOIN dim_spec_parameter p ON p.param_id=f.param_id
   WHERE f.model_id=?
   ```
3. 计算 `missing = PARAM_ORDER - 已有`。
4. 调用 `from .backfill import backfill_model_params` 执行三层回溯（L1945）：
   - `skip_ai=True`：只走第 1 层（db）+ 第 2 层（site 标记），用于自动溯源。
   - `skip_ai=False`：走全部三层含 AI 查找官网 URL，用于手动触发。
5. 返回 `{'results': {str(mid): {'model_code','missing','backfill','message'}}, 'total_models': n}`。

### `api_backfill_status(model_id)` — 回溯补全状态查询（L1956–L1975）

**函数签名**：`api_backfill_status(model_id) -> dict`

**SQL**（型号信息，L1963）：
```sql
SELECT model_code, brand, brand_raw FROM dim_product_model WHERE model_id=?
```
**SQL**（已有参数，L1968）：
```sql
SELECT p.param_code, p.param_name, f.value_text, f.source_table
FROM fact_spec_value f
JOIN dim_spec_parameter p ON p.param_id=f.param_id
WHERE f.model_id=?
```
**返回**：`{'model_id','model_code','brand','have_count','missing':[...],'have':[{param_code,param_name,value,source}, ...]}`

### `api_backfill_crawl(adapter_code, series)` — 触发第 2 层实际抓取（L1978–L1984）

**函数签名**：`api_backfill_crawl(adapter_code, series) -> dict`

- 需 `adapter_code` 和 `series` 都非空，否则 `{'error': '需指定 adapter_code 和 series'}`。
- 调 `from .backfill import trigger_site_crawl(adapter_code, series, SETTINGS['db'])` 重抓指定系列。
- 直接返回 `trigger_site_crawl` 的结果。

### `api_auto_trace(region=None, size=None, write_back=True, max_models=50, with_ai=False)` — 全库自动溯源（L1987–L2046）

**函数签名**：`api_auto_trace(region=None, size=None, write_back=True, max_models=50, with_ai=False) -> dict`

**逻辑**：扫描所有型号缺失参数，三层回溯补全，写回 `fact_spec_value`。

**核心 SQL**（L1997–L2009，一次性查出所有型号及其缺失参数码）：
```sql
SELECT m.model_id, m.model_code, m.brand, m.size_inch,
       (SELECT GROUP_CONCAT(DISTINCT p.param_code)
        FROM dim_spec_parameter p
        WHERE p.param_id NOT IN (
            SELECT f2.param_id FROM fact_spec_value f2 WHERE f2.model_id=m.model_id
        )) as missing_codes
FROM dim_product_model m
WHERE m.model_code IS NOT NULL
  [AND m.size_inch=?]      -- size 非空时追加
ORDER BY m.our_brand DESC, m.model_id
LIMIT ?                    -- max_models
```
- 参数：`size` 非空时 `[int(size), max_models]`，否则 `[max_models]`。

**逐型号处理**（L2015–L2042）：
1. 解析 `missing_codes`（逗号分隔），只保留在 `PARAM_ORDER` 内的码。
2. 调 `backfill_model_params(mid, missing, SETTINGS['db'], skip_ai=not with_ai)`。
3. 对每个 `confidence > 0` 且 `value` 非空非 `(待重抓)` 的结果，写回数据库（L2026–L2038）：
   ```sql
   SELECT param_id FROM dim_spec_parameter WHERE param_code=?
   ```
   ```sql
   INSERT OR REPLACE INTO fact_spec_value
       (model_id, param_id, value_text, value_numeric, value_unit, source_table, source_term, captured_at, confidence)
       VALUES (?, ?, ?, NULL, '', ?, ?, datetime('now'), ?)
   ```
   - `source_table` 写 `auto-trace-{info['source']}`，`source_term` 写 `[{source}] {detail前100字}`。
4. 累计 `total_filled / total_missing`。

**返回**：`{'total_models','total_missing','total_filled','coverage': 'filled/missing','results': [{'model_id','model_code','brand','missing_count','filled_count','still_missing'}, ...]}`

---

## 三、API 端点清单

### GET /api/health — 健康检查

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/health`（L1854–L1860） |
| 参数 | 无 |
| 核心逻辑 | 读 `config.XINGHAI_API_KEY` 判断 AI 是否就绪；返回版本、AI 模型/基址 |
| SQL | 无（纯配置查询） |
| 行号 | L1854–L1860 |

**返回**：
```json
{
  "status": "ok",
  "version": "<__version__>",
  "ai_ready": true,
  "ai_model": "xinghai-ultra",
  "ai_base": "http://127.0.0.1:8082/v1"
}
```

### GET /api/countries — 国家列表

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/countries`（L1861–L1862） |
| 参数 | 无 |
| 核心逻辑 | 调 `api_countries()`，直接从 `COUNTRIES` 常量导出，不查库 |
| SQL | 无 |
| 行号 | L1369–L1373, L1861–L1862 |

**返回**：
```json
{
  "countries": [
    {"code":"JP","name":"日本","currency":"JPY","flag":"🇯🇵","group":"日本"}, ...
  ]
}
```

### GET /api/sizes — 某区域可用尺寸列表

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/sizes`（L1863–L1864） |
| 参数 | `region`（str，选填，默认 `"JP"`） |
| 核心逻辑 | 调 `api_sizes(region)` → `_region_filter(region)` → 执行分组 SQL，过滤白名单尺寸 |
| 行号 | L1402–L1417, L1863–L1864 |

**核心 SQL**：
```sql
SELECT size_inch, COUNT(*)
FROM dim_product_model
WHERE size_inch IS NOT NULL AND {region_filter_where}
GROUP BY size_inch
ORDER BY size_inch
```
> `{region_filter_where}` 由 `_region_filter` 生成，详见上文。

**返回**：
```json
{
  "region": "JP",
  "currency": "JPY",
  "sizes": [{"size": 55, "model_count": 8}, {"size": 65, "model_count": 12}, ...]
}
```

### GET /api/models — 某区域某尺寸的型号列表

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/models`（L1865–L1866） |
| 参数 | `region`（str，默认 `"JP"`）；`size`（str，默认 `"65"`） |
| 核心逻辑 | 调 `api_models(region, size)`：先按区域过滤查型号，再逐个查均价，过滤脏数据与无价型号，判价格带 |
| 行号 | L1420–L1454, L1865–L1866 |

**核心 SQL 1（型号主查询）**：
```sql
SELECT model_id, brand, model_code, brand_raw, our_brand
FROM dim_product_model
WHERE size_inch=? AND model_code IS NOT NULL AND {region_filter_where}
ORDER BY our_brand DESC, brand
```
**核心 SQL 2（均价，逐型号）**：
```sql
SELECT AVG(ps.price)
FROM price_snapshot ps
JOIN product p ON p.id=ps.product_id
WHERE p.model=? AND ps.currency=?
```

**返回**：
```json
{
  "region": "JP", "currency": "JPY", "size": 65,
  "models": [
    {"model_id":1,"brand":"Hisense","model_code":"65U8R","price":180000,"price_band":"中端","our_brand":1,"is_our":true},
    ...
  ]
}
```

### GET /api/ai-recommend — AI 智能推荐对阵组合

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/ai-recommend`（L1867–L1868） |
| 参数 | `region`（默认 `"JP"`）；`size`（默认 `"65"`） |
| 核心逻辑 | 调 `api_ai_recommend(region, size)`：基于 `api_models`，按价格带均衡选 3 个我方型号，每个匹配同档竞品 |
| SQL | 复用 `api_models` 的两条 SQL |
| 行号 | L1472–L1504, L1867–L1868 |

**返回**：
```json
{
  "region": "JP", "size": 65,
  "recommendations": [
    {"our": {型号对象}, "competitors": [型号对象×≤4], "price_band": "中端",
     "reason": "按价格带「中端」匹配，我方¥180,000 对阵同档竞品"}
  ]
}
```

### GET /api/spec — 规格对比矩阵

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/spec`（L1869–L1871） |
| 参数 | `model_ids`（str，**必填**，逗号分隔的 model_id，如 `"1,2,3"`） |
| 核心逻辑 | 调 `api_spec_compare(ids)`，region 默认 JP；按 `PARAM_ORDER` 构建归一化对比矩阵 |
| 行号 | L1507–L1584, L1869–L1871 |

**核心 SQL**：
```sql
-- 1. 型号信息
SELECT model_id, brand, model_code, our_brand, size_inch FROM dim_product_model WHERE model_id=?

-- 2. 均价（逐型号）
SELECT AVG(ps.price), ps.currency FROM price_snapshot ps
JOIN product p ON p.id=ps.product_id WHERE p.model=? AND ps.currency=? GROUP BY ps.currency

-- 3. 参数元信息（逐 param_code）
SELECT param_id, param_name, param_unit, is_numeric, higher_is_better
FROM dim_spec_parameter WHERE param_code=?

-- 4. 规格值（逐 model×param，含优先级排序）
SELECT value_text, value_numeric, source_term FROM fact_spec_value
WHERE model_id=? AND param_id=?
ORDER BY CASE WHEN value_text LIKE '%VIDAA Voice%' THEN 2
              WHEN value_text LIKE '%附属品%' THEN 2
              WHEN value_text LIKE '%取扱説明書%' THEN 2
              WHEN value_text LIKE '%音声操作%' THEN 2
              WHEN value_text LIKE '%语音操作%' THEN 2
              WHEN value_text LIKE '%ハンズフリー%' THEN 2
              WHEN value_text LIKE '%语音助手%' THEN 2
              WHEN value_text='●' THEN 1 ELSE 0 END,
         LENGTH(COALESCE(value_text,'')) DESC
```

**返回**：
```json
{
  "spec_matrix": {
    "PEAK_BRIGHTNESS": {"param_code":"PEAK_BRIGHTNESS","param_name":"...","param_name_zh":"峰值亮度",
      "param_unit":"nits","is_numeric":1,"higher_is_better":1,"category":"画质",
      "values": {"65U8R": {"display":"1000 nits","numeric":1000,"raw":"..."}, "65C855": {"display":"—","raw":null}}}
  },
  "models": [{"model_id":1,"brand":"...","model_code":"65U8R","is_our":true,"size_inch":65,"price":180000,"currency":"JPY"}]
}
```

### GET /api/radar — 六维雷达图得分

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/radar`（L1872–L1874） |
| 参数 | `model_ids`（str，**必填**，逗号分隔） |
| 核心逻辑 | 调 `api_radar(ids)`，内部复用 `api_spec_compare`，按 `RADAR_DIMS` 六维算分 |
| SQL | 复用 `api_spec_compare` 的 SQL |
| 行号 | L1587–L1641, L1872–L1874 |

**返回**：
```json
{
  "radar_dims": ["画质","音质","智能","游戏","连接","设计"],
  "scores": {"65U8R": {"画质":82,"音质":80,"智能":55,"游戏":90,"连接":70,"设计":65}, ...},
  "models": [...]
}
```

### GET /api/ai-analysis — AI 对阵结论（GET 形式）

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/ai-analysis`（L1875–L1877） |
| 参数 | `model_ids`（str，**必填**，逗号分隔）；`region`（str，选填） |
| 核心逻辑 | 调 `api_ai_analysis(ids, region)`：构造 prompt → 调 `LLMClient.chat` → 解析 JSON；失败走 `_fallback_analysis` |
| SQL | 复用 `api_spec_compare` + `api_radar` 的 SQL |
| 行号 | L1644–L1714, L1875–L1877 |

**返回**（成功）：
```json
{
  "ai_conclusion": {
    "advantage_points": [{"point":"...","evidence":"...","confidence":0.9}],
    "disadvantage_points": [...],
    "opportunity_points": [...],
    "strategy_suggestion": "..."
  },
  "model": "xinghai-ultra (via ...)",
  "summary": "区域: JP\n我方(海信) ... \n规格对比:\n  峰值亮度: ..."
}
```
**返回**（兜底，含 `fallback: true`）：
```json
{
  "fallback": true,
  "ai_conclusion": {"advantage_points":[...], "disadvantage_points":[...], "opportunity_points":[...], "strategy_suggestion":"..."},
  "model": "xinghai-ultra（未配置，规则兜底）",
  "summary": "...",
  "config_needed": "设置 OVERSEAS_XINGHAI_API_KEY / OVERSEAS_XINGHAI_USER_KEY / OVERSEAS_XINGHAI_BASE_URL 后即可调用 AI"
}
```

### POST /api/ai-analysis — AI 对阵结论（POST 形式）

| 项 | 内容 |
|----|------|
| 路由 | `POST /api/ai-analysis`（L1888–L1892） |
| 请求体 | `{"model_ids": [int, ...], "region": "JP"}` |
| 核心逻辑 | 与 GET 形式相同，调 `api_ai_analysis(ids, region)` |
| 行号 | L1888–L1892 |

### POST /api/spec — 规格对比矩阵（POST 形式）

| 项 | 内容 |
|----|------|
| 路由 | `POST /api/spec`（L1893–L1894） |
| 请求体 | `{"model_ids": [int, ...], "region": "JP"}` |
| 核心逻辑 | 调 `api_spec_compare(payload.get("model_ids", []), payload.get("region", "JP"))` |
| 行号 | L1893–L1894 |

### GET /api/backfill — 指定型号缺失参数补全

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/backfill`（L1878–L1879） |
| 参数 | `model_ids`（str，逗号分隔）；`region`（str，选填） |
| 核心逻辑 | 调 `api_backfill(model_ids, region)`，默认 `skip_ai=False`（走全部三层） |
| 行号 | L1908–L1953, L1878–L1879 |

**核心 SQL**：
```sql
-- 1. 型号编码
SELECT model_code FROM dim_product_model WHERE model_id=?

-- 2. 已有参数
SELECT p.param_code FROM fact_spec_value f
JOIN dim_spec_parameter p ON p.param_id=f.param_id WHERE f.model_id=?
```
> 实际补全由 `backfill.backfill_model_params` 完成（三层回溯）。

**返回**：
```json
{
  "results": {"1": {"model_code":"65U8R","missing":["HDR","WIFI"],"backfill":{...},"message":"缺失2个参数，已回溯"}},
  "total_models": 1
}
```

### POST /api/backfill — 指定型号缺失参数补全（POST，可跳过 AI）

| 项 | 内容 |
|----|------|
| 路由 | `POST /api/backfill`（L1895–L1897） |
| 请求体 | `{"model_ids": [int,...], "region": "JP", "skip_ai": false}` |
| 核心逻辑 | 调 `api_backfill(model_ids, region, skip_ai=bool(skip_ai))` |
| 行号 | L1895–L1897 |

### GET /api/backfill-status — 型号回溯补全状态

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/backfill-status`（L1880–L1881） |
| 参数 | `model_id`（str，必填） |
| 核心逻辑 | 调 `api_backfill_status(model_id)` |
| 行号 | L1956–L1975, L1880–L1881 |

**核心 SQL**：
```sql
SELECT model_code, brand, brand_raw FROM dim_product_model WHERE model_id=?;

SELECT p.param_code, p.param_name, f.value_text, f.source_table
FROM fact_spec_value f
JOIN dim_spec_parameter p ON p.param_id=f.param_id
WHERE f.model_id=?;
```

**返回**：
```json
{
  "model_id": 1, "model_code": "65U8R", "brand": "Hisense",
  "have_count": 15,
  "missing": ["WIFI","BLUETOOTH"],
  "have": [{"param_code":"HDR","param_name":"...","value":"...","source":"..."}]
}
```

### POST /api/backfill-crawl — 触发第 2 层实际抓取

| 项 | 内容 |
|----|------|
| 路由 | `POST /api/backfill-crawl`（L1898–L1899） |
| 请求体 | `{"adapter_code": "xxx", "series": "xxx"}` |
| 核心逻辑 | 调 `api_backfill_crawl(adapter_code, series)` → `backfill.trigger_site_crawl` |
| 行号 | L1978–L1984, L1898–L1899 |

### GET /api/auto-trace — 全库自动溯源

| 项 | 内容 |
|----|------|
| 路由 | `GET /api/auto-trace`（L1882–L1884） |
| 参数 | `region`（选填）；`size`（选填）；`max_models`（int，默认 50）；`with_ai`（"0"/"1"/"true"/"yes"，默认 "0"） |
| 核心逻辑 | 调 `api_auto_trace(region, size, max_models=int, with_ai=bool)`：扫库 → 三层回溯 → 写回 |
| 行号 | L1987–L2046, L1882–L1884 |

**核心 SQL**：
```sql
SELECT m.model_id, m.model_code, m.brand, m.size_inch,
       (SELECT GROUP_CONCAT(DISTINCT p.param_code)
        FROM dim_spec_parameter p
        WHERE p.param_id NOT IN (
            SELECT f2.param_id FROM fact_spec_value f2 WHERE f2.model_id=m.model_id
        )) as missing_codes
FROM dim_product_model m
WHERE m.model_code IS NOT NULL
  [AND m.size_inch=?]
ORDER BY m.our_brand DESC, m.model_id
LIMIT ?;
```
**写回 SQL**：
```sql
SELECT param_id FROM dim_spec_parameter WHERE param_code=?;

INSERT OR REPLACE INTO fact_spec_value
    (model_id, param_id, value_text, value_numeric, value_unit, source_table, source_term, captured_at, confidence)
    VALUES (?, ?, ?, NULL, '', ?, ?, datetime('now'), ?);
```

**返回**：
```json
{
  "total_models": 50, "total_missing": 120, "total_filled": 95,
  "coverage": "95/120",
  "results": [{"model_id":1,"model_code":"65U8R","brand":"Hisense","missing_count":3,"filled_count":3,"still_missing":0}]
}
```

### POST /api/auto-trace — 全库自动溯源（POST）

| 项 | 内容 |
|----|------|
| 路由 | `POST /api/auto-trace`（L1900–L1903） |
| 请求体 | `{"region": "JP", "size": "65", "max_models": 50, "with_ai": false}` |
| 核心逻辑 | 同 GET，调 `api_auto_trace(...)` |
| 行号 | L1900–L1903 |

### GET / 及静态文件 — 前端页面

| 项 | 内容 |
|----|------|
| 路由 | `GET /` 及任意非 `/api` 路径（L1826–L1828, L1783–L1805） |
| 参数 | 无（路径即文件路径） |
| 核心逻辑 | `do_GET` 判断非 `/api` 前缀 → `_serve_static(raw)`：根路径映射 `/index.html`，从 `STATIC_DIR` 读文件，`mimetypes` 猜类型，支持 CORS |
| 行号 | L1783–L1805, L1822–L1828 |

### OPTIONS / 及 HEAD / — 预检与缓存验证

| 项 | 内容 |
|----|------|
| `OPTIONS` | 返回 `204 {}`（L1810–L1811），用于 CORS 预检 |
| `HEAD` | 非 `/api` 路径 → `_serve_static(raw, head_only=True)`（只发头不发体）；`/api` 路径 → `200 {}`（L1813–L1820） |

---

## 四、HTTP Handler 与路由分发

### Handler 类（L1760–L1905）

继承 `BaseHTTPRequestHandler`，`server_version = f"benchmark-api/{__version__}"`（L1761）。

**方法清单**：

| 方法 | 行号 | 作用 |
|------|------|------|
| `_send(status, payload)` | L1763–L1778 | JSON 响应：设 Content-Type/Length/CORS 头，写 body（HEAD 不写，连接断开忽略） |
| `_error(status, msg)` | L1780–L1781 | 错误响应 `{"error": msg}` |
| `_serve_static(path, head_only=False)` | L1783–L1805 | 静态文件服务：根 → `/index.html`，`mimetypes` 猜类型，支持 HEAD |
| `log_message(fmt, *args)` | L1807–L1808 | 日志重定向到 stderr，前缀 `[对阵]` |
| `do_OPTIONS()` | L1810–L1811 | CORS 预检 → `204 {}` |
| `do_HEAD()` | L1813–L1820 | HEAD 请求：非 `/api` 走静态，`/api` 返回 `200 {}` |
| `do_GET()` | L1822–L1834 | GET：非 `/api` 走静态；`/api` 路径 `rstrip("/")` 后交 `_route` |
| `do_POST()` | L1836–L1849 | POST：读 body → `json.loads`（失败 `{}`）→ `_route_post` |
| `_route(path, query)` | L1851–L1886 | GET 路由分发（见下） |
| `_route_post(path, payload)` | L1888–L1905 | POST 路由分发（见下） |

**GET 路由表**（`_route`，L1851–L1886）：

| 路径 | 处理函数 | 默认参数 |
|------|----------|----------|
| `/api/health` | 内联 | — |
| `/api/countries` | `api_countries()` | — |
| `/api/sizes` | `api_sizes(q("region","JP"))` | region=JP |
| `/api/models` | `api_models(q("region","JP"), q("size","65"))` | region=JP, size=65 |
| `/api/ai-recommend` | `api_ai_recommend(q("region","JP"), q("size","65"))` | region=JP, size=65 |
| `/api/spec` | `api_spec_compare([int...])` | model_ids 必填 |
| `/api/radar` | `api_radar([int...])` | model_ids 必填 |
| `/api/ai-analysis` | `api_ai_analysis(ids, q("region"))` | model_ids 必填 |
| `/api/backfill` | `api_backfill(q("model_ids"), q("region"))` | — |
| `/api/backfill-status` | `api_backfill_status(q("model_id"))` | — |
| `/api/auto-trace` | `api_auto_trace(region, size, max_models, with_ai)` | max_models=50, with_ai=0 |

辅助函数 `q(name, default="")`（L1852）：从 `query` 取首个值并 strip，缺省返回 default。

**POST 路由表**（`_route_post`，L1888–L1905）：

| 路径 | 处理函数 | 请求体字段 |
|------|----------|------------|
| `/api/ai-analysis` | `api_ai_analysis(model_ids, region)` | `model_ids`: list, `region`: str |
| `/api/spec` | `api_spec_compare(model_ids, region)` | `model_ids`: list, `region`: str(默认 JP) |
| `/api/backfill` | `api_backfill(model_ids, region, skip_ai)` | `model_ids`: list, `region`: str, `skip_ai`: bool |
| `/api/backfill-crawl` | `api_backfill_crawl(adapter_code, series)` | `adapter_code`, `series` |
| `/api/auto-trace` | `api_auto_trace(region, size, max_models, with_ai)` | 见上 |

**CORS 行为**：仅当 `SETTINGS['cors']` 非空时，`_send` 与 `_serve_static` 附加 `Access-Control-Allow-Origin/Headers/Methods` 头（L1768–L1772, L1800–L1802）。

---

## 五、启动入口与 LLM 配置注入

### `_setup_xinghai()` — LLM 配置注入（L2049–L2098）

**作用**：从 DSH settings 注入 xinghai-ultra 本地转换代理配置到 `config` 模块，使 `LLMClient` 走 `http://127.0.0.1:8082/v1`（OpenAI 兼容协议）。

**注入逻辑**：
1. 设置本地代理 URL `proxy_url = "http://127.0.0.1:8082/v1"`，`placeholder_key = "local-proxy-placeholder"`（L2057–L2058）。
2. **仅在环境变量未显式设置时注入**（L2060–L2078）：
   - `OVERSEAS_LLM_BASE_URL / API_KEY / MODEL / PROVIDER`
   - `OVERSEAS_XINGHAI_BASE_URL / API_KEY / USER_KEY`
3. 强制重载 `config` 模块属性（L2081–L2092，因 `config` 在 import 时已读一次环境变量）：
   - `_cfg.XINGHAI_BASE_URL/API_KEY/USER_KEY/MODEL` = 代理地址/占位值/xinghai-ultra
   - `_cfg.LLM_BASE_URL/API_KEY/MODEL/PROVIDER/USER_KEY` = 同上
   - `_cfg._LLM_PROVIDER_DEFAULTS["xinghai"]["uses_user_key"] = False`（本地代理不需 user_key 校验）
4. 关闭流式与 SSL 校验（L2094–L2098）：
   - `OVERSEAS_LLM_STREAM=false`，`OVERSEAS_LLM_SSL_VERIFY=false`
   - `_cfg.LLM_STREAM=False`，`_cfg.LLM_SSL_VERIFY=False`

**行号**：L2049–L2098。

### `main()` — 启动入口（L2101–L2178）

**流程**：
1. 调 `_setup_xinghai()` 注入 LLM 配置（L2102）。
2. 命令行参数（L2103–L2110）：
   - `--host`（默认 `127.0.0.1`）
   - `--port`（int，默认 `8090`）
   - `--db`（默认 `DEFAULT_DB`）
   - `--cors`（默认空）
   - `--enable-scheduled-trace`（flag，启用定时溯源，默认不开）
3. 写入 `SETTINGS['db']` 与 `SETTINGS['cors']`（L2112–L2113）。
4. 打印启动信息（L2115–L2128）：版本、DB、静态目录、xinghai-ultra 地址，并用 `urllib.request` 探测 `BASE_URL/models` 验证 AI 代理可用性（超时 3 秒，失败打印"将走规则兜底"）。
5. 创建 `ThreadingHTTPServer((host, port), Handler)`（L2131）。
6. **定时溯源**（L2135–L2167，仅 `--enable-scheduled-trace` 启用）：
   - 起一个 daemon `threading.Timer`，定时为"明天凌晨 2 点"。
   - 到点执行 `_scheduled_trace()`：调 `api_auto_trace(max_models=100, with_ai=False)`，打印扫描/补全数，并重排下一次（明天 2 点）。
   - 未启用则打印提示。
7. `server.serve_forever()`（L2170–L2174），`KeyboardInterrupt` 时 `server.shutdown()` 并打印"已停止"。

**启动命令**：
```bash
python -m overseas.benchmark_server                    # 127.0.0.1:8090
python -m overseas.benchmark_server --port 8090 --cors "*"
python -m overseas.benchmark_server --enable-scheduled-trace   # 启用每天 2 点全库溯源
```

---

## 附录：数据库表速查

| 表名 | 用途 | 在本服务中的使用 |
|------|------|------------------|
| `dim_product_model` | 型号主表 | 区域过滤、型号列表、尺寸统计 |
| `dim_spec_parameter` | 参数定义表 | 取 `param_id/param_name/param_unit/is_numeric/higher_is_better` |
| `fact_spec_value` | 规格值表 | 取 `value_text/value_numeric/source_term`，写回溯源结果 |
| `product` | 产品表（含 model 字段） | 均价查询的 JOIN |
| `price_snapshot` | 价格快照表（含 currency/price） | 均价查询、区域货币过滤 |
| `dim_spec_item_map` | 参数项映射表 | 本服务未直接使用（定义存在） |
| `dim_spec_normalization_map` | 归一化映射表 | 本服务用代码内 `_DISPLAY_NORMALIZE` 代替，未直接查此表 |
| `fact_benchmark_comparison` | 对阵结论存储表 | 本服务未直接写读（定义存在） |

> 本服务对价格的取数原则：**严格按 region 货币过滤，不 fallback 到任意货币**（"缺价 > 错价"），无该货币价格即跳过型号。
