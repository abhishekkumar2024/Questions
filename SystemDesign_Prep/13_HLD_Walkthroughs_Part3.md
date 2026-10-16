# 13. Full HLD Walkthroughs, Part 3 (Cases 11–15)

---

## Case 11: Distributed File Storage (Dropbox / Google Drive)

### 1. Requirements
- Upload, download, and sync files across devices; folders; versions (keep 30 days); sharing with permissions; offline edits sync when online.
- Files up to 50 GB; sync latency < 10 s for small changes; no lost updates on concurrent edits (conflict copies, never silent overwrite).
- Scale: 100M users, 500M devices, average 2 GB stored per user → **200 PB** total logical data before dedupe. (Say this explicitly, then scope: design the mechanisms that scale to that.)

### 2. Estimates
- Changes: 500M devices × 10 file changes/day ≈ 5B events/day ≈ **58K change events/s** average.
- Bandwidth: with chunk-level sync, a typical 1 MB edit to a 1 GB file transfers ~4 MB of chunks, not 1 GB.
- Chunks: 4 MB average chunk → 200 PB / 4 MB ≈ **50 billion chunks**. Chunk index must be sharded by hash.
- Metadata: 100M users × 10K files × 500 B ≈ 500 TB. Sharded relational store.

### 3. Core idea: chunking + content addressing
- Split file into chunks using **content-defined chunking** (rolling hash, e.g., Rabin fingerprint, boundary when hash matches a pattern). An insertion at the start of a file only changes nearby chunks, not all of them (fixed-size chunks would shift everything).
- Each chunk stored under `sha256(content)`. Identical chunks across users are stored once (dedupe, per-user encryption complicates cross-user dedupe — mention the trade-off).
- A file = ordered list of chunk hashes ("manifest"). Version = new manifest.

### 4. API
```
POST /v1/files/upload-session  {path, size, chunk_hashes[]}  -> {missing_chunks[], upload_urls[]}
PUT  <presigned chunk url>       (chunk body)
POST /v1/files/commit          {path, manifest_id, base_version}  -> 200 or 409 (conflict)
GET  /v1/changes?cursor=       -> change feed for sync
GET  /v1/files/{id}/versions
```
The client sends hashes first; the server replies which chunks are missing. Only missing chunks upload.

### 5. Architecture
```
Client: chunker + local index (path → manifest, chunk hashes), change watcher
   │ metadata calls                       │ chunk bytes
   ▼                                      ▼
Metadata service (sharded by user_id)   Block service → object store (chunks keyed by hash)
   ├─ file tree, versions, ACLs           (pre-signed URLs; CDN for downloads)
   ├─ change log (per user, ordered)
Notification service: long-poll / WebSocket "namespace X changed, cursor=N"
Sync: client fetches changes after cursor, downloads missing chunks, reassembles
```

### 6. Data model
- **files**: `(user_id, file_id)` PK, `path, current_version, deleted, updated_at`.
- **versions**: `(file_id, version)` PK, `manifest_id, size, created_at, device_id`.
- **manifests**: `manifest_id → [chunk_hash...]` (stored as blob or rows).
- **chunks**: `chunk_hash PK, size, storage_location, refcount`.
- **change_log**: `(user_id, seq)` PK — monotonically increasing per user; sync reads by seq.

### 7. Flows
- **Upload:** chunk locally → send hashes → get missing list → upload missing chunks → commit manifest with `base_version`. If `base_version` ≠ current, return 409 → client creates a conflict copy `file (conflicted copy from device X)`.
- **Download/sync:** client reads change feed after cursor; for each changed file, fetches manifest, requests chunks it lacks, reassembles to a temp file, atomic rename.

### 8. Deep dives
**a) Conflict resolution.** Optimistic concurrency via `base_version`. Never last-write-wins silently for files; create conflict copies. Text merge possible for specific formats (not general files).
**b) Garbage collection.** Chunk refcount decremented when versions expire; GC is a background mark-and-sweep: mark chunks reachable from live manifests, delete unreferenced chunks older than a grace period (protects in-flight uploads).
**c) Consistency between metadata and blobs.** Upload chunks first, then commit metadata. A commit referencing missing chunks is rejected (server checks existence). Orphan chunks (uploaded but never committed) are reclaimed by GC after grace period.
**d) Large files and resumability.** Chunk-level resumes: upload session records which chunks are done; retries re-send only missing chunks. Chunk uploads are idempotent (same hash = same content).
**e) Sync latency at scale.** Change feed per user is tiny; notification service fans out "changed" pings only to devices of that user (per-user channels), not broadcasts.
**f) Sharing.** ACL table `(file_id, principal, role)`; permission checks at metadata service; links with tokens and expiry; download via CDN with signed URLs.
**g) Encryption.** Per-user or per-file keys (envelope encryption); cross-user dedupe trades privacy (convergent encryption leaks equality). State the choice.

