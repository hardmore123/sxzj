# 十五、Agent核心公式与原理（下）

为什么 Agent 会死循环？为什么它会拿着几十万的 Token 账单烧钱？

答案在于，我们之前对 Agent 的理解可能还停留在“大模型+几行代码”的堆砌。实际上，**要让 Agent 不失控，我们必须把它当成一套严谨的“软件系统”来设计。**

这就引出了今天课程的第一个核心公式：**`Agent = LLM（大脑）+ 工具（手脚）+ 循环（神经系统）`**。
在工业界的视角里，这个公式进一步具象化为**四大组件**：**感知模块、决策模块、执行模块、记忆模块**。

而这四大组件是怎么协同工作，形成连贯的**决策轨迹**的呢？在学术界和工业界，我们将这个运作过程细化为五个严谨的步骤：**感知 -> 规划 -> 决策 -> 执行 -> 再感知**。接下来，我们把这五步掰开揉碎，深入它的底层原理。

------

### 一、第一步：感知（Perception）—— 接收并理解“当前状态” ![感知](C:\Users\likunyuan\Desktop\新建文件夹\感知.png)

**(大模型接入用户输入、环境、记忆的图示)**

什么是“感知”？很多同学觉得很简单，不就是“用户输入一句话”吗？
**不对。在 Agent 的底层逻辑里，感知绝不是单纯的文本接收，而是把所有“环境信息”转化为大模型能理解的“状态空间（State Space）”。**

一个真正成熟的 Agent，它的感知模块包含三个维度的信息：

1. **指令层**：用户当前的提问（比如“帮我订票”）。
2. **环境层**：宿主程序帮它获取的附加信息（比如系统时间、用户当前的 GPS 坐标）。
3. **记忆层**：**这是被很多人忽略的核心。** 感知不仅是“听现在的话”，还要“回忆过去的事”。
   - 它需要从**工作记忆**中读取当前任务进行到哪里了（比如“上一轮已经查到了航班列表，现在该做预算了”）；
   - 它还需要从**长期记忆**（向量数据库）中提取用户的长期偏好（比如“用户过去总是喜欢靠窗的座位”）。

*【原理深挖】：* 在计算机科学的 MDP（马尔可夫决策过程）理论中，“感知”就是把真实世界的这些多维信息**向量化**，构造成一个丰富的“初始状态”，喂给决策模块，作为它做出第一次决策的**起点**。

------

### 第二步：规划（Planning）—— 大脑的“思维链”拆解 

![规划](C:\Users\likunyuan\Desktop\新建文件夹\规划.png)

**(PPT展示：大脑思考分解任务的图示)**![规划](C:\Users\likunyuan\Desktop\规划.png)

有了当前的状态，LLM 这个“**决策模块**”就会进入“规划”阶段。这就是我们常说的任务分解。

这也是大模型和传统自动化脚本最大的区别。传统脚本的规划是写死的：“先做A，再做B，再做C”。如果中间出错了，程序崩了，它绝对不会绕道。
而 Agent 的规划是**基于当前状态的动态生成**。比如，用户的指令是：“帮我订下周三去上海的票，预算1500”。大模型的规划可能是：

- *动态思考*：“哦，我需要先推算下周三的日期 -> 查 1500 元内的机票 -> 查出机票后，万一没有符合价格的，我得先思考备选方案（比如高铁）。”

**实战避坑提醒（关键）**：在这个阶段，有一个巨大的工程陷阱——**规划粒度**。

- *想一步干一步*：**太慢，频繁调用大模型极其昂贵**，而且没法做全局统筹。
- *全部想完再干*：**不现实，外部世界是变化的**。如果你让大模型规划了 A-B-C-D-E 五步，结果 A 步执行完发现外部天气突变（比如航班取消），后面 B 到 E 的规划全成了废纸，又得推倒重来。
- *正确的工程解法*：在代码层面，通常采用**“多步微规划”**策略。决策模块只规划接下来最关键的 **1 到 2 个动作**，执行完拿到结果后，立刻进行“再感知”，基于新状态重新规划。这就是为什么 Agent 看起来“随机应变”。

------

### 第三步：决策（Decision）—— 从“思路”变成“指令”

![决策](C:\Users\likunyuan\Desktop\新建文件夹\决策.png)

(大模型输出 JSON 格式的工具调用指令的截图/模拟)

有了规划之后，决策模块就要把自己的“想法”翻译成机器能懂的**“决策”**。大家注意，决策**不是**自然语言（不是说“我先查个时间吧”），它必须是**绝对结构化的机器指令**。

打个比方：大模型说“我决定调用 `get_current_date` 这个工具，参数不需要”。
它在后台实际生成的是这样一个 **JSON 格式的调用指令**（对应 Function Calling 原理）：

json

```
{
  "tool_calls": [
    {
      "function": { 
        "name": "get_current_date", 
        "arguments": "{}" 
      }
    }
  ]
}
```



*(授课建议：这里可以穿插一个小科普：大模型在底层训练时，被赋予了特殊的 `<function_call>` Token。一旦它决定“我该干活了”，它的输出概率分布就会瞬间切换，从“生成字词”变成“生成这种结构化的函数调用指令”。)*

![工作图](C:\Users\likunyuan\Desktop\新建文件夹\工作图.png)

**【决策的高阶模式——并行调用】**
在现实中，稍微聪明一点的决策模块会在此阶段做出**“并行决策”**。比如，查日历和查机票预算这两个动作互不依赖，模型可以一次性输出两个 JSON 指令，宿主代码并发执行，这就极大地节省了 Token 消耗和循环等待的时间。

------

### 四、第四步：执行（Execution）—— 大脑不动手，动手的是宿主

**(PPT展示：宿主代码运行工具的图示，明确标出“大模型”和“宿主代码”的界限)

![执行](C:\Users\likunyuan\Desktop\新建文件夹\执行.png)

这一步最为关键：**大模型始终只是“大脑”，不是“手脚”。**
请大家在脑子里画一条清晰的边界线：**模型绝对不会去直接调用 API，它只负责下达 JSON “口谕”。真正去调用外部天气 API 或者数据库的，是你的宿主代码（Python/Java/C++ 程序）。**

**全链路执行过程如下：**

1. 宿主代码收到大模型发来的 JSON 指令。
2. 宿主代码解析 JSON 里的 `name` 字段，在本地找到对应的 Python 函数。
3. **绝对安全边界（避坑核心）**：在正式执行前，宿主代码一定要执行“参数校验”和“权限校验”。比如大模型要调用 `send_email`，宿主必须先校验：当前用户是否有发邮件的权限？目标邮箱是否在白名单内？**绝不能盲目信任大模型的指令。**
4. 宿主代码执行函数（发出 HTTP 请求）。
5. 拿到 API 返回的原始数据。

------

### 五、第五步：再感知（Re-Perception）—— 状态更新，注入下一轮循环 

**(PPT展示：工具结果回到大模型的环形图示，并在这个环节中标注“记忆更新”)**![再感知](C:\Users\likunyuan\Desktop\新建文件夹\再感知.png)

执行拿到结果后就结束了吗？没有！还差整个 Agent Loop **最灵魂的一步**——**再感知**。

宿主程序拿到工具的结果后，**绝不可以**直接把那堆原始的 JSON 数据扔给大模型完事。它必须做两件事：

1. **数据清洗与格式化**：把机器读的 JSON 翻译成大模型容易理解的自然语言结构。比如：工具返回“`2026-08-05`”，宿主包装成“`工具结果查询：当前日期为 2026-08-05，对应的下周三日期是 2026-08-12。`”。
2. **状态空间更新（记忆注入）**：将这个清洗后的结果，**强制塞回大模型的上下文，并立即存入它的短期记忆**。大模型看到这个“观察（Observation）”后，会恍然大悟：“哦！现在下周三的日期确定了！”

**重点深挖**：这种“再感知”带来的结果是，**大模型的“状态空间”发生了质变**。它刚才还是个“不知道今天日期”的盲人，现在吸收了新状态，立刻会进入新一轮的规划。同时，如果这个操作结果很重要（比如预订成功，或者用户拒绝了某个选项），Agent 会调用记忆模块的写入接口，将信息同步写进**长期记忆（向量数据库）**，为未来的跨任务交互做准备。

