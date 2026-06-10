# Pi Skill 生命周期与 LLM 上下文组织完整例子

本文用 `pi-mono` 仓库里的真实项目 skill 作为例子，串起 Pi 的 skill 机制和每轮发送给 LLM 的上下文组织方式。

示例 skill：

```text
pi-mono/.pi/skills/add-llm-provider.md
```

示例用户请求：

```text
帮我给 packages/ai 增加一个 acme LLM provider
```

## 1. 总览

Pi 的 skill 机制可以概括为：

```text
skill 文件在磁盘上
-> 启动或 reload 时发现 skill
-> 解析 frontmatter，加载 name/description/location
-> 每轮 system prompt 放 skill 索引
-> LLM 根据索引用 read 拉取 skill 全文
-> skill 全文作为普通 user/toolResult 进入 messages
-> LLM 按 skill 指令使用普通工具完成任务
-> 历史变长后，skill 全文被 session compaction 总结掉
-> 后续需要精确流程时，再根据索引重新 read
```

Pi 每次真正发给 LLM 的上下文都是三块：

```ts
{
  systemPrompt,
  messages,
  tools
}
```

真正的 LLM 调用边界在：

```text
pi-mono/packages/agent/src/agent-loop.ts
streamAssistantResponse()
```

这里会把内部 `AgentMessage[]` 转成 LLM provider 可接受的 `Message[]`，再构造：

```ts
const llmContext = {
  systemPrompt: context.systemPrompt,
  messages: llmMessages,
  tools: context.tools,
};
```

## 2. 磁盘阶段：skill 文件存在

示例文件：

```text
pi-mono/.pi/skills/add-llm-provider.md
```

它的 frontmatter：

```yaml
---
name: add-llm-provider
description: Checklist for adding a new LLM provider to packages/ai. Covers core types, provider implementation, lazy registration, model generation, the full test matrix, coding-agent wiring, and docs.
---
```

正文是一份 checklist，说明新增 LLM provider 需要处理：

```text
packages/ai/src/types.ts
packages/ai/src/providers/
packages/ai/src/providers/register-builtins.ts
packages/ai/scripts/generate-models.ts
packages/ai/test/*
packages/coding-agent/docs/providers.md
...
```

注意：这个示例是 Pi 原生 `.pi/skills` 根目录 `.md` skill，不是目录里的 `SKILL.md`。Pi 支持这种形式。

## 3. 发现阶段：扫描 skill 路径

启动或 reload 时，`DefaultResourceLoader.reload()` 会调用 package/resource 解析逻辑，收集启用的 skill 路径。

相关入口：

```text
pi-mono/packages/coding-agent/src/core/resource-loader.ts
DefaultResourceLoader.reload()
```

项目级自动发现目录包括：

```text
.pi/skills/
.agents/skills/
```

用户级目录包括：

```text
~/.pi/agent/skills/
~/.agents/skills/
```

还可以来自：

```text
package resources
settings.json 的 skills 数组
CLI --skill <path>
extension resources_discover
```

`.pi/skills` 的扫描规则在：

```text
pi-mono/packages/coding-agent/src/core/package-manager.ts
collectSkillEntries()
```

Pi 模式下，如果扫描目录是 root，并且直接子文件是 `.md`，就把它作为 skill entry。

所以这个文件会被发现：

```text
pi-mono/.pi/skills/add-llm-provider.md
```

## 4. 加载阶段：解析 frontmatter

发现路径后，`resource-loader` 调用：

```text
pi-mono/packages/coding-agent/src/core/skills.ts
loadSkills()
```

进一步调用：

```text
loadSkillFromFile()
```

它读取 markdown，解析 frontmatter，生成类似对象：

```ts
{
  name: "add-llm-provider",
  description: "Checklist for adding a new LLM provider to packages/ai...",
  filePath: "/Users/henry/work/developer/codex/pi-mono/.pi/skills/add-llm-provider.md",
  baseDir: "/Users/henry/work/developer/codex/pi-mono/.pi/skills",
  disableModelInvocation: false
}
```

加载时的几个关键规则：

- `description` 必须存在，否则 skill 不加载。
- `name` 优先来自 frontmatter；没有时用父目录名。
- `disable-model-invocation: true` 时，不进入模型可见的 skill 索引，但仍可通过 `/skill:name` 手动调用。
- 同名 skill 冲突时，先加载的胜出，后加载的记录 collision diagnostic。

## 5. System Prompt 阶段：只放 skill 索引

`AgentSession` 会根据当前工具、项目上下文、skills 等重建 system prompt。

相关位置：

```text
pi-mono/packages/coding-agent/src/core/agent-session.ts
AgentSession._rebuildSystemPrompt()
```

它把 loaded skills 传给：

