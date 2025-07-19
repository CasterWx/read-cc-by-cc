# Claude Code 工具执行流程深度分析

## 1. 工具执行总流程概览

**核心函数**: `src/query.ts:285` - `runToolUse()`

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,              // LLM生成的工具调用信息
  siblingToolUseIDs: Set<string>,     // 同时执行的其他工具ID
  assistantMessage: AssistantMessage, // LLM的完整响应消息
  canUseTool: CanUseToolFn,          // 权限检查函数
  toolUseContext: ToolUseContext,     // 执行上下文
  shouldSkipPermissionCheck?: boolean, // 是否跳过权限检查
): AsyncGenerator<Message, void>
```

## 2. 执行流程详细步骤

### 步骤1: 工具查找和验证
```typescript
// src/query.ts:293-313
const toolName = toolUse.name
const tool = toolUseContext.options.tools.find(t => t.name === toolName)

// 🎯 关键检查点：工具是否存在
if (!tool) {
  logEvent('tengu_tool_use_error', {
    error: `No such tool available: ${toolName}`,
    messageID: assistantMessage.message.id,
    toolName,
    toolUseID: toolUse.id,
  })
  
  // 返回错误消息给LLM
  yield createUserMessage([{
    type: 'tool_result',
    content: `Error: No such tool available: ${toolName}`,
    is_error: true,
    tool_use_id: toolUse.id,
  }])
  return
}
```

### 步骤2: 中断检查
```typescript
// src/query.ts:318-328
if (toolUseContext.abortController.signal.aborted) {
  logEvent('tengu_tool_use_cancelled', {
    toolName: tool.name,
    toolUseID: toolUse.id,
  })
  
  const message = createUserMessage([
    createToolResultStopMessage(toolUse.id),
  ])
  yield message
  return
}
```

### 步骤3: 权限和参数验证流程
**关键函数**: `src/query.ts:365` - `checkPermissionsAndCallTool()`

#### 3.1 Zod Schema输入验证
```typescript
// src/query.ts:375-394
// 🎯 第一层验证：Zod Schema类型检查
const isValidInput = tool.inputSchema.safeParse(input)
if (!isValidInput.success) {
  logEvent('tengu_tool_use_error', {
    error: `InputValidationError: ${isValidInput.error.message}`,
    messageID: assistantMessage.message.id,
    toolName: tool.name,
    toolInput: JSON.stringify(input).slice(0, 200),
  })
  
  yield createUserMessage([{
    type: 'tool_result',
    content: `InputValidationError: ${isValidInput.error.message}`,
    is_error: true,
    tool_use_id: toolUseID,
  }])
  return
}
```

#### 3.2 输入规范化
```typescript
// src/query.ts:396
const normalizedInput = normalizeToolInput(tool, input)

// 规范化示例 - BashTool特殊处理
export function normalizeToolInput(
  tool: Tool,
  input: { [key: string]: boolean | string | number },
): { [key: string]: boolean | string | number } {
  switch (tool) {
    case BashTool: {
      const { command, timeout } = BashTool.inputSchema.parse(input)
      return {
        command: command.replace(`cd ${getCwd()} && `, ''),  // 移除冗余的cd命令
        ...(timeout ? { timeout } : {}),
      }
    }
    default:
      return input
  }
}
```

#### 3.3 工具自定义验证
```typescript
// src/query.ts:398-420
// 🎯 第二层验证：工具自定义业务逻辑验证
const isValidCall = await tool.validateInput?.(
  normalizedInput as never,
  context,
)

if (isValidCall?.result === false) {
  logEvent('tengu_tool_use_error', {
    error: isValidCall?.message.slice(0, 2000),
    messageID: assistantMessage.message.id,
    toolName: tool.name,
    toolInput: JSON.stringify(input).slice(0, 200),
    ...(isValidCall?.meta ?? {}),
  })
  
  yield createUserMessage([{
    type: 'tool_result',
    content: isValidCall!.message,
    is_error: true,
    tool_use_id: toolUseID,
  }])
  return
}
```

#### 3.4 权限检查
```typescript
// src/query.ts:422-437
// 🎯 第三层验证：用户权限确认
const permissionResult = shouldSkipPermissionCheck
  ? ({ result: true } as const)
  : await canUseTool(tool, normalizedInput, context, assistantMessage)

