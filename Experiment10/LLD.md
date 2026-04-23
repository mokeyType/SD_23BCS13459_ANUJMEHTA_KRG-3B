# Low-Level Design — Food Delivery Platform

## Overview
This LLD translates high-level components into concrete classes, database schema, sequence diagrams and patterns suitable for implementation in a microservices environment.

## 1. Class Diagrams (conceptual)

Below are primary domain classes expressed as concise definitions. Use these as the basis for class diagrams in UML tools.

- `User` {id, name, phone, email, addresses[], walletBalance}
- `Restaurant` {id, name, address, openingHours, menuVersionId, rating}
- `MenuItem` {id, restaurantId, name, description, basePrice, modifiers[]}
- `Cart` {userId, items[], couponCode, totalPrice}
- `Order` {id, userId, restaurantId, items[], total, status, createdAt, updatedAt}
- `OrderItem` {orderId, menuItemId, qty, price, modifiers}
- `Payment` {id, orderId, amount, method, status, pspReference}
- `DeliveryPartner` {id, name, currentLocation, status, rating, lastSeen}
- `Assignment` {id, orderId, partnerId, assignedAt, status}

OOP principles applied:
- Encapsulation: domain objects expose behavior (e.g., `Order.calculateTotal()`), not just data.
- Single Responsibility: separate `PricingService` from `Order` to handle promotions and surge.
- Dependency Injection: controllers receive interfaces (Repository, PaymentGateway) to ease testing.
- Interface segregation: different client-facing controllers for user/restaurant/courier APIs.

## 2. Database Schema (core tables)

Notes: primary OLTP store is Postgres (ACID). Use UUIDs for global uniqueness. Add created_at/updated_at on all tables.

- `users`
  - id UUID PK
  - phone VARCHAR(15) UNIQUE
  - email VARCHAR(255) UNIQUE
  - name VARCHAR(100)
  - default_address_id UUID
  - created_at TIMESTAMP
  - INDEX: phone, email

- `restaurants`
  - id UUID PK
  - name TEXT
  - address TEXT
  - location GEOGRAPHY(POINT) -- PostGIS
  - opening_hours JSONB
  - rating DECIMAL(2,1)
  - created_at TIMESTAMP
  - INDEX: location (GIN), name

- `menu_items`
  - id UUID PK
  - restaurant_id UUID FK -> restaurants(id)
  - name TEXT
  - description TEXT
  - base_price INT (in paise/cents)
  - is_available BOOLEAN
  - modifiers JSONB
  - INDEX: restaurant_id, is_available

- `orders`
  - id UUID PK
  - user_id UUID FK
  - restaurant_id UUID FK
  - status VARCHAR(32) -- enum (CREATED, PLACED, CONFIRMED, PREPARING, PICKED, DELIVERED, CANCELLED)
  - total_amount BIGINT
  - payment_status VARCHAR(32)
  - idempotency_key VARCHAR(255) UNIQUE NULLABLE
  - created_at TIMESTAMP
  - INDEX: user_id, restaurant_id, status, idempotency_key

- `order_items`
  - id UUID PK
  - order_id UUID FK -> orders(id)
  - menu_item_id UUID
  - quantity INT
  - price BIGINT
  - modifiers JSONB
  - INDEX: order_id

- `payments`
  - id UUID PK
  - order_id UUID FK
  - amount BIGINT
  - method VARCHAR(32)
  - status VARCHAR(32)
  - psp_reference VARCHAR(255)
  - created_at TIMESTAMP
  - INDEX: order_id, psp_reference

- `delivery_partners`
  - id UUID PK
  - name TEXT
  - phone VARCHAR(15)
  - current_location GEOGRAPHY(POINT)
  - status VARCHAR(16) (AVAILABLE, ON_TRIP, OFFLINE)
  - skills JSONB (e.g., bike/car)
  - INDEX: current_location (GiST)

- `assignments`
  - id UUID PK
  - order_id UUID FK
  - partner_id UUID FK
  - status VARCHAR(32)
  - created_at TIMESTAMP
  - INDEX: partner_id, order_id

