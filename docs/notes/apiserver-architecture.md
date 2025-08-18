# API Server 架构分析

本文档分析了 [unla](/README.md) 项目中 API Server 的主要逻辑和架构。API Server 是整个系统的核心组件之一，负责提供 REST API 接口，处理用户认证、配置管理、聊天记录存储等功能。

## 入口点

API Server 的入口点位于 [/cmd/apiserver/main.go](/cmd/apiserver/main.go)。该文件使用 [Cobra](/pkg/helper/config.go#L25) 命令行框架来管理命令，支持 `version` 和 `run` 两个子命令。

## 主要组件和初始化流程

### 1. 初始化流程

API Server 的启动流程包括以下步骤：

1. **配置加载** - 通过 [initConfig()](/cmd/apiserver/main.go#L57-L63) 函数加载配置文件
2. **日志初始化** - 通过 [initLogger()](/cmd/apiserver/main.go#L48-L55) 函数初始化日志系统
3. **国际化支持** - 通过 [initI18n()](/cmd/apiserver/main.go#L301-L312) 函数初始化国际化翻译
4. **通知服务** - 通过 [initNotifier()](/cmd/apiserver/main.go#L74-L84) 函数初始化配置变更通知服务
5. **数据库连接** - 通过 [initDatabase()](/cmd/apiserver/main.go#L86-L96) 函数初始化数据库连接
6. **超级管理员** - 通过 [initSuperAdmin()](/cmd/apiserver/main.go#L115-L139) 函数创建超级管理员账户
7. **存储服务** - 通过 [initStore()](/cmd/apiserver/main.go#L105-L113) 函数初始化配置存储服务
8. **路由初始化** - 通过 [initRouter()](/cmd/apiserver/main.go#L149-L274) 函数初始化 HTTP 路由和处理函数
9. **服务启动** - 通过 [startServer()](/cmd/apiserver/main.go#L286-L296) 函数启动 HTTP 服务

### 2. 核心服务组件

#### 认证服务
- **JWT 服务** - 用于用户身份验证和授权
- **OAuth 服务** - 支持第三方登录（Google、GitHub）

#### 数据库服务
- 使用 [database.Database](/internal/apiserver/database/database.go#L26-L32) 接口管理用户、租户、聊天会话等数据
- 支持多种数据库后端（SQLite、MySQL、PostgreSQL）

#### 存储服务
- 使用 [storage.Store](/internal/mcp/storage/interface.go#L23-L30) 接口管理 MCP 配置存储
- 支持多种存储后端（文件系统、数据库等）

#### 通知服务
- 使用 [notifier.Notifier](/internal/mcp/storage/notifier/interface.go#L24-L28) 接口在配置变更时通知 MCP Gateway

## API 路由结构

API Server 提供了丰富的 REST API 接口，按照功能划分为以下几个主要模块：

### 认证相关路由 (`/api/auth`)
- **登录** - `POST /api/auth/login`
- **OAuth 登录** - `/api/auth/oauth/*`
- **用户信息** - `GET /api/auth/user/info`
- **用户管理** - `/api/auth/users/*` (管理员专用)
- **租户管理** - `/api/auth/tenants/*` (管理员专用)
- **修改密码** - `POST /api/auth/change-password`

### MCP 配置管理路由 (`/api/mcp`)
- **获取配置列表** - `GET /api/mcp/configs`
- **创建配置** - `POST /api/mcp/configs`
- **更新配置** - `PUT /api/mcp/configs`
- **删除配置** - `DELETE /api/mcp/configs/:tenant/:name`
- **同步配置** - `POST /api/mcp/configs/sync`
- **配置版本管理** - `/api/mcp/configs/versions/*`

### 聊天相关路由 (`/api/chat`)
- **会话管理** - `/api/chat/sessions/*`
- **消息管理** - `POST /api/chat/messages`
- **系统提示词** - `/api/chat/systemprompt`

### OpenAPI 路由 (`/api/openapi`)
- **导入 OpenAPI** - `POST /api/openapi/import`

### 运行时配置路由
- **前端运行时配置** - `GET /api/runtime-config`

## 核心处理模块

API Server 的业务逻辑主要分布在 [/internal/apiserver/handler](/internal/apiserver/handler) 目录下的各个处理模块中：

### 1. 认证处理模块 ([auth.go](/internal/apiserver/handler/auth.go))
处理用户登录、用户管理、租户管理、密码修改等功能。

### 2. 聊天处理模块 ([chat.go](/internal/apiserver/handler/chat.go))
处理聊天会话和消息的增删改查操作。

### 3. MCP 配置处理模块 ([mcp.go](/internal/apiserver/handler/mcp.go))
处理 MCP 配置的增删改查、版本管理、同步等功能。

### 4. OpenAPI 处理模块 ([openapi.go](/internal/apiserver/handler/openapi.go))
处理 OpenAPI 规范的导入和转换。

### 5. OAuth 处理模块 ([oauth.go](/internal/apiserver/handler/oauth.go))
处理第三方登录（Google、GitHub）相关逻辑。

### 6. 系统提示词处理模块 ([systemprompt.go](/internal/apiserver/handler/systemprompt.go))
处理系统提示词的读取和保存。

## 中间件

API Server 使用了以下中间件：

### 1. JWT 认证中间件 ([middleware/auth.go](/internal/apiserver/middleware/auth.go))
用于验证用户身份和权限。

### 2. 管理员权限中间件 ([handler/auth.go](/internal/apiserver/handler/auth.go))
用于限制某些路由只能由管理员访问。

## 数据流

1. **用户请求** - 用户通过前端或 API 调用发送请求
2. **路由匹配** - Gin 框架根据 URL 路径匹配对应的处理函数
3. **中间件处理** - 经过认证等中间件处理
4. **业务逻辑处理** - 调用相应的 handler 处理业务逻辑
5. **数据访问** - 通过 database 和 storage 接口访问数据
6. **结果返回** - 将处理结果返回给客户端
7. **通知机制** - 对于配置变更，通过 notifier 通知 MCP Gateway

## 总结

API Server 是 unla 系统的核心管理组件，负责提供 REST API 接口，处理用户认证、配置管理、聊天记录存储等功能。它通过模块化的设计，将不同的业务逻辑分离到不同的 handler 中，并通过中间件机制实现认证和权限控制。同时，它还通过 notifier 机制与 MCP Gateway 保持配置同步，确保系统的实时性。