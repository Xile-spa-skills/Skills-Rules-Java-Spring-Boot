---
name: spring-jpa-query-optimizer
description: >-
  Use this skill to audit, diagnose, and optimize Spring Data JPA and Hibernate performance bottlenecks.
  Eliminates N+1 query problems, replaces eager joins with projections, and configures fetch plans.
---

# Skill: Spring JPA Query Optimizer Workflow

This workflow provides a systematic guide to identify and eliminate database performance bottlenecks in Spring Data JPA applications.

---

## 1. Diagnostic Phase

### Step 1: Enable SQL Query Logging
In `application-local.yml`, enable formatting and parameter binding display:
```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        generate_statistics: true
logging:
  level:
    org.hibernate.orm.jdbc.bind: trace
```

### Step 2: Detect Common Anti-Patterns
1. **Repeated `SELECT ... WHERE parent_id = ?` statements:** N+1 Query Problem detected.
2. **Missing `fetch = FetchType.LAZY` on `@ManyToOne` or `@OneToOne`:** Triggers automatic eager loading.
3. **Huge `SELECT *` loading 50 columns for a 2-field dropdown:** Wasted memory and bandwidth.

---

## 2. Optimization Protocol

### Scenario A: Resolving N+1 with `@EntityGraph`
When loading an entity and needing its related child collections or references:
```java
public interface OrderRepository extends JpaRepository<Order, UUID> {

    @EntityGraph(attributePaths = {"customer", "items.product"})
    @Query("SELECT o FROM Order o WHERE o.id = :id")
    Optional<Order> findByIdWithDetails(@Param("id") UUID id);
}
```

### Scenario B: High-Performance Read Projections (Java Record)
When rendering a dashboard or listing, bypass Hibernate entity management completely:
```java
// Immutable projection record
public record OrderListItemDTO(
    UUID orderId,
    String customerName,
    BigDecimal totalAmount,
    Instant orderDate
) {}

public interface OrderRepository extends JpaRepository<Order, UUID> {

    @Query("""
        SELECT new com.enterprise.app.features.billing.api.OrderListItemDTO(
            o.id,
            c.name,
            o.totalAmount,
            o.createdAt
        )
        FROM Order o
        JOIN o.customer c
        WHERE o.status = :status
    """)
    Page<OrderListItemDTO> findSummariesByStatus(@Param("status") OrderStatus status, Pageable pageable);
}
```

### Scenario C: Replacing Count Query with `Slice`
For infinite scroll or pagination where total count is not displayed:
```java
// Avoids 'SELECT count(*) FROM orders'
Slice<Order> findByStatus(OrderStatus status, Pageable pageable);
```

---

## 3. Verification Steps
1. Run the test with SQL logging enabled.
2. Verify that exactly **1 SQL query** is issued.
3. Ensure no lazy initialization exceptions occur after closing the transaction.
