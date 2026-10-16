# High-Level System Design (HLD) Guide

---

## Part A: The Interview Framework (45–60 min)

| Step | Time (45 min) | What you do | Output |
|---|---|---|---|
| 1. Requirements | 5 min | Ask questions; split functional vs non-functional | Scope list |
| 2. Estimation | 3–5 min | QPS, storage, bandwidth | 3–4 numbers |
| 3. API and data model | 5 min | Endpoints, entities, storage choice | API list, schema |
| 4. High-level design | 10–12 min | Draw the components and the main flow | Diagram |
| 5. Deep dives | 15 min | 2–3 bottlenecks or hard parts | Trade-offs |
| 6. Wrap-up | 3 min | Failures, monitoring, what's next | Summary |

### Step 1: Requirements questions to ask
- **Users:** who uses it, how many (DAU/MAU), where (regions)?
- **Core features:** what must work in v1? What is out of scope?
- **Scale:** read vs write ratio, peak traffic, data size growth.
- **Non-functional:** latency target, availability (99.9% vs 99.99%), consistency (strong vs eventual), durability.
- **Constraints:** real-time or batch? Mobile or web? Compliance (GDPR, PCI)?

Write requirements on the board and confirm them out loud: "So I'm designing X for Y users with Z latency. Is that right?"

### Step 4: How to draw
- Start with the simplest working design, then add components only when a requirement needs them.
- Standard boxes: Client → DNS/CDN → Load balancer → API servers → Cache → DB; plus queues, workers, and object storage as needed.
- Draw one **main flow** end to end (e.g., "user posts a tweet" or "driver sends location").
- Name the protocol on arrows where it matters (HTTP, gRPC, WebSocket, Kafka topic).

### Step 5: Deep dives: pick the right ones
Good deep-dive topics: hot keys, a single DB becoming the bottleneck, fan-out, failure of one component, race conditions, exactly-once vs at-least-once, data partitioning key, cache invalidation, multi-region.

### Phrases that score points
- "The bottleneck here will be..., so I'll..."
- "The trade-off is X vs Y. I'm choosing X because our requirement is..."
- "If this component fails, the impact is... and we handle it by..."
- "At 10× scale, the first thing that breaks is..."

---

## Part B: Building Blocks (know the trade-offs, not just the names)

### B1. DNS, CDN, Load Balancers
- **CDN:** caches static assets and sometimes API responses near users. Use for images, video, JS/CSS.
- **Load balancer:** L4 (TCP/UDP, fast) vs L7 (HTTP, can route by path/header).
- **Algorithms:** round robin, least connections, consistent hashing (for sticky/cache-affine routing), weighted.
- **Health checks** remove unhealthy nodes. **API gateway** adds auth, rate limiting, routing, and request logging.

### B2. Caching
- **Patterns:**
  - *Cache-aside:* app reads cache, falls back to DB, then populates cache. Most common.
  - *Write-through:* write to cache and DB together. Consistent, slower writes.
  - *Write-behind (write-back):* write to cache, flush to DB later. Fast, risk of data loss.
  - *Read-through:* the cache loads from DB on miss, handled by the cache layer.
- **Eviction:** LRU (most common), LFU, TTL.
- **Problems:**
  - *Cache stampede / thundering herd:* many misses hit the DB at once. Fix with locking/single-flight, staggered TTLs, or early refresh.
  - *Hot keys:* one key gets most traffic. Fix with local in-process cache, key replication (`key#1..N`), or splitting the value.
  - *Invalidation:* the hardest part. Use TTLs plus explicit invalidation on writes, or versioned keys.
- **Where:** client, CDN, app-local (e.g., Caffeine), distributed (Redis, Memcached).

### B3. Databases
- **SQL (PostgreSQL, MySQL):** ACID, joins, flexible queries. Scale vertically first, then read replicas, then sharding.
- **NoSQL:**
  - Key-value (Redis, DynamoDB): fast point lookups.
  - Wide-column (Cassandra, HBase): high write throughput, time-series-like.
  - Document (MongoDB): flexible schema.
  - Graph (Neo4j): relationship-heavy queries.
