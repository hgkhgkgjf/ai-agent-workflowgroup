---
description: 启动轻量 Planning Team 输出实施计划，不建 session
argument-hint: <任务描述>
---

# /plan

轻量入口：启动最小 Planning Team，不创建 orchestration session。适合**已知需求、需要拆解步骤**的中小型任务。

## 使用

`/plan $ARGUMENTS`

## 流程

1. 主会话先声明 Planning Team：
   - 默认：`planner`
   - 架构不确定：追加 `architect`
   - 测试策略复杂：追加 `tdd-guide`
   - auth / 支付 / PII：追加 `security-reviewer`
2. 主会话创建 agent team，spawn teammates using 对应 agent type，并给每个 teammate 传入：
   - 任务描述（`$ARGUMENTS`）
   - 当前 git 状态摘要（`git status` + `git log --oneline -10`）
   - 相关文件、约束、验收标准
   - 期望输出：计划、风险、依赖、验证建议
3. `planner` teammate 按 `.claude/agents/planner.md` 的输出格式产出主计划；其他成员只补充约束和风险
4. 主会话合并团队结论，把计划呈现给用户，等待 CONFIRM 后再实施
5. **不要**自动开始实施——计划是讨论稿，不是执行令

## 与 `/workflow-start` 的区别

| 维度 | `/plan` | `/workflow-start` |
|------|---------|-------------------|
| 是否建 session | 否 | 是 |
| 适用规模 | 最小 Planning Team 即够 | 涉及 ≥2 个 worker 协作并需要证据链 |
| 后续流程 | 用户自决，不强制 phase 链 | 8 phase 心智模型 |
| 产物归属 | 直接呈现给用户 | `.orchestration/<session>/<worker>/handoff.md` |

何时升级到 `/workflow-start`：Planning Team 发现需要多 worker 执行、需要追踪 handoff、或需要跨 phase 验收时，可手动 `node scripts/orchestration/session.cjs init <name>` 升级。
