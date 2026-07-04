# Pattern 2 — Managing Long-Running Tasks

> **Problem:** A user action triggers work that takes seconds to hours: transcoding a video, generating a 10 GB report, training a model. You cannot hold the HTTP connection open for that long — timeouts, retries, and load balancer limits kick in.

## The Recipe

```
Client → API: "do X"
API:     enqueue Job{id, payload} → Queue
API →   Client: 202 Accepted, {jobId}

Worker: pull from Queue → do work → update JobStatus{id, state, result}

Client polls  GET /jobs/{id}   OR   subscribes via SSE/WS/webhook
```

Three moving parts: **API (enqueue)**, **Queue (durable buffer)**, **Worker (executor)**.

## Queue Choices

| Queue | Delivery | Ordering | Throughput | Use When |
|-------|----------|----------|------------|----------|
| **SQS** | At-least-once (Standard) / Exactly-once (FIFO) | None / FIFO | Millions/sec | AWS default, simple |
| **Kafka** | At-least-once | Per-partition | 10M+/sec | High-throughput, replay, event streaming |
| **RabbitMQ** | At-most-once / at-least-once | FIFO per queue | ~50k/sec | Complex routing, priorities |
| **Redis Streams** | At-least-once | Per-stream | 100k+/sec | Simple, if Redis already in stack |
| **Celery / Sidekiq / BullMQ** | App-level abstraction | — | Depends | Ergonomic in Python/Ruby/Node |
| **AWS Step Functions / Temporal** | Durable workflow | Per-execution | Moderate | Multi-step, retries, human tasks |

## Worker Design

**Consumers are idempotent.** Assume every message can be delivered twice. Include a `jobId` — if `JobStatus.state == DONE`, drop the duplicate.

**Concurrency**: fixed pool per worker (Java `ExecutorService`, Go worker goroutines). Auto-scale on **queue depth** (SQS `ApproximateNumberOfMessages`, Kafka lag).

**Timeouts**: every job has a visibility timeout. If the worker crashes mid-job, the message becomes visible again for another worker.

**Poison messages**: after N retries, send to a **Dead Letter Queue** (DLQ) and alert.

## Notifying the Client of Completion

| Method | Trade-off |
|--------|-----------|
| **Polling** `GET /jobs/{id}` | Simple, but chatty; exponential backoff |
| **Webhook** — client provides callback URL | Server → client push; requires client to be reachable |
| **WebSocket/SSE** | Best UX for logged-in users |
| **Push notification (mobile)** | Best for mobile background |
| **Email** | Best for very long jobs (report ready) |

## Reference Architecture — Video Transcoding

```
1. Client → API: POST /upload  (presigned URL flow, see pattern 6)
2. Client → S3: uploads raw MP4
3. S3 → EventBridge → SNS → SQS: "new object"
4. Worker fleet pulls SQS, runs FFmpeg for each resolution (240/480/720/1080/4K)
5. Worker writes outputs to S3, updates DynamoDB VideoStatus
6. Worker publishes SNS "video-ready" → user gets push notification
```

**Why fan-out to multiple queues?** Each resolution is an independent task; parallelism = # of resolutions × # of workers. Failure of the 4K job doesn't block 480p from being watchable.

## Handling Very Long Jobs (Hours to Days)

- **Checkpointing**: worker writes progress every N minutes. On restart, resume from checkpoint.
- **Heartbeats**: extend visibility timeout while alive; SQS lets you `ChangeMessageVisibility`.
- **Sub-tasks**: split a 10-hour job into 100 six-minute jobs. Better parallelism, easier retries.
- **Workflow engines** (Temporal, Step Functions, Cadence) — durable state, retries, timers, human approval steps.

## Interview Q&A

**Q: Why not just spin up a thread and do it inline?**
- The API server can crash mid-work → job lost.
- The client can disconnect → wasted work.
- No back-pressure — a spike floods the server.
- Can't scale the worker fleet independently of the API tier.

**Q: How do you guarantee exactly-once processing?**
- Almost impossible end-to-end. Aim for **at-least-once + idempotent workers**.
- Idempotency = write the result to a store keyed by `jobId` with a conditional put (`IF NOT EXISTS`).
- SQS FIFO gives exactly-once within a 5-minute dedup window.

**Q: How do you handle a poison message that keeps crashing workers?**
- Redrive policy: after N receive attempts, move to DLQ.
- Alert on DLQ depth > 0. Manual triage or an automated "quarantine" job.

**Q: How do you prioritize urgent jobs?**
- Separate queues per priority (high/mid/low). Workers weighted-round-robin.
- Or RabbitMQ priority queues (0-255).
- Beware of starvation — reserve some capacity for low priority.

**Q: How do you rate-limit expensive jobs per user?**
- Token bucket keyed by userId at API layer, or a per-user queue with concurrency cap.

**Q: Cost optimization for spiky workloads?**
- Spot instances for stateless workers.
- Scale to zero (Lambda, Cloud Run, K8s HPA with scale-from-zero).
- Batch small jobs together (transcoding pipeline groups 10 short clips per invocation).

## Real-World Examples
- **YouTube upload**: enqueue → transcode workers → notify user via bell icon (SSE).
- **Stripe payouts**: nightly job splits per merchant → Kafka → worker pool → webhook.
- **GitHub Actions**: job queue → runner pool → status via WebSocket to browser.
- **Figma exports**: enqueue → renderer → S3 → email download link.

## Gotchas
- **Queue depth ≠ latency**: 10k messages with 100 workers = 10s p99; without back-pressure it hides latency.
- **Worker fan-out amplifies failures**: a single bad deploy kills the whole pool. Use canary workers.
- **DLQ silence** — nobody watches. Add on-call alert on non-zero DLQ.
- **Retries with side effects**: sending 3 duplicate emails because retry logic wraps a non-idempotent SMTP call. Make idempotency the API contract.
