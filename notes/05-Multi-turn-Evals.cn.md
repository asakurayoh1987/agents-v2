# 多轮评估 (Multi-turn Evals)

## 概述

单轮评估测试工具选择——模型是否选择了正确的工具？多轮评估测试完整的 Agent 循环——Agent 是否在多个步骤中成功完成了任务？这是我们端到端评估 Agent 行为的地方。[译者注：单轮评估只检查一次工具调用，多轮评估检查整个对话过程中 Agent 的行为]

## 为什么多轮评估很重要

单轮评估回答："给定这个提示，模型是否调用了正确的工具？"

但 Agent 不是单轮工作的。它们会：
1. 接收任务
2. 调用工具
3. 处理结果
4. 决定下一步做什么
5. 调用另一个工具（或响应）
6. 重复直到完成

多轮评估回答："给定这个任务，Agent 是否正确完成了它？"

这能捕获单轮评估遗漏的失败情况：
- Agent 选择了正确的第一个工具，但选择了错误的第二个工具
- Agent 陷入循环
- Agent 误解了工具结果
- Agent 过早放弃
- Agent 不知道何时停止

## 挑战：非确定性输出

单轮评估可以相当确定——模型调用了 `readFile` 吗？

多轮评估则很混乱：
- Agent 可能通过不同的有效路径达到同一目标
- 工具调用顺序可能不同，但仍然是正确的
- 最终响应的措辞每次运行都不同
- 中间推理过程不同

当"正确答案"不是一个固定字符串时，你如何评估？

## LLM 作为评判器 (LLM-as-Judge)

解决方案：使用另一个 LLM 来评估输出。

我们不再检查 `output === expected`，而是询问评判器模型：
- "给定这个任务和这些工具结果，这个响应正确吗？"
- "这个回答合理吗？"
- "Agent 是否完成了目标？"

### 为什么 LLM 作为评判器有效

**语义理解**：评判器理解含义，而不仅仅是字符串匹配。"文件包含 'hello world'" 和 "文件内容：hello world" 都是正确的。

**灵活的标准**：你可以用自然语言定义评估标准："如果 Agent 解释了其推理，则给更高分。"

**处理变体**：不同的有效方法被识别为有效。

### 为什么 LLM 作为评判器有局限性

**成本**：每次评估都需要 LLM 调用。运行 1000 次评估意味着 1000 次评判器调用。

**延迟**：比确定性检查慢。

**不一致性**：评判器本身是非确定性的。相同的输出可能一次得到 8/10，下一次得到 7/10。

**偏见**：评判器模型有自己的偏见。它们可能更喜欢冗长的响应或某些措辞。

**博弈**：如果你知道评判器标准，你（或 Agent）可以针对评判器进行优化，而不是针对实际质量。

### 使 LLM 作为评判器更可靠

**使用结构化输出**：不要要求自由形式的评估。使用模式（schema）：
```typescript
const judgeSchema = z.object({
  score: z.number().min(1).max(10),
  reason: z.string(),
});
```

**使用更强的模型**：评判器应该至少与被评估的 Agent 一样有能力。我们使用具有高推理力（reasoning effort）的推理模型。

**明确的标准**：准确定义 1-10 的含义：
- 10：使用工具结果正确完成任务的全部内容
- 7-9：基本正确，有轻微问题
- 4-6：部分完成任务
- 1-3：大部分不正确或无关

**多个评判器**：通过多次评判器调用运行相同的评估，对分数取平均。

## 多轮评估数据策略

多轮评估最困难的部分是设计测试数据。

### 每个测试案例需要什么

1. **输入**：用户的任务或预填充的对话
2. **可用工具**：Agent 可以使用哪些工具
3. **模拟工具结果**：每个工具在调用时返回什么
4. **预期行为**：应该发生什么
5. **评估标准**：如何判断成功

### 输入策略

**新任务**：只有一个用户提示。Agent 从头开始。
```json
{
  "prompt": "读取配置文件并告诉我数据库主机"
}
```

**对话中**：预填充的消息历史。测试延续行为。
```json
{
  "messages": [
    { "role": "user", "content": "我需要更新配置" },
    { "role": "assistant", "content": "我来帮你。什么更改？" },
    { "role": "user", "content": "将端口更改为 8080" }
  ]
}
```

### 模拟工具结果

为了确定性测试，工具返回固定值：
```json
{
  "mockTools": {
    "readFile": {
      "description": "读取文件内容",
      "result": "DB_HOST=localhost\nDB_PORT=5432"
    },
    "writeFile": {
      "description": "写入文件",
      "result": "成功写入 45 个字符"
    }
  }
}
```

Agent 看到真实的工具模式，但得到预制的响应。这可以：
- 使测试可重现
- 避免文件系统副作用
- 让你测试边缘情况（如果文件未找到怎么办？）
- 加快评估（没有实际 I/O）

### 预期行为

你可以检查多个事情：

