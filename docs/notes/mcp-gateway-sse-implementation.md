# MCP Gateway SSE 实现分析

## 概述

MCP Gateway 使用 Server-Sent Events (SSE) 作为与 MCP 客户端通信的一种方式。SSE 是一种允许服务器向客户端推送实时更新的 HTTP 技术。本文档详细分析了 MCP Gateway 中 SSE 的实现机制。

## SSE 基础知识

Server-Sent Events (SSE) 是一种服务器推送技术，允许服务器实时向客户端发送数据更新。与 WebSocket 不同，SSE 使用标准 HTTP 协议，只支持单向通信（服务器到客户端）。SSE 的主要特点包括：

1. 基于 HTTP 协议，易于实现和调试
2. 自动重连机制
3. 支持事件类型区分
4. 内置错误处理

## MCP Gateway 中的 SSE 实现

### 1. SSE 连接处理

MCP Gateway 中的 SSE 连接处理在 [internal/core/sse.go](/internal/core/sse.go) 文件中实现。主要入口点是 [handleSSE](/internal/core/sse.go#L15-L124) 函数。

```go
func (s *Server) handleSSE(c *gin.Context) {
    // 设置 SSE 必需的响应头
    c.Writer.Header().Set("Content-Type", "text/event-stream")
    c.Writer.Header().Set("Cache-Control", "no-cache, no-transform")
    c.Writer.Header().Set("Connection", "keep-alive")
    
    // 解析请求路径中的前缀
    prefix := strings.TrimSuffix(c.Request.URL.Path, "/sse")
    if prefix == "" {
        prefix = "/"
    }
    
    // 创建会话元数据
    sessionID := uuid.New().String()
    meta := &session.Meta{
        ID:        sessionID,
        CreatedAt: time.Now(),
        Prefix:    prefix,
        Type:      "sse",
        Request:   requestInfo,
    }
    
    // 注册会话
    conn, err := s.sessions.Register(c.Request.Context(), meta)
    if err != nil {
        // 错误处理
        return
    }
    
    // 发送初始端点事件
    _, err = fmt.Fprintf(c.Writer, "event: endpoint\ndata: %s\n\n", endpointURL)
    if err != nil {
        // 错误处理
        return
    }
    c.Writer.Flush()
    
    // 主事件循环
    for {
        select {
        case event := <-conn.EventQueue():
            // 处理不同类型事件
            switch event.Event {
            case "message":
                _, err = fmt.Fprintf(c.Writer, "event: message\ndata: %s\n\n", event.Data)
                // 错误处理
            default:
                _, err = fmt.Fprint(c.Writer, event)
                // 错误处理
            }
            c.Writer.Flush()
        case <-c.Request.Context().Done():
            // 客户端断开连接
            return
        case <-s.shutdownCh:
            // 服务关闭
            return
        }
    }
}
```

### 2. 会话管理

MCP Gateway 使用会话管理系统来跟踪 SSE 连接。每个 SSE 连接都会创建一个唯一的会话 ID，并将连接信息存储在会话存储中。

关键步骤包括：
1. 生成唯一会话 ID
2. 创建会话元数据
3. 注册会话到会话存储
4. 通过会话事件队列发送数据

### 3. 事件格式

MCP Gateway 使用标准的 SSE 事件格式：

```
event: <event_type>
data: <json_data>

```

主要事件类型包括：
- `endpoint`: 初始连接事件，提供消息端点 URL
- `message`: 消息事件，包含实际的 MCP 消息数据

### 4. 错误处理和连接管理

SSE 实现包括完善的错误处理和连接管理机制：

1. **客户端断开连接检测**：通过监听 `c.Request.Context().Done()` 事件检测客户端断开连接
2. **服务关闭处理**：通过监听 `s.shutdownCh` 事件处理服务关闭
3. **错误响应**：使用 `sendErrorResponse` 函数通过 SSE 通道发送错误信息

### 5. MCP 代理中的 SSE 传输

在 [internal/core/mcpproxy/sse.go](/internal/core/mcpproxy/sse.go) 中实现了 MCP 代理的 SSE 传输机制。这允许 MCP Gateway 通过 SSE 连接到后端 MCP 服务器。

关键组件包括：
1. [SSETransport](/internal/core/mcpproxy/sse.go#L17-L20) 结构体：实现 Transport 接口
2. [Start](/internal/core/mcpproxy/sse.go#L24-L57) 方法：初始化 SSE 连接
3. [FetchTools](/internal/core/mcpproxy/sse.go#L59-L101) 方法：获取工具列表
4. [CallTool](/internal/core/mcpproxy/sse.go#L103-L152) 方法：调用工具

```go
func (t *SSETransport) Start(ctx context.Context, tmplCtx *template.Context) error {
    // 创建 SSE 传输
    sseTransport, err := transport.NewSSE(t.cfg.URL)
    if err != nil {
        return fmt.Errorf("failed to create SSE transport: %w", err)
    }

    // 启动传输
    if err := sseTransport.Start(ctx); err != nil {
        return fmt.Errorf("failed to start SSE transport: %w", err)
    }

    // 创建客户端
    c := client.NewClient(sseTransport)

    // 初始化客户端
    initRequest := mcpgo.InitializeRequest{}
    initRequest.Params.ProtocolVersion = mcpgo.LATEST_PROTOCOL_VERSION
    initRequest.Params.ClientInfo = mcpgo.Implementation{
        Name:    cnst.AppName,
        Version: version.Get(),
    }

    _, err = c.Initialize(ctx, initRequest)
    if err != nil {
        _ = sseTransport.Close()
        return fmt.Errorf("failed to initialize SSE client: %w", err)
    }

    t.client = c
    return nil
}
```

## 数据流

1. 客户端发起 SSE 连接请求到 `/prefix/sse` 端点
2. 服务器设置 SSE 响应头并创建会话
3. 服务器发送初始 `endpoint` 事件告知客户端消息端点
4. 客户端可以向 `/prefix/message` 端点发送消息
5. 服务器通过 SSE 连接将响应发送回客户端
6. 连接保持打开状态，服务器可以继续推送事件

## 主要特点

### 1. 实时通信
- 支持服务器向客户端实时推送 MCP 消息
- 保持长连接，减少连接建立开销

### 2. 会话管理
- 每个连接都有唯一会话 ID
- 通过会话存储管理连接状态和事件队列

### 3. 错误处理
- 完善的错误处理机制
- 通过 SSE 通道发送错误信息

### 4. 生命周期管理
- 正确处理客户端断开连接
- 服务关闭时清理所有连接

## 使用场景

1. **MCP 客户端连接**：MCP 客户端通过 SSE 连接到 MCP Gateway
2. **工具调用响应**：将工具调用结果通过 SSE 推送回客户端
3. **实时通知**：向客户端推送实时状态更新

## 总结

MCP Gateway 的 SSE 实现提供了一种简单而有效的服务器到客户端实时通信机制。通过标准的 HTTP 协议和 SSE 技术，实现了与 MCP 客户端的可靠通信，同时保持了良好的错误处理和连接管理能力。SSE 传输在 MCP 代理中也得到了很好的支持，使得 MCP Gateway 可以通过 SSE 连接到后端 MCP 服务器。