------

### 六、本部分总结与过渡 

**(PPT展示：把五步闭环和四大组件合并成一个宏观的循环架构图)**![5+4](C:\Users\likunyuan\Desktop\新建文件夹\5+4.png)我们把刚才讲的逻辑串起来。
整个流程就是：**“感知环境（输入） -> 大脑规划 -> 输出决策指令 -> 宿主执行 -> 再感知反馈结果”**。这就形成了现代 AI Agent 的**知行合一**。

回到开头的那个经典公式：

- **感知、规划、决策**，由 LLM 大脑主导；
- **执行**，由宿主代码和外部工具负责；
- **把结果重新注入上下文的再感知**，就是维系整个生命周期的“神经反射（循环）”。

虽然听上去天衣无缝，但这套系统在真实的代码落地中，**就像一台没有刹车的跑车**。

- 如果工具 API 突然报错，大模型可能会反复尝试，陷入**工具死循环**；
- 如果大模型觉得“我需要再查一次对比”，它可能会无限自我博弈，耗尽天价的 Token 成本。

**那么，我们究竟该如何给这台跑车装上智能的“刹车”和“限速器”？当任务达成、或者遇到无法挽回的失败时，Agent 到底是如何判定自己“该停下来”并优雅退出的？*

### 第三部分讲稿：Agent Loop 微观解剖与优雅终止条件

前面我们讲了 Agent 闭环的五步原理，听起来完美。但在代码里，这等于给了 Agent 一辆跑车，却没有刹车和限速器。

要解决这个问题，必须深入 Agent Loop 的微观世界，看它到底怎么“边走边看”，以及我们怎么给它装上“三层刹车”。

![三层刹车](C:\Users\likunyuan\Desktop\新建文件夹\三层刹车.png)

------

#### 一、微观推演：ReAct 模式下的“订机票”实战

Agent 不是死板的工作流，它的决策是动态的。学术界管这叫 **ReAct（推理+行动）**。我们跟着 Agent 的“思维”走一遍：

**【设定】** 当前时间 2026-08-02，用户指令：“订一张下周三去上海、1500元内、早8点出发的机票，偏好南航。”

- **第1轮**：Thought（思考）→“下周三不知道几号，先查今天日期。” Action（行动）→ 调用 `get_current_date()`。 Observation（观察）→ 返回 `2026-08-02`，推算目标日为 8月5日。
- **第2轮**：Thought → “搜南航 8月5日早班机。” Action → `search_flights(南航, 8.5)`。 Observation → 最便宜早班机 **1680元**，超预算。
- **第3轮（关键转折）**：Thought → “超预算了。如果非要南航就得加钱，或者我放宽航司限制试试？” Action → 调用 `search_flights(不限航司, 1500元内)`。 Observation → 搜到 **东航 8:15 出发，1420元**，完美符合。
- **第4轮**：Thought → “找到了。但不能直接下单，先请求确认。” Action → 生成预订摘要，请求用户确认。 Observation → 用户回复：“确认下单。”
- **第5轮**：Thought → “任务达成。” Action → 调用 `book_flight()`，输出 Final Answer，**立即停止循环**。

**（要点）**：注意第3轮，传统脚本遇到“南航超预算”可能直接报错，但 Agent 具备**动态调整策略**的能力——这既是它的优势，也是它容易“跑飞”的根源。

------

#### 二、优雅终止：给跑车装上的“三层刹车”

大模型是概率机器，追求“最优解”。为了找一张 1400 元的票，它可能循环查 50 次。工程上必须设计**三层刹车机制**：

**刹车 1：硬限制（Hard Limit）—— 防火墙**
在宿主代码层面强制掐断。

- `max_iterations = 10`（跑满 10 轮强制抛异常）。
- 设定最大 Token 消耗，超限熔断。

> **这是防止账户被烧穿的底线。**

**刹车 2：软信号（Soft Signal）—— Agent 主动喊停**
让 LLM 输出特定标志位，宿主代码监听并截断。

- 在 System Prompt 中硬性规定：“任务完成时，必须输出 `Final Answer: [结果]`，并**立刻停止**。”
- 宿主代码检测到 `Final Answer` 关键字，立即 `break` 出循环。

> **避坑**：别写“请停止”，模型可能理解成“闭嘴”。要命令它输出特定标记，由外部代码执行真正的停止。

**刹车 3：工具状态码（External State）—— 强制关门**
业务层面已终结，外部代码强行截断，不让模型再做多余动作。

- 例如，执行“退款”或“下单”成功后，工具返回 `{"status": "terminal"}`。
- 宿主代码检测到该状态码，直接设置 `loop_active = False`，哪怕模型还想发个“再见”邮件，循环也已被强制终结。

**伪代码演示：**

python

```
max_steps = 10
while steps < max_steps:          # 刹车1
    response = llm.invoke(...)
    if "Final Answer" in response: # 刹车2
        break
    if tool_result.get("status") == "terminal": # 刹车3
        break
    steps += 1
```



------

#### 三、灾难警示：没有刹车的代价

想象一个真实故障：
用户问：“查一下老王欠我多少钱？” Agent 调用查账工具，数据库返回 `ERROR: 连接超时`。

- **没有硬限制**：Agent 不会停，反复重试。
- **没有状态码**：它把报错当成“需要换个参数再试”。

结果：**死循环跑了 50 轮，API 账单凭空多扣了 100 美金。**

Agent Loop 本身不是核心，**绑定了“优雅终止条件”的 Agent Loop 才是工程核心。** 你不仅要教大模型怎么干活，更要在代码里把它捆好，限制它的疯狂。这三层刹车，是生产环境上线的铁律。

## 第二章：把 Agent Loop “拆开”——不用框架怎么实现？

### 2.1 原始初代机：最简单的死循环

> 我们先忘掉 LangChain，忘掉 Graph。只用 `while True` 和 `openai.ChatCompletion`。

python

```
import openai

messages = [{"role": "user", "content": "帮我查一下青岛今天的天气"}]

while True:
    response = openai.ChatCompletion.create(model="gpt-4", messages=messages)
    # 假设返回的文本里包含 [ACTION: get_weather]
    if "[ACTION:" in response:
        # 粗暴解析（极其脆弱）
        tool_name = extract_tool(response) 
        result = call_tool(tool_name)
        messages.append({"role": "assistant", "content": result})
    else:
        print(response)
        break
```



### 2.2 加 State（状态管理）

> 刚才的代码没有记忆，没有状态。真实场景下，Agent 需要知道“我查了天气，下一步该规划路线了”。

python

```
class AgentState:
    def __init__(self):
        self.messages = []
        self.current_step = 0
        self.tool_results = {}

# 每一次循环，State 都在变化
state.tool_results["weather"] = "25度，晴"
```

State 就是 Agent 的“瞬时记忆”，它记录了**已经做了什么**。

### 2.3 加 Router（路由判断）

> LLM 返回的结果不是最终答案时，我们需要一个 Router（路由）来决定下一步。

python

```
def router(response):
    if response.get("tool_calls"):
        return "TOOL"
    elif response.get("intent") == "ask_user":
        return "HUMAN"
    else:
        return "FINAL"
```

![路由判断](C:\Users\likunyuan\Desktop\新建文件夹\路由判断.png)

**用流程图解释**：LLM 生成判断 → Router 解析 JSON → 分发给 Tool 或 Final。

### 2.4 加错误处理与 Retry（重试）

> 现场演示：假装天气 API 挂了。如果 Agent 直接崩溃，太傻了。

python

```
max_retry = 3
for attempt in range(max_retry):
    try:
        result = execute_tool()
        break
    except Exception as e:
        # 把错误信息追加到 State 里，让 LLM 自己看报错再重试
        state.messages.append({"role": "system", "content": f"报错了：{e}，请重试"})
```



### 2.5 加最大循环次数（无限循环防护）

> **这是工程红线！** 如果 LLM 抽风，它会无限调用“查天气”直到把你的 API 额度耗尽。

python

