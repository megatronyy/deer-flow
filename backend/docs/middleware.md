# DeerFlow 中间件系统详细分析

## 概述

DeerFlow 的中间件系统是基于 LangGraph Agent Middleware 架构构建的，它允许在 Agent 执行的不同生命周期阶段插入自定义逻辑。中间件提供了一种模块化的方式来扩展 Agent 功能，无需修改核心 Agent 代码。

### 中间件架构

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware[AgentState]):
    state_schema = ...  # 可选的状态模式

    def before_agent(self, state, runtime) -> dict | None:
        """Agent 执行前调用"""

    def after_agent(self, state, runtime) -> dict | None:
        """Agent 执行后调用"""

    def before_model(self, state, runtime) -> dict | None:
        """LLM 调用前调用"""

    def after_model(self, state, runtime) -> dict | None:
        """LLM 调用后调用"""

    def wrap_model_call(self, request, handler) -> ModelCallResult:
        """包装 LLM 调用"""

    def wrap_tool_call(self, request, handler) -> ToolMessage | Command:
        """包装工具调用"""
```

---

## 中间件链顺序

### 严格依赖顺序

中间件链的顺序是严格定义的，必须遵守以下依赖关系：

```python
# 在 _build_middlewares() 中的顺序
middlewares = [
    # 阶段 1: 基础设施层（必须最先执行）
    1. ThreadDataMiddleware       # 创建线程目录
    2. UploadsMiddleware         # 跟踪上传文件（依赖 thread_id）
    3. SandboxMiddleware         # 获取沙箱环境

    # 阶段 2: 修复和准备层
    4. DanglingToolCallMiddleware # 修复中断的工具调用
    5. GuardrailMiddleware        # 工具调用授权（可选）
    6. ToolErrorHandlingMiddleware # 工具异常处理

    # 阶段 3: 优化和增强层（可选）
    7. SummarizationMiddleware     # 上下文压缩
    8. TodoListMiddleware          # 任务跟踪
    9. TokenUsageMiddleware        # Token 使用日志

    # 阶段 4: 生成和记录层
    10. TitleMiddleware             # 自动生成标题
    11. MemoryMiddleware           # 队列化记忆更新

    # 阶段 5: 特殊功能层（条件性）
    12. ViewImageMiddleware        # 注入 base64 图片（仅视觉模型）
    13. DeferredToolFilterMiddleware # 过滤延迟工具（tool_search）
    14. SubagentLimitMiddleware    # 限制并发子代理

    # 阶段 6: 安全和终止层（必须最后）
    15. LoopDetectionMiddleware     # 检测和打破循环
    16. ClarificationMiddleware      # 拦截澄清请求（必须最后）
]
```

### 依赖关系说明

1. **ThreadDataMiddleware 必须最先**：创建线程目录，其他中间件依赖 thread_id
2. **UploadsMiddleware 依赖 ThreadDataMiddleware**：需要 thread_id 来定位上传目录
3. **SandboxMiddleware 依赖 ThreadDataMiddleware**：需要 thread_id 来获取/创建沙箱
4. **DanglingToolCallMiddleware 在 LLM 调用前**：修复消息历史中的悬挂工具调用
5. **GuardrailMiddleware 在工具调用前**：评估工具调用授权
6. **ClarificationMiddleware 必须最后**：拦截澄清请求并中断执行
7. **SubagentLimitMiddleware 在 after_model**：需要截断模型生成的多余 task 调用

---

## 核心中间件详细分析

### 1. ThreadDataMiddleware

**文件位置**: `agents/middlewares/thread_data_middleware.py`

**功能**：为每个线程创建数据目录结构

**执行时机**: `before_agent` (Agent 执行前)

**生命周期**:
- `lazy_init=True` (默认): 只计算路径，目录按需创建
- `lazy_init=False`: 在 `before_agent` 中立即创建目录

**目录结构**:
```
backend/.deer-flow/threads/{thread_id}/user-data/
├── workspace/    # 工作目录（临时文件）
├── uploads/      # 用户上传的文件
└── outputs/      # 最终交付物
```

**实现逻辑**:
```python
def before_agent(self, state, runtime):
    # 1. 获取 thread_id
    thread_id = runtime.context.get("thread_id")
    if not thread_id:
        thread_id = get_config()["configurable"]["thread_id"]

    if not thread_id:
        raise ValueError("Thread ID is required")

    # 2. 根据模式创建或计算路径
    if self._lazy_init:
        paths = self._get_thread_paths(thread_id)  # 只计算
    else:
        paths = self._create_thread_directories(thread_id)  # 创建目录

    # 3. 返回状态更新
    return {
        "thread_data": {
            "workspace_path": str(workspace_dir),
            "uploads_path": str(uploads_dir),
            "outputs_path": str(outputs_dir),
        }
    }
```

**关键方法**:
- `_get_thread_paths()`: 获取路径但不创建
- `_create_thread_directories()`: 调用 `self._paths.ensure_thread_dirs(thread_id)`

---

### 2. UploadsMiddleware

**文件位置**: `agents/middlewares/uploads_middleware.py`

**功能**：注入上传文件信息到 Agent 上下文

**执行时机**: `before_agent` (Agent 执行前)

**实现逻辑**:
```python
def before_agent(self, state, runtime):
    messages = state.get("messages", [])
    last_message = messages[-1]

    # 1. 检查是否为 HumanMessage
    if not isinstance(last_message, HumanMessage):
        return None

    # 2. 从 additional_kwargs.files 提取新上传的文件
    new_files = self._files_from_kwargs(last_message, uploads_dir)

    # 3. 扫描上传目录获取历史文件
    historical_files = []
    if uploads_dir.exists():
        for file_path in uploads_dir.iterdir():
            if file_path.name not in {f["filename"] for f in new_files}:
                historical_files.append(...)

    # 4. 创建文件信息消息
    files_message = self._create_files_message(new_files, historical_files)

    # 5. 前置到最后一条 HumanMessage
    updated_message = HumanMessage(
        content=f"{files_message}\n\n{original_content}",
        id=last_message.id,
        additional_kwargs=last_message.additional_kwargs,  # 保留原始元数据
    )

    return {
        "uploaded_files": new_files,
        "messages": [updated_message],
    }
