# Order Service — Principal HLD

## 1. Problem Statement

The Order Service represents a customer's purchase and manages the order lifecycle from creation to completion.

It owns:

- order information
- order items
- order lifecycle state
- customer order history
- order-level business operations such as cancellation

It does not own:

- inventory quantity
- payment transaction state
- shipment state
- product catalog

Those responsibilities belong to their respective services.

The Order Service coordinates the workflow but does not need a distributed database transaction across all services.

---

# 2. Requirements

## 2.1 Functional Requirements

### Create Order

Create an order from a customer's purchase request.

### Get Order

Retrieve one order and its items.

### List Orders

Retrieve a customer's recent orders.

### Cancel Order

Cancel an order when its current state permits cancellation.

### Process Internal Events

React to events such as:

- inventory reserved
- inventory reservation failed
- payment succeeded
- payment failed
- shipment started
- delivery completed

### Recover Stuck Orders

Detect orders that remain in an intermediate state longer than expected.

---

# 3. Scale and Workload

Estimate:

- order creation QPS
- peak checkout QPS
- order-read QPS
- read/write ratio
- average items per order
- orders per customer
- total historical orders
- status-filter frequency

A likely read-heavy access pattern is:

> "Give me the customer's latest orders."

A likely write-heavy pattern is:

> "Create order during checkout."

The first bottleneck may therefore be different from the bottleneck in Inventory or Payment.

---

# 4. Domain Model

## Order

Represents the customer's purchase.

```text
Order
-----
orderId
userId
status
currency
subtotal
discount
total
createdAt
updatedAt
version
```

---

## Order Item

Represents an item captured inside the order.

```text
OrderItem
---------
orderItemId
orderId
itemId
quantity
unitPrice
total
```

---

## Price Snapshot

When the order is created, the order should normally store the price used for that purchase.

Why?

Suppose the catalog price is:

```text
₹100
```

Customer creates the order.

Later catalog price becomes:

```text
₹130
```

The old order should still represent the original purchase at ₹100.

Therefore historical order data should not depend on the current catalog price.

### Trade-off

We duplicate a small amount of product/pricing information, but we gain historical correctness.

---

# 5. Business Invariants

Important rules include:

1. An order must contain valid items.
2. The stored total must match the order's pricing rules.
3. A terminal order must not move back to an active state.
4. Cancellation is allowed only from permitted states.
5. Duplicate requests must not create duplicate orders.
6. Duplicate or stale events must not cause invalid state transitions.

---

# 6. State Machine

## Example States

```text
CREATED
INVENTORY_RESERVED
PAYMENT_PENDING
PAYMENT_SUCCESSFUL
SHIPPING
DELIVERED

PAYMENT_FAILED
CANCELLED
TIMED_OUT
```

The exact state model depends on business requirements.

---

## Transition Table

| Current State | Event | New State | Actor |
|---|---|---|---|
| CREATED | InventoryReserved | INVENTORY_RESERVED | Order |
| CREATED | InventoryFailed | CANCELLED/FAILED | Order |
| CREATED | Timeout | TIMED_OUT | Order/Reconciler |
| INVENTORY_RESERVED | PaymentStarted | PAYMENT_PENDING | Order |
| PAYMENT_PENDING | PaymentSucceeded | PAYMENT_SUCCESSFUL | Order |
| PAYMENT_PENDING | PaymentFailed | PAYMENT_FAILED | Order |
| PAYMENT_SUCCESSFUL | ShipmentStarted | SHIPPING | Order |
| SHIPPING | DeliveryCompleted | DELIVERED | Order |
| Active state | CustomerCancel | CANCELLED | Order |
| Terminal state | Duplicate event | Same state | Order |

The exact transitions should be clarified with the interviewer.

---

## Important Race: Payment Success vs Timeout

Suppose:

```text
PaymentSucceeded
```

and:

```text
OrderTimeout
```

arrive almost simultaneously.

The order must not depend on whichever thread happens to execute last.

Instead, the transition should verify the current expected state/version.

For example:

```text
PAYMENT_PENDING + PaymentSucceeded
        → PAYMENT_SUCCESSFUL
```

while:

```text
PAYMENT_PENDING + Timeout
        → TIMED_OUT
```

Only one valid transition should win.

---

## Draw.io State Diagram

**Place the Draw.io diagram here.**

The diagram should be created from the transition table and race analysis.

---

# 7. APIs

## Create Order

```http
POST /orders
Idempotency-Key: abc123
```

The request should contain the information required to create the order.

---

## Get Order

```http
GET /orders/{orderId}
```

---

## List Customer Orders

```http
GET /users/{userId}/orders
```

Likely supports:

- pagination
- optional status filter
- ordering by creation time

---

## Cancel Order

```http
POST /orders/{orderId}/cancel
```

The service decides whether cancellation is allowed.

Do not expose arbitrary status mutation.

---

# 8. Access Patterns

Important access patterns:

1. Get order by ID.
2. Get all items for an order.
3. Get latest orders for a customer.
4. Get customer orders filtered by status.
5. Find order by idempotency key.
6. Update order state by order ID.
7. Find stale non-terminal orders.