```
MAX_STEPS = 10
while state.current_step < MAX_STEPS and not finished:
    # ... Agent 逻辑
if state.current_step >= MAX_STEPS:
    return "抱歉，处理超时了。"
```



### 2.6 加 HITL（人工审批 Human-in-the-loop）

> 这是 Agent 上生产环境的**必选项**。

python

```
if tool.requires_permission:  # 比如：发送邮件、扣费
    human_input = input(f"Agent 准备执行：{tool.name}，是否允许？(y/n)")
    if human_input != 'y':
        break
```



**此时总结代码行数**：我们从一个 10 行的简单对话，加到现在已经写了 **150+ 行 Python 代码**，且充满了 `if/else` 和 `try/except`。

![进化](C:\Users\likunyuan\Desktop\新建文件夹\进化.png)

# 第三章：构建 Agent，人到底做什么？——LLM 与工程系统的职责边界

------

# 一、一个误区

很多人第一次接触 Agent，会有一个误解：

> “现在有大模型了，是不是只需要告诉它目标，它自己就能完成任务？”

例如：

```
帮我做一个市场调研报告。
```

然后期待：

```
LLM
 ↓
自动搜索
 ↓
自动分析
 ↓
自动写报告
 ↓
自动发送
```

但是现实中的 Agent 并不是这样。

一个真正可用的 Agent，需要大量人为设计：

```
目标是什么？
有哪些能力？
可以访问什么数据？
可以调用什么工具？
什么时候停止？
什么事情必须人工确认？
```

所以：

> **Agent 的智能来自 LLM，但 Agent 的可靠性来自人的工程设计。**

------

# 二、Agent 系统中的角色分工

先建立一个总体模型：

```
             人
             |
             |
      定义规则和边界
             |
             ↓

    ┌─────────────────┐
    │      Agent      │
    │                 │
    │       LLM       │
    │                 │
    └─────────────────┘
             |
             |
      动态推理和决策
             |
             ↓

    Tools / Memory / State / Environment
```

这里有一个核心思想：

> LLM 不是整个 Agent。

LLM 只是 Agent 中负责：

```
理解
推理
规划
判断
生成决策
```

的部分。

------

# 三、一个核心原则

## LLM 负责什么？

总结：

> **处理不确定性。**

例如：

用户说：

> “帮我安排一个适合商务交流的上海三日行程。”

这里很多东西是不确定的：

- 什么叫商务友好？
- 用户喜欢什么？
- 哪些地点合适？
- 如何排序？

这些适合交给 LLM。

------

## 人负责什么？

总结：

> **定义确定性。**

例如：

- 哪些工具可以调用？
- 工具接口是什么？
- 哪些行为禁止？
- 哪些操作需要审批？
- State 保存什么？

这些必须由工程人员决定。

------

# 四、Agent 各组件中，人和 LLM 的职责

## 总表

| Agent组件  | LLM负责          | 人负责         |
| ---------- | ---------------- | -------------- |
| Goal       | 理解用户目标     | 定义任务边界   |
| Planning   | 动态生成计划     | 设计规划原则   |
| Tool       | 选择调用工具     | 开发工具能力   |
| State      | 使用当前状态     | 设计状态结构   |
| Memory     | 利用历史信息     | 设计记忆机制   |
| Router     | 根据情况判断路径 | 定义流程逻辑   |
| Execution  | 生成调用请求     | 实际执行系统   |
| Permission | 提出操作需求     | 控制权限规则   |
| Loop       | 根据结果继续思考 | 设置循环机制   |
| Error      | 判断异常情况     | 设计恢复策略   |
| Human      | 请求人工介入     | 高风险决策审批 |

![3.1](C:\Users\likunyuan\Desktop\新建文件夹\3.1.png)

# 五、逐个展开讲

------

# 1. Goal（目标）

## LLM负责：

理解用户真正想要什么。

例如：

用户：

> “帮我准备一次会议。”

LLM理解：

可能包含：

```
准备资料
邀请人员
安排时间
生成议程
```

------

## 人负责：

定义 Agent 的任务范围。

例如：

会议助手：

允许：

```
查询日历
发送邀请
生成议程
```

不允许：

```
自动取消会议
删除文件
修改合同
```

所以：

> 用户目标由 LLM 理解，但是任务边界由人定义。

------

# 2. Planning（规划）

## LLM负责：

动态规划。

例如：

任务：

```
完成市场调研
```

LLM可能规划：

```
搜索资料
 ↓
整理数据
 ↓
分析竞争对手
 ↓
生成报告
```

------

## 人负责：

设计规划规则。

例如：

规定：

```
必须先验证数据来源

不能直接引用未知网站

金额超过10000必须确认
```

------

# 3. Tool（工具）

这是 Agent 最核心的扩展能力。

## LLM负责：

决定：

```
什么时候调用工具？
调用哪个工具？
参数是什么？
```

例如：

```
我要查询天气
```

生成：

```
{
"name":"get_weather",
"arguments":{
"city":"上海"
}
}
```

------

## 人负责：

创造工具。

例如：

开发：

```
get_weather()

search_database()

send_email()
```

并定义：

- 输入格式
- 输出格式
- 权限
- 错误处理

------

# 4. State（状态）

## LLM负责：

读取状态。

例如：

```
当前任务：
寻找上海酒店

已经完成：
机票查询

下一步：
酒店筛选
```

------

## 人负责：

设计 State。

例如：

决定保存：

```
state={
"user_goal":"",
"budget":"",
"history":"",
"tool_result":"",
"task_status":""
}
```

------

为什么重要？

因为：

> State 决定 Agent 记得什么。

------

# 5. Memory（记忆）

## LLM负责：

利用记忆。

例如：

发现：

```
用户喜欢靠窗座位
```

以后推荐：

```
优先靠窗
```

------

## 人负责：

设计：

什么值得记？

例如：

长期记忆：

```
用户偏好
历史习惯
专业背景
```

短期记忆：

```
当前任务状态
最近工具结果
```

------

# 6. Router（路由）

Router 决定：

> 下一步走哪里。

例如：

```
             Agent

              ↓

          是否需要工具？

          /          \

       是             否

       ↓              ↓

     Tool          Answer
```

------

## LLM负责：

判断：

```
现在需要搜索吗？
```

------

## 人负责：

定义：

有哪些路径。

例如：

```
查询天气
 ↓
天气不好？
 ↓
推荐室内方案
```

------

# 7. Execution（执行）

这里一定强调：

## LLM 不执行。

LLM：

```
我要调用 send_email
```

真正执行：

```
Python程序
API
数据库
操作系统
```

------

所以：

人负责：

- 编写执行代码
- 管理资源
- 控制安全

------

# 8. Permission（权限）

这是生产级 Agent 必须讲的。

LLM：

```
我要转账10000元
```

但是：

系统：

```
是否允许？
```

------

人负责：

定义：

```
什么事情自动执行？

什么事情必须确认？
```

例如：

| 操作     | 策略     |
| -------- | -------- |
| 查询天气 | 自动     |
| 搜索网页 | 自动     |
| 发送邮件 | 确认     |
| 付款     | 人工审批 |

------

# 9. Loop（循环）

Agent 最大特点：

不是一次输出。

而是：

```
思考
 ↓
行动
 ↓
观察
 ↓
继续
```

------

LLM：

决定：

```
下一步做什么
```

------

人：

控制：

```
最多循环多少次

什么时候停止

失败怎么办
```

例如：

```
MAX_STEP=10
```

防止：

```
LLM
 ↓
Tool
 ↓
LLM
 ↓
Tool
 ↓
无限循环
```

------

# 10. Error（错误）

LLM：

识别：

```
工具失败
```

------

人：

设计：

失败怎么办？

例如：

```
API失败

第一次：
重新请求

第二次：
换备用API

第三次：
通知用户
```

------

# 11. Human（人工）

这是 Agent 系统的重要设计。

LLM：

知道：

```
这个事情风险较高
```

请求：

```
需要人工确认
```

------

人：

负责：

最终决策。

例如：

```
购买商品
发送合同
金融操作
删除数据
```

------

# 六、总结：Agent 是人和 LLM 的协作系统

最后形成：