```

**消息格式**:
```
<uploaded_files>
The following files were uploaded in this message:
- example.pdf (125.3 KB)
  Path: /mnt/user-data/uploads/example.pdf

The following files were uploaded in previous messages and are still available:
- data.json (12.1 KB)
  Path: /mnt/user-data/uploads/data.json

You can read these files using the `read_file` tool with the paths shown above.
</uploaded_files>
```

**关键方法**:
- `_files_from_kwargs()`: 从 `additional_kwargs.files` 提取文件信息
- `_create_files_message()`: 格式化文件列表消息
- 验证文件存在性（如果提供 `uploads_dir`）

---

### 3. SandboxMiddleware

**文件位置**: `sandbox/middleware.py`

**功能**：管理沙箱环境的生命周期

**执行时机**:
- `before_agent`: 获取沙箱（eager 模式）
- `after_agent`: 释放沙箱

**生命周期**:
- `lazy_init=True` (默认): 首次工具调用时获取
- `lazy_init=False`: 在 `before_agent` 中立即获取
- 沙箱在多次轮次中复用
- 应用关闭时通过 `SandboxProvider.shutdown()` 清理

**实现逻辑**:
```python
def __init__(self, lazy_init: bool = True):
    self._lazy_init = lazy_init

def before_agent(self, state, runtime):
    # Eager 模式：立即获取沙箱
    if not self._lazy_init:
        if "sandbox" not in state or state["sandbox"] is None:
            thread_id = runtime.context.get("thread_id")
            sandbox_id = self._acquire_sandbox(thread_id)
            return {"sandbox": {"sandbox_id": sandbox_id}}

def after_agent(self, state, runtime):
    # 释放沙箱
    sandbox = state.get("sandbox")
    if sandbox:
        sandbox_id = sandbox["sandbox_id"]
        get_sandbox_provider().release(sandbox_id)
```

**关键方法**:
- `_acquire_sandbox()`: 调用 `SandboxProvider.acquire(thread_id)`
- 沙箱 ID 存储在 `state["sandbox"]["sandbox_id"]`

---

### 4. DanglingToolCallMiddleware

**文件位置**: `agents/middlewares/dangling_tool_call_middleware.py`

**功能**：修复悬挂的工具调用（缺少响应的工具调用）

**执行时机**: `wrap_model_call` (LLM 调用包装)

**问题背景**:
当 AIMessage 包含 tool_calls 但没有对应的 ToolMessage 时（如用户中断或请求取消），会导致 LLM 错误。

**实现逻辑**:
```python
def wrap_model_call(self, request, handler):
    # 1. 收集所有现有的 ToolMessage ID
    existing_tool_msg_ids = set()
    for msg in request.messages:
        if isinstance(msg, ToolMessage):
            existing_tool_msg_ids.add(msg.tool_call_id)

    # 2. 检查是否需要修复
    needs_patch = False
    for msg in request.messages:
        if getattr(msg, "type") == "ai":
            for tc in getattr(msg, "tool_calls", None) or []:
                if tc["id"] not in existing_tool_msg_ids:
                    needs_patch = True
                    break
        if needs_patch:
            break

    if not needs_patch:
        return handler(request)

    # 3. 构建修复后的消息列表
    patched = []
    patched_ids = set()
    for msg in request.messages:
        patched.append(msg)

        if getattr(msg, "type") == "ai":
            for tc in getattr(msg, "tool_calls", None) or []:
                if tc["id"] not in existing_tool_msg_ids and tc["id"] not in patched_ids:
                    # 插入占位符 ToolMessage
                    patched.append(ToolMessage(
                        content="[Tool call was interrupted and did not return a result.]",
                        tool_call_id=tc["id"],
                        name=tc["name"],
                        status="error",
                    ))
                    patched_ids.add(tc["id"])

    # 4. 使用修复后的消息调用
    return handler(request.override(messages=patched))
```

**关键特性**:
- 使用 `wrap_model_call` 而不是 `before_model`，确保修复的消息在正确位置（紧随悬挂的 AIMessage 之后）
- 占位符 ToolMessage 包含错误状态和描述性消息

---

### 5. GuardrailMiddleware

**文件位置**: `guardrails/middleware.py`

**功能**：在工具执行前评估工具调用授权

**执行时机**: `wrap_tool_call` (工具调用包装)

**实现逻辑**:
```python
def wrap_tool_call(self, request, handler):
    # 1. 构建授权请求
    gr = GuardrailRequest(
        tool_name=request.tool_call.get("name", ""),
        tool_input=request.tool_call.get("args", {}),
        agent_id=self.passport,
        timestamp=datetime.now(UTC).isoformat(),
    )

    # 2. 调用 Provider 评估
    try:
        decision = self.provider.evaluate(gr)
    except Exception:
        logger.exception("Guardrail provider error")
        if self.fail_closed:
            # 失败时阻止
            decision = GuardrailDecision(
                allow=False,
                reasons=[GuardrailReason(code="oap.evaluator_error", message="guardrail provider error")]
            )
        else:
            # 失败时放行
            return handler(request)

    # 3. 根据决策处理
    if not decision.allow:
        logger.warning(f"Guardrail denied: tool={gr.tool_name} policy={decision.policy_id}")
        # 返回错误 ToolMessage
        return ToolMessage(
            content=f"Guardrail denied: tool '{gr.tool_name}' was blocked. Reason: {decision.reasons[0].message}",
            tool_call_id=request.tool_call.get("id", ""),
            name=request.tool_call.get("name", "unknown"),
            status="error",
        )

    # 4. 允许执行
    return handler(request)
