# AGENTS.md — E-commerce Service

> **Audience:** AI coding agents (Devin, Claude Code, etc.)
> **Purpose:** Authoritative scaffold specification for the E-commerce Service. Follow every section exactly. Do not deviate from structure, naming, or workflow unless a story-level spec explicitly overrides a specific item.

---

## 1. Stack

| Technology | Role |
|---|---|
| **Java 21** | Primary language (use virtual threads where applicable) |
| **Spring Boot 3.x** | Application framework, auto-configuration, embedded Tomcat |
| **Spring Security 6.x** | Security filter chain, JWT validation, method-level authorization |
| **Spring Data JPA / Hibernate** | ORM, repository abstraction over PostgreSQL |
| **Spring Data Redis** | Cache abstraction, session/token blacklist, product search caching |
| **Spring Cache (`@Cacheable`)** | Declarative caching backed by Redis |
| **PostgreSQL 16** | Primary relational datastore |
| **Redis 7** | Cache layer and JWT revocation store |
| **Flyway** | Database schema migrations (versioned SQL scripts) |
| **JJWT (io.jsonwebtoken)** | JWT creation, signing (RS256), and validation |
| **MapStruct** | Compile-time DTO ↔ entity mapping |
| **Lombok** | Boilerplate reduction (`@Builder`, `@Data`, etc.) |
| **SpringDoc OpenAPI 3** | Auto-generated API documentation (`/v3/api-docs`, `/swagger-ui`) |
| **Testcontainers** | Real PostgreSQL + Redis containers in integration tests |
| **JUnit 5 + Mockito** | Unit and slice testing |
| **JaCoCo** | Code coverage enforcement (90% minimum) |
| **Maven Wrapper (`mvnw`)** | Reproducible builds — never require a global Maven install |
| **Docker / Docker Compose** | Containerisation and local dev orchestration |
| **GitHub Actions** | CI pipeline |

---

## 2. Project Structure

Create exactly the following layout. Every path is relative to the repository root.

