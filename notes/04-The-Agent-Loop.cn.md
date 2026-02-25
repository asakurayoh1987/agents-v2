# 智能体循环

## 概述

智能体循环是区分智能体与简单 LLM 调用的关键机制。它允许 AI 采取行动、观察结果，并决定下一步做什么——反复进行——直到任务完成。

## LLM vs 工作流 vs 智能体

这些术语经常被混淆。让我们来澄清一下：

### LLM（单次调用）

```
用户 → LLM → 响应
```

一个输入，一个输出。模型基于提示生成文本。没有工具，没有迭代，没有行动。这是最简单形式的 ChatGPT。

**特点：**
- 无状态（调用之间没有记忆）
- 无外部行动
- 单次生成
- 得到什么就是什么

### 工作流（编排管道）

```
用户 → 步骤 1 → 步骤 2 → 步骤 3 → 响应
         ↓        ↓        ↓
       [LLM]   [工具]   [LLM]
```

预定义的步骤序列。你决定顺序。每个步骤可能涉及 LLM 调用或工具，但流程是固定的。比如："先总结，再翻译，最后格式化。"

**特点：**
- 确定性流程
- 人工设计的序列
- LLM 不决定接下来发生什么
- 可预测但不灵活

### 智能体（自主循环）

```
用户 → 智能体循环 ←→ 工具
           ↓
        响应
```

LLM 决定做什么。它可以调用工具、观察结果，并选择下一步行动。循环持续进行，直到智能体决定完成。

**特点：**
- LLM 控制流程
- 动态工具选择
- 迭代优化
- 不可预测但灵活

## 代理能力光谱

代理能力不是二元的。它是一个光谱：

### 无代理能力（纯 LLM）
模型生成文本。就这样。没有工具，没有行动。

### 低代理能力（单次工具调用）
模型可以调用一个工具，然后响应。没有迭代。比如："调用天气 API 并告诉我结果。"

### 中等代理能力（固定迭代）
模型可以调用工具，但你限制迭代次数。"最多进行 3 次工具调用，然后响应。" 防止无限循环，但限制了复杂任务。

### 高代理能力（完整循环）
模型循环直到它决定停止。它可以按任何顺序调用所需的任意数量的工具，直到任务完成。

### 完全自主（多智能体）
多个智能体协调。智能体生成子智能体。人类只介入进行批准。这就是事情变得有趣（且有风险）的地方。

## 为什么要构建智能体？

### 任务需要多个步骤

"读取配置文件并更新端口"需要：
1. 读取文件
2. 解析内容
3. 修改值
4. 写回文件

单次 LLM 调用无法做到这一点。它只能告诉你该做什么。

### 信息收集是迭代性的

"查找所有导入 auth 模块的文件"可能需要：
1. 列出 src/ 中的文件
2. 检查每个文件的导入
3. 一些文件从其他导入 auth 的文件导入
4. 递归探索

你预先不知道需要多少步。智能体在探索过程中逐步发现。

### 真实问题有依赖关系

步骤 2 依赖于步骤 1 的输出。你不能并行化所有事情。智能体需要在决定下一步之前看到结果。

### 人类不想微观管理

如果你必须告诉 AI 每一步，你不如自己做。价值在于说"做 X"，然后让它弄清楚如何做。

## 什么是循环？

本质上：

```
while (未完成) {
  1. 向 LLM 发送消息
  2. LLM 响应（文本和/或工具调用）
  3. 如果有工具调用：执行它们，将结果添加到消息中
  4. 如果没有工具调用：我们完成了
}
```

就是这样。其他都是细节。

### 循环的伪代码

```typescript
while (true) {
  // 询问模型该做什么
  response = await llm.generate(messages)

  // 将响应添加到对话中
  messages.push(response)

  // 检查模型是否想调用工具
  if (response.hasToolCalls) {
    for (toolCall of response.toolCalls) {
      // 执行每个工具
      result = await executeTool(toolCall)
      // 将结果添加到对话中
      messages.push(toolResult(result))
    }
    // 再次循环 - 让模型看到结果
  } else {
    // 没有工具调用 = 模型完成
    break
  }
}
```

### 为什么用 `while(true)`？

智能体预先不知道需要多少次迭代。有些任务需要 1 次工具调用。有些需要 20 次。循环一直运行，直到模型停止调用工具。

## 循环何时停止？

这很关键。无限循环会浪费金钱和时间。

### 自然完成