```

**配置**:
```yaml
guardrails:
  enabled: true
  fail_closed: false  # Provider 异常时是否阻止
  passport: "my-agent"  # 额外的上下文信息
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      allowlist: ["bash", "read_file", "write_file"]
```

**Provider 协议**:
```python
class GuardrailProvider(Protocol):
    name: str

    def evaluate(self, request: GuardrailRequest) -> GuardrailDecision:
        """同步评估"""
        ...

    async def aevaluate(self, request: GuardrailRequest) -> GuardrailDecision:
        """异步评估"""
        ...
```

---

### 6. ToolErrorHandlingMiddleware

**文件位置**: `agents/middlewares/tool_error_handling_middleware.py`

**功能**：将工具异常转换为错误 ToolMessage

**执行时机**: `wrap_tool_call` / `awrap_tool_call`

**实现逻辑**:
```python
def wrap_tool_call(self, request, handler):
    try:
        return handler(request)
    except GraphBubbleUp:
        # 保留 LangGraph 控制流信号（interrupt/pause/resume）
        raise
    except Exception as exc:
        logger.exception(f"Tool execution failed: name={request.tool_call.get('name')}")

        # 构建错误 ToolMessage
        tool_name = str(request.tool_call.get("name") or "unknown_tool")
        tool_call_id = str(request.tool_call.get("id") or "missing_tool_call_id")
        detail = str(exc).strip() or exc.__class__.__name__

        if len(detail) > 500:
            detail = detail[:497] + "..."

        return ToolMessage(
            content=f"Error: Tool '{tool_name}' failed with {exc.__class__.__name__}: {detail}. Continue with available context, or choose an alternative tool.",
            tool_call_id=tool_call_id,
            name=tool_name,
            status="error",
        )
```

**关键特性**:
- 捕获所有异常（除了 `GraphBubbleUp`）
- 返回带有详细错误信息的 ToolMessage
- 错误消息截断到 500 字符

---

### 7. SummarizationMiddleware

**文件位置**: LangChain 内置 (`langchain.agents.middleware.SummarizationMiddleware`)

**功能**：自动对话摘要以减少 token 使用

**配置**: `config.yaml` → `summarization`

```yaml
summarization:
  enabled: true
  model_name: null  # None = 使用轻量级模型
  trigger:
    - type: messages
      value: 50  # 50 条消息时触发
    - type: tokens
      value: 4000  # 4000 tokens 时触发
    - type: fraction
      value: 0.8  # 80% 最大输入 tokens 时触发
  keep:
    type: messages
    value: 20  # 保留最近 20 条消息
  trim_tokens_to_summarize: 4000  # 摘要前最大 tokens
  summary_prompt: null  # 自定义摘要提示词
```

**工作流程**:
1. 检查是否满足任一触发条件
2. 将旧消息发送给 LLM 进行摘要
3. 保留最近的消息（根据 keep 策略）
4. 将摘要替换旧消息

---

### 8. TodoListMiddleware (TodoMiddleware)

**文件位置**: `agents/middlewares/todo_middleware.py`

**功能**：扩展 TodoListMiddleware，检测上下文丢失并注入提醒

**执行时机**: `before_model` (LLM 调用前)

**问题背景**:
当消息历史被截断（如 SummarizationMiddleware），原始 `write_todos` 工具调用可能被滚动出活动上下文窗口。

**实现逻辑**:
```python
def before_model(self, state, runtime):
    todos = state.get("todos") or []
    if not todos:
        return None

    messages = state.get("messages") or []

    # 1. 检查 write_todos 是否仍在上下文中
    if _todos_in_messages(messages):
        return None  # 仍可见，无需注入

    # 2. 检查是否已注入提醒
    if _reminder_in_messages(messages):
        return None  # 已注入，无需重复

    # 3. 注入提醒消息
    formatted = _format_todos(todos)  # "- [pending] task1\n- [in_progress] task2"
    reminder = HumanMessage(
        name="todo_reminder",
        content=(
            "<system_reminder>\n"
            "Your todo list from earlier is no longer visible in the current context window, "
            "but it is still active. Here is the current state:\n\n"
            f"{formatted}\n\n"
            "Continue tracking and updating this todo list as you work. "
            "Call `write_todos` whenever the status of any item changes.\n"
            "</system_reminder>"
        ),
    )

    return {"messages": [reminder]}
```

**关键方法**:
- `_todos_in_messages()`: 检查消息中是否包含 `write_todos` 工具调用
- `_reminder_in_messages()`: 检查是否已注入提醒消息
- `_format_todos()`: 格式化 todo 列表

---

### 9. TitleMiddleware

**文件位置**: `agents/middlewares/title_middleware.py`

**功能**：在第一次完整交互后自动生成线程标题

**执行时机**: `after_model` (LLM 调用后)

**实现逻辑**:
```python
def after_model(self, state, runtime):
    # 1. 检查是否应该生成标题
    if not self._should_generate_title(state):
        return None

    # 2. 构建提示词
    prompt, user_msg = self._build_title_prompt(state)

    # 3. 调用 LLM 生成标题
    config = get_title_config()
    model = create_chat_model(name=config.model_name, thinking_enabled=False)

    try:
        response = model.invoke(prompt)
        title = self._parse_title(response.content)
        if not title:
            title = self._fallback_title(user_msg)
    except Exception:
        logger.exception("Failed to generate title")
        title = self._fallback_title(user_msg)

    return {"title": title}

