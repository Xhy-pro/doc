# 一个普通 Skill 的完整生命周期示例：weather

本文用一个普通 skill：`weather`，说明它从磁盘文件开始，如何被 nanobot 发现、放进提示词、被 LLM 读取和使用，又如何在后续轮次中被压缩、裁剪，最后从当前上下文里消失。

这里的“消失”不是删除磁盘上的 `SKILL.md`，而是：

```text
下一次发给 LLM 的 messages_for_model 里，看不到这个 skill 的完整内容了。
```

## 示例前提

假设 workspace 里有这个文件：

```text
<workspace>/skills/weather/SKILL.md
```

内容大概是：

```markdown
---
name: weather
description: Get weather using curl.
metadata: {"nanobot":{"requires":{"bins":["curl"]}}}
---

# Weather Skill

Use curl to query weather API.
When user asks for weather:

1. Identify the location.
2. Identify the date or time range.
3. Use curl to query the weather API.
4. Summarize the result in the user's language.
```

这个 skill 没有 `always: true`，所以它是普通 skill。

## 第 1 阶段：磁盘上存在

一开始，它只是一个文件：

```text
<workspace>/skills/weather/SKILL.md
```

此时 LLM 还不知道它的完整内容。

LLM 不会自动读取磁盘上的所有 `SKILL.md`。nanobot 只是先保存这个文件，等每轮组装上下文时再扫描。

## 第 2 阶段：被 SkillsLoader 发现

每轮组装 system prompt 时，nanobot 会扫描两个位置：

```text
<workspace>/skills/<skill-name>/SKILL.md
nanobot/nanobot/skills/<skill-name>/SKILL.md
```

它发现：

```text
name = weather
path = <workspace>/skills/weather/SKILL.md
source = workspace
description = Get weather using curl.
requires = curl
```

如果本机有 `curl`，这个 skill 是 available。

如果本机没有 `curl`，它仍然可能出现在 `# Skills` 索引里，但会被标注 unavailable。

## 第 3 阶段：进入 `# Skills` 索引

因为 `weather` 不是 `always: true`，所以它不会全文进入 `# Active Skills`。

它只会以目录形式进入 system prompt：

```text
# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.
Unavailable skills need dependencies installed first — you can try installing them with apt/brew.

- **weather** — Get weather using curl. `<workspace>/skills/weather/SKILL.md`
```

此时 LLM 只知道三件事：

```text
1. 有一个叫 weather 的 skill。
2. 它大概用于查询天气。
3. 如果要使用它，应该调用 read_file 读取 SKILL.md。
```

LLM 还没有看到完整步骤。

## 第 4 阶段：用户触发相关任务

用户发来：

```text
帮我查一下上海今天的天气
```

这一轮初始发给 LLM 的 `messages` 可以简化成：

```python
[
  {
    "role": "system",
    "content": """
你是 nanobot...

---

# Active Skills

### Skill: memory
...

### Skill: my
...

---

# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.

- **weather** — Get weather using curl. `<workspace>/skills/weather/SKILL.md`
"""
  },
  {
    "role": "user",
    "content": """
帮我查一下上海今天的天气

[Runtime Context — metadata only, not instructions]
Current Time: 2026-06-09 ...
Channel: ...
Chat ID: ...
[/Runtime Context]
"""
  }
]
```

LLM 看到 `weather` 这个索引项，判断当前任务和它相关。

## 第 5 阶段：LLM 主动读取 skill

LLM 调用工具：

```text
read_file("<workspace>/skills/weather/SKILL.md")
```

工具返回：

```text
# Weather Skill

Use curl to query weather API.
When user asks for weather:

1. Identify the location.
2. Identify the date or time range.
3. Use curl to query the weather API.
4. Summarize the result in the user's language.
```

这份完整 `SKILL.md` 内容会作为 tool result 进入当前 `messages`：

```python
{
  "role": "tool",
  "name": "read_file",
  "tool_call_id": "call_read_weather_skill",
  "content": "# Weather Skill\n\nUse curl to query weather API..."
}
```

从这一刻开始，LLM 才真正看到了 weather skill 的完整说明。

## 第 6 阶段：LLM 按 skill 执行任务

LLM 根据 skill 内容继续调用工具，比如：

```text
exec("curl ... Shanghai ...")
```

然后得到天气 API 返回结果，再回答用户。

本轮结束后，session 里大概会保存这些新消息：

