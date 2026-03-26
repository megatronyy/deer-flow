# DeerFlow Harness 架构详细分析

## 概述

DeerFlow Harness 是一个可发布的 AI Agent 框架包（`deerflow-harness`），提供了构建和运行 Agent 所需的全部核心能力：Agent 编排、工具、沙箱、模型、MCP、技能、配置等。它位于 `packages/harness/deerflow/` 目录下，使用 `deerflow.*` 作为导入前缀。

## 模块结构图

```
deerflow/
├── agents/              # Agent 系统（核心）
│   ├── lead_agent/      # Lead Agent 工厂和系统提示词
│   ├── middlewares/     # 12个中间件组件
│   ├── memory/          # 长期记忆系统
│   ├── checkpointer/    # 状态持久化
│   └── thread_state.py  # ThreadState 数据模型
├── sandbox/             # 沙箱执行系统
│   ├── local/           # 本地文件系统实现
│   ├── sandbox.py       # 抽象接口
│   ├── sandbox_provider.py # 提供者模式
│   ├── middleware.py    # 沙箱生命周期管理
│   └── tools.py         # bash、ls、read/write/str_replace
├── subagents/           # 子代理委托系统
│   ├── builtins/        # general-purpose、bash agents
│   ├── executor.py      # 后台执行引擎
│   ├── config.py        # 子代理配置
│   └── registry.py      # Agent 注册表
├── tools/               # 工具系统
│   ├── builtins/        # 内置工具
│   └── tools.py         # 工具组装器
├── mcp/                 # MCP 集成
│   ├── client.py        # MCP 客户端
│   ├── tools.py         # MCP 工具加载器
│   ├── cache.py         # 工具缓存（mtime失效）
│   └── oauth.py         # OAuth 令牌处理
├── models/              # 模型工厂
│   ├── factory.py       # 模型创建器
│   ├── claude_provider.py # Claude Code OAuth
│   └── openai_codex_provider.py # Codex CLI 支持
├── skills/              # 技能系统
│   ├── loader.py        # 技能发现和加载
│   ├── parser.py        # SKILL.md 解析器
│   ├── installer.py     # .skill 安装器
│   └── types.py         # Skill 数据模型
├── config/              # 配置系统
│   ├── app_config.py    # 主配置加载器（config.yaml）
│   ├── model_config.py  # 模型配置
│   ├── sandbox_config.py # 沙箱配置
│   ├── extensions_config.py # MCP 和技能配置
│   └── ...              # 其他配置模块
├── guardrails/          # 工具调用授权
│   ├── provider.py      # GuardrailProvider 协议
│   ├── builtin.py       # 内置 AllowlistProvider
│   └── middleware.py    # 中间件集成
├── reflection/          # 动态模块加载
│   └── resolvers.py     # resolve_variable/resolve_class
├── community/           # 社区工具集成
│   ├── tavily/          # 网络搜索
│   ├── jina_ai/         # Jina Reader API
│   ├── firecrawl/       # 网页抓取
│   ├── image_search/    # 图片搜索
│   ├── aio_sandbox/     # Docker 沙箱
│   └── infoquest/       # InfoQuest 集成
├── uploads/             # 文件上传系统
│   └── manager.py       # 上传管理器（纯业务逻辑）
├── utils/               # 工具函数
└── client.py            # 嵌入式 Python 客户端
```

---

## 核心模块详细分析

### 1. Agent 系统 (`agents/`)

#### 1.1 ThreadState (`thread_state.py`)

**功能**：定义 Agent 的状态数据结构

**关键特性**：
- 继承自 LangChain 的 `AgentState`
- 扩展字段：
  - `sandbox`: 沙箱状态（sandbox_id）
  - `thread_data`: 线程数据（workspace、uploads、outputs 路径）
  - `title`: 会话标题
  - `artifacts`: 产出的文件列表（带去重合并器）
  - `todos`: 待办事项列表
  - `uploaded_files`: 上传的文件列表
  - `viewed_images`: 查看的图片（base64、mime_type）

**自定义 Reducer**：
- `merge_artifacts`: 合并 artifacts 列表并去重
- `merge_viewed_images`: 合并图片字典，空字典表示清除所有

#### 1.2 Lead Agent (`lead_agent/agent.py`)

**功能**：主 Agent 工厂函数 `make_lead_agent()`

**执行流程**：
```python
def make_lead_agent(config: RunnableConfig):
    # 1. 解析模型名称（支持运行时覆盖、fallback）
    model_name = _resolve_model_name(requested_model_name)

    # 2. 加载模型配置
    model_config = get_app_config().get_model_config(model_name)

    # 3. 检查 thinking 模式兼容性
    if thinking_enabled and not model_config.supports_thinking:
        logger.warning("Model does not support thinking, fallback to non-thinking mode")
        thinking_enabled = False

    # 4. 注入运行元数据（用于 LangSmith 追踪）
    config["metadata"].update({
        "agent_name": agent_name or "default",
        "model_name": model_name,
        "thinking_enabled": thinking_enabled,
        ...
    })

    # 5. 构建中间件链（按严格顺序）
    middlewares = _build_middlewares(config, model_name, agent_name)

    # 6. 获取可用工具（沙箱、内置、MCP、社区、子代理）
    tools = get_available_tools(model_name, subagent_enabled)

    # 7. 生成系统提示词（技能、记忆、子代理）
    system_prompt = apply_prompt_template(subagent_enabled, ...)

    # 8. 创建 Agent
    return create_agent(
        model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled),
        tools=tools,
        middleware=middlewares,
        system_prompt=system_prompt,
        state_schema=ThreadState,
    )
```

**中间件链顺序**（严格依赖关系）：
```python
def _build_middlewares(config, model_name, agent_name):
    middlewares = [
        # 1. ThreadDataMiddleware - 创建线程目录
        # 2. UploadsMiddleware - 跟踪上传文件
        # 3. SandboxMiddleware - 获取沙箱环境
        # 4. DanglingToolCallMiddleware - 修复中断的工具调用
        # 5. GuardrailMiddleware - 工具调用授权（可选）
        # 6. SummarizationMiddleware - 上下文压缩（可选）
        # 7. TodoListMiddleware - 任务跟踪（可选）
        # 8. TitleMiddleware - 自动生成标题
        # 9. MemoryMiddleware - 队列化记忆更新
        # 10. ViewImageMiddleware - 注入 base64 图片（仅视觉模型）
        # 11. SubagentLimitMiddleware - 限制并发子代理数（可选）
        # 12. LoopDetectionMiddleware - 检测循环
        # 13. ClarificationMiddleware - 拦截澄清请求（必须最后）
    ]
    return middlewares
```

