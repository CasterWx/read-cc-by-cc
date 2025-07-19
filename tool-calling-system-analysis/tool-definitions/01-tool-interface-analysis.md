# Claude Code 工具接口定义分析

## 1. Tool接口结构逆向

基于StickerRequestTool的完整实现，逆向出Tool接口的结构：

```typescript
interface Tool {
  name: string                                    // 工具名称(用于LLM调用)
  userFacingName: () => string                   // 面向用户的显示名称
  description: (input?: any) => Promise<string>  // 工具描述(传给LLM)
  inputSchema: z.ZodSchema                       // Zod输入参数验证模式
  isEnabled: () => Promise<boolean>              // 是否启用该工具
  isReadOnly: () => boolean                      // 是否为只读工具(影响并发执行)
  needsPermissions: () => boolean                // 是否需要用户权限确认
  prompt: (options?: any) => Promise<string>     // 传给LLM的工具使用指导
  
  // 核心执行方法 - 异步生成器
  call: (
    input: any,                                  // LLM提供的工具参数
    context: ToolUseContext,                     // 执行上下文
    canUseTool?: CanUseToolFn                   // 权限检查函数
  ) => AsyncGenerator<ToolResult, void>
  
  // UI渲染方法
  renderToolUseMessage: (input: any) => React.ReactNode | string
  renderToolUseRejectedMessage: (input: any) => React.ReactNode
  renderResultForAssistant: (content: string) => string
  
  // 可选属性
  validateInput?: (input: any, context: ToolUseContext) => Promise<ValidationResult>
  inputJSONSchema?: any                          // 直接提供JSON Schema(优先级高于Zod)
}
```

## 2. 内置工具完整清单

**文件位置**: `src/tools.ts:23-39`

### 2.1 核心工具集合
```typescript
export const getAllTools = (): Tool[] => {
  return [
    AgentTool,           // AI代理工具 - 启动子Agent执行复杂任务
    BashTool,            // Bash命令执行
    GlobTool,            // 文件模式匹配搜索
    GrepTool,            // 文件内容搜索(基于ripgrep)
    LSTool,              // 目录和文件列表
    FileReadTool,        // 文件读取
    FileEditTool,        // 文件编辑(查找替换)
    FileWriteTool,       // 文件写入
    NotebookReadTool,    // Jupyter笔记本读取
    NotebookEditTool,    // Jupyter笔记本编辑
    StickerRequestTool,  // 贴纸请求工具(彩蛋功能)
    ThinkTool,           // 思考工具(内部思考过程)
    ...(process.env.USER_TYPE === 'ant' ? ANT_ONLY_TOOLS : []),
  ]
}
```

### 2.2 特殊权限工具
```typescript
const ANT_ONLY_TOOLS = [
  MemoryReadTool,      // 记忆读取(仅内部用户)
  MemoryWriteTool,     // 记忆写入(仅内部用户)
]
```

### 2.3 可选工具
```typescript
// 条件性启用的工具
if (enableArchitect) {
  tools.push(ArchitectTool)  // 架构分析工具
}

// 外部MCP工具
const mcpTools = await getMCPTools()  // 通过MCP协议集成的外部工具
```

## 3. 典型工具定义示例

### 3.1 BashTool定义结构
```typescript
export const BashTool = {
  name: 'Bash',
  
  // 动态描述生成(使用LLM)
  async description({ command }) {
    const result = await queryHaiku({
      systemPrompt: ['You are a command description generator...'],
      userPrompt: command
    })
    return result.message.content[0]?.text || 'Executes bash commands'
  },
  
  // Zod输入验证
  inputSchema: z.strictObject({
    command: z.string().describe('The command to execute'),
    timeout: z.number().optional().describe('Optional timeout in milliseconds (max 600000)'),
  }),
  
  // 权限和能力检查
  isEnabled: () => true,
  isReadOnly: () => false,
  needsPermissions: () => true,  // 需要用户确认
  
  // 工具使用指导
  prompt: async (options) => PROMPT,  // 从专门的prompt.ts文件加载
  
  // 核心执行逻辑
  async *call(input, context, canUseTool) {
    // 参数验证
    const { command, timeout } = inputSchema.parse(input)
    
    // 权限检查
    const hasPermission = await canUseTool?.(BashTool, input, context)
    if (!hasPermission?.result) {
      throw new Error('Permission denied')
    }
    
    // 执行命令
    const shell = PersistentShell.getInstance()
    const result = await shell.exec(command, timeout)
    
    // 返回结果
    yield {
      type: 'result',
      resultForAssistant: formatOutput(result),
      data: result
    }
  },
  
  // UI渲染
  renderToolUseMessage: (input) => `$ ${input.command}`,
  renderToolUseRejectedMessage: () => <Text color="red">Command rejected</Text>,
  renderResultForAssistant: (content) => content,
}
```

