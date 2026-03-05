# agent(msg) 执行机制详解

## 概述

`agent(msg)` 是 AgentScope 中 Agent 的**可调用接口**，它调用的是 `AgentBase.__call__` 方法，而该方法内部调用 `reply()` 方法。`reply()` 是 ReAct 模式的**核心入口**，实现了推理-行动（Reasoning-Acting）循环。

---

## 一、调用链路

```
agent(msg)
    │
    ▼
AgentBase.__call__(msg)
    │
    ├── 1. 生成唯一 reply_id
    ├── 2. 调用 self.reply(msg)  ──────────────────┐
    │                                               │
    │   ┌───────────────────────────────────────────┘
    │   ▼
    │   CoPawAgent.reply() / ReActAgent.reply()
    │   │
    │   ├── 1. 记录输入消息到 memory
    │   ├── 2. 长期记忆检索 (可选)
    │   ├── 3. 知识库检索 (可选)
    │   │
    │   ├── 4. ReAct 循环 (max_iters 次)
    │   │     │
    │   │     ├── _reasoning() → 调用 LLM
    │   │     │     └── 生成 tool_use 或 text
    │   │     │
    │   │     ├── _acting() → 执行工具
    │   │     │     └── 获取 tool_result
    │   │     │
    │   │     └── 判断是否继续循环
    │   │
    │   └── 5. 返回最终回复
    │
    ▼
返回 Msg 对象
```

---

## 二、核心代码

### 2.1 `AgentBase.__call__` (入口)

```python
# agentscope/agent/_agent_base.py:448-467
async def __call__(self, *args: Any, **kwargs: Any) -> Msg:
    """Call the reply function with the given arguments."""
    self._reply_id = shortuuid.uuid()  # 生成唯一ID

    reply_msg: Msg | None = None
    try:
        self._reply_task = asyncio.current_task()
        reply_msg = await self.reply(*args, **kwargs)  # 调用 reply()
    except asyncio.CancelledError:
        reply_msg = await self.handle_interrupt(*args, **kwargs)  # 处理中断
    finally:
        if reply_msg:
            await self._broadcast_to_subscribers(reply_msg)
    return reply_msg
```

### 2.2 `ReActAgent.reply` (ReAct 循环)

```python
# agentscope/agent/_react_agent.py:376-537
async def reply(
    self,
    msg: Msg | list[Msg] | None = None,
    structured_model: Type[BaseModel] | None = None,
) -> Msg:
    # 1. 记录输入消息到 memory
    await self.memory.add(msg)

    # 2. 长期记忆检索 (可选)
    await self._retrieve_from_long_term_memory(msg)

    # 3. 知识库检索 (可选)
    await self._retrieve_from_knowledge(msg)

    # 4. ReAct 循环
    for _ in range(self.max_iters):
        # 4.1 内存压缩检查
        await self._compress_memory_if_needed()

        # 4.2 推理 (调用 LLM)
        msg_reasoning = await self._reasoning(tool_choice)

        # 4.3 行动 (执行工具)
        futures = [self._acting(tool_call) for tool_call in tool_calls]
        structured_outputs = await asyncio.gather(*futures)

        # 4.4 检查退出条件
        if not msg_reasoning.has_content_blocks("tool_use"):
            # 没有工具调用，返回文本回复
            reply_msg = msg_reasoning
            break

    # 5. 返回最终回复
    return reply_msg
```

### 2.3 `CoPawAgent.reply` (扩展)

```python
# src/copaw/agents/react_agent.py:526-557
async def reply(
    self,
    msg: Msg | list[Msg] | None = None,
    structured_model: Type[BaseModel] | None = None,
) -> Msg:
    # 1. 处理文件和媒体块
    if msg is not None:
        await process_file_and_media_blocks_in_message(msg)

    # 2. 检查是否是系统命令
    query = last_msg.get_text_content()
    if self.command_handler.is_command(query):
        # 处理命令 (/compact, /new, /clear 等)
        msg = await self.command_handler.handle_command(query)
        await self.print(msg)
        return msg

    # 3. 正常消息处理 (调用父类 reply)
    return await super().reply(msg=msg, structured_model=structured_model)
```

---