### 9. Failure table
| Failure | Behavior |
|---|---|
| Object store region down | Replicated buckets; read failover; uploads queued on client |
| Client crash mid-sync | Temp file discarded; cursor not advanced until apply completes |
| Metadata shard down | That user's sync pauses; no corruption since commits are atomic |

### 10. Probes
- *Why content-defined chunking?* Fixed-size chunks re-upload everything after an insertion; CDC limits changes to nearby chunks.
- *How do you handle a 50 GB upload on flaky wifi?* Resumable chunked upload sessions with per-chunk acks.
- *Dedupe across users is a privacy risk; how do you mitigate?* Per-user keys make cross-user dedupe impossible; or dedupe only within a user, or use convergent encryption knowingly.

---

## Case 12: Metrics / Monitoring System (Prometheus / Datadog)

### 1. Requirements
- Ingest metrics from 100K hosts × 500 series × every 10 s ≈ **5M points/s**.
- Dashboard queries < 1 s for last 24 h; alerting evaluation every 30 s; retention: raw 7 days, 1-minute rollups 90 days, 1-hour rollups 2 years.
- Multi-tenant; cardinality limits per tenant.

### 2. Estimates
- 5M points/s × 16 B (compressed to ~1.5 B/point with Gorilla-style compression) ≈ 7.5 MB/s compressed ingest → ~650 GB/day raw compressed? Compute carefully: 5M × 86,400 = 432B points/day; × 1.5 B ≈ **650 GB/day** compressed. With 7-day raw retention ≈ 4.5 TB, replication ×3 ≈ 14 TB.
- Series count: 100K hosts × 500 = **50M active series**. Each series index entry ~200 B → 10 GB of index: keep index in memory across shards.

### 3. Data model
- A series = metric name + label set (`host`, `service`, `region`). Identity = hash of sorted labels.
- Storage: time-bucketed blocks (2-hour blocks in Prometheus-style TSDB), each with series index + compressed chunks.
- Inverted index: label → series IDs (postings lists), intersected for queries like `service="api" AND region="eu"`.

### 4. Architecture
```
Agents/SDKs → ingestion gateway (auth, batch, rate and cardinality limits)
  → Kafka (partition by series hash, for ordering per series and parallelism)
  → Writers (write-ahead log → in-memory head block → flush to block store every 2 h)
  → Storage: local SSD (recent) + object store (blocks, long-term)
  → Query frontend (split queries by time, cache results, fan out to shards)
  → Rules engine: recording rules (pre-aggregate) + alert rules every 30 s
  → Alert manager: dedupe, group, silence, route (paging, chat)
Downsampler: raw → 1 min → 1 h rollups as background jobs
```

### 5. Compression
- Timestamps: delta-of-delta encoding (regular 10 s intervals compress to ~1 bit per point).
- Values: XOR encoding of floats (consecutive values often similar → few bits).
- Result ~1–2 bytes per point (Gorilla paper's approach).

### 6. Deep dives
**a) Cardinality explosion.** A label with unbounded values (user_id, request path with IDs) creates millions of series; memory and index explode. Defenses: label allowlist, cardinality limits per tenant (reject new series beyond quota), normalize paths to route templates, monitor series churn.
**b) Late and out-of-order data.** Agents buffer during outages and send old points. Accept within a window (e.g., 1 h); head block supports out-of-order inserts within its range; older data needs a backfill path.
**c) Query performance.** Split long ranges into sub-queries by time; cache results of immutable past buckets; pre-aggregate with recording rules for dashboards that run constantly.
**d) Downsampling correctness.** Store aggregates per bucket (min, max, sum, count) not just average, so averages of averages don't lie; percentiles need histograms or sketches (t-digest), not averaged percentiles.
**e) Alert evaluation at scale.** 50M series with thousands of rules: evaluate rules by query groups; sharded evaluators; a missing data alert is separate from threshold alerts (no data ≠ zero).
**f) Ingestion durability.** Kafka between agents and writers buffers spikes; writers ack after WAL write; replicas across zones.
**g) Cost.** Tiered storage: recent on SSD, older on object store; retention per tenant; sampling for high-volume debug metrics.

