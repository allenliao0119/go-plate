# Data Model: Restaurant Self-Pickup Ordering Platform

**Branch**: `001-restaurant-pickup-platform` | **Date**: 2025-11-11

## Overview

This document defines the entity relationships, database schema design, and validation rules for the restaurant pickup platform. The model is designed to support all functional requirements while maintaining data integrity through constraints, indexes, and proper normalization.

---

## Entity Relationship Diagram

```
┌─────────────┐         ┌──────────────────┐         ┌─────────────┐
│    User     │         │   Restaurant     │         │  MenuItem   │
│             │         │                  │         │             │
│ id (PK)     │         │ id (PK)          │◄────────│ id (PK)     │
│ email       │         │ name             │  1:N    │ restaurant_ │
│ password_   │         │ owner_id (FK) ───┼────┐    │   id (FK)   │
│   hash      │         │ cuisine_type     │    │    │ name        │
│ name        │         │ address          │    │    │ price       │
│ phone       │         │ rating_avg       │    │    │ category    │
│ role        │         │ operating_hours  │    │    │ available   │
└──────┬──────┘         └───────┬──────────┘    │    └─────────────┘
       │                        │               │
       │ owner_id               │               │
       │ (role=restaurant_owner)│               │
       │                        │               │
       │                        │ 1:N           │
       │                ┌───────┴───────┐       │
       │                │     Order     │       │
       │  1:N           │               │       │
       └───────────────►│ id (PK)       │       │
         customer_id    │ customer_id   │       │
                        │   (FK)        │       │
                        │ restaurant_id │       │
                        │   (FK) ───────┼───────┘
                        │ order_number  │
                        │ state         │
                        │ version       │◄──┐ optimistic
                        │ pickup_time   │   │   locking
                        │ total_amount  │   │
                        └───────┬───────┘   │
                                │           │
                                │ 1:N       │
                        ┌───────┴───────┐   │
                        │  OrderItem    │   │
                        │               │   │
                        │ id (PK)       │   │
                        │ order_id (FK) │   │
                        │ menu_item_id  │   │
                        │   (FK)        │   │
                        │ quantity      │   │
                        │ unit_price    │   │
                        │ customizations│   │
                        └───────────────┘   │
                                            │
                        ┌───────────────────┤
                        │ PaymentTransaction│
                        │                   │
                        │ id (PK)           │
                        │ order_id (FK) ────┘
                        │ stripe_payment_   │
                        │   intent_id       │
                        │ amount            │
                        │ status            │
                        │ idempotency_key   │
                        └───────────────────┘
```

---

## Entity Definitions

### 1. User

Represents all platform users (customers, restaurant owners, admins). Centralized user table with role-based differentiation.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `email` (VARCHAR(255), UNIQUE, NOT NULL): User email for login
- `password_hash` (VARCHAR(255), NOT NULL): Bcrypt hashed password
- `name` (VARCHAR(255), NOT NULL): Full name
- `phone` (VARCHAR(20)): Contact phone number
- `role` (VARCHAR(20), NOT NULL): User role - `customer`, `restaurant_owner`, `admin`
- `email_verified` (BOOLEAN, DEFAULT FALSE): Email verification status
- `created_at` (TIMESTAMP, DEFAULT NOW())
- `updated_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (role IN ('customer', 'restaurant_owner', 'admin'))`
- `CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')` (email format validation)
- `CHECK (LENGTH(password_hash) >= 60)` (bcrypt hash length)

**Indexes:**
- `idx_users_email` on `email` (frequent login lookups)
- `idx_users_role` on `role` (role-based queries)

**Validation Rules (Application Layer):**
- FR-014: Password must be min 8 chars, mixed case, numbers, special characters (NFR-014)
- FR-001: Email verification required before placing orders
- FR-003: Password reset via email token (implement separate `password_reset_tokens` table)

---

### 2. Restaurant

