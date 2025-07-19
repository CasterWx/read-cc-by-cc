# Claude Code CLI 架构分层图

## 1. 整体架构分层图

```mermaid
graph TB
    subgraph "表示层 (Presentation Layer)"
        A1[REPL.tsx<br/>主交互界面]
        A2[PromptInput.tsx<br/>用户输入]
        A3[Message.tsx<br/>消息渲染]
        A4[Logo.tsx<br/>状态显示]
    end
    
    subgraph "业务逻辑层 (Business Logic Layer)"
        B1[query.ts<br/>查询引擎]
        B2[commands/<br/>命令系统]
        B3[context.ts<br/>上下文管理]
        B4[tools.ts<br/>工具注册]
    end
    
    subgraph "服务层 (Service Layer)"
        C1[claude.ts<br/>LLM API服务]
        C2[mcpClient.ts<br/>MCP协议]
        C3[auth.ts<br/>认证服务]
    end
    
    subgraph "工具层 (Tool Layer)"
        D1[FileTools<br/>文件操作]
        D2[BashTool<br/>系统命令]
        D3[SearchTools<br/>搜索工具]
        D4[AgentTools<br/>智能工具]
    end
    
    subgraph "数据访问层 (Data Access Layer)"
        E1[config.ts<br/>配置管理]
        E2[file.ts<br/>文件系统]
        E3[git.ts<br/>Git集成]
        E4[state.ts<br/>状态管理]
    end
    
    subgraph "基础设施层 (Infrastructure Layer)"
        F1[Ink Framework<br/>终端UI]
        F2[TypeScript<br/>类型系统]
        F3[Zod<br/>数据验证]
        F4[AbortController<br/>并发控制]
    end
    
    A1 --> B1
    A2 --> B1
    A3 --> B1
    B1 --> C1
    B1 --> D1
    B1 --> D2
    B1 --> D3
    B1 --> D4
    B2 --> B1
    B3 --> E1
    B3 --> E2
    B3 --> E3
    C1 --> C3
    C2 --> D4
    D1 --> E2
    D2 --> E2
    B1 --> F4
    A1 --> F1
    B1 --> F3
```

## 2. 核心数据流架构图

```mermaid
graph LR
    subgraph "用户交互流程"
        U1[用户输入] --> U2[PromptInput.tsx]
        U2 --> U3[processUserInput]
        U3 --> U4[Message创建]
    end
    
    subgraph "上下文增强"
        C1[getContext] --> C2[Git状态]
        C1 --> C3[目录结构]
        C1 --> C4[项目文档]
        C1 --> C5[用户配置]
    end
    
    subgraph "LLM查询引擎"
        Q1[query函数] --> Q2[消息标准化]
        Q2 --> Q3[上下文注入]
        Q3 --> Q4[LLM API调用]
        Q4 --> Q5[流式响应处理]
    end
    
    subgraph "工具调用系统"
        T1[工具选择] --> T2[参数验证]
        T2 --> T3[权限检查]
        T3 --> T4[工具执行]
        T4 --> T5[结果处理]
    end
    
    subgraph "响应渲染"
        R1[Message.tsx] --> R2[静态内容]
        R1 --> R3[动态内容]
        R2 --> R4[终端显示]
        R3 --> R4
    end
    
    U4 --> C1
    C5 --> Q1
    Q5 --> T1
    Q5 --> R1
    T5 --> Q1
```

## 3. 工具调用系统架构图

```mermaid
graph TB
    subgraph "工具调用协调器"
        TC1[runToolUse函数]
        TC2[工具并发控制]
        TC3[异步生成器]
    end
    
    subgraph "验证层次"
        V1[类型验证<br/>Zod Schema]
        V2[业务验证<br/>工具特定逻辑]
        V3[权限验证<br/>用户授权]
    end
    
    subgraph "工具分类"
        subgraph "只读工具 (并发执行)"
            RO1[FileReadTool]
            RO2[GrepTool]
            RO3[LSTool]
        end
        
        subgraph "写入工具 (串行执行)"
            WR1[FileEditTool]
            WR2[BashTool]
            WR3[FileWriteTool]
        end
        
        subgraph "智能工具"
            AI1[AgentTool]
            AI2[ThinkTool]
            AI3[ArchitectTool]
        end
    end
    
    subgraph "执行结果处理"
        ER1[结果类型判断]
        ER2[progress消息]
        ER3[result消息]
        ER4[error处理]
    end
    
    TC1 --> V1
    V1 --> V2
    V2 --> V3
    V3 --> TC2
    TC2 --> RO1
    TC2 --> RO2
    TC2 --> RO3
    TC2 --> WR1
    TC2 --> WR2
    TC2 --> WR3
    TC2 --> AI1
    TC2 --> AI2
    TC2 --> AI3
    RO1 --> ER1
    WR1 --> ER1
    AI1 --> ER1
    ER1 --> ER2
    ER1 --> ER3
    ER1 --> ER4
    ER2 --> TC3
    ER3 --> TC3
    ER4 --> TC3
```

## 4. 配置管理系统架构图

```mermaid
graph TB
    subgraph "配置层次结构"
        G1[全局配置<br/>~/.claude/config.json]
        P1[项目配置<br/>.claude/config.json]
        R1[运行时配置<br/>命令行参数]
    end
    
    subgraph "配置内容"
        C1[工具权限配置]
        C2[API密钥配置]
        C3[项目上下文配置]
        C4[MCP服务器配置]
        C5[用户偏好配置]
    end
    
    subgraph "配置加载优先级"
        L1[命令行参数 (最高)]
        L2[项目配置]
        L3[全局配置 (最低)]
    end
    
    subgraph "配置验证"
        V1[JSON Schema验证]
        V2[必需字段检查]
        V3[类型安全验证]
    end
    
    G1 --> C1
    G1 --> C2
    G1 --> C5
    P1 --> C1
    P1 --> C3
    P1 --> C4
    R1 --> C1
    R1 --> C2
    
    L1 --> V1
    L2 --> V1
    L3 --> V1
    V1 --> V2
    V2 --> V3
```

