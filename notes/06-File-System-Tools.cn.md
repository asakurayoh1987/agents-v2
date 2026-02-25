# 文件系统工具

## 概述

文件系统访问是你可以赋予智能体最强大的能力之一。表面上它很简单——读取文件、写入文件、列出目录。但其影响远超基本的 I/O 操作。文件成为智能体的记忆、工作空间以及与更广泛系统的接口。

## 为什么文件对智能体很重要

### 显而易见的用途

这些直接的用例正如你所预期：
- **读取源代码** 以理解代码库
- **编写代码文件** 当实现功能时
- **读取配置** (package.json, .env 等)
- **写入输出** (报告、生成的内容、导出)
- **管理数据** (CSV, JSON, 日志)

仅这些就使智能体很有用。但文件解锁了更多可能性。

### 文件作为智能体记忆

LLM 是无状态的。每个请求都是独立的——模型在调用之间没有记忆。对话历史只是我们每次重新发送的文本。

文件改变了这一点。具有文件访问权限的智能体可以：
- 写笔记以便以后记住
- 存储研究发现
- 追踪已尝试的方法
- 跨会话积累知识

```
# 智能体的草稿板.md
## 已尝试的解决方案
- 尝试方法 A：因 X 失败
- 尝试方法 B：部分成功，被 Y 阻塞

## 关键发现
- 认证系统使用 JWT 令牌
- 速率限制为 100 请求/分钟
- 配置文件位于 /etc/app/config.yaml
```

这是持久记忆。智能体可以在下次会话中读取此文件并从中断处继续。

### 文件作为状态

复杂任务需要跟踪状态：
- 我们在哪个步骤？
- 已完成什么？
- 待处理什么？

你可以在对话中跟踪这一点，但这会变得混乱并消耗 token。文件更简洁：

```json
{
  "task": "migrate-database",
  "currentStep": 3,
  "completed": ["backup", "schema-update"],
  "pending": ["data-migration", "verification"],
  "metadata": {
    "startedAt": "2024-01-15T10:30:00Z",
    "backupLocation": "/backups/db-20240115.sql"
  }
}
```

智能体写入此文件，稍后读取它，随着工作进展更新它。状态在会话、崩溃和上下文窗口重置之间持久存在。

### 文件作为临时草稿板

有时智能体需要"大声思考"，需要比上下文窗口允许的更多空间：
- 转储大型 API 响应以逐段分析
- 编写中间计算
- 逐步存储数据转换

```typescript
// 智能体的工作流：
// 1. 获取大型数据集 → 写入 /tmp/data.json
// 2. 读取前 100 条记录，分析
// 3. 将分析写入 /tmp/analysis-part1.md
// 4. 读取下 100 条记录，分析
// 5. 将分析合并为最终报告
```

文件系统成为外部工作记忆。智能体分块处理数据而不会压倒其上下文窗口。

### 文件作为上下文加载

与临时草稿板相关——智能体可以策略性地加载上下文：
- 仅读取大文件的相关部分
- 维护内容位置的索引
- 按需加载上下文而不是一次性全部加载

Claude Code 和类似工具经常这样做。它们不会将整个代码库加载到上下文中。它们根据索引和搜索结果按需读取特定文件。

### 文件作为智能体间通信

当你有多个智能体或进程时：
- 智能体 A 将结果写入文件
- 智能体 B 读取该文件作为输入
- 不需要直接通信

这简单、可靠且可调试。你可以检查中间文件以查看智能体之间传递的确切内容。

### 文件作为审计跟踪

智能体所做的一切都可以被记录：
- 执行的命令
- 做出的决策
- 遇到的错误
- 推理过程

```
# audit-log-2024-01-15.md
10:30:15 - 用户请求："修复登录 bug"
10:30:16 - 正在读取 src/auth/login.ts
10:30:18 - 识别问题：第 47 行缺少空值检查
10:30:19 - 建议修复：添加可选链
10:30:20 - 正在写入更新后的文件
10:30:21 - 正在运行测试
10:30:45 - 测试通过 (23/23)
10:30:46 - 任务完成
```

这对调试、合规性和与用户建立信任很重要。

### 文件作为工具输出存储

某些操作会产生大型输出：
- 编译结果
- 测试输出
- 搜索结果
- API 响应

与其将这些内容塞入上下文，不如将它们写入文件。智能体可以根据需要引用、搜索或总结它们。

### 文件作为配置

智能体可以读取和修改自己的行为：
- 读取配置文件以了解偏好
- 根据用户反馈更新设置
- 存储学习的模式以供将来使用

## 四个核心操作