模型简单地响应文本而没有工具调用。它决定任务已完成。这是理想情况。

### 完成原因

AI SDK 提供 `finishReason`：
- `"stop"` - 模型正常完成
- `"tool-calls"` - 模型想调用工具
- `"length"` - 达到 token 限制
- `"content-filter"` - 内容被过滤

我们检查：如果 `finishReason !== "tool-calls"`，循环结束。

### 最大迭代次数

安全网。即使模型不断调用工具，在 N 次迭代后停止：

```typescript
const MAX_ITERATIONS = 20;
let iterations = 0;

while (iterations < MAX_ITERATIONS) {
  // ... 循环体
  iterations++;
}
```

### 最大 Token 数

如果上下文太大，停止或压缩。我们在上下文管理课程中会讲到这一点。

### 用户干预

让用户可以取消。Ctrl+C、取消按钮、超时——给人类一个逃生口。

### 错误阈值

如果工具重复失败，停止。智能体可能卡住了：

```typescript
let consecutiveErrors = 0;
if (toolFailed) {
  consecutiveErrors++;
  if (consecutiveErrors > 3) break;
} else {
  consecutiveErrors = 0;
}
```

## 循环中的流式传输

我们使用 `streamText` 而不是 `generateText` 来实现实时输出：

```typescript
const result = streamText({
  model: openai(MODEL_NAME),
  messages,
  tools,
});

for await (const chunk of result.fullStream) {
  if (chunk.type === "text-delta") {
    // 立即向用户流式传输文本
    callbacks.onToken(chunk.text);
  }
  if (chunk.type === "tool-call") {
    // 收集工具调用
    toolCalls.push(chunk);
  }
}
```

用户在生成时就能看到文本，而不是之后。这使得智能体即使在思考时也感觉响应迅速。

[译者注：流式传输（Streaming）是指数据逐步传输和显示的技术，而不是等到全部生成后才显示。这提供了更好的用户体验，因为用户可以立即看到部分结果。]

## 代码

### src/agent/run.ts

完整的智能体循环实现：

```typescript
import { streamText, type ModelMessage } from "ai";
import { openai } from "@ai-sdk/openai";
import { getTracer } from "@lmnr-ai/lmnr";
import { tools } from "./tools/index.ts";
import { executeTool } from "./executeTool.ts";
import { SYSTEM_PROMPT } from "./system/prompt.ts";
import { Laminar } from "@lmnr-ai/lmnr";
import type { AgentCallbacks, ToolCallInfo } from "../types.ts";
import { filterCompatibleMessages } from "./system/filterMessages.ts";

Laminar.initialize({
  projectApiKey: process.env.LMNR_API_KEY,
});

const MODEL_NAME = "gpt-5-mini";

export async function runAgent(
  userMessage: string,
  conversationHistory: ModelMessage[],
  callbacks: AgentCallbacks,
): Promise<ModelMessage[]> {
  // 在开始之前过滤并检查是否需要压缩对话历史
  const workingHistory = filterCompatibleMessages(conversationHistory);

  const messages: ModelMessage[] = [
    { role: "system", content: SYSTEM_PROMPT },
    ...workingHistory,
    { role: "user", content: userMessage },
  ];

  let fullResponse = "";

  while (true) {
    const result = streamText({
      model: openai(MODEL_NAME),
      messages,
      tools,
      experimental_telemetry: {
        isEnabled: true,
        tracer: getTracer(),
      },
    });

    const toolCalls: ToolCallInfo[] = [];
    let currentText = "";
    let streamError: Error | null = null;

    try {
      for await (const chunk of result.fullStream) {
        if (chunk.type === "text-delta") {
          currentText += chunk.text;
          callbacks.onToken(chunk.text);
        }

        if (chunk.type === "tool-call") {
          const input = "input" in chunk ? chunk.input : {};
          toolCalls.push({
            toolCallId: chunk.toolCallId,
            toolName: chunk.toolName,
            args: input as Record<string, unknown>,
          });
          callbacks.onToolCallStart(chunk.toolName, input);
        }
      }
    } catch (error) {
      streamError = error as Error;
      // 如果我们有一些文本，继续处理
      // 否则，如果不是"无输出"错误，则重新抛出
      if (
        !currentText &&
        !streamError.message.includes("No output generated")
      ) {
        throw streamError;
      }
    }

    fullResponse += currentText;

    // 如果流式传输因"无输出"而错误且我们没有文本，尝试恢复
    if (streamError && !currentText) {
      // 添加回退响应
      fullResponse =
        "I apologize, but I wasn't able to generate a response. Could you please try rephrasing your message?";
      callbacks.onToken(fullResponse);
      break;
    }

    const finishReason = await result.finishReason;

    if (finishReason !== "tool-calls" || toolCalls.length === 0) {
      const responseMessages = await result.response;
      messages.push(...responseMessages.messages);
      break;
    }

    const responseMessages = await result.response;
    messages.push(...responseMessages.messages);

    for (const tc of toolCalls) {
      const result = await executeTool(tc.toolName, tc.args);
      callbacks.onToolCallEnd(tc.toolName, result);

      messages.push({
        role: "tool",
        content: [
          {
            type: "tool-result",
            toolCallId: tc.toolCallId,
            toolName: tc.toolName,
            output: { type: "text", value: result },
          },
        ],
      });
    }
  }

  callbacks.onComplete(fullResponse);

  return messages;
}
```

