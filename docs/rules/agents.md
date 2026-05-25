# Agent Team 编排规则

> Claude Code 端使用官方 **Agent Teams** 能力：主会话是 Team Lead，spawn 多个独立 Claude Code teammates，并可复用 `.claude/agents/<name>.md` 的 project-level agent 定义。`.claude/settings.json` 默认写入 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`。

## 核心原则

1. **Team first**：非平凡任务先判断是否需要 agent team，再执行。不要把复杂需求压成一个万能 subagent。
2. **Main session leads**：主会话拥有需求、计划、状态、验收和 `.orchestration/` 产物；teammate 拥有各自专业判断和任务输出。
3. **Specialist owns specialist work**：代码实现、TDD、审查、安全、E2E、构建修复、文档更新分别交给对应 agent。
4. **Parallel where independent**：探索、审查、测试、文档核对等无依赖任务并行派遣；实施链路按依赖串行。
5. **Lead controls membership**：teammate 不创建嵌套 team、不私自扩队。需要新增成员时，由 Team Lead 追加 spawn 并合并结果。
6. **Evidence over vibes**：每个 teammate 返回文件、命令、结果、风险和后续建议；主会话最终汇报必须包含验证证据。

## Agent Team vs Subagent

| 机制 | 适合场景 | 选择规则 |
|------|----------|----------|
| Agent Team | 多个 teammate 需要并行探索、互相分享发现、协同拆任务 | 跨层功能、复杂 review、根因不明 debug、新模块 |
| Subagent | 只需要一个 focused worker 返回摘要 | 单点审查、一次性搜索、小范围修复 |

Agent Teams 需要 Claude Code v2.1.32 或更新版本。可用 `claude --version` 检查。

如果 Claude Code 版本低于 agent teams 要求，或用户拒绝实验功能，按同一团队拓扑回退为普通 subagent 派遣，并在汇报中说明。

## 可用代理

Claude Code 实际启用位置：`.claude/agents/<name>.md`。`aigroup init` 按 manifest 模块从 `agents/` 源池安装。创建 agent team 时，使用这些名字作为 teammate 的 agent type。

默认 `agents-core`：

| Agent | 用途 | 何时使用 |
|-------|------|----------|
| `planner` | 实施规划 | 复杂功能、重构、跨模块任务 |
| `architect` | 系统设计 | 架构决策、边界调整、关键技术选择 |
| `tdd-guide` | TDD 实施 | 新功能、bugfix、需要测试先行的改动 |
| `code-reviewer` | 代码审查 | 所有代码变更后，必须使用 |
| `security-reviewer` | 安全分析 | auth、支付、PII、权限、加密、外部输入 |
| `build-error-resolver` | 构建修复 | build、typecheck、依赖、配置失败 |
| `e2e-runner` | 端到端验证 | 关键用户流程、回归风险、UI/API 集成 |
| `refactor-cleaner` | 清理重构 | 死代码、重复逻辑、结构整理 |
| `doc-updater` | 文档更新 | README、API 文档、变更说明同步 |
| `rust-reviewer` | Rust 审查 | Rust 代码质量、安全、性能 |
| `init-architect` | 项目初始化 | `/init-project` 生成项目 AI 上下文 |
| `get-current-datetime` | 时间工具 | 需要可靠 UTC 时间戳 |

可选模块会安装语言、质量、运维、领域专项 agent。Team Lead 只能 spawn 已安装 agent type；未安装时说明缺口，选择最接近的已安装 agent 或提示用户运行 `aig update` 增补模块。

## 触发规则

使用 aiGroup 框架处理任务时，以下场景默认创建 Agent Team：

| 场景 | 默认团队 | 备注 |
|------|----------|------|
| 复杂功能 / 跨模块需求 | `planner` → `tdd-guide` → `code-reviewer` | 有架构不确定性时在最前加 `architect` |
| 架构变更 / 重大重构 | `architect` + `planner` → 实施 agent → `code-reviewer` | `architect` 与 `planner` 可先并行收集视角，再由主会话合并 |
| Bug 修复 | `tdd-guide` → `code-reviewer` | 根因不明时先派探索/调试类 agent 或使用 debugging skill |
| 构建 / 类型失败 | `build-error-resolver` → `code-reviewer` | 目标是让一个失败命令变绿，不顺手重构 |
| 安全敏感变更 | `code-reviewer` + `security-reviewer` | auth、支付、PII、权限边界必须并行审 |
| 关键用户流程 | `tdd-guide` + `e2e-runner` → `code-reviewer` | UI/API/状态流组合风险高时使用 |
| 文档或契约更新 | `doc-updater` + 相关 reviewer | 文档必须和代码事实一致 |

轻量任务可以使用最小团队，或在 agent team 不划算时回退为 subagent：

- 纯问答、笔误、单行配置：主会话直接处理。
- 只要计划：`planner` 单人团队，若发现架构/安全/测试风险再扩队。
- 只要审查：`code-reviewer` 单人团队，敏感路径扩为 review team。

## Team 拓扑

### Feature Team

适用：新功能、较大 bugfix、跨层改动。

1. `planner` 输出任务拆解、依赖、风险和验收点。
2. `architect` 在架构不确定时给边界和方案约束。
3. `tdd-guide` 按 Red → Green → Refactor 实施并附测试证据。
4. `code-reviewer` 做 Stage 1 规格符合性 + Stage 2 代码质量审查。
5. `security-reviewer`、`e2e-runner` 按风险加入。

### Review Team

适用：PR、git diff、高风险改动。

并行派遣：

- `code-reviewer`：正确性、维护性、测试缺口。
- `security-reviewer`：安全和隐私风险。
- `e2e-runner`：关键流程是否可执行。
- 语言专项 reviewer：语言/框架细节。

主会话按严重度合并发现，去重冲突结论，并向用户输出可操作清单。

### Build Recovery Team

适用：build/typecheck/test infra/依赖配置失败。

1. 主会话先获取完整失败命令和输出。
2. `build-error-resolver` 只修导致该命令失败的最小问题。
3. 主会话重跑同一命令。
4. `code-reviewer` 审查最小 diff 和副作用。

### Documentation Team

适用：README、API、架构文档、迁移说明。

1. `doc-updater` 根据代码事实更新文档。
2. 需要事实校验时，派相关 reviewer 或 explorer 并行核对。
3. 主会话确认文档没有承诺不存在的行为。

## Spawn 合同

每次 spawn teammate 时，prompt 必须包含：

```markdown
Teammate: <stable name>
Agent type: <agent name from .claude/agents/>
Objective: <本次明确目标>
Context:
- 用户请求：...
- 当前阶段 / session：...
- 相关文件：...
- 已知约束：...
Do:
- ...
Do not:
- ...
Expected output:
- Summary
- Files inspected / changed
- Validation commands and results
- Risks / blockers
- Follow-ups
```

实现类 teammate 还必须说明：

- 可修改范围和禁止修改范围。
- 测试命令或验证口径。
- 是否允许新增依赖。
- 是否需要遵守 TDD。

审查类 teammate 还必须说明：

- 审查范围：路径、diff、commit 区间或 session 产物。
- Stage 1 基准：PRD、计划、需求、架构约束。
- 输出只列确定问题，按严重度排序。

## 编排流程

1. **判断规模**：轻量直接处理；需要两个以上专业视角时建 team；需要证据链时用 `/workflow-start` 建 session。
2. **宣布团队**：告诉用户将创建 agent team、有哪些 teammate、为什么、哪些并行、哪些串行。
3. **创建并 spawn**：创建 agent team，按 agent type spawn teammates；独立任务并行；有依赖任务等待前置 handoff。
4. **写入状态**：工作流 session 中，每个 worker 使用 `.orchestration/<session>/<worker>/` 三件套。
5. **合并结论**：主会话解决冲突、去重发现、明确风险。
6. **验证收口**：运行必要检查，或说明未能运行的原因。
7. **清理 team**：任务完成后由 Team Lead 清理 agent team，避免残留会话。

## 反模式

- 主会话直接实现复杂代码，绕过 `tdd-guide` / 语言专项 agent。
- 所有问题都派给一个“全能” agent。
- 先实现再补计划，或先合并再审查。
- 安全敏感改动只做普通 code review。
- 派遣 prompt 只有一句话，缺少文件、约束和验收标准。
- teammate 创建嵌套 team 或私自扩队，导致 Team Lead 失去状态控制。
- agent 输出没有命令、文件、证据，主会话仍然宣称完成。
