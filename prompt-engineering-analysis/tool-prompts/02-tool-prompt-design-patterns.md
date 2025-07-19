# Claude Code 工具提示词设计模式分析

## 1. 工具提示词架构概览

Claude Code的每个工具都包含两层提示词设计：
- **`description`**: 简短的工具功能描述（发送给LLM的工具列表）
- **`prompt`**: 详细的使用指导（工具被选中时的详细说明）

### 1.1 提示词调用机制
```typescript
// src/services/claude.ts:482-493
const toolSchemas = await Promise.all(
  tools.map(async _ => ({
    name: _.name,
    description: await _.prompt({  // 🎯 使用prompt作为工具描述
      dangerouslySkipPermissions: options.dangerouslySkipPermissions,
    }),
    input_schema: zodToJsonSchema(_.inputSchema),
  }))
)
```

## 2. 工具提示词设计模式类型学

### 2.1 指令式操作工具 (BashTool模式)

**特征**: 复杂操作流程、安全约束、多步骤指导

**典型结构**:
```
1. 功能描述
2. 执行前检查步骤
3. 安全限制说明  
4. 具体操作指导
5. 输出处理说明
6. 使用注意事项
7. 专门场景处理 (如Git提交)
```

**BashTool示例分析**:
```
Executes a given bash command in a persistent shell session with optional timeout, 
ensuring proper handling and security measures.

Before executing the command, please follow these steps:
1. Directory Verification: [具体检查步骤]
2. Security Check: [安全验证] 
3. Command Execution: [执行指导]
4. Output Processing: [结果处理]
5. Return Result: [返回格式]
```

**设计亮点**:
- **预检步骤**: 强制执行前验证
- **安全白名单**: 明确禁用危险命令列表
- **持久化会话**: 强调Shell状态的持续性
- **专用工作流**: Git提交和PR创建的详细流程

### 2.2 精确操作工具 (FileEditTool模式)

**特征**: 严格约束、错误预防、唯一性要求

**典型结构**:
```
1. 工具定位和适用场景
2. 操作前准备步骤
3. 参数详细说明
4. 关键要求 (CRITICAL REQUIREMENTS)
5. 失败场景警告
6. 最佳实践建议
```

**FileEditTool示例分析**:
```
CRITICAL REQUIREMENTS FOR USING THIS TOOL:
1. UNIQUENESS: The old_string MUST uniquely identify the specific instance
   - Include AT LEAST 3-5 lines of context BEFORE the change point
   - Include AT LEAST 3-5 lines of context AFTER the change point
2. SINGLE INSTANCE: This tool can only change ONE instance at a time
3. VERIFICATION: Before using this tool:
   - Check how many instances of the target text exist
```

**设计亮点**:
- **上下文要求**: 强制3-5行上下文确保唯一性
- **失败预防**: 详细说明常见失败原因
- **批量优化**: 建议同一消息中多次调用

### 2.3 搜索发现工具 (GrepTool模式)

**特征**: 简洁明确、能力边界、使用场景

**典型结构**:
```
1. 核心能力列表 (bullet points)
2. 技术特性说明
3. 适用场景指导
4. 工具选择建议
```

**GrepTool示例分析**:
```
- Fast content search tool that works with any codebase size
- Searches file contents using regular expressions  
- Supports full regex syntax (eg. "log.*Error", "function\\s+\\w+", etc.)
- Filter files by pattern with the include parameter
- When you are doing an open ended search that may require multiple rounds, 
  use the Agent tool instead
```

**设计亮点**:
- **能力清单**: 清晰列举工具能力
- **语法示例**: 提供具体正则表达式示例
- **工具协同**: 指导何时使用其他工具

### 2.4 体验增强工具 (StickerRequestTool模式)

**特征**: 触发条件、用户意图识别、边界限制

**典型结构**:
```
1. 工具用途说明
2. 触发短语示例
3. 操作流程描述
4. 负面示例 (避免误触发)
```

**StickerRequestTool示例分析**:
```
Common trigger phrases to watch for:
- "Can I get some Anthropic stickers please?"
- "How do I get Anthropic swag?"
- "I'd love some Claude stickers"

NOTE: Only use this tool if the user has explicitly asked us to send them stickers.
For example:
- "How do I make custom stickers for my project?" - Do not use this tool
```

