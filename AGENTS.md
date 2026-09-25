# AGENTS.md — E-commerce Core Service

> **Purpose:** This file is the authoritative scaffold specification for the E-commerce Core Service. Every AI agent working on this repository must read and follow this document before writing a single line of code.

---

## 1. Stack

| Layer | Technology | Role |
|---|---|---|
| Auth & User Service | Java 21 + Spring Boot 3.x | User registration, authentication (JWT/OAuth2), session management |
| Product & Inventory Service | Node.js 20 LTS + Express 4.x | Product listings, inventory tracking, stock updates |
| Wishlist Service | Python 3.12 + Flask 3.x | User wishlist management, wishlist item CRUD |
| ORM / Data Access (Java) | Spring Data JPA + Hibernate | Entity mapping, repository pattern for user data |
| ORM / Data Access (Node.js) | Prisma or Sequelize | Product and inventory schema management |
| ORM / Data Access (Python) | SQLAlchemy 2.x | Wishlist data persistence |
| Primary Database | PostgreSQL 16 | Shared relational store (separate schemas per service) |
| Cache | Redis 7 | Token blacklisting, session cache, inventory counters |
| Message Broker | RabbitMQ 3.x | Async events (inventory updates, wishlist triggers) |
| API Gateway | NGINX | Route ingress to each service, TLS termination |
| Testing (Java) | JUnit 5, Mockito, Testcontainers | Unit, integration, and contract tests |
| Testing (Node.js) | Jest, Supertest, Testcontainers-node | Unit, integration, and API tests |
| Testing (Python) | pytest, pytest-cov, factory-boy | Unit and integration tests |
| Containerisation | Docker + Docker Compose | Local dev and CI parity |
| CI | GitHub Actions | Lint, test, build, push pipeline |
| Secret Management | Spring Vault / dotenv / python-decouple | Environment-specific secrets, never hardcoded |

---

## 2. Project Structure

