# Claude Code 工具调用决策机制逆向分析

## 1. 调用决策核心流程

**关键代码位置**: `src/query.ts:169-178`

```typescript
// LLM响应后的工具调用检测逻辑
export async function* query(...): AsyncGenerator<Message, void> {
  // 1. 获取LLM响应
  const assistantMessage = await querySonnet(...)
  yield assistantMessage

  // 2. 🔍 关键决策点：检测是否包含工具调用
  // @see https://docs.anthropic.com/en/docs/build-with-claude/tool-use
  // Note: stop_reason === 'tool_use' is unreliable -- it's not always set correctly
  const toolUseMessages = assistantMessage.message.content.filter(
    _ => _.type === 'tool_use',  // 🎯 核心判断逻辑
  )

  // 3. 决策分支
  if (!toolUseMessages.length) {
    return  // 无工具调用，流程结束
  }

  // 4. 有工具调用，继续执行工具
  // ...执行工具逻辑
}
```

## 2. LLM响应结构分析

**Anthropic API返回的message.content结构**:
```typescript
type ContentBlock = 
  | { type: 'text', text: string }           // 普通文本回复
  | { type: 'tool_use', id: string, name: string, input: object }  // 🎯 工具调用标识

// LLM可以在一个响应中同时包含文本和工具调用
assistantMessage.message.content: ContentBlock[] = [
  { type: 'text', text: 'I'll help you read that file.' },
  { 
    type: 'tool_use', 
    id: 'toolu_01234567890abcdef',
    name: 'Read',                    // 🎯 工具名称
    input: { file_path: '/path/to/file.txt' }  // 🎯 工具参数
  }
]
```

## 3. 工具调用决策的完整检测链

### 3.1 第一层：内容类型过滤
```typescript
// src/query.ts:171-173
const toolUseMessages = assistantMessage.message.content.filter(
  _ => _.type === 'tool_use',  // 只保留tool_use类型的内容块
)
```

### 3.2 第二层：工具存在性验证
```typescript
// src/query.ts:294
const tool = toolUseContext.options.tools.find(t => t.name === toolName)

if (!tool) {
  // 工具不存在，返回错误
  yield createUserMessage([{
    type: 'tool_result',
    content: `Error: No such tool available: ${toolName}`,
    is_error: true,
    tool_use_id: toolUse.id,
  }])
  return
}
```

### 3.3 第三层：并发执行决策
```typescript
// src/query.ts:184-188
// 决定是并发还是串行执行工具
if (
  toolUseMessages.every(msg =>
    toolUseContext.options.tools.find(t => t.name === msg.name)?.isReadOnly(),
  )
) {
  // 所有工具都是只读 → 并发执行
  for await (const message of runToolsConcurrently(...)) {
    yield message
  }
} else {
  // 有写操作工具 → 串行执行
  for await (const message of runToolsSerially(...)) {
    yield message
  }
}
```

## 4. 工具调用的数据结构解析

### 4.1 ToolUseBlock结构
```typescript
// 从LLM响应中提取的工具调用信息
interface ToolUseBlock {
  type: 'tool_use'
  id: string        // 工具调用的唯一标识符(如: 'toolu_01234567890abcdef')
  name: string      // 工具名称(如: 'Read', 'Bash', 'FileEdit')
  input: object     // 工具参数(LLM生成的JSON对象)
}

// 实际示例
const toolUseExample: ToolUseBlock = {
  type: 'tool_use',
  id: 'toolu_01A2B3C4D5E6F789',
  name: 'Read',
  input: {
    file_path: '/Users/example/project/README.md',
    limit: 50,
    offset: 0
  }
}
```

### 4.2 工具调用识别的关键属性

**关键判断点**:
1. **`_.type === 'tool_use'`** - 内容块类型必须是工具调用
2. **`_.name`** - 工具名称必须在可用工具列表中
3. **`_.input`** - 参数对象必须符合工具的inputSchema

## 5. 并发执行决策算法