def _should_generate_title(self, state):
    config = get_title_config()
    if not config.enabled:
        return False

    # 检查是否已有标题
    if state.get("title"):
        return False

    # 检查是否为第一次完整交互
    messages = state.get("messages", [])
    user_messages = [m for m in messages if m.type == "human"]
    assistant_messages = [m for m in messages if m.type == "ai"]

    return len(user_messages) == 1 and len(assistant_messages) >= 1
```

**内容规范化**:
```python
def _normalize_content(self, content):
    """处理 str、list、dict 格式的内容"""
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        parts = [self._normalize_content(item) for item in content]
        return "\n".join(part for part in parts if part)
    if isinstance(content, dict):
        text_value = content.get("text")
        if text_value:
            return text_value
        nested = content.get("content")
        if nested:
            return self._normalize_content(nested)
    return ""
```

**配置**:
```yaml
title:
  enabled: true
  max_words: 6
  max_chars: 60
  model_name: null  # None = 使用默认模型
  prompt_template: "Generate a concise title (max {max_words} words)..."
```

---

### 10. MemoryMiddleware

**文件位置**: `agents/middlewares/memory_middleware.py`

**功能**：队列化对话用于记忆更新

**执行时机**: `after_agent` (Agent 执行后)

**实现逻辑**:
```python
def after_agent(self, state, runtime):
    config = get_memory_config()
    if not config.enabled:
        return None

    # 1. 获取 thread_id
    thread_id = runtime.context.get("thread_id")
    if not thread_id:
        return None

    # 2. 过滤消息（仅保留用户输入和最终 AI 响应）
    filtered_messages = _filter_messages_for_memory(state.get("messages", []))

    # 3. 检查是否有有意义的对话
    user_messages = [m for m in filtered_messages if getattr(m, "type") == "human"]
    assistant_messages = [m for m in filtered_messages if getattr(m, "type") == "ai"]

    if not user_messages or not assistant_messages:
        return None

    # 4. 队列化更新
    queue = get_memory_queue()
    queue.add(thread_id=thread_id, messages=filtered_messages, agent_name=self._agent_name)

    return None  # 无状态更新
```

**消息过滤逻辑**:
```python
def _filter_messages_for_memory(messages):
    """过滤消息以保留用户输入和最终 AI 响应

    排除：
    - Tool messages（工具调用结果）
    - 带有 tool_calls 的 AI 消息（中间步骤）
    - <uploaded_files> 块（会话范围，不应持久化）

    保留：
    - Human messages（移除 <uploaded_files> 后）
    - 不带 tool_calls 的 AI messages（最终响应）
    """
    _UPLOAD_BLOCK_RE = re.compile(r"<uploaded_files>[\s\S]*?</uploaded_files>\n*", re.IGNORECASE)

    filtered = []
    skip_next_ai = False

    for msg in messages:
        if msg.type == "human":
            content = str(msg.content)
            if "<uploaded_files>" in content:
                stripped = _UPLOAD_BLOCK_RE.sub("", content).strip()
                if not stripped:
                    # 整个回合是上传记录，跳过
                    skip_next_ai = True
                    continue
                # 重建消息并保留用户问题
                clean_msg = copy(msg)
                clean_msg.content = stripped
                filtered.append(clean_msg)
            else:
                filtered.append(msg)
            skip_next_ai = False

        elif msg.type == "ai":
            if not msg.tool_calls and not skip_next_ai:
                filtered.append(msg)
            # 跳过带 tool_calls 的 AI 消息
        # 跳过 Tool messages

    return filtered
```

**记忆队列** (`memory/queue.py`):
```python
class MemoryUpdateQueue:
    """带防抖机制的记忆更新队列"""

    def add(self, thread_id, messages, agent_name):
        """添加对话到更新队列"""
        context = ConversationContext(
            thread_id=thread_id,
            messages=messages,
            agent_name=agent_name,
        )

        with self._lock:
            # 替换同一线程的旧更新
            self._queue = [c for c in self._queue if c.thread_id != thread_id]
            self._queue.append(context)
            self._reset_timer()  # 重置防抖定时器

    def _reset_timer(self):
        """重置防抖定时器"""
        if self._timer:
            self._timer.cancel()

        config = get_memory_config()
        self._timer = threading.Timer(config.debounce_seconds, self._process_queue)
        self._timer.daemon = True
        self._timer.start()

    def _process_queue(self):
        """处理所有排队的对话上下文"""
        updater = MemoryUpdater()

        for context in contexts_to_process:
            updater.update_memory(
                messages=context.messages,
                thread_id=context.thread_id,
                agent_name=context.agent_name,
            )
            time.sleep(0.5)  # 避免速率限制
```

---

### 11. ViewImageMiddleware

**文件位置**: `agents/middlewares/view_image_middleware.py`

**功能**：当 view_image 工具完成后，在 LLM 调用前注入图片详情

**执行时机**: `before_model` (LLM 调用前)

**实现逻辑**:
```python
def before_model(self, state, runtime):
    # 1. 检查是否应该注入图片消息
    if not self._should_inject_image_message(state):
        return None

    # 2. 创建包含图片的混合内容消息
    image_content = self._create_image_details_message(state)

    human_msg = HumanMessage(content=image_content)

    return {"messages": [human_msg]}

