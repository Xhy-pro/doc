# Skill 机制说明

本文梳理 `nanobot` 的 Skill 完整生命周期，包括初始化、发现、加载、使用、从上下文中剔除、跨轮回放，以及 Dream 自动创建 Skill 的流程。

这里说的“剔除”指 **Skill 全文不再出现在下一次发给模型的 `messages_for_model` 中**，不是删除磁盘上的 `SKILL.md` 文件。

## 总览

```mermaid
flowchart TD
    A["读取配置 disabledSkills<br/>schema.py:144 / loop.py:335,378"] --> B["AgentLoop 初始化 ContextBuilder<br/>loop.py:264"]
    B --> C["ContextBuilder 初始化 SkillsLoader<br/>context.py:60-64"]
    C --> D["扫描 workspace 和 builtin skills<br/>skills.py:29,51"]
    D --> E["构建 system prompt<br/>context.py:66"]
    E --> F{"是否 always skill?<br/>skills.py:203"}
    F -->|是| G["全文注入 # Active Skills<br/>context.py:87-91"]
    F -->|否| H["只把摘要注入 # Skills<br/>context.py:93-95"]
    H --> I["模型通过 read_file 读取 SKILL.md<br/>skills_section.md:3"]
    I --> J["read_file 返回 Skill 全文<br/>filesystem.py:162 / runner.py:270"]
    J --> K["全文作为 tool result 进入本轮 messages<br/>runner.py:360-382"]
    K --> L["下一次模型调用前做上下文治理<br/>runner.py:293-300"]
    L --> M{"被压缩或剔除?"}
    M -->|旧工具结果| N["_microcompact 摘要化<br/>runner.py:1204"]
    M -->|结果过大| O["落盘并只保留预览<br/>helpers.py:322"]
    M -->|上下文超预算| P["_snip_history 裁掉旧的非 system 历史<br/>runner.py:1250"]
    M -->|跨轮历史过旧| Q["session replay / consolidation 不再回放<br/>manager.py:132 / memory.py:670"]
```

## 初始化与发现

```mermaid
flowchart TD
    A["Config.agents.defaults.disabled_skills<br/>schema.py:144"] --> B["AgentLoop.from_config 传入 disabled_skills<br/>loop.py:335,378"]
    B --> C["AgentLoop.__init__ 创建 ContextBuilder<br/>loop.py:264"]
    B --> D["AgentLoop.__init__ 创建 SubagentManager<br/>loop.py:271,279"]
    C --> E["ContextBuilder.__init__ 创建 SkillsLoader<br/>context.py:60-64"]
    E --> F["SkillsLoader.workspace_skills = workspace/skills<br/>skills.py:29-32"]
    E --> G["SkillsLoader.builtin_skills = nanobot/skills<br/>skills.py:29-32"]
    F --> H["list_skills 先扫描 workspace<br/>skills.py:51,61"]
    G --> I["再扫描 builtin，跳过 workspace 同名 skill<br/>skills.py:63-65"]
    H --> J["disabled_skills 过滤<br/>skills.py:68-69"]
    I --> J
    J --> K["requirements 检查 bins/env<br/>skills.py:189"]
    K --> L["得到可用 skill 列表<br/>skills.py:72-73"]
```

Skill 查找目录：

```text
<workspace>/skills/<skill-name>/SKILL.md
nanobot/skills/<skill-name>/SKILL.md
```

规则：

- workspace skill 优先级高于 builtin skill。
- 如果 workspace 和 builtin 里有同名 skill，builtin 版本会被跳过。
- `disabledSkills` 同时作用于 workspace skill、builtin skill、主 agent 和 subagent。
- 依赖检查来自 frontmatter 的 `metadata.nanobot.requires`。
- 兼容 `metadata.openclaw` 结构。

依赖声明示例：

```yaml
metadata:
  nanobot:
    requires:
      bins: ["gh"]
      env: ["GITHUB_TOKEN"]
```

## Prompt 加载策略