```typescript
// src/query.ts:184-188
function shouldRunConcurrently(toolUseMessages: ToolUseBlock[], tools: Tool[]): boolean {
  return toolUseMessages.every(msg => {
    const tool = tools.find(t => t.name === msg.name)
    return tool?.isReadOnly() === true  // 🎯 关键：所有工具都必须是只读的
  })
}

// 并发执行的工具类型示例
const readOnlyTools = [
  'Read',      // 文件读取
  'Glob',      // 文件搜索  
  'Grep',      // 内容搜索
  'LS',        // 目录列表
]

// 串行执行的工具类型示例
const writeTools = [
  'Bash',      // 命令执行(可能修改系统)
  'FileEdit',  // 文件编辑
  'FileWrite', // 文件写入
]
```

## 6. 错误处理中的决策点

### 6.1 工具不存在的处理
```typescript
// src/query.ts:297-313
if (!tool) {
  logEvent('tengu_tool_use_error', {
    error: `No such tool available: ${toolName}`,
    messageID: assistantMessage.message.id,
    toolName,
    toolUseID: toolUse.id,
  })
  
  // 🎯 关键：将错误作为tool_result返回给LLM
  yield createUserMessage([{
    type: 'tool_result',
    content: `Error: No such tool available: ${toolName}`,
    is_error: true,           // 🎯 错误标识
    tool_use_id: toolUse.id,  // 🎯 关联到原始工具调用
  }])
  return
}
```

### 6.2 中断检测
```typescript
// src/query.ts:318-328
if (toolUseContext.abortController.signal.aborted) {
  logEvent('tengu_tool_use_cancelled', {
    toolName: tool.name,
    toolUseID: toolUse.id,
  })
  
  // 🎯 用户中断也要通知LLM
  const message = createUserMessage([
    createToolResultStopMessage(toolUse.id),
  ])
  yield message
  return
}
```

## 7. 工具调用循环机制

```typescript
// src/query.ts:234-242
// 🔄 递归调用：工具执行完成后继续LLM对话
yield* await query(
  [...messages, assistantMessage, ...orderedToolResults],  // 🎯 添加工具结果到历史
  systemPrompt,
  context,
  canUseTool,
  toolUseContext,
  getBinaryFeedbackResponse,
)
```

**循环终止条件**:
1. LLM响应中没有`tool_use`类型的内容块
2. 用户主动中断(`abortController.signal.aborted`)
3. 工具执行出现不可恢复的错误

## 8. 实际调用决策示例

### 8.1 场景1：纯文本响应
```javascript
// LLM响应
{
  "content": [
    { "type": "text", "text": "Hello! How can I help you today?" }
  ]
}

// 决策结果：无工具调用，流程结束
toolUseMessages = []  // 长度为0
// → return (不执行工具)
```

### 8.2 场景2：单工具调用
```javascript
// LLM响应
{
  "content": [
    { "type": "text", "text": "I'll read that file for you." },
    { 
      "type": "tool_use", 
      "id": "toolu_123", 
      "name": "Read", 
      "input": { "file_path": "/path/to/file.txt" }
    }
  ]
}

// 决策结果：执行Read工具
toolUseMessages = [{ type: "tool_use", name: "Read", ... }]
// → 执行工具 → 将结果返回LLM → 可能触发新一轮对话
```

### 8.3 场景3：多工具并发调用
```javascript
// LLM响应
{
  "content": [
    { "type": "text", "text": "I'll search for files and check their contents." },
    { "type": "tool_use", "id": "toolu_1", "name": "Glob", "input": {"pattern": "*.js"} },
    { "type": "tool_use", "id": "toolu_2", "name": "Read", "input": {"file_path": "README.md"} }
  ]
}

// 决策结果：并发执行(都是只读工具)
toolUseMessages.every(msg => isReadOnly(msg.name)) === true
// → runToolsConcurrently() → 同时执行Glob和Read
```

## 9. 关键设计洞察

1. **基于内容类型的简单判断**: 不依赖`stop_reason`，而是直接检查内容块类型
2. **容错设计**: 工具不存在时返回错误而不是崩溃
3. **性能优化**: 只读工具可以并发执行，提升效率
4. **循环对话**: 工具结果自动加入对话历史，支持多轮工具调用
5. **中断支持**: 随时可以取消工具执行，用户体验友好

这种设计确保了工具调用的可靠性、性能和用户体验。