## 循环分解

### 1. 设置消息

```typescript
const messages: ModelMessage[] = [
  { role: "system", content: SYSTEM_PROMPT },
  ...workingHistory,
  { role: "user", content: userMessage },
];
```

从系统提示开始，添加对话历史，添加新的用户消息。

### 2. 流式传输响应

```typescript
const result = streamText({
  model: openai(MODEL_NAME),
  messages,
  tools,
});
```

使用消息和可用工具调用模型。我们使用 `streamText` 进行实时输出。

### 3. 处理流

```typescript
for await (const chunk of result.fullStream) {
  if (chunk.type === "text-delta") {
    callbacks.onToken(chunk.text);
  }
  if (chunk.type === "tool-call") {
    toolCalls.push(chunk);
  }
}
```

当数据块到达时：
- 文本增量立即发送到 UI
- 工具调用被收集以供执行

### 4. 检查是否完成

```typescript
const finishReason = await result.finishReason;

if (finishReason !== "tool-calls" || toolCalls.length === 0) {
  break;
}
```

如果模型没有请求工具调用，我们就完成了。中断循环。

### 5. 执行工具

```typescript
for (const tc of toolCalls) {
  const result = await executeTool(tc.toolName, tc.args);
  callbacks.onToolCallEnd(tc.toolName, result);

  messages.push({
    role: "tool",
    content: [
      {
        type: "tool-result",
        toolCallId: tc.toolCallId,
        toolName: tc.toolName,
        output: { type: "text", value: result },
      },
    ],
  });
}
```

执行每个工具，将结果添加到消息中。模型将在下一次迭代中看到这些结果。

### 6. 再次循环

回到步骤 2。模型现在有了工具结果，可以决定下一步做什么。

## 消息格式

工具结果有特定的结构：

```typescript
{
  role: "tool",
  content: [
    {
      type: "tool-result",
      toolCallId: tc.toolCallId,  // 链接到原始调用
      toolName: tc.toolName,
      output: { type: "text", value: result },
    },
  ],
}
```

`toolCallId` 至关重要——它将结果链接回特定的工具调用。模型可以在每轮中进行多次工具调用，每个调用的结果都需要正确匹配。

## UI 更新的回调

循环使用回调与 UI 通信：

```typescript
callbacks.onToken(chunk.text);       // 流式传输文本
callbacks.onToolCallStart(name, args); // 工具执行开始
callbacks.onToolCallEnd(name, result); // 工具执行完成
callbacks.onComplete(fullResponse);    // 智能体完成
```

这保持了循环的纯净——它不了解 React 或 Ink 或任何 UI 框架。它只是在事件发生时调用函数。

[译者注：回调（Callback）是一种将函数作为参数传递的模式，允许被调用的代码在特定事件发生时通知调用者。这种设计模式实现了关注点分离，使智能体循环逻辑与 UI 框架解耦。]

## 常见陷阱

### 无限循环
模型永远不断地调用工具。始终要有最大迭代限制。

### 丢失工具结果
忘记将工具结果添加到消息中。模型将看不到它们并且会感到困惑。

### 错误的消息顺序
消息必须按顺序排列：user → assistant → tool → assistant → ... 模型期望这种结构。

### 不处理错误
工具执行可能失败。捕获错误并将它们作为工具结果添加，以便模型可以适应。

### 阻塞 UI
长时间的工具执行而没有流式反馈。用户会认为它冻结了。
