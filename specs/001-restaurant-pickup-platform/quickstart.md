# Quickstart Guide: Restaurant Self-Pickup Ordering Platform

**Branch**: `001-restaurant-pickup-platform` | **Date**: 2025-11-11

## Overview

This guide will help you set up the development environment and start building the restaurant pickup platform in under 30 minutes. We'll use Test-Driven Development (TDD) principles from the project constitution.

---

## Prerequisites

Before starting, ensure you have these installed:

```bash
# Required
- Go 1.23+ (https://golang.org/dl/)
- PostgreSQL 15+ (https://www.postgresql.org/download/)
- Git

# Recommended
- Docker & Docker Compose (for easy PostgreSQL/Redis setup)
- Make (for running Makefile commands)
- Postman or curl (for API testing)
```

**Verify installations:**
```bash
go version        # Should show go1.23 or higher
psql --version    # Should show PostgreSQL 15 or higher
docker --version  # Optional but recommended
```

---

## Quick Setup (5 Minutes)

### 1. Clone and Initialize Repository

```bash
# Clone the repository
git clone <repository-url>
cd restaurant-pickup-platform

# Checkout the feature branch
git checkout 001-restaurant-pickup-platform

# Initialize Go module
go mod init github.com/yourusername/restaurant-pickup-platform

# Install initial dependencies
go get github.com/gin-gonic/gin
go get xorm.io/xorm
go get github.com/lib/pq
go get github.com/stretchr/testify
go get github.com/stripe/stripe-go/v76
go get github.com/golang-migrate/migrate/v4
go get github.com/robfig/cron/v3
```

### 2. Environment Setup

```bash
# Copy environment template
cp .env.example .env

# Edit .env with your configuration
```

**`.env.example` contents:**
```env
# Server
PORT=8080
ENV=development

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=restaurant_pickup
DB_SSLMODE=disable

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT
JWT_SECRET=your-secret-key-change-in-production

# Stripe (use test keys)
STRIPE_SECRET_KEY=sk_test_your_key_here
STRIPE_PUBLISHABLE_KEY=pk_test_your_key_here

# SendGrid
SENDGRID_API_KEY=your_sendgrid_key

# Twilio
TWILIO_ACCOUNT_SID=your_account_sid
TWILIO_AUTH_TOKEN=your_auth_token
TWILIO_PHONE_NUMBER=+1234567890
```

### 3. Start Infrastructure with Docker Compose

```bash
# Start PostgreSQL and Redis
docker-compose up -d

# Verify services are running
docker-compose ps
```

**`docker-compose.yml`:**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: restaurant_pickup
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### 4. Run Database Migrations

```bash
# Install golang-migrate
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Run migrations
make migrate-up

# Or manually:
migrate -path ./migrations -database "postgresql://postgres:postgres@localhost:5432/restaurant_pickup?sslmode=disable" up
```

---

## Project Structure Tour

```
restaurant-pickup-platform/
├── cmd/server/main.go           # Application entry point - START HERE
├── internal/
│   ├── models/                  # Domain models (User, Order, Restaurant)
│   ├── repository/              # Data access layer (XORM)
│   ├── service/                 # Business logic (TDD starts here!)
│   ├── handler/                 # HTTP handlers (Gin)
│   ├── middleware/              # Auth, logging, rate limiting
│   └── scheduler/               # Cron jobs for timeouts
├── pkg/                         # Reusable utilities
├── tests/                       # Integration and E2E tests
├── migrations/                  # Database migrations
├── docs/api/openapi.yaml        # API specification (API-First!)
└── Makefile                     # Common commands
```

---

## Development Workflow (TDD - Constitution Required!)

### The TDD Cycle: Red → Green → Refactor

Per the project constitution, **MUST write tests before implementation**. Here's how:

### Example: Building the Order Service (Payment Flow)

#### Step 1: Write the Test First (RED)

Create `internal/service/order_service_test.go`:

