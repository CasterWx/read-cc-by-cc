# Claude Code Agent 记忆系统深度分析

## 🎯 分析概述

本文档专门针对AI工程师关心的记忆系统核心问题进行深度代码分析，覆盖历史记录存储、上下文注入、Prompt构建、压缩机制和窗口管理等关键领域。

---

## 1. 📚 历史记录数据结构分析

### 1.1 核心Message类型定义

**文件位置**: `src/query.ts:35-66`

```typescript
// 用户消息结构
export type UserMessage = {
  message: MessageParam          // Anthropic API标准消息格式
  type: 'user'
  uuid: UUID                     // 唯一标识符
  toolUseResult?: FullToolUseResult  // 工具执行结果(可选)
}

// AI助手消息结构
export type AssistantMessage = {
  costUSD: number               // API调用成本
  durationMs: number           // 响应时间
  message: APIAssistantMessage // Anthropic API响应
  type: 'assistant'
  uuid: UUID
  isApiErrorMessage?: boolean  // 错误消息标记
}

// 进度消息结构(工具执行过程中的实时反馈)
export type ProgressMessage = {
  content: AssistantMessage     // 内容载体
  normalizedMessages: NormalizedMessage[]  // 标准化消息
  siblingToolUseIDs: Set<string>          // 并行工具ID集合
  tools: Tool[]                           // 可用工具列表
  toolUseID: string                       // 当前工具ID
  type: 'progress'
  uuid: UUID
}

// 联合类型 - 系统中的消息数组就是这个类型
export type Message = UserMessage | AssistantMessage | ProgressMessage
```

### 1.2 消息存储机制

**REPL组件中的消息数组** (`src/screens/REPL.tsx:127`):
```typescript
// 核心历史记录存储 - React状态数组
const [messages, setMessages] = useState<MessageType[]>(initialMessages ?? [])

// 消息累积过程
for await (const message of query(...)) {
  setMessages(oldMessages => [...oldMessages, message])  // 数组追加模式
}
```

**持久化存储** (`src/utils/log.ts`):
- 每个会话保存为独立JSON文件: `~/.config/claude/messages/YYYY-MM-DD-HHMMSS-fork{N}.json`
- 文件内容: 完整的Message数组序列化
- 自动清理: 30天后删除旧记录

### 1.3 消息标准化处理

**标准化转换** (`src/utils/messages.tsx:550`):
```typescript
export type NormalizedMessage = NormalizedUserMessage | NormalizedAssistantMessage | ProgressMessage

// 为API调用准备的消息格式化
export function normalizeMessagesForAPI(messages: Message[]): MessageParam[] {
  return messages
    .filter(msg => msg.type !== 'progress')  // 过滤进度消息
    .map(msg => msg.type === 'user' 
      ? userMessageToMessageParam(msg)
      : assistantMessageToMessageParam(msg)
    )
}
```

---

## 2. 🔄 上下文注入机制分析

### 2.1 自动上下文收集

**核心函数**: `src/context.ts:157`
```typescript
export const getContext = memoize(async (): Promise<{[k: string]: string}> => {
  const codeStyle = getCodeStyle()
  const projectConfig = getCurrentProjectConfig()
  const dontCrawl = projectConfig.dontCrawlDirectory
  
  // 并行获取所有上下文信息
  const [gitStatus, directoryStructure, claudeFiles, readme] = await Promise.all([
    getGitStatus(),                    // Git状态和提交历史
    dontCrawl ? Promise.resolve('') : getDirectoryStructure(),  // 目录结构
    dontCrawl ? Promise.resolve('') : getClaudeFiles(),         // CLAUDE.md文件
    getReadme(),                       // README.md内容
  ])
  
  return {
    ...projectConfig.context,          // 用户自定义上下文
    ...(directoryStructure ? { directoryStructure } : {}),
    ...(gitStatus ? { gitStatus } : {}),
    ...(codeStyle ? { codeStyle } : {}),
    ...(claudeFiles ? { claudeFiles } : {}),
    ...(readme ? { readme } : {}),
  }
})
```

### 2.2 文件内容注入机制

**工具触发上下文更新**:
当用户使用文件相关工具时，文件内容通过以下路径进入对话：

1. **工具执行** → **文件读取** → **结果返回** → **LLM接收**

