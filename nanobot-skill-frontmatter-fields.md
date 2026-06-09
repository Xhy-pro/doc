# Nanobot Skill 支持的属性参数

本文根据 nanobot 源码整理 `SKILL.md` frontmatter 支持哪些字段、哪些字段运行时真正生效、哪些字段只是校验器允许或生态兼容字段。

结论先说：

```text
运行时最核心、真正影响加载行为的字段：

- description
- always
- metadata.nanobot.always
- metadata.nanobot.requires.bins
- metadata.nanobot.requires.env
- metadata.openclaw.always
- metadata.openclaw.requires.bins
- metadata.openclaw.requires.env
```

其他字段可能会被 YAML 解析出来，但当前 `SkillsLoader` 不一定使用它们做加载决策。

## 一、示例 frontmatter

一个比较完整的 skill frontmatter 可以长这样：

```yaml
---
name: weather
description: Get current weather and forecasts. Use when the user asks about weather, forecast, temperature, rain, wind, or local conditions.
always: false
metadata:
  nanobot:
    emoji: "🌤️"
    requires:
      bins: ["curl"]
      env: ["WEATHER_API_KEY"]
    install:
      - id: brew
        kind: brew
        formula: curl
        bins: ["curl"]
        label: Install curl
license: MIT
allowed-tools: read_file, exec
---
```

但不是每个字段都会被 nanobot 运行时真正使用。

## 二、运行时真正使用的字段

运行时逻辑主要在：

```text
nanobot/nanobot/agent/skills.py
```

`SkillsLoader` 会读取 `SKILL.md` 的 YAML frontmatter，并用其中一部分字段决定：

- skill 是否出现在 `# Skills` 索引里。
- skill 的索引描述是什么。
- skill 是否进入 `# Active Skills` 全文注入。
- skill 是否因为依赖缺失被标注 unavailable。

### 1. `description`

作用：

```text
进入 # Skills 索引，帮助 LLM 判断什么时候应该读取这个 skill。
```

例如：

```yaml
description: Get current weather and forecasts.
```

system prompt 中会出现：

```text
- **weather** — Get current weather and forecasts. `/path/to/weather/SKILL.md`
```

如果没有 `description`，运行时会 fallback 到 skill 名字。

注意：`description` 很重要。普通 skill 的正文不会自动注入，LLM 每轮主要靠 `description` 判断要不要 `read_file`。

### 2. `always`

作用：

```text
如果 always: true，并且 requirements 满足，则 skill 全文进入 # Active Skills。
```

例如：

```yaml
always: true
```

结果：

```text
# Active Skills

### Skill: memory

完整 SKILL.md 正文...
```

`always` 适合非常基础、每轮都应该被模型看到的短 skill。

缺点是：它每轮都占用 system prompt 预算。

### 3. `metadata.nanobot.always`

作用和顶层 `always` 类似。

例如：

```yaml
metadata:
  nanobot:
    always: true
```

`get_always_skills()` 会检查：

```text
顶层 always
或 metadata.nanobot.always
或 metadata.openclaw.always
```

所以这也是有效的 always 标记。

### 4. `metadata.nanobot.requires.bins`

作用：

```text
声明这个 skill 依赖哪些 CLI 命令。
```

例如：

```yaml
metadata:
  nanobot:
    requires:
      bins: ["gh"]
```

运行时会用 `shutil.which("gh")` 检查命令是否存在。

如果不存在：

- `list_skills(filter_unavailable=True)` 会过滤掉它。
- always skill 不会进入 `# Active Skills`。
- `build_skills_summary()` 仍会把它放进 `# Skills`，但标注 unavailable。

索引中可能显示：

```text
- **github** — Interact with GitHub using the gh CLI. (unavailable: CLI: gh) `/path/to/github/SKILL.md`
```

### 5. `metadata.nanobot.requires.env`

作用：

```text
声明这个 skill 依赖哪些环境变量。
```

例如：

```yaml
metadata:
  nanobot:
    requires:
      env: ["GITHUB_TOKEN"]
```

运行时会检查：

```python
os.environ.get("GITHUB_TOKEN")
```

如果环境变量不存在，处理方式和缺少 CLI 类似。

### 6. `metadata.openclaw.*`

nanobot 兼容 OpenClaw 风格 metadata。

也就是说，下面也会被识别：

```yaml
metadata:
  openclaw:
    requires:
      bins: ["gh"]
    always: true
```

源码中的解析逻辑会优先取 `nanobot`，如果没有则取 `openclaw`：

```python
payload = data.get("nanobot", data.get("openclaw", {}))
```

所以支持：

- `metadata.openclaw.always`
- `metadata.openclaw.requires.bins`
- `metadata.openclaw.requires.env`

## 三、会被解析但当前运行时不强制使用的字段

