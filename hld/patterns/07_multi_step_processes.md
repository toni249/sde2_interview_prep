# Pattern 7 — Multi-Step Processes (Distributed Workflows)

> **Problem:** A single business operation touches many services: place order → charge card → reserve inventory → ship → notify. Each step can fail. You cannot use one DB transaction across services. How do you keep the whole thing consistent?

## The Core Question — Consistency Model

**Distributed transactions (2PC)** exist but are:
- Slow (two round-trips + coordinator commit).
- Fragile (coordinator crash → participants stuck).
- Not supported across most modern service boundaries.

**Realistic answer:** eventual consistency + explicit compensation. Three main tools:

| Tool                                                    | Idea                                          | Best for                             |
| ------------------------------------------------------- | --------------------------------------------- | ------------------------------------ |
| **Saga (choreography)**                                 | Each service listens to events, emits its own | Small workflows (3-5 steps)          |
| **Saga (orchestration)**                                | Central orchestrator calls each service       | Larger workflows, visibility         |
| **Workflow engine** (Temporal, Step Functions, Cadence) | Durable code that resumes across crashes      | Complex, long-running, human-in-loop |

## Sagas

A saga is a sequence of local transactions. Each has a **compensating transaction** to undo it.

### Choreography (event-driven)
```
OrderService  → publishes OrderPlaced
PaymentService listens → charges → publishes PaymentSucceeded
InventoryService listens → reserves → publishes InventoryReserved
ShippingService listens → schedules → publishes ShipmentScheduled
```

Failure:
```
InventoryService → publishes InventoryFailed
PaymentService listens → refunds → publishes PaymentRefunded
OrderService listens → marks order failed
```

**Pros:** loosely coupled, no central point of failure.
**Cons:** hard to see the full workflow; risk of cyclic dependencies; onboarding new team members is painful.

### Orchestration (central controller)
```
OrderOrchestrator:
  1. call PaymentService.charge()      → on fail: end (nothing to undo)
  2. call InventoryService.reserve()   → on fail: PaymentService.refund()
  3. call ShippingService.schedule()   → on fail: Inventory.release, Payment.refund
  4. mark order complete
```

**Pros:** one place to read the workflow; easier ops.
**Cons:** orchestrator becomes complex; must be durable.

## Workflow Engines (Temporal / Step Functions / Cadence / Airflow)

Write orchestration as **normal-looking code**, engine persists state.
```java
@WorkflowMethod
public OrderResult placeOrder(OrderRequest req) {
    var payment = paymentActivity.charge(req);       // durable
    try {
        inventoryActivity.reserve(req);              // durable
    } catch (Exception e) {
        paymentActivity.refund(payment);             // compensation
        throw e;
    }
    Workflow.sleep(Duration.ofHours(1));             // survives crashes
    return shippingActivity.schedule(req);
}
```

**How it survives crashes:** every activity call and every `Workflow.sleep` is checkpointed to durable storage. Replaying the code from history recreates state.

**Great for:**
- Human-in-the-loop (`Workflow.await` for external signal).
- Multi-day workflows (subscription renewal, KYC).
- Complex retry policies.

## The Outbox Pattern (Reliable Event Publishing)

Common bug: service commits DB row but crashes before publishing the Kafka event → downstream never learns.

**Fix — Outbox table:**
```
Local txn:
  INSERT INTO orders (...)         -- business row
  INSERT INTO outbox (event_id, payload, status='PENDING')

Both commit atomically (single DB txn).

Separate poller (or Debezium CDC on the outbox table):
  SELECT * FROM outbox WHERE status='PENDING'
  publish to Kafka
  mark as SENT
```

Guarantees at-least-once event publish tied to the DB write.

## Idempotency Everywhere

Every step must tolerate being retried:
- **Idempotency key** on API endpoints (`Idempotency-Key: uuid-v4` header, common in Stripe).
- Conditional writes (`IF NOT EXISTS`, DB unique constraints).
- Natural idempotency: `SET x=10` is idempotent, `INCR x` is not — prefer the former.