```go
package service_test

import (
    "context"
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/yourusername/restaurant-pickup-platform/internal/models"
    "github.com/yourusername/restaurant-pickup-platform/internal/service"
    "github.com/yourusername/restaurant-pickup-platform/internal/testutil"
)

func TestOrderService_CreateOrder_Success(t *testing.T) {
    // Arrange
    ctx := context.Background()
    mockRepo := new(testutil.MockOrderRepository)
    mockPayment := new(testutil.MockPaymentService)
    svc := service.NewOrderService(mockRepo, mockPayment)

    orderReq := &models.CreateOrderRequest{
        CustomerID:   "customer-123",
        RestaurantID: "restaurant-456",
        Items: []models.OrderItemRequest{
            {MenuItemID: "item-1", Quantity: 2},
        },
        PickupTime: time.Now().Add(1 * time.Hour),
    }

    // Mock payment authorization
    mockPayment.On("AuthorizePayment", mock.Anything, mock.Anything).
        Return(&models.PaymentTransaction{
            ID:     "payment-789",
            Status: "authorized",
            Amount: 25.00,
        }, nil)

    // Mock order creation
    mockRepo.On("Create", mock.Anything, mock.AnythingOfType("*models.Order")).
        Return(nil)

    // Act
    order, err := svc.CreateOrder(ctx, orderReq)

    // Assert
    assert.NoError(t, err)
    assert.NotNil(t, order)
    assert.Equal(t, "placed", order.State)
    assert.Equal(t, "authorized", order.Payment.Status)
    mockPayment.AssertExpectations(t)
    mockRepo.AssertExpectations(t)
}

// Run test - it will FAIL (RED) because OrderService doesn't exist yet
```

**Run the test:**
```bash
go test ./internal/service -v -run TestOrderService_CreateOrder_Success
```

**Expected output:** `FAIL` - This is good! (RED phase)

---

#### Step 2: Implement Minimum Code to Pass (GREEN)

Create `internal/service/order_service.go`:

```go
package service

import (
    "context"
    "fmt"
    "time"

    "github.com/google/uuid"
    "github.com/yourusername/restaurant-pickup-platform/internal/models"
    "github.com/yourusername/restaurant-pickup-platform/internal/repository"
)

// OrderService handles order business logic
type OrderService struct {
    orderRepo      repository.OrderRepository
    paymentService PaymentService
}

// NewOrderService creates a new OrderService
// Per Constitution Principle VII: Dependency injection via interfaces
func NewOrderService(orderRepo repository.OrderRepository, paymentService PaymentService) *OrderService {
    if orderRepo == nil || paymentService == nil {
        panic("dependencies cannot be nil") // Validate injected dependencies
    }
    return &OrderService{
        orderRepo:      orderRepo,
        paymentService: paymentService,
    }
}

// CreateOrder creates a new order with payment authorization
// Per Constitution Principle II: This function has 100% test coverage
func (s *OrderService) CreateOrder(ctx context.Context, req *models.CreateOrderRequest) (*models.Order, error) {
    // Step 1: Authorize payment (FR-018, FR-019)
    payment, err := s.paymentService.AuthorizePayment(ctx, &models.PaymentRequest{
        Amount:       req.TotalAmount,
        CustomerID:   req.CustomerID,
        PaymentToken: req.PaymentToken,
    })
    if err != nil {
        return nil, fmt.Errorf("payment authorization failed: %w", err)
    }

    // Step 2: Create order in "placed" state
    order := &models.Order{
        ID:               uuid.New().String(),
        OrderNumber:      generateOrderNumber(),
        CustomerID:       req.CustomerID,
        RestaurantID:     req.RestaurantID,
        State:            "placed", // Initial state per FR-029
        Version:          1,         // Optimistic locking per FR-030
        PickupTime:       req.PickupTime,
        TotalAmount:      req.TotalAmount,
        Payment:          payment,
        ServerReceivedAt: time.Now(), // For 5-min cancellation window per FR-034
    }

    // Step 3: Persist order (within transaction per Constitution Principle IV)
    if err := s.orderRepo.Create(ctx, order); err != nil {
        // Rollback payment authorization if order creation fails
        s.paymentService.ReleasePayment(ctx, payment.ID)
        return nil, fmt.Errorf("failed to create order: %w", err)
    }

    return order, nil
}

func generateOrderNumber() string {
    return fmt.Sprintf("ORD%s%05d", time.Now().Format("20060102"), time.Now().Unix()%100000)
}
```

**Run the test again:**
```bash
go test ./internal/service -v -run TestOrderService_CreateOrder_Success
```

**Expected output:** `PASS` (GREEN phase)

---

#### Step 3: Refactor and Add More Tests (REFACTOR)

Add edge case tests:

