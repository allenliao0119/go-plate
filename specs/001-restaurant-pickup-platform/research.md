# Research: Restaurant Self-Pickup Ordering Platform

**Branch**: `001-restaurant-pickup-platform` | **Date**: 2025-11-11

## Overview

This document consolidates research findings for technology choices and best practices for implementing the restaurant pickup platform. All decisions align with the project constitution's principles (API-First, TDD, Error Handling, Transactions, Code Organization, Documentation, Dependency Injection).

---

## 1. Payment Processing: Stripe Auth/Capture Flow in Go

### Decision
Use Stripe Go SDK with Payment Intents API for authorization-then-capture flow.

### Rationale
- **Native auth/capture support**: Stripe Payment Intents API provides explicit `authorize` (payment intent creation) and `capture` (payment intent capture) methods matching FR-018 to FR-021
- **PCI DSS compliance**: Stripe handles sensitive card data via tokenization, keeping our application out of PCI scope (NFR-012)
- **Idempotency built-in**: Stripe supports idempotency keys natively, critical for preventing duplicate charges during retries (Constitution Principle IV)
- **Well-documented Go SDK**: Official `stripe-go` library with comprehensive examples and active maintenance
- **24-hour hold period**: Stripe authorizations are valid for 7 days by default, exceeding our 10-minute acceptance window requirement

### Implementation Pattern
```go
// Authorization at checkout (FR-018)
params := &stripe.PaymentIntentParams{
    Amount: stripe.Int64(orderTotal),
    Currency: stripe.String(string(stripe.CurrencyUSD)),
    CaptureMethod: stripe.String("manual"), // Auth only, no capture
    Customer: stripe.String(customerID),
}
params.SetIdempotencyKey(orderID) // Prevent duplicate authorizations
pi, err := paymentintent.New(params)

// Capture when restaurant accepts (FR-020)
captureParams := &stripe.PaymentIntentCaptureParams{}
captureParams.SetIdempotencyKey(fmt.Sprintf("%s-capture", orderID))
pi, err := paymentintent.Capture(pi.ID, captureParams)

// Cancel/release if rejected or timeout (FR-021)
cancelParams := &stripe.PaymentIntentCancelParams{}
pi, err := paymentintent.Cancel(pi.ID, cancelParams)
```

### Alternatives Considered
- **PayPal**: Less suitable for auth/capture flow, more complex integration for split timing
- **Braintree**: Good alternative but Stripe has better Go SDK documentation and community support
- **In-house payment processing**: Rejected due to PCI compliance costs ($50k-500k+) and security risks

### Testing Strategy (per Constitution Principle II)
- Unit tests with mocked Stripe client interface (100% branch coverage for payment flows)
- Integration tests using Stripe test mode with test card tokens
- Test scenarios: successful auth, failed auth, timeout before capture, capture success, capture failure, idempotency key collision

### References
- Stripe Payment Intents: https://stripe.com/docs/payments/payment-intents
- stripe-go SDK: https://github.com/stripe/stripe-go
- Auth and Capture: https://stripe.com/docs/payments/capture-later

---

## 2. Real-Time Notifications Architecture

### Decision
Hybrid approach: Server-Sent Events (SSE) for in-app order status updates + Third-party services (SendGrid for email, Twilio for SMS) for out-of-app notifications.

### Rationale
**For in-app real-time updates (order status)**:
- **SSE over WebSocket**: Simpler implementation for unidirectional server→client updates, which is our primary use case (restaurant notifying customer of status changes)
- **HTTP/2 multiplexing**: SSE works over standard HTTP, easier to deploy behind proxies and load balancers
- **Automatic reconnection**: Browsers handle reconnection automatically with EventSource API
- **Lower complexity**: No need for WebSocket connection management, handshake protocols, or bidirectional message handling
- **NFR-007 compliance**: SSE can easily deliver notifications within 10-second requirement

**For out-of-app notifications**:
- **SendGrid (Email)**: Reliable SMTP service with API, transactional email templates, delivery tracking
- **Twilio (SMS)**: Industry standard for SMS with excellent Go SDK, delivery receipts, international support
- **Future: Firebase Cloud Messaging (Push)**: For mobile app push notifications when iOS/Android apps are developed

