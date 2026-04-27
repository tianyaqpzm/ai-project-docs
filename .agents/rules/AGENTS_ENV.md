---
trigger: always_on
---

# Terminal Execution Constraint

When executing any terminal commands, scripts, builds, or tests (especially those involving `node`, `npm`, `yarn`, or `pnpm`), you MUST explicitly source the Zsh configuration file first to ensure all required environment variables and PATHs are loaded.

You MUST ALWAYS prepend `source ~/.zshrc && ` to your intended command.

** Examples of correct execution:**
- Instead of `node test.js`, you MUST execute: `source ~/.zshrc && node test.js`
- Instead of `npm run build`, you MUST execute: `source ~/.zshrc && npm run build`
- Instead of `whereis node`, you MUST execute: `source ~/.zshrc && whereis node`

Do not execute raw commands without this prefix, otherwise it will result in "command not found" errors.

# Project Rule Paths (各工程规则路径规范)

为了确保各子工程的编码规范 (Workspace Rules) 能被正确识别，必须遵循以下统一的路径存放规范。

| 工程名 (Project) | 规则文件路径 (Relative Path) | 说明 |
| :--- | :--- | :--- |
| **ms-java-gateway** | `.agent/rules/ai-code-ws.md` | Spring Cloud Gateway 规范 |
| **ms-java-biz** | `.agent/rules/ai-code-ws.md` | Java RAG & MCP 规范 |
| **ms-py-agent** | `.agent/rules/ai-code-ws.md` | FastAPI & LangGraph 规范 |
| **ms-ng-view** | `.agent/rules/ai-code-ws.md` | Angular & RxJS 规范 |
| **docs** | `.agents/rules/CONTRIBUTING.md` | 全局流程与 RAG 规范 |
| **docs** | `.agents/rules/AGENTS_ENV.md` | 环境与终端执行约束 (本文件) |

**约束 (Constraint)**: 严禁在工程根目录下直接新建 `ai-code-ws.md`，所有 Workspace 级别的经验总结必须追加到上述指定的 `.agent/rules/` 目录下。