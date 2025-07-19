# Claude Code 用户输入到输出流程分析

## 主流程概览

用户输入 → 命令解析 → LLM处理 → 工具执行 → 结果渲染 → 用户输出

## 详细流程分解

### 1. 程序启动与初始化
**入口**: `src/entrypoints/cli.tsx:1043 main()`

```typescript
// 启动流程
main() {
  enableConfigs()           // 启用配置系统
  parseArgs()              // 解析命令行参数
    ↓
  showSetupScreens()       // 显示设置界面(首次运行)
    ↓  
  setup(cwd)               // 初始化工作环境
    ↓
  render(<REPL />)         // 启动交互界面
}
```

### 2. 用户输入处理
**核心文件**: `src/components/PromptInput.tsx`, `src/utils/messages.tsx`

#### 2.1 输入接收
```typescript
// PromptInput组件接收用户输入
onSubmit = async (input: string, mode: 'bash' | 'prompt') => {
  // 输入验证和预处理
  const processedInput = preprocessInput(input, mode)
  
  // 调用父组件的查询处理
  await onQuery(newMessages, abortController)
}
```

#### 2.2 消息处理
```typescript
// src/utils/messages.tsx:processUserInput
export async function processUserInput(
  input: string,
  mode: 'bash' | 'prompt',
  setToolJSX: Function,
  context: ToolUseContext
): Promise<MessageType[]> {
  
  // 1. 检查是否为斜杠命令
  if (input.startsWith('/')) {
    return await handleSlashCommand(input, context)
  }
  
  // 2. 检查是否为bash命令
  if (mode === 'bash') {
    return await handleBashCommand(input, context)
  }
  
  // 3. 处理为常规用户消息
  return [createUserMessage(input)]
}
```

### 3. 核心查询处理
**核心文件**: `src/query.ts:124`

```typescript
// 主要查询循环
export async function* query(
  messages: Message[],
  systemPrompt: string[],
  context: { [k: string]: string },
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext
): AsyncGenerator<Message, void> {

  // 步骤1: 格式化系统提示词
  const fullSystemPrompt = formatSystemPromptWithContext(systemPrompt, context)
  
  // 步骤2: 调用LLM API
  const assistantMessage = await querySonnet(
    normalizeMessagesForAPI(messages),
    fullSystemPrompt,
    toolUseContext.options.maxThinkingTokens,
    toolUseContext.options.tools,
    toolUseContext.abortController.signal
  )
  
  yield assistantMessage  // 输出LLM响应
  
  // 步骤3: 检查是否需要工具调用
  const toolUseMessages = assistantMessage.message.content.filter(
    _ => _.type === 'tool_use'
  )
  
  if (!toolUseMessages.length) {
    return  // 无工具调用，流程结束
  }
  
  // 步骤4: 执行工具调用
  const toolResults: UserMessage[] = []
  
  // 并发或串行执行工具
  if (canRunConcurrently(toolUseMessages)) {
    yield* runToolsConcurrently(...)
  } else {
    yield* runToolsSerially(...)
  }
  
  // 步骤5: 递归处理(如果需要进一步LLM调用)
  yield* query([...messages, assistantMessage, ...toolResults], ...)
}
```

### 4. LLM API交互
**核心文件**: `src/services/claude.ts:402`

```typescript
export async function querySonnet(...): Promise<AssistantMessage> {
  
  // 1. 获取Anthropic客户端
  const anthropic = getAnthropicClient(options.model)
  
  // 2. 准备请求参数
  const system = formatSystemPrompt(systemPrompt)
  const toolSchemas = await formatTools(tools)
  
  // 3. 发送流式请求
  const response = await withRetry(async attempt => {
    const stream = anthropic.beta.messages.stream({
      model: options.model,
      max_tokens: getMaxTokensForModel(options.model),
      messages: addCacheBreakpoints(messages),
      temperature: MAIN_QUERY_TEMPERATURE,
      system,
      tools: toolSchemas,
      metadata: getMetadata(),
    }, { signal })
    
    return handleMessageStream(stream)
  })
  
  // 4. 处理响应和计费
  const costUSD = calculateCost(response.usage)
  addToTotalCost(costUSD, durationMs)
  
  return formatAssistantMessage(response)
}
```

