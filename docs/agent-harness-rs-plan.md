# agent-harness.rs 实现计划

## 项目概述

**目标**: 用 Rust 重新实现 DeerFlow Harness 的核心功能，提供高性能、类型安全的 AI Agent 框架。

**参考项目**: `deerflow-harness` (Python)

**设计原则**:
- 类型安全 - 利用 Rust 类型系统防止运行时错误
- 高性能 - 零成本抽象、高效并发
- 可扩展性 - Trait-based 架构，易于自定义
- 互操作性 - 提供 Python FFI 绑定，兼容现有生态

---

## 项目结构

```
agent-harness-rs/
├── Cargo.toml                    # 项目配置
├── README.md                     # 项目文档
├── LICENSE                       # 许可证
├── examples/                     # 示例代码
│   ├── simple_agent/            # 简单 Agent 示例
│   ├── custom_middleware/       # 自定义中间件示例
│   └── with_sandbox/            # 沙箱集成示例
├── crates/                       # 子 crate（可选）
│   ├── agent-harness-macros/    # 过程宏（derive 代码生成）
│   └── agent-harness-python/    # Python 绑定 (PyO3)
└── src/
    ├── lib.rs                   # 库入口
    ├── prelude.rs               # 常用导入
    │
    ├── config/                  # 配置系统
    │   ├── mod.rs
    │   ├── app.rs              # AppConfig
    │   ├── model.rs            # ModelConfig
    │   ├── agents.rs           # AgentConfig
    │   ├── sandbox.rs          # SandboxConfig
    │   ├── memory.rs           # MemoryConfig
    │   ├── guardrails.rs       # GuardrailsConfig
    │   └── loader.rs           # 配置加载器
    │
    ├── agents/                  # Agent 系统
    │   ├── mod.rs
    │   ├── state.rs            # AgentState, ThreadState
    │   ├── builder.rs          # AgentBuilder
    │   ├── executor.rs         # Agent 执行器
    │   ├── lead_agent.rs       # Lead Agent 实现
    │   └── middlewares/        # 中间件系统
    │       ├── mod.rs
    │       ├── trait.rs        # Middleware trait
    │       ├── chain.rs        # MiddlewareChain
    │       ├── manager.rs      # 中间件管理器
    │       └── implementations/ # 各中间件实现
    │
    ├── models/                  # 模型系统
    │   ├── mod.rs
    │   ├── factory.rs          # ModelFactory
    │   ├── base.rs             # Model trait
    │   ├── providers/          # 各 LLM 提供商实现
    │   │   ├── mod.rs
    │   │   ├── anthropic.rs    # Claude
    │   │   ├── openai.rs       # OpenAI
    │   │   ├── deepseek.rs     # DeepSeek
    │   │   └── google.rs       # Gemini
    │   └── thinking.rs         # 思维模式支持
    │
    ├── tools/                   # 工具系统
    │   ├── mod.rs
    │   ├── trait.rs            # Tool trait
    │   ├── registry.rs         # 工具注册表
    │   ├── executor.rs         # 工具执行器
    │   ├── builtin/            # 内置工具
    │   │   ├── mod.rs
    │   │   ├── bash.rs         # bash 命令
    │   │   ├── read_file.rs    # 读取文件
    │   │   ├── write_file.rs   # 写入文件
    │   │   ├── ls.rs           # 列出目录
    │   │   ├── str_replace.rs  # 字符串替换
    │   │   ├── view_image.rs   # 查看图片
    │   │   ├── ask_clarification.rs  # 请求澄清
    │   │   └── present_files.rs      # 展示文件
    │   ├── mcp/                # MCP 工具集成
    │   │   ├── mod.rs
    │   │   ├── client.rs       # MCP 客户端
    │   │   ├── adapter.rs      # MCP 工具适配器
    │   │   └── cache.rs        # MCP 工具缓存
    │   └── community/          # 社区工具
    │       ├── mod.rs
    │       ├── tavily.rs       # Tavily 搜索
    │       ├── jina.rs         # Jina Reader
    │       └── firecrawl.rs    # Firecrawl
    │
    ├── sandbox/                 # 沙箱系统
    │   ├── mod.rs
    │   ├── trait.rs            # Sandbox trait
    │   ├── provider.rs         # SandboxProvider trait
    │   ├── local/              # 本地沙箱
    │   │   ├── mod.rs
    │   │   └── provider.rs
    │   ├── docker/             # Docker 沙箱
    │   │   ├── mod.rs
    │   │   └── provider.rs
    │   ├── path_mapper.rs      # 虚拟路径映射
    │   └── tools.rs            # 沙箱工具实现
    │
    ├── subagents/               # 子代理系统
    │   ├── mod.rs
    │   ├── executor.rs         # 子代理执行器
    │   ├── registry.rs         # 子代理注册表
    │   ├── task_tool.rs        # task 工具实现
    │   ├── builtin/            # 内置子代理
    │   │   ├── mod.rs
    │   │   ├── general.rs      # 通用子代理
    │   │   └── bash.rs         # Bash 子代理
    │   └── pool.rs             # 线程池管理
    │
    ├── memory/                  # 记忆系统
    │   ├── mod.rs
    │   ├── data.rs             # 记忆数据结构
    │   ├── updater.rs          # 记忆更新器
    │   ├── queue.rs            # 更新队列（防抖）
    │   ├── injection.rs        # 记忆注入
    │   └── storage.rs          # 记忆存储
    │
    ├── skills/                  # 技能系统
    │   ├── mod.rs
    │   ├── loader.rs           # 技能加载器
    │   ├── parser.rs           # SKILL.md 解析器
    │   ├── registry.rs         # 技能注册表
    │   └── prompt.rs           # 技能提示词生成
    │
    ├── guardrails/              # 防护栏系统
    │   ├── mod.rs
    │   ├── trait.rs            # GuardrailProvider trait
    │   ├── middleware.rs       # 中间件实现
    │   ├── builtin/            # 内置提供者
    │   │   ├── mod.rs
    │   │   └── allowlist.rs    # 白名单提供者
    │   └── request.rs          # GuardrailRequest
    │
    ├── channels/                # IM 通道集成（可选）
    │   ├── mod.rs
    │   ├── base.rs             # Channel trait
    │   ├── slack.rs            # Slack 集成
    │   ├── telegram.rs         # Telegram 集成
    │   └── feishu.rs           # 飞书集成
    │
    ├── runtime/                 # 运行时系统
    │   ├── mod.rs
    │   ├── context.rs          # RuntimeContext
    │   ├── config.rs           # RuntimeConfig
    │   └── metadata.rs         # 元数据
    │
    ├── messages/                # 消息系统
    │   ├── mod.rs
    │   ├── base.rs             # Message trait
    │   ├── human.rs            # HumanMessage
    │   ├── ai.rs               # AIMessage
    │   ├── tool.rs             # ToolMessage
    │   ├── content.rs          # Content 类型（文本/图片/混合）
    │   └── converter.rs        # 消息转换器
    │
    ├── prompts/                 # 提示词系统
    │   ├── mod.rs
    │   ├── template.rs         # 模板引擎
    │   ├── builder.rs          # 提示词构建器
    │   └── lead_agent.rs       # Lead Agent 提示词
    │
    ├── utils/                   # 工具函数
    │   ├── mod.rs
    │   ├── network.rs          # 网络请求
    │   ├── file.rs             # 文件操作
    │   └── json.rs             # JSON 辅助
    │
    └── error/                   # 错误处理
        ├── mod.rs
        ├── result.rs           # Result 类型别名
        └── kinds.rs            # 错误类型定义
```

