# 工具调用 (Tool Calling)

## 概述

本课程介绍工具调用——这是让大语言模型（LLM）能够与外部世界交互的机制。我们将创建第一个工具并将其连接到我们的智能体。

## 什么是工具调用？

工具调用（也称为函数调用）是一种让 LLM 请求执行函数的能力。模型不仅可以生成文本，还可以：

1. 识别任务何时需要外部能力
2. 选择合适的工具来使用
3. 为该工具生成正确的参数
4. 接收结果并将其整合到响应中

**函数调用就是工具调用**——它们是同一回事。"函数调用"是最初的术语（来自 OpenAI），但现在"工具调用"更常用，因为它更准确地描述了正在发生的事情：模型使用工具来完成任务。

## 工具调用如何工作

1. **你定义工具** - 每个工具都有名称、描述和参数模式
2. **你将工具与提示一起发送** - 模型可以看到有哪些工具可用
3. **模型决定** - 根据用户的请求，模型要么直接响应，要么请求调用工具
4. **你执行** - 如果模型请求工具调用，你运行该函数并返回结果
5. **模型继续** - 模型看到结果后可以响应或调用更多工具

模型**从不**自己执行工具。它只生成一个结构化请求，说"我想用参数 Y 调用工具 X"。你需要负责实际运行代码。

## 为什么工具对智能体很重要

没有工具，LLM 只能：
- 根据训练数据回答
- 生成文本
- 对提示中的信息进行推理

有了工具，LLM 可以：
- 读写文件
- 搜索网络
- 执行代码
- 查询数据库
- 调用 API
- 与你暴露的任何系统交互

工具正是将 LLM 从聊天机器人转变为智能体的关键。

## 工具的剖析

每个工具有三个部分：

```typescript
{
  description: "这个工具的作用 - 帮助模型决定何时使用它",
  inputSchema: z.object({ /* 工具接受的参数 */ }),
  execute: async (args) => { /* 实际实现 */ }
}
```

- **description** - 对模型理解何时使用此工具至关重要。要具体明确。
- **inputSchema** - 定义参数的 Zod 模式。模型使用它来生成有效的参数。
- **execute** - 当工具被调用时运行的函数。返回模型可用的结果。

## 代码

### src/agent/tools/dateTime.ts

创建你的第一个工具——获取当前日期和时间：

```typescript
import { tool } from "ai";
import { z } from "zod";

export const getDateTime = tool({
  description: "Get the current date and time",
  inputSchema: z.object({}),
  execute: async () => {
    return new Date().toISOString();
  },
});
```

[译者注：`inputSchema: z.object({})` 表示这个工具不需要任何参数。]

### src/agent/tools/index.ts

从中央索引导出所有工具：

```typescript
import { getDateTime } from "./dateTime.ts";

// All tools combined for the agent
export const tools = {
  getDateTime,
};
```

[译者注：将所有工具集中到一个对象中，方便在 agent 中统一注册和管理。]

### src/agent/executeTool.ts

创建一个按名称执行工具的辅助函数：

```typescript
import { tools } from "./tools/index.ts";

export type ToolName = keyof typeof tools;

export async function executeTool(
  name: string,
  args: Record<string, unknown>,
): Promise<string> {
  const tool = tools[name as ToolName];

  if (!tool) {
    return `Unknown tool: ${name}`;
  }

  const execute = tool.execute;
  if (!execute) {
    // Provider tools (like webSearch) are executed by OpenAI, not us
    return `Provider tool ${name} - executed by model provider`;
  }

  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const result = await execute(args as any, {
    toolCallId: "",
    messages: [],
  });

  return String(result);
}
```

[译者注：这个函数处理两种工具：
1. 本地工具 - 有 `execute` 函数，由我们执行
2. Provider 工具 - 由模型提供商（如 OpenAI）直接执行，例如 webSearch
]

### src/agent/run.ts

将工具添加到 generateText 调用中：

```typescript
import { generateText, type ModelMessage } from "ai";
import { openai } from "@ai-sdk/openai";

import { tools } from "./tools/index.ts";
import { SYSTEM_PROMPT } from "./system/prompt.ts";

import type { AgentCallbacks } from "../types.ts";

const MODEL_NAME = "gpt-5-mini";

export async function runAgent(
  userMessage: string,
  conversationHistory: ModelMessage[],
  callbacks: AgentCallbacks,
): Promise<any> {
  // Filter and check if we need to compact the conversation history before starting
  const { text } = await generateText({
    model: openai(MODEL_NAME),
    prompt: userMessage,
    system: SYSTEM_PROMPT,
    tools,
  });

  console.log(text);
}
```

[译者注：将 `tools` 对象传递给 `generateText`，这样模型就可以看到可用的工具并选择是否调用它们。]

## 关键要点

1. **工具是声明式的** - 你描述它们的作用，模型决定何时使用它们
2. **好的描述很重要** - 模型依赖你的描述来选择正确的工具
3. **模式验证** - Zod 模式确保模型生成有效的参数
4. **你控制执行** - 模型请求，你执行。这是一个安全边界。

