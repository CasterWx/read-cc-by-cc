# Claude Code 工具结果返回和上下文整合机制分析

## 1. 工具结果数据结构

### 1.1 FullToolUseResult定义
**文件位置**: `src/utils/messages.tsx:106-109`

```typescript
export type FullToolUseResult = {
  data: unknown                                    // 🎯 工具返回的结构化数据
  resultForAssistant: ToolResultBlockParam['content']  // 🎯 发送给LLM的文本内容
}
```

### 1.2 工具yield的结果类型
```typescript
// 工具call方法的返回类型
type ToolYieldResult = {
  type: 'result'
  resultForAssistant: string    // 🎯 必需：LLM看到的内容
  data?: any                   // 🎯 可选：结构化数据(用于UI显示等)
} | {
  type: 'progress'             // 🎯 进度更新
  content: string
  normalizedMessages?: NormalizedMessage[]
  tools?: Tool[]
}
```

## 2. 工具结果转换为消息流程

### 2.1 成功结果处理
**文件位置**: `src/query.ts:444-462`

```typescript
// 工具执行成功后的处理
case 'result':
  logEvent('tengu_tool_use_success', {
    messageID: assistantMessage.message.id,
    toolName: tool.name,
  })
  
  // 🎯 关键转换：工具结果 → UserMessage
  yield createUserMessage(
    [{
      type: 'tool_result',              // 🎯 Anthropic API标准格式
      content: result.resultForAssistant,  // 🎯 LLM接收的内容
      tool_use_id: toolUseID,           // 🎯 关联到原始工具调用
    }],
    {
      data: result.data,                // 🎯 附加结构化数据
      resultForAssistant: result.resultForAssistant,
    },
  )
  return
```

### 2.2 进度结果处理
```typescript
// 工具执行过程中的进度更新
case 'progress':
  logEvent('tengu_tool_use_progress', {
    messageID: assistantMessage.message.id,
    toolName: tool.name,
  })
  
  // 🎯 创建进度消息(不会发送给LLM)
  yield createProgressMessage(
    toolUseID,
    siblingToolUseIDs,
    result.content,
    result.normalizedMessages,
    result.tools,
  )
```

## 3. 消息创建机制详解

### 3.1 createUserMessage函数
**文件位置**: `src/utils/messages.tsx:111-125`

```typescript
export function createUserMessage(
  content: string | ContentBlockParam[],  // 🎯 消息内容
  toolUseResult?: FullToolUseResult,     // 🎯 工具结果元数据
): UserMessage {
  const m: UserMessage = {
    type: 'user',
    message: {
      role: 'user',
      content,                          // 🎯 发送给LLM的实际内容
    },
    uuid: randomUUID(),                 // 🎯 唯一标识符
    toolUseResult,                      // 🎯 工具结果附加信息
  }
  return m
}
```

### 3.2 工具结果的标准格式
```typescript
// Anthropic API标准tool_result格式
const toolResultContent: ToolResultBlockParam = {
  type: 'tool_result',
  content: result.resultForAssistant,   // 🎯 工具返回的文本
  tool_use_id: toolUseID,              // 🎯 对应的工具调用ID
  is_error?: boolean,                  // 🎯 可选：是否为错误结果
}
```

## 4. 上下文整合机制

### 4.1 消息历史整合
**文件位置**: `src/query.ts:234-242`

```typescript
// 🎯 关键：工具结果自动加入对话历史
yield* await query(
  [...messages, assistantMessage, ...orderedToolResults],  // 🎯 完整对话历史
  systemPrompt,
  context,
  canUseTool,
  toolUseContext,
  getBinaryFeedbackResponse,
)
```

**消息序列示例**:
```typescript
// 执行前的消息历史
const messages = [
  { type: 'user', content: 'Read the README.md file' },
]

// LLM响应(包含工具调用)
const assistantMessage = {
  type: 'assistant',
  content: [
    { type: 'text', text: 'I\'ll read the README.md file for you.' },
    { type: 'tool_use', id: 'toolu_123', name: 'Read', input: {...} }
  ]
}

// 工具执行结果
const toolResults = [{
  type: 'user',
  content: [{
    type: 'tool_result',
    content: 'File content here...',  // 🎯 文件实际内容
    tool_use_id: 'toolu_123'
  }]
}]

// 🎯 整合后传给下一轮LLM的完整历史
const nextRoundMessages = [
  { type: 'user', content: 'Read the README.md file' },
  { 
    type: 'assistant', 
    content: [
      { type: 'text', text: 'I\'ll read the README.md file for you.' },
      { type: 'tool_use', id: 'toolu_123', name: 'Read', input: {...} }
    ]
  },
  { 
    type: 'user',
    content: [{
      type: 'tool_result',
      content: 'File content here...',  // 🎯 LLM现在可以看到文件内容
      tool_use_id: 'toolu_123'
    }]
  }
]
```

### 4.2 工具结果排序保证
```typescript
// src/query.ts:223-233
// 🎯 确保多工具结果按LLM请求顺序返回
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

## 5. 具体工具结果处理示例

### 5.1 FileReadTool结果处理
```typescript
// 工具内部处理
async *call(input, context) {
  const content = await readFile(file_path, 'utf-8')
  const lines = content.split('\n')
  const formattedContent = lines
    .map((line, index) => `${index + 1}→${line}`)
    .join('\n')
  
  // 🎯 返回给系统的结果
  yield {
    type: 'result',
    resultForAssistant: formattedContent,  // 🎯 LLM看到带行号的内容
    data: { 
      content,                            // 🎯 原始文件内容
      totalLines: lines.length,           // 🎯 统计信息
      filePath: file_path                 // 🎯 文件路径
    }
  }
}

