# Rule 02: REST API & Controller Design

> **Category:** Web API Layer  
> **Target:** Spring Boot 3.x+ / Spring MVC  
> **Enforcement Level:** Mandatory  

---

## 1. RESTful Standards & HTTP Status Codes

### Principle
Controllers are strictly HTTP boundary adapters. They must:
1. Accept and validate HTTP requests.
2. Delegate business logic to services.
3. Return semantic HTTP status codes.

| Operation | HTTP Method | Path Pattern | Success Status |
| :--- | :--- | :--- | :--- |
| Read Collection | `GET` | `/api/v1/orders` | `200 OK` |
| Read Single Item | `GET` | `/api/v1/orders/{id}` | `200 OK` |
| Create Resource | `POST` | `/api/v1/orders` | `201 Created` with `Location` header |
| Replace Resource | `PUT` | `/api/v1/orders/{id}` | `200 OK` or `204 No Content` |
| Partial Update | `PATCH` | `/api/v1/orders/{id}` | `200 OK` |
| Delete Resource | `DELETE` | `/api/v1/orders/{id}` | `204 No Content` |

---

## 2. Input Validation with Jakarta Bean Validation

### Principle
All request DTOs must be validated at the boundary using `@Valid` on controller methods and declarative Jakarta annotations (`@NotBlank`, `@NotNull`, `@Min`, `@Max`, `@Pattern`, `@Size`) on DTO fields. Never write manual `if (dto.getName() == null)` checks inside service methods for basic validation.

#### ❌ Bad: Ad-hoc validation inside controller or service
```java
@PostMapping
public ResponseEntity<?> createProduct(@RequestBody ProductDTO dto) {
    if (dto.name() == null || dto.name().trim().isEmpty()) {
        return ResponseEntity.badRequest().body("Name is required");
    }
    if (dto.price() == null || dto.price().compareTo(BigDecimal.ZERO) <= 0) {
        return ResponseEntity.badRequest().body("Price must be positive");
    }
    return ResponseEntity.ok(productService.create(dto));
}
```

#### ✅ Good: Declarative Bean Validation with `@Valid`
```java
public record CreateProductRequest(
    @NotBlank(message = "Product title must not be blank")
    @Size(max = 120, message = "Product title cannot exceed 120 characters")
    String title,

    @NotNull(message = "Price is mandatory")
    @DecimalMin(value = "0.01", inclusive = true, message = "Price must be greater than zero")
    BigDecimal price,

    @NotNull(message = "Category ID is mandatory")
    UUID categoryId
) {}

@PostMapping
public ResponseEntity<ProductResponse> createProduct(
        @Valid @RequestBody CreateProductRequest request) {
    ProductResponse response = productService.createProduct(request);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(response.id())
            .toUri();
    return ResponseEntity.created(location).body(response);
}
```

---

## 3. Centralized Exception Handling: RFC 7807 ProblemDetail

### Principle
Never write try-catch blocks in controller endpoints just to format HTTP error responses. Use Spring Boot 3's built-in **RFC 7807 (`ProblemDetail`)** standard combined with `@RestControllerAdvice`.

#### ❌ Bad: Try-catch block boilerplate inside controllers
```java
@GetMapping("/{id}")
public ResponseEntity<?> getCustomer(@PathVariable UUID id) {
    try {
        return ResponseEntity.ok(customerService.findById(id));
    } catch (CustomerNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of("error", ex.getMessage()));
    } catch (Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Server error");
    }
}
```

#### ✅ Good: Global `@RestControllerAdvice` producing RFC 7807 ProblemDetail
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleResourceNotFound(ResourceNotFoundException ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        problemDetail.setTitle("Resource Not Found");
        problemDetail.setProperty("timestamp", Instant.now());
        return problemDetail;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationExceptions(MethodArgumentNotValidException ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
                HttpStatus.BAD_REQUEST, "Invalid request payload content");
        problemDetail.setTitle("Validation Failed");
        
        Map<String, String> errors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .collect(Collectors.toMap(
                        FieldError::getField,
                        fieldError -> Optional.ofNullable(fieldError.getDefaultMessage()).orElse("Invalid value"),
                        (existing, replacement) -> existing
                ));

        problemDetail.setProperty("invalidFields", errors);
        problemDetail.setProperty("timestamp", Instant.now());
        return problemDetail;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGenericException(Exception ex) {
        ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected system error occurred");
        problemDetail.setTitle("Internal Server Error");
        problemDetail.setProperty("timestamp", Instant.now());
        return problemDetail;
    }
}
```