```mermaid
flowchart TD
    A["ContextBuilder.build_messages<br/>context.py:179"] --> B["build_system_prompt<br/>context.py:66"]
    B --> C["加载 identity/bootstrap/tool contract/memory<br/>context.py:73-86"]
    C --> D["get_always_skills<br/>skills.py:203"]
    D --> E{"always 且 requirements 满足?"}
    E -->|是| F["load_skills_for_context<br/>skills.py:94"]
    F --> G["load_skill 读取 SKILL.md<br/>skills.py:75"]
    G --> H["_strip_frontmatter 去掉 YAML<br/>skills.py:161"]
    H --> I["全文注入 # Active Skills<br/>context.py:87-91"]
    E -->|否| J["build_skills_summary<br/>skills.py:111"]
    I --> J
    J --> K["摘要包含 name/description/path/availability<br/>skills.py:124-140"]
    K --> L["渲染 skills_section.md<br/>context.py:93-95"]
    L --> M["# Skills 提示模型用 read_file 读取 skill<br/>skills_section.md:1-6"]
```

加载策略分两类：

```text
always skill = 每轮把全文注入 system prompt
普通 skill = 每轮只注入摘要和路径
```

普通 skill 不会因为“可用”就自动把全文塞进上下文。模型必须显式读取对应的 `SKILL.md`。

## 使用 Skill

```mermaid
sequenceDiagram
    participant M as 模型
    participant R as AgentRunner
    participant T as ToolRegistry
    participant F as ReadFileTool
    participant FS as 文件系统

    M->>R: 看到 # Skills 摘要，决定使用某个 skill
    Note over M,R: skills_section.md:3 指示用 read_file 读取 SKILL.md
    M->>R: tool_call read_file(path=.../SKILL.md)
    R->>T: execute / prepare tool call<br/>registry.py:54,78
    T->>F: ReadFileTool.execute
    Note over F: ReadFileTool 属于 core scope<br/>filesystem.py:162
    F->>FS: _resolve(path)
    Note over F,FS: builtin skills 目录通过 extra_allowed_dirs 放行<br/>filesystem.py:51-64,76-86
    FS-->>F: SKILL.md 内容
    F-->>R: tool result：Skill 全文
    R->>R: _normalize_tool_result<br/>runner.py:1109
    R->>M: 下一次 messages_for_model 带上该 tool result<br/>runner.py:293-314
```

内置 skills 不在用户 workspace 里，所以 `ReadFileTool.create()` 专门把内置 skills 目录加入额外可读目录：

```mermaid
flowchart LR
    A["ReadFileTool.create<br/>filesystem.py:51"] --> B["extra_read = [BUILTIN_SKILLS_DIR]<br/>filesystem.py:52,60"]
    B --> C["extra_allowed_dirs=extra_read<br/>filesystem.py:64"]
    C --> D["_resolve 调用 resolve_workspace_path<br/>filesystem.py:76-86"]
    D --> E["允许 workspace / media / builtin skills<br/>path_utils.py:17-24"]
```

## 本轮内的上下文治理

每次请求模型前，`AgentRunner` 不直接发送原始 `messages`，而是先构造并治理 `messages_for_model`：

```mermaid
flowchart TD
    A["原始 messages<br/>runner.py:270"] --> B["复制并治理为 messages_for_model<br/>runner.py:293"]
    B --> C["_drop_orphan_tool_results<br/>runner.py:293,1137"]
    C --> D["_backfill_missing_tool_results<br/>runner.py:294,1163"]
    D --> E["_microcompact<br/>runner.py:295,1204"]
    E --> F["_apply_tool_result_budget<br/>runner.py:296,1229"]
    F --> G["_snip_history<br/>runner.py:297,1250"]
    G --> H["再次清理 orphan/backfill<br/>runner.py:299-300"]
    H --> I["发送给 provider<br/>runner.py:314"]

    E --> E1{"compactable tool results > 10<br/>且旧结果 >= 500 字符?"}
    E1 -->|是| E2["旧 read_file 结果变为<br/>[read_file result omitted from context]<br/>runner.py:1204-1226"]

    F --> F1{"tool result > max_tool_result_chars?"}
    F1 -->|是| F2["maybe_persist_tool_result 写入磁盘<br/>helpers.py:322"]
    F2 --> F3["上下文只保留路径和预览<br/>helpers.py:272,363"]

    G --> G1{"估算 token > budget?"}
    G1 -->|是| G2["保留 system + 最近合法非 system 后缀<br/>runner.py:1250"]
    G2 --> G3["旧 read_file(SKILL.md) tool result 可能消失<br/>runner.py:1290-1323"]
```

