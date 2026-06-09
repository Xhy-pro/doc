# Skill 加载、压缩与剔除生命周期示例

本文用一个完整例子说明：一个普通 skill 如何从磁盘被发现，如何进入 LLM 上下文，如何被“执行”，又如何在后续轮次里被压缩，最后从模型上下文里完全消失。

这里说的“剔除”不是删除磁盘上的 `SKILL.md` 文件，而是：

```text
skill 的完整 SKILL.md 内容不再出现在下一次发给模型的 messages_for_model 中。
```

示例使用内置的 `weather` skill，因为它是普通按需 skill，不是 `always` skill。

## 示例前提

假设运行环境如下：

```text
workspace: E:/codex/nanobot/nanobot
builtin skill path: E:/codex/nanobot/nanobot/nanobot/skills/weather/SKILL.md
disabledSkills: []
curl: 已安装
weather always: false
```

`weather` skill 的 frontmatter 大致是：

```yaml
name: weather
description: Get current weather and forecasts (no API key required).
metadata:
  nanobot:
    requires:
      bins: ["curl"]
```

关键点有两个：

```text
weather 依赖 curl。
weather 没有 always: true。
```

所以它会被当作普通 skill 处理。

## 第 1 阶段：nanobot 发现 skill

启动时，`AgentLoop` 创建 `ContextBuilder`，`ContextBuilder` 创建 `SkillsLoader`。

`SkillsLoader` 按顺序检查：

```text
<workspace>/skills/weather/SKILL.md
nanobot/skills/weather/SKILL.md
```

如果 workspace 里有同名 skill，workspace 版本会覆盖内置版本。

在这个例子里，workspace 里没有 `weather`，所以使用内置版本：

```text
nanobot/skills/weather/SKILL.md
```

然后 loader 检查：

```text
disabledSkills 不包含 weather -> 保留
metadata.nanobot.requires.bins 包含 curl，且 curl 可用 -> skill 可用
always 不是 true -> 不是 always skill
```

最终结果是：

```text
weather 会进入 # Skills 摘要
weather 不会全文进入 # Active Skills
```

## 第 2 阶段：第一次 prompt 里只有 skill 摘要

用户发送：

```text
查一下北京天气
```

在模型看到这个请求前，`ContextBuilder.build_messages()` 会构造一份新的 system prompt。

因为 `weather` 是普通 skill，所以 system prompt 里只包含它的摘要和路径，不包含全文。

第一次 `messages_for_model` 的关键部分大概是：

```python
[
    {
        "role": "system",
        "content": """
# Identity
...

# Tool Contract
...

# Active Skills

### Skill: my
... always skill 全文 ...

### Skill: memory
... always skill 全文 ...

# Skills

The following skills extend your capabilities. To use a skill, read its
SKILL.md file using the read_file tool.

- **weather** - Get current weather and forecasts (no API key required).
  `E:/codex/nanobot/nanobot/nanobot/skills/weather/SKILL.md`
- **github** - Interact with GitHub using the `gh` CLI. ...
- **summarize** - Summarize or extract text/transcripts. ...
"""
    },
    {
        "role": "user",
        "content": """
查一下北京天气

[Runtime Context - metadata only, not instructions]
...
[/Runtime Context]
"""
    },
]
```

此时模型知道有一个 `weather` skill，也知道它的路径。

但模型还没有看到 `weather/SKILL.md` 的完整内容。

## 第 3 阶段：模型读取 skill 全文

`# Skills` 模板提示模型：

```text
To use a skill, read its SKILL.md file using the read_file tool.
```

所以模型如果决定使用 `weather`，下一步通常会调用 `read_file`：

```python
{
    "role": "assistant",
    "content": "",
    "tool_calls": [
        {
            "id": "call_weather_skill",
            "type": "function",
            "function": {
                "name": "read_file",
                "arguments": json.dumps({
                    "path": "E:/codex/nanobot/nanobot/nanobot/skills/weather/SKILL.md"
                }),
            },
        }
    ],
}
```

