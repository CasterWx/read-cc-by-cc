# Claude Code 用户提示词优化技巧分析

## 1. 用户提示词优化策略概览

Claude Code通过多层优化策略帮助用户更好地与AI交互，包括响应格式优化、示例驱动学习、动态提示建议等。

## 2. 响应格式优化 (CLI特化)

### 2.1 强制简洁性约束
**文件位置**: `src/constants/prompts.ts:42-44`

```typescript
// 核心约束指令
`IMPORTANT: Keep your responses short, since they will be displayed on a command line interface. 
You MUST answer concisely with fewer than 4 lines (not including tool use or code generation), 
unless user asks for detail. One word answers are best.`
```

**优化技巧**:
- **行数限制**: 强制4行以内回复
- **单词优先**: "One word answers are best"
- **条件例外**: "unless user asks for detail"

### 2.2 禁止冗余表达
```typescript
// 禁止套话模式
`You MUST avoid text before/after your response, such as:
- "The answer is <answer>."
- "Here is the content of the file..." 
- "Based on the information provided, the answer is..."
- "Here is what I will do next..."`
```

**反模式识别**:
- **前置套话**: 避免"答案是..."类开头
- **后置解释**: 避免"基于...，答案是..."
- **过程描述**: 避免"我将要做..."类说明

## 3. Few-Shot示例驱动优化

### 3.1 问答简洁性示例
**文件位置**: `src/constants/prompts.ts:45-82`

```xml
<example>
user: 2 + 2
assistant: 4
</example>

<example>
user: what is 2+2?
assistant: 4
</example>

<example>
user: is 11 a prime number?
assistant: true
</example>
```

**示例设计原则**:
- **一致性**: 相同问题的不同表达都给出相同简洁回答
- **布尔简化**: 是非问题直接回答true/false
- **数值直接**: 计算问题直接给数字

### 3.2 工具使用示例
```xml
<example>
user: what files are in the directory src/?
assistant: [runs ls and sees foo.c, bar.c, baz.c]
user: which file contains the implementation of foo?
assistant: src/foo.c
</example>

<example>
user: write tests for new feature
assistant: [uses grep and glob search tools to find where similar tests are defined, 
uses concurrent read file tool use blocks in one tool call to read relevant files, 
uses edit file tool to write new tests]
</example>
```

**工具使用指导**:
- **工具链思维**: 展示多工具协同使用
- **并发优化**: 演示同时调用多个工具
- **上下文利用**: 基于现有代码模式编写新代码

## 4. 动态示例命令生成

### 4.1 智能示例命令系统
**文件位置**: `src/utils/exampleCommands.ts:64`

```typescript
export const getExampleCommands = memoize(async (): Promise<string[]> => {
  // 1. 获取频繁修改的文件
  const frequentFile = projectConfig.exampleFiles?.length
    ? sample(projectConfig.exampleFiles)  // 随机选择一个常用文件
    : '<filepath>'
  
  // 2. 生成个性化示例
  return [
    'fix lint errors',
    'fix typecheck errors', 
    `how does ${frequentFile} work?`,      // 🎯 基于用户项目
    `refactor ${frequentFile}`,            // 🎯 个性化建议
    'how do I log an error?',
    `edit ${frequentFile} to...`,          // 🎯 上下文相关
    `write a test for ${frequentFile}`,    // 🎯 项目特定
    'create a util logging.py that...',
  ]
})
```

### 4.2 Git历史分析生成个性化建议
```typescript
// 分析用户最常修改的文件
async function getFrequentlyModifiedFiles(): Promise<string[]> {
  // 1. 获取用户最近1000次提交的修改文件
  const { stdout: userFilenames } = await execPromise(
    'git log -n 1000 --pretty=format: --name-only --diff-filter=M --author=$(git config user.email) | sort | uniq -c | sort -nr | head -n 20'
  )
  
  // 2. 使用Haiku模型智能筛选核心文件
  const response = await queryHaiku({
    systemPrompt: [
      "You are an expert at analyzing git history. Return exactly five filenames that are frequently modified and represent core application logic (not auto-generated files, dependencies, or configuration). Return only the filenames' basenames (without the path) separated by newlines with no explanation."
    ],
    userPrompt: filenames,
  })
  
  return chosenFilenames
}
```

**个性化策略**:
- **用户行为分析**: 基于Git提交历史
- **AI筛选**: 使用LLM过滤出核心文件
- **项目感知**: 排除配置文件和依赖
- **时效性**: 每周更新一次文件列表

## 5. 上下文感知提示优化

### 5.1 环境信息自动注入
**文件位置**: `src/constants/prompts.ts:129`

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

