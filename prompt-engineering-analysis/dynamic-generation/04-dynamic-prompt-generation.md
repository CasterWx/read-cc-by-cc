# Claude Code 动态提示词生成机制分析

## 1. 动态提示词生成架构概览

Claude Code采用多层次的动态提示词生成策略，从简单的参数插值到复杂的AI驱动内容生成，实现了高度自适应的提示词系统。

## 2. AI生成AI提示词模式

### 2.1 BashTool的智能描述生成
**文件位置**: `src/tools/BashTool/BashTool.tsx:39-68`

```typescript
async description({ command }) {
  try {
    const result = await queryHaiku({
      systemPrompt: [
        `You are a command description generator. Write a clear, concise description of what this command does in 5-10 words. Examples:

        Input: ls
        Output: Lists files in current directory

        Input: git status
        Output: Shows working tree status

        Input: npm install
        Output: Installs package dependencies

        Input: mkdir foo
        Output: Creates directory 'foo'`,
      ],
      userPrompt: `Describe this command: ${command}`,
    })
    
    const description = result.message.content[0]?.type === 'text'
      ? result.message.content[0].text
      : null
    return description || 'Executes a bash command'
  } catch (error) {
    logError(error)
    return 'Executes a bash command'  // 🎯 降级处理
  }
}
```

**设计创新点**:
1. **Meta-AI架构**: 使用Haiku为Sonnet生成工具描述
2. **Few-shot模板**: 4个标准示例确保输出格式一致
3. **长度约束**: 强制5-10词简洁描述
4. **容错机制**: API失败时使用默认描述
5. **实时生成**: 每次工具调用时动态生成

### 2.2 智能文件筛选系统
**文件位置**: `src/utils/exampleCommands.ts:43-48`

```typescript
// 使用AI筛选最重要的文件
const response = await queryHaiku({
  systemPrompt: [
    "You are an expert at analyzing git history. Given a list of files and their modification counts, return exactly five filenames that are frequently modified and represent core application logic (not auto-generated files, dependencies, or configuration). Make sure filenames are diverse, not all in the same folder, and are a mix of user and other users. Return only the filenames' basenames (without the path) separated by newlines with no explanation."
  ],
  userPrompt: filenames,
})
```

**AI筛选策略**:
- **领域专家角色**: "expert at analyzing git history"
- **具体约束**: 精确5个文件，排除自动生成文件
- **多样性要求**: 不同文件夹，混合用户来源
- **输出格式**: 仅文件名，无解释

## 3. 上下文驱动的动态生成

### 3.1 环境信息动态注入
**文件位置**: `src/constants/prompts.ts:129-142`

```typescript
export async function getEnvInfo(): Promise<string> {
  const [model, isGit] = await Promise.all([
    getSlowAndCapableModel(),
    getIsGit(),
  ])
  return `Here is useful information about the environment you are running in:
<env>
Working directory: ${getCwd()}
Is directory a git repo: ${isGit ? 'Yes' : 'No'}
Platform: ${env.platform}
Today's date: ${new Date().toLocaleDateString()}
Model: ${model}
</env>`
}
```

**动态元素**:
- **实时工作目录**: `getCwd()`
- **Git状态检测**: 异步检查当前目录
- **平台信息**: 运行时获取OS类型
- **时间戳**: 当前日期
- **模型信息**: 当前使用的LLM模型

### 3.2 项目特定上下文生成
**文件位置**: `src/context.ts:157-180`

