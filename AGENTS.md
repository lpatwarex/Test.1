# AGENTS.md — E-Commerce Core Service

## 1. Stack

| Technology | Role |
|---|---|
| **Node.js + Express** | API Gateway layer — routes HTTP requests, handles auth middleware, rate limiting, request validation |
| **Java 21 + Spring Boot 3.x** | Core business logic microservice — user registration/authentication, profile management, product CRUD |
| **Python 3.12 + FastAPI** | Search microservice — wraps Elasticsearch queries, handles product search and listing endpoints |
| **Elasticsearch 8.x** | Search and product index store — full-text search, faceted filtering, product catalogue indexing |
| **PostgreSQL 16** | Primary relational store — users, profiles, product records (source of truth) |
| **Redis 7** | Session/token cache, rate-limit counters, short-lived auth state |
| **JWT (jsonwebtoken / jjwt)** | Stateless authentication tokens across all service boundaries |
| **Docker + Docker Compose** | Local orchestration of all services |
| **GitHub Actions** | CI pipeline — lint, test, build, push images |
| **Flyway** | Database migration management for PostgreSQL (Spring Boot service) |
| **Jest** | Unit and integration tests for Node.js/Express layer |
| **JUnit 5 + Mockito** | Unit and integration tests for Spring Boot service |
| **pytest + httpx** | Unit and integration tests for Python search service |

---

## 2. Project Structure

