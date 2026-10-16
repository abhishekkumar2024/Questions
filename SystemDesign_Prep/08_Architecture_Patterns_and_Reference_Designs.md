# 08. Architecture Patterns and Reference Designs

---

## 8.1 Architectural Styles

### Monolith
- One deployable unit. Simple to develop, test, deploy, and debug. Transactions are local.
- Scales vertically and by horizontal replicas of the whole app. Breaks down when teams and deploy frequency collide.
- **Modular monolith:** strict module boundaries, separate schemas, internal APIs. Good default for small teams. Can be split later.

### Microservices
- Independently deployable services, each owning its data.
- **Benefits:** team autonomy, independent scaling, fault isolation, technology choice.
- **Costs:** network calls (latency, failures), distributed transactions, observability, versioning, operational overhead, data duplication, harder testing.
- **Decompose by:** business capability or bounded context (DDD), not technical layer. Signs of a bad split: every feature touches 5 services; chatty calls; shared database.
- **Database per service:** each service owns its data; others access via API or events.
- **Start monolith, split when pain is real.** Strangler fig migration: route pieces of traffic to new services gradually behind a facade.

### Event-driven architecture
- Services publish facts ("OrderPlaced") to a broker; interested services react.
- **Choreography** (decentralized) vs **orchestration** (central workflow).
- Decouples producers and consumers in time and space; enables new consumers without changing producers.
- **Risks:** eventual consistency, duplicate events, out-of-order events, schema drift, hard-to-trace flows (needs tracing and event catalogs).
- **Event notification** (small "something happened", fetch details) vs **event-carried state transfer** (include the data) vs **event sourcing** (events are the source of truth).

### Event sourcing
- Store the sequence of events as the source of truth; current state = fold over events. Snapshots every N events for speed.
- **Benefits:** full audit trail, temporal queries, replay for new projections, debugging.
- **Costs:** schema evolution of events (upcasting), eventual consistency for read models, complexity, GDPR deletion (crypto-shredding: encrypt per-user data with a key you delete).

### CQRS (Command Query Responsibility Segregation)
- Separate write model (commands, normalized, consistent) and read models (queries, denormalized, optimized per view).
- Read models are updated from events (eventual consistency). Pair with event sourcing often, but not required.
- Use when read and write shapes or scaling needs differ greatly (e.g., feed rendering vs order placement).

### Layered / Hexagonal / Clean architecture
- **Hexagonal (ports and adapters):** domain core has no framework dependency; adapters implement ports for DB, HTTP, queues. Makes testing and swapping infrastructure easier.
- Dependency rule: dependencies point inward toward the domain.

### Serverless
- Functions triggered by events; scale to zero; pay per invocation.
- **Limits:** cold starts, execution time limits, no long-lived connections (WebSockets need a managed gateway), vendor lock-in, harder local testing, connection pool exhaustion to databases (use proxies like RDS Proxy).
- Good for spiky, event-triggered, glue workloads.

### Cell-based architecture
- Partition the whole stack into self-contained cells, each serving a slice of users, with its own data and dependencies.
- Limits blast radius (a bad deploy or hot tenant affects only one cell). A thin routing layer maps users to cells.

### Data architectures
- **Data warehouse:** structured, curated, schema-on-write, SQL analytics (Snowflake, BigQuery, Redshift).
- **Data lake:** raw files on object storage, schema-on-read; with table formats (Iceberg) becomes a "lakehouse."
- **Lambda:** batch + speed layers (two code paths).
- **Kappa:** stream-only, replay for reprocessing.
- **Medallion (bronze/silver/gold):** raw → cleaned → business-level aggregates.

### Service mesh and sidecars
- Sidecar proxy (Envoy) per pod handles mTLS, retries, timeouts, load balancing, metrics, tracing, without app code changes.
- Control plane (Istio, Linkerd) pushes config. Adds latency and resource overhead; adopt when fleet size justifies it.

### BFF (Backend for Frontend)
- A thin backend per client type (web, mobile, TV) aggregating downstream APIs into the shape each client needs.

### Gateway and aggregator
- API gateway for cross-cutting concerns; aggregation layer (GraphQL or BFF) for composing responses.

---

## 8.2 Reference Architectures for Common Systems

Use these as starting skeletons. In an interview, adapt to requirements and say what you'd change.