```
             人

 ┌─────────────────────┐
 │ 定义目标             │
 │ 设计工具             │
 │ 设计状态             │
 │ 设置权限             │
 │ 制定规则             │
 └─────────┬───────────┘

           ↓


 ┌─────────────────────┐
 │        LLM           │
 │                     │
 │ 理解                │
 │ 推理                │
 │ 规划                │
 │ 决策                │
 └─────────┬───────────┘

           ↓


      Tools / Environment
```

![3.2](C:\Users\likunyuan\Desktop\新建文件夹\3.2.png)

# 七、本章最重要的一句话

> **LLM 负责“不确定性”，人负责“确定性”。**

展开解释：

- LLM 擅长：

```
理解复杂需求
生成方案
处理未知情况
动态调整
```

- 人擅长：

```
定义目标
设计系统
建立规则
控制风险
保证可靠
```

所以：

> **真正优秀的 Agent，不是让 LLM 无限自由，而是在人的工程设计下，让 LLM 在安全边界内发挥自主性。**

------

# 第四章：Agent Framework——为什么需要框架？

> 如果我要开发一个真正的 Agent 呢？

需求变成：

```
10个Tool
3种Memory
5种Router
错误重试
并行执行
人工审批
状态持久化
日志
监控
多Agent
```

自己写：

```
Loop
State
Router
Tool
Memory
Checkpoint
Retry
HITL
...
```

代码很快膨胀。

于是：

> **Agent Framework 出现了。**

# 4.2 当 Agent 规模扩大后，会发生什么？

假设我们开发一个企业级 Agent。

需求：

帮助员工完成：

- 查询资料
- 写报告
- 发邮件
- 数据分析
- 生成 PPT
- 操作数据库
- 调用企业系统

那么系统可能变成：

```
20个 Tool

10个 Node

5种 Memory

多个 Agent

并行执行

状态持久化

日志系统

监控系统

人工审批

错误恢复
```

------

架构：

```
                  用户

                   ↓


             Agent System


       ┌──────────┼──────────┐

       ↓          ↓          ↓

    Planner    Researcher   Writer


       ↓          ↓          ↓


     Tool1      Tool2       Tool3


       ↓          ↓          ↓


    Memory    Database    API


       ↓

    Checkpoint


       ↓

    Human Approval
```

------

这时候问题出现：

## 问题1：代码越来越乱

最开始：

```
agent.py
```

可能只有：

```
100行
```

后来：

```
agent.py

tool.py

memory.py

router.py

permission.py

state.py

workflow.py

monitor.py
```

几千行代码。

------

## 问题2：流程难以维护

例如：

原流程：

```
用户问题

↓

搜索

↓

总结

↓

输出
```

后来增加：

```
如果搜索失败：

    换备用搜索

如果数据敏感：

    请求审批

如果金额超过限制：

    人工确认

如果结果质量不足：

    重新生成
```

流程变成：

```
if

elif

else

try

except

while
```

大量业务逻辑堆积。

------

## 问题3：状态越来越复杂

简单 Agent：

```
state={
"user_input"
}
```

企业 Agent：

可能需要：

```
state={

"user":

{},

"task":

{},

"conversation":

[],

"tool_history":

[],

"approval":

{},

"memory":

{},

"error":

{}

}
```

如何管理？

如何保存？

如何恢复？

都是问题。

------

# 4.3 Framework 解决什么？

这里引出核心：

> **Agent Framework 的价值，不是让 LLM 更聪明，而是让 Agent 系统更工程化。**

------

换句话：

LLM 解决：

```
怎么想？
```

Framework 解决：

```
怎么组织？
怎么运行？
怎么管理？
怎么控制？
```

------

# Framework 主要解决八类问题

------

# ① State 管理

没有 Framework：

自己维护：

```
state={}
```

问题：

- 谁修改？
- 什么时候保存？
- 如何恢复？

Framework：

提供：

```
State Schema

State Update

Checkpoint

Persistence
```

------

例如 LangGraph：

```
class AgentState:

    messages:list

    task:str

    result:str
```

状态自动流转。

------

# ② 流程编排（Workflow Orchestration）

Agent 最大特点：

不是一个函数。

而是：

```
思考

↓

行动

↓

观察

↓

再思考
```

Framework 提供：

```
Node

Edge

Router

Condition
```

把流程图代码化。

------

例如：

```
        Agent

          |

    是否调用工具？

       /       \

     Yes       No

      |         |

    Tool     Answer

      |

 Observation

      |

    Agent
```

------

# ③ Tool 管理

没有 Framework：

自己维护：

```
tools=[
search,
weather,
email
]
```

Framework：

提供：

```
Tool Registry

Tool Description

Tool Calling

Tool Validation
```

让 LLM 知道：

有哪些能力。

------

# ④ Memory 管理

Agent 需要：

短期记忆：

```
当前任务
```

长期记忆：

```
用户偏好
历史经验
```

Framework 帮助：

```
Memory Store

Vector Database

Conversation History
```

------

# ⑤ Loop 管理

手写：

```
while True:

    agent()
```

生产环境需要：

```
最大步骤限制

失败退出

暂停恢复

重新执行
```

Framework 提供：

```
Loop Controller

Checkpoint

Resume
```

------

# ⑥ Human-in-the-loop

企业场景：

不能完全自动。

例如：

```
Agent:

准备发送合同


↓

Framework:

等待人工确认


↓

继续执行
```

Framework 管理：

```
暂停

保存状态

等待

恢复
```

------

# ⑦ Error Handling

现实：

工具一定会失败。

例如：

```
API失败

↓

Retry

↓

备用方案

↓

人工介入
```

Framework 提供：

```
Retry Policy

Fallback

Exception Handling
```

------

# ⑧ Observability（可观测性）

企业必须知道：

Agent 为什么这样做？

例如：

日志：

```
10:01

用户请求


10:02

Agent规划


10:03

调用搜索工具


10:04

生成结果
```

Framework 提供：

```
Tracing

Logging

Monitoring

Evaluation
```

------

# 4.4 Framework ≠ Agent

这是必须强调的一句话。

> **Framework 是脚手架，不是建筑物。**

------

类比：

## 建筑

```
建筑设计
+
建筑材料
+
施工人员
```

才能形成建筑。

------

对应：

## Agent

```
目标设计
+
Prompt设计
+
Tool设计
+
Memory设计
+
流程设计
+
LLM
```

才能形成 Agent。

------

Framework 类似：

```
脚手架
```

帮助你：

- 搭结构
- 管流程
- 管组件

但是：

它不会告诉你：

```
你的 Agent 应该解决什么问题？
```

------

# 例如 LangGraph

它不会自动创造：

```
智能客服Agent
```

它只是提供：

```
State

Node

Edge

Router

Loop

Checkpoint
```

你需要自己设计：

```
客服流程

工具

知识库

权限

异常处理
```

------

# 4.5 Framework 的本质

可以总结：

## 没有 Framework：

你写：

```
Agent = LLM + 大量控制代码
```

------

## 有 Framework：

变成：

```
Agent =
LLM
+
Framework提供的运行骨架
+
你的业务逻辑
```

------

图示：

```
          业务开发者


              ↓


    ┌─────────────────┐
    │   Agent Logic   │
    │                 │
    │ Goal            │
    │ Tool            │
    │ Prompt          │
    │ Memory          │
    └────────┬────────┘

             ↓


    ┌─────────────────┐
    │ Agent Framework │
    │                 │
    │ State           │
    │ Router          │
    │ Loop            │
    │ Memory          │
    │ Checkpoint      │
    └────────┬────────┘

             ↓


            LLM
```

------

# 4.6 本章总结

![4](C:\Users\likunyuan\Desktop\新建文件夹\4.png)

------

## Agent Framework 解决什么？

不是：

❌ 让模型变聪明

而是：

✅ 让 Agent 可以被工程化

------

它解决：

```
状态管理

流程控制

工具管理

记忆管理

循环执行

权限控制

错误恢复

人工协作

系统监控
```

## 总结：

> **Agent 是能力，Framework 是组织能力的工程体系。**

# 第五章：主流 Agent Framework——从架构思想理解 LangChain、LangGraph、AutoGen

真正需要理解的是：

> **不同 Framework 试图解决 Agent 系统中的不同工程难题。**

