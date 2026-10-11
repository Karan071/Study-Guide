# Caching Techniques: A Backend Engineer's Reference

Audience: SDEs designing, building, and operating production backends, web applications, and distributed systems.
Scope: 20 core concepts, each organized around **Core Mechanism**, **Engineering Blueprint**, **Production Pitfalls**, **SDE Best Practices**, and **Resources**.
Last reviewed: 2026-10-11. HTTP caching standards and vendor behavior evolve; consult the linked specifications and product documentation before relying on implementation-specific behavior.

Related guide: [Authentication, Authorization & Modern Security Protocols](./AuthenticationConcepts.md), especially its sections on sessions, authorization, and tenant isolation.

## Table of Contents

0. [System Map: How Caching Concepts Fit Together](#0-system-map-how-caching-concepts-fit-together)
1. [Cache Fundamentals](#1-cache-fundamentals)
2. [Cache-Aside (Lazy Loading)](#2-cache-aside-lazy-loading)
3. [Read-Through Caching](#3-read-through-caching)
4. [Write-Through Caching](#4-write-through-caching)
5. [Write-Behind (Write-Back) Caching](#5-write-behind-write-back-caching)
6. [Refresh-Ahead Caching](#6-refresh-ahead-caching)
7. [TTL and Expiration](#7-ttl-and-expiration)
8. [Eviction Policies](#8-eviction-policies)
9. [Cache Invalidation](#9-cache-invalidation)
10. [Consistency and Staleness](#10-consistency-and-staleness)
11. [Cache Stampedes and Miss Coalescing](#11-cache-stampedes-and-miss-coalescing)
12. [Negative Caching](#12-negative-caching)
13. [Cache-Key Design and Namespacing](#13-cache-key-design-and-namespacing)
14. [HTTP Caching Semantics](#14-http-caching-semantics)
15. [CDNs and Shared Edge Caches](#15-cdns-and-shared-edge-caches)
16. [Browser and Client-Side Caching](#16-browser-and-client-side-caching)
17. [Distributed Caches](#17-distributed-caches)
18. [In-Process Caches](#18-in-process-caches)
19. [Redis as a Cache](#19-redis-as-a-cache)
20. [Cache Security and Isolation](#20-cache-security-and-isolation)
21. [Production Checklist and Cheat Sheet](#21-production-checklist-and-cheat-sheet)
22. [Self-Check Questions](#22-self-check-questions)
23. [Master Resource List](#23-master-resource-list)

---

## 0. System Map: How Caching Concepts Fit Together

A cache is a performance and load-management layer, not the authoritative source of truth. Start with the data's correctness requirements, then choose where copies live and how those copies become fresh enough.

| Concern | Design question | Relevant concepts |
| --- | --- | --- |
| Read path | Who loads a missing value? | 2, 3, 6 |
| Write path | When does the source of truth change? | 4, 5, 9 |
| Freshness | How old may a value be? | 7, 9, 10 |
| Capacity | What happens when memory is full? | 8, 17, 18, 19 |
| Concurrency | What happens when many requests miss together? | 11 |
| Data shape | How are entries identified and partitioned? | 12, 13, 20 |
| HTTP delivery | Which layer may store and reuse a response? | 14, 15, 16 |
| Operations | How do we know whether caching helps? | 21, 22 |

### Common cache topology

```text
Client / Browser
      |
      | HTTP cache directives [14,16]
      v
CDN / Edge cache [15] ---- cache hit ----> response
      |
      | miss / revalidation
      v
Application
  |       \
  |        \ local in-process cache [18]
  v
Distributed cache [17,19] ---- hit ----> response
  |
  | miss (cache-aside / read-through) [2,3]
  v
Database / origin (source of truth)

Writes: application -> source of truth -> invalidation/update events -> cache [4,9]
```

### Decision defaults

| Decision | Sensible starting point |
| --- | --- |
| First application cache pattern | Cache-aside with explicit ownership of load and invalidation |
| Freshness | Set a TTL from the product's staleness budget, not from guesswork |
| Writes | Commit authoritative data first, then invalidate/update caches reliably |
| Concurrent misses | Coalesce duplicate work and bound origin concurrency |
| Cache outage | Define a deliberate fail-open or fail-closed policy per operation |
| HTTP responses | Use explicit `Cache-Control`; use `private` for user-specific responses |
| Cache key | Include every input that changes the result, including tenant and authorization scope |
| Metrics | Track hit/miss, latency, evictions, errors, origin load, and stale responses |
| Correctness | Treat cache contents as disposable and rebuildable unless the architecture explicitly makes them durable |

### Staleness budget

For each cached value, write down the maximum acceptable age and what happens when it is exceeded. A product list can often tolerate seconds of staleness; an account balance or authorization decision may require a direct authoritative read. “We set a TTL” is not a consistency specification: expiration bounds age only when refreshes and failures behave as expected.

---

## 1. Cache Fundamentals

A cache stores a derived or copied value near the consumer to reduce latency, repeated computation, or load on an origin. It trades additional state and invalidation complexity for faster or cheaper reads.

### Core mechanism

- A lookup is a **hit** when a usable entry exists and a **miss** otherwise.
- On a miss, some component obtains or computes the value from an origin, then optionally stores it.
- Cache value depends on the full path: cache lookup overhead, serialization, network hops, origin latency, and the probability of a hit.
- A cache can improve average latency while making tail latency or failure behavior worse if misses overload the origin.

### Engineering blueprint

Estimate benefit before introducing a cache:

```text
expected_read_cost =
  hit_rate * cache_hit_cost
  + (1 - hit_rate) * (cache_miss_cost + origin_cost)

origin_read_rate = request_rate * (1 - hit_rate)
```

These estimates are only useful if the workload and hit rate are measured at the same key and time scale. For a cache with hit rate `h`, a rough origin-load reduction is `1 - h`, but miss bursts, refreshes, retries, and invalidations add load.

### Production pitfalls

- Caching a low-cost operation can add more complexity and latency than it removes.
- Averages conceal hot keys, cold-start behavior, and p99 miss latency.
- A high hit rate does not prove correctness; stale or cross-user data can still be returned quickly.
- A cache is not automatically durable storage, even if the product persists snapshots.

### SDE best practices

- Establish a baseline for origin QPS and latency before rollout.
- Define the data owner, freshness tolerance, cache-outage behavior, and recovery plan.
- Make cache use observable and controllable with a feature flag or configuration.
- Test cold start, eviction, timeout, and cache outage—not only the warm-hit path.

### Resources

- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
- [AWS: Caching best practices](https://aws.amazon.com/caching/best-practices/)
- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)

---

## 2. Cache-Aside (Lazy Loading)

In cache-aside, the application checks the cache first. On a miss, it loads from the source of truth, populates the cache, and returns the result. The application explicitly owns the read-through sequence.

### Core mechanism

```text
get(key):
  value = cache.get(key)
  if value is present:
    return value

  value = origin.load(key)
  if value is cacheable:
    cache.set(key, value, ttl)
  return value
```

On writes, a common strategy is to update the origin and then invalidate the corresponding cache key. The next read repopulates it.

A concrete version with Redis (Python, `redis-py`):

```python
import json, random

TTL = 300
NOT_FOUND = "__nf__"           # sentinel so "missing" is distinct from a cache miss

def get_product(product_id: str):
    key = f"v1:product:{product_id}"
    cached = r.get(key)
    if cached is not None:
        return None if cached == NOT_FOUND else json.loads(cached)

    row = db.fetch_product(product_id)                     # source of truth
    if row is None:
        r.set(key, NOT_FOUND, ex=30)                       # short negative-cache TTL
        return None
    r.set(key, json.dumps(row), ex=TTL + random.randint(0, 30))  # jitter spreads expiries
    return row

def update_product(product_id: str, fields: dict):
    db.update_product(product_id, fields)                  # commit first
    r.delete(f"v1:product:{product_id}")                   # then invalidate (delete, not set)
```

Deleting on write is usually safer than writing the new value into the cache: two concurrent writers that each `SET` can finish in the opposite order from their database commits and leave the older value cached.

### Engineering blueprint

- Distinguish a missing key from a cached null/negative result.
- Set a finite TTL as a recovery mechanism for missed invalidations.
- Apply timeouts and concurrency limits to origin loads.
- Decide whether origin errors should be returned, served from a known stale copy, or mapped to a domain fallback.
- Make cache population best-effort only when that is an explicit policy; report cache failures rather than disguising them as origin success.

### Production pitfalls

- The first request after expiry or invalidation pays full origin latency.
- Concurrent misses for the same key can stampede the origin.
- Updating the database and deleting the key are not one atomic operation; races can repopulate stale values.
- A value that is not cacheable (for example, “not found”) can trigger repeated origin work.

### SDE best practices

- Use cache-aside for data whose reads are common and whose write paths can own invalidation.
- Keep the authoritative read path correct when the cache is empty.
- Use miss coalescing for expensive or highly popular keys.
- Add bounded retries only where safe; retries can multiply origin load during an outage.

### Resources

- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)

---

## 3. Read-Through Caching

In read-through caching, the cache abstraction loads a missing value from the backing store. Callers ask the cache for data and do not implement the miss-and-populate sequence themselves.

### Core mechanism

- The cache coordinates lookup, load, and insertion through a configured loader.
- It can centralize serialization, expiry, and duplicate-load coordination.
- From the caller's perspective, `get(key)` can return either a cached value or a loaded value.

### Engineering blueprint

Before adopting a read-through implementation, verify:

- Loader timeouts and cancellation behavior.
- Whether concurrent requests for one key share a single loader call.
- How nulls, exceptions, and partial results are handled.
- Which metrics expose loader latency and failures.
- Whether invalidation APIs are available to all write paths.

### Production pitfalls

- Hiding the origin read inside a cache library can obscure latency and error ownership.
- Loader failures may be retried implicitly or transformed into cache misses, creating retry storms.
- A cache loader may not know enough domain context to construct a safe key.
- It does not solve invalidation or consistency by itself.

### SDE best practices

- Use read-through when the cache product has clear loader semantics and the application benefits from a shared abstraction.
- Keep the data contract and failure policy explicit at the call site or cache boundary.
- Verify that loader execution is bounded and observable under load.
- Prefer one ownership model per data set; avoid mixing hidden loaders with ad hoc direct cache writes.

### Resources

- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) (contrasts application-managed loading with cache-managed patterns)
- [Caffeine: LoadingCache](https://github.com/ben-manes/caffeine/wiki/Population)
- [Spring Framework: Cache abstraction](https://docs.spring.io/spring-framework/reference/integration/cache/annotations.html)

---

## 4. Write-Through Caching

In write-through, a write updates the cache and backing store as part of the same logical operation. The goal is for subsequent cache reads to see the newly written value.

### Core mechanism

- The cache write path forwards the change to the origin.
- Success semantics must specify whether both cache and origin writes are required.
- Some products provide a write-through loader/writer; others require application-level orchestration.

### Engineering blueprint

Define the ordering and failure behavior explicitly:

| Origin write | Cache write | Safe interpretation |
| --- | --- | --- |
| Fails | Not attempted | Return failure; origin remains authoritative |
| Succeeds | Succeeds | Return success after both are confirmed |
| Succeeds | Fails | Origin changed; report/repair cache failure or invalidate the key |

If the origin and cache do not share a transaction, write-through is not atomic. A reliable alternative is to commit the origin and publish a durable change event (for example, with an outbox), then update or invalidate the cache asynchronously.

### Production pitfalls

- A cache failure after an origin commit can produce an ambiguous client response and duplicate retries.
- Writing cache first can expose a value that the database later rejects.
- Cache writes on every mutation can cost more than the read savings for write-heavy data.
- Cross-region cache propagation is not necessarily synchronous.

### SDE best practices

- Prefer the database or other source of truth as the commit authority.
- Make operations idempotent or provide a way to resolve ambiguous outcomes.
- Invalidate on cache-write failure when stale reads are unacceptable.
- Use transactional messaging when cache updates must eventually happen despite process crashes.

### Resources

- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS Prescriptive Guidance: Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)

---

## 5. Write-Behind (Write-Back) Caching

Write-behind acknowledges a write to the cache first and propagates it to the backing store later, often in batches. It can reduce write latency and combine repeated updates, but makes the cache part of the durability path.

### Core mechanism

- The cache buffers mutations and asynchronously flushes them to the origin.
- A queue, log, or durable journal is needed if acknowledged writes must survive process or node failure.
- Flush ordering, retries, deduplication, and conflict handling become data-correctness concerns.

### Engineering blueprint

Use only after answering:

1. What does a successful client response guarantee?
2. Is the pending write durable across cache node and region failure?
3. How are repeated or out-of-order updates reconciled?
4. How are backlogs bounded and alerted?
5. Can a value be read from the origin before its buffered write arrives?

### Production pitfalls

- A cache restart can lose acknowledged writes if the buffer is volatile.
- A growing write queue can make data increasingly stale and exhaust memory.
- Retrying a non-idempotent mutation may apply it more than once.
- Reads from the backing store can return older data than the cache.

### SDE best practices

- Do not use write-behind for financial, inventory, or other critical records without an explicit durability and recovery design.
- Prefer durable queues/logs and idempotent events when asynchronous propagation is required.
- Monitor oldest pending-write age, queue depth, flush failures, and replay progress.
- Document reconciliation and disaster recovery procedures before launch.

### Resources

- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS Prescriptive Guidance: Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html)

---

## 6. Refresh-Ahead Caching

Refresh-ahead refreshes a value before it expires, commonly when it is requested and nearing expiry. It aims to keep popular data warm and avoid user-visible cold misses.

### Core mechanism

- The cache tracks age or expiry.
- A request can receive the current value while a refresh runs in the background, or wait for refreshed data depending on the consistency contract.
- Refreshes are useful only when access frequency and refresh cost justify the added work.

### Engineering blueprint

- Refresh only entries with a meaningful chance of being requested again.
- Add jitter to refresh times to avoid synchronized fleet-wide refreshes.
- Coalesce refresh work per key.
- Define what readers receive when refresh fails: current value until hard expiry, origin error, or a bounded stale value.
- Bound background concurrency and prioritize high-value keys.

### Production pitfalls

- Refreshing every key can create more origin traffic than demand-driven loading.
- A background refresh can race with a newer write and overwrite fresh cache data.
- A refresh failure can leave an entry appearing healthy beyond the intended staleness limit.

### SDE best practices

- Separate soft expiry (refresh eligible) from hard expiry (must not be served).
- Use version checks or compare-and-set semantics so an old refresh cannot replace a newer value.
- Measure refresh success, refresh age, and origin work, not just hit rate.

### Resources

- [Redis: Client-side caching](https://redis.io/docs/latest/develop/clients/client-side-caching/) (invalidation and tracking mechanics)
- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)

---

## 7. TTL and Expiration

Time-to-live (TTL) limits how long an entry is considered fresh. Expiration may be enforced lazily on access or actively by background cleanup; those mechanisms can have different memory and timing behavior.

### Core mechanism

- **TTL** is often a duration from insertion or refresh.
- **Absolute expiration** is a specific deadline.
- **Idle expiration** extends lifetime when an entry is accessed; without a hard maximum age, hot data can remain indefinitely.
- Expired entries may be removed later than their logical expiry, depending on the implementation.

### Engineering blueprint

Choose expiry from an explicit maximum-staleness target and refresh behavior:

```text
soft_ttl = refresh eligibility deadline
hard_ttl = latest time at which the value may be served
serve_stale = allowed only when age <= hard_ttl and origin policy permits it
```

Randomize TTLs within a small, intentional range when many entries are written together. Keep the randomization bounded by the allowed freshness window.

### Production pitfalls

- A TTL does not guarantee an entry is refreshed before it expires.
- A long TTL can preserve stale or improperly scoped data.
- A short TTL can cause excessive misses, origin cost, and latency.
- Applying a new TTL on every read can accidentally turn a bounded cache into an indefinitely fresh one.

### SDE best practices

- Set TTL per data class, not as one global constant.
- Combine TTL with event invalidation where faster freshness is required.
- Test boundary behavior and clock assumptions.
- Keep stale-serving behavior separate from cache expiry and document its upper bound.

### Resources

- [Redis: EXPIRE command](https://redis.io/docs/latest/commands/expire/)
- [RFC 9111: HTTP Caching, freshness](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.2)

---

## 8. Eviction Policies

Eviction is capacity-driven removal of entries before their TTL expires. It is different from expiration: eviction says the cache needs room, not that the value is semantically stale.

### Core mechanism

Common policies include:

| Policy | Approximate behavior | Trade-off |
| --- | --- | --- |
| LRU | Evict least recently used | Works when recency predicts reuse; metadata costs apply |
| LFU | Evict least frequently used | Preserves popular values; counters need aging/approximation |
| FIFO | Evict oldest inserted | Simple; ignores access popularity |
| Random | Evict arbitrary entries | Simple and sometimes robust for uniform access |
| TTL-oriented | Prefer entries near expiry | Can reduce waste but does not reflect value |

Real cache products often use approximate or segmented variants rather than exact textbook algorithms.

### Engineering blueprint

- Set memory limits and observe actual memory overhead, not just serialized payload size.
- Understand whether the selected policy evicts keys, rejects new writes, or applies both behaviors.
- Test with a representative key popularity distribution and object size.
- For Redis, select `maxmemory` and an appropriate `maxmemory-policy` for the workload:

| Redis policy | Evicts | Use when |
| --- | --- | --- |
| `noeviction` (default) | Nothing; writes fail when memory is full | Redis holds data that must not silently disappear |
| `allkeys-lru` | Any key, least recently used first | General-purpose cache; a good default |
| `allkeys-lfu` | Any key, least frequently used first | Stable popular set with one-off scans that would pollute LRU |
| `volatile-lru` / `volatile-lfu` | Only keys with a TTL | Mixing cache keys (with TTL) and persistent keys (without) in one instance |
| `volatile-ttl` | Keys with a TTL, shortest remaining TTL first | Expiry time is a good proxy for value |
| `allkeys-random` / `volatile-random` | Random | Uniform access patterns |

A pure cache with `noeviction` left in place will start returning errors on writes when it fills up — a common production surprise.

### Production pitfalls

- A high eviction rate can convert into origin overload and a feedback loop.
- Large values can displace many small hot entries.
- Memory accounting may include indexes, allocator fragmentation, and replication buffers.
- A policy mismatch can evict hot keys or retain useless cold entries.

### SDE best practices

- Monitor used memory, fragmentation, evictions, rejected writes, and miss rate together.
- Load-test with skewed/hot-key traffic and realistic object sizes.
- Avoid treating cache eviction as data loss if the origin is authoritative.
- Reserve headroom for traffic spikes, replication, and maintenance operations.

### Resources

- [Redis: Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Caffeine: Efficiency and eviction](https://github.com/ben-manes/caffeine/wiki/Efficiency)

---

## 9. Cache Invalidation

Invalidation removes or updates a cached copy after its source changes. It is difficult because the write and cache are usually separate systems without a shared transaction.

### Core mechanism

Common approaches:

- **Delete-on-write:** commit the origin, then delete the affected key.
- **Update-on-write:** write the new value to the cache after the origin commits.
- **Event-based invalidation:** publish a change event and invalidate consumers asynchronously.
- **Versioned keys:** include a generation/version in keys so old values become unreachable.
- **Short TTL fallback:** let stale entries age out if an invalidation is missed.

### Engineering blueprint

The classic delete-on-write race:

```text
Reader: cache miss -> reads old database value
Writer: commits new value -> deletes cache key
Reader: stores old value in cache
```

Possible mitigations include per-key coordination, version checks, delayed invalidation, compare-and-set writes, or versioned keys. Each has costs; choose based on the staleness budget and write/read concurrency.

Two practical mitigations:

```text
Delayed double delete:
  writer: commit DB -> delete key -> wait ~ (longest read+fill time, e.g. 500 ms) -> delete key again
  The second delete removes a stale value that a racing reader stored in between.
  Cheap, but it narrows the window instead of closing it.

Versioned keys (generation counter):
  read:  gen = cache.get("gen:product:42") or 0
         value = cache.get(f"product:42:g{gen}")
  write: commit DB -> cache.incr("gen:product:42")
  Old entries become unreachable immediately and expire through TTL.
  Costs one extra lookup per read; useful for invalidating whole groups (e.g. "all of tenant 7").
```

For tight bounds, store the source row's version or `updated_at` with the cached value and only overwrite when the new version is greater (for example, with a Redis Lua script that compares versions atomically).

### Production pitfalls

- Invalidating only one representation misses derived lists, aggregates, and related entities.
- Event delivery can be delayed, duplicated, reordered, or lost unless the pipeline handles these cases.
- A database commit followed by process failure can skip an in-process invalidation.
- Broad flushes can create a sudden cold-cache stampede.

### SDE best practices

- Maintain an explicit map from mutations to affected cache keys or tags.
- Use an outbox or durable event stream when invalidation must survive process failure.
- Make invalidation idempotent and safe to replay.
- Avoid global flushes for routine deploys or data changes.

### Resources

- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [Redis: Client-side caching and invalidation](https://redis.io/docs/latest/develop/clients/client-side-caching/)

---

## 10. Consistency and Staleness

Consistency describes how cached copies relate to the authoritative value over time. A cache may be strongly coordinated, eventually updated, bounded-stale, or explicitly best-effort; “consistent” should be defined for a read and write path.

### Core mechanism

- **Read-after-write:** a client sees its own successful write on subsequent reads.
- **Eventual consistency:** replicas/caches converge if updates stop and propagation succeeds.
- **Bounded staleness:** data may lag by a defined maximum under stated assumptions.
- **Strong consistency:** reads observe the latest committed value according to the system's consistency model; this often reduces cache independence or availability.

### Engineering blueprint

For each endpoint or data class, record:

| Requirement | Example question |
| --- | --- |
| Maximum age | How old may a product price be? |
| Read-after-write | Must the writer immediately see the update? |
| Failure mode | Is stale data safer than an error? |
| Scope | Does the requirement apply across regions and clients? |
| Enforcement | What mechanism and metric prove the bound? |

Consider bypassing cache or using a version/session token for reads that require read-after-write behavior.

### Production pitfalls

- A finite TTL alone cannot bound staleness if refresh repeatedly fails and stale data is served.
- Multiple layers (browser, CDN, app, database replica) compound freshness delays.
- Replica lag plus cache TTL creates more staleness than either layer alone.
- Product teams may assume stronger guarantees than the implementation provides.

### SDE best practices

- Express freshness as a measurable service-level objective.
- Document whether stale-on-error is allowed and the maximum stale age.
- Include source version/timestamp when useful for detecting old results.
- Build tests around concurrent reads and writes, not only sequential behavior.

### Resources

- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)

---

## 11. Cache Stampedes and Miss Coalescing

A stampede occurs when many requests miss or refresh the same key at once, causing duplicate expensive work and possibly overwhelming the origin.

### Core mechanism

- **Single-flight / request coalescing:** one loader runs per key; concurrent callers await the same result.
- **Jitter:** spreads expirations or refreshes over time.
- **Stale-while-revalidate:** serve an acceptable stale value while one refresh runs.
- **Probabilistic early refresh:** increasingly likely to refresh as an entry approaches expiry.
- **Distributed lock:** coordinates loaders across processes, but introduces lock ownership, lease, and failure concerns.

### Engineering blueprint

For a loader:

1. Check the cache.
2. On miss, join an existing per-key load if present.
3. Otherwise acquire a bounded loader slot and fetch from origin.
4. Populate only if the fetched version is not older than the currently stored version.
5. Release coordination on success, error, timeout, and cancellation.

Never let an unbounded number of waiters or origin calls accumulate.

In-process single-flight in Go:

```go
var group singleflight.Group

func GetUser(ctx context.Context, id string) (*User, error) {
    if u, ok := cache.Get(id); ok {
        return u, nil
    }
    // All concurrent callers for the same id share one database call.
    v, err, _ := group.Do(id, func() (any, error) {
        u, err := db.LoadUser(ctx, id)
        if err == nil {
            cache.Set(id, u, 5*time.Minute)
        }
        return u, err
    })
    if err != nil {
        return nil, err
    }
    return v.(*User), nil
}
```

Across replicas, a short-lived Redis lock (`SET lock:key <token> NX PX 5000`) can elect one loader per key; other callers briefly wait and re-check the cache, or serve stale.

Probabilistic early expiration ("XFetch", Vattani et al.) avoids locks entirely. Each reader independently decides to refresh early, with a probability that rises as expiry approaches:

```text
# delta = how long the last recompute took; beta = 1.0 by default (> 1 refreshes earlier)
if now - delta * beta * ln(random()) >= expiry:     # ln(random()) is negative, so this adds time
    recompute and reset the entry
else:
    serve cached value
```

With many readers, one of them usually refreshes slightly before expiry, so the key never goes cold under load.

### Production pitfalls

- A process-local lock does not coordinate across application replicas.
- A distributed lock can expire while its owner is still working; fencing/version checks may be needed.
- Serving stale indefinitely hides refresh failures.
- Retry-on-miss combined with retry-on-origin-error can amplify an outage.

### SDE best practices

- Prefer single-flight and jitter before adding distributed locking.
- Put explicit limits on concurrent origin loads and waiting callers.
- Use stale serving only with a hard freshness bound.
- Exercise expiry bursts and origin slowness in load tests.

### Resources

- [Cloudflare: Sometimes I cache](https://blog.cloudflare.com/sometimes-i-cache/) (request coalescing and cache behavior)
- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
- [Go: `singleflight` package](https://pkg.go.dev/golang.org/x/sync/singleflight)
- [Vattani, Chierichetti, Lowenstein: Optimal Probabilistic Cache Stampede Prevention](https://cseweb.ucsd.edu/~avattani/papers/cache_stampede.pdf)
- [Scaling Memcache at Facebook (NSDI 2013)](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf) (leases for stampedes and stale sets)

---

## 12. Negative Caching

Negative caching stores the fact that a lookup had no result (or returned a cacheable error) so repeated requests do not repeatedly hit the origin.

### Core mechanism

- Represent negative results distinctly from cache misses; do not overload `null` if it is also a valid result.
- Use a shorter TTL than for positive results when the entity may be created soon.
- Decide which failures are cacheable: authoritative `not found` may be; timeout, 5xx, and permission failures usually should not be.

### Engineering blueprint

Define a typed result such as:

```text
LookupResult =
  Found(value, version)
  | NotFound
```

Cache `NotFound` only after an authoritative lookup and invalidate it when a create operation succeeds. Keep tenant and authorization scope in the key where the result depends on either.

### Production pitfalls

- Caching transient errors can turn a short outage into a prolonged outage.
- Negative results can become stale after object creation.
- An attacker can generate many unique misses and consume cache capacity (cache pollution).
- Caching authorization denials can leak results across users if the key omits identity or policy context.

### SDE best practices

- Use explicit result types and short, reasoned TTLs.
- Apply admission controls or quotas for attacker-controlled lookup keys.
- Invalidate negative entries on create or permission changes.
- Do not cache transient infrastructure failures as “not found.”

### Resources

- [RFC 2308: Negative Caching of DNS Queries](https://www.rfc-editor.org/rfc/rfc2308.html)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)

---

## 13. Cache-Key Design and Namespacing

A cache key identifies one exact result. If an input that changes the result is omitted, callers can receive incorrect data or data belonging to another user or tenant.

### Core mechanism

A conceptual key is a deterministic encoding of all result-affecting inputs:

```text
v3:tenant:{tenantId}:user:{userId}:locale:{locale}:product:{productId}
```

Potential key dimensions include entity ID, tenant, locale, currency, feature/configuration version, authorization scope, query filters, and representation version.

### Engineering blueprint

- Normalize values before encoding (case, Unicode, defaults, and ordering).
- Use structured encoding or unambiguous separators/length prefixes.
- Hash very long or sensitive inputs when appropriate, while retaining enough observability to debug safely.
- Version key schemas when serialization or result semantics change.
- Set limits on key length and user-controlled dimensions.

### Production pitfalls

- Omitting tenant or authorization context creates cross-tenant or cross-user disclosure.
- Different strings can collide if separators or normalization are ambiguous.
- High-cardinality query keys can exhaust memory with low reuse.
- Including irrelevant parameters fragments the cache and lowers hit rate.

### SDE best practices

- Centralize key construction; do not hand-build keys in multiple services.
- Document key dimensions and invalidation ownership next to the cached data model.
- Treat cache keys as untrusted input when they come from clients.
- Add tests that vary each result-affecting dimension and verify isolation.

### Resources

- [Redis: Keyspace and key design](https://redis.io/docs/latest/develop/using-commands/keyspace/)
- [AWS Whitepaper: SaaS tenant isolation strategies](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/saas-tenant-isolation-strategies.html)

---

## 14. HTTP Caching Semantics

HTTP caching uses response metadata and request directives to determine whether a response may be stored, reused, validated, or must be fetched again. It applies to browsers, proxies, and CDNs, not only application caches.

### Core mechanism

Important response directives:

| Directive | Meaning (simplified) |
| --- | --- |
| `Cache-Control: max-age=N` | Fresh for N seconds in any cache (browser, proxy, CDN) |
| `s-maxage=N` | Freshness lifetime for shared caches only; overrides `max-age` there |
| `no-store` | Do not store the response |
| `no-cache` | May store, but must validate before reuse |
| `private` | Shared caches must not store this response |
| `public` | Explicitly permits shared-cache storage where otherwise restricted |
| `must-revalidate` | Do not reuse stale response without successful validation |
| `stale-while-revalidate=N` | May serve stale while asynchronously revalidating, within the extension's window |
| `stale-if-error=N` | May serve stale for up to N seconds if the origin returns an error or is unreachable |
| `immutable` | The body will never change during its freshness lifetime; skip revalidation even on reload |

Validators such as `ETag` and `Last-Modified` allow conditional requests (`If-None-Match`, `If-Modified-Since`) and `304 Not Modified` responses.

Typical header sets:

```http
# Fingerprinted static asset: /static/app.3f9a1c.js
Cache-Control: public, max-age=31536000, immutable

# Public API data that can be a minute stale; CDN may keep it longer and serve stale on errors
Cache-Control: public, max-age=60, s-maxage=300, stale-while-revalidate=30, stale-if-error=86400

# User-specific page: browser may store but must check with the server each time
Cache-Control: private, no-cache
ETag: "u42-v19"

# Sensitive response (bank statement, token endpoint)
Cache-Control: no-store
```

### Engineering blueprint

For every cacheable route:

- Define whether the response is public, private, or non-storable.
- Set freshness and revalidation behavior intentionally.
- Include `Vary` for request headers that affect the representation, when appropriate.
- Ensure validators represent the selected representation and change when it changes.
- Verify behavior through the full proxy/CDN chain.

### Production pitfalls

- `no-cache` does not mean “do not store”; `no-store` is the directive for that intent.
- Missing or incorrect `Vary` can mix compressed, localized, or content-negotiated responses.
- A shared cache can expose personalized content if the response is marked public or the key omits variation.
- `Set-Cookie` behavior and shared-cache rules depend on directives and cache implementation; do not rely on accidental defaults.

### SDE best practices

- Use `private, no-cache` when storage is acceptable but every reuse must validate; use `no-store` for sensitive responses that must not be stored.
- Keep cache directives consistent across success and error responses.
- Test freshness, conditional requests, and `Vary` with real clients or a standards-aware test harness.
- Treat RFC behavior as baseline and confirm vendor-specific cache configuration.

### Resources

- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 5861: `stale-while-revalidate` and `stale-if-error`](https://www.rfc-editor.org/rfc/rfc5861.html)
- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)

---

## 15. CDNs and Shared Edge Caches

A content delivery network (CDN) caches content at geographically distributed edge locations. It reduces origin traffic and latency for reusable content, but introduces a shared trust boundary and another invalidation layer.

### Core mechanism

- A cache key selects an object at an edge point of presence.
- Freshness directives, vendor configuration, and purge APIs determine reuse and removal.
- Requests may be shielded through an intermediate cache before reaching the origin.
- Cache hit behavior can vary by location and key distribution.

### Engineering blueprint

- Start with static, versioned assets and clearly public responses.
- Define the edge cache key explicitly: host, path, query parameters, and selected headers.
- Keep cookies and authorization headers out of cache variation unless intentionally required.
- Use immutable, content-hashed asset names for long-lived static resources.
- Choose targeted purge/tag invalidation where available and establish propagation expectations.

### Production pitfalls

- A misconfigured shared key can serve one user's response to another.
- Purges are often eventually propagated and should not be treated as a globally atomic operation.
- Query-string normalization or ignored parameters can cause incorrect cache sharing.
- A cold or purged CDN can shift substantial load to the origin.

### SDE best practices

- Test from multiple edge regions and inspect response headers such as `Age` and vendor cache status.
- Protect the origin from direct traffic if CDN traversal is part of the capacity plan.
- Keep a purge/runbook and a fallback plan for origin load.
- Do not cache authenticated or tenant-specific responses at a shared edge without a reviewed key and policy.

### Resources

- [Cloudflare: Cache concepts](https://developers.cloudflare.com/cache/concepts/)
- [AWS CloudFront: How CloudFront works with HTTP headers and caching](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/RequestAndResponseBehaviorCustomOrigin.html)
- [RFC 9111: Shared caches](https://www.rfc-editor.org/rfc/rfc9111.html)

---

## 16. Browser and Client-Side Caching

Browsers and client applications cache responses, assets, and application data to reduce network work. Client-side copies are outside the server's immediate control and may outlive a user's session or permission changes.

### Core mechanism

- HTTP cache directives govern browser HTTP caching.
- Service workers can implement application-controlled caching strategies.
- Mobile and desktop clients may keep additional local data stores.
- Cache storage, memory cache, and disk cache can have different lifetimes.

### Engineering blueprint

- Separate public static assets from user-specific API data.
- Give content-hashed assets long immutable lifetimes; deploy new content under a new URL.
- Avoid persisting sensitive API responses unless the product needs it and has secure storage, logout, and revocation behavior.
- On logout/account switch, clear application-managed caches and isolate them by account where appropriate.
- Use service-worker versioning and migration logic for old cached assets.

### Production pitfalls

- A server-side permission revocation does not erase a response already stored on a device.
- Service workers can continue serving an old application shell after deployment.
- Shared-device browser profiles can retain personal information.
- `no-cache` requires revalidation; it does not guarantee that the body is absent from local storage.

### SDE best practices

- Make data classification drive client caching policy.
- Test first install, update, offline, logout, and account-switch flows.
- Provide a way to invalidate or version application-managed cached content.
- Never rely on client-side cache clearing as the sole authorization control; enforce access at the server.

### Resources

- [MDN: HTTP caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [MDN: Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [web.dev: Service worker caching](https://web.dev/learn/pwa/caching/)

---

## 17. Distributed Caches

A distributed cache is accessed by multiple application instances, usually over a network. It provides shared cache state across a fleet but adds network latency, availability, partitioning, and capacity concerns.

### Core mechanism

- A client routes an operation to a cache node or cluster.
- Data may be replicated, partitioned, or both.
- A network failure can make the cache slower or unavailable while the origin remains healthy.
- Cluster topology and failover behavior affect latency and consistency.

### Engineering blueprint

- Set connection, read, and write timeouts shorter than the request's remaining deadline.
- Bound connection pools and concurrent operations.
- Decide whether cache failure should bypass to origin, serve a local copy, or fail the request.
- Protect the origin with concurrency limits and load shedding if bypass is allowed.
- Understand replica consistency, failover, and data-loss guarantees.

### Production pitfalls

- A cache outage can become an origin outage if every application instance falls back at once.
- Slow cache calls can consume request threads and connection pools.
- Cross-zone or cross-region network cost can erase the expected latency improvement.
- Failover may return missing, stale, or differently distributed keys.
- **Hot keys:** one very popular key (a celebrity profile, a flash-sale item) lives on one shard, so that shard saturates while others idle. Mitigate with a short-TTL in-process near-cache for that key, or by writing replicas `key#0..key#N-1` and reading a random one.

### SDE best practices

- Use circuit breakers and bounded fallback; never let fallback be unbounded.
- Capacity-test both normal operation and cache-loss scenarios.
- Consider a small local near-cache only with explicit invalidation and staleness rules.
- Keep cache traffic isolated from critical database connection and thread pools.

### Resources

- [AWS: Caching best practices](https://aws.amazon.com/caching/best-practices/)
- [Redis: Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [AWS Builders' Library: Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)

---

## 18. In-Process Caches

An in-process cache stores values in an application's own memory. It offers very low lookup latency and avoids a network hop, but every process has a separate copy and capacity is tied to the process.

### Core mechanism

- Each application instance maintains its own entries and eviction state.
- Instances can observe different values until TTL expiry or invalidation propagation.
- A restart clears volatile entries, causing cold-start misses.

### Engineering blueprint

- Set both entry count and memory limits where available.
- Configure per-entry TTL and record-size limits.
- Choose a policy that fits actual key reuse (often bounded LRU/LFU).
- Avoid caching mutable objects that can be modified by callers after insertion.
- Coordinate invalidation across instances if freshness requires it.

### Production pitfalls

- Per-instance caches can multiply memory usage by fleet size.
- Autoscaling and restarts repeatedly create cold caches.
- A local hit rate can hide poor global hit rate because each instance has a separate working set.
- Unbounded maps can cause memory pressure or process failure.

### SDE best practices

- Prefer a tested bounded cache library over a hand-built map.
- Export per-instance and fleet-level metrics.
- Keep cache values immutable or defensively copied.
- Use local caches for small, stable, high-reuse data, not as an accidental source of truth.

### Resources

- [Caffeine: A high-performance caching library for Java](https://github.com/ben-manes/caffeine)
- [Go: `sync.Map`](https://pkg.go.dev/sync#Map) (concurrent map; not a bounded cache)
- [Microsoft: Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)

---

## 19. Redis as a Cache

Redis is an in-memory data store often used for shared caching, counters, coordination, and ephemeral state. It provides more than simple key/value get/set operations, so its memory, persistence, eviction, and data-structure semantics must be understood.

### Core mechanism

- Keys may expire by TTL and be removed according to configured memory policy.
- Clients communicate over a network and must handle timeouts, reconnects, and partial failures.
- Persistence, replication, and clustering are configurable; none makes every Redis deployment a durable database by default.
- Client-side caching/tracking can reduce repeated network reads but requires invalidation handling.
- Licensing note: Redis moved from BSD to source-available licenses in 2024 (version 7.4) and added AGPLv3 as an option in Redis 8 (2025). The Linux Foundation fork **Valkey** stays BSD-licensed and is protocol-compatible; several cloud providers' managed offerings now use it. Commands in this section apply to both.

Commands worth knowing for caching:

| Command | Use |
| --- | --- |
| `SET key value EX 300` | Store with a TTL in one atomic step (prefer over `SET` then `EXPIRE`) |
| `SET lock:key token NX PX 5000` | Set only if absent, with a millisecond TTL; basis for a simple loader lock |
| `MGET k1 k2 k3` | Read many keys in one round trip |
| `UNLINK key` | Delete, freeing memory in the background (non-blocking for large values) |
| `SCAN 0 MATCH v1:product:* COUNT 1000` | Iterate keys incrementally; use instead of `KEYS` in production |
| `TTL key` | Remaining lifetime in seconds (`-1` = no expiry, `-2` = missing) |
| `INFO stats` / `INFO memory` | Hit/miss counters (`keyspace_hits`, `keyspace_misses`), evictions, memory use |
| `redis-cli --bigkeys` / `--hotkeys` | Find oversized keys and popular keys (hotkeys requires an LFU policy) |

Use pipelining (sending many commands without waiting for each reply) when issuing many independent commands; it removes per-command round-trip latency.

### Engineering blueprint

- Configure memory limits, eviction policy, authentication, TLS, and network exposure.
- Use atomic commands/scripts for multi-step cache operations where necessary.
- Apply TTLs to transient entries and use bounded key/value sizes.
- Separate cache workloads from durable or latency-sensitive workloads when noisy-neighbor effects matter.
- Handle `MOVED`/cluster routing, failover, and reconnect behavior using a maintained client.

### Production pitfalls

- `KEYS` and other blocking/expensive operations can harm production latency; use appropriate scan patterns.
- A single large key or unbounded data structure can cause memory and latency problems.
- Persistence or replication settings can create latency and operational costs.
- Misconfigured eviction can reject writes or evict unexpected keys.
- Cache flush commands can cause sudden origin load spikes.

### SDE best practices

- Use explicit namespacing and schema/version conventions.
- Monitor memory, fragmentation, evictions, rejected writes, command latency, network, replication lag, and connection saturation.
- Test behavior during node failover and cache cold start.
- Avoid storing irreplaceable data in a cache deployment whose durability contract does not meet the requirement.

### Resources

- [Redis documentation](https://redis.io/docs/latest/)
- [Redis: Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis: Key expiration](https://redis.io/docs/latest/commands/expire/)
- [Redis: Client-side caching](https://redis.io/docs/latest/develop/clients/client-side-caching/)
- [Redis: SET command options](https://redis.io/docs/latest/commands/set/)
- [Valkey](https://valkey.io/)

---

## 20. Cache Security and Isolation

Caches can expose data across users or tenants when keys, HTTP directives, or invalidation rules do not preserve security boundaries. Cached data should be treated with the same classification as its source.

### Core mechanism

- Cache keys and namespaces define who can retrieve an entry.
- Shared HTTP caches may reuse responses across requests and users.
- Cache contents can remain after logout, permission changes, or tenant membership changes.
- Cache interfaces may be reachable over a network and require normal authentication and transport security.

### Engineering blueprint

- Include tenant, user, role/policy version, and representation dimensions when they affect the result.
- Use `private` or `no-store` for user-specific HTTP responses as required by the data policy.
- Do not trust a client-supplied cache key as authorization.
- Protect cache endpoints with private network placement, authentication, TLS, and least privilege.
- Avoid putting credentials, tokens, or sensitive personal data in keys, logs, or metric labels.

### Production pitfalls

- Cache-key omission causes cross-tenant leakage (a cache form of broken access control).
- Cached authorization decisions can outlive role revocation.
- Cache poisoning can occur when untrusted request fields influence a shared cache key or response.
- Debug logs and metrics can leak keys or values that contain sensitive data.

### SDE best practices

- Perform authorization independently of cache lookup unless the cached object is explicitly a safely scoped authorization result.
- Version or invalidate entries when permissions change.
- Add cross-user and cross-tenant isolation tests for every shared cache layer.
- Apply data retention and deletion requirements to cache copies and backups.

### Resources

- [OWASP Web Security Testing Guide: Testing for Web Cache Poisoning](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/17-Testing_for_Web_Cache_Poisoning)
- [RFC 9111: Caching and authenticated responses](https://www.rfc-editor.org/rfc/rfc9111.html)
- [AWS Whitepaper: SaaS tenant isolation strategies](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/saas-tenant-isolation-strategies.html)

---

## 21. Production Checklist and Cheat Sheet

### Cache design record

| Field | Record for every important cache |
| --- | --- |
| Source of truth | Which system owns the durable value? |
| Data classification | Public, internal, confidential, or regulated? |
| Cache owner | Which service reads, writes, invalidates, and monitors entries? |
| Key schema | Which inputs affect the result, tenant, or permission scope? |
| Freshness | Soft TTL, hard TTL, invalidation path, and allowed stale age |
| Miss behavior | Loader, timeout, concurrency limit, negative-cache policy |
| Failure behavior | Bypass, stale response, or error; fallback limits |
| Capacity | Maximum memory, entry size, cardinality, and eviction policy |
| Operations | Dashboards, alerts, runbook, cold-start and outage test |

### Pre-launch checklist

**Correctness and freshness**
- [ ] Source of truth and cache ownership are documented.
- [ ] Maximum acceptable staleness is defined for each cached data class.
- [ ] Key includes every result-affecting field, including tenant and authorization scope.
- [ ] Write paths invalidate/update all affected representations.
- [ ] Invalidation is safe to retry and recover after process failure.
- [ ] Cache outage, stale data, and origin timeout behavior are explicit.
- [ ] Negative entries distinguish not-found from transient errors.

**Capacity and resilience**
- [ ] Memory and key cardinality are bounded.
- [ ] TTLs are deliberate and synchronized expiry bursts are mitigated.
- [ ] Hot-key misses are coalesced; origin concurrency and retries are bounded.
- [ ] Cache-loss behavior has been load-tested against origin capacity.
- [ ] Connection pools, deadlines, and circuit breakers are configured.
- [ ] Evictions, rejected writes, fragmentation, and cache errors are monitored.

**HTTP and edge**
- [ ] Public, private, and non-storable responses use correct `Cache-Control`.
- [ ] `Vary`, `ETag`, and conditional request behavior have been tested.
- [ ] CDN cache key and query-string/header normalization are reviewed.
- [ ] Authenticated responses cannot be shared across users or tenants.
- [ ] Asset versioning and purge procedures are documented.

**Security and operations**
- [ ] Cache network access uses least privilege and encryption where required.
- [ ] Sensitive values do not appear in keys, logs, or metric labels.
- [ ] Logout, account switching, permission changes, and deletion are considered.
- [ ] Dashboards include hit/miss ratio, latency, origin load, errors, evictions, and staleness.
- [ ] Cold start, invalidation race, stampede, failover, and outage drills are covered.

### Useful metrics

| Metric | What it can reveal |
| --- | --- |
| Hit ratio (requests and bytes) | Reuse, though not correctness |
| Hit/miss latency p50/p95/p99 | Network or cache saturation |
| Origin QPS and latency | Whether cache reduces real backend work |
| Evictions / rejected writes | Capacity or policy mismatch |
| Refresh and invalidation lag | Freshness risk |
| Loader concurrency / coalesced waiters | Stampede control effectiveness |
| Cache errors and timeout rate | Dependency health |
| Stale responses served | Trade-off used during origin trouble |
| Key cardinality and top keys | Pollution, leaks, or hot-key imbalance |

### Quick glossary

| Term | One line |
| --- | --- |
| Origin | System from which a cache obtains a response or value |
| Cache hit / miss | Entry reused / entry unavailable and needs another path |
| Cache-aside | Application checks cache, then loads and fills it on miss |
| Read-through | Cache abstraction invokes a loader on a miss |
| Write-through | Write updates cache and backing store in one logical path |
| Write-behind | Cache acknowledges and persists the write asynchronously |
| TTL | Duration an entry is considered valid after a defined event |
| Eviction | Capacity-driven removal of a still-unexpired entry |
| Invalidation | Removal or replacement after source data changes |
| Stampede | Concurrent misses or refreshes overload the origin |
| Negative caching | Caching an authoritative absence/failure result |
| `ETag` | HTTP validator identifying a representation version |
| `Vary` | HTTP response metadata that adds request headers to cache selection |
| Near-cache | Small, often in-process cache in front of a shared cache |
| Stale-while-revalidate | Serve an allowed stale response while refreshing asynchronously |

---

## 22. Self-Check Questions

Try to answer each question before opening the answer.

<details>
<summary>1. On a write, should you update the cache or delete the key? Why?</summary>

Usually delete. Two concurrent writers that each set the cache can finish in a different order from their database commits and leave the older value cached. Deleting forces the next reader to load the committed value. Update-on-write is reasonable only with version checks.
</details>

<details>
<summary>2. Describe the race that can leave stale data in cache-aside even when you delete on write.</summary>

A reader misses and reads the old value from the database. A writer commits the new value and deletes the key. The reader then stores the old value it read. The cache now holds stale data until TTL expiry. Mitigate with a finite TTL, delayed double delete, versioned keys, or version-checked writes.
</details>

<details>
<summary>3. What is a cache stampede, and name three ways to prevent it.</summary>

Many requests miss the same popular key at once (often right after expiry) and all hit the origin. Prevent with single-flight / request coalescing, jittered TTLs, stale-while-revalidate, probabilistic early refresh (XFetch), or a short-lived loader lock.
</details>

<details>
<summary>4. What is the difference between expiration and eviction?</summary>

Expiration removes an entry because its TTL passed: the data is considered too old. Eviction removes an entry before its TTL because the cache needs memory: the data may still be fresh.
</details>

<details>
<summary>5. What does Cache-Control: no-cache mean? How is it different from no-store?</summary>

`no-cache` allows storing the response but requires revalidation with the origin before every reuse. `no-store` forbids storing it at all. Use `no-store` for sensitive data.
</details>

<details>
<summary>6. A CDN served one user's account page to another user. What went wrong and how do you prevent it?</summary>

A personalized response was stored in a shared cache, usually because it was marked `public` (or had no directives and the CDN was configured to cache it) and the cache key did not include the user. Mark user-specific responses `private` or `no-store`, keep authenticated routes out of CDN caching rules, and test with two users.
</details>

<details>
<summary>7. Why should negative caching use a shorter TTL, and what should never be negatively cached?</summary>

The missing object may be created soon, and a long negative TTL hides it. Never cache transient failures (timeouts, 5xx, connection errors) as "not found": that turns a brief outage into a long one.
</details>

<details>
<summary>8. Your Redis cache is full and writes start failing with OOM errors. What is the likely configuration problem?</summary>

`maxmemory-policy` is still the default `noeviction`. For a pure cache, set `allkeys-lru` or `allkeys-lfu` so Redis evicts instead of rejecting writes.
</details>

<details>
<summary>9. When is write-behind caching dangerous?</summary>

When the cache acknowledges a write before it is durable in the source of truth. A cache node failure can lose acknowledged writes. Avoid it for money, inventory, and other critical data unless a durable log and recovery process exist.
</details>

<details>
<summary>10. Hit rate is 98%, but the database still falls over during deploys. Why?</summary>

Deploys restart in-process caches or flush shared ones, so the hit rate drops to near zero at once and the full request load hits the database (cold start). Warm caches before shifting traffic, roll out gradually, avoid global flushes, and size the origin (or limit concurrency to it) for the cache-loss case.
</details>

<details>
<summary>11. What must a cache key include in a multi-tenant system?</summary>

Every input that changes the result: tenant ID, and user or permission scope when results differ by caller, plus locale, currency, feature flags, query parameters, and a schema version. Missing the tenant ID is a data-leak bug, not just a correctness bug.
</details>

## 23. Master Resource List

### HTTP standards

| Topic | Link |
| --- | --- |
| HTTP Caching | [RFC 9111](https://www.rfc-editor.org/rfc/rfc9111.html) |
| HTTP Semantics | [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html) |
| `stale-while-revalidate` and `stale-if-error` | [RFC 5861](https://www.rfc-editor.org/rfc/rfc5861.html) |
| Cache-Status response field | [RFC 9211](https://www.rfc-editor.org/rfc/rfc9211.html) |
| Negative DNS caching | [RFC 2308](https://www.rfc-editor.org/rfc/rfc2308.html) |

### Cloud and architecture guidance

| Topic | Link |
| --- | --- |
| Caching challenges and strategies | [AWS Builders' Library](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/) |
| Caching best practices | [AWS](https://aws.amazon.com/caching/best-practices/) |
| Cache-aside pattern | [Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside) |
| Transactional outbox pattern | [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html) |
| SaaS tenant isolation strategies | [AWS Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/saas-tenant-isolation-strategies/saas-tenant-isolation-strategies.html) |

### Product and implementation documentation

| Topic | Link |
| --- | --- |
| Redis documentation | [Redis](https://redis.io/docs/latest/) |
| Redis eviction | [Redis reference](https://redis.io/docs/latest/develop/reference/eviction/) |
| Redis client-side caching | [Redis reference](https://redis.io/docs/latest/develop/clients/client-side-caching/) |
| CloudFront cache behavior | [Amazon CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/RequestAndResponseBehaviorCustomOrigin.html) |
| Cloudflare cache concepts | [Cloudflare Docs](https://developers.cloudflare.com/cache/concepts/) |
| Caffeine Java cache | [Caffeine documentation](https://github.com/ben-manes/caffeine) |
| Go single-flight | [`golang.org/x/sync/singleflight`](https://pkg.go.dev/golang.org/x/sync/singleflight) |

### Learning and security

| Topic | Link |
| --- | --- |
| HTTP caching | [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching) |
| Service Worker API | [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) |
| Service worker caching | [web.dev](https://web.dev/learn/pwa/caching/) |
| Web cache poisoning testing | [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/17-Testing_for_Web_Cache_Poisoning) |