# DTO 与参数校验

## Jakarta Validation 优先

- DTO 字段非空、长度、范围、格式等校验，必须优先使用 `jakarta.validation` 包下的注解。
- 禁止使用 `javax.validation` 包；Spring Boot 3+ 项目统一使用 Jakarta Validation。
- Controller 入参 DTO 使用 `@Valid` 或 `@Validated` 触发校验。
- 嵌套对象、集合元素需要级联校验时使用 `@Valid`。
- DTO 字段不要设置默认值，避免框架反序列化和业务语义混淆。

常用注解优先级：

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.AssertTrue;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Positive;
import jakarta.validation.constraints.Size;
```

推荐写法：

```java
public class CreateUserDTO {

    @NotBlank(message = "用户名不能为空")
    @Size(max = 64, message = "用户名长度不能超过64")
    private String userName;

    @NotNull(message = "组织ID不能为空")
    @Positive(message = "组织ID必须为正数")
    private Long orgId;

    @Email(message = "邮箱格式不正确")
    private String email;

    @Valid
    @NotEmpty(message = "明细不能为空")
    private List<CreateUserItemDTO> items;
}
```

Controller 示例：

```java
@PostMapping("/users")
public Result<Void> createUser(@Valid @RequestBody CreateUserDTO request) {
    userService.createUser(request);
    return Result.success();
}
```

## 代码校验只作为补充

- 简单字段校验不得在 Service 中重复写 `if` 判断，应交给 DTO 注解。
- 以下场景可以使用代码校验：
  - 跨字段关系，例如 `startTime` 必须早于 `endTime`。
  - 依赖数据库、缓存、外部服务的业务校验。
  - 权限、幂等、防重放、防刷等上下文相关校验。
  - 注解表达成本过高或会显著降低可读性的复杂条件。
- 跨字段规则优先考虑 `@AssertTrue`、自定义约束注解或类级别 Validator；仍不适合时才写业务代码校验。
- 方法参数校验可在类上使用 `@Validated`，并在 public 方法参数上使用 Jakarta Validation 注解。

示例：

```java
@AssertTrue(message = "开始时间必须早于结束时间")
public boolean isTimeRangeValid() {
    if (startTime == null || endTime == null) {
        return true;
    }
    return startTime.isBefore(endTime);
}
```
