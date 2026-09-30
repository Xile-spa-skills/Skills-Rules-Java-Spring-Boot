# Rule 04: Data Access & JPA / Hibernate Performance

> **Category:** Database & Persistence  
> **Target:** Spring Data JPA / Hibernate 6+  
> **Enforcement Level:** Mandatory  

---

## 1. The Cardinal Rule: All Associations Must Be `FetchType.LAZY`

### Principle
- **`@OneToMany` & `@ManyToMany`** default to `LAZY`. Keep them that way.
- **`@ManyToOne` & `@OneToOne`** default to `FetchType.EAGER` in JPA specification. **This is a massive performance trap.** It causes Hibernate to execute eager joins or secondary select statements every time the parent entity is queried.

**Mandate:** Explicitly declare `fetch = FetchType.LAZY` on **every** `@ManyToOne` and `@OneToOne` association without exception.

#### ❌ Bad: Default `@ManyToOne` causing eager queries
```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    private UUID id;

    // Catastrophic: Defaults to FetchType.EAGER! Loading an Order always loads Customer
    @ManyToOne 
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

#### ✅ Good: Explicit `FetchType.LAZY`
```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;
}
```

---

## 2. Eliminating the N+1 Query Problem

### Principle
When querying an entity with associated collections or lazily-loaded relations in a loop, Hibernate issues 1 query for the parent list, and N additional queries for each child. For 1,000 orders, this executes 1,001 SQL queries.

Use **`@EntityGraph`** or **`JOIN FETCH`** to load the required associations in a single query.

#### ❌ Bad: Triggering N+1 queries in loop
```java
// Repository
public interface OrderRepository extends JpaRepository<Order, UUID> {}

// Service
@Transactional(readOnly = true)
public List<OrderSummaryResponse> getAllSummaries() {
    List<Order> orders = orderRepository.findAll(); // 1 query
    
    // N queries fired here when accessing lazy associations!
    return orders.stream()
            .map(o -> new OrderSummaryResponse(o.getId(), o.getCustomer().getName(), o.getItems().size()))
            .toList();
}
```

#### ✅ Good: Single query via `@EntityGraph` or `JOIN FETCH`
```java
public interface OrderRepository extends JpaRepository<Order, UUID> {

    // Option A: Declarative EntityGraph (Generates single SQL LEFT JOIN)
    @EntityGraph(attributePaths = {"customer", "items"})
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findAllByStatusWithDetails(@Param("status") OrderStatus status);

    // Option B: Explicit JPQL JOIN FETCH
    @Query("SELECT o FROM Order o JOIN FETCH o.customer c WHERE o.createdAt >= :since")
    List<Order> findAllSinceWithCustomer(@Param("since") Instant since);
}
```

---

## 3. High-Performance Projections with Java Records

### Principle
If you only need 3 fields from a table with 30 columns for a read-only screen, do not fetch the full managed `@Entity`. Managed entities consume heap memory in the Hibernate Persistence Context (1st level cache) and dirty-checking tracking.

Use **Java Record Constructor Expressions** in JPQL.

#### ❌ Bad: Loading entire managed entity graph for 2 fields
```java
@Query("SELECT u FROM User u WHERE u.active = true")
List<User> findActiveUsers(); // Loads 40 columns + attaches all entities to persistence context
```

#### ✅ Good: Record projection (Bypasses Hibernate entity lifecycle)
```java
public record UserSummaryDTO(UUID id, String email, String fullName) {}

public interface UserRepository extends JpaRepository<User, UUID> {

    @Query("""
        SELECT new com.enterprise.app.features.user.api.UserSummaryDTO(u.id, u.email, u.fullName)
        FROM User u
        WHERE u.active = true
    """)
    List<UserSummaryDTO> findActiveUserSummaries();
}
```

---

## 4. Mandatory Pagination: Never Return Unbounded Collections

### Principle
Never expose an API endpoint that executes `findAll()` without pagination. As data grows, this will trigger `OutOfMemoryError` (OOM) and crash production JVMs.

- Use **`Page<T>`** when the client requires the total page count and total items (e.g. desktop data grid).
- Use **`Slice<T>`** when building infinite scroll or mobile feeds (avoids the expensive `SELECT count(*)` SQL query).

#### ✅ Good: Paginated queries with Pageable
```java
@GetMapping
public ResponseEntity<Page<OrderResponse>> getOrders(
        @PageableDefault(size = 20, sort = "createdAt", direction = Sort.Direction.DESC) Pageable pageable) {
    return ResponseEntity.ok(orderService.getOrders(pageable));
}
```
