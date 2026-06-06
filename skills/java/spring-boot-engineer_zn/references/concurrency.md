# 并发规则

- 线程资源必须通过线程池提供，不允许随意 `new Thread`。
- 线程池不使用 `Executors` 快捷方法，应通过 `ThreadPoolExecutor` 明确核心参数。
- 线程和线程池必须有可识别的业务名称。
- `ThreadLocal` 使用后必须在 `finally` 中清理。
- 多资源加锁必须保持一致顺序，避免死锁。
- 并发更新同一记录必须考虑锁、版本号、幂等或唯一约束。
- `SimpleDateFormat` 不得作为共享静态变量；优先使用 `java.time`。
