# Transactions at Scale: An End-to-End Study Guide (Frontend → Backend → Database)

Audience: SDEs who build systems that move money, inventory, credits, or any state that must never be lost, duplicated, or corrupted.
Scope: Database transactions (ACID, isolation, locking), money-safe data modeling (double-entry ledgers), idempotency, distributed transactions (2PC, sagas, outbox), payment-provider integration, scaling to millions of users, and the frontend UX that keeps users from creating duplicate or "lost" transactions.
Last reviewed: 2026-10-12. Examples use PostgreSQL, Node.js/TypeScript, and React. Database and vendor behavior changes between versions; confirm details in the linked documentation before relying on them.

Related guides: [APIs](./APIs.md) (idempotency, webhooks, status codes), [Caching Techniques](./CachingTechniques.md) (consistency and staleness), [Backend Optimization](./BackendOptmization.md) (connection pools, query plans), [Frontend](./Frontend.md), [Testing Framework](./TestingFramework.md), and [Authentication Concepts](./AuthenticationConcepts.md) (step-up auth, authorization).

The word **transaction** has two meanings in this guide:

- A **database transaction** is a group of reads and writes that the database treats as one unit: all of it commits, or none of it does.
- A **business transaction** is a user-visible operation, such as "send $20 to Alice" or "buy this item". One business transaction can span several database transactions, several services, and outside systems such as a card network.

Most production bugs happen at the boundary between the two. A system can use ACID database transactions correctly and still charge a customer twice.

## Table of Contents

