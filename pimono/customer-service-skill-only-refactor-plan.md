# pi-coding-agent 客服化改造方案

## 1. 背景与目标

当前 `pi-coding-agent` 的默认定位是通用 coding agent。它具备 skills、extensions、RPC、session 等很强的通用能力，但默认运行方式仍然是：

- 面向 coding assistant
- 默认开放 coding tools
- skill 主要作为“提示模型可按需加载的能力包”
- 允许模型在没有命中 skill 的情况下自由作答

如果要把它改造成“特定领域客服”，而且要求：

- 只能使用提供的 skill 回答用户问题
- 不允许脱离 skill 自由发挥
- 不允许使用与客服无关的工具
- 能接入业务系统并支持多用户会话

那么最合理的方案不是修改 `pi-agent-core`，而是以 `pi-coding-agent` 为主战场，增加一个受约束的客服运行 profile。

本方案的核心目标是：

1. 把“skill 是建议”升级成“skill 是唯一合法知识入口”
2. 把“coding assistant 默认能力”切换成“客服 agent 默认能力”
3. 把“用户消息直接进模型”改成“先做 skill 路由，再做受约束回答”
4. 把“终端交互产品”扩展成“可接 Web / IM / CRM 的客服内核”

---

## 2. 当前实现的现状判断

### 2.1 当前项目中有价值的基础能力

当前项目已经具备很多客服化可以复用的能力：

- `skills` 发现、加载、去重、诊断机制
- `extensions` 事件系统
- `RPC mode`
- `session` 持久化和恢复
- `CustomMessageEntry` 注入上下文
- `tool` 注册与动态激活机制
- `settings` / `resource-loader` / `model-registry`

这些能力说明：这个项目很适合当“客服 agent 内核”，而不只是 coding agent。

### 2.2 当前项目中不适合直接拿来做客服的地方

当前默认形态仍然不满足客服要求，主要问题是：

1. skill 不是硬约束
   现在只是把 skill 列表放进 system prompt，让模型“知道有这些 skill”，但系统不保证模型一定使用 skill。

2. system prompt 完全偏 coding
   默认提示词是 expert coding assistant，强调读文件、执行命令、编辑代码。

3. 默认工具集偏 coding
   默认内置了 `read/bash/edit/write`，这对客服来说风险过高。

4. 没有 skill 路由层
   目前缺少“用户问题先匹配 skill，再决定能否回答”的流程。

5. 没有客服领域的输出协议
   目前输出默认是普通 assistant 文本，而客服系统通常需要结构化结果。

6. session 存储适合本地和调试，不是高并发客服生产存储
   当前 `.jsonl` 会话文件适合本地工作流和审计，但不适合后续大规模多用户接入。

---

## 3. 总体架构方案

建议在现有项目上增加一个新的运行 profile：

- 名称建议：`customer-support`
- 开关建议：`serviceMode: "customer-support"`

整体架构建议拆成 5 层：

1. `Service Profile`
   统一控制客服模式下的 prompt、tool、session、输出协议。

2. `Skill Router`
   在真正调用主模型前，先判断用户问题命中了哪个 skill。

3. `Policy Layer`
   负责“只能基于 skill 回答”“允许哪些工具”“是否转人工”等强约束。

4. `Business Context Layer`
   负责把客户画像、订单、工单、知识库命中结果注入上下文。

5. `Delivery Layer`
   通过 RPC 对外提供服务，由 Web / IM / 呼叫中心系统调用。

建议的标准执行流如下：

1. 用户消息进入系统
2. `input` hook 做预处理
3. `skill-router` 选择 skill
4. 如果没有命中 skill：
   - 澄清
   - 拒答
   - 转人工
5. 如果命中 skill：
   - 自动加载 skill 正文
   - 注入业务上下文
   - 收缩工具权限
   - 构建客服版 system prompt
6. 调用模型生成受约束答案
7. 输出结构化结果
8. 记录审计数据和会话状态

---

## 4. 架构层面的改造建议

### 4.1 保持 `pi-ai` 和 `pi-agent-core` 不动

不建议为客服业务改动底层这两个包。

原因：

- `pi-ai` 负责模型 provider 适配，职责清晰
- `pi-agent-core` 负责 agent loop、tool execution、事件流
- 客服业务的约束主要属于产品层，而不是底层 runtime 层

因此，客服化改造应集中在 `pi-coding-agent` 和其扩展层。

