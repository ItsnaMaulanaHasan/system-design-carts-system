# System Design Cart System

## Requirements

- Add prouduct to cart
- Remove product from cart
- Update quantity
- Remove cart/Checkout
- Consistent/Multi device (accross sessions) per account
- Race condition (only 1 cart can be checkout)
- 1 day 1 million data add to cart
- Expired cart 1 month
- Data integrity (no price manipulation, and parameter tampering)
- Validation stock when checkout

## Storage Type Selection Strategy

1. Redis

   - To store active cart data
   - Automatic expiration (TTL)
   - Fast and inexpensive (suitable for frequently changing data, such as carts)

Redis key structue:

```
# Cart data
cart:{user_id} → Hash of cart items
  - TTL: 30 days
  - Value: {"product_id": {"qty": 2, "price_snapshot": 15000, "added_at": timestamp}}

# Locks
lock:cart:{user_id}:{product_id} → "locked"
  - TTL: 60 seconds
  - Value: timestamp of lock creation

lock:product:{product_id} → "locked"
  - TTL: 60 seconds
  - Used during add/update to prevent overselling

lock:checkout:{user_id} → "locked"
  - TTL: 120 seconds
  - Longer TTL for checkout process
```

2. PostgreSQL
   - Stores product data (name, price, stock, etc.)
   - Order data after checkout
   - PostgreSQL is an RDBMS that can maintain data consistency and data integrity.

## Architecture Diagram

```mermaid
flowchart LR
    User@{ shape: rect, label: User<br/>Web / Mobile }
    FE@{ shape: rect, label: Frontend App }
    LB@{ shape: rect, label: API Gateaway / Load Balancer }
    BE@{ shape: rect, label: "Backend Application<br/>(Cart & Order Module)" }

    R@{ shape: cyl, label: Redis<br/>Cart State + TTL }
    DB@{ shape: cyl, label: "PostgreSQL<br/>Products, Orders, Carts" }

    User -- HTTPS + JWT --> FE -- API Request --> LB --> BE -- Get / Update Cart --> R
    BE -- Read, Write, Backup carts --> DB

    subgraph Data Layer
        R
        DB
    end

```

## ERD

```mermaid
erDiagram
    USERS {
        uuid id PK
        varchar full_name
        varchar email "UNIQUE"
        text password_hash
        timestamp created_at
        timestamp updated_at
    }

    PRODUCTS {
        uuid id PK
        varchar sku "UNIQUE"
        varchar name
        numeric price "price > 0"
        bigint stock "stock >= 0"
        timestamp created_at
        timestamp updated_at
    }

    CARTS {
        uuid id PK
        uuid user_id FK
        varchar status
        timestamp expired_at
        timestamp created_at
    }

    CART_ITEMS {
        bigint id PK
        uuid cart_id FK
        uuid product_id FK
        int quantity "quantity > 0"
        numeric price_snapshot ">= 0"
    }

    ORDERS {
        uuid id PK
        uuid user_id FK
        varchar status
        numeric total_amount "total_amount >= 0"
        timestamp created_at
    }

    ORDER_ITEMS {
        bigint id PK
        uuid order_id FK
        uuid product_id FK
        int quantity "quantity > 0"
        numeric price_snapshot " price_snapshot >= 0"
        numeric subtotal_snapshot "subtotal_snapshot >= 0"
    }

    USERS ||--o{ CARTS : owns
    CARTS ||--o{ CART_ITEMS : contains
    PRODUCTS ||--o{ CART_ITEMS : referenced_by

    USERS ||--o{ ORDERS : places
    ORDERS ||--o{ ORDER_ITEMS : has
    PRODUCTS ||--o{ ORDER_ITEMS : referenced_by
```

# Flowchart

## Add Item

