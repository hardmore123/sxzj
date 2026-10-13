## 1. **职责定位**

![](<images/2.2.1 Agent - ChiefArchitect（总架构师）-52d3f6afdf886dc2e819bd13f594ada1.png>)



## **核心代码位置**

**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">文件路径 </span>**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">：`backend/app/service/deep_research_v2/agents/architect.py`（351行）</span>

## **Prompt 设计**

### **1. PLANNING\_PROMPT（第29-63行）**

这是初始规划的核心 Prompt，要求 LLM 生成扁平化的 JSON 结构：

```python
PLANNING_PROMPT = """研究课题：{query}

请为该课题生成研究大纲和研究假设，输出JSON格式如下：

{{
  "hypothesis_1": "关于市场/行业趋势的假设（需要验证）",
  "hypothesis_2": "关于竞争格局或技术发展的假设（需要验证）",
  "hypothesis_3": "关于政策或外部因素影响的假设（需要验证）",
  "sec_1_title": "市场概况",
  "sec_1_desc": "描述市场规模、增速",
  "sec_1_query": "搜索关键词",
  "sec_2_title": "竞争格局",
  "sec_2_desc": "描述主要企业",
  "sec_2_query": "搜索关键词",
  ...
  "questions": "核心问题1;核心问题2;核心问题3"
}}
```

**设计要点&#x20;**：

* 使用扁平化结构（ `sec_1_title` , `sec_2_desc` ）而非嵌套数组，降低 LLM 输出错误率

* 每个章节包含 `title` 、 `desc` 、 `query` 三元组

* 研究假设（ `hypothesis_1/2/3` ）用于后续假设驱动研究

* 核心问题（ `questions` ）用分号分隔，便于解析



### **2. REVISION\_PROMPT（第65-96行）**

用于动态调整研究大纲：

```python
REVISION_PROMPT = """你是总架构师，需要根据研究进展动态调整大纲。

## 原始问题
{query}

## 当前大纲
{current_outline}

## 新发现的重要信息
{new_findings}

## 当前进度
- 已完成章节: {completed_sections}
- 收集的事实数量: {facts_count}
- 发现的数据点: {data_points_count}

## 任务
评估是否需要调整大纲。可能的调整包括：
1. 新增章节（发现了重要的新方向）
2. 删除章节（发现某方向信息太少）
3. 调整章节顺序或优先级
4. 细化或合并章节

输出JSON格式：
{{
    "needs_revision": true或false,
    "revision_reason": "调整原因",
    "revised_outline": [...],  // 如果needs_revision为true
    "new_search_queries": ["新增的搜索关键词"]
}}
```

**设计要点&#x20;**：

* 根据研究进展动态调整

* 支持增、删、改章节

* 可以根据新发现的信息调整搜索方向

## **代码实现**

### **1. process() 入口（第156-167行）**

```python
async def process(self, state: ResearchState) -> ResearchState:
    """
    处理入口

    根据当前阶段执行不同的规划任务
    """
    if state["phase"] == ResearchPhase.INIT.value:
        return await self._initial_planning(state)
    elif state["phase"] == ResearchPhase.REVIEWING.value:
        return await self._check_revision(state)
    else:
        return state
```

**路由逻辑&#x20;**：

* `INIT` 阶段 → 执行初始规划

* `REVIEWING` 阶段 → 检查是否需要修订大纲（当前未启用）

* 其他阶段 → 直接返回



### **2. \_initial\_planning() 核心流程（第169-310行）**

完整流程：

