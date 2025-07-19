# Claude Code 上下文感知的提示词策略分析

## 1. 上下文感知架构概览

Claude Code实现了多维度的上下文感知系统，通过自动收集、智能分析和动态注入项目信息，为AI提供丰富的环境认知能力。

## 2. 项目环境上下文感知

### 2.1 Git仓库状态感知
**文件位置**: `src/context.ts:85-152`

```typescript
export const getGitStatus = memoize(async (): Promise<string | null> => {
  if (!(await getIsGit())) {
    return null  // 非Git仓库直接返回
  }

  try {
    // 🎯 并行获取多维Git信息
    const [branch, mainBranch, status, log, authorLog] = await Promise.all([
      execFileNoThrow('git', ['branch', '--show-current']),          // 当前分支
      execFileNoThrow('git', ['rev-parse', '--abbrev-ref', 'origin/HEAD']), // 主分支
      execFileNoThrow('git', ['status', '--short']),                 // 工作区状态
      execFileNoThrow('git', ['log', '--oneline', '-n', '5']),       // 最近提交
      execFileNoThrow('git', ['log', '--oneline', '-n', '5', '--author', gitEmail]), // 用户提交
    ])

    // 🎯 长状态截断处理
    const statusLines = status.split('\n').length
    const truncatedStatus = statusLines > 200
      ? status.split('\n').slice(0, 200).join('\n') + 
        '\n... (truncated because there are more than 200 lines)'
      : status

    // 🎯 结构化Git上下文信息
    return `This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.
Current branch: ${branch}

Main branch (you will usually use this for PRs): ${mainBranch}

Status:
${truncatedStatus || '(clean)'}

Recent commits:
${log}

Your recent commits:
${authorLog || '(no recent commits)'}`
  } catch (error) {
    return null
  }
})
```

**Git上下文设计亮点**:
- **完整分支信息**: 当前分支+主分支，为PR创建提供基础
- **状态快照**: 明确说明这是对话开始时的状态
- **双重提交历史**: 区分所有提交和用户提交
- **长度保护**: 超过200行的状态自动截断
- **用户友好**: 为空状态提供"(clean)"描述

### 2.2 项目结构智能分析
**文件位置**: `src/context.ts:186-224`

```typescript
export const getDirectoryStructure = memoize(async function (): Promise<string> {
  let lines: string
  try {
    const abortController = new AbortController()
    setTimeout(() => abortController.abort(), 1_000)  // 🎯 1秒超时保护
    
    const model = await getSlowAndCapableModel()
    const resultsGen = LSTool.call({ path: '.' }, {
      abortController,
      options: {
        commands: [], tools: [], slowAndCapableModel: model,
        forkNumber: 0, messageLogName: 'unused', maxThinkingTokens: 0,
      },
      messageId: undefined,
      readFileTimestamps: {},
    })
    
    const result = await lastX(resultsGen)
    lines = result.data
  } catch (error) {
    logError(error)
    return ''
  }

  return `Below is a snapshot of this project's file structure at the start of the conversation. This snapshot will NOT update during the conversation.

${lines}`
})
```

**项目结构感知策略**:
- **工具复用**: 使用LSTool获取目录结构
- **超时保护**: 1秒超时避免阻塞启动
- **静态快照**: 明确说明不会实时更新
- **错误容忍**: 失败时返回空字符串而不是报错

### 2.3 代码风格自动推断
**文件位置**: `src/utils/style.ts` (推断位置)

```typescript
// 基于现有代码自动推断项目风格
export const getCodeStyle = (): string | null => {
  // 分析项目中的代码文件
  // 推断缩进风格、命名约定、库使用等
  // 返回风格描述字符串
}
```

## 3. 文档和配置上下文感知

### 3.1 README.md自动整合
**文件位置**: `src/context.ts:71-83`

```typescript
export const getReadme = memoize(async (): Promise<string | null> => {
  try {
    const readmePath = join(getCwd(), 'README.md')
    if (!existsSync(readmePath)) {
      return null
    }
    const content = await readFile(readmePath, 'utf-8')
    return content
  } catch (e) {
    logError(e)
    return null
  }
})
```

### 3.2 CLAUDE.md项目记忆文件
**文件位置**: `src/context.ts:24-48`