普通 skill 全文是 `read_file` 的 tool result，因此和其他可压缩工具结果走同一套治理逻辑：

- `_microcompact()`：把较旧的可压缩工具结果摘要化。
- `_apply_tool_result_budget()`：把超大的工具结果落盘，上下文里只留引用和预览。
- `_snip_history()`：上下文超预算时，裁掉旧的非 system 历史。

`always` skill 不同。它属于 system prompt 内容，不走这些 tool-result 剔除规则。

## 保存与跨轮回放

```mermaid
flowchart TD
    A["AgentRunner 返回 all_messages<br/>loop.py:1418-1444"] --> B["_state_save<br/>loop.py:1450"]
    B --> C["_save_turn 写入 session.messages<br/>loop.py:1541"]
    C --> D{"role == tool?"}
    D -->|是| E["超过 max_tool_result_chars 则截断<br/>loop.py:1541-1561"]
    D -->|否| F["保存 user/assistant 消息<br/>loop.py:1562-1585"]
    B --> G["session.enforce_file_cap<br/>loop.py:1474 / manager.py:348"]
    B --> H["后台 maybe_consolidate_by_tokens<br/>loop.py:1479 / memory.py:670"]

    I["下一轮 Session.get_history<br/>manager.py:132"] --> J["只取 last_consolidated 之后的消息<br/>manager.py:144"]
    J --> K["按 max_messages 取尾部窗口<br/>manager.py:145-146"]
    K --> L["避免从孤立 tool result 开始<br/>manager.py:159-161"]
    L --> M{"max_tokens > 0?"}
    M -->|是| N["从尾部按 token 预算保留<br/>manager.py:232-260"]
    M -->|否| O["进入 ContextBuilder.build_messages<br/>context.py:179"]
    N --> O
```

跨轮时，普通 skill 全文只有同时满足以下条件才会再次进入上下文：

1. 对应的 `read_file` tool result 还在 `session.messages` 里。
2. 它位于 `session.last_consolidated` 之后。
3. 它还在 `max_messages` 尾部窗口内。
4. 它符合 replay token budget。
5. 它仍然有合法的 assistant tool-call 边界。

## Consolidation 与长期退出

```mermaid
flowchart TD
    A["保存后调用 maybe_consolidate_by_tokens<br/>loop.py:1479"] --> B["Consolidator.maybe_consolidate_by_tokens<br/>memory.py:670"]
    B --> C["估算 replay tokens<br/>memory.py:670-726"]
    C --> D{"超过目标预算?"}
    D -->|否| E["不处理，历史继续可回放"]
    D -->|是| F["pick_consolidation_boundary<br/>memory.py:491"]
    F --> G["选择可安全移出 replay 的消息块<br/>memory.py:491-557"]
    G --> H["_consolidate_replay_overflow 总结旧消息<br/>memory.py:559"]
    H --> I["写入 memory/history.jsonl 或 raw_archive<br/>memory.py:417,559-668"]
    I --> J["推进 session.last_consolidated<br/>memory.py:670-766"]
    J --> K["Session.get_history 不再原样回放这些消息<br/>manager.py:144"]
```

Consolidation 不是 skill 专用机制。它处理所有旧消息，其中也包括旧的 `read_file(SKILL.md)` 工具结果。

## Dream 自动创建 Skill