------

# 5.1 开场：为什么需要多个 Framework？

前面我们已经知道：

一个完整 Agent 需要：

```
Goal
 ↓
Planning
 ↓
Tool
 ↓
State
 ↓
Memory
 ↓
Execution
 ↓
Loop
```

当系统越来越复杂时，问题出现：

------

简单 Agent：

```
用户
 ↓
LLM
 ↓
工具
 ↓
结果
```

几十行代码即可。

------

复杂 Agent：

```
用户

 ↓

Planner

 ↓

Research Agent

 ↓

Tool

 ↓

Database

 ↓

Memory

 ↓

Reviewer

 ↓

Human Approval

 ↓

Final Answer
```

此时需要解决：

- 状态如何传递？
- 节点如何连接？
- 多 Agent 如何通信？
- 如何暂停恢复？
- 如何监控？

因此产生不同 Framework。

------

# 5.2 Framework 的三个主要发展方向

目前主流 Agent Framework，大致可以分成三个思想：

------

## 第一类：组件化 Agent 开发

代表：

> LangChain

核心思想：

> **提供构建 Agent 的各种零件。**

类似：

```
乐高积木
```

你自己组合。

------

## 第二类：状态图 Agent Workflow

代表：

> LangGraph

核心思想：

> **把 Agent 看成一个可控制的状态机。**

类似：

```
流程图
+
状态机
```

------

## 第三类：Multi-Agent 社会系统

代表：

> AutoGen

核心思想：

> **多个 Agent 通过通信完成任务。**

类似：

```
一个团队
```

------

# 5.3 LangChain：Agent 应用开发生态

------

# 核心定位

一句话：

> **LangChain 是 LLM 应用开发组件生态。**

它最初解决的问题：

不是复杂 Agent。

而是：

> 如何快速把 LLM 接入真实应用。

------

# LangChain 提供什么？

核心组件：

```
Model

Prompt

Tool

Retriever

Memory

Parser

Agent
```

------

## 1. Model

负责连接：

```
GPT
Claude
Gemini
开源模型
```

统一接口。

例如：

```
model.invoke(
    "你好"
)
```

不用关心底层模型。

------

## 2. Prompt

管理：

```
System Prompt

Template

Few-shot Example
```

例如：

```
你是一个专业客服。

用户问题：

{question}

回答：
```

------

## 3. Tool

给 Agent 增加能力：

例如：

```
搜索
计算
数据库查询
API调用
```

------

## 4. Retriever

主要用于：

RAG。

流程：

```
用户问题

↓

Embedding

↓

Vector Database

↓

相关文档

↓

LLM回答
```

------

## 5. Memory

保存：

```
聊天历史

用户偏好

长期信息
```

------

## 6. Agent

让：

```
LLM

+

Tool

+

Loop
```

结合。

例如：

```
用户：

北京天气？


Agent：

需要调用天气工具


Tool：

返回结果


LLM：

生成回答
```

------

# LangChain 适合什么？

非常适合：

```
快速开发

RAG系统

知识库问答

LLM应用

简单Agent原型

Tool Calling
```

------

# LangChain 的不足

随着 Agent 复杂化：

问题：

```
复杂状态不好管理

复杂流程不好表达

长期运行困难
```

于是：

> LangGraph 出现。

![langchain](C:\Users\likunyuan\Desktop\新建文件夹\langchain.png)



------

# 5.4 LangGraph：状态驱动的 Agent Workflow

------

# 核心定位

一句话：

> **LangGraph 解决复杂 Agent 的流程控制问题。**

它不是重新创造 Agent。

而是：

让 Agent 变成：

```
State Machine
```

------

# 核心抽象：

```
State

↓

Node

↓

Edge

↓

Router

↓

Loop
```

------

## State（状态）

整个系统共享的数据。

例如：

```
state={

"user_input":

"帮我分析股票",

"messages":

[],

"tool_result":

None

}
```

------

## Node（节点）

每一步执行逻辑。

例如：

```
Node1:

LLM规划


Node2:

调用搜索


Node3:

生成报告
```

------

## Edge（边）

控制流程：

```
Node A

↓

Node B
```

------

## Router（条件路由）

根据状态决定下一步：

例如：

```
             Agent

              |

        是否需要工具？

          /        \

        是          否

        ↓            ↓

      Tool        Answer
```

------

## Loop（循环）

Agent 最核心：

```
思考

↓

行动

↓

观察

↓

再次思考
```

LangGraph 天然支持。

------

# LangGraph 特别适合：

```
复杂状态

循环Agent

条件分支

长期运行任务

人工审批

Checkpoint

企业级Agent
```

------

例如：

企业审批 Agent：

```
用户提交申请

↓

Agent分析

↓

风险判断

↓

需要审批？

    ↓

Human

↓

继续执行
```

------

# LangGraph 的设计思想：

不是：

```
写一个Agent
```

而是：

```
设计一个Agent Workflow
```

![langgraph](C:\Users\likunyuan\Desktop\新建文件夹\langgraph.png)

![langchain vs langgraph](C:\Users\likunyuan\Desktop\新建文件夹\langchain vs langgraph.png)

# 5.5 AutoGen：Multi-Agent 协作框架

------

# 核心定位

一句话：

> **AutoGen 关注多个 Agent 如何交流和协作。**

------

单 Agent：

```
User

 ↓

Agent

 ↓

Tool
```

------

Multi-Agent：

```
             Manager

                |

       ----------------

       |              |

 Researcher       Coder

       |              |

       ----------------

              |

           Reviewer
```

------

# AutoGen 的核心思想：

> 一个复杂任务，不一定交给一个 Agent，而是拆成多个角色。

------

例如：

生成代码任务：

```
用户需求

↓

Manager

↓

Coder

↓

Tester

↓

Reviewer

↓

修改

↓

完成
```

------

每个 Agent：

拥有：

```
Role

Goal

Memory

Tool

Communication
```

------

# AutoGen 适合：

```
Multi-Agent系统

复杂任务分工

模拟团队协作

自动代码开发

研究型Agent
```

------

# AutoGen 的特点：

它强调：

不是：

```
一个大脑
```

而是：

```
多个专业角色组成团队。
```

------

![multi-agent](C:\Users\likunyuan\Desktop\新建文件夹\multi-agent.png)

# 5.6 三个框架核心思想比较

|               | LangChain        | LangGraph         | AutoGen       |
| ------------- | ---------------- | ----------------- | ------------- |
| 核心思想      | 组件生态         | 状态图工作流      | Agent通信     |
| 解决问题      | 快速构建LLM应用  | 复杂流程控制      | 多Agent协作   |
| 核心抽象      | Chain/Tool/Agent | State/Node/Edge   | Agent/Message |
| 重点          | 组件组合         | Workflow          | Communication |
| 状态管理      | 基础             | 强                | 中等          |
| 循环控制      | 基础             | 强                | 通过Agent交互 |
| Multi-Agent   | 支持             | 支持              | 核心          |
| RAG           | 强               | 继承LangChain生态 | 支持          |
| Human-in-loop | 支持             | 强                | 支持          |
| 复杂业务      | 一般             | 优秀              | 优秀          |

------

# 5.7 不要死记框架定位

> **不要认为 LangChain 只能做简单 Agent，LangGraph 只能做流程，AutoGen 只能做多 Agent。**

现实：

它们正在融合。

------

例如：

现在完全可能：

```
LangGraph

+
LangChain Tool

+
AutoGen Agent

+
Vector Database

+
MCP
```

共同组成系统。

------

真正比较应该看：

------

# ① 核心抽象

问：

这个框架如何描述 Agent？

例如：

LangGraph：

```
Graph
```

AutoGen：

```
Conversation
```

------

# ② 状态管理能力

复杂 Agent 必须考虑：

```
状态保存

恢复

共享

更新
```

------

# ③ 流程控制能力

是否支持：

```
条件分支

循环

暂停

恢复
```

------

# ④ Tool 生态

包括：

```
工具定义

工具发现

工具调用

参数验证
```

------

# ⑤ Multi-Agent 能力

是否支持：

```
Agent通信

角色定义

任务分配
```

------

