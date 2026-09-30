# Rule 05: Security & Configuration Standards

> **Category:** Security & Environment Configuration  
> **Target:** Spring Security 6+ | Spring Boot 3.x+  
> **Enforcement Level:** Zero Tolerance  

---

## 1. Modern Spring Security 6: Lambda DSL & SecurityFilterChain

### Principle
Legacy configurations extending `WebSecurityConfigurerAdapter` are completely removed in Spring Boot 3. All security configuration must be defined as a `@Bean` returning `SecurityFilterChain` using the modern fluent **Lambda DSL**.

For stateless REST APIs (JWT-based):
1. Configure `SessionCreationPolicy.STATELESS`.
2. Disable CSRF (safe for stateless bearer token architectures; browser does not attach Authorization headers automatically).
3. Configure explicit CORS policies.
4. Protect endpoints with principle of least privilege.

#### ❌ Bad: Deprecated configurations and unsafe security filter chains
```java
// Banned: WebSecurityConfigurerAdapter does not exist in Spring Boot 3
@Configuration
public class LegacySecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors().and().csrf().disable() // Legacy chaining style
            .authorizeRequests().anyRequest().authenticated();
    }
}
```

#### ✅ Good: Modern Spring Security 6 Lambda DSL
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // Enables @PreAuthorize, @PostAuthorize
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;
    private final CustomAuthenticationEntryPoint authenticationEntryPoint;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(AbstractHttpConfigurer::disable)
                .cors(cors -> cors.configurationSource(corsConfigurationSource()))
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/v1/auth/**", "/v3/api-docs/**", "/swagger-ui/**").permitAll()
                        .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated()
                )
                .exceptionHandling(ex -> ex.authenticationEntryPoint(authenticationEntryPoint))
                .authenticationProvider(authenticationProvider)
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://enterprise.app", "http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## 2. Zero Hardcoded Secrets & Type-Safe Configuration

### Principle
- **Never commit secrets:** Passwords, private keys, API secrets, and database credentials must never exist in git repositories or `application.yml`.
- Inject secrets via environment variables (`${DATABASE_PASSWORD}`) or external vaults.
- Avoid loose `@Value("${app.jwt.secret}")` annotations scattered throughout classes.
- Use validated **`@ConfigurationProperties`** bound to immutable Java Records.

#### ❌ Bad: Hardcoded credentials and scattered `@Value`
```java
@Component
public class JwtService {
    // Dangerous: Secret exposed in source code, easily leaked into version control
    private String secret = "super_secret_key_12345678901234567890";

    @Value("${app.jwt.expiration-hours}") // No validation; fails at runtime if missing
    private int expirationHours;
}
```

#### ✅ Good: Type-safe, validated `@ConfigurationProperties` Record
```java
// 1. Immutable configuration record with Jakarta Validation
@ConfigurationProperties(prefix = "app.security.jwt")
@Validated
public record JwtProperties(
    @NotBlank(message = "JWT secret key must be configured")
    @Size(min = 32, message = "JWT secret must be at least 256 bits (32 characters)")
    String secretKey,

    @NotNull
    @Min(value = 1, message = "Expiration must be at least 1 minute")
    Long expirationMinutes,

    @NotNull
    Long refreshExpirationDays
) {}

// 2. Enable in Spring Boot application
@Configuration
@EnableConfigurationProperties(JwtProperties.class)
public class AppConfig {}

// 3. Inject typed properties cleanly
@Service
@RequiredArgsConstructor
public class JwtService {
    private final JwtProperties jwtProperties;
    
    public long getExpiration() {
        return jwtProperties.expirationMinutes();
    }
}
```

---

## 3. SQL Injection Prevention

### Principle
Never construct SQL or JPQL queries using string concatenation with user-supplied input. Always use Spring Data query methods or named parameter bindings.

#### ❌ Bad: String concatenation in query
```java
@Repository
public class UserRepository {
    @PersistenceContext
    private EntityManager em;

    public List<User> searchUsers(String input) {
        // CATASTROPHIC SQL INJECTION: Input like "' OR '1'='1" dumps whole database
        String sql = "SELECT * FROM users WHERE username = '" + input + "'";
        return em.createNativeQuery(sql, User.class).getResultList();
    }
}
```

#### ✅ Good: Parameterized query
```java
@Repository
public interface UserRepository extends JpaRepository<User, UUID> {

    @Query("SELECT u FROM User u WHERE LOWER(u.username) LIKE LOWER(CONCAT('%', :query, '%'))")
    List<User> searchByUsername(@Param("query") String query);
}
```
