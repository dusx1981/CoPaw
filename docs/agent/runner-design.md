# Runner 模块设计分析

## 一、设计思想

### 1.1 核心定位
Runner 是 CoPaw 系统的**查询处理器**，负责协调用户请求 → 代理执行 → 响应的完整流程。基于 [AgentScope Runtime](https://github.com/agentscope-ai/agentscope-runtime) 的 `Runner` 抽象构建。

### 1.2 架构哲学

| 设计原则 | 体现 |
|---------|------|
| **双路径执行** | 命令路径（轻量） vs 代理路径（完整） |
| **会话隔离** | 每个 session/user 独立状态，通过 JSON 文件持久化 |
| **热重载** | MCP 客户端可动态更新，无需重启 |
| **跨平台兼容** | Windows 文件名特殊字符处理 |
| **内存管理** | 集成 MemoryManager 进行上下文压缩 |

---

## 二、实现机制

### 2.1 双路径执行模型

```
用户输入
    │
    ├── 是命令? (/restart, /compact, /new...)
    │   └── 命令路径 → run_command_path() (轻量,无需创建完整Agent)
    │
    └── 普通查询
        └── 代理路径 → AgentRunner.query_handler() (完整ReAct循环)
```

### 2.2 命令路径 (Command Path)
- **轻量化**: 不创建完整 CoPawAgent
- **用途**: `/compact` (内存压缩), `/new` (新对话), `/clear` (清除历史)
- **守护进程命令**: `/restart`, `/status`, `/logs`

### 2.3 代理路径 (Agent Path)
完整执行流程:
```
1. 解析请求 (session_id, user_id, channel)
2. 构建环境上下文 (build_env_context)
3. 获取 MCP 客户端 (热重载)
4. 创建 CoPawAgent (注册工具/MCP/内存管理器)
5. 加载会话状态 (session.load_session_state)
6. 重建系统提示 (agent.rebuild_sys_prompt)
7. 执行 ReAct 循环 (stream_printing_messages)
8. 保存会话状态 (session.save_session_state)
```

### 2.4 会话管理机制

**SafeJSONSession**:
- 继承自 AgentScope 的 `JSONSession`
- 添加 Windows 文件名兼容处理
- 将 `\/:*?"<>|` 替换为 `--`

**状态保存内容**:
- Agent 内存 (对话历史)
- 压缩摘要 (当内存超过阈值时)

---

## 三、核心功能

### 3.1 AgentRunner 类方法

| 方法 | 职责 |
|------|------|
| `query_handler()` | 主入口，处理异步查询流 |
| `init_handler()` | 初始化: 加载 .env, 创建 Session, 启动 MemoryManager |
| `shutdown_handler()` | 关闭: 停止 MemoryManager |
| `set_chat_manager()` | 注入 ChatManager (用于创建/更新会话) |
| `set_mcp_manager()` | 注入 MCP 客户端管理器 (支持热重载) |

### 3.2 命令分发

**命令检测** (`command_dispatch.py`):
```python
_is_command(query)  # 判断是否命令
run_command_path()  # 执行命令路径
```

**命令类型**:
| 类型 | 示例 | 特点 |
|------|------|------|
| 守护进程命令 | `/restart`, `/status`, `/logs` | 控制服务生命周期 |
| 会话命令 | `/compact`, `/new`, `/clear`, `/history` | 操作会话状态 |

### 3.3 错误处理与调试

- **异常捕获**: 记录完整堆栈，写入调试转储文件
- **取消处理**: `asyncio.CancelledError` 时中断 Agent
- **调试信息**: 保存 `locals()` 到文件供事后分析

### 3.4 环境上下文

`build_env_context()` 构建包含以下信息的提示:
- 当前 session_id, user_id, channel
- 工作目录路径
- 使用技巧提示 (优先使用 skills, 注意文件覆盖)

---

## 四、数据流图

```
┌─────────────────────────────────────────────────────────────────┐
│                        AgentRunner                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────────────────────────────┐  │
│  │ init_handler │───▶│ 1. load_dotenv(.env)                 │  │
│  └──────────────┘    │ 2. SafeJSONSession                  │  │
│                      │ 3. MemoryManager.start()             │  │
│                      └──────────────────────────────────────┘  │
│                                   │                              │
│  ┌───────────────────────────────┼──────────────────────────┐  │
│  │        query_handler          ▼                           │  │
│  │  ┌──────────────────┐   ┌─────────────┐                  │  │
│  │  │ _is_command?    │──▶│ YES:        │                  │  │
│  │  └──────────────────┘   │ run_command_│                  │  │
│  │         │                │    path()   │                  │  │
│  │         │ NO            └─────────────┘                  │  │
│  │         ▼                                            │  │
│  │  ┌──────────────────────────────────────────────┐      │  │
│  │  │ 1. build_env_context()                       │      │  │
│  │  │ 2. mcp_manager.get_clients() (热重载)       │      │  │
│  │  │ 3. CoPawAgent(工具/MCP/内存)                 │      │  │
│  │  │ 4. session.load_session_state()             │      │  │
│  │  │ 5. agent.rebuild_sys_prompt()             │      │  │
│  │  │ 6. stream_printing_messages(agent(msgs))  │      │  │
│  │  │ 7. session.save_session_state()           │      │  │
│  │  └──────────────────────────────────────────────┘      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────┐                                           │
│  │shutdown_handler  │───▶ MemoryManager.close()                │
│  └──────────────────┘                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、关键依赖

| 组件 | 作用 |
|------|------|
| `agentscope_runtime.engine.runner.Runner` | 基类 |
| `CoPawAgent` | 核心代理 (ReAct 循环) |
| `SafeJSONSession` | 跨平台会话持久化 |
| `MemoryManager` | 长期记忆 + 自动压缩 |
| `MCPClientManager` | MCP 工具热重载 |
| `CommandHandler` | 会话命令处理 |
| `stream_printing_messages` | 流式响应输出 |

---

## 六、源代码文件

| 文件 | 职责 |
|------|------|
| `runner.py` | AgentRunner 主类，query_handler 入口 |
| `command_dispatch.py` | 命令检测与分发 |
| `session.py` | SafeJSONSession 跨平台会话 |
| `utils.py` | 环境上下文构建，消息格式转换 |
| `daemon_commands.py` | 守护进程命令处理 |
| `manager.py` | Chat 管理器 |
| `models.py` | 数据模型定义 |
| `query_error_dump.py` | 错误调试转储 |
| `api.py` | API 端点定义 |

---

## 七、总结

Runner 模块是 CoPaw 的**请求编排中心**，核心价值:

1. **双路径设计**: 轻量命令 vs 完整代理，资源按需分配
2. **状态持久化**: JSON 会话支持多端恢复
3. **热重载**: MCP 配置变更无需重启
4. **内存优化**: 自动上下文压缩，避免超过模型上下文限制
5. **跨平台**: Windows 文件名兼容