```mermaid
flowchart TD
    Start@{ shape: circle, label: "Start" }
    Stop@{ shape: dbl-circ, label: "Stop" }

    UserAdd@{ shape: lean-r, label: "User add product to cart" }

    TryLockProduct@{ shape: rect, label: "Try acquire lock:product:{product_id}<br/>TTL 60s" }
    LockProductOK@{ shape: diamond, label: "Lock acquired?" }
    RetryWait@{ shape: rect, label: "Wait 100ms & retry<br/>(max 3x)" }
    ErrorLocked@{ shape: lean-r, label: "Product temporarily locked" }

    GetProduct@{ shape: rect, label: "Get product from DB<br/>(price, stock)" }
    ValidateStock@{ shape: diamond, label: "quantity > 0 &&<br/>quantity <= stock" }
    ErrorStock@{ shape: lean-r, label: "Insufficient stock" }

    TryLockCart@{ shape: rect, label: "Try acquire lock:cart:{user_id}:{product_id}<br/>TTL 60s" }
    LockCartOK@{ shape: diamond, label: "Lock acquired?" }

    GetCart@{ shape: rect, label: "Get cart from Redis" }
    CheckExisting@{ shape: diamond, label: "Product exists<br/>in cart?" }

    CalcNewQty@{ shape: rect, label: "Calculate new quantity" }
    ValidateNewQty@{ shape: diamond, label: "new_qty <= stock" }

    AddNewItem@{ shape: rect, label: "Add new item to cart" }
    UpdateQty@{ shape: rect, label: "Update quantity" }

    SaveSnapshot@{ shape: rect, label: "Save price_snapshot" }
    SaveRedis@{ shape: rect, label: "HSET cart:{user_id}<br/>EXPIRE 30 days" }

    ReleaseLocks@{ shape: rect, label: "Release all locks<br/>(cart + product)" }
    Success@{ shape: lean-r, label: "Return updated cart" }

    Start --> UserAdd --> TryLockProduct --> LockProductOK

    LockProductOK -- false --> RetryWait
    RetryWait -- Max retries --> ErrorLocked --> Stop
    RetryWait -- Retry --> TryLockProduct

    LockProductOK -- true --> GetProduct --> ValidateStock

    ValidateStock -- false --> ErrorStock
    ValidateStock -- true --> TryLockCart --> LockCartOK

    LockCartOK -- false --> RetryWait
    LockCartOK -- true --> GetCart --> CheckExisting

    CheckExisting -- true --> CalcNewQty --> ValidateNewQty
    CheckExisting -- false --> AddNewItem --> SaveSnapshot

    ValidateNewQty -- false --> ErrorStock --> ReleaseLocks
    ValidateNewQty -- true --> UpdateQty --> SaveRedis

    SaveSnapshot --> SaveRedis --> Success --> ReleaseLocks --> Stop
```

## Update Quantity

```mermaid
flowchart TD
    Start@{ shape: circle, label: "Start" }
    Stop@{ shape: dbl-circ, label: "Stop" }

    UserUpdate@{ shape: lean-r, label: "User update quantity" }

    TryLockProduct@{ shape: rect, label: "Acquire lock:product:{product_id}<br/>TTL 60s" }
    TryLockCart@{ shape: rect, label: "Acquire lock:cart:{user_id}:{product_id}<br/>TTL 60s" }

    LockOK@{ shape: diamond, label: "Both locks<br/>acquired?" }
    ErrorLock@{ shape: lean-r, label: "Unable to acquire lock" }

    GetProduct@{ shape: rect, label: "Get product from DB" }
    ValidateStock@{ shape: diamond, label: "new_qty > 0 &&<br/>new_qty <= stock" }

    ErrorStock@{ shape: lean-r, label: "Invalid quantity" }

    UpdateCart@{ shape: rect, label: "Update cart in Redis<br/>HSET cart:{user_id}" }
    ReleaseLocks@{ shape: rect, label: "Release locks" }
    Success@{ shape: lean-r, label: "Return updated cart" }

    Start --> UserUpdate --> TryLockProduct --> TryLockCart --> LockOK

    LockOK -- false --> ErrorLock --> Stop
    LockOK -- true --> GetProduct --> ValidateStock

    ValidateStock -- false --> ErrorStock --> ReleaseLocks
    ValidateStock -- true --> UpdateCart --> Success --> ReleaseLocks --> Stop
```

## Checkout

