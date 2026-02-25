# 网络搜索 + 上下文管理

## 概述

本课程涵盖两个相关主题：为您的代理提供网络访问权限，以及管理因累积工具结果而产生的不可避免的上下文窗口膨胀问题。网络搜索功能强大，但会返回大量文本。这些文本会不断累积。最终你会触及限制。

## 第一部分：代理的网络搜索

### 为什么网络搜索很重要

LLM 的知识在其训练截止日期被冻结。它不知道：
- 今天的新闻
- 最新的软件包版本
- 当前文档
- 实时数据（股票价格、天气、体育比分）
- 训练后发生的任何事情

网络搜索让您的代理能够访问当前信息。它将静态知识库转变为动态知识库。

### 网络搜索的两种方法

#### 1. 原生/提供商网络搜索

一些模型提供商具有内置的网络搜索功能。模型本身可以搜索并整合结果。

**OpenAI**：使用其 Responses API 的 `web_search` 工具
**Perplexity**：Sonar 模型内置搜索
**Google Gemini**：搜索接地功能

**优点：**
- 快速 - 无需额外的 API 调用
- 无额外成本（通常）
- 与模型紧密集成
- 结果针对模型进行了优化

**缺点：**
- 仅适用于特定的模型/提供商
- 对搜索行为的控制较少
- 无法自定义搜索源
- 供应商锁定

#### 2. 基于工具的网络搜索

你将搜索实现为代理可以调用的工具。该工具调用搜索 API（Google、Bing、Exa、Tavily 等）并返回结果。

**优点：**
- 适用于任何支持工具调用的模型
- 完全控制搜索参数
- 可以使用专门的搜索 API
- 可以自定义结果格式
- 与模型无关

**缺点：**
- 额外的 API 成本
- 额外的延迟（工具调用往返）
- 你需要处理结果格式化
- 每次搜索需要两次 LLM 生成（调用工具、处理结果）

### 我们使用的方法

我们使用 OpenAI 的原生网络搜索工具。这是最简单的方法 - 只需一行代码：

```typescript
export const webSearch = openai.tools.webSearch({});
```

这是一个 **提供商工具** - 执行由 OpenAI 处理，而不是我们的工具执行器。结果直接在模型的响应流中返回。我们不需要自己实现搜索逻辑。

对于生产系统，您可能需要基于工具的搜索以获得更多控制权。但对于学习概念而言，原生搜索保持简单。

### 多步骤模式

网络搜索代理通常遵循两代模式：

1. **第一代**：模型决定搜索，生成带有查询的工具调用
2. **搜索执行**：结果返回给模型
3. **第二代**：模型将结果综合成响应

这就是为什么代理需要我们构建的循环 - 搜索不是单个请求/响应。

## 第二部分：上下文窗口问题

### 什么是词元（Token）？

词元是 LLM 处理的基本单位。它们不完全是单词，不完全是字符 - 它们是模型经过训练识别的文本块。

**示例：**
- "hello" → 1 个词元
- "indistinguishable" → 4 个词元 ("ind", "ist", "ingu", "ishable")
- 常见单词 → 通常 1 个词元
- 罕见单词 → 多个词元
- 代码 → 通常比等效的散文占用更多词元

**经验法则**：英语中约每 4 个字符 1 个词元，或每 0.75 个单词 1 个词元。

### 输入词元与输出词元

**输入词元**：你发送给模型的所有内容
- 系统提示
- 对话历史
- 工具定义
- 用户消息
- 工具结果

**输出词元**：模型生成的所有内容
- 响应文本
- 工具调用
- 推理（如果使用思维链）

两者都计入限制。两者都要花钱。输出词元的成本通常是输入词元的 3-4 倍。

[译者注：这里指在 API 调用时，输入和输出的词元都需要付费，且输出词元的单价通常更高]

### 什么是上下文窗口？

上下文窗口是模型可以在单个请求中处理的最大词元数。它包括输入和输出。

**当前限制（2025 年）：**
- GPT-4o：128K 词元
- Claude 3.5 Sonnet：200K 词元
- Gemini 1.5 Pro：2M 词元

听起来很多，对吧？让我们算一下：
- 平均代码文件：约 500 个词元
- 单个网络搜索结果：约 1000-3000 个词元
- 10 次交换后的对话历史：约 5000 个词元
- 工具定义：约 500-1000 个词元

累积很快，尤其是对于多次循环的代理。

### 为什么上下文窗口有限制？

约束是架构性的 - 它来自 transformer 的工作原理。

#### 注意力机制

[译者注：注意力机制是 Transformer 架构的核心创新，允许模型在处理序列时关注相关部分]

Transformer 使用"自注意力"来理解词元之间的关系。每个词元都关注每个其他词元。这很强大但很昂贵。

**数学**：注意力呈二次方缩放 - O(n²)，其中 n 是序列长度。
- 1K 词元：100 万次注意力计算
- 10K 词元：1 亿次计算
- 100K 词元：100 亿次计算

随着上下文的增长，内存和计算呈爆炸式增长。

#### 训练数据分布