## 三、语法示例

### 3.1 基本调用

```python
# 创建 Agent
agent = CoPawAgent(
    max_iters=50,
    max_input_length=131072,
)

# 创建消息
msg = Msg(
    name="user",
    content="请帮我写一个 hello world 程序",
    role="user"
)

# 调用 agent (触发 ReAct 循环)
result = await agent(msg)
print(result.content)
```

### 3.2 消息列表输入

```python
# 多轮对话
msgs = [
    Msg("user", "请帮我写一个程序", "user"),
    Msg("assistant", "好的，请告诉我什么程序?", "assistant"),
    Msg("user", "hello world", "user"),
]

result = await agent(msgs)
```

### 3.3 流式调用 (CoPaw 使用方式)

```python
# Runner 中的调用方式
async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),  # ← 这里 agent(msgs) 返回协程
):
    yield msg, last
```

### 3.4 带结构化输出

```python
from pydantic import BaseModel

class ResponseSchema(BaseModel):
    title: str
    content: str
    tags: list[str]

# 调用时指定结构化模型
result = await agent(msg, structured_model=ResponseSchema)
# result.metadata 会包含结构化数据
```

---

## 四、ReAct 循环详解

### 4.1 循环流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    ReAct Loop (max_iters)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 1. Memory Add                                           │  │
│  │    await self.memory.add(msg)                           │ ──────────────────────────────────────────────────────────┘  │
│                           │
│  └ │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. Reasoning (_reasoning)                               │  │
│  │    - 格式化消息 (formatter)                               │  │
│  │    - 调用 LLM (model)                                   │  │
│  │    - 返回: Msg (含 tool_use 或 text)                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 3. Acting (_acting)                                     │  │
│  │    - 解析 tool_use 块                                   │  │
│  │    - 调用 toolkit.call_tool_function()                  │  │
│  │    - 获取 tool_result                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 4. Check Exit Condition                                 │  │
│  │    - 有 tool_use? → 继续循环                            │  │
│  │    - 无 tool_use? → 退出，返回文本                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│           ┌───────────────┴───────────────┐                    │
│           ▼                               ▼                    │
│    继续循环                          返回 Msg                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 `_reasoning` 方法

```python
async def _reasoning(self, tool_choice) -> Msg:
    # 1. 格式化消息
    prompt = await self.formatter.format([
        Msg("system", self.sys_prompt, "system"),
        *await self.memory.get_memory(),
    ])

    # 2. 调用 LLM
    res = await self.model(
        prompt,
        tools=self.toolkit.get_json_schemas(),
        tool_choice=tool_choice,
    )

    # 3. 处理流式输出
    msg = Msg(name=self.name, content=[], role="assistant")
    async for content_chunk in res:
        msg.content = content_chunk.content
        await self.print(msg, False)  # 流式打印

    # 4. 打印最终消息
    await self.print(msg, True)
    return msg
```

### 4.3 `_acting` 方法

```python
async def _acting(self, tool_call: ToolUseBlock) -> dict | None:
    # 1. 执行工具
    tool_res = await self.toolkit.call_tool_function(tool_call)

    # 2. 处理流式工具结果
    tool_res_msg = Msg("system", content=[...], role="system")
    async for chunk in tool_res:
        tool_res_msg.content[0]["output"] = chunk.content
        await self.print(tool_res_msg, chunk.is_last)

    # 3. 记录到 memory
    await self.memory.add(tool_res_msg)
    return None
```

---

## 五、CoPawAgent 的扩展

### 5.1 扩展点

| 扩展点 | 说明 |
|--------|------|
| **文件/媒体处理** | `process_file_and_media_blocks_in_message()` |
| **命令处理** | `command_handler.is_command()` - 检测 `/compact` 等命令 |
| **内存管理** | `MemoryManager` - 长期记忆 + 自动压缩 |
| **Hook 机制** | `BootstrapHook`, `MemoryCompactionHook` |
| **MCP 集成** | 动态注册 MCP 工具 |

### 5.2 命令拦截

```python
# CoPawAgent.reply() 中
query = last_msg.get_text_content()
if self.command_handler.is_command(query):
    # 不进入 ReAct 循环，直接处理命令
    msg = await self.command_handler.handle_command(query)
    return msg
```