对于大多数智能体用例，你需要四个文件操作：

### 1. 读取
理解任何内容的基础。无法修改你看不到的东西。

### 2. 写入
创建新文件或覆盖现有文件。智能体产生输出的主要方式。

### 3. 列表
导航文件系统。发现可用的内容。探索的基础。

### 4. 删除
清理临时文件。删除过时内容。重置状态。

某些实现添加更多功能（复制、移动、追加、搜索），但这四个操作涵盖大多数需求。

## 实现注意事项

### 路径处理

智能体会尝试创造性的路径：
- 相对路径：`./src/index.ts`
- 绝对路径：`/Users/scott/project/src/index.ts`
- 父级遍历：`../other-project/secrets.txt`（危险！）

决定你的策略：
- 允许任何路径？（危险但灵活）
- 限制到工作目录？（更安全）
- 允许列表特定目录？（最安全）

[译者注：这是一个重要的安全考虑。在生产环境中，应该始终限制智能体只能访问预定义的目录，并防止路径遍历攻击。]

### 错误处理

文件操作经常失败：
- 文件未找到
- 权限被拒绝
- 磁盘已满
- 路径太长
- 无效字符

返回清晰的错误消息。智能体需要了解出了什么问题才能尝试替代方案。

### 目录创建

写入文件时，父目录可能不存在：
- 如果 `new-folder/` 不存在，`/new-folder/file.txt` 会失败

大多数实现使用 `mkdir -p` 语义自动创建父目录。这减少了摩擦。

### 大文件

当智能体尝试读取 10MB 日志文件时会发生什么？
- 占用所有上下文 token
- 可能超过模型限制
- 使一切变慢

选项：
- 截断到前 N 行/字节
- 返回错误，建议特定的行范围
- 自动摘要大文件

[译者注：处理大文件时，最好让智能体知道文件大小并主动要求它指定要读取的部分，而不是盲目读取整个文件。]

### 二进制文件

图像、PDF、编译代码——并非一切都是文本。决定：
- 以清晰错误拒绝二进制文件
- 返回 base64（昂贵，通常无用）
- 仅返回元数据（文件类型、大小等）

对于大多数智能体用例，文本文件就足够了。

## 代码

### src/agent/tools/file.ts

完整的文件工具实现：

```typescript
import { tool } from "ai";
import { z } from "zod";
import fs from "fs/promises";
import path from "path";

/**
 * 读取文件内容
 */
export const readFile = tool({
  description:
    "Read the contents of a file at the specified path. Use this to examine file contents.",
  inputSchema: z.object({
    path: z.string().describe("The path to the file to read"),
  }),
  execute: async ({ path: filePath }: { path: string }) => {
    try {
      const content = await fs.readFile(filePath, "utf-8");
      return content;
    } catch (error) {
      const err = error as NodeJS.ErrnoException;
      if (err.code === "ENOENT") {
        return `Error: File not found: ${filePath}`;
      }
      return `Error reading file: ${err.message}`;
    }
  },
});

/**
 * 将内容写入文件
 */
export const writeFile = tool({
  description:
    "Write content to a file at the specified path. Creates the file if it doesn't exist, overwrites if it does.",
  inputSchema: z.object({
    path: z.string().describe("The path to the file to write"),
    content: z.string().describe("The content to write to the file"),
  }),
  execute: async ({
    path: filePath,
    content,
  }: {
    path: string;
    content: string;
  }) => {
    try {
      // 如果父目录不存在，则创建它们
      const dir = path.dirname(filePath);
      await fs.mkdir(dir, { recursive: true });

      await fs.writeFile(filePath, content, "utf-8");
      return `Successfully wrote ${content.length} characters to ${filePath}`;
    } catch (error) {
      const err = error as NodeJS.ErrnoException;
      return `Error writing file: ${err.message}`;
    }
  },
});

/**
 * 列出目录中的文件
 */
export const listFiles = tool({
  description:
    "List all files and directories in the specified directory path.",
  inputSchema: z.object({
    directory: z
      .string()
      .describe("The directory path to list contents of")
      .default("."),
  }),
  execute: async ({ directory }: { directory: string }) => {
    try {
      const entries = await fs.readdir(directory, { withFileTypes: true });
      const items = entries.map((entry) => {
        const type = entry.isDirectory() ? "[dir]" : "[file]";
        return `${type} ${entry.name}`;
      });
      return items.length > 0
        ? items.join("\n")
        : `Directory ${directory} is empty`;
    } catch (error) {
      const err = error as NodeJS.ErrnoException;
      if (err.code === "ENOENT") {
        return `Error: Directory not found: ${directory}`;
      }
      return `Error listing directory: ${err.message}`;
    }
  },
});

/**
 * 删除文件
 */
export const deleteFile = tool({
  description:
    "Delete a file at the specified path. Use with caution as this is irreversible.",
  inputSchema: z.object({
    path: z.string().describe("The path to the file to delete"),
  }),
  execute: async ({ path: filePath }: { path: string }) => {
    try {
      await fs.unlink(filePath);
      return `Successfully deleted ${filePath}`;
    } catch (error) {
      const err = error as NodeJS.ErrnoException;
      if (err.code === "ENOENT") {
        return `Error: File not found: ${filePath}`;
      }
      return `Error deleting file: ${err.message}`;
    }
  },
});
```