模型在特定长度的序列上进行训练。在长于训练数据的序列上性能会下降。你可以在更长的序列上训练，但是：
- 需要更多内存
- 需要更长时间
- 成本更高
- 长训练序列在自然数据中很少见

#### "迷失在中间"问题

[译者注：研究发现 LLM 在处理长上下文时，对开头和结尾的信息记忆较好，但对中间部分的信息容易遗忘，这种现象被称为"迷失在中间"]

研究表明模型具有 U 型召回率：它们能很好地记住上下文的开始和结束，但在中间部分表现挣扎。这是由于：
- 位置编码限制
- 注意力自然集中在边界上
- 训练数据模式

即使你*可以*容纳 100K 词元，模型也可能无法有效使用它们。

### 管理上下文的策略

#### 1. 压缩/摘要（我们的方法）

当上下文太大时，总结迄今为止的对话。用压缩的摘要替换详细的历史记录。

**优点：**
- 保留关键信息
- 对话可以无限期继续
- 优雅降级

**缺点：**
- 失去细节
- 摘要消耗词元
- 可能失去重要的细微差别

#### 2. 驱逐/滑动窗口

达到限制时删除旧消息。只保留最近的 N 条消息。

**优点：**
- 实现简单
- 无摘要成本
- 可预测的行为

**缺点：**
- 失去所有旧上下文
- 代理"忘记"早期的对话
- 可能破坏多步骤任务

#### 3. 具有独立窗口的子代理

为特定任务生成子代理。每个都有自己的新上下文窗口。

**优点：**
- 关注点清晰分离
- 每个任务获得完整的上下文预算
- 父级只看到结果，而不是细节

**缺点：**
- 协调开销
- 结果无论如何都需要摘要
- 更复杂的架构

#### 4. RAG（检索增强生成）

[译者注：RAG 是一种将信息检索与生成结合的技术，通过从外部知识库中检索相关文档来增强 LLM 的回答]

外部存储对话历史。按需检索相关部分。

**优点：**
- 可扩展到无限历史
- 只检索相关内容
- 可以跨对话搜索

**缺点：**
- 需要向量数据库
- 检索可能错过重要的上下文
- 增加基础设施

#### 5. 重新开始

只需开始一个新的对话。手动导出/导入关键事实。

**优点：**
- 非常简单
- 清新的开始
- 无累积的混乱

**缺点：**
- 用户体验中断
- 手动上下文传输
- 失去对话流程

#### 6. 首先防止膨胀

设计工具和提示以最小化词元使用：
- 截断长工具结果
- 高效格式化响应
- 仅包含必要的上下文
- 使用结构化输出（JSON）而非散文

这通常是最好的第一道防线。

## 我们如何实现它

我们使用简单的压缩策略：
1. 在每轮之前估计词元使用
2. 如果超过阈值（上下文窗口的 80%），触发压缩
3. 将对话历史总结为压缩形式
4. 用摘要替换历史
5. 继续对话

这不是最复杂的方法，但它可靠且易于理解。

## 代码

### src/agent/tools/webSearch.ts

最简单的可能网络搜索 - OpenAI 的原生提供商工具：

```typescript
import { openai } from "@ai-sdk/openai";

/**
 * OpenAI 原生网络搜索工具
 *
 * 这是一个提供商工具 - 执行由 OpenAI 处理，而不是我们的工具执行器。
 * 结果直接在模型的响应流中返回。
 */
export const webSearch = openai.tools.webSearch({});
```

就是这样。一行。`openai.tools.webSearch({})` 返回一个工具配置，AI SDK 知道如何处理它。当模型调用它时，OpenAI 在内部处理搜索。

### src/agent/tools/index.ts

注册网络搜索工具：

```typescript
import { readFile, writeFile, listFiles, deleteFile } from "./file.ts";
import { runCommand } from "./shell.ts";
import { executeCode } from "./codeExecution.ts";
import { webSearch } from "./webSearch.ts";

// 为代理组合所有工具
export const tools = {
  readFile,
  writeFile,
  listFiles,
  deleteFile,
  runCommand,
  executeCode,
  webSearch,
};

// 为 evals 中选择性使用导出各个工具
export { readFile, writeFile, listFiles, deleteFile } from "./file.ts";
export { runCommand } from "./shell.ts";
export { executeCode } from "./codeExecution.ts";
export { webSearch } from "./webSearch.ts";

// 用于 evals 的工具集
export const fileTools = {
  readFile,
  writeFile,
  listFiles,
  deleteFile,
};

export const shellTools = {
  runCommand,
};
```

### src/agent/context/modelLimits.ts

用于检查词元阈值的函数。更新 `isOverThreshold` 和 `calculateUsagePercentage`：

```typescript
export function isOverThreshold(
  totalTokens: number,
  contextWindow: number,
  threshold: number = DEFAULT_THRESHOLD,
): boolean {
  return totalTokens > contextWindow * threshold;
}

export function calculateUsagePercentage(
  totalTokens: number,
  contextWindow: number,
): number {
  return (totalTokens / contextWindow) * 100;
}
```

### src/agent/context/compaction.ts

摘要提示和压缩逻辑：