```
ecommerce-core-service/
│
├── AGENTS.md                          # This file
├── tasks.md                           # Agent-generated task tracker (created before coding)
├── docker-compose.yml                 # Orchestrates all services locally
├── docker-compose.override.yml        # Local dev overrides (ports, volumes, hot reload)
├── .env.example                       # Template for required environment variables
├── .gitignore
├── README.md
│
├── gateway/                           # Node.js + Express API Gateway
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   ├── .eslintrc.json
│   ├── .prettierrc
│   ├── jest.config.js
│   ├── src/
│   │   ├── app.js                     # Express app factory (no listen call)
│   │   ├── server.js                  # Entry point — calls app.listen()
│   │   ├── config/
│   │   │   ├── index.js               # Centralised env-var config with validation
│   │   │   └── logger.js              # Winston logger setup
│   │   ├── middleware/
│   │   │   ├── auth.middleware.js      # JWT verification, attach req.user
│   │   │   ├── error.middleware.js     # Global error handler
│   │   │   ├── rateLimiter.middleware.js
│   │   │   └── validate.middleware.js  # Joi/Zod schema validation factory
│   │   ├── routes/
│   │   │   ├── index.js               # Mounts all routers
│   │   │   ├── auth.routes.js         # POST /auth/register, POST /auth/login
│   │   │   ├── users.routes.js        # GET/PUT /users/:id (profile)
│   │   │   ├── products.routes.js     # GET/POST/PUT/DELETE /products
│   │   │   └── search.routes.js       # GET /search/products
│   │   ├── proxies/
│   │   │   ├── core.proxy.js          # http-proxy-middleware → Spring Boot
│   │   │   └── search.proxy.js        # http-proxy-middleware → Python search service
│   │   └── utils/
│   │       ├── asyncHandler.js        # Wraps async route handlers, forwards errors
│   │       └── httpClient.js          # Axios instance with retry and timeout
│   └── tests/
│       ├── unit/
│       │   ├── middleware/
│       │   └── utils/
│       └── integration/
│           ├── auth.routes.test.js
│           ├── users.routes.test.js
│           └── products.routes.test.js
│
├── core-service/                      # Java 21 + Spring Boot 3.x
│   ├── Dockerfile
│   ├── pom.xml                        # Maven build descriptor
│   ├── .mvn/wrapper/
│   │   └── maven-wrapper.properties
│   ├── mvnw
│   └── src/
│       ├── main/
│       │   ├── java/com/ecommerce/core/
│       │   │   ├── CoreServiceApplication.java   # @SpringBootApplication entry point
│       │   │   ├── config/
│       │   │   │   ├── SecurityConfig.java        # Spring Security + JWT filter chain
│       │   │   │   ├── JwtConfig.java             # JWT signing key, expiry beans
│       │   │   │   └── RedisConfig.java           # Lettuce connection factory
│       │   │   ├── controller/
│       │   │   │   ├── AuthController.java        # /internal/auth/register, /login
│       │   │   │   ├── UserController.java        # /internal/users/**
│       │   │   │   └── ProductController.java     # /internal/products/**
│       │   │   ├── service/
│       │   │   │   ├── AuthService.java
│       │   │   │   ├── UserService.java
│       │   │   │   └── ProductService.java
│       │   │   ├── repository/
│       │   │   │   ├── UserRepository.java        # Spring Data JPA
│       │   │   │   └── ProductRepository.java
│       │   │   ├── domain/
│       │   │   │   ├── entity/
│       │   │   │   │   ├── User.java              # @Entity
│       │   │   │   │   └── Product.java           # @Entity
│       │   │   │   └── dto/
│       │   │   │       ├── request/
│       │   │   │       │   ├── RegisterRequest.java
│       │   │   │       │   ├── LoginRequest.java
│       │   │   │       │   └── ProductRequest.java
│       │   │   │       └── response/
│       │   │   │           ├── AuthResponse.java
│       │   │   │           ├── UserResponse.java
│       │   │   │           └── ProductResponse.java
│       │   │   ├── exception/
│       │   │   │   ├── GlobalExceptionHandler.java  # @RestControllerAdvice
│       │   │   │   ├── ResourceNotFoundException.java
│       │   │   │   └── AuthException.java
│       │   │   ├── mapper/
│       │   │   │   ├── UserMapper.java             # MapStruct
│       │   │   │   └── ProductMapper.java
│       │   │   └── event/
│       │   │       └── ProductIndexEvent.java      # ApplicationEvent to trigger ES sync
│       │   └── resources/
│       │       ├── application.yml
│       │       ├── application-dev.yml
│       │       ├── application-prod.yml
│       │       └── db/migration/                  # Flyway SQL migrations
│       │           ├── V1__create_users_table.sql
│       │           ├── V2__create_products_table.sql
│       │           └── V3__add_indexes.sql
│       └── test/
│           └── java/com/ecommerce/core/
│               ├── controller/
│               │   ├── AuthControllerTest.java
│               │   ├── UserControllerTest.java
│               │   └── ProductControllerTest.java
│               ├── service/
│               │   ├── AuthServiceTest.java
│               │   ├── UserServiceTest.java
│               │   └── ProductServiceTest.java
│               └── integration/
│                   └── CoreServiceIntegrationTest.java  # @SpringBootTest + Testcontainers
│
├── search-service/                    # Python 3.12 + FastAPI
│   ├── Dockerfile
│   ├── pyproject.toml                 # Poetry project descriptor
│   ├── poetry.lock
│   ├── .ruff.toml                     # Linter config
│   ├── mypy.ini
│   ├── pytest.ini
│   ├── src/
│   │   └── search/
│   │       ├── __init__.py
│   │       ├── main.py                # FastAPI app factory + lifespan
│   │       ├── config.py              # pydantic-settings BaseSettings
│   │       ├── dependencies.py        # Elasticsearch client DI
│   │       ├── router/
│   │       │   ├── __init__.py
│   │       │   └── search.py          # GET /internal/search/products
│   │       ├── service/
│   │       │   ├── __init__.py
│   │       │   └── search_service.py  # Query building, pagination, facets
│   │       ├── schema/
│   │       │   ├── __init__.py
│   │       │   ├── request.py         # Pydantic request models
│   │       │   └── response.py        # Pydantic response models
│   │       ├── indexer/
│   │       │   ├── __init__.py
│   │       │   └── product_indexer.py # Consumes product sync events, indexes to ES
│   │       └── utils/
│   │           ├── __init__.py
│   │           └── es_query_builder.py
│   └── tests/
│       ├── conftest.py                # pytest fixtures, mock ES client
│       ├── unit/
│       │   ├── test_search_service.py
│       │   └── test_es_query_builder.py
│       └── integration/
│           └── test_search_router.py  # httpx AsyncClient against live FastAPI app
│
├── infra/
│   ├── elasticsearch/
│   │   └── mappings/
│   │       └── products.json          # ES index mapping definition
│   └── postgres/
│       └── init.sql                   # DB + role bootstrap (dev only)
│
└── .github/
    └── workflows/
        ├── ci-gateway.yml
        ├── ci-core-service.yml
        └── ci-search-service.yml
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder them.

### Step 1 — Read and Understand Specifications
- Read all story-level spec documents provided in the task context.
- Identify all bounded contexts: authentication, user profiles, product management, product search.
- Note all inter-service contracts (request/response shapes, internal HTTP paths).

### Step 2 — Create `tasks.md`
- Create `tasks.md` in the repository root **before writing any code**.
- Structure it with the following sections:

```markdown
# tasks.md

