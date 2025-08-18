# 前端API与后端服务对应关系

本文档说明了前端服务封装的API方法与其对应的后端服务端点之间的关系。

## 1. 基础API服务 ([api.ts](/web/src/services/api.ts))

### MCP配置管理API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `getMCPServers(tenantId?)` | `GET /api/mcp/configs` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleListMCPServers` |
| `createMCPServer(config)` | `POST /api/mcp/configs` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleMCPServerCreate` |
| `updateMCPServer(config)` | `PUT /api/mcp/configs` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleMCPServerUpdate` |
| `deleteMCPServer(tenant, name)` | `DELETE /api/mcp/configs/:tenant/:name` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleMCPServerDelete` |
| `syncMCPServers()` | `POST /api/mcp/configs/sync` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleSyncMCPServers` |

### 聊天会话API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `getChatMessages(sessionId, page, pageSize)` | `GET /api/chat/sessions/:sessionId/messages` | [handler/chat.go](/internal/apiserver/handler/chat.go) | `HandleGetChatMessages` |
| `getChatSessions()` | `GET /api/chat/sessions` | [handler/chat.go](/internal/apiserver/handler/chat.go) | `HandleGetChatSessions` |
| `deleteChatSession(sessionId)` | `DELETE /api/chat/sessions/:sessionId` | [handler/chat.go](/internal/apiserver/handler/chat.go) | `HandleDeleteChatSession` |
| `updateChatSessionTitle(sessionId, title)` | `PUT /api/chat/sessions/:sessionId/title` | [handler/chat.go](/internal/apiserver/handler/chat.go) | `HandleUpdateChatSessionTitle` |
| `saveChatMessage(message)` | `POST /api/chat/messages` | [handler/chat.go](/internal/apiserver/handler/chat.go) | `HandleSaveChatMessage` |

### OpenAPI导入API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `importOpenAPI(file, tenantName?, prefix?)` | `POST /api/openapi/import` | [handler/openapi.go](/internal/apiserver/handler/openapi.go) | `HandleImport` |

### 租户管理API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `getTenants()` | `GET /api/auth/tenants` | [handler/tenant.go](/internal/apiserver/handler/tenant.go) | `ListTenants` |
| `getTenant(name)` | `GET /api/auth/tenants/:name` | [handler/tenant.go](/internal/apiserver/handler/tenant.go) | `GetTenant` |
| `createTenant(data)` | `POST /api/auth/tenants` | [handler/tenant.go](/internal/apiserver/handler/tenant.go) | `CreateTenant` |
| `updateTenant(data)` | `PUT /api/auth/tenants` | [handler/tenant.go](/internal/apiserver/handler/tenant.go) | `UpdateTenant` |
| `deleteTenant(name)` | `DELETE /api/auth/tenants/:name` | [handler/tenant.go](/internal/apiserver/handler/tenant.go) | `DeleteTenant` |

### 用户管理API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `getUsers()` | `GET /api/auth/users` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `ListUsers` |
| `getUser(username)` | `GET /api/auth/users/:username` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `GetUser` |
| `createUser(data)` | `POST /api/auth/users` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `CreateUser` |
| `updateUser(data)` | `PUT /api/auth/users` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `UpdateUser` |
| `deleteUser(username)` | `DELETE /api/auth/users/:username` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `DeleteUser` |
| `toggleUserStatus(username, isActive)` | `PUT /api/auth/users` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `UpdateUser` |
| `getUserWithTenants(username)` | `GET /api/auth/users/:username` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `GetUser` |
| `getUserAuthorizedTenants()` | `GET /api/auth/user` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `GetUserInfo` |
| `updateUserTenants(userId, tenantIds)` | `PUT /api/auth/users/tenants` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `UpdateUserTenants` |

### MCP配置版本管理API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `getMCPConfigNames(tenant?)` | `GET /api/mcp/configs/names` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleGetMCPConfigNames` |
| `getMCPConfigVersions(tenant?, name?)` | `GET /api/mcp/configs/versions` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleGetMCPConfigVersions` |
| `setActiveVersion(tenant, name, version)` | `POST /api/mcp/configs/:tenant/:name/versions/:version/active` | [handler/mcp.go](/internal/apiserver/handler/mcp.go) | `HandleSetActiveVersion` |

### 用户信息API

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `getCurrentUser()` | `GET /api/auth/user/info` | [handler/auth.go](/internal/apiserver/handler/auth.go) | `GetUserInfo` |

## 2. MCP服务 ([mcp.ts](/web/src/services/mcp.ts))

MCP服务主要通过WebSocket连接与MCP网关进行通信，不直接调用API服务端点。

| 前端方法 | 后端服务端点 | 后端处理文件 | 后端函数 |
|---------|-------------|-------------|---------|
| `connect(config, authToken?, headerName?)` | WebSocket连接到 `/mcp` | [mcp-gateway](/cmd/mcp-gateway/main.go) | `HandleMCPStream` |
| [reconnect(serverName)](/web/src/services/mcp.ts#L119-L130) | WebSocket连接到 `/mcp` | [mcp-gateway](/cmd/mcp-gateway/main.go) | `HandleMCPStream` |
| [getTools(serverName)](/web/src/services/mcp.ts#L132-L147) | MCP协议 `tools/list` | [mcp-gateway](/cmd/mcp-gateway/main.go) | `HandleListTools` |
| `callTool(serverName, toolName, args, onLastEventIdUpdate?)` | MCP协议 `tools/call` | [mcp-gateway](/cmd/mcp-gateway/main.go) | `HandleCallTool` |

## 3. LLM聊天服务 ([llm-chat.ts](/web/src/services/llm-chat.ts))

LLM聊天服务直接与外部LLM提供商通信，不通过后端API服务。

| 前端方法 | 后端服务端点 | 说明 |
|---------|-------------|------|
| `sendMessage(provider, messages, modelId?, availableTools?, onChunk?, onReasoningChunk?, onToolCall?, onComplete?, onError?)` | 直接连接到LLM提供商 | 不通过后端API服务 |
| [callTool(toolName, parameters)](/web/src/services/mcp.ts#L149-L193) | 调用MCP服务的 `callTool` 方法 | 间接通过MCP服务与后端通信 |

## 总结

1. **API服务**：大部分前端功能通过REST API与后端[apiserver](/internal/apiserver/middleware/auth.go#L15)通信，主要处理配置管理、用户认证、聊天记录等。
2. **MCP协议**：MCP相关功能通过WebSocket连接直接与[mcp-gateway](/cmd/mcp-gateway/main.go)通信，实现MCP协议功能。
3. **LLM服务**：LLM聊天功能直接连接外部LLM提供商，不通过后端服务中转。