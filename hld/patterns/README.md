# System Design Patterns — Master Index

Curated deep-dive on the 8 core system design patterns from Hello Interview, expanded with concrete examples, trade-offs, and interview follow-ups.

> **How to use this folder:** Each pattern file is standalone. Read the problem statement → skim the "approaches" table → focus on the approach that fits your interview scenario → memorize the trade-offs and follow-up questions.

## The 8 Patterns

| # | Pattern | Solves | Key Examples |
|---|---------|--------|--------------|
| 1 | [Pushing Real-Time Updates](01_realtime_updates.md) | Server → client push | WhatsApp, Google Docs, live scores |
| 2 | [Managing Long-Running Tasks](02_long_running_tasks.md) | Sync API can't wait for slow work | Video transcode, PDF export, ML training |
| 3 | [Dealing with Contention](03_contention.md) | Many writers on one resource | Ticketing, auctions, inventory |
| 4 | [Scaling Reads](04_scaling_reads.md) | Read-heavy load (10:1+) | Instagram feed, news, product catalog |
| 5 | [Scaling Writes](05_scaling_writes.md) | Write-heavy load, hot shards | Metrics ingestion, tweets, IoT |
| 6 | [Handling Large Blobs](06_large_blobs.md) | Files too big for API pipe | Video upload, image hosting, backups |
| 7 | [Multi-Step Processes](07_multi_step_processes.md) | Distributed workflow with state | Order fulfillment, payments, onboarding |
| 8 | [Proximity-Based Services](08_proximity.md) | "Nearby X" queries at scale | Uber, Yelp, Tinder, Gopuff |

## How Patterns Combine

A real system usually stitches several patterns together. Example — **Uber Eats**:

- Proximity (pattern 8) to find nearby restaurants
- Real-time updates (1) for order status and driver location
- Long-running task (2) for background invoice/receipt PDFs
- Contention (3) on the last available slot at a busy restaurant
- Multi-step process (7) for order → prepare → assign → deliver → pay
- Large blobs (6) for restaurant menu images
- Scaling reads (4) on the restaurant catalog
- Scaling writes (5) on driver GPS pings

## Decision Cheat-Sheet

**"How do I…"**

| Requirement | Reach for |
|-------------|-----------|
| Notify a user *now* about something | Real-time updates (1) |
| Return a response before work completes | Long-running tasks (2) |
| Prevent double-booking / double-spend | Contention (3) |
| Fan out identical read load | Scaling reads (4) |
| Write more than one node can handle | Scaling writes (5) |
| Move files >1 MB through the system | Large blobs (6) |
| Coordinate steps across services with retries | Multi-step (7) |
| Find "things near me" | Proximity (8) |

## Related Study
- [Rate Limiter LLD](../../java-lld/LLD_06_Rate_Limiter.md) — pairs with pattern 5
- [LRU Cache LLD](../../java-lld/LLD_07_LRU_Cache.md) — pairs with pattern 4
- [BookMyShow LLD](../../java-lld/LLD_03_BookMyShow.md) — pairs with pattern 3

Source: [Hello Interview — System Design in a Hurry: Patterns](https://www.hellointerview.com/learn/system-design/in-a-hurry/patterns)
