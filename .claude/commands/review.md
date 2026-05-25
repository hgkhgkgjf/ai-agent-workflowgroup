---
description: 启动轻量 Review Team 审查当前改动，不建 session
argument-hint: [文件 / 范围；省略则审 git diff]
---

# /review

轻量入口：启动 Review Team，输出双阶段审查（Stage 1 规格符合性 + Stage 2 代码质量）。

## 使用

```
/review                 # 审 git diff（HEAD vs working）
/review src/auth/       # 审指定路径
/review HEAD~3..HEAD    # 审某个 commit 区间
```

## 流程

1. 主会话识别审查范围（`$ARGUMENTS` 或 `git diff --name-only HEAD`）
2. 主会话声明 Review Team：
   - 默认：`code-reviewer`
   - auth / 支付 / PII / 权限：并行追加 `security-reviewer`
   - 关键用户流程：并行追加 `e2e-runner`
   - 已安装语言专项 reviewer 且命中对应语言：并行追加该 reviewer
3. 创建 agent team，并行 spawn reviewer teammates using 对应 agent type，传入：
   - 范围内变更文件清单
   - 计划/PRD（如有）作为 Stage 1 基准
   - 关注项、禁止范围、期望输出格式
4. `code-reviewer` 按 `.claude/agents/code-reviewer.md` 的输出格式产出 **Stage 1 + Stage 2** 报告
5. 主会话合并所有 reviewer 结果，按严重度去重输出；若 reviewer 结论冲突，主会话显式裁决或标为开放问题

## 与 `/workflow-start` 的区别

`/review` 是轻量 Review Team——不写 handoff、不进 session。如果是工作流中的 phase 6（测试验证），用 `/workflow-start` 走完整流程，reviewer 产物会写入 `.orchestration/<session>/<worker>/handoff.md`。