支持的命令：
- `/compact` - 手动压缩内存
- `/new` - 开始新对话
- `/clear` - 清除历史
- `/history` - 查看历史

---

## 六、总结

| 层级 | 方法 | 职责 |
|------|------|------|
| **入口** | `AgentBase.__call__` | 生成 reply_id，调用 reply，处理中断 |
| **主逻辑** | `ReActAgent.reply` | ReAct 循环入口，管理整体流程 |
| **扩展** | `CoPawAgent.reply` | 文件处理、命令拦截 |
| **推理** | `_reasoning` | 调用 LLM 生成 tool_use 或 text |
| **行动** | `_acting` | 执行工具，获取 tool_result |

**核心流程**：
```
agent(msg) → __call__ → reply() → for _ in range(max_iters):
    → _reasoning() → LLM → tool_use?
    → _acting() → 工具执行 → tool_result
    → 检查退出条件
→ 返回最终 Msg
```

---

## 七、Pipeline 调用链详解

Pipeline 管道函数接收 **Agent 列表** 和 **消息**，然后依次调用每个 Agent。

### 7.1 sequential_pipeline 调用链

```python
# agentscope/pipeline/_functional.py
async def sequential_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
) -> Msg | list[Msg] | None:
    for agent in agents:
        msg = await agent(msg)  # ← 每次调用 agent(msg)
    return msg
```

**完整调用链**：
```
sequential_pipeline([agent1, agent2, agent3], msg)
    │
    ├── agent1(msg)
    │     │
    │     └── AgentBase.__call__(msg)
    │           │
    │           └── ReActAgent.reply(msg)
    │                 │
    │                 └── ReAct Loop (_reasoning → _acting)
    │
    ├── agent2(output_of_agent1)  ← 上一个输出作为输入
    │     │
    │     └── AgentBase.__call__(output_of_agent1)
    │           │
    │           └── ReActAgent.reply(output_of_agent1)
    │
    └── agent3(output_of_agent2)  ← 上一个输出作为输入
          │
          └── AgentBase.__call__(output_of_agent2)
                │
                └── ReActAgent.reply(output_of_agent2)

返回: agent3 的输出
```

**示例**：
```python
# 研究 → 总结 → 审阅 工作流
researcher = ReActAgent(name="researcher")
summarizer = ReActAgent(name="summarizer")
reviewer = ReActAgent(name="reviewer")

msg = Msg("user", "分析这篇论文创新点", "user")

result = await sequential_pipeline(
    [researcher, summarizer, reviewer],
    msg
)
# researcher 输出 → summarizer 输入 → reviewer 输入 → result
```

---

### 7.2 fanout_pipeline 调用链

```python
# agentscope/pipeline/_functional.py
async def fanout_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
    enable_gather: bool = True,
    **kwargs: Any,
) -> list[Msg]:
    if enable_gather:
        # 并发模式
        tasks = [
            asyncio.create_task(agent(deepcopy(msg), **kwargs))
            for agent in agents
        ]
        return await asyncio.gather(*tasks)
    else:
        # 串行模式
        return [await agent(deepcopy(msg), **kwargs) for agent in agents]
```

**并发模式调用链 (enable_gather=True)**：
```
fanout_pipeline([agent1, agent2, agent3], msg, enable_gather=True)
    │
    ├── deepcopy(msg) ──▶ asyncio.create_task(agent1(msg))
    ├── deepcopy(msg) ──▶ asyncio.create_task(agent2(msg))
    └── deepcopy(msg) ──▶ asyncio.create_task(agent3(msg))
                              │
                              ▼
                    asyncio.gather(*tasks)
                              │
    ┌───────────────────────────┼───────────────────────────┐
    │                           │                           │
    ▼                           ▼                           ▼
[agent1_result,          agent2_result,              agent3_result]

返回: [Msg1, Msg2, Msg3]
```

**串行模式调用链 (enable_gather=False)**：
```
fanout_pipeline([agent1, agent2, agent3], msg, enable_gather=False)
    │
    ├── deepcopy(msg) ──▶ agent1(msg) ──▶ result1
    │
    ├── deepcopy(msg) ──▶ agent2(msg) ──▶ result2
    │
    └── deepcopy(msg) ──▶ agent3(msg) ──▶ result3

返回: [result1, result2, result3]
```