```text
pi-mono/packages/coding-agent/src/core/system-prompt.ts
buildSystemPrompt()
```

`buildSystemPrompt()` 只有在 `read` 工具可用时才追加 skill section。最终格式由：

```text
pi-mono/packages/coding-agent/src/core/skills.ts
formatSkillsForPrompt()
```

生成。

system prompt 里不是 skill 全文，而是索引：

```xml
<available_skills>
  <skill>
    <name>add-llm-provider</name>
    <description>Checklist for adding a new LLM provider to packages/ai...</description>
    <location>/Users/henry/work/developer/codex/pi-mono/.pi/skills/add-llm-provider.md</location>
  </skill>
</available_skills>
```

同时还有一句关键指令：

```text
Use the read tool to load a skill's file when the task matches its description.
```

因此，普通路径下：

- 每轮 LLM 都能看到 skill 的名字、描述、路径。
- 每轮 LLM 不会自动看到 skill 全文。
- LLM 需要自己调用 `read` 获取全文。

## 6. 用户发起任务：第 1 次 LLM 请求前

用户输入：

```text
帮我给 packages/ai 增加一个 acme LLM provider
```

进入 `AgentSession.prompt()` 后，会依次处理：

```text
extension command 检查
-> input hook
-> /skill:name 展开
-> prompt template 展开
-> 如有必要，发送前 compaction 检查
-> 构造 user message
-> 注入 pending next-turn custom messages
-> before_agent_start hook
-> 调 agent.prompt(messages)
```

相关入口：

```text
pi-mono/packages/coding-agent/src/core/agent-session.ts
AgentSession.prompt()
```

如果用户没有显式 `/skill:add-llm-provider`，此时 user message 还是普通文本：

```ts
{
  role: "user",
  content: [
    {
      type: "text",
      text: "帮我给 packages/ai 增加一个 acme LLM provider"
    }
  ],
  timestamp: ...
}
```

## 7. 第 1 次发给 LLM 的上下文

第一次 LLM 请求大致是：

```ts
{
  systemPrompt: `
    You are an expert coding assistant...

    Available tools:
    - read: Read file contents
    - bash: ...
    - edit: ...
    - write: ...

    Guidelines:
    ...

    Project-specific instructions:
    <project_instructions path=".../AGENTS.md">
    ...
    </project_instructions>

    The following skills provide specialized instructions...
    <available_skills>
      <skill>
        <name>add-llm-provider</name>
        <description>Checklist for adding a new LLM provider...</description>
        <location>.../.pi/skills/add-llm-provider.md</location>
      </skill>
    </available_skills>

    Current date: ...
    Current working directory: ...
  `,
  messages: [
    ...之前可见的 session messages,
    {
      role: "user",
      content: "帮我给 packages/ai 增加一个 acme LLM provider"
    }
  ],
  tools: [
    read,
    bash,
    edit,
    write,
    ...
  ]
}
```

这次 LLM 只知道：

```text
有一个 add-llm-provider skill
它适用于新增 LLM provider
文件位置在哪里
```

但还不知道 checklist 正文。

## 8. LLM 匹配 skill：调用 read

模型看到任务和 skill description 匹配后，理想行为是返回 tool call：

```json
{
  "name": "read",
  "arguments": {
    "path": "/Users/henry/work/developer/codex/pi-mono/.pi/skills/add-llm-provider.md"
  }
}
```

这条 assistant message 会加入当前上下文和 session：

```ts
{
  role: "assistant",
  content: [
    {
      type: "toolCall",
      name: "read",
      arguments: {
        path: ".../.pi/skills/add-llm-provider.md"
      }
    }
  ]
}
```

Pi 执行 `read` 工具，得到 tool result：

```ts
{
  role: "toolResult",
  content: [
    {
      type: "text",
      text: "---\nname: add-llm-provider\n...\n# Adding a New LLM Provider (packages/ai)\n..."
    }
  ]
}
```

从这一刻开始，skill 全文作为普通 tool result 进入 messages。

## 9. 第 2 次发给 LLM 的上下文：skill 全文可见

工具结果加入后，agent loop 会继续，再发一次 LLM 请求。

第二次上下文变成：

```ts
{
  systemPrompt: "同上，仍包含 skill 索引",
  messages: [
    ...之前可见历史,
    {
      role: "user",
      content: "帮我给 packages/ai 增加一个 acme LLM provider"
    },
    {
      role: "assistant",
      content: [
        {
          type: "toolCall",
          name: "read",
          arguments: {
            path: ".../.pi/skills/add-llm-provider.md"
          }
        }
      ]
    },
    {
      role: "toolResult",
      content: "# Adding a New LLM Provider (packages/ai)\n\nA new provider touches multiple files..."
    }
  ],
  tools: [
    read,
    bash,
    edit,
    write,
    ...
  ]
}
```