```mermaid
flowchart TD
    Start@{ shape: circle, label: "Start" }
    Stop@{ shape: dbl-circ, label: "Stop" }

    UserCheckout@{ shape: lean-r, label: "User checkout" }

    AcquireLock@{ shape: rect, label: "Acquire lock:checkout:{user_id}<br/>TTL 120s" }
    LockOK@{ shape: diamond, label: "Lock acquired?" }
    ErrorLock@{ shape: lean-r, label: "Checkout in progress" }

    GetCart@{ shape: rect, label: "Get cart from Redis" }
    CartEmpty@{ shape: diamond, label: "Cart exists?" }
    ErrorEmpty@{ shape: lean-r, label: "Cart is empty" }

    BeginTX@{ shape: rect, label: "BEGIN Transaction" }
    LockProducts@{ shape: rect, label: "SELECT products FOR UPDATE<br/>(lock rows in DB)" }

    ValidateStock@{ shape: rect, label: "Check stock for each item" }
    StockOK@{ shape: diamond, label: "Stock sufficient?" }

    ValidatePrice@{ shape: rect, label: "Compare price_snapshot<br/>with current price" }
    PriceOK@{ shape: diamond, label: "Price unchanged?" }

    NotifyPrice@{ shape: lean-r, label: "Notify price change<br/>Ask confirmation" }

    CreateOrder@{ shape: rect, label: "Insert order & order_items" }
    UpdateStock@{ shape: rect, label: "Update product stock" }
    Commit@{ shape: rect, label: "COMMIT" }

    Rollback1@{ shape: rect, label: "ROLLBACK" }
    Rollback2@{ shape: rect, label: "ROLLBACK" }

    DeleteCart@{ shape: rect, label: "Delete cart from Redis" }
    ReleaseLock@{ shape: rect, label: "Release checkout lock" }

    Success@{ shape: lean-r, label: "Return order confirmation" }
    ErrorStock@{ shape: lean-r, label: "Stock insufficient" }

    Start --> UserCheckout --> AcquireLock --> LockOK

    LockOK -- false --> ErrorLock --> Stop
    LockOK -- true --> GetCart --> CartEmpty

    CartEmpty -- false --> ErrorEmpty --> ReleaseLock
    CartEmpty -- true --> BeginTX --> LockProducts --> ValidateStock --> StockOK

    StockOK -- false --> Rollback1 --> ErrorStock --> ReleaseLock
    StockOK -- true --> ValidatePrice --> PriceOK

    PriceOK -- false --> Rollback2 --> NotifyPrice --> ReleaseLock
    PriceOK -- true --> CreateOrder --> UpdateStock --> Commit --> DeleteCart --> Success --> ReleaseLock --> Stop
```

# Sequence Diagram

## Add and Update Cart

```mermaid
sequenceDiagram
    actor User
    participant FE as Frontend
    participant BE as Backend API
    participant DB as PostgreSQL
    participant R as Redis

    User ->> FE: Click Add / Update Cart
    FE ->> BE: POST /cart/items (product_id, qty)

    BE ->> DB: Get product (price, stock)
    DB -->> BE: Product data

    BE ->> BE: Validate qty <= stock

    BE ->> R: GET cart:{user_id}
    R -->> BE: Cart data (if exists)

    BE ->> R: Update qty + price_snapshot
    BE ->> R: EXPIRE cart:{user_id} (30 days)

    BE -->> FE: Return updated cart
    FE -->> User: Render cart
```

## Checkout

```mermaid
sequenceDiagram
    actor User
    participant FE as Frontend
    participant BE as Backend API
    participant R as Redis
    participant DB as PostgreSQL

    User ->> FE: Click Checkout
    FE ->> BE: POST /checkout

    BE ->> R: SETNX checkout_lock:{user_id}
    alt Lock exists
        R -->> BE: FAIL
        BE -->> FE: Checkout in progress
    else Lock acquired
        R -->> BE: OK

        BE ->> R: GET cart:{user_id}
        R -->> BE: Cart data

        BE ->> DB: BEGIN TRANSACTION
        BE ->> DB: SELECT products FOR UPDATE
        DB -->> BE: Locked rows

        BE ->> BE: Validate stock

        BE ->> DB: INSERT orders
        BE ->> DB: INSERT order_items
        BE ->> DB: UPDATE products.stock
        BE ->> DB: COMMIT

        BE ->> R: DEL cart:{user_id}
        BE ->> R: DEL checkout_lock:{user_id}

        BE -->> FE: Checkout success
    end
```

## Checkout (Failed)

```mermaid
sequenceDiagram
    participant BE as Backend API
    participant DB as PostgreSQL

    BE ->> DB: SELECT products FOR UPDATE
    DB -->> BE: Stock insufficient

    BE ->> DB: ROLLBACK
    BE -->> BE: Return error (out of stock)

```
