# Skill 加载、压缩与剔除讨论总结

本文总结我们关于 nanobot skill 机制的讨论：skill 如何被发现、如何进入上下文、LLM 如何“执行”skill、skill 全文如何被压缩和剔除，以及这些机制对编写 skill 的具体影响。

## 什么是 skill 剔除

这里说的“剔除”不是删除磁盘上的 `SKILL.md` 文件。

它指的是：

```text
某个 skill 的完整 SKILL.md 内容不再出现在下一次发给 LLM 的 messages_for_model 里。
```

即使 skill 全文已经从当前模型上下文里消失，文件仍然可能存在于：

```text
<workspace>/skills/<skill-name>/SKILL.md
nanobot/skills/<skill-name>/SKILL.md
```

所以要区分两件事：

```text
磁盘上的 skill 文件是否存在
当前模型请求里是否还能看到 skill 全文
```

后者才是这次讨论里的“剔除”。

## Skill 是如何被发现的

nanobot 通过 `SkillsLoader` 发现 skill。加载顺序是：

```text
1. <workspace>/skills/<skill-name>/SKILL.md
2. nanobot/skills/<skill-name>/SKILL.md
```

如果 workspace skill 和内置 skill 同名，workspace 版本优先，内置版本会被跳过。

发现文件之后，nanobot 会继续做两个关键判断：

1. 是否在 `agents.defaults.disabledSkills` 里。
2. 是否满足 `metadata.nanobot.requires` 声明的依赖，例如命令行工具或环境变量。

`disabledSkills` 会把 skill 从主 agent 的 skill 摘要、always 注入和 subagent 的 skill 摘要里隐藏。它不会删除文件。

## 普通 skill 如何进入上下文

普通 skill 不会自动把完整 `SKILL.md` 注入 system prompt。

每轮 system prompt 里稳定出现的是 `# Skills` 摘要，大概像这样：

```text
- **weather** - Get current weather and forecasts ...
  `.../nanobot/skills/weather/SKILL.md`
```

LLM 需要自己判断：

```text
这个用户问题是否应该使用某个 skill
```

如果模型决定使用，它才会调用 `read_file` 读取对应的 `SKILL.md`。只有这一步发生后，skill 全文才会作为一个普通工具结果进入会话历史。

这和 `always` skill 不同。带有 `always: true` 的 skill 会在每轮被全文注入 `# Active Skills`。

## “执行 skill”到底是什么意思

skill 本身不是 Python runtime handler。它是一个 Markdown 说明文件。

普通 skill 的执行过程更准确地说是：

```text
1. LLM 在 # Skills 摘要里看到某个 skill。
2. LLM 调用 read_file 读取 SKILL.md。
3. LLM 阅读 Markdown 里的流程、命令、规则和输出格式。
4. LLM 按说明调用普通工具，例如 exec、read_file、web_fetch。
5. LLM 根据工具结果回答用户。
```

以 `weather` skill 为例：

```text
用户问北京天气
-> LLM 在 # Skills 里看到 weather
-> LLM 调用 read_file(.../weather/SKILL.md)
-> LLM 看到 wttr.in 的 curl 用法
-> LLM 调用 exec(curl ...)
-> LLM 总结天气结果并回答
```

所以“执行 skill”的核心不是运行 skill 代码，而是 LLM 读取 skill 说明，然后使用已有工具完成任务。

## 普通 skill 的完整生命周期

一个普通 skill 可能经历这些阶段：

```text
1. 磁盘阶段
   SKILL.md 存在于 workspace 或内置 skills 目录。

2. 发现阶段
   SkillsLoader 找到它，并检查 disabledSkills 和 requirements。

3. 摘要阶段
   system prompt 只在 # Skills 里列出 name、description、path。

4. 全文加载阶段
   LLM 调用 read_file(SKILL.md)。

5. 使用阶段
   LLM 按 skill 说明调用普通工具。

6. 跨轮回放阶段
   如果 read_file 工具结果还在最近、未 consolidation 的历史里，下一轮可能继续看到全文。

7. 压缩阶段
   _microcompact() 可能把旧的 read_file 结果替换成：
   [read_file result omitted from context]

8. 裁剪阶段
   _snip_history() 可能把旧的 tool call 和 tool result 从 messages_for_model 里整段裁掉。

9. consolidation 阶段
   maybe_consolidate_by_tokens() 可能推进 session.last_consolidated，让普通历史回放跳过旧消息。

10. idle auto-compact 阶段
    空闲自动压缩可能重写 session 文件，只保留最近合法后缀和归档摘要。
```

## 压缩过程是如何发生的

压缩主要发生在 `AgentRunner` 每次请求模型之前。

nanobot 通常不会直接修改原始 `messages`，而是生成一个将要发给模型的副本 `messages_for_model`，再对这个副本做上下文治理。

关键流程是：

```python
messages_for_model = self._drop_orphan_tool_results(messages)
messages_for_model = self._backfill_missing_tool_results(messages_for_model)
messages_for_model = self._microcompact(messages_for_model)
messages_for_model = self._apply_tool_result_budget(spec, messages_for_model)
messages_for_model = self._snip_history(spec, messages_for_model)
messages_for_model = self._drop_orphan_tool_results(messages_for_model)
messages_for_model = self._backfill_missing_tool_results(messages_for_model)
```

