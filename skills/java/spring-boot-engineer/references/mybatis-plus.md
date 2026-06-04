# Data Access - MyBatis-Plus

## Core Rule

Use MyBatis-Plus to remove repetitive single-table SQL, not to hide business rules.

| Scenario | Preferred Pattern |
|----------|-------------------|
| CRUD by id | `BaseMapper#selectById`, `insert`, `updateById`, `deleteById` |
| Simple single-table filters | `LambdaQueryWrapper` / `LambdaUpdateWrapper` |
| Simple paging | `BaseMapper#selectPage` with `Page<T>` |
| Count / exists | `selectCount` with a lambda wrapper |
| Complex joins | Mapper interface method + XML SQL |
| Aggregation / reports | XML SQL with dedicated result DTO |
| Dynamic SQL with many branches | XML SQL, not a huge Java wrapper chain |
| Vendor-specific SQL | XML SQL, isolated in the mapper layer |

Do not write XML for trivial `WHERE id = ?`, simple `LIKE`, simple status filters, or normal pagination. Do not force complex joins into wrapper chains. When a query needs `JOIN`, `GROUP BY`, window functions, `UNION`, CTE, or a hand-tuned execution plan, put it in XML.

## Package Layout

Keep MyBatis-Plus details inside the data/service boundary. Controllers should talk to services and DTOs, not mappers or persistence objects.

```text
src/main/java/com/example/user/
  controller/
    UserController.java
  dto/
    UserCreateRequest.java
    UserResponse.java
    UserSearchCriteria.java
    UserOrderSummary.java
  entity/
    UserDO.java
  mapper/
    UserMapper.java
  service/
    UserService.java

src/main/resources/
  mapper/
    UserMapper.xml
```

If the project already uses `Entity`, `PO`, or another suffix, follow the project. In new MyBatis-Plus code, prefer `DO` for database objects so they do not look like API DTOs or domain responses.

## Dependency and Configuration

Use the project-managed MyBatis-Plus version. For MyBatis-Plus `3.5.9+`, add `mybatis-plus-jsqlparser` when using `PaginationInnerInterceptor`.

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
</dependency>

<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-jsqlparser</artifactId>
</dependency>
```

```yaml
mybatis-plus:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.example.user.entity
  configuration:
    map-underscore-to-camel-case: true
  global-config:
    db-config:
      id-type: auto
      logic-delete-field: deleted
      logic-delete-value: 1
      logic-not-delete-value: 0
```

Keep SQL logging in local/test profiles only. Do not enable verbose SQL logs with bind values in production when the query may contain PII.

## MyBatis-Plus Configuration

```java
@Configuration
@MapperScan("com.example.user.mapper")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        var interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        return interceptor;
    }
}
```

Adjust `DbType` to the actual database. Add tenant or dynamic-table interceptors only when the project really needs them.

## Entity Pattern

Entities are mutable persistence objects. Keep API validation and response shaping in DTOs, not in `DO` classes.

```java
@TableName("sys_user")
@Getter
@Setter
@NoArgsConstructor
public class UserDO {

    @TableId(type = IdType.AUTO)
    private Long id;

    private Long tenantId;

    private String email;

    private String username;

    private UserStatus status;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createdAt;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updatedAt;

    @TableLogic
    private Integer deleted;

    @Version
    private Integer version;

    @TableField(exist = false)
    private List<String> roleNames;
}
```

Prefer `@Getter` / `@Setter` over `@Data` for persistence objects. `@Data` generates `equals`, `hashCode`, and `toString`, which can accidentally include large or sensitive fields.

## Auto Fill

Use MyBatis-Plus auto-fill for audit fields, but keep user-specific audit data explicit when it depends on security context.

```java
@Component
public class AuditMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        var now = LocalDateTime.now();
        strictInsertFill(metaObject, "createdAt", LocalDateTime.class, now);
        strictInsertFill(metaObject, "updatedAt", LocalDateTime.class, now);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        strictUpdateFill(metaObject, "updatedAt", LocalDateTime.class, LocalDateTime.now());
    }
}
```

Auto-fill runs when MyBatis-Plus receives an entity. If you call `update(Wrapper<T>)` without an entity, update fill will not run. Prefer `updateById(entity)` or `update(entity, wrapper)` when `updatedAt` must be maintained automatically.

## Mapper Pattern

Simple mappers extend `BaseMapper` and stay empty until a real custom query is needed.

```java
public interface UserMapper extends BaseMapper<UserDO> {