### 8.2.1 E-commerce
```
Client → CDN (static, product images) → API gateway (auth, rate limit)
  → Catalog service (search index: Elasticsearch; product DB; cache)
  → Cart service (Redis, TTL per user)
  → Order service (saga/orchestrator; outbox → Kafka)
      → Inventory (reservation with TTL; atomic decrement)
      → Payment (idempotent; PSP adapter; webhooks)
      → Shipping / Notification (async consumers)
  → Analytics: Kafka → lake (Iceberg) → warehouse
```
**Key details:** inventory reservation with expiry (avoid overselling via conditional decrement); checkout saga with compensations (release inventory on payment failure); search index updated via CDC; price consistency at checkout.

### 8.2.2 Social feed (Twitter/Instagram-like)
```
Post service → DB (sharded by author) + object store (media) → Kafka "post.created"
  → Fan-out workers → per-user timeline cache (Redis lists/sorted sets)
  → Feed service: merge precomputed timeline + pulled celebrity posts; ranking
  → Edge cache for hot feeds; CDN for media
Graph service: follow relations (sharded adjacency lists, cached counts)
```
**Key details:** hybrid fan-out; cursor pagination; ranking with a separate scoring service; counts are approximate (async counters).

### 8.2.3 Messaging (WhatsApp/Slack-like)
```
Clients ⇄ WebSocket gateways (stateful; connection registry in Redis: user → gateway id)
  → Message service: assign per-chat sequence IDs, persist (wide-column or sharded DB by chat_id)
  → Pub/sub routing (Kafka partitioned by chat or user; or Redis pub/sub channel per gateway)
  → Delivery to online recipients; push notifications (APNs/FCM) for offline
  → Media: upload to object store with pre-signed URLs; thumbnails async
Presence: heartbeats with TTL in Redis; coarse updates
```
**Key details:** ordering per chat; at-least-once delivery with client ACKs and dedupe by message ID; end-to-end encryption shifts key management to clients; multi-device sync via per-device cursor.

### 8.2.4 Video streaming (YouTube/Netflix-like)
```
Upload (resumable, pre-signed multipart) → raw storage → event
  → Transcoding workers (DAG of jobs: per resolution/codec; chunked parallel encoding)
  → Packaging (HLS/DASH segments + manifests) → origin object store
  → CDN with multi-tier caching (edge + regional + origin shield)
  → Playback: client adaptive bitrate (ABR) with buffer-based or throughput-based selection
Metadata DB, search index, recommendations, view counts (batched aggregation)
```
**Key details:** transcoding cost control (encode only needed renditions; popularity-based extra encodes); DRM; thumbnail generation; start-up latency.

### 8.2.5 Ride sharing (Uber-like)
```
Driver app → location updates (every 3–5 s) → Location ingestion (in-memory geo index per cell: H3/S2)
Rider app → ride request → Matching service (query nearby cells; ETA via routing service; offer with timeout)
Trip service: state machine in DB (requested → matched → en route → started → completed)
Pricing: surge computed per cell from supply/demand stream
Realtime: WebSocket/push to rider and driver; maps/ETA service with precomputed road graph
```
**Key details:** location is ephemeral (keep latest only; sample for history); offer locking so a driver gets one offer; ETA uses road-network routing (contraction hierarchies), not straight-line.

### 8.2.6 Payments
```
Merchant → Payment API (Idempotency-Key) → Payment orchestrator (state machine)
  → Ledger (double-entry, append-only, strongly consistent DB)
  → PSP adapter (card networks / bank rails) → webhooks → reconciliation
  → Risk/fraud service (sync scoring in path, timeouts with fallback rules)
  → Audit log, PCI scope limited to tokenization service
```
**Key details:** exactly-once effect via idempotency and ledger constraints; money as integer minor units; async webhooks with signature verification; daily reconciliation against settlement files; refunds and chargebacks as separate ledger events.

### 8.2.7 Search (Google/Elasticsearch-like)
```
Crawler/ingest → normalization → indexing pipeline (Kafka → indexers)
  → Inverted index shards (term → postings; doc values for sorting) with replicas
  → Query: parse → fan-out to shards → merge top-k → rank (BM25 + ML re-rank)
  → Cache for popular queries; autocomplete via separate prefix index
```
**Key details:** near-real-time indexing (refresh interval); segment merges; relevance tuning; typo tolerance (edit distance); sharding by document, querying all shards.

### 8.2.8 Distributed key-value store (Dynamo/Cassandra-like)
```
Client → coordinator node (any node) → consistent-hash ring with vnodes
  → N replicas (preference list), quorum W/R, vector clocks / LWW for conflicts
  → Storage engine: commit log + memtable + SSTables (LSM) + Bloom filters
  → Gossip membership, failure detection (phi accrual), hinted handoff, read repair, Merkle-tree anti-entropy
```
**Key details:** tunable consistency per request; no single point of failure; compaction strategy; tombstones and TTL.