### 3.2 FileReadTool定义结构
```typescript
export const FileReadTool = {
  name: 'Read',
  
  inputSchema: z.strictObject({
    file_path: z.string().describe('The absolute path to the file to read'),
    limit: z.number().optional().describe('The number of lines to read'),
    offset: z.number().optional().describe('The line number to start reading from'),
  }),
  
  isEnabled: () => true,
  isReadOnly: () => true,  // 只读工具可以并发执行
  needsPermissions: () => false,
  
  async *call(input, context) {
    const { file_path, limit, offset } = inputSchema.parse(input)
    
    // 文件存在性检查
    if (!existsSync(file_path)) {
      throw new Error(`File does not exist: ${file_path}`)
    }
    
    // 读取文件
    const content = await readFile(file_path, 'utf-8')
    const lines = content.split('\n')
    
    // 处理分页
    const startLine = offset || 0
    const endLine = limit ? startLine + limit : lines.length
    const selectedLines = lines.slice(startLine, endLine)
    
    // 格式化输出
    const formattedContent = selectedLines
      .map((line, index) => `${startLine + index + 1}→${line}`)
      .join('\n')
    
    yield {
      type: 'result',
      resultForAssistant: formattedContent,
      data: { content, totalLines: lines.length }
    }
  }
}
```

## 4. 工具结果类型定义

```typescript
type ToolResult = {
  type: 'result'
  resultForAssistant: string    // 返回给LLM的文本内容
  data?: any                    // 可选的结构化数据
} | {
  type: 'progress'              // 进度更新(用于长时间运行的工具)
  content: string
  normalizedMessages?: NormalizedMessage[]
  tools?: Tool[]
}
```

## 5. 工具上下文 (ToolUseContext)

```typescript
interface ToolUseContext {
  abortController: AbortController     // 用于取消操作
  messageId?: string                   // 消息ID
  options: {
    commands: Command[]                // 可用命令
    tools: Tool[]                      // 可用工具
    slowAndCapableModel: string        // LLM模型名称
    forkNumber: number                 // 会话分支号
    messageLogName: string             // 日志文件名
    maxThinkingTokens: number          // 思考token限制
  }
  readFileTimestamps: Record<string, number>  // 文件读取时间戳
  setToolJSX?: (jsx: React.ReactNode | null) => void  // 设置工具UI
}
```

## 6. 工具发现和加载机制

```typescript
// src/tools.ts:41
export const getTools = memoize(async (enableArchitect?: boolean): Promise<Tool[]> => {
  // 1. 获取所有内置工具
  const tools = [...getAllTools(), ...(await getMCPTools())]

  // 2. 条件性添加特殊工具
  if (enableArchitect) {
    tools.push(ArchitectTool)
  }

  // 3. 检查工具是否启用
  const isEnabled = await Promise.all(tools.map(tool => tool.isEnabled()))
  
  // 4. 过滤出启用的工具
  return tools.filter((_, i) => isEnabled[i])
})
```

这种设计允许：
- **动态工具发现**: 运行时决定哪些工具可用
- **条件性启用**: 基于配置、用户类型、特性开关控制工具可用性
- **外部工具集成**: 通过MCP协议集成第三方工具
- **权限分级**: 不同工具有不同的权限要求