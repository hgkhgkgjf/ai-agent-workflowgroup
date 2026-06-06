---
name: spring-boot-engineer_zn
description: 中文版 Java / Spring Boot 工程规约与代码 Review 标准。覆盖 Jakarta Validation 参数校验、空安全工具类、常量与枚举、Lombok/POJO、异常与日志、安全、MySQL/ORM、并发、单元测试、工程分层与 Review 清单。当用中文进行 Java/Spring Boot 代码生成、改造或 Review 时加载。
license: MIT
metadata:
  version: "1.0.0"
  domain: backend
  triggers: Java 代码规约, Spring Boot 规约, Jakarta Validation, Lombok, MyBatis, 阿里巴巴 Java 开发手册, 代码 Review, 工程分层
  role: specialist
  scope: standards
  output-format: code
  related-skills: spring-boot-engineer, java-architect
  language: zh-CN
---

# Spring Boot 工程规约（中文）

## 适用范围

本 skill 以《Java 开发手册》为基准，适用于 Java / Spring / Spring Boot 项目的代码生成、修改和 Review，强制遵守命名、格式、异常日志、安全、数据库、分层与设计规约。

加载时机：当用中文进行 Java / Spring Boot 代码生成、改造或 Review，且需要超出纯编码风格（见 `docs/rules/java_zn/coding-style.md`）的工程规约时，加载本 skill。生成代码时优先遵守项目现有约定；项目无明确约定时按本 skill 执行。

## 参考指南

按上下文加载对应参考文件：

| 主题 | 参考文件 | 何时加载 |
|------|----------|----------|
| DTO 与参数校验 | `references/validation.md` | Jakarta Validation 注解、`@Valid`/`@Validated`、跨字段与业务校验 |
| OOP / POJO / Lombok | `references/pojo-lombok.md` | POJO 设计、Lombok 注解组合、Builder、`@Data` 取舍 |
| 异常与日志 | `references/exception-logging.md` | 异常处理完整规则、SLF4J 日志、脱敏 |
| 安全 | `references/security.md` | 参数校验、权限、脱敏、SQL 注入、CSRF、幂等防刷 |
| MySQL 与 ORM | `references/persistence.md` | 表结构规约、MyBatis、分页、事务 |
| 并发 | `references/concurrency.md` | 线程池、`ThreadLocal`、锁、并发更新 |
| 单元测试 | `references/testing.md` | 测试编写、覆盖范围、独立可重复 |
| 工程结构与设计 | `references/architecture.md` | 重复代码抽象、分层依赖、单一职责、设计原则 |

## 核心规约

以下为高频内联规约，完整迁移自原规则。

### 值判断与工具类使用

- 字符串、包装类型、对象、集合、数组等值判断，应优先使用空安全工具类或 JDK 工具方法，避免直接使用 `==` / `!=` 做相等判断。
- 字符串空白判断优先使用 `org.apache.commons.lang3.StringUtils.isBlank` / `isNotBlank`，或项目已有的 `org.springframework.util.StringUtils.hasText`。
- 字符串相等优先使用 `org.apache.commons.lang3.StringUtils.equals` / `equalsIgnoreCase`。
- 对象相等优先使用 `java.util.Objects.equals`。
- Boolean 包装类型判断优先使用 `org.apache.commons.lang3.BooleanUtils.isTrue` / `isFalse`。
- 集合或 Map 判空优先使用 `org.springframework.util.CollectionUtils.isEmpty`。
- 数组、对象整体判空可使用 `org.springframework.util.ObjectUtils.isEmpty`。
- 枚举相等在明确非空语义下可以使用 `EnumConstant == value`；如果枚举变量可能为 null，优先使用 `Objects.equals(value, EnumConstant)` 或将常量放左侧。
- 基本类型大小比较仍使用 `>`、`<`、`>=`、`<=`。

禁止写法：

```java
if (userName == "") {
    return;
}

if (status == 1) {
    return;
}

if (enabled == true) {
    return;
}
```

推荐写法：

```java
if (StringUtils.isBlank(userName)) {
    return;
}

if (Objects.equals(status, ENABLED_STATUS)) {
    return;
}

if (BooleanUtils.isTrue(enabled)) {
    return;
}

if (CollectionUtils.isEmpty(userList)) {
    return;
}
```

### 常量与枚举

- 禁止魔法值直接出现在代码中，应定义为常量或枚举。
- `Long` 字面量使用大写 `L`，例如 `1L`。
- 固定范围的状态、类型、渠道、来源等优先使用枚举。
- 常量按功能归类，不要维护一个“大而全”的常量类。
- 枚举类建议以 `Enum` 结尾，枚举项全大写并使用下划线分隔。

### DTO 与参数校验要点

- DTO 字段校验优先使用 `jakarta.validation` 注解，Controller 入参用 `@Valid` / `@Validated` 触发，嵌套对象与集合元素用 `@Valid` 级联。
- 禁止使用 `javax.validation` 包；Spring Boot 3+ 统一 Jakarta Validation。
- 简单字段校验交给 DTO 注解，不在 Service 中重复写 `if`；跨字段、依赖外部数据、权限幂等等复杂校验才用代码校验。

> 完整规则与示例见 `references/validation.md`。

### Lombok 与 POJO 要点

- 创建 DO / DTO / VO / BO / Query / Command 等数据承载对象时，优先使用 Lombok 注解生成 getter、setter、构造器、`toString`、Builder 等样板代码。
- 字段较多或可选参数较多时优先 `@Builder` + `Xxx.builder().field(value).build()`；框架反射创建对象时保留无参构造器。
- 谨慎使用 `@Data`：涉及继承、实体、缓存 key、集合元素去重或需精确控制 `equals` / `hashCode` 的类，改用更明确的 `@Getter`、`@Setter`、`@ToString`、`@EqualsAndHashCode`。

> 完整规则与示例见 `references/pojo-lombok.md`。

## Review 检查清单

Review Java 代码时至少检查：

- DTO 字段校验是否优先使用 `jakarta.validation` 注解。
- 是否误用了 `javax.validation`。
- 字符串、包装类型、对象、集合判断是否使用空安全工具类。
- 是否存在 `==` 比较字符串、包装类型或业务值对象。
- 数据承载对象是否优先使用 Lombok 注解，字段较多的对象创建是否优先使用 Builder。
- 是否出现魔法值。
- 同类或跨类重复代码是否出现 3 次及以上。
- 是否存在过深嵌套、超长方法或复杂条件表达式。
- 异常是否被吞掉，日志是否重复打印或泄露敏感信息。
- SQL 是否显式列字段、使用参数绑定并命中必要索引。
- 新增核心逻辑是否有单元测试覆盖。