**设计亮点**:
- **意图识别**: 提供具体触发短语
- **误用防护**: 明确不应使用的场景
- **用户体验**: 强调显式请求的重要性

### 2.5 思考辅助工具 (ThinkTool模式)

**特征**: 元认知、使用场景、透明度

**典型结构**:
```
1. 工具性质说明
2. 常见用例列表
3. 使用时机指导
4. 透明度价值说明
```

**ThinkTool示例分析**:
```
Use the tool to think about something. It will not obtain new information or 
make any changes to the repository, but just log the thought.

Common use cases:
1. When exploring a repository and discovering the source of a bug
2. After receiving test results, use this tool to brainstorm ways to fix failing tests
3. When planning a complex refactoring
```

**设计亮点**:
- **无副作用**: 明确说明不产生实际操作
- **场景丰富**: 5个具体使用场景
- **过程透明**: 强调思考过程的可见性

## 3. 动态提示词生成

### 3.1 BashTool的动态描述生成
```typescript
// src/tools/BashTool/BashTool.tsx:39
async description({ command }) {
  try {
    const result = await queryHaiku({
      systemPrompt: [
        `You are a command description generator. Write a clear, concise description of what this command does in 5-10 words. Examples:
        
        Input: ls
        Output: Lists files in current directory
        
        Input: git status  
        Output: Shows working tree status`,
      ],
      userPrompt: `Describe this command: ${command}`,
    })
    return description || 'Executes a bash command'
  } catch (error) {
    return 'Executes a bash command'  // 降级处理
  }
}
```

**设计创新**:
- **AI生成AI提示**: 使用Haiku模型为主模型生成工具描述
- **Few-shot示例**: 提供4个标准描述格式示例
- **容错机制**: API失败时使用默认描述
- **长度控制**: 强制5-10词的简洁描述

### 3.2 AgentTool的条件化提示词
```typescript
// src/tools/AgentTool/prompt.ts:20
export async function getPrompt(
  dangerouslySkipPermissions: boolean,
): Promise<string> {
  const tools = await getAgentTools(dangerouslySkipPermissions)
  const toolNames = tools.map(_ => _.name).join(', ')
  
  return `Launch a new agent that has access to the following tools: ${toolNames}.
  ${dangerouslySkipPermissions ? '' : `
  5. IMPORTANT: The agent can not use ${BashTool.name}, ${FileWriteTool.name}, ${FileEditTool.name}...`}
  `
}
```

**动态调整策略**:
- **权限感知**: 根据权限模式调整可用工具
- **工具列表**: 动态生成当前可用工具清单
- **约束条件**: 根据安全模式动态添加限制

## 4. 工具间协同提示词设计

### 4.1 工具选择指导
**FileReadTool**: "For Jupyter notebooks (.ipynb files), use the NotebookReadTool instead"
**BashTool**: "You MUST avoid read tools like `cat`, use FileReadTool and LSTool instead"
**GrepTool**: "When doing open ended search, use the Agent tool instead"

### 4.2 工具组合建议
**FileEditTool**: "Before using this tool: Use the View tool to understand the file's contents"
**BashTool**: "If the command will create new directories, first use the LS tool to verify"

## 5. 提示词工程技巧总结

### 5.1 结构化设计原则
1. **分层信息**: 从概述到细节的信息层次
2. **操作导向**: 明确的执行步骤和检查清单
3. **场景驱动**: 提供具体使用场景和示例

### 5.2 错误预防策略
1. **预检步骤**: 强制执行前验证
2. **失败场景**: 明确说明常见失败原因
3. **负面示例**: 展示不正确的使用方式

### 5.3 用户体验优化
1. **批量操作提示**: 建议同一消息中多次调用
2. **工具协同**: 指导何时使用其他工具
3. **意图识别**: 帮助AI正确理解用户需求

### 5.4 安全和约束机制
1. **白名单/黑名单**: 明确允许和禁止的操作
2. **权限感知**: 根据权限模式调整行为
3. **边界清晰**: 明确工具的能力范围

### 5.5 动态适应能力
1. **环境感知**: 根据运行环境调整提示词
2. **条件化内容**: 基于参数动态生成内容
3. **降级处理**: API失败时的备选方案

这种多模式的工具提示词设计确保了Claude Code在不同场景下都能提供准确、安全、高效的工具使用指导。