#### 1.3 系统提示词生成 (`lead_agent/prompt.py`)

**功能**：动态生成 Agent 系统提示词

**组成部分**：
```python
SYSTEM_PROMPT_TEMPLATE = """
<role>
You are {agent_name}, an open-source super agent.
</role>

{soul}                           # Agent 个性（SOUL.md）
{memory_context}                 # 长期记忆注入

<thinking_style>
- Think concisely and strategically
- **PRIORITY CHECK: clarify unclear/missing/ambiguous requirements FIRST**
{subagent_thinking}              # 子代理分解指导
- Always respond after thinking
</thinking_style>

<clarification_system>
**WORKFLOW PRIORITY: CLARIFY → PLAN → ACT**
1. FIRST: Analyze request - identify what's unclear
2. SECOND: If clarification needed, call ask_clarification IMMEDIATELY
3. THIRD: Only after clarifications resolved, proceed

**MANDATORY Clarification Scenarios:**
1. Missing Information (missing_info)
2. Ambiguous Requirements (ambiguous_requirement)
3. Approach Choices (approach_choice)
4. Risky Operations (risk_confirmation)
5. Suggestions (suggestion)
</clarification_system>

{skills_section}                 # 可用技能列表
{deferred_tools_section}         # 延迟加载工具名称
{subagent_section}               # 子代理系统说明

<working_directory>
- User uploads: /mnt/user-data/uploads
- User workspace: /mnt/user-data/workspace
- Output files: /mnt/user-data/outputs
</working_directory>

<response_style>
- Clear and Concise
- Natural Tone
- Action-Oriented
</response_style>

<citations>
**CRITICAL: Always include citations when using web search results**
- Format: [citation:TITLE](URL)
- Include "Sources" section at end
</citations>

<critical_reminders>
- Clarification First
{subagent_reminder}               # 并发限制提醒
- Skill First: Load skills before complex tasks
- Progressive Loading
- Language Consistency
- Always Respond
</critical_reminders>

<current_date>{current_date}</current_date>
"""
```

**动态内容生成**：
- `memory_context`: 从记忆文件加载，token 限制内注入 top 15 facts
- `skills_section`: 扫描 skills/ 目录，生成技能列表
- `subagent_section`: 根据并发限制动态生成（如 `MAX 3 task calls per response`）
- `deferred_tools_section`: tool_search 启用时列出延迟工具名称

---

### 2. 沙箱系统 (`sandbox/`)

#### 2.1 抽象接口 (`sandbox.py`)

**Sandbox 抽象类**：
```python
class Sandbox(ABC):
    _id: str

    @abstractmethod
    def execute_command(self, command: str) -> str:
        """执行 bash 命令"""

    @abstractmethod
    def read_file(self, path: str) -> str:
        """读取文件内容"""

    @abstractmethod
    def list_dir(self, path: str, max_depth=2) -> list[str]:
        """列出目录内容"""

    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None:
        """写入文件"""

    @abstractmethod
    def update_file(self, path: str, content: bytes) -> None:
        """更新文件（二进制）"""
```

#### 2.2 提供者模式 (`sandbox_provider.py`)

**SandboxProvider 抽象类**：
```python
class SandboxProvider(ABC):
    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str:
        """获取沙箱环境，返回 ID"""

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None:
        """通过 ID 获取沙箱"""

    @abstractmethod
    def release(self, sandbox_id: str) -> None:
        """释放沙箱"""
```

**单例管理**：
- `get_sandbox_provider()`: 获取缓存的提供者实例
- `reset_sandbox_provider()`: 重置缓存（不调用 shutdown）
- `shutdown_sandbox_provider()`: 正确关闭并清理
- `set_sandbox_provider()`: 注入自定义提供者（测试用）

**动态加载**：
```python
def get_sandbox_provider(**kwargs) -> SandboxProvider:
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls(**kwargs)
    return _default_sandbox_provider
```

#### 2.3 沙箱中间件 (`middleware.py`)

**生命周期管理**：
```python
class SandboxMiddleware(AgentMiddleware):
    def __init__(self, lazy_init: bool = True):
        """
        Args:
            lazy_init: True - 首次工具调用时获取沙箱（默认）
                      False - 首次 Agent 调用时获取沙箱
        """

    def before_agent(self, state, runtime):
        if not self._lazy_init:
            # Eager initialization
            sandbox_id = self._acquire_sandbox(thread_id)
            return {"sandbox": {"sandbox_id": sandbox_id}}

    def after_agent(self, state, runtime):
        # 释放沙箱（注意：当前实现会立即释放）
        sandbox_id = state.get("sandbox", {}).get("sandbox_id")
        if sandbox_id:
            get_sandbox_provider().release(sandbox_id)
```

**注意**：当前实现在 `after_agent` 中释放沙箱，这意味着每次 Agent 调用后都会释放。对于需要保持沙箱状态的场景，可能需要调整此行为。

#### 2.4 虚拟路径系统 (`tools.py`)

**路径映射**：
```
虚拟路径（容器内）          物理路径（宿主机）
/mnt/user-data/workspace  → backend/.deer-flow/threads/{thread_id}/user-data/workspace
/mnt/user-data/uploads    → backend/.deer-flow/threads/{thread_id}/user-data/uploads
/mnt/user-data/outputs    → backend/.deer-flow/threads/{thread_id}/user-data/outputs
/mnt/skills               → deer-flow/skills/
```

**核心函数**：
- `replace_virtual_path()`: 将虚拟路径转换为物理路径
- `replace_virtual_paths_in_command()`: 替换命令中的所有虚拟路径
- `is_local_sandbox()`: 检查是否为本地沙箱（`sandbox_id == "local"`）

---

### 3. 配置系统 (`config/`)

#### 3.1 AppConfig (`app_config.py`)

**配置优先级**：
```
1. config_path 参数
2. DEER_FLOW_CONFIG_PATH 环境变量
3. 当前目录的 config.yaml
4. 父目录的 config.yaml（推荐位置：项目根目录）
```