## Status Legend
- [ ] Not started
- [~] In progress
- [x] Complete

## Phase 1: Scaffolding
- [ ] Initialise gateway (Node.js/Express)
- [ ] Initialise core-service (Spring Boot)
- [ ] Initialise search-service (Python/FastAPI)
- [ ] Create docker-compose.yml
- [ ] Create .env.example

## Phase 2: Domain Implementation
- [ ] User registration + auth (core-service)
- [ ] User profile CRUD (core-service)
- [ ] Product CRUD (core-service)
- [ ] Product index sync to Elasticsearch (search-service indexer)
- [ ] Product search endpoint (search-service)
- [ ] Gateway routing + proxy configuration

## Phase 3: Testing
- [ ] Gateway unit tests (≥90% coverage)
- [ ] Core-service unit tests (≥90% coverage)
- [ ] Search-service unit tests (≥90% coverage)
- [ ] Integration tests for each service

## Phase 4: Validation
- [ ] All tests pass
- [ ] Coverage thresholds met
- [ ] Docker Compose stack starts cleanly
- [ ] Lint passes on all services
```

- Update task status as work progresses.

### Step 3 — Implement in Dependency Order
1. **Infra first** — write Flyway migrations, ES index mapping, `docker-compose.yml`.
2. **Core-service** — domain entities → repositories → services → controllers → exception handlers.
3. **Search-service** — config → ES client → indexer → query builder → router.
4. **Gateway** — config → middleware → proxies → routes.
5. Wire inter-service communication last (gateway → core, gateway → search, core → search indexer event).

### Step 4 — Test
- Write tests alongside each implementation file, not after all files are written.
- Run the full test suite for each service before moving to the next.
- Fix all failures before proceeding.

### Step 5 — Validate
- Run `docker compose up --build` and confirm all containers reach healthy state.
- Execute end-to-end smoke: register user → login → create product → search product.
- Confirm coverage reports meet the 90% threshold for all three services.
- Run linters (`eslint`, `checkstyle`/`spotbugs`, `ruff`, `mypy`) with zero errors.

---

## 4. Coding Conventions

### General (All Services)
- All internal service-to-service HTTP paths are prefixed with `/internal/` and are **not** exposed through the gateway to external clients.
- All external-facing paths are mounted at the gateway level only.
- Environment variables are the only acceptable configuration mechanism — no hardcoded secrets, ports, or hostnames anywhere.
- Every service must emit structured JSON logs with fields: `timestamp`, `level`, `service`, `traceId`, `message`.

### Node.js / Express (Gateway)
- Use **ES Modules** (`"type": "module"` in `package.json`).
- File naming: `kebab-case` for all files (e.g., `auth.middleware.js`, `users.routes.js`).
- Function naming: `camelCase`. Class naming: `PascalCase`.
- All route handlers must be wrapped with `asyncHandler` — never use bare `try/catch` in route files.
- Use `Zod` for request schema validation; define schemas in a `schemas/` directory co-located with routes.
- No business logic in route files — routes call proxy or service functions only.
- Use `winston` for logging; never use `console.log` in production code paths.
- HTTP client calls use the shared `httpClient.js` Axios instance (configured with timeout and retry).
- ESLint config: `eslint:recommended` + `plugin:node/recommended`. Prettier enforced.

### Java / Spring Boot (Core Service)
- Java package root: `com.ecommerce.core`.
- Class naming: `PascalCase`. Method/field naming: `camelCase`. Constants: `UPPER_SNAKE_CASE`.
- Use **records** for DTOs where the object is immutable (Java 21 records).
- All service methods must be annotated with `@Transactional` where DB writes occur.
- Use `MapStruct` for entity-to-DTO mapping — no manual mapping code in controllers or services.
- Passwords must be hashed with **BCrypt** (strength 12) via Spring Security's `PasswordEncoder`.
- JWT signing must use **RS256** (asymmetric) — store private key path in config, never inline.
- All controller endpoints must have `@Valid` on request body parameters.
- `@RestControllerAdvice` in `GlobalExceptionHandler` must handle: `MethodArgumentNotValidException`, `ResourceNotFoundException`, `AuthException`, and generic `Exception`.
- Use `@Slf4j` (Lombok) for logging — never use `System.out`.
- Spring profiles: `dev` (H2 or local PG), `prod` (production PG). Activate via `SPRING_PROFILES_