**示例**：
```python
# 并发：同时获取多源信息
weather = ReActAgent(name="weather")
stocks = ReActAgent(name="stocks")
news = ReActAgent(name="news")

msg = Msg("user", "今天怎么样?", "user")

# 并发执行
results = await fanout_pipeline([weather, stocks, news], msg)
# results = [weather_result, stocks_result, news_result]

# 串行执行 (逐个等待)
results = await fanout_pipeline([weather, stocks, news], msg, enable_gather=False)
```

---

### 7.3 stream_printing_messages 调用链

这是 **CoPaw 实际使用** 的管道，用于流式输出。

```python
# agentscope/pipeline/_functional.py
async def stream_printing_messages(
    agents: list[AgentBase],
    coroutine_task: Coroutine,
    queue: asyncio.Queue | None = None,
    end_signal: str = "[END]",
    yield_speech: bool = False,
) -> AsyncGenerator[...]:
    # 1. 创建消息队列
    queue = queue or asyncio.Queue()

    # 2. 启用 agent 的消息收集
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
        yield printing_msg

    # 6. 检查异常
    exception = task.exception()
    if exception is not None:
        raise exception
```

**完整调用链**：
```
stream_printing_messages(agents=[agent], coroutine_task=agent(msgs))
    │
    ├── 1. 创建 asyncio.Queue()
    │
    ├── 2. agent.set_msg_queue_enabled(True, queue)
    │     └── 启用 agent 内部的消息队列收集
    │
    ├── 3. asyncio.create_task(agent(msgs))
    │     │
    │     └── AgentBase.__call__(msgs)
    │           │
    │           └── ReActAgent.reply(msgs)
    │                 │
    │                 └── ReAct Loop:
    │                       │
    │                       ├── _reasoning() → await self.print(msg, ...)
    │                       │      │
    │                       │      └── queue.put((msg, is_last, speech))
    │                       │
    │                       └── _acting() → await self.print(tool_result, ...)
    │                              │
    │                              └── queue.put((tool_result, is_last, speech))
    │
    ├── 4. while True: msg = await queue.get()
    │     └── 消费者从队列获取消息，yield 出去
    │
    └── 5. 收到 end_signal，退出循环

yield: 实时流式消息 (Msg, is_last_chunk)
```

**CoPaw 中的使用**：
```python
# src/copaw/app/runner/runner.py
agent = CoPawAgent(...)

async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),  # ← 返回协程
):
    yield msg, last  # ← 实时流式输出
```

---

### 7.4 stream_printing_messages 任务执行详解

下面详细说明 `stream_printing_messages` 是如何执行 task 的：

#### 第一步：创建队列并启用消息收集

```python
# 1. 创建队列
queue = asyncio.Queue()

# 2. 启用 agent 的消息队列
agent.set_msg_queue_enabled(True, queue)
```

`set_msg_queue_enabled` 的实现：
```python
# agentscope/agent/_agent_base.py:712-736
def set_msg_queue_enabled(self, enabled: bool, queue: Queue | None = None):
    if enabled:
        if queue is None:
            if self.msg_queue is None:
                self.msg_queue = asyncio.Queue(maxsize=100)
        else:
            self.msg_queue = queue
    else:
        self.msg_queue = None
    self._disable_msg_queue = not enabled
```

#### 第二步：创建异步任务

```python
# 注意：传入的是协程，不是已执行的task
coroutine_task = agent(msgs)  # ← 这是一个协程对象
task = asyncio.create_task(coroutine_task)  # ← 创建task并开始执行
```

#### 第三步：Agent 执行过程中调用 print

在 `ReActAgent._reasoning()` 和 `_acting()` 中，会调用 `await self.print(msg, ...)`：

```python
# agentscope/agent/_agent_base.py:205-228
async def print(self, msg: Msg, last: bool = True, speech: AudioBlock | None = None):
    # 如果启用了消息队列，将消息放入队列
    if not self._disable_msg_queue:
        await self.msg_queue.put((deepcopy(msg), last, speech))
        await asyncio.sleep(0)  # 让出控制权给消费者
    
    # ... 继续控制台输出 ...
```

