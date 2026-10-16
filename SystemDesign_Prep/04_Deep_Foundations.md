# 04. Deep Foundations: Networking, Time, Storage Engines, Indexes, Transactions

Self-contained. Each section: what it is, how it works inside, trade-offs, failure modes, and what to say in an interview.

---

## 4.1 Latency and Throughput Numbers (memorize the orders of magnitude)

| Operation | Approx. time |
|---|---|
| L1 cache reference | 1 ns |
| Branch mispredict | 3 ns |
| L2 cache | 4 ns |
| Mutex lock/unlock | 17 ns |
| Main memory (RAM) reference | 100 ns |
| Compress 1 KB (Snappy/Zippy) | 2 µs |
| Send 1 KB over 1 Gbps | 10 µs |
| SSD random 4 KB read | 16–100 µs |
| Sequential read 1 MB from RAM | 250 µs |
| Datacenter round trip | 0.5 ms |
| Sequential read 1 MB from SSD | 1 ms |
| HDD seek | 10 ms |
| Sequential read 1 MB from HDD | 20 ms |
| Packet CA → Netherlands → CA | 150 ms |

**Lessons:** memory is ~100x faster than SSD random reads; disk seeks dominate; network round trips across regions are the floor for any synchronous cross-region call. Never make a cross-region synchronous call on a hot path.

**Throughput math:** a single Redis instance does on the order of 100K+ simple ops/s; a single Postgres node commonly handles low thousands to tens of thousands of simple writes/s depending on hardware and durability settings; Kafka partitions can each do tens of MB/s. Always state these as rough, hardware-dependent figures.

---

## 4.2 Networking

### 4.2.1 The request path
`Client → DNS → (CDN) → L4/L7 load balancer → API gateway → service → cache → DB`

### 4.2.2 DNS
- Hierarchical resolution: recursive resolver → root → TLD → authoritative.
- Records: A/AAAA (IP), CNAME (alias), MX (mail), TXT, SRV, NS.
- **TTL** controls caching. Low TTL = fast failover, more DNS load. Common in failover designs.
- **GeoDNS / latency-based routing:** return different IPs by client location.
- Limitation: clients and resolvers cache beyond TTL sometimes; DNS failover is not instant.

### 4.2.3 TCP vs UDP
- **TCP:** reliable, ordered, congestion control, 3-way handshake (1 RTT) + TLS handshake. Head-of-line blocking on packet loss.
- **UDP:** unreliable, unordered, no handshake. Used for DNS, video, gaming, QUIC.
- **Connection costs:** pooling and keep-alive matter because handshakes add RTTs.

### 4.2.4 HTTP versions
- **HTTP/1.1:** one request per connection at a time (pipelining rarely used). Browsers open ~6 connections per host.
- **HTTP/2:** multiplexed streams over one TCP connection, binary framing, header compression (HPACK). Still suffers TCP head-of-line blocking.
- **HTTP/3:** runs over **QUIC** (UDP). Stream-level independence, 0-RTT or 1-RTT setup, connection migration across networks.

### 4.2.5 TLS
- TLS 1.3: 1-RTT handshake (0-RTT with resumption, with replay risk for non-idempotent requests).
- Terminate TLS at the load balancer or at the edge; use **mTLS** service-to-service inside a mesh.

### 4.2.6 Real-time communication options (know all four)
| Mechanism | Direction | How | Use when |
|---|---|---|---|
| Short polling | Client pulls | Repeated HTTP requests | Simple, low frequency |
| Long polling | Client pulls | Server holds request until data or timeout | Notifications, moderate scale |
| Server-Sent Events (SSE) | Server → client | One long HTTP response streaming events | One-way feeds, dashboards, LLM token streams |
| WebSocket | Bidirectional | Upgraded TCP connection | Chat, multiplayer, collaboration |

**Stateful gateways:** WebSocket servers hold connections; you need a registry (user → gateway node) and a way to route messages to the right node (pub/sub). Scaling connections means memory and file-descriptor limits (~tens of thousands to hundreds of thousands per box with tuning).