这些步骤里，和普通 skill 最相关的是：

```text
_microcompact()
_apply_tool_result_budget()
_snip_history()
Session.get_history()
maybe_consolidate_by_tokens()
idle auto-compact
```

## microcompact 会把旧工具结果压成一行

`_microcompact()` 只处理 compactable tools。

compactable tools 包括：

```text
read_file
exec
grep
find_files
web_search
web_fetch
list_dir
list_exec_sessions
```

规则是：

```text
如果 compactable tool result 数量 <= 10：
    不压缩。

如果 compactable tool result 数量 > 10：
    保留最近 10 个 compactable tool result 的全文。
    更旧的结果如果 content 是字符串且长度 >= 500 字符，就替换成一行占位。
```

普通 skill 全文是 `read_file(SKILL.md)` 的 tool result，所以它也会被这套规则处理。

压缩前：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_skill",
    "name": "read_file",
    "content": "weather/SKILL.md 的完整内容，很多很多行..."
}
```

压缩后：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_skill",
    "name": "read_file",
    "content": "[read_file result omitted from context]"
}
```

注意这里保留了：

```text
role
tool_call_id
name
```

只替换了 `content`。这样消息序列仍然是合法的 tool-call 结构，但昂贵的大段内容不再占用上下文。

## 超大 tool result 会落盘，只留引用和预览

另一个压缩路径是 `_apply_tool_result_budget()`。

如果某个工具结果超过 `agents.defaults.maxToolResultChars`，nanobot 可能把完整输出保存到：

```text
<workspace>/.nanobot/tool-results/<session>/<tool_call_id>.txt
```

然后上下文里只保留引用和预览：

```text
[tool output persisted]
Full output saved to: ...
Original size: ...
Preview:
...
```

这不是 skill 专用机制。任何超大的工具输出都可能走这条路径。一个特别大的 `SKILL.md` 被 `read_file` 读取后，也可能被这样处理。

## snip_history 会把旧历史整段裁掉

如果 `_microcompact()` 和 tool result budget 之后，prompt 仍然超过预算，`_snip_history()` 会裁剪历史。

它的策略大致是：

```text
保留所有 system messages。
从非 system messages 的尾部开始，保留最近的合法后缀。
尽量让保留的部分从 user message 开始。
修复 tool-call 边界，避免留下孤立 tool result。
```

这时旧的 skill 读取记录可能被完整移除：

```text
assistant tool call: read_file(weather/SKILL.md)
tool result: [read_file result omitted from context]
```

snip 之后，模型请求里可能连这个占位符都看不到。

这就是“完全剔除”的第一种情况：它只影响当前 `messages_for_model`，不一定改磁盘上的 session 历史。

## 跨轮回放也会裁剪

下一轮开始前，`Session.get_history()` 会决定哪些旧消息能进入历史回放。

它会做几件事：

```text
1. 只取 session.last_consolidated 之后的消息。
2. 按 max_messages 从尾部取窗口。
3. 如果传入 max_tokens，再从尾部按 token budget 裁剪。
4. 避免从孤立 tool result 开始回放。
```

所以即使上一轮 `messages_for_model` 还包含 skill 全文，下一轮也可能因为历史窗口或 replay token budget 太小而不再回放。

## consolidation 会改变后续回放边界

token-driven consolidation 会把旧消息总结进 memory，并推进：

```python
session.last_consolidated = end_idx
```

之后 `Session.get_history()` 从 `last_consolidated` 后面开始取历史，所以旧的 `read_file(SKILL.md)` 不再原样回放。

模型最多会在 system prompt 里看到类似这样的归档摘要：

```text
[Archived Context Summary]

用户之前查询过北京天气。助手使用 weather skill，并通过 curl 查询 wttr.in。
```

这保留了语义，但不保留原始结构化工具调用和 skill 全文。

## idle auto-compact 会重写 session 文件

idle auto-compact 比请求时压缩更强。

它会：

```text
1. 检查空闲 session。
2. 保留最近合法后缀，目前是 8 条消息。
3. 把旧前缀总结进 memory/history.jsonl。
4. 将 session.messages 替换为保留下来的后缀。
5. 将 session.last_consolidated 重置为 0。
6. 保存 session 文件。
```

如果旧的 `read_file(weather/SKILL.md)` 不在最近 8 条合法消息里，它会从 session 文件里消失。

这是最强形式的剔除。之后只能通过归档摘要知道曾经用过这个 skill，不能从 session 文件恢复原始 tool call 和 tool result。

## 不同阶段 LLM 看到的上下文

使用 skill 之前：

```python
[
    {
        "role": "system",
        "content": "# Skills\n- **weather** ... `.../weather/SKILL.md`"
    },
    {
        "role": "user",
        "content": "查一下北京天气"
    }
]
```

读取 skill 之后：

