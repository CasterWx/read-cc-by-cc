# Claude Code 工具调用错误处理和异常捕获机制分析

## 1. 错误处理架构概览

Claude Code的工具调用系统采用**多层错误处理架构**，确保任何层级的错误都能被妥善处理并反馈给LLM。

```
错误层级：
1. 工具查找错误 (runToolUse)
2. 用户中断错误 (abortController)  
3. 输入验证错误 (Zod + 自定义验证)
4. 权限检查错误 (canUseTool)
5. 工具执行异常 (tool.call)
6. 结果格式化错误 (formatError)
```

## 2. 核心错误捕获机制

### 2.1 主要try-catch块位置
**文件位置**: `src/query.ts:440-495`

```typescript
// 🎯 最外层异常捕获 - 捕获工具执行中的所有异常
try {
  const generator = tool.call(normalizedInput as never, context, canUseTool)
  for await (const result of generator) {
    // 处理工具结果...
  }
} catch (error) {
  // 🎯 关键错误处理逻辑
  const content = formatError(error)
  logError(error)
  logEvent('tengu_tool_use_error', {
    error: content.slice(0, 2000),
    messageID: assistantMessage.message.id,
    toolName: tool.name,
    toolInput: JSON.stringify(input).slice(0, 1000),
  })
  
  // 🎯 将异常转换为tool_result返回给LLM
  yield createUserMessage([{
    type: 'tool_result',
    content,                    // 格式化的错误信息
    is_error: true,            // 🎯 错误标识
    tool_use_id: toolUseID,
  }])
}
```

### 2.2 错误格式化函数
**文件位置**: `src/query.ts:497-516`

```typescript
function formatError(error: unknown): string {
  if (!(error instanceof Error)) {
    return String(error)
  }
  
  const parts = [error.message]
  
  // 🎯 特殊处理：包含stderr和stdout的错误(如Shell命令执行错误)
  if ('stderr' in error && typeof error.stderr === 'string') {
    parts.push(error.stderr)
  }
  if ('stdout' in error && typeof error.stdout === 'string') {
    parts.push(error.stdout)
  }
  
  const fullMessage = parts.filter(Boolean).join('\n')
  
  // 🎯 长错误消息截断处理(避免超长错误信息)
  if (fullMessage.length <= 10000) {
    return fullMessage
  }
  
  const halfLength = 5000
  const start = fullMessage.slice(0, halfLength)
  const end = fullMessage.slice(-halfLength)
  return `${start}\n\n... [${fullMessage.length - 10000} characters truncated] ...\n\n${end}`
}
```

## 3. 分层错误处理详解

### 3.1 第一层：工具存在性检查
**文件位置**: `src/query.ts:297-313`

```typescript
// 🎯 错误类型：工具不存在
const tool = toolUseContext.options.tools.find(t => t.name === toolName)

if (!tool) {
  logEvent('tengu_tool_use_error', {
    error: `No such tool available: ${toolName}`,
    messageID: assistantMessage.message.id,
    toolName,
    toolUseID: toolUse.id,
  })
  
  yield createUserMessage([{
    type: 'tool_result',
    content: `Error: No such tool available: ${toolName}`,
    is_error: true,
    tool_use_id: toolUse.id,
  }])
  return
}
```

**错误场景**:
- LLM请求了不存在的工具
- 工具被动态禁用
- 工具名称拼写错误

### 3.2 第二层：用户中断检查
**文件位置**: `src/query.ts:318-328`

```typescript
// 🎯 错误类型：用户中断操作
if (toolUseContext.abortController.signal.aborted) {
  logEvent('tengu_tool_use_cancelled', {
    toolName: tool.name,
    toolUseID: toolUse.id,
  })
  
  const message = createUserMessage([
    createToolResultStopMessage(toolUse.id),  // 🎯 专门的中断消息
  ])
  yield message
  return
}
```

**错误场景**:
- 用户按Ctrl+C中断
- 工具执行超时
- 系统资源不足强制终止

### 3.3 第三层：输入验证错误
**文件位置**: `src/query.ts:375-394`