- **Storage engines:**
  - *B-tree* (PostgreSQL, MySQL InnoDB): read-optimized, in-place updates.
  - *LSM tree* (Cassandra, RocksDB): write-optimized, compaction costs.
- **Indexes:** speed reads, slow writes. Composite index order matters (leftmost prefix). Covering index avoids table lookups.
- **Replication:**
  - Leader–follower: simple; replication lag causes stale reads.
  - Multi-leader: multi-region writes; conflict resolution needed.
  - Leaderless (Dynamo-style): quorum reads/writes.
- **Sharding:** split data by a key. Choose a key with high cardinality and even access. Problems: hot shards, cross-shard joins and transactions, resharding (use consistent hashing or a directory).

### B4. Consistent Hashing
- Hash nodes and keys onto a ring; a key belongs to the next node clockwise.
- Adding or removing a node moves only about `1/N` of the keys.
- **Virtual nodes** (e.g., 100–200 per physical node) even out the load.
- Used by: Dynamo, Cassandra, Memcached clients, Redis Cluster (uses hash slots, a variant).
- *Your resume project* implemented this with 200 virtual nodes. Be ready to explain it.

### B5. Replication and Quorum
- Leaderless with N replicas: write quorum W, read quorum R.
- **R + W > N** gives overlap, so reads see the latest write (strong-ish consistency).
- Example: N=3, W=2, R=2.
- Sloppy quorum and hinted handoff help availability during failures.

### B6. CAP and PACELC
- **CAP:** during a network partition you choose consistency (C) or availability (A).
- **PACELC:** if Partition, choose A or C; Else (normal operation), choose Latency or Consistency.
- Say in interviews: "This is a payment ledger, so I choose CP. A product-view counter is fine with AP."

### B7. Message Queues and Streams (Kafka)
- **Kafka:** partitioned, ordered per partition, retained log. Consumers track offsets. Consumer groups share partitions.
- **Delivery semantics:**
  - At-most-once: may lose messages.
  - At-least-once: may duplicate. The default in most systems. Requires **idempotent consumers**.
  - Exactly-once: possible within Kafka transactions and idempotent producers; end-to-end still needs idempotent sinks.
- **Partition key:** choose so related events stay in order (e.g., `user_id`) and load is spread.
- **Consumer lag:** a key metric. Rising lag means consumers are too slow.
- **Dead letter queue (DLQ):** holds messages that fail after retries.
- **Ordering:** only within a partition. Global ordering is expensive.
- **Backpressure:** bounded queues, consumer scaling, and rate limiting upstream.

### B8. Idempotency, Outbox, Sagas
- **Idempotency key:** client sends a unique key; server stores the result and returns it on retry. Needed for payments and anything with retries.
- **Transactional outbox:** write the event to an outbox table in the same DB transaction as the business change; a relay publishes it to Kafka. Avoids "DB committed but event lost."
- **Saga:** a sequence of local transactions with compensating actions on failure. Choreography (events) or orchestration (a coordinator, e.g., Temporal). Use for multi-service workflows like booking + payment + notification.

### B9. Rate Limiting
| Algorithm | How | Pros | Cons |
|---|---|---|---|
| Fixed window | Counter per time window | Simple, cheap | Burst at window edges |
| Sliding window log | Store timestamps | Accurate | Memory heavy |
| Sliding window counter | Weighted two windows | Good balance | Approximate |
| Token bucket | Tokens refill at rate r, capacity b | Allows bursts | Tuning |
| Leaky bucket | Constant outflow queue | Smooth output | Queue latency |

- Distributed: store counters in Redis; use a Lua script for atomic check-and-increment.
- Return `429` with `Retry-After`. Rate limit per user, per IP, and per API key.

### B10. Unique IDs
- **Auto-increment:** doesn't scale across shards.
- **UUID v4:** unique, random (poor index locality).
- **Snowflake:** 64-bit: timestamp + machine ID + sequence. Time-sortable, no coordination.
- **Ticket server / DB range allocation:** each worker grabs a range of IDs.

### B11. Search
- **Inverted index:** term → list of document IDs. Used by Elasticsearch / OpenSearch.
- **Typeahead:** prefix-based trie or precomputed top-k per prefix, cached.