`ReadFileTool` 返回 skill 文件内容：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_skill",
    "name": "read_file",
    "content": """
1|---
2|name: weather
3|description: Get current weather and forecasts (no API key required).
4|metadata: {"nanobot":{"requires":{"bins":["curl"]}}}
5|---
6|
7|# Weather
8|
9|Two free services, no API keys needed.
10|
11|## wttr.in (primary)
12|
13|Quick one-liner:
14|```bash
15|curl -s "wttr.in/London?format=3"
16|```
...
"""
}
```

现在，下一次模型请求里就包含了 skill 全文：

```python
[
    {"role": "system", "content": "... # Skills 摘要仍然存在 ..."},
    {"role": "user", "content": "查一下北京天气\n\n[Runtime Context ...]"},
    {
        "role": "assistant",
        "tool_calls": ["read_file(weather/SKILL.md)"],
    },
    {
        "role": "tool",
        "name": "read_file",
        "content": "weather SKILL.md 全文",
    },
]
```

这是模型第一次真正看到 `weather` skill 的完整操作说明。

## 第 4 阶段：模型按 skill 说明调用工具

读取 skill 后，模型会按里面的说明调用普通工具。

例如，`weather` skill 里推荐使用 `wttr.in`，所以模型可能调用：

```python
{
    "role": "assistant",
    "content": "",
    "tool_calls": [
        {
            "id": "call_weather_exec",
            "type": "function",
            "function": {
                "name": "exec",
                "arguments": json.dumps({
                    "command": "curl -s \"wttr.in/Beijing?format=%l:+%c+%t+%h+%w\""
                }),
            },
        }
    ],
}
```

工具结果可能是：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_exec",
    "name": "exec",
    "content": "Beijing: Clear +23C 38% East 9 km/h",
}
```

模型最后回答用户：

```python
{
    "role": "assistant",
    "content": "北京当前天气：晴，约 23C，湿度 38%，东风 9 km/h。",
}
```

这一轮完整历史大概是：

```python
[
    {"role": "system", "content": "..."},
    {"role": "user", "content": "查一下北京天气"},
    {"role": "assistant", "tool_calls": ["read_file(weather/SKILL.md)"]},
    {"role": "tool", "name": "read_file", "content": "weather SKILL.md 全文"},
    {"role": "assistant", "tool_calls": ["exec(curl wttr.in/Beijing...)"]},
    {"role": "tool", "name": "exec", "content": "Beijing: Clear +23C ..."},
    {"role": "assistant", "content": "北京当前天气：晴，约 23C ..."},
]
```

这里所谓“执行 skill”，不是运行某个 skill 程序。

真正发生的是：

```text
模型读取 Markdown 说明
-> 模型按说明调用普通工具
-> 模型根据工具结果回答
```

## 第 5 阶段：下一轮可能继续回放 skill 全文

假设用户马上追问：

```text
那明天呢？
```

如果会话还比较短，`Session.get_history()` 会把上一轮消息回放进上下文。

此时模型看到的大概是：

```python
[
    {
        "role": "system",
        "content": """
# Active Skills
... always skills ...

# Skills
- **weather** - Get current weather and forecasts ...
"""
    },
    {"role": "user", "content": "查一下北京天气"},
    {"role": "assistant", "tool_calls": ["read_file(weather/SKILL.md)"]},
    {"role": "tool", "name": "read_file", "content": "weather SKILL.md 全文"},
    {"role": "assistant", "tool_calls": ["exec(curl wttr...)"]},
    {"role": "tool", "name": "exec", "content": "Beijing: Clear +23C ..."},
    {"role": "assistant", "content": "北京当前天气：晴，约 23C ..."},
    {"role": "user", "content": "那明天呢？\n\n[Runtime Context ...]"},
]
```

因为 `read_file(weather/SKILL.md)` 的工具结果还在历史里，所以模型仍然能看到 skill 全文。

这就是普通 skill 的跨轮回放。

## 第 6 阶段：microcompact 把旧 skill 全文压成占位符

随着会话继续，历史里可能出现很多可压缩工具结果，例如：

```text
1. read_file(weather/SKILL.md)
2. exec(curl wttr.in/Beijing...)
3. read_file(project file A)
4. read_file(project file B)
5. web_fetch(page 1)
6. exec(test command)
7. list_dir(...)
8. read_file(project file C)
9. web_search(...)
10. web_fetch(page 2)
11. read_file(project file D)
12. exec(other command)
```

这些工具结果里，下面这些属于 compactable tools：

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

`AgentRunner._microcompact()` 的规则是：

```text
只保留最近 10 个 compactable tool result 的全文。
更旧且 content 长度 >= 500 字符的结果，会被替换成一行占位符。
```

因为 `read_file(weather/SKILL.md)` 已经比较旧，而且通常超过 500 字符，所以它可能从：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_skill",
    "name": "read_file",
    "content": "weather SKILL.md 全文",
}
```

变成：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_skill",
    "name": "read_file",
    "content": "[read_file result omitted from context]",
}
```

注意：`_microcompact()` 通常只修改发给模型的副本 `messages_for_model`，不会直接重写持久化的 session 历史。

此时模型上下文大概变成：