- `promotions`
  - id UUID PK
  - code VARCHAR(64) UNIQUE
  - discount JSONB
  - eligibility JSONB
  - expires_at TIMESTAMP
  - INDEX: code

Design decisions & indexing rationale:
- PostGIS types for efficient geospatial queries (nearest courier, serviceability) with GiST indexes.
- `idempotency_key` unique index on `orders` to prevent duplicate order creation.
- Use JSONB for modifiers and opening_hours to allow flexible attributes and fast partial indexing.

## 3. Relationships between entities

- `User` 1..* `Order`
- `Restaurant` 1..* `MenuItem`
- `Order` 1..* `OrderItem`
- `Order` 1 `Assignment` (0..1 while not assigned; 1..N for reassign history if needed)
- `Order` 1 `Payment` (support multiple payments for retries/refunds)

## 4. Sequence Diagrams (textual, developer-ready)

### Sequence 1: Order Placement

Actors: UserApp, API Gateway, OrderService, CatalogService, PricingService, PaymentService, Kafka, RestaurantApp

1. UserApp -> API Gateway: POST /v1/orders {cart, address, idempotencyKey}
2. API Gateway -> OrderService: forward request
3. OrderService -> CatalogService: validate items (cache-first, fallback DB)
4. OrderService -> PricingService: compute final price (apply taxes/fees/promos)
5. OrderService: start DB transaction; insert order row with idempotencyKey
6. OrderService -> PaymentService: create payment intent (idempotencyKey forwarded)
7. PaymentService -> PSP (external): process payment
8. PSP -> PaymentService: payment success
9. PaymentService -> OrderService: payment confirmed
10. OrderService: mark order CONFIRMED; commit transaction
11. OrderService -> Kafka: publish order.confirmed
12. RestaurantApp (subscribed) receives push -> shows new order

Notes: Steps 5-10 must be ACID. If payment succeeds but publish fails, Kafka producer must retry (at-least-once) and system must tolerate duplicate events.

### Sequence 2: Delivery Assignment

Actors: OrderService (event source), Kafka, AssignmentService, DeliveryPartnerService, RealTimeGateway, CourierApp

1. OrderService -> Kafka: publish order.confirmed
2. AssignmentService (consumer) receives event
3. AssignmentService -> Redis/GeoIndex: query nearby AVAILABLE partners (S2 / PostGIS tiles cached in Redis)
4. AssignmentService -> DeliveryPartnerService: push tentative assignment (push notification)
5. CourierApp -> AssignmentService: accept/reject
6. AssignmentService -> Orders DB: update assignment row; publish assignment.status
7. RealTimeGateway -> UserApp: notify assignment & ETA

Notes: Assignment must be idempotent and support reassignments. Use short-lived locks in Redis to avoid double-assign.

## 5. Design Patterns used (with justification)

- Factory Pattern: PaymentGatewayFactory produces objects for `StripeGateway`, `RazorpayGateway`, `PaytmGateway`. Justification: PSP integrations vary; factory isolates instantiation and configuration.

- Strategy Pattern: Pricing strategies (flat-fee, distance-based, surge-based) are pluggable. Justification: business rules change frequently and A/B testing is necessary.

- Repository Pattern: Data access through `OrderRepository`, `MenuRepository` to abstract DB technology and support mock testing.

- Observer/Event-Driven: Order events published to Kafka; multiple consumers react (Assignment, Notifications, Analytics). Justification: decouples services and improves scalability.

- Singleton: Service configuration loader and metrics registry as singletons within a process. Keep to process scope only.

- Circuit Breaker & Bulkhead: For Payment and external Map APIs to prevent cascading failures.

- Adapter: Wrap external PSPs and Map providers behind adapters to normalize APIs.

## 6. Operational concerns

- Idempotency: `orders` table has `idempotency_key` unique constraint; services honor `Idempotency-Key` header.
- Retries: use exponential backoff for transient external failures; fail fast for permanent errors.
- Observability: OpenTelemetry traces for cross-service flows; structured logs (json) with request id.

---
End of LLD