### B12. Geospatial
- **Geohash:** encodes lat/long into a string; shared prefix means nearby. Easy to shard.
- **Quadtree:** recursively split space by density. Good for uneven distribution.
- **S2 / H3:** Google/Uber cell systems used in production.

### B13. Blob Storage and CDN
- Store large files (images, video, backups) in S3-like object storage. Store only metadata in the DB.
- Use pre-signed URLs for upload/download. Multipart upload for big files.
- Video: transcode to multiple bitrates (adaptive streaming: HLS/DASH), serve via CDN.

### B14. Observability
- **Metrics:** RED (Rate, Errors, Duration) for services; USE (Utilization, Saturation, Errors) for resources.
- **Logs and traces:** correlation IDs across services.
- **Alerts:** on SLO burn rate, not raw CPU.

### B15. Security basics
- AuthN (who you are) vs AuthZ (what you can do). OAuth2/OIDC for login; JWT for stateless access tokens; RBAC for roles.
- Encrypt in transit (TLS) and at rest. Never store raw passwords (bcrypt/argon2).

### B16. Failure handling (say this in every interview)
- **Retries** with exponential backoff and jitter.
- **Timeouts** on every network call.
- **Circuit breaker:** stop calling a failing dependency for a while.
- **Bulkheads:** isolate resource pools per dependency.
- **Graceful degradation:** serve stale or partial data.
- **Replication and multi-AZ** for availability.

---

## Part C: Estimation Cheat Sheet

### Powers of 2 / 10
| Power | Approx | Unit |
|---|---|---|
| 2^10 | 1 thousand | KB |
| 2^20 | 1 million | MB |
| 2^30 | 1 billion | GB |
| 2^40 | 1 trillion | TB |

### Time
- 1 day ≈ 86,400 s ≈ **10^5 s** (use this for quick math)
- 1 month ≈ 2.5 × 10^6 s
- 1 year ≈ 3 × 10^7 s

### Latency (order of magnitude)
- L1 cache: ~1 ns
- Main memory: ~100 ns
- SSD random read: ~100 µs
- Same-DC round trip: ~0.5 ms
- HDD seek: ~10 ms
- Cross-continent round trip: ~150 ms

### Formulas
- **Average QPS** = (DAU × actions per user per day) / 86,400
- **Peak QPS** ≈ 2–3 × average
- **Storage per year** = (writes per day × size per record × 365)
- **Bandwidth** = QPS × average payload size
- **Read:write ratio** drives caching and replica decisions.

### Worked example: URL shortener
- 100M new URLs/month → 100M / 2.5M s ≈ **40 writes/s** (average), peak ~100.
- Read:write = 100:1 → **4,000 reads/s** average, peak ~10,000.
- Each record ~500 bytes → 100M × 12 × 500 B ≈ **600 GB/year**.
- Conclusion: a single relational DB could handle writes, but read cache and sharding planning are needed later.

**Practice:** do this estimate for every case study below. Interviewers care more about the reasoning than the exact number.

---

## Part D: Case Studies

Each case gives: requirements, scale, main flow, components, deep dives, and common mistakes. Use the framework from Part A.

---

### Case 1: URL Shortener (TinyURL)
**Requirements:** shorten a long URL; redirect; optional custom alias; expiry; analytics (basic).
**Scale:** 100M URLs/month, 100:1 read:write.
**API:**
- `POST /urls` {long_url, alias?, ttl?} → {short_url}
- `GET /{code}` → 301/302 redirect

**Data model:** `(short_code PK, long_url, created_at, expires_at, user_id)`. KV store works well.
**Short code:** base62 of a unique ID (Snowflake or range allocator), 7 chars gives 62^7 ≈ 3.5 trillion codes.
**Flow:** write → generate ID → base62 → store → return. Read → cache lookup → DB on miss → redirect.
**Components:** LB → app servers → Redis (cache-aside, 80/20 rule) → KV DB (sharded by code) → analytics via Kafka.
**Deep dives:**
- ID generation without collisions across servers (range allocation, Snowflake).
- Hot links going viral: cache with TTL; CDN for redirects.
- 301 (cacheable, less analytics) vs 302 (analytics-friendly).
**Mistake to avoid:** hashing the long URL (MD5) and taking the first 7 chars, without handling collisions.