这些字段可以出现在内置 skill 中，也可能被 YAML 解析出来，但当前 `SkillsLoader` 不用它们决定加载、过滤或注入。

### 1. `name`

示例：

```yaml
name: weather
```

运行时发现 skill 时，主要使用目录名：

```text
skills/weather/SKILL.md
```

也就是说，运行时 entry 的 name 来自目录 `weather`，不是强依赖 frontmatter 的 `name`。

不过创建 skill 时仍然应该写 `name`，并且让它和目录名一致。

### 2. `metadata.nanobot.emoji`

示例：

```yaml
metadata:
  nanobot:
    emoji: "🌤️"
```

内置 skill 里有使用，比如 weather、github、tmux。

但当前 `SkillsLoader` 不用 `emoji` 做：

- 可用性判断。
- Active 注入判断。
- summary 文本生成。

它更像展示/生态元信息。

### 3. `metadata.nanobot.install`

示例：

```yaml
metadata:
  nanobot:
    install:
      - id: brew
        kind: brew
        formula: gh
        bins: ["gh"]
        label: Install GitHub CLI
```

内置 github/summarize skill 有类似字段。

但当前 `SkillsLoader` 不会自动执行安装，也不会用它参与 requirements 检查。

真正会检查的是：

```text
metadata.nanobot.requires.bins
metadata.nanobot.requires.env
```

`install` 只是给人或其他工具看的安装提示。

### 4. `metadata.nanobot.os`

示例：

```yaml
metadata:
  nanobot:
    os: ["darwin", "linux"]
```

内置 tmux skill 写了 `os`。

但当前 `_check_requirements()` 只检查：

```text
requires.bins
requires.env
```

不检查 `os`。

所以 `os` 当前不会阻止 skill 出现在索引里。

### 5. `homepage`

示例：

```yaml
homepage: https://wttr.in/:help
```

内置 weather、summarize、clawhub 有写。

运行时 loader 会通过 YAML 读到这个字段，但当前 `SkillsLoader` 不用它生成 summary，也不用它判断可用性。

### 6. `license`

skill-creator 校验器允许 `license`。

示例：

```yaml
license: MIT
```

运行时 `SkillsLoader` 不使用它。

### 7. `allowed-tools`

skill-creator 校验器允许 `allowed-tools`。

示例：

```yaml
allowed-tools: read_file, exec
```

当前 nanobot 运行时不会因为这个字段限制 LLM 能调用哪些工具。真正传给 LLM 的工具列表来自 `ToolRegistry`。

所以它更像文档/生态兼容字段，而不是运行时权限控制。

## 四、skill-creator 校验器允许的字段

skill-creator 的校验脚本在：

```text
nanobot/nanobot/skills/skill-creator/scripts/quick_validate.py
```

它允许的 top-level frontmatter key 是：

```python
ALLOWED_FRONTMATTER_KEYS = {
    "name",
    "description",
    "metadata",
    "always",
    "license",
    "allowed-tools",
}
```

也就是：

```yaml
---
name: my-skill
description: ...
metadata: ...
always: true
license: MIT
allowed-tools: read_file, exec
---
```

注意：这和运行时 loader 的宽松解析不完全一样。

运行时 loader 会读 YAML 中的所有 key，但只使用部分字段。

skill-creator 校验器会拒绝不在允许列表里的 key。

例如内置 skill 中出现的：

```yaml
homepage: https://wttr.in/:help
```

运行时可以读到，但 skill-creator 的严格校验器不在 `ALLOWED_FRONTMATTER_KEYS` 里列 `homepage`，因此可能被认为是 unexpected key。

## 五、skill-creator 校验规则

校验器不只是检查字段名，还检查字段格式。

### 1. `name` 必填

必须存在：

```yaml
name: my-skill
```

### 2. `description` 必填

必须存在：

```yaml
description: Use this skill when ...
```

### 3. `name` 必须是小写 hyphen-case

合法：

```text
my-skill
github
pdf-tools
skill-123
```

不推荐或不合法：

```text
MySkill
my_skill
my--skill
my skill
```

校验正则：

```text
[a-z0-9]+(?:-[a-z0-9]+)*
```

### 4. `name` 最长 64 字符

超过会校验失败。

### 5. `name` 必须和目录名一致

例如目录是：

```text
skills/weather/
```

frontmatter 应该是：

```yaml
name: weather
```

如果目录叫 `weather`，frontmatter 却写：

```yaml
name: forecast
```

校验会失败。

### 6. `description` 必须是字符串

不能是数组、对象或数字。

### 7. `description` 不能为空

空字符串会失败。

### 8. `description` 不能包含 TODO 占位符

例如：

```yaml
description: TODO
```

会失败。

### 9. `description` 不能包含尖括号

不能包含：

```text
<
>
```

