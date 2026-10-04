# Architecture overview

## 1. Business goals

The system is a high-load internet store with:
- large catalog of products
- complex pricing logic
- multiple variants and attributes
- discounts and promotions
- coupons and personalized offers
- payments by multiple providers
- order lifecycle management
- resilient distributed processing
- admin management for products, pricing, promotions, and orders

The architecture must support:
- thousands of product requests per second
- high write/read load on orders and catalog
- asynchronous processing for payments and order events
- separation of critical business flows
- straightforward scaling of services under load
- operational observability and recovery after failures

## 2. System architecture

### 2.1 High-level structure

```text
Client (Web / Mobile / Admin)
           |
           v
      API Gateway
           |
   +-------+---------------------------+
   |                                   |
   v                                   v
Auth Service                    Storefront Services
User / Session / RBAC          Product / Cart / Order / Payment
   |                                   |
   +-----------+---------------+-------+
               |               |
               v               v
        Discovery / Registry   Message Broker (Kafka)
               |               |
               v               v
         Config Server     Event consumers / async workers
               |
               v
      Monitoring and Observability
```

## 3. Core domain model

### 3.1 Product domain

The product domain is the foundation of the store.

Key entities:
- Product
- ProductVariant
- ProductAttribute
- Category
- Brand
- MediaAsset
- ProductInventory
- ProductPrice
- ProductReview

### 3.2 Product structure

A product can be abstract or concrete depending on the business strategy.

Example:
- Product: "Smartphone X"
- Variants: red / 128GB / black / 256GB
- Attributes: color, memory, warranty, ram, processor
- Price logic: base price + attribute modifiers + promotion discount + shipping rule

### 3.3 Price calculation model

The total item price depends on multiple layers:

1. Base price from product catalog
2. Variant or attribute modifier
3. Customer group or segment price rules
4. Promotion discount
5. Coupon discount
6. Tax and shipping

Example:

Base price = 1000 USD
Color: black = +0
Memory: 256GB = +150 USD
Warranty: 2 years = +50 USD
Subtotal = 1200 USD
Promo 10% = -120 USD
Coupon 100 USD = -100 USD
Final price = 980 USD

This means price must be calculated by a dedicated pricing service rather than directly in the product service.

## 4. Microservices decomposition

### 4.1 Product Service

Responsibilities:
- manage products and categories
- manage product attributes and variants
- serve product cards and catalog lists
- maintain product metadata and media
- provide search indexing data

### 4.2 Inventory Service

Responsibilities:
- stock quantity tracking
- reservations
- warehouse and fulfillment status
- stock movement history

### 4.3 Pricing Service

Responsibilities:
- calculate final product price
- evaluate attribute modifiers
- apply dynamic pricing rules
- apply promotions and coupons
- calculate taxes where needed

This service is critical because pricing logic is often complex and high-impact.

### 4.4 Promotion Service

Responsibilities:
- product promotions
- category promotions
- cart-level promotions
- coupon validation
- promotional eligibility rules
- usage limits and expiration logic

### 4.5 Cart Service

Responsibilities:
- cart lifecycle
- add/remove/update product quantity
- persist cart summary
- calculate cart totals with pricing service
- validate stock during cart interactions

### 4.6 Order Service

Responsibilities:
- create order from cart
- validate order against stock and pricing
- manage order status lifecycle
- emit order events

### 4.7 Payment Service

Responsibilities:
- credit card payment
- bank transfer
- wallet or external provider integration
- payment status tracking
- transaction verification and refund processing

### 4.8 Delivery Service

Responsibilities:
- shipping calculation
- delivery options
- shipment tracking
- warehouse coordination

### 4.9 Auth + User Service

Responsibilities:
- authentication
- roles and permissions
- customer profile and addresses
- admin accounts

### 4.10 Admin Service

Responsibilities:
- product administration
- pricing management
- promotion management
- order review and case handling
- sales analytics

## 5. Event-driven architecture

The system should avoid synchronous coupling between critical services.

Main events:
- ProductCreated
- ProductUpdated
- InventoryReserved
- InventoryReleased
- OrderCreated
- PaymentSucceeded
- PaymentFailed
- OrderConfirmed
- OrderShipped
- OrderCancelled
- PromotionApplied

