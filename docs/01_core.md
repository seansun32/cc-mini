# cc-mini 核心文件与抽象映射

## 1. 定义 Agent 核心抽象的文件

### 1.1 Tool 抽象 — `src/core/tools/base.py`

所有工具的基石。定义了两个核心类型：

| 类型 | 行号 | 说明 |
|------|------|------|
| `ToolResult` | 5-8 | dataclass，包含 `content: str` 和 `is_error: bool` |
| `Tool` | 10-35 | 抽象基类，规定每个工具必须实现的协议 |

`Tool` 抽象的关键接口：

```
属性：name, description, input_schema (JSON Schema)
方法：execute(**kwargs) → ToolResult    # 抽象，子类必须实现
      to_api_schema() → dict           # 具体，生成 LLM 可见的 tool schema
      is_read_only() → bool            # 具体，默认 False，影响权限检查
```

### 1.2 Engine — `src/core/engine.py` (L99-432)

Agent 的运行时核心。不是抽象类，而是唯一的具体实现。关键状态字段：

| 字段 | 类型 | 职责 |
|------|------|------|
| `_messages` | `list[dict]` | 对话历史（短期记忆） |
| `_tools` | `dict[str, Tool]` | 工具注册表（按 name 索引） |
| `_client` | `LLMClient` | LLM 通信客户端 |
| `_system_prompt` | `str` | 系统提示词 |
| `_permissions` | `PermissionChecker` | 权限检查器 |
| `_session_store` | `SessionStore \| None` | 会话持久化 |
| `_cost_tracker` | `CostTracker \| None` | 用量追踪 |
| `_aborted` | `bool` | 中止标志 |
| `_active_stream` | stream \| None | 当前活跃的 HTTP 流 |
| `_turn_start_len` | `int` | 回滚锚点（cancel_turn 用） |

### 1.3 LLMClient — `src/core/llm.py` (L79-241)

LLM 通信的策略抽象。通过 `provider` 参数在运行时选择后端：

| 组件 | 行号 | 职责 |
|------|------|------|
| `LLMClient` | 79-241 | 统一接口：`stream_messages()` / `create_message()` |
| `_AnthropicStream` | 243-281 | Anthropic SDK 流式包装 |
| `_OpenAIStream` | 284-374 | OpenAI 格式转换为 Anthropic 格式 |
| `LLMUsage` | 29-35 | 用量数据 dataclass |
| `LLMMessage` | 37-40 | 响应消息 dataclass |

### 1.4 辅助抽象

| 文件 | 核心类型 | 行号 | 说明 |
|------|----------|------|------|
| `permissions.py` | `PermissionChecker` | 31-48 | 工具执行的权限守门人 |
| `session.py` | `SessionStore`, `SessionMeta` | 30-210 | 会话持久化与元数据 |
| `memory.py` | 函数集合 | 1-330 | 跨会话记忆系统（无 class，纯函数式） |
| `compact.py` | `CompactService` | — | 上下文压缩 |
| `context.py` | `build_system_prompt()` | 27+ | System Prompt 动态组装 |
| `config.py` | `AppConfig` | 128+ | 配置 dataclass |

---

## 2. 控制 Agent 执行循环的文件

**核心文件**：`src/core/engine.py`
**核心方法**：`Engine.submit()` (L234-365)

### 循环结构

```python
def submit(self, user_input) -> Iterator[tuple]:
    # [1] 追加用户消息
    self._messages.append(user_msg)        # L249
    self._persist(user_msg)                # L253

    # [2] ReAct 主循环
    while True:                            # L256
        # [2a] 调用 LLM（含重试）
        for attempt in range(3):           # L264
            with self._client.stream_messages(...) as stream:  # L267
                for text in stream.text_stream:
                    yield ("text", text)   # L281 — 实时文本
                final = stream.get_final_message()  # L289

        tool_uses = [b for b in final.content if b["type"] == "tool_use"]  # L300

        # [2b] 追加 assistant 消息
        self._messages.append(assistant_msg)   # L335

        # [2c] 判断是否继续
        if not tool_uses:
            break                          # L342 — 无工具调用，循环结束

        # [2d] 执行工具
        for tool_use in tool_uses:         # L345
            yield ("tool_call", name, input)
            result = self._execute_tool(tool_use)   # L349
            yield ("tool_result", name, input, result)  # L350

        # [2e] 工具结果回注
        self._messages.append(tool_results_msg)   # L358
        # continue → 回到 [2a]，带着工具结果再次调用 LLM
```

### 循环终止条件

| 条件 | 触发位置 | 行为 |
|------|----------|------|
| 无 tool_use | L342 | `break` — 正常完成 |
| 用户中止（Esc） | L304-305, 325-326 | `AbortedError` → `cancel_turn()` 回滚 |
| 认证失败 | L308-310 | 弹出消息，立即返回 |
| 重试耗尽 | L311-320 | yield error，立即返回 |