---

## 核心依赖 (Cargo.toml)

```toml
[package]
name = "agent-harness"
version = "0.1.0"
edition = "2021"
rust-version = "1.75.0"
authors = ["DeerFlow Contributors"]
license = "MIT OR Apache-2.0"
repository = "https://github.com/bytedance/deer-flow"
description = "High-performance AI agent harness framework"

[dependencies]
# 核心依赖
tokio = { version = "1.40", features = ["full"] }
async-trait = "0.1"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"

# 错误处理
thiserror = "1.0"
anyhow = "1.0"

# 配置
config = "0.14"
notify = "6.1"           # 文件监控

# HTTP 客户端
reqwest = { version = "0.12", features = ["json", "stream"] }
http = "1.1"

# 类型系统
strum = { version = "0.26", features = ["derive"] }
strum_macros = "0.26"
derive_more = { version = "1.0", features = ["display", "error", "from"] }

# 日志
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# 时间处理
chrono = "0.4"

# UUID
uuid = { version = "1.10", features = ["v4", "serde"] }

# 正则表达式
regex = "1.11"

# 路径处理
path-clean = "1.0"

# Docker 支持 (可选)
bollard = { version = "0.17", optional = true }

# Kubernetes 支持 (可选)
kube = { version = "0.93", features = ["runtime"], optional = true }

# Python 绑定 (可选)
pyo3 = { version = "0.22", features = ["extension-module"], optional = true }

# 工具
url = "2.5"

# 加密
base64 = "0.22"

[dev-dependencies]
tokio-test = "0.4"
criterion = "0.5"
mockito = "1.5"

[features]
default = []
docker = ["bollard"]
kubernetes = ["kube"]
python = ["pyo3"]
full = ["docker", "kubernetes", "python"]

[[bench]]
name = "middleware"
harness = false
```

