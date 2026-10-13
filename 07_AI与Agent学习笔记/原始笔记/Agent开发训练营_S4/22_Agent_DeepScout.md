## 1. **职责定位**

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">这是 </span>**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">最复杂的 Agent </span>**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">，代码长达 </span>**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">1390 行 </span>**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">。</span>

![](<images/2.2.2 Agent - DeepScout（深度侦探）-a96a05b7469dbf74fc0873366b04770e.png>)



## **核心代码位置**

**文件路径&#x20;**：\`backend/app/service/deep\_research\_v2/agents/scout.py\`（1390行）



## **Prompt 设计**

![](<images/2.2.2 Agent - DeepScout（深度侦探）-3586082de619833e846388c828dfccef.png>)

### 1. **SEARCH\_ANALYSIS\_PROMPT（第59-124行）**

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">用于分析搜索结果并提取结构化信息。</span>

**关键特性&#x20;**：

* 支持假设验证（ `hypothesis_support: "supports/refutes/neutral"` ）

* 信源可信度评分（ `credibility_score: 0.0-1.0` ）

* 信源追溯（ `source_tracing_queries` ）

* 线索追踪（ `follow_up_queries` ）

![](<images/2.2.2 Agent - DeepScout（深度侦探）-1ad2b25573e8a96cd34cbbb3840e8c7b.png>)



### **DEEP\_READ\_PROMPT（第126-164行）**

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">用于深度阅读长文本（网页正文）。</span>

**功能&#x20;**：

* 提取文档摘要

* 提取关键事实

* 提取数据表格

* 提取重要原文引用



### **补充搜索分析 Prompt（第472-503行，内嵌在代码中）**

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">用于分析补充搜索结果（CriticMaster 审核后发现信息缺失时使用）。</span>



### **深度搜索分析 Prompt（第954-991行，内嵌在代码中）**

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">用于分析递归搜索的结果（信源追溯或线索追踪）。 </span> <span style="color: rgb(216,57,49); background-color: rgba(255,246,122,0.8)">【核心机制】</span>

![](<images/2.2.2 Agent - DeepScout（深度侦探）-49714baf2d346e4aac532cebb526e262.png>)



## **核心实现**

### **1. process() 入口（第193-272行）**

**关键逻辑&#x20;**：

1\. **双模式路由&#x20;**：正常研究 vs 补充搜索

2\. **股票数据自动识别&#x20;**：如果用户查询涉及上市公司，自动调用股票 API

3\. **并行搜索&#x20;**： <span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">使用 `asyncio.gather()` 同时研究多个章节</span>

4\. **实时事件流&#x20;**：发送 SSE 事件到前端



### **2. 并行搜索实现（asyncio.gather）**

#### **\_research\_section() - 章节研究（第533-795行）**

**并行策略&#x20;**：

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">每个 </span>`_research_section()`  <span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">是一个独立的协程，同时执行，提升效率。</span>



### **3. 信源可信度评级算法**

#### **评分标准（SEARCH\_ANALYSIS\_PROMPT 第116-123行）**

#### **LLM 自动评分**

LLM 根据 `source_name` 和 `source_url` 判断类型，输出 `credibility_score` ：



### **4. 交叉验证机制**

#### **事实去重（第1318-1333行）**

#### **指纹计算（第1307-1316行）**

**原理&#x20;**：

* 提取内容中的数字和中文关键词

* 拼接后计算 MD5 哈希

* 相同指纹认为是重复事实



### **5. 递归搜索（深度追踪）**

#### **\_execute\_deep\_search() - <span style="color: rgb(216,57,49); background-color: rgba(255,246,122,0.8)">递归搜索核心（第796-931行）</span>**

**递归深度控制&#x20;**：

* `depth=1` ：第一层递归

* `depth=2` ：第二层递归（最大深度）

* 超过 `max_depth` 自动停止

**两种递归类型&#x20;**：

1\. **信源追溯&#x20;**（source\_tracing）：例如文章说"据国家统计局数据"，自动搜索"国家统计局 2024 汽车销量"

2\. **线索追踪&#x20;**（follow\_up）：例如发现"特斯拉在华销量"，自动追踪"特斯拉 中国市场份额"



### **6. 网络搜索实现**

#### **\_execute\_search() - Bocha API（第1058-1122行）**