### 事件流类型

| 事件 | 格式 | 消费者 |
|------|------|--------|
| `("text", chunk)` | 流式文本片段 | REPL 实时打印 |
| `("waiting",)` | 文本结束，工具调用即将开始 | REPL 显示 spinner |
| `("tool_call", name, input)` | 工具执行前 | REPL 打印工具预览 |
| `("tool_result", name, input, result)` | 工具执行后 | REPL 打印结果 |
| `("usage", usage)` | Token 用量 | CostTracker |
| `("error", msg)` | 非致命错误 | REPL 打印错误 |

---

## 3. LLM 调用逻辑集中位置

**文件**：`src/core/llm.py`

### 调用链

```
Engine.submit()                          # engine.py L267-289
  └─ LLMClient.stream_messages()         # llm.py L126-153
       ├─ Anthropic → _AnthropicStream   # llm.py L243-281
       │    └─ client.messages.stream()  # Anthropic SDK
       └─ OpenAI → _OpenAIStream        # llm.py L284-374
            └─ client.chat.completions.create(stream=True)
```

### LLMClient 关键方法

| 方法 | 行号 | 说明 |
|------|------|------|
| `stream_messages()` | 126-153 | 流式调用，返回 stream context manager |
| `create_message()` | 99-124 | 非流式调用（用于 compact、dream 等内部用途） |
| `is_authentication_error()` | — | 错误分类：认证失败 |
| `is_retryable_error()` | — | 错误分类：可重试（过载/超时） |

### 请求参数组装

在 `Engine.submit()` 中（L267-275）：

```python
self._client.stream_messages(
    model=self._model,
    max_tokens=self._max_tokens,
    system=self._system_prompt,
    tools=[t.to_api_schema() for t in self._tools.values()],
    messages=self._messages,
    effort=self._effort,
)
```

### 响应格式归一化

| 函数 | 行号 | 说明 |
|------|------|------|
| `_normalize_anthropic_content()` | 377-416 | Anthropic 原生格式标准化 |
| `_normalize_openai_message()` | 419-440 | OpenAI 格式 → Anthropic 格式 |

两个 Provider 的响应最终统一为 `LLMMessage(content=list[dict], usage=LLMUsage)` 格式，Engine 无需关心底层差异。

---

## 4. 工具注册与调用位置

### 4.1 工具注册

**文件**：`src/core/main.py`

```
_build_base_tools()                   # L546-551 — 构建 6 个内置工具
  └─ [FileReadTool, GlobTool, GrepTool, FileEditTool, FileWriteTool, BashTool]

_build_tools_for_mode(coordinator)    # L595-604 — 按模式扩展
  ├─ base tools + AskUserQuestionTool
  └─ if coordinator: + AgentTool, SendMessageTool, TaskStopTool

Engine.__init__(tools=...)            # engine.py L122
  └─ self._tools = {t.name: t for t in tools}   # 按 name 建立注册表
```

### 4.2 工具调用

**文件**：`src/core/engine.py`

**调用入口**：`Engine._execute_tool(tool_use)` (L367-405)

```
_execute_tool(tool_use)
  │
  ├─ 查找：self._tools.get(tool_name)          # L371
  │   └─ 未找到 → ToolResult(is_error=True)
  │
  ├─ 权限检查：self._permissions.check(tool, input)  # L376
  │   └─ "deny" → ToolResult("Permission denied")
  │
  ├─ 执行：tool.execute(**tool_input)           # L381
  │   └─ 返回 ToolResult(content, is_error)
  │
  └─ 行变更追踪（Edit/Write）                    # L392-401
```

### 4.3 权限检查流程

**文件**：`src/core/permissions.py` (L31-48)

```
PermissionChecker.check(tool, inputs)
  ├─ tool.is_read_only() → "allow"              # 只读直接放行
  ├─ self._auto_approve → "allow"               # --auto-approve 模式
  ├─ tool in self._always_allow → "allow"       # 用户已选 "always"
  ├─ Bash + sandbox auto-allow → "allow"        # 沙箱自动放行
  └─ _prompt_user(tool, inputs) → y/n/a         # 交互确认
```

### 4.4 具体工具实现一览

| 文件 | 工具名 | 只读 | 核心逻辑 |
|------|--------|------|----------|
| `tools/file_read.py` | Read | Yes | 读文件，支持 offset/limit |
| `tools/glob_tool.py` | Glob | Yes | pathlib.glob，按 mtime 排序 |
| `tools/grep_tool.py` | Grep | Yes | subprocess 调 rg，降级 Python regex |
| `tools/file_edit.py` | Edit | No | 字符串唯一匹配替换 |
| `tools/file_write.py` | Write | No | 创建/覆盖文件 |
| `tools/bash.py` | Bash | No | subprocess.run，支持 sandbox 和 timeout |
| `tools/agent.py` | Agent | No | WorkerManager.spawn() 启动后台 Worker |
| `tools/ask_user.py` | AskUserQuestion | No | 向用户提问并等待回答 |

