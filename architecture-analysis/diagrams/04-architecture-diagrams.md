# Claude Code 架构图与流程图

## 1. 系统整体架构图

```mermaid
graph TB
    subgraph "用户界面层 (UI Layer)"
        CLI[CLI入口<br/>cli.tsx:271]
        REPL[REPL交互界面<br/>REPL.tsx:88]
        UI_COMPONENTS[UI组件集合<br/>components/]
    end
    
    subgraph "业务逻辑层 (Business Layer)"
        COMMANDS[命令系统<br/>commands.ts]
        QUERY[查询引擎<br/>query.ts:124]
        MESSAGES[消息处理<br/>messages.tsx]
    end
    
    subgraph "工具执行层 (Tool Layer)"
        TOOLS[工具管理<br/>tools.ts]
        TOOL_IMPL[具体工具实现<br/>tools/]
        PERMISSIONS[权限系统<br/>permissions.ts]
    end
    
    subgraph "服务层 (Service Layer)"
        CLAUDE_API[Claude API<br/>claude.ts:402]
        MCP[MCP集成<br/>mcpClient.ts]
        CONFIG[配置管理<br/>config.ts]
    end
    
    subgraph "存储层 (Storage Layer)"
        CONTEXT[上下文存储<br/>context.ts]
        HISTORY[历史记录<br/>history.ts]
        LOGS[日志系统<br/>log.ts]
    end
    
    CLI --> REPL
    REPL --> UI_COMPONENTS
    REPL --> MESSAGES
    MESSAGES --> COMMANDS
    MESSAGES --> QUERY
    QUERY --> TOOLS
    QUERY --> CLAUDE_API
    TOOLS --> TOOL_IMPL
    TOOLS --> PERMISSIONS
    CLAUDE_API --> MCP
    QUERY --> CONTEXT
    QUERY --> HISTORY
    QUERY --> LOGS
    CONFIG --> REPL
    CONFIG --> TOOLS
```

## 2. 用户输入处理流程图

```mermaid
flowchart TD
    START([用户输入]) --> INPUT_CHECK{输入类型检查}
    
    INPUT_CHECK -->|斜杠命令| SLASH_CMD[解析斜杠命令<br/>commands.ts:104]
    INPUT_CHECK -->|Bash命令| BASH_CMD[执行Bash命令<br/>BashTool]
    INPUT_CHECK -->|常规输入| USER_MSG[创建用户消息<br/>createUserMessage]
    
    SLASH_CMD --> CMD_TYPE{命令类型}
    CMD_TYPE -->|prompt| LLM_PROMPT[转LLM提示词]
    CMD_TYPE -->|local| LOCAL_EXEC[本地执行]
    CMD_TYPE -->|local-jsx| JSX_RENDER[渲染JSX组件]
    
    BASH_CMD --> BASH_RESULT[Bash执行结果]
    USER_MSG --> QUERY_ENGINE[查询引擎<br/>query.ts:124]
    LLM_PROMPT --> QUERY_ENGINE
    
    QUERY_ENGINE --> SYSTEM_PROMPT[格式化系统提示词]
    SYSTEM_PROMPT --> API_CALL[调用Claude API<br/>claude.ts:513]
    
    API_CALL --> RESPONSE[LLM响应]
    RESPONSE --> TOOL_CHECK{是否需要工具调用?}
    
    TOOL_CHECK -->|否| OUTPUT[输出结果]
    TOOL_CHECK -->|是| TOOL_EXTRACT[提取工具调用信息]
    
    TOOL_EXTRACT --> PERMISSION_CHECK[权限检查<br/>permissions.ts]
    PERMISSION_CHECK -->|拒绝| ERROR_MSG[权限错误消息]
    PERMISSION_CHECK -->|允许| TOOL_EXEC[执行工具<br/>runToolUse]
    
    TOOL_EXEC --> TOOL_RESULT[工具执行结果]
    TOOL_RESULT --> RECURSIVE_QUERY[递归查询引擎]
    RECURSIVE_QUERY --> OUTPUT
    
    LOCAL_EXEC --> OUTPUT
    JSX_RENDER --> OUTPUT
    BASH_RESULT --> OUTPUT
    ERROR_MSG --> OUTPUT
    
    OUTPUT --> END([渲染到UI])
```

## 3. 工具执行流程图

```mermaid
sequenceDiagram
    participant User as 用户
    participant REPL as REPL界面
    participant Query as 查询引擎
    participant API as Claude API
    participant Tool as 工具系统
    participant Perm as 权限系统
    
    User->>REPL: 输入请求
    REPL->>Query: 处理用户消息
    Query->>API: 发送消息和工具定义
    API-->>Query: 返回包含工具调用的响应
    
    Query->>Tool: 解析工具调用请求
    Tool->>Tool: 验证输入参数
    Tool->>Perm: 检查工具使用权限
    
    alt 权限被拒绝
        Perm-->>Query: 返回权限错误
        Query-->>REPL: 显示错误消息
    else 权限通过
        Perm-->>Tool: 权限通过
        Tool->>Tool: 执行具体工具逻辑
        Tool-->>Query: 返回工具执行结果
        Query->>API: 将结果发送给LLM
        API-->>Query: 返回最终响应
        Query-->>REPL: 返回完整对话结果
    end
    
    REPL-->>User: 显示最终结果
```

## 4. 组件协同关系图