**关键特性&#x20;**：

* 使用 Bocha Web Search API（国内搜索引擎）

* 支持搜索结果摘要（ `summary=True` ）

* 内存缓存避免重复搜索

* 异步执行（ `asyncio.to_thread` ）



### **7. 本地知识库搜索**

#### **\_execute\_local\_search() - Milvus 向量检索（第1002-1055行）**

**向量检索流程&#x20;**：

1. 调用 `generate_embedding()` 生成查询向量

2. Milvus 向量相似度搜索

3) 返回 Top-K 结果



### **8. 补充搜索（审核后回退）**

#### **\_supplementary\_research() - 补充搜索阶段（第274-371行）**

**触发条件&#x20;**：

* CriticMaster 审核发现信息缺失

* 设置 `state["phase"] = ResearchPhase.RE_RESEARCHING.value`

* 设置 `state["pending_search_queries"] = ["查询1", "查询2"]`







## **深度阅读（长文本处理）**

### **deep\_read\_url() - 网页抓取（第1172-1214行）**

### **\_extract\_text\_from\_html() - 正文提取（第1216-1305行）**



**三种策略&#x20;**：

1\. **trafilatura&#x20;**：专业网页正文提取库（效果最好，优先使用）

2\. **BeautifulSoup&#x20;**：通用 HTML 解析（备选）

3\. **正则表达式&#x20;**：简单粗暴的文本提取（最后的备选）



## **知识图谱更新**

### **\_update\_knowledge\_graph() - 添加实体和关系（第1335-1363行）**

**示例输出&#x20;**：



## **假设验证**

### **\_update\_hypothesis\_status() - 更新假设状态（第1365-1388行）**

**状态转换&#x20;**：

* `unverified` → `supported` （有2条以上支持证据）

* `unverified` → `refuted` （有2条以上反驳证据）

* `unverified` → `partially_supported` （有证据但不足）





## **模型选择**

**配置位置&#x20;**：\`backend/app/config/llm\_config.py\`





## **SSE 事件流**

DeepScout 发送的 SSE 事件：

| 事件类型                | 说明              | 示例                                                |
| ------------------- | --------------- | ------------------------------------------------- |
| `research_step`     | 开始/完成搜索         | `{"step_type": "searching", "status": "running"}` |
| `action`            | Agent 动作        | `{"tool": "parallel_search", "queries": [...]}`   |
| `search_progress`   | 搜索进度            | `{"query": "AI市场", "results_count": 10}`          |
| `search_results`    | 搜索结果（增量）        | `{"results": [...], "isIncremental": true}`       |
| `observation`       | 观察结果            | `{"facts_count": 15, "duplicates_removed": 3}`    |
| `stock_quote`<br /> | 股票行情（不包含）<br /> | `{"code": "600519", "price": "1500.00"}`          |
| `thought`           | 思考过程            | `{"content": "追溯原始数据源..."}`                       |



## **总结**

DeepScout 是系统中 **<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">最复杂的 Agent </span>**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">（1390行代码） </span>，其核心能力：

1\. **并行搜索&#x20;**：使用 \`asyncio.gather()\` 同时搜索多个章节

<span style="color: rgb(216,57,49); background-color: rgba(255,246,122,0.8)">2. </span>**<span style="color: rgb(216,57,49); background-color: rgba(255,246,122,0.8)">递归追踪 </span>**<span style="color: rgb(216,57,49); background-color: rgba(255,246,122,0.8)">：自动信源追溯和线索追踪（最大深度2层）【核心】</span>

3\. **信源评级&#x20;**：LLM 自动评估来源可信度（0-1分）

4\. **去重机制&#x20;**：基于语义指纹的事实去重

5\. **假设验证&#x20;**：寻找支持或反驳研究假设的证据

6\. **深度阅读&#x20;**：提取网页正文，支持 trafilatura、BeautifulSoup、正则三种策略

7\. **知识图谱&#x20;**：自动构建实体关系图

8\. **双模搜索&#x20;**：支持网络搜索（Bocha API）和本地知识库（Milvus）

9\. **股票集成&#x20;**：自动识别上市公司并获取实时行情（不包含）

10\. **补充搜索&#x20;**：审核后智能回退，补充缺失信息



下一章节将介绍 DataAnalyst（数据分析师）。