    IPage<UserOrderSummary> selectOrderSummaries(
        Page<UserOrderSummary> page,
        @Param("criteria") UserOrderSearchCriteria criteria
    );
}
```

Use `@MapperScan` at configuration level. Add `@Mapper` only if the project does not use scanner configuration.

## BaseMapper and IService

Both `BaseMapper` and `IService` are MyBatis-Plus built-in capabilities. This guide defaults to explicit constructor injection of mapper interfaces because it matches the service-layer style used by this skill.

If an existing project already extends `ServiceImpl<UserMapper, UserDO>`, keep that convention. Use built-in methods such as `getById`, `list`, `page`, `save`, and `updateById` for simple operations, but do not let service classes become CRUD pass-through wrappers with no business meaning.

Avoid the static `Db` kit in normal service code. It makes dependencies implicit and is harder to test than constructor-injected mappers or services.

## Simple Queries

For simple reads, build lambda wrappers in the service layer. Use method references so refactors fail at compile time instead of becoming broken SQL column strings.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserService {

    private static final long MAX_PAGE_SIZE = 100;

    private final UserMapper userMapper;

    public Optional<UserResponse> findById(Long id) {
        return Optional.ofNullable(userMapper.selectById(id))
            .map(UserResponse::from);
    }

    public IPage<UserResponse> search(UserSearchCriteria criteria) {
        var page = Page.<UserDO>of(
            criteria.pageNo(),
            Math.min(criteria.pageSize(), MAX_PAGE_SIZE)
        );

        var query = Wrappers.lambdaQuery(UserDO.class)
            .eq(UserDO::getTenantId, criteria.tenantId())
            .eq(criteria.status() != null, UserDO::getStatus, criteria.status())
            .like(StringUtils.hasText(criteria.keyword()), UserDO::getUsername, criteria.keyword())
            .orderByDesc(UserDO::getCreatedAt);

        return userMapper.selectPage(page, query)
            .convert(UserResponse::from);
    }

    public boolean existsByEmail(Long tenantId, String email) {
        var query = Wrappers.lambdaQuery(UserDO.class)
            .eq(UserDO::getTenantId, tenantId)
            .eq(UserDO::getEmail, email);

        return userMapper.selectCount(query) > 0;
    }
}
```

Do not return unbounded `List<T>` from public APIs. Use pagination unless the result set is naturally tiny and bounded by business rules.

## Simple Writes

Keep transaction boundaries on services. Check affected rows for updates and deletes that must touch exactly one row.

```java
@Service
@RequiredArgsConstructor
public class UserCommandService {

    private final UserMapper userMapper;

    @Transactional
    public UserResponse create(UserCreateRequest request) {
        var user = new UserDO();
        user.setTenantId(request.tenantId());
        user.setEmail(request.email());
        user.setUsername(request.username());
        user.setStatus(UserStatus.ACTIVE);

        userMapper.insert(user);
        return UserResponse.from(user);
    }

    @Transactional
    public void disable(Long tenantId, Long userId) {
        var update = new UserDO();
        update.setStatus(UserStatus.DISABLED);

        var condition = Wrappers.lambdaUpdate(UserDO.class)
            .eq(UserDO::getTenantId, tenantId)
            .eq(UserDO::getId, userId)
            .eq(UserDO::getStatus, UserStatus.ACTIVE);

        int rows = userMapper.update(update, condition);
        if (rows != 1) {
            throw new UserNotFoundException(userId);
        }
    }
}
```

Avoid `update(null, wrapper)` when audit auto-fill or optimistic locking matters. Avoid `remove` / `delete` operations without a business condition.

## Complex XML Queries

Use XML when the query is a real SQL artifact. Keep the Java interface small and make parameters explicit with `@Param`.