These patterns should drive the physical data model.

---

# 9. Database / Storage Decision

## Storage Requirements

We need:

- durable order state
- relationship between order and order items
- atomic creation of order and items
- customer history queries
- uniqueness/idempotency
- conditional state updates
- predictable reads

---

## Relational Database

A relational database is a strong candidate because:

- order and order items have a natural relationship
- creating an order and its items benefits from a transaction
- uniqueness constraints are useful
- indexes support customer history queries
- conditional updates help protect state transitions

---

## Document Database

A document database could work well if:

- an order is always treated as one aggregate
- orders are primarily read as complete documents
- cross-order relational queries are limited

The order and items could be stored together.

However, frequent querying across customers, statuses or other dimensions may be less natural depending on the document model.

---

## Key-Value / Wide-Column

Could be appropriate at very large scale if access patterns are simple and predictable.

However, some transactional and relational guarantees would need more application-level handling depending on the chosen technology.

---

## Decision

For the current transactional order requirements, a relational database is a strong initial choice.

This is **not a rule that Order Service must use SQL**.

The decision comes from:

```text
Order + Items relationship
        +
Transactional creation
        +
Customer history queries
        +
State transitions
        +
Idempotency
```

If the workload later becomes large enough to exceed the practical limits of the relational architecture, revisit the storage decision using measured workload data.

---

# 10. Schema and Indexes

Possible relational representation:

### `orders`

```text
order_id
user_id
status
currency
subtotal
discount
total
created_at
updated_at
version
idempotency_key
```

### `order_items`

```text
order_item_id
order_id
item_id
quantity
unit_price
total
```

---

## Indexes

For:

> Get customer's latest orders.

Use:

```text
(user_id, created_at DESC)
```

For:

> Get customer's latest orders in a particular status.

Potentially:

```text
(user_id, status, created_at DESC)
```

But only add it if this query is frequent enough.

For:

> Find all items for an order.

Use:

```text
(order_id)
```

For idempotency:

> Use an appropriate uniqueness constraint on the idempotency key based on its scope.

Every index has a cost:

- additional storage
- additional write work
- maintenance

Therefore every index needs a real query behind it.

---

# 11. Consistency and Concurrency

## Concurrent State Transitions

Suppose a timeout worker and a payment callback both try to update the order.

### Decision

Use an expected-state/version condition.

Example:

```sql
UPDATE orders
SET status = :newStatus,
    version = version + 1
WHERE order_id = :orderId
  AND status = :expectedStatus
  AND version = :version;
```

If zero rows are updated, another actor already changed the order.

---

## Why

It prevents a stale event from overwriting a newer state.

---

## Alternative

Pessimistic row locking.

### Trade-off

Locking can make the transition easy to reason about but may create contention. Optimistic concurrency allows more parallelism but requires conflict handling.

---

# 12. Idempotency

## Create Order

A client can experience:

```text
Order committed
      ↓
Response lost
      ↓
Client retries
```

Without idempotency:

```text
Order A
Order B
```

could be created for the same logical checkout.

### Decision

Use an idempotency key.

The service stores the association between the key and the resulting order.

On retry, return the existing result.

---

# 13. High-Level Architecture

```text
                         Client
                           |
                           v
                     Order Service
                       /       \
                      /         \
                     v           v
              Order Store      Outbox
                                  |
                                  v
                              Event Bus
                           /      |       \
                          v       v        v
                    Inventory  Payment  Shipping
```

## Ownership

Order Service owns order state.

Inventory owns inventory state.

Payment owns payment state.

Shipping owns shipment state.

No service should directly modify another service's database.

---

# 14. Synchronous vs Asynchronous Communication

Not every interaction should use the same communication style.

## Synchronous

Useful when the caller needs an immediate response.

Example:

```text
GET /orders/{id}
```

The caller needs the order now.

---

## Asynchronous

Useful when work can happen later or when we want failure isolation.

Example:

```text
OrderCreated → Event Bus → downstream processing
```

### Decision

Use synchronous APIs for immediate reads/commands where the caller needs an answer.

Use asynchronous events for workflow progression and notifications where immediate completion is unnecessary.

### Trade-off

Asynchronous communication improves decoupling and failure isolation but introduces:

- eventual consistency
- retries
- duplicate events
- ordering concerns
- more complex debugging

---

# 15. Core Flow — Create Order

1. Receive request.
2. Validate input.
3. Check idempotency.
4. Create order and items in one local transaction.
5. Record the event/outbox message required to start downstream processing.
6. Commit.
7. Publish asynchronously through the outbox.
8. Downstream services process their own responsibilities.

The Order Service does not hold a distributed transaction open while waiting for Inventory and Payment.

---

# 16. Core Flow — Inventory and Payment

A possible workflow:

```text
Order Created
      ↓
Inventory Reservation
      ↓
Inventory Reserved
      ↓
Payment Processing
      ↓
Payment Successful
      ↓
Shipping
      ↓
Delivered
```

Failure paths must be explicitly modeled.

For example:

