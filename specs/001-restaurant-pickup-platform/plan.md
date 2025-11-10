# Implementation Plan: Restaurant Self-Pickup Ordering Platform

**Branch**: `001-restaurant-pickup-platform` | **Date**: 2025-11-11 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-restaurant-pickup-platform/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Build a web-based restaurant self-pickup ordering platform that allows customers to browse restaurants, place orders online with scheduled pickup times, and receive real-time order status notifications. Restaurants manage incoming orders, update preparation status, and coordinate pickups. The platform handles payment authorization at checkout with capture upon restaurant acceptance, includes automated order timeout and cancellation workflows with optimistic locking for race condition prevention, and provides administrative oversight for restaurant applications and dispute resolution. Technical approach uses Go 1.23+ with Gin framework for RESTful API, PostgreSQL 15+ for data persistence, XORM as ORM, Stripe for payment processing, and follows API-first design with OpenAPI 3.x specifications. Architecture follows clean layered approach: handler → service → repository with comprehensive TDD coverage especially for payment flows.

## Technical Context

**Language/Version**: Go 1.23+
**Primary Dependencies**:
- Gin (RESTful API routing and HTTP handling)
- XORM (PostgreSQL ORM for database operations)
- Testify (unit and integration testing framework)
- Stripe Go SDK (payment processing)
- JWT-Go (authentication)
- golang-migrate (database migrations)

**Storage**: PostgreSQL 15+ (primary data store), Redis (session management and caching)
**Testing**: Testify (testify/assert, testify/mock, testify/suite), table-driven tests, integration tests with real PostgreSQL via docker-compose
**Target Platform**: Linux server with Docker containerization
**Project Type**: Web application with RESTful API backend
**Performance Goals**:
- Restaurant listings load within 2 seconds (NFR-005)
- Payment authorization completes within 5 seconds (NFR-006)
- Real-time notifications delivered within 10 seconds (NFR-007)
- Support 100 concurrent users without degradation (NFR-003)

**Constraints**:
- PCI DSS compliance for payment data handling (NFR-012)
- 99.5% uptime during business hours 8 AM-11 PM (NFR-009)
- HTTPS/TLS 1.3+ for all communications (NFR-013)
- Payment flows require 100% branch coverage (Constitution Principle II)

**Scale/Scope**:
- MVP: 50 restaurants, 10,000 customers, 100 concurrent users (NFR-001 to NFR-003)
- Design for 10x growth: 500 restaurants, 100,000 customers, 1,000 concurrent users (NFR-004)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Requirement | Status | Notes |
|-----------|-------------|--------|-------|
| **I. API-First Design** | OpenAPI 3.x spec before implementation | ✅ READY | Will generate OpenAPI 3.0 spec in Phase 1, define all endpoints with request/response schemas before coding |
| **II. Test-Driven Development** | Tests before implementation, 100% coverage for payment flows | ✅ READY | TDD workflow planned with testify framework, payment service will achieve 100% branch coverage per constitution requirement |
| **III. Error Handling** | Consistent error format with proper HTTP status codes | ✅ READY | Will define standard error response: `{"error": {"code": "ERR_CODE", "message": "...", "details": {...}}}` with correlation IDs |
| **IV. Database Transactions** | Atomic multi-step operations | ✅ READY | Payment workflows and state transitions will use explicit PostgreSQL transactions with documented boundaries |
| **V. Code Organization** | handler → service → repository separation | ✅ READY | Standard Go project layout with clear layer separation: `internal/handler/`, `internal/service/`, `internal/repository/` |
| **VI. Documentation** | GoDoc comments for all public functions | ✅ READY | Will enforce GoDoc standards: comments starting with function name, package-level docs, examples for complex logic |
| **VII. Dependency Injection** | Interfaces for testability | ✅ READY | Services will accept interfaces for repositories, payment gateways, notification services; constructors will validate non-nil dependencies |

**Complexity Constraints Check**:
- ✅ Business logic separated from handlers (Principle V)
- ✅ Repository pattern for data access (Principle V)
- ✅ Standard library first (using standard Go conventions, minimal third-party deps)
- ⚠️ **Redis caching layer** - Justified: NFR-005 requires 2s restaurant listing load times; profiling restaurant menu queries with 50 restaurants × 50 menu items = 2,500 records without caching would exceed performance budget
- ⚠️ **Scheduled jobs (cron)** - Justified: FR-059 requires auto-rejecting orders after 10 minutes; FR-037 requires auto-approving cancellations; these are explicit requirements, not premature optimization