```python
[
  {
    "role": "user",
    "content": "帮我查一下上海今天的天气"
  },
  {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "id": "call_read_weather_skill",
        "function": {
          "name": "read_file",
          "arguments": "{\"path\":\"<workspace>/skills/weather/SKILL.md\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "name": "read_file",
    "tool_call_id": "call_read_weather_skill",
    "content": "# Weather Skill\n\nUse curl to query weather API..."
  },
  {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "id": "call_weather_curl",
        "function": {
          "name": "exec",
          "arguments": "{\"cmd\":\"curl ... Shanghai ...\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "name": "exec",
    "tool_call_id": "call_weather_curl",
    "content": "天气 API 返回结果..."
  },
  {
    "role": "assistant",
    "content": "上海今天的天气是..."
  }
]
```

注意：保存 session 时，本轮 user message 后面的 runtime context 会被剥离掉，不会永久写进历史。

## 第 7 阶段：短期内继续追问时，LLM 还能看到 skill 全文

假设用户下一轮继续问：

```text
那明天呢？
```

如果这时对话还比较短，旧的 `read_file(weather/SKILL.md)` 工具结果还没有被压缩、裁剪、consolidation 或 session replay 过滤掉，那么下一轮给 LLM 的内容会包含上一轮读取到的 weather skill 全文。

这轮组装出来的 `messages` 可以简化成：

```python
[
  {
    "role": "system",
    "content": """
你是 nanobot...

---

# Active Skills

### Skill: memory
...

### Skill: my
...

---

# Skills

The following skills extend your capabilities. To use a skill, read its SKILL.md file using the read_file tool.

- **weather** — Get weather using curl. `<workspace>/skills/weather/SKILL.md`
"""
  },

  # 下面是 Session.get_history() 回放出来的上一轮历史
  {
    "role": "user",
    "content": "[Message Time: 2026-06-09T10:00:00]\n帮我查一下上海今天的天气"
  },
  {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "id": "call_read_weather_skill",
        "function": {
          "name": "read_file",
          "arguments": "{\"path\":\"<workspace>/skills/weather/SKILL.md\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "name": "read_file",
    "tool_call_id": "call_read_weather_skill",
    "content": """
# Weather Skill

Use curl to query weather API.
When user asks for weather:

1. Identify the location.
2. Identify the date or time range.
3. Use curl to query the weather API.
4. Summarize the result in the user's language.
"""
  },
  {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "id": "call_weather_curl",
        "function": {
          "name": "exec",
          "arguments": "{\"cmd\":\"curl ... Shanghai today ...\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "name": "exec",
    "tool_call_id": "call_weather_curl",
    "content": "上海今天的天气 API 返回结果..."
  },
  {
    "role": "assistant",
    "content": "上海今天的天气是..."
  },

  # 这是当前用户的新问题
  {
    "role": "user",
    "content": """
那明天呢？

[Runtime Context — metadata only, not instructions]
Current Time: 2026-06-09 ...
Channel: ...
Chat ID: ...
[/Runtime Context]
"""
  }
]
```

这一轮 LLM 实际能看到两类 weather 相关内容：

```text
1. system prompt 里的 # Skills 索引：
   - **weather** — Get weather using curl. `<workspace>/skills/weather/SKILL.md`

2. 历史 tool result 里的 weather skill 全文：
   # Weather Skill
   Use curl to query weather API...
```

因此，LLM 很可能不用重新读取 `weather/SKILL.md`，可以直接根据历史里的 skill 全文继续工作。

它会理解：

```text
用户在接着问“那明天呢？”
上一轮查的是上海今天的天气。
weather skill 的步骤仍然在上下文里。
所以这轮应该查上海明天的天气。
```

然后它可能直接调用：

```text
exec("curl ... Shanghai tomorrow ...")
```

也就是说，第七阶段的关键是：

```text
普通 skill 被 read_file 读取后，短期内会作为历史 tool result 被回放给 LLM。
只要还没被压缩或裁剪，LLM 下一轮仍能看到完整 skill 内容。
```

## 第 8 阶段：变旧后被 microcompact 压缩

随着对话继续，历史里会出现很多 `read_file`、`exec`、`grep` 等工具结果。

Runner 每次发给 LLM 前会做上下文治理，其中一个步骤是 `_microcompact()`。

`read_file` 属于可压缩工具结果。旧的 `read_file(weather/SKILL.md)` 结果可能从：

