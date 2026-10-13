## 目录



***

## 概述

### 1.1 什么是SSE？

**SSE (Server-Sent Events)&#x20;**&#x662F;一种服务器向客户端单向推送数据的技术，基于HTTP协议，使用长连接实现实时数据流传输。

**核心特点&#x20;**：

* 基于HTTP，无需额外协议

* 自动重连机制

* 支持事件类型

* 轻量级（相比WebSocket）

* 单向推送（服务器→客户端）

* 文本格式（不支持二进制）

### 1.2 为什么选择SSE？

在本项目中，深度研究流程需要实时推送：

* 研究阶段变更（PLANNING → RESEARCHING → ...）

* 搜索结果（每找到一个新来源）

* 图表生成（每生成一个图表）

* 章节撰写（每完成一个章节）

* 最终报告（研究完成）

**SSE vs WebSocket 对比&#x20;**：

| 特性    | SSE         | WebSocket         |
| ----- | ----------- | ----------------- |
| 协议    | HTTP        | WebSocket (ws://) |
| 通信方向  | 单向（服务器→客户端） | 双向                |
| 重连    | 自动          | 需手动实现             |
| 浏览器支持 | 所有现代浏览器     | IE 10+            |
| 复杂度   | 低           | 中                 |
| 适用场景  | 实时通知、进度推送   | 聊天、游戏、协作编辑        |

**本项目选择SSE的理由&#x20;**：

1. 只需服务器推送，客户端无需发送消息（单向通信）

2. 自动重连机制简化错误处理

3) 实现简单，易于调试

4) HTTP协议，穿透防火墙更容易

***

## 2. 核心概念

### 2.1 SSE消息格式

SSE使用纯文本格式，每条消息以 `\n\n` （两个换行符）结尾：

```plaintext
data: {"type": "phase_change", "phase": "researching"}

data: {"type": "search_result", "title": "AI市场报告"}

event: custom_event
data: {"custom": "data"}
id: 123
```

**字段说明&#x20;**：

* `data:` - 消息内容（必需）

* `event:` - 事件类型（可选，默认为 `message` ）

* `id:` - 消息ID（可选，用于重连）

* `retry:` - 重连间隔（可选，单位毫秒）

### 2.2 事件类型设计

**文件位置&#x20;**: `/backend/app/service/deep_research_v2/service.py` (行号: 50-80)

本项目定义的事件类型：

```python
class SSEEventType:
    """SSE事件类型常量"""

    # 研究流程事件
    RESEARCH_START = "research_start"           # 研究开始
    PHASE_CHANGE = "phase_change"               # 阶段变更
    RESEARCH_COMPLETED = "research_completed"   # 研究完成
    RESEARCH_FAILED = "research_failed"         # 研究失败

    # Agent执行事件
    OUTLINE_GENERATED = "outline_generated"     # 大纲生成
    SEARCH_RESULT = "search_result"             # 搜索结果
    FACT_DISCOVERED = "fact_discovered"         # 事实发现
    CHART_GENERATED = "chart_generated"         # 图表生成
    SECTION_DRAFTED = "section_drafted"         # 章节草稿
    CRITIC_FEEDBACK = "critic_feedback"         # 审核反馈

    # 进度事件
    PROGRESS_UPDATE = "progress_update"         # 进度更新
    LOG_MESSAGE = "log_message"                 # 日志消息

    # 错误事件
    ERROR = "error"                             # 错误
    WARNING = "warning"                         # 警告


# 事件数据结构
class SSEEvent:
    """SSE事件数据结构"""

    def __init__(
        self,
        type: str,
        data: Dict[str, Any],
        id: Optional[str] = None,
        retry: Optional[int] = None
    ):
        self.type = type
        self.data = data
        self.id = id
        self.retry = retry

    def to_sse_format(self) -> str:
        """转换为SSE格式"""
        lines = []

        # event类型
        if self.type:
            lines.append(f"event: {self.type}")

        # id
        if self.id:
            lines.append(f"id: {self.id}")

        # retry
        if self.retry:
            lines.append(f"retry: {self.retry}")

        # data (JSON序列化)
        data_json = json.dumps(self.data, ensure_ascii=False)
        lines.append(f"data: {data_json}")

        # 两个换行符结尾
        return "\n".join(lines) + "\n\n"
```

***