```go
func TestOrderService_CreateOrder_PaymentFailed(t *testing.T) {
    // Test payment authorization failure
    ctx := context.Background()
    mockRepo := new(testutil.MockOrderRepository)
    mockPayment := new(testutil.MockPaymentService)
    svc := service.NewOrderService(mockRepo, mockPayment)

    orderReq := &models.CreateOrderRequest{
        CustomerID:   "customer-123",
        RestaurantID: "restaurant-456",
        Items:        []models.OrderItemRequest{{MenuItemID: "item-1", Quantity: 2}},
        PickupTime:   time.Now().Add(1 * time.Hour),
    }

    // Mock payment failure
    mockPayment.On("AuthorizePayment", mock.Anything, mock.Anything).
        Return(nil, fmt.Errorf("card declined"))

    // Act
    order, err := svc.CreateOrder(ctx, orderReq)

    // Assert
    assert.Error(t, err)
    assert.Nil(t, order)
    assert.Contains(t, err.Error(), "payment authorization failed")

    // Order should NOT be created if payment fails
    mockRepo.AssertNotCalled(t, "Create")
}

// Add more tests for:
// - Pickup time in the past
// - Invalid restaurant ID
// - Slot capacity exceeded
// - Optimistic locking conflicts
```

**Run all tests with coverage:**
```bash
go test ./internal/service -v -coverprofile=coverage.out
go tool cover -html=coverage.out
```

**Target: 100% coverage for payment flows** (Constitution requirement)

---

## Common Development Commands

Create a `Makefile` for frequently used commands:

```makefile
.PHONY: build test run migrate-up migrate-down lint clean

# Build application
build:
	go build -o bin/server cmd/server/main.go

# Run all tests
test:
	go test ./... -v -cover

# Run tests with coverage report
test-coverage:
	go test ./... -coverprofile=coverage.out
	go tool cover -html=coverage.out

# Run only unit tests (co-located with code)
test-unit:
	go test ./internal/... ./pkg/... -v

# Run integration tests
test-integration:
	docker-compose -f tests/testutil/docker-compose.test.yml up -d
	go test ./tests/integration/... -tags=integration -v
	docker-compose -f tests/testutil/docker-compose.test.yml down

# Run E2E tests
test-e2e:
	go test ./tests/e2e/... -tags=e2e -v

# Run application
run:
	go run cmd/server/main.go

# Run with live reload (install air: go install github.com/cosmtrek/air@latest)
dev:
	air

# Database migrations
migrate-up:
	migrate -path ./migrations -database "postgresql://postgres:postgres@localhost:5432/restaurant_pickup?sslmode=disable" up

migrate-down:
	migrate -path ./migrations -database "postgresql://postgres:postgres@localhost:5432/restaurant_pickup?sslmode=disable" down 1

migrate-create:
	migrate create -ext sql -dir migrations -seq $(name)

# Code quality
lint:
	golangci-lint run

fmt:
	go fmt ./...

# Generate mocks (install mockery: go install github.com/vektra/mockery/v2@latest)
mocks:
	mockery --all --output=internal/testutil/mocks

# Clean build artifacts
clean:
	rm -rf bin/
	rm -f coverage.out

# Setup development environment
setup:
	docker-compose up -d
	make migrate-up
	@echo "Development environment ready!"
```

**Usage:**
```bash
make test           # Run all tests
make test-coverage  # View coverage report
make run            # Start server
make migrate-up     # Run database migrations
make setup          # Initialize dev environment
```

---

## API Development Workflow (API-First!)

Per Constitution Principle I, **OpenAPI spec comes BEFORE implementation**:

### 1. Start with OpenAPI Spec

Open `docs/api/openapi.yaml` and find the endpoint you're implementing:

```yaml
/orders:
  post:
    summary: Create new order
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/CreateOrderRequest'
    responses:
      '201':
        description: Order created successfully
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/Order'
```

### 2. Generate Contract Tests

```bash
# Install OpenAPI validator
npm install -g @stoplight/spectral-cli

# Validate OpenAPI spec
spectral lint docs/api/openapi.yaml

# Generate test fixtures from examples
# (You can use tools like openapi-mock-generator)
```

### 3. Implement Handler (TDD)

Write handler test first:

```go
// internal/handler/order_handler_test.go
func TestOrderHandler_CreateOrder_Returns201(t *testing.T) {
    // Setup
    gin.SetMode(gin.TestMode)
    mockService := new(testutil.MockOrderService)
    handler := NewOrderHandler(mockService)
    router := gin.Default()
    router.POST("/api/v1/orders", handler.CreateOrder)

    // Mock service response
    mockService.On("CreateOrder", mock.Anything, mock.Anything).
        Return(&models.Order{ID: "order-123", State: "placed"}, nil)

    // Create request matching OpenAPI spec
    requestBody := `{
        "restaurant_id": "rest-123",
        "pickup_time": "2025-11-11T15:00:00Z",
        "items": [{"menu_item_id": "item-1", "quantity": 2}],
        "payment_method": {"type": "card", "token": "tok_visa"}
    }`

    req, _ := http.NewRequest(http.MethodPost, "/api/v1/orders", strings.NewReader(requestBody))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer valid-token")

    w := httptest.NewRecorder()
    router.ServeHTTP(w, req)

    // Assert response matches OpenAPI spec
    assert.Equal(t, http.StatusCreated, w.Code)
    var response models.Order
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "order-123", response.ID)
    assert.Equal(t, "placed", response.State)
}
```