**自动重新加载**：
```python
def get_app_config() -> AppConfig:
    """自动检测配置文件变化并重新加载"""
    global _app_config, _app_config_path, _app_config_mtime

    resolved_path = AppConfig.resolve_config_path()
    current_mtime = _get_config_mtime(resolved_path)

    should_reload = (
        _app_config is None
        or _app_config_path != resolved_path
        or _app_config_mtime != current_mtime
    )
    if should_reload:
        _load_and_cache_app_config(str(resolved_path))
    return _app_config
```

**环境变量解析**：
- 配置值以 `$` 开头时解析为环境变量（如 `$OPENAI_API_KEY`）
- 递归解析嵌套的字典和列表

**配置版本检查**：
```python
def _check_config_version(config_data, config_path):
    user_version = int(config_data.get("config_version", 0))
    example_version = int(load_example_yaml().get("config_version", 0))
    if user_version < example_version:
        logger.warning("Config is outdated. Run `make config-upgrade`")
```

**配置结构**：
```yaml
models: []           # LLM 配置
tools: []            # 工具配置
tool_groups: []      # 工具组
sandbox: {...}       # 沙箱配置
skills: {...}        # 技能路径
extensions: {...}    # MCP 服务器和技能状态
tool_search: {...}   # 工具搜索/延迟加载
token_usage: {...}   # Token 使用追踪
checkpointer: {...}  # 状态持久化配置
```

---

### 4. 模型工厂 (`models/`)

#### 4.1 create_chat_model (`factory.py`)

**动态模型创建**：
```python
def create_chat_model(name: str, thinking_enabled: bool = False, **kwargs) -> BaseChatModel:
    # 1. 获取模型配置
    model_config = get_app_config().get_model_config(name)

    # 2. 通过反射解析模型类
    model_class = resolve_class(model_config.use, BaseChatModel)

    # 3. 提取配置参数（排除元数据字段）
    model_settings = model_config.model_dump(
        exclude_none=True,
        exclude={"use", "name", "display_name", "supports_thinking", ...}
    )

    # 4. 处理 thinking 模式
    if thinking_enabled and model_config.when_thinking_enabled:
        model_settings.update(model_config.when_thinking_enabled)
    elif not thinking_enabled:
        # 显式禁用 thinking
        kwargs.update({"thinking": {"type": "disabled"}})

    # 5. Codex 特殊处理（不支持 max_tokens）
    if issubclass(model_class, CodexChatModel):
        model_settings.pop("max_tokens", None)
        # 映射 thinking 模式到 reasoning_effort
        model_settings["reasoning_effort"] = resolve_reasoning_effort(...)

    # 6. 附加 LangSmith 追踪器
    if is_tracing_enabled():
        model_instance.callbacks.append(LangChainTracer(...))

    return model_class(**model_settings, **kwargs)
```

**支持的提供者**：
- LangChain 标准：`langchain_openai:ChatOpenAI`, `langchain_anthropic:ChatAnthropic`
- CLI 支持：
  - Codex CLI: `deerflow.models.openai_codex_provider:CodexChatModel`
  - Claude Code OAuth: `deerflow.models.claude_provider:ClaudeChatModel`

---

### 5. 工具系统 (`tools/`)

#### 5.1 get_available_tools (`tools.py`)

**工具组装流程**：
```python
def get_available_tools(groups, include_mcp, model_name, subagent_enabled):
    tools = []

    # 1. 配置定义的工具（通过 resolve_variable 解析）
    loaded_tools = [
        resolve_variable(tool.use, BaseTool)
        for tool in config.tools
        if groups is None or tool.group in groups
    ]

    # 2. 内置工具
    builtin_tools = [
        present_file_tool,      # 展示输出文件
        ask_clarification_tool, # 请求澄清
    ]

    # 3. 子代理工具（如果启用）
    if subagent_enabled:
        builtin_tools.append(task_tool)

    # 4. 视觉模型专用工具
    if model_config.supports_vision:
        builtin_tools.append(view_image_tool)

    # 5. MCP 工具（带缓存和 mtime 失效）
    if include_mcp:
        extensions_config = ExtensionsConfig.from_file()  # 始终从磁盘读取
        if extensions_config.get_enabled_mcp_servers():
            mcp_tools = get_cached_mcp_tools()

            # tool_search 模式：注册延迟工具并添加 tool_search
            if config.tool_search.enabled:
                registry = DeferredToolRegistry()
                for t in mcp_tools:
                    registry.register(t)
                set_deferred_registry(registry)
                builtin_tools.append(tool_search_tool)

    return loaded_tools + builtin_tools + mcp_tools
```

#### 5.2 内置工具

**present_files**: 将输出文件展示给用户
```python
@tool
def present_files(file_paths: list[str]) -> str:
    """标记文件为最终产出，使其对用户可见

    只接受 /mnt/user-data/outputs 下的路径
    """
```

**ask_clarification**: 请求用户澄清
```python
@tool
def ask_clarification(
    question: str,
    clarification_type: str,
    context: str | None = None,
    options: list[str] | None = None
) -> str:
    """请求澄清，触发 ClarificationMiddleware 中断执行"""
```

**view_image**: 读取图片为 base64
```python
@tool
def view_image(image_path: str) -> str:
    """读取图片文件并返回 base64 编码

    仅当模型支持视觉时才添加到工具集
    """
```

---

### 6. MCP 集成 (`mcp/`)

#### 6.1 工具加载 (`tools.py`)

**异步工具同步包装器**：
```python
def _make_sync_tool_wrapper(coro, tool_name):
    """为异步工具创建同步包装器"""
    def sync_wrapper(*args, **kwargs):
        try:
            loop = asyncio.get_running_loop()
        except RuntimeError:
            loop = None

        if loop is not None and loop.is_running():
            # 使用全局线程池避免嵌套事件循环问题
            future = _SYNC_TOOL_EXECUTOR.submit(asyncio.run, coro(*args, **kwargs))
            return future.result()
        else:
            return asyncio.run(coro(*args, **kwargs))
    return sync_wrapper

async def get_mcp_tools():
    """加载所有启用的 MCP 服务器的工具"""
    extensions_config = ExtensionsConfig.from_file()  # 始终从磁盘读取
    servers_config = build_servers_config(extensions_config)

    # 注入初始 OAuth 头
    initial_oauth_headers = await get_initial_oauth_headers(extensions_config)
    for server_name, auth_header in initial_oauth_headers.items():
        servers_config[server_name]["headers"]["Authorization"] = auth_header

    # 创建 OAuth 拦截器
    tool_interceptors = []
    oauth_interceptor = build_oauth_tool_interceptor(extensions_config)
    if oauth_interceptor:
        tool_interceptors.append(oauth_interceptor)

    # 创建多服务器客户端
    client = MultiServerMCPClient(servers_config, tool_interceptors=tool_interceptors)
    tools = await client.get_tools()

    # 包装异步工具以支持同步调用
    for tool in tools:
        if getattr(tool, "func", None) is None and getattr(tool, "coroutine", None):
            tool.func = _make_sync_tool_wrapper(tool.coroutine, tool.name)

    return tools
```