## 3. FastAPI后端实现

### 3.1 SSE流式端点

**文件位置&#x20;**: `/backend/app/router/research_router.py` (行号: 50-120)

```python
from fastapi import APIRouter, HTTPException
from fastapi.responses import StreamingResponse
from typing import AsyncGenerator
import asyncio
import json

router = APIRouter(prefix="/research", tags=["深度研究"])


@router.post("/stream")
async def stream_research(request: ResearchRequest) -> StreamingResponse:
    """
    流式深度研究接口

    请求体:
        - query: 研究问题
        - session_id: 会话ID
        - search_modes: ["web", "local"]
        - version: "v2"

    响应: SSE流式事件
    """

    async def event_generator() -> AsyncGenerator[str, None]:
        """SSE事件生成器"""

        try:
            # 1. 创建研究服务
            service = DeepResearchV2Service()

            # 2. 发送开始事件
            yield format_sse_event({
                "type": SSEEventType.RESEARCH_START,
                "session_id": request.session_id,
                "query": request.query,
                "timestamp": datetime.utcnow().isoformat()
            })

            # 3. 执行研究（异步生成器）
            async for event in service.research(
                query=request.query,
                session_id=request.session_id,
                search_modes=request.search_modes
            ):
                # 发送每个事件
                yield format_sse_event(event)

            # 4. 发送完成事件
            yield format_sse_event({
                "type": SSEEventType.RESEARCH_COMPLETED,
                "session_id": request.session_id,
                "timestamp": datetime.utcnow().isoformat()
            })

        except asyncio.CancelledError:
            # 客户端取消连接
            logger.info(f"客户端取消连接: {request.session_id}")
            yield format_sse_event({
                "type": SSEEventType.WARNING,
                "message": "客户端取消连接"
            })

        except Exception as e:
            # 发送错误事件
            logger.error(f"研究失败: {e}")
            yield format_sse_event({
                "type": SSEEventType.ERROR,
                "error": str(e),
                "traceback": traceback.format_exc()
            })

    # 返回SSE响应
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no"  # 禁用Nginx缓冲
        }
    )


def format_sse_event(data: Dict[str, Any]) -> str:
    """格式化SSE事件"""
    # 序列化为JSON
    json_data = json.dumps(data, ensure_ascii=False)

    # SSE格式: data: {json}\n\n
    return f"data: {json_data}\n\n"
```

**关键点讲解&#x20;**：

1. **AsyncGenerator&#x20;**：使用 `yield` 返回流式数据

2. **StreamingResponse&#x20;**：FastAPI的流式响应类

3) **media\_type&#x20;**：必须设置为 `text/event-stream`

4) **Cache-Control&#x20;**：禁用缓存，确保实时性

5. **X-Accel-Buffering&#x20;**：禁用Nginx缓冲（生产环境重要）

### 3.2 研究服务异步生成器

**文件位置&#x20;**: `/backend/app/service/deep_research_v2/service.py` (行号: 150-300)

```python
class DeepResearchV2Service:
    """深度研究服务V2"""

    async def research(
        self,
        query: str,
        session_id: str,
        search_modes: List[str] = ["web", "local"]
    ) -> AsyncGenerator[Dict[str, Any], None]:
        """
        执行深度研究（流式输出）

        Yields:
            SSE事件字典
        """

        # 1. 创建事件队列
        event_queue = asyncio.Queue()

        # 2. 初始化State
        state = ResearchState(
            query=query,
            session_id=session_id,
            phase="INIT",
            iteration=0,
            search_web="web" in search_modes,
            search_local="local" in search_modes,
            messages=event_queue,  # 传入队列
            # ... 其他字段
        )

        # 3. 创建LangGraph
        graph = DeepResearchGraph()

        # 4. 启动后台任务执行图
        async def run_graph_task():
            """后台执行图"""
            try:
                config = {"configurable": {"thread_id": session_id}}
                async for _ in graph.graph.astream(state, config):
                    pass  # 图执行完成
            except Exception as e:
                # 错误推入队列
                await event_queue.put({
                    "type": SSEEventType.ERROR,
                    "error": str(e)
                })

        # 启动后台任务
        graph_task = asyncio.create_task(run_graph_task())

        # 5. 从队列中取出事件并yield
        try:
            while True:
                # 设置超时，避免无限等待
                try:
                    event = await asyncio.wait_for(
                        event_queue.get(),
                        timeout=1.0
                    )

                    # yield事件给客户端
                    yield event

                    # 如果是完成或失败事件，结束
                    if event["type"] in [
                        SSEEventType.RESEARCH_COMPLETED,
                        SSEEventType.RESEARCH_FAILED
                    ]:
                        break

                except asyncio.TimeoutError:
                    # 超时，检查后台任务是否完成
                    if graph_task.done():
                        break

                    # 发送心跳保持连接
                    yield {
                        "type": "heartbeat",
                        "timestamp": datetime.utcnow().isoformat()
                    }

        finally:
            # 清理：取消后台任务
            if not graph_task.done():
                graph_task.cancel()
                try:
                    await graph_task
                except asyncio.CancelledError:
                    pass
```