## 5. 上下文感知系统架构图

```mermaid
graph TB
    subgraph "上下文收集器"
        CC1[getContext函数]
        CC2[并行收集策略]
        CC3[缓存机制]
    end
    
    subgraph "Git上下文"
        G1[当前分支状态]
        G2[工作区变更]
        G3[提交历史]
        G4[用户提交记录]
    end
    
    subgraph "项目上下文"
        P1[目录结构快照]
        P2[README.md内容]
        P3[CLAUDE.md指令]
        P4[代码风格推断]
    end
    
    subgraph "用户行为分析"
        U1[Git提交历史分析]
        U2[常用文件识别]
        U3[个性化建议生成]
        U4[AI智能筛选]
    end
    
    subgraph "上下文注入"
        I1[系统提示词构建]
        I2[环境信息注入]
        I3[项目信息注入]
        I4[用户偏好注入]
    end
    
    CC1 --> CC2
    CC2 --> G1
    CC2 --> G2
    CC2 --> G3
    CC2 --> G4
    CC2 --> P1
    CC2 --> P2
    CC2 --> P3
    CC2 --> P4
    G4 --> U1
    U1 --> U2
    U2 --> U3
    U3 --> U4
    CC3 --> I1
    G1 --> I2
    P1 --> I3
    U3 --> I4
```

## 6. 异步生成器流式架构图

```mermaid
graph LR
    subgraph "异步生成器模式"
        AG1[async function*]
        AG2[yield消息]
        AG3[流式处理]
    end
    
    subgraph "消息类型"
        M1[AssistantMessage<br/>LLM响应]
        M2[ToolUseMessage<br/>工具调用]
        M3[ProgressMessage<br/>进度反馈]
        M4[UserMessage<br/>工具结果]
    end
    
    subgraph "流式渲染"
        R1[React状态更新]
        R2[静态内容缓存]
        R3[动态内容实时]
        R4[终端增量渲染]
    end
    
    subgraph "并发控制"
        CC1[AbortController]
        CC2[只读工具并发]
        CC3[写入工具串行]
        CC4[超时处理]
    end
    
    AG1 --> AG2
    AG2 --> M1
    AG2 --> M2
    AG2 --> M3
    AG2 --> M4
    AG3 --> R1
    R1 --> R2
    R1 --> R3
    R2 --> R4
    R3 --> R4
    CC1 --> AG3
    CC2 --> AG3
    CC3 --> AG3
    CC4 --> AG3
```

## 7. MCP集成架构图

```mermaid
graph TB
    subgraph "MCP协议层"
        MCP1[Model Context Protocol]
        MCP2[JSON-RPC通信]
        MCP3[工具发现机制]
    end
    
    subgraph "MCP客户端"
        MC1[mcpClient.ts]
        MC2[服务器连接管理]
        MC3[工具代理]
    end
    
    subgraph "外部MCP服务器"
        MS1[文件系统服务器]
        MS2[数据库服务器]
        MS3[API集成服务器]
        MS4[自定义工具服务器]
    end
    
    subgraph "工具集成"
        TI1[工具注册]
        TI2[权限控制]
        TI3[调用代理]
        TI4[结果处理]
    end
    
    MCP1 --> MC1
    MCP2 --> MC2
    MCP3 --> MC3
    MC2 --> MS1
    MC2 --> MS2
    MC2 --> MS3
    MC2 --> MS4
    MC3 --> TI1
    TI1 --> TI2
    TI2 --> TI3
    TI3 --> TI4
```

## 8. 安全和权限架构图

```mermaid
graph TB
    subgraph "权限控制层"
        PC1[canUseTool函数]
        PC2[工具权限配置]
        PC3[用户确认机制]
    end
    
    subgraph "安全验证"
        SV1[输入数据验证]
        SV2[路径安全检查]
        SV3[命令白名单]
        SV4[敏感操作检测]
    end
    
    subgraph "权限级别"
        PL1[只读权限<br/>文件读取、搜索]
        PL2[写入权限<br/>文件修改]
        PL3[执行权限<br/>命令执行]
        PL4[系统权限<br/>系统配置]
    end
    
    subgraph "安全措施"
        SM1[沙箱执行]
        SM2[操作日志]
        SM3[回滚机制]
        SM4[异常监控]
    end
    
    PC1 --> SV1
    PC2 --> SV2
    PC3 --> SV3
    SV4 --> PL1
    PL1 --> PL2
    PL2 --> PL3
    PL3 --> PL4
    PL4 --> SM1
    SM1 --> SM2
    SM2 --> SM3
    SM3 --> SM4
```

这些架构图从不同维度展现了Claude Code CLI的设计理念：

1. **分层清晰**: 从表示层到基础设施层的清晰分离
2. **流式处理**: 异步生成器实现的流式用户体验  
3. **工具系统**: 插件化的工具架构和权限控制
4. **上下文感知**: 智能的项目和用户行为分析
5. **安全可靠**: 多层次的安全验证和权限控制
6. **可扩展性**: MCP协议支持的第三方工具集成

整体架构体现了现代CLI工具的最佳实践，在功能完整性和代码可维护性之间取得了良好平衡。