## Project Structure

### Documentation (this feature)

```text
specs/001-restaurant-pickup-platform/
├── spec.md              # Feature specification (complete)
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output - technology decisions and best practices
├── data-model.md        # Phase 1 output - entity relationships and database schema
├── quickstart.md        # Phase 1 output - developer onboarding guide
├── contracts/           # Phase 1 output - OpenAPI 3.0 specifications
│   ├── openapi.yaml     # Main API specification
│   ├── schemas/         # Reusable schemas
│   └── examples/        # Request/response examples
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created yet)
```

### Source Code (repository root)

```text
restaurant-pickup-platform/
├── cmd/
│   └── server/
│       ├── main.go                    # Application entry point
│       └── main_test.go               # Entry point tests
│
├── internal/                          # Private application code
│   ├── models/                        # Domain models with validation
│   │   ├── user.go
│   │   ├── user_test.go              # Unit tests co-located
│   │   ├── restaurant.go
│   │   ├── restaurant_test.go
│   │   ├── order.go
│   │   ├── order_test.go
│   │   ├── menu_item.go
│   │   ├── menu_item_test.go
│   │   ├── payment.go
│   │   └── payment_test.go
│   │
│   ├── repository/                    # Data access layer (implements repository interfaces)
│   │   ├── interfaces.go             # Repository interfaces
│   │   ├── user_repository.go
│   │   ├── user_repository_test.go
│   │   ├── order_repository.go
│   │   ├── order_repository_test.go
│   │   ├── restaurant_repository.go
│   │   ├── restaurant_repository_test.go
│   │   ├── menu_repository.go
│   │   └── menu_repository_test.go
│   │
│   ├── service/                       # Business logic layer
│   │   ├── interfaces.go             # Service interfaces
│   │   ├── order_service.go          # Order placement, state transitions
│   │   ├── order_service_test.go
│   │   ├── payment_service.go        # Payment auth/capture/release (100% coverage)
│   │   ├── payment_service_test.go
│   │   ├── notification_service.go   # Email, SMS, push notifications
│   │   ├── notification_service_test.go
│   │   ├── auth_service.go           # JWT token generation, password hashing
│   │   ├── auth_service_test.go
│   │   ├── restaurant_service.go     # Restaurant profile, menu management
│   │   ├── restaurant_service_test.go
│   │   ├── cancellation_service.go   # Cancellation logic with grace periods
│   │   └── cancellation_service_test.go
│   │
│   ├── handler/                       # HTTP handlers (Gin controllers)
│   │   ├── order_handler.go          # Order endpoints
│   │   ├── order_handler_test.go
│   │   ├── restaurant_handler.go     # Restaurant profile/menu endpoints
│   │   ├── restaurant_handler_test.go
│   │   ├── auth_handler.go           # Login, register, password reset
│   │   ├── auth_handler_test.go
│   │   ├── user_handler.go           # Customer profile, order history
│   │   ├── user_handler_test.go
│   │   ├── admin_handler.go          # Admin dashboard, disputes
│   │   └── admin_handler_test.go
│   │
│   ├── middleware/                    # HTTP middleware
│   │   ├── auth.go                   # JWT validation
│   │   ├── auth_test.go
│   │   ├── rate_limiter.go           # Rate limiting (5 attempts per 15 min)
│   │   ├── rate_limiter_test.go
│   │   ├── logger.go                 # Structured logging with correlation IDs
│   │   ├── logger_test.go
│   │   ├── error_handler.go          # Standard error response formatting
│   │   └── error_handler_test.go
│   │
│   ├── scheduler/                     # Time-based task scheduler
│   │   ├── order_timeout.go          # Auto-reject orders after 10 minutes (FR-059)
│   │   ├── order_timeout_test.go
│   │   ├── reminder.go               # Pickup reminders 15 min before (FR-033)
│   │   ├── reminder_test.go
│   │   ├── cancellation_auto.go      # Auto-approve cancellations after 5 min (FR-037)
│   │   └── cancellation_auto_test.go
│   │
│   └── testutil/                      # Shared test utilities
│       ├── mocks.go                   # Common mocks (payment gateway, notification service)
│       ├── fixtures.go                # Test data fixtures
│       └── db.go                      # Database test helpers
│
├── pkg/                               # Public reusable packages
│   ├── validator/                     # Input validation utilities
│   │   ├── validator.go
│   │   └── validator_test.go
│   │
│   ├── timeutil/                      # Time handling with grace buffers
│   │   ├── time.go                   # Server timestamp with 30s grace buffer (FR-034)
│   │   └── time_test.go
│   │
│   └── errors/                        # Standard error types
│       ├── errors.go                 # Error codes and formatting
│       └── errors_test.go
│
├── tests/                             # Integration & E2E tests ONLY
│   ├── integration/                   # Integration tests with real dependencies
│   │   ├── database_test.go          # Database operations
│   │   ├── order_repository_integration_test.go
│   │   ├── payment_integration_test.go  # Payment gateway integration
│   │   └── notification_integration_test.go
│   │
│   ├── e2e/                           # End-to-end business flow tests
│   │   ├── order_flow_test.go        # Complete order placement → pickup
│   │   ├── cancellation_flow_test.go # Cancellation scenarios with race conditions
│   │   ├── payment_capture_test.go   # Auth/capture/release flows
│   │   └── timeout_flow_test.go      # Auto-rejection and auto-approval flows
│   │
│   └── testutil/                      # Test utilities for integration/E2E
│       ├── db_helpers.go             # Database setup/teardown
│       ├── fixtures.go               # Test data
│       └── docker-compose.test.yml   # Test infrastructure (PostgreSQL, Redis)
│
├── migrations/                        # Database migrations (golang-migrate)
│   ├── 000001_create_users_table.up.sql
│   ├── 000001_create_users_table.down.sql
│   ├── 000002_create_restaurants_table.up.sql
│   ├── 000002_create_restaurants_table.down.sql
│   ├── 000003_create_menu_items_table.up.sql
│   ├── 000003_create_menu_items_table.down.sql
│   ├── 000004_create_orders_table.up.sql
│   ├── 000004_create_orders_table.down.sql
│   ├── 000005_create_payment_transactions_table.up.sql
│   └── 000005_create_payment_transactions_table.down.sql
│
├── docs/                              # Documentation
│   ├── api/
│   │   └── openapi.yaml              # OpenAPI 3.0 specification (generated in Phase 1)
│   └── architecture/
│       ├── database_schema.md
│       ├── payment_flow.md           # Payment auth/capture/release workflows
│       └── state_transitions.md      # Order state machine documentation
│
├── scripts/                           # Utility scripts
│   ├── setup_dev_db.sh               # Setup development database
│   ├── run_tests.sh                  # Run all tests with coverage
│   └── migrate.sh                    # Database migration helper
│
├── go.mod                             # Go module definition
├── go.sum                             # Dependency checksums
├── Makefile                           # Common commands (build, test, migrate, lint)
├── docker-compose.yml                 # Development infrastructure (PostgreSQL, Redis)
├── Dockerfile                         # Application container
├── .env.example                       # Environment variables template
├── .gitignore
└── README.md
```