#### 6.2 工具缓存 (`cache.py`)

**mtime 失效机制**：
```python
def get_cached_mcp_tools():
    """获取缓存的 MCP 工具，带延迟初始化和自动失效"""
    global _mcp_tools_cache, _cache_initialized, _config_mtime

    # 检查配置文件是否修改
    if _is_cache_stale():
        reset_mcp_tools_cache()

    # 延迟初始化
    if not _cache_initialized:
        try:
            loop = asyncio.get_event_loop()
            if loop.is_running():
                # 在线程中运行以避免嵌套循环
                with ThreadPoolExecutor() as executor:
                    future = executor.submit(asyncio.run, initialize_mcp_tools())
                    future.result()
            else:
                loop.run_until_complete(initialize_mcp_tools())
        except RuntimeError:
            asyncio.run(initialize_mcp_tools())

    return _mcp_tools_cache or []
```

**缓存失效检测**：
```python
def _is_cache_stale():
    current_mtime = _get_config_mtime()  # extensions_config.json 的 mtime
    if _config_mtime is None or current_mtime is None:
        return False
    return current_mtime > _config_mtime
```

---

### 7. 技能系统 (`skills/`)

#### 7.1 技能加载器 (`loader.py`)

**扫描和加载**：
```python
def load_skills(skills_path=None, use_config=True, enabled_only=False):
    """扫描 skills/ 目录，解析 SKILL.md 文件"""
    skills = []

    # 扫描 public 和 custom 目录
    for category in ["public", "custom"]:
        category_path = skills_path / category
        for root, dirs, files in os.walk(category_path, followlinks=True):
            if "SKILL.md" not in files:
                continue

            skill_file = Path(root) / "SKILL.md"
            relative_path = skill_file.parent.relative_to(category_path)

            skill = parse_skill_file(skill_file, category=category, relative_path=relative_path)
            if skill:
                skills.append(skill)

    # 从 extensions_config.json 更新启用状态
    extensions_config = ExtensionsConfig.from_file()  # 始终从磁盘读取
    for skill in skills:
        skill.enabled = extensions_config.is_skill_enabled(skill.name, skill.category)

    # 按名称排序
    skills.sort(key=lambda s: s.name)
    return skills
```

**技能目录结构**：
```
skills/
├── public/
│   ├── research/SKILL.md
│   ├── report-generation/SKILL.md
│   └── ...
└── custom/
    └── your-skill/SKILL.md
```

#### 7.2 技能解析器 (`parser.py`)

**SKILL.md 格式**：
```markdown
---
name: skill-name
description: Skill description
license: MIT
allowed-tools: ["web_search", "web_fetch"]
---

# Skill Name

Skill description and instructions...

## Resources

- See [resource.md](./resource.md) for more details
```

---

### 8. 子代理系统 (`subagents/`)

#### 8.1 执行器 (`executor.py`)

**双线程池架构**：
```python
# 全局后台任务存储
_background_tasks: dict[str, SubagentResult] = {}

# 调度线程池（3 workers）
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")

# 执行线程池（3 workers）- 支持超时
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")
```

**后台执行流程**：
```python
def execute_async(self, task: str, task_id: str | None = None) -> str:
    """启动后台任务执行"""
    # 1. 创建初始结果（PENDING 状态）
    result = SubagentResult(task_id=task_id, trace_id=self.trace_id, status=SubagentStatus.PENDING)
    _background_tasks[task_id] = result

    # 2. 提交到调度线程池
    def run_task():
        # 更新为 RUNNING
        _background_tasks[task_id].status = SubagentStatus.RUNNING

        # 提交到执行线程池（带超时）
        execution_future = _execution_pool.submit(self.execute, task, result_holder)
        try:
            exec_result = execution_future.result(timeout=self.config.timeout_seconds)
            _background_tasks[task_id] = exec_result
        except FuturesTimeoutError:
            _background_tasks[task_id].status = SubagentStatus.TIMED_OUT

    _scheduler_pool.submit(run_task)
    return task_id
```

**异步执行**：
```python
async def _aexecute(self, task: str, result_holder: SubagentResult | None = None):
    """异步执行任务"""
    agent = self._create_agent()
    state = self._build_initial_state(task)

    # 使用 stream 获取实时更新
    final_state = None
    async for chunk in agent.astream(state, config=run_config, stream_mode="values"):
        final_state = chunk

        # 提取 AI 消息
        messages = chunk.get("messages", [])
        if messages and isinstance(messages[-1], AIMessage):
            message_dict = messages[-1].model_dump()
            if not is_duplicate(message_dict, result.ai_messages):
                result.ai_messages.append(message_dict)

    # 提取最终结果
    last_ai_message = find_last_ai_message(final_state["messages"])
    result.result = extract_text(last_ai_message.content)
    result.status = SubagentStatus.COMPLETED
    return result
```

#### 8.2 内置子代理

**general-purpose**: 通用子代理
- 工具：所有工具除了 `task`
- 用途：任何非平凡任务（研究、代码探索、文件操作、分析）

**bash**: 命令执行专家
- 工具：bash 专用
- 用途：Git、构建、测试、部署操作

---

### 9. 记忆系统 (`agents/memory/`)

#### 9.1 记忆更新器 (`updater.py`)

**数据结构**：
```python
{
    "version": "1.0",
    "lastUpdated": "2026-03-26T00:00:00Z",
    "user": {
        "workContext": {"summary": "...", "updatedAt": "..."},
        "personalContext": {"summary": "...", "updatedAt": "..."},
        "topOfMind": {"summary": "...", "updatedAt": "..."}
    },
    "history": {
        "recentMonths": {"summary": "...", "updatedAt": "..."},
        "earlierContext": {"summary": "...", "updatedAt": "..."},
        "longTermBackground": {"summary": "...", "updatedAt": "..."}
    },
    "facts": [
        {
            "id": "fact_abc123",
            "content": "User prefers Python for data analysis",
            "category": "preference",
            "confidence": 0.9,
            "createdAt": "2026-03-26T00:00:00Z",
            "source": "thread-123"
        }
    ]
}
```

