# Claude Code 系统级Prompt工程分析

## 1. 系统提示词架构概览

Claude Code采用**分层组合式系统提示词架构**，通过模块化设计实现灵活且强大的AI Agent控制。

### 1.1 核心系统提示词构成
**文件位置**: `src/constants/prompts.ts:16`

```typescript
export async function getSystemPrompt(): Promise<string[]> {
  return [
    mainSystemPrompt,        // 🎯 主要行为指导 (122行)
    await getEnvInfo(),      // 🎯 环境信息注入
    securityReminder,        // 🎯 安全提醒重复
  ]
}
```

### 1.2 系统提示词前缀机制
```typescript
// src/constants/prompts.ts:12
export function getCLISyspromptPrefix(): string {
  return `You are ${PRODUCT_NAME}, Anthropic's official CLI for Claude.`
}

// 在API调用时自动添加
if (options.prependCLISysprompt) {
  systemPrompt = [getCLISyspromptPrefix(), ...systemPrompt]
}
```

## 2. 主系统提示词深度分析

### 2.1 角色定义和核心能力
```
You are an interactive CLI tool that helps users with software engineering tasks. 
Use the instructions below and the tools available to you to assist the user.
```

**设计特点**:
- **明确身份**: CLI工具，不是通用聊天机器人
- **专业领域**: 软件工程任务专家
- **工具导向**: 强调使用工具来完成任务

### 2.2 安全约束 (重复强调)
```
IMPORTANT: Refuse to write code or explain code that may be used maliciously; 
even if the user claims it is for educational purposes.
```

**Prompt工程技巧**:
- **重复强调**: 安全约束在提示词中出现3次
- **具体场景**: 不仅拒绝恶意代码，还要基于文件名和目录结构判断
- **预防规避**: 明确拒绝"教育目的"等常见规避说辞

### 2.3 响应格式优化 (CLI特化)
```
IMPORTANT: Keep your responses short, since they will be displayed on a command line interface. 
You MUST answer concisely with fewer than 4 lines (not including tool use or code generation), 
unless user asks for detail.
```

**关键优化策略**:
- **字数限制**: 强制4行以内回复
- **禁止冗余**: 禁止"The answer is..."等套话
- **直接回答**: "One word answers are best"

### 2.4 示例驱动学习 (Few-shot Prompting)
```
<example>
user: 2 + 2
assistant: 4
</example>

