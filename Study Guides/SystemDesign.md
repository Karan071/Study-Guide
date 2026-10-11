# System Design for Interviews: HLD + LLD Reference

Audience: SDEs preparing for system design (HLD) and low-level/object-oriented design (LLD) interview rounds, from mid-level to senior.
Scope: An interview framework, estimation, about 25 HLD building blocks, 12 HLD case studies, OOP and SOLID, 18 design patterns, concurrency, and 9 LLD case studies. Each concept has code you can explain on a whiteboard.
Primary references: *System Design Interview* Vol. 1 & 2 (Alex Xu, with Sahn Lam for Vol. 2), [Hello Interview](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction), *Designing Data-Intensive Applications* (Martin Kleppmann), [ByteByteGo](https://bytebytego.com/), [System Design Primer](https://github.com/donnemartin/system-design-primer), [Refactoring.Guru](https://refactoring.guru/design-patterns), and the [AWS Builders' Library](https://aws.amazon.com/builders-library/).
Last reviewed: 2026-10-12.

Code is TypeScript unless noted. The concurrency section uses Java, because LLD interviews often ask about threads and TypeScript is single-threaded. All patterns carry over directly to Java, Python, Go, and C++.

Related guides in this repo: [Caching Techniques](./CachingTechniques.md), [APIs](./APIs.md), [Backend Optimization](./BackendOptimization.md), [Authentication](./AuthenticationConcepts.md), [Docker & Kubernetes](./DockerKubernetes.md).

## Table of Contents

**Part I: How to Interview**
0. [Study Roadmap](#0-study-roadmap)
1. [The HLD Interview Framework](#1-the-hld-interview-framework)
2. [Back-of-the-Envelope Estimation](#2-back-of-the-envelope-estimation)

**Part II: HLD Building Blocks**
3. [Scaling from Zero to Millions of Users](#3-scaling-from-zero-to-millions-of-users)
4. [Networking: DNS, Load Balancers, Proxies, API Gateway](#4-networking-dns-load-balancers-proxies-api-gateway)
5. [API Design and Real-Time Communication](#5-api-design-and-real-time-communication)
6. [Choosing a Database](#6-choosing-a-database)
7. [Indexing and Storage Engines](#7-indexing-and-storage-engines)
8. [Replication](#8-replication)
9. [Sharding / Partitioning](#9-sharding--partitioning)
10. [Consistent Hashing](#10-consistent-hashing)
11. [CAP, PACELC, and Consistency Models](#11-cap-pacelc-and-consistency-models)
12. [Caching in System Design](#12-caching-in-system-design)
13. [Message Queues, Streams, and Async Processing](#13-message-queues-streams-and-async-processing)
14. [Rate Limiting](#14-rate-limiting)
15. [Unique ID Generation](#15-unique-id-generation)
16. [Distributed Transactions, Sagas, and Idempotency](#16-distributed-transactions-sagas-and-idempotency)
17. [Contention: Locks and Concurrency Control](#17-contention-locks-and-concurrency-control)
18. [Blob Storage and CDNs](#18-blob-storage-and-cdns)
19. [Search and Inverted Indexes](#19-search-and-inverted-indexes)
20. [Geospatial Indexing](#20-geospatial-indexing)
21. [Probabilistic Data Structures](#21-probabilistic-data-structures)
22. [Reliability Patterns](#22-reliability-patterns)
23. [Consensus, Leader Election, and Coordination](#23-consensus-leader-election-and-coordination)
24. [Observability](#24-observability)
25. [Microservices vs Monolith](#25-microservices-vs-monolith)

**Part III: HLD Case Studies**
26. [URL Shortener (Bitly)](#26-url-shortener-bitly)
27. [Distributed Rate Limiter](#27-distributed-rate-limiter)
28. [News Feed (Facebook / Twitter)](#28-news-feed-facebook--twitter)
29. [Chat System (WhatsApp)](#29-chat-system-whatsapp)
30. [Video Streaming (YouTube)](#30-video-streaming-youtube)
31. [File Sync (Dropbox / Google Drive)](#31-file-sync-dropbox--google-drive)
32. [Ride Sharing (Uber)](#32-ride-sharing-uber)
33. [Ticket Booking (Ticketmaster)](#33-ticket-booking-ticketmaster)
34. [Web Crawler](#34-web-crawler)
35. [Top-K / Leaderboard / Ad Click Aggregator](#35-top-k--leaderboard--ad-click-aggregator)
36. [Notification System](#36-notification-system)
37. [Search Autocomplete (Typeahead)](#37-search-autocomplete-typeahead)
38. [Distributed Key-Value Store](#38-distributed-key-value-store)

**Part IV: LLD**
39. [The LLD Interview Framework](#39-the-lld-interview-framework)
40. [OOP Fundamentals and UML Relationships](#40-oop-fundamentals-and-uml-relationships)
41. [SOLID Principles](#41-solid-principles)
42. [Other Design Principles](#42-other-design-principles)
43. [Creational Patterns](#43-creational-patterns)
44. [Structural Patterns](#44-structural-patterns)
45. [Behavioral Patterns](#45-behavioral-patterns)
46. [Concurrency for LLD (Java)](#46-concurrency-for-lld-java)

**Part V: LLD Case Studies**
47. [Parking Lot](#47-parking-lot)
48. [LRU Cache](#48-lru-cache)
49. [Rate Limiter (In-Process)](#49-rate-limiter-in-process)
50. [Elevator System](#50-elevator-system)
51. [Vending Machine](#51-vending-machine)
52. [Splitwise](#52-splitwise)
53. [Movie Ticket Booking (BookMyShow)](#53-movie-ticket-booking-bookmyshow)
54. [Logger / Pub-Sub](#54-logger--pub-sub)
55. [Tic-Tac-Toe / Board Games](#55-tic-tac-toe--board-games)

**Part VI: Final Prep**
56. [Trade-Off Cheat Sheet](#56-trade-off-cheat-sheet)
57. [Common Mistakes and Interviewer Signals](#57-common-mistakes-and-interviewer-signals)
58. [Master Resource List](#58-master-resource-list)

---

# Part I: How to Interview

## 0. Study Roadmap

### What each round tests

| Round | Typical length | What they want to see | Main output |
| --- | --- | --- | --- |
| HLD (system design) | 45–60 min | Requirements gathering, sensible architecture, scaling, trade-offs, depth in 2–3 areas | Box-and-arrow diagram, APIs, data model, deep dives |
| LLD (OOD / machine coding) | 45–90 min | Clean object model, SOLID, patterns used correctly, extensibility, sometimes runnable code | Class diagram + working code |

### Level expectations (Hello Interview's framing)

| Level | HLD expectation |
| --- | --- |
| Mid-level (L4/SDE2) | Working end-to-end design that meets functional requirements. Some depth when the interviewer probes. The interviewer may steer. |
| Senior (L5/SDE3) | You drive the interview. You find bottlenecks before you're asked, go deep in 2–3 areas, and explain trade-offs with numbers. |
| Staff+ (L6+) | Breadth plus real depth, with practical experience. You call out what *doesn't* matter, operational concerns, and how the system evolves. |

### 6-week plan

| Week | HLD | LLD |
| --- | --- | --- |
| 1 | Sections 1–5: framework, estimation, scaling basics, networking, APIs | 39–42: OOP, SOLID, principles |
| 2 | 6–12: databases, indexing, replication, sharding, consistent hashing, CAP, caching | 43–44: creational + structural patterns |
| 3 | 13–17: queues, rate limiting, IDs, transactions, contention | 45–46: behavioral patterns + concurrency |
| 4 | 18–25: blobs, search, geo, probabilistic, reliability, consensus | 47–50: Parking Lot, LRU, Rate Limiter, Elevator |
| 5 | 26–32: case studies (do each one yourself in 45 min *before* reading) | 51–55: remaining LLD case studies |
| 6 | 33–38 + mock interviews (at least 4) | Timed machine-coding practice (90 min each) |

**Practice method:** For each case study, set a 45-minute timer and design on paper or Excalidraw. Then compare with the reference solution. Write down every gap you find. Explaining your design out loud matters as much as the design itself.

---

## 1. The HLD Interview Framework

Two widely used frameworks say the same thing at different levels of detail. Alex Xu uses 4 steps. Hello Interview's "Delivery Framework" uses 6. Use the Hello Interview version because it keeps you on schedule.

### Alex Xu's 4 steps

1. **Understand the problem and establish design scope** (3–10 min)
2. **Propose high-level design and get buy-in** (10–15 min)
3. **Design deep dive** (10–25 min)
4. **Wrap up** (3–5 min): bottlenecks, failure modes, monitoring, next scale step

### Hello Interview delivery framework (45-minute budget)

```text
┌──────────────────────────────────────────────────────────────────────┐
│ 1. Requirements (~5 min)                                             │
│    • Functional: "Users should be able to..." (top 3, no more)       │
│    • Non-functional: scale, latency, availability vs consistency,    │
│      durability, security. Quantify them ("p99 < 200 ms").           │
│    • Capacity estimation: ONLY if it changes a design decision       │
├──────────────────────────────────────────────────────────────────────┤
│ 2. Core Entities (~2 min)   User, Tweet, Follow, ...                 │
├──────────────────────────────────────────────────────────────────────┤
│ 3. API / Interface (~5 min) REST endpoints mapped to each FR         │
├──────────────────────────────────────────────────────────────────────┤
│ 4. [Data Flow] (~5 min)     only for pipeline-style systems          │
├──────────────────────────────────────────────────────────────────────┤
│ 5. High-Level Design (~10-15 min)                                    │
│    Satisfy FRs one by one with simple boxes. Don't optimize yet.     │
├──────────────────────────────────────────────────────────────────────┤
│ 6. Deep Dives (~10 min)                                              │
│    Go back to the NFRs: scale, latency, consistency, failure.        │
│    Find bottlenecks and fix them with explicit trade-offs.           │
└──────────────────────────────────────────────────────────────────────┘
```

### Requirement questions to ask

| Category | Questions |
| --- | --- |
| Users & scale | How many DAU? Peak vs average? Read:write ratio? Global or one region? |
| Features | Which 3 features are most important? Any out of scope (auth, payments, analytics)? |
| Data | How large is each object? How long is data retained? Is there media? |
| Latency | What's acceptable p99 for the hot path? Is it real-time (<1 s) or eventual (minutes)? |
| Consistency | Is stale data OK? Where is strong consistency a hard requirement (money, inventory, seats)? |
| Availability | 99.9% or 99.99%? Which parts can degrade gracefully? |

### Non-functional requirements: how to phrase them

Bad: "The system should be scalable and highly available."
Good: "Support 100M DAU and 10k writes/s at peak. Feed reads p99 < 200 ms. Availability is preferred over consistency for feeds: a post can take up to 1 minute to appear. Posts must never be lost once acknowledged."

### Talking checklist for each component you draw

- **Why** is it there? Which requirement does it serve?
- **What** technology, and why that one (Postgres vs DynamoDB vs Cassandra)?
- **How** does it scale (stateless + horizontal, sharded, replicated)?
- **What happens when it fails?**

---

## 2. Back-of-the-Envelope Estimation

Estimates help you make decisions, such as "does this fit on one machine?" or "do we need sharding?". You don't need precision. Round aggressively.

### Powers of two

| Power | Approx value | Name |
| --- | --- | --- |
| 2^10 | 1 thousand | 1 KB |
| 2^20 | 1 million | 1 MB |
| 2^30 | 1 billion | 1 GB |
| 2^40 | 1 trillion | 1 TB |
| 2^50 | 1 quadrillion | 1 PB |

### Latency numbers every programmer should know (Jeff Dean, updated)

| Operation | Latency | Intuition |
| --- | --- | --- |
| L1 cache reference | 0.5 ns | |
| Branch mispredict | 5 ns | |
| L2 cache reference | 7 ns | |
| Mutex lock/unlock | 25 ns | |
| Main memory reference | 100 ns | |
| Compress 1 KB (Snappy/Zippy) | 3 µs | |
| Read 1 MB sequentially from memory | 10 µs | RAM is fast |
| Send 1 KB over 1 Gbps network | 10 µs | |
| Random read from SSD | 16–100 µs | |
| Read 1 MB sequentially from SSD | ~50 µs–1 ms | |
| Round trip within same datacenter | 0.5 ms | |
| Redis GET over network | ~0.5–1 ms | |
| Disk seek (HDD) | 10 ms | Avoid random HDD I/O |
| Read 1 MB sequentially from HDD | 20 ms | |
| Packet CA → Netherlands → CA | 150 ms | Cross-continent hurts |

**Conclusions:** memory is fast and disk is slow. Avoid disk seeks. Compress before sending over the network. Cross-region round trips cost about 100 ms or more.

### Modern hardware numbers (Hello Interview's "numbers to know")

Modern hardware is stronger than many candidates assume. Don't shard too early.

| Component | Rough capacity |
| --- | --- |
| Single app server | 1k–10k+ RPS for simple requests (CPU-bound work is lower) |
| Postgres/MySQL (one well-tuned node) | ~10k–50k simple reads/s, ~1k–10k+ writes/s, multiple TB of storage |
| Redis (one node) | ~100k+ ops/s, sub-ms latency, up to ~hundreds of GB RAM |
| Kafka (one broker) | ~100s of MB/s, ~1M msgs/s per cluster easily |
| Server RAM | Commodity boxes with 256 GB–1 TB+ |
| Network | 10–25 Gbps NICs are common in datacenters |

### Availability "nines"

| Availability | Downtime/year | Downtime/day |
| --- | --- | --- |
| 99% | 3.65 days | 14.4 min |
| 99.9% | 8.77 hours | 1.44 min |
| 99.99% | 52.6 min | 8.6 s |
| 99.999% | 5.26 min | 0.86 s |

Components in **series** multiply their availability: 99.9% × 99.9% ≈ 99.8%. Redundant components in **parallel** improve it: 1 − (0.001)² = 99.9999%.

### Estimation template

```text
DAU                      = 100M
Actions per user per day = 10 reads, 1 write
Seconds per day          ≈ 86,400 ≈ 10^5          (round!)

Write QPS   = 100M × 1 / 10^5      = 1,000 /s
Read QPS    = 100M × 10 / 10^5     = 10,000 /s
Peak QPS    = 2–3× average         ≈ 30,000 reads/s

Storage/day = 100M writes × 1 KB   = 100 GB/day
Storage/5y  = 100 GB × 365 × 5     ≈ 180 TB         → needs sharding / object store
Bandwidth   = 30k reads/s × 1 KB   = 30 MB/s egress
Cache size  = 20% of daily reads (80/20 rule) × 1 KB
```

### Estimation helper (useful for practice)

```ts
const SECONDS_PER_DAY = 86_400;

function estimate(opts: {
  dau: number;
  readsPerUser: number;
  writesPerUser: number;
  bytesPerWrite: number;
  peakFactor?: number;
  retentionYears?: number;
}) {
  const peak = opts.peakFactor ?? 3;
  const writeQps = (opts.dau * opts.writesPerUser) / SECONDS_PER_DAY;
  const readQps = (opts.dau * opts.readsPerUser) / SECONDS_PER_DAY;
  const bytesPerDay = opts.dau * opts.writesPerUser * opts.bytesPerWrite;
  const totalBytes = bytesPerDay * 365 * (opts.retentionYears ?? 5);
  return {
    writeQps: Math.round(writeQps),
    readQps: Math.round(readQps),
    peakReadQps: Math.round(readQps * peak),
    storagePerDayGB: bytesPerDay / 1e9,
    totalStorageTB: totalBytes / 1e12,
  };
}

// Twitter-ish: 300M DAU, 100 timeline reads, 2 tweets of ~300 B (text+metadata)
console.log(estimate({ dau: 300e6, readsPerUser: 100, writesPerUser: 2, bytesPerWrite: 300 }));
// → writeQps ≈ 6,944, readQps ≈ 347,222, peak ≈ 1M, ~180 GB/day, ~330 TB over 5y
```

**Interview tip:** Say what decision each number drives. For example: "347k reads/s means reading the feed on the fly with fan-in from the DB is too slow, so we precompute feeds in a cache."

---
# Part II: HLD Building Blocks

## 3. Scaling from Zero to Millions of Users

This follows Alex Xu Vol. 1, Chapter 1. Every later design builds on this progression.

```text
Stage 1: Single server         Stage 2: Separate DB          Stage 3: LB + N stateless servers
┌────────────┐                ┌──────┐   ┌────┐            ┌────┐   ┌──────┬──────┬──────┐
│ web+app+DB │                │ app  │──▶│ DB │            │ LB │──▶│ app1 │ app2 │ app3 │
└────────────┘                └──────┘   └────┘            └────┘   └──┬───┴──┬───┴──┬───┘
                                                                       ▼      ▼      ▼
                                                                    primary DB ─▶ replicas

Stage 4: + Cache + CDN          Stage 5: + Message queue       Stage 6: + Sharding, multi-DC
 client ─▶ CDN (static)          app ─▶ queue ─▶ workers       GeoDNS ─▶ DC-us / DC-eu
 app ─▶ Redis ─▶ DB              (async, decoupled)            DB shards by user_id
```

| Step | Technique | Problem solved | New problem introduced |
| --- | --- | --- | --- |
| 1 | Vertical scaling (bigger box) | Simple capacity growth | Hard ceiling, single point of failure (SPOF) |
| 2 | Separate web tier and data tier | Scale each independently | Network hop |
| 3 | Load balancer + horizontal web tier | Web SPOF, capacity | Servers must be **stateless** |
| 4 | Move session state to Redis or a DB (stateless servers) | Any server can serve any request | Shared store becomes critical |
| 5 | DB replication (primary + read replicas) | Read scaling, DB failover | Replication lag (stale reads) |
| 6 | Cache (Redis/Memcached) | DB read load, latency | Invalidation, consistency |
| 7 | CDN | Static/media latency globally | Cache invalidation, cost |
| 8 | Message queue + workers | Slow or spiky work off the request path | Eventual consistency, retries |
| 9 | Logging, metrics, automation | Operating at scale | |
| 10 | DB sharding | Write scaling, data size | Cross-shard queries, hotspots, resharding |
| 11 | Multi-datacenter + GeoDNS | Regional latency, DC failure | Data sync across regions |

**Key idea: stateless services.** Keep no user-specific state in app-server memory. Store session data and uploaded files in external stores (Redis, DB, S3). Then any server can be added, removed, or replaced, and autoscaling just works.

---

## 4. Networking: DNS, Load Balancers, Proxies, API Gateway

### Request journey

```text
Browser ──DNS lookup──▶ DNS (GeoDNS / latency-based) ──▶ returns LB/CDN IP
   │
   └─HTTPS──▶ CDN edge ──miss──▶ L7 Load Balancer / API Gateway
                                   │  TLS termination, auth, rate limit, routing
                                   ▼
                          Service instances (stateless) ──▶ cache / DB / queue
```

### OSI layers you'll be asked about

| Layer | Protocols | Interview relevance |
| --- | --- | --- |
| L3 Network | IP | Routing, anycast (CDNs, DNS) |
| L4 Transport | TCP, UDP, QUIC | TCP = reliable and ordered. UDP = fast and lossy (video calls, gaming). |
| L7 Application | HTTP/1.1, HTTP/2, HTTP/3, gRPC, WebSocket | Routing by path or header, caching, auth |

### Load balancers

| | L4 LB | L7 LB |
| --- | --- | --- |
| Sees | IP + port | Full HTTP request (URL, headers, cookies) |
| Speed | Faster, less CPU | Slower, but smarter |
| Can do | Pass-through TCP, good for long-lived WebSocket connections | Path routing (`/api` → svc A), TLS termination, sticky sessions, retries |
| Examples | AWS NLB, HAProxy (TCP mode) | AWS ALB, Nginx, Envoy |

**Algorithms:**

```ts
interface Server { id: string; weight: number; activeConnections: number; healthy: boolean }

class RoundRobin {
  private i = 0;
  constructor(private servers: Server[]) {}
  next(): Server {
    const healthy = this.servers.filter((s) => s.healthy);
    if (healthy.length === 0) throw new Error("no healthy upstream");
    return healthy[this.i++ % healthy.length];
  }
}

class LeastConnections {
  constructor(private servers: Server[]) {}
  next(): Server {
    return this.servers
      .filter((s) => s.healthy)
      .reduce((a, b) => (a.activeConnections <= b.activeConnections ? a : b));
  }
}

// Smooth weighted round robin (used by Nginx): spreads picks evenly instead of bursts
class WeightedRoundRobin {
  private current = new Map<string, number>();
  constructor(private servers: Server[]) {}
  next(): Server {
    const healthy = this.servers.filter((s) => s.healthy);
    const total = healthy.reduce((sum, s) => sum + s.weight, 0);
    let best: Server | undefined;
    for (const s of healthy) {
      const cw = (this.current.get(s.id) ?? 0) + s.weight;
      this.current.set(s.id, cw);
      if (!best || cw > this.current.get(best.id)!) best = s;
    }
    this.current.set(best!.id, this.current.get(best!.id)! - total);
    return best!;
  }
}
```

| Algorithm | Use when |
| --- | --- |
| Round robin | Homogeneous servers, similar request cost |
| Weighted RR | Mixed instance sizes |
| Least connections | Long-lived or uneven requests (WebSockets, uploads) |
| IP hash / consistent hash | Need affinity (cache locality, sticky WebSocket routing) |
| Power of two choices | Pick 2 random servers and use the less loaded one. Near-optimal and cheap. |

**Health checks:** Active checks probe `/healthz` every N seconds. Passive checks eject a server after M failed real requests. LBs themselves are made redundant with an active-passive pair plus a floating IP, or a managed service.

### Forward proxy vs reverse proxy vs API gateway

- **Forward proxy:** Sits in front of *clients* (corporate proxy, VPN).
- **Reverse proxy:** Sits in front of *servers*. Handles TLS, compression, caching, and load balancing (Nginx, Envoy).
- **API gateway:** A reverse proxy plus API concerns: authentication, rate limiting, request validation, routing to microservices, request aggregation, and API keys/quotas (Kong, AWS API Gateway, Apigee). In interviews, draw one box and say "auth + rate limiting + routing."

### DNS

- Resolution: browser cache → OS cache → recursive resolver → root → TLD (`.com`) → authoritative server.
- TTL controls how fast changes propagate.
- **GeoDNS / latency-based routing** sends users to the nearest region. **Anycast** advertises one IP from many locations, and BGP routes each user to the nearest one (Cloudflare, Google DNS).

---

## 5. API Design and Real-Time Communication

See [APIs.md](./APIs.md) for full detail. Here is the interview summary.

### REST vs gRPC vs GraphQL

| | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| Format | JSON over HTTP | Protobuf over HTTP/2 | JSON, single endpoint |
| Best for | Public APIs, default choice | Internal service-to-service, low latency, streaming | Many client types needing different data shapes |
| Pros | Simple, cacheable, universal | Fast, typed contracts, bidirectional streaming | No over- or under-fetching |
| Cons | Over/under-fetch, chatty | Not browser-native, harder to debug | Caching is hard, N+1 resolvers, complexity |

**Interview default:** REST for client APIs and gRPC between internal services, unless there's a reason otherwise.

### REST API conventions

```http
POST   /v1/tweets                      → create          (body: { text, mediaIds })
GET    /v1/tweets/{id}                 → read
GET    /v1/users/{id}/feed?cursor=abc&limit=20   → cursor pagination
PATCH  /v1/tweets/{id}                 → partial update
DELETE /v1/tweets/{id}
POST   /v1/payments  Idempotency-Key: 7f3a...   → safe retries
```

- Get the user from the **auth token**, not the request body. `POST /tweets` should never accept a `userId` field, because a client could spoof it.
- **Pagination:** Offset (`?page=5`) is simple, but it's slow on deep pages and shifts when data changes. **Cursor/keyset** (`WHERE created_at < :cursor ORDER BY created_at DESC LIMIT 20`) is stable and fast. Use cursors for feeds.

```ts
// Keyset pagination: encode the last-seen sort key as an opaque cursor
type Cursor = { createdAt: string; id: string };
const encode = (c: Cursor) => Buffer.from(JSON.stringify(c)).toString("base64url");
const decode = (s: string): Cursor => JSON.parse(Buffer.from(s, "base64url").toString());

async function getFeed(db: Db, userId: string, cursor?: string, limit = 20) {
  const c = cursor ? decode(cursor) : null;
  const rows = await db.query(
    `SELECT id, author_id, text, created_at FROM posts
     WHERE feed_owner = $1
       AND ($2::timestamptz IS NULL OR (created_at, id) < ($2, $3))
     ORDER BY created_at DESC, id DESC
     LIMIT $4`,
    [userId, c?.createdAt ?? null, c?.id ?? null, limit + 1],
  );
  const hasMore = rows.length > limit;
  const page = rows.slice(0, limit);
  const last = page.at(-1);
  return { items: page, nextCursor: hasMore && last ? encode({ createdAt: last.created_at, id: last.id }) : null };
}
```

### Real-time: polling vs long polling vs SSE vs WebSockets

| Technique | How | Direction | Use for |
| --- | --- | --- | --- |
| Short polling | Client asks every N s | Client → server | Simple, low-frequency updates |
| Long polling | Server holds the request until data arrives or timeout | Pseudo push | Fallback, moderate frequency |
| SSE (Server-Sent Events) | One long-lived HTTP response stream | Server → client only | Live comments, notifications, LLM token streaming, scores |
| WebSocket | Upgraded TCP connection, full-duplex | Both | Chat, multiplayer, collaborative editing |
| WebRTC | Peer-to-peer (with STUN/TURN) | P2P | Video/voice calls |

```ts
// SSE endpoint (Express)
app.get("/events/:liveId", (req, res) => {
  res.set({ "Content-Type": "text/event-stream", "Cache-Control": "no-cache", Connection: "keep-alive" });
  res.flushHeaders();
  const unsubscribe = pubsub.subscribe(`live:${req.params.liveId}`, (msg) => {
    res.write(`id: ${msg.id}\ndata: ${JSON.stringify(msg)}\n\n`);
  });
  req.on("close", unsubscribe);
});
```

**Scaling stateful connections** (a common deep dive): WebSocket and SSE servers hold connections, so a message for user B must reach *the server that holds B's connection*. Options:

1. **Pub/sub fan-out** (Redis Pub/Sub, NATS): every connection server subscribes to the channels its clients care about. This is simple and scales well up to a point.
2. **Connection registry**: Redis stores `userId → serverId`, and the sender routes directly to that server.
3. **Consistent hashing** of users to servers, so you know where a user is without a lookup.

Use an L4 LB, or an L7 LB with WebSocket support. Track connection counts per server, and drain connections gracefully on deploy.

---

## 6. Choosing a Database

### Decision table

| Need | Pick | Examples |
| --- | --- | --- |
| Relational data, transactions, joins, constraints (the default) | Relational SQL | PostgreSQL, MySQL, Aurora |
| Massive write throughput, wide rows, time-ordered, multi-DC | Wide-column (LSM) | Cassandra, ScyllaDB, HBase, Bigtable |
| Simple key lookups at any scale, serverless | Key-value / document | DynamoDB, Redis, Riak |
| Flexible JSON documents | Document | MongoDB, Couchbase, Firestore |
| Relationship traversal (friends-of-friends) | Graph | Neo4j, Neptune (you usually don't need one) |
| Full-text search, fuzzy matching, facets | Search engine | Elasticsearch, OpenSearch |
| Metrics, time-bucketed data | Time-series | InfluxDB, TimescaleDB, Prometheus |
| Analytics over billions of rows | Columnar OLAP | ClickHouse, BigQuery, Snowflake, Druid |
| Files, images, video | Object store | S3, GCS, Azure Blob |
| Strong consistency + global scale | NewSQL | Spanner, CockroachDB, YugabyteDB |

**Hello Interview's advice:** Defaulting to PostgreSQL is fine in most interviews. Switch only for a stated reason (write throughput, access pattern, scale). What matters is giving a reason, not naming a trendy database.

### ACID vs BASE

- **ACID:** Atomicity (all or nothing), Consistency (invariants hold), Isolation (concurrent transactions don't interfere), Durability (committed data survives a crash).
- **BASE:** Basically Available, Soft state, Eventual consistency. Typical of NoSQL at scale.

### Isolation levels (frequently asked)

| Level | Prevents | Still allows |
| --- | --- | --- |
| Read uncommitted | — | Dirty reads |
| Read committed (Postgres default) | Dirty reads | Non-repeatable reads, lost updates |
| Repeatable read / snapshot | + Non-repeatable reads | Write skew (phantoms in some DBs) |
| Serializable | Everything | Lower throughput, aborts and retries |

### Data modeling: normalize or denormalize?

- **Normalize** (3NF) when writes are frequent and correctness matters. Use joins.
- **Denormalize** when reads dominate and joins are expensive or impossible (NoSQL, sharded data). Duplicate data such as `author_name` on a post, and accept the update fan-out.
- **NoSQL rule (DynamoDB/Cassandra):** model around **access patterns**. Choose the partition key and sort key from your queries.

```sql
-- Cassandra: "get messages in a chat, newest first"
CREATE TABLE messages (
  chat_id     uuid,
  message_id  timeuuid,          -- time-ordered, unique
  sender_id   uuid,
  body        text,
  PRIMARY KEY ((chat_id), message_id)   -- partition key = chat_id, clustering = message_id
) WITH CLUSTERING ORDER BY (message_id DESC);
```

---

## 7. Indexing and Storage Engines

### Index types

| Index | Structure | Good for |
| --- | --- | --- |
| B-tree / B+tree | Balanced tree, sorted | Equality + range queries. Default in SQL. |
| Hash | Hash table | Exact lookups only |
| LSM tree | Memtable + sorted SSTables | Write-heavy (Cassandra, RocksDB) |
| Inverted index | term → doc list | Full-text search |
| Geospatial (R-tree, geohash, quadtree) | Spatial | "Nearby" queries |
| Composite | Multi-column B-tree | `WHERE a = ? AND b > ?` (left-prefix rule) |
| Covering | Index includes all needed columns | Index-only scans |

**Composite index left-prefix rule:** An index on `(user_id, created_at)` serves `WHERE user_id = ?` and `WHERE user_id = ? AND created_at > ?`. It does **not** efficiently serve `WHERE created_at > ?` alone.

### B-tree vs LSM tree

```text
B-tree (update in place)                 LSM tree (append-only)
  writes → find page → modify page        writes → WAL + in-memory memtable
  reads  → O(log n) page lookups          memtable full → flush as sorted SSTable
  + fast reads, predictable                background compaction merges SSTables
  − random writes, write amplification    reads → memtable → bloom filter → SSTables
                                           + very fast writes  − read amp, compaction cost
```

### Minimal LSM sketch (to explain the idea)

```ts
class TinyLSM {
  private memtable = new Map<string, string>();
  private sstables: Map<string, string>[] = []; // newest first; each is "sorted & immutable"
  constructor(private memtableLimit = 1000) {}

  put(key: string, value: string) {
    // 1. append to WAL for durability (omitted)
    this.memtable.set(key, value);
    if (this.memtable.size >= this.memtableLimit) this.flush();
  }

  get(key: string): string | undefined {
    if (this.memtable.has(key)) return this.memtable.get(key);
    for (const table of this.sstables) {     // real systems check a bloom filter first
      if (table.has(key)) return table.get(key);
    }
    return undefined;
  }

  private flush() {
    const sorted = new Map([...this.memtable.entries()].sort(([a], [b]) => a.localeCompare(b)));
    this.sstables.unshift(sorted);
    this.memtable = new Map();
    if (this.sstables.length > 4) this.compact();
  }

  private compact() {
    const merged = new Map<string, string>();
    for (const t of [...this.sstables].reverse()) for (const [k, v] of t) merged.set(k, v); // newer wins
    this.sstables = [merged];
  }
}
```

---

## 8. Replication

Replication copies data to multiple nodes for **availability**, **read scaling**, and **latency** (geo-local reads).

| Topology | How | Pros | Cons |
| --- | --- | --- | --- |
| Single-leader (primary-replica) | All writes go to the leader. Replicas apply its log. | Simple, no write conflicts | Leader is the write bottleneck. Failover is tricky. |
| Multi-leader | Several leaders (e.g. one per DC) | Local writes in each region | **Write conflicts** (need last-write-wins, CRDTs, or app-level merge) |
| Leaderless (Dynamo-style) | Client writes to W of N replicas and reads from R | High availability, no failover | Quorum tuning, read repair, sloppy quorums |

### Sync vs async

- **Synchronous:** The leader waits for replica ACKs. Durable, but slow, and a slow replica blocks writes.
- **Asynchronous:** Fast, but the replica lags. Reading your own write from a replica can miss it, and leader failure can lose data.
- **Semi-sync:** Wait for 1 replica and replicate to the rest asynchronously (MySQL semi-sync, Postgres `synchronous_standby_names`).

### Replication-lag anomalies and fixes

| Problem | Fix |
| --- | --- |
| **Read-your-writes:** user posts, refreshes, and the post is gone | Read the user's own data from the leader for N seconds after a write, or track the write's log position (LSN) |
| **Monotonic reads:** data appears then disappears | Pin each user to one replica (hash user → replica) |
| **Consistent prefix:** answer seen before the question | Write causally related data to the same partition |

### Quorum (leaderless)

With N replicas, write quorum W, and read quorum R: **if W + R > N, reads overlap with the latest write**.

```text
N=3, W=2, R=2  → balanced, tolerates 1 node down for reads and writes
N=3, W=3, R=1  → fast reads, writes fail if any node is down
N=3, W=1, R=3  → fast writes, slow reads
```

### Failover pitfalls

Split brain (two leaders), lost async writes, and choosing a failover timeout that's too short (flapping) or too long (downtime). Mitigate with consensus-based leader election (Section 23) and fencing tokens.

---

## 9. Sharding / Partitioning

Sharding splits data across machines so each holds a subset. It scales **writes and storage**, which replication alone does not.

### Strategies

| Strategy | How | Pros | Cons |
| --- | --- | --- | --- |
| Range | `user_id 1–1M → shard1` | Range scans work | Hotspots (new IDs, time ranges) |
| Hash | `hash(key) % N` | Even spread | Range queries scatter. Changing N reshuffles almost everything. |
| Consistent hash | Hash ring (Section 10) | Adding a node moves ~1/N of keys | More complex |
| Directory / lookup | Lookup service maps key → shard | Flexible, supports moving tenants | Lookup is a dependency and possible SPOF |
| Geo | By region | Locality, data residency | Uneven load across regions |

### Choosing a shard key (the most important decision)

1. **High cardinality:** many distinct values.
2. **Even distribution:** no celebrity hotspots.
3. **Matches the main access pattern:** most queries hit **one** shard.

| System | Good shard key | Why |
| --- | --- | --- |
| Chat | `chat_id` | All messages for a conversation on one shard |
| Social feed | `user_id` | User's own data colocated |
| URL shortener | `short_code` | Point lookups |
| Orders | `customer_id` (or `order_id` if no per-customer queries) | |
| Bad | `country`, `created_date`, `status` | Low cardinality or time hotspot |

### Hotspots (celebrity problem)

- **Key salting:** Write `hot_key#0..9` across 10 shards and fan reads in from all 10 (counters, likes).
- **Dedicated shard or cache** for known-hot entities.
- **Split hot partitions** dynamically (DynamoDB adaptive capacity).

```ts
// Sharded counter: spread writes for a hot key across K sub-keys
const K = 16;
async function incrementLikes(redis: Redis, postId: string) {
  const shard = Math.floor(Math.random() * K);
  await redis.incr(`likes:${postId}:${shard}`);
}
async function getLikes(redis: Redis, postId: string): Promise<number> {
  const keys = Array.from({ length: K }, (_, i) => `likes:${postId}:${i}`);
  const vals = await redis.mget(...keys);
  return vals.reduce((sum, v) => sum + Number(v ?? 0), 0);
}
```

### Problems sharding introduces

- **Cross-shard joins:** Denormalize, or do application-level joins.
- **Cross-shard transactions:** Use sagas, or avoid them by choosing the shard key well (Section 16).
- **Global secondary indexes:** Either a local index on each shard (scatter-gather reads) or a global index partitioned by the indexed field (writes go to 2 places).
- **Resharding:** Use consistent hashing, or many logical shards mapped to fewer physical nodes (e.g. 1024 logical → 16 physical).

---

## 10. Consistent Hashing

**Problem:** With `hash(key) % N`, changing N from 4 to 5 remaps about 80% of keys, which invalidates caches and forces mass data movement.

**Solution:** Place servers and keys on a hash ring. Each key belongs to the first server clockwise from it. Adding or removing a node moves only about K/N keys. **Virtual nodes** (each server appears 100–200 times on the ring) even out the load and spread a dead node's keys across many survivors.

```text
                 0 / 2^32
                ●  S1-v0
          k1 ○       ● S3-v1      key → walk clockwise → first server node
        ● S2-v0         ○ k2      each key is owned by the next vnode clockwise
          ○ k3       ● S1-v1
                ● S3-v0
```

```ts
import { createHash } from "node:crypto";

class ConsistentHashRing {
  private ring: { hash: number; node: string }[] = []; // sorted by hash
  constructor(private virtualNodes = 150) {}

  private hash(key: string): number {
    // first 4 bytes of md5 → 32-bit int (use murmur/xxhash in production)
    return createHash("md5").update(key).digest().readUInt32BE(0);
  }

  addNode(node: string) {
    for (let i = 0; i < this.virtualNodes; i++) {
      this.ring.push({ hash: this.hash(`${node}#${i}`), node });
    }
    this.ring.sort((a, b) => a.hash - b.hash);
  }

  removeNode(node: string) {
    this.ring = this.ring.filter((v) => v.node !== node);
  }

  getNode(key: string): string {
    if (this.ring.length === 0) throw new Error("empty ring");
    const h = this.hash(key);
    // binary search for first vnode with hash >= h
    let lo = 0, hi = this.ring.length;
    while (lo < hi) {
      const mid = (lo + hi) >>> 1;
      if (this.ring[mid].hash < h) lo = mid + 1; else hi = mid;
    }
    return this.ring[lo % this.ring.length].node; // wrap around
  }

  // For replication: next N *distinct* physical nodes clockwise
  getNodes(key: string, n: number): string[] {
    const h = this.hash(key);
    let idx = this.ring.findIndex((v) => v.hash >= h);
    if (idx === -1) idx = 0;
    const result = new Set<string>();
    for (let i = 0; result.size < n && i < this.ring.length; i++) {
      result.add(this.ring[(idx + i) % this.ring.length].node);
    }
    return [...result];
  }
}

const ring = new ConsistentHashRing();
["cache-a", "cache-b", "cache-c"].forEach((n) => ring.addNode(n));
console.log(ring.getNode("user:42"), ring.getNodes("user:42", 2));
```

**Used in:** DynamoDB, Cassandra, Riak, Memcached clients (ketama), Discord, CDNs, and load balancers (Maglev, Envoy ring hash).

---

## 11. CAP, PACELC, and Consistency Models

### CAP theorem

During a network **P**artition, a distributed system must choose between **C**onsistency (every read sees the latest write, or gets an error) and **A**vailability (every request gets a non-error response). Partitions will happen, so in practice the choice is CP or AP *for each feature*.

| Choose consistency (CP) | Choose availability (AP) |
| --- | --- |
| Booking the last seat or ticket | Social feed, likes count |
| Inventory and payments | Product catalog, reviews |
| Unique username registration | Viewing a profile |
| Bank balance transfers | Search results, recommendations |

**Interview phrasing:** "For seat booking, we need strong consistency, so we'd rather fail a request than double-book. For browsing events, availability matters more and staleness of a few seconds is fine." Mixing CP and AP in one design is normal and expected.

### PACELC

If there's a **P**artition, choose **A** or **C**. **E**lse (normal operation), choose **L**atency or **C**onsistency. Dynamo/Cassandra are PA/EL. Spanner is PC/EC. This captures the everyday trade-off, which CAP ignores.

### Consistency spectrum (strongest → weakest)

| Model | Guarantee | Example |
| --- | --- | --- |
| Linearizable (strong) | Behaves like one copy. Reads see the latest committed write. | Spanner, etcd, ZooKeeper, single-leader reads |
| Sequential | All nodes see the same order of operations | |
| Causal | Causally related ops seen in order (reply after post) | |
| Read-your-writes | You see your own writes | Session guarantee |
| Monotonic reads | Never go back in time | |
| Eventual | Replicas converge eventually | DNS, Cassandra with CL=ONE |

---

## 12. Caching in System Design

Full deep dive: [CachingTechniques.md](./CachingTechniques.md). Interview essentials:

### Where to cache

```text
Client (browser/app) → CDN → API gateway → App (in-process) → Distributed cache (Redis) → DB (buffer pool)
```

### Patterns

| Pattern | Read path | Write path | Use |
| --- | --- | --- | --- |
| **Cache-aside** (default) | App checks cache, on miss reads DB and fills cache | App writes DB, then **deletes** cache key | General purpose |
| Read-through | Cache library loads from DB itself | — | Simplifies app code |
| Write-through | — | Write cache and DB synchronously | Read-after-write consistency |
| Write-behind | — | Write cache, flush to DB asynchronously | Very write-heavy, can tolerate loss |
| Refresh-ahead | Refresh hot keys before TTL expires | — | Predictable hot keys |

```ts
// Cache-aside with stampede protection (single-flight per key)
const inflight = new Map<string, Promise<unknown>>();

async function getUser(id: string): Promise<User> {
  const key = `user:${id}`;
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  if (!inflight.has(key)) {
    inflight.set(key, (async () => {
      try {
        const user = await db.users.findById(id);
        const ttl = 300 + Math.floor(Math.random() * 60);   // jitter → no synchronized expiry
        await redis.set(key, JSON.stringify(user), "EX", ttl);
        return user;
      } finally {
        inflight.delete(key);
      }
    })());
  }
  return inflight.get(key) as Promise<User>;
}

async function updateUser(id: string, patch: Partial<User>) {
  await db.users.update(id, patch);   // 1. source of truth first
  await redis.del(`user:${id}`);      // 2. invalidate (delete, don't set → avoids races)
}
```

### Cache problems interviewers love

| Problem | Description | Fix |
| --- | --- | --- |
| Cache stampede (thundering herd) | Hot key expires and thousands of requests hit the DB | Single-flight/lock, early probabilistic refresh, TTL jitter |
| Cache penetration | Queries for keys that don't exist always miss | Cache nulls briefly, bloom filter in front |
| Cache avalanche | Many keys expire at once, or the cache dies | TTL jitter, replicas, circuit breaker on the DB |
| Hot key | One key overloads one Redis shard | Local in-process cache, key replication (`key#1..n`) |
| Inconsistency | DB and cache diverge | Delete on write, short TTL, CDC-driven invalidation |

**Eviction:** LRU (default), LFU (stable hot sets), TTL-based. See the [LRU Cache LLD](#48-lru-cache).

---
## 13. Message Queues, Streams, and Async Processing

### Why use a queue

- **Decoupling:** The producer doesn't need the consumer to be up.
- **Buffering spikes:** Absorb bursts and process at a steady rate (load leveling).
- **Async work:** Move slow tasks (video transcoding, emails) off the request path.
- **Fan-out:** One event goes to many consumers.

### Queue vs log (stream)

| | Message queue (RabbitMQ, SQS) | Log / stream (Kafka, Kinesis, Pulsar) |
| --- | --- | --- |
| Model | Message deleted after ACK | Append-only log, retained for days |
| Consumers | Competing consumers share work | Consumer **groups**. Each group gets every message. |
| Replay | No | Yes (rewind offset) |
| Ordering | Per queue (FIFO queues), limited | Per **partition** |
| Use | Task queues, jobs | Event streaming, CDC, analytics, event sourcing |

### Kafka in 60 seconds

```text
Topic "orders" (3 partitions, replication factor 3)
  P0: [m0 m3 m6 ...]  leader broker1, followers broker2/3
  P1: [m1 m4 m7 ...]
  P2: [m2 m5 m8 ...]
Producer: partition = hash(key) % 3  → same key (order_id) ⇒ same partition ⇒ ordered
Consumer group "billing": C1←P0, C2←P1, C3←P2 (max parallelism = #partitions)
Consumer group "analytics": independently reads all partitions with its own offsets
```

### Delivery semantics

| Semantics | How | Risk |
| --- | --- | --- |
| At-most-once | Commit the offset *before* processing | Message loss |
| At-least-once (default) | Commit the offset *after* processing | **Duplicates**, so consumers must be idempotent |
| Exactly-once | Kafka transactions + idempotent producer, or at-least-once + dedupe | Complex. "Effectively once" = at-least-once + idempotent consumer. |

```ts
// Idempotent consumer: dedupe by message ID inside the same DB transaction as the side effect
async function handle(msg: { id: string; orderId: string; amount: number }) {
  await db.tx(async (tx) => {
    const inserted = await tx.query(
      `INSERT INTO processed_messages (message_id) VALUES ($1) ON CONFLICT DO NOTHING RETURNING 1`,
      [msg.id],
    );
    if (inserted.rowCount === 0) return;                 // duplicate → skip
    await tx.query(`UPDATE accounts SET balance = balance - $1 WHERE order_id = $2`, [msg.amount, msg.orderId]);
  });
}
```

### Retries, DLQ, backpressure

- **Retry with exponential backoff.** After N failures, send to a **dead-letter queue (DLQ)** for inspection. Don't block the partition with a "poison" message.
- **Backpressure:** When consumers lag, bound queue size, shed load, autoscale consumers on lag, or slow producers.

### Transactional outbox (do both the DB write and the publish reliably)

Problem: `db.save(order); kafka.publish(OrderCreated)` is a **dual write**. If the process crashes between the two, the systems become inconsistent.

```ts
// 1. In ONE local transaction: write business row + outbox row
await db.tx(async (tx) => {
  await tx.query(`INSERT INTO orders (id, user_id, total) VALUES ($1,$2,$3)`, [id, userId, total]);
  await tx.query(
    `INSERT INTO outbox (id, aggregate_id, type, payload) VALUES (gen_random_uuid(), $1, 'OrderCreated', $2)`,
    [id, JSON.stringify({ id, userId, total })],
  );
});

// 2. Relay (poller or CDC like Debezium) publishes outbox rows, then marks them sent.
async function relay() {
  const rows = await db.query(
    `SELECT * FROM outbox WHERE sent_at IS NULL ORDER BY created_at LIMIT 100 FOR UPDATE SKIP LOCKED`,
  );
  for (const r of rows) {
    await kafka.send({ topic: "orders", messages: [{ key: r.aggregate_id, value: r.payload, headers: { id: r.id } }] });
    await db.query(`UPDATE outbox SET sent_at = now() WHERE id = $1`, [r.id]);
  }
}
// At-least-once publish → consumers dedupe by header id.
```

### Change Data Capture (CDC)

Tail the DB's replication log (Postgres WAL, MySQL binlog) with Debezium and stream changes to Kafka. Uses: keep search indexes and caches in sync, feed analytics, and implement outbox relays.

---

## 14. Rate Limiting

Protect services from abuse and overload. Limit by user ID, API key, or IP. Return **HTTP 429** with `Retry-After` and `X-RateLimit-Remaining` headers.

| Algorithm | Idea | Pros | Cons |
| --- | --- | --- | --- |
| **Token bucket** | Bucket of capacity B refills at rate r. Each request takes 1 token. | Allows bursts. Memory-cheap. Used by AWS and Stripe. | Two parameters to tune |
| Leaky bucket | Queue drains at a fixed rate | Smooth output rate | Bursts get delayed or dropped |
| Fixed window counter | Count per minute window | Simplest | 2× burst at window edges |
| Sliding window log | Store each request timestamp | Exact | Memory O(requests) |
| **Sliding window counter** | Weighted current + previous window | Accurate enough, O(1) memory | Approximation |

```ts
class TokenBucket {
  private tokens: number;
  private lastRefill = Date.now();
  constructor(private capacity: number, private refillPerSec: number) {
    this.tokens = capacity;
  }
  tryConsume(n = 1): boolean {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsed * this.refillPerSec);
    this.lastRefill = now;
    if (this.tokens >= n) { this.tokens -= n; return true; }
    return false;
  }
}

class SlidingWindowCounter {
  private windows = new Map<number, number>(); // windowStart → count
  constructor(private limit: number, private windowMs: number) {}
  allow(now = Date.now()): boolean {
    const curStart = Math.floor(now / this.windowMs) * this.windowMs;
    const prevStart = curStart - this.windowMs;
    const cur = this.windows.get(curStart) ?? 0;
    const prev = this.windows.get(prevStart) ?? 0;
    const prevWeight = 1 - (now - curStart) / this.windowMs;   // overlap fraction of previous window
    if (cur + prev * prevWeight >= this.limit) return false;
    this.windows.set(curStart, cur + 1);
    for (const k of this.windows.keys()) if (k < prevStart) this.windows.delete(k);
    return true;
  }
}
```

The distributed version (a Redis Lua script) is in [Section 27](#27-distributed-rate-limiter).

---

## 15. Unique ID Generation

| Approach | Pros | Cons |
| --- | --- | --- |
| DB auto-increment | Simple, ordered | SPOF, doesn't scale across shards (can use step offsets: shard1 odd, shard2 even) |
| UUID v4 (random 128-bit) | No coordination | Not sortable. Random inserts fragment B-tree indexes. 128 bits. |
| UUID v7 / ULID | Time-ordered + random, no coordination | 128 bits |
| Ticket server (Flickr) | Central counter (`REPLACE INTO`) | SPOF unless doubled up |
| Range allocation (each server leases a block of 10k IDs) | Fast, near-sorted, little coordination | Gaps on crash |
| **Twitter Snowflake** | 64-bit: time + machine + sequence | Sortable by time, 64-bit, ~4M IDs/s per machine | Clock skew. Needs a machine-ID assignment. |

```text
Snowflake 64-bit layout
| 1 bit sign | 41 bits ms timestamp (since custom epoch, ~69 yrs) | 5 bits DC | 5 bits machine | 12 bits sequence |
                                                                    → 1024 machines, 4096 IDs/ms/machine
```

```ts
class Snowflake {
  private static readonly EPOCH = 1704067200000n; // 2024-01-01
  private sequence = 0n;
  private lastTs = -1n;

  constructor(private datacenterId: bigint, private machineId: bigint) {
    if (datacenterId > 31n || machineId > 31n) throw new Error("id out of range");
  }

  nextId(): bigint {
    let ts = BigInt(Date.now());
    if (ts < this.lastTs) throw new Error("clock moved backwards; refuse to generate");
    if (ts === this.lastTs) {
      this.sequence = (this.sequence + 1n) & 0xfffn;     // 12 bits
      if (this.sequence === 0n) {                         // exhausted this ms → spin to next ms
        while (ts <= this.lastTs) ts = BigInt(Date.now());
      }
    } else {
      this.sequence = 0n;
    }
    this.lastTs = ts;
    return ((ts - Snowflake.EPOCH) << 22n) | (this.datacenterId << 17n) | (this.machineId << 12n) | this.sequence;
  }

  static timestampOf(id: bigint): Date {
    return new Date(Number((id >> 22n) + Snowflake.EPOCH));
  }
}
```

---

## 16. Distributed Transactions, Sagas, and Idempotency

When one business operation touches several services or shards, you can't use a single ACID transaction.

### Two-phase commit (2PC)

```text
Coordinator ──PREPARE──▶ P1, P2       (each locks resources, votes YES/NO)
Coordinator ──COMMIT───▶ P1, P2       (if all YES) else ABORT
```

- Atomic, but **blocking**: if the coordinator dies after PREPARE, participants hold locks. It's slow and tightly coupled. Use it only inside one system (e.g. XA, Spanner internals).

### Saga pattern (preferred for microservices)

A sequence of local transactions. Each step has a **compensating action** that undoes it if a later step fails.

| | Choreography | Orchestration |
| --- | --- | --- |
| How | Services react to each other's events | A central orchestrator calls each step |
| Pros | Loose coupling | Clear flow, easier to debug and monitor |
| Cons | Hard to trace, cyclic dependencies | Orchestrator is a central dependency |
| Tools | Kafka events | Temporal, AWS Step Functions, Camunda |

```ts
type Step = { name: string; run: () => Promise<void>; compensate: () => Promise<void> };

async function runSaga(steps: Step[]) {
  const done: Step[] = [];
  for (const step of steps) {
    try {
      await step.run();
      done.push(step);
    } catch (err) {
      for (const s of done.reverse()) {
        await retry(() => s.compensate());   // compensations must be idempotent + retried
      }
      throw new Error(`saga failed at ${step.name}: ${err}`);
    }
  }
}

await runSaga([
  { name: "reserveInventory", run: () => inventory.reserve(orderId, items), compensate: () => inventory.release(orderId) },
  { name: "chargePayment",    run: () => payments.charge(orderId, total),   compensate: () => payments.refund(orderId) },
  { name: "createShipment",   run: () => shipping.create(orderId),          compensate: () => shipping.cancel(orderId) },
]);
```

In production, persist saga state (a workflow engine like **Temporal** does this) so a crashed orchestrator resumes where it stopped.

### Idempotency keys (essential for payments)

Network retries mean the same request can arrive twice. The client sends a unique `Idempotency-Key`. The server stores the result and returns it for repeats.

```ts
app.post("/v1/payments", async (req, res) => {
  const key = req.header("Idempotency-Key");
  if (!key) return res.status(400).send("Idempotency-Key required");

  // Atomically claim the key; unique constraint on (user_id, key)
  const claimed = await db.query(
    `INSERT INTO idempotency (user_id, key, status) VALUES ($1,$2,'processing')
     ON CONFLICT (user_id, key) DO NOTHING RETURNING 1`,
    [req.user.id, key],
  );
  if (claimed.rowCount === 0) {
    const row = await db.one(`SELECT status, response FROM idempotency WHERE user_id=$1 AND key=$2`, [req.user.id, key]);
    if (row.status === "processing") return res.status(409).send("in progress");
    return res.status(200).json(row.response);           // replay stored response
  }

  const result = await paymentProvider.charge(req.body, { idempotencyKey: key });
  await db.query(`UPDATE idempotency SET status='done', response=$3 WHERE user_id=$1 AND key=$2`,
    [req.user.id, key, result]);
  res.status(201).json(result);
});
```

---

## 17. Contention: Locks and Concurrency Control

Hello Interview's "Dealing with Contention" pattern. This appears in Ticketmaster, flash sales, auctions, and inventory problems.

| Technique | How | When |
| --- | --- | --- |
| Pessimistic lock | `SELECT ... FOR UPDATE` locks the row until commit | High contention, short transactions, single DB |
| Optimistic concurrency control (OCC) | Version column. Update only if the version is unchanged, otherwise retry. | Low or moderate contention |
| Atomic conditional update | `UPDATE ... SET stock = stock - 1 WHERE id=? AND stock > 0` | Counters, inventory (simplest and best) |
| Distributed lock with TTL | Redis `SET key val NX PX 30000` | Cross-service reservation holds (e.g. 10-minute seat hold) |
| Serializable isolation | DB detects conflicts | Complex invariants, low throughput |
| Queue / single writer | Serialize all writes for a key through one partition | Extreme hot items (flash sale) |

```sql
-- Optimistic locking
UPDATE seats SET status = 'BOOKED', user_id = :u, version = version + 1
WHERE id = :seat AND version = :expectedVersion AND status = 'AVAILABLE';
-- rowsAffected == 0 → someone else won → retry or report "seat taken"

-- Pessimistic
BEGIN;
SELECT * FROM seats WHERE id = :seat FOR UPDATE;   -- others block here
UPDATE seats SET status='BOOKED' WHERE id = :seat;
COMMIT;
```

```ts
// Redis distributed lock with safe release (only the owner may delete)
const RELEASE = `if redis.call("get", KEYS[1]) == ARGV[1] then return redis.call("del", KEYS[1]) else return 0 end`;

async function withLock<T>(key: string, ttlMs: number, fn: () => Promise<T>): Promise<T> {
  const token = crypto.randomUUID();
  const ok = await redis.set(key, token, "PX", ttlMs, "NX");
  if (!ok) throw new Error("lock busy");
  try {
    return await fn();
  } finally {
    await redis.eval(RELEASE, 1, key, token);
  }
}
```

**Caveat:** A Redis lock can expire while the holder is paused (GC pause, network delay). For correctness-critical writes, add a **fencing token**: an increasing number issued with the lock that the storage layer checks, rejecting writes with a stale token. ZooKeeper and etcd give you this through zxid or revision numbers.

---

## 18. Blob Storage and CDNs

### Large files: never push them through your app servers

```text
1. Client → API:  POST /uploads {filename, size, sha256}
2. API → S3: create presigned PUT URL (or multipart upload), save metadata row status=PENDING
3. Client ──PUT bytes directly──▶ S3      (parallel multipart chunks, resumable)
4. S3 event → queue → worker: validate, scan, transcode/thumbnail → status=READY
5. Reads: client ← CDN ← S3 (signed URLs / signed cookies for private content)
```

```ts
import { S3Client, PutObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({ region: "us-east-1" });

app.post("/uploads", async (req, res) => {
  const key = `uploads/${req.user.id}/${crypto.randomUUID()}`;
  await db.files.insert({ key, owner: req.user.id, status: "PENDING", size: req.body.size });
  const url = await getSignedUrl(s3, new PutObjectCommand({
    Bucket: "media", Key: key, ContentType: req.body.contentType,
  }), { expiresIn: 900 });
  res.json({ key, uploadUrl: url });
});
```

- **Multipart upload:** Split into 5–100 MB parts, upload in parallel, retry failed parts only, and support resume.
- **Durability:** S3 is designed for 11 nines through erasure coding across availability zones.

### CDN

- **Push CDN:** You upload content to the CDN ahead of time. Good for small or rarely changing sites.
- **Pull CDN:** The edge fetches from the origin on the first miss and caches it per TTL. This is the default.
- Version asset URLs (`app.3f9a1c.js`) so you never need to invalidate.
- Use signed URLs for private media. For video, the CDN serves HLS/DASH segments (Section 30).

---

## 19. Search and Inverted Indexes

An inverted index maps each **term** to a **posting list** of documents that contain it.

```ts
class InvertedIndex {
  private index = new Map<string, Map<string, number>>(); // term → (docId → term frequency)
  private docLengths = new Map<string, number>();

  private tokenize(text: string): string[] {
    return text.toLowerCase().replace(/[^a-z0-9\s]/g, " ").split(/\s+/).filter((t) => t && !STOP.has(t));
  }

  add(docId: string, text: string) {
    const terms = this.tokenize(text);
    this.docLengths.set(docId, terms.length);
    for (const t of terms) {
      const postings = this.index.get(t) ?? new Map();
      postings.set(docId, (postings.get(docId) ?? 0) + 1);
      this.index.set(t, postings);
    }
  }

  // TF-IDF ranking (Elasticsearch uses BM25, a refinement)
  search(query: string, k = 10): { docId: string; score: number }[] {
    const N = this.docLengths.size;
    const scores = new Map<string, number>();
    for (const t of this.tokenize(query)) {
      const postings = this.index.get(t);
      if (!postings) continue;
      const idf = Math.log(N / postings.size);
      for (const [doc, tf] of postings) {
        scores.set(doc, (scores.get(doc) ?? 0) + (tf / this.docLengths.get(doc)!) * idf);
      }
    }
    return [...scores].map(([docId, score]) => ({ docId, score })).sort((a, b) => b.score - a.score).slice(0, k);
  }
}
const STOP = new Set(["the", "a", "an", "and", "of", "to", "is"]);
```

**In interviews:** Keep the primary DB as the source of truth and sync to **Elasticsearch** through CDC/Kafka. Search becomes eventually consistent, typically within about 1 s. Elasticsearch shards the index across nodes and runs queries as scatter-gather. Mention analyzers (stemming, n-grams for prefix search) and that Elasticsearch is not a primary database.

---

## 20. Geospatial Indexing

"Find drivers or restaurants near me" needs 2-D lookups, which a single B-tree index can't do well.

| Technique | Idea | Notes |
| --- | --- | --- |
| **Geohash** | Interleave lat/lng bits → base32 string. A shared prefix means nearby. | Works with any KV store or B-tree. Query the cell plus its 8 neighbors (edge problem). |
| **Quadtree** | Recursively split a region into 4 until each leaf holds ≤ K points | Adapts to density (cities vs oceans). In-memory. |
| Uber H3 | Hexagonal hierarchical grid | All neighbors equidistant |
| Google S2 | Sphere → cube → Hilbert curve cells | Used by Google Maps, Foursquare |
| PostGIS / R-tree | DB-native spatial index | Great default for moderate scale |
| Redis GEO | `GEOADD` / `GEOSEARCH` (geohash in a sorted set) | Great for high-churn locations (drivers) |

| Geohash length | Cell size (approx) |
| --- | --- |
| 4 | 39 km × 19.5 km |
| 5 | 4.9 km × 4.9 km |
| 6 | 1.2 km × 0.6 km |
| 7 | 153 m × 153 m |

```ts
const BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz";

function geohashEncode(lat: number, lng: number, precision = 6): string {
  let latR = [-90, 90], lngR = [-180, 180];
  let hash = "", bit = 0, ch = 0, even = true;
  while (hash.length < precision) {
    const r = even ? lngR : latR;
    const v = even ? lng : lat;
    const mid = (r[0] + r[1]) / 2;
    if (v >= mid) { ch = (ch << 1) | 1; r[0] = mid; } else { ch = ch << 1; r[1] = mid; }
    even = !even;
    if (++bit === 5) { hash += BASE32[ch]; bit = 0; ch = 0; }
  }
  return hash;
}
// geohashEncode(37.7749, -122.4194, 6) → "9q8yyk" (San Francisco)
```

```ts
// Quadtree (point region)
type Pt = { id: string; x: number; y: number };
type Box = { x: number; y: number; w: number; h: number };
const contains = (b: Box, p: Pt) => p.x >= b.x && p.x < b.x + b.w && p.y >= b.y && p.y < b.y + b.h;
const intersects = (a: Box, b: Box) => !(b.x > a.x + a.w || b.x + b.w < a.x || b.y > a.y + a.h || b.y + b.h < a.y);

class QuadTree {
  private points: Pt[] = [];
  private children: QuadTree[] | null = null;
  constructor(private box: Box, private capacity = 100) {}

  insert(p: Pt): boolean {
    if (!contains(this.box, p)) return false;
    if (!this.children && this.points.length < this.capacity) { this.points.push(p); return true; }
    if (!this.children) this.subdivide();
    return this.children!.some((c) => c.insert(p));
  }

  private subdivide() {
    const { x, y, w, h } = this.box, hw = w / 2, hh = h / 2;
    this.children = [
      new QuadTree({ x, y, w: hw, h: hh }, this.capacity),
      new QuadTree({ x: x + hw, y, w: hw, h: hh }, this.capacity),
      new QuadTree({ x, y: y + hh, w: hw, h: hh }, this.capacity),
      new QuadTree({ x: x + hw, y: y + hh, w: hw, h: hh }, this.capacity),
    ];
    for (const p of this.points) this.children.some((c) => c.insert(p));
    this.points = [];
  }

  query(range: Box, out: Pt[] = []): Pt[] {
    if (!intersects(this.box, range)) return out;
    for (const p of this.points) if (contains(range, p)) out.push(p);
    this.children?.forEach((c) => c.query(range, out));
    return out;
  }
}
```

**Choosing:** For high-frequency location updates (drivers every 4 s), use **Redis GEO** or geohash buckets, because rebuilding a quadtree on every update is costly. For mostly static places (Yelp), use a **quadtree** or **PostGIS**.

---

## 21. Probabilistic Data Structures

Trade a small error for huge memory savings.

| Structure | Answers | Error | Use |
| --- | --- | --- | --- |
| **Bloom filter** | "Is X in the set?" | False positives possible, **no false negatives** | Crawler seen-URLs, cache penetration, LSM SSTable skipping |
| **Count-Min Sketch** | "How often has X appeared?" | Overestimates | Top-K heavy hitters, trending |
| **HyperLogLog** | "How many distinct items?" | ~0.81% with 12 KB | Unique visitors (Redis `PFADD`/`PFCOUNT`) |

```ts
class BloomFilter {
  private bits: Uint8Array;
  constructor(private m: number, private k: number) { this.bits = new Uint8Array(Math.ceil(m / 8)); }

  // Optimal sizing: m = -n ln p / (ln2)^2, k = (m/n) ln2
  static forCapacity(n: number, p = 0.01) {
    const m = Math.ceil((-n * Math.log(p)) / Math.LN2 ** 2);
    return new BloomFilter(m, Math.max(1, Math.round((m / n) * Math.LN2)));
  }

  private hashes(s: string): number[] {
    // double hashing: h_i = h1 + i*h2
    let h1 = 2166136261, h2 = 5381;
    for (let i = 0; i < s.length; i++) {
      h1 = Math.imul(h1 ^ s.charCodeAt(i), 16777619) >>> 0;
      h2 = ((h2 << 5) + h2 + s.charCodeAt(i)) >>> 0;
    }
    return Array.from({ length: this.k }, (_, i) => (h1 + i * h2) % this.m);
  }

  add(s: string) { for (const h of this.hashes(s)) this.bits[h >> 3] |= 1 << (h & 7); }
  mightContain(s: string) { return this.hashes(s).every((h) => (this.bits[h >> 3] & (1 << (h & 7))) !== 0); }
}
// 1 billion URLs at 1% FP ≈ 1.2 GB of bits (vs ~100 GB storing URLs)
```

```ts
class CountMinSketch {
  private table: Uint32Array[];
  constructor(private width = 2048, private depth = 5) {
    this.table = Array.from({ length: depth }, () => new Uint32Array(width));
  }
  private h(s: string, seed: number) {
    let h = seed ^ 2166136261;
    for (let i = 0; i < s.length; i++) h = Math.imul(h ^ s.charCodeAt(i), 16777619);
    return (h >>> 0) % this.width;
  }
  add(s: string, n = 1) { for (let d = 0; d < this.depth; d++) this.table[d][this.h(s, d)] += n; }
  estimate(s: string) { return Math.min(...this.table.map((row, d) => row[this.h(s, d)])); }
}
```

---

## 22. Reliability Patterns

| Pattern | Problem | Solution |
| --- | --- | --- |
| Timeouts | Hung calls tie up threads | Always set connect + read timeouts below the caller's timeout |
| **Retry + exponential backoff + jitter** | Transient failures | Retry idempotent ops with randomized delays to avoid synchronized retry storms |
| **Circuit breaker** | Calling a dead dependency piles up failures | Fail fast after N errors, probe after a cooldown |
| Bulkhead | One slow dependency exhausts all threads | Separate pools or limits per dependency |
| Fallback / graceful degradation | Dependency down | Serve cached or default data, hide the feature |
| Load shedding | Overload | Reject low-priority work early (429/503) to protect core paths |
| Health checks + autoscaling | Bad instances, load growth | LB ejects unhealthy instances, autoscaler adds capacity |
| Redundancy + multi-AZ | Hardware or zone failure | N+1 or N+2 capacity across availability zones |

```ts
async function retryWithBackoff<T>(fn: () => Promise<T>, opts = { retries: 5, baseMs: 100, maxMs: 10_000 }): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt >= opts.retries || !isRetryable(err)) throw err;
      const cap = Math.min(opts.maxMs, opts.baseMs * 2 ** attempt);
      const delay = Math.random() * cap;            // "full jitter" (AWS Architecture Blog)
      await new Promise((r) => setTimeout(r, delay));
    }
  }
}
const isRetryable = (e: any) => [429, 502, 503, 504].includes(e?.status) || e?.code === "ECONNRESET";
```

```ts
type State = "CLOSED" | "OPEN" | "HALF_OPEN";

class CircuitBreaker {
  private state: State = "CLOSED";
  private failures = 0;
  private openedAt = 0;
  constructor(private threshold = 5, private cooldownMs = 30_000) {}

  async call<T>(fn: () => Promise<T>, fallback?: () => T): Promise<T> {
    if (this.state === "OPEN") {
      if (Date.now() - this.openedAt < this.cooldownMs) {
        if (fallback) return fallback();
        throw new Error("circuit open");
      }
      this.state = "HALF_OPEN";                       // let one trial request through
    }
    try {
      const result = await fn();
      this.state = "CLOSED";
      this.failures = 0;
      return result;
    } catch (err) {
      this.failures++;
      if (this.state === "HALF_OPEN" || this.failures >= this.threshold) {
        this.state = "OPEN";
        this.openedAt = Date.now();
      }
      if (fallback) return fallback();
      throw err;
    }
  }
}
```

### Disaster recovery vocabulary

- **RPO (Recovery Point Objective):** How much data loss is acceptable. Async replication means RPO > 0.
- **RTO (Recovery Time Objective):** How long until service is restored.
- Strategies, from cheapest to fastest recovery: backup/restore → pilot light → warm standby → multi-region active-active.

---

## 23. Consensus, Leader Election, and Coordination

- **Consensus:** Nodes agree on a value or log order despite failures. **Raft** (etcd, Consul, CockroachDB) and **Paxos** (Spanner, Chubby) are the main algorithms. They need a **majority quorum**: 2f+1 nodes tolerate f failures, so 3 nodes tolerate 1 and 5 tolerate 2.
- **Raft in 4 lines:** (1) Nodes are follower, candidate, or leader. (2) A follower with no heartbeat before a randomized timeout becomes a candidate and requests votes for a new **term**. (3) A majority vote makes it leader. (4) The leader replicates log entries, and an entry is committed once a majority stores it.
- **Coordination services** (ZooKeeper, etcd) provide leader election, distributed locks, service discovery, and configuration. They're built on consensus. Use them for **small metadata**, never bulk data.
- **Gossip protocols** (Cassandra, Consul membership) spread state epidemically. They're decentralized and eventually consistent, and they're used for failure detection.
- **Vector clocks** detect concurrent writes in leaderless stores (Dynamo). **Lamport clocks** give a causal order.

```ts
// Vector clock: detect whether two versions are ordered or concurrent (conflict)
type VC = Record<string, number>;
const increment = (vc: VC, node: string): VC => ({ ...vc, [node]: (vc[node] ?? 0) + 1 });
function compare(a: VC, b: VC): "before" | "after" | "equal" | "concurrent" {
  let aLess = false, bLess = false;
  for (const k of new Set([...Object.keys(a), ...Object.keys(b)])) {
    if ((a[k] ?? 0) < (b[k] ?? 0)) aLess = true;
    if ((a[k] ?? 0) > (b[k] ?? 0)) bLess = true;
  }
  if (aLess && bLess) return "concurrent";
  return aLess ? "before" : bLess ? "after" : "equal";
}
```

---

## 24. Observability

- **Metrics:** Track RED (Rate, Errors, Duration) for services and USE (Utilization, Saturation, Errors) for resources. Alert on SLO burn rate, not on CPU.
- **Logs:** Structured JSON with a request/trace ID.
- **Traces:** Distributed tracing (OpenTelemetry) across services to find where latency comes from.
- **SLI / SLO / SLA:** An SLI is the measurement (p99 latency). An SLO is the internal target (p99 < 300 ms for 99.9% of minutes). An SLA is the contractual promise with penalties.
- In interviews, mention monitoring in the wrap-up: "We'd alert on consumer lag, p99 latency, 5xx rate, and cache hit ratio."

---

## 25. Microservices vs Monolith

| | Monolith (or modular monolith) | Microservices |
| --- | --- | --- |
| Pros | Simple deploys, local calls, easy transactions, easy debugging | Independent deploys and scaling, team autonomy, fault isolation, mixed tech |
| Cons | Scales as one unit, large codebase coupling | Network failures, distributed transactions, observability burden, operational cost |
| Choose when | Small team, early product | Many teams, clear domain boundaries, different scaling needs |

Split along **bounded contexts** (Domain-Driven Design). Each service owns its data, and no other service reads its database directly. Services communicate through APIs (sync) or events (async). In interviews, splitting into a few services by scaling profile is good. For example, "Upload service vs Feed service: different load shapes, so scale them separately."

---
# Part III: HLD Case Studies

Each case follows the delivery framework: **Requirements → Entities → API → High-level design → Deep dives**. Try each one yourself for 45 minutes before reading it.

## 26. URL Shortener (Bitly)

References: Alex Xu Vol. 1 Ch. 8; [Hello Interview: Bitly](https://www.hellointerview.com/learn/system-design/problem-breakdowns/bitly).

**Functional:** (1) Create a short URL from a long URL, with an optional custom alias and expiry. (2) Visiting a short URL redirects to the original.
**Non-functional:** Redirect latency < 100 ms. 100M DAU. Read-heavy (about 100:1). Short codes are unique. High availability for redirects.

**Estimation:** 100M new URLs/day ≈ 1.2k writes/s and 120k reads/s at 100:1. Over 5 years that's 180B URLs × ~500 B ≈ 90 TB. A 7-character base62 code gives 62^7 ≈ 3.5 trillion codes, which is enough.

**API:**

```http
POST /urls      { longUrl, customAlias?, expiresAt? }  → 201 { shortUrl }
GET  /{code}    → 302 Location: longUrl
```

**301 vs 302:** A 301 (permanent) is cached by browsers, so there's less load but you lose click analytics. A 302 (temporary) sends every click through you, which keeps analytics. Choose 302 if analytics matter.

```text
          ┌─────────── write path ───────────┐
Client ──▶ LB ──▶ Shortener svc ──▶ ID generator (Redis INCR / counter ranges)
   │                    │
   │                    └──▶ DB: urls(code PK, long_url, user_id, expires_at, created_at)
   │
   └── GET /abc123 ──▶ LB ──▶ Redirect svc ──▶ Redis cache (code → long_url) ──miss──▶ DB
                                     └──async click event──▶ Kafka ──▶ analytics
```

### Deep dive: generating unique short codes

| Approach | How | Trade-off |
| --- | --- | --- |
| Hash + truncate | `base62(md5(longUrl))[:7]`, check for collision and re-hash with a salt | Collisions need a DB check. The same long URL gives the same code (could be desired). |
| **Counter + base62** | Global counter → base62 encode | No collisions, short. Codes are predictable (fix by shuffling bits or a bijective cipher). |
| Counter ranges | Each server leases 1M IDs from Redis/ZooKeeper | No per-request coordination |
| Pre-generated key service | Offline generate random codes, hand them out | Extra service, but random and collision-free |

```ts
const ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

function toBase62(n: bigint): string {
  if (n === 0n) return "0";
  let s = "";
  while (n > 0n) { s = ALPHABET[Number(n % 62n)] + s; n /= 62n; }
  return s;
}

// Each app server leases a block of IDs so Redis is hit once per 1000 creates
class IdRangeAllocator {
  private next = 0n;
  private end = 0n;
  constructor(private redis: Redis, private blockSize = 1000n) {}
  async nextId(): Promise<bigint> {
    if (this.next >= this.end) {
      const hi = BigInt(await this.redis.incrby("url:counter", Number(this.blockSize)));
      this.next = hi - this.blockSize;
      this.end = hi;
    }
    return this.next++;
  }
}

async function createShortUrl(longUrl: string, alias?: string): Promise<string> {
  const code = alias ?? toBase62(await allocator.nextId() + 100_000_000n); // offset → ≥ 5 chars
  // unique constraint on code; alias collision → 409
  await db.query(`INSERT INTO urls (code, long_url) VALUES ($1, $2)`, [code, longUrl]);
  return `https://sho.rt/${code}`;
}
```

### Deep dive: fast redirects at 120k reads/s

- Cache-aside in Redis (code → long_url). The 80/20 rule means caching about 20% of daily URLs gives a high hit rate.
- The redirect service is stateless and horizontally scaled. Shard the DB by `code` if needed. A single Postgres plus read replicas likely handles this load.
- CDN/edge caching of 301 responses for very hot links.
- Check expiry on read and clean up expired rows lazily or with a batch job.

---

## 27. Distributed Rate Limiter

References: Alex Xu Vol. 1 Ch. 4; [Hello Interview](https://www.hellointerview.com/learn/system-design/problem-breakdowns/distributed-rate-limiter).

**Functional:** Limit requests per client (user ID, IP, API key) per rule. Return 429 with headers.
**Non-functional:** Adds < 5 ms latency. Accurate across many gateway nodes. Fail-open if the limiter is down (usually). 1M requests/s.

```text
Client ──▶ API Gateway (rate-limit middleware) ──allowed──▶ services
                 │
                 ├──▶ Redis Cluster (counters, sharded by client key)
                 └──◀ rules config (from config svc / ZooKeeper, cached locally)
```

### Deep dive: atomicity

Several gateways hitting the same counter with a read-then-write would race. Use a **Lua script** so Redis runs the whole check atomically.

```lua
-- token_bucket.lua   KEYS[1]=bucket key   ARGV: capacity, refill_per_sec, now_ms, cost
local cap    = tonumber(ARGV[1])
local rate   = tonumber(ARGV[2])
local now    = tonumber(ARGV[3])
local cost   = tonumber(ARGV[4])
local b      = redis.call("HMGET", KEYS[1], "tokens", "ts")
local tokens = tonumber(b[1]) or cap
local ts     = tonumber(b[2]) or now
tokens = math.min(cap, tokens + (now - ts) / 1000 * rate)
local allowed = tokens >= cost
if allowed then tokens = tokens - cost end
redis.call("HSET", KEYS[1], "tokens", tokens, "ts", now)
redis.call("PEXPIRE", KEYS[1], math.ceil(cap / rate * 1000) * 2)
return { allowed and 1 or 0, math.floor(tokens) }
```

```ts
async function rateLimit(req: Request, res: Response, next: NextFunction) {
  const key = `rl:${req.user?.id ?? req.ip}:${req.route.path}`;
  const rule = rules.get(req.route.path) ?? { capacity: 100, refillPerSec: 10 };
  try {
    const [allowed, remaining] = await redis.evalsha(TOKEN_BUCKET_SHA, 1, key, rule.capacity, rule.refillPerSec, Date.now(), 1);
    res.set("X-RateLimit-Remaining", String(remaining));
    if (!allowed) return res.status(429).set("Retry-After", "1").send("Too Many Requests");
  } catch {
    // Redis down → fail open (availability) but emit a metric; fail closed for e.g. login endpoints
  }
  next();
}
```

### Other deep dives

- **Scale:** Shard Redis by client key (Redis Cluster). Hash-tag keys like `{user123}` so related keys land on the same slot.
- **Latency:** Keep a local in-memory pre-check per gateway and sync to Redis in batches, trading exactness for speed.
- **Multi-region:** Use per-region limits (limit / regions) or async global sync, accepting a slight overshoot.
- **Hot keys:** A single abusive IP is one Redis key. A local cache of "blocked until T" stops it at the gateway.

---

## 28. News Feed (Facebook / Twitter)

References: Alex Xu Vol. 1 Ch. 11; [Hello Interview: FB News Feed](https://www.hellointerview.com/learn/system-design/problem-breakdowns/fb-news-feed).

**Functional:** (1) Create posts. (2) Follow users. (3) View a feed of posts from followed users, newest first, paginated.
**Non-functional:** 500M DAU. Feed p99 < 500 ms. Eventual consistency OK (post visible within about 1 minute). Follow is unlimited, so some users have 100M+ followers.

**Entities:** User, Post, Follow(follower_id, followee_id).

```http
POST /posts                          { content, mediaIds }
PUT  /users/{id}/followers           (follow)
GET  /feed?cursor=&limit=20
```

### Fan-out on write vs read

| | Fan-out on **write** (push) | Fan-out on **read** (pull) |
| --- | --- | --- |
| How | On post, append post_id to every follower's precomputed feed | On read, query posts from everyone I follow and merge |
| Read cost | O(1): read my feed list | O(#followees) queries + merge |
| Write cost | O(#followers), huge for celebrities | O(1) |
| Good for | Normal users | Celebrities, inactive users |

**Answer: a hybrid.** Push for normal users. For celebrities (> ~10k–100k followers), don't fan out. At read time, merge the precomputed feed with recent posts from the celebrities the user follows.

```text
POST /posts ──▶ Post svc ──▶ Posts DB (sharded by post_id / Cassandra)
                    │
                    └──▶ Kafka "post-created" ──▶ Fan-out workers
                                                   │  for each follower (skip if author is celebrity):
                                                   │    ZADD feed:{followerId} ts postId ; ZREMRANGEBYRANK keep 500
                                                   ▼
                                           Feed cache (Redis sorted sets)

GET /feed ──▶ Feed svc ──▶ feed:{me} (Redis)  +  recent posts of celebrities I follow
                         ──▶ merge, sort by time ──▶ hydrate posts (Post cache) + authors (User cache)
```

```ts
async function fanOut(post: { id: string; authorId: string; createdAt: number }) {
  const followerCount = await graph.followerCount(post.authorId);
  if (followerCount > CELEBRITY_THRESHOLD) return;            // pulled at read time instead

  for await (const batch of graph.followersInBatches(post.authorId, 1000)) {
    const pipe = redis.pipeline();
    for (const followerId of batch) {
      pipe.zadd(`feed:${followerId}`, post.createdAt, post.id);
      pipe.zremrangebyrank(`feed:${followerId}`, 0, -501);    // keep newest 500
    }
    await pipe.exec();
  }
}

async function getFeed(userId: string, beforeTs = Date.now(), limit = 20) {
  const [pushed, celebs] = await Promise.all([
    redis.zrevrangebyscore(`feed:${userId}`, `(${beforeTs}`, "-inf", "WITHSCORES", "LIMIT", 0, limit),
    graph.celebritiesFollowedBy(userId),
  ]);
  const celebPosts = await postStore.recentByAuthors(celebs, beforeTs, limit);
  const merged = mergeByTimestamp(toPairs(pushed), celebPosts).slice(0, limit);
  return hydrate(merged);  // batch-get post bodies + author profiles from cache
}
```

### Other deep dives

- **Follower graph storage:** A `follows` table indexed both ways (follower → followees, followee → followers), or a graph-ish KV store.
- **Hot posts (viral likes/comments):** Sharded counters (Section 9) and a cache in front.
- **Inactive users:** Don't fan out to users inactive for 30+ days. Rebuild their feed on login.
- **Ranking:** Retrieve candidates, then a ranking service scores them, then return the top N. Mention it, but chronological order is usually enough for the interview.

---

## 29. Chat System (WhatsApp)

References: Alex Xu Vol. 1 Ch. 12; [Hello Interview: WhatsApp](https://www.hellointerview.com/learn/system-design/problem-breakdowns/whatsapp).

**Functional:** (1) 1:1 and group chats (≤ 100 members). (2) Send and receive messages in real time. (3) Offline delivery when the user reconnects. (4) Media attachments. Optional: delivered/read receipts, online presence.
**Non-functional:** < 500 ms delivery. Messages never lost once acknowledged, and ordered within a chat. 1B users, ~100M concurrent connections.

```text
Client ══WebSocket══▶ Chat server (stateful, holds connections; L4 LB / consistent hashing)
                         │ 1. persist message (Cassandra: partition chat_id, cluster message_id)
                         │ 2. ACK sender (message stored → "sent ✓")
                         │ 3. lookup recipients' connection servers (Redis: userId → serverId)
                         │ 4. publish to recipient's server (Redis Pub/Sub / direct gRPC)
                         ▼
                   Recipient's chat server ══▶ recipient device ("delivered ✓✓")
                         │ offline? → store in inbox table; push notification (APNs/FCM)

Media: client → presigned S3 upload → send message with media URL
Presence: heartbeat every 30 s → Redis key with TTL; publish changes to friends lazily
```

**Data model:**

```sql
messages(chat_id, message_id TIMEUUID, sender_id, body, media_url)  -- PK ((chat_id), message_id)
chat_members(chat_id, user_id)              -- and inverse user_chats(user_id, chat_id)
inbox(user_id, message_id, chat_id)         -- undelivered messages per device; deleted on ACK
```

### Deep dives

- **Delivery guarantee:** The client resends until the server ACKs, deduping by a client-generated `clientMessageId`. The server keeps the message in the recipient's inbox until the *recipient's* device ACKs it. Together this gives at-least-once delivery with idempotent dedupe.
- **Ordering:** A per-chat sequence number or Snowflake ID. Clients sort by it, not by arrival time.
- **Routing between servers:** Redis Pub/Sub channel per user (subscribe when connected), or a registry lookup then direct server-to-server RPC.
- **Group fan-out:** For ≤ 100 members, fan out on write to each member's server or inbox. Large channels (Slack/Discord) fan out by reading per-channel logs instead.
- **Multi-device:** The inbox is tracked per device. Each device has its own cursor.
- **Scale numbers:** ~100M connections at ~1M per tuned server (C10M, Erlang/Elixir) → about 100–200 servers. Plan for 10× that with headroom at ~100k per server.

---

## 30. Video Streaming (YouTube)

References: Alex Xu Vol. 1 Ch. 14; [Hello Interview: YouTube](https://www.hellointerview.com/learn/system-design/problem-breakdowns/youtube).

**Functional:** Upload videos. Watch videos (adaptive quality). View metadata.
**Non-functional:** Large uploads (up to 10s of GB, resumable). Low startup latency and smooth playback worldwide. High availability for viewing (eventual consistency OK for new uploads). 1M uploads/day, 100M+ views/day.

```text
UPLOAD
Client ──▶ API: POST /videos (metadata) → presigned multipart URLs
Client ══chunks══▶ S3 (raw bucket)
S3 event ──▶ Kafka/SQS ──▶ Transcoding pipeline (DAG):
     split into segments ─▶ parallel workers encode each segment × {240p,480p,720p,1080p,4K} × {H.264, VP9/AV1}
     ─▶ generate HLS/DASH manifests (.m3u8/.mpd) ─▶ thumbnails ─▶ S3 (processed) ─▶ metadata status=READY

WATCH
Client ──GET /videos/{id}──▶ API → metadata + manifest URL
Client ──▶ CDN ──▶ manifest ──▶ fetch 2–10 s segments; player switches bitrate based on bandwidth (ABR)
```

### Deep dives

- **Adaptive bitrate streaming (HLS/DASH):** The video is cut into small segments at several bitrates. The manifest lists them, and the player picks the quality per segment based on measured throughput.
- **Parallel transcoding:** Splitting into segments makes encoding parallel, cutting a 1-hour video from hours to minutes. Orchestrate with a DAG (Temporal or Step Functions) and retry failed segments only.
- **Resumable upload:** Multipart upload with part checksums. The client asks which parts are done and resumes.
- **CDN cost:** Popular content is served from the CDN, while the long tail is served from origin or regional caches (Netflix Open Connect places boxes inside ISPs).
- **View counts:** Batch increments through Kafka, then aggregate. Don't run `UPDATE ... +1` per view.

---

## 31. File Sync (Dropbox / Google Drive)

References: Alex Xu Vol. 1 Ch. 15; [Hello Interview: Dropbox](https://www.hellointerview.com/learn/system-design/problem-breakdowns/dropbox).

**Functional:** Upload and download files. Sync automatically across devices. Share with others.
**Non-functional:** Files up to 50 GB. Upload and sync quickly. Strong durability (never lose data). Consistency over availability for metadata.

```text
Client (watcher + chunker)
   │ 1. split file into 4 MB chunks, SHA-256 each (content-defined chunking for better dedupe)
   │ 2. POST /files/upload-intent {path, chunkHashes[]} → server says which chunks are missing
   │ 3. upload only missing chunks directly to S3 (presigned URLs)      ← dedupe + delta sync
   │ 4. POST /files/commit → metadata DB: file → version → ordered chunk list
   ▼
Metadata DB (Postgres, sharded by owner/namespace) ──▶ change log ──▶ Notification svc
Other devices ◀── long-poll / WebSocket "namespace changed, cursor=X" ── then GET /changes?cursor=X
```

### Deep dives

- **Chunking** enables resumable uploads, parallelism, and **delta sync**: editing 1 byte re-uploads one 4 MB chunk, not 50 GB. Chunks are **content-addressed**, so identical chunks across users are stored once.
- **Conflict handling:** Each file has a version. Commits use optimistic concurrency (`base_version`). On conflict, keep both and create "file (conflicted copy)".
- **Sync notifications:** Clients long-poll a notification service per namespace, then pull the change list with a cursor.
- **Security:** Signed URLs, encryption at rest, and ACLs in metadata.

---

## 32. Ride Sharing (Uber)

References: Alex Xu Vol. 2 Ch. 1 (Proximity) & Ch. 3 (Google Maps); [Hello Interview: Uber](https://www.hellointerview.com/learn/system-design/problem-breakdowns/uber).

**Functional:** (1) Rider requests a fare estimate. (2) Rider requests a ride and is matched with a nearby driver. (3) Driver accepts or declines. (4) Both see live location.
**Non-functional:** Matching < 1 minute. **A driver is never assigned two rides at once** (strong consistency). Drivers send location every ~5 s. Scale: ~10M drivers online peak → about 2M location updates/s.

```text
Driver app ──location every 5 s──▶ Location svc ──▶ Redis GEO (GEOADD drivers lng lat id) + TTL heartbeat
Rider ──POST /rides──▶ Ride svc ──▶ Matching svc:
                                     1. GEOSEARCH drivers within 3 km, ASC, COUNT 10
                                     2. for each candidate (by ETA): acquire lock driver:{id} (SET NX PX 10000)
                                     3. push offer to driver (APNs/FCM or WebSocket); wait ≤10 s
                                     4. accept → ride.status=ACCEPTED (DB txn); decline/timeout → release, next
Rides DB (Postgres) : rides(id, rider_id, driver_id, status, pickup, dropoff, fare, ...)
```

```ts
async function matchRide(ride: Ride): Promise<string | null> {
  const candidates = (await redis.geosearch(
    "drivers:available", "FROMLONLAT", ride.pickup.lng, ride.pickup.lat,
    "BYRADIUS", 3, "km", "ASC", "COUNT", 10,
  )) as string[];

  for (const driverId of candidates) {
    const locked = await redis.set(`lock:driver:${driverId}`, ride.id, "PX", 10_000, "NX");
    if (!locked) continue;                           // another ride is offering this driver
    const accepted = await offerToDriver(driverId, ride, { timeoutMs: 10_000 });
    if (accepted) {
      await db.tx(async (tx) => {
        await tx.query(`UPDATE rides SET driver_id=$1, status='ACCEPTED' WHERE id=$2 AND status='REQUESTED'`, [driverId, ride.id]);
      });
      await redis.zrem("drivers:available", driverId);
      return driverId;
    }
    await redis.del(`lock:driver:${driverId}`);
  }
  return null; // expand radius / retry via queue
}
```

### Deep dives

- **Location write load (2M/s):** Redis GEO sharded by region or geohash prefix. Throttle updates adaptively (less often when the driver is stationary). Don't write every ping to the primary DB; stream to Kafka for trip history.
- **No double assignment:** Per-driver lock with TTL, plus a conditional DB update.
- **Matching durability:** Put ride requests in a queue (or a Temporal workflow) so a crashed matcher doesn't lose a request.
- **Surge hotspots (stadium events):** Shard by geography, and add capacity for known events.

---

## 33. Ticket Booking (Ticketmaster)

Reference: [Hello Interview: Ticketmaster](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ticketmaster). The classic contention problem.

**Functional:** (1) View events. (2) Search events. (3) Book tickets: pick seats, reserve them, pay.
**Non-functional:** **No double booking** (strong consistency for booking). High availability for viewing and search. Handle surges (10M users for a Taylor Swift on-sale). Search < 500 ms.

```text
Browse/search (AP):  Client → CDN/cache → Event svc → Postgres (events, venues, seats); Elasticsearch for search
Booking (CP):
  1. POST /bookings/reserve {eventId, seatIds}
       → Redis: SET seat-hold:{event}:{seat} {userId} NX EX 600   (10-min hold, auto-expires)
       → Postgres: tickets.status = RESERVED  (conditional update WHERE status='AVAILABLE')
  2. Client pays (Stripe, with idempotency key)
  3. Stripe webhook → POST /bookings/confirm → tickets.status = BOOKED (txn) → release hold
Surge: Virtual waiting room (queue) admits N users/s to the booking flow; real-time seat map via SSE
```

### Deep dives

- **Reservation hold:** Options are (a) a DB status plus an `expires_at` column with a cleanup cron, (b) a **Redis TTL lock** where expiry is automatic, and (c) a DB lock held across payment (bad, because it's a long transaction). Use (b) with a DB conditional update as the final guard.
- **Virtual waiting queue:** For hot events, users enter a Redis sorted-set queue. The system admits batches and pushes "your turn" over WebSocket/SSE. This protects the booking service.
- **Caching:** Event and venue details are heavily cached. Seat availability is cached briefly or pushed live.
- **Search:** Elasticsearch synced via CDC. Cache popular queries. Use a CDN for static content.

---

## 34. Web Crawler

References: Alex Xu Vol. 1 Ch. 9; [Hello Interview: Web Crawler](https://www.hellointerview.com/learn/system-design/problem-breakdowns/web-crawler).

**Functional:** Crawl the web starting from seed URLs, extract text, and store it (e.g. to train an LLM or build a search index).
**Non-functional:** 10B pages in ~5 days ≈ 23k pages/s. **Politeness** (respect robots.txt, max 1 request/s per domain). Fault-tolerant. No duplicate crawling.

```text
Seeds ─▶ URL Frontier (Kafka/SQS: priority queues + per-domain politeness queues)
             │
             ▼
     Fetcher workers ──DNS cache──▶ internet
             │ robots.txt cache, per-domain rate limiter (Redis), timeouts, retries w/ backoff
             ▼
     Raw HTML ─▶ S3 ──event──▶ Parser workers ─▶ extracted text → S3
                                     │
                                     └─ extract links ─▶ normalize ─▶ URL seen? (Bloom filter + DB)
                                                           ─▶ content seen? (hash / SimHash) ─▶ back to Frontier
```

**Deep dives:** Separate fetching (I/O-bound) from parsing (CPU-bound) so each scales independently. Bloom filter for seen URLs (Section 21). Politeness through per-domain queues. Avoid crawler traps (max depth, URL length limits). Use checkpoints and queue visibility timeouts for fault tolerance. 23k pages/s × 2 MB ≈ 46 GB/s across a fleet → hundreds of fetcher machines.

---

## 35. Top-K / Leaderboard / Ad Click Aggregator

References: Alex Xu Vol. 2 Ch. 6 (Ad Click), Ch. 10 (Leaderboard); Hello Interview [Top-K](https://www.hellointerview.com/learn/system-design/problem-breakdowns/top-k) and [Ad Click Aggregator](https://www.hellointerview.com/learn/system-design/problem-breakdowns/ad-click-aggregator).

### Real-time leaderboard (gaming)

Redis **sorted sets** (skip list) give O(log n) updates and rank queries.

```ts
await redis.zincrby("lb:2026-10", points, userId);                 // score update
const top10 = await redis.zrevrange("lb:2026-10", 0, 9, "WITHSCORES");
const myRank = await redis.zrevrank("lb:2026-10", userId);          // 0-based
const around = await redis.zrevrange("lb:2026-10", Math.max(0, myRank! - 5), myRank! + 5, "WITHSCORES");
```

At 100M+ users, shard by score range, or accept approximate rank for the long tail.

### Top-K heavy hitters (e.g. most-viewed videos in the last hour/day)

```text
View events ─▶ Kafka (partitioned by videoId) ─▶ Flink: tumbling/sliding windows,
     per-partition counts → local min-heap top-K → merge partial top-Ks → Redis / OLAP store
Batch path (Lambda architecture): Spark over S3 every hour for exact counts / reconciliation
```

```ts
// Top-K from counts using a min-heap of size K: O(n log K)
function topK(counts: Map<string, number>, k: number): [string, number][] {
  const heap: [string, number][] = [];                 // min-heap by count
  const up = (i: number) => { while (i && heap[(i - 1) >> 1][1] > heap[i][1]) { [heap[i], heap[(i - 1) >> 1]] = [heap[(i - 1) >> 1], heap[i]]; i = (i - 1) >> 1; } };
  const down = (i: number) => {
    for (;;) {
      let m = i; const l = 2 * i + 1, r = l + 1;
      if (l < heap.length && heap[l][1] < heap[m][1]) m = l;
      if (r < heap.length && heap[r][1] < heap[m][1]) m = r;
      if (m === i) return;
      [heap[i], heap[m]] = [heap[m], heap[i]]; i = m;
    }
  };
  for (const e of counts) {
    if (heap.length < k) { heap.push(e); up(heap.length - 1); }
    else if (e[1] > heap[0][1]) { heap[0] = e; down(0); }
  }
  return heap.sort((a, b) => b[1] - a[1]);
}
```

### Ad click aggregator key points

- Clicks are redirected through your server (log the click, then 302 to the advertiser).
- Kafka → Flink computes per-minute aggregates by ad_id → OLAP store (Druid/ClickHouse/Pinot) for advertiser queries.
- **Exactly-once-ish counting:** Dedupe clicks by impression ID, use Flink checkpoints, and reconcile with a batch job over raw events in S3.
- Hot ad IDs: salt the partition key, then merge.

---

## 36. Notification System

References: Alex Xu Vol. 1 Ch. 10.

**Functional:** Send push (iOS APNs / Android FCM), SMS (Twilio), and email (SES/SendGrid). Support user preferences and opt-outs. Support scheduled and bulk sends.
**Non-functional:** Soft real-time. **No lost notifications** (at-least-once, deduped). 10M+ notifications/day with bursts.

```text
Services ─▶ Notification API (auth, validate, rate limit per user, idempotency key)
                │ check preferences / opt-out (cache) ; render template
                ▼
            Kafka topics per channel: push | sms | email
                ▼
            Channel workers ──▶ APNs / FCM / Twilio / SES   (retry w/ backoff, DLQ, circuit breaker per provider)
                │
                └─▶ notification_log (status: queued/sent/delivered/failed) + analytics events
Scheduler: delayed jobs table polled by time / Redis ZSET (score = sendAt)
```

**Deep dives:** Separate queues per channel isolate a slow provider. Dedupe on `(user, eventId)`. Per-user frequency caps. Device-token management (remove invalid tokens on provider error). Priority lanes keep OTP traffic separate from marketing.

---

## 37. Search Autocomplete (Typeahead)

References: Alex Xu Vol. 1 Ch. 13.

**Functional:** As the user types, return the top 5 suggestions for the prefix, ranked by popularity.
**Non-functional:** < 100 ms. 10M DAU × 10 searches × ~20 keystrokes ≈ 24k QPS. Popularity updates can lag by hours.

**Data structure:** A **trie** where each node caches its top-K completions, so a query is O(prefix length) instead of a subtree walk.

```ts
class TrieNode {
  children = new Map<string, TrieNode>();
  count = 0;                                  // frequency if a word ends here
  top: { word: string; count: number }[] = [];// cached top-K for this prefix
}

class Autocomplete {
  private root = new TrieNode();
  constructor(private k = 5) {}

  insert(word: string, count: number) {
    let node = this.root;
    const path: TrieNode[] = [node];
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
      node = node.children.get(ch)!;
      path.push(node);
    }
    node.count = count;
    for (const n of path) this.updateTop(n, word, count);   // refresh cached top-K along the path
  }

  private updateTop(n: TrieNode, word: string, count: number) {
    n.top = n.top.filter((e) => e.word !== word);
    n.top.push({ word, count });
    n.top.sort((a, b) => b.count - a.count);
    if (n.top.length > this.k) n.top.length = this.k;
  }

  suggest(prefix: string): string[] {
    let node = this.root;
    for (const ch of prefix.toLowerCase()) {
      const next = node.children.get(ch);
      if (!next) return [];
      node = next;
    }
    return node.top.map((e) => e.word);
  }
}
```

```text
Query logs ─▶ Kafka ─▶ weekly/hourly aggregation (Spark) ─▶ build trie offline ─▶ serialize to S3
Trie servers load snapshot (sharded by prefix range: a–m, n–z…) ; Redis cache prefix → top5
Client: debounce 100–200 ms, browser cache responses, CDN for short prefixes
```

---

## 38. Distributed Key-Value Store

References: Alex Xu Vol. 1 Ch. 6; the Amazon Dynamo paper. This problem ties many building blocks together.

| Requirement | Technique |
| --- | --- |
| Partition data | Consistent hashing with virtual nodes (Section 10) |
| Replicate | N replicas on the next N distinct nodes clockwise |
| Tunable consistency | Quorum W + R > N (Section 8) |
| Concurrent writes | Vector clocks (Section 23) or last-write-wins |
| Temporary failure | **Sloppy quorum + hinted handoff:** another node accepts the write and hands it back later |
| Permanent failure / divergence | **Anti-entropy with Merkle trees:** compare hash trees and sync only the differing ranges |
| Failure detection | Gossip protocol (heartbeat counters) |
| Write path | Commit log → memtable → SSTable (LSM, Section 7) |
| Read path | Memtable → bloom filter → SSTables. Read repair on mismatched replicas. |

```text
Client ─▶ any node (coordinator) ─▶ hash(key) → replicas [A, B, C]
   write: send to all 3, succeed after W=2 ACKs
   read : ask all 3, return after R=2 responses, newest version wins (fix stale replica = read repair)
```

---
# Part IV: Low-Level Design (LLD)

## 39. The LLD Interview Framework

References: [Hello Interview: LLD in a Hurry](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/introduction), [awesome-low-level-design](https://github.com/ashishps1/awesome-low-level-design), *Head First Design Patterns*.

LLD rounds come in two forms. In **OOD whiteboard** rounds you draw classes and write key methods. In **machine coding** rounds (common in India: Flipkart, Swiggy, Uber, Razorpay) you write runnable, modular code in 60–90 minutes.

```text
1. Clarify requirements (5 min)   – features in/out, actors, constraints, scale (single process? concurrent?)
2. Identify entities (5 min)      – nouns → classes; verbs → methods; note states (enums)
3. Relationships (5 min)          – is-a vs has-a; multiplicities; draw a quick class diagram
4. Define interfaces/APIs (5 min) – public methods of the main service/facade
5. Code core flows (25-40 min)    – happy path first; then edge cases & validations
6. Patterns & extensibility (5 m) – "to add X we'd just add a new Strategy class"
7. Concurrency & testing (5 min)  – what needs locking? how would you unit test?
```

### What interviewers grade

| Signal | Looks like |
| --- | --- |
| Modeling | Correct entities, no god class, enums for fixed states |
| SOLID | Small classes with single responsibilities. Depend on interfaces. |
| Extensibility | New vehicle type, pricing rule, or payment method needs **no edits** to existing classes |
| Patterns | Used where they fit and named. Not forced in. |
| Code quality | Clear names, validation, custom exceptions, no magic numbers |
| Concurrency | Identifies shared mutable state and protects it |
| Testability | Dependencies injected, so clocks and IDs can be mocked |

### Layering for machine-coding rounds

```text
src/
  models/        # entities: data + invariants (Vehicle, Spot, Ticket)
  enums/         # VehicleType, SpotStatus
  strategies/    # pluggable algorithms (pricing, spot assignment)
  repositories/  # in-memory storage behind interfaces (swap for DB later)
  services/      # business flows (ParkingService) – orchestration
  exceptions/    # domain errors
  main.ts        # demo driver / CLI
```

---

## 40. OOP Fundamentals and UML Relationships

### The four pillars

| Pillar | Meaning | Example |
| --- | --- | --- |
| Encapsulation | Hide state and expose behavior. Protect invariants. | `account.withdraw(x)` validates instead of exposing `balance` as public |
| Abstraction | Expose *what*, hide *how* | `PaymentGateway.pay()` interface |
| Inheritance | "is-a" reuse | `Car extends Vehicle` (use sparingly) |
| Polymorphism | One interface, many implementations | `shape.area()` on Circle/Square |

```ts
class BankAccount {
  #balance = 0;                              // private: only changed through methods
  constructor(readonly id: string) {}

  deposit(amount: number) {
    if (amount <= 0) throw new Error("amount must be positive");
    this.#balance += amount;
  }
  withdraw(amount: number) {
    if (amount > this.#balance) throw new InsufficientFundsError(this.id, amount);
    this.#balance -= amount;
  }
  get balance() { return this.#balance; }
}
class InsufficientFundsError extends Error {
  constructor(id: string, amt: number) { super(`account ${id}: insufficient funds for ${amt}`); }
}
```

### UML class relationships

| Relationship | Meaning | UML | Code |
| --- | --- | --- | --- |
| Association | Uses or knows about | `A ──> B` | Field reference |
| Aggregation | Has-a, **independent** lifecycle | `A ◇── B` | Team has Players (players exist without the team) |
| Composition | Has-a, **owned** lifecycle | `A ◆── B` | House has Rooms (rooms die with the house) |
| Inheritance | Is-a | `B ──▷ A` | `extends` |
| Realization | Implements | `B ┄┄▷ I` | `implements` |
| Dependency | Temporarily uses | `A ┄┄> B` | Parameter or local variable |

```text
           «interface»
          PaymentMethod
          +pay(amt): Receipt
              △
      ┌───────┴────────┐
  CardPayment      UpiPayment
                                 Order ◆──── 1..* OrderItem
  Customer 1 ───── * Order       Order ────> PaymentMethod
```

---

## 41. SOLID Principles

### S: Single Responsibility Principle

A class should have one reason to change.

```ts
// ❌ Report knows data, formatting AND delivery
class Report {
  generate() { /* query */ }
  toPdf() { /* format */ }
  emailTo(addr: string) { /* SMTP */ }
}

// ✅ separate reasons to change
class ReportGenerator { generate(): ReportData { /* ... */ return {} as ReportData; } }
class PdfFormatter   { format(d: ReportData): Buffer { /* ... */ return Buffer.from(""); } }
class EmailSender    { send(to: string, attachment: Buffer) { /* ... */ } }
```

### O: Open/Closed Principle

Open for extension, closed for modification. Add behavior by adding classes, not by editing `if/else` chains.

```ts
// ❌ every new discount = edit this function
function price(type: string, amount: number) {
  if (type === "festive") return amount * 0.8;
  if (type === "member") return amount * 0.9;
  return amount;
}

// ✅ new rule = new class
interface DiscountRule { apply(amount: number): number }
class FestiveDiscount implements DiscountRule { apply(a: number) { return a * 0.8; } }
class MemberDiscount implements DiscountRule { apply(a: number) { return a * 0.9; } }
class Checkout {
  constructor(private rules: DiscountRule[]) {}
  total(amount: number) { return this.rules.reduce((a, r) => r.apply(a), amount); }
}
```

### L: Liskov Substitution Principle

Subtypes must be usable anywhere the base type is used without breaking expectations.

```ts
// ❌ Classic violation: Square changes behavior of Rectangle's setters
class Rectangle { constructor(public w: number, public h: number) {} setW(w: number) { this.w = w; } area() { return this.w * this.h; } }
class Square extends Rectangle { setW(w: number) { this.w = w; this.h = w; } } // surprises callers

// ❌ Penguin extends Bird with fly() that throws
// ✅ Model capabilities separately
interface Bird { eat(): void }
interface Flyable { fly(): void }
class Sparrow implements Bird, Flyable { eat() {} fly() {} }
class Penguin implements Bird { eat() {} }
```

### I: Interface Segregation Principle

Clients shouldn't depend on methods they don't use. Prefer small role interfaces.

```ts
// ❌ fat interface
interface Machine { print(): void; scan(): void; fax(): void }
// ✅
interface Printer { print(doc: string): void }
interface Scanner { scan(): string }
class SimplePrinter implements Printer { print(doc: string) {} }
class MultiFunctionDevice implements Printer, Scanner { print(doc: string) {} scan() { return ""; } }
```

### D: Dependency Inversion Principle

High-level modules depend on abstractions, not concrete classes. Inject dependencies so they can be swapped and tested.

```ts
interface NotificationChannel { send(userId: string, msg: string): Promise<void> }
class SmsChannel implements NotificationChannel { async send(u: string, m: string) { /* Twilio */ } }
class FakeChannel implements NotificationChannel { sent: string[] = []; async send(_: string, m: string) { this.sent.push(m); } }

class OrderService {
  constructor(private notifier: NotificationChannel) {}   // depends on abstraction
  async placeOrder(userId: string) { /* ... */ await this.notifier.send(userId, "Order placed"); }
}
// prod: new OrderService(new SmsChannel())   test: new OrderService(new FakeChannel())
```

---

## 42. Other Design Principles

| Principle | Meaning |
| --- | --- |
| **Composition over inheritance** | Build behavior by combining objects (`has-a`), not deep class trees |
| DRY | Don't Repeat Yourself: one source of truth for each piece of knowledge |
| KISS | Keep It Simple |
| YAGNI | You Aren't Gonna Need It: don't build speculative features |
| Law of Demeter | Talk to direct friends only (`a.getB().getC().do()` is a smell) |
| Tell, don't ask | `account.withdraw(x)` instead of `if (account.balance > x) account.balance -= x` |
| Program to an interface | Declare variables by interface type |
| Fail fast | Validate inputs at boundaries and throw domain-specific errors |

```ts
// Composition over inheritance: behaviours are pluggable objects
interface MoveBehavior { move(): string }
class Walk implements MoveBehavior { move() { return "walks"; } }
class Fly implements MoveBehavior { move() { return "flies"; } }

class Character {
  constructor(public name: string, private mover: MoveBehavior) {}
  setMover(m: MoveBehavior) { this.mover = m; }       // change at runtime – impossible with inheritance
  act() { return `${this.name} ${this.mover.move()}`; }
}
```

---

## 43. Creational Patterns

### Singleton

One instance with global access. Typical uses: config, connection pool, logger. **Caveats:** it's hidden global state that makes testing harder. Prefer dependency injection with a single instance.

```ts
class ConfigManager {
  private static instance: ConfigManager | null = null;
  private constructor(private settings: Record<string, string>) {}
  static getInstance(): ConfigManager {
    if (!this.instance) this.instance = new ConfigManager(loadFromEnv());
    return this.instance;
  }
  get(key: string) { return this.settings[key]; }
}
```

The thread-safe Java version (double-checked locking, enum, or holder idiom) is in [Section 46](#46-concurrency-for-lld-java).

### Factory Method / Simple Factory

Centralize object creation so callers don't depend on concrete classes.

```ts
interface Vehicle { readonly type: VehicleType; readonly plate: string }
enum VehicleType { BIKE = "BIKE", CAR = "CAR", TRUCK = "TRUCK" }
class Bike implements Vehicle { readonly type = VehicleType.BIKE; constructor(readonly plate: string) {} }
class Car implements Vehicle { readonly type = VehicleType.CAR; constructor(readonly plate: string) {} }
class Truck implements Vehicle { readonly type = VehicleType.TRUCK; constructor(readonly plate: string) {} }

class VehicleFactory {
  private static registry: Record<VehicleType, new (plate: string) => Vehicle> = {
    [VehicleType.BIKE]: Bike, [VehicleType.CAR]: Car, [VehicleType.TRUCK]: Truck,
  };
  static create(type: VehicleType, plate: string): Vehicle {
    const Ctor = this.registry[type];
    if (!Ctor) throw new Error(`unknown vehicle type ${type}`);
    return new Ctor(plate);
  }
}
```

### Abstract Factory

Create **families** of related objects (e.g. a UI theme, or cloud provider resources) without naming concrete classes.

```ts
interface Button { render(): string } interface Checkbox { render(): string }
interface UIFactory { button(): Button; checkbox(): Checkbox }

class DarkFactory implements UIFactory {
  button() { return { render: () => "<dark-button>" }; }
  checkbox() { return { render: () => "<dark-checkbox>" }; }
}
class LightFactory implements UIFactory {
  button() { return { render: () => "<light-button>" }; }
  checkbox() { return { render: () => "<light-checkbox>" }; }
}
function renderForm(f: UIFactory) { return f.button().render() + f.checkbox().render(); }
```

### Builder

Construct complex objects step by step, especially those with many optional parameters. Avoids "telescoping constructors."

```ts
class HttpRequest {
  constructor(
    readonly method: string, readonly url: string, readonly headers: Record<string, string>,
    readonly body?: string, readonly timeoutMs = 5000,
  ) {}
  static builder(url: string) { return new HttpRequestBuilder(url); }
}

class HttpRequestBuilder {
  private method = "GET";
  private headers: Record<string, string> = {};
  private body?: string;
  private timeoutMs = 5000;
  constructor(private url: string) {}
  withMethod(m: string) { this.method = m; return this; }
  header(k: string, v: string) { this.headers[k] = v; return this; }
  json(obj: unknown) { this.body = JSON.stringify(obj); return this.header("Content-Type", "application/json"); }
  timeout(ms: number) { this.timeoutMs = ms; return this; }
  build() {
    if (this.method === "GET" && this.body) throw new Error("GET cannot have a body");
    return new HttpRequest(this.method, this.url, { ...this.headers }, this.body, this.timeoutMs);
  }
}

const req = HttpRequest.builder("https://api.x.com/orders").withMethod("POST").json({ id: 1 }).timeout(2000).build();
```

### Prototype

Clone existing objects instead of building them from scratch. Useful when construction is expensive, or for templates.

```ts
interface Prototype<T> { clone(): T }
class Document implements Prototype<Document> {
  constructor(public title: string, public sections: string[], public meta: { author: string }) {}
  clone() { return new Document(this.title, [...this.sections], { ...this.meta }); } // deep enough copy
}
const template = new Document("Invoice", ["header", "items", "footer"], { author: "system" });
const inv = template.clone(); inv.title = "Invoice #42";
```

---

## 44. Structural Patterns

### Adapter

Make an incompatible interface fit the one your code expects. Common for third-party SDKs.

```ts
interface PaymentGateway { pay(amountPaise: number, ref: string): Promise<string> }

// third-party SDK with a different shape
class StripeSdk { async createCharge(opts: { amount_cents: number; currency: string; metadata: object }) { return { id: "ch_1" }; } }

class StripeAdapter implements PaymentGateway {
  constructor(private stripe: StripeSdk) {}
  async pay(amountPaise: number, ref: string) {
    const r = await this.stripe.createCharge({ amount_cents: amountPaise, currency: "inr", metadata: { ref } });
    return r.id;
  }
}
```

### Decorator

Add responsibilities dynamically by wrapping an object that has the **same interface**. Classic examples: coffee add-ons, and adding logging, caching, or retries to a service.

```ts
interface DataSource { read(key: string): Promise<string | null> }
class DbSource implements DataSource { async read(key: string) { return `db:${key}`; } }

class CachingSource implements DataSource {
  private cache = new Map<string, string | null>();
  constructor(private inner: DataSource) {}
  async read(key: string) {
    if (!this.cache.has(key)) this.cache.set(key, await this.inner.read(key));
    return this.cache.get(key)!;
  }
}
class LoggingSource implements DataSource {
  constructor(private inner: DataSource) {}
  async read(key: string) { const t = Date.now(); const v = await this.inner.read(key); console.log(`read ${key} ${Date.now() - t}ms`); return v; }
}
const source: DataSource = new LoggingSource(new CachingSource(new DbSource()));  // stack decorators
```

```ts
// Coffee example (often asked)
interface Beverage { cost(): number; description(): string }
class Espresso implements Beverage { cost() { return 100; } description() { return "Espresso"; } }
abstract class AddOn implements Beverage {
  constructor(protected base: Beverage) {}
  abstract cost(): number; abstract description(): string;
}
class Milk extends AddOn { cost() { return this.base.cost() + 20; } description() { return this.base.description() + ", milk"; } }
class Caramel extends AddOn { cost() { return this.base.cost() + 30; } description() { return this.base.description() + ", caramel"; } }
new Caramel(new Milk(new Espresso())).cost(); // 150
```

### Facade

Give callers a simple interface over a complex subsystem.

```ts
class OrderFacade {
  constructor(private inventory: InventoryService, private payment: PaymentService, private shipping: ShippingService, private notify: NotificationService) {}
  async placeOrder(userId: string, items: Item[], card: Card) {
    await this.inventory.reserve(items);
    const paymentId = await this.payment.charge(card, total(items));
    const tracking = await this.shipping.schedule(userId, items);
    await this.notify.orderConfirmed(userId, tracking);
    return { paymentId, tracking };
  }
}
```

### Proxy

A stand-in object that controls access: lazy loading (virtual proxy), access control (protection proxy), remote calls, or caching.

```ts
interface Image { display(): void }
class RealImage implements Image {
  constructor(private file: string) { console.log(`loading ${file} from disk`); } // expensive
  display() { console.log(`showing ${this.file}`); }
}
class LazyImageProxy implements Image {
  private real?: RealImage;
  constructor(private file: string) {}
  display() { (this.real ??= new RealImage(this.file)).display(); }  // load on first use
}
```

### Composite

Treat individual objects and groups the same way (tree structures: file systems, org charts, UI trees).

```ts
interface FsNode { name: string; size(): number; print(indent?: string): void }
class File implements FsNode {
  constructor(public name: string, private bytes: number) {}
  size() { return this.bytes; }
  print(i = "") { console.log(`${i}${this.name} (${this.bytes})`); }
}
class Directory implements FsNode {
  private children: FsNode[] = [];
  constructor(public name: string) {}
  add(n: FsNode) { this.children.push(n); return this; }
  size(): number { return this.children.reduce((s, c) => s + c.size(), 0); }
  print(i = "") { console.log(`${i}${this.name}/`); this.children.forEach((c) => c.print(i + "  ")); }
}
```

### Bridge and Flyweight (know the definitions)

- **Bridge:** Separate an abstraction from its implementation so both can vary independently. For example, `Shape` × `Renderer` (vector or raster) instead of a class for every combination.
- **Flyweight:** Share intrinsic state among many fine-grained objects to save memory. For example, a text editor shares one glyph object per character and font, and stores position separately.

---

## 45. Behavioral Patterns

### Strategy (the pattern used most in LLD)

Swap algorithms at runtime behind an interface. Use it for pricing, spot assignment, payment methods, sorting, and splitting expenses.

```ts
interface PricingStrategy { calculate(hours: number): number }
class HourlyPricing implements PricingStrategy { constructor(private rate: number) {} calculate(h: number) { return Math.ceil(h) * this.rate; } }
class FlatPlusHourly implements PricingStrategy {
  calculate(h: number) { return h <= 2 ? 50 : 50 + Math.ceil(h - 2) * 30; }
}
class WeekendPricing implements PricingStrategy {
  constructor(private base: PricingStrategy, private multiplier = 1.5) {}
  calculate(h: number) { return this.base.calculate(h) * this.multiplier; }
}
```

### Observer

A subject notifies its subscribers when its state changes. Use it for event systems, stock tickers, notifications, and UI updates.

```ts
type Listener<T> = (event: T) => void;

class EventBus<Events extends Record<string, unknown>> {
  private listeners = new Map<keyof Events, Set<Listener<any>>>();
  on<K extends keyof Events>(type: K, fn: Listener<Events[K]>) {
    if (!this.listeners.has(type)) this.listeners.set(type, new Set());
    this.listeners.get(type)!.add(fn);
    return () => this.listeners.get(type)!.delete(fn);         // unsubscribe
  }
  emit<K extends keyof Events>(type: K, event: Events[K]) {
    this.listeners.get(type)?.forEach((fn) => {
      try { fn(event); } catch (e) { console.error("listener failed", e); } // one bad listener ≠ outage
    });
  }
}

const bus = new EventBus<{ orderPlaced: { orderId: string }; orderShipped: { orderId: string; tracking: string } }>();
bus.on("orderPlaced", (e) => console.log("email for", e.orderId));
bus.emit("orderPlaced", { orderId: "o-1" });
```

### State

An object changes its behavior when its internal state changes. Each state is a class. Use it for vending machines, orders, elevators, documents, and traffic lights. It replaces large `switch(state)` blocks.

```ts
interface OrderState {
  pay(o: Order): void; ship(o: Order): void; cancel(o: Order): void; readonly name: string;
}
class Order { state: OrderState = new Created(); pay() { this.state.pay(this); } ship() { this.state.ship(this); } cancel() { this.state.cancel(this); } }

class Created implements OrderState {
  name = "CREATED";
  pay(o: Order) { o.state = new Paid(); }
  ship() { throw new Error("cannot ship unpaid order"); }
  cancel(o: Order) { o.state = new Cancelled(); }
}
class Paid implements OrderState {
  name = "PAID";
  pay() { throw new Error("already paid"); }
  ship(o: Order) { o.state = new Shipped(); }
  cancel(o: Order) { /* refund */ o.state = new Cancelled(); }
}
class Shipped implements OrderState { name = "SHIPPED"; pay() { throw new Error("paid"); } ship() { throw new Error("shipped"); } cancel() { throw new Error("cannot cancel shipped order"); } }
class Cancelled implements OrderState { name = "CANCELLED"; pay() { throw new Error("cancelled"); } ship() { throw new Error("cancelled"); } cancel() {} }
```

### Command

Wrap a request as an object. This enables undo/redo, queues, macros, and transaction logs.

```ts
interface Command { execute(): void; undo(): void }

class TextEditor { text = ""; }
class InsertCommand implements Command {
  constructor(private ed: TextEditor, private pos: number, private s: string) {}
  execute() { this.ed.text = this.ed.text.slice(0, this.pos) + this.s + this.ed.text.slice(this.pos); }
  undo() { this.ed.text = this.ed.text.slice(0, this.pos) + this.ed.text.slice(this.pos + this.s.length); }
}

class History {
  private undoStack: Command[] = []; private redoStack: Command[] = [];
  run(c: Command) { c.execute(); this.undoStack.push(c); this.redoStack = []; }
  undo() { const c = this.undoStack.pop(); if (c) { c.undo(); this.redoStack.push(c); } }
  redo() { const c = this.redoStack.pop(); if (c) { c.execute(); this.undoStack.push(c); } }
}
```

### Chain of Responsibility

Pass a request along a chain of handlers until one handles it, or each handler processes and forwards it. Use it for middleware, logging levels, approval workflows, ATM cash dispensing, and request validation.

```ts
abstract class Handler<T> {
  private next?: Handler<T>;
  setNext(h: Handler<T>) { this.next = h; return h; }          // fluent chain building
  handle(req: T): string | null { return this.next ? this.next.handle(req) : null; }
}

type Expense = { amount: number };
class Manager extends Handler<Expense> { handle(r: Expense) { return r.amount <= 1_000 ? "Manager approved" : super.handle(r); } }
class Director extends Handler<Expense> { handle(r: Expense) { return r.amount <= 10_000 ? "Director approved" : super.handle(r); } }
class CFO extends Handler<Expense> { handle(r: Expense) { return r.amount <= 100_000 ? "CFO approved" : super.handle(r); } }

const chain = new Manager(); chain.setNext(new Director()).setNext(new CFO());
chain.handle({ amount: 5_000 }); // "Director approved"
```

### Template Method

A base class defines an algorithm's skeleton, and subclasses fill in specific steps.

```ts
abstract class DataExporter {
  export(): string {                         // the template — not overridden
    const data = this.fetch();
    const valid = data.filter((r) => this.validate(r));
    return this.format(valid);
  }
  protected abstract fetch(): Row[];
  protected validate(_r: Row) { return true; }  // hook with default
  protected abstract format(rows: Row[]): string;
}
class CsvExporter extends DataExporter {
  protected fetch() { return db.rows(); }
  protected format(rows: Row[]) { return rows.map((r) => Object.values(r).join(",")).join("\n"); }
}
```

### Iterator, Mediator, Memento, Visitor (know the definitions)

| Pattern | One-liner | Example |
| --- | --- | --- |
| Iterator | Traverse a collection without exposing its internals | `[Symbol.iterator]()`, paginated API iterator |
| Mediator | Objects talk through a central mediator, not each other | Chat room, air-traffic control, elevator dispatcher |
| Memento | Capture and restore state without breaking encapsulation | Undo snapshots, game saves |
| Visitor | Add operations to an object structure without changing its classes | AST walkers, tax calculation over a cart |

### Pattern selection cheat sheet

| If the problem says... | Reach for |
| --- | --- |
| "Different algorithms / pricing / strategies" | Strategy |
| "Notify when X changes" | Observer |
| "Behaves differently depending on its status" | State |
| "Undo / redo / queue operations" | Command (+ Memento) |
| "Series of handlers / approvals / middleware" | Chain of Responsibility |
| "Create objects by type" | Factory |
| "Many optional config fields" | Builder |
| "Add features without subclass explosion" | Decorator |
| "Integrate a third-party API" | Adapter |
| "Simplify a complex subsystem" | Facade |
| "Tree / part-whole hierarchy" | Composite |
| "Exactly one shared instance" | Singleton (or DI scope) |

---

## 46. Concurrency for LLD (Java)

LLD interviewers often ask: "What if two users book the same seat at the same time?" Know these tools.

### Thread-safe Singleton

```java
// 1. Double-checked locking (volatile is required!)
public final class Registry {
    private static volatile Registry instance;
    private Registry() {}
    public static Registry getInstance() {
        if (instance == null) {
            synchronized (Registry.class) {
                if (instance == null) instance = new Registry();
            }
        }
        return instance;
    }
}

// 2. Bill Pugh holder idiom – lazy, thread-safe, no locking
public final class Config {
    private Config() {}
    private static class Holder { static final Config INSTANCE = new Config(); }
    public static Config getInstance() { return Holder.INSTANCE; }
}

// 3. Enum singleton – serialization & reflection safe (Effective Java)
public enum Logger { INSTANCE; public void log(String m) { System.out.println(m); } }
```

### Locks and atomic types

```java
class SeatInventory {
    private final Map<String, String> seatToUser = new ConcurrentHashMap<>();

    // putIfAbsent is atomic → exactly one thread wins the seat
    public boolean book(String seatId, String userId) {
        return seatToUser.putIfAbsent(seatId, userId) == null;
    }
}

class Account {
    private final ReentrantLock lock = new ReentrantLock();
    private long balance;

    public void withdraw(long amt) {
        lock.lock();
        try {
            if (amt > balance) throw new IllegalStateException("insufficient funds");
            balance -= amt;
        } finally {
            lock.unlock();
        }
    }
}

// Transfer between accounts (assumes Account has a String id): lock in a GLOBAL ORDER to avoid deadlock
static void transfer(Account a, Account b, long amt) {
    Account first = a.id.compareTo(b.id) < 0 ? a : b;
    Account second = first == a ? b : a;
    synchronized (first) { synchronized (second) { a.withdraw(amt); b.deposit(amt); } }
}

AtomicLong counter = new AtomicLong();          // lock-free CAS increments
counter.incrementAndGet();
```

### Producer–consumer with BlockingQueue

```java
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(1000);     // bounded = backpressure

Runnable producer = () -> { while (running) queue.put(nextTask()); };   // blocks when full
Runnable consumer = () -> { while (running) process(queue.take()); };   // blocks when empty

ExecutorService pool = Executors.newFixedThreadPool(8);
for (int i = 0; i < 4; i++) pool.submit(consumer);
pool.submit(producer);
```

### Read-write lock (many readers, rare writers)

```java
class ThreadSafeCache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReadWriteLock rw = new ReentrantReadWriteLock();
    public V get(K k) { rw.readLock().lock(); try { return map.get(k); } finally { rw.readLock().unlock(); } }
    public void put(K k, V v) { rw.writeLock().lock(); try { map.put(k, v); } finally { rw.writeLock().unlock(); } }
}
```

### Concurrency vocabulary

| Term | Meaning |
| --- | --- |
| Race condition | Outcome depends on thread timing (check-then-act, read-modify-write) |
| Deadlock | Circular wait. The four Coffman conditions: mutual exclusion, hold-and-wait, no preemption, circular wait. Fix by ordering locks or using `tryLock` with a timeout. |
| Livelock / starvation | Threads keep retrying with no progress, or one thread never gets the lock (use fair locks) |
| Visibility | Without `volatile` or synchronization, writes may not be seen by other threads |
| CAS | Compare-and-swap. Lock-free atomic update (`AtomicInteger`, `ConcurrentHashMap`). |
| Semaphore | Allows N concurrent holders (connection pools, rate limiting) |
| CountDownLatch / CyclicBarrier | Wait for N events, or N threads to reach a point |
| Virtual threads (Java 21+) | Cheap threads for blocking I/O. Thread-per-request at scale. |

**In TypeScript/Node:** There's a single event loop, so in-memory check-then-act is safe *within one synchronous block*. However, an `await` between the check and the act lets other requests interleave. Across multiple processes, rely on DB constraints, conditional updates, or distributed locks (Section 17).

---
# Part V: LLD Case Studies

Each case gives requirements, entities, patterns used, and core code. Practice by writing it yourself first, then compare.

## 47. Parking Lot

**Requirements:** Multiple floors. Spot sizes (small, medium, large). Vehicles (bike, car, truck) park in a compatible spot. Issue a ticket on entry and compute the fee on exit. Show free-spot counts. Pricing and spot-allocation rules should be pluggable. Handle concurrent entry gates.

**Entities:** ParkingLot, Floor, Spot, Vehicle, Ticket, Gate; strategies `SpotAssignmentStrategy` and `PricingStrategy`.
**Patterns:** Strategy (pricing, allocation), Factory (vehicles), Singleton/DI (lot), Observer (display boards).

```text
ParkingLot ◆── Floor ◆── Spot
ParkingLot ──> SpotAssignmentStrategy («interface»)  ◁┄ NearestFirst, FillLowestFloor
ParkingLot ──> PricingStrategy («interface»)          ◁┄ HourlyPricing, FlatPlusHourly
Ticket ──> Vehicle, Spot
```

```ts
enum SpotSize { SMALL = 1, MEDIUM = 2, LARGE = 3 }
enum VehicleType { BIKE = "BIKE", CAR = "CAR", TRUCK = "TRUCK" }
const REQUIRED_SIZE: Record<VehicleType, SpotSize> = {
  [VehicleType.BIKE]: SpotSize.SMALL, [VehicleType.CAR]: SpotSize.MEDIUM, [VehicleType.TRUCK]: SpotSize.LARGE,
};

class Vehicle { constructor(readonly plate: string, readonly type: VehicleType) {} }

class Spot {
  private vehicle: Vehicle | null = null;
  constructor(readonly id: string, readonly floor: number, readonly size: SpotSize) {}
  get isFree() { return this.vehicle === null; }
  canFit(v: Vehicle) { return this.isFree && this.size >= REQUIRED_SIZE[v.type]; }
  park(v: Vehicle) { if (!this.canFit(v)) throw new Error(`spot ${this.id} unavailable`); this.vehicle = v; }
  release() { this.vehicle = null; }
}

class Ticket {
  exitTime?: Date; fee?: number;
  constructor(readonly id: string, readonly vehicle: Vehicle, readonly spot: Spot, readonly entryTime: Date) {}
}

interface SpotAssignmentStrategy { find(spots: Spot[], v: Vehicle): Spot | undefined }
class SmallestFitFirst implements SpotAssignmentStrategy {
  // prefer the smallest compatible spot so large spots stay free for trucks
  find(spots: Spot[], v: Vehicle) {
    return spots.filter((s) => s.canFit(v)).sort((a, b) => a.size - b.size || a.floor - b.floor)[0];
  }
}

interface PricingStrategy { fee(ticket: Ticket, exit: Date): number }
class HourlyPricing implements PricingStrategy {
  constructor(private ratePerHour: Record<VehicleType, number>) {}
  fee(t: Ticket, exit: Date) {
    const hours = Math.max(1, Math.ceil((exit.getTime() - t.entryTime.getTime()) / 3_600_000));
    return hours * this.ratePerHour[t.vehicle.type];
  }
}

interface Clock { now(): Date }                             // injectable for tests
const systemClock: Clock = { now: () => new Date() };

class ParkingLot {
  private spots: Spot[] = [];
  private active = new Map<string, Ticket>();               // plate → ticket
  private seq = 0;
  private listeners: ((free: number) => void)[] = [];

  constructor(
    private assigner: SpotAssignmentStrategy,
    private pricing: PricingStrategy,
    private clock: Clock = systemClock,
  ) {}

  addFloor(floor: number, layout: SpotSize[]) {
    layout.forEach((size, i) => this.spots.push(new Spot(`F${floor}-${i}`, floor, size)));
  }

  onAvailabilityChange(fn: (free: number) => void) { this.listeners.push(fn); }   // Observer
  private notify() { const free = this.spots.filter((s) => s.isFree).length; this.listeners.forEach((l) => l(free)); }

  // Node is single-threaded: no await between find and park → atomic.
  // In Java, guard with a lock (or lock per spot + tryLock) for multiple gates.
  enter(v: Vehicle): Ticket {
    if (this.active.has(v.plate)) throw new Error(`${v.plate} already parked`);
    const spot = this.assigner.find(this.spots, v);
    if (!spot) throw new Error("lot full for this vehicle type");
    spot.park(v);
    const ticket = new Ticket(`T${++this.seq}`, v, spot, this.clock.now());
    this.active.set(v.plate, ticket);
    this.notify();
    return ticket;
  }

  exit(plate: string): Ticket {
    const ticket = this.active.get(plate);
    if (!ticket) throw new Error(`no active ticket for ${plate}`);
    ticket.exitTime = this.clock.now();
    ticket.fee = this.pricing.fee(ticket, ticket.exitTime);
    ticket.spot.release();
    this.active.delete(plate);
    this.notify();
    return ticket;
  }
}

// demo
const lot = new ParkingLot(new SmallestFitFirst(), new HourlyPricing({ BIKE: 10, CAR: 40, TRUCK: 100 }));
lot.addFloor(0, [SpotSize.SMALL, SpotSize.MEDIUM, SpotSize.MEDIUM, SpotSize.LARGE]);
lot.onAvailabilityChange((free) => console.log(`display: ${free} free`));
const t = lot.enter(new Vehicle("KA01AB1234", VehicleType.CAR));
console.log(lot.exit("KA01AB1234").fee);
```

**Follow-ups:** EV charging spots (add a `SpotFeature` set), reservations, multiple gates (a lock per floor, or `ConcurrentHashMap` plus CAS on spot status), and payment methods (Strategy).

---

## 48. LRU Cache

**Requirements:** `get(key)` and `put(key, value)` in **O(1)**. Evict the least recently used entry when at capacity. LeetCode 146, and very common.

**Design:** A hash map from key to node, plus a doubly linked list ordered by recency. The head is most recent and the tail is least recent. Sentinel nodes remove edge cases.

```ts
class DNode<K, V> {
  prev: DNode<K, V> | null = null;
  next: DNode<K, V> | null = null;
  constructor(public key: K, public value: V) {}
}

class LRUCache<K, V> {
  private map = new Map<K, DNode<K, V>>();
  private head = new DNode<K, V>(null as K, null as V);   // sentinel: most recent after head
  private tail = new DNode<K, V>(null as K, null as V);   // sentinel: least recent before tail

  constructor(private capacity: number) {
    if (capacity <= 0) throw new Error("capacity must be > 0");
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  get(key: K): V | undefined {
    const node = this.map.get(key);
    if (!node) return undefined;
    this.moveToFront(node);
    return node.value;
  }

  put(key: K, value: V): void {
    const existing = this.map.get(key);
    if (existing) {
      existing.value = value;
      this.moveToFront(existing);
      return;
    }
    if (this.map.size === this.capacity) {
      const lru = this.tail.prev!;
      this.unlink(lru);
      this.map.delete(lru.key);
    }
    const node = new DNode(key, value);
    this.addToFront(node);
    this.map.set(key, node);
  }

  private unlink(n: DNode<K, V>) { n.prev!.next = n.next; n.next!.prev = n.prev; }
  private addToFront(n: DNode<K, V>) {
    n.next = this.head.next; n.prev = this.head;
    this.head.next!.prev = n; this.head.next = n;
  }
  private moveToFront(n: DNode<K, V>) { this.unlink(n); this.addToFront(n); }
}
```

```ts
// Interview shortcut: JS Map preserves insertion order → delete + re-set = "move to end"
class LRUMap<K, V> {
  private m = new Map<K, V>();
  constructor(private cap: number) {}
  get(k: K) { if (!this.m.has(k)) return undefined; const v = this.m.get(k)!; this.m.delete(k); this.m.set(k, v); return v; }
  put(k: K, v: V) {
    this.m.delete(k); this.m.set(k, v);
    if (this.m.size > this.cap) this.m.delete(this.m.keys().next().value!);  // oldest = first key
  }
}
```

**Follow-ups:** Thread safety (one lock, or a striped/segmented cache like Caffeine), TTL per entry, LFU (a frequency map plus a linked list per frequency, still O(1)), and pluggable eviction (`EvictionPolicy` Strategy).

---

## 49. Rate Limiter (In-Process)

**Requirements:** Per-client limits, configurable algorithm per API, thread-safe. The algorithms are in [Section 14](#14-rate-limiting). Here is the LLD structure.

```ts
interface RateLimitAlgorithm { allow(clientId: string): boolean }

class TokenBucketAlgorithm implements RateLimitAlgorithm {
  private buckets = new Map<string, { tokens: number; last: number }>();
  constructor(private capacity: number, private refillPerSec: number, private clock = () => Date.now()) {}
  allow(clientId: string) {
    const now = this.clock();
    const b = this.buckets.get(clientId) ?? { tokens: this.capacity, last: now };
    b.tokens = Math.min(this.capacity, b.tokens + ((now - b.last) / 1000) * this.refillPerSec);
    b.last = now;
    const ok = b.tokens >= 1;
    if (ok) b.tokens -= 1;
    this.buckets.set(clientId, b);
    return ok;
  }
}

class SlidingLogAlgorithm implements RateLimitAlgorithm {
  private logs = new Map<string, number[]>();
  constructor(private limit: number, private windowMs: number, private clock = () => Date.now()) {}
  allow(clientId: string) {
    const now = this.clock();
    const log = (this.logs.get(clientId) ?? []).filter((t) => t > now - this.windowMs);
    const ok = log.length < this.limit;
    if (ok) log.push(now);
    this.logs.set(clientId, log);
    return ok;
  }
}

type AlgoKind = "TOKEN_BUCKET" | "SLIDING_LOG";
class RateLimiterFactory {
  static create(kind: AlgoKind, cfg: { limit: number; windowMs: number }): RateLimitAlgorithm {
    switch (kind) {
      case "TOKEN_BUCKET": return new TokenBucketAlgorithm(cfg.limit, cfg.limit / (cfg.windowMs / 1000));
      case "SLIDING_LOG": return new SlidingLogAlgorithm(cfg.limit, cfg.windowMs);
    }
  }
}

class RateLimiterService {
  private perEndpoint = new Map<string, RateLimitAlgorithm>();
  register(endpoint: string, algo: RateLimitAlgorithm) { this.perEndpoint.set(endpoint, algo); }
  isAllowed(endpoint: string, clientId: string) {
    return this.perEndpoint.get(endpoint)?.allow(clientId) ?? true;  // no rule → allow
  }
}
```

**Patterns:** Strategy (algorithm), Factory, injectable clock for tests. **Java concurrency:** `ConcurrentHashMap.compute(clientId, ...)` updates the bucket atomically.

---

## 50. Elevator System

**Requirements:** N elevators and M floors. Hall calls (up/down buttons on each floor) and car calls (buttons inside the elevator). A dispatcher assigns requests. Each elevator moves, stops, and opens doors. Optimize wait time. Pluggable scheduling.

**Patterns:** State (elevator: IDLE / MOVING_UP / MOVING_DOWN / MAINTENANCE), Strategy (dispatch algorithm), Mediator/Controller (dispatcher), Observer (floor displays).

**SCAN / LOOK algorithm:** Keep moving in the current direction while there are requests ahead, then reverse. This works like a disk-arm scheduler.

```ts
enum Direction { UP = "UP", DOWN = "DOWN", IDLE = "IDLE" }

class Elevator {
  currentFloor = 0;
  direction = Direction.IDLE;
  private upStops = new Set<number>();
  private downStops = new Set<number>();

  constructor(readonly id: number, readonly minFloor: number, readonly maxFloor: number) {}

  addStop(floor: number) {
    if (floor < this.minFloor || floor > this.maxFloor) throw new Error("invalid floor");
    if (floor === this.currentFloor && this.direction === Direction.IDLE) return; // already here
    (floor > this.currentFloor ? this.upStops : this.downStops).add(floor);
    if (this.direction === Direction.IDLE) this.direction = floor > this.currentFloor ? Direction.UP : Direction.DOWN;
  }

  get pendingStops() { return this.upStops.size + this.downStops.size; }

  /** Advance one tick (one floor). Returns true if the elevator stopped at this floor. */
  step(): boolean {
    if (this.direction === Direction.IDLE) return false;
    this.currentFloor += this.direction === Direction.UP ? 1 : -1;
    const stops = this.direction === Direction.UP ? this.upStops : this.downStops;
    const stopped = stops.delete(this.currentFloor);
    if (stopped) console.log(`Elevator ${this.id} stops at ${this.currentFloor}, doors open`);
    this.updateDirection();
    return stopped;
  }

  private updateDirection() {
    // LOOK: reverse only when no more stops in the current direction
    if (this.direction === Direction.UP && this.upStops.size === 0) {
      this.rebucket();
      this.direction = this.downStops.size ? Direction.DOWN : Direction.IDLE;
    } else if (this.direction === Direction.DOWN && this.downStops.size === 0) {
      this.rebucket();
      this.direction = this.upStops.size ? Direction.UP : Direction.IDLE;
    }
  }

  // after reversing, stops are re-bucketed relative to the new current floor
  private rebucket() {
    const all = [...this.upStops, ...this.downStops];
    this.upStops.clear(); this.downStops.clear();
    for (const f of all) (f > this.currentFloor ? this.upStops : this.downStops).add(f);
  }
}

interface DispatchStrategy { choose(elevators: Elevator[], floor: number, dir: Direction): Elevator }

class NearestSuitableElevator implements DispatchStrategy {
  choose(elevators: Elevator[], floor: number, dir: Direction) {
    const cost = (e: Elevator) => {
      const distance = Math.abs(e.currentFloor - floor);
      const onTheWay =
        (e.direction === Direction.UP && dir === Direction.UP && e.currentFloor <= floor) ||
        (e.direction === Direction.DOWN && dir === Direction.DOWN && e.currentFloor >= floor);
      if (e.direction === Direction.IDLE) return distance;
      return onTheWay ? distance : distance + 2 * (e.maxFloor - e.minFloor);  // penalty: must turn around
    };
    return elevators.reduce((best, e) => (cost(e) < cost(best) ? e : best));
  }
}

class ElevatorController {
  constructor(private elevators: Elevator[], private strategy: DispatchStrategy) {}
  hallCall(floor: number, dir: Direction) { this.strategy.choose(this.elevators, floor, dir).addStop(floor); }
  carCall(elevatorId: number, floor: number) { this.elevators[elevatorId].addStop(floor); }
  tick() { this.elevators.forEach((e) => e.step()); }      // driven by a scheduler/timer
}
```

**Follow-ups:** Capacity and weight limits, an emergency or maintenance state, VIP or fire mode, energy-saving parking, and concurrency (a request queue consumed by the controller thread).

---

## 51. Vending Machine

**Requirements:** Select a product, insert coins/notes, dispense the product and change, cancel with a refund, and track inventory. This is the textbook **State pattern** problem.

```text
         selectProduct          insertMoney (enough)        dispense
  Idle ───────────────▶ HasSelection ───────────────▶ Dispensing ─────▶ Idle
   ▲                         │ cancel → refund                │
   └─────────────────────────┘                                └ change returned
```

```ts
interface VMState {
  select(vm: VendingMachine, code: string): void;
  insert(vm: VendingMachine, amount: number): void;
  cancel(vm: VendingMachine): void;
}

class IdleState implements VMState {
  select(vm: VendingMachine, code: string) {
    const item = vm.inventory.get(code);
    if (!item || item.qty === 0) throw new Error("sold out");
    vm.selected = code;
    vm.setState(new HasSelectionState());
  }
  insert() { throw new Error("select a product first"); }
  cancel() {}
}

class HasSelectionState implements VMState {
  select() { throw new Error("already selected; cancel first"); }
  insert(vm: VendingMachine, amount: number) {
    if (!VendingMachine.ACCEPTED.has(amount)) throw new Error(`unsupported denomination ${amount}`);
    vm.balance += amount;
    const price = vm.inventory.get(vm.selected!)!.price;
    if (vm.balance >= price) {
      vm.setState(new DispensingState());
      vm.dispense();
    }
  }
  cancel(vm: VendingMachine) { vm.refund(); vm.setState(new IdleState()); }
}

class DispensingState implements VMState {
  select() { throw new Error("busy dispensing"); }
  insert() { throw new Error("busy dispensing"); }
  cancel() { throw new Error("cannot cancel while dispensing"); }
}

class VendingMachine {
  static readonly ACCEPTED = new Set([1, 2, 5, 10, 20, 50, 100]);
  private state: VMState = new IdleState();
  inventory = new Map<string, { name: string; price: number; qty: number }>();
  selected: string | null = null;
  balance = 0;

  setState(s: VMState) { this.state = s; }
  select(code: string) { this.state.select(this, code); }
  insert(amount: number) { this.state.insert(this, amount); }
  cancel() { this.state.cancel(this); }

  dispense() {
    const item = this.inventory.get(this.selected!)!;
    item.qty--;
    const change = this.balance - item.price;
    console.log(`Dispensed ${item.name}; change: ${JSON.stringify(makeChange(change))}`);
    this.balance = 0; this.selected = null;
    this.setState(new IdleState());
  }
  refund() { console.log(`Refunded ${this.balance}`); this.balance = 0; this.selected = null; }
}

// Greedy change works for canonical coin systems (₹/$). For arbitrary coins use DP.
function makeChange(amount: number, coins = [100, 50, 20, 10, 5, 2, 1]) {
  const out: Record<number, number> = {};
  for (const c of coins) { const n = Math.floor(amount / c); if (n) { out[c] = n; amount -= n * c; } }
  return out;
}
```

---

## 52. Splitwise

**Requirements:** Users and groups. Add expenses split **equally**, by **exact amounts**, or by **percentage**. Show balances (who owes whom). **Simplify debts** to the minimum number of transactions.

**Patterns:** Strategy (split type) and Factory. The balance sheet is a nested map.

```ts
type UserId = string;
interface Split { user: UserId; amount: number }

interface SplitStrategy { compute(total: number, participants: UserId[], meta?: number[]): Split[] }

class EqualSplit implements SplitStrategy {
  compute(total: number, users: UserId[]) {
    const cents = Math.round(total * 100);
    const base = Math.floor(cents / users.length);
    let remainder = cents - base * users.length;                 // distribute leftover paise/cents
    return users.map((user) => ({ user, amount: (base + (remainder-- > 0 ? 1 : 0)) / 100 }));
  }
}
class ExactSplit implements SplitStrategy {
  compute(total: number, users: UserId[], amounts: number[] = []) {
    const sum = amounts.reduce((a, b) => a + b, 0);
    if (Math.abs(sum - total) > 0.01) throw new Error("exact amounts must sum to total");
    return users.map((user, i) => ({ user, amount: amounts[i] }));
  }
}
class PercentSplit implements SplitStrategy {
  compute(total: number, users: UserId[], pct: number[] = []) {
    if (Math.abs(pct.reduce((a, b) => a + b, 0) - 100) > 0.001) throw new Error("percentages must sum to 100");
    return users.map((user, i) => ({ user, amount: Math.round(total * pct[i]) / 100 }));
  }
}

class ExpenseManager {
  // balances.get(a).get(b) = amount b owes a  (positive)
  private balances = new Map<UserId, Map<UserId, number>>();

  addExpense(paidBy: UserId, total: number, participants: UserId[], strategy: SplitStrategy, meta?: number[]) {
    for (const { user, amount } of strategy.compute(total, participants, meta)) {
      if (user === paidBy) continue;
      this.adjust(paidBy, user, amount);    // user owes paidBy
      this.adjust(user, paidBy, -amount);
    }
  }

  private adjust(a: UserId, b: UserId, delta: number) {
    if (!this.balances.has(a)) this.balances.set(a, new Map());
    const m = this.balances.get(a)!;
    m.set(b, Math.round(((m.get(b) ?? 0) + delta) * 100) / 100);
  }

  netBalances(): Map<UserId, number> {             // + = gets back, − = owes
    const net = new Map<UserId, number>();
    for (const [a, m] of this.balances) for (const v of m.values()) net.set(a, (net.get(a) ?? 0) + v);
    return net;
  }

  // Greedy simplification: repeatedly settle max creditor with max debtor (≤ n−1 transactions)
  simplify(): { from: UserId; to: UserId; amount: number }[] {
    const creditors = [...this.netBalances()].filter(([, v]) => v > 0.005).sort((a, b) => b[1] - a[1]);
    const debtors = [...this.netBalances()].filter(([, v]) => v < -0.005).map(([u, v]) => [u, -v] as [UserId, number]).sort((a, b) => b[1] - a[1]);
    const txns: { from: UserId; to: UserId; amount: number }[] = [];
    let i = 0, j = 0;
    while (i < creditors.length && j < debtors.length) {
      const amt = Math.min(creditors[i][1], debtors[j][1]);
      txns.push({ from: debtors[j][0], to: creditors[i][0], amount: Math.round(amt * 100) / 100 });
      creditors[i][1] -= amt; debtors[j][1] -= amt;
      if (creditors[i][1] < 0.005) i++;
      if (debtors[j][1] < 0.005) j++;
    }
    return txns;
  }
}

const em = new ExpenseManager();
em.addExpense("alice", 300, ["alice", "bob", "carol"], new EqualSplit());
em.addExpense("bob", 100, ["alice", "bob"], new PercentSplit(), [40, 60]);
console.log(em.simplify());
```

**Note:** In production, store money as integer minor units (paise/cents), not floats. The true minimum-transactions problem is NP-hard, so greedy is the accepted answer.

---

## 53. Movie Ticket Booking (BookMyShow)

**Requirements:** Cities → theatres → screens → shows. Seat map with categories and prices. Users select seats, which are **locked for 10 minutes** while they pay. Booking is confirmed on payment success. **No double booking under concurrency.**

**Entities:** Movie, Theatre, Screen, Seat, Show, ShowSeat (seat state per show), Booking, Payment.
**Patterns:** Strategy (payment, pricing), State (booking: PENDING → CONFIRMED / CANCELLED / EXPIRED), Observer (notifications). Concurrency is the core of this problem.

```ts
enum SeatStatus { AVAILABLE = "AVAILABLE", LOCKED = "LOCKED", BOOKED = "BOOKED" }
enum BookingStatus { PENDING = "PENDING", CONFIRMED = "CONFIRMED", EXPIRED = "EXPIRED", CANCELLED = "CANCELLED" }

class ShowSeat {
  status = SeatStatus.AVAILABLE;
  lockedBy: string | null = null;
  lockExpiresAt = 0;
  constructor(readonly seatId: string, readonly price: number) {}
  isAvailable(now: number) {
    return this.status === SeatStatus.AVAILABLE || (this.status === SeatStatus.LOCKED && now > this.lockExpiresAt);
  }
}

class Show {
  readonly seats = new Map<string, ShowSeat>();
  constructor(readonly id: string, readonly movieId: string, readonly startsAt: Date) {}
}

class Booking {
  status = BookingStatus.PENDING;
  constructor(readonly id: string, readonly userId: string, readonly show: Show, readonly seatIds: string[], readonly amount: number, readonly expiresAt: number) {}
}

interface PaymentStrategy { pay(amount: number, ref: string): Promise<boolean> }

class SeatLockService {
  constructor(private lockMs = 10 * 60_000, private now = () => Date.now()) {}

  /** All-or-nothing lock. Synchronous body → atomic on Node's event loop.
   *  Multi-instance: same logic via DB conditional UPDATE or Redis SET NX per seat (Section 17). */
  lock(show: Show, seatIds: string[], userId: string): boolean {
    const t = this.now();
    const seats = seatIds.map((id) => show.seats.get(id));
    if (seats.some((s) => !s || !s.isAvailable(t))) return false;
    for (const s of seats as ShowSeat[]) {
      s.status = SeatStatus.LOCKED; s.lockedBy = userId; s.lockExpiresAt = t + this.lockMs;
    }
    return true;
  }

  isHeldBy(show: Show, seatIds: string[], userId: string) {
    const t = this.now();
    return seatIds.every((id) => { const s = show.seats.get(id)!; return s.status === SeatStatus.LOCKED && s.lockedBy === userId && t <= s.lockExpiresAt; });
  }

  release(show: Show, seatIds: string[]) {
    for (const id of seatIds) { const s = show.seats.get(id)!; if (s.status === SeatStatus.LOCKED) { s.status = SeatStatus.AVAILABLE; s.lockedBy = null; } }
  }
}

class BookingService {
  private seq = 0;
  constructor(private locks: SeatLockService, private payment: PaymentStrategy) {}

  createBooking(userId: string, show: Show, seatIds: string[]): Booking {
    if (!this.locks.lock(show, seatIds, userId)) throw new Error("one or more seats unavailable");
    const amount = seatIds.reduce((sum, id) => sum + show.seats.get(id)!.price, 0);
    return new Booking(`B${++this.seq}`, userId, show, seatIds, amount, Date.now() + 10 * 60_000);
  }

  async confirm(booking: Booking): Promise<Booking> {
    if (booking.status !== BookingStatus.PENDING) throw new Error(`booking is ${booking.status}`);
    if (!this.locks.isHeldBy(booking.show, booking.seatIds, booking.userId)) {
      booking.status = BookingStatus.EXPIRED;
      throw new Error("seat hold expired");
    }
    const ok = await this.payment.pay(booking.amount, booking.id);
    // re-check after await: another request may have run while we waited on payment
    if (!ok || !this.locks.isHeldBy(booking.show, booking.seatIds, booking.userId)) {
      this.locks.release(booking.show, booking.seatIds);
      booking.status = BookingStatus.CANCELLED;      // (refund if paid but hold lost)
      throw new Error("payment failed or hold lost");
    }
    for (const id of booking.seatIds) booking.show.seats.get(id)!.status = SeatStatus.BOOKED;
    booking.status = BookingStatus.CONFIRMED;
    return booking;
  }
}
```

**DB version of the lock (production):**

```sql
UPDATE show_seats SET status='LOCKED', locked_by=:user, lock_expires_at=now() + interval '10 min'
WHERE show_id=:show AND seat_id = ANY(:seats)
  AND (status='AVAILABLE' OR (status='LOCKED' AND lock_expires_at < now()));
-- rowsAffected must equal number of seats, else ROLLBACK (all-or-nothing in one transaction)
```

---

## 54. Logger / Pub-Sub

### Logger (Chain of Responsibility + Singleton + Strategy for sinks)

```ts
enum Level { DEBUG = 10, INFO = 20, WARN = 30, ERROR = 40 }

interface LogSink { write(line: string): void }
class ConsoleSink implements LogSink { write(l: string) { console.log(l); } }
class FileSink implements LogSink { constructor(private path: string) {} write(l: string) { /* fs.appendFile */ } }

interface Formatter { format(level: Level, msg: string, ctx: object): string }
class JsonFormatter implements Formatter {
  format(level: Level, msg: string, ctx: object) { return JSON.stringify({ ts: new Date().toISOString(), level: Level[level], msg, ...ctx }); }
}

class Logger {
  private static instance?: Logger;
  private constructor(private minLevel: Level, private sinks: LogSink[], private fmt: Formatter) {}
  static get(): Logger { return (this.instance ??= new Logger(Level.INFO, [new ConsoleSink()], new JsonFormatter())); }

  private log(level: Level, msg: string, ctx: object = {}) {
    if (level < this.minLevel) return;
    const line = this.fmt.format(level, msg, ctx);
    for (const s of this.sinks) s.write(line);
  }
  debug(m: string, c?: object) { this.log(Level.DEBUG, m, c); }
  info(m: string, c?: object) { this.log(Level.INFO, m, c); }
  warn(m: string, c?: object) { this.log(Level.WARN, m, c); }
  error(m: string, c?: object) { this.log(Level.ERROR, m, c); }
}
```

### In-memory pub/sub (Kafka-lite LLD question)

**Requirements:** Topics, publishers, and subscribers. Each subscriber consumes at its own pace with an offset. Support resetting the offset (replay).

```ts
class Topic {
  readonly messages: string[] = [];
  constructor(readonly name: string) {}
}

interface Subscriber { readonly id: string; consume(msg: string): Promise<void> }

class TopicSubscription {
  offset = 0;
  private running = false;
  constructor(readonly topic: Topic, readonly sub: Subscriber) {}

  async drain() {                        // single consumer loop per subscription → ordered
    if (this.running) return;
    this.running = true;
    try {
      while (this.offset < this.topic.messages.length) {
        await this.sub.consume(this.topic.messages[this.offset]);
        this.offset++;                    // commit after successful processing (at-least-once)
      }
    } finally {
      this.running = false;
    }
  }
}

class Broker {
  private topics = new Map<string, Topic>();
  private subs = new Map<string, TopicSubscription[]>();

  createTopic(name: string) { this.topics.set(name, new Topic(name)); this.subs.set(name, []); }
  subscribe(topic: string, s: Subscriber) { this.subs.get(topic)!.push(new TopicSubscription(this.topics.get(topic)!, s)); }
  publish(topic: string, msg: string) {
    this.topics.get(topic)!.messages.push(msg);
    for (const ts of this.subs.get(topic)!) void ts.drain();
  }
  resetOffset(topic: string, subId: string, offset: number) {
    const ts = this.subs.get(topic)!.find((t) => t.sub.id === subId)!;
    ts.offset = offset;
    void ts.drain();
  }
}
```

---

## 55. Tic-Tac-Toe / Board Games

**Requirements:** An N×N board, 2+ players, turns, and win/draw detection. Extensible to other games (Snake & Ladder, Chess).
**Key optimization:** Check for a win in **O(1)** per move with row, column, and diagonal counters, instead of scanning the board in O(N²).

```ts
class Player { constructor(readonly name: string, readonly symbol: string) {} }

class TicTacToe {
  private board: (string | null)[][];
  private rows: number[][]; private cols: number[][];
  private diag: number[]; private anti: number[];
  private moves = 0;
  private turn = 0;
  winner: Player | null = null;

  constructor(private n: number, private players: Player[]) {
    this.board = Array.from({ length: n }, () => Array(n).fill(null));
    this.rows = players.map(() => Array(n).fill(0));
    this.cols = players.map(() => Array(n).fill(0));
    this.diag = players.map(() => 0);
    this.anti = players.map(() => 0);
  }

  move(r: number, c: number): "WIN" | "DRAW" | "CONTINUE" {
    if (this.winner) throw new Error("game over");
    if (r < 0 || c < 0 || r >= this.n || c >= this.n || this.board[r][c]) throw new Error("invalid move");
    const p = this.turn, player = this.players[p];
    this.board[r][c] = player.symbol;
    this.moves++;
    const n = this.n;
    if (++this.rows[p][r] === n || ++this.cols[p][c] === n ||
        (r === c && ++this.diag[p] === n) || (r + c === n - 1 && ++this.anti[p] === n)) {
      this.winner = player;
      return "WIN";
    }
    if (this.moves === n * n) return "DRAW";
    this.turn = (this.turn + 1) % this.players.length;
    return "CONTINUE";
  }
}
```

The `&&` and `||` short-circuit means a counter may not increment once a win is already found. That's fine because the game ends there. To make other games pluggable, extract `Board`, `Piece`, `MoveValidator` (Strategy per piece in chess), and `WinningStrategy` interfaces.

---

# Part VI: Final Prep

## 56. Trade-Off Cheat Sheet

| Decision | Option A | Option B | Rule of thumb |
| --- | --- | --- | --- |
| SQL vs NoSQL | Joins, transactions, flexible queries | Horizontal write scale, simple access patterns | Start with SQL. Move when the access pattern or scale demands it. |
| Consistency vs availability | Correct (money, seats) | Always responds (feeds, likes) | Choose **per feature** |
| Push vs pull (fan-out) | Fast reads | Cheap writes | Hybrid for celebrities |
| Sync vs async | Simple, immediate result | Resilient, absorbs spikes | Async for anything slow or non-critical to the response |
| Cache | Fast | Stale data, complexity | Cache read-heavy, rarely changing data. Define a staleness budget. |
| Normalize vs denormalize | Write integrity | Read speed | Denormalize for hot read paths |
| Polling vs WebSocket | Simple, stateless | Real-time, stateful servers | SSE when updates only go server → client |
| Monolith vs microservices | Simple | Independent scale | Split only by team or scaling need |
| Strong vs eventual (replicas) | Read from the leader | Read from replicas | Read-your-writes for the author |
| Batch vs stream | Accurate, cheap | Low latency | Lambda/Kappa. Reconcile streams with batch. |
| Optimistic vs pessimistic lock | Low contention | High contention | OCC by default. Pessimistic for hot rows. |
| Range vs hash sharding | Range scans | Even spread | Hash unless range queries dominate |

### Default technology picks (say these with confidence)

| Need | Default |
| --- | --- |
| Primary DB | PostgreSQL |
| Massive write / time-series per key | Cassandra / DynamoDB |
| Cache, counters, leaderboards, locks, geo | Redis |
| Queue / event stream | Kafka (SQS for simple job queues) |
| Search | Elasticsearch |
| Blobs | S3 + CloudFront CDN |
| Workflow / saga orchestration | Temporal |
| Stream processing | Flink |
| Analytics | ClickHouse / BigQuery / Snowflake |
| Coordination | ZooKeeper / etcd |

---

## 57. Common Mistakes and Interviewer Signals

### HLD mistakes

1. **Jumping into the design** without requirements. Always spend the first 5 minutes on FRs and NFRs.
2. **Over-engineering early:** Kafka, microservices, and sharding before showing a simple working design. Make it work, then make it scale.
3. **Spending 10 minutes on math** that doesn't change any decision.
4. **Buzzwords without reasons:** "We'll use Cassandra" with no explanation of why.
5. **Ignoring the hard part:** Every problem has one (contention in Ticketmaster, fan-out in feeds, matching in Uber, connection routing in chat). Find it and go deep.
6. **No failure discussion:** What happens when Redis, a worker, or a region dies?
7. **Silent drawing:** Narrate your trade-offs out loud.
8. **Not checking in:** Ask "Does this look reasonable before I deep-dive into X?"

### LLD mistakes

1. A god class (`ParkingLotManager` does everything).
2. `if/else` on type everywhere. Replace with polymorphism, Strategy, or State.
3. Public mutable fields. Encapsulate and validate.
4. Ignoring concurrency when the problem obviously has shared resources (seats, spots, accounts).
5. Forcing patterns that don't fit. Name only the ones you use, and explain why.
6. Running out of time before having working core flows. Code the happy path first.
7. No demo driver or tests. In machine-coding rounds, show it running.

### Phrases that signal seniority

- "The bottleneck here is ___, because at ___ QPS a single ___ can't keep up."
- "We'd trade ___ for ___ here, which is acceptable because the requirement says ___."
- "This part needs strong consistency, but this other part can be eventually consistent."
- "To avoid the dual-write problem, I'd use a transactional outbox."
- "This must be idempotent because the queue gives at-least-once delivery."
- "If this grows 10×, the first thing to break is ___, and we'd ___."

---

## 58. Master Resource List

### Books

- **Alex Xu, *System Design Interview – An Insider's Guide*, Vol. 1:** Scaling, estimation, rate limiter, consistent hashing, KV store, unique IDs, URL shortener, crawler, notifications, news feed, chat, autocomplete, YouTube, Google Drive.
- **Alex Xu & Sahn Lam, Vol. 2:** Proximity service, nearby friends, Google Maps, distributed message queue, metrics monitoring, ad click aggregation, hotel reservation, distributed email, S3-like object storage, real-time gaming leaderboard, payment system, digital wallet, stock exchange.
- **Martin Kleppmann, *Designing Data-Intensive Applications* (DDIA):** The deep theory: replication, partitioning, transactions, consistency, consensus, and stream processing. Read chapters 5–9 at minimum.
- ***Head First Design Patterns*** (Freeman & Robson): The most approachable way to learn patterns.
- ***Design Patterns*** (Gang of Four): The original catalog.
- ***Clean Code*** / ***Clean Architecture*** (Robert C. Martin): SOLID and code quality.
- ***Effective Java*** (Joshua Bloch) and ***Java Concurrency in Practice*** (Brian Goetz): For Java-based LLD rounds.

### Websites and courses

- **Hello Interview:** [System Design in a Hurry](https://www.hellointerview.com/learn/system-design/in-a-hurry/introduction), [Core Concepts](https://www.hellointerview.com/learn/system-design/in-a-hurry/core-concepts), [Key Technologies](https://www.hellointerview.com/learn/system-design/in-a-hurry/key-technologies), [Patterns](https://www.hellointerview.com/learn/system-design/in-a-hurry/patterns), [Problem Breakdowns](https://www.hellointerview.com/learn/system-design/in-a-hurry/problem-breakdowns), [Low-Level Design](https://www.hellointerview.com/learn/low-level-design/in-a-hurry/introduction). Their free deep dives on [Redis](https://www.hellointerview.com/learn/system-design/deep-dives/redis), [Kafka](https://www.hellointerview.com/learn/system-design/deep-dives/kafka), [Elasticsearch](https://www.hellointerview.com/learn/system-design/deep-dives/elasticsearch), [Cassandra](https://www.hellointerview.com/learn/system-design/deep-dives/cassandra), and [DynamoDB](https://www.hellointerview.com/learn/system-design/deep-dives/dynamodb) are excellent.
- **ByteByteGo** ([bytebytego.com](https://bytebytego.com/)): Alex Xu's course and newsletter, with great diagrams.
- **System Design Primer** ([github.com/donnemartin/system-design-primer](https://github.com/donnemartin/system-design-primer)): A free, comprehensive overview with Anki flashcards.
- **Refactoring.Guru** ([refactoring.guru/design-patterns](https://refactoring.guru/design-patterns)): The best visual pattern reference, with code in TS and Java.
- **awesome-low-level-design** ([github.com/ashishps1/awesome-low-level-design](https://github.com/ashishps1/awesome-low-level-design)): LLD problems with solutions in many languages.
- **AWS Builders' Library** ([aws.amazon.com/builders-library](https://aws.amazon.com/builders-library/)): Timeouts, retries, jitter, load shedding, and caching from Amazon engineers.
- **Martin Fowler's blog** ([martinfowler.com](https://martinfowler.com/)): Microservices, event sourcing, CQRS, and the patterns of distributed systems.
- **High Scalability** ([highscalability.com](http://highscalability.com/)): Real architecture case studies.

### Papers worth knowing by name

Amazon **Dynamo** (2007), Google **Bigtable** (2006), **GFS** (2003), **MapReduce** (2004), **Spanner** (2012), **Raft** ("In Search of an Understandable Consensus Algorithm", 2014), Facebook **TAO** (2013), Facebook **Memcache at scale** (2013), LinkedIn **Kafka** (2011).

### Engineering blogs (real-world deep dives)

Netflix TechBlog, Uber Engineering, Discord ("How Discord stores trillions of messages"), Meta Engineering, Airbnb, Stripe (idempotency, rate limiters), Cloudflare, Slack, Pinterest, Shopify (flash sales), and Figma (multiplayer).

### YouTube

Hello Interview (mock interviews and breakdowns), ByteByteGo, Jordan Has No Life (deep systems content), Gaurav Sen (fundamentals), and Concept && Coding (LLD in Java).

---

**Last step before the interview:** Redo three case studies from memory under a timer. Explain them out loud to someone or record yourself. Review Sections 1, 2, 56, and 57 the night before.
