# Inventory Service — Principal HLD

## 1. Problem Statement

The Inventory Service keeps track of how many units of an item are available at each warehouse.

Its most important responsibility is **correctness**:

> Two customers must not both successfully reserve the same unit.

The service therefore manages both inventory quantity and the lifecycle of inventory reservations.

The Inventory Service owns:

- inventory quantity
- inventory reservation state
- reservation expiration

It does not own:

- orders
- payments
- product pricing
- shipping

Those are owned by other services.

---

# 2. Requirements

## 2.1 Functional Requirements

### Check availability

A caller should be able to ask:

> "How many units of item X are available at warehouse Y?"

or:

> "Which warehouses have item X available?"

### Reserve inventory

Temporarily hold a quantity for an order.

### Confirm reservation

Once the order/payment workflow reaches the required point, convert the reservation into a confirmed allocation.

### Release reservation

Return the held quantity when an order is cancelled or the reservation expires.

### Get reservation status

Allow the caller to determine whether a reservation is active, confirmed, released or failed.

---

## 2.2 Non-Functional Requirements

The critical requirement is:

> **Available inventory must never become negative.**

Other requirements:

- reservation operations should have predictable low latency
- duplicate requests must be safe
- concurrent reservations must be handled correctly
- reservations that become stuck must eventually be detected
- historical data should not unnecessarily burden the hot operational store

---

## 2.3 Assumptions

For this design:

- one inventory record represents one `(warehouse, item)` combination
- reservations expire after a configurable TTL
- the reservation API is idempotent
- the initial design does not require a single atomic reservation across multiple warehouses

The last assumption is important. If a single request must reserve inventory from multiple warehouses atomically, the transaction and data model become significantly more complex.

---

# 3. Scale and Workload

Before selecting storage, estimate:

- availability-read QPS
- reservation-write QPS
- peak reservation QPS
- number of warehouses
- number of inventory records
- reservations per day
- retention period
- read/write ratio

## Important workload characteristic

Inventory has a potential **hot-key problem**.

Suppose a very popular product has only 10 units available and thousands of customers attempt to reserve it simultaneously.

The challenge is not simply:

> "Can the database handle 50,000 requests/sec?"

The harder question is:

> "How do we safely coordinate thousands of writes competing for the same inventory record?"

This observation directly influences the concurrency design.

---

# 4. Domain Model

## Inventory

Represents the current quantity of an item in one warehouse.

Conceptually:

```text
Inventory
---------
warehouseId
itemId
availableQuantity
reservedQuantity
version
updatedAt
```

Business uniqueness:

> There should normally be exactly one inventory record for a `(warehouseId, itemId)` pair.

---

## Reservation

Represents one logical request to hold inventory.

```text
Reservation
-----------
reservationId
orderId
status
idempotencyKey
expiresAt
createdAt
updatedAt
```

---

## Reservation Item

If a reservation can contain multiple inventory records:

```text
ReservationItem
---------------
reservationId
inventoryId
quantity
```

This separates the reservation lifecycle from the physical inventory record.

---

# 5. Business Invariants

## Invariant 1 — Available quantity cannot become negative

The system must never allow:

```text
availableQuantity < 0
```

This is the most important invariant.

### Why application-only validation is insufficient

Imagine:

```text
available = 1
```

Two requests both read `1`.

Both applications conclude:

> "There is enough inventory."

If they then independently subtract one, we could end up at `-1`.

Therefore the final quantity check must happen atomically with the update.

---

## Invariant 2 — The same reservation cannot consume inventory twice

A retry of:

> "Reserve 2 units"

must not reserve another 2 units.

---

## Invariant 3 — Terminal reservations do not move backward

For example:

```text
CONFIRMED → RELEASED
```

should not happen unless the business explicitly supports that transition.

---

## Invariant 4 — Idempotency

The same logical request should produce the same logical reservation result.

---

# 6. State Machine

## States