## Reference Architecture — E-commerce Order

Orchestration via Temporal:

```
placeOrder workflow:
  1. inventoryActivity.reserve(items)         [30s timeout, 3 retries]
  2. paymentActivity.authorize(cardId, total) [10s timeout, 3 retries]
  3. inventoryActivity.commit()               [best-effort]
  4. paymentActivity.capture()                [best-effort]
  5. shippingActivity.schedule()              [start async pipeline]
  6. notifyActivity.email('order confirmed')

Compensations (on failure at step N):
  step 4 fails → refund + release inventory + email failure
  step 3 fails → void auth + release inventory + email failure
  step 2 fails → release inventory + email failure
  step 1 fails → email out-of-stock
```

**Observability:** Temporal UI shows every workflow's current step, retry history, and pending timers. Priceless in prod.

## Interview Q&A

**Q: 2PC vs Saga — how do you choose?**
- 2PC needs all participants to support prepare/commit (few services do). Coordinator crash = stuck txns.
- Saga is the pragmatic default. You give up atomic isolation; you gain resilience and horizontal scale.

**Q: What's an intermediate state a user might see?**
- Payment charged but order pending → show "processing".
- Order placed but not yet shipped → show "confirmed, preparing".
- Compensating: "your card was refunded because item went out of stock".
- Never expose the raw distributed-system state; wrap in user-friendly stages.

**Q: How do you handle a compensation that fails?**
- **Retry the compensation** with exponential backoff.
- Escalate to human if retries exhausted (customer service ticket).
- Never let workflow silently give up — durable state must always converge.

**Q: What if two compensations conflict (e.g., refund succeeded but inventory release failed and item was already sold to someone else)?**
- Order refunded, second buyer gets the item — that's fine.
- Design workflows so compensations are always feasible.

**Q: How do you avoid double-charging on retries?**
- Client sends `Idempotency-Key`. Payment service dedupes by key.
- Store txn status: attempting → success/failure. Retry checks status first.

**Q: When would you *not* use a workflow engine?**
- Single-service, single-DB operation — just use a DB transaction.
- Extremely high throughput (Temporal has overhead ~1-5ms per activity).
- Very short workflows (few steps, no waits) — Saga via events is enough.

**Q: How do you version workflows in production?**
- Running instances keep running with old code (event-sourced replay).
- New starts use new code. Temporal has `patched()` markers to branch behavior.
- Never rename or reorder activities in an already-running workflow — replay breaks.

**Q: How do you test a multi-step workflow?**
- Unit test each activity in isolation.
- Test the orchestration logic with a fake clock and mocked activities (Temporal has this).
- Chaos: kill workers mid-workflow, verify recovery.

## Real-World Examples
- **Amazon**: fulfillment pipeline runs as multi-step process across dozens of services; each step is an eventual-consistency handoff.
- **Uber**: Cadence (open-sourced → Temporal fork) orchestrates ride matching, pricing, payment.
- **DoorDash**: order lifecycle via Kafka events + Postgres outbox; Temporal for delivery ops.
- **Netflix**: Conductor workflow engine for content ingestion pipelines.
- **Stripe**: idempotency keys + async webhook delivery with retries + dead-letter for failed webhooks.

## Related Patterns
- **Event sourcing**: store all events, derive state — natural fit for workflows.
- **CQRS**: separate write model (workflow state) from read model (query optimized).
- **Outbox / Inbox**: reliable event publish and consume.

## Gotchas
- **Long-lived transactions**: don't hold DB row locks across service calls. Ever.
- **Compensation isn't atomic**: refund succeeds, but the "notify user" step fails. Design each compensation to be independently idempotent and retryable.
- **Event ordering**: choreography relies on events arriving; ordering guarantees depend on the broker (Kafka per-partition, RabbitMQ per-queue).
- **Duplicate events**: consumers must dedupe. Track `processed_event_ids` with TTL.
- **Cycles in choreography**: service A emits event that A also consumes. Watch topology diagrams for loops.
- **Timeouts pyramid**: workflow timeout > sum of activity timeouts + retries + backoff.