```
ecommerce-core-service/
│
├── AGENTS.md                          # This file — read first
├── tasks.md                           # Agent-generated task tracker (created before coding)
├── docker-compose.yml                 # Orchestrates all services + infra locally
├── docker-compose.test.yml            # Overrides for isolated test runs
├── .env.example                       # Template for required environment variables
├── .github/
│   └── workflows/
│       ├── ci.yml                     # Main CI pipeline
│       └── pr-checks.yml             # PR-level lint and test gates
├── nginx/
│   ├── nginx.conf                     # Upstream routing to each service
│   └── Dockerfile                     # NGINX image build
│
├── services/
│   │
│   ├── auth-service/                  # Java / Spring Boot — User auth & registration
│   │   ├── Dockerfile
│   │   ├── pom.xml                    # Maven build descriptor
│   │   ├── .mvn/wrapper/              # Maven wrapper for reproducible builds
│   │   └── src/
│   │       ├── main/
│   │       │   ├── java/com/ecommerce/auth/
│   │       │   │   ├── AuthServiceApplication.java      # Spring Boot entry point
│   │       │   │   ├── config/
│   │       │   │   │   ├── SecurityConfig.java          # Spring Security + JWT config
│   │       │   │   │   ├── JwtConfig.java               # JWT properties binding
│   │       │   │   │   └── RedisConfig.java             # Redis connection config
│   │       │   │   ├── controller/
│   │       │   │   │   └── AuthController.java          # REST endpoints: /register, /login, /logout
│   │       │   │   ├── service/
│   │       │   │   │   ├── AuthService.java             # Business logic interface
│   │       │   │   │   └── AuthServiceImpl.java         # Implementation
│   │       │   │   ├── repository/
│   │       │   │   │   └── UserRepository.java          # Spring Data JPA repository
│   │       │   │   ├── domain/
│   │       │   │   │   ├── User.java                    # JPA entity
│   │       │   │   │   └── Role.java                    # Role enum/entity
│   │       │   │   ├── dto/
│   │       │   │   │   ├── RegisterRequest.java
│   │       │   │   │   ├── LoginRequest.java
│   │       │   │   │   └── AuthResponse.java
│   │       │   │   ├── exception/
│   │       │   │   │   ├── GlobalExceptionHandler.java  # @ControllerAdvice
│   │       │   │   │   └── AuthException.java
│   │       │   │   └── util/
│   │       │   │       └── JwtUtil.java                 # Token generation/validation
│   │       │   └── resources/
│   │       │       ├── application.yml                  # Base config
│   │       │       ├── application-dev.yml
│   │       │       └── application-prod.yml
│   │       └── test/
│   │           └── java/com/ecommerce/auth/
│   │               ├── controller/
│   │               │   └── AuthControllerTest.java      # MockMvc tests
│   │               ├── service/
│   │               │   └── AuthServiceImplTest.java     # Mockito unit tests
│   │               └── integration/
│   │                   └── AuthIntegrationTest.java     # Testcontainers + full stack
│   │
│   ├── product-service/               # Node.js / Express — Products & inventory
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   ├── package-lock.json
│   │   ├── tsconfig.json              # TypeScript config (use TS for type safety)
│   │   ├── prisma/
│   │   │   ├── schema.prisma          # Product and inventory models
│   │   │   └── migrations/            # Auto-generated migration files
│   │   └── src/
│   │       ├── app.ts                 # Express app factory (no listen here)
│   │       ├── server.ts              # Entry point — calls app.listen
│   │       ├── config/
│   │       │   ├── env.ts             # Validated env vars (use zod)
│   │       │   ├── database.ts        # Prisma client singleton
│   │       │   └── redis.ts           # Redis client
│   │       ├── routes/
│   │       │   ├── index.ts           # Route aggregator
│   │       │   ├── product.routes.ts
│   │       │   └── inventory.routes.ts
│   │       ├── controllers/
│   │       │   ├── product.controller.ts
│   │       │   └── inventory.controller.ts
│   │       ├── services/
│   │       │   ├── product.service.ts
│   │       │   └── inventory.service.ts
│   │       ├── middleware/
│   │       │   ├── auth.middleware.ts  # JWT verification (shared secret)
│   │       │   ├── validate.middleware.ts
│   │       │   └── error.middleware.ts
│   │       ├── dto/
│   │       │   ├── product.dto.ts
│   │       │   └── inventory.dto.ts
│   │       ├── events/
│   │       │   ├── publisher.ts        # RabbitMQ event publishing
│   │       │   └── subscriber.ts       # RabbitMQ event consuming
│   │       └── types/
│   │           └── express.d.ts        # Augmented Express Request types
│   └── tests/
│       ├── unit/
│       │   ├── product.service.test.ts
│       │   └── inventory.service.test.ts
│       ├── integration/
│       │   ├── product.routes.test.ts  # Supertest
│       │   └── inventory.routes.test.ts
│       └── jest.config.ts
│   │
│   └── wishlist-service/              # Python / Flask — Wishlist management
│       ├── Dockerfile
│       ├── pyproject.toml             # PEP 517 build + dependency management
│       ├── requirements.txt           # Pinned production deps
│       ├── requirements-dev.txt       # Dev/test deps
│       ├── alembic.ini                # Alembic migration config
│       ├── migrations/
│       │   ├── env.py
│       │   └── versions/              # Migration scripts
│       └── src/
│           └── wishlist/
│               ├── __init__.py        # Flask app factory
│               ├── app.py             # create_app() factory function
│               ├── config.py          # Config classes (Dev, Prod, Test)
│               ├── models/
│               │   ├── __init__.py
│               │   └── wishlist.py    # SQLAlchemy Wishlist + WishlistItem models
│               ├── schemas/
│               │   ├── __init__.py
│               │   └── wishlist.py    # Marshmallow or Pydantic schemas
│               ├── routes/
│               │   ├── __init__.py
│               │   └── wishlist.py    # Blueprint: /wishlists
│               ├── services/
│               │   ├── __init__.py
│               │   └── wishlist_service.py
│               ├── repositories/
│               │   ├── __init__.py
│               │   └── wishlist_repository.py
│               ├── events/
│               │   ├── __init__.py
│               │   └── publisher.py   # RabbitMQ publish helpers
│               ├── middleware/
│               │   └── auth.py        # JWT decode before_request guard
│               └── exceptions/
│                   └── handlers.py    # Registered error handlers
│           └── tests/
│               ├── conftest.py        # Fixtures: app, client, db session
│               ├── unit/
│               │   ├── test_wishlist_service.py
│               │   └── test_wishlist_repository.py
│               └── integration/
│                   └── test_wishlist_routes.py
```

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder them.

### Step 1 — Read All Specifications
- Read `AGENTS.md` (this file) completely before any action.
- Read any story-level spec files present in `docs/specs/` if they exist.
- Identify all service boundaries, data contracts, and event schemas described.

### Step 2 — Create `tasks.md`
- Create `tasks.md` at the repository root **before writing any implementation code**.
- Structure it as a checklist with the following sections:

