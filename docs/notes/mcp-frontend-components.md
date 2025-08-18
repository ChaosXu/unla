# MCP功能对应的前端页面组件

本文档说明了后端MCP配置管理功能对应的前端页面组件，以及它们之间的映射关系。

## 主要页面组件

### 1. MCP网关管理页面 ([gateway-manager.tsx](/web/src/pages/gateway/gateway-manager.tsx))

这是MCP配置管理的主要前端页面，实现了大部分MCP配置管理功能。

#### 对应的后端功能：

1. **创建MCP服务器配置**
   - 前端组件方法：[handleCreate](/web/src/pages/gateway/gateway-manager.tsx#L342-L381)
   - 调用的API函数：[createMCPServer](/web/src/services/api.ts#L55-L65)
   - 对应的后端处理函数：[HandleMCPServerCreate](/internal/apiserver/handler/mcp.go#L368-L459)

2. **列出MCP服务器配置**
   - 前端组件方法：[fetchMCPServers](/web/src/pages/gateway/gateway-manager.tsx#L137-L152) (useEffect)
   - 调用的API函数：[getMCPServers](/web/src/services/api.ts#L27-L35)
   - 对应的后端处理函数：[HandleListMCPServers](/internal/apiserver/handler/mcp.go#L226-L366)

3. **编辑MCP服务器配置**
   - 前端组件方法：[handleEdit](/web/src/pages/gateway/gateway-manager.tsx#L117-L128)
   - 调用的API函数：[updateMCPServer](/web/src/services/api.ts#L37-L45)
   - 对应的后端处理函数：[HandleMCPServerUpdate](/internal/apiserver/handler/mcp.go#L116-L197)

4. **删除MCP服务器配置**
   - 前端组件方法：[handleDelete](/web/src/pages/gateway/gateway-manager.tsx#L308-L319) 和 [confirmDelete](/web/src/pages/gateway/gateway-manager.tsx#L321-L337)
   - 调用的API函数：[deleteMCPServer](/web/src/services/api.ts#L47-L53)
   - 对应的后端处理函数：[HandleMCPServerDelete](/internal/apiserver/handler/mcp.go#L461-L519)

5. **导出MCP服务器配置**
   - 前端组件方法：[handleExport](/web/src/pages/gateway/gateway-manager.tsx#L339-L340)
   - 调用的API函数：[exportMCPServer](/web/src/services/api.ts#L67-L81)
   - 对应的后端处理函数：无专门处理函数，前端直接处理导出逻辑

6. **同步MCP服务器配置**
   - 前端组件方法：[handleSync](/web/src/pages/gateway/gateway-manager.tsx#L354-L370)
   - 调用的API函数：[syncMCPServers](/web/src/services/api.ts#L83-L89)
   - 对应的后端处理函数：[HandleMCPServerSync](/internal/apiserver/handler/mcp.go#L521-L553)

### 2. 配置版本管理页面 ([config-versions.tsx](/web/src/pages/gateway/config-versions.tsx))

这个页面专门用于管理MCP配置的版本控制功能。

#### 对应的后端功能：

1. **获取配置版本列表**
   - 前端组件方法：[fetchVersions](/web/src/pages/gateway/config-versions.tsx#L47-L57) (useCallback)
   - 调用的API函数：[getMCPConfigVersions](/web/src/services/api.ts#L248-L257)
   - 对应的后端处理函数：[HandleGetConfigVersions](/internal/apiserver/handler/mcp.go#L556-L647)

2. **设置活动版本**
   - 前端组件方法：[handleSetActive](/web/src/pages/gateway/config-versions.tsx#L60-L67)
   - 调用的API函数：[setActiveVersion](/web/src/services/api.ts#L267-L271)
   - 对应的后端处理函数：[HandleSetActiveVersion](/internal/apiserver/handler/mcp.go#L649-L714)

3. **获取配置名称列表**
   - 前端组件方法：[fetchConfigNames](/web/src/pages/gateway/config-versions.tsx#L27-L34) (useCallback)
   - 调用的API函数：[getMCPConfigNames](/web/src/services/api.ts#L259-L265)
   - 对应的后端处理函数：[HandleGetConfigNames](/internal/apiserver/handler/mcp.go#L717-L781)

## 辅助组件

### OpenAPI导入组件 ([OpenAPIImport](/web/src/pages/gateway/components/OpenAPIImport.tsx))

该组件实现了OpenAPI导入功能：

- 前端组件方法：[handleSubmit](/web/src/pages/gateway/components/OpenAPIImport.tsx#L49-L82)
- 调用的API函数：[importOpenAPI](/web/src/services/api.ts#L121-L137)
- 对应的后端处理函数：[HandleImport](/internal/apiserver/handler/openapi.go#L33-L171)

### 配置编辑器组件 ([ConfigEditor](/web/src/pages/gateway/components/ConfigEditor.tsx))

该组件提供YAML配置编辑器功能，被[gateway-manager.tsx](/web/src/pages/gateway/gateway-manager.tsx)使用。

## 总结

前端MCP配置管理功能主要分布在两个主要页面中：

1. **[gateway-manager.tsx](/web/src/pages/gateway/gateway-manager.tsx)** - 实现了MCP配置的增删改查、导入导出和同步功能
2. **[config-versions.tsx](/web/src/pages/gateway/config-versions.tsx)** - 实现了配置版本管理功能

这些前端组件通过调用[/web/src/services/api.ts](/web/src/services/api.ts)中封装的API函数与后端进行通信，后端通过[/internal/apiserver/handler/mcp.go](/internal/apiserver/handler/mcp.go)和[/internal/apiserver/handler/openapi.go](/internal/apiserver/handler/openapi.go)中的处理函数响应请求，形成了完整的MCP配置管理功能体系。