### 10. `description` 最长 1024 字符

超过会失败。

### 11. `always` 如果存在，必须是 boolean

合法：

```yaml
always: true
always: false
```

不合法：

```yaml
always: "true"
always: yes
always: 1
```

实际 YAML 里 `always: true` 会被解析成 Python boolean `True`。

## 六、运行时字段表

| 字段 | 运行时是否使用 | 作用 |
|------|----------------|------|
| `name` | 弱使用 | 运行时主要用目录名；创建/校验时要求它和目录名一致 |
| `description` | 是 | 进入 `# Skills` 索引，帮助 LLM 判断是否读取 skill |
| `always` | 是 | `true` 时全文注入 `# Active Skills` |
| `metadata.nanobot.always` | 是 | 另一种 always 标记 |
| `metadata.openclaw.always` | 是 | OpenClaw 兼容 always 标记 |
| `metadata.nanobot.requires.bins` | 是 | 检查 CLI 是否存在 |
| `metadata.nanobot.requires.env` | 是 | 检查环境变量是否存在 |
| `metadata.openclaw.requires.bins` | 是 | OpenClaw 兼容 CLI 依赖 |
| `metadata.openclaw.requires.env` | 是 | OpenClaw 兼容环境变量依赖 |
| `metadata.nanobot.emoji` | 否 | 当前 loader 不使用，展示/生态信息 |
| `metadata.nanobot.install` | 否 | 当前 loader 不自动安装，只是安装提示信息 |
| `metadata.nanobot.os` | 否 | 当前 loader 不检查 OS |
| `homepage` | 否 | 当前 loader 不使用 |
| `license` | 否 | 校验器允许，运行时不使用 |
| `allowed-tools` | 否 | 校验器允许，运行时不限制工具 |

## 七、最小推荐写法

普通 skill：

```yaml
---
name: weather
description: Get current weather and forecasts. Use when the user asks about weather, forecast, temperature, rain, wind, or local conditions.
metadata:
  nanobot:
    requires:
      bins: ["curl"]
---
```

常驻 skill：

```yaml
---
name: memory
description: Two-layer memory system with Dream-managed knowledge files.
always: true
---
```

带安装提示的 skill：

```yaml
---
name: github
description: Interact with GitHub using the gh CLI. Use for issues, PRs, workflow runs, and GitHub API operations.
metadata:
  nanobot:
    emoji: "🐙"
    requires:
      bins: ["gh"]
    install:
      - id: brew
        kind: brew
        formula: gh
        bins: ["gh"]
        label: Install GitHub CLI
---
```

带环境变量要求的 skill：

```yaml
---
name: private-api
description: Use the private API. Use when the user asks to query internal service data.
metadata:
  nanobot:
    requires:
      env: ["PRIVATE_API_TOKEN"]
---
```

## 八、实践建议

### 1. 把触发条件写进 `description`

普通 skill 的正文不会自动进入上下文。

LLM 每轮能稳定看到的是 `# Skills` 索引里的 `description`。

所以 description 应该写：

```text
这个 skill 做什么，以及什么时候应该使用它。
```

不要只写：

```yaml
description: Weather tool.
```

更好的写法：

```yaml
description: Get current weather and forecasts. Use when the user asks about weather, forecast, temperature, rain, wind, or local conditions.
```

### 2. 谨慎使用 `always: true`

`always: true` 会让 skill 全文每轮都进入 system prompt。

适合：

- 很短。
- 非常基础。
- 几乎每轮都可能需要。
- 不希望依赖 LLM 主动 read_file。

不适合：

- 很长。
- 很少用。
- 只在特定领域任务才用。

### 3. requirements 只写真正会阻塞使用的依赖

如果没有某个 CLI 或环境变量就完全不能用，才写进 `requires`。

否则 skill 会被标 unavailable，甚至 always skill 不会进入 Active。

### 4. 不要以为 `install` 会自动安装

当前 loader 不会执行：

```yaml
install:
  - ...
```

如果需要安装，LLM 只能根据 skill 正文或 unavailable 提示，引导用户安装。

### 5. 不要以为 `allowed-tools` 会限制工具权限

当前运行时工具权限来自 `ToolRegistry`，不是 skill frontmatter 的 `allowed-tools`。

如果要限制工具，需要从工具注册/配置层处理，而不是只写在 skill frontmatter。

## 九、一句话总结

nanobot skill frontmatter 的核心有效字段其实很少：

```text
description 决定普通 skill 是否容易被 LLM 发现；
always 决定是否全文常驻 system prompt；
metadata.nanobot.requires.bins/env 决定依赖是否满足；
metadata.openclaw.* 是兼容格式。
```

其他字段可以作为生态、展示、安装提示或校验信息存在，但当前 `SkillsLoader` 不用它们做核心加载决策。