---

## 核心 Trait 设计

### 1. Middleware Trait

```rust
use async_trait::async_trait;

/// 中间件钩子点
pub enum HookPoint {
    BeforeAgent,
    AfterAgent,
    BeforeModel,
    AfterModel,
}

/// 中间件上下文
pub struct MiddlewareContext<'a, S> {
    pub state: &'a mut S,
    pub runtime: &'a RuntimeContext,
}

/// 中间件包装器类型
pub type MiddlewareResult = Result<Option<StateUpdate>>;

/// 中间件核心 Trait
#[async_trait]
pub trait Middleware: Send + Sync + 'static {
    /// 中间件名称（用于日志和调试）
    fn name(&self) -> &'static str {
        std::any::type_name::<Self>()
    }

    /// 在 Agent 执行前调用
    async fn before_agent(&self, ctx: MiddlewareContext<'_>) -> MiddlewareResult {
        Ok(None)
    }

    /// 在 Agent 执行后调用
    async fn after_agent(&self, ctx: MiddlewareContext<'_>) -> MiddlewareResult {
        Ok(None)
    }

    /// 在模型调用前调用
    async fn before_model(&self, ctx: MiddlewareContext<'_>) -> MiddlewareResult {
        Ok(None)
    }

    /// 在模型调用后调用
    async fn after_model(&self, ctx: MiddlewareContext<'_>) -> MiddlewareResult {
        Ok(None)
    }

    /// 包装模型调用（可修改请求或响应）
    async fn wrap_model_call(
        &self,
        request: ModelRequest,
        handler: Pin<Box<dyn Future<Output = Result<ModelResponse>> + Send>>,
    ) -> Result<ModelResponse> {
        handler.await
    }

    /// 包装工具调用（可拦截或修改）
    async fn wrap_tool_call(
        &self,
        request: ToolCallRequest,
        handler: Pin<Box<dyn Future<Output = Result<ToolMessage>> + Send>>,
    ) -> Result<ToolMessage> {
        handler.await
    }
}
```

### 2. Model Trait

```rust
/// LLM 模型 Trait
#[async_trait]
pub trait Model: Send + Sync + 'static {
    /// 模型名称
    fn name(&self) -> &str;

    /// 是否支持思维模式
    fn supports_thinking(&self) -> bool;

    /// 是否支持视觉
    fn supports_vision(&self) -> bool;

    /// 同步调用
    async fn invoke(&self, messages: Vec<Message>) -> Result<AIMessage>;

    /// 流式调用
    async fn stream(
        &self,
        messages: Vec<Message>,
    ) -> Result<Pin<Box<dyn Stream<Item = Result<StreamEvent>> + Send>>>;

    /// 获取 token 使用情况
    fn get_usage(&self) -> Option<TokenUsage>;
}
```