if (permissionResult.result === false) {
  yield createUserMessage([{
    type: 'tool_result',
    content: permissionResult.message,
    is_error: true,
    tool_use_id: toolUseID,
  }])
  return
}
```

### 步骤4: 工具实际执行
```typescript
// src/query.ts:440-476
try {
  // 🎯 核心执行：调用工具的call方法
  const generator = tool.call(normalizedInput as never, context, canUseTool)
  
  for await (const result of generator) {
    switch (result.type) {
      case 'result':
        // 🎯 成功结果处理
        logEvent('tengu_tool_use_success', {
          messageID: assistantMessage.message.id,
          toolName: tool.name,
        })
        
        yield createUserMessage(
          [{
            type: 'tool_result',
            content: result.resultForAssistant,  // 🎯 返回给LLM的内容
            tool_use_id: toolUseID,
          }],
          {
            data: result.data,                   // 🎯 结构化数据(可选)
            resultForAssistant: result.resultForAssistant,
          },
        )
        return
        
      case 'progress':
        // 🎯 进度更新处理
        logEvent('tengu_tool_use_progress', {
          messageID: assistantMessage.message.id,
          toolName: tool.name,
        })
        
        yield createProgressMessage(
          toolUseID,
          siblingToolUseIDs,
          result.content,
          result.normalizedMessages,
          result.tools,
        )
    }
  }
} catch (error) {
  // 🎯 异常处理(详见错误处理章节)
}
```

## 3. 具体工具执行示例

### 3.1 FileReadTool执行流程
```typescript
// 工具调用输入
const toolUse: ToolUseBlock = {
  type: 'tool_use',
  id: 'toolu_123',
  name: 'Read',
  input: {
    file_path: '/Users/example/project/README.md',
    limit: 50,
    offset: 0
  }
}

// 执行步骤：
// 1. 工具查找 ✓ (找到FileReadTool)
// 2. Zod验证 ✓ (file_path: string, limit?: number, offset?: number)
// 3. 自定义验证 ✓ (文件是否存在，路径是否安全)
// 4. 权限检查 ✓ (FileReadTool.needsPermissions() === false，跳过)
// 5. 执行 FileReadTool.call()

async *call(input, context) {
  const { file_path, limit, offset } = this.inputSchema.parse(input)
  
  // 文件读取逻辑
  const content = await readFile(file_path, 'utf-8')
  const lines = content.split('\n')
  const selectedLines = lines.slice(offset || 0, (offset || 0) + (limit || lines.length))
  
  // 格式化返回
  const formattedContent = selectedLines
    .map((line, index) => `${(offset || 0) + index + 1}→${line}`)
    .join('\n')
  
  yield {
    type: 'result',
    resultForAssistant: formattedContent,
    data: { content, totalLines: lines.length }
  }
}
```

### 3.2 BashTool执行流程(需要权限)
```typescript
// 工具调用输入
const toolUse: ToolUseBlock = {
  type: 'tool_use',
  id: 'toolu_456',
  name: 'Bash',
  input: {
    command: 'ls -la',
    timeout: 30000
  }
}

// 执行步骤：
// 1. 工具查找 ✓
// 2. Zod验证 ✓
// 3. 输入规范化 ✓ (移除多余的cd命令)
// 4. 自定义验证 ✓ (检查是否为危险命令)
// 5. 权限检查 🚨 (BashTool.needsPermissions() === true)
//    → 显示权限确认界面
//    → 用户确认或拒绝
// 6. 执行 BashTool.call()