### Implementation Pattern
```go
// SSE endpoint for order status updates
func (h *OrderHandler) StreamOrderStatus(c *gin.Context) {
    orderID := c.Param("id")
    c.Header("Content-Type", "text/event-stream")
    c.Header("Cache-Control", "no-cache")
    c.Header("Connection", "keep-alive")

    // Subscribe to order status changes
    updates := h.orderService.SubscribeToOrder(orderID)

    for update := range updates {
        c.SSEvent("status", update)
        c.Writer.Flush()
    }
}

// Email notification via SendGrid
func (ns *NotificationService) SendEmailNotification(to, subject, body string) error {
    from := mail.NewEmail("Restaurant Platform", "noreply@platform.com")
    toEmail := mail.NewEmail("", to)
    message := mail.NewSingleEmail(from, subject, toEmail, body, body)

    response, err := ns.sendgridClient.Send(message)
    // Handle response...
}
```

### Alternatives Considered
- **WebSocket**: More complex, bidirectional communication not needed for status updates
- **Long polling**: Inefficient, higher server load, not truly real-time
- **AWS SNS/SQS**: Over-engineered for MVP scale (100 concurrent users), adds infrastructure complexity
- **Self-hosted SMTP**: Rejected due to deliverability issues, spam management overhead

### Performance Considerations
- SSE connection pooling: Limit max concurrent SSE connections per server to prevent resource exhaustion
- Close SSE connections after order reaches terminal state (completed, cancelled, no-show)
- SendGrid/Twilio rate limiting: Implement retry logic with exponential backoff

### References
- SSE with Go/Gin: https://github.com/gin-contrib/sse
- SendGrid Go SDK: https://github.com/sendgrid/sendgrid-go
- Twilio Go SDK: https://github.com/twilio/twilio-go

---

## 3. XORM Configuration for PostgreSQL Connection Pooling

### Decision
Use XORM with optimized connection pool settings for 100 concurrent users (MVP) with capacity for 10x growth.

### Rationale
- **XORM advantages**: ORM specified in tech stack, provides good balance of type safety and raw SQL access
- **Connection pooling**: XORM uses `database/sql` underneath, which provides built-in connection pooling
- **Performance**: Properly tuned pool prevents connection exhaustion under load while minimizing idle connections

### Configuration
```go
import (
    "xorm.io/xorm"
    _ "github.com/lib/pq" // PostgreSQL driver
)

func NewDatabase(connString string) (*xorm.Engine, error) {
    engine, err := xorm.NewEngine("postgres", connString)
    if err != nil {
        return nil, err
    }

    // Connection pool settings for 100 concurrent users
    engine.SetMaxOpenConns(25)        // Max connections (100 users / 4 avg req/user)
    engine.SetMaxIdleConns(10)        // Idle connections for quick reuse
    engine.SetConnMaxLifetime(300)    // 5 minutes max connection lifetime
    engine.SetConnMaxIdleTime(60)     // 1 minute max idle time

    // For 10x growth (1000 concurrent users)
    // engine.SetMaxOpenConns(250)
    // engine.SetMaxIdleConns(50)

    // Logging for development (disable in production)
    engine.ShowSQL(true)
    engine.Logger().SetLevel(core.LOG_DEBUG)

    // Caching (Constitution: complexity justified by NFR-005)
    cacher := xorm.NewLRUCacher(xorm.NewMemoryStore(), 1000)
    engine.SetDefaultCacher(cacher)

    return engine, nil
}
```

### Calculation Rationale
- **MaxOpenConns (25)**: Based on 100 concurrent users with average 4 requests/second per active user = 400 req/s. Assuming 50ms avg query time, need ~20 connections. Set to 25 for headroom.
- **MaxIdleConns (10)**: 40% of max open connections for rapid connection reuse during burst traffic
- **Connection lifetime limits**: Prevent stale connections, force periodic refresh for load balancing across DB replicas

### Monitoring & Tuning
- Track metrics: `db.Stats().OpenConnections`, `db.Stats().WaitCount`, `db.Stats().WaitDuration`
- Alert if `WaitCount` increases (indicates connection pool exhaustion)
- Tune `MaxOpenConns` based on actual production load patterns

### References
- XORM documentation: https://xorm.io/
- Go database/sql pooling: https://pkg.go.dev/database/sql#DB.SetMaxOpenConns
- PostgreSQL connection best practices: https://www.postgresql.org/docs/current/runtime-config-connection.html

---

## 4. Time-Based Task Scheduler: robfig/cron

### Decision
Use `robfig/cron` v3 for scheduled background tasks (auto-rejection, auto-approval, reminders).

### Rationale
- **Cron syntax familiarity**: Uses standard cron expressions, easy for ops teams to understand
- **Type-safe job scheduling**: Go functions as jobs, compile-time safety
- **Built-in panic recovery**: Jobs that panic don't crash entire scheduler
- **Lightweight**: No external dependencies (Redis, RabbitMQ), suitable for MVP scale
- **Timezone support**: Important for pickup time reminders across regions