### 3. Tool Trait

```rust
/// 工具定义
pub struct ToolDefinition {
    pub name: String,
    pub description: String,
    pub parameters: JsonSchema,
    pub group: Option<String>,
}

/// 工具 Trait
#[async_trait]
pub trait Tool: Send + Sync + 'static {
    /// 工具定义
    fn definition(&self) -> &ToolDefinition;

    /// 执行工具
    async fn execute(&self, args: Value, context: &ToolContext) -> Result<ToolOutput>;

    /// 验证参数
    fn validate_args(&self, args: &Value) -> Result<()> {
        // 默认使用 JSON Schema 验证
        Ok(())
    }
}
```

### 4. Sandbox Trait

```rust
/// 沙箱 Trait
#[async_trait]
pub trait Sandbox: Send + Sync + 'static {
    /// 沙箱 ID
    fn sandbox_id(&self) -> &str;

    /// 执行命令
    async fn execute_command(&self, command: &str) -> Result<CommandOutput>;

    /// 读取文件
    async fn read_file(&self, path: &str) -> Result<String>;

    /// 写入文件
    async fn write_file(&self, path: &str, content: &str) -> Result<()>;

    /// 列出目录
    async fn list_dir(&self, path: &str) -> Result<Vec<DirEntry>>;

    /// 检查文件存在
    async fn file_exists(&self, path: &str) -> Result<bool>;
}

/// 沙箱提供者 Trait
#[async_trait]
pub trait SandboxProvider: Send + Sync + 'static {
    type Sandbox: Sandbox;

    /// 获取或创建沙箱
    async fn acquire(&self, thread_id: Option<&str>) -> Result<String>;

    /// 获取沙箱实例
    async fn get(&self, sandbox_id: &str) -> Option<Self::Sandbox>;

    /// 释放沙箱
    async fn release(&self, sandbox_id: &str) -> Result<()>;

    /// 关闭所有沙箱
    async fn shutdown(&self) -> Result<()>;
}
```

---

## 实现阶段

### Phase 1: 基础框架（2-3周）

**目标**: 建立项目骨架和核心类型系统

**任务**:
- [ ] 初始化项目，配置 Cargo.toml
- [ ] 实现核心错误类型 (`error/`)
- [ ] 实现消息类型系统 (`messages/`)
  - [ ] Message trait
  - [ ] HumanMessage, AIMessage, ToolMessage
  - [ ] Content 类型（文本/图片/混合）
- [ ] 实现 AgentState 和 ThreadState (`agents/state.rs`)
- [ ] 实现基础配置系统 (`config/`)
  - [ ] 配置加载器
  - [ ] YAML 解析
  - [ ] 环境变量替换
- [ ] 实现运行时上下文 (`runtime/`)

**交付物**:
- 可编译的项目骨架
- 基础类型定义
- 配置加载功能

### Phase 2: 模型系统（2-3周）

**目标**: 实现统一的 LLM 模型接口

**任务**:
- [ ] 实现 Model trait (`models/base.rs`)
- [ ] 实现模型工厂 (`models/factory.rs`)
- [ ] 实现 Anthropic 提供商 (`models/providers/anthropic.rs`)
  - [ ] 支持 Claude Opus/Sonnet/Haiku
  - [ ] 思维模式支持
  - [ ] 视觉模式支持
- [ ] 实现 OpenAI 提供商 (`models/providers/openai.rs`)
- [ ] 实现 DeepSeek 提供商 (`models/providers/deepseek.rs`)
- [ ] 实现流式响应处理
- [ ] 实现 Token 使用统计

**交付物**:
- 完整的模型抽象层
- 至少 2 个 LLM 提供商实现
- 流式响应支持

### Phase 3: 工具系统（2-3周）