```python
async def _initial_planning(self, state: ResearchState) -> ResearchState:
    """初始规划"""
    self.logger.info(f"Starting initial planning for: {state['query'][:50]}...")

    # 1. 发送 research_step 开始事件
    self.add_message(state, "research_step", {
        "step_id": f"step_planning_{uuid.uuid4().hex[:8]}",
        "step_type": "planning",
        "title": "研究计划",
        "subtitle": "分析问题，制定大纲",
        "status": "running",
        "stats": {}
    })

    # 2. 发送状态消息
    self.add_message(state, "thought", {
        "agent": self.name,
        "content": "正在分析研究问题，构建知识图谱和研究大纲..."
    })

    # 3. 调用LLM生成规划 - 带重试机制
    prompt = self.PLANNING_PROMPT.format(query=state["query"])
    result = None
    max_retries = 2

    for attempt in range(max_retries + 1):
        response = await self.call_llm(
            system_prompt="你是一位专业的行业研究规划师。请严格按照要求的JSON格式输出，不要添加任何额外内容。",
            user_prompt=prompt,
            json_mode=True,
            temperature=0.3,
            max_tokens=16000
        )

        # Debug: 记录原始响应
        self.logger.info(f"Raw LLM response length: {len(response)} (attempt {attempt + 1})")

        result = self.parse_json_response(response)

        # 检查是否是扁平格式，需要转换
        if result and result.get("sec_1_title") and not result.get("outline"):
            result = self._convert_flat_to_outline(result)

        if result and result.get("outline") and len(result.get("outline", [])) >= 3:
            self.logger.info(f"Successfully parsed outline with {len(result['outline'])} sections")
            break

        # 诊断失败原因并重试
        if attempt < max_retries:
            self.logger.warning(f"Outline generation failed or incomplete, retrying...")
            # 简化提示词重试
            prompt = f"""请为"{state['query']}"生成研究大纲。

输出JSON格式：
{{"outline": [
    {{"id": "sec_1", "title": "章节标题", "description": "描述", "section_type": "mixed", ...}},
    ...更多章节(共5-8个)...
], "research_questions": ["问题1", "问题2"], "key_entities": []}}
"""

    if not result:
        state["errors"].append("Failed to generate research plan after retries")
        return state

    # 4. 更新状态
    state["key_entities"] = [e.get("name", "") for e in result.get("key_entities", []) if isinstance(e, dict)]
    state["mind_map"] = result.get("mind_map", {})
    state["research_questions"] = result.get("research_questions", [])
    state["hypotheses"] = result.get("hypotheses", [])
    state["knowledge_graph"] = {"nodes": [], "edges": []}

    # 5. 处理大纲 - 确保每个章节都有必要字段
    processed_outline = []
    for i, section in enumerate(result.get("outline", [])):
        if not isinstance(section, dict):
            continue
        processed_section = {
            "id": section.get("id", f"sec_{i+1}"),
            "title": section.get("title", f"章节{i+1}"),
            "description": section.get("description", ""),
            "section_type": section.get("section_type", "mixed"),
            "requires_data": section.get("requires_data", False),
            "requires_chart": section.get("requires_chart", False),
            "priority": section.get("priority", i+1),
            "search_queries": section.get("search_queries", [section.get("title", "")]),
            "status": "pending"
        }
        # 确保 search_queries 是列表且非空
        if not isinstance(processed_section["search_queries"], list):
            processed_section["search_queries"] = [str(processed_section["search_queries"])]
        processed_section["search_queries"] = [q for q in processed_section["search_queries"] if q and q.strip()]
        if not processed_section["search_queries"]:
            processed_section["search_queries"] = [processed_section["title"]]
        processed_outline.append(processed_section)

    state["outline"] = processed_outline

    # 6. 发送大纲事件
    self.add_message(state, "outline", {
        "understanding": result.get("understanding", {}),
        "key_entities": result.get("key_entities", []),
        "outline": result.get("outline", []),
        "research_questions": state["research_questions"]
    })

    # 7. 更新阶段
    state["phase"] = ResearchPhase.PLANNING.value

    # 8. 发送 research_step 完成事件
    self.add_message(state, "research_step", {
        "step_type": "planning",
        "title": "研究计划",
        "subtitle": "分析问题，制定大纲",
        "status": "completed",
        "stats": {
            "sections_count": len(processed_outline),
            "questions_count": len(state["research_questions"])
        }
    })

    return state
```



### **3. \_convert\_flat\_to\_outline() 格式转换（第107-154行）**

将扁平化 JSON 转换为标准 outline 格式：

```python
def _convert_flat_to_outline(self, flat_result: Dict) -> Dict:
    """将扁平JSON格式转换为标准outline格式"""
    outline = []
    for i in range(1, 10):  # 最多支持9个章节
        title_key = f"sec_{i}_title"
        desc_key = f"sec_{i}_desc"
        query_key = f"sec_{i}_query"

        if title_key not in flat_result:
            break

        section = {
            "id": f"sec_{i}",
            "title": flat_result.get(title_key, f"章节{i}"),
            "description": flat_result.get(desc_key, ""),
            "section_type": "mixed",
            "requires_data": i <= 2,  # 前两章需要数据
            "requires_chart": i <= 2,
            "search_queries": [flat_result.get(query_key, flat_result.get(title_key, ""))]
        }
        outline.append(section)

    # 处理研究问题
    questions_str = flat_result.get("questions", "")
    if isinstance(questions_str, str):
        research_questions = [q.strip() for q in questions_str.split(";") if q.strip()]
    else:
        research_questions = []

    # 处理研究假设
    hypotheses = []
    for i in range(1, 6):  # 最多5个假设
        h_key = f"hypothesis_{i}"
        if h_key in flat_result and flat_result[h_key]:
            hypotheses.append({
                "id": f"h_{i}",
                "content": flat_result[h_key],
                "status": "unverified",  # unverified, supported, refuted, partially_supported
                "evidence_for": [],
                "evidence_against": []
            })

    return {
        "outline": outline,
        "research_questions": research_questions,
        "hypotheses": hypotheses,
        "key_entities": []
    }
```