---

### Case 2: Distributed Rate Limiter
**Requirements:** limit requests per user/API key/IP; return 429; low overhead (<1 ms); works across many app servers.
**Scale:** 1M users, 100K req/s overall.
**Where:** at the API gateway or as middleware. Store counters in Redis.
**Algorithm:** token bucket with a Lua script in Redis (atomic). Or sliding window counter for simpler limits.
**Deep dives:**
- Race conditions: use Lua script or `INCR` with `EXPIRE` in a transaction.
- Redis failure: fail open (allow traffic) vs fail closed. Choose per use case; usually fail open for public APIs.
- Multi-region: per-region limits, or a global limit with async sync (accept some overage).
- Hot users: separate limits per tier; local in-memory pre-check to reduce Redis load.
**Mistake to avoid:** using a database for counters.

---

### Case 3: Distributed Cache (like Redis or Memcached)
**Requirements:** get/set/delete with TTL; horizontal scaling; high availability; sub-ms latency.
**Scale:** 1 TB data, 500K ops/s.
**Components:**
- Client-side or proxy-based routing using consistent hashing.
- Partitions with replicas (master–slave). Failover via a coordinator (ZooKeeper, etcd) or gossip.
**Deep dives:**
- Consistent hashing with virtual nodes; rebalancing on node add/remove (only ~1/N keys move).
- Replication: async master → replica (fast, may lose recent writes on failover). Sync is slower.
- Eviction: LRU with approximate tracking (sample k keys).
- Hot keys and cache stampede.
- Persistence: snapshot (RDB) and append log (AOF).
**Link to your project:** explain how your Node.js cache handled node churn, why you chose 200 vnodes, and what the failover looked like.

---

### Case 4: Notification System
**Requirements:** push, SMS, email, in-app; templates; user preferences; rate limits per user; retries; tracking delivery status.
**Scale:** 10M notifications/day, bursts 10× during events.
**Flow:** service → API → validate and dedupe → queue (Kafka, one topic per channel) → workers per channel → third-party provider (APNs, FCM, Twilio, SES) → status events.
**Deep dives:**
- Priority: separate queues for OTP/transactional vs marketing, so promotions don't delay OTPs.
- Idempotency: notification ID as key; providers may duplicate.
- Retries with backoff; DLQ after N attempts.
- User preference and quiet hours: check before sending.
- Fan-out to millions (broadcast): batch, rate-limit to provider limits.
- Template rendering: cache compiled templates.

---

### Case 5: Chat Application (WhatsApp / Slack)
**Requirements:** 1:1 and group chat; delivery receipts; online presence; message history; multi-device; end-to-end encryption (optional).
**Scale:** 50M DAU, 10 messages per user per day → 500M msgs/day ≈ 6K msgs/s average.
**Components:**
- WebSocket gateway servers (stateful connection); a connection registry maps user → gateway server (Redis).
- Message service writes to a store (wide-column or sharded DB keyed by `chat_id`, ordered by time/Snowflake ID).
- Pub/sub (Kafka or Redis pub/sub) routes messages to the gateway holding the recipient's connection.
- Offline users: store and push notification.
**Deep dives:**
- Ordering: per-chat sequence numbers.
- Delivery guarantees: client ACKs; resend on no ACK; idempotent message IDs.
- Presence: heartbeats with TTL in Redis; avoid broadcasting presence to everyone (fan-out limits).
- Group fan-out: small groups fan out on write; large channels fan out on read.
- Sync across devices: per-device cursor.
**Mistake to avoid:** polling for messages (ok for small scale; WebSocket for real-time).

---

### Case 6: News Feed / Twitter Timeline
**Requirements:** post, follow, home timeline; timeline latency <200 ms; celebrities with millions of followers.
**Scale:** 300M DAU, 2 posts/day/user for a small fraction; 10:1 read:write.
**Core decision: fan-out on write vs fan-out on read.**
- *Write (push):* on post, insert into each follower's precomputed timeline (Redis list). Fast reads, expensive for celebrities.
- *Read (pull):* merge posts from followed users at read time. Cheap writes, slow reads.
- *Hybrid:* push for normal users; pull for celebrities at read time and merge.
**Components:** post service → Kafka → fan-out workers → timeline cache (Redis) → feed service. Media on object storage + CDN. Ranking service for ML-based ordering (optional).
**Deep dives:** celebrity problem, hot tweets, cache rebuild after eviction, pagination with cursor (not offset).