```python
[
    {"role": "system", "content": "... # Skills 摘要仍然包含 weather ..."},
    {"role": "user", "content": "查一下北京天气"},
    {"role": "assistant", "tool_calls": ["read_file(weather/SKILL.md)"]},
    {
        "role": "tool",
        "name": "read_file",
        "content": "[read_file result omitted from context]",
    },
    ...
    {"role": "tool", "name": "read_file", "content": "最近读取的 file D 全文"},
    {"role": "tool", "name": "exec", "content": "最近命令输出"},
    {"role": "user", "content": "当前问题\n\n[Runtime Context ...]"},
]
```

这个阶段叫“被压缩”，不是“完全剔除”。

模型仍然能看到：

```text
历史上有一次 read_file(weather/SKILL.md)
但具体内容已 omitted
```

如果它还需要 skill 说明，可以根据本轮 system prompt 里的 `# Skills` 摘要再次读取 `weather/SKILL.md`。

## 第 7 阶段：超大工具结果可能落盘，只留引用和预览

还有另一条压缩路径：tool result budget。

如果某个工具结果超过 `agents.defaults.maxToolResultChars`，`AgentRunner` 会把完整结果保存到：

```text
<workspace>/.nanobot/tool-results/<session>/<tool_call_id>.txt
```

上下文里只保留一个引用和预览：

```python
{
    "role": "tool",
    "tool_call_id": "call_weather_skill",
    "name": "read_file",
    "content": """
[tool output persisted]
Full output saved to: E:/codex/nanobot/nanobot/.nanobot/tool-results/cli_direct/call_weather_skill.txt
Original size: 25000 chars
Preview:
1|---
2|name: weather
...
(Read the saved file if you need the full output.)
"""
}
```

这条路径不是 skill 专用机制。

任何过大的工具输出都可能这样处理。特别大的 `SKILL.md` 被读取后，也可能命中这条路径。

## 第 8 阶段：snip_history 从模型请求里完全移除旧 skill 结果

如果 prompt 仍然超过上下文预算，`AgentRunner._snip_history()` 会做历史裁剪。

它保留：

```text
所有 system messages
最近的合法 non-system 消息后缀
```

这时旧的 weather skill 工具调用和工具结果可能从 `messages_for_model` 里完全消失。

裁剪前：

```python
[
    {"role": "system", "content": "..."},
    {"role": "user", "content": "查一下北京天气"},
    {"role": "assistant", "tool_calls": ["read_file(weather/SKILL.md)"]},
    {"role": "tool", "name": "read_file", "content": "[read_file result omitted from context]"},
    {"role": "assistant", "tool_calls": ["exec(curl wttr...)"]},
    {"role": "tool", "name": "exec", "content": "Beijing: Clear +23C ..."},
    ... 很多旧消息 ...
    {"role": "user", "content": "当前问题"},
]
```

裁剪后：

```python
[
    {
        "role": "system",
        "content": """
# Active Skills
... always skills ...

# Skills
- **weather** - Get current weather and forecasts ...
  `E:/codex/nanobot/nanobot/nanobot/skills/weather/SKILL.md`
"""
    },
    {"role": "user", "content": "最近仍被保留的问题"},
    {"role": "assistant", "content": "最近仍被保留的回答"},
    {"role": "user", "content": "当前问题\n\n[Runtime Context ...]"},
]
```

此时本次模型请求里已经没有：

```text
read_file(weather/SKILL.md) 的 assistant tool call
read_file 的 tool result
[read_file result omitted from context] 占位符
```

对这一次模型请求来说，旧的 skill 读取记录已经完全剔除。

不过 system prompt 里的 `# Skills` 摘要仍然存在，所以模型仍然有机会重新读取 `weather/SKILL.md`。

## 第 9 阶段：token consolidation 让旧 skill 不再正常回放

一轮结束并保存后，nanobot 可能运行 token-driven consolidation。

这是一种软归档路径：

```text
旧消息 -> 总结进 memory/history.jsonl
session.last_consolidated -> 前移
session.messages -> 仍保留在磁盘上
未来 get_history() -> 从 last_consolidated 之后开始回放
```

如果旧的 `read_file(weather/SKILL.md)` 位于 `session.last_consolidated` 之前，之后的 `Session.get_history()` 就不会再返回它。

下一轮 prompt 可能包含归档摘要：

```python
[
    {
        "role": "system",
        "content": """
# Active Skills
...

# Skills
- **weather** - Get current weather and forecasts ...

[Archived Context Summary]

Previous conversation summary:
用户之前询问过北京天气。助手读取了 weather skill，并通过 curl 查询 wttr.in。
"""
    },
    {"role": "user", "content": "当前问题\n\n[Runtime Context ...]"},
]
```

