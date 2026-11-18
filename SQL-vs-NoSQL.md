# SQL vs NoSQL: HLD Interview Notes

Source article: https://www.hellointerview.com/blog/sql-vs-nosql
Status: Parts 1–5b and the decision framework are done. Part 5c onwards is still to learn.

---

## Table of contents

1. Transactions
2. ACID properties
3. Isolation levels and anomalies
4. Connection pools and PgBouncer
5. NoSQL: why it exists, families, consistency, transactions
6. SQL vs NoSQL decision framework (from the article)
7. Model interview answers
8. Practice questions (with answers)
9. Yet to learn

---

## 1. Transactions

### 1.1 What a transaction is

A transaction is a group of database operations that must all succeed or all fail, as one unit.

Example: Abhi sends ₹500 to a friend. This takes two writes.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 'abhi';
UPDATE accounts SET balance = balance + 500 WHERE id = 'friend';
COMMIT;
```

- At `COMMIT`, the database applies both updates.
- If something fails before `COMMIT`, it runs `ROLLBACK` and applies neither.

### 1.2 Crash cases

**The app server crashes during a transaction**
1. The TCP connection closes.
2. The database sees that no `COMMIT` arrived.
3. The database rolls back automatically.

**The database crashes during a transaction.** The Write-Ahead Log (WAL) handles this case.

- The WAL is an append-only file on disk.
- Every change goes into the WAL **before** the database changes the data files.
- On `COMMIT`, the database:
  1. Writes a `COMMIT` record to the WAL.
  2. Calls `fsync()` to force that record onto the physical disk.
  3. Sends "OK" to the client. It does this only after step 2.
- When the database restarts, it runs **recovery**:
  - A transaction that has a `COMMIT` record is **redone**.
  - A transaction that has no `COMMIT` record is **discarded**.

```
WAL:  [BEGIN 42] [42: abhi 1000 -> 500]  ---- CRASH ----   (no COMMIT 42)
Recovery: 42 never committed -> discard -> abhi = 1000
```

Simple Python model of recovery:

```python
def recover(wal, data):
    committed = {r["tx"] for r in wal if r["type"] == "COMMIT"}
    for r in wal:
        if r["type"] == "UPDATE" and r["tx"] in committed:
            data[r["key"]] = r["new"]   # redo only committed work
    return data