这一次 LLM 才真正看到 skill 正文。

它会知道这件事不是只改一个文件，而要按 checklist 做完整链路：

```text
1. 更新 packages/ai/src/types.ts
2. 新增 provider implementation
3. 更新 provider exports 和 lazy registration
4. 更新 model generation
5. 补测试矩阵
6. 补 coding-agent provider wiring
7. 更新文档和 changelog
```

## 10. 执行阶段：同一用户轮次内多次 LLM/tool 循环

Skill 本身不是运行时函数。它只是 instruction。

后续真正做事靠普通工具：

```text
LLM -> toolCall(read packages/ai/src/types.ts)
Pi  -> toolResult(types.ts 内容)

LLM -> toolCall(edit packages/ai/src/types.ts)
Pi  -> toolResult(edit 成功)

LLM -> toolCall(write packages/ai/src/providers/acme.ts)
Pi  -> toolResult(write 成功)

LLM -> toolCall(read packages/ai/src/providers/register-builtins.ts)
Pi  -> toolResult(...)

LLM -> toolCall(edit ...)
Pi  -> toolResult(...)

LLM -> toolCall(bash npm test ...)
Pi  -> toolResult(test 输出)

LLM -> final assistant response
```

每次工具执行后，`toolResult` 都会追加到 `currentContext.messages`，然后继续请求 LLM。

这个循环在：

```text
pi-mono/packages/agent/src/agent-loop.ts
runLoop()
```

关键结构：

```text
stream assistant response
-> 如果 assistant 有 toolCall
-> executeToolCalls()
-> 把 toolResults push 到 currentContext.messages
-> 下一次 assistant response 用更新后的 messages
```

所以一轮用户请求可能包含多次 LLM 请求。

## 11. 显式 /skill 分支：第 1 次请求就带全文

如果用户输入的是：

```text
/skill:add-llm-provider 帮我给 packages/ai 增加 acme provider
```

那流程不同。

`AgentSession._expandSkillCommand()` 会直接读取 skill 文件，去掉 frontmatter，把正文包进 user message：

```xml
<skill name="add-llm-provider" location="/Users/henry/work/developer/codex/pi-mono/.pi/skills/add-llm-provider.md">
References are relative to /Users/henry/work/developer/codex/pi-mono/.pi/skills.

# Adding a New LLM Provider (packages/ai)

A new provider touches multiple files. Work through these steps in order.
...
</skill>

帮我给 packages/ai 增加 acme provider
```

因此，第 1 次 LLM 请求的 messages 中就已经包含 skill 全文：

```ts
{
  role: "user",
  content: [
    {
      type: "text",
      text: "<skill name=\"add-llm-provider\" ...>...</skill>\n\n帮我给 packages/ai 增加 acme provider"
    }
  ]
}
```

这条路径不需要模型先根据索引主动 `read`。

## 12. 发送前的内部消息转换

Pi 内部有一些非标准 message role。发送给 LLM 前会通过：

```text
pi-mono/packages/coding-agent/src/core/messages.ts
convertToLlm()
```

转换规则：

```text
bashExecution      -> user message
custom             -> user message
branchSummary      -> user message，带 summary 包装
compactionSummary  -> user message，带 summary 包装
user               -> 原样
assistant          -> 原样
toolResult         -> 原样
```

所以 LLM 最终看到的是 provider 支持的标准消息结构。

## 13. 保存阶段：所有消息写入 session

一次完整任务会把这些内容写入 session：

```text
user message
assistant read skill toolCall
skill file toolResult
后续 read/edit/write/bash tool calls
后续 toolResults
final assistant message
```

持久化在：

```text
pi-mono/packages/coding-agent/src/core/agent-session.ts
_handleAgentEvent()
```

当 `message_end` 事件发生时：

```text
user / assistant / toolResult -> sessionManager.appendMessage()
custom                       -> appendCustomMessageEntry()
```

这意味着 skill 全文一旦通过 `read` 或 `/skill` 进入对话，就只是普通历史消息的一部分。

## 14. 三种容易混淆的“压缩”

### 14.1 read 工具单次输出截断

`read` 工具读取文本文件时，默认限制：

```text
2000 lines
50KB
```

任一限制命中就截断，并提示继续 offset。

这发生在 skill 全文第一次进入上下文之前。

如果 skill 很大，LLM 可能只看到前半部分：

```text
[Showing lines 1-2000 of 3500. Use offset=2001 to continue.]
```

如果需要完整 workflow，它必须继续调用 `read`。

### 14.2 TUI 显示折叠

当读到文件名是 `SKILL.md` 的文件时，TUI 会把 read 结果折叠显示成：

```text
[skill] name
```

这只是 UI 展示层 compact，不代表 LLM 上下文被压缩。

本文示例文件叫：

