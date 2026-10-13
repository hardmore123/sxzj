# 设计思路、误区、注意事项与踩坑经验

> 本文档整理竞品对阵分析系统在数据库设计、归一化引擎、区域过滤、AI分析、跨区域匹配等方面的**设计思路**、**常见误区**、**注意事项**和**踩坑经验**。
> 供下一位开发者理解"为什么这样设计"以及"千万不要怎么做"。

---

## 一、数据库设计误区与注意事项

### 误区1：以为fact_spec_value可以直接存原始规格值

**❌ 错误理解**：直接把spec_row的values_json或product_spec的value存到fact_spec_value即可。

**✅ 正确设计**：必须经过`_normalize_display()`正则归一化后才入库。

**为什么**：
- spec_row的原始值是多语言的（日文`倍速120Hz対応`、德文`120Hz`、法文`120Hz`）
- product_spec的原始值是英文格式（`120Hz Refresh Rate`、`Motion Rate 120`）
- 同一参数（刷新率120Hz）在不同品牌有5种以上写法
- 前端展示需统一格式（`120Hz`），否则对阵矩阵无法对比

**踩坑经历**：早期直接存原始值，前端对阵矩阵出现"倍速120Hz対応 vs 120Hz"的对比，用户无法理解。

### 误区2：以为dim_spec_item_map的source_table只是分类标签

**❌ 错误理解**：source_table只是一个备注字段，不影响逻辑。

**✅ 正确设计**：source_table决定映射走哪条路径。

```
source_table='spec_row'    → 从spec_row.item_ja映射（多语言项名）
source_table='product_spec' → 从product_spec.key映射（英文键名）
```

**踩坑经历**：早期不分source_table，把spec_row的item_ja和product_spec的key混在一起做LIKE匹配，导致日语项名误匹配英文键名。

### 误区3：以为confidence字段可有可无

**❌ 错误理解**：所有规格值都是抓取来的，可信度一样。

**✅ 正确设计**：confidence分6级，前端可根据置信度筛选展示。

| confidence | 含义 | 行数 | 来源 |
|-----------|------|------|------|
| 1.0 | 直接抓取(最高) | 8,530 | spec_row/product_spec直接映射 |
| 0.9 | 高置信度抓取 | 22,871 | 有extract_regex精确提取 |
| 0.8 | 中置信度 | 7,294 | 模糊匹配 |
| 0.6 | 跨品牌同尺寸 | 5,346 | 同尺寸不同品牌复制 |
| 0.5 | 跨区域复制 | 6,815 | 同品牌不同区域型号复制 |
| 0.4 | 模糊匹配(最低) | 808 | 核心码前4字符模糊匹配 |

**踩坑经历**：早期没有confidence字段，用户看到从欧洲型号复制的规格以为是本地数据，产生误解。

### 误区4：以为dim_product_model.our_brand是布尔值

**❌ 错误理解**：SQLite有BOOLEAN类型，our_brand是TRUE/FALSE。

**✅ 正确设计**：SQLite没有原生BOOLEAN，用TINYINT 0/1表示。

```sql
our_brand TINYINT NOT NULL DEFAULT 0  -- 1=海信(我方), 0=竞品
```

**注意事项**：
- `our_brand=1`的型号在对阵分析中显示为"我方(海信)"
- `our_brand=0`的型号显示为"竞品"
- 前端通过`is_our: bool(r['our_brand'])`转为布尔值

### 误区5：以为price_snapshot.currency总是干净的

**❌ 错误理解**：currency字段一定是标准货币代码（JPY/USD/EUR等）。

**✅ 实际情况**：有大量脏数据！

| 脏currency | 行数 | 原因 |
|-----------|------|------|
| 空字符串`''` | 1,529 | 爬虫未提取到货币 |
| `Saltar a` | 239 | 西班牙语"跳转到"被误识别为货币 |
| `RUB` | 5 | 俄罗斯卢布(未在21国配置中) |
| `$ 49.999` | 4 | 价格文本被误存为currency |
| `$829,00`等 | ~20 | 各种价格格式误存 |

