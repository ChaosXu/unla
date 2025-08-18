# /gateway 请求处理分析

本文档详细分析了 `/gateway` 请求的处理流程，包括请求发往哪里以及由谁处理。

## 概述

在 unla 系统中，`/gateway` 请求主要通过反向代理转发到 MCP Gateway 服务进行处理。MCP Gateway 是系统的核心组件，负责处理所有与 MCP 协议相关的请求。

## 请求路由流程

### 1. Nginx 反向代理

在多容器部署模式下，请求首先通过 Nginx 进行路由：

```nginx
location /gateway/ {
    proxy_pass http://mcp-gateway;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

在 all-in-one 部署模式下，配置类似：

```nginx
location /gateway/ {
    proxy_pass http://mcp-gateway/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### 2. MCP Gateway 服务接收

MCP Gateway 服务监听在特定端口（默认为 5235），接收来自 Nginx 的请求。

## MCP Gateway 内部路由处理

### 1. 主要入口点

MCP Gateway 的主要入口在 [cmd/mcp-gateway/main.go](/cmd/mcp-gateway/main.go) 中：

```go
// 创建服务实例
server, err := core.NewServer(logger, cfg.Port, store, sessionStore, a, cfg.Forward)

// 注册路由
err = server.RegisterRoutes(ctx)

// 启动服务
server.Start()
```

### 2. 路由注册

在 [internal/core/server.go](/internal/core/server.go) 中的 [RegisterRoutes](/internal/core/server.go#L105-L152) 方法中注册了核心路由：

```go
// 注册根处理器
s.router.NoRoute(s.handleRoot)
```

### 3. 请求处理流程

所有请求最终由 [handleRoot](/internal/core/server.go#L155-L232) 方法处理：

1. 解析请求路径，分离前缀和端点：
   ```go
   path := c.Request.URL.Path
   parts := strings.Split(strings.Trim(path, "/"), "/")
   endpoint := parts[len(parts)-1]
   prefix := "/" + strings.Join(parts[:len(parts)-1], "/")
   ```

2. 验证认证信息（如果配置了 OAuth2）：
   ```go
   auth := s.state.GetAuth(prefix)
   if auth != nil && auth.Mode == cnst.AuthModeOAuth2 {
       // 验证访问令牌
       if !s.isValidAccessToken(c.Request) {
           // 返回认证错误
       }
   }
   ```

3. 应用 CORS 配置（如果配置了 CORS）：
   ```go
   if cors := s.state.GetCORS(prefix); cors != nil {
       s.corsMiddleware(cors)(c)
   }
   ```

4. 根据端点类型分发请求：
   ```go
   switch endpoint {
   case "sse":
       s.handleSSE(c)
   case "message":
       s.handleMessage(c)
   case "mcp":
       s.handleMCP(c)
   default:
       s.sendProtocolError(c, nil, "Invalid endpoint", http.StatusNotFound, mcp.ErrorCodeInvalidRequest)
   }
   ```

### 4. MCP 协议处理

对于 `/mcp` 端点，请求由 [handleMCP](/internal/core/streamable.go#L20-L40) 方法处理：

```go
func (s *Server) handleMCP(c *gin.Context) {
    switch c.Request.Method {
    case http.MethodOptions:
        c.Status(http.StatusOK)
        return
    case http.MethodGet:
        s.handleGet(c)
    case http.MethodPost:
        s.handlePost(c)
        return
    case http.MethodDelete:
        s.handleDelete(c)
        return
    default:
        c.Header("Allow", "GET, POST, DELETE")
        s.sendProtocolError(c, nil, "Method not allowed", http.StatusMethodNotAllowed, mcp.ErrorCodeConnectionClosed)
        return
    }
}
```

具体的处理方法包括：
- [handleGet](/internal/core/streamable.go#L48-L110): 处理 GET 请求，建立流式连接
- [handlePost](/internal/core/streamable.go#L113-L234): 处理 POST 请求，处理 MCP 协议消息
- [handleDelete](/internal/core/streamable.go#L237-L271): 处络 DELETE 请求，关闭会话

### 5. MCP 协议消息处理

在 [handlePost](/internal/core/streamable.go#L113-L234) 方法中，MCP 协议消息被解析并分发到相应的处理器：

```go
// 解析 MCP 请求
req, err := mcp.ParseRequest(body)
if err != nil {
    s.sendProtocolError(c, id, "Failed to parse request", http.StatusBadRequest, mcp.ErrorCodeInvalidRequest)
    return
}

// 根据方法类型处理请求
switch req.Method {
case "tools/list":
    s.handleListTools(c, req, id, prefix)
case "tools/call":
    s.handleCallTool(c, req, id, prefix)
case "resources/list":
    s.handleListResources(c, req, id, prefix)
case "resources/read":
    s.handleReadResource(c, req, id, prefix)
case "prompts/list":
    s.handleListPrompts(c, req, id, prefix)
case "prompts/get":
    s.handleGetPrompt(c, req, id, prefix)
case "ping":
    s.handlePing(c, req, id, prefix)
default:
    s.sendProtocolError(c, id, "Unsupported method", http.StatusBadRequest, mcp.ErrorCodeMethodNotSupported)
}
```

主要的处理方法包括：
- [handleListTools](/internal/core/handler.go#L25-L62): 处理工具列表请求
- [handleCallTool](/internal/core/handler.go#L65-L134): 处理工具调用请求
- [handleListResources](/internal/core/handler.go#L137-L160): 处理资源列表请求
- [handleReadResource](/internal/core/handler.go#L163-L201): 处理资源读取请求
- [handleListPrompts](/internal/core/handler.go#L204-L227): 处理提示列表请求
- [handleGetPrompt](/internal/core/handler.go#L230-L277): 处理提示获取请求
- [handlePing](/internal/core/handler.go#L280-L287): 处理 ping 请求

## 前端调用示例

在前端代码中，[MCPService](/web/src/services/mcp.ts#L24-L250) 通过以下方式构建请求 URL：

```typescript
const gatewayBaseUrl = window.RUNTIME_CONFIG?.VITE_MCP_GATEWAY_BASE_URL || '';
const serverUrl = new URL(`${gatewayBaseUrl}${prefix}/mcp`, window.location.origin);
```

其中：
- `gatewayBaseUrl` 通常为 `/gateway`
- `prefix` 是配置中定义的路由前缀，如 `/user`
- 最终 URL 为 `/gateway/user/mcp`

## 总结

1. **请求入口**: `/gateway` 请求首先通过 Nginx 反向代理转发到 MCP Gateway 服务
2. **路由处理**: MCP Gateway 服务通过 [handleRoot](/internal/core/server.go#L155-L232) 方法解析请求路径并分发到相应的处理器
3. **协议处理**: 对于 `/mcp` 端点，根据 HTTP 方法和 MCP 协议方法进一步分发到具体的处理器方法
4. **核心功能**: MCP Gateway 实现了完整的 MCP 协议，包括工具调用、资源访问、提示词管理等功能

整个流程确保了前端可以通过统一的 `/gateway` 路径访问后端的各种 MCP 服务，同时保持了良好的解耦和可扩展性。