```typescript
// FileReadTool执行流程示例
const fileContent = await readFile(filePath, 'utf-8')
yield {
  type: 'result',
  resultForAssistant: `File: ${filePath}\n\n${fileContent}`,  // 直接注入LLM上下文
  data: fileContent
}
```

2. **CLAUDE.md自动注入**:
```typescript
// src/context.ts:24
export async function getClaudeFiles(): Promise<string | null> {
  const files = await ripGrep(['--files', '--glob', join('**', '*', 'CLAUDE.md')])
  return `NOTE: Additional CLAUDE.md files were found. When working in these directories, make sure to read and follow the instructions in the corresponding CLAUDE.md file:\n${files.map(_ => `- ${_}`).join('\n')}`
}
```

### 2.3 用户自定义上下文

**手动设置上下文** (`src/context.ts:50`):
```typescript
export function setContext(key: string, value: string): void {
  const projectConfig = getCurrentProjectConfig()
  const context = omit(
    { ...projectConfig.context, [key]: value },
    'codeStyle',           // 自动生成，不允许手动覆盖
    'directoryStructure',  // 自动生成，不允许手动覆盖
  )
  saveCurrentProjectConfig({ ...projectConfig, context })
}
```

---

## 3. 🏗️ Prompt构建过程分析

### 3.1 系统提示词构建

**基础系统提示** (`src/constants/prompts.ts:16`):
```typescript
export async function getSystemPrompt(): Promise<string[]> {
  return [
    `You are an interactive CLI tool that helps users with software engineering tasks.
    
IMPORTANT: Refuse to write code or explain code that may be used maliciously...

# Memory
If the current working directory contains a file called CLAUDE.md, it will be automatically added to your context. This file serves multiple purposes:
1. Storing frequently used bash commands (build, test, lint, etc.)
2. Recording the user's code style preferences
3. Maintaining useful information about the codebase structure...

# Tone and style
You should be concise, direct, and to the point...`,
    
    // 其他系统提示段落...
  ]
}
```

### 3.2 上下文整合机制

**关键函数**: `src/services/claude.ts:426`
```typescript
export function formatSystemPromptWithContext(
  systemPrompt: string[],
  context: { [k: string]: string },
): string[] {
  if (Object.entries(context).length === 0) {
    return systemPrompt
  }

  return [
    ...systemPrompt,
    `\nAs you answer the user's questions, you can use the following context:\n`,
    ...Object.entries(context).map(
      ([key, value]) => `<context name="${key}">${value}</context>`,
    ),
  ]
}
```

### 3.3 最终Prompt组装

**查询处理流程** (`src/query.ts:135`):
```typescript
export async function* query(
  messages: Message[],
  systemPrompt: string[],
  context: { [k: string]: string },
  // ...
): AsyncGenerator<Message, void> {
  
  // 1. 整合系统提示和上下文
  const fullSystemPrompt = formatSystemPromptWithContext(systemPrompt, context)
  
  // 2. 调用LLM API
  const assistantMessage = await querySonnet(
    normalizeMessagesForAPI(messages),  // 历史消息
    fullSystemPrompt,                   // 系统提示+上下文
    toolUseContext.options.maxThinkingTokens,
    toolUseContext.options.tools,      // 可用工具定义
    toolUseContext.abortController.signal
  )
  
  // ...
}
```

**API调用时的最终组装** (`src/services/claude.ts:443`):
```typescript
async function querySonnetWithPromptCaching(
  messages: (UserMessage | AssistantMessage)[],
  systemPrompt: string[],
  maxThinkingTokens: number,
  tools: Tool[],
  // ...
): Promise<AssistantMessage> {
  
  // 系统提示分块处理(用于prompt caching)
  const system: TextBlockParam[] = splitSysPromptPrefix(systemPrompt).map(_ => ({
    ...(PROMPT_CACHING_ENABLED ? { cache_control: { type: 'ephemeral' } } : {}),
    text: _,
    type: 'text',
  }))

  // 工具定义生成
  const toolSchemas = await Promise.all(
    tools.map(async _ => ({
      name: _.name,
      description: await _.prompt({dangerouslySkipPermissions: options.dangerouslySkipPermissions}),
      input_schema: zodToJsonSchema(_.inputSchema) as Anthropic.Tool.InputSchema,
    }))
  )

  // 最终API调用
  const response = await anthropic.beta.messages.stream({
    model: options.model,
    max_tokens: Math.max(maxThinkingTokens + 1, getMaxTokensForModel(options.model)),
    messages: addCacheBreakpoints(messages),  // 历史消息+缓存优化
    temperature: MAIN_QUERY_TEMPERATURE,
    system,                                   // 系统提示+上下文
    tools: toolSchemas,                       // 工具定义
    // ...
  })
}
```

---

## 4. 🗜️ /compact 压缩机制深度分析

### 4.1 压缩触发机制

**Token警告系统** (`src/components/TokenWarning.tsx:9-11`):
```typescript
const MAX_TOKENS = 190_000        // 最大token限制(留出/compact的余量)
export const WARNING_THRESHOLD = MAX_TOKENS * 0.6  // 60% 警告阈值
const ERROR_THRESHOLD = MAX_TOKENS * 0.8            // 80% 错误阈值