```text
add-llm-provider.md
```

它不一定触发这个 `SKILL.md` 专用显示分类。

### 14.3 session compaction

这是上下文层真正的压缩。

当上下文 token 超过阈值时，Pi 会触发 compaction：

```text
contextTokens > contextWindow - reserveTokens
```

默认：

```text
reserveTokens = 16384
keepRecentTokens = 20000
```

压缩时会：

```text
从尾部往前保留最近约 keepRecentTokens
旧消息进入 messagesToSummarize
调用 LLM 生成 compaction summary
保存一条 compaction entry
后续上下文用 summary + 最近原文替代完整旧历史
```

## 15. skill 全文如何从上下文消失

假设一开始模型读取过 skill：

```text
toolResult:
  # Adding a New LLM Provider (packages/ai)
  ...
```

随着会话继续：

```text
读代码
改代码
跑测试
修 bug
再跑测试
用户追加要求
继续改
...
```

当 session compaction 触发时，如果 skill toolResult 已经足够旧，它会进入 `messagesToSummarize`。

压缩后的上下文不再包含原始 skill 全文，而是类似：

```text
compactionSummary:
  用户要新增 acme provider。此前读取过 add-llm-provider skill；
  skill 要求更新 provider 实现、模型生成、测试矩阵、coding-agent 文档等。

最近保留的原始消息:
  ...

当前新用户消息:
  ...
```

之后 `buildSessionContext()` 组织 messages 时，会输出：

```text
compactionSummary
firstKeptEntryId 之后保留的最近原始消息
compaction 之后的新消息
```

旧的 skill toolResult 原文不再进入 LLM 请求。

如果再次 compaction，而 summary 没有继续保留 skill 细节，那么连“读过这个 skill”的语义也可能被进一步淡化。

## 16. 压缩后如何恢复 skill

只要 skill 文件还在磁盘上，并且没有被禁用或移除，system prompt 仍会包含索引：

```xml
<skill>
  <name>add-llm-provider</name>
  <description>Checklist for adding a new LLM provider...</description>
  <location>.../.pi/skills/add-llm-provider.md</location>
</skill>
```

所以压缩后如果 LLM 需要精确 checklist，正确动作是重新调用：

```json
{
  "name": "read",
  "arguments": {
    "path": ".../.pi/skills/add-llm-provider.md"
  }
}
```

也就是说：

```text
skill 索引长期稳定可见
skill 全文不是长期稳定可见
全文需要时可重新读取
```

## 17. 端到端时间线

下面是一条完整时间线：

```text
T0: 磁盘
    .pi/skills/add-llm-provider.md 存在

T1: 启动 / reload
    package-manager 收集 .pi/skills/add-llm-provider.md

T2: 加载
    skills.ts 解析 frontmatter
    得到 name/description/filePath/baseDir

T3: 构建 system prompt
    system prompt 包含 <available_skills>
    只含 skill 索引，不含正文

T4: 用户输入
    "帮我给 packages/ai 增加一个 acme LLM provider"

T5: 第 1 次 LLM 请求
    systemPrompt = 默认规则 + 项目指令 + skill 索引
    messages = 旧历史 + 当前 user message
    tools = read/bash/edit/write/...

T6: LLM 选择 skill
    assistant 返回 toolCall(read skill file)

T7: 工具执行
    read 返回 add-llm-provider.md 全文
    toolResult 追加到 messages

T8: 第 2 次 LLM 请求
    messages 现在包含 skill 全文 toolResult
    LLM 按 checklist 规划和执行

T9: 多轮工具循环
    read/edit/write/bash/toolResult 不断追加
    每次 toolResult 后可能再次请求 LLM

T10: 任务完成
    assistant 给最终总结
    所有消息保存到 session

T11: 会话变长
    context token 超阈值
    prepareCompaction 找切点

T12: compaction
    旧 skill toolResult 进入 messagesToSummarize
    LLM 生成 compaction summary

T13: 后续 LLM 请求
    messages = compactionSummary + 最近原文 + 新消息
    skill 全文原文消失

T14: 再次需要 skill
    LLM 仍可从 system prompt 的 skill 索引找到 location
    再次 read skill file
```

## 18. 关键结论

Pi 的 skill 不是“启动时把所有 skill 全文塞进上下文”。

它采用 progressive disclosure：

```text
每轮常驻：skill name + description + location
按需进入：skill 文件全文
执行方式：LLM 读说明后调用普通工具
历史治理：skill 全文作为普通消息被截断、保存、压缩、总结
恢复方式：根据 location 重新 read
```

因此，一个 skill 的完整生命周期不是“加载一次然后永远在上下文里”，而是：

```text
发现 -> 索引 -> 读取 -> 使用 -> 保存 -> 压缩 -> 消失 -> 重新读取
```
