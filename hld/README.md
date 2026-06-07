# HLD — High-Level Design

System design problems for SDE2 / FAANG interviews. **One folder per problem.** Each folder contains:

- `README.md` — the design doc (requirements → architecture → deep dives → interview drills)
- `diagrams/` — Excalidraw drawings embedded into the doc via `![[name]]` wikilinks

Each design doc follows the same structure:

1. **Clarify requirements** (functional + non-functional)
2. **Capacity estimation** (numbers matter — interviewers test this)
3. **API design** (the public contract)
4. **Architecture** (where each piece sits, why)
5. **Data model & storage**
6. **Deep dives** (the hard sub-problems)
7. **Failure modes & tradeoffs**
8. **Real-world examples** (how Stripe / GitHub / Cloudflare actually do it)
9. **Interview drilldown questions**

---

## Topics

| # | Topic | Folder | Status |
|---|---|---|---|
| 01 | Distributed Rate Limiter | [[dist-rate-limiter/README\|dist-rate-limiter]] | ✅ |
| 02 | URL Shortener | _coming soon_ | ⏳ |
| 03 | News Feed | _coming soon_ | ⏳ |
| 04 | Distributed Cache | _coming soon_ | ⏳ |
| 05 | Notification System | _coming soon_ | ⏳ |
| 06 | Chat / Messaging | _coming soon_ | ⏳ |
| 07 | Search Autocomplete | _coming soon_ | ⏳ |

---

## Diagrams

Each topic has a `diagrams/` subfolder with Excalidraw files. They are embedded via `![[basename]]` — Obsidian resolves wikilinks vault-wide by basename, so embeds work regardless of where the doc is.

**To edit a diagram:** open the `.excalidraw.md` file in Obsidian → Command Palette → "Excalidraw: Open as Excalidraw drawing".

---

## HLD vs LLD — When to apply which lens

| Aspect | HLD (this folder) | LLD ([[../java-lld/README\|java-lld]]) |
|---|---|---|
| Scope | System architecture, services, data flow | Classes, interfaces, design patterns |
| Diagrams | Block diagrams, deployment, sequence | Class diagrams, state machines |
| Storage | Choose DB, cache, queue | Choose data structures |
| Concurrency | Distributed consensus, replication | Locks, atomics, JMM |
| Scale | Millions of req/s, regions, sharding | Single-process throughput, GC, contention |
| Failures | Network partitions, region outages | NullPointerException, race conditions |

Most real interviews touch both — start HLD, the interviewer drills into LLD on one component.
