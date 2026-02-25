# 单轮评估测试（Single Turn Evals）

## 概述

本课程介绍如何评估 AI 智能体，重点是单轮工具选择评估和通过链路追踪（tracing）实现的可观测性。我们将使用 Laminar 运行评估，使用开放可观测性标准（OpenTelemetry, OTEL）追踪智能体行为。

## 为什么需要评估测试

评估测试（Evals，evaluations 的缩写）对 AI 智能体开发至关重要，原因如下：

1. **智能体具有非确定性** - 相同的输入可能产生不同的输出，使得传统测试方法不足以评估智能体
2. **质量回归** - 模型更新、提示词变更或工具修改可能会悄无声息地降低性能
3. **部署信心** - 在发布变更之前，你需要可量化的指标
4. **调试** - 当出现问题时，你需要理解智能体*为什么*做出了某些决策

[译者注：非确定性（non-deterministic）是指 LLM 的输出具有随机性，即使相同的输入也可能产生不同的结果。这是由于 LLM 的温度（temperature）参数和其他采样策略导致的。]

没有评估测试，你就像是在盲目飞行。你可能根据一些手动测试认为智能体工作得很好，但在生产环境中它可能以你从未预料的方式失败。

## 离线评估 vs 在线评估

### 离线评估
- 在部署前对**固定数据集**运行
- 测试你精心策划的特定场景
- 开发过程中快速反馈
- 非常适合**回归测试** - 确保变更不会破坏现有行为
- **这就是我们今天要构建的内容**

### 在线评估
- 在**生产环境**中针对真实用户流量运行
- 捕获测试用例未预料到的问题
- 使用采样来评估一定比例的请求
- 通常使用 LLM 作为裁判进行质量评分
- 成本更高，但能捕获真实世界的边缘情况

## 评估数据的来源

1. **合成数据** - 根据预期用例自己编写示例（我们正在做的）
2. **生产日志** - 采样真实用户交互（最适合在线评估）
3. **边缘情况挖掘** - 在生产中发现失败案例并添加到测试套件
4. **红队测试** - 故意尝试破坏智能体并捕获这些案例
5. **LLM 生成** - 使用另一个 LLM 生成测试用例（注意偏见）

最好的评估数据集是所有这些的组合。从合成数据开始，然后持续添加生产失败案例。

## 使用评估测试进行爬山优化

评估测试使你能够进行**爬山优化（hill climbing）** - 迭代地改进智能体：

1. 运行评估测试，获取基线分数
2. 进行变更（提示词、模型、工具等）
3. 再次运行评估测试
4. 如果分数提高，保留变更；如果没有，回滚
5. 重复

[译者注：爬山优化是一种启发式搜索算法，通过在当前解的邻域中寻找更好的解来逐步优化。在 AI 智能体开发中，这意味着通过不断尝试小改进并根据评估指标决定是否保留，来逐步提高智能体的性能。]

这就是你如何系统地改进智能体，而不依赖于"感觉"。每个变更都有数据支持。

## 核心概念

### 单轮评估
单轮评估测试**一次交互** - 一条用户消息和智能体的即时响应。它们非常适合测试：
- **工具选择** - 智能体是否选择了正确的工具？
- **参数提取** - 它是否提取了正确的参数？
- **拒绝行为** - 在不适当的情况下它是否正确地不使用工具？

单轮评估快速、便宜，并且能高信度地告诉你智能体是否理解何时使用哪些工具。

### 评估类别

我们使用三个类别来组织测试用例：

1. **黄金类（Golden）** - 必须选择完全符合预期的工具，不允许任何歧义
2. **次要类（Secondary）** - 可能选择某些工具，但有灵活性。根据精确率/召回率评分
3. **负面类（Negative）** - 必须不选择被禁止的工具。测试智能体是否过度行动

[译者注：精确率（precision）是指被选中的工具中正确的比例，召回率（recall）是指应该被选中的工具中被正确选中的比例。F1 分数是精确率和召回率的调和平均值。]

### 评分器（Evaluators）

评分器是接收智能体输出和预期目标，返回分数（通常为 0-1）的函数：

- `toolsSelected` - 二进制：它是否选择了所有预期工具？
- `toolsAvoided` - 二进制：它是否避免了所有被禁止的工具？
- `toolSelectionScore` - F1 分数（精确率/召回率平衡），用于部分得分

