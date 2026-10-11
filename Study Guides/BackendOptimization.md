# Backend Optimization: A Practical Performance Playbook

Audience: SDEs designing, building, and operating production backend systems, APIs, data services, and distributed services.
Scope: 20 core optimization topics, each organized around **Core Mechanism**, **Engineering Blueprint**, **Production Pitfalls**, **SDE Best Practices**, and **Resources**.
Last reviewed: 2026-10-11. Systems and tooling evolve quickly, so validate product- and infra-specific behavior against current benchmarks and vendor docs before locking in a design.

Related guide: [Authentication, Authorization & Modern Security Protocols](./AuthenticationConcepts.md) and [Caching Techniques](./CachingTechniques.md), especially for session, authz, and cache-related bottlenecks.

---

## Table of Contents

0. [System Map: How the 20 Topics Fit Together](#0-system-map-how-the-20-topics-fit-together)
1. [Latency and Throughput](#1-latency-and-throughput)
2. [Bottleneck Analysis](#2-bottleneck-analysis)
3. [Profiling and Tracing](#3-profiling-and-tracing)
4. [Database Query Optimization](#4-database-query-optimization)
5. [Indexing and Query Plans](#5-indexing-and-query-plans)
6. [Connection Pooling and Resource Limits](#6-connection-pooling-and-resource-limits)
7. [Asynchronous Work and Concurrency](#7-asynchronous-work-and-concurrency)
8. [I/O, Serialization, and Payload Design](#8-io-serialization-and-payload-design)
9. [Caching Strategies](#9-caching-strategies)
10. [Read/Write Path Optimization](#10-readwrite-path-optimization)
11. [Batching and Bulk Operations](#11-batching-and-bulk-operations)
12. [Queueing and Backpressure](#12-queueing-and-backpressure)
13. [Horizontal Scaling and Sharding](#13-horizontal-scaling-and-sharding)
14. [Statelessness and Service Boundaries](#14-statelessness-and-service-boundaries)
15. [Load Testing and Capacity Planning](#15-load-testing-and-capacity-planning)
16. [Observability and Performance SLOs](#16-observability-and-performance-slos)
17. [Failure Isolation and Resilience](#17-failure-isolation-and-resilience)
18. [Data Access Hotspots](#18-data-access-hotspots)
19. [Performance Review Workflow](#19-performance-review-workflow)
20. [Production Checklist and Cheat Sheet](#20-production-checklist-and-cheat-sheet)
21. [Self-Check Questions](#21-self-check-questions)
22. [Master Resource List](#22-master-resource-list)

---

## 0. System Map: How the 20 Topics Fit Together

Most backend speed issues are not a single bug. They are a chain of mismatches between request patterns, database behavior, network boundaries, and operational controls.

| Concern | Design question | Relevant concepts |
| --- | --- | --- |
| User perception | How fast does a request feel? | 1, 16, 17 |
| Root cause | Where is time actually spent? | 2, 3, 18 |
| Data access | How expensive is a database round trip? | 4, 5, 6, 10 |
| Work scheduling | Can we decouple or parallelize work? | 7, 11, 12 |
| Payload cost | What is the cost of bytes over the wire? | 8, 9 |
| Scale | How do we handle more load without re-architecting the whole service? | 13, 14, 15 |
| Reliability | What happens when the system is under pressure? | 17, 19 |
| Operations | How do we keep performance in check over time? | 15, 16, 19 |

### Typical request path

```text
Client
  |
  v
API Gateway / Edge
  |
  |-- auth / rate limiting / request validation
  v
Application service
  |-- business logic
  |-- cache reads [9]
  |-- calls to DB / RPC / queue
  v
Database / downstream systems

At each hop, latency is added, queued, retried, or failed.
```

### Decision defaults

| Decision | Sensible starting point |
| --- | --- |
| First optimization target | Measure the slowest high-volume endpoint or query; avoid random micro-optimizations |
| Performance metric | Track p50, p95, p99, saturation, and error rate together |
| Database | Focus on indexes, query shape, and N+1s before changing infrastructure |
| Cache | Add only when a measured hot path benefits from reuse |
| Async work | Use async for I/O-bound tasks and bounded queues for bursty work |
| Scaling | Prefer stateless services; keep state in durable systems with clear ownership |
| Failure | Preserve tail latency and reliability under load with retry budgets, timeouts, and circuit breakers |

### Performance budget

A performance budget is a statement of maximum acceptable cost for a user-facing workflow. For example, "checkout API must return in under 300 ms p95 for 99% of traffic". Without a budget, optimization gets fuzzy and service owners optimize the wrong thing.

---

## 1. Latency and Throughput

Latency is the time to complete one request. Throughput is the amount of work completed per unit of time. They are related, but they are not the same.

### Core mechanism

- Latency is composed of queue time, service compute, network transit, and downstream waiting.
- Throughput is limited by CPU, memory bandwidth, storage IOPS, network capacity, and the shape of the workload.
- When concurrency rises, throughput may improve until a resource becomes saturated; beyond that point, latency explodes and failures increase.
- A backend can be fast on average and still terrible in p99 if a small fraction of requests hit slow paths, retries, or lock contention.

### Engineering blueprint

```text
Request latency = queue + compute + IO + network + retries + downstream delay

Capacity = min(cpu_capacity, network_capacity, db_capacity, cache_capacity)

If saturation > 1.0:
  latency increases sharply
  tail latency worsens
  queue depth grows
```

Measure these directly with a benchmark or trace: p50, p95, p99, CPU percent, heap usage, GC pause time, lock contention, DB response time, and queue depth.

### Little's Law

The most useful formula in backend capacity work:

```text
L = λ × W

L = average number of requests in the system (in flight)
λ = arrival rate (requests per second)
W = average time each request spends in the system (seconds)
```

Worked example: an API receives 500 req/s and each request takes 80 ms on average. In-flight requests = 500 × 0.08 = **40**. If every request holds a DB connection for its full duration, the service needs about 40 connections at steady state. If latency rises to 400 ms because the DB slows down, the same traffic needs 200 concurrent slots. This is why a slow dependency exhausts thread pools and connection pools: latency growth turns directly into concurrency growth.

### Why latency explodes near saturation

For a simple single-server queue, average time in system ≈ `service_time / (1 - utilization)`:

| Utilization | Multiplier on service time |
| --- | --- |
| 50% | 2x |
| 80% | 5x |
| 90% | 10x |
| 95% | 20x |

Real systems differ in detail, but the shape holds: the last 10–20% of utilization costs far more latency than the first 50%. Plan capacity so steady-state utilization of the bottleneck stays well below the knee, commonly 60–70%.

### Tail latency amplification

If one user request fans out to `N` backends in parallel, it is as slow as the slowest one. With each backend having a 1% chance of a slow response, the chance that a request is slow is `1 - 0.99^N`: about 10% at N = 10 and 63% at N = 100. Fan-out turns rare backend slowness into common user-visible slowness.

### Production pitfalls

- Optimizing average latency while ignoring p99 causes user-visible spikes.
- A system can be "efficient" but not "fast" if queueing delays dominate.
- Performance is workload-dependent; a read-heavy cache hit workload is different from a write-heavy transactional workload.
- Throughput numbers are misleading without concurrency and burst assumptions.

### SDE best practices

- Define target SLOs for critical endpoints and background jobs.
- Measure before and after every meaningful change.
- Separate steady-state metrics from burst behavior.
- Favor reducing total work or process time over shaving microseconds from a hot branch that is not on the critical path.

### Resources

- [Google SRE Workbook: Service Level Objectives](https://sre.google/workbook/)
- [Dean and Barroso: The Tail at Scale](https://research.google/pubs/the-tail-at-scale/)
- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)

---

## 2. Bottleneck Analysis

A bottleneck is the resource or process that limits the end-to-end system's throughput or adds the most latency under load.

### Core mechanism

The goal is not to "speed up everything"; it is to identify the limiting factor:

- CPU saturation
- Disk or storage IOPS saturation
- Database lock waits or deadlocks
- Network bandwidth or connection exhaustion
- GC pauses
- External API latency
- Thread or worker starvation
- Serialization overhead

Bottlenecks can be local (one service) or systemic (a shared database or cache cluster).

### Engineering blueprint

Use a systematic loop:

1. Instrument the system.
2. Observe saturated resources.
3. Correlate with request patterns.
4. Compare the cost of retries, queue growth, and wait time.
5. Remove or reduce the limiting chokepoint.

Two standard checklists make step 2 systematic:

- **USE method** (for every resource: CPU, memory, disk, network, pools): check **U**tilization, **S**aturation (queued work), and **E**rrors.
- **RED method** (for every service or endpoint): check **R**ate, **E**rrors, and **D**uration (latency distribution).

USE finds the overloaded resource; RED finds which service or endpoint users are feeling it through.

Example readings:

```text
DB CPU 92%, query latency high, few lock waits
  -> DB is CPU-bound: look for full scans, missing indexes, or expensive queries (pg_stat_statements).
DB CPU 30%, query latency high, many lock waits
  -> contention: long transactions or hot rows, not raw capacity.
Service CPU low, request latency high, pool wait time high
  -> requests are queuing for connections or threads; check the downstream they wait on.
Service queue depth increasing, service CPU low
  -> downstream saturation or throttling, not this service's compute.
```

### Production pitfalls

- Misreading a symptom as a cause. For example, high app latency may be caused by slow DB responses rather than app code.
- Fixing the first bottleneck without checking the next one can create a new one elsewhere.
- Many performance issues are hidden behind retries and timeout storms.
- Cloud metrics often flatten variance; trace-based data is required to see the real bottleneck.

### SDE best practices

- Create a dependency map of critical calls and their latency budgets.
- Investigate full-stack metrics: app, DB, cache, and network.
- Use concurrency testing to reveal overload points.
- Verify the bottleneck under realistic traffic, not just single-threaded benchmarks.

### Resources

- [Brendan Gregg: The USE Method](https://www.brendangregg.com/usemethod.html)
- [Grafana: The RED Method](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)
- [Brendan Gregg: Systems Performance, 2nd edition](https://www.brendangregg.com/sysperfbook.html)

---

## 3. Profiling and Tracing

Profiling answers, "What code is spending time?" Tracing connects a single request across services and infrastructure.

### Core mechanism

- CPU profiles show where a process spends cycles.
- Heap profiles show memory growth and retention.
- Flame graphs make hot path analysis easier.
- Traces show spans for DB calls, cache hits, RPCs, retries, message processing, and external dependencies.

Profiling and tracing are complementary: profiling is a local code view, tracing is a system-wide request view.

### Engineering blueprint

```text
Get request ID -> propagate through services -> capture spans -> correlate with logs, metrics, traces
```

A common production workflow:

- Start from a slow p99 request.
- Find traces with the worst latency.
- Inspect the critical spans and DB calls.
- Capture a CPU or heap profile of the service under the same load.
- Validate the fix with load tests and a repeat trace.

### Production pitfalls

- Sampling too aggressively loses the rare but costly request types.
- Logging every function call adds overhead and hides the actual hotspot.
- Profiling in prod has to be safe; keep memory and CPU overhead under control.
- Tracing without service ownership leads to poor instrumentation and bad data.

### SDE best practices

- Add request IDs and trace propagation early.
- Keep service-level metrics and logs aligned to trace IDs.
- Establish a low-overhead sampling policy with higher sampling for slow or erroring requests.
- Review the flame graph before optimizing code.

### Resources

- [OpenTelemetry](https://opentelemetry.io/)
- [pprof](https://github.com/google/pprof)
- [Brendan Gregg: Flame Graphs](https://www.brendangregg.com/flamegraphs.html)
- [Jaeger](https://www.jaegertracing.io/)

---

## 4. Database Query Optimization

The database is a common bottleneck because it sits on the critical path of reads and writes, often across many services.

### Core mechanism

Database performance is shaped by:

- query shape and execution plan
- table cardinality and join patterns
- index coverage
- lock duration
- transaction isolation level
- row size and blob usage
- memory and I/O pressure

The fastest query is often the one that avoids work entirely: fewer rows, fewer joins, fewer scans, and fewer round trips.

### Engineering blueprint

```text
Bad: N+1 queries in app code
Better: one query with set-based filtering, joins, or a derived table
Best: the query matches the access pattern and the index supports it
```

The N+1 problem in practice:

```python
# N+1: one query for orders, then one query per order for its customer.
orders = db.query("SELECT * FROM orders WHERE created_at > %s LIMIT 100", [since])
for o in orders:
    o.customer = db.query("SELECT * FROM customers WHERE id = %s", [o.customer_id])  # 100 more round trips

# Fixed: two queries total, regardless of how many orders.
orders = db.query("SELECT * FROM orders WHERE created_at > %s LIMIT 100", [since])
ids = list({o.customer_id for o in orders})
customers = {c.id: c for c in db.query("SELECT * FROM customers WHERE id = ANY(%s)", [ids])}
for o in orders:
    o.customer = customers[o.customer_id]
```

At 1 ms per round trip, the first version costs about 100 ms of pure network wait; the second about 2 ms. ORMs hide this: look for "eager loading" options (`select_related`/`prefetch_related` in Django, `JOIN FETCH` in JPA, `include` in Prisma), and in GraphQL use a DataLoader-style batcher.

Reading a query plan (PostgreSQL):

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
```

Look for: `Seq Scan` on a large table (missing or unusable index), a large gap between estimated and actual rows (stale statistics; run `ANALYZE`), a `Sort` node spilling to disk, and nested loops over many rows. To find which queries to examine first, rank by total time in `pg_stat_statements`, not by slowest single execution.

Common techniques:

- reduce round trips
- paginate large result sets
- filter early in the database
- avoid large `SELECT *` in hot paths
- pre-aggregate or denormalize only when the cost/benefit is validated
- use `EXPLAIN` and query plans for each hot query

### Production pitfalls

- Query optimization without realistic data volume is worthless.
- Denormalization can improve reads but worsen writes and consistency.
- Indexes improve reads but may hurt write throughput and storage size.
- Repeated complex query patterns can mask a poor schema or missing constraint.

### SDE best practices

- Treat the schema as a performance contract.
- Measure with production-like data distributions, not just toy datasets.
- Include query latency and row counts in code reviews for critical paths.
- Prefer single, predictable access patterns over ad hoc queries.

### Resources

- [Postgres Query Planning](https://www.postgresql.org/docs/current/using-explain.html)
- [MySQL EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html)
- [MongoDB Performance Best Practices](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)
- [PostgreSQL: pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

---

## 5. Indexing and Query Plans

Indexes do not make queries magically faster; they make the database use the right plan for the data distribution.

### Core mechanism

- A B-tree index supports efficient equality, range, and ordered lookups.
- Composite indexes help when the leading columns match query filters (the **leftmost-prefix rule**): an index on `(a, b, c)` can serve filters on `a`, `a, b`, or `a, b, c`, but not on `b` alone.
- Partial indexes (`WHERE status = 'pending'`) reduce index size for selective subsets.
- A **covering index** contains every column a query needs, so the database can answer from the index without reading the table (an "index-only scan"). PostgreSQL supports `INCLUDE (...)` for extra non-key columns.
- Hash indexes support only equality lookups.
- Every index adds write cost: each insert, and each update to an indexed column, must also update the index.

### Engineering blueprint

```text
SELECT * FROM orders WHERE tenant_id = ? AND status = ? AND created_at > ? ORDER BY created_at DESC LIMIT 20;
```

An effective index often looks like:

```text
CREATE INDEX idx_orders_tenant_status_created
ON orders (tenant_id, status, created_at DESC);
```

Column order rule of thumb: **equality columns first, then the range or sort column**. Here `tenant_id` and `status` are equality filters, and `created_at` serves both the range filter and the `ORDER BY`, so the database can read exactly 20 index entries and stop. If the order were `(created_at, tenant_id, status)`, the range on `created_at` would come first and the database would have to scan every recent order across all tenants.

But a good index depends on actual cardinality, query frequency, and update churn.

### Production pitfalls

- Creating indexes without observation can increase write amplification and disk usage.
- Using a wide composite index that doesn't match the query predicate does not help.
- Columns after the first range condition in a composite index cannot narrow the index seek; they can only filter rows already found.
- Wrapping an indexed column in a function (`WHERE lower(email) = ?`, `WHERE created_at::date = ?`) prevents normal index use unless you create an expression index.
- Low-selectivity indexes (for example, a boolean column where 50% of rows match) are often ignored by the planner.
- Query plans can change after a data skew or a schema change.
- Creating an index on a large live table can lock writes; use `CREATE INDEX CONCURRENTLY` (PostgreSQL) or an online DDL tool.

### SDE best practices

- Start from the actual query pattern, not a generic index recipe.
- Review explain plans before rollout.
- Remove unused indexes; most databases expose index usage statistics.
- Rebuild or revisit indexes as data distribution shifts.

### Resources

- [Postgres Index Types](https://www.postgresql.org/docs/current/indexes.html)
- [Postgres: Index-only scans and covering indexes](https://www.postgresql.org/docs/current/indexes-index-only-scans.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)

---

## 6. Connection Pooling and Resource Limits

A backend service can spend a surprising amount of time waiting on database or downstream connections.

### Core mechanism

Connections cost memory, file descriptors, and thread or process context. If a service creates too many or keeps them unnecessarily, it wastes resources and can trigger cascading failures.

Connection pools guarantee a bounded number of usable connections and reduce the overhead of repeated creation. They are not free; the pool itself is a resource limit that must be tuned.

### Engineering blueprint

Size the pool from two directions and take the smaller answer:

```text
Demand side (Little's Law):
  connections_needed ≈ queries_per_second × average_query_seconds
  e.g. 2,000 qps × 0.005 s = 10 connections busy on average; add headroom for bursts.

Database side (HikariCP's starting heuristic for one database server):
  total_connections ≈ (db_cpu_cores × 2) + effective_spindle_count
  e.g. 8-core DB with SSDs ≈ 16–20 connections in total, across ALL app instances.

Fleet check:
  app_instances × pool_size_per_instance ≤ DB max_connections − admin/reserved headroom
```

The fleet check is the one teams forget: 50 pods × 20 connections = 1,000 connections, which overwhelms most PostgreSQL servers. Under autoscaling, use a connection pooler such as PgBouncer (transaction mode) or a managed proxy between the app fleet and the database.

A smaller pool with a short wait queue usually gives better throughput than a large one: a database with 8 cores cannot run 200 queries at once; it only context-switches between them.

Operational rules:

- Keep request concurrency under the pool limit.
- Set timeout, max lifetime, and idle timeout values deliberately.
- Close or recycle connections before they become stale.
- Protect against connection storms during failover or retries.

### Production pitfalls

- Setting the pool too high starves other resources and produces queueing.
- Setting it too low causes request backpressure and poor throughput.
- Under heavy load, retry storms create a thundering herd of new connections.
- Idle connections can hide the fact that the real problem is concurrency, not pool size only.

### SDE best practices

- Define a service connection budget by workload.
- Keep a headroom buffer for burst traffic and failover.
- Track pool usage, wait time, and checkout latency.
- Use circuit breakers and throttling instead of unlimited retries.

### Resources

- [HikariCP: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [PgBouncer](https://www.pgbouncer.org/)
- [PostgreSQL: Connection settings](https://www.postgresql.org/docs/current/runtime-config-connection.html)

---

## 7. Asynchronous Work and Concurrency

Most backends are a mix of synchronous request handling and asynchronous work. Optimization is easier when the system separates the two cleanly.

### Core mechanism

- Synchronous work blocks request execution and adds latency.
- Asynchronous work decouples long-running or I/O-heavy work from the user request path.
- Concurrency increases throughput but introduces contention, scheduling, and coordination complexity.
- Thread pools, worker queues, and event loops each have different tradeoffs and saturation points.

### Engineering blueprint

```text
Fast path: validate input -> call DB/cache -> respond
Slow path: enqueue work -> ack -> process later -> update state
```

A good async design keeps side effects explicit and observable:

- retries with jitter
- idempotent handlers
- dead-letter queues for poison messages
- bounded concurrency to avoid overload

Bounded fan-out (Python `asyncio`; the same idea applies with a semaphore or worker pool in any language):

```python
sem = asyncio.Semaphore(10)              # at most 10 concurrent calls to the downstream

async def fetch_one(item_id):
    async with sem:
        return await asyncio.wait_for(client.get(item_id), timeout=2.0)

results = await asyncio.gather(*(fetch_one(i) for i in ids), return_exceptions=True)
```

Without the semaphore, a request for 5,000 IDs opens 5,000 concurrent calls and can take down the downstream service.

Choose the concurrency model by workload: I/O-bound work (waiting on networks and databases) benefits from async I/O or many lightweight threads; CPU-bound work is limited by cores, so more concurrency than cores only adds context switching.

### Production pitfalls

- Converting everything to async can hide actual design problems.
- Unbounded fan-out can cause cascade failure.
- Retry loops can amplify load when downstream services are slow or unavailable.
- Event ordering and exactly-once semantics are rarely free.

### SDE best practices

- Keep the request path lean; do expensive work outside the critical path.
- Define the concurrency limit per worker pool and downstream dependency.
- Prefer idempotent operations and deduplication where retries may happen.
- Measure queue depth, age, worker saturation, and processing lag.

### Resources

- [Microsoft: Competing Consumers pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/competing-consumers)
- [Microsoft: Cloud design patterns catalog](https://learn.microsoft.com/en-us/azure/architecture/patterns/)
- [Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781449373320/)

---

## 8. I/O, Serialization, and Payload Design

A backend can be slow even when logic is simple if it moves too much data or serializes poorly.

### Core mechanism

- Large payloads create network and memory overhead.
- Serialization cost scales with object size and field count.
- Unnecessary data in responses wastes CPU and bandwidth.
- Compressing or chunking can help only when the workload matches the strategy.

### Engineering blueprint

```text
Before: return full user + nested relation + metadata
After: return only the required projection for the requested action
```

Patterns to consider:

- field selection / projection
- DTOs instead of raw entities
- gzip or brotli where appropriate
- partial responses for large list views
- batch requests when the client needs several entities

### Production pitfalls

- Returning too much data from a list endpoint multiplies all downstream costs.
- JSON and protobuf are both fast in the right context; the wrong format can become a bottleneck.
- Serializing large arrays in the hot path can dominate CPU time.
- Many payload optimizations are invisible without measuring bytes sent and deserialization costs.

### SDE best practices

- Keep API contracts explicit and minimal.
- Measure bytes-in/bytes-out per endpoint.
- Collapse repeated nested structures when the client does not need them.
- Profile serialization hot spots before adding compression or custom encodings.

### Resources

- [JSON schema / API design references](https://json-schema.org/)
- [Protocol Buffers](https://protobuf.dev/)
- [HTTP compression guidance](https://developer.mozilla.org/en-US/docs/Web/HTTP/Compression)

---

## 9. Caching Strategies

Caching is a performance layer that stores derived or copied values near consumers to avoid repeated work or remote reads.

### Core mechanism

The value is not free. Caching adds complexity around invalidation, freshness, and failure handling.

Common patterns:

- cache-aside
- read-through
- write-through
- write-behind
- refresh-ahead
- negative caching

### Engineering blueprint

```text
Lookup key -> cache hit or miss
If miss: fetch from origin -> populate cache -> return value
If write occurs: invalidate or update the cache and the source of truth
```

The key design matters:

- include tenant or auth scope
- include locale or region if relevant
- include version or sharding dimension
- separate user-specific caches from shared caches

### Production pitfalls

- Cache-stampede when many requests miss simultaneously.
- Eventual consistency windows that surprise product owners.
- Invalidation bugs that silently return stale data.
- Unbounded memory growth or eviction churn.

### SDE best practices

- Cache data only when the miss cost and hit rate justify it.
- Measure hit rate, TTL, and eviction rates.
- Invalidate on writes or use short, explicit TTLs when correctness is harder.
- Fail with explicit, observable behavior during cache outage.

For full coverage of patterns, invalidation races, stampede control, HTTP caching, and Redis, see [CachingTechniques.md](./CachingTechniques.md).

### Resources

- [Caching Challenges and Strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
- [Redis documentation](https://redis.io/docs/)
- [Caching with Envoy / CDN / API gateway docs](https://www.envoyproxy.io/)

---

## 10. Read/Write Path Optimization

The “hot path” often contains a series of expensive operations that are not obviously expensive in isolation.

### Core mechanism

A read path might include:

- authentication check
- authorization decision
- cache lookup
- database read
- serialization
- network response

A write path might include:

- validation
- duplicate checks
- DB transaction
- audit log
- downstream publish
- cache invalidation

Neither path is optimized by default; each must be designed.

### Engineering blueprint

Use a clear distinction between:

- read-mostly paths
- write-heavy transactional paths
- background processing paths

Apply the right optimization pattern to each:

- read-mostly: caches, denormalized views, indexes, partial responses
- write-heavy: batching, reduction, transactional boundaries, async tasks

### Production pitfalls

- Optimizing the read path while ignoring write amplification.
- Moving work from a request path to a background queue without controlling drift or duplicate processing.
- Accepting a “fast enough” read that causes database write contention elsewhere.

### SDE best practices

- Sketch the request lifecycle before optimization.
- Make write operations cheaper without violating correctness.
- Ensure cache invalidation matches the actual write path.
- Separate hot path behavior from expensive, noncritical operations.

### Resources

- [CQRS and read models](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Transactional boundaries design](https://martinfowler.com/eaaCatalog/transactionScript.html)
- [Data-intensive application design principles](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781449373320/)

---

## 11. Batching and Bulk Operations

Many systems pay a large fixed cost per operation, which means the right optimization is often to amortize it over many items.

### Core mechanism

Examples:

- batch inserts into a DB
- bulk updates with one query or a transactional batch
- coalescing many downstream calls
- grouping logs or metrics before flushing

The cost of doing work in batches often depends on the fixed overhead of each DB round trip, RPC, or serialization step.

### Engineering blueprint

```text
If the system does 1000 tiny writes, batch them when:
- the data is naturally grouped
- ordering is not strict
- failure and partial success are handled explicitly
```

Use chunking and deadlines to protect against giant batches.

```sql
-- 1,000 separate statements: 1,000 round trips and 1,000 commits.
INSERT INTO events (user_id, kind) VALUES (1, 'click');
...

-- One multi-row statement per chunk of, say, 500 rows: 2 round trips, 2 commits.
INSERT INTO events (user_id, kind) VALUES (1, 'click'), (2, 'view'), (3, 'click'), ...;

-- Bulk loading (PostgreSQL): COPY is usually much faster than INSERT for large loads.
COPY events (user_id, kind) FROM STDIN WITH (FORMAT csv);
```

The same idea applies outside databases: Redis pipelining, Kafka producer batching (`linger.ms`, `batch.size`), and batch APIs on cloud services all trade a little latency per item for much higher throughput.

### Production pitfalls

- Large batches create timeouts and partial failures.
- Batching can hide queue lag and distort metrics.
- A batch that exceeds memory or network capacity may create unrelated failures.
- Reordering or deduplicating writes can be difficult when the model is not idempotent.

### SDE best practices

- Design batches with predictable maximum sizes.
- Include retry and checkpoint semantics for long-running jobs.
- Track end-to-end batch latency, lag, and failure counts.
- Do not batch work that must respond synchronously to a user.

### Resources

- [PostgreSQL: Populating a database](https://www.postgresql.org/docs/current/populate.html)
- [PostgreSQL: COPY](https://www.postgresql.org/docs/current/sql-copy.html)
- [microservices.io: Messaging pattern](https://microservices.io/patterns/communication-style/messaging.html)

---

## 12. Queueing and Backpressure

Queues smooth traffic bursts and decouple producers from consumers. But they also hide the fact that the system may be overloaded.

### Core mechanism

A queue is a buffer, not a solution. It protects the system from a sudden spike only if:

- the queue is bounded
- consumers keep up
- retries are limited
- consumer backlog is visible
- dead-letter and timeout policies exist

Without these, a queue becomes a backlog that quietly grows until latency or failure becomes permanent.

### Engineering blueprint

```text
Producer -> queue -> worker pool -> downstream
```

Guardrails:

- set queue depth limits
- implement max-concurrency and rate limits
- measure backlog age
- handle poison messages and duplicate delivery
- reject load when the queue is overloaded

Sizing consumers with Little's Law:

```text
Arrival rate: 300 messages/s. Each message takes 200 ms to process.
Busy workers needed = 300 × 0.2 = 60. Run ~80 for headroom.

If only 50 workers run: capacity = 50 / 0.2 = 250 msg/s.
Backlog grows by 50 msg/s = 180,000 messages per hour, and never drains on its own.
```

**Backpressure** means a slow consumer can make producers slow down, instead of letting work pile up without limit. Mechanisms include bounded queues that block or reject when full, `429`/`503` responses with `Retry-After`, credit-based flow control, and pull-based consumers that only fetch what they can process.

**Load shedding** means deliberately rejecting some work early (cheaply) when overloaded, so the work you do accept still meets its latency goal. Rejecting at the edge costs microseconds; timing out after 30 seconds of work wastes all of it.

### Production pitfalls

- Backlogs can become invisible until a downstream failure creates a crisis.
- Unlimited retries and redelivery storms amplify the problem.
- A queue can increase end-to-end latency by moving work out of the user path.
- Message ordering and idempotence assumptions are often false in distributed systems.

### SDE best practices

- Consider queueing only for work that is safe to defer.
- Keep a visible queue-age metric.
- Bound retries and use exponential backoff with jitter.
- Set a clear policy for overload: reject, retry later, or shed work intentionally.

### Resources

- [AWS Builders' Library: Avoiding insurmountable queue backlogs](https://aws.amazon.com/builders-library/avoiding-insurmountable-queue-backlogs/)
- [AWS Builders' Library: Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/)
- [Microsoft: Queue-Based Load Leveling pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling)
- [Kafka documentation](https://kafka.apache.org/documentation/)

---

## 13. Horizontal Scaling and Sharding

A single instance can only do so much. Scaling out distributes traffic and storage, but introduces new complexity.

### Core mechanism

Horizontal scaling adds more nodes that share the same workload. Sharding partitions the data so each node handles a subset of keys.

Good scaling depends on:

- even distribution of keys
- a stable partition key
- clear ownership of data movement and rebalancing
- a predictable hot-key strategy

### Engineering blueprint

```text
hash(user_id) -> shard
route request to shard using consistent hashing or a routing table
```

Requirements for success:

- routing metadata must be discovered reliably
- rebalancing must be automatic or very controlled
- cross-shard transactions must be avoided unless necessary

Order of operations before sharding a database (each step is much cheaper than the next):

1. Fix queries and indexes.
2. Add caching for hot reads.
3. Scale up the database instance.
4. Add read replicas for read-heavy traffic (accepting replica lag).
5. Split by domain: move unrelated tables to separate databases.
6. Shard by key.

Why consistent hashing: with `hash(key) % N`, changing `N` from 10 to 11 moves about 90% of keys. With consistent hashing (or a fixed number of virtual partitions mapped to nodes), adding a node moves only about `1/N` of the keys.

### Production pitfalls

- Uneven shard distribution creates hotspots and hot keys.
- Cross-shard joins become expensive and brittle.
- Rebalancing during traffic spikes can create latency and operational instability.
- More instances can hide poor application-level concurrency and resource limits.

### SDE best practices

- Start with a clear sharding key and stable data ownership model.
- Prefer stateless services; keep state in a controlled backend.
- Build in observability for per-shard saturation and variance.
- Keep a small, explicit set of cross-shard operations.

### Resources

- [Consistent hashing](https://en.wikipedia.org/wiki/Consistent_hashing)
- [Microsoft: Sharding pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [MongoDB: Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [AWS Builders' Library: Workload isolation using shuffle sharding](https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/)

---

## 14. Statelessness and Service Boundaries

Stateless services are easier to scale, replicate, and recover. Stateful logic should live in durable, explicit systems.

### Core mechanism

A stateless service makes decisions without depending on local mutable state. If an instance fails, another can replace it without data migration.

This matters because the main performance problem in many systems is not raw compute speed; it is coordination and local state loss.

### Engineering blueprint

```text
Stateful: user session, cached tokens, local counters, in-memory locks
Stateless: request handlers, business logic, transformation pipeline
```

Move state to:

- databases for durable shared state
- caches for ephemeral hot data
- message queues for asynchronous work
- session stores for user state only when required

### Production pitfalls

- Storing session state in local memory causes uneven scaling and restarts.
- Overly chatty service boundaries create large latency overhead.
- Too many fine-grained services create coordination and monitoring burden.
- Shared mutable global state in stateless services is often a hidden bottleneck.

### SDE best practices

- Keep service boundaries around ownership and data domain, not just team structure.
- Make critical state external and durable.
- Use idempotent request semantics where retries are expected.
- Avoid redistributing work by local memory state.

### Resources

- [The Twelve-Factor App](https://12factor.net/)
- [Architecture patterns for stateless services](https://learn.microsoft.com/en-us/azure/architecture/)
- [Service ownership and boundaries](https://martinfowler.com/articles/microservices.html)

---

## 15. Load Testing and Capacity Planning

Optimization without load testing is guesswork. Backends behave differently under concurrency, spikes, and failures.

### Core mechanism

Capacity planning estimates how much load the system can sustain while meeting its latency and reliability targets. It must consider both steady-state and burst scenarios.

A service may pass a single-request benchmark but fail under 100 concurrent users due to queueing, unbounded retries, or lock contention.

### Engineering blueprint

```text
Find normal throughput
Increase concurrency gradually
Measure latency, saturation, queue depth, error rate
Identify knee of the curve
Set capacity with headroom
```

Load testing should include:

- baseline traffic
- burst traffic
- gradual ramp-up and ramp-down
- failure injection
- cold cache and warm cache scenarios

### Production pitfalls

- Testing only one endpoint or one thread count.
- Ignoring warm-up effects and cache fill behavior.
- Assuming the system is “fine” because p50 is low.
- Missing the fact that a single bottleneck may hide behind a high aggregate throughput number.

### SDE best practices

- Run performance tests in a pre-production environment with realistic networking.
- Test both steady and burst conditions.
- Put explicit concurrency budgets in service configuration.
- Use the load test results to set autoscaling or traffic shaping thresholds.

### Resources

- [k6](https://k6.io/)
- [Locust](https://locust.io/)
- [Grafana k6 and load testing best practices](https://grafana.com/docs/k6/latest/)

---

## 16. Observability and Performance SLOs

You cannot optimize a system you cannot observe. A backend without metrics, logs, and traces is effectively blind.

### Core mechanism

Observability is the degree to which internal states are visible from external signals. For performance, the critical signals are:

- latency by endpoint and dependency
- queue depth and backlog age
- error rate, retries, and timeouts
- CPU, memory, GC, and disk usage
- DB query latency and connection wait time

### Engineering blueprint

```text
Endpoint latency SLO -> service dependency map -> metrics and tracing -> alerting -> burn rate -> action
```

SLOs should be specific:

- p95 response time
- error budget
- failed request rate
- per-dependency latency budget

### Production pitfalls

- Alerting on infrastructure metrics without user impact.
- Using only a single latency metric that hides percentile distributions.
- Failing to track downstream data and queue lag.
- Logging too much noise at high volume causes observability debt.

### SDE best practices

- Define alerting around both user-facing and internal dependency signals.
- Have a per-service dashboard with latency, saturation, and error rate.
- Tie a performance regression to a release or deployment.
- Keep a budget for observability instrumentation, not a “nice to have.”

### Resources

- [Google SRE: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Prometheus](https://prometheus.io/)

---

## 17. Failure Isolation and Resilience

A backend can only be fast if it remains predictable under failure. Poor isolation turns a small failure into a service-wide outage.

### Core mechanism

Resilience comes from isolating failures and limiting blast radius:

- timeouts
- retries with bounded backoff
- circuit breakers
- bulkheads
- rate limiting
- graceful degradation

### Engineering blueprint

```text
slow dependency -> timeout -> circuit breaker -> degraded response -> recover automatically
```

A resilient system does not fail gracefully by accident. It defines the expected behavior under degraded conditions and tests it.

Key mechanisms in more detail:

- **Timeouts:** set from the dependency's measured latency (for example, slightly above its p99.9), not a round default such as 30 s. Propagate a **deadline** so downstream calls do not keep working after the caller has given up.
- **Retry budget:** cap retries as a fraction of normal traffic (for example, retries ≤ 10% of requests). Retry at one layer only; three layers each retrying 3 times turns 1 failed call into 27.
- **Circuit breaker:** three states.

  ```text
  CLOSED    --(error rate > threshold)-->  OPEN       (fail fast, do not call dependency)
  OPEN      --(cool-down elapsed)------->  HALF-OPEN  (allow a few trial calls)
  HALF-OPEN --(trial calls succeed)----->  CLOSED
  HALF-OPEN --(trial call fails)-------->  OPEN
  ```

- **Bulkhead:** give each dependency its own bounded pool, so a slow recommendations service cannot use up the threads checkout needs.
- **Graceful degradation:** return a reduced response (cached data, default recommendations, a hidden widget) instead of an error when a non-critical dependency fails.

### Production pitfalls

- Retrying aggressively can turn a partial outage into a full overload.
- One dependency failure cascades as threads, connections, and worker pools saturate.
- Synchronous calls to slow downstreams increase tail latency for unrelated users.
- Error handling often becomes “just log it,” leaving the system unsure of what to do.

### SDE best practices

- Put explicit timeouts and retry budgets around every dependency.
- Use bulkheads to separate critical and noncritical workloads.
- Treat degraded mode as a first-class feature, not an accident.
- Validate resilience behavior under chaos tests.

### Resources

- [Google SRE Book: Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [Microsoft: Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Microsoft: Bulkhead pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/bulkhead)
- [Resilience4j](https://resilience4j.readme.io/) (Netflix Hystrix, often cited, has been in maintenance mode since 2018)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)

---

## 18. Data Access Hotspots

The biggest optimization wins often come from removing expensive data access patterns, not from optimizations in the application logic itself.

### Core mechanism

Hotspots are repeated access patterns that generate high cost or high contention:

- expensive joins on large tables
- frequent object hydration across many users
- repeated cache misses for the same hot keys
- read amplification caused by denormalization mistakes
- high-cardinality queries with poor index support

### Engineering blueprint

1. Find the hottest read or write path.
2. Measure the query count and latency by service and dependency.
3. Reduce the number of round trips.
4. Remove unnecessary data movement.
5. Re-check concurrency and lifecycle after the fix.

Hot-row contention is a common case. Example: every page view runs `UPDATE posts SET views = views + 1 WHERE id = 7`. Each update locks the same row, so throughput is limited to how fast one row can be locked and committed, no matter how many servers you add. Fixes:

- **Sharded counter:** spread writes across `K` rows (`post_id, shard = random(0..K-1)`) and sum them on read.
- **Buffer and flush:** increment in Redis (`INCR`) or in memory, and write the total to the database every few seconds.
- **Append, then aggregate:** insert view events and compute counts asynchronously.

The same pattern applies to hot cache keys (replicate the key or add a local near-cache) and hot partitions (add randomness to the partition key).

### Production pitfalls

- Hotspots often hide behind “just one code path” but are duplicated thousands of times.
- Data access patterns change under load in ways that are hard to see in a unit test.
- A quick fix can shift the hotspot to another service or table.

### SDE best practices

- Build a data access map for each high-volume domain.
- Optimize from the outside in: request flow, DB, cache, then service logic.
- Add safeguards for large results and pagination heavy endpoints.
- Measure the hotspot before and after changes—not just CPU time.

### Resources

- [PostgreSQL: Performance tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Scaling Memcache at Facebook (NSDI 2013)](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf) (hot keys, leases, and thundering herds at scale)

---

## 19. Performance Review Workflow

The best optimization teams use a repeatable process. A single happy-path benchmark is not a performance review.

### Core mechanism

A performance review workflow is a disciplined sequence of investigation and validation:

- define the problem
- capture baseline metrics
- isolate the bottleneck
- validate the cure with reproducible scenarios
- measure production impact after rollout

### Engineering blueprint

```text
Problem statement -> workload model -> benchmark -> trace -> fix -> benchmark -> deploy -> monitor
```

The review should include:

- endpoint or job name
- traffic profile
- baseline latency and resource utilization
- regression risk
- rollback/fallback plan

### Production pitfalls

- “We changed a query and it seems faster” without reproduction and comparison.
- Optimizing a feature that is not actually on the critical path.
- Forgetting to measure under a realistic workload and real concurrency.
- Missing the post-deploy performance check after rollout.

### SDE best practices

- Make performance a requirement in design and review.
- Keep benchmark artifacts and queries in version control.
- Add performance tests to the CI or release process when the workload is stable.
- Review p95 and p99 as part of the release checklist.

### Resources

- [Google SRE books](https://sre.google/books/)
- [Google SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [github-action-benchmark: tracking benchmark regressions in CI](https://github.com/benchmark-action/github-action-benchmark)

---

## 20. Production Checklist and Cheat Sheet

Use this as an operational default. Not every project needs all of it, but most need several of these rules.

### Core mechanism

Production optimization is not only code; it is configuration, metrics, and operational discipline.

### Checklist

- Measure baseline latency, throughput, CPU, memory, DB latency, and queue age.
- Identify p95/p99 and worst dependency spans.
- Check query plans and indexes for hot paths.
- Bound connection pools and worker pools.
- Add timeouts, retries with jitter, and circuit breakers.
- Keep caches explicit, measurable, and invalidation-safe.
- Put limits on async jobs and backlog sizes.
- Add proper observability before scaling or optimizing.
- Validate under realistic concurrency and failure scenarios.
- Define rollback plans and production monitoring guardrails.

### Common cheat sheet

```text
If latency is high and CPU is low -> network, DB, or external dependency issue
If CPU is high and queue depth grows -> app-level saturation or lock contention
If latency is stable but errors rise -> client behavior, throttling, or dependency failures
If cache hit rate is low -> key design, stale TTL, or invalidation issue
If queue age grows -> consumer throughput is below demand
If p99 is bad but p50 is fine -> tail-latency issue, often retries or contention
```

### SDE best practices

- Optimize measured bottlenecks; avoid chasing micro-optimizations.
- Keep performance artifacts with code changes.
- Review fail-over, backpressure, and degradation paths in the same sprint as the hot path change.
- Keep a performance runbook for each critical service.

---

## 21. Self-Check Questions

Try to answer each question before opening the answer.

<details>
<summary>1. A service handles 1,000 req/s with 50 ms average latency. How many requests are in flight? What happens if a dependency pushes latency to 500 ms?</summary>

Little's Law: 1,000 × 0.05 = 50 in flight. At 500 ms it becomes 500 in flight, so thread pools, connection pools, and memory need 10x the capacity. If pools are capped at, say, 100, requests start queuing and latency grows further.
</details>

<details>
<summary>2. p50 latency is 40 ms but p99 is 2 s. What are likely causes?</summary>

A small fraction of requests take a slow path: GC pauses, lock contention, cache misses hitting the database, retries after timeouts, a slow shard or hot key, or fan-out where one slow backend delays the whole request. Look at traces of p99 requests, not averages.
</details>

<details>
<summary>3. What is the N+1 query problem and how do you fix it?</summary>

Loading a list with one query, then running one more query per item to load related data. Fix it with a join or a single batched `WHERE id IN (...)` query, ORM eager loading, or a DataLoader-style batcher.
</details>

<details>
<summary>4. You have an index on (a, b, c). Which of these queries can use it for an index seek: WHERE a=1; WHERE b=2; WHERE a=1 AND c=3; WHERE a=1 AND b>5 AND c=3?</summary>

`a=1`: yes. `b=2`: no, it skips the leading column. `a=1 AND c=3`: seeks on `a` only; `c` is filtered afterward. `a=1 AND b>5 AND c=3`: seeks on `a` and the range on `b`; `c` cannot narrow the seek because it comes after a range column.
</details>

<details>
<summary>5. Why can increasing the DB connection pool size make things slower?</summary>

The database can only execute about as many queries at once as it has cores (plus I/O waits). Extra connections add memory, context switching, and lock contention without adding throughput. Across a fleet of instances, large pools can also exceed `max_connections`.
</details>

<details>
<summary>6. A queue's oldest-message age keeps growing. What does it mean and what do you do?</summary>

Consumers process slower than producers produce, so the backlog will not drain on its own. Increase consumer throughput (more workers, batching, faster handlers), reduce input (throttle or shed producers), and check for poison messages or a slow downstream blocking consumers.
</details>

<details>
<summary>7. Why are retries dangerous during an outage, and what limits them?</summary>

Retries multiply load on a dependency that is already failing, which can turn a partial outage into a full one. Limit them with a retry budget, exponential backoff with jitter, retrying at only one layer, circuit breakers, and idempotency so retries are safe.
</details>

<details>
<summary>8. Explain the three circuit breaker states.</summary>

Closed: calls pass through and failures are counted. Open: calls fail immediately without touching the dependency. Half-open: after a cool-down, a few trial calls are allowed; success closes the breaker and failure reopens it.
</details>

<details>
<summary>9. One database row receives thousands of updates per second (a like counter). How do you remove the bottleneck?</summary>

Spread writes across several counter rows and sum them on read, buffer increments in Redis or memory and flush periodically, or append events and aggregate asynchronously. Adding app servers does not help because all of them wait on the same row lock.
</details>

<details>
<summary>10. What is the difference between the USE and RED methods?</summary>

USE (Utilization, Saturation, Errors) is applied to resources such as CPU, disks, and pools to find what is overloaded. RED (Rate, Errors, Duration) is applied to services and endpoints to measure what users experience.
</details>

---

## 22. Master Resource List

This is a curated, practical list for backend optimization work.

### Systems and performance

- [Systems Performance by Brendan Gregg](https://www.brendangregg.com/sysperfbook.html)
- [Designing Data-Intensive Applications](https://www.oreilly.com/library/view/designing-data-intensive-applications/9781449373320/)
- [Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832)
- [AWS Builders' Library](https://aws.amazon.com/builders-library/)

### Observability and tracing

- [OpenTelemetry](https://opentelemetry.io/)
- [Prometheus](https://prometheus.io/)
- [Jaeger](https://www.jaegertracing.io/)
- [Grafana](https://grafana.com/)

### Databases

- [PostgreSQL documentation](https://www.postgresql.org/docs/current/)
- [MySQL documentation](https://dev.mysql.com/doc/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [MongoDB performance docs](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)

### Caching and data access

- [Redis docs](https://redis.io/docs/)
- [Caching challenges and strategies](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
- [Apache Kafka docs](https://kafka.apache.org/documentation/)
- [RabbitMQ docs](https://www.rabbitmq.com/documentation.html)

### Resilience and operations

- [SRE Workbook](https://sre.google/workbook/)
- [The Twelve-Factor App](https://12factor.net/)
- [Resilience4j](https://resilience4j.readme.io/)
- [Chaos engineering principles](https://principlesofchaos.org/)

---

End note: backend optimization is not a one-time cleanup project; it is a continuous product and operations discipline. In practice, the biggest gains usually come from measuring correctly, removing redundant work, and designing for predictable saturation rather than chasing isolated micro-optimizations.