```

> **Rule:** A transaction is durable only after its COMMIT record is on disk. If the database crashes before that point, the whole transaction is lost.

### 1.3 The ambiguous-commit edge case (payments)

Suppose the database commits and then crashes before it sends "OK". The money moved, but the app thinks the request failed. If the app retries, the user is charged twice.

**Fix:** use an **idempotency key**. This is a unique request ID with a UNIQUE constraint, stored in the same transaction as the payment. When a retry arrives with the same key, the insert fails and the duplicate is rejected.

### 1.4 Why WAL + fsync is still fast

1. **Data pages change in RAM** (the buffer pool, `shared_buffers`). A background **checkpoint** writes them to the data files later.
2. On the commit path, the database only appends a small WAL record and calls one `fsync`.
3. **WAL writes are sequential appends**, which are much faster than random writes.
4. **Group commit:** many concurrent commits share one `fsync`.

Approximate cost of one fsync. These numbers are only orders of magnitude, so measure your own disk with `pg_test_fsync`.

| Storage | fsync |
|---|---|
| HDD | 5–10 ms |
| Consumer NVMe | ~0.5–5 ms |
| Enterprise SSD (power-loss protected) | ~0.02–0.1 ms |
| Cloud network disk (EBS gp3) | ~0.5–2 ms |

**Trade-off setting:** `synchronous_commit = off` sends "OK" before the fsync. This is faster, but a crash can lose the last few hundred ms of commits. That is acceptable for analytics and NOT acceptable for payments.

### 1.5 Corrected mental model

| Buffer | Reaches disk | Lost on crash? |
|---|---|---|
| WAL buffer | At every COMMIT (fsync before "OK") | **No** |
| Dirty data pages | Later, at checkpoint | Yes, but recovery rebuilds them from the WAL |

- A transaction that received "OK" is never lost (with the default settings).
- Durability is **not** a SQL-vs-NoSQL property. It depends on configuration:
  - Postgres can lose data if you set `synchronous_commit = off`.
  - Cassandra uses `commitlog_sync: periodic` by default (fsync about every 10 s), but replication (RF = 3) protects the data.
  - MongoDB 5.0+ uses write concern `majority` by default, which is journaled on a majority of nodes.
  - DynamoDB acknowledges a write only after it is stored on replicas in several availability zones.
- There are two ways to make a write durable:
  - **Local:** fsync to this machine's disk before "OK".
  - **Distributed:** replicate the write to N machines before "OK".

---

## 2. ACID properties

| Property | Question it answers | Mechanism |
|---|---|---|
| **Atomicity** | Did all of it happen, or none? | WAL + rollback |
| **Consistency** | Do all the rules still hold? | Constraints (CHECK, FOREIGN KEY, UNIQUE, NOT NULL) |
| **Isolation** | Can parallel transactions corrupt each other? | Row locks + MVCC |
| **Durability** | Will a committed write survive a crash? | WAL + fsync, optionally replication |

### Consistency example

```sql
CREATE TABLE accounts (
  id      TEXT PRIMARY KEY,
  balance NUMERIC NOT NULL CHECK (balance >= 0)
);
```

If Abhi has ₹300 and sends ₹500, the CHECK constraint fails and the whole transaction rolls back.

### ⚠️ ACID "C" ≠ CAP "C"

| Term | Meaning |
|---|---|
| ACID Consistency | Data satisfies the declared rules (constraints) |
| CAP Consistency (linearizability) | Every read sees the latest write, across all replicas |

"NoSQL is eventually consistent" refers to the **CAP** meaning.

### Isolation + Consistency together (two ₹700 sends from ₹1000)

```
Tx1: UPDATE balance-700 -> takes ROW LOCK
Tx2: UPDATE balance-700 -> BLOCKS
Tx1: COMMIT -> 300, lock released
Tx2: wakes up, re-reads 300 -> 300-700 < 0 -> CHECK fails -> ROLLBACK
```

- The database takes a **row lock**, not a table lock.
- Atomicity rolls back all of Tx2.

### The lost-update trap

```python
bal = SELECT balance ...          # both requests read 1000
UPDATE ... SET balance = bal-700  # both write 300 -> CHECK never fails
```

**Fix:** do the arithmetic in the database (`SET balance = balance - 700`), or lock at read time with `SELECT ... FOR UPDATE`.

> **Rule:** Constraints protect **what** you write. Isolation protects **what you read before you write**.

### Locks that are never released

- A lock is released at `COMMIT`, at `ROLLBACK`, or when the connection dies (the database then rolls back).
- **The real danger is a long transaction**, for example an HTTP call to a payment gateway while a `FOR UPDATE` lock is held.
  - The session shows as `idle in transaction`.
  - Requests pile up behind the lock and the connection pool runs out.
- **Fixes:**
  - Never call another service over the network inside a transaction.
  - Use a **state machine**:
    1. Insert `PENDING` and COMMIT.
    2. Call the gateway with no lock held.
    3. Update to `SUCCESS` or `FAILED` and COMMIT.
  - `FOR UPDATE` is fine in production **when the transaction is short**.
- **Timeouts:**
  - `idle_in_transaction_session_timeout`
  - `lock_timeout`
  - `statement_timeout`
- `NOWAIT` returns an error immediately if the row is locked. `SKIP LOCKED` skips locked rows, which is the standard pattern for a job queue on Postgres.
- **Deadlock:** Postgres detects it after `deadlock_timeout` (1 s), aborts one transaction, and the app must retry. To prevent deadlocks, always lock rows in the same order (for example, by ID).
- **Half-open connections:** TCP keepalive detects a dead client.

---

## 3. Isolation levels and anomalies

### Anomalies

1. **Dirty read:** you read data from a transaction that has not committed yet.
2. **Non-repeatable read:** you read the same row twice and get different values.
3. **Phantom read:** you run the same range query twice and get a different set of rows.
4. **Lost update:** two read-modify-write cycles overlap, and one write overwrites the other.
5. **Write skew:** two transactions read the same data and write *different* rows, and together they break a rule.
   - Example: a rule that at least one doctor must be on call.
   - Example: a limit of 100 tickets per show.

### Levels

| Level | Dirty | Non-repeatable | Phantom | Lost update | Write skew |
|---|---|---|---|---|---|
| Read Uncommitted | ✗ | ✗ | ✗ | ✗ | ✗ |
| **Read Committed** (Postgres default) | ✅ | ✗ | ✗ | ✗* | ✗ |
| Repeatable Read (Postgres = snapshot isolation) | ✅ | ✅ | ✅ | ✅ | ✗ |
| Serializable (Postgres = SSI) | ✅ | ✅ | ✅ | ✅ | ✅ |

✅ = prevented, ✗ = possible.
\* An atomic `SET x = x - n` is still safe at Read Committed.

- **Postgres default:** Read Committed. Check it with `SHOW default_transaction_isolation;`.
- **MySQL InnoDB default:** Repeatable Read.
- Higher isolation causes more aborts, so the app **needs retry logic**.

### Payments on Read Committed are safe if:

1. Balance changes are atomic updates in the database.
2. The code uses `SELECT ... FOR UPDATE` when it must read before it writes.
3. Idempotency keys have a UNIQUE constraint.
4. State transitions use conditional updates:

```sql
UPDATE payments SET status = 'SUCCESS'
WHERE id = 'pay_123' AND status = 'PENDING';   -- 0 rows -> already changed
```

### Ticket booking fix (write skew via a phantom)

```sql
UPDATE shows SET sold = sold + 1 WHERE id = 7 AND sold < 100;  -- 0 rows = sold out
```

Other fixes:
- Lock the parent row with `SELECT ... FOR UPDATE`.
- Pre-create seat rows and claim them with `SKIP LOCKED`.
- Use Serializable with retries.

**Interview sentence:** "Postgres defaults to Read Committed. For invariants like balances or inventory, I'd use atomic conditional updates or row locks. I'd use Serializable with retries only when a rule spans many rows."

---

## 4. Connection pools and PgBouncer

### 4.1 Why connections are expensive

Opening a connection costs:
- a TCP handshake
- a TLS handshake
- authentication
- a **forked Postgres backend process** (several MB of RAM)

Total: several ms to tens of ms. A query on a warm connection takes under 1 ms.

### 4.2 How a pool works

- The app opens N connections once and keeps them open.
- Each request follows this cycle:
  1. Checkout: borrow a connection, and wait if all are busy.
  2. Use it.
  3. Return it. The pool **resets** the connection (rollback) and health-checks it.
- **Connection lifetime ≠ transaction lifetime.** One connection runs thousands of short transactions over its life.
- Risks:
  - A connection goes back to the pool with a transaction still open. Fix: reset on return. sqlx's `Transaction` rolls back on drop.
  - **Connection leaks.** Fix: always return the connection in `finally` or use an RAII guard.
- Libraries:
  - Java: HikariCP
  - Python: SQLAlchemy
  - Rust: bb8, deadpool, r2d2, sqlx

### 4.3 Sizing

- Start with roughly **2 × the DB's CPU cores** active connections, then load-test.
- **Total connections = pool size × app instances.**
  - Keep the total below `max_connections`.
  - Leave headroom for superuser, migration and monitoring connections.
  - Example: 5 instances × 15 each = 75, which leaves room under 100.
- If you go over the limit, Postgres returns `FATAL: sorry, too many clients already`.

### 4.4 Cores vs processes vs `max_connections`

- Cores limit how many processes **run** at once, not how many **exist**. The OS time-slices the processes on the cores.
- Most Postgres backends are idle or blocked on I/O, so they use no CPU.
- Postgres uses **one OS process per connection**, with no green threads. Green threads belong to app runtimes such as Tokio and Go.
- `max_connections = 100` is a **conservative default**. Each connection costs even when it is idle:
  - per-process memory
  - `work_mem` per sort or hash operation
  - shared-memory structures sized by `max_connections`
  - snapshot cost (MVCC)

| Number | Meaning | Typical value |
|---|---|---|
| `max_connections` | Maximum **open** connections | 100 default, often a few hundred |
| Active concurrency | Queries running at the same time | ≈ 1–3 × cores |

### 4.5 PgBouncer

- PgBouncer is a lightweight **event-loop proxy** that speaks the Postgres protocol.
- It maps **many client connections onto a few real server connections**.

```
20 pods × 50 = 1000 client conns -> [PgBouncer] -> 40 real conns -> Postgres
```

| Mode | Server connection held for | Multiplexing |
|---|---|---|
| Session | The whole client connection | None |
| **Transaction** | One transaction | High, and the most common mode in production |
| Statement | One statement | Highest, but multi-statement transactions are not allowed |

**Caveat in transaction mode:** session state is not kept between transactions. This breaks:
- `SET` (use `SET LOCAL` instead)
- session advisory locks (use `pg_advisory_xact_lock` instead)
- `LISTEN/NOTIFY`
- temp tables that must last across transactions

PgBouncer 1.21+ supports prepared statements (`max_prepared_statements`).

**Deployment:**
- A central PgBouncer. Run 2+ instances so it is not a single point of failure.
- A sidecar per pod.
- Managed alternatives: RDS Proxy, Supavisor, PgCat, Odyssey.

You usually still keep a small pool in the app.

---

## 5. NoSQL

### 5.1 Why it exists

In the mid-2000s, some companies had more write volume and data than one machine could handle, and they also needed high availability. This led to the **Dynamo paper** (Amazon, 2007) and the **Bigtable paper** (Google, 2006).

### 5.2 Core mechanism: partitioning by key

1. Every item has a **partition key**.
2. The key is hashed (consistent hashing) to choose the owning node.
3. Each partition is replicated (for example, 3 copies).

- A request for one key touches **one partition**. To scale, you add nodes.
- **The price:** anything that spans many partitions is slow or impossible:
  - joins
  - cross-partition transactions
  - foreign keys
  - queries on non-key fields (scatter-gather to every shard)
- What NoSQL gave up is **cross-partition** ACID. Operations inside one partition are usually atomic. For example, a single MongoDB document write is always atomic.
- SQL **can** be sharded too (Vitess, Citus). It is just not built in.

### 5.3 The four families

| Family | Model | Query by | Strength | Examples | Use for |
|---|---|---|---|---|---|
| Key-value | key → blob | key only | Speed | Redis, Memcached, DynamoDB | Cache, sessions, rate limits |
| Document | key → JSON doc | key + indexed fields | Flexible schema, whole-entity reads | MongoDB, Couchbase, DynamoDB* | Catalogs, profiles |
| Wide-column | partition → rows sorted by clustering key | partition key + range | Very high write throughput (LSM tree) | Cassandra, ScyllaDB, HBase | Logs, chat, time series |
| Graph | nodes + edges | traversals | Relationship queries | Neo4j, Neptune | Social graphs, fraud rings |

\* AWS describes DynamoDB as a "key-value and document" database. Its model is a partition key + sort key.

- **Why writes are fast in wide-column stores:** an LSM tree writes to an in-memory memtable and an append-only commit log. Background compaction merges the data files later.
- **Document queries on fields** are fast only when an index exists. Without one, a query on a non-shard-key field is scatter-gather.

### 5.4 BASE

- **Basically Available:** the system answers even when some nodes fail.
- **Soft state:** replicas may disagree for a short time.
- **Eventual consistency:** when writes stop, the replicas converge.

```
t0 write 300 -> A    t1 A=300, B=1000, C=1000    t2 read B -> 1000 (stale)    t3 all = 300
```

### 5.5 Quorums (N, R, W)

- **N** = number of replicas
- **W** = replicas that must confirm a write
- **R** = replicas that are read

> **If R + W > N, reads are strongly consistent**, because the read set always overlaps the latest write.

| Setting (N=3) | Latency | Consistency | Survives |
|---|---|---|---|
| W=1, R=1 | Lowest | Eventual | Writes work with 1 node alive |
| W=2, R=2 (QUORUM) | Medium | Strong | 1 node down |
| W=3, R=1 | Slow writes | Strong | Writes fail if any node is down |

Cassandra lets you set this per query: `ONE`, `QUORUM`, `ALL`, or `LOCAL_QUORUM`.

### 5.6 Conflicts

- **Last-write-wins** by timestamp is the Cassandra default. One of the two writes is silently lost.
- **Conditional writes / compare-and-set** avoid the lost update.

### 5.7 Transactions today ("NoSQL has no ACID" is false)

**MongoDB**
- A single-document write is atomic.
- Multi-document ACID transactions are supported since 4.0 (replica sets) and 4.2 (sharded clusters). They are slower.
- `writeConcern` and `readConcern` control durability and consistency.

**DynamoDB**
- Reads are eventually consistent by default.
- `ConsistentRead=true` costs 2× the read capacity.
- `ConditionExpression` gives conditional writes.
- `TransactWriteItems` is all-or-nothing across up to 100 items, at about 2× the cost.

**Cassandra**
- There are no multi-partition transactions.
- **Lightweight transactions** (`IF NOT EXISTS`, `IF x = y`) use Paxos and are about 4× slower.

DynamoDB bank transfer:

```python
dynamodb.transact_write_items(TransactItems=[
  {"Update": {"TableName": "accounts", "Key": {"id": {"S": "abhi"}},
     "UpdateExpression": "SET balance = balance - :a",
     "ConditionExpression": "balance >= :a",
     "ExpressionAttributeValues": {":a": {"N": "500"}}}},
  {"Update": {"TableName": "accounts", "Key": {"id": {"S": "friend"}},
     "UpdateExpression": "SET balance = balance + :a",
     "ExpressionAttributeValues": {":a": {"N": "500"}}}},
])
```

> **Rule:** NoSQL uses eventual consistency by default. You can pay for stronger guarantees per request (quorums, conditional writes, limited transactions), and the cost is latency and money.

---

## 6. SQL vs NoSQL decision framework (from the article)

The article's flowchart is an image. The flow below is rebuilt from the article's text.

```
1. Probe the requirements: data volume (numbers), read/write ratio,
   query complexity, consistency needs
        │
