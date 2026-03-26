# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Important Development Guidelines

### Documentation Update Policy
**CRITICAL: Always update README.md and CLAUDE.md after every code change**

When making code changes, you MUST update the relevant documentation:
- Update `README.md` for user-facing changes (features, setup, usage instructions)
- Update `CLAUDE.md` for development changes (architecture, commands, workflows, internal systems)
- Keep documentation synchronized with the codebase at all times
- Ensure accuracy and timeliness of all documentation

## Project Overview

DeerFlow is a full-stack "super agent harness" that orchestrates sub-agents, memory, and sandboxes to perform complex tasks. Built on LangGraph and LangChain, it provides an extensible AI agent system with sandboxed execution, persistent memory, and a skills framework.

**Architecture**:
- **LangGraph Server** (port 2024): Agent runtime and workflow execution
- **Gateway API** (port 8001): REST API for models, MCP, skills, memory, artifacts, uploads
- **Frontend** (port 3000): Next.js web interface
- **Nginx** (port 2026): Unified reverse proxy entry point

**Technology Stack**:
- Backend: Python 3.12+, LangGraph, LangChain, FastAPI
- Frontend: Next.js 16, React 19, TypeScript 5.8, Tailwind CSS 4
- Package Managers: `uv` (Python), `pnpm` (Node.js)

## Repository Structure

```
deer-flow/
├── backend/               # Python backend (LangGraph + Gateway API)
│   ├── packages/harness/  # deerflow-harness package (import: deerflow.*)
│   ├── app/               # FastAPI Gateway API (import: app.*)
│   ├── tests/             # Backend test suite
│   └── docs/              # Backend documentation
├── frontend/              # Next.js frontend application
├── skills/                # Agent skills directory
│   ├── public/            # Built-in skills (committed)
│   └── custom/            # Custom skills (gitignored)
├── docker/                # Docker configuration
├── scripts/               # Utility scripts
├── Makefile               # Root commands
├── config.yaml            # Main application configuration (gitignored)
├── config.example.yaml    # Configuration template
└── extensions_config.json # MCP servers and skills (gitignored)
```

**Dependency Rule**: The backend is split into two layers:
- **Harness** (`packages/harness/deerflow/`): Publishable agent framework (import: `deerflow.*`)
- **App** (`app/`): Unpublished application code (import: `app.*`)
- App imports deerflow, but deerflow never imports app (enforced by CI tests)

## Development Commands

### Root Directory Commands

```bash
# Initial setup
make check           # Verify required tools (Node.js 22+, pnpm, uv, nginx)
make config          # Generate local config files (aborts if config exists)
make install         # Install all dependencies (backend + frontend)

# Local development
make dev             # Start all services (LangGraph + Gateway + Frontend + Nginx)
make stop            # Stop all running services
make clean           # Clean up processes and temporary files

# Docker development
make docker-init     # Pull sandbox image
make docker-start   # Start Docker dev services (mode-aware from config.yaml)
make docker-stop    # Stop Docker dev services

# Production Docker
make up              # Build and start production services
make down            # Stop and remove production containers

# Configuration
make config-upgrade  # Merge new fields from config.example.yaml into config.yaml
```

### Backend-Specific Commands (run from `backend/`)

```bash
make install         # Install Python dependencies
make dev             # Run LangGraph server only (port 2024)
make gateway         # Run Gateway API only (port 8001)
make test            # Run all backend tests
make lint            # Lint with ruff
make format          # Format code with ruff
```

### Frontend-Specific Commands (run from `frontend/`)

```bash
pnpm install         # Install dependencies
pnpm dev             # Dev server with Turbopack (http://localhost:3000)
pnpm build           # Production build (requires BETTER_AUTH_SECRET set)
pnpm lint            # ESLint only
pnpm typecheck       # TypeScript type check
```

**Note**: Frontend build requires `BETTER_AUTH_SECRET` for production validation. Set it or use `SKIP_ENV_VALIDATION=1`.

### Test Commands

```bash
# Run all backend tests
cd backend && make test

# Run specific test file
cd backend && PYTHONPATH=. uv run pytest tests/test_feature.py -v

# Run tests with coverage
cd backend && PYTHONPATH=. uv run pytest tests/ --cov=deerflow --cov=app
```

## Configuration

### Main Configuration (`config.yaml`)

Generated from `config.example.yaml`. Place in **project root** (not in backend/).