### 7. Failure table
| Failure | Behavior |
|---|---|
| Writer down | Kafka retains; partition reassigned; replay from offset |
| Query shard down | Partial results flagged; retry other replica |
| Alerting pipeline lag | Alert delay monitored by meta-alert (dead man's switch) |

### 8. Your project mapping
Your Druid work is the same family: Kafka ingestion, segments, rollup, compaction, and query latency. Explain rollup as the time-series downsampling story and compaction as the small-files story from your Iceberg work.

### 9. Probes
- *Why not store metrics in PostgreSQL?* Write rate and compression; time-series engines have append-optimized blocks and time partitioning.
- *Average latency dashboard is misleading — why?* Averages hide tails; show percentiles from histograms.
- *High-cardinality request IDs in labels?* Put them in logs or traces, not metrics.

---

## Case 13: Video Streaming (YouTube / Netflix)

### 1. Requirements
- Upload videos (up to 12 hours, 20 GB); transcode to multiple resolutions and codecs; playback with adaptive bitrate; start playback < 2 s.
- Search and recommendations (overview only); view counts; comments (out of scope detail).
- Scale: 500 hours uploaded per minute ≈ **30,000 hours/day**; 1B views/day.

### 2. Estimates
- Raw upload: 30,000 h/day × ~1 GB/hour source ≈ **30 TB/day raw**.
- Transcoded output: ~6 renditions (240p–4K) with total ≈ 3× raw bitrate → 90 TB/day output; storage per year ≈ 30 PB (before tiering). Justify tiering: cold renditions for long-tail videos.
- Playback: 1B views × avg 15 min × ~3 Mbps ≈ 1B × 0.34 GB ≈ **340 PB/day egress** — this is why CDN is the core of the design. (Show the order-of-magnitude math; mention most traffic is served by edge caches.)
- Concurrent viewers at peak: 1B views spread over day with peaks ~5× average → ~50M concurrent streams. Edge capacity is the constraint.

### 3. Architecture
```
Upload: client → resumable upload (pre-signed multipart) → raw storage
  → event → Transcode orchestrator (DAG per video)
       ├─ split into chunks (GOP-aligned) → parallel encode workers (per resolution/codec)
       ├─ audio encode, thumbnails, subtitles
       └─ package: HLS/DASH segments (2–6 s) + manifests (.m3u8 / .mpd)
  → origin store (object storage) → CDN (edge → regional mid-tier → origin shield)
Playback: client fetches manifest → picks rendition by bandwidth/buffer (ABR) → fetches segments from CDN
Metadata DB + search index + recommendation service + view counter pipeline (Kafka → batch aggregation)
```

### 4. Transcoding details
- Split by GOP-aligned chunks (e.g., 30 s) and encode in parallel → speed by parallelism; stitch segments.
- Encode ladder per content: complex content (sports) needs higher bitrates; simple (animation) less — per-title encoding saves 20–40% bandwidth. Mention as an optimization.
- Priority: encode the first low-res rendition fast so the video is playable quickly; higher resolutions follow.
- Cost control: encode popular-predicted ladder fully; long-tail videos get fewer renditions and encode on demand for rarely watched ones.

### 5. Adaptive bitrate (ABR)
- Manifest lists renditions; client measures throughput per segment and buffer level.
- Algorithms: throughput-based (EWMA of download speed) and buffer-based (BOLA, buffer thresholds). Hybrid in practice.
- Segment duration trade-off: shorter segments adapt faster but cost more requests and overhead; 2–6 s typical.

### 6. Deep dives
**a) Egress is the cost.** CDN hit ratio is the key metric. Popular content lives at edges; long tail is pulled from regional tier. Origin shield prevents thundering herd on a newly viral video (all edges missing at once).
**b) Start-up latency.** Manifest and first segments cached at edge; first segment at lowest rendition; pre-fetch first segments for trending videos.
**c) View counts.** 1B views/day = 11.5K writes/s. Don't write per view synchronously. Edge/clients batch view beacons; Kafka → windowed aggregation (1 min) → periodic write; displayed counts are approximate and rounded ("1.2M").
**d) Upload reliability.** Multipart pre-signed uploads with resumable parts; checksum per part; transcode starts only after full upload verified.
**e) DRM and access control.** License server issues keys per session; signed URLs/tokens to CDN with short expiry; prevents hotlinking.
**f) Recommendations and search are separate systems.** Keep them out of the playback path; playback uses only the manifest and CDN.

### 7. Failure table
| Failure | Behavior |
|---|---|
| Transcode worker crash | Chunk retried; idempotent output keyed by (video, rendition, chunk) |
| CDN POP down | Anycast reroutes to next POP; brief rebuffering |
| Origin overload | Origin shield; request coalescing at mid-tier |