```mermaid
graph LR
    subgraph "React组件层"
        REPL_COMP[REPL组件]
        PROMPT_INPUT[PromptInput]
        MESSAGE_COMP[Message组件]
        PERMISSION_REQ[PermissionRequest]
    end
    
    subgraph "业务逻辑层"
        QUERY_ENGINE[Query引擎]
        CMD_SYSTEM[命令系统]
        TOOL_SYSTEM[工具系统]
    end
    
    subgraph "数据服务层"
        CONFIG_SERVICE[配置服务]
        CONTEXT_SERVICE[上下文服务]
        LOG_SERVICE[日志服务]
    end
    
    REPL_COMP -.->|props传递| PROMPT_INPUT
    REPL_COMP -.->|props传递| MESSAGE_COMP
    REPL_COMP -.->|state控制| PERMISSION_REQ
    
    PROMPT_INPUT -->|onQuery回调| REPL_COMP
    PERMISSION_REQ -->|onDone回调| REPL_COMP
    
    REPL_COMP -->|调用| QUERY_ENGINE
    QUERY_ENGINE -->|调用| CMD_SYSTEM
    QUERY_ENGINE -->|调用| TOOL_SYSTEM
    
    REPL_COMP -->|读取| CONFIG_SERVICE
    QUERY_ENGINE -->|读写| CONTEXT_SERVICE
    QUERY_ENGINE -->|写入| LOG_SERVICE
    
    CONFIG_SERVICE -.->|配置变化| REPL_COMP
    CONTEXT_SERVICE -.->|上下文变化| QUERY_ENGINE
```

## 5. 数据流向图

```mermaid
flowchart LR
    subgraph "输入源"
        USER_INPUT[用户输入]
        CONFIG_FILE[配置文件]
        CONTEXT_DATA[上下文数据]
    end
    
    subgraph "处理管道"
        PARSE[解析层]
        VALIDATE[验证层]
        EXECUTE[执行层]
        FORMAT[格式化层]
    end
    
    subgraph "输出目标"
        UI_RENDER[UI渲染]
        LOG_FILE[日志文件]
        CONFIG_UPDATE[配置更新]
    end
    
    USER_INPUT --> PARSE
    CONFIG_FILE --> VALIDATE
    CONTEXT_DATA --> EXECUTE
    
    PARSE --> VALIDATE
    VALIDATE --> EXECUTE
    EXECUTE --> FORMAT
    
    FORMAT --> UI_RENDER
    FORMAT --> LOG_FILE
    EXECUTE --> CONFIG_UPDATE
    
    CONFIG_UPDATE -.->|反馈| CONFIG_FILE
    LOG_FILE -.->|历史| CONTEXT_DATA
```

## 6. 异步处理时序图

```mermaid
sequenceDiagram
    participant UI as UI线程
    participant Query as Query引擎
    participant Tool1 as 工具1
    participant Tool2 as 工具2
    participant API as Claude API
    
    UI->>Query: 启动查询
    Query->>API: 发送请求
    API-->>Query: 返回包含多个工具调用
    
    par 并发工具执行
        Query->>Tool1: 异步执行工具1
        and
        Query->>Tool2: 异步执行工具2
    end
    
    Tool1-->>Query: 返回结果1
    UI<<--Query: 实时更新UI (progress)
    
    Tool2-->>Query: 返回结果2  
    UI<<--Query: 实时更新UI (progress)
    
    Query->>API: 发送所有工具结果
    API-->>Query: 返回最终响应
    Query-->>UI: 完成查询
```

## 7. 错误处理流程图

```mermaid
flowchart TD
    ERROR_OCCUR[错误发生] --> ERROR_TYPE{错误类型}
    
    ERROR_TYPE -->|API错误| API_ERROR[API错误处理<br/>claude.ts:543]
    ERROR_TYPE -->|工具错误| TOOL_ERROR[工具错误处理<br/>query.ts:477]
    ERROR_TYPE -->|权限错误| PERM_ERROR[权限错误处理<br/>permissions.ts]
    ERROR_TYPE -->|配置错误| CONFIG_ERROR[配置错误处理<br/>config.ts]
    
    API_ERROR --> RETRY_CHECK{是否可重试?}
    RETRY_CHECK -->|是| RETRY_API[API重试机制<br/>withRetry]
    RETRY_CHECK -->|否| ERROR_MSG[生成错误消息]
    
    TOOL_ERROR --> ERROR_MSG
    PERM_ERROR --> ERROR_MSG
    CONFIG_ERROR --> ERROR_MSG
    
    RETRY_API --> SUCCESS{重试成功?}
    SUCCESS -->|是| CONTINUE[继续处理]
    SUCCESS -->|否| ERROR_MSG
    
    ERROR_MSG --> LOG_ERROR[记录错误日志<br/>log.ts]
    LOG_ERROR --> DISPLAY_ERROR[显示用户友好错误]
    
    DISPLAY_ERROR --> END[流程结束]
    CONTINUE --> END
```

## 图例说明

- **实线箭头** (→): 直接调用关系
- **虚线箭头** (-.->): 数据流或配置传递
- **双向箭头** (↔): 双向通信
- **并行块** (par): 并发执行
- **判断菱形** ({}): 条件分支
- **圆角矩形** ([]): 处理步骤
- **圆形** (()): 开始/结束节点

这些图表展示了Claude Code系统的完整架构设计，包括组件关系、数据流向、异步处理和错误处理等关键方面。