#### 第四步：消费者从队列获取消息

```python
# stream_printing_messages 中的消费者循环
while True:
    printing_msg = await queue.get()  # 阻塞等待消息
    
    if isinstance(printing_msg, str) and printing_msg == end_signal:
        break
    
    # 解包消息
    msg, last, speech = printing_msg
    yield msg, last  # yield 给调用者
```

#### 完整执行时序图

```
时间轴 →
====================================================================

stream_printing_messages                      Agent 执行
─────────────────────────────────────────────────────────────────────
                                             
1. 创建 queue                                  (空闲)
                                             
2. set_msg_queue_enabled(True, queue)        
   ─────────────────────────────────────────▶ 启用消息队列
                                             
3. create_task(agent(msgs))                  
   ─────────────────────────────────────────▶ 开始执行 _reasoning()
                                             
                                             
   (空闲)                                      4. LLM 调用完成
                                             
   (空闲)                                      5. await self.print(msg)
                                              ◀───────────────────────────────────────
                                              │ queue.put((msg, last, speech))
                                              │
6. await queue.get() ◀───────────────────────┘
   (获取到消息)
   
7. yield msg, last
   (流式输出)
                                             
   (等待下一个消息)                            8. 继续执行 / 工具调用
                                             
   (等待下一个消息)                            9. await self.print(tool_result)
                                              ◀───────────────────────────────────────
                                              │ queue.put((result, last, speech))
                                              │
10. await queue.get() ◀──────────────────────┘
    (获取到工具结果)
    
11. yield tool_result
    (流式输出)
    
    ... 继续循环 ...
                                             
    (等待)                                    N. 执行完成
    
12. queue.get() ← "[END]" 信号              N+1. task.done() 触发 callback
    (获取结束信号)                            │ queue.put_nowait("[END]")
    │
13. break 退出循环
```

#### 关键代码路径

```python
# 1. stream_printing_messages 接收协程
coroutine_task = agent(msgs)  # agent() 返回协程，未执行
task = asyncio.create_task(coroutine_task)  # 创建task，开始执行

# 2. Agent 执行过程中调用 print
# 在 _reasoning() 中:
await self.print(msg, False)  # 流式输出中间结果
await self.print(msg, True)   # 流式输出最后一个chunk

# 3. print() 方法将消息放入队列
async def print(self, msg, last, speech):
    if not self._disable_msg_queue:
        await self.msg_queue.put((deepcopy(msg), last, speech))
        await asyncio.sleep(0)  # 让出控制权

# 4. stream_printing_messages 从队列获取并yield
while True:
    printing_msg = await queue.get()
    if printing_msg == end_signal:
        break
    yield printing_msg
```

---

### 7.5 Pipeline 与 Agent 的交互关系

| Pipeline | Agent 调用方式 | 输入输出 |
|----------|---------------|----------|
| `sequential_pipeline` | `await agent(msg)` | 上一个 agent 输出作为下一个输入 |
| `fanout_pipeline` | `await agent(deepcopy(msg))` | 同一消息并行/串行分发 |
| `stream_printing_messages` | `agent(msg)` 返回协程 | 异步执行，队列收集中间消息 |

---

### 7.6 完整调用链对比图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Sequential Pipeline                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  msg ──▶ Agent1.reply() ──▶ output1 ──▶ Agent2.reply() ──▶ output2  │
│                                         ──▶ Agent3.reply() ──▶ final  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         Fanout Pipeline (并发)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│         ┌───▶ Agent1.reply(msg) ──┐                                    │
│         │                          │                                    │
│  msg ───┼───▶ Agent2.reply(msg) ──┼───▶ [r1, r2, r3]               │
│         │                          │                                    │
│         └───▶ Agent3.reply(msg) ──┘                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    Stream Printing Pipeline (CoPaw)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  agent(msgs) ──▶ __call__ ──▶ reply() ──▶ ReAct Loop                   │
│       │                                    │                             │
│       │                                    ├── _reasoning() ──▶ print │
│       │                                    │         │                  │
│       │                                    │         ▼                  │
│       │                                    │    queue.put()            │
│       │                                    │         │                  │
│       │                                    ├── _acting() ──▶ print     │
│       │                                    │         │                  │
│       │                                    │         ▼                  │
│       │                                    │    queue.put()            │
│       │                                    │         │                  │
│       ▼                                    │         ▼                  │
│  coroutine_task                      while queue.get() ──▶ yield msg   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
agent(msg) → __call__ → reply() → for _ in range(max_iters):
    → _reasoning() → LLM → tool_use?
    → _acting() → 工具执行 → tool_result
    → 检查退出条件