---

### Case 7: Typeahead / Autocomplete
**Requirements:** top 5 suggestions per prefix; <100 ms; updated daily or hourly; typos optional.
**Scale:** 1B searches/day → ~12K/s average, ~40K/s peak.
**Components:**
- Data collection: query logs → Kafka → aggregation (Spark/Flink) → counts per phrase.
- Build a trie or precomputed top-k per prefix; store in a KV or cached per prefix.
- Serve from an in-memory trie / cache close to users.
**Deep dives:**
- Trie memory: compress (radix tree); top-k stored at each node.
- Update strategy: periodic rebuild (blue/green) rather than live mutation.
- Personalization and filtering of unsafe terms.
- Client-side debounce to cut requests.

---

### Case 8: Ride Sharing (Uber)
**Requirements:** rider requests ride; match with nearby driver; live location; ETA; pricing; trip history.
**Scale:** 5M drivers sending location every 4 s → ~1.25M location updates/s.
**Components:**
- Location service: stores driver location in an in-memory geospatial index (geohash/H3 cells), sharded by cell.
- Matching service: query nearby cells, rank drivers by ETA, send offers with timeout.
- Trip service: state machine (requested → matched → picked up → completed); persist in DB.
- Real-time updates to rider via WebSocket.
**Deep dives:**
- High write rate for location: don't store every update durably; keep latest position in memory; sample for history.
- Matching race: a driver accepts two rides; use a lock or atomic "offer state" transition.
- Surge pricing: compute per cell from demand/supply in a stream job.
- Region failover and dispatch correctness.

---

### Case 9: Payment System
**Requirements:** pay for an order; refunds; no double charge; PCI compliance; reconciliation with the bank.
**Scale:** 1M transactions/day (~12/s average, peak ~100/s).
**Components:** payment API → idempotency store → ledger service (double-entry, append-only) → payment processor adapter → webhook handler → reconciliation job.
**Deep dives:**
- **Idempotency:** client sends `Idempotency-Key`; store key + response; return the same result on retry.
- **State machine:** `created → processing → succeeded / failed → refunded`. Transitions are atomic (DB constraints or versioning).
- **Exactly-once effect:** at-least-once messages + idempotent consumers + ledger uniqueness.
- **Money:** store amounts as integers in minor units; never floats.
- **Reconciliation:** nightly job matches internal ledger with the processor's settlement file.
- **Strong consistency:** choose a relational DB with transactions.

---

### Case 10: Web Crawler
**Requirements:** crawl billions of pages; respect robots.txt; politeness per domain; avoid duplicates; refresh periodically.
**Scale:** 1B pages/month → ~400 pages/s.
**Components:**
- URL frontier: priority queue + per-domain queues for politeness.
- Fetcher workers (distributed); DNS cache.
- Content parser → dedupe with content hash (SimHash/MinHash for near-duplicates).
- URL seen-set: Bloom filter (memory efficient, false positives OK) backed by a store.
- Storage: object store for raw HTML; index pipeline separate.
**Deep dives:** politeness and robots.txt caching; traps (infinite URLs); prioritizing fresh pages; fault tolerance (checkpoint frontier).

---

### Case 11: Distributed File Storage (Dropbox / Google Drive)
**Requirements:** upload/download files; sync across devices; versioning; sharing; large files.
**Scale:** 100M users, 500 MB average storage each → 50 PB (say so; then scope).
**Components:**
- Metadata service (file tree, versions, permissions) on a relational DB, sharded by user.
- Block storage: split files into chunks (4 MB), content-addressed by hash → dedupe.
- Object store for chunks; CDN for downloads.
- Sync service: notifications via long-poll/WebSocket; change log per user to compute deltas.
**Deep dives:**
- Chunking: only changed chunks upload (delta sync).
- Conflict resolution on concurrent edits (last writer wins vs create conflict copy).
- Upload resumability with chunk offsets.
- Consistency between metadata and blobs: write blob first, then metadata; GC orphaned blobs.

