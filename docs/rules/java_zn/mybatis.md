---
paths:
  - "**/*.java"
  - "**/*Mapper.xml"
  - "**/mapper/**/*.xml"
---

# MyBatis 数据访问与分层 Rule

本规则约定 MyBatis(-Plus) 项目的数据访问方式与 Service / Mapper 分层依赖方向。生成或修改数据访问相关代码时遵守本规则；完整 MySQL/ORM 与工程分层规约见结尾「参考」节。

## 1. 分层与依赖方向

- **Mapper 在底层**：只负责持久化与数据查询，不写业务逻辑。
- **Service 在上层**：承载业务编排、结果组装与事务边界。
- **依赖只能自上而下**：`Service → Mapper`；Mapper 不得反向依赖 Service。

```
上层   Service  ──依赖──▶  底层   Mapper（接口 + XML）
```

## 2. 简单查询用 IService，复杂联查走 Mapper XML 原生 SQL

- **简单**（单表 CRUD、简单条件查询、分页）：Service 接口继承 MyBatis-Plus `IService<T>`、实现类继承 `ServiceImpl<Mapper, T>`，**直接使用继承得到的现成接口**（`getById`、`getOne`、`list`、`listByIds`、`save`、`saveBatch`、`updateById`、`removeById`、`page`、`lambdaQuery()` / `lambdaUpdate()` 等），不写 XML、不重复封装。
- **复杂多表联查**（多表 JOIN、聚合、分组、子查询等）用 `IService` / Wrapper 表达不便或可读性差时：在 **Mapper 接口声明方法**，在 **XML 中写原生 SQL** 完成查询，由 Service 调用该 Mapper 方法（`ServiceImpl` 中通过 `baseMapper` 访问）。
- XML 原生 SQL 要求：参数一律用 `#{}` 绑定（禁止 `${}` 拼接用户输入）、显式列出字段（禁止 `select *`）、结果用 `resultMap` 或 DTO 映射、命中必要索引；分页 `count` 为 0 时直接返回。

示例：

```java
// Mapper 接口：仅声明复杂联查方法；简单 CRUD 由 BaseMapper 提供，无需声明
public interface OrderMapper extends BaseMapper<OrderDO> {

    List<OrderDetailDTO> listOrderDetail(@Param("query") OrderQuery query);
}
```

```xml
<!-- OrderMapper.xml：原生 SQL 写复杂联查 -->
<select id="listOrderDetail" resultType="com.example.order.dto.OrderDetailDTO">
    SELECT o.id, o.order_no, u.user_name, p.product_name, oi.quantity
    FROM t_order o
    JOIN t_user u ON u.id = o.user_id
    JOIN t_order_item oi ON oi.order_id = o.id
    JOIN t_product p ON p.id = oi.product_id
    WHERE o.org_id = #{query.orgId}
    <if test="query.status != null">
        AND o.status = #{query.status}
    </if>
</select>
```

```java
// Service 接口：继承 IService，获得现成 CRUD / 简单查询能力
public interface OrderService extends IService<OrderDO> {

    List<OrderDetailDTO> listDetail(OrderQuery query);   // 复杂联查单独声明
}

// 实现类：继承 ServiceImpl 并绑定对应 Mapper
@Service
public class OrderServiceImpl extends ServiceImpl<OrderMapper, OrderDO> implements OrderService {

    // 简单查询直接用继承自 IService 的方法，无需另写：
    //   getById(id) / list(wrapper) / save(entity) / page(page, wrapper) / lambdaQuery()...

    // 复杂联查：调用 Mapper 的原生 SQL 方法（baseMapper 即 OrderMapper）
    @Override
    @Transactional(readOnly = true)
    public List<OrderDetailDTO> listDetail(OrderQuery query) {
        return baseMapper.listOrderDetail(query);
    }
}
```

## 3. 跨 Service 复用：依赖 Service，不要越层调 Mapper

- 其他 Service 需要这块数据或能力时，**依赖这个 Service**（`Service → Service`），由它对外暴露方法。
- **禁止**跨模块直接调用别人的 Mapper 越过其 Service 层——绕过 Service 会丢失业务规则、事务与权限控制。
- Service 之间避免循环依赖；出现循环依赖时，把公共逻辑下沉到更底层的 Service / Manager 或独立组件。

```java
// 其他 Service 依赖 OrderService，而不是直接用 OrderMapper
@Service
public class FulfillmentService {

    private final OrderService orderService;   // 依赖上层暴露的 Service

    public FulfillmentService(OrderService orderService) {
        this.orderService = orderService;
    }

    public void fulfill(OrderQuery query) {
        List<OrderDetailDTO> details = orderService.listDetail(query);
        // ... 履约业务编排，不直接接触 OrderMapper
    }
}
```

## 参考

完整数据访问规约（MySQL/ORM、`#{}` 参数绑定、索引、分页、事务）与工程分层、依赖方向见 skill：`spring-boot-engineer_zn`（`references/persistence.md`、`references/architecture.md`）。