**更新流程**：
```python
def update_memory(self, messages, thread_id=None, agent_name=None):
    # 1. 获取当前记忆
    current_memory = get_memory_data(agent_name)

    # 2. 格式化对话
    conversation_text = format_conversation_for_update(messages)

    # 3. 构建提示词
    prompt = MEMORY_UPDATE_PROMPT.format(
        current_memory=json.dumps(current_memory),
        conversation=conversation_text
    )

    # 4. 调用 LLM
    response = model.invoke(prompt)
    update_data = json.loads(_extract_text(response.content))

    # 5. 应用更新
    updated_memory = self._apply_updates(current_memory, update_data, thread_id)

    # 6. 去除文件上传相关内容
    updated_memory = _strip_upload_mentions_from_memory(updated_memory)

    # 7. 原子写入
    return _save_memory_to_file(updated_memory, agent_name)
```

**事实去重**：
```python
def _apply_updates(self, current_memory, update_data, thread_id):
    # 添加新事实前检查重复
    existing_fact_keys = {
        _fact_content_key(fact.get("content"))
        for fact in current_memory.get("facts", [])
    }

    for fact in new_facts:
        normalized_content = fact.get("content", "").strip()
        fact_key = _fact_content_key(normalized_content)

        # 跳过重复事实
        if fact_key in existing_fact_keys:
            continue

        current_memory["facts"].append(fact_entry)
        existing_fact_keys.add(fact_key)
```

**文件上传内容过滤**：
```python
_UPLOAD_SENTENCE_RE = re.compile(
    r"[^.!?]*\b(?:"
    r"upload(?:ed|ing)?(?:\s+\w+){0,3}\s+(?:file|files?|document|documents?|attachment|attachments?)"
    r"|file\s+upload"
    r"|/mnt/user-data/uploads/"
    r"|<uploaded_files>"
    r")[^.!?]*[.!?]?\s*",
    re.IGNORECASE
)

def _strip_upload_mentions_from_memory(memory_data):
    """从所有记忆摘要和事实中删除文件上传相关句子"""
    # 清理 user/history 部分的摘要
    for section in ("user", "history"):
        for val in section_data.values():
            if isinstance(val, dict) and "summary" in val:
                val["summary"] = _UPLOAD_SENTENCE_RE.sub("", val["summary"]).strip()

    # 删除描述上传事件的事实
    memory_data["facts"] = [f for f in facts if not _UPLOAD_SENTENCE_RE.search(f.get("content", ""))]
```

**mtime 缓存**：
```python
def get_memory_data(agent_name=None):
    """获取记忆数据（带 mtime 检查的缓存）"""
    file_path = _get_memory_file_path(agent_name)
    current_mtime = file_path.stat().st_mtime if file_path.exists() else None

    cached = _memory_cache.get(agent_name)
    if cached is None or cached[1] != current_mtime:
        memory_data = _load_memory_from_file(agent_name)
        _memory_cache[agent_name] = (memory_data, current_mtime)
        return memory_data

    return cached[0]
```

---

### 10. 反射系统 (`reflection/`)

#### 10.1 动态模块加载 (`resolvers.py`)

**resolve_variable**: 解析变量路径
```python
def resolve_variable(variable_path: str, expected_type: type[T] | None = None) -> T:
    """
    Args:
        variable_path: "module.path:variable_name"
        expected_type: 可选类型验证

    Returns:
        解析后的变量

    Raises:
        ImportError: 模块路径无效或属性不存在
        ValueError: 类型验证失败
    """
    module_path, variable_name = variable_path.rsplit(":", 1)
    module = import_module(module_path)
    variable = getattr(module, variable_name)

    if expected_type is not None and not isinstance(variable, expected_type):
        raise ValueError(f"{variable_path} is not an instance of {expected_type}")

    return variable
```

**resolve_class**: 解析类路径
```python
def resolve_class(class_path: str, base_class: type[T] | None = None) -> type[T]:
    """
    Args:
        class_path: "langchain_openai:ChatOpenAI"
        base_class: 可选基类验证

    Returns:
        解析后的类

    Raises:
        ValueError: 不是类或不继承自 base_class
    """
    model_class = resolve_variable(class_path, expected_type=type)

    if not isinstance(model_class, type):
        raise ValueError(f"{class_path} is not a valid class")

    if base_class is not None and not issubclass(model_class, base_class):
        raise ValueError(f"{class_path} is not a subclass of {base_class}")

    return model_class
```

**依赖提示**：
```python
MODULE_TO_PACKAGE_HINTS = {
    "langchain_google_genai": "langchain-google-genai",
    "langchain_anthropic": "langchain-anthropic",
    "langchain_openai": "langchain-openai",
    ...
}

def _build_missing_dependency_hint(module_path, err):
    """构建可操作的依赖缺失提示"""
    package_name = MODULE_TO_PACKAGE_HINTS.get(module_root, missing_module)
    return f"Missing dependency '{missing_module}'. Install it with `uv add {package_name}`"
```

---

### 11. Guardrails 系统 (`guardrails/`)

#### 11.1 Provider 协议 (`provider.py`)

**GuardrailProvider 协议**：
```python
@runtime_checkable
class GuardrailProvider(Protocol):
    """工具调用授权的契约

    任何具有这些方法的类都可以使用 - 无需基类
    """
    name: str

    def evaluate(self, request: GuardrailRequest) -> GuardrailDecision:
        """评估工具调用是否应继续"""
        ...

    async def aevaluate(self, request: GuardrailRequest) -> GuardrailDecision:
        """异步变体"""
        ...
```

**数据结构**：
```python
@dataclass
class GuardrailRequest:
    tool_name: str
    tool_input: dict[str, Any]
    agent_id: str | None = None
    thread_id: str | None = None
    is_subagent: bool = False
    timestamp: str = ""

@dataclass
class GuardrailReason:
    code: str
    message: str = ""

@dataclass
class GuardrailDecision:
    allow: bool
    reasons: list[GuardrailReason] = field(default_factory=list)
    policy_id: str | None = None
    metadata: dict[str, Any] = field(default_factory=dict)
```