### 8. Probes
- *Why not serve the original file?* Devices and bandwidth vary; ABR needs multiple renditions; codecs differ.
- *Why segments instead of one file?* Enables adaptation, caching of small pieces, and resuming.
- *Biggest cost driver?* Egress and transcoding compute; CDN hit ratio and encode ladder decisions move both.

---

## Case 14: Distributed Job Scheduler / Workflow Engine

### 1. Requirements
- Schedule one-time jobs (run at time T) and recurring jobs (cron); workflows with multiple steps, retries, timeouts, and compensation.
- At-least-once execution with idempotent handlers; at most one concurrent run per job unless configured; visibility (status, history, logs).
- Scale: 100M scheduled jobs; 50K jobs starting per second at peak on top of the minute boundary (e.g., many cron jobs at `00:00`).

### 2. Estimates
- 100M job records × 1 KB ≈ 100 GB — fits in a sharded DB.
- Dispatch rate: 50K/s at boundaries; average much lower. Design for bursts with a queue.
- Workers: each job 1 s average → 50K concurrent executions → 500 workers with 100 concurrent tasks each.

### 3. Data model
- **jobs**: `job_id PK, type, payload, schedule (cron/at), next_run_at, status, attempts, lease_owner, lease_expires_at, version`.
- Index: `(status, next_run_at)` for due-job scan.
- **runs/history**: `(job_id, run_id)` with state transitions (immutable events, event-sourced).
- Partition by `job_id` hash (or tenant) for write scale.

### 4. Architecture (two designs)
**Design A — Poll-based (simple):**
```
Scheduler nodes poll: SELECT ... WHERE status='SCHEDULED' AND next_run_at <= now() LIMIT 1000 FOR UPDATE SKIP LOCKED
  → claim with lease (lease_owner, lease_expires_at = now + 30 s)
  → enqueue to task queue (Kafka/SQS/Redis streams) → workers execute
  → heartbeat extends lease; on lease expiry, job becomes claimable again
```
**Design B — Time wheel / delayed queue (fine precision):**
- Hierarchical timing wheel in memory per scheduler node, backed by the DB for durability; jobs due in next N seconds loaded from DB into the wheel. Used when precision < 1 s and volume is huge.

**Durable execution / workflows (Temporal-style):**
```
Client starts workflow → workflow history persisted (event-sourced)
Workflow code (deterministic) → schedules activities → activities run on worker task queues
Activity results recorded in history; on worker crash, another worker replays history to rebuild state
Timers (sleep 7 days) are durable events in history
```

### 5. Deep dives
**a) Exactly-once-ish execution.** Leases prevent two schedulers from claiming the same job, but lease expiry during a long pause can cause double execution. Defense: fencing — each claim gets an incrementing `version`; the job's side effects use the version as an idempotency or fencing token; handlers must be idempotent.
**b) Retries and backoff.** Per-job policy: max attempts, exponential backoff with jitter, non-retryable error types. Dead-lettered after max attempts with the error stored.
**c) Thundering herd at minute boundaries.** Jitter cron jobs (spread start over a window) — a default of spreading by hash of job ID across 0–30 s. Protect DB from scanning at boundaries by prefetching into the queue ahead of time.
**d) Clock skew.** Use DB time (`now()` on the DB) for due checks, not each scheduler's wall clock; or monotonic scheduling with leases.
**e) Concurrency control per job.** "Only one run at a time": a job-level lock (lease); skip or queue the new run if the previous is still running (policy choice, state it).
**f) Workflow determinism (Temporal).** Workflow code must not call non-deterministic APIs directly (current time, random, network); use the SDK's deterministic equivalents. Versioning of workflow code when running instances exist: patch markers so old histories replay correctly.
**g) Visibility.** Query runs by status and time; stream state transitions to a search index; keep logs linked by run ID.
**h) Multi-tenant fairness.** Per-tenant queues with weighted fair scheduling so one tenant cannot starve others.

### 6. Failure table
| Failure | Behavior |
|---|---|
| Scheduler dies | Leases expire; another scheduler claims due jobs; slight delay |
| Worker dies mid-activity | Lease/heartbeat timeout; activity retried; handler idempotent |
| DB slow | Scheduling delayed; queue buffers; alerts on due-job lag |

### 7. Probes
- *Cron on one server is simplest; why not?* Single point of failure, no retries, no visibility, no scale; cron is fine for a single box.
- *How do you guarantee a job runs even if the scheduler crashes just after claiming?* Lease expires → reclaim → at-least-once; handler idempotency makes it effectively once.
- *How does Temporal recover workflow state?* Event history replay with deterministic code; not by serializing memory.

---

## Case 15: Event Pipeline / Log Analytics