2. Is scale genuinely massive?
   ├─ NO  -> Lean SQL (Postgres/MySQL)
   └─ YES -> 3. What is the access pattern?
              ├─ Write-heavy, simple reads   -> Cassandra / DynamoDB
              ├─ Full-text search            -> Elasticsearch
              └─ Simple KV at extreme scale  -> Redis / Memcached
        │
4. Polyglot: Postgres (core) + Redis (cache/sessions)
             + Elasticsearch (search) + S3 (blobs)
```

**Key recommendations from the article**
- Ask about the requirements before you choose a database.
- **Default to SQL.** For most applications, a SQL database with a cache is the right answer.
- Don't repeat outdated 2015-era talking points such as "NoSQL for scale, SQL for consistency".
- Don't say only "it depends". Explain **how** it depends.
- Explain the operational and team costs, not only the technical ones.
- If the interviewer pushes back on a reasonable SQL choice, defend it briefly. If they still disagree after about 20 seconds, move on.
- Treat SQL vs NoSQL as a false choice, because modern systems often use both.

**Other categories the article mentions:**
- Time series: InfluxDB, TimescaleDB
- Search: Elasticsearch, Typesense
- Vector: Pinecone, Weaviate
- Distributed SQL: Aurora DSQL
- Blob storage: S3

---

## 7. Model interview answers

### "What's the difference between SQL and NoSQL?" (~1 min)

> "SQL databases store data in tables with a fixed schema. They model relationships with foreign keys, support joins, and give full ACID transactions across rows. NoSQL is a group of models: key-value like Redis, document like MongoDB, wide-column like Cassandra, and graph like Neo4j. Their schemas are usually flexible. On scaling, SQL usually goes vertical first, then adds read replicas, then shards. NoSQL is built to partition by key, so it scales out horizontally more easily. The trade-off is that you lose joins and easy cross-partition transactions. Many NoSQL stores default to eventual consistency, but most now offer tunable or stronger options."

### Points that show maturity

1. The two sides overlap today. Postgres has JSONB. MongoDB and DynamoDB have transactions.
2. In NoSQL, you design the data around the queries. Know your access patterns first, and denormalize.
3. Default to SQL unless the scale truly requires something else.
4. Justify the choice with requirements, for example: "payments need multi-row transactions, so Postgres".

### Connection pools (~30 s)

> "Opening a Postgres connection is expensive: TCP and TLS handshakes, auth, and a forked backend process. So each app server uses a pool that opens a fixed number of connections and reuses them. A request borrows a connection, runs a short transaction, and returns it. The pool resets and health-checks it. Postgres caps total connections (100 by default), so the pool size times the number of app instances must stay under that, with headroom. When the app tier scales out a lot, I'd put PgBouncer in transaction mode in front of Postgres, so many client connections share a few real connections."

### PgBouncer (~20 s)

> "PgBouncer in transaction mode accepts thousands of cheap client connections and multiplexes them onto a small number of real Postgres connections, one transaction at a time. The trade-off is that session state such as SET, session advisory locks and LISTEN/NOTIFY doesn't carry across transactions."

---

## 8. Practice questions (with answers)

**Q1. Abhi has ₹1000. Two requests each send ₹700 at the same moment. What stops the balance from going negative?**
<details><summary>Answer</summary>
Isolation and Consistency together. The first UPDATE takes a row lock, and the second waits. After the first commits, the second re-reads ₹300. The CHECK (balance >= 0) fails, and atomicity rolls back the whole second transaction.
</details>

**Q2. Ticket booking: `SELECT COUNT(*)`, then `INSERT` if the count is under 100. Two users book the last ticket at the same time.**
<details><summary>Answer</summary>
Anomaly: write skew via a phantom (the result is 101 bookings). Read Committed does not prevent it. Postgres Repeatable Read does not prevent it either. Serializable does.
Fixes: a counter with `UPDATE shows SET sold = sold + 1 WHERE id = 7 AND sold < 100`, `FOR UPDATE` on the show row, pre-created seat rows claimed with SKIP LOCKED, or Serializable with retries.
</details>

**Q3. A nightly job uses session-level `pg_advisory_lock(42)`. You move to PgBouncer in transaction mode. What breaks?**
<details><summary>Answer</summary>
The lock stays on a server connection that goes back to the pool. It may never be released, or another client may appear to own it. Fix: use `pg_advisory_xact_lock` inside one transaction, or a direct or session-mode connection for that job.
</details>

**Q4. Cassandra, N = 3. You want strongly consistent reads, and writes must still work with one node down. Choose R and W. What happens with two nodes down?** *(still open, so try it first)*
<details><summary>Answer</summary>
W = 2, R = 2 (QUORUM). 2 + 2 > 3, so every read overlaps the latest write, and one node can be down. With two nodes down, only one replica remains. QUORUM reads and writes fail because they cannot reach 2 replicas. You must either accept errors (keep consistency) or drop to ONE (keep availability, accept stale reads). That choice is the CAP trade-off.
</details>

**Q5. "NoSQL scales horizontally and SQL doesn't." True?**
<details><summary>Answer</summary>
Not exactly. SQL scales vertically, then with read replicas, then by sharding (Vitess, Citus). Distributed SQL products such as CockroachDB, Spanner and Aurora DSQL also exist. NoSQL has partitioning built in, which makes horizontal scaling easier, at the cost of joins and cross-partition transactions.
</details>

---

## 9. Yet to learn

### Part 5c: NoSQL data modeling
- Access-pattern-first design. List every query before you design the tables.
- Denormalization: copy data so that each query reads from one partition.
- Choosing a partition key: high cardinality, even spread, and no **hot partitions** (for example, a celebrity user).
- Sort keys / clustering keys for range queries (for example, the last 50 messages).
- DynamoDB single-table design, GSIs (global secondary indexes) and LSIs.
- Cassandra: one table per query pattern.
- MongoDB: embed or reference.
- Keeping denormalized copies in sync: dual writes, CDC and streams.

### Part 5d: CAP and PACELC
- CAP: during a network partition, choose consistency or availability.
- PACELC: when there is no partition, choose latency or consistency.
- Where Postgres, Cassandra, DynamoDB and MongoDB sit.

### Part 6: Scaling SQL in depth
- Vertical scaling, read replicas, and replication lag (read-your-writes problems).
- Sharding strategies: range, hash and directory. Choosing a shard key and resharding.
- Vitess, Citus, and distributed SQL (CockroachDB, Spanner, Aurora DSQL).

### Part 7: Storage engines
- B-tree (Postgres/MySQL) vs LSM tree (Cassandra/RocksDB): read, write and space amplification.
- Indexes: how they speed up reads and slow down writes.

### Part 8: Specialized stores (from the article)
- Search: Elasticsearch (inverted index), and syncing it from Postgres with CDC.
- Time series: TimescaleDB, InfluxDB.
- Vector: Pinecone, Weaviate, pgvector.
- Blob storage: S3, with metadata kept in the database.
- Cache: Redis patterns (cache-aside, TTL, invalidation).

### Part 9: Final mock interview
- A full SQL vs NoSQL interview round based on the article, with feedback.
- Practice choosing a database for common designs: payments, chat, URL shortener, news feed, ride-hailing.
