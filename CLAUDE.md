# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## Project Overview

aiGroup is a **dual-harness AI team collaboration framework** (Claude Code + Codex). The main session analyzes requirements, assembles an **Agent Team**, and delegates execution to specialist agents. The main session is the team lead/orchestrator; it does not directly write code, design UI, or run tests for non-trivial tasks.

| 目录 | 内容 |
|------|------|
| `agents/` | Agent 源池（开发态分发位置，由 `aigroup init` 选装到 `.claude/agents/`） |
| `skills/` | Skill 源池（按分类分组 `skills/<category>/<name>/SKILL.md`，分类含 workflow/product/frontend/backend/database/java/python/javascript/go-rust-cpp/dotnet/swift/php/ruby/infra/ai-ml/quality；`aigroup init` 安装时扁平化到目标项目 `.claude/skills/<name>/` 以保留原 skill ID） |
| `docs/` | 知识库主源 |
| `manifests/` | 安装清单（agent / skill 分组定义） |
| `.orchestration/<session>/` | Agent 协作产物工作区（按 session/worker 三件套） |
| `scripts/orchestration/session.cjs` | 协调 CLI |
| `scripts/hooks/dispatcher.cjs` | hook dispatcher（手动调用入口） |

## Claude Code Agent Team Protocol

Claude Code 端默认采用 **Agent Team** 模式，而不是临时单个 subagent 调度。`.claude/settings.json` 已启用 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`；Agent Teams 需要 Claude Code v2.1.32+。若当前版本不支持，则按同一团队拓扑回退为普通 subagent 派遣。

1. **先组队再执行**：遇到复杂功能、bugfix、重构、审查、构建失败、跨模块任务时，主会话先声明团队组成、每个 agent 的目标、并行/串行关系。
2. **主会话是 Team Lead**：负责需求澄清、任务拆分、派遣、合并结论、写 `.orchestration/` 产物、向用户汇报；不亲自替代 specialist 做实现/设计/测试/审查。
3. **创建 team 而不是单次调用**：让 Claude 创建 agent team，并 spawn teammates using `<agent-name>` agent type，复用 `.claude/agents/<name>.md` 的 project-level agent 定义。
4. **每个 teammate 都带完整任务包**：目标、上下文、相关文件、约束、验收标准、期望输出格式必须写进 spawn prompt，因为 teammate 不继承主会话完整历史。
5. **收口必须综合验收**：Team Lead 等待 teammates 完成，汇总 handoff，处理冲突，确认验证证据，再给用户最终结论并清理 team。

常用团队形态：

| 场景 | Agent Team |
|------|------------|
| 新功能 / bugfix | `planner` → `tdd-guide` → `code-reviewer`，敏感路径加 `security-reviewer` |
| 架构变更 / 重构 | `architect` + `planner` → 实施 agent → `code-reviewer` |
| 构建 / 类型失败 | `build-error-resolver` → `code-reviewer` |
| 高风险审查 | `code-reviewer` + `security-reviewer` + `e2e-runner` 并行 |
| 文档同步 | `doc-updater`，必要时并行 `code-reviewer` 校验事实 |

完整派遣矩阵见 `docs/rules/agents.md`。

## 知识库地图（按需查阅）

| 需要了解 | 查阅 |
|---------|------|
| 项目架构、目录结构 | `docs/ARCHITECTURE.md` |
| 工作流 phase 心智模型 | `docs/workflow-pipeline.md` |
| 危险信号与阻止行动 | `docs/red-flags.md` |
| 项目实例上下文（/init-project 生成） | `docs/PROJECT_CONTEXT.md` |