```text
Inventory Failed → Order cannot proceed
Payment Failed   → Order enters payment failure handling
Timeout          → Reconciliation
```

---

# 17. Duplicate and Out-of-Order Events

Assume:

```text
PaymentSucceeded
PaymentPending
```

arrive in that order.

The older `PaymentPending` event must not move the order backward.

Therefore event handling should:

1. identify the order
2. inspect current state/version
3. determine whether the event represents a valid transition
4. ignore/reject stale events when appropriate
5. apply the transition atomically

Duplicate events should produce no duplicate side effect.

---

# 18. Transactional Outbox

Without an outbox:

```text
DB commit succeeds
       ↓
Service crashes
       ↓
Event never published
```

This creates:

```text
Order says CREATED
Downstream services never receive OrderCreated
```

With an outbox:

```text
Same DB transaction:
    create order
    insert outbox event
        ↓
      COMMIT
        ↓
Outbox publisher
        ↓
Event Bus
```

The publisher can retry independently.

---

# 19. Caching

Order reads can become much larger than order writes.

If this becomes a measured bottleneck:

### Decision

Cache read-heavy order summaries or frequently requested order data.

### Why

Reduce repeated database reads.

### Alternative

Read directly from the database.

### Trade-off

Caching improves latency and database capacity but introduces stale data and invalidation complexity.

The authoritative order state remains in the primary storage.

---

# 20. Reliability and Reconciliation

An order may remain in an intermediate state because:

- downstream service is unavailable
- event is lost
- worker crashes
- timeout occurs
- provider response is ambiguous

A reconciler should find these records.

It should not simply say:

> "Older than one hour means failed."

Instead:

1. find stale orders
2. inspect current order state
3. check relevant downstream state
4. determine the valid transition
5. apply it conditionally
6. repair missing side effects where safe
7. alert if the outcome remains ambiguous

---

# 21. Failure Scenarios

| Failure | Impact | Recovery |
|---|---|---|
| DB unavailable | Order cannot be committed | Fail/bounded retry |
| Response lost after commit | Client retries | Idempotency |
| Duplicate event | Same event arrives again | Idempotent processing |
| Out-of-order event | Could regress state | State/version validation |
| Payment success races timeout | Two actors compete | Conditional transition |
| Event publish failure | Downstream misses event | Outbox retry |
| Worker crash | Order may remain stuck | Reconciliation |
| Cache failure | Read optimization unavailable | Read authoritative store |

---

# 22. Scalability and Evolution

Initial architecture:

```text
Order Service
      ↓
Relational/Chosen Authoritative Store
```

Then evolve based on measured bottlenecks.

For read pressure:

```text
Indexes → replicas/cache
```

For large historical data:

```text
Partition/archive
```

For very large write volume:

```text
Partition/sharding or reconsider storage model
```

The important Principal-level question is:

> "What bottleneck are we solving?"

---

# 23. Observability

Monitor:

- order creation QPS
- order read QPS
- p95/p99 latency
- DB latency
- connection pool utilization
- state-transition conflicts
- stuck orders
- event lag
- retries
- DLQ
- reconciliation backlog

Trace:

```text
Client
  ↓
Order
  ↓
Inventory
  ↓
Payment
  ↓
Shipping
```

---

# 24. Security

- authenticate callers
- authorize cancellation
- derive user identity from authenticated context
- protect sensitive payment information
- encrypt communication
- use least privilege
- audit important order changes

---

# 25. Key Architectural Decisions

| Decision | Reasoning | Alternative | Trade-off |
|---|---|---|---|
| Order owns order state | Gives one clear owner | Shared database | Shared ownership creates coupling |
| Store order + items transactionally | Prevents partially created orders | Separate writes | More failure handling |
| Idempotency for create | Makes retries safe | Client-only retry control | Requires persistent key/result |
| Conditional state transition | Protects against races/stale events | Pessimistic locking | Requires conflict handling |
| Async downstream workflow | Failure isolation and decoupling | Fully synchronous workflow | Eventual consistency and workflow complexity |
| Outbox | Prevents lost events after DB commit | Direct publish | Extra publisher/storage |
| Storage chosen from requirements | Avoids SQL-first thinking | Default SQL | Requires explicit evaluation |

---

# 26. Open Questions

1. Can an order contain items from multiple warehouses?
2. Is partial cancellation required?
3. Can cancellation happen after payment succeeds?
4. What happens if payment succeeds after the order times out?
5. What are peak checkout QPS and retention requirements?
6. What ordering guarantees exist for events?
7. What are RPO/RTO requirements?

---

# 27. Interview Summary

> "The Order Service owns the order lifecycle, but it does not own inventory, payment or shipping state. I model the lifecycle explicitly because events and timeout workers can race. I use idempotency for client retries and conditional state transitions for concurrent updates. I choose storage from the workload and access patterns; a relational database is a strong initial choice because order and order-item creation is transactional and customer history is queryable, but I would revisit that decision if the workload demanded a different model. I use asynchronous events for decoupled workflow progression and an outbox where database state and event publication must remain reliably connected."
