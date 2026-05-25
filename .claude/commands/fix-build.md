---
description: Build Recovery Team 修复构建/类型/依赖/配置错误，不建 session
argument-hint: [失败命令；省略则用最近一次 build/typecheck]
---

# /fix-build

轻量入口：启动 Build Recovery Team，以最小 diff 让构建变绿。**不做架构调整、不做功能修改、不顺手重构**。

## 使用

```
/fix-build                  # 主会话从最近的失败诊断
/fix-build npm run build    # 指定要修复的命令
/fix-build tsc --noEmit     # typecheck 失败
```

## 流程

1. 主会话获取失败命令的完整输出（首次出问题就跑一次完整命令，捕获 stderr）
2. 主会话声明 Build Recovery Team：
   - 默认：`build-error-resolver` → `code-reviewer`
   - 若错误集中在已安装语言专项栈，可追加对应 language reviewer 做最小修复校验
3. 创建 agent team，spawn `build-error-resolver` teammate using `build-error-resolver` agent type，传入：
   - 失败命令
   - 完整错误输出
   - 项目栈（package.json / pyproject.toml / go.mod）
   - 最小 diff 约束和禁止重构范围
4. build-error-resolver 按 `.claude/agents/build-error-resolver.md` 输出"修复报告"
5. 主会话重跑同一命令并派 `code-reviewer` 审查修复 diff
6. 主会话呈现 diff + 重跑证据（同一命令变绿）

## 边界

- 单一命令变绿即停，不顺手修其他失败
- 不修测试断言（除非断言本身错误且必须报告）
- 不做依赖大版本升级（除非失败信息明确要求）