### Implementation Pattern
```go
import "github.com/robfig/cron/v3"

func SetupScheduler(orderService OrderService) *cron.Cron {
    c := cron.New(cron.WithSeconds()) // Enable second-level precision

    // Auto-reject orders after 10 minutes (FR-059)
    // Check every 30 seconds for orders pending > 10 minutes
    c.AddFunc("*/30 * * * * *", func() {
        orderService.AutoRejectTimedOutOrders()
    })

    // Auto-approve cancellations after 5 minutes (FR-037)
    // Check every 30 seconds for cancellation requests pending > 5 minutes
    c.AddFunc("*/30 * * * * *", func() {
        orderService.AutoApprovePendingCancellations()
    })

    // Send pickup reminders 15 minutes before scheduled time (FR-033)
    // Check every minute for orders due in 15-16 minutes
    c.AddFunc("0 * * * * *", func() {
        orderService.SendPickupReminders()
    })

    c.Start()
    return c
}
```

### Alternatives Considered
- **time.Ticker**: Too low-level, requires manual goroutine management and no cron syntax
- **Kubernetes CronJobs**: Over-engineered for MVP, requires separate deployment artifacts
- **Temporal/Cadence workflows**: Excellent for complex workflows but massive overkill for simple time-based triggers
- **Database polling in each request**: Rejected due to performance overhead and lack of guarantees

### Scaling Considerations
- **Single-instance assumption for MVP**: Only one scheduler instance should run (prevent duplicate job execution)
- **For 10x scale**: Add distributed locking (Redis SETNX or PostgreSQL advisory locks) before job execution
- **Job execution tracking**: Log job runs with timestamps for audit trail

### Error Handling
- Jobs should be idempotent (safe to run multiple times)
- Log errors but don't crash scheduler
- Implement alerting for consecutive job failures

### References
- robfig/cron: https://github.com/robfig/cron
- Cron expression syntax: https://crontab.guru/

---

## 5. Image Upload and Storage

### Decision
Phase 1 (MVP): Local filesystem storage with structured directories
Phase 2 (10x scale): Migrate to S3-compatible storage (AWS S3 or MinIO)

### Rationale
**Local filesystem for MVP**:
- **Simplicity**: No additional infrastructure dependencies (Constitution: avoid premature complexity)
- **Sufficient for 50 restaurants**: Assuming 10 images per restaurant (profile + menu items) × 2MB average = 1GB total storage, easily manageable locally
- **Fast access**: Local disk I/O faster than network calls for MVP scale
- **Easy backup**: Standard file backup tools

**S3-compatible for scale**:
- **Horizontal scaling**: Decouple storage from application servers
- **CDN integration**: CloudFront or similar for image caching and faster delivery
- **Durability**: 99.999999999% durability vs local disk RAID
- **Cost-effective at scale**: S3 pricing better than scaling local storage

### Implementation Pattern
```go
// Phase 1: Local filesystem
type LocalImageStore struct {
    basePath string // e.g., "/var/app/uploads"
}

func (s *LocalImageStore) SaveImage(file multipart.File, filename string, category string) (string, error) {
    // Organize: /uploads/{category}/{date}/{uuid}_{filename}
    date := time.Now().Format("2006-01-02")
    uuid := uuid.New().String()
    relPath := filepath.Join(category, date, fmt.Sprintf("%s_%s", uuid, filename))
    absPath := filepath.Join(s.basePath, relPath)

    // Ensure directory exists
    os.MkdirAll(filepath.Dir(absPath), 0755)

    // Write file
    dst, err := os.Create(absPath)
    if err != nil {
        return "", err
    }
    defer dst.Close()

    if _, err := io.Copy(dst, file); err != nil {
        return "", err
    }

    // Return URL path (served by Gin static file handler)
    return fmt.Sprintf("/uploads/%s", relPath), nil
}

// Phase 2: S3-compatible (interface stays same, swap implementation)
type S3ImageStore struct {
    s3Client *s3.Client
    bucket   string
}

func (s *S3ImageStore) SaveImage(file multipart.File, filename string, category string) (string, error) {
    // Implementation with AWS SDK or MinIO client
    // Returns CDN URL
}
```

### File Validation (Security)
- **Mime type checking**: Validate actual file content, not just extension
- **Size limits**: Max 5MB per image (prevent DoS via large uploads)
- **Image format restrictions**: JPEG, PNG, WebP only
- **Virus scanning**: For production, integrate ClamAV or similar

