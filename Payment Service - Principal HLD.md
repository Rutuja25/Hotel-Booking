# Payment Service — Principal HLD

## 1. Problem Statement

The Payment Service manages the application's internal payment lifecycle and communicates with an external payment provider.

Its most important responsibility is preventing an incorrect financial outcome.

The system must avoid:

- charging the customer twice
- marking a successful payment as failed
- losing track of a payment whose provider outcome is uncertain
- processing the same webhook twice

The Payment Service owns our internal payment state.

The external provider owns the actual provider-side transaction outcome.

That distinction becomes critical when network failures occur.

---

# 2. Requirements

## Functional Requirements

### Create/Initiate Payment

Start payment processing for an order.

### Get Payment

Retrieve payment status for an order/payment.

### Handle Provider Webhook

Process asynchronous provider notifications.

### Refund

Initiate and track refunds if supported.

### Reconciliation

Resolve payments whose final outcome is uncertain or whose expected webhook/status update has not arrived.

---

# 3. Scale and Workload

Estimate:

- payment initiation QPS
- webhook QPS
- payment read QPS
- active payment volume
- historical payment volume
- provider rate limits
- reconciliation volume

## Important Observation

Our own service throughput is not the only limit.

For example:

```text
Our service:       50,000 req/s
Provider limit:     5,000 req/s
```

The provider becomes the bottleneck.

Therefore the architecture may eventually need:

- throttling
- queueing
- backpressure
- provider routing
- multiple providers if supported by the business

---

# 4. Domain Model

## Payment

```text
Payment
-------
paymentId
orderId
userId
amount
currency
status
provider
providerPaymentId
idempotencyKey
version
createdAt
updatedAt
```

The exact fields depend on the provider.

---

## Important Distinction

There are two kinds of truth:

### Our internal state

Example:

```text
PAYMENT_PENDING
UNKNOWN
SUCCESS
```

### Provider-side state

Example:

```text
provider transaction = captured
```

When a network timeout occurs, our service may not know the provider state immediately.

Therefore we cannot treat every local timeout as a definitive payment failure.

---

# 5. Business Invariants

## Invariant 1 — No duplicate charge

Retrying the same logical payment must not create a second provider charge.

---

## Invariant 2 — Successful payment must not become failed

A stale webhook or worker must not overwrite a newer successful state.

---

## Invariant 3 — Timeout does not necessarily mean failure

If the provider request times out, we may not know whether the provider processed the payment.

Therefore:

```text
Network timeout ≠ Payment failure
```

---

## Invariant 4 — Duplicate webhook is harmless

Processing the same webhook twice must not:

- change state twice
- create a duplicate refund
- send duplicate business events

---

## Invariant 5 — Amount and currency should be stable

Once payment processing begins, the amount and currency should normally not change unless the business explicitly supports such behavior.

---

# 6. State Machine

## States

A useful lifecycle is:

```text
CREATED
PAYMENT_PENDING
UNKNOWN
SUCCESS
FAILED

REFUND_PENDING
REFUNDED
```

---

## Transition Table

| Current State | Event | New State | Actor |
|---|---|---|---|
| CREATED | Initiate | PAYMENT_PENDING | Payment Service |
| PAYMENT_PENDING | Provider success | SUCCESS | Payment/Webhook |
| PAYMENT_PENDING | Definitive provider failure | FAILED | Payment/Webhook |
| PAYMENT_PENDING | Ambiguous timeout | UNKNOWN | Payment Service |
| UNKNOWN | Provider success | SUCCESS | Webhook/Reconciler |
| UNKNOWN | Provider failure | FAILED | Webhook/Reconciler |
| SUCCESS | Refund requested | REFUND_PENDING | Payment |
| REFUND_PENDING | Refund confirmed | REFUNDED | Payment/Webhook |

---

# 7. Why UNKNOWN Is Important

Consider:

```text
1. Payment Service sends charge request.
2. Provider receives it.
3. Provider charges customer.
4. Network connection fails.
5. Payment Service receives no response.
```

What does the service know?

It knows:

> "I did not receive the response."

It does **not** know:

> "The provider did not charge the customer."

Therefore marking the payment:

```text
FAILED
```

could be dangerous.

The customer may actually have been charged.

---

## Decision

Represent the outcome as `UNKNOWN` when the system cannot establish whether the provider operation succeeded.