关键实现细节：

**readFile:**
- 使用 Node.js `fs.promises` 进行异步文件操作
- 直接将文件内容作为字符串返回
- 以清晰的错误消息处理 `ENOENT`（文件未找到）
- 其他问题的通用错误回退

**writeFile:**
- 使用 `mkdir({ recursive: true })` 自动创建父目录
- 报告写入的字节数以进行确认
- 覆盖现有文件（无追加模式）

**listFiles:**
- 使用 `withFileTypes: true` 来区分文件和目录
- 为清晰起见，使用 `[dir]` 或 `[file]` 前缀条目
- 优雅地处理空目录

**deleteFile:**
- 使用 `unlink`（删除文件，而非目录）
- 如果文件不存在则显示清晰错误
- 描述警告不可逆性

### src/agent/tools/index.ts

注册文件工具：

```typescript
import { readFile, writeFile, listFiles, deleteFile } from "./file.ts";

// 为智能体组合所有工具
export const tools = {
  readFile,
  writeFile,
  listFiles,
  deleteFile,
};

// 导出单个工具以在评估中选择性使用
export { readFile, writeFile, listFiles, deleteFile } from "./file.ts";

// 用于评估的工具集
export const fileTools = {
  readFile,
  writeFile,
  listFiles,
  deleteFile,
};
```

我们将文件工具分组以便在评估中轻松选择性使用。有时你想仅使用文件工具进行测试，而不需要 shell 或 web 访问。

## 设计决策

### 为什么使用分离的工具而不是一个"文件"工具？

我们可以使用一个带有 `operation` 参数的工具：
```typescript
file({ operation: "read", path: "foo.txt" })
file({ operation: "write", path: "foo.txt", content: "..." })
```

我们选择分离的工具是因为：
- **意图更清晰**：模型确切知道每个工具的作用
- **更好的描述**：每个工具都有集中的文档
- **更简单的模式**：每个工具只有相关参数
- **更容易的权限**：可以允许读取但阻止写入

[译者注：这是一个重要的 API 设计原则。分离的工具使 LLM 更容易理解每个工具的用途，并且可以在权限系统中进行细粒度的控制。]

### 为什么将错误作为字符串返回而不是抛出异常？

注意我们返回错误消息如 `"Error: File not found"` 而不是抛出异常。

智能体需要优雅地处理错误。如果我们抛出异常，智能体循环可能会崩溃或表现异常。通过返回错误字符串：
- 智能体在工具输出中看到错误
- 智能体可以决定如何继续（尝试不同路径、询问用户、放弃）
- 循环中不需要特殊错误处理

### 为什么自动创建目录？

写入 `/deep/nested/path/file.txt` 时，我们自动创建 `/deep/nested/path/`。

这符合开发者的期望。当你使用 `mkdir -p` 或大多数文件 API 时，父级创建是自动的。要求智能体手动创建每个目录级别将是繁琐且容易出错的。

## 常见模式

### 读取-修改-写入

```
1. readFile("config.json")
2. 解析 JSON，修改值
3. writeFile("config.json", updated)
```

智能体读取现有内容，进行更改，写回。

### 探索-然后-读取

```
1. listFiles("src/")
2. listFiles("src/components/")
3. readFile("src/components/Button.tsx")
```

导航目录结构，深入钻研，然后读取特定文件。

### 写入-然后-验证

```
1. writeFile("output.txt", content)
2. readFile("output.txt")
3. 确认内容符合预期
```

偏执但安全。捕获写入失败或意外转换。

### 临时草稿板模式

```
1. 从某个源获取大数据
2. writeFile("/tmp/scratch.json", data)
3. readFile("/tmp/scratch.json") // 或读取部分
4. 处理，写入结果
5. deleteFile("/tmp/scratch.json")
```

使用临时文件作为工作记忆，完成后清理。