## 使用 Laminar 进行链路追踪和可观测性

### 为什么链路追踪很重要

当智能体做出决策时，你需要理解：
- 它有什么上下文？
- 它考虑了哪些工具？
- 为什么选择那个工具？
- 每一步花了多长时间？

OTEL（开放可观测性标准）链路追踪通过分层 spans（跨度）显示完整的执行流程，为你提供这种可见性。

[译者注：Span（跨度）是 OpenTelemetry 中的基本单元，表示一个命名的时间区间。Spans 可以嵌套形成父子关系，从而表示操作的层次结构。]

### 设置 Laminar

1. 在 [laminar.ai](https://www.lmnr.ai/) 创建免费账户
2. 创建一个新项目
3. 复制你的 API 密钥
4. 添加到你的 `.env` 文件：
   ```
   LMNR_API_KEY=your_api_key_here
   ```

Laminar 提供：
- 链路追踪可视化
- 评估执行和跟踪
- 随时间聚合的分数
- 数据集管理

## 代码

### src/agent/run.ts

向智能体添加 Laminar 初始化和 OTEL 链路追踪：

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
  // Filter and check if we need to compact the conversation history before starting
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
      // If we have some text, continue processing
      // Otherwise, rethrow if it's not a "no output" error
      if (
        !currentText &&
        !streamError.message.includes("No output generated")
      ) {
        throw streamError;
      }
    }

    fullResponse += currentText;

    // If stream errored with "no output" and we have no text, try to recover
    if (streamError && !currentText) {
      // Add a fallback response
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

### evals/evaluators.ts

添加用于评估工具选择的评分器函数：

```typescript
export function toolsSelected(
  output: SingleTurnResult | MultiTurnResult,
  target: EvalTarget | MultiTurnTarget,
): number {
  const expectedTools =
    "expectedTools" in target
      ? target.expectedTools
      : "expectedToolOrder" in target
        ? target.expectedToolOrder
        : undefined;

  if (!expectedTools?.length) return 1;

  const selected = new Set(
    "toolNames" in output ? output.toolNames : output.toolsUsed,
  );

  return expectedTools.every((t) => selected.has(t)) ? 1 : 0;
}

/**
 * Evaluator: Check if forbidden tools were avoided.
 * Returns 1 if NONE of the forbidden tools are in the output, 0 otherwise.
 * For negative prompts.
 */
export function toolsAvoided(
  output: SingleTurnResult | MultiTurnResult,
  target: EvalTarget | MultiTurnTarget,
): number {
  if (!target.forbiddenTools?.length) return 1;

  const selected = new Set(
    "toolNames" in output ? output.toolNames : output.toolsUsed,
  );

  return target.forbiddenTools.some((t) => selected.has(t)) ? 0 : 1;
}

/**
 * Evaluator: Check if tools were called in the expected order.
 * Returns the fraction of expected tools found in sequence.
 * Order matters but tools don't need to be consecutive.
 */
export function toolOrderCorrect(
  output: MultiTurnResult,
  target: MultiTurnTarget,
): number {
  if (!target.expectedToolOrder?.length) return 1;

  const actualOrder = output.toolCallOrder;

  // Check if expected tools appear in order (not necessarily consecutive)
  let expectedIdx = 0;
  for (const toolName of actualOrder) {
    if (toolName === target.expectedToolOrder[expectedIdx]) {
      expectedIdx++;
      if (expectedIdx === target.expectedToolOrder.length) break;
    }
  }

  return expectedIdx / target.expectedToolOrder.length;
}
```

### evals/executors.ts

添加带有模拟工具的单轮执行器：

```typescript
import { generateText, stepCountIs, tool, type ToolSet } from "ai";
import { openai } from "@ai-sdk/openai";
import { z } from "zod";

import type {
  EvalData,
  SingleTurnResult,
  MultiTurnEvalData,
  MultiTurnResult,
} from "./types.ts";
import { buildMessages, buildMockedTools } from "./utils.ts";

/**
 * Tool definitions for mocked single-turn evaluations.
 * These define the schema the LLM sees without real implementations.
 */
const TOOL_DEFINITIONS: Record<
  string,
  { description: string; parameters: z.ZodObject<z.ZodRawShape> }
> = {
  // File tools
  readFile: {
    description: "Read the contents of a file at the specified path",
    parameters: z.object({
      path: z.string().describe("The path to the file to read"),
    }),
  },
  writeFile: {
    description: "Write content to a file at the specified path",
    parameters: z.object({
      path: z.string().describe("The path to the file to write"),
      content: z.string().describe("The content to write to the file"),
    }),
  },
  listFiles: {
    description: "List all files in a directory",
    parameters: z.object({
      path: z.string().describe("The directory path to list files from"),
    }),
  },
  deleteFile: {
    description: "Delete a file at the specified path",
    parameters: z.object({
      path: z.string().describe("The path to the file to delete"),
    }),
  },
  // Shell tools
  runCommand: {
    description: "Execute a shell command and return its output",
    parameters: z.object({
      command: z.string().describe("The shell command to execute"),
    }),
  },
};

/**
 * Single-turn executor with mocked tools.
 * Uses predefined tool definitions - tools never execute, only selection is tested.
 */
export async function singleTurnWithMocks(
  data: EvalData,
): Promise<SingleTurnResult> {
  const messages = buildMessages(data);

  // Build mocked tools from definitions
  const tools: ToolSet = {};
  for (const toolName of data.tools) {
    const def = TOOL_DEFINITIONS[toolName];
    if (def) {
      tools[toolName] = tool({
        description: def.description,
        inputSchema: def.parameters,
      });
    }
  }

  const result = await generateText({
    model: openai(data.config?.model ?? "gpt-4o-mini"),
    messages,
    tools,
    stopWhen: stepCountIs(1),
    temperature: data.config?.temperature ?? undefined,
  });

  // Extract tool calls from the result
  const toolCalls = (result.toolCalls ?? []).map((tc) => ({
    toolName: tc.toolName,
    args: "args" in tc ? tc.args : {},
  }));

  const toolNames = toolCalls.map((tc) => tc.toolName);

  return {
    toolCalls,
    toolNames,
    selectedAny: toolNames.length > 0,
  };
}

/**
 * Multi-turn executor with mocked tools.
 * Runs a complete agent loop with tools returning fixed values.
 */
```

### evals/file-tools.eval.ts

创建文件工具选择的评估文件：

```typescript
import { evaluate } from "@lmnr-ai/lmnr";
import {
  toolsSelected,
  toolsAvoided,
  toolSelectionScore,
} from "./evaluators.ts";
import type { EvalData, EvalTarget } from "./types.ts";
import dataset from "./data/file-tools.json" with { type: "json" };
import { singleTurnWithMocks } from "./executors.ts";

/**
 * File Tools Selection Evaluation
 *
 * Tests whether the LLM correctly selects file-related tools
 * (readFile, writeFile, listFiles, deleteFile) based on user prompts.
 *
 * Categories:
 * - golden: Must select specific expected tools
 * - secondary: Likely selects certain tools, scored on precision/recall
 * - negative: Must NOT select any file tools
 */

// Executor that runs single-turn tool selection with mocked tools
const executor = async (data: EvalData) => {
  return singleTurnWithMocks(data);
};

// Run the evaluation
evaluate({
  data: dataset as Array<{ data: EvalData; target: EvalTarget }>,
  executor,
  evaluators: {
    // For golden prompts: did it select all expected tools?
    toolsSelected: (output, target) => {
      if (target?.category !== "golden") return 1; // Skip for non-golden
      return toolsSelected(output, target);
    },
    // For negative prompts: did it avoid forbidden tools?
    toolsAvoided: (output, target) => {
      if (target?.category !== "negative") return 1; // Skip for non-negative
      return toolsAvoided(output, target);
    },
    // For secondary prompts: precision/recall score
    selectionScore: (output, target) => {
      if (target?.category !== "secondary") return 1; // Skip for non-secondary
      return toolSelectionScore(output, target);
    },
  },
  config: {
    projectApiKey: process.env.LMNR_API_KEY,
  },
  groupName: "file-tools-selection",
});
```

### evals/shell-tools.eval.ts

创建 Shell 工具选择的评估文件：

```typescript
import { evaluate } from "@lmnr-ai/lmnr";
import {
  toolsSelected,
  toolsAvoided,
  toolSelectionScore,
} from "./evaluators.ts";
import type { EvalData, EvalTarget } from "./types.ts";
import dataset from "./data/shell-tools.json" with { type: "json" };
import { singleTurnWithMocks } from "./executors.ts";

/**
 * Shell Tools Selection Evaluation
 *
 * Tests whether the LLM correctly selects the shell command tool
 * (runCommand) based on user prompts.
 *
 * Categories:
 * - golden: Must select runCommand for explicit shell requests
 * - secondary: Likely selects runCommand, scored on precision/recall
 * - negative: Must NOT use shell for non-shell tasks
 */

// Executor that runs single-turn tool selection with mocked tools
const executor = async (data: EvalData) => {
  return singleTurnWithMocks(data);
};

// Run the evaluation
evaluate({
  data: dataset as Array<{ data: EvalData; target: EvalTarget }>,
  executor,
  evaluators: {
    // For golden prompts: did it select runCommand?
    toolsSelected: (output, target) => {
      if (target?.category !== "golden") return 1;
      return toolsSelected(output, target);
    },
    // For negative prompts: did it avoid runCommand?
    toolsAvoided: (output, target) => {
      if (target?.category !== "negative") return 1;
      return toolsAvoided(output, target);
    },
    // For secondary prompts: precision/recall score
    selectionScore: (output, target) => {
      if (target?.category !== "secondary") return 1;
      return toolSelectionScore(output, target);
    },
  },
  config: {
    projectApiKey: process.env.LMNR_API_KEY,
  },
  groupName: "shell-tools-selection",
});
```

### evals/data/file-tools.json

文件工具评估的示例数据集：

```json
[
  {
    "data": {
      "prompt": "Read the contents of package.json",
      "tools": ["readFile", "writeFile", "listFiles", "deleteFile"]
    },
    "target": {
      "expectedTools": ["readFile"],
      "category": "golden"
    },
    "metadata": {
      "description": "Direct file read request - should use readFile"
    }
  },
  {
    "data": {
      "prompt": "Show me all the files in the src directory",
      "tools": ["readFile", "writeFile", "listFiles", "deleteFile"]
    },
    "target": {
      "expectedTools": ["listFiles"],
      "category": "golden"
    },
    "metadata": {
      "description": "Directory listing request - should use listFiles"
    }
  },
  {
    "data": {
      "prompt": "Create a new file called hello.txt with the content 'Hello World'",
      "tools": ["readFile", "writeFile", "listFiles", "deleteFile"]
    },
    "target": {
      "expectedTools": ["writeFile"],
      "category": "golden"
    },
    "metadata": {
      "description": "File creation request - should use writeFile"
    }
  },
  {
    "data": {
      "prompt": "What's in this project? Show me around.",
      "tools": ["readFile", "writeFile", "listFiles", "deleteFile"]
    },
    "target": {
      "expectedTools": ["listFiles"],
      "category": "secondary"
    },
    "metadata": {
      "description": "Ambiguous exploration request - likely uses listFiles"
    }
  },
  {
    "data": {
      "prompt": "What is the capital of France?",
      "tools": ["readFile", "writeFile", "listFiles", "deleteFile"]
    },
    "target": {
      "forbiddenTools": ["readFile", "writeFile", "listFiles", "deleteFile"],
      "category": "negative"
    },
    "metadata": {
      "description": "General knowledge question - should NOT use any file tools"
    }
  }
]
```

### evals/data/shell-tools.json

Shell 工具评估的示例数据集：

```json
[
  {
    "data": {
      "prompt": "Run npm install to install the dependencies",
      "tools": ["runCommand"]
    },
    "target": {
      "expectedTools": ["runCommand"],
      "category": "golden"
    },
    "metadata": {
      "description": "Package installation request - should use runCommand"
    }
  },
  {
    "data": {
      "prompt": "Check the git status of this repository",
      "tools": ["runCommand"]
    },
    "target": {
      "expectedTools": ["runCommand"],
      "category": "golden"
    },
    "metadata": {
      "description": "Git status request - should use runCommand"
    }
  },
  {
    "data": {
      "prompt": "What is TypeScript used for?",
      "tools": ["runCommand"]
    },
    "target": {
      "forbiddenTools": ["runCommand"],
      "category": "negative"
    },
    "metadata": {
      "description": "General knowledge question - should NOT use shell"
    }
  }
]
```

## 运行评估测试

```bash
# 运行文件工具评估
npx tsx evals/file-tools.eval.ts

# 运行 Shell 工具评估
npx tsx evals/shell-tools.eval.ts
```

结果将在你的 Laminar 仪表板中可见，显示每个评分器的分数和聚合指标。