```typescript
const SUMMARIZATION_PROMPT = `You are a conversation summarizer. Your task is to create a concise summary of the conversation so far that preserves:

1. Key decisions and conclusions reached
2. Important context and facts mentioned
3. Any pending tasks or questions
4. The overall goal of the conversation

Be concise but complete. The summary should allow the conversation to continue naturally.

Conversation to summarize:
`;
```

以及压缩函数：

```typescript
export async function compactConversation(
  messages: ModelMessage[],
  model: string = "gpt-5-mini",
): Promise<ModelMessage[]> {
  // 过滤掉系统消息 - 它们单独处理
  const conversationMessages = messages.filter((m) => m.role !== "system");

  if (conversationMessages.length === 0) {
    return [];
  }

  const conversationText = messagesToText(conversationMessages);

  const { text: summary } = await generateText({
    model: openai(model),
    prompt: SUMMARIZATION_PROMPT + conversationText,
  });

  // 创建压缩的消息
  const compactedMessages: ModelMessage[] = [
    {
      role: "user",
      content: `[CONVERSATION SUMMARY]\nThe following is a summary of our conversation so far:\n\n${summary}\n\nPlease continue from where we left off.`,
    },
    {
      role: "assistant",
      content:
        "I understand. I've reviewed the summary of our conversation and I'm ready to continue. How can I help you next?",
    },
  ];

  return compactedMessages;
}
```

关键实现细节：
- 系统消息被过滤掉（它们每轮都重新添加）
- 对话被转换为纯文本以进行摘要
- 结果是两条消息的"种子"，为对话继续做准备
- 假的助手响应有助于保持对话流程

### src/agent/run.ts

将上下文管理逻辑添加到代理循环。首先，导入和词元使用报告：

```typescript
import {
  estimateMessagesTokens,
  getModelLimits,
  isOverThreshold,
  calculateUsagePercentage,
  compactConversation,
  DEFAULT_THRESHOLD,
} from "./context/index.ts";
```

然后，在循环开始之前，检查我们是否需要压缩：

```typescript
const modelLimits = getModelLimits(MODEL_NAME);

// 过滤并检查我们是否需要在开始之前压缩对话历史
let workingHistory = filterCompatibleMessages(conversationHistory);
const preCheckTokens = estimateMessagesTokens([
  { role: "system", content: SYSTEM_PROMPT },
  ...workingHistory,
  { role: "user", content: userMessage },
]);

if (isOverThreshold(preCheckTokens.total, modelLimits.contextWindow)) {
  // 压缩对话
  workingHistory = await compactConversation(workingHistory, MODEL_NAME);
}
```

在循环中添加词元使用报告：

```typescript
// 报告初始词元使用
const reportTokenUsage = () => {
  if (callbacks.onTokenUsage) {
    const usage = estimateMessagesTokens(messages);
    callbacks.onTokenUsage({
      inputTokens: usage.input,
      outputTokens: usage.output,
      totalTokens: usage.total,
      contextWindow: modelLimits.contextWindow,
      threshold: DEFAULT_THRESHOLD,
      percentage: calculateUsagePercentage(
        usage.total,
        modelLimits.contextWindow,
      ),
    });
  }
};

reportTokenUsage();
```

在每次对消息的重大更改后调用 `reportTokenUsage()`：
- 添加响应消息后
- 添加工具结果后

## 压缩策略解释

我们的方法故意简单：

1. **预检查**：在开始一轮之前，估计总词元数
2. **阈值**：如果超过上下文窗口的 80%，则压缩
3. **摘要**：使用 LLM 本身来总结对话
4. **替换**：用压缩的摘要交换详细的历史
5. **种子**：用摘要消息开始压缩的上下文

这发生在轮次开始*之前*，所以代理总是有空间工作。

### 为什么是 80%？

我们需要空间用于：
- 新的用户消息
- 工具调用和结果
- 助手的响应
- 估计错误的安全余量

80% 为完整的轮次周期留下约 20% 的空间。

### 权衡

**我们保留的内容：**
- 总体目标和意图
- 做出的关键决策
- 提到的重要事实

**我们失去的内容：**
- 确切的措辞
- 详细的工具输出
- 逐步推理
- 情感细微差别

对于专注于任务的代理，这种权衡通常有效。对于语气重要的对话代理，您可能需要不同的方法。

## 资料来源

- [Web Search Agent - AI SDK Cookbook](https://ai-sdk.dev/cookbook/node/web-search-agent)
- [Understanding LLM Context Window Limits](https://demiliani.com/2025/11/02/understanding-llm-performance-degradation-a-deep-dive-into-context-window-limits/)
- [The Context Window Paradox](https://medium.com/@shashwatabhattacharjee9/the-context-window-paradox-engineering-trade-offs-in-modern-llm-architecture-d22d8f954a05)
- [The Context Window Problem: Scaling Agents Beyond Token Limits](https://factory.ai/news/context-window-problem)
- [Why Large Language Models Struggle with Long Contexts](https://www.understandingai.org/p/why-large-language-models-struggle)
- [5 Approaches to Solve LLM Token Limits](https://www.deepchecks.com/5-approaches-to-solve-llm-token-limits/)