Represents food establishments offering pickup orders.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `owner_id` (UUID, FK → users.id, NOT NULL): Restaurant owner reference
- `name` (VARCHAR(255), NOT NULL): Restaurant name
- `description` (TEXT): Restaurant description
- `cuisine_type` (VARCHAR(100), NOT NULL): Cuisine category (e.g., "Italian", "Chinese")
- `address` (TEXT, NOT NULL): Physical address
- `phone` (VARCHAR(20), NOT NULL): Contact phone
- `latitude` (DECIMAL(10,8)): Geolocation for distance calculations
- `longitude` (DECIMAL(11,8)): Geolocation for distance calculations
- `operating_hours` (JSONB, NOT NULL): Weekly hours - `{"monday": {"open": "09:00", "close": "21:00"}, ...}`
- `special_closures` (JSONB): Holiday/special closure dates - `[{"date": "2025-12-25", "reason": "Christmas"}]`
- `rating_avg` (DECIMAL(3,2), DEFAULT 0.00): Average rating (0.00 to 5.00)
- `rating_count` (INTEGER, DEFAULT 0): Total number of ratings
- `min_preparation_time` (INTEGER, DEFAULT 30): Minimum prep time in minutes
- `slot_duration` (INTEGER, DEFAULT 15): Pickup slot duration in minutes (FR-062)
- `slot_capacity` (INTEGER, DEFAULT 5): Max orders per slot (FR-063)
- `is_active` (BOOLEAN, DEFAULT TRUE): Restaurant active status
- `is_approved` (BOOLEAN, DEFAULT FALSE): Admin approval status (FR-078, FR-079)
- `created_at` (TIMESTAMP, DEFAULT NOW())
- `updated_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (rating_avg >= 0.00 AND rating_avg <= 5.00)`
- `CHECK (min_preparation_time >= 15)` (reasonable minimum)
- `CHECK (slot_duration >= 5 AND slot_duration <= 60)`
- `CHECK (slot_capacity >= 1 AND slot_capacity <= 100)`
- `FOREIGN KEY (owner_id) REFERENCES users(id) ON DELETE RESTRICT`

