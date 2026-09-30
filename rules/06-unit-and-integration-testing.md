# Rule 06: Unit & Integration Testing Standards

> **Category:** Quality Assurance & Testing  
> **Target:** JUnit 5 | Mockito | Testcontainers | Spring Boot Test  
> **Enforcement Level:** Mandatory  

---

## 1. The Testing Pyramid & Slicing Strategy

A resilient test suite must be partitioned into three distinct tiers:

1. **Unit Tests (Fastest - 90%):** Test domain logic and service classes in isolation using **JUnit 5 + Mockito**. **Never start the Spring Application Context (`@SpringBootTest`) for unit tests.** Execution must take milliseconds.
2. **Web Slice Tests (`@WebMvcTest`):** Test HTTP routing, serialization, validation, and status codes without spinning up the database or business services.
3. **Integration Tests (`@SpringBootTest` + Testcontainers):** Test complete end-to-end database transactions against a **real PostgreSQL/MySQL container**. **Never use in-memory H2 for integration testing** (H2 hides dialect-specific SQL bugs, JSONB errors, and concurrency locks).

---

## 2. Unit Testing Standard: Strict AAA Pattern & Mockito

Every unit test must be divided cleanly into the **AAA (Arrange - Act - Assert)** model with explicit comments or visual separation. Use BDDMockito (`given()`, `willReturn()`, `then()`) for readable intent.

#### ❌ Bad: Monolithic slow test using full Spring Context for simple business logic
```java
@SpringBootTest // Anti-pattern: Starts full server (takes 15-30s) just to test 1 if-statement
class UserServiceTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test
    void testCreateUser() {
        // Messy unpartitioned setup, mutates real database
        User u = new User();
        u.setEmail("test@test.com");
        User saved = userService.createUser(u);
        assert saved.getId() != null;
    }
}
```

#### ✅ Good: Blazing-fast isolated Unit Test with JUnit 5 + Mockito
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private OrderMapper orderMapper;

    @InjectMocks
    private OrderServiceImpl orderService;

    @Test
    @DisplayName("Should create and return order when request is valid")
    void shouldCreateOrderSuccessfully() {
        // 1. Arrange
        CreateOrderRequest request = new CreateOrderRequest(UUID.randomUUID(), BigDecimal.valueOf(150.00));
        Order orderEntity = new Order();
        Order savedEntity = new Order();
        OrderResponse expectedResponse = new OrderResponse(UUID.randomUUID(), OrderStatus.PENDING);

        given(orderMapper.toEntity(request)).willReturn(orderEntity);
        given(orderRepository.save(orderEntity)).willReturn(savedEntity);
        given(orderMapper.toResponse(savedEntity)).willReturn(expectedResponse);

        // 2. Act
        OrderResponse actualResponse = orderService.createOrder(request);

        // 3. Assert
        assertThat(actualResponse).isNotNull();
        assertThat(actualResponse.id()).isEqualTo(expectedResponse.id());
        assertThat(actualResponse.status()).isEqualTo(OrderStatus.PENDING);

        then(orderRepository).should(times(1)).save(orderEntity);
        then(orderMapper).should(times(1)).toResponse(savedEntity);
    }
}
```

---

## 3. Web Slice Testing: `@WebMvcTest`

Test HTTP status codes, request validation, and JSON serialization without bootstrapping the full Spring application context.

#### ✅ Good: WebMvcTest with MockMvc
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private UserService userService;

    @Test
    @DisplayName("POST /api/v1/users returns 400 Bad Request when email is invalid")
    void shouldReturn400WhenEmailIsInvalid() throws Exception {
        // Arrange
        CreateUserRequest invalidRequest = new CreateUserRequest("john", "invalid-email-format", "password123");

        // Act & Assert
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidRequest)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.title").value("Validation Failed"))
                .andExpect(jsonPath("$.invalidFields.email").exists());
        
        then(userService).shouldHaveNoInteractions();
    }
}
```

---

## 4. Integration Testing with Testcontainers

Never rely on in-memory H2 databases for testing JPA repositories. Use real Docker containers via **Testcontainers**.

#### ✅ Good: Testcontainers PostgreSQL Integration Test
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    @ServiceConnection // Spring Boot 3.1+ automatically configures datasource
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("Should find orders by status using real PostgreSQL database")
    void shouldFindOrdersByStatus() {
        // Arrange
        Order order = new Order();
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);

        // Act
        List<Order> results = orderRepository.findByStatus(OrderStatus.PENDING);

        // Assert
        assertThat(results).hasSize(1);
        assertThat(results.get(0).getStatus()).isEqualTo(OrderStatus.PENDING);
    }
}
```