### 8.2.9 Metrics/monitoring (Prometheus/Datadog-like)
```
Agents/SDKs → ingestion gateway (auth, rate limits, cardinality limits)
  → Kafka (partition by series hash) → TSDB writers (time-bucketed blocks, compression: delta-of-delta + XOR)
  → Query layer (downsampling, caching, fan-out) → dashboards
  → Rule evaluation / alerting service (dedupe, grouping, routing)
Long-term: rollups in columnar store (Druid / ClickHouse)
```

### 8.2.10 Log analytics and event pipeline
```
SDK / agents → collectors (batch, compress) → Kafka (topics by domain, schema registry)
  → Stream processing (enrich, dedupe, aggregate; watermarks)
  → Hot store (Druid/ClickHouse: sub-second queries on recent data)
  → Cold store (Iceberg/Parquet on S3: cheap, full history, compaction)
  → Query engine (Trino/Spark) for ad hoc; retention policies per tier
```

### 8.2.11 Job scheduler / workflow engine
```
API → workflow/job definition store
  → Durable execution engine (Temporal-style: workflow history event-sourced; activities retried)
  → Task queues per worker pool; leases and heartbeats; timers for delays
  → Visibility store (search by status), audit history
```

### 8.2.12 File sync (Dropbox-like)
```
Client: chunking (content-defined chunking, e.g., Rabin fingerprint), hashing, local index
  → Metadata service (sharded DB: file tree, versions, ACLs)
  → Block service (content-addressed chunks in object store, dedupe across users)
  → Notification service (long-poll / WebSocket: "changes available for namespace X")
  → Sync engine: delta computation from change log cursor
```

### 8.2.13 Web crawler
```
Seeds → URL frontier (priority + per-host politeness queues)
  → DNS cache → fetchers (robots.txt cache; rate per host)
  → Parser → content dedupe (SimHash) → URL normalization → seen-set (Bloom filter + store)
  → Storage (raw object store, index pipeline)
```

---

## 8.3 Pattern Selection Cheat Sheet

| Need | Pattern |
|---|---|
| Atomic multi-service action | Saga (orchestration) + outbox + idempotency |
| Reliable event publishing | Transactional outbox / CDC |
| Fast, varied reads from writes | CQRS with read models |
| Full audit history | Event sourcing |
| Many consumers of a stream | Kafka consumer groups |
| Protect dependency from overload | Timeout + bulkhead + circuit breaker + load shedding |
| Massive hot reads | CDN + multi-tier cache + read replicas |
| Hot write key | Sharding by key + salting + batching |
| Isolate failures | Cells, bulkheads, per-tenant queues |
| Multi-region writes | Regional ownership (geo-partitioning) or CRDTs |
| Zero-downtime release | Canary + feature flags + expand/contract migrations |
| Cross-service auth | OIDC at edge + JWT propagation + mTLS internally + authz at service |

---

## 8.4 How to Pick a Database (decision guide)

- Relational with joins and transactions, moderate scale → PostgreSQL / MySQL (+ replicas, partitioning, later sharding like Citus, Vitess).
- Global strong consistency, horizontal SQL → Spanner, CockroachDB, YugabyteDB.
- Wide-column, massive writes, known query patterns → Cassandra, ScyllaDB, HBase, Bigtable.
- Key-value, low latency, managed scale → DynamoDB (single-table design), Redis (in-memory, rich types).
- Document, flexible schema → MongoDB, Couchbase.
- Search and text → Elasticsearch / OpenSearch.
- Time-series → Prometheus TSDB, InfluxDB, TimescaleDB, Druid.
- OLAP analytics → ClickHouse, Druid, Pinot, BigQuery, Snowflake.
- Graph relationships → Neo4j, or a sharded adjacency store.
- Object/blob → S3, GCS, Azure Blob.

**Justify with access patterns, consistency needs, scale, and team familiarity.** Never pick on popularity alone.

---

## 8.5 Interview Technique for Architecture Questions

1. Start from the user flow and draw the simplest version that works.
2. Add components only when a requirement forces it, and say why ("we need async because the email provider is slow and we don't want checkout to wait").
3. Make every arrow a decision: sync or async, which protocol, what happens on failure.
4. Name the bottleneck before the audience does.
5. For each component: its data, its failure mode, its scaling plan.
6. End with what you'd monitor and what you'd build next.