```text
CREATED
RESERVED
CONFIRMED
RELEASED
FAILED
```

## Transition Table

| Current State | Event | New State | Actor | Side Effect |
|---|---|---|---|---|
| CREATED | reservation succeeds | RESERVED | Inventory Service | decrease available, increase reserved |
| CREATED | insufficient inventory | FAILED | Inventory Service | no inventory held |
| RESERVED | confirm | CONFIRMED | Inventory Service | reservation becomes final |
| RESERVED | cancel | RELEASED | Inventory Service | return reserved quantity |
| RESERVED | expiration | RELEASED | Reconciliation/Worker | return reserved quantity |
| Terminal | duplicate command | Same state | Inventory Service | no second side effect |

---

## Important Race: Confirm vs Expire

Suppose a reservation reaches its expiry time at the same moment that an order tries to confirm it.

We cannot allow both operations to succeed.

The state transition must be conditional on the expected current state/version.

For example:

```text
RESERVED + Confirm → CONFIRMED
```

and:

```text
RESERVED + Expire → RELEASED
```

Only one should win.

---

## Draw.io State Diagram

**Place the Draw.io diagram here.**

The diagram should show:

```text
CREATED
   |
   +---- success ----> RESERVED
   |                       |
   |                       +---- confirm ---> CONFIRMED
   |                       |
   |                       +---- cancel ----> RELEASED
   |                       |
   |                       +---- expire ----> RELEASED
   |
   +---- insufficient ----> FAILED
```

---

# 7. APIs

## Check Availability

```http
GET /inventory/availability?itemId=123&warehouseId=456
```

Returns the current authoritative availability.

This response can potentially be slightly stale if used only for display, but it must not be used as the final authorization for a reservation.

---

## Create Reservation

```http
POST /inventory/reservations
Idempotency-Key: abc123
```

Request:

```json
{
  "orderId": "order-123",
  "items": [
    {
      "itemId": "item-1",
      "warehouseId": "warehouse-1",
      "quantity": 2
    }
  ]
}
```

The idempotency key is important because the client may retry after a timeout.

---

## Confirm Reservation

```http
POST /inventory/reservations/{id}/confirm
```

This is a business operation rather than:

```http
PUT /inventory/reservations/{id}/status
```

because the Inventory Service should decide whether confirmation is currently valid.

---

## Release Reservation

```http
POST /inventory/reservations/{id}/release
```

---

## Get Reservation

```http
GET /inventory/reservations/{id}
```

---

# 8. Access Patterns

Important queries:

1. Find inventory by `(warehouseId, itemId)`.
2. Find warehouses containing a given item.
3. Find reservation by reservation ID.
4. Find reservation by idempotency key.
5. Find active reservations whose expiration time has passed.
6. Atomically decrease available quantity.

These access patterns are inputs to the storage decision.

---

# 9. Concurrency Design

## Problem

Two customers attempt to reserve the final unit.

Starting state:

```text
available = 1
```

Both requests arrive simultaneously.

A naïve implementation:

```text
read available
if available >= requested:
    update available
```

is unsafe because both requests can read the same value.

---

## Decision

Perform the quantity check and update atomically.

A relational implementation could look like:

```sql
UPDATE inventory
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE inventory_id = :inventoryId
  AND available_quantity >= :quantity;
```

The application checks the number of affected rows.

- `1 row` → reservation succeeded
- `0 rows` → insufficient inventory or the record changed

If version-based optimistic concurrency is also required, add a version condition.

---

## Why

The correctness rule is enforced at the point where the value changes.

Application code can decide whether to retry or return failure, but the storage operation prevents two requests from consuming the same unit.

---

## Alternative — Pessimistic Lock

Another approach is:

```text
BEGIN
SELECT inventory FOR UPDATE
check quantity
update quantity
COMMIT
```

The database locks the row while the transaction runs.

### Advantage

The behavior is straightforward to reason about.

### Disadvantage