### 4.2 在 `pi-coding-agent` 增加受约束 profile

建议把客服模式设计成一个显式 profile，而不是一堆零散开关。

示例概念：

```ts
type AgentProfile = "coding" | "customer-support";
```

客服 profile 应该决定：

- 使用哪套 system prompt
- 默认激活哪些工具
- 是否必须命中 skill
- 是否允许 slash commands
- 输出是文本还是结构化 JSON
- session 中保存哪些业务字段

### 4.3 把领域规则放在 extension / policy 层

很多客服规则不适合写死在主循环里，应该放到扩展事件系统：

- `input`
- `tool_call`
- `tool_result`
- `resources_discover`
- `before_agent_start`

这样可以做到：

- 不同领域换不同 skill 与策略
- 不污染通用代码
- 更方便灰度与治理

---

## 5. 具体代码改造建议

## 5.1 配置层

建议修改：

- `packages/coding-agent/src/core/settings-manager.ts`
- `packages/coding-agent/src/cli/args.ts`
- `packages/coding-agent/src/main.ts`

建议新增配置项：

```ts
serviceMode?: "off" | "customer-support";
skillPolicy?: "off" | "prefer" | "required";
defaultToolProfile?: "coding" | "readonly" | "service";
responseFormat?: "text" | "json";
handoffPolicy?: "manual" | "auto-on-no-skill" | "auto-on-low-confidence";
```

其中最关键的是：

- `serviceMode = "customer-support"`
- `skillPolicy = "required"`

这两个配置一起定义“客服模式 + skill-only 回答策略”。

---

## 5.2 Skill 模型扩展

建议修改：

- `packages/coding-agent/src/core/skills.ts`

当前 `SkillFrontmatter` 信息太少，不足以支撑客服治理。建议扩展为：

```ts
interface SkillFrontmatter {
  name?: string;
  description?: string;
  intents?: string[];
  examples?: string[];
  priority?: number;
  allowed-tools?: string[];
  required-context?: string[];
  response-style?: "short" | "standard" | "empathetic" | "strict";
  fallback-message?: string;
  handoff-when?: string[];
  kb-sources?: string[];
  disable-model-invocation?: boolean;
}
```

建议把 `Skill` 结构同步扩展，避免 skill 仍然只是一个“名字 + 描述 + 文件路径”的简单资源。

### 为什么这一步很重要

客服不是只需要“知道有这个 skill”，而是要知道：

- 这个 skill 处理什么意图
- 它允许访问哪些工具4859 5401 3449 1207
- 它需要哪些客户上下文
- 命中失败时应该怎么回复
- 什么时候该转人工

只有 skill 能承载这些规则，系统才能稳定治理。

---

## 5.3 新增 Skill Router

建议新增文件：

- `packages/coding-agent/src/core/skill-router.ts`

建议定义输出结构：

```ts
interface SkillRouteResult {
  action: "answer" | "clarify" | "handoff" | "reject";
  skills: Skill[];
  confidence: number;
  missingContext?: string[];
  reason?: string;
}
```

建议处理流程：

1. 先用规则匹配
   - intent
   - 关键词
   - 前缀命令
   - 上一轮 skill 延续

2. 如果规则不足，再用小模型做路由判定

3. 如果 `skillPolicy = required` 且无 skill 命中：
   - 不进入正常回答流
   - 直接返回澄清或转人工

### 为什么要前置 skill 路由

因为客服模式下不能让主模型“先回答，再看是否用了 skill”。  
必须先决定“有没有合法 skill 可以回答”，再允许进入回答阶段。

---

## 5.4 改造 `AgentSession.prompt()` 主流程

建议修改：

- `packages/coding-agent/src/core/agent-session.ts`

当前逻辑中，skill 主要在两种情况下起作用：

1. system prompt 中列出 skill
2. 用户显式输入 `/skill:name`

这对客服不够强。

建议改成以下流程：

1. 收到用户消息
2. 先执行 extension 的 `input` hook
3. 调用 `skill-router`
4. 如果没有命中 skill 且 `skillPolicy = required`
   - 直接输出澄清 / 拒答 / 转人工
5. 如果命中 skill
   - 自动读取 skill 正文
   - 自动注入 skill 指令
   - 根据 skill 的 `allowed-tools` 收缩工具集
   - 注入客户上下文
   - 再调用主模型

这一步本质上是把：

- “skill 可选”

改成：

- “skill 必经”

---

