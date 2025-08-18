# MCP Handler 分析

本文档分析了 [/internal/apiserver/handler/mcp.go](/internal/apiserver/handler/mcp.go) 文件，该文件是处理 MCP 配置相关请求的核心组件。

## 结构和依赖

该文件定义了一个 [MCP](/internal/apiserver/handler/mcp.go#L26-L31) 结构体，包含以下依赖：
- `db`: 数据库接口，用于访问用户和租户信息
- `store`: 存储接口，用于管理 MCP 配置
- `notifier`: 通知器，用于通知 MCP 网关配置变更
- `logger`: 日志记录器

## 主要功能

### 1. 权限检查
[checkTenantPermission](/internal/apiserver/handler/mcp.go#L34-L114) 方法负责检查用户对特定租户的访问权限：
- 验证租户名称是否为空
- 获取用户身份信息
- 获取租户信息并验证用户权限（管理员有全部访问权限）
- 验证路由前缀是否符合租户路径要求

### 2. MCP 配置管理
- [HandleMCPServerCreate](/internal/apiserver/handler/mcp.go#L368-L459): 创建 MCP 服务器配置
  - 读取并验证 YAML 格式的配置内容
  - 检查配置名称是否已存在
  - 验证租户权限
  - 验证配置有效性
  - 保存配置到存储
  - 通知 MCP 网关更新配置

- [HandleListMCPServers](/internal/apiserver/handler/mcp.go#L226-L366): 列出 MCP 服务器配置
  - 根据租户 ID 过滤配置
  - 根据用户角色（管理员/普通用户）应用不同的访问权限
  - 将配置转换为 DTO 格式返回

- [HandleMCPServerUpdate](/internal/apiserver/handler/mcp.go#L116-L197): 更新 MCP 服务器配置
  - 读取并验证 YAML 格式的配置内容
  - 检查配置是否存在
  - 验证租户权限
  - 验证配置有效性
  - 更新存储中的配置
  - 通知 MCP 网关更新配置

- [HandleMCPServerDelete](/internal/apiserver/handler/mcp.go#L461-L519): 删除 MCP 服务器配置
  - 检查配置是否存在
  - 验证租户权限
  - 从存储中删除配置
  - 通知 MCP 网关更新配置

- [HandleMCPServerSync](/internal/apiserver/handler/mcp.go#L521-L553): 同步所有 MCP 服务器配置
  - 仅限管理员调用
  - 通知 MCP 网关重新加载所有配置

### 3. 版本管理
- [HandleGetConfigVersions](/internal/apiserver/handler/mcp.go#L556-L647): 获取配置版本列表
  - 根据名称和租户过滤配置
  - 根据用户权限过滤可访问的配置
  - 获取每个配置的版本信息

- [HandleSetActiveVersion](/internal/apiserver/handler/mcp.go#L649-L714): 设置活动版本
  - 验证租户权限
  - 设置指定版本为活动版本
  - 通知 MCP 网关更新配置

- [HandleGetConfigNames](/internal/apiserver/handler/mcp.go#L717-L781): 获取所有配置名称
  - 根据用户权限过滤可访问的配置
  - 返回配置名称列表

## 安全和验证机制

1. **JWT 认证**: 所有端点都通过中间件进行 JWT 认证
2. **租户权限控制**: 通过 [checkTenantPermission](/internal/apiserver/handler/mcp.go#L34-L114) 方法实现租户级别的访问控制
3. **配置验证**: 使用 YAML 解析和配置验证确保配置格式正确
4. **输入验证**: 对所有用户输入进行验证，防止无效数据

## 通知机制

每当配置发生变化（创建、更新、删除、设置版本），都会通过 notifier 通知 MCP 网关，确保配置变更能及时生效。

## 数据转换

配置在存储和传输之间进行转换，使用 DTO（数据传输对象）模式，将内部配置结构转换为适合前端使用的格式。

## 总结

MCP Handler 是 API Server 中负责管理 MCP 配置的核心组件，它提供了完整的配置生命周期管理功能，包括创建、读取、更新、删除和版本控制。通过严格的权限控制和验证机制，确保配置管理的安全性。通过通知机制，与 MCP 网关保持实时同步。