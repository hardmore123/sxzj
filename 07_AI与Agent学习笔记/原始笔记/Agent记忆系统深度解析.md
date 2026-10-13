# Agent 记忆系统深度解析：从 Context Window 到长期记忆（Memory Architecture）

前面的 Agent 架构中，我们讲：

```
感知 → 规划 → 决策 → 执行 → 再感知
```

但是有一个问题：

> Agent 每一次循环结束后，它还能不能记住刚才发生的事情？

如果不能：

- 每次都不知道用户是谁；
- 不知道任务做到哪里；
- 不知道过去的选择；
- 不知道用户长期偏好。

那么它只是一个“高级聊天机器人”，不是智能 Agent。

真正的 Agent 必须具备：

> **状态保持能力（State Persistence）**

也就是：

> 记忆系统。

------

# 一、Agent记忆的整体架构

工业级 Agent 的记忆不是一个数据库，而是分层系统。

整体：

```
                    用户

                     |
                     ↓

                  Agent


          ┌───────────────────┐
          │                   │
          ↓                   ↓

    短期记忆              长期记忆

    Short Memory          Long Memory


          ↓                   ↓

   当前任务状态          用户历史经验

          ↓                   ↓

   Context Window       Vector DB


          ↓

       Memory Manager

          ↓

    写入 / 检索 / 更新 / 删除
```

------

# 二、为什么不能直接把所有历史塞进 Prompt？

很多初学者第一反应：

> “记忆不就是把聊天记录全部放进去吗？”

例如：

```
System Prompt

用户第一句话

用户第二句话

用户第三句话

......

用户第1000句话

当前问题
```

然后发送给 LLM。

看似简单。

实际上会出现严重问题。

------

## 1. Token成本爆炸

假设：

每天聊天：

10000 token

一年：

```
10000 × 365

=365万token
```

不可能每次调用都发送。

------

## 2. Attention稀释

Transformer：

核心：

Self Attention。

公式：
$$
Attention(Q,K,V)
=
softmax(\frac{QK^T}{\sqrt d})V
$$
简单理解：

模型计算：

> 当前问题(Query)应该关注哪些历史信息(Key)

例如：

用户：

100轮以前：

```
我喜欢靠窗座位
```

现在：

```
帮我订机票
```

中间：

大量：

```
查询航班

天气

价格

酒店

支付
```

那么：

“靠窗”

这个信息可能被大量无关Token淹没。

------

# 三、短期记忆（Short-term Memory）

短期记忆主要解决：

> 当前任务进行到哪里？

它不是简单历史。

分为两个部分：

------

# 1. Conversation History（对话历史）

例如：

```
用户：

我要订上海机票


Agent：

什么时候？


用户：

下周三
```

保存：

```
User:
我要订上海机票

Assistant:
什么时候？

User:
下周三
```

------

作用：

保持聊天连续性。

------

# 2. Working Memory（工作记忆）

这是 Agent 最核心部分。

例如：

订机票任务。

用户：

> 帮我订北京到上海机票

Agent内部状态：

```
{
"task":
"订机票",

"departure":
"北京",

"arrival":
"上海",

"date":
null,

"budget":
null,

"candidate_flights":
[]
}
```

这不是给用户看的。

这是 Agent 的内部状态。

------

# 四、为什么需要 Working Memory？

因为 Agent 是状态机。

传统程序：

```
输入

↓

执行

↓

结束
```

无状态。

------

Agent：

```
状态S0

↓

思考

↓

动作

↓

得到结果

↓

状态S1

↓

继续思考
```

数学上：

类似 MDP：
$$
S_t,A_t,R_t,S_{t+1}
$$
其中：

S：

当前状态。

------

例如：

第一轮：

状态：

```
日期未知
```

调用：

```
get_date()
```

得到：

```
2026-08-05
```

更新：

```
日期=2026-08-05
```

下一轮继续。

------

# 五、Working Memory如何实现？

一般不是数据库。

通常：

## 方案1：JSON State

例如 LangGraph：

```
state={

"user_goal":
"订机票",

"date":
"2026-08-05",

"budget":
1500,

"flight_options":
[]

}
```

每个节点修改 State。

------

## 方案2：State Machine

例如：

```
START

 ↓

collect_info

 ↓

search

 ↓

confirm

 ↓

book

 ↓

END
```

每一步：

更新状态。

------

# 六、Context Window管理

LLM真正看到的是：

```
Prompt

+

Messages

+

Memory Injection
```

例如：

发送：

```
System:

你是旅游助手


Memory:

用户喜欢南航


Current Task:

北京上海机票


Conversation:

今天讨论内容


User:

帮我订票
```

------

# 七、上下文压缩（Context Compression）

当越来越长：

必须压缩。

主要方法：

------

# 方法1：滑动窗口

保留最近N轮。

例如：

10轮：

```
1 2 3 4 5 6 7 8 9 10


超过：

删除1

保留2-11
```

优点：

简单。

缺点：

可能丢重要信息。

------

# 方法2：摘要

让模型总结。

原始：

5000 token

变：

300 token

例如：

原始：

```
用户讨论了20个航班

比较价格

修改时间

最终选择东航MU123
```

摘要：

```
用户已选择东航MU123，
时间8月5日上午8点，
预算1420元。
```

------

# 方法3：信息抽取

不是摘要。

而是结构化。

例如：

输入：

