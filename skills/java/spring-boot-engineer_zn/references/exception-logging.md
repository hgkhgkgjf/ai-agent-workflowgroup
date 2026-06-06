# 异常处理与日志

## 异常处理

- 异常不得用于流程控制。
- 能通过预检查避免的 RuntimeException，不应依赖 catch 兜底。
- catch 后必须处理、转换或记录，不允许空 catch。
- 事务代码中 catch 后如果需要回滚，必须显式处理回滚或继续抛出异常。
- `finally` 中禁止 `return`。
- 流和资源优先使用 `try-with-resources`。
- 对外接口使用统一错误码或统一响应结构；应用内部优先通过明确的业务异常表达失败。
- 方法返回值允许为 null 时，必须通过注释、Optional 或调用约定说明清楚。

## 日志规则

- 使用 SLF4J 等统一日志门面，不直接依赖具体日志实现 API。
- 日志变量拼接使用占位符，不使用字符串拼接。
- `trace` / `debug` / 大量 `info` 日志输出前必须判断日志级别。
- 不要重复打印同一个异常；上层已记录时下层不要重复记录完整堆栈。
- 异常日志必须包含现场信息和异常堆栈。
- 生产环境禁止输出敏感数据，用户敏感信息必须脱敏。

示例：

```java
if (log.isDebugEnabled()) {
    log.debug("Create user request, orgId={}, userName={}", orgId, userName);
}

log.error("Create user failed, orgId={}, userName={}", orgId, userName, exception);
```