# ⑥ Memory

支持：

```
短期记忆

长期记忆

向量数据库

历史恢复
```

------

# ⑦ Human-in-the-loop

企业非常重要：

```
暂停

审批

恢复
```

------

# ⑧ 部署能力

生产环境需要：

```
服务化

监控

日志

扩展
```

------

# ⑨ 学习成本

开发团队需要考虑：

```
简单项目

↓

LangChain


复杂项目

↓

LangGraph


研究型Multi-Agent

↓

AutoGen
```

# 5.5 三种架构思想

关键词理解：

| 框架      | 核心理解                            |
| --------- | ----------------------------------- |
| LangChain | **组件与应用开发抽象**              |
| LangGraph | **State + Graph + Workflow**        |
| AutoGen   | **Agent + Message + Collaboration** |

因此：

```
LangChain
    ↓
我有哪些组件？
怎么快速构建 LLM/Agent 应用？

LangGraph
    ↓
我的 Agent 如何运行？
状态如何流动？
流程如何控制？

AutoGen
    ↓
多个 Agent 如何交流？
如何分工？
如何协作？
```

# 第六章：如何选择 Agent Framework？

![5](C:\Users\likunyuan\Desktop\新建文件夹\5.png)

## 5.1 先纠正一个误区：没有“最好的框架”

很多初学者会问：

> LangGraph 和 AutoGen 哪个更强？

这个问题本身不准确。

因为：

不同框架解决的问题不同。

例如：

一个客服机器人：

```
用户问题
 ↓
查询知识库
 ↓
回答
```

和：

一个自动科研系统：

```
提出问题
 ↓
搜索论文
 ↓
设计实验
 ↓
写代码
 ↓
运行实验
 ↓
分析结果
 ↓
修改方案
```

它们需要的 Agent 架构完全不同。

------

所以：

> **框架选择不是比较谁功能多，而是匹配任务复杂度。**

------

# 5.2 第一个判断维度：任务流程复杂度

这是最重要的指标。

可以画：

```
任务复杂度

低
│
│   Prompt
│
│   Chain
│
│   Agent
│
│   Graph Workflow
│
│   Multi-Agent
│
高
```

------

## Level 1：简单调用

特点：

流程固定。

例如：

```
用户输入

↓

LLM

↓

回答
```

需求：

- Prompt
- API调用

即可。

不需要 Agent Framework。

------

例如：

文本总结：

```
输入文章

↓

生成摘要
```

------

## Level 2：Chain

特点：

步骤固定。

例如：

RAG：

```
问题

↓

检索

↓

生成答案
```

流程：

```
Step1()
Step2()
Step3()
```

人工定义。

------

适合：

- 文档问答
- 内容生成
- 信息抽取

------

对应：

LangChain。

------

# Level 3：Single Agent

特点：

流程不完全确定。

例如：

客服 Agent：

```
用户问题

↓

判断

↓

查询？

↓

调用工具？

↓

回答
```

------

这里：

下一步由 LLM 决定。

需要：

- Tool Calling
- Memory
- Loop

------

适合：

- AI助手
- 智能客服
- Copilot

------

# Level 4：Graph Workflow

特点：

任务存在：

- 多分支
- 循环
- 状态保存
- 人工审批

例如：

贷款审批 Agent：

```
申请

 ↓

资料检查

 ↓

风险评估

 ↓

人工审批

 ↓

结果通知
```

------

如果用普通 Agent：

容易：

- 状态丢失
- 死循环
- 难调试

------

需要：

Graph：

```
State

↓

Node

↓

Edge

↓

Router
```

------

适合：

LangGraph。

------

# Level 5：Multi-Agent

特点：

多个角色协作。

例如：

科研系统：

```
             Manager

                ↓

     --------------------

     ↓          ↓        ↓

 Research   Coding   Reviewer
```

------

每个 Agent：

有：

- 目标
- Prompt
- Tool
- Memory

------

适合：

AutoGen。

![复杂度](C:\Users\likunyuan\Desktop\新建文件夹\复杂度.png)

------

# 5.3 第二个判断维度：State复杂度

这是很多人忽略的。

Agent真正复杂的地方：

不是模型。

而是：

> 状态如何管理。

------

## 简单状态

例如：

聊天：

```
history=[]
```

即可。

------

## 复杂状态

例如：

一个自动编程 Agent：

State：

```
{
 "task":"",
 "files":[],
 "code":"",
 "test_result":"",
 "error":"",
 "plan":"",
 "iteration":5
}
```

------

此时需要：

- State持久化
- Checkpoint
- Resume

------

所以：

判断：

如果任务需要：

```
暂停

恢复

长期运行

中断继续

人工接管
```

优先考虑：

Graph类框架。

------

# 5.4 第三个判断维度：Agent数量

## 单 Agent

结构：

```
       Agent

          |

        Tools
```

例如：

个人助手。

------

## Multi-Agent

结构：

```
             Manager

                |

      ----------------

      |       |       |

   Search   Code   Review
```

------

什么时候需要多个Agent？

不是因为：

> 多Agent更高级。

而是：

角色之间存在明显分工。

------

例如：

软件开发：

一个Agent：

```
分析需求
写代码
测试
审核
```

可能Prompt越来越复杂。

拆成：

```
产品Agent

↓

开发Agent

↓

测试Agent

↓

审核Agent
```

更容易管理。

------

# 5.5 第四个判断维度：控制程度 vs 自主程度

这是最核心的工程思想。

![控制vs自由程度](C:\Users\likunyuan\Desktop\新建文件夹\控制vs自由程度.png)

实际上：

二者存在权衡。

------

## 高控制 Workflow

特点：

人决定流程。

例如：

```
Step1

↓

Step2

↓

Step3
```

优点：

- 稳定
- 可预测
- 易审计

缺点：

- 灵活性低

------

适合：

金融

医疗

审批

------

## 高自主 Agent

特点：

LLM决定：

```
下一步做什么
```

优点：

灵活。

缺点：

不可预测。

------

适合：

探索任务：

- Research
- 创意
- 数据分析

------

# 5.6 一个更工程化的选择矩阵

| 需求          | 推荐方向                |
| ------------- | ----------------------- |
| 简单问答      | 直接调用LLM             |
| 固定流程      | Chain                   |
| 需要工具调用  | Agent Framework         |
| 复杂状态流程  | LangGraph               |
| 多人协作      | AutoGen                 |
| 代码开发Agent | Codex/Claude Code类产品 |
| 企业生产系统  | Runtime + Framework组合 |

------

# 5.7 不同框架定位

## LangChain

定位：

> LLM应用组件库

优势：

```
Prompt

Tool

Retriever

Memory

Model
```

适合：

快速开发。

------

## LangGraph

定位：

> Stateful Agent Workflow Framework

优势：

```
State

Graph

Loop

Checkpoint

Human approval
```

适合：

复杂业务Agent。

------

## AutoGen

定位：

> Multi-Agent Conversation Framework

优势：

```
Agent通信

角色协作

消息管理
```

适合：

多Agent系统。

------

# 5.8 最终选择流程图

可以设计成：

```
                  开始

                    ↓

          是否需要自主决策？

              /          \

            否            是

            ↓              ↓

          Chain        是否有复杂状态？

                         /       \

                       否         是

                       ↓          ↓

                    Agent      Graph


                                  ↓

                         是否多个角色？

                              /     \

                            否       是

                            ↓        ↓

                       LangGraph   AutoGen
```

------

# 最后总结

![5.8](C:\Users\likunyuan\Desktop\新建文件夹\5.8.png)

> **选择 Agent Framework，本质是在选择“控制权分配方式”。**

也就是：

> **多少事情交给程序确定，多少事情交给 LLM 动态决定。**

------

最终形成：

```
低自主、高控制：

Workflow


中间：

Graph Agent


高自主：

Agent


多个自主角色：

Multi-Agent
```

这句话可以作为整个 Framework 章节的收尾：

> **优秀的 Agent 系统，不是让 LLM 拥有无限自由，而是在自主性和可控性之间找到工程上的最佳平衡。**

------

# 九、第五章最后：真正用框架创造 Agent

这时候再进入实战。

建议不要一下子上复杂项目。

