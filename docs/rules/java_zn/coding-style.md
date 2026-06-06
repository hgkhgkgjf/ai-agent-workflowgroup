---
paths:
  - "**/*.java"
---

# Java 代码风格 Rule

本规则以《Java 开发手册》为基准，覆盖 Java / Spring / Spring Boot 项目的**纯编码风格**。生成代码时优先遵守项目现有约定；项目无明确约定时按本规则执行。

完整工程规约（DTO 校验、Lombok、异常与日志、安全、MySQL/ORM、并发、单元测试、工程分层、Review 清单）见结尾「参考」节。

## 1. 总体原则

- 代码应清晰、稳定、可维护，命名表达完整业务含义，不为追求简短牺牲可读性。
- 强制遵守命名、格式、不可变性、错误处理等编码规约。
- 不引入无必要的新依赖；项目已使用 Spring Utils、Apache Commons Lang3、Jakarta Validation 时优先复用。
- 修改代码时同步补充或调整必要的单元测试。

## 2. 命名规则

- 类名使用 `UpperCamelCase`。
- 方法名、参数名、成员变量、局部变量使用 `lowerCamelCase`。
- 常量使用全大写，下划线分隔，如 `MAX_STOCK_COUNT`。
- 包名全小写，使用单数语义，避免拼音、中文、随意缩写。
- 抽象类以 `Abstract` 或 `Base` 开头；异常类以 `Exception` 结尾；测试类以被测类名开头并以 `Test` 结尾。
- 领域模型命名遵守：
  - 数据对象：`xxxDO`
  - 数据传输对象：`xxxDTO`
  - 展示对象：`xxxVO`
  - 业务对象：`xxxBO`
  - 禁止命名为 `xxxPOJO`
- Service / DAO 暴露接口，实现类使用 `Impl` 后缀。
- 获取单个对象用 `get` 前缀，获取集合用 `list` 前缀，统计用 `count` 前缀，新增用 `save` / `insert`，删除用 `remove` / `delete`，修改用 `update`。

## 3. 格式规则

- 使用 4 个空格缩进，禁止 tab。
- 单行代码建议不超过 120 字符，超出需按语义换行。
- `if` / `for` / `while` / `switch` / `do` 与左括号之间必须有空格。
- 二目、三目运算符两侧必须有空格。
- 非空代码块左大括号不换行，右大括号独占一行；`else` 等连续结构紧跟右大括号。
- 所有 `if` / `else` / `for` / `while` / `do` 语句必须使用大括号。
- 单个方法建议不超过 80 行。超过时优先拆分为语义明确的私有方法或独立组件。
- 不同逻辑、不同语义、不同业务代码之间用空行分隔。
- 每个文件只放一个 public 顶层类型；成员顺序建议：常量、字段、构造器、public 方法、protected 方法、private 方法。

## 4. 不可变性

- 值类型优先使用 `record`（Java 16+）。
- 字段默认声明为 `final`，仅在确有需要时才使用可变状态。
- 公共 API 返回防御性拷贝：`List.copyOf()`、`Map.copyOf()`、`Set.copyOf()`。
- Copy-on-write：返回新实例而不是就地修改已有对象。

```java
// 推荐 —— 不可变值类型
public record OrderSummary(Long id, String customerName, BigDecimal total) {}

// 推荐 —— final 字段，无 setter
public class Order {
    private final Long id;
    private final List<LineItem> items;

    public List<LineItem> getItems() {
        return List.copyOf(items);
    }
}
```

## 5. 现代 Java 特性

在能提升清晰度的场景使用现代语言特性：

- **Record** 用于 DTO 和值类型（Java 16+）。
- **Sealed 类**用于封闭类型层次（Java 17+）。
- **`instanceof` 模式匹配** —— 无需显式强转（Java 16+）。
- **文本块**用于多行字符串 —— SQL、JSON 模板（Java 15+）。
- **Switch 表达式**使用箭头语法（Java 14+）。
- **Switch 模式匹配** —— 对 sealed 类型做穷尽处理（Java 21+）。

```java
// instanceof 模式匹配
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
}

// sealed 类型层次
public sealed interface PaymentMethod permits CreditCard, BankTransfer, Wallet {}

// switch 表达式
String label = switch (status) {
    case ACTIVE -> "Active";
    case SUSPENDED -> "Suspended";
    case CLOSED -> "Closed";
};
```

## 6. Optional 使用

- finder 方法可能无结果时返回 `Optional<T>`。
- 使用 `map()`、`flatMap()`、`orElseThrow()`；不要在未 `isPresent()` 判断时裸调 `get()`。
- 不要把 `Optional` 作为字段类型或方法参数类型。

```java
// 推荐
return repository.findById(id)
    .map(ResponseDto::from)
    .orElseThrow(() -> new OrderNotFoundException(id));

// 禁止 —— Optional 作参数
public void process(Optional<String> name) {}
```

## 7. 错误处理（核心）

- 领域错误优先使用非受检异常。
- 自定义领域异常继承 `RuntimeException`。
- 避免宽泛的 `catch (Exception e)`，仅顶层处理器例外。
- 异常消息必须带上下文信息。
- 流和资源使用 `try-with-resources`。
- 禁止空 catch；`finally` 中禁止 `return`。

```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) {
        super("Order not found: id=" + id);
    }
}
```

> 完整异常处理与日志规则见 skill：`spring-boot-engineer_zn` 的 `references/exception-logging.md`。

## 8. 集合与 Streams

集合处理：

- `Set` 中对象必须正确实现 `equals` 和 `hashCode`。
- 不要在 `foreach` 中对集合执行 `add` / `remove`，删除时使用 `Iterator`。
- 集合转数组必须使用 `toArray(T[] array)`。
- `Arrays.asList` 返回的集合不可直接增删。
- `subList` 结果不可强转为 `ArrayList`，且要注意原集合修改带来的影响。
- `Map` 遍历优先使用 `entrySet`。
- 集合初始化时尽量指定容量。
- 调用 `addAll` 前应对入参集合做空判断。

Streams：

- 使用 stream 做转换，管道保持简短（最多 3–4 个操作）。
- 可读时优先使用方法引用：`.map(Order::getTotal)`。
- 避免在 stream 操作中产生副作用。
- 逻辑复杂时优先使用循环，而不是写出晦涩的 stream 管道。

## 9. 控制语句

- 表达异常分支时优先使用卫语句，减少嵌套。
- `if ... else if ... else` 嵌套不要超过 3 层；超过时应拆分方法、使用策略模式、状态模式或表驱动。
- 条件表达式中不要执行复杂逻辑，先赋值给语义明确的变量。
- 不要在条件表达式中插入赋值语句。
- 循环体内尽量避免重复创建对象、重复查询数据库、重复执行正则编译等高成本操作。

## 10. 注释规则

- 类、类属性、类方法按项目要求使用 Javadoc，接口和抽象方法必须说明参数、返回值、异常和语义。
- 注释应解释业务语义、边界条件、设计意图，不要复述代码。
- 修改代码时同步更新注释。
- 不要长期保留无用的注释代码；确需保留时说明原因、负责人和时间。

## 参考

完整 Java / Spring Boot 工程规约（DTO 校验、Lombok、异常与日志、安全、MySQL/ORM、并发、单元测试、工程分层、Review 清单）见 skill：`spring-boot-engineer_zn`。