**关键点讲解&#x20;**：

1. **事件队列&#x20;**（ `asyncio.Queue` ）：

   * Agent将事件推入队列

   * 主流程从队列取出并yield给客户端

   * 解耦Agent执行和SSE推送

2. **后台任务&#x20;**（ `asyncio.create_task` ）：

   * 在后台执行LangGraph

   * 主协程负责推送事件

   * 避免阻塞SSE连接

3) **超时机制&#x20;**：

   * `asyncio.wait_for` 设置1秒超时

   * 超时后发送心跳，保持连接

   * 防止长时间无响应

4) **清理机制&#x20;**：

   * `finally` 块取消后台任务

   * 捕获 `CancelledError` 避免异常传播

### <span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">3.3 Agent推送事件（实际实现）</span>

**文件位置&#x20;**: `/backend/app/service/deep_research_v2/agents/base.py` (行号: 80-100)

```python
class BaseAgent:
    """Agent基类"""

    async def send_event(self, event: Dict[str, Any]):
        """
        发送SSE事件到队列

        Args:
            event: 事件数据字典
        """
        # 添加时间戳
        event["timestamp"] = datetime.utcnow().isoformat()

        # 推入队列
        if hasattr(self, "state") and "messages" in self.state:
            queue = self.state["messages"]
            if isinstance(queue, asyncio.Queue):
                await queue.put(event)
        else:
            logger.warning("未找到事件队列，无法发送事件")


# 在具体Agent中使用
class ChiefArchitect(BaseAgent):
    async def process(self, state: ResearchState) -> ResearchState:
        # ... 生成大纲

        # 发送事件
        await self.send_event({
            "type": SSEEventType.OUTLINE_GENERATED,
            "outline": plan["outline"],
            "hypotheses": plan["hypotheses"]
        })

        return state
```

***

## 4. 前端接收实现

### 4.1 EventSource API

**文件位置&#x20;**: `/frontend/src/pages/chat/index.tsx` (行号: 200-350)