### Gin Static File Serving (MVP)
```go
router.Static("/uploads", "/var/app/uploads")
```

### Alternatives Considered
- **Cloudinary**: Third-party image CDN with transformations, but adds cost and external dependency
- **Database BLOB storage**: Poor performance, bloats database, complicates backups

### References
- AWS S3 Go SDK: https://github.com/aws/aws-sdk-go-v2
- MinIO Go Client: https://github.com/minio/minio-go
- Gin static files: https://github.com/gin-gonic/gin#serving-static-files

---

## 6. Gin Middleware Best Practices

### Decision
Implement layered middleware stack: Logger → Recovery → CORS → Auth → Rate Limiter → Error Handler

### Rationale
- **Order matters**: Middleware executes in registration order, recovery must be first to catch panics
- **Separation of concerns**: Each middleware handles one responsibility (Constitution Principle V)
- **Reusability**: Middleware can be selectively applied to route groups

### Implementation Pattern
```go
func SetupRouter(authService AuthService, rateLimiter RateLimiter) *gin.Engine {
    router := gin.New()

    // 1. Recovery (must be first to catch panics)
    router.Use(gin.Recovery())

    // 2. Request logging with correlation ID
    router.Use(middleware.Logger())

    // 3. CORS for frontend access
    router.Use(middleware.CORS())

    // 4. Global rate limiting (per NFR-015: 5 failed logins per 15 min)
    router.Use(middleware.RateLimiter(rateLimiter))

    // Public routes (no auth)
    public := router.Group("/api/v1")
    {
        public.POST("/auth/register", authHandler.Register)
        public.POST("/auth/login", authHandler.Login)
        public.GET("/restaurants", restaurantHandler.List) // Browse restaurants
    }

    // Protected routes (require authentication)
    protected := router.Group("/api/v1")
    protected.Use(middleware.AuthRequired(authService)) // JWT validation
    {
        protected.POST("/orders", orderHandler.CreateOrder)
        protected.GET("/orders/:id", orderHandler.GetOrder)
        protected.POST("/orders/:id/cancel", orderHandler.CancelOrder)
    }

    // Admin routes (require admin role)
    admin := router.Group("/api/v1/admin")
    admin.Use(middleware.AuthRequired(authService))
    admin.Use(middleware.RequireRole("admin"))
    {
        admin.GET("/restaurants/pending", adminHandler.ListPendingRestaurants)
        admin.POST("/restaurants/:id/approve", adminHandler.ApproveRestaurant)
    }

    // Error handler (last middleware, formats all errors)
    router.Use(middleware.ErrorHandler())

    return router
}
```

### Key Middleware Components

**1. Correlation ID Logger**
```go
func Logger() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Generate or extract correlation ID
        correlationID := c.GetHeader("X-Correlation-ID")
        if correlationID == "" {
            correlationID = uuid.New().String()
        }
        c.Set("correlation_id", correlationID)
        c.Header("X-Correlation-ID", correlationID)

        // Log request
        start := time.Now()
        c.Next()
        duration := time.Since(start)

        log.WithFields(log.Fields{
            "correlation_id": correlationID,
            "method":         c.Request.Method,
            "path":           c.Request.URL.Path,
            "status":         c.Writer.Status(),
            "duration_ms":    duration.Milliseconds(),
        }).Info("Request processed")
    }
}
```

**2. JWT Authentication**
```go
func AuthRequired(authService AuthService) gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(401, gin.H{
                "error": gin.H{
                    "code":    "ERR_UNAUTHORIZED",
                    "message": "Authorization header required",
                },
            })
            return
        }

        claims, err := authService.ValidateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{
                "error": gin.H{
                    "code":    "ERR_INVALID_TOKEN",
                    "message": "Invalid or expired token",
                },
            })
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("user_role", claims.Role)
        c.Next()
    }
}
```

**3. Error Handler (Constitution Principle III)**
```go
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()

        // Check if any errors occurred
        if len(c.Errors) > 0 {
            err := c.Errors.Last()

            // Map error types to HTTP status codes
            var statusCode int
            var errorCode string

            switch err.Type {
            case gin.ErrorTypeBind:
                statusCode = 400
                errorCode = "ERR_INVALID_REQUEST"
            case gin.ErrorTypePrivate:
                statusCode = 500
                errorCode = "ERR_INTERNAL_SERVER"
            default:
                statusCode = 500
                errorCode = "ERR_UNKNOWN"
            }

            correlationID, _ := c.Get("correlation_id")

            c.JSON(statusCode, gin.H{
                "error": gin.H{
                    "code":           errorCode,
                    "message":        err.Error(),
                    "correlation_id": correlationID,
                },
            })
        }
    }
}
```

