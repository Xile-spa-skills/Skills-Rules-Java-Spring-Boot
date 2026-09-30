---
name: spring-test-generator
description: >-
  Use this skill when generating behavior-driven unit and integration tests for Spring Boot applications.
  Enforces JUnit 5, Mockito, strict AAA (Arrange-Act-Assert) pattern, @WebMvcTest, and Testcontainers.
---

# Skill: Spring Boot Test Generator Workflow

This workflow guides the generation of clean, maintainable unit and integration tests adhering to the **Arrange - Act - Assert (AAA)** pattern.

---

## 1. Test Tier Selection Matrix

Before writing a test, select the right tier:
1. **Testing Business Logic in Service / Domain:**
   👉 Pure **JUnit 5 + Mockito** (`@ExtendWith(MockitoExtension.class)`). Do not start Spring Context!
2. **Testing HTTP Controller, Serialization, Validation:**
   👉 **`@WebMvcTest(Controller.class)`** with `MockMvc`. Mock services with `@MockBean`.
3. **Testing JPA Repositories / Real Queries / Transactions:**
   👉 **`@DataJpaTest`** or **`@SpringBootTest` with Testcontainers** (PostgreSQL/MySQL).

---

## 2. Unit Test Template (JUnit 5 + Mockito with AAA Pattern)

```java
@ExtendWith(MockitoExtension.class)
class CustomerServiceTest {

    @Mock
    private CustomerRepository customerRepository;

    @Mock
    private CustomerMapper customerMapper;

    @InjectMocks
    private CustomerServiceImpl customerService;

    @Nested
    @DisplayName("Create Customer Tests")
    class CreateCustomerTests {

        @Test
        @DisplayName("Should successfully create and return customer when email is unique")
        void shouldCreateCustomerWhenEmailIsUnique() {
            // 1. Arrange
            CreateCustomerRequest request = new CreateCustomerRequest("John Doe", "john@example.com");
            Customer customerEntity = new Customer();
            Customer savedCustomer = new Customer();
            CustomerResponse expectedResponse = new CustomerResponse(UUID.randomUUID(), "John Doe", "john@example.com");

            given(customerRepository.existsByEmail("john@example.com")).willReturn(false);
            given(customerMapper.toEntity(request)).willReturn(customerEntity);
            given(customerRepository.save(customerEntity)).willReturn(savedCustomer);
            given(customerMapper.toResponse(savedCustomer)).willReturn(expectedResponse);

            // 2. Act
            CustomerResponse actualResponse = customerService.createCustomer(request);

            // 3. Assert
            assertThat(actualResponse).isNotNull();
            assertThat(actualResponse.email()).isEqualTo("john@example.com");

            then(customerRepository).should(times(1)).existsByEmail("john@example.com");
            then(customerRepository).should(times(1)).save(customerEntity);
        }

        @Test
        @DisplayName("Should throw DuplicateResourceException when email already exists")
        void shouldThrowExceptionWhenEmailExists() {
            // 1. Arrange
            CreateCustomerRequest request = new CreateCustomerRequest("John Doe", "existing@example.com");
            given(customerRepository.existsByEmail("existing@example.com")).willReturn(true);

            // 2. Act & Assert
            assertThatThrownBy(() -> customerService.createCustomer(request))
                    .isInstanceOf(DuplicateResourceException.class)
                    .hasMessageContaining("Customer with email already exists");

            then(customerRepository).should(never()).save(any());
        }
    }
}
```

---

## 3. Controller Web Slice Test Template (`@WebMvcTest`)

```java
@WebMvcTest(CustomerController.class)
class CustomerControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private CustomerService customerService;

    @Test
    @DisplayName("POST /api/v1/customers returns 201 Created when payload is valid")
    void shouldReturn201WhenPayloadIsValid() throws Exception {
        // 1. Arrange
        CreateCustomerRequest request = new CreateCustomerRequest("Jane Doe", "jane@example.com");
        CustomerResponse response = new CustomerResponse(UUID.randomUUID(), "Jane Doe", "jane@example.com");

        given(customerService.createCustomer(request)).willReturn(response);

        // 2. Act & Assert
        mockMvc.perform(post("/api/v1/customers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(header().exists("Location"))
                .andExpect(jsonPath("$.id").value(response.id().toString()))
                .andExpect(jsonPath("$.email").value("jane@example.com"));
    }
}
```

---

## 4. Quality Checklist
- [ ] Test names describe explicit behavior (`should<ExpectedResult>When<Condition>`).
- [ ] Strict separation of Arrange, Act, and Assert steps.
- [ ] No `System.out.println` left in test methods.
- [ ] Assertions use AssertJ (`assertThat`) for readable assertion chains.