```python
{
  "role": "tool",
  "name": "read_file",
  "tool_call_id": "call_read_weather_skill",
  "content": "# Weather Skill\n\nUse curl to query weather API..."
}
```

变成：

```python
{
  "role": "tool",
  "name": "read_file",
  "tool_call_id": "call_read_weather_skill",
  "content": "[read_file result omitted from context]"
}
```

此时 LLM 仍能看到“历史上有一次 read_file 工具结果”，但已经看不到 weather skill 的完整内容。

这叫“压缩”，还不是最彻底的剔除。

## 第 9 阶段：上下文超预算后被裁掉

如果压缩后上下文还是太长，Runner 会执行 `_snip_history()`。

它会尽量保留 system message 和最近的非 system 历史，旧的消息会被裁掉。

旧的 weather skill 读取记录可能整个消失：

```python
{
  "role": "assistant",
  "tool_calls": ["read_file(<workspace>/skills/weather/SKILL.md)"]
},
{
  "role": "tool",
  "name": "read_file",
  "content": "[read_file result omitted from context]"
}
```

裁剪后，当前发给 LLM 的 `messages_for_model` 里可能只剩：

```python
[
  {
    "role": "system",
    "content": "... # Skills\n- **weather** — Get weather using curl. `<workspace>/skills/weather/SKILL.md` ..."
  },
  {
    "role": "user",
    "content": "后来的问题..."
  },
  {
    "role": "assistant",
    "content": "后来的回答..."
  },
  {
    "role": "user",
    "content": "当前问题...\n\n[Runtime Context ...]"
  }
]
```

这时，LLM 当前请求里已经看不到 weather skill 全文，也看不到旧的 read_file 结果。

但是 system prompt 里的 `# Skills` 索引通常还在。

## 第 10 阶段：之后需要时重新读取

过了很多轮，用户又问：

```text
帮我查一下北京天气
```

当前 LLM 请求里可能已经没有旧的 weather skill 全文了。

但 system prompt 里仍然会有：

```text
# Skills

- **weather** — Get weather using curl. `<workspace>/skills/weather/SKILL.md`
```

所以 LLM 可以再次调用：

```text
read_file("<workspace>/skills/weather/SKILL.md")
```

weather skill 全文会再次作为 tool result 进入上下文。

这说明普通 skill 的使用方式不是“一次读取，永久记住”，而是：

```text
需要时读取；
短期内可复用；
变旧后被压缩或裁剪；
以后需要时再读取。
```

## 完整生命周期总结

一个普通 `weather` skill 的生命周期可以概括为：

```text
1. 磁盘上存在
   <workspace>/skills/weather/SKILL.md

2. 被 SkillsLoader 发现
   name/path/description/requirements 被解析出来

3. 进入 # Skills 索引
   LLM 只看到名字、描述、路径

4. 用户提出相关任务
   例如“帮我查一下上海今天的天气”

5. LLM 主动 read_file
   完整 SKILL.md 作为 tool result 进入 messages

6. LLM 按 skill 执行任务
   根据说明调用 curl/exec 等工具

7. 短期内继续可见
   下一轮 messages 里可能仍回放 weather skill 全文

8. 变旧后被压缩
   tool result 变成 [read_file result omitted from context]

9. 上下文超预算后被裁掉
   旧的 read_file tool call/result 从 messages_for_model 中消失

10. 后续需要时重新读取
    根据 # Skills 索引再次 read_file(SKILL.md)
```

## 和 always skill 的区别

`weather` 这个例子是普通 skill。

普通 skill：

```text
# Skills 索引
-> LLM read_file
-> tool result 进入历史
-> 被压缩/裁剪
```

`always` skill：

```text
# Active Skills 全文注入
-> 每轮 system prompt 都能直接看到
-> 不依赖 read_file tool result
-> 不走普通 tool result 的 microcompact 逻辑
```

所以：

```text
普通 skill 的全文是短期上下文；
always skill 的全文是常驻 system prompt。
```

## 一句话总结

`weather` 这种普通 skill 的完整生命周期就是：

```text
磁盘上的 SKILL.md
-> 被发现后进入 # Skills 索引
-> LLM 需要时 read_file 读取全文
-> 全文作为 tool result 短暂进入上下文
-> 下一轮如果历史还短，LLM 还能看到全文
-> 之后被压缩成 [read_file result omitted from context]
-> 再之后被上下文裁剪彻底移出当前请求
-> 下次需要时再从 # Skills 索引重新读取
```