async *call(input, context) {
  const { command, timeout } = this.inputSchema.parse(input)
  
  // 使用持久化Shell执行
  const shell = PersistentShell.getInstance()
  const result = await shell.exec(command, timeout)
  
  yield {
    type: 'result',
    resultForAssistant: formatOutput(result),
    data: result
  }
}
```

## 4. 并发vs串行执行机制

### 4.1 并发执行条件
```typescript
// src/query.ts:184-188
if (
  toolUseMessages.every(msg =>
    toolUseContext.options.tools.find(t => t.name === msg.name)?.isReadOnly(),
  )
) {
  // 🎯 所有工具都是只读 → 并发执行
  for await (const message of runToolsConcurrently(...)) {
    yield message
  }
}
```

### 4.2 并发执行实现
```typescript
// src/query.ts:244-264
async function* runToolsConcurrently(
  toolUseMessages: ToolUseBlock[],
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
  shouldSkipPermissionCheck?: boolean,
): AsyncGenerator<Message, void> {
  // 🎯 使用all()工具函数实现并发
  yield* all(
    toolUseMessages.map(toolUse =>
      runToolUse(
        toolUse,
        new Set(toolUseMessages.map(_ => _.id)),  // 传入所有兄弟工具ID
        assistantMessage,
        canUseTool,
        toolUseContext,
        shouldSkipPermissionCheck,
      ),
    ),
    MAX_TOOL_USE_CONCURRENCY,  // 最大并发数 = 10
  )
}
```

### 4.3 串行执行实现
```typescript
// src/query.ts:266-283
async function* runToolsSerially(
  toolUseMessages: ToolUseBlock[],
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext,
  shouldSkipPermissionCheck?: boolean,
): AsyncGenerator<Message, void> {
  // 🎯 逐个执行工具
  for (const toolUse of toolUseMessages) {
    yield* runToolUse(
      toolUse,
      new Set(toolUseMessages.map(_ => _.id)),
      assistantMessage,
      canUseTool,
      toolUseContext,
      shouldSkipPermissionCheck,
    )
  }
}
```

## 5. 工具结果排序机制

```typescript
// src/query.ts:223-233
// 🎯 确保工具结果按照LLM请求的顺序返回
const orderedToolResults = toolResults.sort((a, b) => {
  const aIndex = toolUseMessages.findIndex(
    tu => tu.id === (a.message.content[0] as ToolUseBlock).id,
  )
  const bIndex = toolUseMessages.findIndex(
    tu => tu.id === (b.message.content[0] as ToolUseBlock).id,
  )
  return aIndex - bIndex
})
```

## 6. 输入验证的三层架构

### 第一层：Zod Schema类型验证
- **目的**: 确保LLM传入的参数类型正确
- **验证内容**: 数据类型、必需字段、格式约束
- **失败处理**: 返回`InputValidationError`

### 第二层：工具自定义业务验证  
- **目的**: 验证业务逻辑和安全性
- **验证内容**: 文件路径安全、命令危险性、参数合理性
- **失败处理**: 返回具体的业务错误信息

### 第三层：用户权限确认
- **目的**: 确保用户同意执行潜在危险操作
- **验证内容**: 用户交互确认
- **失败处理**: 返回权限拒绝消息

## 7. 执行上下文 (ToolUseContext) 的作用

```typescript
interface ToolUseContext {
  abortController: AbortController,     // 🎯 中断控制
  messageId?: string,                   // 🎯 消息追踪
  options: {
    commands: Command[],                // 🎯 可用命令列表
    tools: Tool[],                      // 🎯 可用工具列表
    slowAndCapableModel: string,        // 🎯 LLM模型信息
    forkNumber: number,                 // 🎯 会话分支号
    messageLogName: string,             // 🎯 日志文件名
    maxThinkingTokens: number,          // 🎯 思考token限制
  },
  readFileTimestamps: Record<string, number>,  // 🎯 文件读取缓存
  setToolJSX?: (jsx: React.ReactNode) => void, // 🎯 工具UI渲染
}
```

## 8. 关键设计模式

### 8.1 异步生成器模式
- **优势**: 支持流式处理和实时进度反馈
- **应用**: 工具执行可以yield多次进度更新

### 8.2 责任链模式
- **应用**: 三层验证机制
- **优势**: 每层负责特定验证职责，失败时提前返回

### 8.3 策略模式
- **应用**: 并发vs串行执行决策
- **优势**: 根据工具特性选择最优执行策略

### 8.4 命令模式
- **应用**: 工具调用的统一接口
- **优势**: 所有工具都有相同的调用方式

这种执行流程设计确保了工具调用的安全性、可靠性和性能优化。