def _should_inject_image_message(self, state):
    messages = state.get("messages", [])

    # 1. 获取最后一条 AI 消息
    last_assistant_msg = self._get_last_assistant_message(messages)
    if not last_assistant_msg:
        return False

    # 2. 检查是否包含 view_image 工具调用
    if not self._has_view_image_tool(last_assistant_msg):
        return False

    # 3. 检查所有工具是否已完成
    if not self._all_tools_completed(messages, last_assistant_msg):
        return False

    # 4. 检查是否已注入过图片详情消息
    assistant_idx = messages.index(last_assistant_msg)
    for msg in messages[assistant_idx + 1:]:
        if isinstance(msg, HumanMessage):
            content_str = str(msg.content)
            if "Here are the images you've viewed" in content_str:
                return False  # 已注入

    return True

def _create_image_details_message(self, state):
    viewed_images = state.get("viewed_images", {})

    content_blocks = [
        {"type": "text", "text": "Here are the images you've viewed:"}
    ]

    for image_path, image_data in viewed_images.items():
        mime_type = image_data.get("mime_type", "unknown")
        base64_data = image_data.get("base64", "")

        # 添加文本描述
        content_blocks.append({"type": "text", "text": f"\n- **{image_path}** ({mime_type})"})

        # 添加实际图片数据（LLM 可以"看到"）
        if base64_data:
            content_blocks.append({
                "type": "image_url",
                "image_url": {"url": f"data:{mime_type};base64,{base64_data}"},
            })

    return content_blocks
```

**消息格式**:
```
[
    {"type": "text", "text": "Here are the images you've viewed:"},
    {"type": "text", "text": "\n- **/mnt/user-data/uploads/image.png** (image/png)"},
    {"type": "image_url", "image_url": {"url": "data:image/png;base64,iVBORw0KGgoAAAANS..."}}
]
```

---

### 12. SubagentLimitMiddleware

**文件位置**: `agents/middlewares/subagent_limit_middleware.py`

**功能**：截断模型响应中多余的并发子代理调用

**执行时机**: `after_model` (LLM 调用后)

**限制范围**: `[2, 4]`（可通过构造函数覆盖）

**实现逻辑**:
```python
def after_model(self, state, runtime):
    messages = state.get("messages", [])
    last_msg = messages[-1]

    if getattr(last_msg, "type") != "ai":
        return None

    tool_calls = getattr(last_msg, "tool_calls", None)
    if not tool_calls:
        return None

    # 1. 统计 task 工具调用
    task_indices = [i for i, tc in enumerate(tool_calls) if tc.get("name") == "task"]

    if len(task_indices) <= self.max_concurrent:
        return None  # 未超限

    # 2. 构建要丢弃的索引集合
    indices_to_drop = set(task_indices[self.max_concurrent :])

    # 3. 创建截断后的工具调用列表
    truncated_tool_calls = [tc for i, tc in enumerate(tool_calls) if i not in indices_to_drop]

    logger.warning(f"Truncated {len(indices_to_drop)} excess task tool call(s)")

    # 4. 替换 AIMessage（使用相同 ID 触发替换）
    updated_msg = last_msg.model_copy(update={"tool_calls": truncated_tool_calls})

    return {"messages": [updated_msg]}
```

**关键特性**:
- 仅截断 `task` 工具调用，不影响其他工具
- 使用 `model_copy()` 和 `update=` 创建新消息
- 保持原始消息 ID 确保正确替换

---

### 13. LoopDetectionMiddleware

**文件位置**: `agents/middlewares/loop_detection_middleware.py`

**功能**：检测和打破重复的工具调用循环

**执行时机**: `after_model` (LLM 调用后)

**检测策略**:
1. 在每次模型响应后，哈希工具调用（name + args）
2. 在滑动窗口中跟踪最近的哈希
3. 如果相同哈希出现 ≥ warn_threshold 次，注入警告消息
4. 如果出现 ≥ hard_limit 次，移除所有 tool_calls 强制文本输出

**实现逻辑**:
```python
def _hash_tool_calls(tool_calls):
    """确定性地哈希一组工具调用"""
    # 1. 标准化每个工具调用
    normalized = [
        {"name": tc["name"], "args": tc["args"]}
        for tc in tool_calls
    ]

    # 2. 排序以确保顺序无关性
    normalized.sort(
        key=lambda tc: (tc["name"], json.dumps(tc["args"], sort_keys=True))
    )

    # 3. MD5 哈希
    blob = json.dumps(normalized, sort_keys=True)
    return hashlib.md5(blob.encode()).hexdigest()[:12]

def _track_and_check(self, state, runtime):
    thread_id = self._get_thread_id(runtime)
    call_hash = _hash_tool_calls(last_msg.tool_calls)

    with self._lock:
        # LRU 更新
        if thread_id in self._history:
            self._history.move_to_end(thread_id)
        else:
            self._history[thread_id] = []
            self._evict_if_needed()  # 超过 100 个线程时驱逐最旧的

        history = self._history[thread_id]
        history.append(call_hash)
        if len(history) > self.window_size:
            history[:] = history[-self.window_size:]

        count = history.count(call_hash)

        if count >= self.hard_limit:
            return _HARD_STOP_MSG, True

        if count >= self.warn_threshold:
            warned = self._warned[thread_id]
            if call_hash not in warned:
                warned.add(call_hash)
                return _WARNING_MSG, False

    return None, False