0. [System Map: One Transfer, End to End](#0-system-map-one-transfer-end-to-end)
1. [ACID Fundamentals](#1-acid-fundamentals)
2. [Isolation Levels and Anomalies](#2-isolation-levels-and-anomalies)
3. [Concurrency Control: Locks, MVCC, Optimistic vs Pessimistic](#3-concurrency-control-locks-mvcc-optimistic-vs-pessimistic)
4. [Deadlocks, Timeouts, and Retries](#4-deadlocks-timeouts-and-retries)
5. [Modeling Money: The Double-Entry Ledger](#5-modeling-money-the-double-entry-ledger)
6. [Transaction State Machines](#6-transaction-state-machines)
7. [Idempotency End to End](#7-idempotency-end-to-end)
8. [API Design for Transactions](#8-api-design-for-transactions)
9. [Distributed Transactions and Two-Phase Commit](#9-distributed-transactions-and-two-phase-commit)
10. [Sagas and Compensation](#10-sagas-and-compensation)
11. [Transactional Outbox, Inbox, and CDC](#11-transactional-outbox-inbox-and-cdc)
12. [Delivery Semantics: At-Least-Once and "Exactly-Once"](#12-delivery-semantics-at-least-once-and-exactly-once)
13. [Integrating External Payment Providers](#13-integrating-external-payment-providers)
14. [Reconciliation and Auditing](#14-reconciliation-and-auditing)
15. [Scaling the Database: Pools, Replicas, Partitioning](#15-scaling-the-database-pools-replicas-partitioning)
16. [Sharding and Cross-Shard Transactions](#16-sharding-and-cross-shard-transactions)
17. [Hot Accounts and Contention](#17-hot-accounts-and-contention)
18. [Read Paths: History, Balances, and CQRS](#18-read-paths-history-balances-and-cqrs)
19. [Frontend: Transaction UX and Double-Submit Safety](#19-frontend-transaction-ux-and-double-submit-safety)
20. [Frontend: Status Updates, Payment Forms, and PCI Scope](#20-frontend-status-updates-payment-forms-and-pci-scope)
21. [Security, Abuse, and Fraud Controls](#21-security-abuse-and-fraud-controls)
22. [Observability and SLOs](#22-observability-and-slos)
23. [Testing Transactional Systems](#23-testing-transactional-systems)
24. [Capacity Planning for a Million Users](#24-capacity-planning-for-a-million-users)
25. [Reference Architecture](#25-reference-architecture)
26. [Interview Questions and Study Plan](#26-interview-questions-and-study-plan)
27. [Production Checklist](#27-production-checklist)
28. [Master Resource List](#28-master-resource-list)

---

## 0. System Map: One Transfer, End to End

Each section in this guide covers one step of this flow. The running example is a wallet app where users send money to each other and top up through a card provider.

```text
 Browser / mobile app
   | 1. User clicks "Send $20"; button disables; idempotency key already exists
   | 2. POST /v1/transfers  Idempotency-Key: 7f3c...   (HTTPS)
   v
 CDN / WAF / API gateway
   | 3. TLS, auth token check, coarse rate limit, request size limit
   v
 Transfer API service (stateless, N replicas)
   | 4. Authenticate -> authorize (owns source account?) -> validate
   | 5. Risk / velocity checks (fraud service, limits)
   | 6. ONE database transaction:
   |      - insert idempotency record (unique key)
   |      - lock source + destination accounts in a fixed order
   |      - check balance, write 2 ledger entries (debit + credit)
   |      - update cached balances, set transfer status = succeeded
   |      - insert outbox event "transfer.succeeded"
   |    COMMIT
   | 7. Return 201 + transfer body (same body is replayed on retries)
   v
 PostgreSQL primary  ---- streaming replication ---->  read replicas
   |                                                     (history, statements)
   | 8. Outbox relay / CDC reads committed events
   v
 Kafka / queue  ---> notification service (push, email)
                ---> analytics / search index
                ---> fraud model features
   |
   v
 Browser: 9. Shows "Sent". If the response was lost, the client retries with the
             SAME key and gets the SAME result, or polls GET /v1/transfers/{id}.

 Nightly + continuous: 10. Reconciliation compares ledger vs provider vs bank files.
```

| Layer | Main risk | Main defense | Section |
| --- | --- | --- | --- |
| Frontend | Double click, retry after timeout, reload mid-payment | Stable idempotency key, disabled button, "unknown" state | 19, 20 |
| API | Duplicate requests, unauthorized transfers | Idempotency-Key, authorization per account | 7, 8, 21 |
| Service logic | Partial failure across steps | State machine, single local DB transaction where possible | 6, 10 |
| Database | Lost updates, write skew, deadlocks | Correct isolation, row locks, lock ordering, retries | 2, 3, 4 |
| Data model | Money created or destroyed by bugs | Double-entry ledger, integer amounts, immutability | 5 |
| Events | Event published but DB rolled back (or reverse) | Transactional outbox, idempotent consumers | 11, 12 |
| External providers | Timeout with unknown outcome, duplicate webhooks | Provider idempotency keys, webhook dedup, reconciliation | 13, 14 |
| Scale | Pool exhaustion, hot rows, big tables | PgBouncer, partitioning, sharding, account splitting | 15, 16, 17 |

### The core invariants

Write down your invariants before you write code. Every design choice in this guide protects one of these:

```text
I1. Money is conserved: for every transfer, SUM(ledger entries) = 0.
I2. No account goes below its allowed minimum (usually 0).
I3. One user intent produces at most one effect (no double charge).
I4. Every committed state change is durable and auditable (who, what, when, why).
I5. Every non-terminal transaction eventually reaches a terminal state
    (succeeded / failed / reversed). Nothing stays "pending" forever.
I6. Internal records agree with external records (provider, bank) after reconciliation.
```

### Decision defaults

| Question | Default answer |
| --- | --- |
| Which database? | A relational database with strong transactions (PostgreSQL, MySQL/InnoDB). Move to distributed SQL only when a single primary cannot keep up. |
| Isolation level? | READ COMMITTED + explicit row locks or atomic conditional updates for money paths; SERIALIZABLE + retry loop where invariants span rows you cannot easily lock. |
| How to store amounts? | Signed 64-bit integers in minor units (cents), plus an ISO 4217 currency code. Never floating point. |
| How to avoid duplicates? | Client-generated idempotency key, unique constraint in the database, stored response replayed on retry. |
| Cross-service consistency? | Saga with explicit state machine + transactional outbox. Avoid 2PC across services. |
| Events? | At-least-once delivery + idempotent consumers. |

---

## 1. ACID Fundamentals

### Core mechanism

ACID describes the guarantees a database transaction gives:

| Letter | Meaning | How the database provides it | What it does NOT cover |
| --- | --- | --- | --- |
| **Atomicity** | All writes commit, or none do. | Write-ahead log (WAL) and undo / MVCC versions; rollback on error or crash. | Side effects outside the database (emails, HTTP calls, Kafka messages). |
| **Consistency** | Each transaction moves the database from one valid state to another. | Constraints you declare: `PRIMARY KEY`, `UNIQUE`, `FOREIGN KEY`, `CHECK`, `NOT NULL`, triggers. | Rules you did not declare. "C" is mostly the application's job. |
| **Isolation** | Concurrent transactions do not see each other's partial work. | Locks, MVCC snapshots, serializable conflict detection. | Full protection at weaker levels. The default level is weaker than serializable. |
| **Durability** | After COMMIT returns, the data survives a crash. | WAL flushed with `fsync` before commit is acknowledged; replication. | Disk loss without replicas or backups; `synchronous_commit = off`. |

```sql
BEGIN;
UPDATE accounts SET balance_minor = balance_minor - 2000 WHERE id = 1;
UPDATE accounts SET balance_minor = balance_minor + 2000 WHERE id = 2;
COMMIT;   -- both updates become visible together, or (on error / ROLLBACK) neither does
```

### Engineering blueprint

- Keep each transaction **short**: begin, do database work only, commit. A transaction that holds locks while it waits on an HTTP call holds those locks for the whole call.
- Put every write that must succeed together into **one** transaction: the ledger entries, the balance update, the status change, the idempotency record, and the outbox event.
- Express invariants as **database constraints** whenever possible. Application checks can race; constraints cannot.

```sql
ALTER TABLE accounts ADD CONSTRAINT balance_non_negative
  CHECK (allow_negative OR balance_minor >= 0);
ALTER TABLE transfers ADD CONSTRAINT amount_positive CHECK (amount_minor > 0);
ALTER TABLE transfers ADD CONSTRAINT one_effect_per_key UNIQUE (requester_id, idempotency_key);
```

### Production pitfalls

- **Autocommit by accident.** Many drivers run each statement in its own transaction unless you call `BEGIN`. Two separate `UPDATE` calls without `BEGIN` are two transactions. A crash between them loses money.
- **Using different connections inside one "transaction".** In Node.js, `pool.query()` may pick a different connection for each call. You must check out one client (`pool.connect()`) and run `BEGIN ... COMMIT` on that client.
- **Side effects inside the transaction.** Sending an email, then rolling back, means the user got an email for a transfer that never happened. Use the outbox (Section 11).
- **Swallowed errors.** In PostgreSQL, after any error the transaction is aborted. Every following statement fails until you `ROLLBACK`.
- **Unknown commit outcome.** If the connection drops while `COMMIT` is in flight, the client cannot know if it committed. Only idempotency (Section 7) makes a retry safe.

### SDE best practices

- Wrap transactions in one helper that always does `ROLLBACK` on error and always releases the connection (code in Section 4).
- Never mix "read from cache, write to DB" for invariant checks. Read the authoritative row inside the transaction.
- Log the transaction's business ID (`transfer_id`), not just "transaction failed".

### Resources

- PostgreSQL: Transactions tutorial — https://www.postgresql.org/docs/current/tutorial-transactions.html
- PostgreSQL: Reliability and the WAL — https://www.postgresql.org/docs/current/wal-intro.html
- Martin Kleppmann, *Designing Data-Intensive Applications*, Chapter 7 "Transactions"

---

## 2. Isolation Levels and Anomalies

### Core mechanism

Isolation levels define which **anomalies** can occur when transactions run concurrently. Stronger levels prevent more anomalies but cause more waiting or more retries.

| Anomaly | What happens | Money example |
| --- | --- | --- |
| **Dirty read** | T2 reads data T1 wrote but has not committed. T1 rolls back. | Showing a balance that never existed. |
| **Non-repeatable read** | T1 reads a row twice and gets different values because T2 committed in between. | Balance check passes, then the value changes before the debit. |
| **Phantom read** | T1 runs the same `WHERE` query twice and gets a different set of rows. | "Count today's transfers for the limit check" changes mid-transaction. |
| **Lost update** | T1 and T2 both read a value, both compute a new value, both write. One write is lost. | Two concurrent $20 withdrawals from $100 both write $80. User spent $40, balance shows $80. |
| **Write skew** | T1 and T2 read overlapping data, each checks an invariant, each writes a **different** row. Together they break the invariant. | Joint account rule: "sum of both sub-accounts >= 0". Two withdrawals from two sub-accounts each pass the check; sum goes negative. |

What the standard levels allow (PostgreSQL behavior shown; other databases differ):

| Level | Dirty read | Non-repeatable read | Phantom | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| READ UNCOMMITTED (PG treats as READ COMMITTED) | No | Possible | Possible | Possible | Possible |
| **READ COMMITTED** (PG default) | No | Possible | Possible | Possible* | Possible |
| REPEATABLE READ (PG = snapshot isolation) | No | No | No (in PG) | Detected → error 40001 | **Possible** |
| SERIALIZABLE (PG = SSI) | No | No | No | No | No (one txn gets error 40001) |

\* At READ COMMITTED, an atomic `UPDATE ... SET balance = balance - 20` is safe because PostgreSQL re-reads the latest committed row version before it applies the update. The lost update happens when you `SELECT` the balance into application memory, compute, and then `UPDATE ... SET balance = <computed value>`.

**Database differences matter:**

- **PostgreSQL** default: READ COMMITTED. REPEATABLE READ is snapshot isolation. SERIALIZABLE uses Serializable Snapshot Isolation (SSI): no extra blocking, but conflicts abort with SQLSTATE `40001`.
- **MySQL InnoDB** default: REPEATABLE READ. Plain `SELECT` uses a snapshot; locking reads (`FOR UPDATE`) and writes use next-key / gap locks, which can block inserts into ranges.
- **Oracle**: "SERIALIZABLE" is snapshot isolation and allows write skew.
- **Distributed SQL**: CockroachDB defaults to SERIALIZABLE. Google Spanner provides external consistency (strict serializability).

### Engineering blueprint

The lost-update bug and three fixes:

```sql
-- BUG (READ COMMITTED): read-compute-write in application code
SELECT balance_minor FROM accounts WHERE id = 1;          -- app sees 10000
-- app computes 10000 - 2000 = 8000
UPDATE accounts SET balance_minor = 8000 WHERE id = 1;    -- overwrites a concurrent change

-- FIX 1: atomic conditional update (best for single-row invariants)
UPDATE accounts
   SET balance_minor = balance_minor - 2000
 WHERE id = 1 AND balance_minor >= 2000
RETURNING balance_minor;          -- 0 rows returned => insufficient funds

-- FIX 2: pessimistic lock, then compute
SELECT balance_minor FROM accounts WHERE id = 1 FOR UPDATE;   -- others wait here
UPDATE accounts SET balance_minor = $computed WHERE id = 1;

-- FIX 3: optimistic version check
UPDATE accounts
   SET balance_minor = $computed, version = version + 1
 WHERE id = 1 AND version = $version_read;                    -- 0 rows => retry
```

Write skew needs either SERIALIZABLE or a lock on a **shared** row that every conflicting transaction touches:

```sql
-- Invariant spans sub-accounts 10 and 11 that belong to joint account 7.
BEGIN;
SELECT 1 FROM joint_accounts WHERE id = 7 FOR UPDATE;   -- materialize the conflict on one row
SELECT SUM(balance_minor) FROM accounts WHERE joint_account_id = 7;
-- check invariant, then write the debit to sub-account 10 or 11
COMMIT;
```

### Production pitfalls

- Assuming "the database handles concurrency" at the default level. READ COMMITTED does not prevent lost updates in read-compute-write code.
- Turning on SERIALIZABLE without a **retry loop**. Under load, a percentage of transactions will abort with `40001`. Without retries, those become user-facing errors.
- Running SERIALIZABLE transactions with long reads. SSI tracks read dependencies, so large scans increase false-positive aborts.
- Mixing isolation levels for transactions that touch the same data and assuming you get serializable behavior. In PostgreSQL, SERIALIZABLE guarantees only apply among transactions that run at SERIALIZABLE.
- Porting code between MySQL and PostgreSQL without re-checking isolation behavior.

### SDE best practices

- For each write path, name the anomaly you are defending against and the mechanism that stops it. Put that in a code comment or design doc.
- Prefer FIX 1 (atomic conditional update) for single-row balance changes. It needs no extra round trip and no explicit lock.
- Use SERIALIZABLE for complex invariants, with a bounded retry loop (3-5 attempts, jittered backoff).
- Run the Hermitage test suite against your database version to learn its real behavior.

### Resources

- PostgreSQL: Transaction Isolation — https://www.postgresql.org/docs/current/transaction-iso.html
- Berenson et al., "A Critique of ANSI SQL Isolation Levels" (1995) — https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/
- Hermitage: testing isolation levels in real databases — https://github.com/ept/hermitage
- Jepsen consistency models map — https://jepsen.io/consistency
- MySQL InnoDB locking — https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html

---

## 3. Concurrency Control: Locks, MVCC, Optimistic vs Pessimistic

### Core mechanism

**MVCC (Multi-Version Concurrency Control)**: writers create new row versions instead of overwriting. Readers see a consistent snapshot without blocking writers. In PostgreSQL, readers never block writers and writers never block readers. Writers that touch the **same row** still block each other.

**Pessimistic concurrency control**: take a lock before you act. Others wait.

```sql
SELECT ... FOR UPDATE            -- exclusive row lock; blocks other FOR UPDATE and UPDATE/DELETE
SELECT ... FOR NO KEY UPDATE     -- weaker; does not block FK checks from child inserts (PG)
SELECT ... FOR SHARE             -- shared lock; blocks writers, not other sharers
SELECT ... FOR UPDATE NOWAIT     -- fail immediately if locked
SELECT ... FOR UPDATE SKIP LOCKED -- skip locked rows (job queues, outbox relays)
SELECT pg_advisory_xact_lock(hashtext('user:42'));  -- app-defined lock, released at commit
```

**Optimistic concurrency control (OCC)**: do not lock. Read a version number, then write with `WHERE version = $read_version`. If zero rows changed, someone else won; reload and retry.

| | Pessimistic | Optimistic |
| --- | --- | --- |
| Best when | Contention is high (same account updated often) | Contention is low (user edits own profile) |
| Cost under conflict | Waiting | Wasted work + retry |
| Risk | Deadlocks, long lock waits | Retry storms under high contention |
| Works across HTTP requests | No (never hold DB locks across requests) | Yes (send version / ETag to client) |

### Engineering blueprint

Use optimistic locking across the HTTP boundary with ETags (for example, editing a scheduled payment):

```http
GET /v1/scheduled-payments/sp_123
200 OK
ETag: "v7"

PUT /v1/scheduled-payments/sp_123
If-Match: "v7"
-> 412 Precondition Failed if someone else changed it (version is now v8)
```

```sql
UPDATE scheduled_payments
   SET amount_minor = $1, version = version + 1
 WHERE id = $2 AND version = $3;   -- 0 rows => 412
```

Use `SKIP LOCKED` to build a safe work queue in Postgres (multiple workers, no double processing):

```sql
WITH next AS (
  SELECT id FROM payout_jobs
   WHERE status = 'queued' AND run_at <= now()
   ORDER BY run_at
   LIMIT 50
   FOR UPDATE SKIP LOCKED
)
UPDATE payout_jobs j
   SET status = 'running', locked_by = $worker, locked_at = now()
  FROM next WHERE j.id = next.id
RETURNING j.*;
```

### Production pitfalls

- `FOR UPDATE` on a query without a selective index locks far more rows than intended (and in MySQL, gap locks can block inserts).
- Holding a row lock while calling an external API. Every other request for that account queues behind the network call. Under load this becomes an outage.
- Forgetting that foreign-key checks take `FOR KEY SHARE` locks on the parent row. Inserting many ledger entries for one account contends with `FOR UPDATE` on that account. Use `FOR NO KEY UPDATE` when you are not changing the key.
- Advisory locks with a session-mode API (`pg_advisory_lock`) under a transaction-mode pooler. The lock can stay with a pooled connection after your request finishes. Use the `_xact_` variants.

### SDE best practices

- Default to the **narrowest** lock: atomic update > `FOR NO KEY UPDATE` > `FOR UPDATE` > table lock.
- Never hold a DB lock across a user think-time or a network call to another service.
- Use `SKIP LOCKED` for queues; use `NOWAIT` when you prefer a fast "try again" over waiting.

### Resources

- PostgreSQL: Explicit Locking — https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL: `SELECT` locking clause — https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- PostgreSQL: MVCC introduction — https://www.postgresql.org/docs/current/mvcc-intro.html

---

## 4. Deadlocks, Timeouts, and Retries

### Core mechanism

A **deadlock** happens when T1 holds lock A and waits for B, while T2 holds B and waits for A. PostgreSQL detects this after `deadlock_timeout` (default 1s) and aborts one transaction with SQLSTATE `40P01`.

Classic transfer deadlock:

```text
T1: transfer 1 -> 2   locks account 1, then wants 2
T2: transfer 2 -> 1   locks account 2, then wants 1     => deadlock
```

Fix: **always lock in a global order** (for example, ascending account ID), regardless of transfer direction.

### Engineering blueprint

A transaction helper with retries for serialization failures and deadlocks:

```ts
import { Pool, PoolClient } from 'pg';

type Isolation = 'READ COMMITTED' | 'REPEATABLE READ' | 'SERIALIZABLE';
const RETRYABLE = new Set(['40001', '40P01']); // serialization_failure, deadlock_detected

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));
const backoff = (attempt: number) => Math.random() * Math.min(1000, 25 * 2 ** attempt); // full jitter

export async function withTransaction<T>(
  pool: Pool,
  fn: (client: PoolClient) => Promise<T>,
  { isolation = 'READ COMMITTED' as Isolation, maxAttempts = 4 } = {},
): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query(`BEGIN ISOLATION LEVEL ${isolation}`);
      await client.query(`SET LOCAL lock_timeout = '2s'`);
      await client.query(`SET LOCAL statement_timeout = '5s'`);
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err: any) {
      await client.query('ROLLBACK').catch(() => {});
      if (!RETRYABLE.has(err.code) || attempt >= maxAttempts) throw err;
      await sleep(backoff(attempt));
    } finally {
      client.release();
    }
  }
}
```

The `fn` you pass must be **safe to re-run from the start**: no external side effects, and all state comes from the database inside the transaction.

Transfer with lock ordering:

```ts
export async function transfer(pool: Pool, t: { id: string; from: bigint; to: bigint; amount: bigint; currency: string }) {
  return withTransaction(pool, async (c) => {
    // Lock both accounts in ascending ID order => no deadlock between opposite transfers.
    const { rows } = await c.query(
      `SELECT id, balance_minor, currency, allow_negative
         FROM accounts WHERE id = ANY($1::bigint[])
        ORDER BY id
          FOR NO KEY UPDATE`,
      [[t.from, t.to]],
    );
    const from = rows.find((r) => BigInt(r.id) === t.from);
    const to = rows.find((r) => BigInt(r.id) === t.to);
    if (!from || !to) throw new AppError('ACCOUNT_NOT_FOUND');
    if (from.currency !== t.currency || to.currency !== t.currency) throw new AppError('CURRENCY_MISMATCH');
    if (!from.allow_negative && BigInt(from.balance_minor) < t.amount) throw new AppError('INSUFFICIENT_FUNDS');

    await c.query(
      `INSERT INTO ledger_entries (transfer_id, account_id, amount_minor, currency)
       VALUES ($1, $2, $3, $5), ($1, $4, $6, $5)`,
      [t.id, t.from, (-t.amount).toString(), t.to, t.currency, t.amount.toString()],
    );
    await c.query(`UPDATE accounts SET balance_minor = balance_minor - $2 WHERE id = $1`, [t.from, t.amount.toString()]);
    await c.query(`UPDATE accounts SET balance_minor = balance_minor + $2 WHERE id = $1`, [t.to, t.amount.toString()]);
    await c.query(`UPDATE transfers SET status = 'succeeded', completed_at = now() WHERE id = $1`, [t.id]);
    await c.query(
      `INSERT INTO outbox (aggregate_id, event_type, payload) VALUES ($1, 'transfer.succeeded', $2)`,
      [t.id, JSON.stringify({ transferId: t.id })],
    );
  });
}
```

Note: the `pg` driver returns `BIGINT` as strings by default. Convert to `BigInt` in code. Never convert money to a JavaScript `number` above 2^53.

Timeouts every money service should set:

| Setting | Purpose | Typical value |
| --- | --- | --- |
| `lock_timeout` | Fail instead of queueing forever behind a lock | 1-5 s |
| `statement_timeout` | Kill runaway queries | 2-10 s for OLTP |
| `idle_in_transaction_session_timeout` | Kill sessions that did `BEGIN` and then stalled (holding locks) | 10-60 s |
| HTTP server timeout | Must be **longer** than DB timeouts so the DB error, not a 504, reaches the handler | DB timeout + margin |
| Client timeout | Must allow server work + network; after it fires, treat outcome as **unknown** | 10-30 s |

### Production pitfalls

- Retrying at every layer (client, gateway, service, DB helper). Three layers with 3 attempts each = 27 attempts. This is a retry storm. Choose one layer per hop to own retries.
- Retrying non-idempotent work. The DB helper can retry because the whole transaction rolled back. An HTTP call to a provider cannot be retried blindly.
- Deadlocks between a "transfer" path and a "close account" path that lock tables in a different order. Lock ordering must be global, not per function.
- Long `deadlock_timeout` hiding contention. Monitor `deadlocks` in `pg_stat_database`.

### SDE best practices

- Document one lock order for the whole codebase (for example: user → account (by id) → transfer → ledger).
- Treat `40001` and `40P01` as normal, expected events. Track their rate as a metric.
- Use jittered exponential backoff and a small max attempt count.

### Resources

- PostgreSQL error codes — https://www.postgresql.org/docs/current/errcodes-appendix.html
- PostgreSQL client settings (`lock_timeout`, `idle_in_transaction_session_timeout`) — https://www.postgresql.org/docs/current/runtime-config-client.html
- AWS Architecture Blog, "Exponential Backoff and Jitter" — https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/

---

## 5. Modeling Money: The Double-Entry Ledger

### Core mechanism

A **double-entry ledger** records every movement of value as at least two **entries** that sum to zero: a debit from one account and a credit to another. Money is never created or destroyed by a single write. That makes invariant I1 checkable with one query.

```text
Transfer tr_1: Bob sends Alice $20.00 (USD)
  entry  account          amount_minor
  e1     bob_wallet          -2000
  e2     alice_wallet        +2000
                    sum =       0   ✔

Card top-up tp_9: Bob adds $50.00 by card
  e3     provider_clearing   -5000     (value the provider now owes us)
  e4     bob_wallet          +5000
                    sum =       0   ✔

Settlement st_4: provider pays out $48.25 and keeps a $1.75 fee
  e5     provider_clearing   +5000     (clearing returns to 0 for this payment)
  e6     settlement_bank     -4825     (cash that arrived from the provider)
  e7     provider_fees        -175     (fee expense)
                    sum =       0   ✔
```

Pick one sign convention (here: `+` = value flows into the account) and use it everywhere. Draw every flow, including fees, refunds, and chargebacks, as entries **before** writing code. If a flow does not sum to zero on paper, the code will not balance either.

Rules:

1. **Entries are immutable.** Never `UPDATE` or `DELETE` a ledger entry. Fix mistakes with a **reversal** transfer that posts opposite entries.
2. **Amounts are integers in minor units** (`BIGINT` cents) with an explicit currency. Currency exponents differ: USD 2, JPY 0, KWD 3. Use `NUMERIC` only if you need sub-minor precision (FX rates, crypto).
3. **Every account has one currency.** Cross-currency movement is two transfers through an FX account.
4. **Internal accounts are real accounts.** Fees, provider clearing, suspense, and FX all get ledger accounts so the total always balances.
5. **Balances are derived.** The true balance is `SUM(entries)`. A stored `balance_minor` is a cache, updated in the same transaction and checked by reconciliation.

### Engineering blueprint

```sql
CREATE TABLE accounts (
  id              BIGINT PRIMARY KEY,
  owner_type      TEXT NOT NULL CHECK (owner_type IN ('user','system')),
  owner_id        BIGINT,
  currency        CHAR(3) NOT NULL,
  balance_minor   BIGINT NOT NULL DEFAULT 0,
  allow_negative  BOOLEAN NOT NULL DEFAULT false,
  status          TEXT NOT NULL DEFAULT 'active',
  version         BIGINT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  CONSTRAINT balance_floor CHECK (allow_negative OR balance_minor >= 0)
);

CREATE TABLE transfers (
  id               UUID PRIMARY KEY,               -- UUIDv7 keeps inserts index-friendly
  requester_id     BIGINT NOT NULL,
  idempotency_key  TEXT NOT NULL,
  kind             TEXT NOT NULL,                  -- p2p, topup, payout, reversal, fee
  status           TEXT NOT NULL,                  -- see Section 6
  from_account_id  BIGINT NOT NULL REFERENCES accounts(id),
  to_account_id    BIGINT NOT NULL REFERENCES accounts(id),
  amount_minor     BIGINT NOT NULL CHECK (amount_minor > 0),
  currency         CHAR(3) NOT NULL,
  reverses_id      UUID REFERENCES transfers(id),
  failure_code     TEXT,
  metadata         JSONB NOT NULL DEFAULT '{}',
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at     TIMESTAMPTZ,
  UNIQUE (requester_id, idempotency_key)
);

CREATE TABLE ledger_entries (
  id            BIGINT GENERATED ALWAYS AS IDENTITY,
  transfer_id   UUID NOT NULL,
  account_id    BIGINT NOT NULL,
  amount_minor  BIGINT NOT NULL CHECK (amount_minor <> 0),   -- signed: negative = debit
  currency      CHAR(3) NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (created_at, id)                               -- partition-ready (Section 15)
) PARTITION BY RANGE (created_at);

CREATE INDEX ON ledger_entries (account_id, created_at DESC, id DESC);  -- statement queries
CREATE INDEX ON ledger_entries (transfer_id);

-- The application role can insert entries but never change them.
REVOKE UPDATE, DELETE, TRUNCATE ON ledger_entries FROM app_rw;
```

Enforce "entries of a transfer sum to zero" at commit time with a deferred constraint trigger:

```sql
CREATE FUNCTION check_transfer_balanced() RETURNS trigger AS $$
BEGIN
  IF (SELECT COALESCE(SUM(amount_minor), 0) FROM ledger_entries WHERE transfer_id = NEW.transfer_id) <> 0 THEN
    RAISE EXCEPTION 'ledger imbalance for transfer %', NEW.transfer_id;
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER transfer_balanced
  AFTER INSERT ON ledger_entries
  DEFERRABLE INITIALLY DEFERRED
  FOR EACH ROW EXECUTE FUNCTION check_transfer_balanced();
```

(Constraint triggers on partitioned tables require PostgreSQL 13+ — check your version. At very high volume, run this check in the application plus continuous reconciliation instead of per-row triggers.)

Holds (authorizations) for two-step flows such as card authorization then capture:

```text
authorize $50:  available -5000 , held +5000      (user can't spend it; it isn't gone)
capture  $50:   held -5000      , merchant +5000
void:           held -5000      , available +5000
```

Model "available" and "held" as separate ledger accounts (or a `bucket` column) so holds also obey double entry.

### Production pitfalls

- `FLOAT`/`DOUBLE` for money: `0.1 + 0.2 ≠ 0.3`. Rounding errors accumulate across millions of rows.
- JavaScript `number` for 64-bit amounts. Use `BigInt` or a decimal library, and send amounts in JSON as **strings** or integers well inside 2^53.
- Rounding without a rule. Splitting $10.00 three ways yields 333 + 333 + 334. Decide who gets the extra cent and test it.
- Editing a ledger row to "fix" a bug. This destroys the audit trail. Post a reversal.
- A single stored balance with no entries behind it. You cannot answer "why is this balance X?".

### SDE best practices

- Write an invariant query and run it continuously: `SELECT transfer_id FROM ledger_entries GROUP BY transfer_id HAVING SUM(amount_minor) <> 0`.
- Name system accounts clearly: `system:fees:usd`, `system:provider_clearing:stripe:usd`, `system:suspense:usd`.
- Treat the ledger as its own service with a narrow API (`post_transfer(entries[])`). Other services never write entries directly.

### Resources

- Modern Treasury, "Accounting for Developers" — https://www.moderntreasury.com/journal/accounting-for-developers-part-i
- Modern Treasury, "How to Scale a Ledger" — https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-i
- Square Engineering, "Books: an immutable double-entry accounting database service" — https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/
- TigerBeetle (purpose-built financial transactions database) docs — https://docs.tigerbeetle.com/
- ISO 4217 currency codes and minor units — https://www.iso.org/iso-4217-currency-codes.html

---

## 6. Transaction State Machines

### Core mechanism

A business transaction that involves anything outside one DB transaction (a card network, another service, a bank file) is **not** instantaneous. Model it as an explicit **state machine** with a small set of states and allowed transitions. Store the state in the database. Every transition is one atomic DB transaction.

```text
                 +-----------+
   create ------>|  created  |
                 +-----+-----+
                       | risk ok, funds held
                       v
                 +-----------+   provider timeout / unknown
                 | processing|-------------------+
                 +--+-----+--+                   |
       provider ok  |     | provider declined    v
                    v     v               +-------------+
           +-----------+ +--------+       | needs_review|  (ops / reconciliation decides)
           | succeeded | | failed |       +-------------+
           +-----+-----+ +--------+
                 | refund / chargeback
                 v
           +-----------+
           | reversed  |
           +-----------+
  Terminal states: succeeded, failed, reversed. Only these may be shown as "final" to users.
```

### Engineering blueprint

Enforce transitions in SQL so concurrent workers cannot apply two different transitions:

```sql
UPDATE transfers
   SET status = 'succeeded', completed_at = now()
 WHERE id = $1 AND status = 'processing'      -- compare-and-set on the state
RETURNING id;
-- 0 rows => someone else already moved it; read current state and act accordingly
```

Keep a history table for audit and debugging:

```sql
CREATE TABLE transfer_events (
  transfer_id UUID NOT NULL,
  seq         INT NOT NULL,
  from_status TEXT,
  to_status   TEXT NOT NULL,
  reason      TEXT,
  actor       TEXT NOT NULL,          -- 'api', 'webhook:stripe', 'worker:recon', 'ops:alice'
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (transfer_id, seq)
);
```

Add a **sweeper** job that finds transactions stuck in non-terminal states longer than an SLA (for example `processing` > 15 minutes) and drives them forward: query the provider, retry, or move them to `needs_review`. This enforces invariant I5.

### Production pitfalls

- Boolean flags instead of states (`is_paid`, `is_refunded`, `is_failed`). You end up with impossible combinations such as paid + failed.
- Treating "timeout" as "failed". A timeout means **unknown**. Marking it failed and letting the user retry can cause a double charge.
- No sweeper. Rare failures leave transfers in `processing` forever, with money held.
- Skipping states during webhook processing. Webhooks arrive out of order (a `refunded` event can arrive before `succeeded`).

### SDE best practices

- Keep the state list short and documented. Every state has an owner and a maximum dwell time.
- Make transitions idempotent: applying `succeeded` to an already-`succeeded` transfer is a no-op, not an error.
- Expose only terminal states as "final" in the UI. Show `processing` honestly.

### Resources

- Stripe PaymentIntents lifecycle (a good real-world state machine) — https://docs.stripe.com/payments/paymentintents/lifecycle
- microservices.io, "Saga" (state-driven coordination) — https://microservices.io/patterns/data/saga.html

---

## 7. Idempotency End to End

### Core mechanism

An operation is **idempotent** if doing it once or many times has the same effect. Networks fail in the worst place: after the server committed but before the client saw the response. The client must retry, and the server must recognize the retry.

```text
Client --POST /transfers (key K)--> Server --COMMIT--> DB   ✔ committed
Client <------ response lost / timeout ------ Server
Client --POST /transfers (key K) again-->  Server sees K already done
Client <------ same 201 + same transfer id ---- Server      ✔ no double charge
```

The idempotency key must be:

- **Generated by the client** once per user intent (not per HTTP attempt).
- **Scoped** to the caller (`(user_id, key)`), so users cannot collide or probe each other's keys.
- **Bound to the request**: store a hash of the request body; reject the same key with a different body.
- **Stored durably** with the result, for a retention window (commonly 24 hours to 7 days).

### Engineering blueprint

**Case A: the whole operation is one local DB transaction** (internal P2P transfer). Put the idempotency record in the **same** transaction. The `UNIQUE` index does the work: a concurrent duplicate waits on the index until the first commits, then hits a conflict.

```sql
CREATE TABLE idempotency_keys (
  user_id        BIGINT NOT NULL,
  key            TEXT NOT NULL,
  request_hash   BYTEA NOT NULL,
  response_code  INT,
  response_body  JSONB,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, key)
);
```

```ts
async function createTransferHandler(req: Req, res: Res) {
  const key = req.header('Idempotency-Key');
  if (!key || key.length > 255) return res.status(400).json({ error: 'IDEMPOTENCY_KEY_REQUIRED' });
  const hash = sha256(canonicalJson(req.body));

  // 1. Fast path: already completed?
  const prior = await db.query(
    `SELECT request_hash, response_code, response_body FROM idempotency_keys WHERE user_id = $1 AND key = $2`,
    [req.user.id, key],
  );
  if (prior.rowCount) return replay(prior.rows[0], hash, res);

  try {
    const result = await withTransaction(pool, async (c) => {
      // Claims the key. A concurrent duplicate blocks here until we commit/rollback.
      await c.query(
        `INSERT INTO idempotency_keys (user_id, key, request_hash) VALUES ($1, $2, $3)`,
        [req.user.id, key, hash],
      );
      const transfer = await executeTransfer(c, req.user, req.body);   // ledger, balances, outbox
      const body = serializeTransfer(transfer);
      await c.query(
        `UPDATE idempotency_keys SET response_code = 201, response_body = $3 WHERE user_id = $1 AND key = $2`,
        [req.user.id, key, body],
      );
      return body;
    });
    return res.status(201).json(result);
  } catch (err: any) {
    if (err.code === '23505' /* unique_violation on idempotency_keys */) {
      const row = (await db.query(
        `SELECT request_hash, response_code, response_body FROM idempotency_keys WHERE user_id = $1 AND key = $2`,
        [req.user.id, key],
      )).rows[0];
      return replay(row, hash, res);
    }
    if (err instanceof AppError) {
      // Business rejections (insufficient funds) are deterministic. Store them so retries
      // return the same answer, in a separate small transaction.
      await storeRejection(req.user.id, key, hash, err);
      return res.status(err.httpStatus).json(err.toJson());
    }
    throw err;  // unexpected error: nothing was stored, so a retry re-executes safely
  }
}

function replay(row: any, hash: Buffer, res: Res) {
  if (!row.request_hash.equals(hash)) return res.status(422).json({ error: 'IDEMPOTENCY_KEY_REUSED' });
  return res.status(row.response_code).json(row.response_body);
}
```

**Case B: the operation calls an external system** (charging a card). You cannot hold a DB transaction open across the provider call. Use **recovery points** (the pattern Stripe and Brandur Leach describe):

```text
idempotency_keys.recovery_point:  started -> transfer_created -> provider_called -> finished

Request (or retry) with key K:
  1. TX: insert key (or lock existing row; if locked_at recent => 409 "in progress").
  2. If recovery_point = started:
       TX: create transfer row status=processing; set recovery_point=transfer_created.
  3. If recovery_point = transfer_created:
       Call provider WITH provider idempotency key = transfer.id   (safe to repeat)
       TX: record provider result, post ledger entries, outbox; recovery_point=provider_called.
  4. If recovery_point = provider_called:
       TX: build response, store it; recovery_point=finished.
  5. If finished: replay stored response.
A crash at any point is resumed by the next retry or by a background "completer" job.
```

The key idea: **pass your own stable ID as the idempotency key to the next system**. The chain of keys makes the whole path idempotent.

Status codes (aligned with the IETF Idempotency-Key draft and common practice):

| Situation | Response |
| --- | --- |
| Key missing on an endpoint that requires it | `400 Bad Request` |
| Same key, same body, completed | Replay original status and body |
| Same key, request still in progress | `409 Conflict` (client retries later) |
| Same key, different body | `422 Unprocessable Content` |

### Production pitfalls

- Generating the key **per click** or **per HTTP attempt**. Then every retry is a new operation.
- Storing the key in Redis only, separate from the DB write. Redis says "new", the DB commits, Redis write fails → duplicate on retry. Keep the key in the same database transaction as the effect, or treat Redis only as an optimization in front of the DB unique constraint.
- Storing 5xx responses as final. A transient failure must remain retryable.
- Expiring keys before clients stop retrying (for example, mobile apps that retry queued payments after being offline for a day).
- Not scoping keys per user: a malicious user can replay another user's key to read their response.

### SDE best practices

- Require `Idempotency-Key` on every `POST` that moves value. Reject requests without it.
- Use natural idempotency where it exists: `UNIQUE (provider, provider_event_id)` for webhooks, `UNIQUE (order_id)` for "one payment per order".
- Test it: fire the same request 20 times concurrently and assert exactly one ledger effect (Section 23).

### Resources

- Stripe, "Designing robust and predictable APIs with idempotency" — https://stripe.com/blog/idempotency
- Stripe API, Idempotent requests — https://docs.stripe.com/api/idempotent_requests
- Brandur Leach, "Implementing Stripe-like Idempotency Keys in Postgres" — https://brandur.org/idempotency-keys
- IETF draft, "The Idempotency-Key HTTP Header Field" — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- See also: [APIs guide, Section 3](./APIs.md#3-http-methods-and-idempotency)

---
## 8. API Design for Transactions

### Core mechanism

A transaction API must make three things easy for clients: **create safely** (idempotent), **learn the outcome** (even after a lost response), and **list history** (paginated, stable under concurrent inserts).

### Engineering blueprint

```http
POST /v1/transfers
Authorization: Bearer <token>
Idempotency-Key: 0b9d5e1e-6c1f-4a36-9a5b-0f0a0d1f6e11
Content-Type: application/json

{ "from_account_id": "acc_1", "to_account_id": "acc_2",
  "amount": { "value": "2000", "currency": "USD" },      // minor units, as a string
  "note": "Dinner" }

201 Created
Location: /v1/transfers/tr_01J9Z...
{ "id": "tr_01J9Z...", "status": "succeeded",
  "amount": { "value": "2000", "currency": "USD" },
  "created_at": "2026-10-12T09:15:02.311Z" }
```

| Endpoint | Purpose | Notes |
| --- | --- | --- |
| `POST /v1/transfers` | Create | Requires `Idempotency-Key`. Returns `201` (done) or `202` (accepted, `status: processing`). |
| `GET /v1/transfers/{id}` | Read status | The client polls this when the outcome is pending or unknown. |
| `GET /v1/transfers?idempotency_key=K` | Find by key | Lets a client that lost the response (and the id) recover. |
| `GET /v1/accounts/{id}/entries?limit=50&cursor=...` | History | Keyset pagination (below). |
| `POST /v1/transfers/{id}/reverse` | Refund / reverse | Also idempotent. Creates a **new** transfer that references the original. |

Use **`202 Accepted`** when the work continues asynchronously (bank payouts, card charges that need 3-D Secure). Return the resource with `status: processing` and a `Location` to poll.

**Errors** should be specific and machine-readable, so the frontend can decide whether a retry makes sense:

```json
{ "error": { "code": "INSUFFICIENT_FUNDS", "message": "Balance is too low for this transfer.",
             "retryable": false, "transfer_id": "tr_01J9Z..." } }
```

| Code | HTTP | Retry with same key? |
| --- | --- | --- |
| `INSUFFICIENT_FUNDS`, `LIMIT_EXCEEDED`, `ACCOUNT_FROZEN` | 422 / 403 | No. Same key replays the same rejection. |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | 409 | Yes, after a short delay. |
| `RATE_LIMITED` | 429 + `Retry-After` | Yes, after `Retry-After`. |
| Internal error / timeout | 500 / 503 / 504 | Yes, **same key**. |

**History pagination**: use keyset (cursor) pagination, not `OFFSET`. Offsets skip or repeat rows when new transactions are inserted, and get slower with depth.

```sql
SELECT id, transfer_id, amount_minor, created_at
  FROM ledger_entries
 WHERE account_id = $1
   AND (created_at, id) < ($2, $3)          -- cursor = last row of previous page
 ORDER BY created_at DESC, id DESC
 LIMIT 50;
```

### Production pitfalls

- Returning amounts as JSON floating-point numbers (`20.0`). Use integer minor units or decimal strings.
- `DELETE /transfers/{id}` to "cancel". Money movements are never deleted. Use state transitions (`cancel` while still `created`) or reversals.
- `GET` endpoints with side effects (for example, "GET /confirm?token=..." that moves money). Link prefetchers and crawlers will trigger it.
- Leaking existence: `GET /transfers/{id}` for someone else's transfer should return `404`, not `403`.

### SDE best practices

- Use opaque, prefixed, non-sequential IDs (`tr_`, `acc_`) based on UUIDv7 or ULID. They are safe to expose and sort by time.
- Return the full resource from `POST` so the client does not need a follow-up read (which might hit a lagging replica).
- Version the API; add fields, never change field meaning.

### Resources

- Stripe API reference (good transaction API design) — https://docs.stripe.com/api
- RFC 9110 HTTP Semantics (202, 409, 412, 422) — https://www.rfc-editor.org/rfc/rfc9110
- RFC 9457 Problem Details for HTTP APIs — https://www.rfc-editor.org/rfc/rfc9457
- See also: [APIs guide](./APIs.md)

---

## 9. Distributed Transactions and Two-Phase Commit

### Core mechanism

When one business transaction must change data in **several** databases or services, a single local ACID transaction no longer covers it. The classic answer is **two-phase commit (2PC)**:

```text
Coordinator                Participant A (wallet DB)     Participant B (orders DB)
   | -- PREPARE ------------------> |                            |
   | -- PREPARE ----------------------------------------------->  |
   | <- YES (locks held, durable) - |                            |
   | <- YES ----------------------------------------------------- |
   | (writes COMMIT decision to its log)
   | -- COMMIT -------------------> |                            |
   | -- COMMIT ------------------------------------------------>  |
```

PostgreSQL supports the participant side with `PREPARE TRANSACTION` / `COMMIT PREPARED` (disabled by default: `max_prepared_transactions = 0`). XA is the cross-vendor standard.

**Why 2PC is usually avoided between microservices:**

- **Blocking**: if the coordinator crashes after participants voted YES, participants hold locks until the coordinator recovers. Stuck prepared transactions also block vacuum in Postgres.
- **Latency**: at least two round trips to every participant, with locks held throughout.
- **Coupling**: every participant must support the protocol. HTTP APIs, Kafka, Stripe, and banks do not.
- **Availability**: the transaction can only commit if every participant is up.

**Where distributed atomic commit does work well**: **inside** a distributed database. Google Spanner (2PC over Paxos groups, TrueTime), CockroachDB (parallel commits), YugabyteDB, and TiDB run a commit protocol internally, so you write normal SQL transactions across shards. You pay with higher commit latency, especially across regions.

### Engineering blueprint

Decision guide:

| Situation | Approach |
| --- | --- |
| All data in one database | Local ACID transaction. Simplest and best. |
| Data across shards of one distributed SQL database | Let the database handle it (Spanner / CockroachDB transactions). |
| Data across services you own | Saga + outbox (Sections 10-11). |
| Data across you and an external provider | State machine + provider idempotency keys + webhooks + reconciliation (Sections 6, 13, 14). |

Design to **avoid** the need: put data that must change atomically into the **same** database (same service boundary). Draw service boundaries around transactional consistency, not around nouns.

### Production pitfalls

- Splitting "wallet" and "ledger" into two services with two databases, then needing a distributed transaction for every transfer.
- Turning on `max_prepared_transactions` and forgetting to monitor `pg_prepared_xacts`. Orphaned prepared transactions hold locks forever.
- Believing a message broker plus a database gives atomicity without the outbox pattern (the "dual write" problem, Section 11).

### SDE best practices

- If a consistency boundary forces a distributed transaction on a hot path, merge the boundary.
- Read Pat Helland's "Life Beyond Distributed Transactions" to see how systems work without global transactions.

### Resources

- PostgreSQL: `PREPARE TRANSACTION` — https://www.postgresql.org/docs/current/sql-prepare-transaction.html
- Pat Helland, "Life Beyond Distributed Transactions: an Apostate's Opinion" — https://queue.acm.org/detail.cfm?id=3025012
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (OSDI 2012) — https://research.google/pubs/spanner-googles-globally-distributed-database-2/
- CockroachDB, "Parallel Commits" — https://www.cockroachlabs.com/blog/parallel-commits/
- *Designing Data-Intensive Applications*, Chapter 9

---

## 10. Sagas and Compensation

### Core mechanism

A **saga** splits a business transaction into a sequence of **local** transactions. Each step commits on its own and publishes an event or calls the next step. If a later step fails, the saga runs **compensating transactions** to semantically undo earlier steps.

Example: "Buy a concert ticket with wallet balance."

```text
Step                         Compensation
1. Order:   create order (pending)          -> cancel order
2. Wallet:  hold $80 on user's wallet       -> release hold
3. Tickets: reserve seat                    -> release seat
4. Wallet:  capture hold (PIVOT)            (no compensation after this point)
5. Order:   mark confirmed; send ticket     (retry until success)
```

Step types (from microservices.io):

- **Compensatable**: can be undone (hold funds, reserve seat).
- **Pivot**: the point of no return. After it succeeds, the saga must finish.
- **Retriable**: after the pivot; guaranteed to succeed eventually with retries (send email, mark confirmed).

Order steps so that risky, likely-to-fail steps run **before** the pivot, and the pivot is the step that is hardest to undo (capturing money).

**Orchestration vs choreography:**

| | Orchestration | Choreography |
| --- | --- | --- |
| How | A central orchestrator tells each service what to do and tracks state | Each service reacts to events and emits its own |
| Visibility | One place to see saga state | State spread across services |
| Good for | Money flows, many steps, complex compensation | Simple 2-3 step flows |
| Tools | Temporal, AWS Step Functions, Camunda, a custom state table | Kafka / queue events |

### Engineering blueprint

Orchestrated saga with a durable state table (or a workflow engine such as Temporal, which persists this for you):

```ts
// Pseudo-code for an orchestrator step runner. Each step is idempotent, keyed by saga id.
async function runPurchaseSaga(sagaId: string) {
  const s = await loadSaga(sagaId);  // { state, orderId, holdId, seatId }
  try {
    if (s.state === 'started')      { await orders.create(sagaId);                 await advance(s, 'order_created'); }
    if (s.state === 'order_created'){ s.holdId = await wallet.hold(sagaId, s.amount); await advance(s, 'funds_held'); }
    if (s.state === 'funds_held')   { s.seatId = await tickets.reserve(sagaId);    await advance(s, 'seat_reserved'); }
    if (s.state === 'seat_reserved'){ await wallet.capture(sagaId, s.holdId);      await advance(s, 'captured'); }   // pivot
    if (s.state === 'captured')     { await retryForever(() => orders.confirm(sagaId)); await advance(s, 'done'); }
  } catch (err) {
    if (isBeforePivot(s.state)) await compensate(s);   // release seat, release hold, cancel order (each idempotent)
    else throw err;                                    // after pivot: keep retrying, alert if stuck
  }
}
```

Each service call carries `sagaId` (or a derived step key) as its idempotency key, so the orchestrator can safely re-run any step after a crash.

### Production pitfalls

- **No isolation between sagas.** Other requests can see intermediate state (an order that is "pending" and later cancelled). Use semantic locks (a `pending` flag that other flows respect), or design reads to tolerate it.
- **Compensations that can fail.** A compensation must be retriable and idempotent. "Refund" can fail at the provider; you need a retry and an alert, not a log line.
- **Compensation is not rollback.** If the user got a notification after step 2, compensation cannot "unsend" it. Delay irreversible side effects until after the pivot.
- **Timeouts treated as failure.** A timed-out `wallet.hold` might have succeeded. Compensation must handle "release a hold that may or may not exist".

### SDE best practices

- Use orchestration for money. You need one place to answer "where is this purchase stuck?".
- Prefer a durable workflow engine (Temporal or Step Functions) over a hand-rolled state machine once you have more than a few sagas.
- Prefer **holds** (reserve, then capture) over "debit, then refund on failure". A released hold leaves no trace on the user's statement; a refund does.

### Resources

- microservices.io, Saga pattern — https://microservices.io/patterns/data/saga.html
- Garcia-Molina & Salem, "Sagas" (1987) — https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
- Temporal documentation — https://docs.temporal.io/
- AWS Step Functions saga example — https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga.html

---

## 11. Transactional Outbox, Inbox, and CDC

### Core mechanism

**The dual-write problem**: a service writes to its database and publishes to a message broker. These are two systems with no shared transaction.

```text
Option 1: commit DB, then publish  -> crash in between => event lost (notification never sent, ledger sync missed)
Option 2: publish, then commit DB  -> commit fails     => event for something that never happened
```

**Transactional outbox**: write the event into an `outbox` table **in the same DB transaction** as the business change. A separate relay reads committed outbox rows and publishes them. The event exists if and only if the business change committed.

```text
 API TX: [ update ledger + insert outbox row ] COMMIT
                                   |
             relay (poller) or CDC (Debezium reading the WAL)
                                   v
                         Kafka topic "transfers"
                                   v
            consumers (dedupe with an inbox table, then act)
```

**Inbox (idempotent consumer)**: the consumer records each processed message ID in its own DB, in the same transaction as its side effect. Duplicates hit the unique key and are skipped.

### Engineering blueprint

```sql
CREATE TABLE outbox (
  id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  event_id      UUID NOT NULL UNIQUE DEFAULT gen_random_uuid(),
  aggregate_id  TEXT NOT NULL,          -- used as the Kafka partition key => per-aggregate ordering
  event_type    TEXT NOT NULL,
  payload       JSONB NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at  TIMESTAMPTZ
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

Polling relay:

```sql
-- In a short transaction, by one or more relay workers:
SELECT id, event_id, aggregate_id, event_type, payload
  FROM outbox
 WHERE published_at IS NULL
 ORDER BY id
 LIMIT 500
 FOR UPDATE SKIP LOCKED;
-- publish batch to Kafka (await acks), then:
UPDATE outbox SET published_at = now() WHERE id = ANY($1);
```

Consumer with inbox:

```sql
BEGIN;
INSERT INTO inbox (consumer, event_id) VALUES ('notifications', $1)
ON CONFLICT DO NOTHING;               -- 0 rows inserted => duplicate, skip work, COMMIT
-- ... side effect stored in this DB (e.g. insert notification row) ...
COMMIT;
-- then commit the Kafka offset
```

**CDC (Change Data Capture)** alternative: Debezium reads PostgreSQL's logical replication stream and publishes outbox inserts (its Outbox Event Router does this). Pros: no polling, low latency, preserves commit order. Cons: more infrastructure; replication slots retain WAL if the connector stops, which can fill the disk.

### Production pitfalls

- Relay that tracks "last published id" and polls `WHERE id > last`. Identity/sequence values are assigned at insert time but transactions commit in a different order, so a lower id can become visible **after** a higher id was read. That event is skipped forever. Use `published_at IS NULL` (as above) or CDC.
- Expecting global ordering. With multiple relay workers and `SKIP LOCKED`, events can be published out of order. Design consumers to need ordering only per aggregate (partition key = `aggregate_id`), and include a version number in the payload.
- Outbox table growing forever. Delete or partition-drop published rows after a retention window.
- Inactive Debezium replication slot → WAL piles up → primary disk full. Alert on `pg_replication_slots` lag.

### SDE best practices

- Every state change that other services care about writes an outbox row in the same transaction. No direct publishing from request handlers.
- Make events self-describing: `event_id`, `event_type`, `aggregate_id`, `aggregate_version`, `occurred_at`, payload.
- Every consumer is idempotent. Test by delivering each message twice.

### Resources

- microservices.io, Transactional outbox — https://microservices.io/patterns/data/transactional-outbox.html
- microservices.io, Idempotent consumer — https://microservices.io/patterns/communication-style/idempotent-consumer.html
- Debezium Outbox Event Router — https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html
- PostgreSQL logical replication — https://www.postgresql.org/docs/current/logical-replication.html

---

## 12. Delivery Semantics: At-Least-Once and "Exactly-Once"

### Core mechanism

| Semantics | Meaning | How you get it | Risk |
| --- | --- | --- | --- |
| At-most-once | Each message is processed 0 or 1 times | Ack before processing | Lost messages |
| At-least-once | Each message is processed 1 or more times | Ack (commit offset) **after** processing | Duplicates |
| Exactly-once **effect** | Duplicates happen, but the effect happens once | At-least-once + idempotent processing | Needs dedup state |

True exactly-once **delivery** over an unreliable network is not achievable. What systems achieve is exactly-once **processing within a closed system**. Kafka's idempotent producer and transactions give exactly-once for read-from-Kafka → write-to-Kafka pipelines. As soon as the consumer writes to a database, calls an API, or sends an email, you are back to at-least-once and need idempotency.

### Engineering blueprint

Consumer loop that is safe under redelivery:

```text
for message in poll():
    BEGIN
      INSERT INTO inbox(consumer, event_id) ON CONFLICT DO NOTHING
      if inserted: apply effect in same DB transaction
    COMMIT
    commit Kafka offset            // if we crash before this, message is redelivered => inbox dedups
```

For effects outside your DB (email, push notification, third-party API), pass `event_id` as the downstream idempotency key, or accept rare duplicates where they are harmless (a duplicate "you received $20" push is annoying; a duplicate payout is not acceptable).

**Dead-letter queues (DLQ)**: after N failed attempts, move the message to a DLQ and alert. Never silently drop a money event. Build tooling to inspect and replay DLQ messages.

### Production pitfalls

- Auto-commit of offsets (Kafka `enable.auto.commit=true`) with slow processing: offsets get committed before processing finishes, so a crash loses messages.
- Poison messages that crash the consumer and block the partition forever.
- Assuming order across partitions.

### SDE best practices

- Default to at-least-once + idempotent consumers everywhere.
- Partition by the entity that needs ordering (account ID, transfer ID).
- Track consumer lag as an SLO input: lag means users see stale history and late notifications.

### Resources

- Confluent, "Exactly-once Semantics Are Possible: Here's How Kafka Does It" — https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- Kafka documentation, delivery semantics — https://kafka.apache.org/documentation/#semantics
- Tyler Treat, "You Cannot Have Exactly-Once Delivery" — https://bravenewgeek.com/you-cannot-have-exactly-once-delivery/

---

## 13. Integrating External Payment Providers

### Core mechanism

Card payments, bank transfers, and payouts go through providers (Stripe, Adyen, Braintree, PayPal, bank APIs). The provider is a separate system with its own state. Three channels connect you:

1. **Synchronous API calls** (create / confirm / capture / refund).
2. **Asynchronous webhooks** (the provider tells you about state changes later).
3. **Settlement and reconciliation files/reports** (the source of truth for money that actually moved).

Common card flow (authorize then capture):

```text
Browser ──(card details in provider iframe)──> Provider  → returns payment method token
Browser ──POST /checkout (token, Idempotency-Key)──> Your API
Your API: TX create payment(status=processing)
Your API ──create PaymentIntent (idempotency key = payment.id)──> Provider
   ├─ requires_action (3-D Secure) → return client_secret → browser completes challenge
   ├─ succeeded / failed → TX update payment, ledger, outbox
   └─ timeout → leave status=processing; webhook or sweeper resolves it
Provider ──webhook payment_intent.succeeded──> Your webhook endpoint (verify signature, dedupe, apply)
```

### Engineering blueprint

Rules for the outbound call:

- Create **your** record first (status `processing`), commit, then call the provider. Your record ID is the provider idempotency key.
- **Never** call the provider inside an open DB transaction.
- On timeout or 5xx: do not mark failed. Retry with the **same** idempotency key, or let the sweeper query the provider by your ID.

Webhook handler:

```ts
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  let event;
  try {
    event = stripe.webhooks.constructEvent(req.body, req.header('Stripe-Signature')!, WEBHOOK_SECRET);
  } catch {
    return res.status(400).end();   // bad signature: reject
  }

  // Store first, process asynchronously. Respond fast so the provider does not time out and retry.
  await db.query(
    `INSERT INTO provider_events (provider, event_id, type, payload)
     VALUES ('stripe', $1, $2, $3) ON CONFLICT (provider, event_id) DO NOTHING`,
    [event.id, event.type, event],
  );
  res.status(200).end();
  // A worker processes provider_events rows: maps to our payment by metadata / our id,
  // applies an allowed state transition (CAS on status), posts ledger entries, writes outbox.
});
```

- Webhooks are **at-least-once** and **unordered**. Dedupe by event ID, and apply only allowed state transitions. If needed, re-fetch the object from the provider API for its latest state instead of trusting the event order.
- Verify signatures with the **raw** body. Body parsers that re-serialize JSON break the signature.

### Production pitfalls

- Treating the synchronous API response as the only signal. Some flows (3-D Secure, bank debits, delayed captures) complete only via webhook.
- Webhook endpoint that does heavy work inline and times out. The provider retries, and you process duplicates concurrently.
- Not storing the provider's object IDs on your records. Reconciliation becomes guesswork.
- Refund flows that are not idempotent: an ops person clicks "refund" twice.
- Ignoring chargebacks/disputes: they are a separate, late state change (weeks later) and need ledger entries too.

### SDE best practices

- Keep a **provider abstraction** with your own state machine, but store raw provider payloads for audit.
- Use provider test modes and test clocks to simulate 3-D Secure, declines, disputes, and delayed webhooks.
- Alert on webhook signature failures (misconfiguration or attack) and on webhook processing lag.

### Resources

- Stripe, Webhooks best practices — https://docs.stripe.com/webhooks
- Stripe, Accept a payment (PaymentIntents) — https://docs.stripe.com/payments/accept-a-payment
- Stripe, Place a hold on a payment method (authorize & capture) — https://docs.stripe.com/payments/place-a-hold-on-a-payment-method
- Adyen, Webhooks — https://docs.adyen.com/development-resources/webhooks/

---

## 14. Reconciliation and Auditing

### Core mechanism

**Reconciliation** compares two independent records of the same money and flags differences. It is the final safety net that catches bugs every other layer missed. Financial teams rely on it; regulators and auditors require it.

Three levels:

| Level | Compares | Catches |
| --- | --- | --- |
| Internal | Cached `accounts.balance_minor` vs `SUM(ledger_entries)`; every transfer sums to 0 | Application bugs, partial writes, manual DB edits |
| Internal vs provider | Your payments vs provider's charges/refunds/disputes (API or report) | Missed webhooks, stuck `processing`, double charges |
| Provider vs bank | Provider payouts vs bank statement deposits | Fees, FX differences, missing settlements |

### Engineering blueprint

```sql
-- 1. Balance cache drift
SELECT a.id, a.balance_minor, COALESCE(SUM(e.amount_minor), 0) AS ledger_sum
  FROM accounts a
  LEFT JOIN ledger_entries e ON e.account_id = a.id
 GROUP BY a.id
HAVING a.balance_minor <> COALESCE(SUM(e.amount_minor), 0);

-- 2. Unbalanced transfers (should always be empty)
SELECT transfer_id FROM ledger_entries GROUP BY transfer_id HAVING SUM(amount_minor) <> 0;

-- 3. Stuck transfers
SELECT id, status, created_at FROM transfers
 WHERE status IN ('created','processing') AND created_at < now() - interval '15 minutes';
```

At millions of accounts, run (1) incrementally: maintain daily per-account entry totals (snapshots) and only re-sum entries since the last verified snapshot.

Provider reconciliation job:

```text
For each day D:
  ours   = payments where provider='stripe' and created on D        (keyed by provider_payment_id)
  theirs = provider balance transactions for D (API or downloaded report)
  match by id → compare amount, currency, status, fee
  unmatched in ours   → missed charge? stuck? → auto-query provider, then needs_review
  unmatched in theirs → payment we never recorded (lost response) → create + alert
  amount mismatch     → needs_review, never auto-fix silently
```

**Audit trail** requirements:

- Append-only ledger + transfer event history (who/what/when/why).
- Every manual adjustment is a ledger transfer with `kind = 'adjustment'`, a reason, a ticket reference, and an approver (four-eyes principle).
- Database role separation: the app role cannot update/delete ledger rows; ops tools go through the same API.

### Production pitfalls

- Reconciliation that only logs. Every mismatch must create a tracked item with an owner.
- Auto-fixing mismatches by overwriting balances. Fix the cause and post adjusting entries.
- Time zone mismatches: the provider's "day" is UTC; the bank's day might be local time. Define cut-off times.

### SDE best practices

- Run internal reconciliation continuously (every few minutes for recent data) and full reconciliation daily.
- Make the zero-mismatch count a dashboard metric and a release gate.

### Resources

- Modern Treasury, reconciliation guides — https://www.moderntreasury.com/learn/reconciliation
- Stripe, Reports and reconciliation — https://docs.stripe.com/reports/reconciliation
- Square, "Books" (immutable ledger and audit) — https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/

---

## 15. Scaling the Database: Pools, Replicas, Partitioning

### Core mechanism

A single well-tuned PostgreSQL primary on modern hardware handles **thousands of short write transactions per second** and far more reads. For most "million user" products, the first limits you hit are not raw throughput. They are:

1. **Connections**: each Postgres connection is a process (~several MB of memory). Hundreds of app pods × a pool of 20 each overwhelms the server.
2. **Table and index size**: ledgers grow forever. Vacuum, index bloat, and backups slow down.
3. **Read load**: history and statement queries compete with the write path.
4. **Hot rows**: one popular account serializes all its transfers (Section 17).

### Engineering blueprint

**Connection pooling**

```text
 App pods (each with a small client pool, e.g. 5-10)
        |
   PgBouncer (transaction pooling mode)  — thousands of client conns → ~50-200 server conns
        |
   PostgreSQL primary (max_connections ~ 200-500)
```

- Transaction pooling assigns a server connection only for the duration of a transaction. Session state does not survive between transactions: avoid session-level `SET` (use `SET LOCAL`), session advisory locks, `LISTEN/NOTIFY`, and temp tables across transactions. Check your PgBouncer version and driver settings for prepared statement support.
- Size the server pool from the database's capacity (roughly a small multiple of CPU cores), not from app demand. Excess requests should queue in the pooler, not create more DB processes.

**Read replicas**

- Send history, search, and analytics reads to replicas. Send anything that **decides** money (balance checks, limit checks, idempotency lookups) to the **primary**.
- Replicas lag. A user who just paid and then opens history may not see the payment. Fixes: return the created resource from the `POST` and insert it into the client cache; route reads for a few seconds after a write to the primary ("read-your-writes" via a session flag or LSN tracking).

**Partitioning (one database, many physical tables)**

```sql
-- Monthly partitions of the ledger
CREATE TABLE ledger_entries_2026_10 PARTITION OF ledger_entries
  FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
-- Automate creation ahead of time (pg_partman or a cron job).
-- Old partitions: detach and archive to cheap storage after the retention period.
```

- Benefits: smaller indexes for recent data, fast drop/archive of old data, vacuum per partition.
- Queries must include the partition key (`created_at`) to prune partitions. Statement queries by `account_id` alone scan all partitions' indexes; always bound them by date range.

**Other levers** (see [Backend Optimization](./BackendOptmization.md)): covering indexes for hot queries, `EXPLAIN (ANALYZE, BUFFERS)`, autovacuum tuning on high-churn tables (`accounts`, `outbox`), `synchronous_commit` + synchronous replicas for durability across a node failure (costs latency), and batching inserts.

### Production pitfalls

- Pool exhaustion cascade: one slow query holds connections → requests queue → timeouts → retries add more load. Set pool acquire timeouts and shed load early.
- Reading a balance from a replica to approve a transfer.
- Unbounded `outbox`/`idempotency_keys`/`provider_events` tables. Add TTL cleanup or partition by day.
- `SERIAL` (32-bit) primary keys on high-volume tables. Use `BIGINT`.
- Long-running analytics queries on the primary that hold old snapshots and block vacuum cleanup.

### SDE best practices

- Measure before sharding. Vertical scaling + pooling + replicas + partitioning carries most products well past a million users.
- Separate OLTP from analytics: CDC into a warehouse instead of reporting from production.
- Set up PITR (point-in-time recovery) backups and test restores regularly. Know your RPO and RTO.

### Resources

- PgBouncer features and pooling modes — https://www.pgbouncer.org/features.html
- PostgreSQL table partitioning — https://www.postgresql.org/docs/current/ddl-partitioning.html
- pg_partman — https://github.com/pgpartman/pg_partman
- PostgreSQL high availability and replication — https://www.postgresql.org/docs/current/high-availability.html
- See also: [Backend Optimization, Connection Pooling](./BackendOptmization.md)

---

## 16. Sharding and Cross-Shard Transactions

### Core mechanism

**Sharding** splits data across several independent database clusters. Each shard holds a subset of accounts. You shard when one primary cannot handle the write rate or storage, even after the steps in Section 15.

Shard key choice for a wallet: **account ID** (or user ID, if a user's accounts must be on one shard). Everything for one account (ledger entries, holds, idempotency keys) lives on its shard, so single-account operations stay local.

The hard part: a P2P transfer between users on **different shards** is a distributed transaction.

### Engineering blueprint

Options for cross-shard transfers:

**Option A: Saga with holds (most common)**

```text
1. Shard A (sender):   TX  debit sender available, credit "outbound_in_transit"  + outbox
2. Shard B (receiver): TX  (consume event, inbox dedupe) credit receiver, debit "inbound_in_transit" + outbox
3. Shard A:            TX  mark transfer succeeded; clear in-transit
   Failure at step 2 (receiver account closed): Shard A compensates → refund sender from in-transit.
```

- Each shard's books still balance because in-transit accounts hold the value while it moves.
- The sender sees "sent" after step 1 (money left their available balance); the receiver sees it after step 2, typically milliseconds to seconds later.
- Reconciliation checks that, globally, `outbound_in_transit` on all shards equals `inbound_in_transit` plus pending items.

**Option B: Distributed SQL** (CockroachDB, Spanner, YugabyteDB, TiDB): write normal transactions; the database runs the commit protocol. Simplest code, higher per-commit latency, and you must design keys to avoid hotspots (no sequential primary keys).

**Option C: Purpose-built ledger databases** (for example TigerBeetle) that batch and process very high transfer rates with strict double-entry semantics.

Routing:

```text
shard = lookup(account_id)        // directory service or consistent hashing over virtual shards
                                  // use many logical shards (e.g. 1024) mapped onto fewer physical clusters
                                  // so you can move logical shards without rehashing everything
```

### Production pitfalls

- Sharding by user when the main hot path is between users. Every P2P transfer becomes cross-shard. Measure what fraction of transactions would cross shards before choosing a key.
- Resharding without a plan. Moving accounts between shards while transfers are in flight requires a migration protocol (dual writes or freeze-copy-switch per account).
- Global queries ("total balance of all users") now span all shards. Serve them from a warehouse or aggregated stream, not live fan-out.
- Global uniqueness (idempotency keys, emails) can no longer rely on one unique index.

### SDE best practices

- Delay sharding as long as possible; design the schema so it is **shardable** (every table has the shard key, no cross-account foreign keys on hot paths).
- Use logical shards from day one, even if they all live on one physical database.

### Resources

- Vitess (MySQL sharding) concepts — https://vitess.io/docs/concepts/
- Citus (PostgreSQL distributed tables) — https://docs.citusdata.com/
- CockroachDB transactions layer — https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer
- TigerBeetle design — https://docs.tigerbeetle.com/concepts/
- Stripe, "How Stripe's document databases supported 99.999% uptime with zero-downtime data migrations" — https://stripe.com/blog/how-stripes-document-databases-supported-99.999-uptime-with-zero-downtime-data-migrations

---

## 17. Hot Accounts and Contention

### Core mechanism

Row locks serialize all writers to the same row. If every transaction touches one row, throughput for that row is roughly `1 / (lock hold time)`. With a 5 ms lock hold, one row handles about 200 transactions per second no matter how big the server is.

Typical hot accounts:

- System accounts: `fees`, `provider_clearing`, `promotions`. Every transaction credits them.
- Popular merchants or creators during a sale or live event.
- Flash-sale inventory counters.

### Engineering blueprint

**1. Do not update cached balances for system accounts on the hot path.** Write ledger entries (inserts do not contend on the same row), and compute system account balances asynchronously or on demand. Only accounts that need a synchronous "do not go negative" check need a locked balance row.

**2. Split the account into N sub-accounts (sharded counter).**

```sql
-- fees account becomes 32 buckets
INSERT INTO ledger_entries (..., account_id, ...) VALUES (..., fee_bucket_id(hash(transfer_id) % 32), ...);
-- balance = SUM over buckets
```

For accounts that must not go negative (a merchant paying out), split available balance across buckets and draw from a random bucket that has enough funds; rebalance in the background.

**3. Batch.** Collect many credits to a hot account in a short window (for example 50 ms) and apply them in one transaction: one lock acquisition for hundreds of transfers. This adds a little latency for a large throughput gain (this is the core idea behind TigerBeetle-style batching).

**4. Keep the lock window tiny.** Lock the hot row **last** in the transaction (after all other work), so it is held for the shortest time. Never do network calls while it is locked.

**5. Inventory counters**: use a conditional atomic update (`UPDATE stock SET qty = qty - 1 WHERE id = $1 AND qty > 0`), or pre-split stock into buckets, or front it with Redis atomic decrements plus durable reconciliation to the DB.

### Production pitfalls

- "Lock the hot row last" conflicting with "lock in global id order" — define the global order so system accounts come last.
- Measuring average latency only; hot-row contention shows up in p99 latency and `lock_timeout` errors.
- Redis-based counters without a durable source of truth. A Redis failover can lose recent decrements → oversell.

### SDE best practices

- Find hot rows with `pg_stat_activity` (`wait_event_type = 'Lock'`) and `pg_locks`; log lock waits with `log_lock_waits = on`.
- Load-test the "everyone pays one merchant" scenario before launch events.

### Resources

- PostgreSQL `log_lock_waits` and lock monitoring — https://www.postgresql.org/docs/current/monitoring-stats.html
- Modern Treasury, "How to Scale a Ledger" series (hot accounts, batching) — https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-i
- TigerBeetle, "Why TigerBeetle" (contention and batching) — https://docs.tigerbeetle.com/concepts/

---

## 18. Read Paths: History, Balances, and CQRS

### Core mechanism

Writes need strong consistency; most reads do not. **CQRS** (Command Query Responsibility Segregation) separates the write model (ledger, transfers) from read models built for specific screens (activity feed, monthly statements, search, analytics).

| Read | Consistency needed | Source |
| --- | --- | --- |
| Balance shown before "Send" | Strong (or clearly labeled) | Primary, or replica + server re-check on submit |
| Balance check inside the transfer | Strong | Primary, inside the transaction |
| Activity feed | Seconds-stale is fine | Replica or read model |
| Search ("Dinner", amount range) | Seconds-stale is fine | Search index fed by CDC/outbox |
| Monthly statement | Exact, but for a closed period | Precomputed from ledger after period close |
| Admin dashboards / finance | Minutes-stale is fine | Warehouse |

### Engineering blueprint

- Build read models from **events** (outbox/CDC). Each event has a version; the read model ignores older versions to handle out-of-order delivery.
- Show balance from the primary or a short-TTL cache, and always **re-check on the server** at transaction time. The UI number is informational.
- Activity feed with enrichment: denormalize counterparty names/avatars into the feed row at write time (or cache them) rather than joining across services at read time.
- Cache carefully: cache history pages per account with explicit invalidation on new transfers; do not cache balances for decision-making. See [Caching Techniques, Consistency and Staleness](./CachingTechniques.md#10-consistency-and-staleness).

### Production pitfalls

- Users panic when a payment they just made is missing from history (replica lag). Insert the `POST` response into the client cache optimistically **after** the server confirms (Section 19).
- Read models that silently stop updating when a consumer crashes. Alert on lag.
- Statement totals that differ from balances because pending/holds are counted differently. Define and document "available", "pending", and "ledger" balances.

### SDE best practices

- Show three numbers where relevant: **available** (spendable now), **pending** (holds and in-flight), and **current/ledger** balance.
- Rebuildability: you should be able to drop a read model and rebuild it from the ledger/events.

### Resources

- Martin Fowler, CQRS — https://martinfowler.com/bliki/CQRS.html
- microservices.io, CQRS — https://microservices.io/patterns/data/cqrs.html

---
## 19. Frontend: Transaction UX and Double-Submit Safety

### Core mechanism

The frontend is where most duplicate transactions start: double clicks, the Enter key pressed twice, impatient refreshes, mobile apps retrying after a network switch, and users pressing "Pay" again after a spinner that never ends. The frontend's job:

1. Create **one idempotency key per user intent**, and keep it stable across retries, re-renders, and page reloads.
2. Prevent duplicate submissions locally (but rely on the server for correctness).
3. Model the outcome honestly, including an **unknown** state.
4. Never show "success" until the server confirms it.

```text
UI state machine for "Send money"

 editing ──submit──> submitting ──201 succeeded──────────────> succeeded (terminal)
    ^                   │  ├──202 processing──> processing ──poll/push──> succeeded / failed
    │                   │  ├──4xx business error───────────> failed (show reason; edit → NEW key)
    │                   │  └──timeout / network / 5xx──────> unknown
    │                   │                                       │ retry POST with SAME key
    │                   │                                       │ or GET ?idempotency_key=
    └──user edits amount/recipient (only from editing/failed) ──┘ never from unknown
```

### Engineering blueprint

**A stable idempotency key, scoped to the draft:**

```tsx
import { useCallback, useState } from 'react';

// One key per user intent. Survives re-renders and reloads (sessionStorage).
// Changing the draft (amount, recipient) => different intent => different key.
export function useIdempotencyKey(draftId: string) {
  const storageKey = `idem:${draftId}`;
  const [key, setKey] = useState(() => {
    try {
      const existing = sessionStorage.getItem(storageKey);
      if (existing) return existing;
    } catch { /* storage blocked: fall back to memory */ }
    const fresh = crypto.randomUUID();
    try { sessionStorage.setItem(storageKey, fresh); } catch {}
    return fresh;
  });

  // Call only after a TERMINAL outcome, when the user starts a new intent.
  const rotate = useCallback(() => {
    const fresh = crypto.randomUUID();
    try { sessionStorage.setItem(storageKey, fresh); } catch {}
    setKey(fresh);
  }, [storageKey]);

  return [key, rotate] as const;
}
```

A cleaner option for important flows: have the server create a **draft** first (`POST /v1/transfer-drafts` → `draft_id`) when the confirm screen opens, then `POST /v1/transfer-drafts/{id}/confirm`. The draft ID is the idempotency key and is stored server-side, so it also survives a device change.

**Submit with a synchronous guard, retries with the same key, and an unknown state:**

```tsx
type Phase =
  | { kind: 'editing' }
  | { kind: 'submitting' }
  | { kind: 'processing'; transferId: string }
  | { kind: 'succeeded'; transfer: Transfer }
  | { kind: 'failed'; code: string; message: string }
  | { kind: 'unknown' };

const RETRYABLE_STATUS = new Set([409, 429, 500, 502, 503, 504]);

async function postTransfer(draft: Draft, key: string): Promise<Response> {
  return fetch('/v1/transfers', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Idempotency-Key': key },
    body: JSON.stringify(draft),
    credentials: 'include',
    signal: AbortSignal.timeout(15_000),
  });
}

export function SendMoney({ draft }: { draft: Draft }) {
  const [phase, setPhase] = useState<Phase>({ kind: 'editing' });
  const [key, rotateKey] = useIdempotencyKey(draftId(draft));
  const inFlight = useRef(false); // state updates are async; a ref blocks the 2nd click immediately

  async function submit() {
    if (inFlight.current) return;
    inFlight.current = true;
    setPhase({ kind: 'submitting' });
    try {
      for (let attempt = 1; attempt <= 3; attempt++) {
        try {
          const res = await postTransfer(draft, key);   // SAME key on every attempt
          if (res.status === 201) return setPhase({ kind: 'succeeded', transfer: await res.json() });
          if (res.status === 202) return setPhase({ kind: 'processing', transferId: (await res.json()).id });
          if (!RETRYABLE_STATUS.has(res.status)) {
            const { error } = await res.json();
            return setPhase({ kind: 'failed', code: error.code, message: error.message });
          }
          const retryAfter = Number(res.headers.get('Retry-After')) || 0;
          await sleep(Math.max(retryAfter * 1000, 500 * 2 ** attempt));
        } catch {
          // network error or timeout: outcome unknown; retrying with the same key is safe
          await sleep(500 * 2 ** attempt);
        }
      }
      setPhase({ kind: 'unknown' });   // UI: "We're confirming your payment…" + status lookup
    } finally {
      inFlight.current = false;
    }
  }
  // render: button disabled unless phase is editing; aria-live region announces phase changes
}
```

**Recovering from `unknown`:** look up by key instead of creating a new transfer.

```ts
const res = await fetch(`/v1/transfers?idempotency_key=${encodeURIComponent(key)}`);
// found => show its status; not found after the server's in-progress window => safe to retry POST with same key
```

**Warn on navigation while in flight:**

```ts
useEffect(() => {
  if (phase.kind !== 'submitting') return;
  const handler = (e: BeforeUnloadEvent) => { e.preventDefault(); };
  window.addEventListener('beforeunload', handler);
  return () => window.removeEventListener('beforeunload', handler);
}, [phase.kind]);
```

**Money in the UI**: keep amounts as minor-unit strings or `BigInt`, never `number` arithmetic on decimals.

```ts
// Display: minor units -> localized string. Intl.NumberFormat.format accepts decimal strings in
// modern engines, which avoids float rounding.
export function formatMinor(minor: string, currency: string, locale?: string) {
  const fmt = new Intl.NumberFormat(locale, { style: 'currency', currency });
  const digits = fmt.resolvedOptions().maximumFractionDigits ?? 2;
  const v = BigInt(minor);
  const abs = v < 0n ? -v : v;
  const base = 10n ** BigInt(digits);
  const decimal = `${v < 0n ? '-' : ''}${abs / base}${digits ? '.' + (abs % base).toString().padStart(digits, '0') : ''}`;
  return fmt.format(decimal as unknown as number);
}

// Input: "12.5" -> "1250" (USD). Reject more fraction digits than the currency allows.
export function parseToMinor(input: string, digits: number): string | null {
  const m = /^(\d+)(?:\.(\d*))?$/.exec(input.trim());
  if (!m || (m[2]?.length ?? 0) > digits) return null;
  return (BigInt(m[1]) * 10n ** BigInt(digits) + BigInt((m[2] ?? '').padEnd(digits, '0') || '0')).toString();
}
```

(Locale-aware input — "12,50" in many European locales — needs a locale-aware parser; keep the server as the final validator.)

**Optimistic UI rules for money:**

| OK to do optimistically | Never do optimistically |
| --- | --- |
| Add a "Pending" row to the activity list | Show "Payment sent ✓" |
| Gray out / reduce displayed available balance with a "pending" label | Treat local balance as authoritative for the next transfer |
| Prefill the next screen | Hide the error path after a rollback |

With TanStack Query, use `onMutate` to insert a pending row, `onSuccess` to replace it with the server's resource (this also solves replica lag in Section 18), and `onError` to mark it failed rather than silently removing it.

### Production pitfalls

- `disabled={isSubmitting}` alone. Two clicks within one render frame, or a form submitted by Enter, can both fire before React re-renders. Use a ref guard (above) or the server-draft pattern.
- New key on retry ("Try again" button calls `crypto.randomUUID()`). That turns a retry into a second payment.
- Retrying automatically on `4xx` business errors. `INSUFFICIENT_FUNDS` will not change by retrying.
- Showing "Payment failed" on timeout. The money may have moved. Show "We're confirming…" and resolve the real state.
- Client-computed totals (cart total, fees, FX amount) sent to the server and trusted. The server must recompute; the client may send the **expected** total so the server can reject if prices changed ("price changed, please confirm").
- Losing the key on reload during 3-D Secure redirects. Persist it (sessionStorage or server draft).

### SDE best practices

- Show a **confirmation screen** with exact amount, currency, fees, recipient name, and arrival time before submit.
- Make every phase accessible: `aria-live="polite"` for status, focus moves to the result, errors are text (not only color).
- Track frontend metrics: double-submit attempts blocked, `unknown` outcomes, time to terminal state.

### Resources

- MDN, `crypto.randomUUID()` — https://developer.mozilla.org/en-US/docs/Web/API/Crypto/randomUUID
- MDN, `AbortSignal.timeout()` — https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static
- MDN, `Intl.NumberFormat` — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/NumberFormat
- TanStack Query, Optimistic Updates — https://tanstack.com/query/latest/docs/framework/react/guides/optimistic-updates
- See also: [Frontend guide](./Frontend.md)

---

## 20. Frontend: Status Updates, Payment Forms, and PCI Scope

### Core mechanism

Many transactions finish **after** the first response (`202 processing`): bank transfers, payouts, 3-D Secure, fraud review. The client needs a way to learn the final state, and the payment form must keep card data away from your servers.

**Status update options:**

| Method | How | Good for | Cost at 1M users |
| --- | --- | --- | --- |
| Polling with backoff | `GET /transfers/{id}` at 1s, 2s, 4s… cap 10s, stop after ~2 min | Short waits; simplest | Small: only users with an in-flight transfer poll |
| Server-Sent Events (SSE) | One HTTP stream per open tab; server pushes `transfer.updated` | Live activity feeds | One open connection per active tab; needs a fan-out layer (Redis pub/sub, managed service) |
| WebSockets | Bidirectional connection | Chat-like real-time apps | Same as SSE plus more state |
| Mobile push / email | APNs / FCM / email triggered by outbox events | Long waits (hours/days) | Cheap; works when app is closed |

Default: **poll for short waits, push notification for long waits**, SSE/WebSockets only if real-time UI is a product requirement.

### Engineering blueprint

Polling with backoff and a stop condition:

```ts
export async function waitForTerminal(id: string, signal: AbortSignal): Promise<Transfer | 'timeout'> {
  const deadline = Date.now() + 120_000;
  for (let delay = 1000; Date.now() < deadline; delay = Math.min(delay * 2, 10_000)) {
    const res = await fetch(`/v1/transfers/${id}`, { signal });
    if (res.ok) {
      const t: Transfer = await res.json();
      if (['succeeded', 'failed', 'reversed'].includes(t.status)) return t;
    }
    await sleep(delay * (0.5 + Math.random() / 2));  // jitter avoids synchronized polling
  }
  return 'timeout';  // UI: "This is taking longer than usual. We'll notify you."
}
```

**Card forms and PCI DSS scope:**

- Use the provider's hosted fields / iframes (Stripe Elements or Payment Element, Adyen Drop-in, Braintree Hosted Fields). Raw card numbers go from the browser directly to the provider. Your servers receive only a token. This keeps most of your system out of PCI DSS scope.
- The **payment page itself** is still in scope for script integrity. PCI DSS v4.x added requirements for inventorying, authorizing, and monitoring scripts on payment pages (6.4.3, 11.6.1). Which ones apply depends on your integration and SAQ type; the SAQ A eligibility criteria were revised in 2025. Confirm against current PCI SSC documents with your compliance team.
- Defend the payment page: strict Content-Security-Policy, Subresource Integrity for static scripts, no third-party analytics or tag managers on the payment page, `frame-ancestors` to prevent clickjacking.

**Strong Customer Authentication (SCA) / 3-D Secure 2**: required for many card payments in the EEA/UK (PSD2) and used elsewhere for fraud reduction. The flow returns `requires_action`; the provider SDK shows the bank challenge (modal or redirect). After a redirect, the return URL contains a payment reference. **Do not trust it.** Fetch the status from your server, which checks with the provider.

```ts
// Stripe example sketch
const { error } = await stripe.confirmPayment({
  elements,
  confirmParams: { return_url: `${location.origin}/checkout/complete?order=${orderId}` },
});
if (error) showError(error.message);     // card declined, validation error, etc.
// On /checkout/complete: GET /v1/orders/{orderId} from YOUR backend to read the real status.
```

**Mobile and offline:**

- Persist the idempotency key with the queued request in local storage (or the app database).
- Server-side, reject stale intents (for example, a queued transfer replayed 3 days later) with a clear error, so an old offline action does not move money unexpectedly.
- Re-authenticate for high-value actions when the app resumes.

### Production pitfalls

- Polling every 500 ms forever from every open tab. Add backoff, jitter, a stop condition, and pause polling when the tab is hidden (`document.visibilityState`).
- Loading the whole marketing tag stack on the checkout page. Each script can read keystrokes on the page (Magecart-style skimming).
- Building your own card input "because the iframe is ugly". This pulls your servers into full PCI scope.
- Trusting `?status=success` on the return URL.

### SDE best practices

- One source of truth for status: your backend. Every client channel (poll, SSE, push) reads from it.
- Keep the payment page minimal: first-party code plus the provider SDK.

### Resources

- PCI Security Standards Council document library — https://www.pcisecuritystandards.org/document_library/
- Stripe, Payment Element — https://docs.stripe.com/payments/payment-element
- Stripe, 3D Secure authentication — https://docs.stripe.com/payments/3d-secure
- MDN, Server-sent events — https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events
- MDN, Content Security Policy — https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- MDN, Subresource Integrity — https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity

---

## 21. Security, Abuse, and Fraud Controls

### Core mechanism

A transaction system is a direct target: attackers want to move money out, use stolen cards, or exploit race conditions. Security controls sit at every layer.

| Threat | Example | Control |
| --- | --- | --- |
| Broken object-level authorization (BOLA/IDOR) | `POST /transfers` with someone else's `from_account_id` | Check ownership of **every** account ID on the server, inside the transaction |
| Race-condition exploits | 50 parallel requests to redeem a $10 promo once, or withdraw more than the balance | Atomic conditional updates, unique constraints, row locks (Sections 2-3) |
| Account takeover | Stolen password → drain wallet | MFA, step-up auth (WebAuthn/passkey) for new payees and high amounts, new-device cool-off |
| Card testing | Bots try thousands of stolen cards with $1 charges | Rate limits per IP/device/account, CAPTCHA on anomalies, provider fraud tools (Radar, etc.) |
| Amount / parameter tampering | Client sends `amount: -2000` or `fee: 0` | Server-side validation and recomputation; `CHECK (amount_minor > 0)` |
| CSRF | Malicious site triggers transfer with user's cookies | `SameSite` cookies, CSRF tokens, or bearer tokens not sent automatically |
| Replay of signed requests / webhooks | Old webhook resent by attacker | Signature with timestamp tolerance + event ID dedupe |
| Insider risk | Engineer edits a balance | Append-only ledger, DB role separation, audited admin tools, four-eyes approvals |

### Engineering blueprint

- **Authorization inside the transaction**: `SELECT ... FROM accounts WHERE id = $1 AND owner_id = $user FOR NO KEY UPDATE`. If zero rows: `404`.
- **Limits** as data: per-user daily/monthly limits, per-transaction max, velocity limits (count and amount per hour). Enforce them in the same transaction as the transfer, using a locked per-user limits row or a `SUM` over recent transfers at SERIALIZABLE.
- **Risk scoring** before the money moves: synchronous rules (fast, deterministic) + an ML score with a strict timeout and a safe default (for example, "allow small, review large" if the model is down).
- **Step-up authentication** (see [Authentication Concepts](./AuthenticationConcepts.md)): require a fresh second factor for new payees, unusual amounts, or new devices. Bind the step-up proof to the specific transaction (amount + payee), not just the session.
- **Holds and delays** for new accounts: first payouts held for N days.

### Production pitfalls

- Checking limits with a non-locking `SELECT SUM(...)` at READ COMMITTED. Parallel requests all see the same sum and all pass (write skew).
- Rate limiting only by IP. Attackers rotate IPs; add per-account and per-device limits.
- Logging full card numbers, CVV, bank credentials, or auth tokens. Mask PANs; never log CVV.

### SDE best practices

- Write a "race condition" test for every limit and every one-time action.
- Treat fraud rules as configuration with audit history, so they can change without a deploy.

### Resources

- OWASP API Security Top 10 — https://owasp.org/API-Security/
- OWASP Business Logic / race conditions testing — https://owasp.org/www-project-web-security-testing-guide/
- PortSwigger Web Security Academy, Race conditions — https://portswigger.net/web-security/race-conditions
- Stripe Radar documentation — https://docs.stripe.com/radar

---

## 22. Observability and SLOs

### Core mechanism

For transaction systems, "the API is up" is not enough. You must observe **correctness** (money conserved, nothing stuck, nothing duplicated) as well as **performance**.

### Engineering blueprint

**Business and correctness metrics:**

| Metric | Alert when |
| --- | --- |
| Transfer success rate by type, provider, and error code | Drops below baseline; any spike in a single error code |
| Transfers in non-terminal state older than SLA | > 0 for money-moving types |
| Ledger imbalance query results | > 0 (page immediately) |
| Balance cache drift | > 0 |
| Reconciliation mismatches (count and amount) | > 0 unresolved after N hours |
| Idempotent replays per minute | Sudden spikes (client retry storm or bug) |
| `unknown` outcomes reported by clients | Rising |

**System metrics:**

| Metric | Source |
| --- | --- |
| p50/p95/p99 latency for create transfer | API metrics / tracing |
| Serialization failures (`40001`) and deadlocks (`40P01`) per second | App metrics; `pg_stat_database.deadlocks` |
| Lock waits, longest transaction age | `pg_stat_activity`, `pg_locks`, `log_lock_waits` |
| Pool wait time and saturation | App pool + PgBouncer `SHOW POOLS` |
| Replica lag | `pg_stat_replication` |
| Outbox backlog (unpublished rows, oldest age) | `SELECT count(*), min(created_at) FROM outbox WHERE published_at IS NULL` |
| Consumer lag | Kafka consumer group lag |
| Webhook processing lag | `provider_events` unprocessed age |

**Tracing**: propagate a trace ID (W3C `traceparent`) from the browser through the API, the DB transaction, the outbox event, and consumers. Put `transfer_id` and `idempotency_key` on spans and logs so one search shows the whole life of a transaction.

**SLO examples:**

- 99.9% of `POST /transfers` return a non-5xx response within 1 s (monthly).
- 99.99% of P2P transfers reach a terminal state within 60 s.
- 100% of ledger imbalance checks pass (this is an invariant, not an SLO: any failure is an incident).

### Production pitfalls

- Logging PII and secrets in transaction logs. Use structured logs with explicit allowlisted fields.
- Averages hiding hot-account contention. Use percentiles per account type.
- No alert on "nothing happening" (for example, zero successful payouts in the last hour during business hours).

### SDE best practices

- Build a runbook per alert: what it means, how to find affected transactions, how to mitigate.
- Have an admin tool that shows a transaction's full timeline: API request, state transitions, ledger entries, events, webhooks, provider objects.

### Resources

- Google SRE Book, "Service Level Objectives" — https://sre.google/sre-book/service-level-objectives/
- PostgreSQL monitoring statistics — https://www.postgresql.org/docs/current/monitoring-stats.html
- OpenTelemetry documentation — https://opentelemetry.io/docs/
- W3C Trace Context — https://www.w3.org/TR/trace-context/

---

## 23. Testing Transactional Systems

### Core mechanism

Unit tests with mocked databases cannot catch most transaction bugs: races, isolation anomalies, deadlocks, and partial failures only appear against a **real database** under **real concurrency**. Test the invariants, not only the happy path.

### Engineering blueprint

**1. Integration tests against real PostgreSQL** (Testcontainers or a Docker Compose DB), same major version as production.

**2. Concurrency tests:**

```ts
test('same idempotency key fired 20x concurrently => exactly one transfer', async () => {
  const key = crypto.randomUUID();
  const body = { from_account_id: alice, to_account_id: bob, amount: { value: '1000', currency: 'USD' } };
  const responses = await Promise.all(
    Array.from({ length: 20 }, () => api.post('/v1/transfers', body, { headers: { 'Idempotency-Key': key } })),
  );
  const ids = new Set(responses.filter((r) => r.status === 201).map((r) => r.data.id));
  expect(ids.size).toBe(1);                                   // every success is the same transfer
  expect(responses.every((r) => [201, 409].includes(r.status))).toBe(true);
  expect(await ledgerEntryCount({ account: alice })).toBe(1);
});

test('parallel withdrawals never overdraw', async () => {
  await setBalance(alice, 10_000n);                            // $100
  await Promise.all(
    Array.from({ length: 50 }, () =>
      api.post('/v1/transfers', { from_account_id: alice, to_account_id: bob, amount: { value: '1000', currency: 'USD' } },
               { headers: { 'Idempotency-Key': crypto.randomUUID() } }).catch((e) => e.response)),
  );
  expect(await balance(alice)).toBeGreaterThanOrEqual(0n);
  expect(await balance(alice) + await balance(bob)).toBe(10_000n + initialBob);   // conservation
  await expectNoLedgerImbalance();
});
```

**3. Property-based tests**: generate random sequences of operations (transfers, reversals, holds, captures) and check invariants after each step: sum of all entries per currency is constant, no balance below floor, every transfer sums to zero. Libraries: fast-check (JS), Hypothesis (Python), jqwik (Java).

**4. Fault injection:**

- Kill the process between "provider call succeeded" and "DB updated" → verify the sweeper/webhook recovers.
- Drop the connection during `COMMIT` (Toxiproxy) → verify the client retry with the same key does not double-apply.
- Deliver every webhook and Kafka message twice, and out of order.
- Stop the outbox relay for an hour, restart → verify all events publish, no duplicates in consumers' effects.

**5. Load tests** at expected peak × 2-3, including the hot-account scenario:

```js
// k6 sketch
import http from 'k6/http';
import { check } from 'k6';
import { uuidv4 } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

export const options = {
  scenarios: { transfers: { executor: 'constant-arrival-rate', rate: 1000, timeUnit: '1s',
                            duration: '10m', preAllocatedVUs: 300 } },
  thresholds: { http_req_failed: ['rate<0.001'], http_req_duration: ['p(99)<500'] },
};

export default function () {
  const res = http.post(`${__ENV.BASE}/v1/transfers`, JSON.stringify(randomTransfer()), {
    headers: { 'Content-Type': 'application/json', 'Idempotency-Key': uuidv4(), Authorization: token() },
  });
  check(res, { 'created or business-rejected': (r) => r.status === 201 || r.status === 422 });
}
```

After every load test, run the reconciliation queries. A load test that "passes" on latency but leaves an imbalance is a failure.

**6. Frontend tests**: Playwright tests that double-click "Pay", press Enter twice, go offline mid-request (`context.setOffline(true)`), and reload during the `submitting` phase. Assert one transfer on the server.

### Production pitfalls

- Using SQLite or an in-memory fake in tests while production runs PostgreSQL. Isolation and locking behavior differ.
- Wrapping each test in a transaction that rolls back. Your code's own transactions and concurrency then do not behave like production.
- Testing only with sequential requests.

### SDE best practices

- Keep a dedicated "money invariants" test suite that runs in CI on every change to the transaction paths.
- Read Jepsen reports for the databases and queues you use; they show real failure modes.

### Resources

- Testcontainers — https://testcontainers.com/
- fast-check (property-based testing) — https://fast-check.dev/
- Toxiproxy — https://github.com/Shopify/toxiproxy
- Grafana k6 — https://grafana.com/docs/k6/latest/
- Jepsen analyses — https://jepsen.io/analyses
- See also: [Testing Framework guide](./TestingFramework.md)

---

## 24. Capacity Planning for a Million Users

### Core mechanism

Turn "a million users" into numbers. The numbers usually show that **one well-run relational database is enough for the write path**, and that the real work is correctness, spikes, and hot accounts.

### Engineering blueprint

Worked example (change the assumptions for your product):

| Input | Assumption | Result |
| --- | --- | --- |
| Registered users | 1,000,000 | |
| Daily active users | 30% | 300,000 DAU |
| Money transactions per active user per day | 3 | 900,000 transfers/day |
| Average write rate | 900,000 / 86,400 s | **≈ 10 TPS** |
| Daily peak factor | ~3× average | ≈ 30 TPS |
| Event peak (payday, flash sale, live event) | 20-50× average | **≈ 200-500 TPS** |
| Design target (headroom 2-4×) | | **1,000-2,000 TPS** writes |
| Reads (app opens × API calls) | 300k × 5 opens × 4 calls | 6M/day ≈ 70 RPS avg, ~2-3k RPS peak |

**DB connections (Little's Law)**: busy connections = throughput × time per transaction.

```text
2,000 TPS × 10 ms per transaction  = 20 connections busy on average
2,000 TPS × 50 ms (with lock waits) = 100 connections busy
```

So a PgBouncer pool of ~50-100 server connections is plenty. Hundreds of connections would make it slower, not faster.

**Storage growth** (rough, including indexes):

```text
per transfer: transfers row + 2 ledger entries + transfer_events + idempotency + outbox ≈ 2-4 KB
900k/day × 3 KB ≈ 2.7 GB/day ≈ 1 TB/year (idempotency/outbox purged after days, so less long-term)
```

This is why ledger partitioning and archival matter from year one, while sharding does not.

**Scaling path by order of magnitude:**

| Scale | Typical write peak | Architecture |
| --- | --- | --- |
| ~1M users | hundreds of TPS | One PostgreSQL primary + HA standby + 1-3 replicas, PgBouncer, partitioned ledger, outbox + Kafka/SQS |
| ~10M users | thousands of TPS | Bigger primary, aggressive partitioning, hot-account techniques, CQRS read models, separate ledger service/DB |
| ~100M+ users | tens of thousands of TPS+ | Sharded Postgres/MySQL (Citus, Vitess, custom), or distributed SQL, or a purpose-built ledger DB; cross-shard sagas |

**Availability math**: a 99.99% target allows ~4.3 minutes of downtime per month. Plan for: automatic failover (Patroni or a managed service such as RDS/Aurora/Cloud SQL), multi-AZ synchronous standby (no committed transaction lost on node failure), and tested restores.

### Production pitfalls

- Planning for average load. Money traffic is spiky (salary day, Black Friday, viral events).
- Scaling app pods horizontally without changing DB connection limits. More pods → more connections → DB collapse.
- Ignoring the cost of retries during incidents: a 5% error rate with 3 client retries adds ~15% load exactly when the system is weakest.

### SDE best practices

- Write the capacity model in a doc with assumptions; revisit quarterly with real numbers.
- Add **load shedding**: when the DB is saturated, reject new non-critical work quickly (`503` + `Retry-After`) instead of letting queues grow until everything times out.

### Resources

- Little's Law explained — https://en.wikipedia.org/wiki/Little%27s_law
- Patroni (PostgreSQL HA) — https://patroni.readthedocs.io/
- AWS Builders' Library, "Using load shedding to avoid overload" — https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/
- AWS Builders' Library, "Timeouts, retries, and backoff with jitter" — https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/

---

## 25. Reference Architecture

### Core mechanism

This architecture combines the previous sections for a wallet + card top-up product at roughly 1-10M users.

```text
                     ┌──────────────────────────── Clients ─────────────────────────────┐
                     │ React web app (hosted card fields) · iOS/Android apps             │
                     └───────────────┬───────────────────────────────────────────────────┘
                                     │ HTTPS, Idempotency-Key, traceparent
                           ┌─────────▼─────────┐
                           │ CDN + WAF + API GW │  TLS, bot rules, coarse rate limits, auth
                           └─────────┬─────────┘
          ┌──────────────────────────┼─────────────────────────────┐
  ┌───────▼───────┐        ┌─────────▼─────────┐         ┌─────────▼─────────┐
  │ Auth / Identity│        │  Transfer API      │         │ Webhook receiver   │
  │ (MFA, step-up) │        │ (stateless pods)   │         │ (verify, store,200)│
  └────────────────┘        └──┬──────┬──────────┘         └─────────┬─────────┘
                               │      │ sync, strict timeout         │
                               │  ┌───▼────────┐                     │
                               │  │ Risk/limits │                     │
                               │  └────────────┘                     │
                      ┌────────▼─────────────────────────────────────▼───────┐
                      │ PgBouncer → PostgreSQL primary (ledger, transfers,    │
                      │ idempotency, outbox, provider_events)                 │
                      │ sync standby (HA) · async read replicas               │
                      └────────┬──────────────────────────────────────────────┘
                               │ outbox relay / Debezium CDC
                      ┌────────▼────────┐
                      │ Kafka / SQS/SNS │
                      └─┬──────┬──────┬─┘
          ┌─────────────▼┐ ┌───▼──────────┐ ┌▼───────────────────┐ ┌──────────────────┐
          │ Notifications│ │ Read models  │ │ Saga orchestrator / │ │ Warehouse (CDC)  │
          │ push · email │ │ feed, search │ │ workers (payouts,   │ │ finance, BI      │
          └──────────────┘ └──────────────┘ │ sweeper, completer) │ └──────────────────┘
                                            └─────────┬──────────┘
                                                      │ idempotency key = our id
                                             ┌────────▼────────┐
                                             │ Payment provider │ ◄── daily reports ──┐
                                             └─────────────────┘                      │
                                                     ┌────────────────────────────────┴┐
                                                     │ Reconciliation jobs + ops console │
                                                     └──────────────────────────────────┘
```

### Engineering blueprint

**Latency budget for a P2P transfer (target p99 < 500 ms):**

| Step | Budget |
| --- | --- |
| Client → edge → API (network, TLS) | 50-150 ms |
| Auth token verify (local JWT verify or cached introspection) | < 5 ms |
| Risk / limits check | < 50 ms (timeout → safe default) |
| DB transaction (idempotency + locks + entries + outbox) | 5-30 ms typical, 100 ms p99 |
| Response serialization and return | < 20 ms |

**Failure-mode table** (practice explaining each in interviews):

| Failure | What users see | Why money stays correct |
| --- | --- | --- |
| API pod crashes mid-request before commit | Client times out → retries with same key → success | Transaction rolled back; retry executes once |
| API pod crashes after commit, before response | Client retries → gets replayed response | Idempotency record committed with the transfer |
| Primary DB fails | Seconds of errors during failover; retries succeed | Synchronous standby has every committed txn |
| Outbox relay down | Notifications/history delayed | Events wait in outbox; published on recovery |
| Kafka consumer processes twice | Nothing | Inbox dedupe |
| Provider times out on charge | "Confirming your payment…" | Payment stays `processing`; retry with same provider key; webhook/sweeper resolves |
| Webhook lost | Small delay | Sweeper polls provider; daily reconciliation catches the rest |
| Risk service down | Small transfers proceed; large ones queued for review | Explicit fallback policy |
| Hot merchant during a sale | Slightly higher latency | Bucketed accounts / batching keep lock waits short |

### Production pitfalls

- Architecture diagrams with no failure story. For every arrow, ask: what if this call times out, is repeated, or arrives late?
- Too many services too early. A modular monolith with one database is easier to keep correct; split along consistency boundaries later.

### SDE best practices

- Start as a modular monolith: `ledger`, `transfers`, `payments`, `risk`, `notifications` modules with clear interfaces and one database. Extract services only when team size or scaling forces it.
- Keep the ledger the most boring, most tested, least frequently changed component.

### Resources

- AWS Builders' Library (reliability articles) — https://aws.amazon.com/builders-library/
- Uber Engineering, "Gulfstream: Uber's payment platform" (search the Uber blog for current payment platform posts) — https://www.uber.com/blog/engineering/
- Stripe engineering blog — https://stripe.com/blog/engineering

---

## 26. Interview Questions and Study Plan

### Interview questions (with the key points to hit)

1. **Two users click "send" on the same account at the same time with $100 balance and $80 each. What happens?** Lost update / overdraft risk at READ COMMITTED with read-compute-write. Fixes: atomic conditional `UPDATE ... WHERE balance >= amount`, `FOR UPDATE`, or SERIALIZABLE + retry. Add a `CHECK` constraint as the last defense.
2. **The client times out on a payment request. Should it retry?** Yes, with the same idempotency key. Explain how the server stores key + request hash + response in the same transaction, and the 409/422 cases.
3. **How do you prevent deadlocks in transfers?** Lock accounts in a global order (ascending ID), keep transactions short, set `lock_timeout`, retry on `40P01`.
4. **Explain write skew and how to prevent it.** Two transactions read overlapping data and write different rows. Snapshot isolation allows it. Prevent with SERIALIZABLE or by locking a shared row.
5. **Why not 2PC between microservices?** Blocking on coordinator failure, latency, lock duration, external systems do not support it. Use sagas + outbox + idempotent steps.
6. **How do you publish an event exactly when a DB transaction commits?** Transactional outbox (or CDC). Consumers dedupe with an inbox.
7. **Design a ledger.** Double-entry, immutable entries, integer minor units + currency, reversals instead of edits, derived balances with cached balance + reconciliation, system accounts.
8. **A payment provider webhook arrives before your API call returns. How do you handle it?** Webhook stores event, worker applies allowed state transition by CAS; API path also does CAS; whichever arrives second is a no-op.
9. **How do you scale to 10× traffic?** Capacity math first. Pooling, replicas for reads, partitioning, hot-account techniques, then sharding/distributed SQL with cross-shard sagas.
10. **One merchant receives 5,000 payments per second during a sale. What breaks?** Row-lock contention on that account. Bucketed sub-accounts, batching, async balance materialization, lock the hot row last.
11. **How does the frontend avoid duplicate payments?** One key per intent, persisted; ref-based in-flight guard; same key on retry; `unknown` state with status lookup; never optimistic "success".
12. **How would you know if money was lost?** Continuous invariant checks (entries sum to zero, balance = sum of entries), reconciliation with provider and bank, stuck-transaction sweeper, alerting.
13. **Difference between at-least-once and exactly-once?** Exactly-once delivery is not achievable across arbitrary systems; exactly-once *effect* = at-least-once + idempotent processing.
14. **Read-your-writes with replicas?** Return the resource from the write, route the user's reads to primary for a short window, or wait for the replica to reach the write's LSN.

### Three-week study plan

| Days | Topic | Hands-on lab |
| --- | --- | --- |
| 1-2 | Sections 0-2: ACID, isolation, anomalies | Run two `psql` sessions side by side and reproduce lost update, non-repeatable read, and write skew at each isolation level. Run Hermitage tests. |
| 3-4 | Sections 3-4: locks, MVCC, deadlocks, retries | Create a deadlock with opposite-order transfers; fix with lock ordering. Build the `withTransaction` retry helper. Query `pg_locks` while blocked. |
| 5-6 | Section 5-6: ledger and state machines | Implement the schema. Write the invariant queries. Implement reversal. Draw entries for top-up, fee, refund, chargeback. |
| 7-8 | Sections 7-8: idempotency and API | Build `POST /v1/transfers` with idempotency (Case A). Fire 20 concurrent duplicates and prove one effect. Add keyset-paginated history. |
| 9-10 | Sections 9-12: 2PC, sagas, outbox, delivery semantics | Add outbox + relay → Redpanda/Kafka (Docker). Build a consumer with inbox. Kill the relay mid-run; verify recovery. Sketch a purchase saga with compensation. |
| 11-12 | Sections 13-14: providers and reconciliation | Use Stripe test mode: PaymentIntent with your ID as idempotency key, webhook handler with signature verification + dedupe, 3DS test cards. Write a daily reconciliation script. |
| 13-14 | Sections 19-20: frontend | Build the React "Send money" screen: stable key, ref guard, retry with same key, `unknown` state, polling. Playwright test: double click, offline mid-request, reload. |
| 15-16 | Sections 15-17: scaling and hot accounts | Add PgBouncer (transaction mode) and a replica. Partition the ledger by month. k6 test: normal traffic, then "everyone pays one merchant"; implement bucketed accounts and compare p99. |
| 17-18 | Sections 18, 21-22: read models, security, observability | Build a feed read model from events. Add limits with a race-condition test. Add metrics: stuck transfers, outbox lag, `40001` rate. |
| 19-21 | Sections 23-25: testing, capacity, architecture | Property-based test of ledger invariants. Toxiproxy: cut the connection during COMMIT. Write a one-page design doc for this system with capacity math and a failure-mode table, then explain it out loud as in an interview. |

---

## 27. Production Checklist

**Data and database**
- [ ] Amounts stored as `BIGINT` minor units (or `NUMERIC`) with currency; no floats anywhere (DB, API, client).
- [ ] Double-entry ledger; entries immutable; app role cannot `UPDATE`/`DELETE` entries.
- [ ] `CHECK`, `UNIQUE`, `FOREIGN KEY`, `NOT NULL` constraints express the invariants.
- [ ] Each write path documents its isolation level and anomaly defense.
- [ ] Global lock order documented; retries for `40001`/`40P01` with jitter and a cap.
- [ ] `lock_timeout`, `statement_timeout`, `idle_in_transaction_session_timeout` set.
- [ ] No network calls inside DB transactions.
- [ ] Ledger partitioned by time; retention and archival automated.
- [ ] PITR backups; restore tested; HA failover tested.

**Correctness across systems**
- [ ] `Idempotency-Key` required on all money-moving `POST`s; stored with request hash and response in the same transaction.
- [ ] Explicit state machine with CAS transitions and transition history.
- [ ] Sweeper resolves non-terminal transactions older than SLA.
- [ ] Transactional outbox for all events; idempotent consumers with inbox; DLQ with alerting.
- [ ] Provider calls use our ID as provider idempotency key; timeouts treated as unknown.
- [ ] Webhooks: signature verified on raw body, deduped by event ID, processed async, order-independent.
- [ ] Continuous internal reconciliation + daily provider/bank reconciliation with owned mismatch queue.

**Frontend**
- [ ] One idempotency key per intent, persisted across reloads/redirects; same key on retry.
- [ ] In-flight guard via ref (or server-side draft), not only a disabled button.
- [ ] `unknown` state with status lookup; no "failed" on timeout; no optimistic "success".
- [ ] Hosted payment fields; minimal scripts on payment page; CSP + SRI; PCI scope confirmed.
- [ ] 3-D Secure return handled by server-side status check.
- [ ] Polling uses backoff, jitter, stop condition, and pauses when hidden.

**Security and operations**
- [ ] Object-level authorization on every account ID, inside the transaction.
- [ ] Limits and one-time actions protected against races; race tests in CI.
- [ ] Step-up auth for risky actions; rate limits per account, device, and IP.
- [ ] Dashboards and alerts: success rate, stuck transfers, imbalance, drift, outbox/consumer/webhook lag, lock waits, replica lag.
- [ ] Tracing from browser to consumers with `transfer_id` on every span/log; no PII/secrets in logs.
- [ ] Load test at 2-3× expected peak including hot-account scenario; reconciliation clean afterwards.
- [ ] Runbooks for every alert; audited admin tools with four-eyes approval for adjustments.

---

## 28. Master Resource List

**Books and papers**
- Martin Kleppmann, *Designing Data-Intensive Applications* (Ch. 7 Transactions, Ch. 9 Consistency and Consensus, Ch. 11 Stream Processing) — https://dataintensive.net/
- Berenson et al., "A Critique of ANSI SQL Isolation Levels" — https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/
- Garcia-Molina & Salem, "Sagas" — https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf
- Pat Helland, "Life Beyond Distributed Transactions" — https://queue.acm.org/detail.cfm?id=3025012
- Corbett et al., "Spanner" — https://research.google/pubs/spanner-googles-globally-distributed-database-2/
- Unmesh Joshi, *Patterns of Distributed Systems* — https://martinfowler.com/articles/patterns-of-distributed-systems/

**Database documentation**
- PostgreSQL: Concurrency Control (MVCC, isolation, locking) — https://www.postgresql.org/docs/current/mvcc.html
- PostgreSQL: Partitioning — https://www.postgresql.org/docs/current/ddl-partitioning.html
- PostgreSQL: Error codes — https://www.postgresql.org/docs/current/errcodes-appendix.html
- MySQL InnoDB transaction model — https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-model.html
- PgBouncer — https://www.pgbouncer.org/
- CockroachDB transaction layer — https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer
- TigerBeetle docs — https://docs.tigerbeetle.com/

**Patterns**
- microservices.io: Saga, Transactional Outbox, Idempotent Consumer, CQRS — https://microservices.io/patterns/
- Debezium Outbox Event Router — https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html
- Temporal — https://docs.temporal.io/

**Idempotency and payments**
- Stripe, "Designing robust and predictable APIs with idempotency" — https://stripe.com/blog/idempotency
- Brandur Leach, "Implementing Stripe-like Idempotency Keys in Postgres" — https://brandur.org/idempotency-keys
- IETF Idempotency-Key header draft — https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- Stripe docs: PaymentIntents, Webhooks, 3D Secure, Reconciliation — https://docs.stripe.com/
- Modern Treasury Journal (ledgers, reconciliation) — https://www.moderntreasury.com/journal
- PCI SSC document library — https://www.pcisecuritystandards.org/document_library/

**Reliability and testing**
- AWS Builders' Library — https://aws.amazon.com/builders-library/
- Google SRE books — https://sre.google/books/
- Jepsen — https://jepsen.io/
- Hermitage — https://github.com/ept/hermitage
- Testcontainers, fast-check, Toxiproxy, k6 (links in Section 23)
- PortSwigger Web Security Academy, Race conditions — https://portswigger.net/web-security/race-conditions