**关键细节&#x20;**：

* 前两章默认 `requires_data=True` 和 `requires_chart=True`

* 研究假设初始状态为 `"unverified"`

* 每个假设有 `evidence_for` 和 `evidence_against` 字段，用于后续 DeepScout 收集证据



## **JSON 解析与容错处理**

### **1. 重试机制（第193-236行）**

```python
max_retries = 2

for attempt in range(max_retries + 1):
    response = await self.call_llm(...)
    result = self.parse_json_response(response)

    # 检查是否是扁平格式，需要转换
    if result and result.get("sec_1_title") and not result.get("outline"):
        result = self._convert_flat_to_outline(result)

    if result and result.get("outline") and len(result.get("outline", [])) >= 3:
        break

    # 重试时简化 Prompt
    if attempt < max_retries:
        prompt = f"""请为"{state['query']}"生成研究大纲。

输出JSON格式：
{{"outline": [...], "research_questions": [...], "key_entities": []}}
"""
```



**容错策略&#x20;**：

1\. **格式检测&#x20;**：自动识别扁平格式并转换

2\. **质量检查&#x20;**：确保至少有 3 个章节

3\. **Prompt 简化&#x20;**：重试时使用更简化的 Prompt

4\. **失败记录&#x20;**：记录到 \`state\["errors"]\`



### **2. 字段补全（第257-280行）**



```python
processed_section = {
    "id": section.get("id", f"sec_{i+1}"),
    "title": section.get("title", f"章节{i+1}"),
    "description": section.get("description", ""),
    "section_type": section.get("section_type", "mixed"),
    "requires_data": section.get("requires_data", False),
    "requires_chart": section.get("requires_chart", False),
    "priority": section.get("priority", i+1),
    "search_queries": section.get("search_queries", [section.get("title", "")]),
    "status": "pending"
}

# 确保 search_queries 是列表且非空
if not isinstance(processed_section["search_queries"], list):
    processed_section["search_queries"] = [str(processed_section["search_queries"])]
processed_section["search_queries"] = [q for q in processed_section["search_queries"] if q and q.strip()]
if not processed_section["search_queries"]:
    processed_section["search_queries"] = [processed_section["title"]]
```



**容错逻辑&#x20;**：

* 缺失字段使用默认值

* `search_queries` 确保是非空列表

* 如果 `search_queries` 为空，使用章节标题作为搜索关键词



## **输出格式**

### 1. **outline（章节大纲）**

```json
[
  {
    "id": "sec_1",
    "title": "市场概况",
    "description": "描述市场规模、增速",
    "section_type": "mixed",
    "requires_data": true,
    "requires_chart": true,
    "priority": 1,
    "search_queries": ["中国AI市场规模", "AI市场增长率"],
    "status": "pending"
  },
  {
    "id": "sec_2",
    "title": "竞争格局",
    "description": "描述主要企业",
    "section_type": "qualitative",
    "requires_data": false,
    "requires_chart": false,
    "priority": 2,
    "search_queries": ["AI企业排名", "AI独角兽"],
    "status": "pending"
  }
]
```



### **hypotheses（研究假设）**

```json
[
  {
    "id": "h_1",
    "content": "中国AI市场规模将持续快速增长，CAGR超过30%",
    "status": "unverified",
    "evidence_for": [],
    "evidence_against": []
  },
  {
    "id": "h_2",
    "content": "llm是最大的细分领域",
    "status": "unverified",
    "evidence_for": [],
    "evidence_against": []
  }
]
```



### **3. research\_questions（核心问题）**

```json
[
  "中国AI市场的主要增长驱动因素是什么？",
  "哪些细分领域最具增长潜力？",
  "政策对AI产业的影响如何？"
]
```



## **模型选择策略**

### **配置位置**

**文件路径&#x20;**：\`backend/app/config/llm\_config.py\`