### 4.2.7 Load balancers in depth
- **L4 (transport):** routes by IP/port; very fast; no content awareness. Example: AWS NLB.
- **L7 (application):** routes by path, host, header, cookie; can terminate TLS, rewrite, retry. Example: AWS ALB, Envoy, NGINX.
- **Algorithms:** round robin; least connections; weighted; **power of two choices** (sample 2 random backends, pick the less loaded: near-optimal, cheap); consistent hashing (affinity/cache locality).
- **Health checks:** active (probe endpoint) and passive (observed errors). Use separate liveness vs readiness.
- **Connection draining:** on deploy, stop new requests and let in-flight ones finish.
- **Global LB:** Anycast IP or GeoDNS to pick a region; regional LB inside.

### 4.2.8 CDN
- Edge POPs cache static content and cacheable API responses.
- **Cache keys**, `Cache-Control` (max-age, s-maxage, stale-while-revalidate), `ETag`/`If-None-Match` for conditional requests.
- **Origin shielding:** a mid-tier cache so many edges don't hit origin on a miss.
- **Purge/invalidation:** slow and costly; prefer versioned URLs (`app.v3.js`).

---

## 4.3 Time, Clocks, and Ordering

- **Wall clocks** drift; NTP keeps them within ms on good networks, worse otherwise. Never assume two machines' clocks agree.
- **Monotonic clocks** measure elapsed time locally; use them for timeouts.
- **Lamport clock:** a counter; on send increment, on receive set to `max(local, received) + 1`. Gives a happens-before-consistent order, not real-time order.
- **Vector clock:** a vector of counters, one per node. Detects concurrent (conflicting) updates. Used by Dynamo-style systems for conflict detection. Size grows with nodes (pruning needed).
- **Hybrid Logical Clock (HLC):** combines wall time with a logical counter. Used by CockroachDB-style systems. Close to real time and still causal.
- **TrueTime (Spanner):** GPS and atomic clocks give an uncertainty interval `[earliest, latest]`; commit waits out the uncertainty to get external consistency.
- **Interview line:** "I'd use sequence numbers or a DB-assigned version for ordering, not wall-clock timestamps, because clocks skew."

---

## 4.4 Storage Engines

### 4.4.1 Write-ahead log (WAL)
- Every change is appended to a log before the data page is updated. On crash, replay the log.
- Basis of durability and replication in almost every database (Postgres WAL, MySQL binlog/redo log, Kafka log).

### 4.4.2 B-tree (PostgreSQL, MySQL InnoDB, most RDBMS indexes)
- Balanced tree of pages (typically 4–16 KB). Lookup O(log n) page reads, shallow (3–4 levels for billions of rows).
- Updates in place; page splits on overflow.
- **Read-optimized, good range scans.** Write amplification from page updates and WAL.
- **Fragmentation** and **page splits** under random inserts (e.g., random UUIDs as PKs).

### 4.4.3 LSM tree (Cassandra, RocksDB, LevelDB, HBase, ScyllaDB)
- Writes go to an in-memory **memtable** (plus WAL), flushed as immutable sorted files (**SSTables**).
- Background **compaction** merges SSTables (size-tiered or leveled).
- **Reads** check memtable, then SSTables newest first. Speed-ups: **Bloom filters** (per SSTable, no false negatives, tunable false positives), block cache, index blocks.
- **Trade-off:** excellent write throughput (sequential IO), read and space amplification, compaction CPU/IO spikes, tombstones for deletes (can slow reads until compacted).

### 4.4.4 Bloom filter
- Bit array + k hash functions. "Definitely not in set" or "possibly in set." No false negatives.
- Use cases: avoid disk reads for missing keys (LSM), duplicate checks in crawlers, "has this user seen this item" at scale.
- Size: about 10 bits per element gives ~1% false positive rate.

### 4.4.5 Columnar storage (OLAP)
- Store each column together. Good compression (similar values), reading only needed columns, vectorized execution.
- Formats: **Parquet**, **ORC**. Engines: ClickHouse, Druid (segments), BigQuery, Redshift, Snowflake.
- Row store (OLTP) vs column store (OLAP): transactions and point lookups vs scans and aggregations.