**目标**: 实现可扩展的工具系统

**任务**:
- [ ] 实现 Tool trait (`tools/trait.rs`)
- [ ] 实现工具注册表 (`tools/registry.rs`)
- [ ] 实现工具执行器 (`tools/executor.rs`)
- [ ] 实现内置工具 (`tools/builtin/`)
  - [ ] bash - 命令执行
  - [ ] read_file - 读取文件
  - [ ] write_file - 写入文件
  - [ ] ls - 列出目录
  - [ ] str_replace - 字符串替换
- [ ] 实现 JSON Schema 参数验证
- [ ] 实现工具错误处理

**交付物**:
- 工具系统框架
- 完整的内置工具集
- 工具注册和发现机制

### Phase 4: 中间件系统（3-4周）

**目标**: 实现中间件链和核心中间件

**任务**:
- [ ] 实现 Middleware trait (`agents/middlewares/trait.rs`)
- [ ] 实现中间件链 (`agents/middlewares/chain.rs`)
- [ ] 实现中间件管理器 (`agents/middlewares/manager.rs`)
- [ ] 实现核心中间件 (`agents/middlewares/implementations/`)
  - [ ] ThreadDataMiddleware - 线程目录创建
  - [ ] SandboxMiddleware - 沙箱生命周期
  - [ ] TitleMiddleware - 标题生成
  - [ ] MemoryMiddleware - 记忆队列
  - [ ] DanglingToolCallMiddleware - 悬挂工具修复
  - [ ] ClarificationMiddleware - 澄清拦截
  - [ ] LoopDetectionMiddleware - 循环检测
  - [ ] ToolErrorHandlingMiddleware - 工具异常处理
- [ ] 实现中间件顺序管理
- [ ] 实现钩子点调用

**交付物**:
- 完整的中间件框架
- 8+ 核心中间件实现
- 中间件链执行引擎

### Phase 5: Agent 执行器（2-3周）

**目标**: 实现 Agent 核心执行逻辑

**任务**:
- [ ] 实现 AgentBuilder (`agents/builder.rs`)
- [ ] 实现 AgentExecutor (`agents/executor.rs`)
- [ ] 实现 Lead Agent (`agents/lead_agent.rs`)
- [ ] 实现提示词模板系统 (`prompts/`)
  - [ ] 模板引擎
  - [ ] Lead Agent 提示词
  - [ ] 动态内容注入
- [ ] 实现工具调用循环
- [ ] 实现状态管理

**交付物**:
- 完整的 Agent 执行器
- Lead Agent 实现
- 提示词系统

### Phase 6: 沙箱系统（2-3周）

**目标**: 实现隔离执行环境

**任务**:
- [ ] 实现 Sandbox trait (`sandbox/trait.rs`)
- [ ] 实现 SandboxProvider trait (`sandbox/provider.rs`)
- [ ] 实现本地沙箱 (`sandbox/local/`)
- [ ] 实现虚拟路径映射 (`sandbox/path_mapper.rs`)
- [ ] 实现沙箱工具 (`sandbox/tools.rs`)
- [ ] 实现 Docker 沙箱（可选）(`sandbox/docker/`)

**交付物**:
- 本地沙箱实现
- 路径映射系统
- Docker 沙箱（可选）

### Phase 7: MCP 集成（2周）

**目标**: 集成 Model Context Protocol

**任务**:
- [ ] 实现 MCP 客户端 (`tools/mcp/client.rs`)
- [ ] 实现 MCP 工具适配器 (`tools/mcp/adapter.rs`)
- [ ] 实现 MCP 工具缓存 (`tools/mcp/cache.rs`)
- [ ] 支持 stdio 传输
- [ ] 支持 SSE 传输
- [ ] 支持 HTTP 传输

**交付物**:
- MCP 客户端
- 工具适配器
- 多传输支持

### Phase 8: 子代理系统（2-3周）

**目标**: 实现任务分解和并行执行