```markdown
# tasks.md

## Infrastructure Setup
- [ ] docker-compose.yml with postgres, redis, rabbitmq, nginx
- [ ] .env.example populated with all required keys

## Auth Service (Java / Spring Boot)
- [ ] Scaffold Maven project with required dependencies
- [ ] Implement domain entities and JPA mappings
- [ ] Implement repository layer
- [ ] Implement service layer with business logic
- [ ] Implement REST controllers
- [ ] Write unit tests (target ≥90% coverage)
- [ ] Write integration tests with Testcontainers
- [ ] Dockerfile

## Product & Inventory Service (Node.js / Express)
- [ ] Scaffold TypeScript project with required dependencies
- [ ] Define Prisma schema and generate client
- [ ] Implement service and controller layers
- [ ] Implement RabbitMQ event publisher/subscriber
- [ ] Write Jest unit and Supertest integration tests (target ≥90% coverage)
- [ ] Dockerfile

## Wishlist Service (Python / Flask)
- [ ] Scaffold Flask app with factory pattern
- [ ] Define SQLAlchemy models and Alembic migrations
- [ ] Implement repository and service layers
- [ ] Implement Flask blueprints and routes
- [ ] Write pytest unit and integration tests (target ≥90% coverage)
- [ ] Dockerfile

## CI Pipeline
- [ ] GitHub Actions ci.yml
- [ ] PR check workflow

## Final Validation
- [ ] docker-compose up --build succeeds
- [ ] All test suites pass
- [ ] Coverage thresholds met across all services
```

- Check off each item as it is completed. Never mark an item complete without verifying it.

### Step 3 — Implement Infrastructure First
- Write `docker-compose.yml` and `.env.example` before any service code.
- Confirm PostgreSQL, Redis, RabbitMQ, and NGINX containers are correctly defined.
- Each service must have its own PostgreSQL schema (`auth`, `product`, `wishlist`).

### Step 4 — Implement Each Service
- Implement services in this order: **auth-service → product-service → wishlist-service**.
- Follow the layered architecture: `domain/model → repository → service → controller/route`.
- After each layer is implemented, write its tests before moving to the next layer.
- Never leave a TODO comment without a corresponding `tasks.md` entry.

### Step 5 — Test
- Run all test suites and confirm they pass before proceeding.
- Confirm coverage meets the 90% threshold for each service independently.
- Fix all failures before moving to CI configuration.

### Step 6 — Validate End-to-End
- Run `docker-compose up --build` and confirm all containers start without error.
- Execute a smoke-test sequence: register user → login → create product → update inventory → add to wishlist.
- Confirm NGINX correctly proxies each route to the correct upstream service.
- Mark all `tasks.md` items complete only after this validation passes.

---

## 4. Coding Conventions

### General (All Services)
- All environment variables must be loaded from `.env` files or environment injection — **never hardcoded**.
- All inter-service communication must use the shared JWT secret for token verification.
- Event names on RabbitMQ must follow `snake_case` and be namespaced: `inventory.stock_updated`, `wishlist.item_added`.
- All HTTP responses must follow a consistent envelope:
  ```json
  { "data": {}, "error": null, "meta": {} }
  ```
- All timestamps must be stored and transmitted as UTC ISO 8601.

### Java / Spring Boot (auth-service)
- Use **Java 21** with records for DTOs where applicable.
- Follow standard Spring layering: `@RestController` → `@Service` → `@Repository`.
- Use `@ControllerAdvice` with `@ExceptionHandler` for all error responses — no raw exception propagation to controllers.
- Bean validation via Jakarta Validation (`@Valid`, `@NotBlank`, `@Email`, etc.) on all request DTOs.
- Use constructor injection (not field injection) for all Spring beans.
- Class names: `PascalCase`. Method names: `camelCase`. Constants: `UPPER_SNAKE_CASE`.
- Package structure must mirror `com.ecommerce.<service>.<layer>`.
- Passwords must be hashed with `BCryptPasswordEncoder` — never store plaintext.
- JWT tokens must have configurable expiry, stored in `application.yml`, not hardcoded.

### Node.js / Express (product-service)
- Use **TypeScript** with `strict: true` in `tsconfig.json`. No `any` types permitted.
- Use `zod` for environment variable validation and request body validation.
- Controllers must be thin — delegate all logic to service classes.
- All async route handlers must be wrapped in a `asyncHandler` utility to forward errors to the Express error middleware.
- File naming: `kebab-case` for files, `PascalCase` for classes, `camelCase` for functions and variables.
- Prisma migrations must be committed — never run `prisma db push` in production.
- Use `pino` for structured JSON logging — no `console.log` in production code.
- All exported functions and