Then resolve it using:

- webhook
- provider status API
- reconciliation

---

## Alternative

Immediately mark the payment failed.

### Trade-off

Much simpler state management, but potentially incorrect financial state.

For payments, that is an unacceptable trade-off.

---

# 8. Draw.io State Diagram

**Place the Draw.io diagram here.**

The diagram should emphasize:

```text
PAYMENT_PENDING
       |
       +---- success ------> SUCCESS
       |
       +---- failure ------> FAILED
       |
       +---- timeout ------> UNKNOWN
                              |
                              +---- provider success ---> SUCCESS
                              |
                              +---- provider failure ---> FAILED
```

Refund states can be added if refund is in scope.

---

# 9. APIs

## Create Payment

```http
POST /payments
Idempotency-Key: abc123
```

---

## Get Payment

```http
GET /payments/{paymentId}
```

---

## Get Payment for Order

```http
GET /orders/{orderId}/payment
```

---

## Refund

```http
POST /payments/{paymentId}/refund
```

---

## Provider Webhook

```http
POST /payments/provider-webhook
```

The webhook must be authenticated/verified according to the provider's mechanism.

Do not expose:

```http
PUT /payments/{id}/status
```

because arbitrary callers should not control payment state.

---

# 10. Access Patterns

Important queries:

1. Payment by ID.
2. Payment by order ID.
3. Payment by provider transaction ID.
4. Payment by idempotency key.
5. Customer payment history.
6. Payments stuck in `PENDING` or `UNKNOWN`.
7. Refund by payment/reference ID.

These access patterns influence the storage decision and indexes.

---

# 11. Idempotency

## Problem

Suppose:

```text
Client → Payment Service → Provider
```

The provider successfully charges the customer.

But our request times out.

The client retries.

If we create a new provider transaction:

```text
Attempt 1 → charge
Attempt 2 → charge again
```

The customer may be charged twice.

---

## Decision

Use a stable idempotency/reference key for the logical payment operation.

Where the provider supports idempotency, pass that same logical key to the provider.

Internally, also record the association between the key and our payment.

---

## Why

Both our service and the provider can recognize that a retry represents the same logical operation.

---

## Alternative

Create a new provider transaction on every retry.

### Trade-off

Simpler implementation but unacceptable duplicate-charge risk.

---

# 12. Webhook Handling

A provider webhook may arrive:

- before our synchronous response
- after our synchronous response
- more than once
- out of order

Therefore webhook processing must be independently idempotent.

A safe flow:

1. verify webhook authenticity
2. identify provider transaction/event
3. find the internal payment
4. determine whether the event was already processed
5. check the current payment state
6. apply only a valid transition
7. publish downstream events safely

---

# 13. Webhook vs Reconciliation Race

Suppose:

```text
Webhook → SUCCESS
```

arrives while:

```text
Reconciler → believes payment failed
```

is being processed.

### Decision

Both operations must use the same state-transition rules.

A stale operation must not overwrite a newer valid state.

### Alternative

Serialize every payment operation using a long-held lock.

### Trade-off

Locks can simplify reasoning but reduce concurrency and increase contention. Conditional transitions require more explicit logic but scale better.

---

# 14. Database / Storage Decision

## Storage Requirements

We need:

- durable payment state
- uniqueness
- idempotency
- conditional state transitions
- lookup by order
- lookup by provider transaction
- customer payment history
- auditability
- reconciliation queries

---

## Relational Database

A relational database is a strong candidate because:

- payment state is transactional
- uniqueness constraints are useful
- conditional updates map naturally to state transitions
- payment/order relationships are useful
- operational/audit queries are straightforward

---

## Key-Value / Distributed NoSQL

A distributed key-value system could work if:

- the access pattern is predominantly point lookup
- extremely high scale is required
- the chosen technology provides appropriate conditional writes
- relationships and complex queries are limited

But uniqueness, state coordination and some consistency requirements may become application responsibilities.

---

## Decision

For a transactional payment workflow with strong correctness requirements, a relational database is a strong initial choice.

Again, this is **not "SQL first."**

The decision comes from the required behavior.

If the system later reaches a scale or availability requirement where the relational model becomes the bottleneck, evaluate alternatives against the same requirements.

---

# 15. Schema and Indexes

