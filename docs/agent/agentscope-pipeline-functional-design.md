# AgentScope Pipeline 函数式模块设计分析

## 概述

`_functional.py` 是 AgentScope Pipeline 模块的**函数式实现层**，提供三个核心异步管道函数，用于多代理工作流的编排。该模块是 CoPaw 实现流式输出的基础设施。

---

## 一、设计思想

### 1.1 核心定位

提供**轻量级、函数式**的管道 API，用于：
- 多代理顺序执行
- 多代理并行分发
- 代理执行过程的流式消息收集

### 1.2 架构哲学

| 原则 | 体现 |
|------|------|
| **异步优先** | 基于 `asyncio` 的并发与流式处理 |
| **函数式接口** | 简单函数调用，无需实例化类 |
| **不可变性** | 使用 `deepcopy` 避免消息共享副作用 |
| **队列驱动** | 通过 `asyncio.Queue` 收集中间消息 |

### 1.3 模块依赖

```python
import asyncio              # 异步并发
from copy import deepcopy   # 消息深拷贝
from typing import ...      # 类型注解

from ..agent import AgentBase    # 代理基类
from ..message import Msg, AudioBlock  # 消息类型
```

---

## 二、核心函数详解

### 2.1 `sequential_pipeline` - 顺序管道

**签名**:
```python
async def sequential_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
) -> Msg | list[Msg] | None
```

**实现逻辑**:
```python
for agent in agents:
    msg = await agent(msg)  # 上一个输出作为下一个输入
return msg
```

**执行流程**:
```
输入消息 (msg)
    │
    ▼
┌─────────────────────────────────────────────┐
│  for agent in agents:                        │
│    msg = await agent(msg)                   │
└─────────────────────────────────────────────┘
    │
    ▼
最终消息 (最后一个agent的输出)
```

**特点**:
| 特性 | 说明 |
|------|------|
| 串行执行 | 每个 agent 依次等待前一个完成 |
| 输出传递 | 上一个 agent 的输出作为下一个的输入 |
| 链式处理 | 适合 提取 → 转换 → 验证 流程 |

**使用示例**:
```python
agent1 = ReActAgent(name="researcher")
agent2 = ReActAgent(name="summarizer")
agent3 = ReActAgent(name="reviewer")

msg = Msg("user", "请分析这篇论文", "user")
result = await sequential_pipeline([agent1, agent2, agent3], msg)
# agent1输出 → agent2输入 → agent3输入 → result
```

---

### 2.2 `fanout_pipeline` - 扇出管道

**签名**:
```python
async def fanout_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
    enable_gather: bool = True,
    **kwargs: Any,
) -> list[Msg]
```

**实现逻辑**:
```python
if enable_gather:
    # 并发执行
    tasks = [asyncio.create_task(agent(deepcopy(msg), **kwargs)) for agent in agents]
    return await asyncio.gather(*tasks)
else:
    # 串行执行
    return [await agent(deepcopy(msg), **kwargs) for agent in agents]
```

**执行流程**:
```
原始消息 (msg)
    │
    ├── deepcopy(msg) ──▶ Agent 1 ──┐
    ├── deepcopy(msg) ──▶ Agent 2 ──┼──▶ [结果1, 结果2, ..., 结果N]
    └── deepcopy(msg) ──▶ Agent N ──┘
```

**关键设计点**:

| 设计点 | 说明 |
|--------|------|
| `deepcopy(msg)` | 每次复制消息，避免代理间状态污染 |
| `enable_gather` | 控制并发/串行执行模式 |
| `**kwargs` | 传递额外参数给每个 agent |

**使用示例**:
```python
# 并发模式：同时获取多源信息
weather_agent = ReActAgent(name="weather")
stock_agent = ReActAgent(name="stock")
news_agent = ReActAgent(name="news")

msg = Msg("user", "今天怎么样?", "user")
results = await fanout_pipeline([weather_agent, stock_agent, news_agent], msg)
# [天气结果, 股票结果, 新闻结果]

# 串行模式：逐个执行
results = await fanout_pipeline(..., enable_gather=False)
```

---

### 2.3 `stream_printing_messages` - 流式消息管道

**签名**:
```python
async def stream_printing_messages(
    agents: list[AgentBase],
    coroutine_task: Coroutine,
    queue: asyncio.Queue | None = None,
    end_signal: str = "[END]",
    yield_speech: bool = False,
) -> AsyncGenerator[
    Tuple[Msg, bool] | Tuple[Msg, bool, AudioBlock | list[AudioBlock] | None],
    None,
]
```

**这是 CoPaw 使用的核心管道**，用于实时展示 AI 推理过程。

**实现逻辑**:
```python
# 1. 创建/复用队列
queue = queue or asyncio.Queue()

# 2. 启用所有 agent 的消息队列收集
for agent in agents:
    agent.set_msg_queue_enabled(True, queue)

# 3. 异步执行任务
task = asyncio.create_task(coroutine_task)

# 4. 设置结束信号
if task.done():
    await queue.put(end_signal)
else:
    task.add_done_callback(lambda _: queue.put_nowait(end_signal))

# 5. 监听队列，yield 消息
while True:
    printing_msg = await queue.get()
    if isinstance(printing_msg, str) and printing_msg == end_signal:
        break
    yield printing_msg  # 或 yield msg, last
```