### 5. 工具执行流程
**核心文件**: `src/query.ts:285`

```typescript
export async function* runToolUse(
  toolUse: ToolUseBlock,
  siblingToolUseIDs: Set<string>,
  assistantMessage: AssistantMessage,
  canUseTool: CanUseToolFn,
  toolUseContext: ToolUseContext
): AsyncGenerator<Message, void> {

  // 1. 查找工具
  const tool = toolUseContext.options.tools.find(t => t.name === toolUse.name)
  
  // 2. 验证输入
  const isValidInput = tool.inputSchema.safeParse(toolUse.input)
  const isValidCall = await tool.validateInput?.(normalizedInput, context)
  
  // 3. 权限检查
  const permissionResult = await canUseTool(tool, normalizedInput, context, assistantMessage)
  
  // 4. 执行工具
  const generator = tool.call(normalizedInput, context, canUseTool)
  for await (const result of generator) {
    switch (result.type) {
      case 'result':
        yield createUserMessage([{
          type: 'tool_result',
          content: result.resultForAssistant,
          tool_use_id: toolUseID,
        }])
        return
        
      case 'progress':
        yield createProgressMessage(toolUseID, ...)
    }
  }
}
```

### 6. 结果渲染和输出
**核心文件**: `src/screens/REPL.tsx:413`

```typescript
// REPL组件的消息渲染
const messagesJSX = useMemo(() => {
  return [
    // 静态Logo
    { type: 'static', jsx: <Logo /> },
    
    // 动态消息列表
    ...reorderMessages(normalizedMessages).map(message => {
      const shouldRenderStatically = checkIfStatic(message)
      
      return {
        type: shouldRenderStatically ? 'static' : 'transient',
        jsx: (
          <Message
            message={message}
            tools={tools}
            verbose={verbose}
            debug={debug}
            shouldAnimate={!toolJSX && !toolUseConfirm}
          />
        )
      }
    })
  ]
}, [messages, tools, verbose, debug, ...])

// 分层渲染: 静态内容 + 动态内容
return (
  <>
    <Static items={messagesJSX.filter(_ => _.type === 'static')}>
      {_ => _.jsx}
    </Static>
    {messagesJSX.filter(_ => _.type === 'transient').map(_ => _.jsx)}
    
    {/* 工具执行界面 */}
    {toolJSX ? toolJSX.jsx : null}
    
    {/* 权限确认界面 */}
    {toolUseConfirm && <PermissionRequest />}
    
    {/* 输入界面 */}
    <PromptInput onQuery={onQuery} />
  </>
)
```

## 特殊流程处理

### 斜杠命令流程
```typescript
// 1. 命令识别
if (input.startsWith('/')) {
  const [commandName, ...args] = input.slice(1).split(' ')
  
  // 2. 命令查找
  const command = getCommand(commandName, commands)
  
  // 3. 命令执行
  switch (command.type) {
    case 'prompt':
      // 转换为LLM提示词
      const promptMessages = await command.getPromptForCommand(args.join(' '))
      return promptMessages
      
    case 'local':
      // 本地执行
      const result = await command.call(args.join(' '), context)
      return [createAssistantMessage(result)]
      
    case 'local-jsx':
      // JSX组件渲染
      const jsx = await command.call(onDone, context)
      setToolJSX({ jsx, shouldHidePromptInput: true })
      return []
  }
}
```

### Bash命令流程
```typescript
// 直接执行bash命令，不经过LLM
if (mode === 'bash') {
  const result = await BashTool.call({ command: input }, context, canUseTool)
  return [
    createUserMessage(`$ ${input}`),
    createAssistantMessage(result.resultForAssistant)
  ]
}
```

## 流程控制要点

1. **异步生成器模式**: 核心使用`async function*`支持流式处理
2. **错误处理**: 每个环节都有完善的错误捕获和重试机制
3. **权限控制**: 工具执行前必须通过权限检查
4. **状态同步**: 通过React状态管理保持UI同步
5. **中断支持**: 使用AbortController支持用户中断操作

这种流程设计既保证了用户体验的流畅性，又确保了系统的安全性和可控性。