```typescript
export const getContext = memoize(async (): Promise<{[k: string]: string}> => {
  const codeStyle = getCodeStyle()
  const projectConfig = getCurrentProjectConfig()
  const dontCrawl = projectConfig.dontCrawlDirectory
  
  // 🎯 并行获取所有上下文信息
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

**智能上下文收集**:
- **Git分析**: 自动获取分支、状态、最近提交
- **目录结构**: 使用LSTool生成项目概览
- **代码风格**: 分析现有代码推断风格偏好
- **文档整合**: 自动读取README和CLAUDE.md
- **用户配置**: 合并用户自定义上下文

## 4. 条件化提示词生成

### 4.1 权限感知的AgentTool提示词
**文件位置**: `src/tools/AgentTool/prompt.ts:20-41`

```typescript
export async function getPrompt(
  dangerouslySkipPermissions: boolean,
): Promise<string> {
  const tools = await getAgentTools(dangerouslySkipPermissions)  // 🎯 动态工具列表
  const toolNames = tools.map(_ => _.name).join(', ')
  
  return `Launch a new agent that has access to the following tools: ${toolNames}.
  
  Usage notes:
  4. The agent's outputs should generally be trusted${
    dangerouslySkipPermissions ? '' : `
  5. IMPORTANT: The agent can not use ${BashTool.name}, ${FileWriteTool.name}, ${FileEditTool.name}, ${NotebookEditTool.name}, so can not modify files. If you want to use these tools, use them directly instead of going through the agent.`
  }`
}
```

**条件化策略**:
- **权限模式检测**: 根据`dangerouslySkipPermissions`调整
- **工具列表生成**: 动态获取可用工具并生成列表
- **约束条件**: 安全模式下添加工具限制说明
- **灵活性**: 同一工具在不同模式下有不同行为

### 4.2 工具可用性动态检测
**文件位置**: `src/tools/AgentTool/prompt.ts:11-18`

```typescript
export async function getAgentTools(
  dangerouslySkipPermissions: boolean,
): Promise<Tool[]> {
  // 🎯 根据权限模式选择工具集
  return (
    await (dangerouslySkipPermissions ? getTools() : getReadOnlyTools())
  ).filter(_ => _.name !== AgentTool.name)  // 防止递归Agent
}
```

## 5. 时序感知的动态更新

### 5.1 定期更新的示例命令
**文件位置**: `src/utils/exampleCommands.ts:64-94`

```typescript
export const getExampleCommands = memoize(async (): Promise<string[]> => {
  const projectConfig = getCurrentProjectConfig()
  const now = Date.now()
  const lastGenerated = projectConfig.exampleFilesGeneratedAt ?? 0
  const oneWeek = 7 * 24 * 60 * 60 * 1000
  
  // 🎯 时效性检查：超过一周则重新生成
  if (now - lastGenerated > oneWeek) {
    projectConfig.exampleFiles = []
  }
  
  // 🎯 后台异步更新，不阻塞用户交互
  if (!projectConfig.exampleFiles?.length) {
    getFrequentlyModifiedFiles().then(files => {
      if (files.length) {
        saveCurrentProjectConfig({
          ...getCurrentProjectConfig(),
          exampleFiles: files,
          exampleFilesGeneratedAt: Date.now(),
        })
      }
    })
  }
  
  // 🎯 基于分析结果生成个性化示例
  const frequentFile = projectConfig.exampleFiles?.length
    ? sample(projectConfig.exampleFiles)  // 随机选择常用文件
    : '<filepath>'
  
  return [
    'fix lint errors',
    'fix typecheck errors',
    `how does ${frequentFile} work?`,      // 个性化建议
    `refactor ${frequentFile}`,
    `edit ${frequentFile} to...`,
    `write a test for ${frequentFile}`,
  ]
})
```

**时序更新策略**:
- **缓存机制**: memoize避免重复计算
- **时效检查**: 7天更新周期
- **异步更新**: 后台更新不影响用户体验
- **随机化**: 每次随机选择不同的常用文件

## 6. 会话状态驱动生成

### 6.1 compact命令的动态摘要生成
**文件位置**: `src/commands/compact.ts:30-32`

```typescript
// 🎯 基于当前对话历史动态生成摘要请求
const summaryRequest = createUserMessage(
  "Provide a detailed but concise summary of our conversation above. Focus on information that would be helpful for continuing the conversation, including what we did, what we're doing, which files we're working on, and what we're going to do next."
)