**Key sections**:
- `models[]` - LLM configurations with provider class paths
- `tools[]` - Tool configurations
- `sandbox.use` - Sandbox provider (local or Docker)
- `skills.path` / `skills.container_path` - Skills directories
- `memory` - Memory system settings
- `subagents.enabled` - Subagent delegation
- `summarization` - Context reduction settings

**Config Versioning**: `config.example.yaml` has a `config_version` field. Run `make config-upgrade` to auto-merge missing fields when the schema changes.

**Config Caching**: `get_app_config()` automatically reloads when the config file's mtime increases - no manual restart needed for most config changes.

### Extensions Configuration (`extensions_config.json`)

MCP servers and skills configuration. Place in **project root**.

**Key sections**:
- `mcpServers` - MCP server configurations (enabled, type, command, args, env, url, oauth)
- `skills` - Skill enabled/disabled states

Both configs can be modified at runtime via Gateway API endpoints.

## Important Architecture Concepts

### Sandbox System

Each thread runs in an isolated environment:
- **Local mode**: Direct execution on host (development)
- **Docker mode**: Containerized execution (production)
- **Provisioner mode**: Kubernetes pods via provisioner service

**Virtual path mappings**:
- `/mnt/user-data/workspace` → `backend/.deer-flow/threads/{thread_id}/user-data/workspace`
- `/mnt/user-data/uploads` → `backend/.deer-flow/threads/{thread_id}/user-data/uploads`
- `/mnt/user-data/outputs` → `backend/.deer-flow/threads/{thread_id}/user-data/outputs`
- `/mnt/skills` → `skills/` (project root)

### Middleware Chain

Middlewares execute in strict order in the LangGraph agent:
1. ThreadDataMiddleware - Initialize per-thread directories
2. UploadsMiddleware - Track and inject uploaded files
3. SandboxMiddleware - Acquire sandbox environment
4. DanglingToolCallMiddleware - Handle interrupted tool calls
5. GuardrailMiddleware - Pre-tool-call authorization (optional)
6. SummarizationMiddleware - Context reduction (optional)
7. TodoListMiddleware - Task tracking for plan mode (optional)
8. TitleMiddleware - Auto-generate thread titles
9. MemoryMiddleware - Queue conversations for memory update
10. ViewImageMiddleware - Inject base64 images for vision models
11. SubagentLimitMiddleware - Enforce concurrent subagent limit
12. ClarificationMiddleware - Handle user clarifications (must be last)

### Skills System

Skills are Markdown files with YAML frontmatter defining capabilities. Located in `skills/public/` (built-in) and `skills/custom/` (user-installed). Loaded progressively - only when needed by the task.

### MCP Integration

MCP servers provide extensible tool integrations. Configured in `extensions_config.json` with support for stdio, SSE, and HTTP transports. OAuth token flows supported for HTTP/SSE servers.

## CI and Testing

- **Backend CI**: `.github/workflows/backend-unit-tests.yml` runs `make lint` and `make test` on PRs
- **Regression tests**: Docker sandbox mode detection, provisioner kubeconfig handling, harness/app boundary enforcement
- **Frontend**: No test framework currently configured

## Platform-Specific Notes

### Windows
- Use Git Bash for `make dev` (auto-detected)
- Docker commands may require WSL2

### Linux
- May need to add user to `docker` group for Docker commands

### macOS
- Supports Apple Container for sandbox images

## Environment Variables

Key environment variables (place in `.env` at project root):
- `OPENAI_API_KEY` - OpenAI API key
- `ANTHROPIC_API_KEY` - Anthropic API key
- `TAVILY_API_KEY` - Tavily search API key
- `GITHUB_TOKEN` - GitHub MCP server
- `BETTER_AUTH_SECRET` - Frontend auth secret (for production build)
- `DEER_FLOW_CONFIG_PATH` - Override default config.yaml location
- `DEER_FLOW_EXTENSIONS_CONFIG_PATH` - Override default extensions_config.json location

## Further Reading

- **Backend details**: See `backend/CLAUDE.md` for backend-specific architecture and API documentation
- **Frontend details**: See `frontend/CLAUDE.md` for frontend-specific architecture and patterns
- **Configuration guide**: See `backend/docs/CONFIGURATION.md` for detailed configuration options
- **Architecture**: See `backend/docs/ARCHITECTURE.md` for system architecture diagrams
- **API reference**: See `backend/docs/API.md` for Gateway API endpoints
- **Contributing**: See `CONTRIBUTING.md` for development workflow