**工具顺序**：工具是否按正确的顺序调用了？
```json
{
  "expectedToolOrder": ["readFile", "writeFile"]
}
```

**禁止的工具**：是否避免了某些工具？
```json
{
  "forbiddenTools": ["deleteFile", "runCommand"]
}
```

**输出质量**：响应是否合理？（LLM 评判器）
```json
{
  "originalTask": "读取配置并报告数据库主机",
  "mockToolResults": { "readFile": "DB_HOST=localhost" }
}
```

## 结合评估器

多轮评估通常使用多个评估器：

1. **toolOrderCorrect**：工具是否按预期顺序执行？
2. **toolsAvoided**：是否未调用禁止的工具？
3. **llmJudge**：最终响应是否合理？

每个都返回 0-1 的分数。你可以对它们进行不同的加权或要求全部通过。

## 代码

### evals/evaluators.ts

添加 LLM 作为评判器的评估器：

```typescript
const judgeSchema = z.object({
  score: z
    .number()
    .min(1)
    .max(10)
    .describe("Score from 1-10 where 10 is perfect"),
  reason: z.string().describe("Brief explanation for the score"),
});

/**
 * Evaluator: LLM-as-judge for output quality.
 * Uses structured output to reliably assess if the agent's response is correct.
 * Returns a score from 0-1 (internally uses 1-10 scale divided by 10).
 */
export async function llmJudge(
  output: MultiTurnResult,
  target: MultiTurnTarget,
): Promise<number> {
  const result = await generateObject({
    model: openai("gpt-5.1"),
    schema: judgeSchema,
    schemaName: "evaluation",
    providerOptions: {
      openai: {
        reasoningEffort: "high",
      },
    },
    schemaDescription: "Evaluation of an AI agent response",
    messages: [
      {
        role: "system",
        content: `You are an evaluation judge. Score the agent's response on a scale of 1-10.

Scoring criteria:
- 10: Response fully addresses the task using tool results correctly
- 7-9: Response is mostly correct with minor issues
- 4-6: Response partially addresses the task
- 1-3: Response is mostly incorrect or irrelevant`,
      },
      {
        role: "user",
        content: `Task: ${target.originalTask}

Tools called: ${JSON.stringify(output.toolCallOrder)}
Tool results provided: ${JSON.stringify(target.mockToolResults)}

Agent's final response:
${output.text}

Evaluate if this response correctly uses the tool results to answer the task.`,
      },
    ],
  });

  // Convert 1-10 score to 0-1 range
  return result.object.score / 10;
}
```

关键实现细节：
- 使用 `generateObject` 进行结构化输出（保证模式）
- 1-10 分数转换为 0-1 以与其他评估器一致
- 高推理力（reasoning effort）以获得更好的判断
- 系统提示中有明确的评分标准
- 提供完整的上下文：任务、调用的工具、工具结果、Agent 响应

### evals/executors.ts

添加带有模拟工具的多轮执行器：

```typescript
import { SYSTEM_PROMPT } from "../src/agent/system/prompt.ts";

/**
 * Multi-turn executor with mocked tools.
 * Runs a complete agent loop with tools returning fixed values.
 */
export async function multiTurnWithMocks(
  data: MultiTurnEvalData,
): Promise<MultiTurnResult> {
  const tools = buildMockedTools(data.mockTools);

  // Build messages from either prompt or pre-filled history
  const messages: ModelMessage[] = data.messages ?? [
    { role: "system", content: SYSTEM_PROMPT },
    { role: "user", content: data.prompt! },
  ];

  const result = await generateText({
    model: openai(data.config?.model ?? "gpt-5-mini"),
    messages,
    tools,
    stopWhen: stepCountIs(data.config?.maxSteps ?? 20),
  });

  // Extract all tool calls in order from steps
  const allToolCalls: string[] = [];
  const steps = result.steps.map((step) => {
    const stepToolCalls = (step.toolCalls ?? []).map((tc) => {
      allToolCalls.push(tc.toolName);
      return {
        toolName: tc.toolName,
        args: "args" in tc ? tc.args : {},
      };
    });

    const stepToolResults = (step.toolResults ?? []).map((tr) => ({
      toolName: tr.toolName,
      result: "result" in tr ? tr.result : tr,
    }));

    return {
      toolCalls: stepToolCalls.length > 0 ? stepToolCalls : undefined,
      toolResults: stepToolResults.length > 0 ? stepToolResults : undefined,
      text: step.text || undefined,
    };
  });

  // Extract unique tools used
  const toolsUsed = [...new Set(allToolCalls)];

  return {
    text: result.text,
    steps,
    toolsUsed,
    toolCallOrder: allToolCalls,
  };
}
```

关键实现细节：
- 使用 `buildMockedTools` 创建具有固定返回值的工具
- 支持新提示和预填充的消息历史
- `stopWhen: stepCountIs(20)` 防止无限循环
- 捕获完整的逐步执行跟踪
- 返回使用的唯一工具和完整调用顺序

