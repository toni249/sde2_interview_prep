# Pattern 3 — Dealing with Contention

> **Problem:** Multiple clients want the same scarce resource: the last concert ticket, the last unit of stock, a domain name that just became available. Without coordination, two users get "success" for the same item — a **race condition**.

## The Four Weapons

| Technique | Where | Cost | When |
|-----------|-------|------|------|
| **Pessimistic locking** | DB or distributed lock | Blocks others | High contention, small critical section |
| **Optimistic concurrency (OCC)** | DB row version | Retry on conflict | Low contention, fast reads |
| **Atomic operations** | DB / Redis primitive | Cheap | Simple counters, single-key work |
| **Serial processing via queue** | Single worker per shard | Adds latency | Very hot resource, want throughput |

## 1. Pessimistic Locking

### SQL — `SELECT ... FOR UPDATE`
```sql
BEGIN;
SELECT quantity FROM inventory WHERE sku='X' FOR UPDATE;   -- row-lock
UPDATE inventory SET quantity = quantity - 1 WHERE sku='X';
COMMIT;
```
- The row is locked until commit; other txns wait.
- **Danger:** long-held locks → cascading waits, deadlocks.
- **Rule:** never do external I/O (HTTP, S3) while holding a DB lock.

### Distributed lock — Redis (Redlock or single-node SET NX)
```
SET lock:concert:42:seatA1 <uuid> NX PX 5000    # acquire with 5s TTL
... critical section ...
if GET == <uuid>: DEL                            # release (Lua for atomicity)
```
- Use a **fencing token** (monotonic version) to protect the downstream write from stale lock holders (see Martin Kleppmann's Redlock critique).
- ZooKeeper / etcd give stronger correctness (linearizable) at the cost of latency.

## 2. Optimistic Concurrency Control (OCC)

Store a `version` (or `updated_at`) column. Read, prepare, then conditionally write:
```sql
UPDATE seat
   SET status='BOOKED', version=version+1
 WHERE id=42 AND version=5;             -- fails if someone else won
-- rowsAffected == 0 → retry or return "seat taken"
```

DynamoDB: `ConditionExpression`. MongoDB: `findAndModify` with filter. Cassandra: LWT (`IF version=5`, expensive).

**Great for:** rarely-contested rows (user profile, order edit).
**Bad for:** hot rows (thundering herd of retries on the concert seat).

## 3. Atomic Primitives

- Redis `INCR`, `DECR`, `SETNX`, `HSETNX` — single-key atomicity, no lock needed.
- Postgres `INSERT ... ON CONFLICT DO NOTHING`.
- Kafka log offset — natural sequencer.

**Example — flash sale counter:**
```
remaining = DECR stock:sku42
if remaining < 0:  INCR stock:sku42;  return "sold out"
else:              enqueue order for downstream persistence
```
No locks, `DECR` is atomic, negative values self-correct.

## 4. Serial Processing via Queue

Route all writes for one hot key through one partition/actor:
```
Client → Kafka topic keyed by seatId → single consumer per partition → DB
```
- Guarantees ordering, eliminates contention entirely.
- Trade-off: added latency, harder to scale a single hot key (a Beyoncé ticket).

## Reference Architecture — Concert Ticket Booking

**Two-phase: reserve, then confirm.**

```
1. Client → API: POST /reserve {seatId}
   Redis: SET seat:{id}:reserved user:{uid} NX PX 300000   (5-min hold)
   API → Client: {reservationToken, expiresAt}

2. Client → API: POST /confirm {reservationToken, paymentInfo}
   API: charge Stripe
   API: DB UPDATE seat SET status='SOLD' WHERE id=? AND reserved_by=?
   Redis: DEL seat:{id}:reserved

3. Timeout: reservation expires → Redis TTL fires → seat is free again
```

**Why this shape?**
- Reserving is cheap (Redis SETNX). Payments (slow, external) happen outside the lock.
- If user abandons, TTL frees the seat — no manual cleanup.
- DB is the source of truth; Redis is a fast reservation layer.

## Interview Q&A

**Q: Optimistic vs pessimistic — how do you pick?**
- Contention rate < 1%? OCC — no lock overhead.
- Contention rate > 10%? Pessimistic — OCC retries thrash.
- Very hot single key (>1k QPS)? Serialize via queue or shard the key.

**Q: What is a deadlock and how do you avoid it?**
- Two txns each hold a lock the other wants.
- **Prevent:** always acquire locks in a canonical order (e.g., sort by `seatId ASC`).
- **Detect:** DB detectors abort one (Postgres does this automatically).
- Keep critical sections short; never nest external I/O.

**Q: How do you handle the "Beyoncé problem" — 1M users refreshing for 1000 tickets?**
- **Virtual waiting room:** Cloudflare / Queue-it — issue a queue-position token before letting through.
- **Randomized admission:** first 10k allowed, others get "try again in 30s" (with jitter).
- **Pre-warmed cache** of seat availability, updated via Kafka.
- Reservation via **atomic counter** first (`DECR remaining`) before per-seat lock.

**Q: What if Redis crashes mid-hold?**
- With TTL, all holds expire → all seats free again (better than stuck).
- For stronger guarantees: use ZooKeeper/etcd for ephemeral locks that auto-release on session loss.
- Or persist reservation to the DB and use Redis only as a cache.

**Q: How does DB isolation level matter?**
- `READ COMMITTED` (Postgres default): each statement sees committed data — susceptible to lost updates.
- `REPEATABLE READ`: prevents non-repeatable reads; MySQL default uses gap locks.
- `SERIALIZABLE`: strongest, effectively serializes conflicting txns; use for critical financial code.
- Or explicit locking (`FOR UPDATE`) regardless of level.

**Q: Idempotency key vs lock — which do you use?**
- Idempotency key = client-supplied UUID; server dedupes retries.
- Lock = server-side mutual exclusion.
- **Both** for payments: idempotency key ensures "charge once" across client retries; lock ensures no two servers charge simultaneously.

## Real-World Examples
- **Ticketmaster**: Redis reservation + Kafka for order pipeline + virtual waiting room.
- **BookMyShow**: pessimistic lock on seat rows, TTL-based hold, retry via OCC (see [LLD_03_BookMyShow.md](../../java-lld/LLD_03_BookMyShow.md)).
- **eBay bidding**: OCC on auction version; hot auctions get serialized.
- **Stripe**: idempotency keys + DB row locks; separate outbox for downstream events.
- **Domain registrars**: single writer per TLD with a queue of registration attempts.

## Gotchas
- **Lock leaks**: forgot the `finally { unlock() }` — resource permanently blocked. TTL is your friend.
- **Retrying non-idempotent operations**: `POST /charge` retried after network blip → double charge. Idempotency keys mandatory.
- **Read-modify-write skew**: reading outside the txn then updating inside — the value you saw may already be stale. Re-read inside the txn.
- **Redlock is not a silver bullet**: single-node Redis with TTL + fencing token is safer than most naive Redlock deployments.