def after_model(self, state, runtime):
    warning, hard_stop = self._track_and_check(state, runtime)

    if hard_stop:
        # 移除 tool_calls 强制文本输出
        messages = state.get("messages", [])
        last_msg = messages[-1]
        stripped_msg = last_msg.model_copy(update={
            "tool_calls": [],
            "content": (last_msg.content or "") + f"\n\n{_HARD_STOP_MSG}",
        })
        return {"messages": [stripped_msg]}

    if warning:
        # 注入警告消息（使用 HumanMessage 避免 Anthropic 错误）
        return {"messages": [HumanMessage(content=warning)]}

    return None
```

**默认值**:
- `warn_threshold`: 3（3 次重复后警告）
- `hard_limit`: 5（5 次重复后强制停止）
- `window_size`: 20（跟踪最近 20 次调用）
- `max_tracked_threads`: 100（LRU 驱逐限制）

**线程安全**:
- 使用 `threading.Lock` 保护历史记录
- 使用 `OrderedDict` 实现 LRU 驱逐

---

### 14. ClarificationMiddleware

**文件位置**: `agents/middlewares/clarification_middleware.py`

**功能**：拦截澄清请求并中断执行，向用户展示问题

**执行时机**: `wrap_tool_call` (工具调用包装)

**实现逻辑**:
```python
def wrap_tool_call(self, request, handler):
    # 1. 检查是否为 ask_clarification 工具调用
    if request.tool_call.get("name") != "ask_clarification":
        return handler(request)

    # 2. 提取澄清参数
    args = request.tool_call.get("args", {})
    question = args.get("question", "")
    clarification_type = args.get("clarification_type", "missing_info")
    context = args.get("context")
    options = args.get("options", [])

    # 3. 格式化澄清消息
    formatted_message = self._format_clarification_message(args)

    # 4. 创建 ToolMessage
    tool_message = ToolMessage(
        content=formatted_message,
        tool_call_id=request.tool_call.get("id", ""),
        name="ask_clarification",
    )

    # 5. 返回中断命令
    return Command(
        update={"messages": [tool_message]},
        goto=END,  # 中断执行
    )
```

**消息格式化**:
```python
def _format_clarification_message(self, args):
    type_icons = {
        "missing_info": "❓",
        "ambiguous_requirement": "🤔",
        "approach_choice": "🔀",
        "risk_confirmation": "⚠️",
        "suggestion": "💡",
    }

    icon = type_icons.get(args["clarification_type"], "❓")
    question = args["question"]
    context = args.get("context")
    options = args.get("options", [])

    message_parts = []

    if context:
        message_parts.append(f"{icon} {context}")
        message_parts.append(f"\n{question}")
    else:
        message_parts.append(f"{icon} {question}")

    if options:
        message_parts.append("")
        for i, option in enumerate(options, 1):
            message_parts.append(f"  {i}. {option}")

    return "\n".join(message_parts)
```

**关键特性**:
- 使用 `Command(goto=END)` 中断执行
- 返回的 ToolMessage 包含格式化的澄清消息
- 前端检测 `ask_clarification` 工具消息并显示

---

### 15. DeferredToolFilterMiddleware

**文件位置**: `agents/middlewares/deferred_tool_filter_middleware.py`

**功能**：从模型绑定中过滤延迟工具的架构

**执行时机**: `wrap_model_call` (LLM 调用包装)

**问题背景**:
当启用 tool_search 时，MCP 工具注册在 DeferredToolRegistry 中并传递给 ToolNode 执行，但其架构不应发送给 LLM（节省上下文 token）。

**实现逻辑**:
```python
def wrap_model_call(self, request, handler):
    registry = get_deferred_registry()
    if not registry:
        return handler(request)

    # 1. 获取延迟工具名称集合
    deferred_names = {e.name for e in registry.entries}

    # 2. 过滤出活跃工具
    active_tools = [
        t for t in request.tools
        if getattr(t, "name", None) not in deferred_names
    ]

    if len(active_tools) < len(request.tools):
        logger.debug(f"Filtered {len(request.tools) - len(active_tools)} deferred tool schema(s)")

    # 3. 使用过滤后的工具调用
    return handler(request.override(tools=active_tools))
```

**关键特性**:
- ToolNode 仍包含所有工具（包括延迟的）用于执行路由
- LLM 只看到活跃工具架构
- 延迟工具通过 `tool_search` 工具在运行时发现

---

### 16. TokenUsageMiddleware

**文件位置**: `agents/middlewares/token_usage_middleware.py`

**功能**：记录 LLM token 使用情况

**执行时机**: `after_model` (LLM 调用后)

**实现逻辑**:
```python
def after_model(self, state, runtime):
    messages = state.get("messages", [])
    if not messages:
        return None

    last = messages[-1]
    usage = getattr(last, "usage_metadata", None)

    if usage:
        logger.info(
            "LLM token usage: input=%s output=%s total=%s",
            usage.get("input_tokens", "?"),
            usage.get("output_tokens", "?"),
            usage.get("total_tokens", "?"),
        )

    return None  # 无状态更新
```

**配置**: `config.yaml` → `token_usage`

```yaml
token_usage:
  enabled: true
```

---

## 中间件组装器

### 构建函数

**文件位置**: `agents/middlewares/tool_error_handling_middleware.py`

```python
def _build_runtime_middlewares(
    *,
    include_uploads: bool,
    include_dangling_tool_call_patch: bool,
    lazy_init: bool = True,
) -> list[AgentMiddleware]:
    """构建共享的基础中间件"""
    middlewares = [
        ThreadDataMiddleware(lazy_init=lazy_init),
        SandboxMiddleware(lazy_init=lazy_init),
    ]

    if include_uploads:
        middlewares.insert(1, UploadsMiddleware())

    if include_dangling_tool_call_patch:
        middlewares.append(DanglingToolCallMiddleware())

    # Guardrail middleware（如果配置）
    guardrails_config = get_guardrails_config()
    if guardrails_config.enabled and guardrails_config.provider:
        provider_cls = resolve_variable(guardrails_config.provider.use)
        provider_kwargs = dict(guardrails_config.provider.config) or {}
        provider = provider_cls(**provider_kwargs)
        middlewares.append(GuardrailMiddleware(
            provider,
            fail_closed=guardrails_config.fail_closed,
            passport=guardrails_config.passport,
        ))

    middlewares.append(ToolErrorHandlingMiddleware())
    return middlewares