For a hot product, many transactions may wait for the same row.

### Trade-off

Pessimistic locking gives simple correctness but can create contention. Conditional/optimistic updates provide more concurrency but require explicit conflict handling.

---

# 10. Idempotency

## Problem

The service successfully reserves inventory.

Before the response reaches the client, the network fails.

The client retries.

Without idempotency:

```text
First request → reserve 2
Retry         → reserve another 2
```

We have incorrectly reserved 4 units.

---

## Decision

Store the idempotency key with the reservation request and make the relevant uniqueness scope explicit.

On retry:

1. find the existing request
2. if it already completed, return its previous result
3. if it is still processing, handle according to the chosen request semantics
4. never perform the inventory side effect twice

---

## Alternative

Require clients never to retry.

### Trade-off

Simpler server implementation, but unrealistic for distributed networks where timeouts and lost responses are normal.

---

# 11. Database / Storage Decision

## First principle

We should **not start with "Inventory means SQL."**

We first derive the storage requirements.

The system needs:

- atomic quantity changes
- strong protection of the non-negative inventory invariant
- predictable point updates
- uniqueness for `(warehouseId, itemId)`
- reservation lookup
- expiration queries
- possibly transactions across reservation and inventory records

---

## Candidate 1 — Relational Database

A relational database fits naturally because it provides:

- atomic updates
- transactions
- uniqueness constraints
- conditional updates
- mature indexing
- straightforward operational tooling

It is especially attractive when inventory correctness is more important than extreme write scale.

---

## Candidate 2 — Key-Value / Distributed NoSQL

A key-value store can be attractive when:

- the access pattern is mostly point lookup/update
- the system needs very high horizontal scale
- the data model can be simplified around keys
- the chosen database provides suitable atomic/conditional operations

However, multi-record transactional behavior and constraints may become harder depending on the technology.

---

## Decision

For a moderate-to-high scale transactional inventory system where correctness and conditional updates are central, a relational database is a strong starting choice.

This is **not because SQL is always better**.

It is because the current requirements make transactional and conditional semantics particularly valuable.

---

## Trade-off

We gain simpler correctness and transactional behavior.

We accept the need to scale the relational database if inventory traffic grows substantially.

If measurements later show that the relational store is the bottleneck, we should reconsider the storage model rather than automatically adding more infrastructure.

---

## What Could Change the Decision?

The decision should be revisited if:

- inventory writes become too large for the chosen database architecture
- one inventory key becomes extremely hot
- access patterns become much simpler
- global multi-region requirements change
- availability requirements make a different distributed model preferable

---

# 12. Schema and Indexes

A relational implementation might use:

### `inventory`

```text
inventory_id
warehouse_id
item_id
available_quantity
reserved_quantity
version
created_at
updated_at
```

Unique constraint:

```text
(warehouse_id, item_id)
```

This prevents two inventory records from accidentally representing the same warehouse/item combination.

---

### `reservation`

```text
reservation_id
order_id
status
idempotency_key
expires_at
created_at
updated_at
```

Potential uniqueness:

```text
idempotency_key
```

depending on its required scope.

---

### Indexes

For:

> Find inventory for one warehouse and item.

Use:

```text
(warehouse_id, item_id)
```

For:

> Find active reservations that have expired.

Potentially use:

```text
(status, expires_at)
```

For:

> Show warehouses where an item is available.

A query/index such as:

```text
(item_id, available_quantity)
```

may help if this is a frequent operation.

But this index should only be added if the workload justifies it because every additional index increases write and storage cost.

---

# 13. High-Level Architecture

A simple initial architecture:

```text
                Client / Order Service
                         |
                         v
                +-------------------+
                | Inventory Service |
                +---------+---------+
                          |
                          v
                 Authoritative Store
```

As requirements grow:

```text
                +-------------------+
                | Inventory Service |
                +----+---------+----+
                     |         |
                     |         +----------------+
                     |                          |
                     v                          v
             Authoritative Store          Outbox
                                               |
                                               v
                                           Event Bus
                                               |
                              +----------------+----------------+
                              |                                 |
                              v                                 v
                        Order Service                    Other Consumers

                    Reconciliation Worker
                              |
                              v
                    Authoritative Store
```

The authoritative store remains the source of truth for inventory mutation.

---

# 14. Caching

Availability reads may be much more frequent than reservation writes.

A cache can therefore be useful for display-oriented reads.

## Decision

Introduce a cache only if measurement shows that availability reads are putting significant load on the authoritative store.

## Why

The cache can serve repeated reads without hitting the database every time.

## Critical Rule

A cached value must **not** be used to authorize the final reservation.

Example:

```text
Cache says: 5 available
Database says: 0 available
```

The database wins.

## Alternative

Always read directly from the authoritative store.

## Trade-off

No cache means simpler consistency but more database reads.

Cache means lower read load/latency but introduces staleness, invalidation and another dependency.

---

# 15. Messaging and Events

Useful events may include:

- `InventoryReserved`
- `InventoryReservationFailed`
- `InventoryConfirmed`
- `InventoryReleased`

Events allow other services to react without directly modifying Inventory's database.

---

## Duplicate Events

Assume duplicate delivery can happen.

For example:

```text
InventoryReserved
InventoryReserved
```

A consumer should not perform its side effect twice.

Therefore consumers need their own idempotency mechanism.

---

# 16. Transactional Outbox

Suppose the Inventory Service does:

```text
1. Update inventory
2. Publish InventoryReserved
```

A failure can occur between the two:

```text
Database update succeeds
        ↓
Service crashes
        ↓
Event never published
```

Now the inventory is correct, but downstream systems do not know about it.

## Decision

Use a transactional outbox when the business requires the event to reliably correspond to the database change.

Flow:

```text
Database Transaction
   |
   +-- update inventory
   |
   +-- insert outbox event
   |
   +-- COMMIT
          |
          v
     Outbox Publisher
          |
          v
       Event Bus
```

The publisher can retry the event later.

---

# 17. Reservation Expiration and Reconciliation

A worker periodically finds reservations that should have expired.

It should not simply do:

> "expiresAt is old → release everything."

It must first verify the current state.

Example:

```text
Worker reads reservation
        ↓
Is it still RESERVED?
        |
      yes
        ↓
Atomically transition RESERVED → RELEASED
        ↓
Return quantity
```

If another request already confirmed the reservation, the expiration operation should not release the inventory.

This is another reason the state transition needs conditional/version protection.

---

# 18. Core Flows

## 18.1 Reserve

1. Receive request.
2. Validate request.
3. Check idempotency key.
4. Identify inventory records.
5. Atomically verify and decrease available quantity.
6. Increase reserved quantity.
7. Create/update reservation state.
8. Record event/outbox if required.
9. Return result.

The exact transaction boundary depends on whether one reservation can contain multiple inventory records.

---

## 18.2 Confirm

1. Load reservation.
2. Verify it is currently `RESERVED`.
3. Atomically transition to `CONFIRMED`.
4. Publish `InventoryConfirmed` if required.
5. Return success.

---

## 18.3 Expire

1. Worker finds expired reservations.
2. Verify reservation is still `RESERVED`.
3. Atomically transition to `RELEASED`.
4. Return reserved quantity to available inventory.
5. Publish `InventoryReleased`.
6. Retry safely if an infrastructure failure occurs.

---

## 18.4 Client Timeout

If reservation succeeded but the response was lost:

```text
Client
  |
  | Reserve(key=A)
  v
Inventory
  |
  | commit
  v
Database
  |
  X response lost
```

Client retries with `key=A`.

Inventory finds the existing reservation and returns the previous result.

No second reservation occurs.

---

# 19. Failure Scenarios