**注意事项**：
- 查询price_snapshot时**必须**加`WHERE currency IN ('JPY','USD',...)`过滤
- `benchmark_server.py`的`_region_filter()`已处理：按区域严格匹配货币
- 清理脏数据时不要直接DELETE，先备份

### 误区6：以为model_id+param_id在fact_spec_value中是唯一的

**❌ 错误理解**：一个型号一个参数只有一条记录。

**✅ 实际情况**：存在重复！

```
model_id=3204 param_id=5: 3条  (同一型号同一参数有3条记录)
model_id=3204 param_id=10: 3条
model_id=4 param_id=6: 2条
```

**原因**：同一参数可能从spec_row和product_spec两个来源都生成了fact_spec_value。

**注意事项**：
- 查询fact_spec_value时用`ORDER BY confidence DESC LIMIT 1`取最高置信度
- `api_spec_compare()`的SQL用`ORDER BY CASE WHEN f.source_table LIKE 'spec_row%' THEN 1 ...`做优先级排序
- 清理重复数据时保留confidence最高的那条

### 误区7：以为spec_row.values_json是单型号值

**❌ 错误理解**：一行spec_row对应一个型号的规格值。

**✅ 正确设计**：一行spec_row对应一个**系列**的所有型号的规格值。

```json
// values_json示例（一个系列4个型号共享一行）：
{
  "K-115XR90M2": "●",
  "K-85XR90M2": "●",
  "K-75XR90M2": "●",
  "K-65XR90M2": "—"
}
```

**注意事项**：
- 解析spec_row时需遍历values_json中的每个型号
- Sony系列的共享值有特殊格式：`【K-65XR50】24.9[26.0]`（需按当前型号提取）
- `benchmark_server.py`的`_normalize_display()`有专门的Sony系列提取逻辑

---

## 二、归一化引擎设计思路与注意事项

### 设计思路：为什么要做两层映射？

**问题**：不同品牌用不同语言/格式描述同一规格：
- 刷新率：海信说"144Hz高刷"、Samsung说"Refresh Rate"、LG说"Motion Xcelerator"、Sony说"倍速対応"
- 分辨率：日文"画素数"、英文"Resolution"、德文"Auflösung"、法文"Résolution"

**解决方案**：两层映射

```
第一层: dim_spec_item_map (项名→参数ID)
  "画素数"(日) → param_id=10
  "Resolution"(英) → param_id=10
  "Auflösung"(德) → param_id=10
  ↓
  统一映射到18个标准参数

第二层: dim_spec_normalization_map (同义术语归一)
  "144Hz高刷" → 高刷族
  "Refresh Rate" → 高刷族
  "Motion Xcelerator" → 高刷族
  "倍速対応" → 高刷族
  ↓
  前端展示时统一归族名
```

### 注意事项1：_DISPLAY_NORMALIZE正则规则的顺序至关重要

**设计原则**：**最具体的规则放前面，最宽泛的规则放后面**（首匹配胜出）。

**踩坑案例**：
```python
# ❌ 错误顺序：通用Mini LED规则在REGZA专用规则前
r'Mini\s*LED.*': 'Mini LED',          # 会先匹配，把REGZA的特殊描述吞掉
r'新開発\s*高輝度\s*ファイン\s*Mini\s*LED.*': 'Mini LED',  # 永远不会被匹配到

# ✅ 正确顺序：REGZA专用规则在前
r'新開発\s*高輝度\s*ファイン\s*Mini\s*LED.*': 'Mini LED',  # 先匹配REGZA专用
r'Mini\s*LED.*': 'Mini LED',          # 兜底匹配通用
```

**踩坑案例2**：TCL的`VAパネル 直下型LEDバックライト`被误映射到刷新率（因为含"VA"被HDMI规则先匹配）：
```python
# 必须在通用VA规则前加TCL专用排除
r'VAパネル\s*直下型.*': '—',  # TCL专用：不是刷新率，标记为无数据
```

### 注意事项2：mm_to_cm特殊处理

**设计原因**：TCL/松下的规格尺寸用mm表示（`幅1428 x 高さ804`），需转cm显示。