**任务**:
- [ ] 实现子代理执行器 (`subagents/executor.rs`)
- [ ] 实现子代理注册表 (`subagents/registry.rs`)
- [ ] 实现 task 工具 (`subagents/task_tool.rs`)
- [ ] 实现线程池管理 (`subagents/pool.rs`)
- [ ] 实现内置子代理
  - [ ] general-purpose - 通用子代理
  - [ ] bash - 命令子代理
- [ ] 实现并发限制中间件

**交付物**:
- 子代理系统
- 并行执行能力
- 并发限制

### Phase 9: 记忆系统（2周）

**目标**: 实现 LLM 驱动的记忆系统

**任务**:
- [ ] 实现记忆数据结构 (`memory/data.rs`)
- [ ] 实现记忆更新器 (`memory/updater.rs`)
- [ ] 实现更新队列（防抖）(`memory/queue.rs`)
- [ ] 实现记忆注入 (`memory/injection.rs`)
- [ ] 实现记忆存储 (`memory/storage.rs`)

**交付物**:
- 记忆数据模型
- LLM 记忆更新
- 防抖队列

### Phase 10: 技能系统（1-2周）

**目标**: 实现渐进式技能加载

**任务**:
- [ ] 实现技能加载器 (`skills/loader.rs`)
- [ ] 实现 SKILL.md 解析器 (`skills/parser.rs`)
- [ ] 实现技能注册表 (`skills/registry.rs`)
- [ ] 实现技能提示词生成 (`skills/prompt.rs`)

**交付物**:
- 技能加载系统
- Markdown 解析
- 提示词集成

### Phase 11: 防护栏系统（1-2周）

**目标**: 实现工具调用授权

**任务**:
- [ ] 实现 GuardrailProvider trait (`guardrails/trait.rs`)
- [ ] 实现防护栏中间件 (`guardrails/middleware.rs`)
- [ ] 实现白名单提供者 (`guardrails/builtin/allowlist.rs`)
- [ ] 实现防护栏请求类型 (`guardrails/request.rs`)

**交付物**:
- 防护栏框架
- 白名单实现
- 中间件集成

### Phase 12: Python 绑定（可选，3-4周）

**目标**: 提供 Python FFI 接口

**任务**:
- [ ] 设置 PyO3 项目结构
- [ ] 实现核心类型绑定
- [ ] 实现 Agent 执行器绑定
- [ ] 实现配置加载绑定
- [ ] 实现 Python 异常转换
- [ ] 编写 Python 使用示例

**交付物**:
- PyO3 模块
- Python API
- 使用文档

---

## 架构决策记录

### ADR-001: 使用异步 Trait

**决策**: 使用 `async-trait` 宏定义异步 trait

**原因**:
- Rust 原生不支持异步 trait（RFC 3185 尚未稳定）
- `async-trait` 是社区标准方案
- 零成本抽象，性能损失可忽略

**替代方案**:
- 使用手动 Future 类型（代码复杂度高）
- 等待原生异步 trait 稳定（时间不确定）

### ADR-002: 错误处理策略

**决策**: 使用 `thiserror` 定义结构化错误类型

**原因**:
- 提供清晰的错误层次
- 支持错误上下文传递
- 便于 Python 绑定

**示例**:
```rust
#[derive(Debug, thiserror::Error)]
pub enum HarnessError {
    #[error("Model error: {0}")]
    Model(#[from] ModelError),

    #[error("Tool execution failed: {tool_name}")]
    ToolError {
        tool_name: String,
        source: Box<dyn std::error::Error + Send + Sync>,
    },
}
```

### ADR-003: 配置格式

**决策**: 继续使用 YAML 格式

**原因**:
- 与 Python 版本兼容
- 更好的注释支持
- 更易读的层次结构

**替代方案**:
- JSON（无注释）
- TOML（不支持复杂嵌套）
- 自定义 DSL（维护成本高）

### ADR-004: 沙箱实现

**决策**: 优先实现本地沙箱，Docker 沙箱作为可选特性

**原因**:
- 本地沙箱覆盖 80% 开发场景
- Docker 增加部署复杂度
- 通过 feature flag 控制