```
ecommerce-service/
├── .github/
│   └── workflows/
│       └── ci.yml                  # GitHub Actions CI pipeline
├── docker/
│   ├── Dockerfile                  # Multi-stage production image
│   └── docker-compose.yml          # Local dev: app + postgres + redis
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/ecommerce/
│   │   │       ├── EcommerceApplication.java          # @SpringBootApplication entry point
│   │   │       ├── config/
│   │   │       │   ├── SecurityConfig.java            # SecurityFilterChain, CORS, CSRF config
│   │   │       │   ├── JwtConfig.java                 # RSA key-pair bean, JJWT builder config
│   │   │       │   ├── RedisConfig.java               # RedisTemplate, CacheManager beans
│   │   │       │   └── OpenApiConfig.java             # SpringDoc customisation, JWT bearer scheme
│   │   │       ├── auth/
│   │   │       │   ├── controller/
│   │   │       │   │   └── AuthController.java        # POST /api/v1/auth/register, /login, /logout
│   │   │       │   ├── dto/
│   │   │       │   │   ├── RegisterRequest.java
│   │   │       │   │   ├── LoginRequest.java
│   │   │       │   │   └── AuthResponse.java          # accessToken, refreshToken, expiresIn
│   │   │       │   ├── service/
│   │   │       │   │   ├── AuthService.java           # Interface
│   │   │       │   │   └── AuthServiceImpl.java       # Registration, login, token refresh, logout
│   │   │       │   ├── security/
│   │   │       │   │   ├── JwtFilter.java             # OncePerRequestFilter — validates Bearer token
│   │   │       │   │   ├── JwtService.java            # Token generation, parsing, revocation (Redis)
│   │   │       │   │   └── UserDetailsServiceImpl.java# Loads UserDetails from DB
│   │   │       │   └── domain/
│   │   │       │       └── User.java                  # @Entity — id, email, passwordHash, roles, timestamps
│   │   │       ├── product/
│   │   │       │   ├── controller/
│   │   │       │   │   └── ProductController.java     # GET /api/v1/products, /{id}, /search
│   │   │       │   ├── dto/
│   │   │       │   │   ├── ProductRequest.java
│   │   │       │   │   ├── ProductResponse.java
│   │   │       │   │   └── ProductSearchRequest.java  # keyword, category, priceRange, pageable
│   │   │       │   ├── service/
│   │   │       │   │   ├── ProductService.java
│   │   │       │   │   └── ProductServiceImpl.java    # CRUD + search; caches results in Redis
│   │   │       │   ├── repository/
│   │   │       │   │   └── ProductRepository.java     # JpaRepository + @Query search methods
│   │   │       │   ├── mapper/
│   │   │       │   │   └── ProductMapper.java         # MapStruct interface
│   │   │       │   └── domain/
│   │   │       │       └── Product.java               # @Entity — id, name, description, price, category, stock
│   │   │       ├── wishlist/
│   │   │       │   ├── controller/
│   │   │       │   │   └── WishlistController.java    # GET|POST|DELETE /api/v1/wishlists/{userId}/items
│   │   │       │   ├── dto/
│   │   │       │   │   ├── WishlistResponse.java
│   │   │       │   │   └── WishlistItemRequest.java   # productId
│   │   │       │   ├── service/
│   │   │       │   │   ├── WishlistService.java
│   │   │       │   │   └── WishlistServiceImpl.java
│   │   │       │   ├── repository/
│   │   │       │   │   ├── WishlistRepository.java
│   │   │       │   │   └── WishlistItemRepository.java
│   │   │       │   ├── mapper/
│   │   │       │   │   └── WishlistMapper.java
│   │   │       │   └── domain/
│   │   │       │       ├── Wishlist.java              # @Entity — id, user (FK), createdAt
│   │   │       │       └── WishlistItem.java          # @Entity — id, wishlist (FK), product (FK), addedAt
│   │   │       ├── common/
│   │   │       │   ├── exception/
│   │   │       │   │   ├── GlobalExceptionHandler.java # @RestControllerAdvice
│   │   │       │   │   ├── ResourceNotFoundException.java
│   │   │       │   │   ├── DuplicateResourceException.java
│   │   │       │   │   └── UnauthorizedException.java
│   │   │       │   ├── dto/
│   │   │       │   │   ├── ApiResponse.java           # Generic wrapper: data, message, timestamp
│   │   │       │   │   ├── ErrorResponse.java         # status, code, message, path, timestamp
│   │   │       │   │   └── PagedResponse.java         # content, page, size, totalElements, totalPages
│   │   │       │   └── validation/
│   │   │       │       └── ValidEmail.java            # Custom constraint annotation example
│   │   │       └── audit/
│   │   │           └── AuditConfig.java               # @EnableJpaAuditing, AuditorAware bean
│   │   └── resources/
│   │       ├── application.yml                        # Base config (references env vars)
│   │       ├── application-local.yml                  # Local dev overrides
│   │       ├── application-test.yml                   # Test profile config
│   │       └── db/
│   │           └── migration/
│   │               ├── V1__create_users_table.sql
│   │               ├── V2__create_products_table.sql
│   │               └── V3__create_wishlists_tables.sql
│   └── test/
│       └── java/
│           └── com/example/ecommerce/
│               ├── auth/
│               │   ├── controller/
│               │   │   └── AuthControllerTest.java    # @WebMvcTest slice
│               │   └── service/
│               │       └── AuthServiceImplTest.java   # Pure unit test, Mockito
│               ├── product/
│               │   ├── controller/
│               │   │   └── ProductControllerTest.java
│               │   └── service/
│               │       └── ProductServiceImplTest.java
│               ├── wishlist/
│               │   ├── controller/
│               │   │   └── WishlistControllerTest.java
│               │   └── service/
│               │       └── WishlistServiceImplTest.java
│               └── integration/
│                   ├── AbstractIntegrationTest.java   # Testcontainers base class
│                   ├── AuthIntegrationTest.java
│                   ├── ProductIntegrationTest.java
│                   └── WishlistIntegrationTest.java
├── tasks.md                                           # Agent-generated task checklist (see §3)
├── .env.example                                       # Documents all required env vars
├── .gitignore
├── mvnw / mvnw.cmd                                    # Maven wrapper scripts
├── .mvn/wrapper/maven-wrapper.properties
└── pom.xml                                            # Single-module Maven POM
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder.

### Step 1 — Read Specifications
- Locate and read every story-level spec file available in the repository before writing any code.
- Extract: entities, endpoints, business rules, validation constraints, and security requirements.

### Step 2 — Create `tasks.md`
- Create `tasks.md` in the repository root **before writing any implementation code**.
- Format it as a Markdown checklist grouped by domain (`auth`, `product`, `wishlist`, `common`, `infra`).
- Each task must be atomic (single class or migration file). Example:

```markdown
## Auth
- [ ] Create `User` entity with Flyway migration V1
- [ ] Implement `JwtService` (generate, validate, revoke)
- [ ] Implement `AuthServiceImpl` (register, login, logout)
- [ ] Write `AuthServiceImplTest` (unit)
- [ ] Write `AuthIntegrationTest`
...
```

- Check off each item (`- [x]`) immediately after completing it.

### Step 3 — Scaffold Infrastructure First
1. Generate `pom.xml` with all dependencies pinned to exact versions.
2. Create `application.yml`, `application-local.yml`, `application-test.yml`.
3. Write all Flyway SQL migrations (`V1`, `V2`, `V3`).
4. Implement `SecurityConfig`, `JwtConfig`, `RedisConfig`, `OpenApiConfig`.
5. Create `GlobalExceptionHandler` and all common DTOs.

### Step 4 — Implement Domain Modules
Implement in this order to respect dependency direction:
1. `auth` (User entity → repository → service → controller)
2. `product` (Product entity → repository → service → controller)
3. `wishlist` (Wishlist/WishlistItem entities → repositories → service → controller)

For each module, complete the full vertical slice (entity → repo → service → mapper → controller → DTOs) before moving to the next module.

### Step 5 — Write Tests
- Write unit tests immediately after each service implementation.
- Write controller slice tests (`@WebMvcTest`) after each controller.
- Write integration tests last, after all modules are implemented.
- Run `./mvnw test` and confirm all tests pass before proceeding.

### Step 6 — Validate
Run the following commands and fix all failures before marking the task complete:

```bash
# Compile
./mvnw clean compile -q