按照：

```
Level 1
最简单 Agent
```

↓

```
Level 2
Agent + Tool
```

↓

```
Level 3
Agent + Tool + Loop
```

↓

```
Level 4
Agent + State
```

↓

```
Level 5
Agent + Memory
```

↓

```
Level 6
Agent + Human
```

↓

```
Level 7
Multi-Agent
```

这样教学效果会非常好。





但是，在刚才的框架里，我们用的是最经典的、也是最基础的 **ReAct（推理+行动）** 架构。在实际的工业界，业务需求是非常复杂的。ReAct 就像是让一个“单兵”去打仗，效率有限。

随着 AI 的发展，业界为了应对不同的场景，衍生出了多种主流的 Agent 架构。今天，我带大家俯瞰一下**“Agent 架构的五座大山”**。大家要知道，现在的主流架构不只是“一个模型跑循环”，它们大致可以分为五种流派。

------

### 一、流派 1：ReAct 架构——单兵冲锋（我们刚才讲的基础）

- **核心特征**：边走边看，Thought -> Action -> Observation 逐步循环。
- **优点**：灵活，适应性强，能实时应对环境变化。
- **缺点**：**效率低**。每次都要等 LLM 推理完才动一步，如果任务有 10 步，就要调用 10 次大模型，Token 成本成倍增加。
- **适用场景**：任务路径不明确，需要实时根据外部环境微调的任务。

### 二、流派 2：Plan-and-Execute 与 ReWOO——先定战略再出兵（降本增效神器）

- *(PPT展示：拆分图，上面是一个“总规划”，下面是并行执行的步骤)*
- **核心特征**：**先把战略定好，再一齐出兵。**
- **工作原理**：
  - **Plan（规划）**：LLM 第一步不是去调工具，而是先一次性生成一个完整的执行计划列表（比如：步骤1-查日期，步骤2-查库存，步骤3-下单）。
  - **Execute（执行）**：系统拿到这个列表，**在宿主机上自动、并行地调用各种工具**，收集到所有结果。
  - **Final Answer（总结）**：最后再把所有结果汇总，喂给 LLM，让它写一份最终报告。
- **ReWOO（Reasoning WithOut Observation）**是这类架构的典型代表。它和 ReAct 最大的区别在于：**ReAct 是把时间花在“观察和重复决策”上，而 ReWOO 把时间花在“一次性做计划和并行执行”上。**
- **价值**：如果一项任务有 10 步，ReAct 要调用 10 次 LLM；而 ReWOO 最多只调用 **3 次 LLM**（定计划 1 次 + 总结 1 次）。**能省下 70% 以上的 API 成本！**

### 三、流派 3：Reflexion 架构——自我复盘的批判者（自我纠错神器）

- *(PPT展示：步骤路径：做错 -> 自我批评 -> 重新做)*
- **核心特征**：**不忙着冲，做错了知道复盘。**
- **工作原理**：
  - Agent 执行完一轮，得到一个错误结果（比如它把用户名字写错了，或者预算算超了）。
  - 它不会直接进入下一轮，而是触发一个 **Critic（评判）** 或 **Reflection（反思）** 模块。
  - LLM 会看着自己刚才犯错的执行日志，生成一句批评：“我刚才犯了错，因为我忽略了用户提供的身份证号，下次执行必须加上这个参数。”
  - **将这段反思（Critique）注入到下一轮的 System Prompt 中，重新执行。**
- **价值**：这是工程上解决“幻觉”和“中间步骤错误”非常有效的办法。它不是靠修改模型权重（微调），而是靠**“口头强化学习（语言层面的强化）”**，非常巧妙。

### 四、流派 4：LATS / MCTS（树搜索架构）——自带“上帝视角”的围棋高手

- *(PPT展示：画一棵树，展示多条分支路径的决策)*
- **核心特征**：**我不是走一步看一步，我是把所有可能的路都模拟一遍，挑最好的那条走。**
- **工作原理**：
  - 这借用了 AlphaGo 的 **MCTS（蒙特卡洛树搜索）** 算法。
  - 遇到复杂问题时，Agent 并不是立刻行动，而是让 LLM 同时 **“幻想（模拟）”** 出 3 条或 5 条不同的解决路径。
  - 它模拟每一条路径可能得到的结果，并对结果打分。
  - 最后，Agent 选择**得分最高（预期最优）**的那条路径去真正执行。
- **价值**：**极其强大，但极其昂贵。** 它非常适合“容错率极低”的任务，比如：规划跨城市的复杂路线、撰写需要深度博弈的商业文案、或者进行复杂的金融策略推演。

### 五、流派 5：多智能体架构（Multi-Agent System）——组队打仗，分工协作

- *(PPT展示：经理、程序员、测试员三个角色互相交流的图示)*

- **核心特征**：**一个 Agent 解决不了的问题，我拉一支 Agent 团队来解决。**

- **主流框架**：微软的 **AutoGen** 和 **CrewAI** 是这方面的代表。

- **工作原理**：

  - 不再是一个大模型单打独斗，而是让多个 Agent 分别扮演不同角色。
  - 比如：
    - **Manager（经理）**：拆解任务，分配工作。
    - **Coder（程序员）**：专门负责写代码。
    - **Critic（测试员）**：负责运行代码，抓 Bug。
    - **Searcher（搜索员）**：负责搜索最新技术文档。
  - 这些 Agent 之间通过对话进行交流。程序员写了代码，测试员说“报错了”，程序员立刻修改，修改完再通知测试员，直到通过。整个过程完全由 AI 自主驱动。

- **价值**：多智能体架构被业界认为是**攻克“极度复杂的长链条任务”的最优解**。单一 LLM 容易“思维退化”，而多智能体之间的观点碰撞和互审，能极大提高最终结果的质量。

  ++复杂规划算法

如果你说的 **“Codex CLI 已经完全取代了所有其他智能体框架”**，我建议先把这句话拆开：**Codex CLI 是一种 Agent/Agent Harness 产品形态，并不等于把 LangGraph、CrewAI、AutoGen 等“框架”全部取代了。**尤其是做企业级、多智能体、可控工作流时，它们解决的问题并不完全相同。

截至 **2026 年**，可以把主流 Agent 技术栈大致分成下面几类：

| 框架/产品                     | 核心定位                   | 最擅长                                        | 与 Codex CLI 的关系                   |
| ----------------------------- | -------------------------- | --------------------------------------------- | ------------------------------------- |
| **OpenAI Codex CLI**          | 编程 Agent / Agent Harness | 写代码、改代码、执行命令、测试、软件工程      | 更像“直接工作的 Agent”                |
| **LangGraph**                 | Agent 编排框架             | 状态机、复杂流程、Human-in-the-loop、多 Agent | 偏“自己搭 Agent 系统”                 |
| **LangChain**                 | LLM/Agent 应用框架         | 模型、Tool、RAG、Agent 快速开发               | LangGraph 的上层生态                  |
| **CrewAI**                    | Multi-Agent 框架           | 角色化 Agent 团队、任务协作                   | 偏“多个 Agent 组成团队”               |
| **Microsoft Agent Framework** | 企业级 Agent 框架          | Agent + Workflow + MCP + 企业部署             | AutoGen/Semantic Kernel 的后继方向    |
| **AutoGen**                   | Multi-Agent 框架           | Agent 对话、协作、实验性多智能体              | 正在向 Microsoft Agent Framework 演进 |
| **Semantic Kernel**           | 企业 AI SDK                | 企业集成、插件、Workflow、微软生态            | 已被 MAF 吸收/演进                    |
| **LlamaIndex**                | 数据/RAG/Agent 框架        | Agent + 数据、知识库、RAG                     | 数据密集型 Agent                      |
| **MetaGPT**                   | 多 Agent 软件工程          | 产品经理→架构师→程序员→测试                   | 软件研发 Multi-Agent                  |
| **Dify**                      | Agent 应用平台             | 可视化 Workflow、RAG、Agent                   | 更偏应用平台而非底层框架              |
| **MCP**                       | Tool/Context 协议          | Agent ↔ 工具/外部系统                         | 不是 Agent 框架，是基础协议           |
| **A2A**                       | Agent 间通信协议           | Agent ↔ Agent                                 | 不是 Agent 框架，是互操作协议         |

