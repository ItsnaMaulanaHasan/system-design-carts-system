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

Key in Redis:

```
cart:{user_id}
```

Value:

```
{
  "product_id_1": {
    "qty": 2,
    "price_snapshot": 15000
  },
  "product_id_2": {
    "qty": 1,
    "price_snapshot": 50000
  }
}

```

2. PostgreSQL
   - Stores product data (name, price, stock, etc.)
   - Order data after checkout
   - PostgreSQL is an RDBMS that can maintain data consistency and data integrity.

## Architecture Diagram

```mermaid
flowchart LR
    U[User<br/>Web / Mobile] -->|HTTPS + JWT| FE[Frontend App]

    FE -->|API Request| LB[API Gateway / Load Balancer]

    LB --> BE["Backend Application<br/>(Cart & Order Module)"]

    BE -->|Get / Update Cart| R[(Redis<br/>Cart State + TTL)]

    BE -->|Read / Write| DB[(PostgreSQL<br/>Products, Orders)]

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

    Add@{ shape: lean-r, label: "User add product to cart" }
    Send@{ shape: rect, label: "Frontend send product_id and quantity" }

    Receive@{ shape: rect, label: "Backend receives request" }
    Get@{ shape: rect, label: "Get data product from database" }

    Check@{ shape: diamond, label: "product && quantity > 0 && quantity <= stock" }

    Error@{ shape: lean-r, label: "Validasi Error" }
    GetDataFromRedis@{ shape: rect, label: "Get data cart from redis by user id" }

    CheckCart@{ shape: diamond, label: "Cart != null" }

    CreateNewCart@{ shape: rect, label: "Create new cart" }
    UpdateQtyCart@{ shape: rect, label: "Update quantity cart" }

    SavePriceSnapshot@{ shape: rect, label: "Save price snapshot" }

    ReturnCart@{ shape: lean-r, label: "Return data cart to user" }

    Start --> Add --> Send--> Receive --> Get --> Check

    Check-- false --> Error --> Stop
    Check-- true --> GetDataFromRedis --> CheckCart

    CheckCart -- false --> CreateNewCart --> SavePriceSnapshot
    CheckCart -- true --> UpdateQtyCart --> ReturnCart

    SavePriceSnapshot --> ReturnCart --> Stop

```

## Update Quantity

```mermaid
flowchart TD
    Start@{ shape: circle, label: "Start" }
    Stop@{ shape: dbl-circ, label: "Stop" }

    UserUpdate@{ shape: lean-r, label: "User update quantity" }
    BackendGet@{ shape: rect, label: "Backend get data product from database" }

    CheckStock@{ shape: diamond, label: "New quantity <= stock" }
    ReturnError@{ shape: lean-r, label: "Validasi Error" }
    UpdateCart@{ shape: rect, label: "Update data cart" }
    SaveToRedis@{ shape: rect, label: "Save new data cart to Redis" }
    ReturnNewCart@{ shape: rect, label: "Return new cart with new quantity" }

    Start --> UserUpdate --> BackendGet --> CheckStock

    CheckStock -- false --> ReturnError --> Stop
    CheckStock -- true --> UpdateCart --> SaveToRedis --> ReturnNewCart --> Stop
```

## Checkout

```mermaid
flowchart TD
    Start@{ shape: circle, label: "Start" }
    Stop@{ shape: dbl-circ, label: "Stop" }

    UserCheckout@{ shape: lean-r, label: "User checkout cart" }
    AcquireLock@{ shape: rect, label: "Acquire Redis checkout lock" }

    LockCheck@{ shape: diamond, label: "Lock acquired?" }
    ReturnLocked@{ shape: lean-r, label: "Checkout in progress" }

    GetCart@{ shape: rect, label: "Get cart from Redis" }
    CartCheck@{ shape: diamond, label: "Cart != null" }

    BeginTX@{ shape: rect, label: "Begin DB transaction" }
    GetItems@{ shape: rect, label: "Get cart items" }

    StockCheck@{ shape: diamond, label: "Stock sufficient?" }

    Rollback@{ shape: rect, label: "Rollback transaction" }
    InsertOrder@{ shape: rect, label: "Insert order & order_items" }
    UpdateStock@{ shape: rect, label: "Update product stock" }
    Commit@{ shape: rect, label: "Commit transaction" }

    DeleteCart@{ shape: rect, label: "Delete cart from Redis" }
    ReleaseLock@{ shape: rect, label: "Release Redis lock" }

    ReturnSuccess@{ shape: lean-r, label: "Return checkout success" }

    Start --> UserCheckout --> AcquireLock --> LockCheck

    LockCheck -- false --> ReturnLocked --> Stop
    LockCheck -- true --> GetCart --> CartCheck

    CartCheck -- false --> ReleaseLock

    CartCheck -- true --> BeginTX --> GetItems --> StockCheck

    StockCheck -- false --> Rollback --> ReleaseLock
    StockCheck -- true --> InsertOrder --> UpdateStock --> Commit --> DeleteCart --> ReleaseLock --> ReturnSuccess --> Stop

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
