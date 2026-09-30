# Skills & Rules: Java Spring Boot — The Definitive Engineering Standard

> **Standard Version:** 2.0.0  
> **Target Environment:** Java 17/21 LTS | Spring Boot 3.x+ | Spring Security 6 | Spring Data JPA / Hibernate 6  
> **Philosophy:** Clean Architecture, Domain-Driven Design, Zero Entity Leaks, Relentless Performance, Bulletproof Security.

---

## Overview

This repository standardizes enterprise backend engineering for modern Java and Spring Boot applications. Structured under the official AI Agent & Engineering Customization architecture (`.agents`), it decouples policies into three focused domains:

```
Skills & Rules Java Spring Boot/
├── README.md               # Master index & engineering standards hub
├── agents/                 # Specialist AI agent personas & system prompts
│   ├── spring-boot-architect.md
│   └── spring-boot-code-reviewer.md
├── rules/                  # Discrete, enforceable technical rules & constraints
│   ├── 01-architecture-package-structure.md
│   ├── 02-rest-api-controller-design.md
│   ├── 03-service-layer-and-transactions.md
│   ├── 04-data-access-jpa-performance.md
│   ├── 05-security-and-configuration.md
│   └── 06-unit-and-integration-testing.md
└── skills/                 # Procedural, step-by-step operational workflows
    ├── spring-feature-scaffold/
    │   └── SKILL.md
    ├── spring-jpa-query-optimizer/
    │   └── SKILL.md
    ├── spring-security-hardening/
    │   └── SKILL.md
    └── spring-test-generator/
        └── SKILL.md
```

---

## 1. Rules (Technical Constraints & Standards)

Every rule file is backed by authoritative rationales and concrete `❌ Bad` vs `✅ Good` code examples:

| Rule | Title | Primary Focus |
| :--- | :--- | :--- |
| [Rule 01](./rules/01-architecture-package-structure.md) | **Architecture & Package Structure** | Package by feature, DTO boundary isolation (banning entity exposure), Java Records, constructor injection. |
| [Rule 02](./rules/02-rest-api-controller-design.md) | **REST API & Controller Design** | Semantic HTTP statuses, Jakarta Bean Validation (`@Valid`), RFC 7807 `ProblemDetail` via `@RestControllerAdvice`. |
| [Rule 03](./rules/03-service-layer-and-transactions.md) | **Service Layer & Transactions** | SRP, class-level `@Transactional(readOnly = true)`, solving self-invocation proxy bypass, zero network I/O in transactions. |
| [Rule 04](./rules/04-data-access-jpa-performance.md) | **Data Access & JPA Performance** | Mandatory `FetchType.LAZY`, eliminating N+1 via `@EntityGraph`/`JOIN FETCH`, Record constructor projections, pagination. |
| [Rule 05](./rules/05-security-and-configuration.md) | **Security & Configuration** | Spring Security 6 Lambda DSL, stateless JWT, zero hardcoded secrets, validated `@ConfigurationProperties`, SQL injection defense. |
| [Rule 06](./rules/06-unit-and-integration-testing.md) | **Unit & Integration Testing** | Testing pyramid, isolated JUnit 5 + Mockito tests (AAA pattern), `@WebMvcTest` slices, Testcontainers (no H2). |

---

## 2. Skills (Operational Workflows)

Modular runbooks that can be executed on demand by engineers or AI coding agents:

| Skill | Trigger / Use Case | Actionable Protocol |
| :--- | :--- | :--- |
| [spring-feature-scaffold](./skills/spring-feature-scaffold/SKILL.md) | Creating a new business feature | Scaffolds Entity, Request/Response Record, Repository, Service, Controller, and Mapper. |
| [spring-jpa-query-optimizer](./skills/spring-jpa-query-optimizer/SKILL.md) | Diagnosing slow queries or N+1 bugs | 3-step diagnostic checklist to inspect SQL execution, apply entity graphs, or write projections. |
| [spring-security-hardening](./skills/spring-security-hardening/SKILL.md) | Hardening Spring Boot 3 security | Standard template for SecurityFilterChain, stateless JWT filters, CORS policies, and RBAC `@PreAuthorize`. |
| [spring-test-generator](./skills/spring-test-generator/SKILL.md) | Writing unit or integration tests | Enforces AAA structure with BDDMockito, `@WebMvcTest` controller tests, and Testcontainers. |

---

## 3. Agents (Specialist Personas)

Dedicated agent system prompts ready to be deployed as AI pair programmers or automated PR reviewers:

- [spring-boot-architect.md](./agents/spring-boot-architect.md): Staff Backend Engineer guiding system design, DDD boundaries, and architecture decisions.
- [spring-boot-code-reviewer.md](./agents/spring-boot-code-reviewer.md): Automated PR reviewer catching entity leaks, missing lazy fetching, transaction proxy bypasses, and security vulnerabilities.

---

## 4. How to Install & Use in Any Project

### Method A: Workspace Customization (`.agents/`)
Copy the `agents/`, `rules/`, and `skills/` folders directly into the `.agents/` directory at the root of your Spring Boot project:
```bash
my-spring-boot-app/
└── .agents/
    ├── agents/
    ├── rules/
    └── skills/
```

### Method B: Global AI Configuration
Copy into your global assistant config (e.g., `~/.gemini/config/` or IDE configuration) to apply these rules universally across all Spring Boot codebases.