其中几个现在尤其值得关注。

### 1. Codex CLI

它的核心不是“帮你调用 LLM”，而是：

```
用户
 ↓
Codex Agent
 ↓
理解任务
 ↓
规划
 ↓
读取代码
 ↓
调用工具
 ├── Shell
 ├── 文件系统
 ├── Git
 ├── 测试
 └── 其他工具
 ↓
修改代码
 ↓
运行验证
 ↓
继续修复
 ↓
最终结果
```

所以它更接近：

> **Agent Harness + Coding Agent + Tool Execution Runtime**

而不是传统意义上的：

```
LLM API
   ↓
Agent Framework
   ↓
Tool
```

------

### 2. LangGraph

这是目前非常重要的一条路线。

[LangGraph 官方](https://www.langchain.com/langgraph?utm_source=chatgpt.com)

它的思想是：

```
              ┌──────────────┐
              │    用户任务   │
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │     Agent    │
              └──────┬───────┘
                     ↓
               判断下一步
              ↙     ↓      ↘
           Tool    Agent    Human
              ↘     ↓      ↙
                State
                  ↓
                下一轮
```

核心是 **State + Node + Edge + Checkpoint**。

它特别适合：

- 长流程 Agent
- 多 Agent
- Human-in-the-loop
- 可恢复执行
- 状态持久化
- 复杂业务 Workflow

官方也明确把 LangGraph 定位为低层 Agent orchestration/runtime。

------

### 3. CrewAI

[CrewAI 官方](https://crewai.com/?utm_source=chatgpt.com)

它的核心思想非常直观：

```
                  Crew
                   │
       ┌───────────┼───────────┐
       ↓           ↓           ↓
   Researcher   Analyst      Writer
       │           │           │
       └───────────┼───────────┘
                   ↓
                 Result
```

也就是：

> **把 Agent 当成一个“员工”，组成一个 AI Team。**

例如：

```
产品需求
   ↓
产品经理 Agent
   ↓
架构师 Agent
   ↓
程序员 Agent
   ↓
测试 Agent
   ↓
代码审查 Agent
```

CrewAI 官方目前仍然把它定位为 open-source multi-agent platform，并强调 Crews + Flows + Tools。

------

### 4. AutoGen

这是微软非常经典的 Multi-Agent 路线。

过去经常是：

```
User
 ↓
Planner Agent
 ↓
Coder Agent
 ↕
Reviewer Agent
 ↓
Executor Agent
```

Agent 之间通过消息进行协作。

但是这里有一个**非常重要的 2026 年变化**：

> **AutoGen 已经不是微软未来主推的最终框架了。**

Microsoft 现在已经推出 **Microsoft Agent Framework（MAF）**，并明确将 AutoGen 和 Semantic Kernel 的能力进行融合。MAF 在 2026 年 4 月达到 1.0 GA。

所以如果你现在学习微软这一条路线：

```
AutoGen
   ↘
    Microsoft Agent Framework
   ↗
Semantic Kernel
```

比单独钻研 AutoGen 更有长期价值。

------

### 5. Microsoft Agent Framework

这是现在特别值得注意的框架。

[Microsoft Agent Framework 官方文档](https://learn.microsoft.com/en-us/agent-framework/?utm_source=chatgpt.com)

它可以理解为：

```
              Microsoft Agent Framework
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
        Agent        Workflow        Tools
          │             │             │
          ↓             ↓             ↓
       Memory       Multi-Agent       MCP
          │             │             │
          └─────────────┼─────────────┘
                        ↓
                  Enterprise App
```

它融合了：

- AutoGen 的 Agent / Multi-Agent 思想
- Semantic Kernel 的企业能力
- Workflow
- Memory
- MCP
- A2A
- Middleware
- Telemetry
- Hosting

微软自己也把它描述为 AutoGen + Semantic Kernel 的下一代统一框架。

------

## 6. LlamaIndex

[LlamaIndex 官方](https://www.llamaindex.ai/?utm_source=chatgpt.com)

它和 LangGraph/CrewAI 的思路不太一样。

更偏：

```
                Agent
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
     文档      数据库      API
        ↓         ↓         ↓
             Data Layer
                  ↓
             Knowledge
```

所以如果你的 Agent 要大量处理：

- PDF
- 企业知识库
- 数据库
- RAG
- 文档
- 非结构化数据

LlamaIndex 很重要。

------

# 7. MetaGPT

这个和 Codex CLI 的关系其实很有意思。

它走的是：

> **Software Company Simulation**

例如：

```
              用户需求
                  ↓
             产品经理 Agent
                  ↓
             架构师 Agent
                  ↓
             工程师 Agent
             ↙         ↘
        程序员 Agent   测试 Agent
             ↓         ↓
              └──→ Code
```

也就是把一个软件公司的岗位拆成多个 Agent。

因此：

**Codex CLI：**

```
一个非常强的 Software Engineer Agent
```

**MetaGPT：**

```
一个 AI Software Company
```

这是两个不同层次。

------

# 8. Dify

Dify 更偏“平台”。

可以理解成：

```
             Dify
               │
      ┌────────┼────────┐
      ↓        ↓        ↓
     RAG     Agent    Workflow
      ↓        ↓        ↓
     Tool     Tool     Tool
      └────────┼────────┘
               ↓
           AI Application
```

如果目标是：

> 快速做一个可以上线的 Agent 应用

Dify 很方便。

如果目标是：

> 研究 Agent Runtime / Agent Architecture

那么 LangGraph、MAF 这类底层框架更值得研究。

------

# 9. MCP

这个特别容易被误认为 Agent Framework。

实际上：

> **MCP 不是 Agent 框架。**

它更像：

```
             Agent
               │
             MCP
        ┌──────┼──────┐
        ↓      ↓      ↓
      GitHub  DB    Browser
        │      │      │
       Tool   Tool   Tool
```

也就是统一 **Agent → Tool/Context** 的连接方式。

所以未来很可能是：

```
Codex
LangGraph
CrewAI
Microsoft Agent Framework
LlamaIndex
        │
        ↓
       MCP
        │
        ↓
各种外部工具
```

而不是 MCP 和这些框架互相竞争。

------

# 真正应该怎么理解这些东西？

我建议你不要把它们全部放在“Agent 框架”这个篮子里。

更准确的架构层次是：

```
                         AI Agent 技术栈
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
       ↓                      ↓                      ↓
  Agent 产品/Harness      Agent Framework        Agent Protocol
       │                      │                      │
       │             ┌────────┼────────┐             │
       ↓             ↓        ↓        ↓             ↓
    Codex CLI    LangGraph  CrewAI    MAF           MCP
                 LangChain  AutoGen                 A2A
                 LlamaIndex
                    │
                    ↓
              Agent Runtime
                    │
                    ↓
             LLM / Tools / Data
```

所以**“Codex CLI 取代所有智能体框架”这个说法并不准确**。

更准确的说法应该是：

> **Codex CLI 正在把“编程 Agent”从传统 Agent Framework 中抽离出来，变成一个高度自治的 Agent Harness。**

而 LangGraph、CrewAI、Microsoft Agent Framework 等仍然解决：

> **如何构建、编排、控制、持久化和部署一个 Agent 系统。**

------

## 如果你现在是在系统学习 Agent

我会把这些框架压缩成 **5 条技术路线**：

```
                    Agent 技术路线
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ↓                 ↓                 ↓
  Coding Agent       Agent Runtime     Multi-Agent
       │                 │                 │
    Codex CLI        LangGraph        CrewAI
                                     MAF/AutoGen
       │
       ↓
  Software Agent
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
           Data/RAG              Protocol
          LlamaIndex             MCP / A2A
```

**如果你的目标是研究“Agent 架构与工作机制”**，我反而建议重点理解：

**LangGraph → CrewAI → AutoGen/MAF → Codex CLI → MCP/A2A**

因为这样可以把我们之前讨论的 **ReAct、Plan-and-Execute、Reflection、Multi-Agent、Hierarchical Agent、Memory、Tool Calling** 都串起来，而不是只学习某一个框架的 API。