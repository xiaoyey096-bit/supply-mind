# SupplyMind 架构

## 仓库职责

| 目录 | 职责 | 禁止包含 |
| --- | --- | --- |
| `backend/` | FastAPI API、领域服务、Celery 任务、数据库迁移、后端测试 | 前端静态资源或生产密钥 |
| `frontend/` | React/TypeScript 控制台、API/SSE 客户端、浏览器测试 | 数据库迁移或服务端凭据 |
| `database/` | 数据库契约、种子数据说明、迁移索引 | 第二套可执行的迁移系统 |
| `infra/` | Compose 支持、Prometheus、Grafana、Helm 与部署清单 | 领域业务逻辑 |
| `docs/` | 架构、配置、API 与运维文档 | 运行时配置或臆造结论 |

## 后端分层

`app/core` 提供配置与跨领域基础设施。`app/modules` 存放有界领域。`app/api.py` 是带版本号的 API 入口，领域服务通过仓储（repository）和已声明的端口（port）访问外部依赖。后台任务必须先持久化状态，再执行外部操作。

## 运行时边界

API 与 Worker 共享平台侧 PostgreSQL 数据库和 Redis 消息代理。MinIO 充当本地 S3 兼容对象存储。客户的 MySQL/PostgreSQL 数据库通过短生命周期、只读连接访问，并且永远不会挂载进应用容器。

## 环境模式

- 开发（Development）：使用 Compose，自动创建数据库结构，写入演示平台种子数据，本地同步执行摄取。
- 测试（Test）：使用隔离的 SQLite/PostgreSQL 测试数据库，模型服务全部 mock，不接触外部客户数据。
- 生产（Production）：显式执行 Alembic 迁移，使用外部 PostgreSQL/Redis/S3/OIDC，不加载演示种子，任务通过 broker 异步执行。

## 企业级 Agent 运行时

根 LangGraph 使用 `TypedDict + Annotated` 作为状态归约器（state reducer），并用 PostgreSQL 做持久化检查点（checkpoint）。Router 输出 `data`、`knowledge`、`hybrid` 或 `unsupported`；hybrid 任务通过 LangGraph 的 `Send` 并发运行数据子代理和知识子代理。Handoff、重试与事实校验都保存在图状态中，而不是放在浏览器代码里。

子代理调用单独部署的 MCP Server。工具调用携带短时租户令牌，经过 RBAC 校验、Schema 校验和审计，并且只允许只读 SQL 或走显式审批流程。分析事件写入 PostgreSQL，再发布到 Redis，最终通过 SSE 以 `Last-Event-ID` 回放。

知识检索采用父子分块（parent/child chunks）、Multi-Query、HyDE、稠密检索与 PostgreSQL BM25 检索、RRF 融合、重排以及父文档上下文映射。用户记忆独立于检查点存储，带有类别白名单、置信度、版本、过期时间和用户自助控制。

## API 稳定性规则

所有变更都必须保持现有 `/api/v1` 契约不变。新增企业级 API 时使用追加式路由（additive routes）。一个模块的改动只有在 API 测试与领域测试同时通过后，才算完成。
