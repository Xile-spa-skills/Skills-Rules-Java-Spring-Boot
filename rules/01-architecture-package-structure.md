# Rule 01: Project Architecture & Package Structure

> **Category:** Architecture & Modularity  
> **Target:** Java 17/21 LTS | Spring Boot 3.x+  
> **Enforcement Level:** Mandatory  

---

## 1. Package by Feature (Domain-Driven Colocation)

### Principle
Group classes by business domain capability (Vertical Slices / Feature-based) rather than purely by technical layer (Horizontal Slices). Placing all controllers in one folder, all services in another, and all repositories in a third breaks encapsulation, impairs navigation, and promotes cyclic dependencies.

Shared cross-cutting concerns (common configurations, security, base exception handlers) reside in a dedicated `common` or `shared` module.

### Canonical Package Structure
```
src/main/java/com/enterprise/app/
├── common/                     # Cross-cutting foundational infrastructure
│   ├── config/                 # SecurityConfig, JacksonConfig, OpenApiConfig
│   ├── exception/              # GlobalExceptionHandler, BaseException, ProblemDetail utils
│   └── security/               # JwtAuthenticationFilter, SecurityUtils
└── features/                   # Business domain modules (Vertical slices)
    ├── auth/                   # Authentication & user identity
    └── billing/                # Billing domain
        ├── api/                # REST Controllers, API Request/Response Records
        │   ├── InvoiceController.java
        │   ├── CreateInvoiceRequest.java
        │   └── InvoiceResponse.java
        ├── domain/             # JPA Entities, Enums, Domain Value Objects
        │   ├── Invoice.java
        │   └── InvoiceStatus.java
        ├── repository/         # Spring Data JPA Repositories
        │   └── InvoiceRepository.java
        ├── service/            # Business logic interfaces & implementations
        │   ├── InvoiceService.java
        │   └── InvoiceServiceImpl.java
        └── mapper/             # MapStruct / Manual mapping utilities
            └── InvoiceMapper.java
```

---

## 2. Strict DTO Boundaries (Never Expose JPA Entities)

### Principle
**JPA `@Entity` classes must NEVER cross the Controller API boundary.** 
Exposing entities directly:
1. Leaks internal database schema and relationship graphs to external clients.
2. Triggers unintended `LazyInitializationException` or catastrophic infinite JSON serialization loops (Jackson cyclic references).
3. Exposes the application to **Over-Posting Attacks** (Mass Assignment vulnerability), where malicious clients inject internal fields like `isAdmin` or `balance`.

Always use immutable Java **`record`** types for API Requests and Responses.

#### ❌ Bad: Leaking JPA Entity directly in REST API
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;

    // Catastrophic: Leaks passwordHash, triggers LazyInitializationException, permits mass assignment
    @GetMapping("/{id}")
    public User getUser(@PathVariable UUID id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("User not found"));
    }

    @PostMapping
    public User createUser(@RequestBody User user) {
        return userRepository.save(user); // Security hole: Client can send id, role="ADMIN"
    }
}
```

#### ✅ Good: Immutable Java Records with MapStruct or Constructor Mapping
```java
// 1. Immutable Request DTO
public record CreateUserRequest(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    String username,

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {}

// 2. Immutable Response DTO (No sensitive fields)
public record UserResponse(
    UUID id,
    String username,
    String email,
    UserStatus status,
    Instant createdAt
) {}

// 3. Controller interacting strictly through DTOs
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable UUID id) {
        return ResponseEntity.ok(userService.getUserById(id));
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
        UserResponse response = userService.createUser(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(response.id())
                .toUri();
        return ResponseEntity.created(location).body(response);
    }
}
```

---

## 3. Dependency Injection: Constructor Injection Only

### Principle
Field injection using `@Autowired` on private fields is strictly forbidden:
- It makes unit testing painful (requires Reflection or Spring Context to inject mocks).
- It obscures circular dependencies until runtime.
- It prevents declaring dependencies as `final` (breaking immutability).

Always use **Lombok `@RequiredArgsConstructor`** or explicit constructor injection on `final` fields.

#### ❌ Bad: Field injection with `@Autowired`
```java
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository; // Banned: Cannot be made final, hard to mock in unit tests

    @Autowired
    private PaymentClient paymentClient;
}
```

#### ✅ Good: Constructor injection on `final` fields via Lombok
```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository; // Immutable, fail-fast, trivially testable
    private final PaymentClient paymentClient;
}
```
