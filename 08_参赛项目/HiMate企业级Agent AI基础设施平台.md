1. # Hi——Agent AI基础设施平台

   ## 项目文档

   **V1.0**

   **2026.09.01**

   **[团队名称]** **[参赛组别]**

   ## 目录

   1 项目概况 1
   1.1 背景和基础 1
   1.2 场景和价值 1
   1.3 所需支持 1
   2 项目规划 1
   2.1 整体目标 1
   2.2 技术创新点 1
   3 实施方案 1
   3.1 技术可行性分析 1
   3.2 技术细节 1
   3.3 计划和分工 1
   4 参考资料 1

   ## 记录更改历史

   | 序号 | 更改原因 | 版本 | 作者 | 更改日期   | 备注     |
   | :--- | :------- | :--- | :--- | :--------- | :------- |
   | 1    | 初始创建 | V1.0 | 团队 | 2026.09.01 | 初赛提交 |

   ## 1 项目概况

   ### 1.1 背景和基础

   **项目起因。** 2026年，全球AI Agent市场正经历从概念验证到规模化部署的关键跨越。IDC数据显示，中国企业活跃智能体数量将从2025年的近200万跃升至2026年预计的500万。然而，科研场景在AI Agent实际落地中仍面临三大核心瓶颈：***工具扩展困难**——大量异构后端服务（ERP、CRM、财务系统等）缺乏统一标准快速转化为Agent可调用工具；**热加载能力缺失**——主流开源框架（如LangChain 0.3、LlamaIndex 0.12等）在工具注册后须重启服务方可生效，无法满足企业高频迭代需求；**跨平台复用不足**——各业务系统重复建设Agent能力，缺乏统一治理与共享机制。*

   **已有工作基础。** Hi项目已完成了核心Agent运行时框架的开发与内部署，验证了三种能力注入方式在真实业务场景中的可行性。具体成果包括：

   - 完成了Agent运行时核心引擎的架构设计与代码实现；
   - 成功集成论文查询、实验数据分析等真实业务线的Skill对接；
   - 团队由计算机科学与技术、人工智能、软件工程等专业研究生组成，具备AI系统架构设计、后端开发、大模型应用等核心技术能力。

   **【图1 此处配图：科研场景AI Agent落地三大核心痛点示意图】**
   *建议用三块障碍图标分别标注“工具扩展困难（异构系统多、缺乏标准）”“热加载缺失（每次新增工具需重启服务）”“跨平台复用不足（各业务线重复建设）”，下方标注“现有开源框架难以满足企业级Agent工程化需求”。*

   ### 1.2 场景和价值

   **应用场景。** Hi定位为面向科研场景Agent AI基础设施平台，首期落地XX大学XX实验室，覆盖以下核心场景：

   | 场景           | 目标用户           | 用户价值主张                                                 |
   | :------------- | :----------------- | :----------------------------------------------------------- |
   | 科研提效       | 研究助理/学生/教师 | 基于业务数据构建专项业务能力，构建多种交互模式，建立新的人机协同办公模式 |
   | 桌面级智能助手 | 全体教师           | 对接学院内部系统/知识库，打造Lead Agent作为工作搭子，辅助日常办公 |

   已落地的具体业务包括：论文查询（面向教师/学生）、实验数据分析（研究助理/学生）。

   **社会价值与市场对比。** 当前市场主流Agent框架在工具集成方面存在明显局限。LangChain采用单一函数装饰器（@tool）方式，工具注入方式单一，且Agent在工具数量增加时性能显著下降，甚至出现“忘记调用工具”或“无法响应任务”的情况；AutoGPT同样仅支持Python函数注入，缺乏工具治理能力。Hi通过**三层差异化能力注入机制**（Skill/Custom Tool/MCP），使开发者可根据服务特征选择最优接入路径，同时提供热加载、结构化IO自动生成、统一权限管理等特性，有效填补了当前开源框架在科研场景下Agent基础设施层面的空白。

   **【图2 此处配图：Hi三大核心应用场景示意图】**
   *建议用三个场景卡片并列展示，每个卡片标注场景名称、目标用户和核心价值，卡片之间用箭头连接指向统一的“Hi平台”。*

   ### 1.3 所需支持

   项目实施过程中所需支持如下：

   - **算力支持**：需要GPU服务器资源用于大语言模型的推理部署（当前使用实验室内部GPU资源）；若采用API调用方式，则无需本地GPU资源；
   - **硬件支持**：标准Linux服务器（Ubuntu/CentOS）用于Agent运行时部署，当前已部署于实验室内部服务器；
   - **数据支持**：需要学校内部业务数据（科研论文资源、实验室平台、计算中心）的访问权限，均已通过学校内部审批流程获得授权；
   - **培训支持**：需要对实验室各个成员进行Skill开发规范的培训，使其掌握三种对接方式的使用方法。

   ## 2 项目规划

   ### 2.1 整体目标

   参赛期间本项目的整体目标如下：

   1. **核心目标**：完成Hi Agent基础设施平台V1.0版本的完整开发与部署，形成可对外展示的**可运行原型系统**；
   2. **业务验证**：在至少2条真实业务线上完成Agent能力的落地验证（已完成论文查询，实验数据分析）；
   3. **性能指标**：端到端Agent推理响应延迟控制在**秒级**以内，支持SSE流式交互；
   4. **成果产出**：形成完整的项目文档、系统架构设计文档、开发者接入指南，以及项目演示视频。

   ### 2.2 技术创新点

   通过系统性的技术调研与对比分析，Hi的核心技术创新点如下：

   **创新点一：标准化三层能力注入体系。** 现有开源框架（LangChain、AutoGPT等）在工具集成方面主要依赖单一方式，缺乏对不同粒度、不同形态后端服务的差异化适配能力。Hi提出三层体系（Skill/Custom Tool/MCP），使开发者可根据服务特征选择最优接入路径：

   | 维度     | Skill                      | Custom Tool              | MCP                    |
   | :------- | :------------------------- | :----------------------- | :--------------------- |
   | 开发量   | 1个SKILL.md + 1个CLI脚本   | 1个Python模块 + 改config | MCP Server + 配置连接  |
   | 热加载   | ✅ 放入skills/即可          | ❌ 需重启                 | ❌ 需重启               |
   | 参数格式 | 命令行字符串               | JSON Schema（自动生成）  | JSON Schema（MCP标准） |
   | 适用场景 | 已有独立后端、调用逻辑复杂 | 高频原子操作、结构化IO   | 跨平台复用             |

   **创新点二：热加载能力。** Skill方式支持**零重启热加载**，将Skill目录放入`skills/`下即可被系统自动发现并注册，解决了LangChain等框架新增工具须重启服务的工程痛点，大幅缩短了企业高频迭代场景下从能力开发到上线的时间周期。

   **创新点三：MCP协议集成。** MCP（Model Context Protocol）由Anthropic提出，已成为AI Agent与外部数据系统通信的事实标准。Google、IBM、Salesforce等企业均已宣布支持MCP协议。Hi率先在科研场景Agent平台中实现MCP协议的stdio和http双模式对接，为跨平台工具复用提供了标准化路径。

   **【图3 此处配图：三种能力注入方式对比雷达图/多维对比图】**
   *建议用雷达图或三维柱状图对比三个维度：开发量（低/中/高）、热加载支持（是/否）、结构化IO能力（强/中/弱），直观展示三种方式的差异化定位。*

   ## 3 实施方案

   ### 3.1 技术可行性分析

   **数据采集与行业知识获取。**

   - **科研论文数据**：来源于学校图书馆科学论文资源文档库，由图书馆提供标准化资源条目；
   - **实验数据**：来源于计算中心系统实验数据，通过标准接口获取；
   - **实验数据分析**：来源于实验室平台已有经验沉淀，通过RAG；
   - **行业知识**：团队通过调研LangChain、AutoGPT等开源框架的技术文档与社区实践，以及MCP协议规范，形成了对Agent工具调用领域技术现状的系统性认知。

   **算力与硬件支持。**

   - 当前已部署于实验室内部服务器，运行环境为Linux系统；
   - 大语言模型推理可选用开源模型本地部署或商用API调用两种方案，均具备技术可行性；
   - 部署方式简单（`cd bin && ./start.sh`），具备快速复制部署的能力。

   **技术风险与应对。**

   - **风险一**：大语言模型推理延迟可能影响用户体验。**应对**：采用流式SSE输出，边生成边返回，用户无需等待完整结果；
   - **风险二**：实验室平台接口可能存在兼容性问题。**应对**：三种对接方式（Skill/Custom Tool/MCP）提供了从简单到复杂的渐进式接入路径，可根据后端服务实际情况灵活选择；
   - **风险三**：多Agent协同场景下的消息通信复杂度。**应对**：已在后续规划中设计了基于MessageBus的消息通信架构，确保可扩展性。

   ### 3.2 技术细节

   #### 3.2.1 整体架构

   Hi系统采用分层架构设计，自顶向下分为四层：

   1. **应用层**：面向终端用户的Web交互界面，支持多端访问；
   2. **Agent运行时层**：核心Agent引擎，负责任务理解、工具调度、对话管理与推理执行；
   3. **能力注入层**：三种标准化对接方式（Skill/Custom Tool/MCP），连接异构后端服务；
   4. **基础设施层**：底层服务治理、配置管理、日志监控等支撑能力。

   系统组件架构涵盖Agent构建、运行、治理、共享四大模块，形成完整的Agent生命周期管理闭环。

   **【图4 此处配图：Hi系统分层架构图】**
   *建议绘制四层矩形堆叠图，从上到下依次为“应用层（前端交互）→ Agent运行时层（核心引擎）→ 能力注入层（Skill/Custom Tool/MCP三种方式并列）→ 基础设施层（治理/配置/监控/日志）”，每层用不同颜色区分。*

   **【图5 此处配图：Hi系统组件架构图】**
   *建议绘制各组件之间的依赖与数据流向关系图，核心组件包括：Agent运行时、工具注册中心（Toolkit）、配置管理、Skill发现模块、MCP连接器、前端网关等。*

   #### 3.2.2 核心技术：三层能力注入机制

   **（一）Skill方式**

   Skill方式的核心思路是通过CLI脚本做HTTP桥接，SKILL.md文档指导Agent如何调用该能力。

   目录结构：

   text

   ```
   skills/{service_name}/
     SKILL.md          # 元数据 + Agent行为指导
     scripts/
       {service_name}.py  # HTTP桥接脚本
       requirements.txt   # 脚本依赖（可选）
   ```

   

   SKILL.md示例：

   yaml

   ```
   name: your-service
   description: "xx服务集成。当用户提到'关键词A'、'关键词B'时触发此skill。"
   # CLI调用方式
   # bash python ./skills/your-service/scripts/your_service.py --query "用户问题"
   # Agent行为指导：将用户原始问题原样传入--query，stdout输出直接作为回复呈现
   ```

   

   **运行流程**：
   用户提问 → Agent匹配description关键词 → Agent读取SKILL.md → Agent执行脚本（HTTP调用后端）→ stdout输出 → Agent呈现结果

   **【图6 此处配图：Skill方式运行流程图】**
   *建议绘制自上而下的流程图，节点包括：用户提问 → Agent匹配description关键词 → Agent读取SKILL.md → Agent执行脚本（HTTP调用后端）→ stdout输出 → Agent呈现结果。每个节点用圆角矩形，箭头连接。*

   **（二）Custom Tool方式**

   Custom Tool方式的核心思路是编写异步Python函数，框架自动包装成Agent可调用的结构化工具。

   文件命名规则：`src/tools/{tool_name}.py`，函数名与文件名一致，docstring自动解析为JSON Schema参数定义。

   核心代码示例：

   python

   ```
   # src/tools/your_tool.py
   "你的工具描述（这行会成为工具的description）"
   import httpx
   from src.core.tool import ToolResponse
   from src.core.message import TextBlock
   
   async def your_tool(query: str, max_results: int = 5) -> ToolResponse:
       """查询XX服务获取结果。
       Args:
           query (str): 用户的问题。
           max_results (int): 最大返回条数。
       Returns:
           ToolResponse: 查询结果。
       """
       async with httpx.AsyncClient(timeout=30) as client:
           resp = await client.post(
               "http://your-service:5520/query",
               json={"query": query, "limit": max_results},
           )
       data = resp.json()
       return ToolResponse(content=[TextBlock(type="text", text=data.get("final_answer", "无结果"))])
   ```

   

   注册方式：在`config/agents/lead/config.yaml`的`custom`列表中添加工具名。

   **运行流程**：
   用户提问 → Agent根据工具description判断调用 → Agent生成tool_use调用 → 框架执行异步函数（HTTP调用后端）→ 返回ToolResponse → Agent整合回复

   **【图7 此处配图：Custom Tool方式运行流程图】**
   *建议绘制自上而下的流程图，节点包括：用户提问 → Agent根据工具description判断调用 → Agent生成tool_use调用 → 框架执行异步函数（HTTP调用后端）→ 返回ToolResponse → Agent整合回复。*

   **（三）MCP协议对接方式**

   MCP方式的核心思路是实现标准MCP Server，平台通过MCP协议连接并自动注册其工具。MCP由Anthropic提出，已成为AI Agent与外部数据系统通信的事实标准。

   MCP Server实现示例（Python FastMCP）：

   python

   ```
   from mcp.server.fastmcp import FastMCP
   mcp = FastMCP("your-service")
   
   @mcp.tool()
   async def query_policy(question: str) -> str:
       """查询政策信息。Args: question: 用户的问题"""
       import httpx
       async with httpx.AsyncClient() as client:
           resp = await client.post("http://your-backend:5520/query", json={"query": question})
       return resp.json().get("final_answer", "未找到")
   
   if __name__ == "__main__":
       mcp.run(transport="stdio")
   ```

   

   支持两种传输模式：

   - **stdio_mcp**：本地进程模式，适用于本地部署的MCP Server；
   - **http_mcp**：远端服务模式，适用于独立部署的MCP服务（支持stateless）。

   配置文件（`local_workspace/lead/.mcp`）示例：

   json

   ```
   [
     {"name": "your-service", "is_stateful": true,
      "mcp_config": {"type": "stdio_mcp", "command": "python", "args": ["your_mcp_server.py"]},
      "execution_timeout": 30.0}
   ]
   ```

   

   **运行流程**：
   系统启动 → 读取.mcp配置 → 连接MCP Server → 获取工具列表 → 注册到Toolkit → 用户提问 → Agent生成tool_use → 框架通过MCP协议调用 → 返回结果

   **【图8 此处配图：MCP方式运行流程图】**
   *建议绘制包含启动阶段和运行阶段两部分的流程图，启动阶段：系统启动→读取.mcp配置→连接MCP Server→获取工具列表→注册到Toolkit；运行阶段：用户提问→Agent生成tool_use→框架通过MCP协议调用→返回结果。*

   #### 3.2.3 部署与预期技术指标

   **部署方式**：

   bash

   ```
   cd bin
   ./start.sh
   ```

   

   **预期技术指标**：

   | 指标                | 目标值   | 说明                      |
   | :------------------ | :------- | :------------------------ |
   | Agent端到端响应延迟 | ≤5秒     | 从用户提问到完整回复返回  |
   | 流式首字延迟        | ≤1秒     | SSE流式首个字符输出时间   |
   | 流式交互支持        | ✅        | SSE（Server-Sent Events） |
   | 超时控制            | 30-120秒 | 可配置                    |
   | Skill热加载生效时间 | ≤3秒     | 放入skills/目录后自动发现 |
   | 并发Agent实例数     | ≥10      | 单节点支持                |

   **【图9 此处配图：已集成业务Skill运行效果截图（2张）】**
   *建议截取对话界面，分别展示：（1）论文查询的查询结果；（2）实验数据分析的结果呈现。*

   ### 3.3 计划和分工

   结合参赛时间点（2026年9月1日为作品提交截止日期），本项目的整体计划和团队分工如下：

   **整体计划：**

   | 阶段               | 时间          | 主要任务                            | 交付物                              |
   | :----------------- | :------------ | :---------------------------------- | :---------------------------------- |
   | 第一阶段（已完成） | 2026年5月-7月 | 需求分析、架构设计、核心框架开发    | 系统架构文档、核心代码库            |
   | 第二阶段（已完成） | 2026年7月-8月 | 三种能力注入方式实现、业务Skill集成 | 论文查询、实验数据分析等业务线Skill |
   | 第三阶段（已完成） | 2026年8月     | 系统部署上线、内部测试与调优        | 可运行系统                          |
   | 第四阶段（冲刺）   | 2026年8月下旬 | 文档整理、视频录制、材料提交        | 项目文档、参赛作品简介、项目视频    |

   **团队分工：**

   | 角色          | 人数 | 主要职责                                                     |
   | :------------ | :--- | :----------------------------------------------------------- |
   | 项目负责人    | 1人  | 整体架构设计、进度管理、文档统筹                             |
   | 全栈开发      | 1人  | Agent运行时开发、三种能力注入机制实现、Skill集成、Web交互界面开发、前后端联调 |
   | 算法/AI工程师 | 1人  | 大模型推理调优、Prompt工程、性能测试Web交互界面开发、前后端联调 |

   **【图11 此处配图：项目时间线/甘特图】**
   *建议绘制横向甘特图，展示四个阶段的时间跨度和主要里程碑。*

   ## 4 参考资料

   [1] IDC. 从200万到500万：智能体从概念走向规模化落地，企业级AI平台的“六层全栈架构”[R]. 2026.

   [2] 沙丘智库. 2026年企业级Agent智能体平台全景图与未来趋势[R]. 2026.

   [3] LangChain. AI agents aren't human-level yet because they're overwhelmed by tools[R]. VentureBeat, 2025.

   [4] LangChain的工具调用 vs 原生Skill API：性能差在哪儿？[EB/OL]. 阿里云开发者社区, 2026.

   [5] Anthropic. Model Context Protocol (MCP) Specification[S]. 2025.

   [6] Google Cloud. Announcing official MCP support for Google services[EB/OL]. 2025.

   [7] IBM. 混合集成技术创新加速企业级AI智能体落地[R]. 2025.

   [8] Salesforce. New MuleSoft Innovations for Secure, Scalable AI Agent Orchestration[R]. 2025.

   [9] Z-SPACE: A Multi-Agent Tool Orchestration Framework for Enterprise-Grade LLM Automation[P]. arXiv, 2025.

   [10] ScaleCall: Agentic Tool Calling at Scale for Fintech: Challenges, Methods, and Deployment Insights[P]. arXiv, 2025.

   [11] TheMCPCompany: Creating General-Purpose Agents with Task-Specific Tools[P]. arXiv, 2025.

   ## 📌 配图汇总清单

   | 图号 | 所在章节          | 图片内容                       | 优先级   | 建议来源                                       |
   | :--- | :---------------- | :----------------------------- | :------- | :--------------------------------------------- |
   | 图1  | 1.1 背景和基础    | 企业AI Agent落地三大痛点示意图 | ⭐⭐⭐ 必放 | 新建绘制（三块障碍图标）                       |
   | 图2  | 1.2 场景和价值    | Hi三大核心应用场景示意图       | ⭐⭐ 建议  | 新建绘制（三个场景卡片）                       |
   | 图3  | 2.2 技术创新点    | 三种能力注入方式对比雷达图     | ⭐⭐ 建议  | 新建绘制（雷达图/柱状图）                      |
   | 图4  | 3.2.1 整体架构    | Hi系统分层架构图               | ⭐⭐⭐ 必放 | 源自PDF第2页，截取复用                         |
   | 图5  | 3.2.1 整体架构    | Hi系统组件架构图               | ⭐⭐⭐ 必放 | 源自PDF第2页，截取复用                         |
   | 图6  | 3.2.2 Skill方式   | Skill方式运行流程图            | ⭐⭐⭐ 必放 | 新建绘制（[Draw.io/PPT](https://draw.io/PPT)） |
   | 图7  | 3.2.2 Custom Tool | Custom Tool方式运行流程图      | ⭐⭐⭐ 必放 | 新建绘制（[Draw.io/PPT](https://draw.io/PPT)） |
   | 图8  | 3.2.2 MCP方式     | MCP方式运行流程图              | ⭐⭐⭐ 必放 | 新建绘制（[Draw.io/PPT](https://draw.io/PPT)） |
   | 图9  | 3.2.3 部署与指标  | 已集成业务Skill运行截图（3张） | ⭐⭐⭐ 必放 | 源自PDF第11-12页，截取复用                     |
   | 图10 | 3.2.3 部署与指标  | 推理性能指标柱状图             | ⭐ 可选   | 新建绘制（如有实测数据）                       |
   | 图11 | 3.3 计划和分工    | 项目时间线/甘特图              | ⭐⭐ 建议  | 新建绘制（横向甘特图）                         |

   **提交文件命名**：
   `（团队名称）_XXX（项目名称）_项目文档.pdf`

   **说明**：请将文档中的`[团队名称]`、`[项目名称]`、`[参赛组别]`替换为实际信息，并根据官方附件2模板的页码要求调整页码标注。所有配图插入后请在正文中增加“如图X所示”的引用语句。