→ 返回最终 Msg
```

---

## 七、Pipeline 调用链详解

Pipeline 管道函数接收 **Agent 列表** 和 **消息**，然后依次调用每个 Agent。

### 7.1 sequential_pipeline 调用链

```python
# agentscope/pipeline/_functional.py
async def sequential_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
) -> Msg | list[Msg] | None:
    for agent in agents:
        msg = await agent(msg)  # ← 每次调用 agent(msg)
    return msg
```

**完整调用链**：
```
sequential_pipeline([agent1, agent2, agent3], msg)
    │
    ├── agent1(msg)
    │     │
    │     └── AgentBase.__call__(msg)
    │           │
    │           └── ReActAgent.reply(msg)
    │                 │
    │                 └── ReAct Loop (_reasoning → _acting)
    │
    ├── agent2(output_of_agent1)  ← 上一个输出作为输入
    │     │
    │     └── AgentBase.__call__(output_of_agent1)
    │           │
    │           └── ReActAgent.reply(output_of_agent1)
    │
    └── agent3(output_of_agent2)  ← 上一个输出作为输入
          │
          └── AgentBase.__call__(output_of_agent2)
                │
                └── ReActAgent.reply(output_of_agent2)

返回: agent3 的输出
```

**示例**：
```python
# 研究 → 总结 → 审阅 工作流
researcher = ReActAgent(name="researcher")
summarizer = ReActAgent(name="summarizer")
reviewer = ReActAgent(name="reviewer")

msg = Msg("user", "分析这篇论文创新点", "user")

result = await sequential_pipeline(
    [researcher, summarizer, reviewer],
    msg
)
# researcher 输出 → summarizer 输入 → reviewer 输入 → result
```

---

### 7.2 fanout_pipeline 调用链

```python
# agentscope/pipeline/_functional.py
async def fanout_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
    enable_gather: bool = True,
    **kwargs: Any,
) -> list[Msg]:
    if enable_gather:
        # 并发模式
        tasks = [
            asyncio.create_task(agent(deepcopy(msg), **kwargs))
            for agent in agents
        ]
        return await asyncio.gather(*tasks)
    else:
        # 串行模式
        return [await agent(deepcopy(msg), **kwargs) for agent in agents]
```

**并发模式调用链 (enable_gather=True)**：
```
fanout_pipeline([agent1, agent2, agent3], msg, enable_gather=True)
    │
    ├── deepcopy(msg) ──▶ asyncio.create_task(agent1(msg))
    ├── deepcopy(msg) ──▶ asyncio.create_task(agent2(msg))
    └── deepcopy(msg) ──▶ asyncio.create_task(agent3(msg))
                              │
                              ▼
                    asyncio.gather(*tasks)
                              │
    ┌───────────────────────────┼───────────────────────────┐
    │                           │                           │
    ▼                           ▼                           ▼
[agent1_result,          agent2_result,              agent3_result]

返回: [Msg1, Msg2, Msg3]
```

**串行模式调用链 (enable_gather=False)**：
```
fanout_pipeline([agent1, agent2, agent3], msg, enable_gather=False)
    │
    ├── deepcopy(msg) ──▶ agent1(msg) ──▶ result1
    │
    ├── deepcopy(msg) ──▶ agent2(msg) ──▶ result2
    │
    └── deepcopy(msg) ──▶ agent3(msg) ──▶ result3

返回: [result1, result2, result3]
```

**示例**：
```python
# 并发：同时获取多源信息
weather = ReActAgent(name="weather")
stocks = ReActAgent(name="stocks")
news = ReActAgent(name="news")