**上下文优化特点**:
- **动态获取**: 每次对话开始时实时获取
- **结构化**: 使用XML标签组织信息
- **关键信息**: 工作目录、Git状态、平台、日期、模型

### 5.2 项目特定上下文整合
通过`src/context.ts:157`的`getContext()`函数：

```typescript
return {
  ...projectConfig.context,           // 用户自定义上下文
  ...(directoryStructure ? { directoryStructure } : {}),  // 项目结构
  ...(gitStatus ? { gitStatus } : {}),                    // Git状态
  ...(codeStyle ? { codeStyle } : {}),                    // 代码风格
  ...(claudeFiles ? { claudeFiles } : {}),                // CLAUDE.md内容
  ...(readme ? { readme } : {}),                          // README内容
}
```

## 6. 工具选择优化指导

### 6.1 工具使用策略指导
**系统提示词中的工具策略**:
```
# Tool usage policy
- When doing file search, prefer to use the Agent tool in order to reduce context usage.
- If you intend to call multiple tools and there are no dependencies between the calls, 
  make all of the independent calls in the same function_calls block.
```

### 6.2 工具间协同优化
**BashTool提示词中的工具协同**:
```
VERY IMPORTANT: You MUST avoid using search commands like `find` and `grep`. 
Instead use GrepTool, GlobTool, or AgentTool to search. 
You MUST avoid read tools like `cat`, `head`, `tail`, and `ls`, 
and use FileReadTool and LSTool to read files.
```

**优化策略**:
- **专用工具优先**: 避免通用命令，使用专门工具
- **批量调用**: 无依赖的工具调用放在同一消息中
- **上下文节省**: 使用Agent工具进行复杂搜索

## 7. 用户意图识别优化

### 7.1 触发词识别 (StickerRequestTool)
```typescript
// 明确的触发短语识别
Common trigger phrases to watch for:
- "Can I get some Anthropic stickers please?"
- "How do I get Anthropic swag?"
- "I'd love some Claude stickers"

// 负面示例避免误触发
For example:
- "How do I make custom stickers for my project?" - Do not use this tool
- "Show me how to implement drag-and-drop sticker placement" - Do not use this tool
```

### 7.2 任务类型识别优化
**系统提示词中的任务分类**:
```
# Doing tasks
The user will primarily request you perform software engineering tasks. 
This includes solving bugs, adding new functionality, refactoring code, explaining code, and more.
For these tasks the following steps are recommended:
1. Use the available search tools to understand the codebase
2. Implement the solution using all tools available to you
3. Verify the solution if possible with tests
4. Run the lint and typecheck commands
```

## 8. 用户体验优化技巧

### 8.1 批量操作建议
**FileEditTool提示词**:
```
Remember: when making multiple file edits in a row to the same file, 
you should prefer to send all edits in a single message with multiple calls to this tool, 
rather than multiple messages with a single call each.
```

### 8.2 预防性指导
**文件编辑的上下文要求**:
```
CRITICAL REQUIREMENTS:
1. UNIQUENESS: Include AT LEAST 3-5 lines of context BEFORE and AFTER the change point
2. VERIFICATION: Check how many instances of the target text exist in the file
```

### 8.3 降级和容错
**动态描述生成的容错**:
```typescript
async description({ command }) {
  try {
    const result = await queryHaiku({...})
    return description || 'Executes a bash command'  // 默认描述
  } catch (error) {
    return 'Executes a bash command'  // 错误时的降级
  }
}
```

## 9. 响应质量控制

### 9.1 输出格式标准化
- **Markdown支持**: "Your responses can use Github-flavored markdown"
- **代码显示**: 适配命令行界面的等宽字体
- **长度控制**: 避免超长输出影响用户体验

### 9.2 主动性平衡
```
# Proactiveness
1. Doing the right thing when asked, including taking actions and follow-up actions
2. Not surprising the user with actions you take without asking
3. Do not add additional code explanation summary unless requested
```

## 10. 用户提示词优化总结

### 10.1 核心优化原则
1. **简洁至上**: CLI环境下的简洁回复
2. **示例驱动**: Few-shot learning指导期望行为
3. **个性化**: 基于项目历史的动态建议
4. **上下文感知**: 自动注入环境和项目信息
5. **工具协同**: 指导最优工具选择和组合

### 10.2 技术实现亮点
1. **AI生成AI提示**: 使用小模型为大模型生成提示
2. **Git历史分析**: 基于用户行为生成个性化建议
3. **动态内容注入**: 实时环境信息整合
4. **多层容错机制**: API失败时的优雅降级
5. **批量优化**: 鼓励高效的工具使用模式

这些优化技巧使Claude Code能够提供高度个性化、上下文感知的用户体验，同时保持CLI环境下的高效交互。