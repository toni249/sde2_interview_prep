# Low-Level Design (LLD) — SDE2 Master Guide

> **Goal:** Crack LLD rounds at FAANG / top-tier product companies.
> **Language:** Java | **Level:** SDE2 (Intermediate → Advanced)
> **Unique focus:** Deep concurrency analysis + "why this pattern and not that one" for every topic.

---

## Study Order (Follow This Sequence)

| # | File | Topic | Priority |
|---|------|--------|----------|
| 1 | [OOP Fundamentals](00_OOP_Fundamentals.md) | 4 Pillars, Java specifics, concurrency tie-in | MUST |
| 2 | [SOLID Principles](SOLID.md) | Clean code foundation | MUST |
| 3 | [UML & Relationships](01_UML_Relationships.md) | Association vs Aggregation vs Composition | MUST |
| 4 | [Concurrency in LLD](02_Concurrency_in_LLD.md) | JMM, volatile, synchronized, Atomic, locks | MUST |
| 5 | [Creational Patterns](Creational_Patterns.md) | Singleton (thread-safe), Factory, Builder, Prototype | HIGH |
| 6 | [Structural Patterns](Structural_Patterns.md) | Adapter, Decorator, Proxy, Facade, Composite, Bridge | HIGH |
| 7 | [Behavioral Patterns](Behavioral_Patterns.md) | Strategy, Observer, Command, State, CoR, Template Method | HIGH |
| 8 | [Parking Lot](LLD_01_Parking_Lot.md) | Striped locking, optimistic concurrency, State+Factory | HIGH |
| 9 | [Vending Machine](LLD_02_Vending_Machine.md) | State machine, synchronized transitions | HIGH |
| 10 | [BookMyShow](LLD_03_BookMyShow.md) | Seat locking, deadlock prevention, distributed scaling | HIGH |
| 11 | [Splitwise](LLD_04_Splitwise.md) | Graph debt simplification, Observer for notifications | MEDIUM |
| 12 | [ATM Design](LLD_05_ATM.md) | State + CoR, distributed transaction problem | MEDIUM |
| 13 | [Rate Limiter](LLD_06_Rate_Limiter.md) | 4 algorithms, Redis-based distributed limiting | HIGH |
| 14 | [LRU Cache](LLD_07_LRU_Cache.md) | Custom DLL + HashMap, TTL, thread-safe variants | HIGH |

---

## How to Approach an LLD Interview (5-Step Framework)

### Step 1 — Clarify Requirements (2-3 min)
Ask these before touching whiteboard:
- "What are the core use cases?"
- "Single machine or distributed?"
- "Concurrent access? Read-heavy or write-heavy?"
- "Any specific error handling / consistency requirements?"

### Step 2 — Identify Entities / Nouns (2 min)
Extract nouns from requirements. Each noun is potentially a class.
> "Design a Parking Lot" → `ParkingLot`, `Floor`, `Slot`, `Vehicle`, `Ticket`, `Payment`

### Step 3 — Define Relationships (3 min)
- Has-A (Composition/Aggregation) vs Is-A (Inheritance)
- Draw quick class diagram (even text-based is fine)

### Step 4 — Apply Patterns + Write Code (10-15 min)
- Identify which patterns fit and explain WHY
- Start coding core classes/interfaces
- Proactively mention thread-safety: "since this is shared state, I'll synchronize..."

### Step 5 — Discuss Trade-offs (3 min)
- What did you skip and why?
- How would you handle concurrency / scale?
- What would change for distributed deployment?

---

## Pattern → Problem Mapping

| Problem Signal | Pattern |
|---|---|
| Object changes behavior with state | **State** |
| Pluggable algorithms at runtime | **Strategy** |
| One-to-many notifications | **Observer** |
| Undo/Redo, queuing, scheduling | **Command** |
| Validation chains, middleware | **Chain of Responsibility** |
| Many optional constructor params | **Builder** |
| Only one instance needed | **Singleton** |
| Create objects without specifying class | **Factory / Abstract Factory** |
| Add behavior without changing class | **Decorator** |
| Simplify complex subsystem | **Facade** |
| Lazy loading, access control, caching | **Proxy** |
| Two incompatible interfaces | **Adapter** |
| Tree-like hierarchy | **Composite** |
| Two independent dimensions of change | **Bridge** |

---

## Concurrency Cheat Sheet for LLD

| Tool | Provides | Use When |
|---|---|---|
| `synchronized` | Atomicity + Visibility + Ordering | Default choice for shared mutable state |
| `volatile` | Visibility + Ordering (not atomicity) | Single writer, multiple readers; flags |
| `AtomicInteger/Long` | CAS-based atomic ops | Counters, sequence numbers (no lock needed) |
| `AtomicReference` | Atomic reference swap | Publishing immutable config objects |
| `ReentrantLock` | Like synchronized + tryLock, timeout | Need non-blocking acquire or multiple conditions |
| `ReadWriteLock` | Concurrent reads, exclusive writes | Read-heavy caches, configuration |
| `ConcurrentHashMap` | Thread-safe map | Shared maps; better throughput than Hashtable |
| `CopyOnWriteArrayList` | Thread-safe list | Observer lists; read-heavy, rare writes |
| `BlockingQueue` | Thread-safe producer-consumer | Command queues, work queues |
| Immutable objects | Zero synchronization needed | Best approach when possible |

### Striped Locking Pattern (Critical for SDE2)
```
Instead of ONE lock on the whole collection → per-element locks
→ Thread A locks Slot1 while Thread B locks Slot2 independently
→ Used in: Parking Lot slots, Seat booking, ConcurrentHashMap internals
```

---

## Most Frequently Asked LLD Questions

### Tier 1 (Almost Always)
- Design an LRU Cache
- Design a Rate Limiter
- Design a Parking Lot

### Tier 2 (Very Common)
- Design a Vending Machine
- Design BookMyShow / Seat Booking
- Design a Logger
- Design an ATM

### Tier 3 (Company Specific)
- Design Splitwise
- Design Cab Booking (Uber)
- Design Chess / Snake & Ladder
- Design Library Management System
- Design Elevator System

---

## Interview Anti-Patterns (Never Do These)

- Jump to code without clarifying requirements
- Use concrete classes instead of interfaces (no abstraction)
- Create God classes (one class doing everything)
- Forget thread safety on shared mutable state
- Use `synchronized` on `this` when you need per-object locking (too coarse)
- Not considering the "why not" alternative patterns