def build_lead_runtime_middlewares(*, lazy_init: bool = True) -> list[AgentMiddleware]:
    """Lead Agent 的共享中间件"""
    return _build_runtime_middlewares(
        include_uploads=True,
        include_dangling_tool_call_patch=True,
        lazy_init=lazy_init,
    )

def build_subagent_runtime_middlewares(*, lazy_init: bool = True) -> list[AgentMiddleware]:
    """子代理的共享中间件"""
    return _build_runtime_middlewares(
        include_uploads=False,
        include_dangling_tool_call_patch=False,
        lazy_init=lazy_init,
    )
```

### Lead Agent 完整中间件链

**文件位置**: `agents/lead_agent/agent.py`

```python
def _build_middlewares(config, model_name, agent_name):
    """构建中间件链"""
    middlewares = build_lead_runtime_middlewares(lazy_init=True)

    # SummarizationMiddleware（如果启用）
    summarization_middleware = _create_summarization_middleware()
    if summarization_middleware:
        middlewares.append(summarization_middleware)

    # TodoListMiddleware（如果 plan_mode）
    is_plan_mode = config.get("configurable", {}).get("is_plan_mode", False)
    todo_middleware = _create_todo_list_middleware(is_plan_mode)
    if todo_middleware:
        middlewares.append(todo_middleware)

    # TokenUsageMiddleware（如果启用）
    if get_app_config().token_usage.enabled:
        middlewares.append(TokenUsageMiddleware())

    # TitleMiddleware
    middlewares.append(TitleMiddleware())

    # MemoryMiddleware
    middlewares.append(MemoryMiddleware(agent_name=agent_name))

    # ViewImageMiddleware（仅视觉模型）
    app_config = get_app_config()
    model_config = app_config.get_model_config(model_name)
    if model_config and model_config.supports_vision:
        middlewares.append(ViewImageMiddleware())

    # DeferredToolFilterMiddleware（如果 tool_search）
    if app_config.tool_search.enabled:
        middlewares.append(DeferredToolFilterMiddleware())

    # SubagentLimitMiddleware（如果子代理启用）
    subagent_enabled = config.get("configurable", {}).get("subagent_enabled", False)
    if subagent_enabled:
        max_concurrent = config.get("configurable", {}).get("max_concurrent_subagents", 3)
        middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent))

    # LoopDetectionMiddleware
    middlewares.append(LoopDetectionMiddleware())

    # ClarificationMiddleware（必须最后）
    middlewares.append(ClarificationMiddleware())

    return middlewares
```

---

## 中间件钩子点

### AgentMiddleware 钩子方法

```python
class AgentMiddleware(Generic[StateT]):
    def before_agent(
        self, state: StateT, runtime: Runtime
    ) -> dict | None:
        """在 Agent 执行前调用

        返回 dict: 合并到 state
        返回 None: 无状态更新
        """

    def after_agent(
        self, state: StateT, runtime: Runtime
    ) -> dict | None:
        """在 Agent 执行后调用

        返回 dict: 合并到 state
        返回 None: 无状态更新
        """

    def before_model(
        self, state: StateT, runtime: Runtime
    ) -> dict | None:
        """在 LLM 调用前调用

        返回 dict: 合并到 state
        返回 None: 无状态更新
        """

    def after_model(
        self, state: StateT, runtime: Runtime
    ) -> dict | None:
        """在 LLM 调用后调用

        返回 dict: 合并到 state
        返回 None: 无状态更新
        """

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelCallResult:
        """包装 LLM 调用（同步）

        可以修改 request 或直接返回结果
        """

    async def awrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], Awaitable[ModelResponse]],
    ) -> ModelCallResult:
        """包装 LLM 调用（异步）"""

    def wrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        """包装工具调用（同步）

        可以拦截工具调用或直接执行
        """

    async def awrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], Awaitable[ToolMessage | Command]],
    ) -> ToolMessage | Command:
        """包装工具调用（异步）"""
```

### 执行流程

```python
# 1. Agent 开始
for middleware in middlewares:
    middleware.before_agent(state, runtime)

# 2. LLM 调用循环
while not done:
    for middleware in middlewares:
        middleware.before_model(state, runtime)

    # wrap_model_call
    for middleware in reversed(middlewares):
        # 中间件可以包装处理器
        # 最后调用的中间件最先执行
        ...

    response = model.invoke(messages)

    for middleware in middlewares:
        middleware.after_model(state, runtime)

    # 工具调用
    for tool_call in response.tool_calls:
        for middleware in reversed(middlewares):
            # wrap_tool_call
            ...

        tool_result = execute_tool(tool_call)

# 3. Agent 结束
for middleware in middlewares:
    middleware.after_agent(state, runtime)
```

---

## 关联的配置系统

### Title 配置

**文件位置**: `config/title_config.py`

```python
class TitleConfig(BaseModel):
    enabled: bool = True
    max_words: int = Field(default=6, ge=1, le=20)
    max_chars: int = Field(default=60, ge=10, le=200)
    model_name: str | None = None
    prompt_template: str = Field(
        default="Generate a concise title (max {max_words} words)..."
    )