### ADR-005: 消息传递

**决策**: 使用 `tokio::sync::mpsc` 进行异步消息传递

**原因**:
- 与 tokio 生态集成
- 高性能通道实现
- 支持背压

---

## 性能目标

| 指标 | 目标 | 测量方法 |
|------|------|----------|
| 冷启动时间 | < 100ms | 从创建配置到 Agent 就绪 |
| 首次响应延迟 | < 500ms | 从发送消息到首次流式事件 |
| 中间件开销 | < 10ms | 单个中间件执行时间 |
| 并发子代理 | 50+ | 同时运行的子代理数 |
| 内存占用 | < 50MB | 空闲 Agent 内存 |

---

## 测试策略

### 单元测试
- 每个模块 80%+ 覆盖率
- 使用 `mockito` 模拟 HTTP 请求
- 使用 `tokio-test` 测试异步代码

### 集成测试
- 端到端 Agent 执行测试
- 多模型提供商测试
- 沙箱隔离测试

### 基准测试
- 中间件性能
- 模型调用延迟
- 工具执行吞吐量

### 示例测试
- 确保 examples/ 目录可编译运行
- 作为文档使用

---

## 兼容性目标

### 与 Python 版本兼容

| 功能 | Python | Rust | 优先级 |
|------|--------|------|--------|
| 配置格式 | YAML | YAML | P0 |
| 消息格式 | Pydantic | Rust structs | P0 |
| 中间件接口 | AgentMiddleware | Middleware trait | P0 |
| 工具接口 | StructuredTool | Tool trait | P0 |
| MCP 协议 | langchain-mcp-adapters | 原生实现 | P1 |

### Python FFI 目标

```python
# 目标 Python API
from agent_harness_rs import Agent, AgentConfig

config = AgentConfig.from_yaml("config.yaml")
agent = Agent(config)

async for event in agent.stream("Hello, agent!"):
    print(event)
```

---

## 文档计划

### API 文档
- 所有公开 API 的 rustdoc
- 示例代码注释
- Trait 文档

### 用户指南
- 快速开始
- 配置参考
- 中间件开发
- 工具开发
- 迁移指南（Python → Rust）

### 架构文档
- 设计决策记录
- 模块依赖图
- 性能优化指南

---

## 发布计划

### v0.1.0 - MVP
- Phase 1-5 完成
- 基本 Agent 功能可用
- 单模型提供商（Anthropic）

### v0.2.0 - 生产就绪
- Phase 6-9 完成
- 多模型支持
- 沙箱隔离
- 记忆系统

### v0.3.0 - 生态完整
- Phase 10-11 完成
- 技能系统
- 防护栏系统
- MCP 集成

### v0.4.0 - 性能优化
- 性能调优
- 基准测试
- 生产验证

### v0.5.0 - Python 兼容
- Phase 12 完成
- Python FFI
- 迁移工具
- 兼容性测试

---

## 风险和缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| 异步 trait 不稳定 | 中 | 低 | 使用 async-trait 宏 |
| LLM API 变更 | 高 | 中 | 版本化 API，适配器模式 |
| 性能不达标 | 高 | 中 | 提前基准测试，优化热点 |
| Python 绑定复杂 | 中 | 高 | 分阶段实现，优先核心功能 |
| 生态碎片化 | 低 | 中 | 保持配置兼容，共享文档 |

---

## 下一步行动

1. **确认计划** - 等待审阅和反馈
2. **初始化仓库** - 创建 GitHub 仓库
3. **设置 CI/CD** - GitHub Actions 配置
4. **开始 Phase 1** - 基础框架实现
5. **周会同步** - 每周进度同步会议

---

## 参考资料

- [DeerFlow 文档](../README.md)
- [LangChain Rust](https://github.com/seluion/langchain-rust)
- [PyO3 文档](https://pyo3.rs/)
- [Tokio 教程](https://tokio.rs/tokio/tutorial)
- [Async Rust Book](https://rust-lang.github.io/async-book/)
