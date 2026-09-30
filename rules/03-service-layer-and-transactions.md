# Rule 03: Service Layer & Transaction Management

> **Category:** Business Logic & Transactions  
> **Target:** Spring Framework / Spring Data  
> **Enforcement Level:** Mandatory  

---

## 1. Single Responsibility Principle (SRP) in Services

### Principle
A service must not be a "God Object" doing user authentication, PDF generation, email sending, and order persistence in a single 1,000-line class.
- **Application Services / Orchestrators:** Coordinate workflows, delegate to domain services, and manage transaction boundaries.
- **Domain Services:** Execute business calculations, validations, and domain rules.
- **Infrastructure Services:** Encapsulate third-party APIs (Stripe, Twilio, SendGrid).

---

## 2. Declarative Transaction Management (`@Transactional`)

### Principles
1. **Class-level `readOnly = true`:** Mark query services or the service class default with `@Transactional(readOnly = true)`. This allows Hibernate to skip dirty checking and allows database drivers to route queries to read replicas.
2. **Method-level Write Transactions:** Annotate mutating methods explicitly with `@Transactional`.
3. **Explicit `rollbackFor`:** Spring defaults to rolling back only on `RuntimeException` and `Error`. Checked exceptions do not trigger rollbacks unless specified: `@Transactional(rollbackFor = Exception.class)`.

#### ❌ Bad: Missing readOnly and swallowing exceptions
```java
@Service
public class OrderService {

    // Bad: Dirty checking runs on all entities loaded in read queries; holds connection unnecessarily
    public OrderResponse getOrder(UUID id) {
        Order order = orderRepository.findById(id).orElseThrow();
        return orderMapper.toResponse(order);
    }

    // Catastrophic: Checked exception does not trigger database rollback!
    @Transactional
    public void processPayment(UUID orderId) throws PaymentFailedException {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(OrderStatus.PAID);
        orderRepository.save(order);

        if (someExternalCheckFailed()) {
            throw new PaymentFailedException("Payment rejected"); // DB change is committed anyway!
        }
    }
}
```

#### ✅ Good: Optimized transaction boundaries
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // Default all methods to read-only optimization
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;

    public OrderResponse getOrder(UUID id) {
        return orderRepository.findById(id)
                .map(orderMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Order not found: " + id));
    }

    @Transactional(rollbackFor = Exception.class) // Explicit write transaction with robust rollback
    public OrderResponse createOrder(CreateOrderRequest request) {
        Order order = orderMapper.toEntity(request);
        order.setStatus(OrderStatus.PENDING);
        Order savedOrder = orderRepository.save(order);
        return orderMapper.toResponse(savedOrder);
    }
}
```

---

## 3. The Self-Invocation Anti-Pattern (AOP Proxy Bypass)

### Principle
Spring `@Transactional` works via dynamic proxies. If method `A()` in class `OrderService` calls method `B()` in the **same class** directly via `this.B()`, the call bypasses the Spring AOP proxy. **Method `B()` will NOT run in a transaction!**

#### ❌ Bad: Self-invocation bypassing transactional proxy
```java
@Service
public class OrderService {

    public void processBatch(List<OrderRequest> requests) {
        for (OrderRequest req : requests) {
            // BUG: Calls 'this.createOrder()', bypassing Spring AOP proxy!
            // No transaction is ever started for createOrder!
            createOrder(req); 
        }
    }

    @Transactional
    public void createOrder(OrderRequest req) {
        // ...
    }
}
```

#### ✅ Good: Extracting to a dedicated collaborator or using programmatic template
```java
// Extract transaction boundary to a separate collaborator bean
@Service
@RequiredArgsConstructor
public class OrderProcessor {

    private final OrderCreationService orderCreationService;

    public void processBatch(List<OrderRequest> requests) {
        for (OrderRequest req : requests) {
            // Properly routed through Spring proxy -> Transaction is active!
            orderCreationService.createOrder(req);
        }
    }
}
```

---

## 4. Never Perform Long Network I/O Inside `@Transactional`

### Principle
Database connection pool threads are a finite, scarce resource (HikariCP defaults to 10 connections). If you call external payment gateways (PayPal, Stripe), email servers (SMTP), or third-party HTTP endpoints inside `@Transactional`, the database connection remains locked for seconds while waiting for network I/O, leading to connection pool exhaustion and complete system outage.

#### ❌ Bad: Holding DB connection during external network latency
```java
@Transactional
public void completeCheckout(CheckoutRequest request) {
    Order order = orderRepository.findById(request.orderId()).orElseThrow();
    
    // FATAL: DB connection is held open during 2-5 second network HTTP call
    PaymentResult result = paymentGatewayClient.charge(request.paymentToken(), order.getTotal());
    
    order.setPaymentReference(result.transactionId());
    order.setStatus(OrderStatus.PAID);
    orderRepository.save(order);
}
```

#### ✅ Good: Isolate external calls outside database transactions
```java
@Service
@RequiredArgsConstructor
public class CheckoutService {

    private final OrderQueryService orderQueryService;
    private final PaymentGatewayClient paymentGatewayClient;
    private final OrderCommandService orderCommandService;

    public CheckoutResponse processCheckout(CheckoutRequest request) {
        // 1. Read-only transaction (releases DB connection immediately)
        OrderDTO order = orderQueryService.getOrder(request.orderId());

        // 2. External network call (ZERO database connections held)
        PaymentResult payment = paymentGatewayClient.charge(request.paymentToken(), order.total());

        // 3. Fast write transaction (holds DB connection for 2 milliseconds only)
        return orderCommandService.finalizePaidOrder(order.id(), payment.transactionId());
    }
}
```