```python
class AgentsConfig:
    """Agent 模型配置"""
    architect: ModelConfig = field(default_factory=lambda: ModelConfig(
        model="qwen-max",
        temperature=0.3,
        max_tokens=16000
    ))
```

### **参数说明**

## **SSE 事件流**

ChiefArchitect 会发送以下 SSE 事件到前端：

### **7.1. research\_step（开始）**

```json
{
  "type": "research_step",
  "step_id": "step_planning_a3f2b1c5",
  "step_type": "planning",
  "title": "研究计划",
  "subtitle": "分析问题，制定大纲",
  "status": "running",
  "stats": {}
}
```



### **7.2. thought（思考过程）**

```json
{
  "type": "thought",
  "agent": "ChiefArchitect",
  "content": "正在分析研究问题，构建知识图谱和研究大纲..."
}
```



### **7.3. outline（大纲生成）**

```json
{
  "type": "outline",
  "understanding": {},
  "key_entities": ["人工智能", "深度学习", "计算机视觉"],
  "outline": [
    {
      "id": "sec_1",
      "title": "市场概况",
      "description": "描述市场规模、增速",
      ...
    }
  ],
  "research_questions": ["问题1", "问题2"]
}
```



### **7.4. research\_step（完成）**

```json
{
  "type": "research_step",
  "step_type": "planning",
  "title": "研究计划",
  "subtitle": "分析问题，制定大纲",
  "status": "completed",
  "stats": {
    "sections_count": 6,
    "questions_count": 3
  }
}
```



## **调试技巧**

### **8.1. 日志输出**

```python
self.logger.info(f"Raw LLM response length: {len(response)} (attempt {attempt + 1})")
self.logger.debug(f"Raw LLM response (first 1000 chars): {response[:1000]}")
self.logger.info(f"Successfully parsed outline with {len(result['outline'])} sections")
```



### **8.2. 诊断失败原因**

```python
if not result:
    self.logger.warning(f"Attempt {attempt + 1}: JSON parsing failed completely")
elif not result.get("outline"):
    self.logger.warning(f"Attempt {attempt + 1}: No 'outline' key in result. Keys: {list(result.keys())}")
elif len(result.get("outline", [])) < 3:
    self.logger.warning(f"Attempt {attempt + 1}: Outline too short: {len(result.get('outline', []))} sections")
```



### **8.3. 打印详细状态**

```python
self.logger.info(f"Parsed result keys: {list(result.keys())}")
outline = result.get("outline", [])
self.logger.info(f"Outline in result: {len(outline)} sections")
if not outline:
    self.logger.warning(f"No outline found! Full parsed result: {str(result)[:500]}")
```



## **假设驱动研究（Hypothesis-Driven Research）**

### **概念**

ChiefArchitect 会提出 3-5 个研究假设，例如：

* "中国AI市场规模将持续快速增长，CAGR超过30%"

* "llm是最大的细分领域"

### **后续验证**

<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">DeepScout 在搜索时会：</span>

1. 寻找支持或反驳假设的证据

2. 更新 `hypothesis.evidence_for` 或 `hypothesis.evidence_against`

3) 更新 `hypothesis.status` ：

   * `unverified` → `supported`

   * `unverified` → `refuted`

   * `unverified` → `partially_supported`

### **代码示例（scout.py 第896-902行）**

```python
# 更新假设证据
hypothesis_support = fact.get("hypothesis_support")
if hypothesis_support and fact.get("related_hypothesis"):
    h_id = fact["related_hypothesis"]
    for h in state.get("hypotheses", []):
        if h.get("id") == h_id:
            if hypothesis_support == "supports":
                h.setdefault("evidence_for", []).append(content[:100])
            elif hypothesis_support == "refutes":
                h.setdefault("evidence_against", []).append(content[:100])
```



## **总结**

ChiefArchitect 是整个研究流程的起点，其核心职责是：

1\. **问题解码&#x20;**：理解用户真实需求

2\. **大纲生成&#x20;**：创建可执行的研究计划（5-8个章节）

3\. **假设提出&#x20;**：提出需要验证的研究假设

4\. **容错设计&#x20;**：通过重试、格式转换、字段补全确保鲁棒性

**关键输出&#x20;**：

* `state["outline"]` ：动态大纲（后续章节逐一研究）

* `state["hypotheses"]` ：研究假设（后续验证）

* `state["research_questions"]` ：核心问题（引导搜索方向）

下一章节将介绍 DeepScout（深度侦探），它是信息收集的核心 Agent。