#### 11.2 中间件集成 (`middleware.py`)

**GuardrailMiddleware**：
- 在工具调用前拦截
- 调用 Provider 的 `evaluate()` 或 `aevaluate()`
- 如果 `allow=False`，返回错误 ToolMessage

**配置**：
```yaml
guardrails:
  enabled: true
  fail_closed: false  # 拒绝时是否阻止所有调用
  passport: {...}    # 额外的上下文信息
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      allowlist: ["bash", "read_file", "write_file"]
```

---

### 12. 中间件系统 (`agents/middlewares/`)

#### 12.1 中间件链顺序

**严格执行顺序**（依赖关系）：
```python
1. ThreadDataMiddleware        - 初始化线程目录（必须最先）
2. UploadsMiddleware          - 跟踪上传文件（依赖 thread_id）
3. SandboxMiddleware          - 获取沙箱环境
4. DanglingToolCallMiddleware - 修复中断的工具调用
5. GuardrailMiddleware         - 工具调用授权（可选）
6. SummarizationMiddleware     - 上下文压缩（可选）
7. TodoListMiddleware          - 任务跟踪（可选）
8. TitleMiddleware             - 生成标题
9. MemoryMiddleware            - 队列化记忆更新
10. ViewImageMiddleware        - 注入 base64 图片（仅视觉模型）
11. SubagentLimitMiddleware    - 限制并发子代理（可选）
12. LoopDetectionMiddleware    - 检测循环
13. ClarificationMiddleware    - 拦截澄清请求（必须最后）
```

#### 12.2 关键中间件实现

**ThreadDataMiddleware**:
```python
def before_agent(self, state, runtime):
    thread_id = runtime.context.get("thread_id")
    if thread_id:
        # 创建线程目录
        workspace = get_paths().sandbox_workspace_dir(thread_id)
        workspace.mkdir(parents=True, exist_ok=True)

        return {
            "thread_data": {
                "workspace_path": str(workspace),
                "uploads_path": str(get_paths().sandbox_uploads_dir(thread_id)),
                "outputs_path": str(get_paths().sandbox_outputs_dir(thread_id))
            }
        }
```

**DanglingToolCallMiddleware**:
```python
def after_model(self, state, runtime):
    """修复没有响应的工具调用"""
    messages = state.get("messages", [])

    for msg in messages:
        if isinstance(msg, AIMessage) and msg.tool_calls:
            for tool_call in msg.tool_calls:
                # 检查是否有对应的 ToolMessage
                if not find_tool_response(tool_call["id"], messages):
                    # 插入占位符 ToolMessage
                    placeholder = ToolMessage(
                        content="Tool call was interrupted",
                        tool_call_id=tool_call["id"],
                        name=tool_call["name"],
                        status="error"
                    )
                    new_messages.append(placeholder)

    return {"messages": new_messages}
```

**SubagentLimitMiddleware**:
```python
def after_model(self, state, runtime):
    """截断多余的 task 工具调用"""
    messages = state.get("messages", [])
    limit = self.max_concurrent

    for msg in messages:
        if isinstance(msg, AIMessage) and msg.tool_calls:
            task_calls = [tc for tc in msg.tool_calls if tc["name"] == "task"]
            other_calls = [tc for tc in msg.tool_calls if tc["name"] != "task"]

            if len(task_calls) > limit:
                logger.warning(f"Truncating {len(task_calls)} task calls to limit {limit}")
                msg.tool_calls = other_calls + task_calls[:limit]
```

**ClarificationMiddleware**:
```python
def after_model(self, state, runtime):
    """拦截 ask_clarification 工具调用并中断执行"""
    messages = state.get("messages", [])

    for msg in messages:
        if isinstance(msg, AIMessage) and msg.tool_calls:
            for tool_call in msg.tool_calls:
                if tool_call["name"] == "ask_clarification":
                    # 中断执行
                    return Command(goto=END)

    return None
```

---

### 13. 文件上传系统 (`uploads/`)

#### 13.1 上传管理器 (`manager.py`)

**路径安全验证**：
```python
def validate_thread_id(thread_id: str):
    """拒绝包含不安全字符的 thread_id"""
    if not thread_id or not _SAFE_THREAD_ID.match(thread_id):
        raise ValueError(f"Invalid thread_id: {thread_id!r}")

def normalize_filename(filename: str) -> str:
    """清理文件名"""
    if not filename:
        raise ValueError("Filename is empty")

    safe = Path(filename).name  # 提取基名
    if not safe or safe in {".", ".."}:
        raise ValueError(f"Filename is unsafe: {filename!r}")

    if "\\" in safe:
        raise ValueError(f"Filename contains backslash: {filename!r}")

    if len(safe.encode("utf-8")) > 255:
        raise ValueError(f"Filename too long: {len(safe)} chars")

    return safe
```

**路径遍历防护**：
```python
def validate_path_traversal(path: Path, base: Path):
    """验证路径在 base 目录内"""
    try:
        path.resolve().relative_to(base.resolve())
    except ValueError:
        raise PathTraversalError(f"Path {path} escapes base directory {base}")
```

**唯一文件名生成**：
```python
def claim_unique_filename(name: str, seen: set[str]) -> str:
    """生成唯一文件名（冲突时添加 _N 后缀）"""
    if name not in seen:
        seen.add(name)
        return name

    stem, suffix = Path(name).stem, Path(name).suffix
    counter = 1
    candidate = f"{stem}_{counter}{suffix}"
    while candidate in seen:
        counter += 1
        candidate = f"{stem}_{counter}{suffix}"

    seen.add(candidate)
    return candidate
```

---

### 14. 嵌入式客户端 (`client.py`)

#### 14.1 DeerFlowClient

**核心功能**：
```python
class DeerFlowClient:
    def __init__(self, config_path=None, checkpointer=None, *, model_name=None, ...):
        """初始化客户端（延迟 Agent 创建）"""
        self._checkpointer = checkpointer
        self._agent = None  # 延迟创建
        self._agent_config_key = None

    def reset_agent(self):
        """强制重新创建 Agent（用于配置变更后）"""
        self._agent = None
        self._agent_config_key = None

    def chat(self, message: str, thread_id: str | None = None) -> str:
        """同步聊天（返回最终文本）"""

    def stream(self, message: str, thread_id: str | None = None):
        """流式聊天（返回 StreamEvent）"""
        for event in self._agent.stream(...):
            yield StreamEvent(type=event.event, data=event.data)
```