// 系统转换为标准消息
const userMessage = createUserMessage([{
  type: 'tool_result',
  content: formattedContent,              // 🎯 格式化的文件内容
  tool_use_id: 'toolu_123'
}], {
  data: { content, totalLines, filePath }, // 🎯 结构化数据
  resultForAssistant: formattedContent
})
```

### 5.2 BashTool结果处理
```typescript
// BashTool内部处理
async *call(input, context) {
  const shell = PersistentShell.getInstance()
  const result = await shell.exec(command, timeout)
  
  // 🎯 返回格式化的命令输出
  yield {
    type: 'result',
    resultForAssistant: formatOutput(result),  // 🎯 格式化的输出
    data: {
      stdout: result.stdout,                  // 🎯 标准输出
      stderr: result.stderr,                  // 🎯 错误输出
      exitCode: result.exitCode,              // 🎯 退出码
      command: command,                       // 🎯 执行的命令
      interrupted: result.interrupted         // 🎯 是否被中断
    }
  }
}

// formatOutput函数处理
function formatOutput(result): string {
  let output = ''
  
  if (result.stdout) {
    output += result.stdout
  }
  
  if (result.stderr) {
    output += result.stderr
  }
  
  if (result.exitCode !== 0) {
    output += `\nExit code: ${result.exitCode}`
  }
  
  return output || '(no output)'
}
```

### 5.3 AgentTool复杂结果处理
```typescript
// AgentTool可以产生多次进度更新
async *call(input, context) {
  const task = input.task
  
  // 🎯 第一次进度更新
  yield {
    type: 'progress',
    content: `Starting to work on: ${task}`,
    normalizedMessages: [...],
    tools: context.options.tools
  }
  
  // 执行子任务...
  const subResults = await executeSubTasks(task)
  
  // 🎯 第二次进度更新
  yield {
    type: 'progress', 
    content: `Completed subtasks, analyzing results...`,
    normalizedMessages: [...],
    tools: context.options.tools
  }
  
  // 🎯 最终结果
  yield {
    type: 'result',
    resultForAssistant: `Task completed successfully: ${task}\n\nResults:\n${subResults}`,
    data: {
      task,
      subResults,
      executionTime: Date.now() - startTime
    }
  }
}
```

## 6. 错误结果处理

### 6.1 工具不存在错误
```typescript
// src/query.ts:304-311
yield createUserMessage([{
  type: 'tool_result',
  content: `Error: No such tool available: ${toolName}`,
  is_error: true,                         // 🎯 错误标识
  tool_use_id: toolUse.id,
}])
```

### 6.2 参数验证错误
```typescript
// src/query.ts:385-393
yield createUserMessage([{
  type: 'tool_result',
  content: `InputValidationError: ${isValidInput.error.message}`,
  is_error: true,                         // 🎯 错误标识
  tool_use_id: toolUseID,
}])
```

### 6.3 工具执行异常
```typescript
// src/query.ts:486-493
catch (error) {
  const content = formatError(error)
  yield createUserMessage([{
    type: 'tool_result',
    content,                              // 🎯 格式化的错误信息
    is_error: true,                       // 🎯 错误标识
    tool_use_id: toolUseID,
  }])
}
```

## 7. 进度消息处理

### 7.1 createProgressMessage函数
**文件位置**: `src/utils/messages.tsx:127`

```typescript
export function createProgressMessage(
  toolUseID: string,                      // 🎯 工具调用ID
  siblingToolUseIDs: Set<string>,         // 🎯 并行工具ID集合
  content: AssistantMessage,              // 🎯 进度内容
  normalizedMessages: NormalizedMessage[], // 🎯 标准化消息历史
  tools: Tool[],                          // 🎯 可用工具列表
): ProgressMessage {
  return {
    type: 'progress',
    toolUseID,
    siblingToolUseIDs,
    content,
    normalizedMessages,
    tools,
    uuid: randomUUID(),
  }
}
```

### 7.2 进度消息的特殊处理
- **不发送给LLM**: 进度消息仅用于UI更新，不会包含在API请求中
- **实时反馈**: 通过异步生成器实现流式进度更新
- **UI渲染**: 可以包含React组件用于复杂的进度显示

## 8. 消息过滤和API格式化

### 8.1 API消息过滤
**文件位置**: `src/utils/messages.tsx` (推断)

```typescript
// 为API调用准备消息时过滤掉进度消息
export function normalizeMessagesForAPI(messages: Message[]): MessageParam[] {
  return messages
    .filter(msg => msg.type !== 'progress')  // 🎯 过滤进度消息
    .map(msg => msg.type === 'user' 
      ? userMessageToMessageParam(msg)
      : assistantMessageToMessageParam(msg)
    )
}
```

### 8.2 用户消息转API格式
```typescript
function userMessageToMessageParam(message: UserMessage): MessageParam {
  return {
    role: 'user',
    content: message.message.content       // 🎯 包含tool_result的内容
  }
}
```

## 9. 关键设计模式和机制

### 9.1 流式处理模式
- **异步生成器**: 支持实时进度更新和结果流式返回
- **背压处理**: 可以控制消息产生速度

### 9.2 消息转换模式
- **标准化接口**: 所有工具结果都转换为统一的UserMessage格式
- **双重数据**: resultForAssistant给LLM，data给UI和日志

### 9.3 上下文累积模式
- **历史保持**: 工具结果自动加入对话历史
- **连续对话**: 支持基于前一轮工具结果的进一步LLM调用

### 9.4 错误传播模式
- **统一错误格式**: 所有错误都以tool_result + is_error形式返回
- **错误上下文**: 包含足够信息让LLM理解和处理错误

这种结果处理机制确保了工具执行结果能够无缝集成到LLM对话流程中，同时保持了数据的完整性和可追溯性。