**实现方式**：_DISPLAY_NORMALIZE中`replacement='mm_to_cm'`是特殊标记，触发专门的单位转换逻辑：
```python
if replacement == 'mm_to_cm':
    # 幅1428 x 高さ804 → 142.8×80.4cm (除以10)
    m = re.match(r'幅\s*(\d+)\s*x\s*高さ\s*(\d+).*', clean)
    return f'{int(m.group(1))/10:g}×{int(m.group(2))/10:g}cm'
```

**注意事项**：新增品牌时如果尺寸单位不是cm，需检查是否需要类似的转换逻辑。

### 注意事项3：纯数字值的兜底处理

**设计原因**：有些规格值是纯数字（如亮度`400`），需自动加单位。

```python
# 纯数字兜底：加单位
if re.match(r'^\d+\.?\d*$', text):
    unit_map = {'PEAK_BRIGHTNESS': 'nits', 'REFRESH_RATE': 'Hz', ...}
    unit = unit_map.get(param_code, '')
    return f'{text}{unit}' if unit else text
```

**踩坑**：不加单位时前端显示`400`，用户不知道是400nits还是400Hz。

### 注意事项4：西班牙语规格归一

**设计原因**：南美爬取的LATAM数据含西班牙语规格值。

```python
# 西班牙语归一规则（加在_DISPLAY_NORMALIZE末尾）
r'Procesador\s*α': 'Alpha处理器',
r'Directa': '直下式LED',
r'Sí\s*\(hasta\s*(\d+)Hz\)': r'\1Hz',  # "Sí (hasta 120Hz)" → "120Hz"
r'No\s*compatible': '—',
```

**注意事项**：新增南美国家时检查是否有新的西班牙语格式需要归一。

---

## 三、区域过滤设计思路与误区

### 设计思路：为什么日本和非日本用不同的过滤策略？

**日本线特殊性**：
- 数据来源：Kakaku价格.com + 品牌官网(日语)
- brand_raw含"日本"标识：`Sony BRAVIA（日本）`、`Hisense（日本）`
- 海信日本型号有独立的JPY价格

**非日本线**：
- 数据来源：各国零售电商站
- 按price_snapshot.currency过滤即可
- 同一型号可能在多个国家有不同货币价格

**实现方式**：
```python
def _region_filter(region):
    if region == 'JP':
        # 日本线：按brand_raw含"日本"过滤 + 海信JPY
        return ("(brand_raw LIKE '%日本%' OR (our_brand=1 AND EXISTS(...JPY)))", [])
    else:
        # 非日本线：按货币过滤型号
        currency = COUNTRIES[region]['currency']
        return ("model_code IN (SELECT p.model FROM product p JOIN price_snapshot ps ...WHERE ps.currency=?)", [currency])
```

### 误区1：以为可以跨货币fallback

**❌ 错误做法**：某区域无JPY价格时，fallback到USD价格显示。

**✅ 正确设计**："缺价 > 错价"——宁可不显示该型号，也不显示错误货币的价格。

```python
# benchmark_server.py 第1442行
if price is None:
    continue  # 确实无价格 → 跳过，不fallback到其他货币
```

**踩坑经历**：早期会fallback到任意货币，导致日本线显示哥伦比亚的COP价格。

### 误区2：以为所有型号名都是标准格式

**❌ 错误理解**：product.model都是真实的型号编码（如`65U8R`、`55QNED70BSA`）。

**✅ 实际情况**：有大量脏数据型号名！

| 脏型号名 | 原因 |
|---------|------|
| `180HZ` | 规格词被误存为型号 |
| `60LED` | 规格词 |
| `38751` | 纯数字（可能是SKU） |
| `REF=SR` | 脏数据标记 |
| `144HZ` | 规格词 |
| `/MasterData/Vendor/...` | URL路径被误存 |

**解决方案**：`_is_valid_model_code()`过滤函数：
```python
def _is_valid_model_code(code):
    if not code or len(code) < 4: return False
    if re.match(r'^\d+$', code): return False  # 纯数字过滤
    if re.match(r'^\d{2,4}[A-Z]{1,3}$', code): return False  # 数字+少量字母(规格词)
    if not any(c.isalpha() for c in code): return False  # 至少含1个字母
    return True
```

**注意事项**：新增站点爬取时检查是否有新的脏数据格式需要过滤。

---