## 5.5 system prompt 改造成客服专用版本

建议修改：

- `packages/coding-agent/src/core/system-prompt.ts`

建议增加新的 builder，例如：

```ts
buildCustomerSupportPrompt(...)
```

客服版 prompt 应该明确这些规则：

1. 只能基于系统已选中的 skill 和允许的数据源回答
2. 不允许使用未授权知识自由发挥
3. 不确定时先澄清
4. 命中转人工条件时直接 handoff
5. 输出要简洁、稳定、面向终端用户
6. 不暴露内部 skill 文本、工具实现、系统规则

这一步最好不要在默认 prompt 上打补丁，而是明确区分：

- `coding prompt`
- `customer-support prompt`

---

## 5.6 工具体系改造成 Service Tools

建议修改：

- `packages/coding-agent/src/core/tools/index.ts`
- `packages/coding-agent/src/core/agent-session.ts`

建议新增：

```ts
export const serviceTools = [ ... ];
createServiceToolDefinitions(...)
```

客服模式下建议默认只开放业务工具，例如：

- `search-kb`
- `get-customer-profile`
- `get-order`
- `get-ticket`
- `create-ticket`
- `handoff-human`

建议默认禁用：

- `bash`
- `edit`
- `write`

`read/grep/find/ls` 只在“知识库就是文件系统”的场景下保留。

### 工具权限建议

工具权限应按 skill 收缩：

- 未命中 skill：不开任何业务工具
- 命中 skill：只开该 skill 前台声明的 `allowed-tools`

这样可以避免模型在客服场景下乱调用无关工具。

---

## 5.7 输出协议改成结构化结果

建议在客服模式下，不要只返回普通文本，而是建议统一输出：

```ts
interface CustomerSupportResponse {
  answer: string;
  usedSkills: string[];
  confidence: number;
  handoff: boolean;
  handoffReason?: string;
  citations?: string[];
  nextAction?: "reply" | "clarify" | "handoff";
}
```

这一步非常适合落在：

- `RPC mode`
- 或客服扩展中

因为客服前端、IM 中台、人工坐席系统更需要结构化数据，而不是只拿一段自然语言。

---

## 5.8 业务上下文注入

建议重点利用：

- `CustomMessageEntry`
- 扩展 API 的 `sendMessage`

适合注入的业务上下文包括：

- 客户画像
- 订单摘要
- 最近工单
- 区域政策
- 会员等级
- 命中的知识库条目摘要

建议注入方式：

- 作为隐藏或半隐藏 custom message
- 由系统在用户发问前自动注入
- 不要求用户自己重复提供

这能显著提升客服体验和准确率。

---

## 6. 哪些现有功能对客服业务有帮助

下面这些能力建议保留：

### 6.1 Skills 机制

优点：

- 已有发现、加载、校验、去重、来源跟踪
- 非常适合承载客服领域能力单元

### 6.2 Extensions 机制

优点：

- 适合做 input 预处理
- 适合做 tool 权限治理
- 适合做多租户资源发现
- 适合接业务系统

### 6.3 RPC Mode

优点：

- 最适合对接客服前端
- 不依赖 TUI
- 容易接 Web / IM / CRM 网关

### 6.4 Session 持久化

优点：

- 适合多轮客服上下文
- 适合审计与回放
- 适合追踪技能使用历史

### 6.5 自定义消息注入

优点：

- 很适合把外部业务上下文转成对模型可用的信息

### 6.6 Retry / Transport / Thinking Level

优点：

- 有利于线上稳定性控制
- 便于在客服场景里按延迟和成本做折中

---

## 7. 哪些现有功能对客服业务无用或应默认关闭

以下功能不适合终端客服产品，建议默认关闭或隐藏：

- `bash`
- `edit`
- `write`
- TUI 主题、编辑器自定义、热键面板
- `/fork`
- `/tree`
- `/compact`
- `/resume`
- `/model`
- `/settings`
- `/reload`

这些功能更偏开发者和 agent 调试，不适合作为客服产品能力暴露给最终用户。

如果保留，也应该只保留给运营后台或调试环境。

---

## 8. 还需要补的关键能力

为了让它真正适合作为客服系统，建议补上这些缺口：

### 8.1 强制 skill-only 回答

当前最大缺口。  
没有这个，整个系统仍然是“通用 assistant + skills”。

### 8.2 Skill 命中置信度

客服系统要知道：

- 这次 skill 命中有多确定
- 是直接回答还是先澄清