**Gateway 等效方法**：
```python
# Models
def list_models(self) -> dict:  # {"models": [...]}
def get_model(self, name: str) -> dict:

# MCP
def get_mcp_config(self) -> dict:  # {"mcp_servers": {...}}
def update_mcp_config(self, servers: dict) -> dict:

# Skills
def list_skills(self) -> dict:  # {"skills": [...]}
def get_skill(self, name: str) -> dict:
def update_skill(self, name: str, enabled: bool) -> dict:
def install_skill(self, path: str) -> dict:

# Memory
def get_memory(self) -> dict:
def reload_memory(self) -> dict:

# Uploads
def upload_files(self, thread_id: str, files: list[Path]) -> dict:
def list_uploads(self, thread_id: str) -> dict:
def delete_upload(self, thread_id: str, filename: str) -> dict:

# Artifacts
def get_artifact(self, thread_id: str, path: str) -> tuple[bytes, str]:
```

**配置缓存失效**：
```python
def _ensure_agent(self, config: RunnableConfig):
    """当配置相关参数变化时重新创建 Agent"""
    config_key = (
        self._agent_name,
        self._model_name,
        self._thinking_enabled,
        self._subagent_enabled,
        self._plan_mode,
        get_app_config(),  # AppConfig 的 mtime 已被检查
    )

    if self._agent is None or self._agent_config_key != config_key:
        self._agent = create_agent(...)
        self._agent_config_key = config_key

    return self._agent
```

---

## 架构设计模式

### 1. 提供者模式
- **SandboxProvider**: 沙箱环境获取/释放
- **GuardrailProvider**: 工具调用授权
- **好处**: 可插拔、可测试、支持多种实现

### 2. 中间件模式
- **AgentMiddleware**: 在 Agent 执行前后插入逻辑
- **好处**: 关注点分离、可组合、可配置

### 3. 工厂模式
- **create_chat_model()**: 动态创建模型实例
- **make_lead_agent()**: 创建 Lead Agent
- **get_sandbox_provider()**: 获取沙箱提供者单例
- **好处**: 封装创建逻辑、支持配置化

### 4. 缓存失效模式
- **mtime 检测**: 监控文件修改时间
- **自动重新加载**: 文件变更时自动刷新缓存
- **应用**: AppConfig、ExtensionsConfig、Memory、MCP Tools

### 5. 反射模式
- **resolve_variable()**: 动态导入变量
- **resolve_class()**: 动态导入类
- **好处**: 配置化、无需硬编码依赖

### 6. 协议模式
- **GuardrailProvider**: 结构化类型（Protocol）
- **好处**: 无继承、鸭子类型、类型安全

---

## 关键流程

### Agent 执行流程
```
1. make_lead_agent(config) 创建 Agent
   ├─ 解析模型名称
   ├─ 加载模型配置
   ├─ 构建中间件链
   ├─ 获取可用工具
   ├─ 生成系统提示词
   └─ create_agent(model, tools, middleware, prompt, state_schema)

2. Agent.invoke(state, config) 执行
   ├─ before_agent (中间件链)
   │  ├─ ThreadDataMiddleware: 创建线程目录
   │  ├─ UploadsMiddleware: 跟踪上传文件
   │  ├─ SandboxMiddleware: 获取沙箱
   │  └─ ...
   ├─ model.invoke(messages)
   │  ├─ 系统提示词注入
   │  ├─ 记忆注入
   │  ├─ 技能列表注入
   │  └─ 模型调用
   ├─ after_model (中间件链)
   │  ├─ SubagentLimitMiddleware: 截断多余 task 调用
   │  ├─ ViewImageMiddleware: 注入 base64 图片
   │  ├─ ClarificationMiddleware: 拦截澄清请求
   │  └─ ...
   ├─ tool_node.execute(tool_calls)
   │  ├─ GuardrailMiddleware: 工具调用授权
   │  ├─ ToolErrorHandlingMiddleware: 异常处理
   │  └─ 工具执行
   ├─ after_tool (中间件链)
   └─ after_agent (中间件链)
      ├─ MemoryMiddleware: 队列化记忆更新
      └─ SandboxMiddleware: 释放沙箱
```

### 子代理执行流程
```
1. task() 工具调用
   ├─ SubagentExecutor.execute_async(task, task_id)
   │  ├─ 创建 SubagentResult (PENDING)
   │  ├─ 提交到 _scheduler_pool
   │  └─ 返回 task_id

2. 后台执行
   ├─ _scheduler_pool 执行 run_task()
   │  ├─ 更新状态为 RUNNING
   │  ├─ 提交到 _execution_pool
   │  └─ 启动超时计时器
   ├─ _execution_pool 执行 execute(task, result_holder)
   │  ├─ asyncio.run(_aexecute(task, result_holder))
   │  │  ├─ 创建子代理 Agent
   │  │  ├─ agent.astream(state, config)
   │  │  │  ├─ 收集 AI 消息（实时）
   │  │  │  └─ 提取最终结果
   │  │  └─ 更新 result_holder
   │  └─ 返回 SubagentResult
   └─ 更新 _background_tasks[task_id]

3. task_tool 轮询
   ├─ get_background_task_result(task_id)
   ├─ 检查 status
   ├─ COMPLETED/FAILED/TIMED_OUT: 返回结果
   ├─ RUNNING: 继续轮询
   └─ cleanup_background_task(task_id)  # 清理已完成任务
```

### 配置加载流程
```
1. get_app_config() 调用
   ├─ 解析配置文件路径（优先级：参数 > 环境变量 > 当前目录 > 父目录）
   ├─ 检查缓存
   │  ├─ _app_config 是否为 None
   │  ├─ _app_config_path 是否变化
   │  └─ _app_config_mtime 是否变化
   ├─ 任一条件为 True 则重新加载
   │  ├─ 加载 YAML 文件
   │  ├─ 检查配置版本
   │  ├─ 解析环境变量（$开头）
   │  ├─ 加载子配置（memory、subagents、tool_search、guardrails、checkpointer）
   │  ├─ 加载 extensions_config.json
   │  └─ 验证 Pydantic 模型
   └─ 返回缓存的 AppConfig
```