```typescript
export async function getClaudeFiles(): Promise<string | null> {
  const abortController = new AbortController()
  const timeout = setTimeout(() => abortController.abort(), 3000)
  
  try {
    // 🎯 搜索所有CLAUDE.md文件
    const files = await ripGrep(
      ['--files', '--glob', join('**', '*', 'CLAUDE.md')],
      getCwd(),
      abortController.signal,
    )
    
    if (!files.length) {
      return null
    }

    // 🎯 生成多文件指导信息
    return `NOTE: Additional CLAUDE.md files were found. When working in these directories, make sure to read and follow the instructions in the corresponding CLAUDE.md file:
${files.map(_ => path.join(getCwd(), _)).map(_ => `- ${_}`).join('\n')}`
  } catch (error) {
    logError(error)
    return null
  } finally {
    clearTimeout(timeout)
  }
}
```

**CLAUDE.md感知特点**:
- **递归搜索**: 查找所有子目录中的CLAUDE.md文件
- **绝对路径**: 提供完整文件路径便于访问
- **批量指导**: 为多个CLAUDE.md文件提供使用指导
- **超时保护**: 3秒超时避免长时间搜索

## 4. 用户行为上下文感知

### 4.1 Git历史行为分析
**文件位置**: `src/utils/exampleCommands.ts:18-62`

```typescript
async function getFrequentlyModifiedFiles(): Promise<string[]> {
  try {
    // 🎯 分析用户最近1000次提交的文件修改模式
    const { stdout: userFilenames } = await execPromise(
      'git log -n 1000 --pretty=format: --name-only --diff-filter=M --author=$(git config user.email) | sort | uniq -c | sort -nr | head -n 20'
    )

    let filenames = 'Files modified by user:\n' + userFilenames

    // 🎯 如果用户提交不足，补充其他用户的提交
    if (userFilenames.split('\n').length < 10) {
      const { stdout: allFilenames } = await execPromise(
        'git log -n 1000 --pretty=format: --name-only --diff-filter=M | sort | uniq -c | sort -nr | head -n 20'
      )
      filenames += '\n\nFiles modified by other users:\n' + allFilenames
    }

    // 🎯 使用AI分析提取核心文件
    const response = await queryHaiku({
      systemPrompt: [
        "You are an expert at analyzing git history. Given a list of files and their modification counts, return exactly five filenames that are frequently modified and represent core application logic (not auto-generated files, dependencies, or configuration). Make sure filenames are diverse, not all in the same folder, and are a mix of user and other users. Return only the filenames' basenames (without the path) separated by newlines with no explanation."
      ],
      userPrompt: filenames,
    })

    const chosenFilenames = content.text.trim().split('\n')
    return chosenFilenames.length >= 5 ? chosenFilenames : []
  } catch (err) {
    logError(err)
    return []
  }
}
```

**用户行为分析策略**:
- **个人化优先**: 优先分析用户自己的提交记录
- **补充策略**: 个人提交不足时加入团队记录
- **AI筛选**: 使用LLM过滤出核心业务文件
- **多样性保证**: 确保文件来自不同目录
- **智能排除**: 自动排除配置文件和依赖

### 4.2 启动频率统计
**文件位置**: `src/utils/exampleCommands.ts:76-81`

```typescript
// 🎯 追踪用户使用频率
const newGlobalConfig = {
  ...globalConfig,
  numStartups: (globalConfig.numStartups ?? 0) + 1,  // 启动次数统计
}
saveGlobalConfig(newGlobalConfig)
```

## 5. 环境运行时上下文感知

### 5.1 实时环境信息注入
**文件位置**: `src/constants/prompts.ts:129-142`

```typescript
export async function getEnvInfo(): Promise<string> {
  const [model, isGit] = await Promise.all([
    getSlowAndCapableModel(),  // 🎯 当前使用的模型
    getIsGit(),               // 🎯 Git仓库检测
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

### 5.2 平台特定适配
```typescript
// Windows平台特殊处理
if (env.platform === 'windows') return []