```python
[
    {"role": "system", "content": "# Skills\n- **weather** ..."},
    {"role": "user", "content": "查一下北京天气"},
    {"role": "assistant", "tool_calls": ["read_file(weather/SKILL.md)"]},
    {"role": "tool", "name": "read_file", "content": "完整 SKILL.md 内容"}
]
```

microcompact 之后：

```python
[
    {"role": "system", "content": "# Skills\n- **weather** ..."},
    {"role": "user", "content": "查一下北京天气"},
    {"role": "assistant", "tool_calls": ["read_file(weather/SKILL.md)"]},
    {
        "role": "tool",
        "name": "read_file",
        "content": "[read_file result omitted from context]"
    }
]
```

snip 或 consolidation 之后：

```python
[
    {"role": "system", "content": "# Skills\n- **weather** ..."},
    {"role": "user", "content": "当前问题"}
]
```

consolidation 并带归档摘要时：

```python
[
    {
        "role": "system",
        "content": """
# Skills
- **weather** ...

[Archived Context Summary]

用户之前查询过北京天气。助手使用 weather skill，并通过 curl 查询 wttr.in。
"""
    },
    {"role": "user", "content": "当前问题"}
]
```

## 这些机制对编写 skill 的影响

普通 skill 的全文可能很快被压缩或剔除，所以 skill 不应该写成需要长期驻留上下文的大文档。

更好的设计目标是：

```text
短入口
强触发
快速流程
可重读
分层 references/scripts
```

写普通 skill 时，应优先做到：

1. `description` 写清楚触发条件。
2. `SKILL.md` 开头 30 行能说明用途、快速流程和关键规则。
3. 命令模板短、稳定、可复制。
4. 输出格式明确，不依赖模型长期记忆。
5. 大量细节放进 `references/`。
6. 重复、确定性的工作放进 `scripts/`。
7. 不要假设模型跨轮还记得 skill 全文。
8. 谨慎使用 `always: true`。

核心假设应该是：

```text
模型长期稳定看到的是 skill 摘要。
完整 SKILL.md 可能只在 read_file 后短暂可见。
```

## “何时重读”应该写在哪里

我们讨论中纠正了一个关键点：

```text
如果“何时重读”的规则只写在 SKILL.md 正文里，那么正文被压缩后，模型就看不到这句话。
```

例如正文里写：

```text
If previous tool results for this skill are omitted, read this file again.
```

一旦 tool result 变成：

```text
[read_file result omitted from context]
```

模型就读不到这条正文规则了。

所以正文里的重读规则只能帮助“刚读完 skill 的当前阶段”，不能作为压缩后的长期恢复入口。

更可靠的位置是：

1. 全局 `# Skills` 模板。
2. 对复杂或高风险 skill，写进 `description` 的短提示。
3. `SKILL.md` 开头重复一遍，作为当前读取后的局部提醒。

最好的框架级做法是在：

```text
nanobot/templates/agent/skills_section.md
```

加入类似规则：

```text
If a previous skill file result is omitted, summarized, or no longer visible,
read the SKILL.md file again before relying on that skill's workflow.
```

这样所有普通 skill 都受益，也不需要每个 skill 的 `description` 都重复这句话。

## description 里是否一定要写重读提示

不一定。

普通 skill 不是“用户问题匹配后系统自动加载全文”。真实机制是：

```text
system prompt 每轮展示 skill 摘要
-> LLM 自己判断是否相关
-> LLM 自己决定是否 read_file(SKILL.md)
```

所以 `description` 的首要任务是帮助 LLM 判断：

```text
这个问题应该使用哪个 skill
```

对于简单 skill，只写清楚触发条件通常就够了：

```yaml
description: Get current weather and forecasts without an API key. Use when the
  user asks about weather, forecasts, temperature, humidity, wind, or city
  weather.
```

对于复杂、高风险、跨轮继续概率高的 skill，可以加一句很短的重读提示：

```yaml
description: Deploy production releases. Use for release planning, rollout,
  rollback, and post-release checks. Read before use; reread if previous
  instructions are omitted or not visible.
```

但不要给每个 skill 都写很长的重读说明。`description` 每轮都会进入 `# Skills` 摘要，过长会增加基础 prompt 成本。

## 编写 skill 的实用 checklist

写或评审 nanobot skill 时，可以按这个清单检查：

1. `description` 是否清楚说明了触发场景。
2. `description` 是否过长，是否污染每轮 skill 摘要。
3. `SKILL.md` 前 30 行是否能恢复核心流程。
4. 是否把最常用路径放在前面。
5. 是否有稳定、可复制的命令模板。
6. 是否明确写了输出格式。
7. 大量背景资料是否拆进 `references/`。
8. 是否告诉模型什么时候读取哪个 reference。
9. 可确定执行的步骤是否放进 `scripts/`。
10. 是否避免依赖跨轮记忆。
11. 是否真的需要 `always: true`。
12. 是否更适合在全局 `skills_section.md` 加重读规则，而不是每个 skill 重复写。

一句话总结：

```text
普通 skill 应该写成可重读、可分层加载的工作流入口，
不要写成必须长期留在上下文里的大型说明书。
```