```typescript
// 🎯 错误类型：Zod Schema验证失败
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

**错误示例**:
```javascript
// LLM提供的错误输入
{
  "name": "Read",
  "input": {
    "file_path": 123,        // 🎯 错误：应该是string
    "limit": "not_a_number"  // 🎯 错误：应该是number
  }
}

// Zod验证错误消息
"InputValidationError: Expected string, received number at path file_path"
```

### 3.4 第四层：自定义业务验证错误
**文件位置**: `src/query.ts:399-420`

```typescript
// 🎯 错误类型：工具自定义验证失败
const isValidCall = await tool.validateInput?.(normalizedInput, context)
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

**自定义验证示例** (FileReadTool):
```typescript
async validateInput(input, context): Promise<ValidationResult> {
  const { file_path } = input
  
  // 🎯 安全检查：文件路径
  if (file_path.includes('..')) {
    return {
      result: false,
      message: 'Path traversal not allowed',
      meta: { securityViolation: true }
    }
  }
  
  // 🎯 存在检查：文件是否存在
  if (!existsSync(file_path)) {
    return {
      result: false,
      message: `File does not exist: ${file_path}`,
      meta: { fileNotFound: true }
    }
  }
  
  return { result: true }
}
```

### 3.5 第五层：权限检查错误
**文件位置**: `src/query.ts:422-437`

```typescript
// 🎯 错误类型：用户拒绝权限
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

**权限拒绝消息示例**:
```
"The user doesn't want to proceed with this tool use. The tool use was rejected (eg. if it was a file edit, the new_string was NOT written to the file). STOP what you are doing and wait for the user to tell you how to proceed."
```

### 3.6 第六层：工具执行异常
**文件位置**: `src/query.ts:477-494`

```typescript
// 🎯 最终异常捕获：工具call方法抛出的任何异常
} catch (error) {
  const content = formatError(error)
  logError(error)                          // 🎯 服务端日志记录
  logEvent('tengu_tool_use_error', {       // 🎯 统计事件记录
    error: content.slice(0, 2000),
    messageID: assistantMessage.message.id,
    toolName: tool.name,
    toolInput: JSON.stringify(input).slice(0, 1000),
  })
  
  yield createUserMessage([{
    type: 'tool_result',
    content,
    is_error: true,
    tool_use_id: toolUseID,
  }])
}
```

## 4. 具体工具的错误处理示例

### 4.1 BashTool错误处理
```typescript
// BashTool内部的错误处理
async *call(input, context) {
  try {
    const shell = PersistentShell.getInstance()
    const result = await shell.exec(command, timeout)
    
    // 🎯 命令执行失败处理
    if (result.exitCode !== 0) {
      yield {
        type: 'result',
        resultForAssistant: `Command failed with exit code ${result.exitCode}:\n${result.stderr}`,
        data: result
      }
      return
    }
    
    yield {
      type: 'result', 
      resultForAssistant: formatOutput(result),
      data: result
    }
  } catch (error) {
    // 🎯 Shell执行异常 → 被外层try-catch捕获
    throw new Error(`Shell execution failed: ${error.message}`)
  }
}
```

### 4.2 FileReadTool错误处理
```typescript
async *call(input, context) {
  const { file_path, limit, offset } = input
  
  try {
    // 🎯 文件读取可能的异常点
    const content = await readFile(file_path, 'utf-8')
    
    // 处理内容...
    yield { type: 'result', resultForAssistant: formattedContent, data: {...} }
    
  } catch (error) {
    // 🎯 常见文件错误处理
    if (error.code === 'ENOENT') {
      throw new Error(`File not found: ${file_path}`)
    }
    if (error.code === 'EACCES') {
      throw new Error(`Permission denied: ${file_path}`)
    }
    if (error.code === 'EISDIR') {
      throw new Error(`Path is a directory, not a file: ${file_path}`)
    }
    
    // 🎯 其他未知错误
    throw new Error(`Failed to read file ${file_path}: ${error.message}`)
  }
}
```

## 5. 错误日志和监控

### 5.1 统计事件记录
**所有错误都会记录统计事件**:
```typescript
logEvent('tengu_tool_use_error', {
  error: errorMessage.slice(0, 2000),      // 🎯 错误信息(截断)
  messageID: assistantMessage.message.id,  // 🎯 关联的消息ID
  toolName: tool.name,                     // 🎯 工具名称
  toolInput: JSON.stringify(input).slice(0, 1000),  // 🎯 输入参数
  ...additionalMeta                        // 🎯 额外元数据
})
```

### 5.2 服务端错误日志
```typescript
// src/utils/log.ts - logError函数
logError(error)  // 🎯 写入错误日志文件，包含堆栈信息
```

### 5.3 错误分类统计
根据代码分析，错误被分类为：
- `tengu_tool_use_error` - 一般工具错误
- `tengu_tool_use_cancelled` - 用户取消
- `tengu_tool_use_success` - 成功执行(对比)
- `tengu_tool_use_progress` - 执行进度(对比)

## 6. 错误恢复机制

### 6.1 LLM错误自愈能力
```typescript
// 错误消息被发送给LLM，LLM可以：
// 1. 重新尝试正确的工具调用
// 2. 使用不同的工具达到目标
// 3. 向用户说明问题并请求帮助