## 四、跨区域匹配设计思路与误区

### 设计思路：为什么要跨区域匹配？

**问题**：同一品牌同一系列的型号在不同区域有不同后缀，但规格相同：
- 美国：`55QNED80PUA`（PUA=美国后缀）
- 加拿大：`55QNED80AUA`（AUA=加拿大后缀）
- 欧洲：`55QNED80PSA`（PSA=欧洲后缀）
- 日本：`55QNED80`（无后缀）

**核心码**：去掉区域后缀后的公共部分`55QNED80`，用来匹配跨区域同款。

### 核心码提取规则

```python
# Samsung: 去UN/GQ/QE前缀 + 尺寸
'UN65QN900B' → 'QN900B'

# LG: 去OLED/QNED/NANO前缀
'OLED65C5PUA' → '65C5'

# Sony: 去K前缀
'K-65XR90M2' → '65XR90M2'

# 去区域后缀
'55QNED80PUA' → '55QNED80'  # 去PUA
'55QNED80PSA' → '55QNED80'  # 去PSA
```

### 误区1：以为核心码匹配是精确的

**❌ 错误理解**：核心码完全相同才能匹配。

**✅ 实际设计**：4轮渐进匹配，从精确到模糊：

| 轮次 | 匹配策略 | confidence | 说明 |
|------|---------|-----------|------|
| 1 | 精确核心码 | 0.5 | 去后缀后完全相同 |
| 2 | 品牌+尺寸+核心码 | 0.5 | 同品牌同尺寸同核心码 |
| 3 | 同尺寸+核心码前4字符 | 0.4 | 模糊匹配 |
| 4 | 同品牌+同尺寸 | 0.4 | 最宽松：同品牌同尺寸即可 |

**注意事项**：越宽松的匹配越容易出错，confidence越低。前端可根据confidence阈值筛选。

### 误区2：以为跨区域复制的source_term是干净的

**✅ 实际设计**：跨区域复制时source_term会加上来源标记：
```python
source_term = original_term + f' (from {source_model_code})'
# 如：'ピーク輝度 (from 55QNED80PUA)'
```

**注意事项**：查询时不要直接用source_term做精确匹配，要用LIKE或提取原始项名。

### 误区3：海信型号的核心码提取与其他品牌不同

**特殊处理**：海信型号如`HISENSE55Q6N`，核心码`Q6N`与已有的`QD6QF`不完全匹配。

**解决方案**：海信用"尺寸+Q6/Q7系列前缀"模糊匹配：
```python
# HISENSE55Q6N → 匹配 55QD6QF (同55寸 + Q6系列前缀)
if brand == 'Hisense':
    # 匹配尺寸+Q6/Q7系列前缀 → 从最近的同尺寸Hisense型号复制
```

---

## 五、AI分析设计思路与误区

### 设计思路：为什么要三层降级？

```
第1层: LLM可用 → xinghai-ultra生成优势/劣势/机会/策略
  ↓ (LLM不可用)
第2层: llm_client模块不存在 → 规则兜底
  ↓ (配置不完整)
第3层: LLM配置但调用失败 → 规则兜底 + 返回error信息
```

### 误区1：以为AI不可用系统就不能用

**✅ 正确设计**：AI不可用时走`_fallback_analysis()`规则兜底，返回结构化JSON：
```python
def _fallback_analysis(spec, radar, our_model):
    # 基于规则分析：数值型参数比较大小，文本型参数比较有无
    # 返回与AI相同的JSON结构，但标注fallback=true
    return {
        'advantage_points': [...],  # 基于数值比较生成
        'disadvantage_points': [...],
        'fallback': True
    }
```

**注意事项**：前端收到`fallback=true`时可以显示"规则分析"标记而非"AI分析"。

### 误区2：以为AI prompt只需传规格数据

**✅ 正确设计**：prompt包含完整的上下文和输出格式要求：