Possible `payment` fields:

```text
payment_id
order_id
user_id
amount
currency
status
provider
provider_payment_id
idempotency_key
version
created_at
updated_at
```

Potential uniqueness:

```text
(provider, provider_payment_id)
```

if provider semantics guarantee the reference is unique.

---

## Indexes

For:

> Find payment for an order.

Use:

```text
(order_id)
```

For:

> Find customer's recent payments.

Use:

```text
(user_id, created_at DESC)
```

For:

> Find unresolved payments.

Potentially:

```text
(status, updated_at)
```

or an index aligned with the reconciliation query.

Every index must have a real query behind it.

---

# 16. Concurrency and State Transitions

Use an expected-state/version condition.

Example:

```sql
UPDATE payment
SET status = :newStatus,
    version = version + 1
WHERE payment_id = :id
  AND status = :expectedStatus
  AND version = :version;
```

If no row is updated, another actor already changed the payment.

This protects against:

- webhook vs webhook
- webhook vs reconciler
- timeout vs provider response
- duplicate worker processing

---

# 17. High-Level Architecture

```text
                    Order Service
                         |
                         v
                  +--------------+
                  |   Payment    |
                  |   Service    |
                  +------+-------+
                         |
              +----------+----------+
              |                     |
              v                     v
       Authoritative Store    Payment Provider
                                    |
                                    | webhook
                                    v
                              Payment Service
                                    |
                                    v
                                  Outbox
                                    |
                                    v
                                Event Bus

                         Reconciliation Worker
                                  |
                                  v
                           Provider + Store
```

---

# 18. Synchronous vs Asynchronous

## Provider Call

The provider interaction may need to be synchronous when the caller requires an immediate response.

However, a timeout means the outcome may be unknown.

---

## Internal Event Propagation

Events such as:

```text
PaymentSucceeded
PaymentFailed
```

can normally be published asynchronously.

### Decision

Choose synchronous/asynchronous communication per interaction.

### Why

The external provider call and internal event propagation solve different problems.

### Trade-off

Synchronous calls provide immediate feedback but couple latency/availability.

Asynchronous events provide decoupling and buffering but introduce eventual consistency and retry complexity.

---

# 19. Core Flow — Successful Payment

1. Receive payment request.
2. Validate amount/order.
3. Check idempotency.
4. Create `PAYMENT_PENDING`.
5. Send provider request with the stable idempotency/reference key.
6. Provider returns success.
7. Atomically transition to `SUCCESS`.
8. Record/publish `PaymentSucceeded`.
9. Return result.

---

# 20. Core Flow — Provider Timeout

1. Create/keep payment as pending.
2. Send provider request.
3. Request times out.
4. Do not assume failure.
5. Represent the ambiguous outcome as `UNKNOWN` where appropriate.
6. Wait for webhook or query provider status.
7. Apply `UNKNOWN → SUCCESS` or `UNKNOWN → FAILED`.
8. Publish the resulting business event.

---

# 21. Transactional Outbox

Suppose:

```text
Payment state → SUCCESS
```

is committed.

Then:

```text
publish PaymentSucceeded
```

fails.

Without an outbox, downstream systems may never learn about the successful payment.

With an outbox:

```text
Same transaction:
    update payment
    insert PaymentSucceeded outbox record
        ↓
      COMMIT
        ↓
Outbox publisher
        ↓
Event Bus
```

The event can be retried until successfully published.

---

# 22. Caching

Payment status can be read frequently, but stale payment status can be dangerous.

### Decision

Only introduce caching for read optimization when measurements justify it.

### Why

Reduce repeated database reads.

### Alternative

Read the authoritative store directly.

### Trade-off

Caching improves latency and read capacity but introduces staleness and invalidation complexity.

Never use stale cache data to make a critical payment decision.

---

# 23. Reconciliation

A payment can become stuck because:

- provider webhook is delayed
- worker crashes
- provider call times out
- network failure occurs
- event publication fails

The reconciler should identify suspicious payments.

For an `UNKNOWN` payment:

1. query provider status
2. compare provider result with internal state
3. apply a valid transition
4. record the repair
5. retry if provider is unavailable
6. alert if the outcome remains unresolved

The reconciler is a **safety net**, not the primary payment state machine.

---

# 24. Provider Rate Limits and Backpressure

Suppose:

```text
Payment Service → 50,000 requests/sec
Provider         → 5,000 requests/sec
```

Sending all requests immediately will overload the provider.

Possible solutions:

- queue payment work
- rate-limit requests
- apply backpressure
- use provider-specific concurrency limits
- distribute traffic across providers where business rules permit

The correct approach depends on whether the payment operation requires synchronous provider confirmation.

---

# 25. Failure Scenarios

| Failure | What Happens | Recovery |
|---|---|---|
| Database unavailable | Payment state cannot safely change | Fail/bounded retry |
| Client timeout after provider success | Client retries | Idempotency |
| Provider timeout | Outcome uncertain | UNKNOWN + provider lookup/webhook |
| Duplicate webhook | Same event arrives twice | Idempotent processing |
| Out-of-order webhook | Older state may arrive later | State/version validation |
| Event publication failure | Payment committed, event missing | Outbox |
| Provider unavailable | Cannot complete request | Timeout/backpressure/retry |
| Reconciler crashes | Unknown payments remain | Next reconciliation run |
| Cache unavailable | Read optimization lost | Read authoritative store |

---

# 26. Scalability and Evolution

Initial:

```text
Payment Service
      ↓
Authoritative Store
      ↓
Provider
```

Then evolve based on measurements:

```text
Indexes
   ↓
Read optimization
   ↓
Queue/rate limiting for provider pressure
   ↓
Partitioning
   ↓
Sharding or alternative storage if truly necessary
```

The external provider's rate limit may become a more important bottleneck than our database.

---

# 27. Observability

Monitor:

- payment QPS
- success/failure rate
- `UNKNOWN` count
- provider latency
- provider timeout rate
- provider error rate
- provider rate-limit responses
- webhook delay
- duplicate webhook count
- reconciliation backlog
- retry count
- DLQ size
- state-transition conflicts

The most important business correctness metric is:

> Number of duplicate charges caused by our system — target: zero.

---

# 28. Security

Payment systems require strong security.

Include:

- authenticated internal callers
- authorization for refunds
- provider webhook signature verification
- secure provider credentials
- encryption in transit
- least privilege
- audit logging
- strict controls around sensitive payment information

Never log:

- card details
- authentication secrets
- provider credentials
- sensitive payment tokens

---

# 29. Key Architectural Decisions

| Decision | Reasoning | Alternative | Trade-off |
|---|---|---|---|
| Provider idempotency | Prevent duplicate charge during retry | New provider transaction | Unsafe duplicate-charge risk |
| `UNKNOWN` state | Timeout does not prove failure | Immediately mark failed | Simpler but financially unsafe |
| Provider as transaction authority | Provider knows whether its transaction happened | Trust local timeout/response | Local state can be ambiguous |
| Conditional state transitions | Protect webhook/reconciliation races | Long-held locks | More explicit conflict handling |
| Relational storage initially | Transactions, uniqueness and state updates fit requirements | Distributed NoSQL | Potential future scaling work |
| Outbox | Prevents lost business events | Direct publish | Extra publisher/storage |
| Cache is non-authoritative | Prevent stale payment decisions | Cache as authority | More authoritative reads |
| Rate limiting/backpressure | Protect provider dependency | Send everything immediately | May increase latency/queueing |

---

# 30. Open Questions

1. Which provider is being used?
2. What idempotency guarantees does it provide?
3. What happens if a provider request times out?
4. Can an order have multiple payment attempts?
5. Can payment remain `UNKNOWN` indefinitely?
6. How long should reconciliation continue?
7. Is refund in scope?
8. What are provider rate limits?
9. What webhook ordering guarantees exist?
10. What compliance/audit retention is required?
11. What are RPO/RTO requirements?

---

# 31. Interview Summary

> "The hardest part of payment is not storing a payment record; it is handling ambiguity. A provider timeout does not tell us whether the customer was charged, so I explicitly model an UNKNOWN state and resolve it through webhooks, provider status APIs and reconciliation. I use idempotency both internally and with the provider to prevent duplicate charges. Webhooks, retries and reconciliation can race, so state transitions are conditional and version-aware. I choose storage based on these requirements rather than assuming SQL; a relational database is a strong initial fit because payment state, uniqueness and transactional updates are important. Internal payment events use an outbox when we need the database state and event publication to remain reliably connected."