// 示例错误恢复流程：
[
  { type: 'user', content: 'Read the file config.yaml' },
  { type: 'assistant', content: [
    { type: 'text', text: 'I\'ll read the config.yaml file.' },
    { type: 'tool_use', name: 'Read', input: { file_path: 'config.yaml' } }
  ]},
  { type: 'user', content: [{
    type: 'tool_result',
    content: 'File not found: config.yaml',  // 🎯 错误信息
    is_error: true,
    tool_use_id: 'toolu_123'
  }]},
  { type: 'assistant', content: [
    { type: 'text', text: 'The file config.yaml was not found. Let me check what files are available.' },
    { type: 'tool_use', name: 'LS', input: { path: '.' } }  // 🎯 LLM自动尝试其他方案
  ]}
]
```

### 6.2 重试机制
Claude Code本身不实现自动重试，但LLM可以基于错误信息进行智能重试：
- 参数错误 → 调整参数重试
- 权限错误 → 请求用户权限后重试  
- 文件不存在 → 搜索文件后重试

## 7. 特殊错误处理场景

### 7.1 工具调用死循环检测
虽然代码中没有显式的死循环检测，但通过以下机制避免：
- `abortController` 中断机制
- Token限制(当context达到限制时会触发/compact)
- 用户可随时中断操作

### 7.2 内存不足处理
```typescript
// 长错误消息处理避免内存爆炸
if (fullMessage.length <= 10000) {
  return fullMessage
}
// 截断处理...
```

### 7.3 并发工具错误处理
并发执行时，任何一个工具出错不会影响其他工具：
```typescript
// 每个工具都有独立的错误处理
for await (const message of runToolsConcurrently(...)) {
  yield message  // 🎯 错误和成功结果都会被yield
}
```

## 8. 错误处理设计模式

### 8.1 责任链模式
错误处理采用责任链模式，每层处理特定类型的错误：
1. 工具查找 → 工具不存在错误
2. 中断检查 → 用户取消错误  
3. 输入验证 → 参数格式错误
4. 权限检查 → 权限拒绝错误
5. 工具执行 → 业务逻辑错误

### 8.2 转换模式
所有错误都转换为统一的`tool_result`格式：
```typescript
{
  type: 'tool_result',
  content: errorMessage,      // 🎯 人类可读的错误描述
  is_error: true,            // 🎯 明确标识为错误
  tool_use_id: toolUseID     // 🎯 关联到原始工具调用
}
```

### 8.3 优雅降级模式
工具执行失败时，系统不会崩溃，而是：
1. 记录错误日志
2. 将错误信息返回给LLM
3. 让LLM决定如何继续对话
4. 保持系统整体稳定运行

这种多层次的错误处理架构确保了Claude Code工具调用系统的高可靠性和良好的用户体验。