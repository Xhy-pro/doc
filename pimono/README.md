<!-- OSS_WEEKEND_START -->
# OSS Weekend

**Issue tracker reopens Monday, April 6, 2026.**

OSS weekend runs Friday, March 27, 2026 through Monday, April 6, 2026. New issues are auto-closed during this time. For support, join [Discord](https://discord.com/invite/3cU7Bz4UPx).
<!-- OSS_WEEKEND_END -->

---

<p align="center">
  <a href="https://shittycodingagent.ai">
    <img src="https://shittycodingagent.ai/logo.svg" alt="pi logo" width="128">
  </a>
</p>
<p align="center">
  <a href="https://discord.com/invite/3cU7Bz4UPx"><img alt="Discord" src="https://img.shields.io/badge/discord-community-5865F2?style=flat-square&logo=discord&logoColor=white" /></a>
  <a href="https://github.com/badlogic/pi-mono/actions/workflows/ci.yml"><img alt="Build status" src="https://img.shields.io/github/actions/workflow/status/badlogic/pi-mono/ci.yml?style=flat-square&branch=main" /></a>
</p>
<p align="center">
  <a href="https://pi.dev">pi.dev</a> domain graciously donated by
  <br /><br />
  <a href="https://exe.dev"><img src="packages/coding-agent/docs/images/exy.png" alt="Exy mascot" width="48" /><br />exe.dev</a>
</p>

# Pi Monorepo

> **在找 pi coding agent？** 安装与使用说明见 **[packages/coding-agent](packages/coding-agent)**。

这是一个围绕 AI agent 构建与 LLM 部署管理的 monorepo。

## 包

| Package | Description |
|---------|-------------|
| **[@mariozechner/pi-ai](packages/ai)** | 统一的多提供商 LLM API（OpenAI、Anthropic、Google 等） |
| **[@mariozechner/pi-agent-core](packages/agent)** | 支持工具调用与状态管理的 agent runtime |
| **[@mariozechner/pi-coding-agent](packages/coding-agent)** | 交互式 coding agent CLI |
| **[@mariozechner/pi-mom](packages/mom)** | 将消息委托给 pi coding agent 的 Slack 机器人 |
| **[@mariozechner/pi-tui](packages/tui)** | 具备差异化渲染能力的终端 UI 库 |
| **[@mariozechner/pi-web-ui](packages/web-ui)** | 面向 AI 聊天界面的 Web 组件 |
| **[@mariozechner/pi-pods](packages/pods)** | 用于管理 GPU pod 上 vLLM 部署的 CLI |

## 贡献

贡献指南见 [CONTRIBUTING.md](CONTRIBUTING.md)，项目规则见 [AGENTS.md](AGENTS.md)。

## 开发

```bash
npm install          # 安装所有依赖
npm run build        # 构建所有包
npm run check        # 运行 lint、format 和类型检查
./test.sh            # 运行测试（没有 API key 时会跳过依赖 LLM 的测试）
./pi-test.sh         # 从源码运行 pi（可在任意目录执行）
```

> **注意：** `npm run check` 之前需要先执行 `npm run build`。`web-ui` 包使用 `tsc`，依赖其他包编译后的 `.d.ts` 文件。

## License

MIT
