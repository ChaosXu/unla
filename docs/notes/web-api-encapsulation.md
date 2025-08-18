# Web前端API请求封装说明

在unla项目的web目录中，前端对后端服务的API请求进行了封装，主要分为以下几个模块：

## 1. 基础API服务 ([api.ts](/web/src/services/api.ts))

这个文件封装了大部分后端API请求，主要包含以下几类：

### MCP配置管理API
- `getMCPServers(tenantId?)` - 获取MCP服务器配置列表
- `createMCPServer(config)` - 创建MCP服务器配置
- `updateMCPServer(config)` - 更新MCP服务器配置
- `deleteMCPServer(tenant, name)` - 删除MCP服务器配置
- `exportMCPServer(server)` - 导出MCP服务器配置
- `syncMCPServers()` - 同步MCP服务器配置

### 聊天会话API
- `getChatMessages(sessionId, page, pageSize)` - 获取聊天消息
- `getChatSessions()` - 获取聊天会话列表
- `deleteChatSession(sessionId)` - 删除聊天会话
- `updateChatSessionTitle(sessionId, title)` - 更新聊天会话标题
- `saveChatMessage(message)` - 保存聊天消息

### OpenAPI导入API
- `importOpenAPI(file, tenantName?, prefix?)` - 导入OpenAPI规范

### 租户管理API
- `getTenants()` - 获取租户列表
- `getTenant(name)` - 获取特定租户
- `createTenant(data)` - 创建租户
- `updateTenant(data)` - 更新租户
- `deleteTenant(name)` - 删除租户

### 用户管理API
- `getUsers()` - 获取用户列表
- `getUser(username)` - 获取特定用户
- `createUser(data)` - 创建用户
- `updateUser(data)` - 更新用户
- `deleteUser(username)` - 删除用户
- `toggleUserStatus(username, isActive)` - 切换用户状态
- `getUserWithTenants(username)` - 获取用户及其关联租户
- `getUserAuthorizedTenants()` - 获取当前用户授权的租户
- `updateUserTenants(userId, tenantIds)` - 更新用户租户关联

### MCP配置版本管理API
- `getMCPConfigNames(tenant?)` - 获取MCP配置名称列表
- `getMCPConfigVersions(tenant?, name?)` - 获取MCP配置版本列表
- `setActiveVersion(tenant, name, version)` - 设置活跃版本

### 用户信息API
- `getCurrentUser()` - 获取当前用户信息

## 2. MCP服务 ([mcp.ts](/web/src/services/mcp.ts))

这个文件封装了与MCP协议相关的操作：

- `connect(config, authToken?, headerName?)` - 连接到MCP服务器
- [reconnect(serverName)](/web/src/services/mcp.ts#L119-L130) - 重新连接到MCP服务器
- [getTools(serverName)](/web/src/services/mcp.ts#L132-L147) - 获取MCP服务器工具列表
- `callTool(serverName, toolName, args, onLastEventIdUpdate?)` - 调用MCP工具
- [terminateSession(serverName)](/web/src/services/mcp.ts#L195-L212) - 终止MCP会话
- [disconnect(serverName)](/web/src/services/mcp.ts#L214-L237) - 断开与MCP服务器的连接
- [disconnectAll()](/web/src/services/mcp.ts#L239-L242) - 断开所有MCP连接

## 3. LLM聊天服务 ([llm-chat.ts](//web/src/services/llm-chat.ts))

这个文件封装了与LLM聊天相关的功能：

- `sendMessage(provider, messages, modelId?, availableTools?, onChunk?, onReasoningChunk?, onToolCall?, onComplete?, onError?)` - 发送消息到LLM
- [callTool(toolName, parameters)](/web/src/services/mcp.ts#L149-L193) - 调用工具（会调用MCP服务的callTool方法）

## 总结

前端通过这些封装好的服务模块，可以方便地调用后端API，主要涉及以下几个方面的功能：

1. **MCP配置管理** - 管理MCP服务器配置
2. **用户和租户管理** - 管理系统用户和租户
3. **聊天功能** - 处理聊天会话和消息
4. **MCP协议交互** - 与MCP服务器进行交互
5. **LLM集成** - 与大语言模型进行交互

这些封装都基于axios库，并添加了统一的错误处理、认证和拦截器，使得前端可以更方便地与后端服务进行交互。