Then implement handler:

```go
// internal/handler/order_handler.go
func (h *OrderHandler) CreateOrder(c *gin.Context) {
    var req models.CreateOrderRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{"error": gin.H{"code": "ERR_INVALID_REQUEST", "message": err.Error()}})
        return
    }

    // Get user from auth middleware
    userID := c.GetString("user_id")
    req.CustomerID = userID

    order, err := h.orderService.CreateOrder(c.Request.Context(), &req)
    if err != nil {
        // Per Constitution Principle III: Standard error format
        c.JSON(500, gin.H{"error": gin.H{
            "code":           "ERR_ORDER_CREATION_FAILED",
            "message":        err.Error(),
            "correlation_id": c.GetString("correlation_id"),
        }})
        return
    }

    c.JSON(201, order)
}
```

---

## Testing Philosophy

### Unit Tests (Co-located with Code)
- **Location**: `internal/service/order_service_test.go` next to `order_service.go`
- **Purpose**: Test business logic in isolation
- **Use mocks**: For dependencies (repositories, external services)
- **Coverage target**: 100% for payment flows, 80%+ for others

### Integration Tests (Separate Directory)
- **Location**: `tests/integration/`
- **Purpose**: Test with real dependencies (database, Redis)
- **Setup**: Use docker-compose for test infrastructure
- **Tag**: `// +build integration`

### E2E Tests (Separate Directory)
- **Location**: `tests/e2e/`
- **Purpose**: Test complete user flows
- **Example**: Order placement → Restaurant acceptance → Payment capture → Pickup
- **Tag**: `// +build e2e`

---

## Common Pitfalls & Solutions

### ❌ Problem: "panic: dependencies cannot be nil"
**Solution**: Always inject dependencies via constructors:
```go
// Wrong
svc := &OrderService{}

// Correct (per Constitution Principle VII)
svc := service.NewOrderService(orderRepo, paymentService)
```

### ❌ Problem: "optimistic lock conflict" in tests
**Solution**: Mock version field properly:
```go
mockRepo.On("UpdateOrderState", mock.Anything, "order-123", 1, "confirmed").
    Return(nil) // Version 1 expected
```

### ❌ Problem: Tests fail with "connection refused"
**Solution**: Ensure test database is running:
```bash
docker-compose -f tests/testutil/docker-compose.test.yml up -d
```

### ❌ Problem: Payment tests fail with "no such host"
**Solution**: Don't hit real Stripe in unit tests - use mocks:
```go
mockPayment := new(testutil.MockPaymentService)
// Not: stripe.Client{} in unit tests
```

---

## Next Steps

1. **Read the Constitution**: `/Users/allenliao/Desktop/code/go-plate/.specify/memory/constitution.md`
2. **Review the Spec**: `specs/001-restaurant-pickup-platform/spec.md`
3. **Study the Data Model**: `specs/001-restaurant-pickup-platform/data-model.md`
4. **Explore OpenAPI Spec**: `specs/001-restaurant-pickup-platform/contracts/openapi.yaml`
5. **Start with User Story 1** (Priority P1): Customer Places Self-Pickup Order
6. **Follow TDD**: Red → Green → Refactor

---

## Resources

- **Go Documentation**: https://golang.org/doc/
- **Gin Framework**: https://github.com/gin-gonic/gin
- **XORM**: https://xorm.io/
- **Testify**: https://github.com/stretchr/testify
- **Stripe Go SDK**: https://github.com/stripe/stripe-go
- **golang-migrate**: https://github.com/golang-migrate/migrate
- **OpenAPI 3.0**: https://swagger.io/specification/

---

## Getting Help

- **Code Issues**: Create an issue on GitHub
- **Architecture Questions**: Review constitution and plan documents
- **Testing Help**: Check existing test examples in `/internal/service/*_test.go`

Happy coding! Remember: **Tests first, implementation second** (Constitution Principle II).