### References
- Gin middleware guide: https://github.com/gin-gonic/gin#using-middleware
- JWT middleware: https://github.com/appleboy/gin-jwt
- CORS middleware: https://github.com/gin-contrib/cors

---

## 7. Database Migration Strategy with golang-migrate

### Decision
Use `golang-migrate` with SQL migration files, versioned in `/migrations/` directory.

### Rationale
- **Language-agnostic**: SQL migrations are portable, not tied to Go code
- **Bidirectional migrations**: Separate `.up.sql` and `.down.sql` for rollback capability
- **Version control**: Migration files committed to git, track schema evolution
- **Production-safe**: Locking mechanism prevents concurrent migrations
- **XORM integration**: Can use XORM for application queries while managing schema with pure SQL

### Migration File Structure
```
migrations/
├── 000001_create_users_table.up.sql
├── 000001_create_users_table.down.sql
├── 000002_create_restaurants_table.up.sql
├── 000002_create_restaurants_table.down.sql
...
```

### Example Migration (Users Table)
```sql
-- 000001_create_users_table.up.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    role VARCHAR(20) NOT NULL CHECK (role IN ('customer', 'restaurant_owner', 'admin')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);

-- 000001_create_users_table.down.sql
DROP TABLE IF EXISTS users CASCADE;
```

### Running Migrations
```bash
# Install golang-migrate
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Run migrations up
migrate -path ./migrations -database "postgresql://user:pass@localhost:5432/dbname?sslmode=disable" up

# Rollback last migration
migrate -path ./migrations -database "postgresql://user:pass@localhost:5432/dbname?sslmode=disable" down 1

# Check current version
migrate -path ./migrations -database "postgresql://user:pass@localhost:5432/dbname?sslmode=disable" version
```

### Integration with Application
```go
import (
    "github.com/golang-migrate/migrate/v4"
    _ "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
)

func RunMigrations(dbURL string) error {
    m, err := migrate.New(
        "file://migrations",
        dbURL,
    )
    if err != nil {
        return err
    }

    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }

    return nil
}
```

### Best Practices
- **Naming convention**: `{version}_{description}.{up|down}.sql` (zero-padded version numbers)
- **Idempotent migrations**: Use `IF NOT EXISTS` for `CREATE` statements when safe
- **Data migrations**: Separate schema changes from data changes when possible
- **Testing migrations**: Test both `up` and `down` in development before production
- **Locking**: golang-migrate uses advisory locks to prevent concurrent migrations

### Alternatives Considered
- **XORM AutoMigrate**: Less control, no versioning, no rollback capability
- **GORM migrations**: Tied to GORM ORM, not using GORM in this project
- **Goose**: Similar to golang-migrate but less popular, smaller community

### References
- golang-migrate: https://github.com/golang-migrate/migrate
- Migration best practices: https://github.com/golang-migrate/migrate/blob/master/MIGRATIONS.md

---

## Summary of Key Decisions

| Area | Technology | Justification |
|------|------------|---------------|
| **Payment Processing** | Stripe Payment Intents API | Native auth/capture support, PCI compliance, idempotency, 7-day hold period |
| **Real-Time Notifications** | SSE + SendGrid + Twilio | SSE for in-app (simpler than WebSocket), SendGrid/Twilio for email/SMS (reliable delivery) |
| **Database Pooling** | XORM with optimized pool settings | 25 max connections for 100 concurrent users, 10 idle connections, 5-min lifetime |
| **Task Scheduler** | robfig/cron v3 | Cron syntax familiarity, type-safe jobs, lightweight, panic recovery |
| **Image Storage** | Local filesystem (MVP) → S3 (scale) | Start simple per constitution, migrate to S3 for 10x growth and CDN |
| **Middleware Stack** | Gin layered middleware | Recovery → Logger → CORS → Auth → Rate Limiter → Error Handler |
| **Database Migrations** | golang-migrate with SQL files | Version control, bidirectional, production-safe locking, SQL portability |

All decisions align with the constitution's complexity constraints: start simple, justify complexity with requirements or measurements, prefer standard library and well-vetted third-party libraries.

---

**Next Phase**: Phase 1 - Design & Contracts (data-model.md, OpenAPI contracts, quickstart.md)