# All tests + coverage report
./mvnw verify

# Check coverage threshold (enforced by JaCoCo — build fails below 90%)
# Report at: target/site/jacoco/index.html

# Code style (if Checkstyle is configured)
./mvnw checkstyle:check

# Start local stack and smoke-test
docker compose -f docker/docker-compose.yml up -d
./mvnw spring-boot:run -Dspring-boot.run.profiles=local
curl -s http://localhost:8080/actuator/health | grep '"status":"UP"'
```

---

## 4. Coding Conventions

### Naming
| Artifact | Convention | Example |
|---|---|---|
| Packages | `com.example.ecommerce.<domain>.<layer>` | `com.example.ecommerce.auth.service` |
| Entities | PascalCase noun | `User`, `Product`, `WishlistItem` |
| DTOs | PascalCase + suffix | `LoginRequest`, `ProductResponse`, `PagedResponse` |
| Services | Interface + `Impl` | `AuthService` / `AuthServiceImpl` |
| Repositories | `<Entity>Repository` | `ProductRepository` |
| Mappers | `<Entity>Mapper` | `ProductMapper` (MapStruct interface) |
| Controllers | `<Domain>Controller` | `WishlistController` |
| REST endpoints | Kebab-case, versioned | `/api/v1/wishlists`, `/api/v1/auth/login` |
| Constants | `UPPER_SNAKE_CASE` | `TOKEN_PREFIX`, `CACHE_PRODUCTS` |
| Test classes | `<ClassUnderTest>Test` | `AuthServiceImplTest` |

### Architecture Patterns
- **Layered architecture:** Controller → Service (interface) → Repository. Controllers must never call repositories directly.
- **DTO boundary:** Entities must never be returned from controllers. Always map to/from DTOs at the controller layer using MapStruct mappers.
- **Service interfaces:** Every service must have an interface. `@Service` annotation goes on the `Impl` class only.
- **Constructor injection:** Use constructor injection everywhere. Never use `@Autowired` on fields. Lombok `@RequiredArgsConstructor` is acceptable.
- **Immutable DTOs:** Prefer `record` types for request/response DTOs in Java 21. Use Lombok `@Builder` + `@Value` on classes where records are insufficient.
- **Pagination:** All list endpoints must accept `Pageable` and return `PagedResponse<T>`.
- **Validation:** Annotate all request DTOs with Bean Validation (`@NotBlank`, `@Email`, `@Size`, etc.). Controllers must carry `@Valid` on `@RequestBody` parameters.
- **Caching strategy:**
  - Cache product list/search results with key `products::<cacheKey>` and TTL 10 minutes.
  - Evict product cache on create/update/delete.
  - Store JWT revocation entries in Redis with key `jwt:revoked::<jti>` and TTL equal to the token's remaining lifetime.
- **Error handling:** `GlobalExceptionHandler` handles all exceptions. Return `ErrorResponse` with RFC 7807-style fields. Never expose stack traces.
- **Auditing:** All entities must extend or embed `@CreatedDate` / `@LastModifiedDate` via Spring Data JPA auditing.
- **Transactions:** `@Transactional` belongs on the service `Impl` class, not on the interface or controller.

### Style
- Java 21 — use records, pattern matching, text blocks, and sealed classes where they add clarity