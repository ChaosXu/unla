# MCP Gateway 实现分析

本文档详细分析了 [mcp-gateway/main.go](/cmd/mcp-gateway/main.go) 的实现，特别是 MCP 协议的实现以及如何实现 OpenAPI 服务的代理。

## 概述

MCP Gateway 是 unla 系统的核心组件，负责将现有的 MCP 服务器和 API 转换为符合 [MCP 协议](https://modelcontextprotocol.io/) 的端点。它支持多种协议类型，包括 RESTful API 到 MCP Server 的转换、代理 MCP 服务、MCP SSE 支持、MCP Streamable HTTP 支持等。

## 程序入口分析

### 主函数和命令行接口

[mcp-gateway/main.go](/cmd/mcp-gateway/main.go) 使用 [cobra](https://github.com/spf13/cobra) 库实现命令行接口，支持以下子命令：

1. **version**: 打印 MCP Gateway 版本号
2. **reload**: 重新加载运行中的 MCP Gateway 实例配置
3. **test**: 测试 MCP Gateway 配置

主程序通过 [run()](/cmd/mcp-gateway/main.go#L124-L254) 函数启动服务：

```go
func run() {
    // 加载配置
    cfg, cfgPath, err := config.LoadConfig[config.MCPGatewayConfig](configPath)
    
    // 初始化日志记录器
    logger, err := logger.NewLogger(&cfg.Logger)
    
    // 初始化 PID 管理器
    pidManager := utils.NewPIDManagerFromConfig(pidFile)
    
    // 初始化存储和加载初始配置
    store, err := storage.NewStore(logger, &cfg.Storage)
    
    // 初始化会话存储
    sessionStore, err := session.NewStore(logger, &cfg.Session)
    
    // 初始化认证服务
    a, err := auth.NewAuth(logger, cfg.Auth)
    
    // 创建服务器实例
    server, err := core.NewServer(logger, cfg.Port, store, sessionStore, a, cfg.Forward)
    
    // 注册路由
    err = server.RegisterRoutes(ctx)
    
    // 初始化通知器
    ntf, err := notifier.NewNotifier(ctx, logger, &cfg.Notifier)
    updateCh, err := ntf.Watch(ctx)
    
    // 启动服务器
    server.Start()
    
    // 主事件循环
    for {
        select {
        case <-quit:
            // 处理关闭信号
        case updateMCPConfig := <-updateCh:
            // 处理配置更新
        case <-ticker.C:
            // 定期重新加载配置
        }
    }
}
```

## MCP 协议实现

### 核心服务 ([core.Server](/internal/core/server.go#L48-L69))

MCP 协议的核心实现在 [internal/core/server.go](/internal/core/server.go) 中，[Server](/internal/core/server.go#L48-L69) 结构体包含以下关键组件：

```go
type Server struct {
    logger *zap.Logger
    port   int
    router *gin.Engine
    // state 包含所有只读共享状态
    state *state.State
    // store 是 MCP 配置的存储服务
    store storage.Store
    // sessions 管理所有活动会话
    sessions session.Store
    // shutdownCh 用于向所有 SSE 连接发送关闭信号
    shutdownCh chan struct{}
    // toolRespHandler 是响应处理程序链
    toolRespHandler ResponseHandler
    lastUpdateTime  time.Time
    auth            auth.Auth
    forwardConfig   config.ForwardConfig
    // 预解析的头部列表，用于高效查找
    ignoreHeaders   []string
    allowHeaders    []string
    caseInsensitive bool
}
```

### 路由注册

[RegisterRoutes](/internal/core/server.go#L105-L152) 方法负责注册路由：

```go
func (s *Server) RegisterRoutes(ctx context.Context) error {
    // 注册健康检查端点
    s.router.GET("/health_check", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "status":  "ok",
            "message": "Health check passed.",
        })
    })

    // 如果配置了 OAuth2，注册 OAuth 路由
    if s.auth.IsOAuth2Enabled() {
        // 注册 OAuth 路由
    }

    // 更新配置
    newState, err := s.updateConfigs(ctx)
    
    // 原子性替换状态
    s.state = newState

    // 注册所有路由的根处理程序
    s.router.NoRoute(s.handleRoot)
    
    return nil
}
```

### 请求处理流程

所有请求由 [handleRoot](/internal/core/server.go#L155-L232) 方法处理：

```go
func (s *Server) handleRoot(c *gin.Context) {
    // 解析路径
    path := c.Request.URL.Path
    parts := strings.Split(strings.Trim(path, "/"), "/")
    endpoint := parts[len(parts)-1]
    prefix := "/" + strings.Join(parts[:len(parts)-1], "/")

    // 检查认证配置
    auth := s.state.GetAuth(prefix)
    if auth != nil && auth.Mode == cnst.AuthModeOAuth2 {
        // 验证访问令牌
    }

    // 动态设置 CORS
    if cors := s.state.GetCORS(prefix); cors != nil {
        s.corsMiddleware(cors)(c)
    }

    // 根据端点类型分发请求
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
}
```

### MCP 协议端点实现

MCP 协议端点由 [handleMCP](/internal/core/streamable.go#L20-L40) 方法处理：

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

#### GET 请求处理 ([handleGet](/internal/core/streamable.go#L48-L110))

处理 SSE 流连接：

```go
func (s *Server) handleGet(c *gin.Context) {
    // 检查 Accept 头部是否包含 text/event-stream
    acceptHeader := c.GetHeader("Accept")
    if !strings.Contains(acceptHeader, "text/event-stream") {
        s.sendProtocolError(c, nil, "Not Acceptable: Client must accept text/event-stream", http.StatusNotAcceptable, mcp.ErrorCodeInvalidRequest)
        return
    }

    conn := s.getSession(c)
    if conn == nil {
        return
    }

    // 设置响应头
    c.Writer.Header().Set("Content-Type", "text/event-stream")
    c.Writer.Header().Set("Cache-Control", "no-cache, no-transform")
    c.Writer.Header().Set("Connection", "keep-alive")
    c.Writer.Header().Set(mcp.HeaderMcpSessionID, conn.Meta().ID)
    c.Writer.Flush()

    // 处理事件流
    for {
        select {
        case event := <-conn.EventQueue():
            switch event.Event {
            case "message":
                _, err := fmt.Fprintf(c.Writer, "event: message\ndata: %s\n\n", event.Data)
                if err != nil {
                    s.logger.Error("failed to send SSE message", zap.Error(err))
                }
            }
            c.Writer.Flush()
        case <-c.Request.Context().Done():
            return
        case <-s.shutdownCh:
            return
        }
    }
}
```

#### POST 请求处理 ([handlePost](/internal/core/streamable.go#L113-L234))

处理 JSON-RPC 消息：

```go
func (s *Server) handlePost(c *gin.Context) {
    // 验证 Accept 和 Content-Type 头部
    accept := c.GetHeader("Accept")
    contentType := c.GetHeader("Content-Type")

    // 绑定 JSON-RPC 请求
    var req mcp.JSONRPCRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        s.sendProtocolError(c, nil, "Invalid JSON-RPC request", http.StatusBadRequest, mcp.ErrorCodeParseError)
        return
    }

    sessionID := c.GetHeader(mcp.HeaderMcpSessionID)

    var (
        conn session.Connection
        err  error
    )
    if req.Method == mcp.Initialize {
        // 处理初始化请求
    } else {
        // 获取现有会话
        conn, err = s.sessions.Get(c.Request.Context(), sessionID)
    }

    s.handleMCPRequest(c, req, conn)
}
```

#### MCP 请求处理 ([handleMCPRequest](/internal/core/streamable.go#L273-L436))

根据方法类型处理不同的 MCP 请求：

```go
func (s *Server) handleMCPRequest(c *gin.Context, req mcp.JSONRPCRequest, conn session.Connection) {
    switch req.Method {
    case mcp.Initialize:
        // 处理初始化请求
    case mcp.NotificationInitialized:
        // 处理初始化通知
    case mcp.ToolsList:
        // 处理工具列表请求
    case mcp.ToolsCall:
        // 处理工具调用请求
    case mcp.PromptsList:
        // 处理提示列表请求
    case mcp.PromptsGet:
        // 处理获取提示请求
    default:
        // 处理未知方法
    }
}
```

### 工具列表处理 ([ToolsList](/internal/core/streamable.go#L273-L436))

```go
case mcp.ToolsList:
    protoType := s.state.GetProtoType(conn.Meta().Prefix)
    
    var tools []mcp.ToolSchema
    var err error
    switch protoType {
    case cnst.BackendProtoHttp:
        tools, err = s.fetchHTTPToolList(conn)
    case cnst.BackendProtoStdio, cnst.BackendProtoSSE, cnst.BackendProtoStreamable:
        transport := s.state.GetTransport(conn.Meta().Prefix)
        tools, err = transport.FetchTools(c.Request.Context())
    default:
        s.sendProtocolError(c, req.Id, "Unsupported protocol type", http.StatusBadRequest, mcp.ErrorCodeInvalidParams)
        return
    }

    s.sendSuccessResponse(c, conn, req, mcp.ListToolsResult{
        Tools: tools,
    }, false)
    return
```

### 工具调用处理 ([ToolsCall](/internal/core/streamable.go#L273-L436))

```go
case mcp.ToolsCall:
    protoType := s.state.GetProtoType(conn.Meta().Prefix)
    
    // 解析工具调用参数
    var params mcp.CallToolParams
    if err := json.Unmarshal(req.Params, &params); err != nil {
        s.sendProtocolError(c, req.Id, fmt.Sprintf("invalid tool call parameters: %v", err), http.StatusBadRequest, mcp.ErrorCodeInvalidParams)
        return
    }

    var (
        result *mcp.CallToolResult
        err    error
    )
    switch protoType {
    case cnst.BackendProtoHttp:
        result = s.callHTTPTool(c, req, conn, params)
    case cnst.BackendProtoStdio, cnst.BackendProtoSSE, cnst.BackendProtoStreamable:
        transport := s.state.GetTransport(conn.Meta().Prefix)
        result, err = transport.CallTool(c.Request.Context(), params, mergeRequestInfo(conn.Meta().Request, c.Request))
    default:
        s.sendProtocolError(c, req.Id, "Unsupported protocol type", http.StatusBadRequest, mcp.ErrorCodeInvalidParams)
        return
    }

    s.sendSuccessResponse(c, conn, req, result, false)
    return
```

## OpenAPI 服务代理实现

### HTTP 工具调用实现

对于 HTTP 类型的工具（即 OpenAPI 转换的工具），实现在 [internal/core/tool.go](/internal/core/tool.go) 中。

#### 获取工具列表 ([fetchHTTPToolList](/internal/core/tool.go#L343-L365))

```go
func (s *Server) fetchHTTPToolList(conn session.Connection) ([]mcp.ToolSchema, error) {
    // 获取此前缀的 HTTP 工具
    tools := s.state.GetToolSchemas(conn.Meta().Prefix)
    if len(tools) == 0 {
        tools = []mcp.ToolSchema{} // 如果未找到前缀则返回空列表
    }
    return tools, nil
}
```

#### 调用 HTTP 工具 ([callHTTPTool](/internal/core/tool.go#L367-L472))

```go
func (s *Server) callHTTPTool(c *gin.Context, req mcp.JSONRPCRequest, conn session.Connection, params mcp.CallToolParams) *mcp.CallToolResult {
    // 查找工具
    tool := s.state.GetTool(conn.Meta().Prefix, params.Name)
    if tool == nil {
        s.sendProtocolError(c, req.Id, "Tool not found", http.StatusNotFound, mcp.ErrorCodeMethodNotFound)
        return nil
    }

    // 转换参数
    var args map[string]any
    if err := json.Unmarshal(params.Arguments, &args); err != nil {
        s.sendProtocolError(c, req.Id, "Invalid tool arguments", http.StatusBadRequest, mcp.ErrorCodeInvalidParams)
        return nil
    }

    // 获取服务器配置
    serverCfg := s.state.GetServerConfig(conn.Meta().Prefix)
    
    // 执行工具
    result, err := s.executeHTTPTool(conn, tool, args, c.Request, serverCfg.Config)
    if err != nil {
        s.sendToolExecutionError(c, conn, req, err, true)
        return nil
    }

    return result
}
```

#### 执行 HTTP 工具 ([executeHTTPTool](/internal/core/tool.go#L100-L298))

```go
func (s *Server) executeHTTPTool(conn session.Connection, tool *config.ToolConfig,
    args map[string]any, request *http.Request, serverCfg map[string]string) (*mcp.CallToolResult, error) {
    // 填充默认参数值
    fillDefaultArgs(tool, args)

    // 准备模板上下文
    tmplCtx, err := template.PrepareTemplateContext(conn.Meta().Request, args, request, serverCfg)
    if err != nil {
        return nil, err
    }

    // 准备 HTTP 请求
    req, err := s.prepareRequest(tool, tmplCtx)
    if err != nil {
        return nil, err
    }

    // 处理参数
    processArguments(req, tool, args)

    // 执行请求
    cli, err := createHTTPClient(tool)
    if err != nil {
        return nil, fmt.Errorf("failed to create HTTP client: %w", err)
    }

    resp, err := cli.Do(req)
    if err != nil {
        return nil, fmt.Errorf("failed to execute request: %w", err)
    }
    defer resp.Body.Close()

    // 处理响应
    callToolResult, err := s.toolRespHandler.Handle(resp, tool, tmplCtx)
    if err != nil {
        return nil, err
    }

    return callToolResult, nil
}
```

#### 响应处理链

响应处理使用责任链模式实现，支持多种类型的响应：

```go
// CreateResponseHandlerChain 创建响应处理程序链
func CreateResponseHandlerChain() ResponseHandler {
    imageHandler := &ImageHandler{}
    audioHandler := &AudioHandler{}
    textHandler := &TextHandler{}

    imageHandler.SetNext(audioHandler)
    audioHandler.SetNext(textHandler)
    return imageHandler
}
```

各类型响应处理器：
1. [ImageHandler](/internal/core/handler.go#L109-L127) - 处理图像响应
2. [AudioHandler](/internal/core/handler.go#L130-L150) - 处理音频响应
3. [TextHandler](/internal/core/handler.go#L63-L86) - 处理文本响应

### MCP 传输实现

对于非 HTTP 类型的 MCP 服务（stdio、SSE、Streamable HTTP），使用 [mcpproxy](/internal/core/mcpproxy) 包实现传输：

```go
// Transport 定义 MCP 传输实现的接口
type Transport interface {
    // FetchTools 获取可用工具列表
    FetchTools(ctx context.Context) ([]mcp.ToolSchema, error)

    // CallTool 调用工具
    CallTool(ctx context.Context, params mcp.CallToolParams, req *template.RequestWrapper) (*mcp.CallToolResult, error)

    // Start 启动传输
    Start(ctx context.Context, tmplCtx *template.Context) error

    // Stop 停止传输
    Stop(ctx context.Context) error

    // IsRunning 返回传输是否正在运行
    IsRunning() bool

    // FetchPrompts 获取可用提示列表
    FetchPrompts(ctx context.Context) ([]mcp.PromptSchema, error)
    // FetchPrompt 根据名称获取特定提示
    FetchPrompt(ctx context.Context, name string) (*mcp.PromptSchema, error)
}
```

具体实现：
1. [SSETransport](/internal/core/mcpproxy/sse.go#L27-L79) - SSE 传输实现
2. [StdioTransport](/internal/core/mcpproxy/stdio.go#L34-L138) - Stdio 传输实现
3. [StreamableTransport](/internal/core/mcpproxy/streamable.go#L26-L90) - Streamable HTTP 传输实现

## 总结

MCP Gateway 的实现具有以下特点：

1. **模块化设计**：使用清晰的模块划分，包括核心服务、路由处理、MCP 协议实现、HTTP 工具处理等。

2. **多协议支持**：支持多种后端协议类型，包括 HTTP（用于 OpenAPI 代理）、stdio、SSE 和 Streamable HTTP。

3. **灵活的配置管理**：通过存储接口支持多种配置存储方式，支持配置热更新和通知机制。

4. **会话管理**：实现完整的会话管理机制，支持会话的创建、获取和注销。

5. **认证和授权**：支持 OAuth2 认证，可以根据前缀配置不同的认证方式。

6. **CORS 支持**：支持动态 CORS 配置，可以根据前缀应用不同的 CORS 策略。

7. **响应处理链**：使用责任链模式处理不同类型的响应，支持文本、图像和音频响应。

8. **模板系统**：使用模板系统处理请求和响应，支持动态参数替换。

9. **代理功能**：实现完整的 HTTP 代理功能，支持请求头转发、参数处理等。

通过这些设计和实现，MCP Gateway 能够将现有的 API 和 MCP 服务转换为统一的 MCP 协议接口，为客户端提供一致的访问方式。