### MCP 工具加载流程
```
1. get_cached_mcp_tools() 调用
   ├─ 检查缓存是否失效（extensions_config.json 的 mtime）
   ├─ 如果失效则重置缓存
   ├─ 如果未初始化则延迟初始化
   │  ├─ initialize_mcp_tools()
   │  │  ├─ ExtensionsConfig.from_file()  # 始终从磁盘读取
   │  │  ├─ build_servers_config()
   │  │  ├─ get_initial_oauth_headers()  # OAuth 令牌
   │  │  ├─ MultiServerMCPClient(servers_config)
   │  │  ├─ client.get_tools()
   │  │  ├─ 包装异步工具为同步
   │  │  └─ 缓存工具和 mtime
   │  └─ 处理事件循环问题（线程池）
   └─ 返回缓存的工具列表
```

---

## 扩展点

### 1. 自定义沙箱提供者
```python
from deerflow.sandbox import Sandbox, SandboxProvider

class MySandbox(Sandbox):
    def execute_command(self, command: str) -> str:
        # 自定义执行逻辑
        pass
    # 实现其他抽象方法...

class MySandboxProvider(SandboxProvider):
    def acquire(self, thread_id: str | None = None) -> str:
        # 获取/创建沙箱
        return sandbox_id

    def get(self, sandbox_id: str) -> Sandbox | None:
        # 返回沙箱实例
        pass

    def release(self, sandbox_id: str) -> None:
        # 释放沙箱
        pass
```

**配置**：
```yaml
sandbox:
  use: my_package.my_module:MySandboxProvider
```

### 2. 自定义 Guardrail 提供者
```python
from deerflow.guardrails import GuardrailProvider, GuardrailRequest, GuardrailDecision

class MyGuardrailProvider:
    name: str = "my-guardrails"

    def evaluate(self, request: GuardrailRequest) -> GuardrailDecision:
        # 自定义授权逻辑
        if request.tool_name in ["dangerous_tool"]:
            return GuardrailDecision(
                allow=False,
                reasons=[GuardrailReason(code="FORBIDDEN", message="Tool not allowed")]
            )
        return GuardrailDecision(allow=True)

    async def aevaluate(self, request: GuardrailRequest) -> GuardrailDecision:
        return self.evaluate(request)
```

**配置**：
```yaml
guardrails:
  enabled: true
  provider:
    use: my_package.my_module:MyGuardrailProvider
```

### 3. 自定义工具
```python
from langchain.tools import tool

@tool
def my_custom_tool(param: str) -> str:
    """工具描述"""
    # 工具逻辑
    return result
```

**配置**：
```yaml
tools:
  - name: my_tool
    use: my_package.my_module:my_custom_tool
    group: my_group
```

### 4. 自定义模型
```python
from langchain.chat_models import BaseChatModel

class MyCustomModel(BaseChatModel):
    # 实现抽象方法
    pass
```

**配置**：
```yaml
models:
  - name: my_model
    display_name: My Custom Model
    use: my_package.my_module:MyCustomModel
    # 模型特定参数
    api_key: $MY_API_KEY
    supports_thinking: false
    supports_vision: false
```

---

## 最佳实践

### 1. 配置管理
- 将 `config.yaml` 放在项目根目录
- 使用环境变量存储敏感信息（`$OPENAI_API_KEY`）
- 运行 `make config-upgrade` 保持配置最新

### 2. 中间件开发
- 遵循中间件顺序依赖
- 实现完整的 `before_agent`、`after_model`、`after_agent`、`wrap_tool_call`
- 使用 `AgentMiddleware` 基类

### 3. 工具开发
- 使用 `@tool` 装饰器
- 提供清晰的工具描述
- 处理异常并返回有意义的错误消息
- 使用虚拟路径（`/mnt/user-data/...`）

### 4. 技能开发
- 在 `skills/custom/` 下创建技能目录
- 编写 `SKILL.md`（包含 YAML frontmatter）
- 引用外部资源（使用相对路径）
- 测试技能加载和执行

### 5. 记忆管理
- 定期检查 `memory.json` 文件
- 使用 `reload_memory_data()` 强制刷新缓存
- 注意事实去重和上传内容过滤

### 6. 测试
- 使用 `set_sandbox_provider()` 注入 Mock 提供者
- 使用 `set_app_config()` 注入测试配置
- 使用 `reset_agent()` 强制重新创建 Agent

---

## 性能优化

### 1. 配置缓存
- `get_app_config()` 使用 mtime 自动失效
- 避免频繁的文件 I/O

### 2. MCP 工具缓存
- `get_cached_mcp_tools()` 延迟初始化
- mtime 失效检测
- 异步工具同步包装（避免嵌套事件循环）

### 3. 记忆缓存
- `get_memory_data()` 使用 mtime 自动失效
- 避免重复加载 JSON 文件

### 4. 沙箱延迟初始化
- `SandboxMiddleware(lazy_init=True)` 延迟获取沙箱
- 首次工具调用时才创建沙箱

### 5. 子代理线程池
- 双线程池架构（调度 + 执行）
- 限制并发数（默认 3）
- 超时保护（默认 15 分钟）

---

## 故障排查

### 1. 配置未生效
- 检查 `config.yaml` 位置（应在项目根目录）
- 检查环境变量是否正确设置
- 运行 `make config-upgrade` 更新配置版本

### 2. MCP 工具未加载
- 检查 `extensions_config.json` 是否正确
- 检查 MCP 服务器是否启用
- 查看日志中的 MCP 初始化错误

### 3. 记忆未更新
- 检查 `memory.enabled` 和 `memory.injection_enabled`
- 检查记忆文件路径
- 使用 `reload_memory_data()` 强制刷新

### 4. 子代理超时
- 检查 `subagents.timeout_seconds` 配置
- 查看子代理日志
- 考虑减少子代理任务复杂度

### 5. 沙箱问题
- 检查沙箱模式（local/Docker/provisioner）
- 检查沙箱镜像是否已拉取
- 查看沙箱日志

---

## 总结

DeerFlow Harness 是一个精心设计的 AI Agent 框架，具有以下特点：

1. **模块化**: 清晰的模块边界，易于扩展和维护
2. **可配置**: 强大的配置系统，支持运行时更新
3. **可扩展**: 提供者模式、中间件模式、协议模式
4. **生产就绪**: 缓存、错误处理、超时保护、资源管理
5. **类型安全**: Pydantic 模型、TypeScript 类型提示
6. **文档完善**: 详细的文档和示例代码

通过理解这些模块的实现逻辑，开发者可以更好地定制和扩展 DeerFlow 系统。