Kafka can be used for:
- order processing stream
- async notifications
- inventory updates
- analytics ingestion
- background promotion recalculation

## 6. Database strategy

### 6.1 Per-service database ownership

Each service owns its own database schema.

Examples:
- product-service: PostgreSQL catalog schema
- inventory-service: PostgreSQL inventory schema
- order-service: PostgreSQL order schema
- payment-service: PostgreSQL payment schema
- promotion-service: PostgreSQL promotion schema

This keeps services loosely coupled and independent.

### 6.2 Shared data patterns

Some data is cross-service and should be replicated or projected.
Examples:
- product catalog summary for cart or order service
- customer information cache in auth/user service
- inventory availability snapshot for pricing or order validation

## 7. Pricing and promotions design

### 7.1 Price components

A final product price can be represented as:

final_price = base_price + attribute_modifiers + surcharges - discounts + tax + shipping

Where:
- base_price: standard product price
- attribute_modifiers: dimensions, colors, materials, packaging, extras
- surcharges: premium delivery, age restrictions, installation fees
- discounts: promotion, coupon, loyalty offers
- tax: VAT or regional taxes
- shipping: order delivery cost

### 7.2 Attribute pricing rules

Product attributes may influence price directly or indirectly.
Examples:
- size: S/M/L = different price
- color: premium color = +10%
- warranty: 6 months / 1 year / 2 years
- packaging: gift box = +30
- installation: service = +50

Key design principle:
Attribute values should not be stored as arbitrary string-only fields when pricing is involved.
They should be modeled with explicit metadata:
- attribute type
- attribute value
- price delta
- optional stock impact
- enabled/disabled flag

### 7.3 Promotions model

Promotions may be:
- percentage discount
- fixed amount discount
- buy one get one
- free shipping
- cart condition promotion
- category promotion
- product-specific promotion
- customer segment promotion

Rules may include:
- minimum order value
- minimum quantity
- product/category restrictions
- day/time restrictions
- user segment restrictions
- coupon stacking rules
- maximum application count

## 8. Admin panel requirements

Admin functionality must support:
- add/update/delete products
- manage categories and brands
- upload media assets
- manage inventrory and warehouse stock
- create promotions and coupon codes
- configure price rules and attribute pricing
- review orders and statuses
- manage refunds and refunds reasons
- generate analytics on revenue and conversions

A typical admin module structure:
- products
- categories
- attributes
- pricing rules
- promotions
- coupons
- inventory
- orders
- customers
- analytics
- settings

## 9. Performance and resilience

### 9.1 Cache layer

Use Redis for:
- product catalog hot data
- category tree
- cart caching
- pricing cache
- session management
- promo validation results

### 9.2 Search layer

Use Elasticsearch for:
- product search
- filters
- faceted category browsing
- search suggestions

### 9.3 Load balancing and redundancy

Use:
- Gateway behind load balancer
- multiple instances of stateless services
- database replication for read operations
- message queue for asynchronous processing
- circuit breakers and retries for external APIs

### 9.4 Operational concerns

Need monitoring for:
- latency p95 / p99
- error rates
- payment failures
- stock mismatch
- queue lag
- CPU / memory / DB IOPS

Use:
- Prometheus
- Grafana
- Loki or ELK
- OpenTelemetry

## 10. First implementation priorities

We will build in phases.

Phase 1:
- product service
- attribute and variant model
- pricing service
- promotion service
- admin endpoints

Phase 2:
- cart service
- order service
- payment service
- Kafka integration

Phase 3:
- delivery service
- inventory service
- notifications
- admin analytics

Phase 4:
- deployment
- CI/CD
- monitoring
- scaling and resilience tests

## 11. Key design principles

1. Each service owns its domain data.
2. Price calculation must be centralized.
3. Promotions must be rule-driven and not hardcoded.
4. Inventory and payment should be event-driven.
5. Admin panel must be separate from storefront APIs.
6. Use asynchronous messaging where consistency can be delayed.
7. Design for fault tolerance before optimizing for speed.

This design gives a strong foundation for a real-world, high-load online store that can evolve to large-scale operations.