---

## 5. Agent 状态存储与更新位置

Agent 的状态分布在三个层次：**运行时（内存）→ 会话级（磁盘）→ 跨会话（记忆系统）**。

### 5.1 运行时状态 — `engine.py`

| 状态 | 存储位置 | 更新时机 |
|------|----------|----------|
| 对话历史 | `Engine._messages` (L125) | 每条消息追加（L249/335/358） |
| 回滚锚点 | `Engine._turn_start_len` (L127) | 每轮开始时设置（L246） |
| 中止标志 | `Engine._aborted` | `abort()` 调用时 |
| 活跃流 | `Engine._active_stream` | API 调用时绑定 |

**消息访问接口**：

```python
get_messages() → list[dict]          # L134 — 返回副本
set_messages(messages)               # L137 — 归一化后设置
cancel_turn()                        # L222 — 回滚到 _turn_start_len
```

### 5.2 会话级持久化 — `session.py`

**存储路径**：`~/.mini-claude/sessions/{sanitized_cwd}/{session_id}.jsonl`

| 操作 | 方法 | 说明 |
|------|------|------|
| 写入 | `SessionStore.append_message()` (L129-141) | JSON 序列化 → 追加到 JSONL |
| 元数据 | `SessionStore._save_meta()` (L143-158) | 写入 `{id}.meta.json` |
| 加载 | `SessionStore.load_messages()` (L163-181) | 读取 JSONL → list[dict] |
| 列表 | `SessionStore.list_sessions()` (L184-198) | 扫描所有 meta.json |
| 恢复 | `SessionStore.load_session()` (L201-210) | 返回 (meta, messages) |

**触发路径**：`Engine.submit()` → `Engine._persist()` (L165-171) → `SessionStore.append_message()`

每条消息（user / assistant / tool_result）都会立即持久化。

### 5.3 跨会话记忆 — `memory.py`

**存储路径**：`~/.mini-claude/memory/`

```
memory/
├── MEMORY.md                          # 整合后的结构化索引
├── .consolidate-lock                  # 整合互斥锁
└── logs/
    └── YYYY/MM/YYYY-MM-DD.md          # 每日追加日志
```

| 操作 | 函数 | 行号 | 说明 |
|------|------|------|------|
| 提取 | `extract_memory_tags(text)` | 158-160 | 从 assistant 消息提取 `<memory>` 标签 |
| 写入 | `append_to_daily_log(dir, entry)` | 36-41 | 追加到当日日志 |
| 读取 | `load_memory_index(dir)` | 48-57 | 读取 MEMORY.md（截断 10k） |
| 注入 | `build_memory_system_section(dir)` | 167-270 | 生成 System Prompt 段落 |
| 整合门控 | `should_auto_dream(dir, ...)` | 132-151 | 检查时间 + 会话数阈值 |
| 加锁 | `try_acquire_lock(dir)` | 78-100 | 防并发整合 |
| 释放 | `release_lock(dir)` | 103-110 | 更新锁 mtime |

**记忆生命周期**：

```
assistant 消息含 <memory> 标签
  → extract_memory_tags()          # main.py L992
  → append_to_daily_log()          # main.py L994
  → (累积到阈值)
  → should_auto_dream() == True    # main.py L997
  → try_acquire_lock()
  → Dream Engine 总结所有日志
  → 更新 MEMORY.md
  → release_lock()
  → 下次对话 System Prompt 注入新索引
```

### 5.4 用量追踪 — `cost_tracker.py`

| 更新时机 | 位置 | 数据 |
|----------|------|------|
| API 调用后 | engine.py L292-299 | input/output/cache tokens + 耗时 |
| 工具执行后 | engine.py L392-401 | Edit/Write 的行变更数 |

---

## 总览：文件-职责对照表

```
src/core/
├── main.py              入口 & REPL 循环 & 工具注册
├── engine.py            ReAct 执行循环 & 运行时状态
├── llm.py               LLM 调用 & Provider 适配
├── tools/
│   ├── base.py          Tool 抽象基类 & ToolResult
│   ├── file_read.py     Read 工具
│   ├── file_edit.py     Edit 工具
│   ├── file_write.py    Write 工具
│   ├── glob_tool.py     Glob 工具
│   ├── grep_tool.py     Grep 工具
│   ├── bash.py          Bash 工具
│   ├── agent.py         Agent/SendMessage/TaskStop 工具
│   └── ask_user.py      AskUserQuestion 工具
├── permissions.py       权限检查
├── session.py           会话持久化（JSONL）
├── memory.py            跨会话记忆（KAIROS）
├── compact.py           上下文压缩
├── context.py           System Prompt 组装
├── config.py            配置加载
├── cost_tracker.py      用量 & 成本追踪
├── coordinator.py       协调器系统提示
└── worker_manager.py    Worker 线程管理
```