### 4.4.6 Indexes
- **Primary (clustered) index:** table stored in PK order (InnoDB). **Secondary index** points to PK.
- **Composite index:** column order matters. `(a, b, c)` serves `a`, `a,b`, `a,b,c` but not `b` alone.
- **Covering index:** includes all selected columns; no table lookup.
- **Partial index:** indexes a subset (e.g., `WHERE status='active'`).
- **Hash index:** O(1) equality, no range scans.
- **Inverted index:** term → postings list (search engines, GIN in Postgres).
- **Geospatial:** R-tree, geohash, quadtree (see HLD guide).
- **Write cost:** each index adds write cost and storage. Index what the queries need.

### 4.4.7 Query planning
- Cost-based optimizer uses statistics (row counts, histograms). Stale statistics produce bad plans: run `ANALYZE`.
- `EXPLAIN` / `EXPLAIN ANALYZE` to read plans. Look for sequential scans on large tables, sort spills, nested-loop joins over large inputs.
- **N+1 queries:** an app pattern (1 query for a list, then N for each item). Fix with joins, batching (DataLoader), or prefetch.

---

## 4.5 Transactions and Isolation

### 4.5.1 ACID
- **Atomicity:** all or nothing (WAL + rollback).
- **Consistency:** invariants hold (application + DB constraints). Not the same as CAP consistency.
- **Isolation:** concurrent transactions behave as if serialized, to a degree set by the level.
- **Durability:** committed data survives crashes (fsync/WAL, replication).

### 4.5.2 Anomalies
| Anomaly | What happens |
|---|---|
| Dirty read | Read uncommitted data that later rolls back |
| Non-repeatable read | Same row read twice, different values |
| Phantom read | Same query returns different rows set (new rows inserted) |
| Lost update | Two read-modify-write cycles overwrite each other |
| Write skew | Two txns read overlapping data, write disjoint rows, violate an invariant (e.g., two doctors both go off-call) |

### 4.5.3 Isolation levels
| Level | Prevents |
|---|---|
| Read uncommitted | Nothing (rare) |
| Read committed | Dirty reads (default in Postgres, Oracle) |
| Repeatable read | Non-repeatable reads (MySQL InnoDB default; uses next-key locks) |
| Snapshot isolation | Reads from a consistent snapshot; prevents dirty/non-repeatable/phantom reads in most engines, **but allows write skew** |
| Serializable | All anomalies (2PL, or serializable snapshot isolation (SSI) as in Postgres) |

### 4.5.4 Concurrency control
- **Pessimistic:** lock rows (`SELECT ... FOR UPDATE`). Simple, blocks others, risk of deadlock.
- **Optimistic:** version column; `UPDATE ... WHERE version = ?`; retry on conflict. Good when conflicts are rare.
- **MVCC:** keep multiple row versions; readers don't block writers. Postgres, MySQL InnoDB, Spanner.
- **Deadlocks:** detected by the DB (wait-for graph); mitigate with consistent lock ordering and short transactions.

### 4.5.5 Interview lines
- "The default isolation is read committed, so for a balance check-and-update I'd use `SELECT FOR UPDATE` or a conditional update to avoid lost updates."
- "Snapshot isolation allows write skew; for a constraint across rows I'd use serializable or an explicit lock."

---

## 4.6 Data Modeling Essentials

- **Normalization** reduces duplication and update anomalies; **denormalization** speeds reads at the cost of update complexity.
- **Access-pattern-driven modeling** (NoSQL): design tables around queries, not entities.
- **Keys:** natural vs surrogate; prefer stable surrogate IDs (Snowflake-style, time-sortable).
- **Soft delete vs hard delete;** tombstones in distributed stores.
- **Schema evolution:** add optional fields; never reuse field numbers in protobuf; expand then contract migrations (add column, backfill, switch reads, drop old).
- **Money:** integers in minor units plus currency code. Never floats.
- **Time:** store UTC; include time zone only for display.