```python
prompt = f"""你是电视产品竞品分析专家。基于以下规格对比数据，生成"我方(海信)"对阵竞品的结论。
要求输出 JSON（不要 markdown 代码块），格式:
{{
  "advantage_points": [{{"point":"...","evidence":"...","confidence":0.9}}],
  "disadvantage_points": [{{"point":"...","evidence":"...","confidence":0.85}}],
  "opportunity_points": [{{"point":"...","evidence":"...","confidence":0.8}}],
  "strategy_suggestion": "一句话策略建议"
}}
优势点：我方领先的参数（数值高/功能全）；劣势点：我方落后之处；机会点：可差异化或补位的市场机会。
每条要可溯源到具体参数或价格。置信度0-1。

规格对比数据:
{summary}
"""
```

**注意事项**：
- 只传前12个参数（`PARAM_ORDER[:12]`），不是全部18个——AI处理太多参数效果下降
- temperature=0.2（低温度保证输出稳定）
- max_tokens=2048（控制响应长度）

### 误区3：以为_setup_xinghai()需要手动配置环境变量

**✅ 正确设计**：`_setup_xinghai()`在服务启动时自动注入LLM配置：
```python
def _setup_xinghai():
    proxy_url = "http://127.0.0.1:8082/v1"
    placeholder_key = "local-proxy-placeholder"
    # 自动注入环境变量（只在未显式设置时）
    if not os.environ.get("OVERSEAS_LLM_BASE_URL"):
        os.environ["OVERSEAS_LLM_BASE_URL"] = proxy_url
    # ... 其他配置
```

**注意事项**：
- 本地代理不需要真实API Key，用占位值`local-proxy-placeholder`
- `uses_user_key`设为False，否则`configured`检查会失败
- SSL验证关闭（本地HTTP不需要）
- 流式关闭（非流式更稳定）

---

## 六、Backfill参数补全设计思路与误区

### 设计思路：三层回溯策略

```
第1层 db   — 在已有数据库中用关键词搜索真实记录的值
  ↓ (未找到)
第2层 site — 调spec_crawl重抓官网规格页（真实抓取）
  ↓ (抓取失败)
第3层 ai   — 调AI查找该型号的官网URL，供第2层真实抓取
  ↓ (AI也找不到)
标记"待维护"，绝不编造
```

### 核心原则：AI永远不产出规格值

**❌ 错误做法**：让AI直接根据型号名猜测规格参数值。

**✅ 正确设计**：AI只负责"查找去哪里能抓到真实数据"，找到URL后交给爬虫真实抓取。

```python
# backfill.py 注释
# 核心原则：
#   - AI 永远不产出规格值，只查找"去哪里能抓到真实数据"
#   - AI 找到 URL 后交给 spec_crawl 真实抓取，抓到的才入库
#   - 找不到 URL 或抓取失败 → 诚实标记"待维护"，绝不编造
```

**踩坑经历**：早期让AI猜测规格值，结果出现了"65寸电视功耗2000W"这种荒谬数据。

### 注意事项：18参数的精确搜索关键词

每个参数有专门的搜索关键词列表，用于第1层数据库回溯：
```python
BACKFILL_KEYWORDS = {
    'PEAK_BRIGHTNESS': ['ピーク輝度', 'Peak Brightness', '最大輝度', ...],
    'REFRESH_RATE': ['倍速対応', '倍速駆動', 'リフレッシュレート', 'Refresh Rate', ...],
    # ... 18个参数各有3-10个关键词
}
```

**注意事项**：新增品牌时检查是否有新的规格项名不在关键词列表中。

---

## 七、价格档位设计思路与误区

### 设计思路：为什么要为每国每尺寸手动配置价格区间？

**问题**：各国消费水平不同，不能自动计算。
- 日本55寸中端：100,000-200,000 JPY（约500-1000美元）
- 美国55寸中端：400-900 USD
- 哥伦比亚55寸中端：1,500,000-3,000,000 COP（约400-800美元）

**设计**：`COUNTRIES`字典中每国每尺寸配4档（入门/中端/高端/旗舰）：
```python
'JP': {'bands': {55: {'入门':(0,100000), '中端':(100000,200000), '高端':(200000,350000), '旗舰':(350000,9e9)}}},
'US': {'bands': {55: {'入门':(0,400), '中端':(400,900), '高端':(900,1800), '旗舰':(1800,9e9)}}},
```

### 误区：以为可以用汇率自动换算

**❌ 错误做法**：用USD基准价 × 汇率 = 各国价格区间。