| Failure | What Happens | Recovery |
|---|---|---|
| Database unavailable | Reservation cannot safely commit | Return failure / bounded retry |
| Client timeout after commit | Client retries | Idempotency |
| Two reservations compete for last unit | One atomic update succeeds | Other receives conflict/failure |
| Confirm races with expiration | One valid state transition wins | Conditional update |
| Event publication fails | Database state still exists | Outbox retry |
| Duplicate event | Consumer receives same event twice | Idempotent consumer |
| Worker crashes | Reservation may remain active | Next reconciliation run |
| Cache unavailable | Read optimization lost | Read authoritative store |

---

# 20. Scalability and Evolution

Start simple.

```text
Authoritative DB
      ↓
Proper indexes
      ↓
Connection pooling
```

If read load grows:

```text
Read optimization / cache / replicas
```

If data volume grows:

```text
Partitioning
```

If write volume becomes too large:

```text
Investigate partitioning/sharding or a different storage model
```

Before sharding, identify:

- the actual bottleneck
- partition key
- hot keys
- cross-partition operations
- operational complexity

A popular inventory item can still be a hot key even after sharding if all writes for that item land on one partition.

---

# 21. Observability

Monitor:

- availability-read QPS
- reservation QPS
- reservation success rate
- reservation failure rate
- p95/p99 latency
- database latency
- concurrency conflicts
- number of expired reservations
- reconciliation backlog
- event lag
- DLQ size
- retry count

Most importantly:

> Inventory invariant violations should always be zero.

Trace:

```text
Order → Inventory → Database → Event Bus
```

---

# 22. Security

- authenticate internal callers
- authorize reservation operations
- use service-to-service identity
- encrypt communication
- protect database credentials
- audit important inventory changes
- do not expose internal storage directly to other services

---

# 23. Key Architectural Decisions

| Decision | Reasoning | Alternative | Trade-off |
|---|---|---|---|
| Authoritative store for reservation decisions | Prevent stale reads from causing overselling | Cache as authority | Cache would create correctness risk |
| Atomic conditional quantity update | Protects `available >= 0` during concurrent requests | Pessimistic row locking | Locking can create hot-row contention |
| Idempotency key | Makes retries safe | Client-only retry discipline | Requires persistent idempotency state |
| Relational storage initially | Transactions, constraints and conditional updates fit the workload | Key-value/distributed NoSQL | Relational scaling may eventually require more work |
| Cache only for reads | Reduces read load without changing correctness | No cache | Simpler but more database reads |
| Outbox | Prevents lost events after successful DB commit | Direct publish | Additional storage/publisher |
| Reconciliation | Repairs stuck/incomplete workflows | No reconciler | Simpler but leaves failures unresolved |

---

# 24. Open Questions

1. Can one reservation span multiple warehouses?
2. If yes, must all inventory reservations succeed atomically?
3. What is the reservation TTL?
4. What are peak reservation QPS and availability QPS?
5. Can availability reads be stale?
6. What events must Inventory publish?
7. What ordering guarantees are required?
8. What are RPO/RTO requirements?

---

# 25. Future Evolution

The initial goal is not to build the most distributed inventory system possible.

The goal is to build a system that is:

- correct
- understandable
- observable
- resilient to normal distributed failures
- capable of handling the expected workload

When measurements show a real bottleneck, evolve that specific part.

---

# 26. Interview Summary

A concise Principal-level explanation would be:

> "The hardest part of inventory is not CRUD; it is protecting the invariant that available inventory can never become negative while many customers compete for the same item. I therefore make the quantity check and update atomic and use idempotency for retries. I model reservation as a state machine so confirmation, cancellation and expiration cannot race into invalid states. I choose storage based on these requirements rather than assuming SQL; a relational database is a strong initial choice because transactions, constraints and conditional updates map naturally to the problem. I would add caching only for read optimization and use an outbox if inventory changes must reliably generate events. As the system grows, I would measure the actual bottleneck before introducing partitioning or sharding."
