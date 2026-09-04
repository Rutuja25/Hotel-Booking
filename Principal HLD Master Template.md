# Principal Software Engineer HLD — Master Template

## How to use this document

This template is designed to help you **think through** an HLD in a Principal Software Engineer interview.

It is intentionally not a list of technologies to mention.

A strong Principal-level answer should make the interviewer feel:

> "This person understands the problem, identifies the important constraints, evaluates alternatives, and chooses architecture based on evidence."

For every major architectural decision, use this structure:

**Decision → Reasoning → Alternative → Trade-off**

Do not write:

> "Use Kafka because the system is distributed."

Instead explain:

> "We need asynchronous processing because the caller does not need the result immediately and we want to isolate the API from downstream failures. A queue gives us buffering and retry capability. A synchronous call would be simpler, but it would couple the availability and latency of the two services."

---

# 1. Problem Statement

Explain the system in simple business language.

Answer:

- What problem are we solving?
- Who uses the system?
- What does this service own?
- What does it not own?
- Which other systems does it depend on?

A reader who knows nothing about the system should understand its purpose after reading this section.

---

# 2. Requirements

## 2.1 Functional Requirements

Separate requirements into:

### Reads

What information must users or other services retrieve?

### Writes

What business operations must the system perform?

### Background/Internal Operations

What work is performed asynchronously or by internal workers?

For each important operation, ask:

> What happens if the caller sends the same request twice?

---

## 2.2 Non-Functional Requirements

Only include requirements that can influence architecture.

Examples:

- availability
- latency
- throughput
- consistency
- durability
- retention
- RPO
- RTO
- security/compliance

Do not invent precise numbers without stating that they are assumptions.

---

## 2.3 Assumptions

Clearly distinguish interviewer-provided facts from assumptions.

Example:

> "I will assume reservations expire after 15 minutes. If the business requires a different TTL, the state/reconciliation design can be adjusted."

---

## 2.4 Out of Scope

Explicitly exclude features that are not necessary for the problem.

This prevents the design from becoming unnecessarily large.

---

# 3. Scale and Workload Estimation

Estimate before choosing infrastructure.

Consider:

### Traffic

- average QPS
- peak QPS
- read/write ratio
- traffic bursts

### Data

- number of records
- record size
- records created per day
- retention
- growth over time

### Access Patterns

- most frequent reads
- most frequent writes
- range queries
- sorting
- aggregations
- hot keys

### Bottleneck Hypothesis

Ask:

> "What will probably become the first bottleneck?"

It could be:

- CPU
- storage IOPS
- database connections
- network
- a hot row/key
- queue throughput
- an external provider's rate limit

Do not add scaling technology until you can identify the problem it is solving.

---

# 4. Domain Model

Before thinking about tables, understand the business objects.

For each important entity explain:

- what it represents
- important attributes
- relationships
- immutable fields
- mutable fields
- lifecycle

This gives us the business model before we choose a storage representation.

---

# 5. Business Invariants

An invariant is a rule that must remain true.

Examples:

> Inventory available quantity must never become negative.

> A successful payment must not later become failed because of an old callback.

> Retrying the same request must not create a second charge.

For every important invariant answer:

1. What is the rule?
2. What can violate it?
3. Where do we enforce it?
4. What happens if enforcement fails?

A Principal-level design should not merely state invariants. It should explain **how they are protected**.

---

# 6. State Machine

Use this section when an entity has a meaningful lifecycle.

Examples:

- order
- payment
- reservation
- booking
- shipment
- job

## 6.1 States

List the meaningful states.

## 6.2 Events

Explain what causes a state change.

## 6.3 Transition Table

| Current State | Event | New State | Actor | Side Effect | Invalid Cases |
|---|---|---|---|---|---|

The table should be complete enough that another engineer could implement the state machine from it.

## 6.4 Important Race Conditions

Describe situations where two actors can try to change the same entity.

Example:

> A timeout worker tries to mark a payment failed at the same time that a provider webhook says the payment succeeded.

Explain which transition wins and why.

## 6.5 Draw.io State Diagram

**Place the Draw.io diagram here.**

The recommended order is:

**State definitions → transition table → race conditions → Draw.io diagram**

This is better than drawing the diagram first because the table forces us to reason about every transition and failure path.

---

# 7. APIs

Now expose the business operations.

For every important API explain:

- purpose
- request
- response
- validation
- errors
- authentication/authorization
- idempotency
- retry behavior

Prefer business operations such as:

`POST /orders/{id}/cancel`

instead of:

`PUT /orders/{id}/status = CANCELLED`

The service should own the rules for whether a transition is allowed.

---

# 8. Access Patterns

Write down the actual queries the system needs.

Example:

> "The most common read is the latest 20 orders for a customer."

Then translate that into a storage requirement:

> "We therefore need efficient lookup by customer and ordering by creation time."

Access patterns should influence the database decision.

---

# 9. Consistency, Concurrency and Idempotency

This section explains what happens when distributed systems behave imperfectly.

Always consider:

### Duplicate Request

What if the client retries?

### Lost Response

What if the server commits but the client never receives the response?

### Concurrent Update

What if two requests update the same entity?

### Duplicate Event

What if the same event is delivered twice?

### Out-of-Order Event

What if an older event arrives after a newer event?

### Timeout

What if a downstream call times out but may actually have succeeded?

For each major mechanism explain:

**Decision**

**Reasoning**

**Alternative**

**Trade-off**

---

# 10. Database / Storage Decision

## Important Rule

**Do not choose SQL first.**

Also do not choose NoSQL first.

First derive the storage requirements from:

- domain
- invariants
- access patterns
- consistency
- transactions
- workload
- scale
- latency
- availability
- retention

Then identify the few storage models that are genuinely credible.

Possible categories:

- relational
- key-value
- document
- wide-column
- time-series
- object storage
- search index

You do not need to discuss every database in the world.

Discuss the alternatives that could realistically work.

## Required Decision Format

### Decision

What are we choosing?

### Reasoning

Which requirements make this a good fit?

### Alternative

What is the strongest alternative?

### Trade-off

What does our decision make easier, and what complexity/cost does it introduce?

### Why Not the Alternative?

Explain the most important reason the alternative is less suitable.

### Evolution

What future workload would make us reconsider the choice?

---

# 11. Schema and Data Model

Only after selecting the storage model should we define the physical representation.

Explain:

- primary key
- uniqueness
- relationships
- important fields
- indexes
- partition key if relevant
- TTL/retention if relevant

For every important index:

**Query Pattern → Index → Benefit → Cost**

Never add an index just because a column might be searched.

---

# 12. High-Level Architecture

Now draw the system.

Start with the minimum:

```text
Client
  ↓
API / Service
  ↓
Authoritative Storage
```

Add components only when requirements justify them:

- cache
- queue
- worker
- event bus
- external provider
- search engine
- object storage

For every component ask:

> "What problem does this component solve?"

Also identify the **source of truth**.

A cache, search index or event stream is usually not automatically the source of truth.

---

# 13. Synchronous vs Asynchronous Communication

For every important interaction decide separately.

## Synchronous

Use when the caller genuinely needs an immediate answer.

## Asynchronous

Use when the work can happen later or when buffering, decoupling, retries or failure isolation are valuable.

Do not make the entire system asynchronous simply because a message broker is available.

Use:

**Decision → Reasoning → Alternative → Trade-off**

---

# 14. Core Flows

Explain the most important workflows step by step.

At minimum cover:

1. happy path
2. dependency failure
3. timeout
4. retry
5. duplicate request
6. duplicate event
7. out-of-order event
8. concurrent update

For complex interactions, use a sequence diagram.

---

# 15. Caching

First establish that caching is actually necessary.

### Decision

What data are we caching?

### Reasoning

Why are database reads too expensive or too slow?

### Cache Key

How is the object identified?

### TTL

How long can stale data be tolerated?

### Invalidation

How does the cache become fresh?

### Failure

What happens if the cache is unavailable?

### Trade-off

Caching improves read performance but adds:

- stale data
- invalidation complexity
- cache misses
- stampede risk
- another operational dependency

Never allow a stale cache to make a critical correctness decision unless the business explicitly permits it.