**✅ 正确设计**：手动配置，因为：
- 关税/运费/渠道成本各国不同
- 消费者购买力不同
- 品牌定位不同（海信在日本是中端，在美国是入门-中端）

---

## 八、已知数据质量问题

### 问题1：spec_row有3,483个item_ja未映射

```
spec_row共有 ~5,500 个不同的item_ja
其中 3,483 个未在dim_spec_item_map中建立映射
```

**影响**：这些规格项的值不会生成fact_spec_value，导致部分型号覆盖率偏低。

**原因**：多为欧洲品牌（德/法/意大利语）的规格项名，日本线开发时未覆盖。

**处理建议**：对新发现的item_ja，在dim_spec_item_map中添加映射（AUTO状态），批量确认后改为CONFIRMED。

### 问题2：product_spec有2,282个key未映射

```
product_spec共有 ~3,300 个不同的key
其中 2,282 个未在dim_spec_item_map中建立映射
```

**影响**：大量零售站规格未归一化。

**原因**：零售站规格键名非常多样（`Product Dimensions`、`Standing screen display size`、`Summary│Screen Size`等）。

### 问题3：dim_product_model有138个空品牌/空型号

**影响**：这些型号无法参与对阵分析。

**处理建议**：清理或补全这些记录。

### 问题4：fact_spec_value存在model_id+param_id重复

```
model_id=3204 param_id=5: 3条  (最多重复3条)
```

**影响**：查询时可能返回不一致的值。

**处理建议**：保留confidence最高的那条，删除其余。

### 问题5：source_term含"from"标记但查询不到

```
fact_spec_value中source_term LIKE '% from %': 0条
```

**说明**：跨区域复制时本应添加`(from XXX)`标记，但实际数据库中没有。可能是批量脚本未加标记或已被清理。

---

## 九、运维注意事项

### 注意事项1：编辑benchmark_server.py前必须停服务

**原因**：Windows文件锁定机制，`__pycache__`中的`.pyc`被占用。

**正确流程**：
```bash
# 1. 停Python进程
taskkill /F /IM python.exe

# 2. 删__pycache__
rmdir /s /q overseas\__pycache__

# 3. 编辑代码

# 4. 重新启动
python run_benchmark_server.py --port 8090
```

**不这么做会报**：`WinError 1175 EIO`错误，文件无法保存。

### 注意事项2：三个数据库副本会漂移

| 数据库 | 用途 | 同步方向 |
|--------|------|---------|
| 源DB `海外\overseas_v4_20260929.db` | 开发主库 | ← 手动同步 |
| 爬取DB `海外数据爬取_部署包\data\overseas.db` | 爬虫写入库 | → 需手动同步到源DB |
| 部署DB `竞品对阵分析_部署包\data\` | 部署副本 | ← 从源DB复制 |

**注意事项**：爬虫在爬取DB写入新数据后，需手动同步到源DB，再复制到部署DB。

### 注意事项3：WAL模式可能导致数据库锁定

**症状**：`sqlite3.OperationalError: database is locked`

**原因**：SQLite WAL模式 + 多线程/多进程同时写入。

**解决**：
```bash
# 删除WAL和SHM文件（会丢失未提交事务）
del data\overseas_v4_20260929.db-wal
del data\overseas_v4_20260929.db-shm
```

### 注意事项4：新增国家/区域的完整步骤

1. 在`COUNTRIES`字典中添加国家配置（货币+价格档位）
2. 在前端`index.html`的国家选择器中添加`<option>`
3. 确认`price_snapshot`中有该货币的数据
4. 确认`dim_product_model`中有该区域的型号
5. 运行跨区域匹配补全该区域无规格型号的fact_spec_value

### 注意事项5：新增品牌的归一化步骤

1. 在`dim_product_model`中标记该品牌的型号
2. 在`dim_spec_item_map`中添加该品牌规格项名的映射
3. 在`_DISPLAY_NORMALIZE`中添加该品牌的归一化正则规则
4. 在`dim_spec_normalization_map`中添加该品牌的同义术语
5. 在`backfill.py`的`BACKFILL_KEYWORDS`中添加该品牌的搜索关键词