---

### Case 12: Metrics / Monitoring System (like Datadog or Prometheus)
**Requirements:** ingest metrics from 100K hosts; query dashboards; alerting; retention 1 year with downsampling.
**Scale:** 100K hosts × 500 metrics × every 10 s → 5M points/s.
**Components:**
- Agents → ingestion gateway → Kafka (partition by metric name/host).
- Time-series DB (Druid, Prometheus TSDB, InfluxDB, or Cassandra-style store) with rollups.
- Query service with caching for dashboards.
- Alert evaluator: streaming or periodic queries; notifications with dedupe and grouping.
**Deep dives:**
- Compression of time-series (delta-of-delta, Gorilla).
- Cardinality explosion (too many label combinations) and limits.
- Downsampling: 10 s raw for 7 days, 1 min for 90 days, 1 h for 1 year.
- Late and out-of-order data: watermarks; windowing.
**Link to your project:** Druid ingestion from Kafka, segment compaction, rollup, and query latency. Use this as a strong story.

---

### Case 13: Video Streaming (YouTube / Netflix)
**Requirements:** upload; transcoding; streaming with adaptive bitrate; search; view counts.
**Scale:** 500 hours uploaded/min; 1B views/day.
**Components:**
- Upload to blob storage (resumable) → event → transcoding workers (multiple resolutions, codecs) → segments (HLS/DASH) → object store.
- CDN for segments; origin shield.
- Metadata DB; search index; view-count pipeline (approximate counts with batching).
**Deep dives:**
- Adaptive bitrate: client picks quality based on bandwidth.
- Transcoding is expensive: priority queues, parallel chunked encoding.
- Hot videos: CDN caching; popularity-based replication.
- View counts: write-behind aggregation, not one DB write per view.

---

### Case 14: Distributed Job Scheduler / Workflow Engine
**Requirements:** schedule one-time and recurring jobs; retries; exactly-once-ish execution; at least 1M jobs/day; visibility.
**Components:**
- API → job store (DB with `next_run_at` index).
- Scheduler polls due jobs (or uses time-wheel / delay queue); claims jobs with lease (`locked_until`).
- Workers execute; heartbeat extends lease; on expiry, another worker retries.
- Workflow orchestration for multi-step flows (Temporal-style durable execution).
**Deep dives:**
- Duplicate execution: leases + idempotent job handlers.
- Clock skew and time-wheel precision.
- Priority and fairness across tenants.
- Partitioning the scheduler (shard by job ID hash).
**Link to your project:** your Temporal workflows: retry policies, timeouts, task queues, and how you handled long-running ETL.

---

### Case 15: Event Pipeline / Log Analytics
**Requirements:** collect events from apps; process in near real time; store for analytics; dashboards; replay.
**Scale:** 1B events/day → ~12K/s average, peak 50K/s.
**Components:** SDK → ingestion API (validate, enrich) → Kafka (topics by domain, partitioned by key) → stream processing (Flink/Spark Structured Streaming, or consumers) → OLAP store (Druid, ClickHouse) and data lake (Iceberg/Parquet on S3).
**Deep dives:**
- Partition key choice (even load, per-entity ordering).
- Exactly-once in aggregations: idempotent writes using event IDs; dedupe window.
- Late data: watermarks and allowed lateness.
- Schema evolution: schema registry (Avro/Protobuf); backward-compatible changes.
- Compaction of small files in the lake (Iceberg); partition pruning for queries.
- Replay: keep Kafka retention or read from the lake for backfills.
**Link to your project:** the Kafka → Druid and Kafka → Iceberg migration. Explain why you moved batch-heavy ingestion, what you gained (40% less supervisor load), and the trade-offs (freshness vs cost).

---

## Part E: Common Mistakes (check yourself)
1. Jumping to a design before asking requirements.
2. Skipping estimates, then choosing a technology with no reason.
3. Drawing many boxes with no data flow.
4. Ignoring failure: "what if this node dies?"
5. Choosing Kafka or Redis as a buzzword without explaining the guarantee you need.
6. Going too deep on one component too early, then running out of time.
7. Forgetting monitoring, security, and cost.