```mermaid
flowchart TD
    A["Dream.run 读取未处理历史<br/>memory.py:1009"] --> B["Phase 1 分析历史<br/>memory.py:1057"]
    B --> C["重复工作流输出 [SKILL]<br/>dream_phase1.md:8,32"]
    C --> D["Phase 2 要求创建 skills/name/SKILL.md<br/>dream_phase2.md:4,10,22"]
    D --> E["Dream._build_tools 注册 read/edit/write<br/>memory.py:906"]
    E --> F["write_file 只能写 workspace/skills<br/>memory.py:927-930"]
    D --> G["_list_existing_skills 提供去重上下文<br/>memory.py:935"]
    G --> H["读取 skill-creator 作为格式参考<br/>memory.py:1086-1098"]
    H --> I["AgentRunner 执行 write_file 创建 workspace skill<br/>memory.py:1009"]
    I --> J["后续 SkillsLoader 自动发现<br/>skills.py:51"]
```

Dream 创建的是 workspace skill，因此如果和 builtin skill 同名，会优先于 builtin skill。

## 状态机视角

```mermaid
stateDiagram-v2
    [*] --> DiskSkill: SKILL.md 存在
    DiskSkill --> Listed: list_skills 扫描到\nskills.py:51
    Listed --> Disabled: 命中 disabledSkills\nskills.py:68
    Listed --> Unavailable: requirements 不满足\nskills.py:189
    Listed --> AlwaysFull: always=true\nskills.py:203
    Listed --> SummaryOnly: 普通 skill\nskills.py:111

    Disabled --> [*]: 不进入 prompt
    Unavailable --> SummaryOnly: 可能以 unavailable 摘要出现\nskills.py:132-138

    AlwaysFull --> SystemPrompt: 每轮全文注入\ncontext.py:87-91
    SystemPrompt --> AlwaysFull: 下一轮重新构建 prompt

    SummaryOnly --> SkillIndex: 摘要和路径进入 # Skills\ncontext.py:93-95
    SkillIndex --> ToolResultFull: 模型用 read_file 读取\nskills_section.md:3
    ToolResultFull --> ToolResultReference: 超过 max_tool_result_chars\nhelpers.py:322
    ToolResultFull --> ToolResultOmitted: 旧 compactable 结果超过保留数量\nrunner.py:1204
    ToolResultFull --> Snipped: prompt 超预算\nrunner.py:1250
    ToolResultReference --> Snipped: prompt 超预算\nrunner.py:1250
    ToolResultOmitted --> Snipped: prompt 超预算\nrunner.py:1250

    ToolResultFull --> SessionHistory: _save_turn 持久化\nloop.py:1541
    ToolResultReference --> SessionHistory: _save_turn 持久化\nloop.py:1541
    ToolResultOmitted --> SessionHistory: microcompact 只影响发给模型的副本\nrunner.py:293

    SessionHistory --> Replayed: get_history 回放\nmanager.py:132
    Replayed --> ToolResultFull: 仍在 replay 窗口和 token 预算内
    SessionHistory --> Consolidated: consolidation 推进 last_consolidated\nmemory.py:670
    Consolidated --> [*]: 不再作为原始历史回放\nmanager.py:144
```

## 关键代码索引

- Skill loader：`nanobot/agent/skills.py:21`
- Loader 初始化：`nanobot/agent/skills.py:29`
- 列出 skills：`nanobot/agent/skills.py:51`
- 读取单个 skill：`nanobot/agent/skills.py:75`
- always skills：`nanobot/agent/skills.py:203`
- system prompt 注入：`nanobot/agent/context.py:66`
- 构建 messages：`nanobot/agent/context.py:179`
- Skill prompt 模板：`nanobot/templates/agent/skills_section.md:1`
- read_file 访问 builtin skills：`nanobot/agent/tools/filesystem.py:51`
- runner 上下文治理：`nanobot/agent/runner.py:293`
- microcompact：`nanobot/agent/runner.py:1204`
- 工具结果预算：`nanobot/agent/runner.py:1229`
- 历史裁剪：`nanobot/agent/runner.py:1250`
- 超大工具结果落盘：`nanobot/utils/helpers.py:322`
- 保存 turn：`nanobot/agent/loop.py:1541`
- session replay：`nanobot/session/manager.py:132`
- consolidation：`nanobot/agent/memory.py:670`
- Dream skill 发现：`nanobot/templates/agent/dream_phase1.md:8`
- Dream skill 创建：`nanobot/templates/agent/dream_phase2.md:4`