```java
public record UserOrderSearchCriteria(
    Long tenantId,
    String keyword,
    LocalDateTime startAt,
    LocalDateTime endAt,
    OrderStatus orderStatus,
    long pageNo,
    long pageSize
) {}

public record UserOrderSummary(
    Long userId,
    String username,
    String email,
    Long orderCount,
    BigDecimal totalAmount,
    LocalDateTime lastOrderAt
) {}
```

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "https://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.user.mapper.UserMapper">

    <resultMap id="UserOrderSummaryMap" type="com.example.user.dto.UserOrderSummary">
        <constructor>
            <arg column="user_id" javaType="java.lang.Long" />
            <arg column="username" javaType="java.lang.String" />
            <arg column="email" javaType="java.lang.String" />
            <arg column="order_count" javaType="java.lang.Long" />
            <arg column="total_amount" javaType="java.math.BigDecimal" />
            <arg column="last_order_at" javaType="java.time.LocalDateTime" />
        </constructor>
    </resultMap>

    <select id="selectOrderSummaries" resultMap="UserOrderSummaryMap">
        SELECT
            u.id AS user_id,
            u.username,
            u.email,
            COUNT(o.id) AS order_count,
            COALESCE(SUM(o.total_amount), 0) AS total_amount,
            MAX(o.created_at) AS last_order_at
        FROM sys_user u
        INNER JOIN orders o
            ON o.user_id = u.id
            AND o.deleted = 0
        WHERE u.deleted = 0
          AND u.tenant_id = #{criteria.tenantId}
        <if test="criteria.keyword != null and criteria.keyword != ''">
          AND (
              u.username LIKE CONCAT('%', #{criteria.keyword}, '%')
              OR u.email LIKE CONCAT('%', #{criteria.keyword}, '%')
          )
        </if>
        <if test="criteria.startAt != null">
          AND o.created_at &gt;= #{criteria.startAt}
        </if>
        <if test="criteria.endAt != null">
          AND o.created_at &lt; #{criteria.endAt}
        </if>
        <if test="criteria.orderStatus != null">
          AND o.status = #{criteria.orderStatus}
        </if>
        GROUP BY u.id, u.username, u.email
        ORDER BY last_order_at DESC
    </select>

</mapper>
```

Let the pagination interceptor add paging SQL. Do not add manual `LIMIT` / `OFFSET` to a paged XML method unless the interceptor is intentionally not used.

## XML Pagination Usage

```java
@Transactional(readOnly = true)
public IPage<UserOrderSummary> searchOrderSummaries(UserOrderSearchCriteria criteria) {
    var page = Page.<UserOrderSummary>of(
        criteria.pageNo(),
        Math.min(criteria.pageSize(), MAX_PAGE_SIZE)
    );

    return userMapper.selectOrderSummaries(page, criteria);
}
```

If a grouped XML page query has an expensive generated count SQL, define a dedicated count statement or split the count query from the data query.

## DTO Mapping

Use records for request/response DTOs when the project is on Java 16+.

```java
public record UserResponse(
    Long id,
    String email,
    String username,
    UserStatus status,
    LocalDateTime createdAt
) {
    public static UserResponse from(UserDO user) {
        return new UserResponse(
            user.getId(),
            user.getEmail(),
            user.getUsername(),
            user.getStatus(),
            user.getCreatedAt()
        );
    }
}
```

Do not expose `UserDO` from controllers. Persistence objects often contain internal fields such as `tenantId`, `deleted`, `version`, or audit metadata.

## SQL Safety

Use `#{}` for all runtime values.

```xml
WHERE u.email = #{email}
```

Avoid `${}`. It performs string substitution and can become SQL injection. The only acceptable cases are whitelisted identifiers such as sort columns or dynamic table names after strict validation.

```java
private static final Map<String, SFunction<UserDO, ?>> SORT_FIELDS = Map.of(
    "createdAt", UserDO::getCreatedAt,
    "username", UserDO::getUsername
);
```

Avoid `wrapper.last(...)`, `apply(...)`, or raw SQL fragments with user input. If a fragment is unavoidable, keep user input bound with parameters and whitelist all identifiers.

## Transactions

Put `@Transactional` on service methods, not controllers or mappers.

| Operation | Transaction Pattern |
|-----------|---------------------|
| Read-only query | `@Transactional(readOnly = true)` |
| Single insert/update with side effects | `@Transactional` |
| Multi-step write | One service method with one transaction boundary |
| XML report query | `readOnly = true`, no state mutation |

Avoid self-invocation traps. A `@Transactional` method called from another method in the same class will not pass through Spring's proxy.

## Logical Delete and Optimistic Locking

Use `@TableLogic` for soft delete when records must be retained. Remember that XML SQL does not automatically add every business rule you forget to write. In XML, explicitly filter `deleted = 0`.

Use `@Version` for rows that can be concurrently edited. Check the update row count and return a conflict/domain exception when the row was not updated.

```java
@Transactional
public void rename(Long userId, String username, Integer version) {
    var user = new UserDO();
    user.setId(userId);
    user.setUsername(username);
    user.setVersion(version);

    int rows = userMapper.updateById(user);
    if (rows != 1) {
        throw new ConcurrentUpdateException("User was modified: id=" + userId);
    }
}
```

## Performance Practices

- Select only needed columns for list pages. Avoid `SELECT *` in XML.
- Add indexes for equality filters, join keys, and common sort fields.
- Keep large report queries in XML so they can be reviewed with `EXPLAIN`.
- Prefer keyset pagination for very deep pages when the UI allows it.
- Do not use `like '%keyword%'` on large tables without understanding index impact.
- Keep batch writes chunked and transactional. Do not send a huge list in one statement by default.
- Avoid N+1 lookup loops. If a list view needs derived data from another table, write one XML query or fetch in batches.
- Keep result maps explicit for complex DTOs, enum columns, JSON columns, or fields with type handlers.

## Testing

Use mapper tests for XML SQL. Use a real database with Testcontainers when SQL depends on dialect behavior.

```java
@SpringBootTest
@Testcontainers
class UserMapperIT {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.4");

    @Autowired
    private UserMapper userMapper;

    @Test
    void selectOrderSummaries_filtersByTenant() {
        var criteria = new UserOrderSearchCriteria(
            1L,
            null,
            null,
            null,
            null,
            1,
            20
        );

        var page = Page.<UserOrderSummary>of(1, 20);
        var result = userMapper.selectOrderSummaries(page, criteria);

        assertThat(result.getRecords())
            .allSatisfy(row -> assertThat(row.userId()).isNotNull());
    }
}
```

Test at least these cases:

- Simple wrapper query includes tenant / ownership filters.
- XML query works with all optional criteria absent.
- XML query works with each optional criterion present.
- Pagination returns stable ordering.
- Logical delete rows are hidden.
- Update operations check affected row count.
- Optimistic lock conflict is handled.

## Database Migrations

Keep table definitions aligned with entity annotations.

```sql
CREATE TABLE sys_user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id BIGINT NOT NULL,
    email VARCHAR(100) NOT NULL,
    username VARCHAR(100) NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted TINYINT NOT NULL DEFAULT 0,
    version INT NOT NULL DEFAULT 0,
    UNIQUE KEY uk_user_tenant_email (tenant_id, email),
    KEY idx_user_tenant_status_created (tenant_id, status, created_at),
    KEY idx_user_username (username)
);
```

Do not rely on MyBatis-Plus annotations as a replacement for Flyway/Liquibase migrations. The database schema is still the source of truth for production.

## Code Generator

Generated code is a starting point, not final code.

- Rename generated entities to match the project suffix (`DO`, `Entity`, or existing convention).
- Remove generated controllers unless they match the API design.
- Replace field injection with constructor injection.
- Replace broad `@Data` if the project prefers `@Getter` / `@Setter`.
- Move complex generated SQL into reviewed XML files.
- Add DTO mapping instead of returning persistence objects directly.

## MUST DO

| Rule | Correct Pattern |
|------|-----------------|
| Simple single-table query | `BaseMapper` + `LambdaQueryWrapper` |
| Complex join query | Mapper method + XML |
| Runtime values in SQL | `#{param}` |
| Dynamic identifiers | Whitelist before using `${}` |
| API response | Map `DO` to DTO/record |
| Pagination | `Page<T>` with max page size |
| Transactions | Service layer only |
| Audit fields | `FieldFill` + `MetaObjectHandler` |
| Safety plugin | `BlockAttackInnerInterceptor` |
| Custom XML | Mapper test with real DB dialect when needed |

## MUST NOT DO

- Do not write XML for simple CRUD that MP already supports.
- Do not force joins, reports, or aggregation queries into wrapper chains.
- Do not inject mappers into controllers.
- Do not return `DO` objects from REST APIs.
- Do not concatenate SQL with user input.
- Do not pass user input into `${}`, `last(...)`, or raw SQL fragments.
- Do not run unbounded list queries from API endpoints.
- Do not ignore affected row counts on important updates/deletes.
- Do not rely on auto-fill when calling `update(Wrapper<T>)` without an entity.
- Do not add global plugins or generator templates before the project needs them.

## Quick Reference

| API / Annotation | Purpose |
|------------------|---------|
| `BaseMapper<T>` | Built-in CRUD mapper |
| `Wrappers.lambdaQuery` | Type-safe query conditions |
| `Wrappers.lambdaUpdate` | Type-safe update conditions |
| `Page<T>` / `IPage<T>` | MyBatis-Plus pagination |
| `@TableName` | Table mapping |
| `@TableId` | Primary key mapping |
| `@TableField` | Field mapping, fill, type handler, non-table fields |
| `FieldFill` | Insert/update auto-fill strategy |
| `MetaObjectHandler` | Auto-fill implementation |
| `@TableLogic` | Logical delete field |
| `@Version` | Optimistic lock field |
| `MybatisPlusInterceptor` | Plugin container |
| `PaginationInnerInterceptor` | Pagination plugin |
| `OptimisticLockerInnerInterceptor` | Optimistic lock plugin |
| `BlockAttackInnerInterceptor` | Blocks full-table update/delete |