### 1. Requirements
- Collect events from web/mobile/backend (clicks, purchases, logs); near real-time dashboards (< 1 min freshness); historical analytics over years; ad-hoc SQL.
- Exactly-once-effect in aggregates; replay for backfills and bug fixes; schema evolution.
- Scale: 1B events/day average (≈ 11.6K/s), peak 50K/s.

### 2. Estimates
- Event size ~500 B → 500 GB/day raw; compressed Parquet ~10x smaller ≈ 50 GB/day ≈ 18 TB/year.
- Kafka: 50K/s × 500 B = 25 MB/s peak ingress; 3 replicas, 7-day retention ≈ 25 MB/s × 86,400 × 7 × 3 ≈ 45 TB stored — plan disk accordingly (or compress and retain 3 days in Kafka, rely on lake for longer).
- Partitions: 25 MB/s ÷ ~10 MB/s per partition target ≈ 3, round up to 30–60 for consumer parallelism headroom.

### 3. Architecture
```
SDK/agents → Collector edge (batching, compression, auth, schema validation)
  → Kafka topics per domain (partition key = user_id or entity_id), schema registry (Avro/Protobuf)
  → Stream processing (Flink / Kafka Streams):
       - enrich (lookups), sessionize (windows keyed by user, 30 min gap), dedupe by event_id (state with TTL)
       - aggregate (1-min tumbling counts) → OLAP store (Druid/ClickHouse) for dashboards
  → Lake sink: Kafka → Iceberg (hourly commits, exactly-once via transactional/idempotent writer)
  → Batch jobs (Spark/Trino): daily aggregates, backfills, compaction
  → Query: dashboards on OLAP; ad hoc on Trino over Iceberg
```

### 4. Key decisions
**Partition key:** user_id gives per-user ordering for sessionization and even spread; a hot bot account can skew — detect and split.
**Event time vs processing time:** dashboards by event time with watermark (allowed lateness 5 min for mobile); late events beyond lateness go to a correction stream and the lake (batch fix-up job).
**Dedupe:** event_id from client (UUID) + server-side dedupe state with TTL = 2× max retry window (e.g., 24 h) in the stream job; or idempotent upserts in OLAP keyed by event_id.
**Schema evolution:** registry with BACKWARD compatibility; add fields with defaults; never change type; breaking change = new topic version.

### 5. Iceberg sink details
- Writers buffer records per partition; commit a new snapshot each hour (or on size threshold) → queries see whole commits only.
- Exactly-once: store Kafka offsets in the snapshot summary (or a transactional sink) so a restart does not double-append.
- Small files: hourly commits produce many small files → compaction job merges into 256–512 MB files; sort within files by common filter columns (user_id, event_type) for pruning.
- Partitioning: `days(event_time), hour` maybe plus `event_type` bucket; avoid high-cardinality partition keys.

### 6. Deep dives
**a) Backpressure.** If the OLAP store slows, Kafka consumer lag grows (buffered, not dropped). Alert on lag growth rate; scale consumers up to partition count.
**b) Replay.** Kafka retention covers recent replays; for older, replay from the Iceberg lake through the same job code (Kappa-style reprocessing with versioned jobs writing to new tables, then swap).
**c) Hot partitions.** Skew from a single high-volume source: split by (entity, random salt) for raw writes; aggregate in second stage.
**d) Cost.** Tiered retention: Kafka 3 days, lake forever; compress with zstd; drop debug-level events at collector for production.
**e) Privacy.** PII removed or tokenized at the collector before Kafka; retention and deletion policies per topic; GDPR deletes via lake rewrite job.

### 7. Failure table
| Failure | Behavior |
|---|---|
| Stream job crash | Restart from checkpoint; state restored; reprocess since checkpoint |
| Bad deploy writes wrong aggregates | Replay from Kafka retention; reprocess into new table; swap |
| Lake commit failure | Retry; snapshot not visible until commit succeeds |
| Schema-incompatible producer | Registry rejects; producer alerts; no corrupt data |

### 8. Your project mapping
Your Kafka → Druid ingestion and Kafka → Iceberg migration map directly here. Be ready to explain: partitioning, supervisor load, compaction, why the lake is the source of truth for history, and the freshness vs cost trade-off that made you split the paths.

### 9. Probes
- *Exactly-once end to end?* Achieved by idempotent sinks and transactional commits within the pipeline; external side effects still need idempotency keys.
- *Why not write directly from producers to the warehouse?* Coupling, no buffering, no replay, and load spikes hit the warehouse directly.
- *Late data after dashboards are published?* Allowed lateness for near-real-time, and correction jobs for finalized daily numbers; label as "preliminary" until finalized.
