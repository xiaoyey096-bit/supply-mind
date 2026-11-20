# SupplyMind API 参考

Base URL：`/api/v1`
认证方式：`Authorization: Bearer <access_token>`
所有响应都包含 `X-Trace-ID`。排查客户端重试问题时，请把同一个值放进 `X-Trace-ID` 请求头进行关联。

## 认证与组织

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| POST | `/auth/login` | 使用 `email`、`password`、`organization_slug` 登录。 |
| POST | `/auth/refresh` | 轮换刷新令牌并签发新的访问令牌。旧令牌被重复使用时会被拒绝。 |
| GET | `/auth/organizations` | 列出当前用户可用的组织。 |
| POST | `/auth/switch-organization` | 签发携带新组织上下文的令牌。 |
| GET | `/auth/oidc/start?organization_slug=` | 携带 state 与 nonce 发起 OIDC 授权。 |
| GET | `/auth/oidc/callback?code=&state=` | 校验 OIDC 声明；除非 `SUPPLYMIND_OIDC_AUTO_PROVISION=true`，否则新身份保持待审批状态。 |
| GET | `/members` | 列出当前组织成员。 |
| POST | `/members/invitations` | 创建邀请。 |
| POST | `/members/invitations/{id}/resend` | 轮换并重发邀请令牌。 |
| POST | `/members/invitations/accept` | 消费一次性邀请令牌。 |
| PATCH | `/members/{user_id}` | 修改成员角色。 |
| PATCH | `/members/{user_id}/status` | 启用或停用成员。 |

## 数据源与 Schema

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| POST | `/data-sources` | 创建 PostgreSQL/MySQL 连接定义。密钥在存储时加密。 |
| POST | `/data-sources/{id}/test` | 测试只读连接。 |
| POST | `/data-sources/{id}/sync` | 排队执行一次真实 Schema 同步任务。 |
| GET | `/data-sources/{id}/sync-tasks` | 查看排队中/运行中/已完成/失败的同步任务。 |
| PATCH | `/data-sources/{id}/tls` | 更新组织管理员对该数据源的 TLS 要求。 |
| GET | `/data-sources/{id}/schema` | 读取最新的 Schema 快照。 |
| GET | `/data-sources/{id}/schema/tables/{table}` | 读取单表的列、键、索引和采样上限。 |
| PATCH | `/data-sources/{id}/allowlist` | 保存从最新 Schema 快照中选出的表白名单。 |
| POST | `/data-sources/{id}/query` | 执行经 Guard 校验的只读查询，并受行数和时间限制。 |
| PATCH | `/data-sources/{id}/status` | 停用或重新启用数据源。已停用的数据源不能查询或分析。 |

## 知识库与检索

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| GET | `/knowledge-bases?page=&page_size=&status=&name=` | 按条件筛选知识库。 |
| POST | `/knowledge-bases` | 创建知识库。 |
| POST | `/knowledge-bases/{id}/documents` | 上传 Markdown、TXT 或 PDF 并开始摄取。 |
| GET | `/knowledge-bases/{id}/documents` | 列出文档、分类与 Embedding 状态。 |
| GET | `/documents/{id}/versions` | 列出当前租户范围内该文档的版本与校验和。 |
| POST | `/documents/{id}/versions/{version}/rollback` | 排队执行受控回滚，并重新摄取选定的版本。 |
| PATCH | `/documents/{id}/metadata` | 更新指标名称、定义、公式、单位、适用范围和生效日期。 |
| GET | `/knowledge-bases/{id}/documents/{document_id}/source` | 读取版本、分块与来源定位。 |
| POST | `/knowledge-bases/{id}/search` | 真实 Embedding 检索。Embedding 缺失或失败时返回明确错误。 |
| POST | `/ingestion-tasks/{id}/retry` | 重试失败的摄取任务。 |
| POST | `/ingestion-tasks/{id}/cancel` | 取消排队中/处理中的任务。 |
| GET | `/ingestion-tasks/{id}` | 读取任务状态与失败详情。 |
| GET | `/ingestion-tasks?status=failed` | 分页查看失败/死信任务。 |
| POST | `/ingestion-tasks/{id}/dead-letter/retry` | 运维人工确认后重新入队死信任务。 |

