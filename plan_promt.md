## Technical Context and Implementation Approach

  ### Core Technology Stack
  - **Language/Version**: Go 1.21+
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

  ### Constitution Principles to Define
  Before proceeding, we should establish project constitution principles:
  1. **API-First Design**: OpenAPI/Swagger documentation before implementation
  2. **Test-Driven Development**: Write tests before implementation (critical for payment flows)
  3. **Error Handling**: Consistent error response format, proper HTTP status codes
  4. **Database Transactions**: All multi-step operations must be atomic
  5. **Code Organization**: Clear separation of concerns (handler -> service -> repository)
  6. **Documentation**: GoDoc comments for all public functions
  7. **Dependency Injection**: Use interfaces for testability

  ### Complexity Constraints
  - Keep business logic separate from HTTP handlers
  - Use repository pattern for data access abstraction
  - Avoid premature optimization; start simple and profile before optimizing
  - Prefer standard library over third-party packages when feasible