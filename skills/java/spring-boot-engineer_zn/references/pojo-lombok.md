# OOP 与 POJO

## OOP 与 POJO 通用规则

- 所有覆写方法必须添加 `@Override`。
- 禁止使用过时类或过时方法。
- DO / DTO / VO 等 POJO 字段必须使用包装类型，局部变量可使用基本类型。
- POJO 类必须提供可读的 `toString`，避免遗漏关键字段；优先使用 Lombok `@ToString` 生成。
- POJO 布尔字段不要以 `is` 开头，避免序列化和反序列化歧义。
- 构造方法中禁止写业务逻辑；初始化逻辑放到明确的初始化方法或工厂方法。
- `BigDecimal` 禁止使用 `new BigDecimal(double)`，应使用字符串或 `BigDecimal.valueOf`。
- 循环体内字符串拼接优先使用 `StringBuilder`。
- 访问控制从严，能 `private` 不 `protected`，能包内可见不 `public`。

## Lombok 使用规则

- 创建 DO / DTO / VO / BO / Query / Command 等数据承载对象时，优先使用 Lombok 注解生成 getter、setter、构造器、`toString`、Builder 等样板代码。
- 常用组合：
  - 可变 DTO / VO：`@Getter`、`@Setter`、`@ToString`、`@NoArgsConstructor`
  - 需要全参构造：增加 `@AllArgsConstructor`
  - 字段较多或可选参数较多：增加 `@Builder`，创建对象时优先使用 `Xxx.builder().field(value).build()`
  - 继承结构下的 Builder：使用 `@SuperBuilder`
  - 不可变值对象：优先使用 `@Value` 或 `@Getter` + `final` 字段，按项目约定选择
- Jackson、MyBatis、JPA 等框架需要反射创建对象时，应保留无参构造器，例如 `@NoArgsConstructor`。
- DTO 字段不要为了 Lombok `@Builder.Default` 设置默认值；确有业务默认值时，应在业务层、工厂方法或明确的初始化流程中处理。
- 谨慎使用 `@Data`：纯 DTO / VO 可按项目约定使用；涉及继承、实体、缓存 key、集合元素去重或需要精确控制 `equals` / `hashCode` 的类，应改用更明确的 `@Getter`、`@Setter`、`@ToString`、`@EqualsAndHashCode`。

推荐写法：

```java
@Getter
@Setter
@ToString
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UserDTO {

    private Long id;

    private String userName;
}

UserDTO user = UserDTO.builder()
    .id(userId)
    .userName(userName)
    .build();
```