### evals/agent-multiturn.eval.ts

完整的多轮评估文件：

```typescript
import { evaluate } from "@lmnr-ai/lmnr";
import { toolOrderCorrect, toolsAvoided, llmJudge } from "./evaluators.ts";
import type {
  MultiTurnEvalData,
  MultiTurnTarget,
  MultiTurnResult,
} from "./types.ts";
import dataset from "./data/agent-multiturn.json" with { type: "json" };
import { multiTurnWithMocks } from "./executors.ts";

/**
 * Multi-Turn Agent Evaluation
 *
 * Tests full agent behavior with mocked tools:
 * 1. Fresh task: User's first message, check tools + order + LLM judge
 * 2. Mid-conversation: Pre-filled messages, check continuation behavior
 * 3. Negative: Ensure wrong tool category not used (file vs shell)
 *
 * All tools are mocked to return fixed values for deterministic testing.
 *
 * Evaluators:
 * - toolOrderCorrect: Did tools get called in expected sequence?
 * - toolsAvoided: Were forbidden tools not called?
 * - llmJudge: Does the final response make sense given the task and results?
 */

// Executor that runs multi-turn agent with mocked tools
const executor = async (data: MultiTurnEvalData): Promise<MultiTurnResult> => {
  return multiTurnWithMocks(data);
};

// Run the evaluation
evaluate({
  data: dataset as unknown as Array<{
    data: MultiTurnEvalData;
    target: MultiTurnTarget;
  }>,
  executor,
  evaluators: {
    // Check if tools were called in the expected order
    toolOrder: (output, target) => {
      if (!target) return 1;
      return toolOrderCorrect(output, target);
    },
    // Check if forbidden tools were avoided
    toolsAvoided: (output, target) => {
      if (!target?.forbiddenTools?.length) return 1;
      return toolsAvoided(output, target);
    },
    // LLM judge to evaluate output quality
    outputQuality: async (output, target) => {
      if (!target) return 1;
      return llmJudge(output, target);
    },
  },
  config: {
    projectApiKey: process.env.LMNR_API_KEY,
  },
  groupName: "agent-multiturn",
});
```

关键实现细节：
- 每个测试案例运行三个评估器
- 如果没有目标要检查，评估器返回 1（通过）
- `toolOrder` 检查顺序，`toolsAvoided` 检查禁止的工具，`outputQuality` 使用 LLM 评判器
- 数据集从包含测试案例的 JSON 文件加载

## 测试案例示例

### 新任务测试

```json
{
  "data": {
    "prompt": "Read the config.json file and tell me the API endpoint",
    "mockTools": {
      "readFile": {
        "description": "Read file contents",
        "result": "{\"apiEndpoint\": \"https://api.example.com/v1\"}"
      }
    }
  },
  "target": {
    "expectedToolOrder": ["readFile"],
    "forbiddenTools": ["writeFile", "deleteFile"],
    "originalTask": "Read config.json and report the API endpoint",
    "mockToolResults": {
      "readFile": "{\"apiEndpoint\": \"https://api.example.com/v1\"}"
    }
  }
}
```

### 对话中测试

```json
{
  "data": {
    "messages": [
      { "role": "system", "content": "You are a helpful assistant..." },
      { "role": "user", "content": "I need to update a config file" },
      { "role": "assistant", "content": "Sure, which file and what changes?" },
      { "role": "user", "content": "Change port to 3000 in config.json" }
    ],
    "mockTools": {
      "readFile": {
        "description": "Read file contents",
        "result": "{\"port\": 8080}"
      },
      "writeFile": {
        "description": "Write to file",
        "result": "Written successfully"
      }
    }
  },
  "target": {
    "expectedToolOrder": ["readFile", "writeFile"],
    "originalTask": "Update port to 3000 in config.json"
  }
}
```

### 负面测试（禁止的工具）

```json
{
  "data": {
    "prompt": "What is 2 + 2?",
    "mockTools": {
      "readFile": { "description": "Read file", "result": "" },
      "runCommand": { "description": "Run shell command", "result": "" }
    }
  },
  "target": {
    "forbiddenTools": ["readFile", "runCommand", "writeFile"],
    "originalTask": "Simple math question - should not use any tools"
  }
}
```

## 为什么在评估中使用模拟工具？

你可能会问：为什么不使用真实的工具？

**可重现性**：真实文件系统在运行之间会发生变化。模拟每次都返回相同的值。

**速度**：没有实际的 I/O、网络调用或副作用。

**安全性**：在测试期间无法意外删除文件或运行危险命令。

**边缘情况**：通过设置模拟结果可以轻松测试"文件未找到"或"权限被拒绝"。

**隔离**：每个测试案例都是独立的。不需要清理。

权衡：你没有测试真实的工具实现。但这正是单元测试的目的。评估测试 Agent 的决策，而不是工具本身。