// 测试环境跳过
if (process.env.NODE_ENV === 'test') return []
```

## 6. 配置驱动的上下文控制

### 6.1 用户配置感知
**文件位置**: `src/context.ts:157-180`

```typescript
export const getContext = memoize(async (): Promise<{[k: string]: string}> => {
  const projectConfig = getCurrentProjectConfig()
  const dontCrawl = projectConfig.dontCrawlDirectory  // 🎯 用户控制的抓取开关
  
  const [gitStatus, directoryStructure, claudeFiles, readme] = await Promise.all([
    getGitStatus(),
    dontCrawl ? Promise.resolve('') : getDirectoryStructure(),  // 🎯 条件执行
    dontCrawl ? Promise.resolve('') : getClaudeFiles(),
    getReadme(),
  ])
  
  return {
    ...projectConfig.context,          // 🎯 用户自定义上下文优先
    ...(directoryStructure ? { directoryStructure } : {}),
    ...(gitStatus ? { gitStatus } : {}),
    ...(codeStyle ? { codeStyle } : {}),
    ...(claudeFiles ? { claudeFiles } : {}),
    ...(readme ? { readme } : {}),
  }
})
```

**配置控制特点**:
- **用户优先**: 自定义上下文具有最高优先级
- **选择性抓取**: `dontCrawlDirectory`控制是否分析项目结构
- **条件执行**: 基于配置决定是否执行耗时操作
- **合并策略**: 自动合并多个上下文源

## 7. 上下文格式化和结构化

### 7.1 XML标签结构化
```xml
<env>
Working directory: /path/to/project
Is directory a git repo: Yes
Platform: darwin
Today's date: 1/19/2025
Model: claude-3-5-sonnet-20241022
</env>

<context name="gitStatus">
Current branch: feature/new-tool
Status: M src/tools/NewTool.tsx
</context>

<context name="directoryStructure">
Below is a snapshot of this project's file structure...
</context>
```

### 7.2 时序性说明
```
This is the git status at the start of the conversation. 
Note that this status is a snapshot in time, and will not update during the conversation.
```

**时序感知设计**:
- **快照性质**: 明确说明信息的时效性
- **更新策略**: 说明哪些信息会/不会更新
- **用户预期**: 帮助用户理解信息的局限性

## 8. 上下文缓存和性能优化

### 8.1 多层缓存策略
```typescript
// 不同粒度的缓存
export const getContext = memoize(async () => {...})          // 会话级
export const getGitStatus = memoize(async () => {...})       // 项目级
export const getDirectoryStructure = memoize(async () => {...}) // 项目级
export const getReadme = memoize(async () => {...})          // 文件级
```

### 8.2 缓存失效机制
```typescript
// /compact命令后清除缓存
getContext.cache.clear?.()
getCodeStyle.cache.clear?.()

// 基于时间的自动失效
const oneWeek = 7 * 24 * 60 * 60 * 1000
if (now - lastGenerated > oneWeek) {
  projectConfig.exampleFiles = []
}
```

## 9. 上下文感知的应用场景

### 9.1 个性化示例生成
```typescript
// 基于项目文件生成个性化示例
const frequentFile = projectConfig.exampleFiles?.length
  ? sample(projectConfig.exampleFiles)
  : '<filepath>'

return [
  `how does ${frequentFile} work?`,      // 🎯 项目特定文件
  `refactor ${frequentFile}`,            // 🎯 基于用户行为
  `write a test for ${frequentFile}`,    // 🎯 上下文相关任务
]
```

### 9.2 智能工具选择
```typescript
// 基于项目类型选择合适的工具
if (isJupyterNotebook(file)) {
  return NotebookEditTool
} else {
  return FileEditTool
}
```

### 9.3 自适应安全策略
```typescript
// 基于环境调整安全策略
if (process.env.NODE_ENV === 'production') {
  return restrictedTools
} else {
  return allTools
}
```

## 10. 上下文感知设计原则

### 10.1 渐进增强原则
- **基础功能**: 无上下文时的基本功能
- **增强体验**: 有上下文时的丰富体验
- **优雅降级**: 上下文缺失时的备选方案

### 10.2 用户控制原则
- **可配置**: 用户可以控制上下文收集范围
- **可见性**: 用户了解收集了哪些信息
- **可关闭**: 重要功能可以禁用

### 10.3 性能平衡原则
- **异步收集**: 不阻塞主要功能
- **智能缓存**: 避免重复收集
- **超时保护**: 防止长时间等待

### 10.4 隐私保护原则
- **本地处理**: 敏感信息不离开本地
- **最小收集**: 只收集必要的上下文
- **用户同意**: 重要信息收集需要用户确认

这种多维度的上下文感知系统使Claude Code能够提供高度个性化和环境适应的AI交互体验，同时保持良好的性能和隐私保护。