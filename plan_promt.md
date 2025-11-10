## Technical Context and Implementation Approach

  ### Core Technology Stack
  - **Language/Version**: Go 1.23+
  - **Web Framework**: Gin (for RESTful API routing and HTTP handling)
  - **ORM**: XORM (for PostgreSQL database operations)
  - **Database**: PostgreSQL 15+ (primary data store)
  - **Testing**: Testify (unit and integration testing framework)
  - **Project Type**: Web application with RESTful API backend

  ### Architecture Style
  - **API Design**: RESTful API with JSON responses
  - **Project Structure**: Standard Go project layout following Go community conventions
    - `cmd/server/` - Main application entry point
    - `internal/` - Private application code (models, services, handlers, repositories)
      - Each package contains its own `*_test.go` files for unit tests (Go convention)
    - `pkg/` - Public reusable packages (also with co-located `*_test.go` files)
    - `tests/` - Integration and E2E tests ONLY (not unit tests)
      - `tests/integration/` - Database and external service integration tests
      - `tests/e2e/` - End-to-end business flow tests
    - `migrations/` - Database migration files
    - `docs/` - API documentation (OpenAPI/Swagger specs)

  ### Additional Technical Decisions Required

  **Authentication & Authorization**:
  - JWT-based authentication for stateless API
  - Role-based access control (Customer, Restaurant Owner, Admin)
  - Secure password hashing (bcrypt)

  **Real-time Notifications**:
  - RESEARCH NEEDED: WebSocket vs Server-Sent Events vs Third-party service (Twilio, SendGrid)
  - Email: SMTP integration (consideration: SendGrid, AWS SES)
  - SMS: Third-party API (consideration: Twilio)
  - Push notifications: Consider Firebase Cloud Messaging or similar

  **Payment Processing**:
  - Integration with payment gateway (Stripe recommended for auth + capture flow)
  - PCI DSS compliance through tokenization
  - Separate payment service module for security isolation

  **Caching & Performance**:
  - Redis for session management and frequently accessed data (restaurant listings, menus)
  - Database query optimization with XORM's caching capabilities
  - Connection pooling for database efficiency

  **Concurrency Control**:
  - Optimistic locking using version field (as per FR-030)
  - Transaction management for critical operations (order placement, state transitions)
  - Goroutines for async operations (notifications, background jobs)

  **File Storage**:
  - Restaurant and menu item images
  - RESEARCH NEEDED: Local filesystem vs S3-compatible storage (MinIO for self-hosted)

  **Time-based Operations**:
  - Cron jobs or scheduled tasks for:
    - Auto-rejecting orders after 10 minutes (FR-059)
    - Sending reminder notifications (FR-033)
    - Auto-approving cancellations after 5 minutes (FR-037)
  - Consider: Go's time.Ticker or third-party scheduler (robfig/cron)

  ### Performance & Scale Goals
  - **Target Scale** (per NFR-001 to NFR-003):
    - 50 restaurants (MVP)
    - 10,000 customers (MVP)
    - 100 concurrent users (MVP)
  - **Performance Targets**:
    - API response time: <2s for listings (NFR-005)
    - Payment authorization: <5s (NFR-006)
    - Notification delivery: <10s (NFR-007)
  - **Future Scale** (NFR-004): Design for 10x growth (500 restaurants, 100k customers)

  ### Deployment & Infrastructure
  - **Target Platform**: Linux server (Docker containerization recommended)
  - **Database Migrations**: golang-migrate or XORM's migration tools
  - **Configuration**: Environment variables for secrets and config
  - **Logging**: Structured logging (logrus or zap)
  - **Monitoring**: Health check endpoints, metrics exposure

  ### Testing Strategy (Go Idiomatic Approach)
  - **Unit Tests**: Co-located with source code using `*_test.go` suffix
    - Example: `internal/service/order_service.go` → `internal/service/order_service_test.go`
    - Use `testify/assert` for assertions, `testify/mock` for mocking dependencies
    - Use `testify/suite` for test fixtures and setup/teardown
    - Can use either `package service` (white-box) or `package service_test` (black-box)
    - Run with: `go test ./internal/...` or `go test ./...`

  - **Integration Tests**: Centralized in `/tests/integration/`
    - Test database operations with real PostgreSQL instance (docker-compose)
    - Test external API integrations (payment gateway, notification services)
    - Require external dependencies (database, Redis, etc.)
    - Run with: `go test ./tests/integration/... -tags=integration`

  - **E2E Tests**: Centralized in `/tests/e2e/`
    - Test complete business flows (order placement → payment → notification → pickup)
    - May use dedicated test environment or in-memory implementations
    - Run with: `go test ./tests/e2e/... -tags=e2e`

  - **Test Utilities**:
    - Shared mocks, fixtures, and helpers in `internal/testutil/`
    - Database test helpers (setup/teardown, fixtures) in `tests/testutil/`

  - **Test Coverage Goal**: Minimum 80% for critical paths (order flow, payment, cancellation)

  ### Security Considerations
  - **HTTPS/TLS**: Required for all client-server communication (NFR-013)
  - **Password Policy**: Min 8 chars, mixed case, numbers, special chars (NFR-014)
  - **Rate Limiting**: 5 failed login attempts per 15 min per IP (NFR-015)
  - **Input Validation**: Request validation middleware in Gin
  - **SQL Injection Prevention**: Parameterized queries via XORM
  - **CORS**: Proper CORS configuration for frontend access

  ### Open Questions for Research (Phase 0)
  1. Best practices for implementing payment auth/capture flow with Stripe in Go
  2. Real-time notification architecture: WebSocket vs SSE vs polling for order status
  3. Optimal XORM configuration for PostgreSQL connection pooling
  4. Time-based automation: Best Go scheduler for cron-like tasks
  5. Image upload and storage: S3-compatible storage patterns in Go
  6. Gin middleware best practices for auth, logging, error handling
  7. Database migration strategy with XORM and PostgreSQL

  ### Detailed Project Structure Example
  ```
  restaurant-pickup-platform/
  ├── cmd/
  │   └── server/
  │       ├── main.go                    # Application entry point
  │       └── main_test.go               # Entry point tests (if needed)
  │
  ├── internal/                          # Private application code
  │   ├── models/                        # Domain models
  │   │   ├── user.go
  │   │   ├── user_test.go              # ✅ Unit tests co-located
  │   │   ├── restaurant.go
  │   │   ├── restaurant_test.go
  │   │   ├── order.go
  │   │   ├── order_test.go
  │   │   ├── menu_item.go
  │   │   └── menu_item_test.go
  │   │
  │   ├── repository/                    # Data access layer
  │   │   ├── user_repository.go
  │   │   ├── user_repository_test.go   # ✅ Unit tests with mocks
  │   │   ├── order_repository.go
  │   │   ├── order_repository_test.go
  │   │   ├── restaurant_repository.go
  │   │   └── restaurant_repository_test.go
  │   │
  │   ├── service/                       # Business logic layer
  │   │   ├── order_service.go
  │   │   ├── order_service_test.go     # ✅ Unit tests
  │   │   ├── payment_service.go
  │   │   ├── payment_service_test.go
  │   │   ├── notification_service.go
  │   │   ├── notification_service_test.go
  │   │   ├── auth_service.go
  │   │   └── auth_service_test.go
  │   │
  │   ├── handler/                       # HTTP handlers (Gin controllers)
  │   │   ├── order_handler.go
  │   │   ├── order_handler_test.go     # ✅ Unit tests
  │   │   ├── restaurant_handler.go
  │   │   ├── restaurant_handler_test.go
  │   │   ├── auth_handler.go
  │   │   └── auth_handler_test.go
  │   │
  │   ├── middleware/                    # HTTP middleware
  │   │   ├── auth.go
  │   │   ├── auth_test.go              # ✅ Unit tests
  │   │   ├── rate_limiter.go
  │   │   ├── rate_limiter_test.go
  │   │   ├── logger.go
  │   │   └── error_handler.go
  │   │
  │   ├── scheduler/                     # Time-based task scheduler
  │   │   ├── order_timeout.go          # Auto-reject after 10 minutes
  │   │   ├── order_timeout_test.go
  │   │   ├── reminder.go               # Pickup reminders
  │   │   └── reminder_test.go
  │   │
  │   └── testutil/                      # Shared test utilities
  │       ├── mocks.go                   # Common mocks
  │       └── fixtures.go                # Test data fixtures
  │
  ├── pkg/                               # Public reusable packages
  │   ├── validator/                     # Input validation utilities
  │   │   ├── validator.go
  │   │   └── validator_test.go         # ✅ Unit tests
  │   │
  │   └── utils/                         # Common utilities
  │       ├── time.go
  │       └── time_test.go
  │
  ├── tests/                             # ⚠️ Integration & E2E tests ONLY
  │   ├── integration/                   # Integration tests
  │   │   ├── database_test.go          # Database operations with real DB
  │   │   ├── order_repository_integration_test.go
  │   │   ├── payment_integration_test.go
  │   │   └── notification_integration_test.go
  │   │
  │   ├── e2e/                           # End-to-end tests
  │   │   ├── order_flow_test.go        # Complete order flow
  │   │   ├── cancellation_flow_test.go # Cancellation scenarios
  │   │   └── pickup_flow_test.go       # Pickup scenarios
  │   │
  │   └── testutil/                      # Test utilities for integration/E2E
  │       ├── db_helpers.go             # Database setup/teardown
  │       ├── fixtures.go               # Test data for integration tests
  │       └── docker-compose.test.yml   # Test infrastructure
  │
  ├── migrations/                        # Database migrations
  │   ├── 001_create_users_table.sql
  │   ├── 002_create_restaurants_table.sql
  │   ├── 003_create_orders_table.sql
  │   └── 004_create_menu_items_table.sql
  │
  ├── docs/                              # Documentation
  │   ├── api/
  │   │   └── openapi.yaml              # OpenAPI 3.0 specification
  │   └── architecture/
  │       └── database_schema.md
  │
  ├── scripts/                           # Utility scripts
  │   ├── setup_dev_db.sh               # Setup development database
  │   └── run_tests.sh                  # Run all tests
  │
  ├── go.mod                             # Go module definition
  ├── go.sum                             # Dependency checksums
  ├── Makefile                           # Common commands (build, test, migrate)
  ├── docker-compose.yml                 # Development infrastructure
  ├── Dockerfile                         # Application container
  ├── .env.example                       # Environment variables template
  ├── .gitignore
  └── README.md
  ```

  ### Key Testing Commands
  ```bash
  # Run all unit tests (co-located with source code)
  go test ./internal/... ./pkg/... -v -cover

  # Run integration tests (requires Docker)
  docker-compose -f tests/testutil/docker-compose.test.yml up -d
  go test ./tests/integration/... -tags=integration -v

  # Run E2E tests
  go test ./tests/e2e/... -tags=e2e -v

  # Run all tests with coverage
  go test ./... -coverprofile=coverage.out
  go tool cover -html=coverage.out

  # Run tests for specific package
  go test ./internal/service -v -run TestOrderService
  ```