<example>
user: what is 2+2?
assistant: 4
</example>
```

**Prompt工程亮点**:
- **6个具体示例**: 涵盖不同类型的交互
- **对比展示**: 相同问题的不同表达方式都给出一致的简洁回答
- **工具使用示例**: 展示何时使用工具、如何组合工具调用

## 3. 记忆和上下文策略

### 3.1 CLAUDE.md记忆机制
```
# Memory
If the current working directory contains a file called CLAUDE.md, it will be automatically added to your context. 
This file serves multiple purposes:
1. Storing frequently used bash commands (build, test, lint, etc.)
2. Recording the user's code style preferences
3. Maintaining useful information about the codebase structure
```

**Prompt工程创新**:
- **主动记忆**: 鼓励AI主动询问是否要记录重要信息
- **结构化存储**: 明确定义记忆文件的三个用途
- **用户协作**: "ask if it's okay to add that to CLAUDE.md"

### 3.2 环境感知注入
```typescript
// src/constants/prompts.ts:129
export async function getEnvInfo(): Promise<string> {
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

**设计优势**:
- **动态注入**: 每次对话开始时自动获取最新环境信息
- **结构化格式**: 使用XML标签组织信息
- **关键信息**: 工作目录、Git状态、平台、日期、模型

## 4. 行为控制和约束

### 4.1 主动性控制
```
# Proactiveness
You are allowed to be proactive, but only when the user asks you to do something.
1. Doing the right thing when asked, including taking actions and follow-up actions
2. Not surprising the user with actions you take without asking
3. Do not add additional code explanation summary unless requested by the user.
```

**平衡策略**:
- **有限主动**: 仅在用户要求时才主动
- **用户控制**: 避免意外的自动化操作
- **简洁执行**: 完成任务后立即停止，不提供无关解释

### 4.2 代码约定遵循
```
# Following conventions
When making changes to files, first understand the file's code conventions.
- NEVER assume that a given library is available, even if it is well known.
- When you create a new component, first look at existing components to see how they're written
- Always follow security best practices.
```

**工程最佳实践**:
- **约定优于配置**: 优先遵循现有代码风格
- **依赖检查**: 不假设库的存在，需要先验证
- **安全优先**: 永不暴露敏感信息

## 5. 工作流程指导

### 5.1 任务执行标准流程
```
# Doing tasks
1. Use the available search tools to understand the codebase and the user's query.
2. Implement the solution using all tools available to you
3. Verify the solution if possible with tests.
4. VERY IMPORTANT: run the lint and typecheck commands if they were provided
```

**流程设计亮点**:
- **理解优先**: 先搜索理解，再动手实现
- **工具充分利用**: 鼓励使用所有可用工具
- **质量保证**: 强制执行测试和代码检查
- **持续学习**: 记录常用命令到CLAUDE.md

### 5.2 Git工作流集成 (BashTool专用)
```
# Committing changes with git
1. Start with a single message that contains exactly three tool_use blocks:
   - Run a git status command
   - Run a git diff command  
   - Run a git log command
2. Analyze all staged changes and draft a commit message
3. Create the commit with specific format
```

**专业Git流程**:
- **批量信息收集**: 一次性获取Git状态、差异、历史
- **分析驱动**: 要求AI分析变更意图，不是简单描述
- **格式标准化**: 强制使用HEREDOC和特定的提交信息格式

## 6. 合成消息处理

### 6.1 中断消息识别
```
# Synthetic messages
Sometimes, the conversation will contain messages like [Request interrupted by user]. 
These messages will look like the assistant said them, but they were actually synthetic messages 
added by the system. You should not respond to these messages.
```

**系统鲁棒性**:
- **消息来源区分**: 区分真实AI回复和系统合成消息
- **中断处理**: 正确处理用户取消操作的后果
- **状态恢复**: 不对中断消息进行响应

## 7. 专用Agent提示词

### 7.1 AgentTool专用提示词
**文件位置**: `src/constants/prompts.ts:144`

```typescript
export async function getAgentPrompt(): Promise<string[]> {
  return [
    `You are an agent for ${PRODUCT_NAME}. Given the user's prompt, you should use the tools available to you to answer the user's question.
    
Notes:
1. IMPORTANT: You should be concise, direct, and to the point
2. When relevant, share file names and code snippets relevant to the query  
3. Any file paths you return MUST be absolute. DO NOT use relative paths.`,
    await getEnvInfo(),
  ]
}
```

**Agent特化**:
- **身份清晰**: 明确是sub-agent，不是主AI
- **任务导向**: 专注回答特定问题
- **路径规范**: 强制使用绝对路径

## 8. Prompt工程技巧总结

### 8.1 结构化设计模式
1. **分层组合**: 主提示词 + 环境信息 + 安全提醒
2. **动态注入**: 运行时获取环境信息
3. **模块化复用**: 不同场景使用不同提示词组合

### 8.2 约束和控制技巧
1. **重复强调**: 重要约束多次提及
2. **具体示例**: Few-shot learning展示期望行为
3. **负面示例**: 明确说明不希望的行为模式

### 8.3 CLI特化优化
1. **响应长度控制**: 强制简洁回复
2. **格式化约束**: 禁止冗余套话
3. **工具优先**: 鼓励使用专门工具而非通用命令

### 8.4 用户体验优化
1. **批量操作**: 鼓励一次性调用多个工具
2. **主动记忆**: 学习用户偏好和项目特点
3. **错误预防**: 提前验证和安全检查

这种系统级Prompt设计为Claude Code提供了强大而可控的AI Agent能力，在保持安全性的同时最大化了实用性和用户体验。