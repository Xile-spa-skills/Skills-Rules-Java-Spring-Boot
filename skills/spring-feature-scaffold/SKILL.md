---
name: spring-feature-scaffold
description: >-
  Use this skill when scaffolding a new business feature module in a Spring Boot 3+ (Java 17/21) project.
  Enforces feature colocation, strict DTO record boundaries, constructor injection, and RESTful controller standards.
---

# Skill: Spring Boot Feature Scaffolding Workflow

This workflow standardizes the creation of vertical slice domain features, ensuring Clean Architecture and zero entity leakage.

---

## 1. Directory Tree Template

When creating a new domain feature (e.g. `product`), scaffold the following package structure under `features/product/`:

```
features/product/
├── api/
│   ├── ProductController.java       # Transport layer
│   ├── CreateProductRequest.java    # Java Record
│   ├── UpdateProductRequest.java    # Java Record
│   └── ProductResponse.java         # Java Record
├── domain/
│   ├── Product.java                 # JPA Entity
│   └── ProductStatus.java           # Enum
├── repository/
│   └── ProductRepository.java       # Spring Data JPA
├── service/
│   ├── ProductService.java          # Interface
│   └── ProductServiceImpl.java      # Implementation (@Transactional)
└── mapper/
    └── ProductMapper.java           # MapStruct or manual mapper
```

---

## 2. Step-by-Step Scaffolding Procedure

### Step 1: Define JPA Entity (`domain/Product.java`)
```java
@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(nullable = false, length = 120)
    private String name;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal price;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 30)
    private ProductStatus status;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private Instant createdAt;
}
```

### Step 2: Define Request and Response Records (`api/`)
```java
public record CreateProductRequest(
    @NotBlank(message = "Product name is required")
    @Size(max = 120)
    String name,

    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be greater than zero")
    BigDecimal price
) {}

public record ProductResponse(
    UUID id,
    String name,
    BigDecimal price,
    ProductStatus status,
    Instant createdAt
) {}
```

### Step 3: Define Repository (`repository/ProductRepository.java`)
```java
@Repository
public interface ProductRepository extends JpaRepository<Product, UUID> {
    boolean existsByNameIgnoreCase(String name);
}
```

### Step 4: Define Service (`service/ProductServiceImpl.java`)
```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public ProductResponse createProduct(CreateProductRequest request) {
        if (productRepository.existsByNameIgnoreCase(request.name())) {
            throw new ConflictException("Product already exists with name: " + request.name());
        }
        Product entity = productMapper.toEntity(request);
        entity.setStatus(ProductStatus.ACTIVE);
        return productMapper.toResponse(productRepository.save(entity));
    }

    @Override
    public ProductResponse getProductById(UUID id) {
        return productRepository.findById(id)
                .map(productMapper::toResponse)
                .orElseThrow(() -> new ResourceNotFoundException("Product not found: " + id));
    }
}
```

### Step 5: Define REST Controller (`api/ProductController.java`)
```java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductService productService;

    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(@Valid @RequestBody CreateProductRequest request) {
        ProductResponse response = productService.createProduct(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}")
                .buildAndExpand(response.id())
                .toUri();
        return ResponseEntity.created(location).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable UUID id) {
        return ResponseEntity.ok(productService.getProductById(id));
    }
}
```

---

## 3. Scaffolding Validation Checklist
- [ ] JPA Entity is never returned from `ProductController`.
- [ ] DTOs are declared as immutable Java `record`s with Jakarta validation annotations.
- [ ] Service has class-level `@Transactional(readOnly = true)` and write methods have `@Transactional(rollbackFor = Exception.class)`.
- [ ] Controller uses constructor injection via `@RequiredArgsConstructor`.
