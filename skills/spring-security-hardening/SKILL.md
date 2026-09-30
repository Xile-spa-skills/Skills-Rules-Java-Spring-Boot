---
name: spring-security-hardening
description: >-
  Use this skill to audit and harden Spring Security 6 configurations in Spring Boot 3+ applications.
  Enforces stateless JWT authentication, CORS policies, secure password hashing, and role-based access control (RBAC).
---

# Skill: Spring Security Hardening Workflow

This workflow guides the implementation and verification of enterprise-grade security for modern Spring Boot 3.x REST APIs.

---

## 1. Security Baseline Architecture

A hardened Spring Boot 3 API requires:
1. Modern `SecurityFilterChain` bean using the Lambda DSL.
2. `SessionCreationPolicy.STATELESS` (No JSESSIONID cookies).
3. CSRF disabled (safe for stateless JWT Bearer authorization headers).
4. Strictly defined CORS origins (reject wildcard `*` with credentials).
5. BCrypt password hashing (minimum cost factor 12) or Argon2.
6. Method security enabled via `@EnableMethodSecurity`.

---

## 2. Standard Hardening Template

### Step 1: Security Configuration
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class SecurityConfiguration {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;
    private final AuthenticationEntryPoint unauthorizedHandler;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(AbstractHttpConfigurer::disable)
                .cors(cors -> cors.configurationSource(corsConfigurationSource()))
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        // Public endpoints
                        .requestMatchers(
                                "/api/v1/auth/**",
                                "/v3/api-docs/**",
                                "/swagger-ui/**",
                                "/actuator/health"
                        ).permitAll()
                        // Role-guarded endpoints
                        .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                        // Default deny/authenticate
                        .anyRequest().authenticated()
                )
                .exceptionHandling(ex -> ex.authenticationEntryPoint(unauthorizedHandler))
                .authenticationProvider(authenticationProvider)
                .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://app.enterprise.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE"));
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### Step 2: Method Security Enforcement
Secure service methods or controller endpoints using `@PreAuthorize`:
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public ResponseEntity<Void> deleteUser(@PathVariable UUID id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## 3. Security Audit Checklist
- [ ] No hardcoded secrets in `application.yml` or Java classes.
- [ ] Password encoder uses BCrypt (strength >= 12) or Argon2.
- [ ] Public endpoints are strictly enumerated; wildcard permitAll (`/**`) is prohibited.
- [ ] JWT tokens have short lifespan (< 15-30 minutes) and use secure HMAC-SHA256 or RSA signatures.