```

### Summarization 配置

**文件位置**: `config/summarization_config.py`

```python
class ContextSize(BaseModel):
    type: Literal["fraction", "tokens", "messages"]
    value: int | float

class SummarizationConfig(BaseModel):
    enabled: bool = False
    model_name: str | None = None
    trigger: ContextSize | list[ContextSize] | None
    keep: ContextSize
    trim_tokens_to_summarize: int | None = 4000
    summary_prompt: str | None = None
```

### Guardrails 配置

**文件位置**: `config/guardrails_config.py`

```python
class GuardrailProviderConfig(BaseModel):
    use: str  # 类路径
    config: dict[str, Any] | None  # 提供者特定配置

class GuardrailsConfig(BaseModel):
    enabled: bool = False
    fail_closed: bool = True
    passport: str | None = None
    provider: GuardrailProviderConfig | None = None
```

### Memory 配置

**文件位置**: `config/memory_config.py`

```python
class MemoryConfig(BaseModel):
    enabled: bool = False
    injection_enabled: bool = False
    storage_path: str | None = None
    debounce_seconds: int = 30
    model_name: str | None = None
    max_facts: int = 100
    fact_confidence_threshold: float = 0.7
    max_injection_tokens: int = 2000
```

---

## 后端关联点

### 1. Gateway API 集成

**文件位置**: `app/gateway/routers/`

中间件通过 LangGraph Server 间接影响 Gateway API：

- **ThreadDataMiddleware**: 影响文件上传端点的目录结构
- **TitleMiddleware**: 标题通过 SSE 事件流式传输
- **MemoryMiddleware**: 记忆更新通过后台线程异步处理

### 2. 前端集成

**文件位置**: `frontend/src/core/`

前端通过 SSE 事件接收中间件产生的状态更新：

```typescript
// 标题事件
on("values", (data) => {
  if (data.title) {
    setThreadTitle(data.title);
  }
});

// Artifact 事件
on("values", (data) => {
  if (data.artifacts) {
    setArtifacts(data.artifacts);
  }
});

// Todo 更新
on("values", (data) => {
  if (data.todos) {
    setTodos(data.todos);
  }
});
```

### 3. 测试覆盖

**测试文件**:
- `tests/test_thread_data_middleware.py`
- `tests/test_dangling_tool_call_middleware.py`
- `tests/test_title_middleware_core_logic.py`
- `tests/test_uploads_middleware_core_logic.py`
- `tests/test_guardrail_middleware.py`
- `tests/test_subagent_limit_middleware.py`
- `tests/test_loop_detection_middleware.py`
- `tests/test_todo_middleware.py`
- `tests/test_tool_error_handling_middleware.py`

---

## 中间件开发指南

### 创建自定义中间件

```python
from langchain.agents.middleware import AgentMiddleware
from langchain.agents import AgentState
from langgraph.runtime import Runtime

class MyCustomMiddleware(AgentMiddleware[AgentState]):
    """自定义中间件示例"""

    state_schema = AgentState  # 或自定义 State 模式

    def __init__(self, param1: str = "default"):
        super().__init__()
        self.param1 = param1

    def before_agent(self, state: AgentState, runtime: Runtime) -> dict | None:
        """Agent 执行前调用"""
        print(f"[MyCustomMiddleware] before_agent: {runtime.context.get('thread_id')}")
        return None  # 无状态更新

    def after_model(self, state: AgentState, runtime: Runtime) -> dict | None:
        """LLM 调用后调用"""
        print(f"[MyCustomMiddleware] after_model: message count = {len(state.get('messages', []))}")
        return None

    async def awrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], Awaitable[ToolMessage | Command]],
    ) -> ToolMessage | Command:
        """包装工具调用（异步）"""
        print(f"[MyCustomMiddleware] tool call: {request.tool_call.get('name')}")
        return await handler(request)
```

### 注册中间件

在 `agents/lead_agent/agent.py` 中：

```python
def _build_middlewares(config, model_name, agent_name):
    middlewares = build_lead_runtime_middlewares(lazy_init=True)

    # 添加自定义中间件
    middlewares.append(MyCustomMiddleware(param1="value"))

    # 确保顺序正确
    # ClarificationMiddleware 必须最后
    middlewares.append(ClarificationMiddleware())

    return middlewares
```

### 中间件最佳实践

1. **保持幂等性**：中间件应该是幂等的，多次调用不应产生副作用
2. **使用状态模式**：定义清晰的 `state_schema` 以获得类型安全
3. **错误处理**：捕获异常并记录日志，避免中断整个执行流程
4. **性能考虑**：避免在中间件中执行耗时操作
5. **线程安全**：如果使用共享状态，确保线程安全
6. **日志记录**：添加详细的日志以便调试
7. **返回值约定**：返回 `None` 表示无状态更新，返回 `dict` 表示要合并的更新

---

## 总结

DeerFlow 中间件系统提供了强大的扩展能力：

1. **模块化设计**：每个中间件专注于单一职责
2. **严格的执行顺序**：通过精心设计的顺序确保依赖关系
3. **灵活的钩子点**：支持在 Agent、Model、Tool 调用前后插入逻辑
4. **配置驱动**：大多数中间件支持通过配置启用/禁用
5. **类型安全**：使用 Pydantic 模型和泛型确保类型安全
6. **异步支持**：所有钩子都提供同步和异步版本
7. **线程安全**：正确处理并发场景

通过理解中间件系统，开发者可以轻松扩展 DeerFlow 的功能，而无需修改核心 Agent 代码。