export function TokenWarning({ tokenUsage }: Props): React.ReactNode {
  // ...
  return (
    <Text color={isError ? theme.error : theme.warning}>
      Context low ({Math.max(0, 100 - Math.round((tokenUsage / MAX_TOKENS) * 100))}% remaining) 
      &middot; Run /compact to compact & continue
    </Text>
  )
}
```

### 4.2 压缩处理流程

**核心函数**: `src/commands/compact.ts:18`
```typescript
async call(_, { options: { tools, slowAndCapableModel }, abortController, setForkConvoWithMessagesOnTheNextRender }) {
  
  // 1. 获取当前所有消息
  const messages = getMessagesGetter()()

  // 2. 创建摘要请求消息
  const summaryRequest = createUserMessage(
    "Provide a detailed but concise summary of our conversation above. " +
    "Focus on information that would be helpful for continuing the conversation, " +
    "including what we did, what we're doing, which files we're working on, " +
    "and what we're going to do next."
  )

  // 3. 调用LLM生成摘要
  const summaryResponse = await querySonnet(
    normalizeMessagesForAPI([...messages, summaryRequest]),
    ['You are a helpful AI assistant tasked with summarizing conversations.'],  // 专用系统提示
    0,                    // 不使用thinking tokens
    tools,
    abortController.signal,
    {
      dangerouslySkipPermissions: false,
      model: slowAndCapableModel,
      prependCLISysprompt: true,
    }
  )

  // 4. 提取摘要内容
  const summary = typeof content === 'string'
    ? content
    : content.length > 0 && content[0]?.type === 'text'
      ? content[0].text
      : null

  // 5. 重置token使用量(欺骗UI显示)
  summaryResponse.message.usage = {
    input_tokens: 0,      // 重置为0
    output_tokens: summaryResponse.message.usage.output_tokens,
    cache_creation_input_tokens: 0,
    cache_read_input_tokens: 0,
  }

  // 6. 清理并重置会话
  await clearTerminal()
  getMessagesSetter()([])  // 清空消息历史
  
  // 7. 用摘要启动新会话
  setForkConvoWithMessagesOnTheNextRender([
    createUserMessage(`Use the /compact command to clear the conversation history, and start a new conversation with the summary in context.`),
    summaryResponse,  // 摘要作为AI回复
  ])
  
  // 8. 清理缓存
  getContext.cache.clear?.()
  getCodeStyle.cache.clear?.()
}
```

### 4.3 压缩使用的Prompt

**摘要生成的具体Prompt**:
```
用户消息: "Provide a detailed but concise summary of our conversation above. Focus on information that would be helpful for continuing the conversation, including what we did, what we're doing, which files we're working on, and what we're going to do next."

