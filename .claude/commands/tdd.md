---
description: Delivery Team：TDD Red → Green → Refactor，不建 session
argument-hint: <功能描述>
---

# /tdd

轻量 Delivery Team：Red → Green → Refactor。**不建 session**，由 `tdd-guide` 主导实施，主会话在完成后派审查成员收口。

## 使用

`/tdd $ARGUMENTS`

## 流程

主会话先声明 Delivery Team：

- 默认：`tdd-guide` → `code-reviewer`
- auth / 支付 / PII / 权限：完成后追加 `security-reviewer`
- 关键用户流程：追加 `e2e-runner`

创建 agent team，先 spawn `tdd-guide` teammate using `tdd-guide` agent type，传入：

- 功能描述（`$ARGUMENTS`）
- 项目栈与现有测试约定
- 任务：先写失败测试 → 写最小实现 → 测试变绿 → 必要时重构

`tdd-guide` 内部完成 Red-Green-Refactor 循环，主会话不需要再分阶段拆派。

主会话验证：让 `tdd-guide` 在最终响应中附 **测试全绿的证据**（命令 + 输出），然后派 `code-reviewer` 审查最终 diff。

## 后续审查

代码变更落定后，必须派 `code-reviewer` 做双阶段审查；安全敏感路径必须并行加 `security-reviewer`。

## 与 `/workflow-start` 的区别

`/tdd` 适合**已知功能边界**的小到中型增量。如果还需要架构决策或跨模块协作，先 `/plan` 或 `/workflow-start`。