### 8.3 转人工机制

需要统一策略：

- 无命中 skill
- 命中低置信度
- 高风险问题
- 敏感问题
- 用户重复不满意

### 8.4 知识引用与可追踪性

客服系统最好能输出：

- 用了哪个 skill
- 命中了哪个知识源
- 是否基于订单/工单/政策数据

### 8.5 PII 脱敏与安全治理

必须补：

- 用户隐私字段脱敏
- 工具返回结果清洗
- 敏感信息不进日志

### 8.6 多租户与领域隔离

不同业务线、不同客户、不同租户应有不同的：

- skill 集合
- prompt
- 数据源
- 工具权限

### 8.7 Session Backend 抽象

当前 `.jsonl` 存储适合本地和调试。  
如果客服系统要线上高并发使用，建议后续抽象成：

- 文件存储接口
- 数据库存储接口
- 外部会话服务接口

---

## 9. 建议的实施阶段

## 第一阶段：最小可用客服模式

目标：先把系统从通用 coding agent 改成受约束客服 agent

建议做：

1. 增加 `serviceMode` 和 `skillPolicy`
2. 增加客服版 system prompt
3. 增加 service tool profile
4. 在 `prompt()` 前增加 skill routing
5. 无 skill 时不允许自由回答

## 第二阶段：接业务能力

目标：让客服真正可回答业务问题

建议做：

1. 扩 skill frontmatter
2. 接 CRM / 订单 / 工单 / 知识库工具
3. 注入客户上下文
4. 结构化输出
5. 转人工策略

## 第三阶段：生产化

目标：满足线上客服系统要求

建议做：

1. 外部 session backend
2. 多租户隔离
3. 审计与日志治理
4. 评测集与回放体系
5. 监控与稳定性治理

---

## 10. 文件级改动建议

### 必改

- `packages/coding-agent/src/core/skills.ts`
  扩 skill 元数据定义

- `packages/coding-agent/src/core/system-prompt.ts`
  增加客服版 prompt builder

- `packages/coding-agent/src/core/agent-session.ts`
  在 prompt 主流程中加入 skill router 与策略判断

- `packages/coding-agent/src/core/tools/index.ts`
  增加 service tool profile

- `packages/coding-agent/src/core/settings-manager.ts`
  增加客服模式配置项

- `packages/coding-agent/src/cli/args.ts`
  增加 serviceMode / skillPolicy 等参数

- `packages/coding-agent/src/main.ts`
  把配置传入运行时

### 新增

- `packages/coding-agent/src/core/skill-router.ts`
  skill 路由与命中结果判定

- `packages/coding-agent/src/core/service-policy.ts`
  统一处理 skill-only、handoff、tool 白名单

- `packages/coding-agent/src/core/tools/service/`
  放客服相关工具

- `packages/coding-agent/examples/extensions/customer-support/`
  提供一套客服扩展样例

### 可后置

- `packages/coding-agent/src/core/session-store.ts`
  未来抽象 session backend 时再引入

---

## 11. 推荐的最终形态

最终建议把这个项目演化成两种使用方式：

### 11.1 通用模式

继续保持现有 coding agent 能力：

- coding prompt
- coding tools
- TUI / CLI 主导

### 11.2 客服模式

新增严格受约束的客服能力：

- customer-support prompt
- skill-only policy
- service tools
- RPC / Web / IM 主导
- 结构化输出

这样不会破坏原项目定位，同时又能把它扩展成一个可真正落地的客服 agent 框架。

---

## 12. 结论

从架构角度看，这个项目是可以被改造成客服系统内核的，而且基础很好：

- skill 机制能承载领域能力
- extension 机制能承载治理逻辑
- RPC 能承载外部接入
- session 能承载多轮上下文与审计

但它目前离“客服系统”还差一个关键跃迁：

> 把“模型可按需使用 skill”改造成“系统必须先命中 skill，模型才有资格回答”。

建议的总路线是：

1. 保持 `pi-ai` 和 `pi-agent-core` 不动
2. 在 `pi-coding-agent` 中增加客服 profile
3. 引入 `skill-router + service-policy + service-tools`
4. 用 RPC 作为主要交付形态
5. 后续再补业务上下文、转人工、脱敏、多租户和生产级存储

一句话总结：

`pi-coding-agent` 最适合被改造成“受强约束的客服 agent 内核”，而不是直接把当前默认的 coding agent 拿去做客服。