**核心机制: 消息队列收集**

```
Agent 执行流程
      │
      │ await self.print(msg)  ← 打印消息
      ▼
┌─────────────────┐
│ agent.msg_queue │ ← 内部队列
└────────┬────────┘
         │ put(msg, is_last_chunk, speech)
         ▼
┌─────────────────┐
│ asyncio.Queue  │ ← 共享队列 (本函数创建)
└────────┬────────┘
         │ get()
         ▼
┌─────────────────┐
│ yield msg       │ ← 消费者
└─────────────────┘
```

**Yield 格式**:

| 模式 | 返回类型 | 说明 |
|------|----------|------|
| 默认 | `Tuple[Msg, bool]` | 消息 + 是否最后一个 chunk |
| `yield_speech=True` | `Tuple[Msg, bool, AudioBlock]` | 包含语音数据 |

**关键设计点**:

| 设计点 | 说明 |
|--------|------|
| `set_msg_queue_enabled(True, queue)` | 启用 agent 的消息队列收集功能 |
| `end_signal` | 特殊字符串 `[END]` 标记流式结束 |
| 异常传播 | `task.exception()` 在结束后检查并抛出 |
| 流式 chunk | `bool` 表示是否同一消息的最后一个 chunk |

**使用示例 (CoPaw)**:
```python
# src/copaw/app/runner/runner.py
agent = CoPawAgent(...)

async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),
):
    yield msg, last  # 实时 yield 推理过程
```

---

## 三、数据流图

### 3.1 顺序管道
```
输入
  │
  ▼
┌─────────────────────────┐
│  Agent 1                │
│  await agent(msg)       │──▶ msg1
└─────────────────────────┘
  │
  ▼ (msg1)
┌─────────────────────────┐
│  Agent 2                │
│  await agent(msg1)      │──▶ msg2
└─────────────────────────┘
  │
  ▼ (msg2)
┌─────────────────────────┐
│  Agent N                │
│  await agent(msgN-1)    │──▶ final_msg
└─────────────────────────┘
  │
  ▼
返回 final_msg
```

### 3.2 扇出管道 (并发模式)
```
输入 msg
  │
  ├── deepcopy(msg) ──▶ Agent 1 ──┐
  │                               │
  ├── deepcopy(msg) ──▶ Agent 2 ──┼──▶ asyncio.gather(*tasks)
  │                               │
  └── deepcopy(msg) ──▶ Agent N ──┘
                              │
                              ▼
                    [result1, result2, ..., resultN]
```

### 3.3 流式消息管道
```
┌──────────────────────────────────────────────────────────────────┐
│                         Agent 执行                                │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │ 1. 推理 (Reasoning)                                      │  │
│   │ 2. 行动 (Acting) - 调用工具                              │  │
│   │ 3. 观察 (Observation) - 获取结果                         │  │
│   │                          │                               │  │
│   │                          ▼ await self.print(msg)        │  │
│   └──────────────────────────│───────────────────────────────┘  │
│                              │                                   │
└──────────────────────────────┼───────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  asyncio.Queue      │
                    │  (msg, last, audio)│
                    └──────────┬──────────┘
                               │
                               │ queue.get()
                               ▼
                    ┌─────────────────────┐
                    │ while True:        │
                    │   msg = await...   │
                    │   if msg == END:   │
                    │     break          │
                    │   yield msg, last  │
                    └─────────────────────┘
                               │
                               ▼
                    实时流式输出给前端
```

---

## 四、与 CoPaw 的集成

### 4.1 Runner 中的调用

```python
# src/copaw/app/runner/runner.py:166-170
async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),
):
    yield msg, last
```

**作用**:
1. 启动 CoPawAgent 的 ReAct 循环
2. 实时捕获 agent 的推理过程、工具调用
3. 流式 yield 给前端展示

### 4.2 支持的消息类型

管道收集的消息内容 (`Msg.content`):

| 类型 | 说明 |
|------|------|
| `text` | 文本回复 |
| `thinking` | 推理过程 (ReAct) |
| `tool_use` | 工具调用请求 |
| `tool_result` | 工具执行结果 |
| `image` | 图片 |
| `audio` | 音频 |

---

## 五、总结

| 函数 | 用途 | 核心机制 |
|------|------|----------|
| `sequential_pipeline` | 链式处理 | 串行执行，输出传递 |
| `fanout_pipeline` | 并行处理 | `asyncio.gather` / 串行 |
| `stream_printing_messages` | 流式输出 | **队列收集，实时 yield** |

**设计亮点**:
1. **函数式接口**: 简单直接，无需类实例化
2. **深拷贝**: 避免多代理间状态污染
3. **并发控制**: `enable_gather` 灵活切换并发/串行
4. **流式收集**: 队列机制实现实时推理过程展示

**在 CoPaw 中的角色**:
- `stream_printing_messages` 是 CoPaw 实现**实时流式输出**的核心依赖，让用户能看到 AI 的思考过程。
