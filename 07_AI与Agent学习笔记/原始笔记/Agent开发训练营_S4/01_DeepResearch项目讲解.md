# DeepResearch项目讲解

# **行业信息助手 - 技术文档大纲总览**

**第一部分: 项目全景 (3个文档)**

### **1.1 项目定位与核心能力**

**核心内容&#x20;**:

* 项目定位: 多智能体AI深度研究系统

* 6个Agent分工协作

* 核心能力拆解

* 技术创新点: 假设驱动研究、信源追溯、实时SSE、检查点恢复

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/graph.py`

* `/backend/app/service/deep_research_v2/state.py`

* `/backend/app/service/deep_research_v2/agents/*.py`

### **1.2 系统架构全景图解读**

**核心内容&#x20;**:

* 四层架构: 前端/API网关/业务逻辑/数据存储

* 数据流详解: 深度研究/知识库查询/Text2SQL

* 核心模块: 多智能体引擎、状态管理、SSE推送

* 数据库设计: PostgreSQL + Milvus + Redis

**关键代码&#x20;**:

* `/backend/app/app_main.py`

* `/backend/app/router/*.py`

* `/frontend/src/pages/chat/`

### **1.3 技术选型与设计决策**

**核心内容&#x20;**:

* LangGraph vs CrewAI/AutoGen: 为什么选LangGraph

* Deepseek + Qwen混合: 性能与成本平衡

* Milvus vs Pinecone: 开源免费,性能优秀

* 多智能体设计原因

* 检查点、SSE、代码沙箱设计决策

**关键代码&#x20;**:

* `/backend/app/config/llm_config.py`

* `/backend/app/service/milvus_service.py`

***

## **第二部分: 多智能体架构 (8个文档)**

### **2.1 LangGraph状态机基础**

**核心内容&#x20;**:

* StateGraph API讲解

* 节点 (Node) 概念

* 边 (Edge) 和条件路由

* ResearchState状态定义

* 状态流转机制

* 实战: 构建简单状态机

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/graph.py` (行号: 67-233)

* `/backend/app/service/deep_research_v2/state.py` (行号: 105-203)

**学习要点&#x20;**:

* StateGraph vs Chains

* 条件路由实现

* 状态持久化

### **2.2.1 Agent - ChiefArchitect**

**核心内容&#x20;**:

* 职责: 问题分析、大纲规划、假设生成

* 输入: user query

* 输出: outline, research\_questions, hypotheses, key\_entities

* Prompt设计思路

* 核心逻辑: `_initial_planning` , `_check_revision`

* 假设驱动研究机制

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/agents/architect.py`

**Prompt位置&#x20;**:

* PLANNING\_PROMPT (行号: 29-63)

* REVISION\_PROMPT (行号: 65-96)

**学习要点&#x20;**:

* 如何生成高质量大纲

* 假设驱动研究的优势

* 动态调整大纲的时机

### **2.2.2 Agent - DeepScout**

**核心内容&#x20;**:

* 职责: 全网搜索、信息收集、信源评级

* 双模式: 网络搜索 (Bocha API) + 本地知识库 (Milvus)

* 递归搜索: 发现线索自动深挖

* 信源追溯: 找到原始数据来源

* 事实去重机制

* 假设验证: 标注支持/反驳假设的证据

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/agents/scout.py`

**Prompt位置&#x20;**:

* SEARCH\_ANALYSIS\_PROMPT (行号: 59-124)

* DEEP\_READ\_PROMPT (行号: 126-164)

**学习要点&#x20;**:

* 信源评级标准 (0-1)

* 递归搜索深度控制

* 本地向量检索集成

### **2.2.3 Agent - DataAnalyst**

**核心内容&#x20;**:

* 职责: 数据提取、知识图谱构建、ECharts配置生成

* 从文本提取结构化数据点

* 时间序列识别

* 知识图谱: 节点(实体) + 边(关系)

* ECharts配置生成 (LLM直接生成JSON)

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/agents/data_analyst.py`

**Prompt位置&#x20;**:

* DATA\_EXTRACTION\_PROMPT (行号: 31-98)

* KNOWLEDGE\_GRAPH\_PROMPT (行号: 100-142)

* CHART\_GENERATION\_PROMPT (行号: 143-242)

**学习要点&#x20;**:

* 如何从非结构化文本提取数据

* 知识图谱的节点类型定义

* ECharts配置生成技巧

### **2.2.4 Agent - CodeWizard**

**核心内容&#x20;**:

* 职责: Python代码生成、图表绘制

* 代码沙箱: 白名单模块、禁止危险操作

* 自愈能力: 执行失败自动修复 (最多3次)

* 代码清理: 处理LLM生成的格式问题

* 图表捕获: plt.savefig → base64编码

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/agents/wizard.py`

**Prompt位置&#x20;**:

* ANALYSIS\_PROMPT (行号: 36-124)

* CHART\_PROMPT (行号: 126-172)

* CODE\_FIX\_PROMPT (行号: 173-215)

**学习要点&#x20;**:

* 代码沙箱安全机制

* 自愈流程设计

* 代码清理技巧

### **2.2.5 Agent - LeadWriter**

**核心内容&#x20;**:

* 职责: 报告撰写、内容整合、Markdown排版

* 逐章节撰写流程

* 报告整合: 添加摘要、结论、参考文献

* Markdown引用格式规范

* 流式输出章节内容

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/agents/writer.py`

**Prompt位置&#x20;**:

* SECTION\_WRITING\_PROMPT (行号: 30-80)

* SYNTHESIS\_PROMPT (行号: 82-179)

* REVISION\_PROMPT (行号: 181-209)

**学习要点&#x20;**:

* 专业研究报告风格

* 引用格式规范

* 流式显示实现

### **2.2.6 Agent - CriticMaster**

**核心内容&#x20;**:

* 职责: 对抗式审核、质量把控

* 审核维度: 信息完整性、来源可靠性、逻辑一致性、偏见识别

* 问题分类: missing\_source, logic\_error, hallucination, bias, outdated

* 严重级别: critical, major, minor

* 智能路由: 决定补充搜索 vs 文字修订

* 质量评分: 1-10分制

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/agents/critic.py`

**Prompt位置&#x20;**:

* REVIEW\_PROMPT (行号: 30-107)

* FINAL\_CHECK\_PROMPT (行号: 109-137)

**学习要点&#x20;**:

* 对抗式审核思维

* 路由决策逻辑

* 质量评分标准

### **2.3 完整工作流演示**

**核心内容&#x20;**:

* 从用户输入到报告输出的完整流程

* 各Agent协作过程时间线

* 状态变化追踪

* SSE事件推送顺序

* 实际案例分析: "人工智能行业分析"

**关键代码&#x20;**:

* `/backend/app/service/deep_research_v2/graph.py` (完整流程)

**学习要点&#x20;**:

* 理解完整工作流

* Agent间如何传递状态

* 如何调试问题

### 2.4 记忆层实现

***

## **第三部分: 工程化实践 (8个文档)**

### **3.1 Docker容器化部署**

**核心内容&#x20;**:

* docker-compose.yml解析

* 各服务容器化策略

* 数据卷挂载

* 网络配置

* 环境变量管理

* 生产部署建议

**关键代码&#x20;**:

* `/docker-compose.yml`

* `/backend/Dockerfile`

* `/frontend/Dockerfile`

### **3.2 PostgreSQL数据库设计**

**核心内容&#x20;**:

* ER图和表结构设计

* 用户、会话、知识库、文档、新闻等表

* JSONB字段使用 (检查点存储)

* 索引优化策略

* 查询性能优化

* 备份恢复方案

**关键代码&#x20;**:

* `/backend/app/models/*.py`

* `/backend/app/core/database.py`

### **3.3 Milvus向量数据库集成**

**核心内容&#x20;**:

* Collection设计: knowledge\_base

* 索引类型: IVF\_FLAT vs HNSW

* 向量化流程: 文档解析 → 分块 → embedding → 插入

* 混合检索: 向量相似度 + 标量过滤

* 性能调优: nlist参数调整

**关键代码&#x20;**:

* `/backend/app/service/milvus_service.py`

* `/backend/app/service/embedding_service.py`

### **3.4 Redis应用场景**

**核心内容&#x20;**:

* 检查点存储 (24小时TTL)

* 会话状态管理

* 搜索结果缓存 (15分钟TTL)

* 分布式锁

* 取消标志位

**关键代码&#x20;**:

* `/backend/app/core/redis_client.py`

* `/backend/app/service/checkpoint_service.py`

### **3.5 Text2SQL服务实现**

**核心内容&#x20;**:

* Schema信息提取

* SQL生成Prompt设计

* SQL安全验证 (禁止DELETE/DROP)

* 结果格式化

* 错误处理和重试

**关键代码&#x20;**:

* `/backend/app/service/text2sql_service.py`

* `/backend/app/service/database_explorer.py`

### **3.6 定时任务调度系统**

**核心内容&#x20;**:

* 新闻采集定时任务

* APScheduler配置

* 任务并发控制

* 失败重试机制

* 日志记录

**关键代码&#x20;**:

* `/backend/app/service/scheduler_service.py`

* `/backend/app/service/news_collection_service.py`

### **3.7 检查点与状态恢复**

**核心内容&#x20;**:

* 检查点设计原则

* 保存时机: 每阶段完成后

* Redis + PostgreSQL双重存储

* 恢复流程

* UI状态同步

* 状态校验

**关键代码&#x20;**:

* `/backend/app/service/checkpoint_service.py`

* `/backend/app/service/deep_research_v2/graph.py` (行号: 146-196, 527-554)

### **3.8 SSE流式输出实现**

***

## **第四部分: 安全可靠性 (5个文档)**

### **4.1 Retry重试机制**

**核心内容&#x20;**:

* LLM调用重试策略 (3次)

* 搜索API重试

* 指数退避算法

* 最大重试次数配置

* 幂等性保证

**实现位置&#x20;**:

* BaseAgent.call\_llm (带重试)

* DeepScout.\_execute\_search

### **4.2 CircuitBreaker熔断器**

**核心内容&#x20;**:

* 熔断器模式原理

* 外部服务熔断 (LLM API, 搜索API)

* 三种状态: Closed, Open, Half-Open

* 降级策略

* 健康检查

**实现建议&#x20;**:

* 使用pybreaker库

* 配置失败率阈值

### **4.3 Fallback降级策略**

**核心内容&#x20;**:

* LLM调用降级: Deepseek失败 → Qwen

* 搜索失败降级: 使用缓存结果

* 图表生成失败降级: 跳过图表,继续流程

* 数据库查询降级: 使用备库

### **4.4 DLQ死信队列**

**核心内容&#x20;**:

* 失败任务处理

* 消息队列设计 (RabbitMQ/Kafka)

* 重试队列

* 人工介入机制

* 告警通知

### **4.5 AgentSFT微调数据**

**核心内容&#x20;**:

* SFT (Supervised Fine-Tuning) 数据收集策略

* 高质量样本标注: 问题-大纲-报告

* 微调流程: 数据清洗 → 格式转换 → 训练

* 效果评估: BLEU/ROUGE/人工评分

* 迭代优化

***

## **第五部分: 代码走读 (3个文档)**

### **5.1 核心文件导读**

**核心内容&#x20;**:

* 项目目录结构解析

* 每个文件的职责说明

* 模块依赖关系图

* 入口文件讲解: app\_main.py

* 重要配置文件: llm\_config.py, docker-compose.yml

**导读文件&#x20;**:

```plaintext
/backend/app/
├── app_main.py (FastAPI入口)
├── router/ (路由层)
│   ├── research_router.py (深度研究)
│   ├── knowledge_router.py (知识库)
│   └── ...
├── service/ (业务逻辑层)
│   ├── deep_research_v2/ (多智能体引擎)
│   │   ├── graph.py (状态机)
│   │   ├── state.py (状态定义)
│   │   └── agents/ (6个Agent)
│   ├── milvus_service.py (向量检索)
│   └── ...
├── models/ (ORM模型)
└── core/ (核心组件)
```

### **5.2 关键设计模式**

**核心内容&#x20;**:

* 策略模式: Agent切换

* 工厂模式: create\_research\_graph()

* 观察者模式: SSE事件推送

* 单例模式: 配置管理 (get\_config)

* 责任链模式: Agent协作流程

* 状态模式: ResearchPhase状态机

**代码示例&#x20;**:

```python
# 工厂模式
def create_research_graph(...) -> DeepResearchGraph:
    return DeepResearchGraph(...)

# 单例模式
_config_instance = None
def get_config() -> LLMConfig:
    global _config_instance
    if _config_instance is None:
        _config_instance = LLMConfig()
    return _config_instance

# 策略模式
class BaseAgent(ABC):
    @abstractmethod
    async def process(self, state) -> state:
        pass
```

### **5.3 性能优化技巧**

**核心内容&#x20;**:

* 异步并发优化: asyncio.gather

* 缓存策略: Redis缓存搜索结果

* 数据库查询优化: 索引、分页、连接池

* LLM调用优化: Prompt压缩、流式输出

* 前端优化: 虚拟滚动、懒加载

**代码示例&#x20;**:

```python
# 并发优化
results = await asyncio.gather(
    llm_service.call(prompt1),
    search_service.search(query),
    db_service.query(sql)
)

# 缓存优化
cache_key = hashlib.md5(query.encode()).hexdigest()
if cache_key in cache:
    return cache[cache_key]
```