**Indexes:**
- `idx_restaurants_cuisine` on `cuisine_type` (filter by cuisine)
- `idx_restaurants_location` on `latitude, longitude` (geographic search)
- `idx_restaurants_rating` on `rating_avg` (sort by rating)
- `idx_restaurants_owner` on `owner_id` (owner's restaurants)
- `idx_restaurants_active_approved` on `is_active, is_approved` (public listing queries)

**Validation Rules:**
- FR-048: Operating hours must be valid time ranges
- FR-049: Special closures must have future dates
- FR-062, FR-063: Slot configuration must be consistent

---

### 3. MenuItem

Represents menu items offered by restaurants.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `restaurant_id` (UUID, FK → restaurants.id, NOT NULL): Parent restaurant
- `name` (VARCHAR(255), NOT NULL): Item name
- `description` (TEXT): Item description
- `price` (DECIMAL(10,2), NOT NULL): Base price in cents (e.g., 1295 = $12.95)
- `category` (VARCHAR(100), NOT NULL): Menu category (e.g., "Appetizers", "Entrees")
- `preparation_time` (INTEGER): Estimated prep time in minutes
- `image_url` (VARCHAR(500)): Image file path or URL
- `dietary_info` (JSONB): Dietary tags - `{"vegetarian": true, "vegan": false, "gluten_free": true, "allergens": ["nuts", "dairy"]}`
- `availability_status` (VARCHAR(20), DEFAULT 'available'): `available`, `sold_out`, `seasonal`
- `customization_options` (JSONB): Available customizations - `[{"name": "Size", "choices": [{"label": "Small", "price_modifier": 0}, {"label": "Large", "price_modifier": 200}]}]`
- `is_active` (BOOLEAN, DEFAULT TRUE): Soft delete flag
- `display_order` (INTEGER, DEFAULT 0): Ordering within category
- `created_at` (TIMESTAMP, DEFAULT NOW())
- `updated_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (price > 0)` (price must be positive)
- `CHECK (availability_status IN ('available', 'sold_out', 'seasonal'))`
- `FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE`

**Indexes:**
- `idx_menu_items_restaurant` on `restaurant_id` (list restaurant menu)
- `idx_menu_items_category` on `category` (category filtering)
- `idx_menu_items_display_order` on `restaurant_id, display_order` (ordered menu display)

**Validation Rules:**
- FR-051: All dietary information must be accurate
- FR-054: Customization options must have valid price modifiers

---

### 4. Order

Represents customer purchase transactions. This is the central entity for order management.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `order_number` (VARCHAR(12), UNIQUE, NOT NULL): Human-readable order number (e.g., "ORD20251111001")
- `customer_id` (UUID, FK → users.id, NOT NULL): Customer reference
- `restaurant_id` (UUID, FK → restaurants.id, NOT NULL): Restaurant reference
- `state` (VARCHAR(20), NOT NULL): Order state - `placed`, `confirmed`, `preparing`, `ready`, `completed`, `cancelled`, `no_show`
- `version` (INTEGER, NOT NULL, DEFAULT 1): Optimistic locking version (FR-030)
- `pickup_time` (TIMESTAMP, NOT NULL): Scheduled pickup time
- `pickup_slot_id` (UUID, FK → pickup_slots.id): Associated time slot
- `total_amount` (DECIMAL(10,2), NOT NULL): Total order amount including taxes/fees
- `subtotal` (DECIMAL(10,2), NOT NULL): Items subtotal before taxes
- `tax_amount` (DECIMAL(10,2), NOT NULL): Tax amount
- `service_fee` (DECIMAL(10,2), DEFAULT 0.00): Platform service fee
- `customer_notes` (TEXT): Special instructions from customer (FR-010)
- `cancellation_reason` (TEXT): Reason if cancelled
- `cancellation_requested_at` (TIMESTAMP): When cancellation was requested (FR-035)
- `server_received_at` (TIMESTAMP, NOT NULL, DEFAULT NOW()): Server timestamp for grace buffer (FR-034)
- `created_at` (TIMESTAMP, DEFAULT NOW())
- `updated_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (state IN ('placed', 'confirmed', 'preparing', 'ready', 'completed', 'cancelled', 'no_show'))`
- `CHECK (total_amount > 0)`
- `CHECK (pickup_time > created_at)` (pickup time must be in future)
- `CHECK (version >= 1)` (version starts at 1)
- `FOREIGN KEY (customer_id) REFERENCES users(id) ON DELETE RESTRICT`
- `FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE RESTRICT`
- `FOREIGN KEY (pickup_slot_id) REFERENCES pickup_slots(id) ON DELETE SET NULL`

**Indexes:**
- `idx_orders_customer` on `customer_id` (customer order history)
- `idx_orders_restaurant` on `restaurant_id` (restaurant order queue)
- `idx_orders_state` on `state` (filter by state)
- `idx_orders_pickup_time` on `pickup_time` (time-based queries for reminders)
- `idx_orders_number` on `order_number` (lookup by order number)
- `idx_orders_version` on `id, version` (optimistic locking checks)

**State Transition Rules (FR-029):**
- `placed` → `confirmed` (restaurant accepts)
- `placed` → `cancelled` (customer cancels within 5 min or restaurant rejects)
- `confirmed` → `preparing` (restaurant starts preparation)
- `preparing` → `ready` (food ready for pickup)
- `ready` → `completed` (customer picks up)
- `ready` → `no_show` (customer doesn't pick up after grace period)
- **NO backward transitions allowed**

**Validation Rules:**
- FR-034: Cancellation within 5 minutes uses `server_received_at` with 30-second grace buffer
- FR-030: Optimistic locking - increment `version` on every state change, check current version matches expected

---

### 5. OrderItem

Represents individual menu items within an order.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `order_id` (UUID, FK → orders.id, NOT NULL): Parent order
- `menu_item_id` (UUID, FK → menu_items.id, NOT NULL): Menu item reference
- `quantity` (INTEGER, NOT NULL): Number of items ordered
- `unit_price` (DECIMAL(10,2), NOT NULL): Price per item at time of order (snapshot)
- `customizations` (JSONB): Selected customizations - `{"Size": "Large", "Spice Level": "Medium"}`
- `special_instructions` (TEXT): Item-specific instructions
- `subtotal` (DECIMAL(10,2), NOT NULL): `quantity * unit_price + customization_modifiers`

**Constraints:**
- `CHECK (quantity > 0)`
- `CHECK (unit_price > 0)`
- `CHECK (subtotal = quantity * unit_price)` (maintain consistency)
- `FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE`
- `FOREIGN KEY (menu_item_id) REFERENCES menu_items(id) ON DELETE RESTRICT`

**Indexes:**
- `idx_order_items_order` on `order_id` (load order items)
- `idx_order_items_menu_item` on `menu_item_id` (popular items analytics)

**Validation Rules:**
- FR-010: Customizations must match available options from `menu_items.customization_options`
- Snapshot `unit_price` at order time to preserve pricing even if menu prices change later

---

### 6. PickupSlot

Represents available time slots for order pickups. Helps enforce capacity limits (FR-063, FR-064).

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `restaurant_id` (UUID, FK → restaurants.id, NOT NULL): Restaurant reference
- `slot_time` (TIMESTAMP, NOT NULL): Slot start time
- `slot_duration` (INTEGER, NOT NULL): Duration in minutes (from restaurant.slot_duration)
- `max_capacity` (INTEGER, NOT NULL): Max orders for this slot (from restaurant.slot_capacity)
- `current_bookings` (INTEGER, DEFAULT 0): Current number of orders in this slot
- `is_available` (BOOLEAN, DEFAULT TRUE): Slot availability

**Constraints:**
- `CHECK (current_bookings >= 0 AND current_bookings <= max_capacity)`
- `CHECK (slot_duration > 0)`
- `FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE`
- `UNIQUE (restaurant_id, slot_time)` (prevent duplicate slots)

**Indexes:**
- `idx_pickup_slots_restaurant_time` on `restaurant_id, slot_time` (lookup available slots)
- `idx_pickup_slots_available` on `is_available, current_bookings` (filter available slots)

**Validation Rules:**
- FR-064: Prevent booking if `current_bookings >= max_capacity`
- FR-065: Calculate earliest available slot as `NOW() + restaurant.min_preparation_time`

---

### 7. PaymentTransaction

Represents payment transactions for orders. Tracks Stripe Payment Intent lifecycle.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `order_id` (UUID, FK → orders.id, NOT NULL, UNIQUE): Associated order (1:1 relationship)
- `stripe_payment_intent_id` (VARCHAR(255), UNIQUE, NOT NULL): Stripe Payment Intent ID
- `amount` (DECIMAL(10,2), NOT NULL): Transaction amount
- `currency` (VARCHAR(3), DEFAULT 'USD'): Currency code
- `status` (VARCHAR(20), NOT NULL): Transaction status - `authorized`, `captured`, `released`, `failed`
- `payment_method` (VARCHAR(50)): Payment method type (e.g., "card", "apple_pay")
- `idempotency_key` (VARCHAR(255), UNIQUE, NOT NULL): Idempotency key for Stripe API
- `error_code` (VARCHAR(100)): Error code if transaction failed
- `error_message` (TEXT): Error description
- `authorized_at` (TIMESTAMP): When payment was authorized (FR-018)
- `captured_at` (TIMESTAMP): When payment was captured (FR-020)
- `released_at` (TIMESTAMP): When authorization was released (FR-021)
- `created_at` (TIMESTAMP, DEFAULT NOW())
- `updated_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (status IN ('authorized', 'captured', 'released', 'failed'))`
- `CHECK (amount > 0)`
- `CHECK (currency IN ('USD', 'CAD', 'EUR'))` (expand as needed)
- `FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE RESTRICT`

**Indexes:**
- `idx_payment_transactions_order` on `order_id` (lookup order payment)
- `idx_payment_transactions_stripe` on `stripe_payment_intent_id` (Stripe webhook lookups)
- `idx_payment_transactions_idempotency` on `idempotency_key` (prevent duplicates)

**Validation Rules:**
- FR-018, FR-019: Create transaction with `status='authorized'` on successful checkout
- FR-020: Update to `status='captured'` when restaurant accepts order
- FR-021: Update to `status='released'` if order rejected or auto-rejected
- Idempotency key format: `order-{order_id}-auth` for authorization, `order-{order_id}-capture` for capture

---

### 8. Review

Represents customer feedback on completed orders.

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `order_id` (UUID, FK → orders.id, NOT NULL, UNIQUE): Associated order (1:1 relationship, FR-070)
- `customer_id` (UUID, FK → users.id, NOT NULL): Reviewer reference
- `restaurant_id` (UUID, FK → restaurants.id, NOT NULL): Reviewed restaurant
- `rating` (INTEGER, NOT NULL): Rating from 1 to 5 stars
- `review_text` (VARCHAR(500)): Optional review text (FR-067: max 500 chars)
- `created_at` (TIMESTAMP, DEFAULT NOW())
- `updated_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (rating >= 1 AND rating <= 5)`
- `CHECK (LENGTH(review_text) <= 500)`
- `FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE`
- `FOREIGN KEY (customer_id) REFERENCES users(id) ON DELETE CASCADE`
- `FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE`

**Indexes:**
- `idx_reviews_restaurant` on `restaurant_id` (list restaurant reviews)
- `idx_reviews_customer` on `customer_id` (user's reviews)
- `idx_reviews_order` on `order_id` (check if order already reviewed)

**Validation Rules:**
- FR-066: Only allow rating after order state is `completed`
- FR-070: Unique constraint on `order_id` prevents multiple ratings for same order
- FR-068: Update `restaurants.rating_avg` and `restaurants.rating_count` via trigger or application logic

---

### 9. Notification

Represents notifications sent to users (email, SMS, push).

**Attributes:**
- `id` (UUID, PK): Unique identifier
- `recipient_id` (UUID, FK → users.id, NOT NULL): Recipient user
- `notification_type` (VARCHAR(50), NOT NULL): Type - `order_confirmed`, `order_ready`, `pickup_reminder`, etc.
- `channel` (VARCHAR(20), NOT NULL): Delivery channel - `email`, `sms`, `push`
- `subject` (VARCHAR(255)): Email subject or push title
- `body` (TEXT, NOT NULL): Notification content
- `related_order_id` (UUID, FK → orders.id): Associated order (if applicable)
- `sent_at` (TIMESTAMP): When notification was sent
- `delivered_at` (TIMESTAMP): When delivery was confirmed (if tracking available)
- `status` (VARCHAR(20), DEFAULT 'pending'): Status - `pending`, `sent`, `delivered`, `failed`
- `error_message` (TEXT): Error details if failed
- `created_at` (TIMESTAMP, DEFAULT NOW())

**Constraints:**
- `CHECK (channel IN ('email', 'sms', 'push'))`
- `CHECK (status IN ('pending', 'sent', 'delivered', 'failed'))`
- `FOREIGN KEY (recipient_id) REFERENCES users(id) ON DELETE CASCADE`
- `FOREIGN KEY (related_order_id) REFERENCES orders(id) ON DELETE SET NULL`

**Indexes:**
- `idx_notifications_recipient` on `recipient_id` (user notification history)
- `idx_notifications_status` on `status` (pending notifications queue)
- `idx_notifications_sent_at` on `sent_at` (chronological sorting)

**Validation Rules:**
- FR-025 to FR-028, FR-031, FR-033: Send appropriate notifications at each order state transition
- Retry failed notifications with exponential backoff

---

## Database Schema Summary

### Table Statistics (Estimated for MVP Scale)

| Table | Estimated Row Count (50 restaurants, 10,000 customers) |
|-------|--------------------------------------------------------|
| `users` | ~10,050 (10,000 customers + 50 restaurant owners) |
| `restaurants` | ~50 |
| `menu_items` | ~2,500 (50 restaurants × 50 items avg) |
| `orders` | ~50,000/year (50 restaurants × 1000 orders/year) |
| `order_items` | ~150,000/year (3 items per order avg) |
| `pickup_slots` | ~500,000/year (dynamic generation) |
| `payment_transactions` | ~50,000/year (1:1 with orders) |
| `reviews` | ~25,000/year (50% review rate) |
| `notifications` | ~500,000/year (10 notifications per order avg) |

### Storage Estimates
- Total database size (1 year): ~500 MB - 1 GB
- Image storage (local filesystem): ~1 GB (50 restaurants × 10 images × 2MB avg)

---

## Optimistic Locking Implementation

Per FR-030, optimistic locking prevents race conditions during order state changes.

### Implementation Pattern
```go
func (r *OrderRepository) UpdateOrderState(ctx context.Context, orderID string, expectedVersion int, newState string) error {
    result, err := r.db.Exec(
        "UPDATE orders SET state = $1, version = version + 1, updated_at = NOW() WHERE id = $2 AND version = $3",
        newState, orderID, expectedVersion,
    )
    if err != nil {
        return err
    }

    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        // Version mismatch - concurrent modification detected
        return ErrOptimisticLockConflict
    }

    return nil
}
```

### Conflict Resolution
- When conflict detected (version mismatch), return error to user
- User must retry with fresh order data
- FR-039: If cancellation request conflicts with restaurant acceptance, restaurant acceptance takes precedence

---

## Data Integrity & Constraints

### Foreign Key Cascades
- `menu_items.restaurant_id`: CASCADE (delete menu items when restaurant deleted)
- `order_items.order_id`: CASCADE (delete order items when order deleted)
- `pickup_slots.restaurant_id`: CASCADE
- **All other FKs**: RESTRICT (prevent deletion of referenced records)

### Triggers (PostgreSQL)
```sql
-- Auto-update restaurant rating average
CREATE OR REPLACE FUNCTION update_restaurant_rating()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE restaurants
    SET rating_avg = (
        SELECT AVG(rating) FROM reviews WHERE restaurant_id = NEW.restaurant_id
    ),
    rating_count = (
        SELECT COUNT(*) FROM reviews WHERE restaurant_id = NEW.restaurant_id
    )
    WHERE id = NEW.restaurant_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_restaurant_rating
AFTER INSERT OR UPDATE ON reviews
FOR EACH ROW EXECUTE FUNCTION update_restaurant_rating();

-- Auto-update timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_timestamp
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Repeat trigger for other tables: restaurants, menu_items, orders, etc.
```

---

## Migration Strategy

Per research.md decision to use `golang-migrate`, the schema will be implemented incrementally:

1. **Migration 001**: Users table (authentication foundation)
2. **Migration 002**: Restaurants table
3. **Migration 003**: Menu items table
4. **Migration 004**: Orders and order items tables (most complex, includes optimistic locking)
5. **Migration 005**: Payment transactions table
6. **Migration 006**: Pickup slots table
7. **Migration 007**: Reviews table
8. **Migration 008**: Notifications table
9. **Migration 009**: Indexes (performance optimization)
10. **Migration 010**: Triggers (rating calculation, timestamp updates)

Each migration will have corresponding `.up.sql` and `.down.sql` for rollback capability.

---

**Next Steps**: Generate OpenAPI contracts based on this data model and functional requirements.