**Structure Decision**: Web application with RESTful API backend using standard Go project layout. The structure follows Go community conventions with `cmd/` for entry points, `internal/` for private application code organized by layer (handler → service → repository per Constitution Principle V), `pkg/` for reusable utilities, and `tests/` for integration/E2E tests only (unit tests co-located with source per Go idioms). This structure supports the clean architecture required by the constitution while maintaining Go best practices.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Redis caching layer | NFR-005 requires restaurant listings load within 2 seconds. With 50 restaurants × ~50 menu items = 2,500 database records per listing query, uncached queries would exceed performance budget based on typical PostgreSQL query times (~100-300ms for complex joins + network latency) | Direct database queries insufficient: simple calculation shows 50 restaurants with menu items, reviews, and ratings require 3-5 table joins per listing, exceeding 2s budget without caching |
| Scheduled jobs (robfig/cron) | FR-059 explicitly requires auto-rejecting orders after 10 minutes; FR-037 requires auto-approving cancellations after 5 minutes if restaurant doesn't respond; FR-033 requires reminder notifications 15 minutes before pickup. These are not optimizations but core requirements | Manual processing insufficient: requirements explicitly demand automated time-based actions at specific intervals; no simpler alternative exists for scheduled background tasks |
| Stripe payment gateway integration | PCI DSS compliance (NFR-012) requires tokenization and secure payment handling; building in-house payment processing would violate security standards and introduce massive compliance overhead | In-house payment processing rejected: PCI DSS Level 1 compliance costs $50k-500k+; Stripe provides required auth-then-capture flow (FR-018 to FR-021) with built-in compliance |

