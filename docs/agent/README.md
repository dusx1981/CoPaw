# CoPaw Agent System Technical Documentation

## Overview

This document provides a comprehensive technical analysis of the CoPaw agent system, covering the agent loop logic, multi-agent collaboration patterns, and core mechanisms. CoPaw is built on top of [AgentScope](https://github.com/agentscope-ai/agentscope) and implements a single-agent architecture with extensible hooks, memory management, and tool orchestration capabilities.

---

## 1. Agent Architecture

### 1.1 Core Agent Implementation

The primary agent class is `CoPawAgent`, located at:

```
src/copaw/agents/react_agent.py
```

`CoPawAgent` extends the `ReActAgent` from AgentScope, which implements the **Reasoning + Acting (ReAct)** pattern. This pattern combines reasoning about the current state with acting to execute tools, creating an iterative agent loop.

#### Class Hierarchy

```
ReActAgent (AgentScope)
    └── CoPawAgent (CoPaw)
```

#### Key Initialization Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `env_context` | `Optional[str]` | `None` | Optional environment context to prepend to system prompt |
| `enable_memory_manager` | `bool` | `True` | Whether to enable memory manager |
| `mcp_clients` | `Optional[List[Any]]` | `None` | List of MCP clients for tool integration |
| `memory_manager` | `MemoryManager \| None` | `None` | Pre-configured memory manager instance |
| `max_iters` | `int` | `50` | Maximum reasoning-acting iterations |
| `max_input_length` | `int` | `131072` (128K tokens) | Maximum input length for model context window |
| `namesake_strategy` | `NamesakeStrategy` | `"skip"` | Strategy for handling duplicate tool names |

### 1.2 Built-in Tools

CoPaw registers the following built-in tools in its toolkit:

| Tool Function | Description |
|---------------|-------------|
| `execute_shell_command` | Execute shell commands on the host system |
| `read_file` | Read file contents from the filesystem |
| `write_file` | Write content to files |
| `edit_file` | Edit existing files with targeted modifications |
| `browser_use` | Browser automation and control |
| `desktop_screenshot` | Capture screenshots of the desktop |
| `send_file_to_user` | Send files to the user via channels |
| `get_current_time` | Get the current time and date |
| `create_memory_search_tool` | Search through agent memory (when memory manager enabled) |

---

## 2. Agent Loop Logic

### 2.1 ReAct Loop Pattern

The agent loop follows the ReAct (Reasoning + Acting) pattern:

```
┌─────────────────────────────────────────────────────────────┐
│                     Agent Loop (max_iters)                   │
├─────────────────────────────────────────────────────────────┤
│  1. Input: User Message                                      │
│           ↓                                                  │
│  2. Reasoning: Model thinks (tool selection or text output) │
│           ↓                                                  │
│  3. Acting: Execute selected tool OR generate response      │
│           ↓                                                  │
│  4. Observation: Get tool results or user feedback          │
│           ↓                                                  │
│  5. Memory Update: Add to conversation history              │
│           ↓                                                  │
│  6. Check: Have we reached max_iters or terminal state?     │
│           ↓                                                  │
│     No → Return to Step 2                                    │
│     Yes → Return final response                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Query Handler Flow

The query handling flow is implemented in `AgentRunner.query_handler()`:

```
src/copaw/app/runner/runner.py
```

#### Query Processing Pipeline

```
User Request
     │
     ▼
┌──────────────────────────────┐
│ 1. Command Path Check         │ ──── If query is a command (/restart, etc.)
│    _is_command(query)         │      Execute command handler
└──────────────────────────────┘
     │
     ▼ (Normal path)
┌──────────────────────────────┐
│ 2. Build Environment Context │
│    build_env_context()        │
└──────────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│ 3. Get MCP Clients            │ ──── Hot-reloadable MCP tools
│    _mcp_manager.get_clients() │
└──────────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│ 4. Create CoPawAgent          │
│    - Load config              │
│    - Initialize toolkit       │
│    - Register skills          │
│    - Setup memory manager    │
└──────────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│ 5. Load Session State        │
│    session.load_session_state │
└──────────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│ 6. Rebuild System Prompt     │
│    agent.rebuild_sys_prompt() │ ──── Reflect latest AGENTS.md/SOUL.md
└──────────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│ 7. Execute Agent Loop        │
│    stream_printing_messages() │
│    └── agent(msgs)            │ ──── ReAct loop
└──────────────────────────────┘
     │
     ▼
┌──────────────────────────────┐
│ 8. Save Session State        │
│    session.save_session_state│
└──────────────────────────────┘
     │
     ▼
   Response
```

### 2.3 Session Management

CoPaw uses `SafeJSONSession` (inherited from AgentScope's `JSONSession`) for session persistence:

- **Session Storage**: JSON files in `{working_dir}/sessions/`
- **Filename Sanitization**: Windows-compatible filenames (replaces `\/:*?"<>|` with `--`)
- **State Saved**: Agent memory, conversation history, compressed summaries

---

## 3. Agent Extensions and Hooks

### 3.1 Hook System

CoPaw implements a hook system for extending agent behavior at specific points. Two main hooks are registered:

#### Bootstrap Hook

**Location**: `src/copaw/agents/hooks/bootstrap.py`

**Purpose**: Provides guidance for first-time users by checking for `BOOTSTRAP.md` in the working directory.

**Trigger**: First interaction with the agent (pre-reasoning)

```python
# Hook registration in CoPawAgent
self.register_instance_hook(
    hook_type="pre_reasoning",
    hook_name="bootstrap_hook",
    hook=bootstrap_hook.__call__,
)
```

#### Memory Compaction Hook

**Location**: `src/copaw/agents/hooks/memory_compaction.py`

**Purpose**: Automatically compresses memory when it exceeds the threshold (default: 70% of max_input_length).

**Trigger**: Pre-reasoning, when memory exceeds `_memory_compact_threshold`

```python
# Configuration
max_input_length = 128 * 1024  # 128K tokens
MEMORY_COMPACT_RATIO = 0.7  # 70%
memory_compact_threshold = max_input_length * MEMORY_COMPACT_RATIO
```

### 3.2 Command Handler

**Location**: `src/copaw/agents/command_handler.py`

The command handler processes system commands that users can invoke during conversation:

| Command | Description |
|---------|-------------|
| `/compact` | Manually trigger memory compaction |
| `/new` | Start a new conversation with summary |
| `/clear` | Clear conversation history |
| `/history` | Display conversation history |
| `/compact_str` | Show compressed summary |
| `/await_summary` | Wait for pending summary tasks |

#### Command Processing Flow

```
User Input (starts with /)
        │
        ▼
┌─────────────────────────┐
│ is_command(query)      │ ──── Check if starts with /
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ handle_command(query)   │
│   └── _process_{cmd}() │ ──── Dispatch to handler
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│ Return system message   │
└─────────────────────────┘
```

---

## 4. Memory Management

### 4.1 Memory Manager Architecture

**Location**: `src/copaw/agents/memory/memory_manager.py`

The `MemoryManager` handles long-term context management with automatic compression:

```
┌─────────────────────────────────────────────────────────────┐
│                     MemoryManager                            │
├─────────────────────────────────────────────────────────────┤
│  Components:                                                 │
│  ├── Token Counter: Counts tokens in messages                │
│  ├── Toolkit: File operations for memory storage            │
│  ├── Summary Tasks: Async background summarization          │
│  └── InMemoryMemory: AgentScope memory instance             │
├─────────────────────────────────────────────────────────────┤
│  Features:                                                   │
│  ├── Auto-compaction at 70% context threshold               │
│  ├── Async summarization of old messages                    │
│  ├── Memory search tool for semantic retrieval               │
│  └── Compressed summary preservation                        │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Memory Compaction Flow

```
Memory Size > Threshold (70% of max_input_length)
        │
        ▼
┌─────────────────────────────┐
│ 1. Mark messages as         │
│    COMPRESSED              │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ 2. Create async summary     │
│    task (background)       │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ 3. Generate summary using  │
│    LLM                      │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ 4. Store compressed        │
│    summary in memory       │
└─────────────────────────────┘
        │
        ▼
    Ready for next iteration
```

---

## 5. MCP (Model Context Protocol) Integration

### 5.1 MCP Client Manager

**Location**: `src/copaw/app/mcp/__init__.py`

CoPaw supports MCP for extending tool capabilities through external services:

```
┌─────────────────────────────────────────┐
│          MCPClientManager                │
├─────────────────────────────────────────┤
│  Client Types:                           │
│  ├── HttpStatefulClient: HTTP-based     │
│  └── StdIOStatefulClient: Process-based│
├─────────────────────────────────────────┤
│  Features:                               │
│  ├── Hot-reload on config change        │
│  ├── Auto-reconnection on failure       │
│  └── Tool registration to toolkit      │
└─────────────────────────────────────────┘
```

### 5.2 MCP Hot-Reload Mechanism

```
Config Change Detected (config.json)
        │
        ▼
┌─────────────────────────────┐
│ MCPConfigWatcher            │
│   (file system watcher)     │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ Rebuild MCP Clients         │
│   - Close old connections   │
│   - Initialize new clients   │
│   - Register to agent        │
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ Agent gets updated tools    │
│   (no restart required)     │
└─────────────────────────────┘
```

### 5.3 MCP Client Recovery

The agent implements automatic recovery for MCP clients:

```python
# Recovery strategy in CoPawAgent
async def _recover_mcp_client(client):
    # 1. Try to reconnect existing client
    if await self._reconnect_mcp_client(client):
        return client
    
    # 2. Rebuild client from stored config
    rebuilt_client = self._rebuild_mcp_client(client)
    if rebuilt_client and await self._reconnect_mcp_client(rebuilt_client):
        return rebuilt_client
    
    return None
```

---

## 6. Skills System

### 6.1 Skill Loading

**Location**: `src/copaw/agents/skills_manager.py`

Skills are Python modules loaded from the working directory:

```
working_dir/
└── skills/
    ├── skill_name_1/
    │   ├── __init__.py
    │   ├── __main__.py
    │   └── ...
    └── skill_name_2/
        └── ...
```

### 6.2 Skill Registration Flow

```
CoPawAgent Initialization
        │
        ▼
┌─────────────────────────────┐
│ ensure_skills_initialized() │
│   - Check skills directory  │
│   - Discover available skills│
└─────────────────────────────┘
        │
        ▼
┌─────────────────────────────┐
│ For each available skill:   │
│   toolkit.register_agent_   │
│     skill(skill_dir)        │
└─────────────────────────────┘
        │
        ▼
    Tools available to agent
```

---

## 7. Multi-Agent Collaboration

### 7.1 Current Status

**As of the current version (v0.0.4), CoPaw does NOT implement multi-agent collaboration.** The system is designed as a **single-agent architecture** where one AI assistant handles all user interactions.

### 7.2 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     CoPaw System                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │  Channel    │    │   Console   │    │   Cron/     │    │
│  │  (DingTalk, │    │   (Web UI)  │    │   Heartbeat │    │
│  │   QQ, etc.) │    │             │    │             │    │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    │
│         │                   │                   │            │
│         └───────────────────┼───────────────────┘            │
│                             ▼                                │
│                  ┌──────────────────┐                        │
│                  │   AgentRunner    │                        │
│                  │  (Query Handler) │                        │
│                  └────────┬─────────┘                        │
│                           │                                  │
│                           ▼                                  │
│                  ┌──────────────────┐                        │
│                  │   CoPawAgent     │  ◄── Single Agent    │
│                  │  (ReAct Loop)    │                        │
│                  └──────────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 Future Plans (Roadmap)

According to the project roadmap, multi-agent capabilities are planned as **long-term goals**:

| Feature | Status | Description |
|---------|--------|-------------|
| Multi-agent | Long-term Planning | Native multi-agent workflows built on AgentScope |

The roadmap states:
> "Multi-agent — Built on AgentScope, native multi-agent workflows"

This indicates that future versions will leverage AgentScope's multi-agent capabilities, which include:
- Agent-to-agent communication
- Task delegation between agents
- Multi-agent workflow orchestration

---

## 8. System Commands Reference

### 8.1 Runtime Commands

These commands are handled by the command dispatch system:

| Command | Description |
|---------|-------------|
| `/restart` | Restart the CoPaw daemon (in-process) |
| `/stop` | Stop the CoPaw daemon |

### 8.2 Conversation Commands

Handled by `CommandHandler`:

| Command | Description |
|---------|-------------|
| `/compact` | Manually compact memory |
| `/new` | Start new conversation |
| `/clear` | Clear all history |
| `/history` | Show conversation history |
| `/compact_str` | Show compressed summary |
| `/await_summary` | Wait for summary tasks |

---

## 9. Key Files Reference

### Core Agent Files

| File | Purpose |
|------|---------|
| `src/copaw/agents/react_agent.py` | Main CoPawAgent implementation |
| `src/copaw/agents/command_handler.py` | System command handling |
| `src/copaw/agents/schema.py` | Type definitions for agent tools |
| `src/copaw/agents/prompt.py` | System prompt building |
| `src/copaw/agents/model_factory.py` | Model and formatter creation |

### Runner Files

| File | Purpose |
|------|---------|
| `src/copaw/app/runner/runner.py` | Agent query execution |
| `src/copaw/app/runner/session.py` | Session state management |
| `src/copaw/app/runner/manager.py` | Chat specification management |
| `src/copaw/app/runner/command_dispatch.py` | Command path execution |

### Memory and Hooks

| File | Purpose |
|------|---------|
| `src/copaw/agents/memory/memory_manager.py` | Long-term memory management |
| `src/copaw/agents/hooks/bootstrap.py` | First-time user guidance |
| `src/copaw/agents/hooks/memory_compaction.py` | Auto-memory compression |

### Integration

| File | Purpose |
|------|---------|
| `src/copaw/app/_app.py` | FastAPI application, lifecycle management |
| `src/copaw/app/mcp/__init__.py` | MCP client management |
| `src/copaw/app/channels/manager.py` | Channel (DingTalk, QQ, etc.) management |

---

## 10. Configuration

### 10.1 Agent Configuration

Agent behavior is configured in `config.json`:

```json
{
  "agents": {
    "running": {
      "max_iters": 50,
      "max_input_length": 131072
    },
    "language": "en"
  }
}
```

### 10.2 Environment Variables

| Variable | Description |
|----------|-------------|
| `ENABLE_MEMORY_MANAGER` | Set to `"false"` to disable memory manager |
| `DASHSCOPE_API_KEY` | API key for DashScope provider |
| `COPAW_CONSOLE_STATIC_DIR` | Override console static files directory |

---

## 11. Summary

CoPaw implements a sophisticated single-agent system with:

1. **ReAct Loop**: Reasoning + Acting iteration pattern with configurable max iterations
2. **Memory Management**: Automatic context compaction with async summarization
3. **Tool Orchestration**: Built-in tools + MCP integration with hot-reload
4. **Extensibility**: Hook system for pre/post processing, skill loading
5. **Session Persistence**: JSON-based session state with cross-platform filename support
6. **Multi-channel Support**: Integration with DingTalk, QQ, Discord, Telegram, etc.

**Multi-agent collaboration is NOT currently implemented** but is planned for future versions as part of the AgentScope integration roadmap.

---

*Document Version: 1.0*  
*Last Updated: 2026-03-05*  
*Project: CoPaw v0.0.4*