```typescript
import { useEffect, useState } from 'react';

interface ResearchStep {
  id: string;
  type: string;
  title: string;
  status: 'pending' | 'running' | 'completed';
  data?: any;
}

export default function ChatPage() {
  const [researchSteps, setResearchSteps] = useState<ResearchStep[]>([]);
  const [isResearching, setIsResearching] = useState(false);

  // 开始研究
  const startResearch = async (query: string) => {
    setIsResearching(true);

    // 初始化步骤
    setResearchSteps([
      { id: '1', type: 'planning', title: '规划研究大纲', status: 'pending' },
      { id: '2', type: 'researching', title: '搜索资料', status: 'pending' },
      { id: '3', type: 'analyzing', title: '数据分析', status: 'pending' },
      { id: '4', type: 'writing', title: '撰写报告', status: 'pending' },
      { id: '5', type: 'reviewing', title: '质量审核', status: 'pending' },
    ]);

    // 创建SSE连接
    const eventSource = new EventSource(
      `http://localhost:8000/research/stream?query=${encodeURIComponent(query)}`
    );

    // 监听消息事件
    eventSource.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        handleSSEEvent(data);
      } catch (e) {
        console.error('解析SSE事件失败:', e);
      }
    };

    // 监听错误
    eventSource.onerror = (error) => {
      console.error('SSE连接错误:', error);
      eventSource.close();
      setIsResearching(false);
    };

    // 返回清理函数
    return () => {
      eventSource.close();
    };
  };

  // 处理SSE事件
  const handleSSEEvent = (event: any) => {
    switch (event.type) {
      case 'research_start':
        console.log('研究开始:', event);
        break;

      case 'phase_change':
        updateStepStatus(event.phase, 'running');
        break;

      case 'outline_generated':
        updateStepData('1', { outline: event.outline });
        updateStepStatus('planning', 'completed');
        break;

      case 'search_result':
        appendSearchResult(event);
        break;

      case 'chart_generated':
        appendChart(event);
        break;

      case 'section_drafted':
        updateSection(event.section_id, event.content);
        break;

      case 'research_completed':
        setIsResearching(false);
        updateStepStatus('reviewing', 'completed');
        break;

      case 'error':
        console.error('研究错误:', event.error);
        setIsResearching(false);
        break;
    }
  };

  // 更新步骤状态
  const updateStepStatus = (phase: string, status: string) => {
    setResearchSteps(prev => prev.map(step =>
      step.type === phase ? { ...step, status } : step
    ));
  };

  return (
    <div>
      <button onClick={() => startResearch('中国AI市场分析')}>
        开始研究
      </button>

      {/* 研究步骤展示 */}
      <div className="research-steps">
        {researchSteps.map(step => (
          <div key={step.id} className={`step step-${step.status}`}>
            {step.title}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 4.2 Fetch API + ReadableStream（更灵活）

如果需要更多控制（如POST请求、自定义Headers），可以使用Fetch API：

```typescript
const startResearchWithFetch = async (query: string) => {
  const response = await fetch('http://localhost:8000/research/stream', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify({
      query,
      session_id: sessionId,
      search_modes: ['web', 'local'],
      version: 'v2'
    })
  });

  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }

  // 获取ReadableStream
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();

      if (done) {
        break;
      }

      // 解码数据
      buffer += decoder.decode(value, { stream: true });

      // 按\n\n分割事件
      const events = buffer.split('\n\n');
      buffer = events.pop() || ''; // 保留不完整的事件

      // 处理每个事件
      for (const eventText of events) {
        if (eventText.trim()) {
          const event = parseSSEEvent(eventText);
          if (event) {
            handleSSEEvent(event);
          }
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
};

// 解析SSE事件
const parseSSEEvent = (eventText: string) => {
  const lines = eventText.split('\n');
  let data = '';

  for (const line of lines) {
    if (line.startsWith('data: ')) {
      data = line.substring(6);
    }
  }

  if (data) {
    try {
      return JSON.parse(data);
    } catch (e) {
      console.error('解析JSON失败:', e);
    }
  }

  return null;
};
```

***

## 5. 错误处理与重连

### 5.1 后端错误处理

```python
@router.post("/stream")
async def stream_research(request: ResearchRequest) -> StreamingResponse:
    async def event_generator() -> AsyncGenerator[str, None]:
        try:
            # 研究逻辑
            async for event in service.research(...):
                yield format_sse_event(event)

        except asyncio.CancelledError:
            # 客户端取消
            logger.info("客户端取消连接")
            yield format_sse_event({
                "type": "warning",
                "message": "连接已取消"
            })

        except HTTPException as e:
            # HTTP异常
            yield format_sse_event({
                "type": "error",
                "code": e.status_code,
                "message": e.detail
            })

        except Exception as e:
            # 未知异常
            logger.error(f"研究失败: {e}", exc_info=True)
            yield format_sse_event({
                "type": "error",
                "error": str(e),
                "traceback": traceback.format_exc()
            })

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )
```

### 5.2 前端自动重连

```typescript
class SSEClient {
  private eventSource: EventSource | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000; // 初始延迟1秒

  connect(url: string, onMessage: (event: any) => void) {
    this.eventSource = new EventSource(url);

    this.eventSource.onmessage = (event) => {
      this.reconnectAttempts = 0; // 重置重连次数
      const data = JSON.parse(event.data);
      onMessage(data);
    };

    this.eventSource.onerror = (error) => {
      console.error('SSE错误:', error);
      this.eventSource?.close();

      // 自动重连
      if (this.reconnectAttempts < this.maxReconnectAttempts) {
        const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts);
        console.log(`${delay}ms后重连...`);

        setTimeout(() => {
          this.reconnectAttempts++;
          this.connect(url, onMessage);
        }, delay);
      } else {
        console.error('达到最大重连次数，停止重连');
      }
    };
  }

  disconnect() {
    this.eventSource?.close();
    this.eventSource = null;
  }
}

// 使用
const client = new SSEClient();
client.connect('http://localhost:8000/research/stream', (event) => {
  console.log('收到事件:', event);
});
```

***

## 6. 性能优化

### 6.1 心跳保持连接

长时间无消息时，连接可能被中间代理关闭。定期发送心跳：

```python
async def event_generator():
    last_heartbeat = time.time()
    heartbeat_interval = 30  # 30秒发送一次心跳

    while True:
        # 尝试获取事件
        try:
            event = await asyncio.wait_for(
                event_queue.get(),
                timeout=heartbeat_interval
            )
            yield format_sse_event(event)
            last_heartbeat = time.time()

        except asyncio.TimeoutError:
            # 超时，发送心跳
            if time.time() - last_heartbeat > heartbeat_interval:
                yield ": heartbeat\n\n"  # 注释行，客户端忽略
                last_heartbeat = time.time()
```

### 6.2 批量发送

对于高频事件（如日志），可以批量发送：

```python
async def event_generator():
    batch = []
    batch_size = 10
    batch_timeout = 0.5  # 500ms

    while True:
        try:
            event = await asyncio.wait_for(
                event_queue.get(),
                timeout=batch_timeout
            )
            batch.append(event)

            # 达到批量大小，立即发送
            if len(batch) >= batch_size:
                yield format_sse_event({
                    "type": "batch",
                    "events": batch
                })
                batch = []

        except asyncio.TimeoutError:
            # 超时，发送现有批次
            if batch:
                yield format_sse_event({
                    "type": "batch",
                    "events": batch
                })
                batch = []
```

### 6.3 Nginx配置

生产环境使用Nginx时，需要禁用缓冲：

```nginx
location /research/stream {
    proxy_pass http://backend;

    # 禁用缓冲
    proxy_buffering off;
    proxy_cache off;

    # 保持连接
    proxy_http_version 1.1;
    proxy_set_header Connection "";

    # SSE超时设置
    proxy_read_timeout 3600s;  # 1小时
    proxy_send_timeout 3600s;

    # 禁用缓存
    add_header Cache-Control "no-cache, no-store, must-revalidate";
    add_header X-Accel-Buffering "no";
}
```

***

## 7. 监控与调试

### 7.1 连接监控

```python
from collections import defaultdict
import threading

class SSEConnectionMonitor:
    """SSE连接监控"""

    def __init__(self):
        self.active_connections = defaultdict(int)
        self.lock = threading.Lock()

    def on_connect(self, session_id: str):
        with self.lock:
            self.active_connections[session_id] += 1
            logger.info(f"SSE连接: {session_id}, 总连接数: {sum(self.active_connections.values())}")

    def on_disconnect(self, session_id: str):
        with self.lock:
            self.active_connections[session_id] -= 1
            if self.active_connections[session_id] <= 0:
                del self.active_connections[session_id]
            logger.info(f"SSE断开: {session_id}, 总连接数: {sum(self.active_connections.values())}")

    def get_metrics(self) -> Dict[str, int]:
        with self.lock:
            return {
                "total_connections": sum(self.active_connections.values()),
                "unique_sessions": len(self.active_connections)
            }

# 全局实例
monitor = SSEConnectionMonitor()

# 在路由中使用
@router.post("/stream")
async def stream_research(request: ResearchRequest):
    monitor.on_connect(request.session_id)

    async def event_generator():
        try:
            # 研究逻辑
            pass
        finally:
            monitor.on_disconnect(request.session_id)

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### 7.2 事件日志

```python
class SSEEventLogger:
    """SSE事件日志"""

    def __init__(self):
        self.event_counts = defaultdict(int)

    def log_event(self, event_type: str, session_id: str, data: Dict):
        self.event_counts[event_type] += 1

        logger.info(
            "SSE事件",
            extra={
                "event_type": event_type,
                "session_id": session_id,
                "event_count": self.event_counts[event_type],
                "data_size": len(json.dumps(data))
            }
        )

# 使用
event_logger = SSEEventLogger()

async def event_generator():
    async for event in service.research(...):
        event_logger.log_event(
            event["type"],
            session_id,
            event
        )
        yield format_sse_event(event)
```

### 7.3 Chrome DevTools调试

**查看SSE连接&#x20;**：

1. 打开Chrome DevTools (F12)

2. 切换到 Network 标签

3) 筛选 EventStream 类型

4) 点击连接查看详细信息

**查看实时消息&#x20;**：

* Messages 标签显示所有接收的SSE事件

* 可以看到每个事件的类型和数据

***

## 8. 最佳实践

### 8.1 事件设计原则

**DO&#x20;**:

* 事件类型明确（用常量定义）

* 数据结构一致（包含timestamp）

* 错误信息详细（包含traceback）

* 进度信息完整（百分比、当前步骤）

**DON'T&#x20;**:

* 频繁发送大量数据（批量发送）

* 事件类型字符串硬编码

* 忽略错误处理

* 无限重连（设置最大次数）

### 8.2 安全考虑

```python
from fastapi import Header, HTTPException

@router.post("/stream")
async def stream_research(
    request: ResearchRequest,
    authorization: str = Header(None)
):
    # 1. 验证Token
    if not authorization or not authorization.startswith("Bearer "):
        raise HTTPException(401, "未授权")

    token = authorization[7:]
    user = verify_token(token)
    if not user:
        raise HTTPException(401, "Token无效")

    # 2. 限流检查
    if not rate_limiter.check(user.id):
        raise HTTPException(429, "请求过于频繁")

    # 3. 会话所有权验证
    session = get_session(request.session_id)
    if session.user_id != user.id:
        raise HTTPException(403, "无权访问此会话")

    # 执行研究
    async def event_generator():
        # ...
        pass

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

### 8.3 资源清理

```python
import weakref

class SSEConnectionManager:
    """SSE连接管理器"""

    def __init__(self):
        self.connections = weakref.WeakValueDictionary()

    async def add_connection(self, session_id: str, queue: asyncio.Queue):
        self.connections[session_id] = queue

    async def cancel_connection(self, session_id: str):
        """取消特定会话的连接"""
        if session_id in self.connections:
            queue = self.connections[session_id]
            # 发送取消事件
            await queue.put({
                "type": "cancelled",
                "message": "会话已被取消"
            })
            del self.connections[session_id]

    async def cleanup_stale_connections(self):
        """清理过期连接"""
        # WeakValueDictionary会自动清理
        pass

# 使用
manager = SSEConnectionManager()

@router.post("/stream")
async def stream_research(request: ResearchRequest):
    event_queue = asyncio.Queue()
    await manager.add_connection(request.session_id, event_queue)

    # ...

@router.post("/{session_id}/cancel")
async def cancel_research(session_id: str):
    await manager.cancel_connection(session_id)
    return {"message": "已取消"}
```

***

## 9. 总结

### 9.1 核心要点

1. **SSE vs WebSocket**

   * 单向推送用SSE（更简单）

   * 双向通信用WebSocket

2. **后端实现**

   * `StreamingResponse` + `AsyncGenerator`

   * `asyncio.Queue` 解耦Agent和推送

   * 后台任务执行，主协程推送

3) **前端实现**

   * `EventSource` API（简单）

   * `Fetch + ReadableStream` （灵活）

   * 自动重连机制

4) **性能优化**

   * 心跳保持连接

   * 批量发送高频事件

   * Nginx禁用缓冲

5. **监控调试**

   * 连接数统计

   * 事件日志记录

   * Chrome DevTools

### 9.2 应用场景

在本项目中，SSE用于：

* 深度研究进度实时推送

* Agent执行状态更新

* 搜索结果流式展示

* 图表生成实时预览

* 最终报告章节逐段呈现

### 9.3 文件位置汇总

**后端核心文件&#x20;**：

* `/backend/app/router/research_router.py` (行50-120) - SSE端点

* `/backend/app/service/deep_research_v2/service.py` (行150-300) - 异步生成器

* `/backend/app/service/deep_research_v2/agents/base.py` (行80-100) - 事件推送

**前端核心文件&#x20;**：

* `/frontend/src/pages/chat/index.tsx` (行200-350) - EventSource接收

* `/frontend/src/api/session.ts` - API调用封装

***

**<span style="color: rgb(36,91,219); background-color: rgba(255,246,122,0.8)">SSE是实时AI应用的核心，掌握它能显著提升用户体验！</span>**