---

# 16. Messaging and Events

Explain why messaging is needed.

Potential reasons:

- asynchronous work
- buffering
- decoupling
- retries
- fan-out
- failure isolation

Then define:

- command vs event
- producer
- consumer
- delivery semantics
- duplicate handling
- ordering
- retry
- DLQ
- event versioning

## Transactional Outbox

Use an outbox when a database change and an event must not become permanently inconsistent.

Typical flow:

1. Business transaction updates the database.
2. The same transaction inserts an outbox record.
3. Transaction commits.
4. Publisher reads the outbox.
5. Publisher publishes the event.
6. Failed publication is retried.

---

# 17. Reliability and Recovery

Consider:

- timeout
- bounded retry
- exponential backoff
- circuit breaking where useful
- worker crash
- database failure
- broker failure
- external dependency failure
- reconciliation
- backup/restore
- disaster recovery

## Reconciliation

A reconciler should be a safety mechanism.

It should not blindly change state based only on age.

A good reconciliation flow is:

1. identify suspicious/stuck records
2. inspect current state
3. consult the authoritative dependency if necessary
4. determine whether a valid transition exists
5. apply the transition safely
6. repair missing side effects when safe
7. alert if the outcome remains ambiguous

---

# 18. Scalability and Evolution

Start with the simplest architecture that satisfies today's requirements.

Then identify:

> What breaks first as traffic/data grows?

Possible evolution:

```text
Simple baseline
      ↓
Measure bottleneck
      ↓
Targeted optimization
      ↓
Partitioning / replication / caching / async processing
      ↓
Sharding only if genuinely necessary
```

Never introduce sharding merely because the system might become large.

Before sharding explain:

- current bottleneck
- partition key
- hotspot risk
- cross-partition operations
- operational complexity

---

# 19. Failure Scenarios

| Failure | Impact | Detection | Immediate Handling | Recovery |
|---|---|---|---|---|

Include distributed failures such as:

- database commit succeeded but response was lost
- event publication failed after commit
- duplicate event
- out-of-order event
- downstream timeout
- worker crash
- cache failure
- broker failure
- external provider ambiguity

---

# 20. Observability

## Metrics

Include metrics that reveal system health and business correctness:

- QPS
- error rate
- p50/p95/p99 latency
- dependency latency
- database latency
- connection pool saturation
- queue lag
- retry count
- DLQ size
- reconciliation backlog
- state-transition conflicts

## Logs

Useful context:

- request/correlation ID
- entity ID
- event ID
- relevant business context

Never log secrets or sensitive credentials.

## Tracing

Trace important cross-service workflows.

---

# 21. Security

Cover only relevant concerns:

- authentication
- authorization
- service-to-service identity
- encryption
- secret management
- least privilege
- audit logging
- sensitive-data handling
- webhook verification

---

# 22. Key Architectural Decisions

Finish with a concise decision table.

| Decision | Reasoning | Alternative | Trade-off |
|---|---|---|---|

Include only decisions that materially affect the architecture.

Examples:

- storage model
- consistency model
- concurrency mechanism
- synchronous vs asynchronous interaction
- cache
- messaging
- outbox
- partitioning
- source-of-truth ownership

---

# 23. Open Questions

Only list questions whose answers could change the architecture.

Good:

> Can one reservation span multiple warehouses?

> What is the external provider's idempotency guarantee?

> What is peak traffic?

Avoid questions that are actually implementation choices.

Bad:

> Should we use Redis?

First determine whether caching is needed.

---

# 24. Future Evolution

Explain:

1. What we build today.
2. Why it satisfies today's requirements.
3. What bottleneck we expect first.
4. How we would evolve when that bottleneck appears.

This demonstrates Principal-level thinking without overengineering the initial system.

---

# Principal-Level Mental Model

Always think in this order:

**Problem → Requirements → Workload → Domain → Invariants → State → APIs → Access Patterns → Consistency → Storage → Architecture → Failure Handling → Scalability**

And for every major decision:

**Decision → Reasoning → Alternative → Trade-off**

The technology is the conclusion of the reasoning, not the starting point.