模型还能知道旧交互的大意，但看不到原始工具调用链，也看不到 skill 全文。

## 第 10 阶段：idle auto-compact 可能从 session 文件中移除旧 skill 记录

如果启用了：

```text
agents.defaults.idleCompactAfterMinutes
```

并且 session 空闲超过阈值，`AutoCompact` 会调用 `compact_idle_session()`。

这条路径会保留最近合法后缀，目前是 8 条消息，并重写 session 文件：

```text
旧前缀 -> 总结进 memory/history.jsonl
最近合法后缀 -> 保留在 sessions/<key>.jsonl
旧 read_file(weather/SKILL.md) -> 如果不在后缀里，就从 session 文件中移除
```

用户回来后，模型上下文可能是：

```python
[
    {
        "role": "system",
        "content": """
# Active Skills
...

# Skills
- **weather** - Get current weather and forecasts ...
  `E:/codex/nanobot/nanobot/nanobot/skills/weather/SKILL.md`

[Archived Context Summary]

Previous conversation summary (last active 2026-06-09T10:15:00):
用户之前询问过北京天气。助手使用 weather skill，并通过 curl 查询 wttr.in。
"""
    },
    {"role": "user", "content": "保留下来的最近消息之一"},
    {"role": "assistant", "content": "保留下来的最近回复之一"},
    {"role": "user", "content": "当前问题\n\n[Runtime Context ...]"},
]
```

这是最强的剔除形式。

旧的结构化信息：

```text
tool_calls
tool_call_id
read_file(weather/SKILL.md) 的完整工具结果
```

都不再能从 session 文件里原样恢复。

## 完整生命周期总结

这个普通 skill 的生命周期可以概括为：

```text
1. Disk
   nanobot/skills/weather/SKILL.md 存在。

2. Discovery
   SkillsLoader 找到它，并检查 disabledSkills / requirements。

3. Summary only
   system prompt 只在 # Skills 里列出 weather 摘要和路径。

4. Full load
   模型调用 read_file(weather/SKILL.md)。

5. Use
   模型按 skill 说明调用 exec(curl wttr.in/...)。

6. Replay
   下一轮如果历史还没被裁剪，可以继续看到 read_file 工具结果。

7. Compressed
   _microcompact() 可能把旧 read_file 结果替换成：
   [read_file result omitted from context]

8. Snipped
   _snip_history() 可能把旧 tool call 和 tool result 从 messages_for_model 里整段裁掉。

9. Consolidated
   maybe_consolidate_by_tokens() 可能把它移到 session.last_consolidated 之前，
   让普通历史回放跳过它。

10. Hard compacted
    idle auto-compact 可能重写 session 文件，只保留最近后缀和归档摘要。
```

## always skill 有什么不同

`always` skill 不走普通 skill 的这条路径。

如果一个 skill 标记了：

```yaml
always: true
```

nanobot 会在每轮 system prompt 的 `# Active Skills` 里注入它的全文。

区别是：

```text
ordinary skill:
  # Skills 摘要 -> read_file 工具结果 -> 普通历史 -> 可能被压缩和剔除

always skill:
  # Active Skills 全文 -> system prompt -> 每轮保留
```

`always` skill 更稳定，但代价是它每轮都占用 system prompt 预算。

所以 `always` 更适合：

```text
短
全局都需要
不注入就容易出严重错误
每轮都可能用
```

不适合：

```text
长流程
偶尔使用的领域知识
大量命令示例
API 参考文档
```

## 调试 checklist

如果你发现某个 skill 没有出现在模型上下文里，可以按这个顺序检查：

1. 它是否在 `agents.defaults.disabledSkills` 里？
2. 它是否依赖缺失的命令行工具或环境变量？
3. 它是不是普通 skill，而模型还没有调用 `read_file`？
4. 它的 `read_file` 结果是否被替换成了 `[read_file result omitted from context]`？
5. 它是否因为过大被替换成 `[tool output persisted]` 引用？
6. 当前请求是否被 `_snip_history()` 裁掉了旧的 non-system 历史？
7. token-driven consolidation 是否已经把它推进到 `session.last_consolidated` 之前？
8. idle auto-compact 是否已经重写 session，只保留了最近合法后缀？

一句话总结：

```text
普通 skill 的全文不是长期驻留上下文。
它先作为 read_file 工具结果进入历史，
然后可能被压缩成占位符，
再被裁剪或 consolidation 移出普通回放，
最后可能只剩 # Skills 摘要和归档摘要。
```