## 分析与报告

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| POST | `/analyses` | 创建幂等的异步分析任务，并返回可恢复的事件流 URL。 |
| GET | `/analyses/{id}/stream` | 回放持久化的 SSE 事件。断线后发送 `Last-Event-ID` 可续传。 |
| POST | `/analyses/stream` | 向后兼容接口：在同一个响应中创建分析并流式返回事件。 |
| GET | `/analyses/{id}` | 读取运行状态、SQL 草稿/最终 SQL、Guard 错误与结果。 |
| GET | `/analyses/{id}/steps` | 读取持久化的 Agent 执行轨迹。 |
| GET | `/analyses/{id}/events` | SSE 断线后恢复一次运行。 |
| POST | `/analyses/{id}/retry` | 带血缘关系重试失败/已取消的运行。 |
| POST | `/analyses/{id}/cancel` | 取消排队中/运行中的任务。 |
| GET | `/reports` | 按标题、状态、创建人、时间范围和分析运行 ID 筛选。 |
| GET | `/reports/{id}` | 读取 Markdown 正文、引用和运行元数据。 |
| POST | `/reports/{id}/exports/pdf` | 排队执行 PDF 导出。加 `?force=true` 可在已有完成导出时，用最新渲染器再生成一份。 |
| GET | `/reports/{id}/exports/pdf` | 读取导出状态（排队中/运行中/已完成/失败）。 |
| POST | `/reports/{id}/exports/{export_id}/retry` | 重试失败的导出。 |
| GET | `/reports/{id}/exports/pdf/download` | 下载已完成的本地产物或 MinIO/S3 对象。 |

## Agent 运行时、记忆与工具

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| GET/PATCH | `/me/memory/settings` | 读取或修改当前用户的长期记忆开关。 |
| GET/POST | `/me/memories` | 列出或添加租户/用户范围内的白名单记忆。 |
| PATCH/DELETE | `/me/memories/{id}` | 编辑或删除单条记忆。 |
| DELETE | `/me/memories` | 清空当前用户的全部记忆。 |
| GET/POST | `/mcp/servers` | 列出或注册经审批的外部 MCP Server（组织管理员）。 |
| PATCH/DELETE | `/mcp/servers/{id}` | 启用、更新或移除已注册的 Server。 |
| POST | `/mcp/servers/{id}/test` | 通过已配置的传输方式发现工具。 |
| GET | `/agent-approvals?status=pending` | 列出待审批的工具调用。 |
| POST | `/agent-approvals/{id}/approve` | 批准导出或其他有副作用的工具请求。 |
| POST | `/agent-approvals/{id}/reject` | 驳回审批请求。 |
| POST | `/ai/evaluations` | 对已批准的数据集版本排队执行评测。 |
| GET | `/ai/evaluations` | 读取评测状态、指标和未通过的门禁项。 |

## A2A

`GET /.well-known/agent-card.json` 暴露只读的 Agent Card。`POST /a2a` 接受 JSON-RPC 的 `message/send`、`tasks/get` 和 `tasks/cancel`；任务事件使用 `GET /a2a/tasks/{id}/stream` 并配合 `Last-Event-ID`。A2A 使用与主 API 相同的 JWT 组织边界，不暴露写工具或凭据。

## 大屏、审计与状态

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| GET | `/dashboards/supply-chain` | 读取五个由真实 SQL 聚合得到的供应链指标组件。 |
| GET | `/dashboards/supply-chain/config` | 读取组织级刷新与组件配置。 |
| PATCH | `/dashboards/supply-chain/config` | 更新组织级刷新与组件配置（仅组织/平台管理员）。 |
| GET | `/dashboards/supply-chain/dimensions` | 读取动态的工厂/产品线/供应商筛选项。 |
| POST | `/dashboards/supply-chain/refresh` | 排队执行一次组织管理员触发的刷新任务。 |
| GET | `/dashboards/supply-chain/refresh/{task_id}` | 读取刷新状态与失败原因。 |
| GET | `/audit` | 按操作、资源、操作人、时间范围、分页和运行 ID 筛选。返回 `X-Total-Count`、`X-Page-Offset`、`X-Page-Limit` 响应头。 |
| GET | `/audit/{event_id}` | 读取当前租户范围内的一条审计事件。敏感值会被脱敏；其他组织访问返回 `404`。 |
| GET | `/system/status` | 读取 PostgreSQL、Redis、Worker、Chat、Embedding、MinIO、MCP 与数据源状态，包括失败/死信任务数量和近期错误摘要。 |
| GET | `/health/live` | 存活检查。 |
| GET | `/health/ready` | 依赖就绪检查。 |

## 跨组织行为

所有资源查询都会带上 JWT 中的组织上下文和 service 层的 `tenant_id`。属于其他组织的资源一律返回 `404`，包括报告、导出、数据源、文档、分析、大屏任务和审计记录。

## 验证顺序

1. `docker compose config --quiet`
2. `docker compose up -d`
3. `docker compose exec -T api alembic upgrade head`
4. `docker compose exec -T api pytest -q`
5. 在 `frontend` 目录执行 `npm run build`
6. 按 `docs/ACCEPTANCE_CHECKLIST.md` 走一遍浏览器验收路径。