系统提示: "You are a helpful AI assistant tasked with summarizing conversations."
```

### 4.4 摘要替换机制

压缩过程实际上是**完全替换**而非追加:
1. 保存当前完整对话历史到变量
2. 生成对话摘要
3. **完全清空**当前消息数组
4. 创建**新的对话开始**，包含：
   - 用户消息: 说明使用了/compact命令
   - AI消息: 包含完整的对话摘要

---

## 5. 🪟 窗口管理与Token检查机制

### 5.1 Token计算机制

**核心函数**: `src/utils/tokens.ts:4`
```typescript
export function countTokens(messages: Message[]): number {
  let i = messages.length - 1
  // 从最新消息向前查找，找到第一个包含usage信息的assistant消息
  while (i >= 0) {
    const message = messages[i]
    if (
      message?.type === 'assistant' &&
      'usage' in message.message &&
      !(
        message.message.content[0]?.type === 'text' &&
        SYNTHETIC_ASSISTANT_MESSAGES.has(message.message.content[0].text)  // 排除合成消息
      )
    ) {
      const { usage } = message.message
      return (
        usage.input_tokens +                          // 输入token
        (usage.cache_creation_input_tokens ?? 0) +   // 缓存创建token
        (usage.cache_read_input_tokens ?? 0) +       // 缓存读取token
        usage.output_tokens                          // 输出token
      )
    }
    i--
  }
  return 0
}
```

### 5.2 实时Token监控

**PromptInput中的监控** (`src/components/PromptInput.tsx:375 & 448`):
```typescript
// 在两个位置显示TokenWarning
<TokenWarning tokenUsage={tokenUsage} />
<TokenWarning tokenUsage={countTokens(messages)} />
```

**触发显示的条件**:
```typescript
// src/components/TokenWarning.tsx:16
if (tokenUsage < WARNING_THRESHOLD) {  // 低于60%不显示
  return null
}

const isError = tokenUsage >= ERROR_THRESHOLD  // 80%以上显示错误色
```

### 5.3 窗口管理策略

**没有自动截断机制** - Claude Code采用的是**用户主动压缩**策略:

1. **被动监控**: 实时计算和显示token使用率
2. **用户提醒**: 达到60%时显示警告，80%时显示错误
3. **主动压缩**: 用户手动执行`/compact`进行压缩
4. **完全重置**: 压缩时完全替换历史，而非截断

**设计考量**:
- 避免自动丢失重要信息
- 让用户掌控压缩时机
- 通过摘要保持上下文连续性

### 5.4 缓存优化机制

**Prompt Caching** (`src/services/claude.ts:642`):
```typescript
function addCacheBreakpoints(messages: (UserMessage | AssistantMessage)[]): MessageParam[] {
  return messages.map((msg, index) => {
    // 只为最近的3条消息启用缓存
    return msg.type === 'user'
      ? userMessageToMessageParam(msg, index > messages.length - 3)
      : assistantMessageToMessageParam(msg, index > messages.length - 3)
  })
}
```

这种设计显著减少了重复内容的token消耗，特别是在长对话场景中。

---

## 🔍 关键代码段位置索引

| 功能模块 | 关键函数/变量 | 文件位置 |
|---------|-------------|----------|
| 消息类型定义 | `Message`, `UserMessage`, `AssistantMessage` | `src/query.ts:35-66` |
| 消息存储数组 | `messages` state | `src/screens/REPL.tsx:127` |
| 上下文收集 | `getContext()` | `src/context.ts:157` |
| Prompt构建 | `formatSystemPromptWithContext()` | `src/services/claude.ts:426` |
| 压缩处理 | `/compact` command | `src/commands/compact.ts:18` |
| Token计算 | `countTokens()` | `src/utils/tokens.ts:4` |
| 窗口警告 | `TokenWarning` | `src/components/TokenWarning.tsx:13` |
| 缓存优化 | `addCacheBreakpoints()` | `src/services/claude.ts:642` |

---

## 💡 工程实践启示

### 1. 记忆系统设计原则

- **渐进式存储**: 消息数组追加模式，保持完整历史
- **智能上下文**: 自动收集项目信息，减少用户输入负担
- **用户控制**: 压缩时机由用户决定，避免信息丢失
- **成本感知**: 实时token监控，帮助用户控制API成本

### 2. 性能优化策略

- **Memoization**: 上下文信息缓存，避免重复计算
- **Prompt Caching**: API级别的内容缓存，减少token消耗
- **并行获取**: 上下文信息并行收集，提升响应速度
- **选择性注入**: 根据需要动态注入文件内容

### 3. 架构设计亮点

- **状态分离**: React状态管理与业务逻辑分离
- **异步流式**: 支持长对话的流式处理
- **错误恢复**: 多层错误处理和用户友好提示
- **扩展性**: 插件化的上下文注入机制

这套记忆系统设计为构建大型语言模型应用提供了完整的参考实现，特别适合需要长期对话记忆和智能上下文管理的AI Agent系统。