// 🎯 使用专门的摘要系统提示词
const summaryResponse = await querySonnet(
  normalizeMessagesForAPI([...messages, summaryRequest]),
  ['You are a helpful AI assistant tasked with summarizing conversations.'],
  0, tools, abortController.signal,
  { dangerouslySkipPermissions: false, model: slowAndCapableModel, prependCLISysprompt: true }
)
```

**会话感知特点**:
- **历史分析**: 基于完整对话历史
- **未来导向**: 关注正在进行和计划的任务
- **文件跟踪**: 特别关注正在操作的文件
- **专用模型**: 使用专门的摘要提示词

## 7. 多层缓存和优化策略

### 7.1 分层缓存机制
```typescript
// 不同层级的缓存策略
export const getContext = memoize(async () => {...})          // 会话级缓存
export const getExampleCommands = memoize(async () => {...})  // 全局缓存
export const getDirectoryStructure = memoize(async () => {...}) // 项目级缓存
```

### 7.2 智能缓存失效
```typescript
// 基于时间的缓存失效
const oneWeek = 7 * 24 * 60 * 60 * 1000
if (now - lastGenerated > oneWeek) {
  projectConfig.exampleFiles = []  // 清除缓存
}

// 基于操作的缓存清除
getContext.cache.clear?.()    // /compact命令后清除上下文缓存
getCodeStyle.cache.clear?.()  // 清除代码风格缓存
```

## 8. 错误处理和降级策略

### 8.1 多层降级机制
```typescript
// BashTool描述生成的降级策略
async description({ command }) {
  try {
    const result = await queryHaiku({...})
    const description = result.message.content[0]?.type === 'text'
      ? result.message.content[0].text
      : null
    return description || 'Executes a bash command'  // 第一层降级
  } catch (error) {
    logError(error)
    return 'Executes a bash command'  // 第二层降级
  }
}
```

### 8.2 容错性设计
```typescript
// 文件列表获取的容错处理
async function getFrequentlyModifiedFiles(): Promise<string[]> {
  if (process.env.NODE_ENV === 'test') return []     // 测试环境跳过
  if (env.platform === 'windows') return []          // Windows平台跳过
  if (!(await getIsGit())) return []                 // 非Git仓库跳过
  
  try {
    // 复杂的Git分析逻辑
  } catch (err) {
    logError(err)
    return []  // 错误时返回空数组
  }
}
```

## 9. 性能优化策略

### 9.1 并行处理
```typescript
// 并行获取多个上下文信息
const [gitStatus, directoryStructure, claudeFiles, readme] = await Promise.all([
  getGitStatus(),
  dontCrawl ? Promise.resolve('') : getDirectoryStructure(),
  dontCrawl ? Promise.resolve('') : getClaudeFiles(),
  getReadme(),
])
```

### 9.2 智能跳过
```typescript
// 基于配置的智能跳过
const dontCrawl = projectConfig.dontCrawlDirectory
const [gitStatus, directoryStructure, claudeFiles, readme] = await Promise.all([
  getGitStatus(),
  dontCrawl ? Promise.resolve('') : getDirectoryStructure(),  // 🎯 条件执行
  dontCrawl ? Promise.resolve('') : getClaudeFiles(),
  getReadme(),
])
```

## 10. 动态提示词生成的设计模式

### 10.1 模板+参数模式
- **基础模板**: 固定的提示词结构
- **动态参数**: 运行时插入的变量
- **条件分支**: 基于状态的内容选择

### 10.2 AI驱动生成模式
- **Meta-AI**: 小模型为大模型生成内容
- **专家角色**: 为AI分配特定领域专家身份
- **结构化输出**: 严格的输出格式要求

### 10.3 上下文感知模式
- **环境检测**: 自动感知运行环境
- **项目分析**: 基于项目特征调整行为
- **历史学习**: 从用户行为中学习偏好

### 10.4 渐进增强模式
- **基础功能**: 无需额外信息的基本功能
- **增强信息**: 有额外信息时的增强体验
- **优雅降级**: 信息缺失时的备选方案

这种多层次的动态提示词生成机制使Claude Code能够提供高度个性化、上下文相关的AI交互体验，同时保持系统的稳定性和性能。