```
我喜欢住机场附近酒店，不喜欢市中心。
```

输出：

```
{
"preference":

{
"hotel_location":
"near_airport",

"avoid":
"downtown"

}

}
```

------

# 八、长期记忆 Long-term Memory

解决：

> 跨会话记忆。

例如：

今天：

用户：

```
我喜欢希尔顿酒店
```

半年后：

用户：

```
帮我订酒店
```

Agent：

知道：

```
优先推荐希尔顿
```

------

# 九、长期记忆存在哪里？

通常：

Vector Database。

例如：

- Milvus
- Pinecone
- Qdrant
- Weaviate

------

# 十、Embedding是什么？

计算机不能理解：

```
喜欢希尔顿
```

转换：

向量。

例如：

```
[0.234,
0.765,
-0.122,
...
]

768维
```

------

相似语义：

距离近。

例如：

```
喜欢希尔顿

↓

酒店偏好五星

距离近
```

------

# 十一、向量检索流程

完整：

```
用户输入

↓

Embedding模型

↓

生成Query Vector

↓

Vector DB搜索

↓

返回Top-K记忆

↓

加入Prompt

↓

LLM
```

------

例如：

用户：

```
帮我订酒店
```

Embedding：

```
[0.32,0.51...]
```

搜索：

返回：

```
Memory1:

用户喜欢机场附近酒店


Memory2:

用户预算500元


Memory3:

用户喜欢早餐
```

------

# 十二、为什么需要HNSW？

假设：

1亿条记忆。

暴力：

比较：

```
Query

vs

100000000 vectors
```

太慢。

------

HNSW：

Hierarchical Navigable Small World。

思想：

建立导航地图。

类似：

找北京某个地址。

不会：

遍历全国。

而是：

```
中国

↓

北京

↓

朝阳区

↓

某街道
```

------

结构：

```
Level 2

       A

      / \


Level 1

 A---B---C


Level 0

A-B-C-D-E-F
```

搜索：

先粗定位。

再精确。

复杂度：

接近：
$$
O(logN)
$$

------

# 十三、为什么单纯向量搜索不够？

例子：

用户：

```
找订单20260101
```

Embedding：

可能理解：

订单相关。

但是：

编号必须精确。

所以：

需要：

## Hybrid Search

组合：

------

## 1. Vector Search

负责：

语义。

例如：

```
喜欢靠近机场
```

匹配：

```
机场附近酒店
```

------

## 2. BM25

关键词。

例如：

```
订单号20260101
```

精准。

------

## 3. Rerank

重新排序。

流程：

```
Query

 ↓

Vector Search

得到50条


+

BM25

得到50条


 ↓

合并100条


 ↓

Reranker


 ↓

Top5
```

------

# 十四、Memory写入机制（最重要）

是不是所有聊天都存？

不是。

否则：

垃圾记忆。

------

# Memory Consolidation

类似人脑：

白天经历：

↓

晚上整理：

↓

重要信息长期保存。

------

流程：

```
对话结束

↓

Memory Evaluator

↓

重要性评分

↓

决定是否保存

↓

Vector DB
```

------

# 十五、重要性评分模型

例如：

输入：

```
用户：

谢谢
```

评分：

0.05

丢弃。

------

输入：

```
以后酒店都选择希尔顿
```

评分：

0.95

保存。

------

分类：

| 类型     | 处理 |
| -------- | ---- |
| 闲聊     | 删除 |
| 临时任务 | 短期 |
| 稳定偏好 | 长期 |
| 关键事实 | 长期 |

------

# 十六、记忆生命周期

真正成熟系统：

不是：

存进去。

而是：

CRUD。

------

## Create

创建：

```
用户喜欢南航
```

------

## Read

查询：

```
订票

↓

读取航空偏好
```

------

## Update

用户：

```
现在喜欢东航
```

修改。

------

## Delete

用户：

```
删除我的偏好
```

删除。

------

# 十七、时间衰减

人的偏好会变化。

所以：

Memory Score：

通常：
$$
Score =
Similarity
\times
Importance
\times
Decay
$$

------

时间：

越久：

Decay越低。

例如：

一年以前：

```
0.5
```

昨天：

```
1.0
```

------

# 十八、完整Agent记忆架构

最终：

```
                 用户


                  |

                 Agent


                  |

        Memory Manager


       /             \


Short Memory       Long Memory


Working State      Vector DB


Context            Embedding


State Machine      Retrieval


                  |

              LLM


                  |

              Tool Use
```

------

# 十九、和人脑类比

| 人脑       | Agent                   |
| ---------- | ----------------------- |
| 工作记忆   | Context + Working State |
| 短期注意力 | Attention               |
| 长期记忆   | Vector DB               |
| 睡眠整理   | Memory Consolidation    |
| 遗忘       | Decay                   |
| 回忆       | Retrieval               |

------

# 二十、最终总结

真正的 Agent Memory：

不是：

> “保存聊天记录”

而是：

> **一个具备状态管理、信息抽取、重要性判断、向量检索、时间衰减、更新删除机制的认知系统。**

完整公式：

```
Agent Memory

=

短期上下文

+

工作状态机

+

长期向量记忆

+

检索策略

+

写入策略

+

遗忘机制
```

未来企业级 Agent 的核心竞争力，很大程度不是模型大小，而是：

**它如何记住、如何忘记、如何在正确时间调用正确的记忆。**