msg = Msg("user", "今天怎么样?", "user")

# 并发执行
results = await fanout_pipeline([weather, stocks, news], msg)
# results = [weather_result, stocks_result, news_result]

# 串行执行 (逐个等待)
results = await fanout_pipeline([weather, stocks, news], msg, enable_gather=False)
```

---

### 7.3 stream_printing_messages 调用链

这是 **CoPaw 实际使用** 的管道，用于流式输出。

```python
# agentscope/pipeline/_functional.py
async def stream_printing_messages(
    agents: list[AgentBase],
    coroutine_task: Coroutine,
    queue: asyncio.Queue | None = None,
    end_signal: str = "[END]",
    yield_speech: bool = False,
) -> AsyncGenerator[...]:
    # 1. 创建消息队列
    queue = queue or asyncio.Queue()

    # 2. 启用 agent 的消息收集
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
        yield printing_msg

    # 6. 检查异常
    exception = task.exception()
    if exception is not None:
        raise exception
```

**完整调用链**：
```
stream_printing_messages(agents=[agent], coroutine_task=agent(msgs))
    │
    ├── 1. 创建 asyncio.Queue()
    │
    ├── 2. agent.set_msg_queue_enabled(True, queue)
    │     └── 启用 agent 内部的消息队列收集
    │
    ├── 3. asyncio.create_task(agent(msgs))
    │     │
    │     └── AgentBase.__call__(msgs)
    │           │
    │           └── ReActAgent.reply(msgs)
    │                 │
    │                 └── ReAct Loop:
    │                       │
    │                       ├── _reasoning() → await self.print(msg, ...)
    │                       │                      │
    │                       │                      └── queue.put((msg, is_last, speech))
    │                       │
    │                       └── _acting() → await self.print(tool_result, ...)
    │                                              │
    │                                              └── queue.put((tool_result, is_last, speech))
    │
    ├── 4. while True: msg = await queue.get()
    │     └── 消费者从队列获取消息，yield 出去
    │
    └── 5. 收到 end_signal，退出循环

yield: 实时流式消息 (Msg, is_last_chunk)
```

**CoPaw 中的使用**：
```python
# src/copaw/app/runner/runner.py
agent = CoPawAgent(...)

async for msg, last in stream_printing_messages(
    agents=[agent],
    coroutine_task=agent(msgs),  # ← 返回协程
):
    yield msg, last  # ← 实时流式输出
```

---

### 7.4 Pipeline 与 Agent 的交互关系

| Pipeline | Agent 调用方式 | 输入输出 |
|----------|---------------|----------|
| `sequential_pipeline` | `await agent(msg)` | 上一个 agent 输出作为下一个输入 |
| `fanout_pipeline` | `await agent(deepcopy(msg))` | 同一消息并行/串行分发 |
| `stream_printing_messages` | `agent(msg)` 返回协程 | 异步执行，队列收集中间消息 |

---

### 7.5 完整调用链对比图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Sequential Pipeline                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  msg ──▶ Agent1.reply() ──▶ output1 ──▶ Agent2.reply() ──▶ output2  │
│                                         ──▶ Agent3.reply() ──▶ final  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         Fanout Pipeline (并发)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│         ┌───▶ Agent1.reply(msg) ──┐                                    │
│         │                          │                                    │
│  msg ───┼───▶ Agent2.reply(msg) ──┼───▶ [r1, r2, r3]               │
│         │                          │                                    │
│         └───▶ Agent3.reply(msg) ──┘                                    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    Stream Printing Pipeline (CoPaw)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  agent(msgs) ──▶ __call__ ──▶ reply() ──▶ ReAct Loop                   │
│       │                                    │                             │
│       │                                    ├── _reasoning() ──▶ print │
│       │                                    │         │                  │
│       │                                    │         ▼                  │
│       │                                    │    queue.put()            │
│       │                                    │         │                  │
│       │                                    ├── _acting() ──▶ print     │
│       │                                    │         │                  │
│       │                                    │         ▼                  │
│       │                                    │    queue.put()            │
│       │                                    │         │                  │
│       ▼                                    │         ▼                  │
│  coroutine_task                      while queue.get() ──▶ yield msg   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```
