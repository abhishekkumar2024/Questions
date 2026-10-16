# 06. Deep Messaging, Streaming, and Caching

---

## 6.1 Messaging Models

| Model | Example | Semantics |
|---|---|---|
| Point-to-point queue | SQS, RabbitMQ queue | Each message consumed by one worker; removed on ack |
| Pub/sub (fan-out) | SNS, Redis pub/sub, RabbitMQ fanout exchange | Each subscriber gets a copy |
| Log (replayable stream) | Kafka, Kinesis, Pulsar | Messages retained by time/size; consumers keep offsets; replay possible |

**Queue vs log, the key distinction:** queues delete on ack and don't support replay or multiple independent readers well; logs retain data and let each consumer group move at its own pace.

### 6.1.1 Choosing
- Task queue with retries and DLQ, no replay needed → SQS or RabbitMQ.
- Event stream, many consumers, replay, high throughput, ordering per key → Kafka.
- Simple in-process or low-stakes notifications → Redis pub/sub (no persistence: a disconnected subscriber misses messages).
- Serverless and managed, moderate throughput → SQS/SNS, EventBridge, Pub/Sub (GCP).

---

## 6.2 Kafka in Depth

### 6.2.1 Structure
- **Topic** → split into **partitions**. Each partition is an ordered, append-only log of records with offsets.
- **Segments:** a partition is stored as segment files (log + index). Old segments deleted by retention (time/size) or compacted.
- **Broker:** a server hosting partition replicas.
- **Replication factor (RF):** typically 3. One replica is the **leader** for a partition; others are followers.
- **ISR (In-Sync Replicas):** the set of replicas caught up with the leader.
- **Controller:** elected broker managing partition leadership. Modern Kafka uses **KRaft** (Raft-based metadata quorum). ZooKeeper support was removed in Kafka 4.0.

### 6.2.2 Producer guarantees
- `acks=0`: fire and forget.
- `acks=1`: leader wrote it. Lost if leader dies before replicating.
- `acks=all` (`-1`): all ISR have it. Combined with `min.insync.replicas=2` and RF=3, tolerates one broker failure without loss.
- **Idempotent producer** (`enable.idempotence=true`): producer ID + per-partition sequence numbers; broker deduplicates retries. No duplicates from producer retries within a session.
- **Transactions:** atomic writes across multiple partitions/topics plus consumer offset commits (`read_committed` consumers see only committed data). Gives exactly-once for Kafka-to-Kafka pipelines (e.g., Kafka Streams).
- Batching: `linger.ms` and `batch.size` trade latency for throughput. Compression (lz4, zstd) reduces network and disk.

### 6.2.3 Consumer model
- **Consumer group:** each partition is assigned to exactly one consumer in a group. Max useful consumers = partitions count.
- **Offsets:** committed per group per partition, stored in `__consumer_offsets`.
- **Commit strategy:**
  - Auto-commit: may commit before processing finishes → message loss on crash.
  - Manual commit after processing → at-least-once (duplicates on crash).
- **Rebalancing:** when consumers join/leave, partitions are reassigned. Cooperative/incremental rebalancing reduces stop-the-world pauses.
- **Consumer lag:** end offset minus committed offset. Alert on growth rate, not just absolute value.
- **Poison message:** a message that always fails. Retry with backoff, then dead letter topic. Don't block the partition forever.

### 6.2.4 Ordering
- Ordering is guaranteed **within a partition** only. Choose the key so events needing order share a key (e.g., `order_id`).
- Changing partition count reshuffles key→partition mapping, breaking ordering for in-flight keys. Plan partition count up front.

### 6.2.5 Retention and compaction
- **Time/size retention:** delete old segments.
- **Log compaction:** keep the latest value per key (changelog/state topics, CDC snapshots).
- Tombstone (null value) deletes a key during compaction.

### 6.2.6 Performance facts
- Sequential disk writes + page cache + zero-copy (`sendfile`) make Kafka fast.
- Throughput bound by partitions, network, and batching. Hot partitions limit throughput: skewed keys are the common cause.

### 6.2.7 Kafka failure scenarios
- **Leader broker dies:** a new leader is elected from ISR. With `acks=all` and `min.insync.replicas`, no acked data lost.
- **Unclean leader election** (allowing non-ISR leader) risks data loss; disable it for correctness-critical topics.
- **Consumer crash:** partitions reassigned; messages after last commit reprocessed (idempotency needed).
- **Producer crash mid-batch:** idempotent producer prevents duplicate append on retry within session.

---

## 6.3 Other Brokers (know the differences)

- **RabbitMQ:** exchanges (direct, topic, fanout, headers) route to queues; acknowledgments; per-message TTL; priorities; good for complex routing and task queues; not built for massive replay.
- **Amazon SQS:** managed queue; standard (at-least-once, best-effort ordering) vs FIFO (exactly-once processing within dedup window, ordered per group); visibility timeout; DLQ.
- **Amazon Kinesis / Google Pub/Sub / Pulsar:** streaming with partitions/shards; Pulsar separates compute and storage (BookKeeper).
- **Redis Streams:** log-like structure with consumer groups and pending entries (ack). Good for moderate scale inside an app ecosystem.

---

## 6.4 Stream Processing

### 6.4.1 Concepts
- **Event time vs processing time:** when it happened vs when it was processed. Always use event time for correct analytics.
- **Windows:**
  - Tumbling: fixed, non-overlapping (every 1 min).
  - Sliding (hopping): fixed size, overlapping (5 min window every 1 min).
  - Session: gap-based (close after 30 min inactivity).
- **Watermark:** a heuristic "no events older than T are expected." Triggers window results.
- **Late data:** events after the watermark. Options: drop, allowed lateness window with updates, or side output for corrections.
- **Stateful processing:** keep per-key state (counts, joins) in local state store (RocksDB) backed by changelog topic.
- **Checkpointing:** periodically snapshot state + source offsets (Flink Chandy–Lamport-style barriers). On failure, restore and replay from offsets → exactly-once state effects.
- **Exactly-once sinks:** transactional or idempotent writes (upsert by key or event ID).

### 6.4.2 Joins in streams
- Stream–stream join: both inputs buffered within a time window.
- Stream–table join: enrich events with latest state from a changelog-backed table.
- Repartitioning is needed so both sides share the join key on the same partitions (co-partitioning).

### 6.4.3 Frameworks
- Kafka Streams (library, no separate cluster), Apache Flink (cluster, rich event-time semantics), Spark Structured Streaming (micro-batch), Beam (portable model).

### 6.4.4 Lambda vs Kappa architecture
- **Lambda:** batch layer (complete, slow, recomputes truth) + speed layer (fresh, approximate) + serving layer merging both. Complex: two code paths.
- **Kappa:** everything is a stream; reprocess by replaying the log with a new job version. Simpler, needs a replayable log with enough retention.

---

## 6.5 Data Lake Table Formats (Iceberg, Delta, Hudi)

- Files on object storage (Parquet) plus **metadata** tracking which files belong to the table snapshot.
- **Iceberg:** snapshot isolation for tables, hidden partitioning, partition evolution, time travel, schema evolution, metadata tree (manifest lists → manifests → data files).
- **Benefits over raw Parquet folders:** atomic commits, no partial reads, safe concurrent writers, fast planning via manifests, file-level pruning by column stats.
- **Small file problem:** many small files from streaming writes slow queries. Fix with compaction (rewrite into larger files) and sorting.
- **Maintenance:** expire snapshots, delete orphan files, rewrite manifests.
- **Query engines:** Spark, Trino/Presto, Flink, DuckDB, Druid ingestion (Iceberg input).

---

## 6.6 Caching in Depth

### 6.6.1 Where caches live
- Browser cache, CDN edge, reverse proxy cache, application in-process cache (Caffeine/Guava, LRU maps), distributed cache (Redis/Memcached), database buffer pool (internal).
- Principle: cache closest to the consumer that can tolerate its staleness.

### 6.6.2 Redis internals (what to say)
- Single-threaded command execution (I/O threads in newer versions), so commands are atomic. Pipelining and Lua scripts give multi-command atomicity.
- Data types: strings, hashes, lists, sets, sorted sets (score-based ranking, presence, leaderboards), streams, HyperLogLog (approximate cardinality), bitmaps.
- **Persistence:** RDB snapshots (point-in-time, fork-based), AOF (append log, fsync policies).
- **Replication:** async primary → replicas. Failover via Sentinel (managed failover for non-cluster) or Redis Cluster (built-in).
- **Redis Cluster:** 16,384 hash slots, distributed across primaries; key hashing with `{hashtag}` for co-location. Multi-key operations need same slot.
- **Memory:** `maxmemory` with eviction policies: `allkeys-lru`, `volatile-lru`, `allkeys-lfu`, `volatile-ttl`, `noeviction`.
- Memcached: multi-threaded, simpler, no persistence or rich types.

### 6.6.3 Eviction policies
- **LRU:** evict least recently used. Good general default.
- **LFU:** evict least frequently used. Better for skewed popularity, handles scan pollution.
- **TTL/random:** simple, cheap.
- **Admission policies (TinyLFU, used by Caffeine):** only admit new items likely to be reused, protects hot set.

### 6.6.4 Consistency patterns
- **Cache-aside (lazy loading):** read: check cache → miss → DB → populate. Write: update DB → **delete** cache key (preferred over set, avoids race with concurrent readers repopulating stale value).
- **Race in cache-aside:** reader reads old DB value, writer updates DB and deletes cache, reader then writes old value into cache → stale until TTL. Mitigate with short TTL, versioned values, or CDC-driven invalidation.
- **Write-through:** write to cache and DB in one path; still need transaction semantics for failure.
- **Write-behind:** async flush from cache to DB; risk of data loss on cache crash; batching gains.
- **Refresh-ahead:** refresh before expiry for predictable hot keys.
- **CDC-based invalidation (Debezium):** DB change log drives cache deletes; robust against app bugs.

### 6.6.5 Stampede and hot-key protection
- **Single-flight / request coalescing:** one request recomputes, others wait for the result.
- **Probabilistic early expiration:** recompute slightly before TTL, randomly by request, so one hot expiry doesn't cascade.
- **Stale-while-revalidate:** serve stale value while refreshing in background.
- **Jitter TTL:** `ttl = base + random(0, base*0.1)` to avoid synchronized expiry.
- **Hot key replicas:** `key#0..key#N` with random selection on read.
- **Local (L1) cache** per app server with short TTL absorbs hot-key reads.

### 6.6.6 Cache penetration and avalanche
- **Penetration:** queries for non-existent keys always miss and hit DB. Fix: cache negative results briefly; Bloom filter of existing keys.
- **Avalanche:** many keys expire together or cache node dies → DB overload. Fix: TTL jitter, multi-node replication, circuit breakers, load shedding on DB.

### 6.6.7 Cache sizing
- Hit ratio drives value: `avg latency = h * t_cache + (1-h) * t_db`. Model hit ratio from Zipf-like access distribution (small set gets most traffic).
- Working set: keys accessed within a window; size cache to hold it.

---

## 6.7 CDN Deep Dive
- Anycast routing directs clients to nearest POP.
- Edge caching with `Cache-Control` (`public, max-age=…, s-maxage=…`, `stale-if-error`, `stale-while-revalidate`).
- Vary headers and cache keys (query string, cookies) can destroy hit rates if too broad.
- Signed URLs/cookies for private content; token auth at edge.
- Video: segmented streaming (HLS/DASH) cached per segment; prefetch next segments.

---

## 6.8 Interview Questions (with short model answers)

1. **Kafka lost messages after broker crash — why?** Likely `acks=1` or unclean leader election, or RF=1. Fix: `acks=all`, `min.insync.replicas=2`, RF=3, disable unclean election.
2. **How do you get exactly-once in a pipeline?** Idempotent producer + transactions for Kafka-to-Kafka; idempotent upserts or dedup keys for external sinks.
3. **Consumer lag keeps growing. What now?** Check consumer throughput per partition, hot partition skew, downstream DB latency; scale consumers to partition count; batch writes; if skew, rekey.
4. **How do you order events per user?** Key by user ID; single partition per key; consumer processes partitions sequentially.
5. **Cache and DB disagree. Why?** Race in cache-aside or failed invalidation; use delete-after-write, TTL bound, CDC invalidation, and reconciliation jobs.
6. **Redis is the bottleneck. What do you do?** Check hot keys (`--hotkeys`, `MONITOR` sampling), add read replicas, shard via Cluster, use local L1 cache, reduce payload size, pipeline commands.
7. **Why do you need watermarks?** To know when a window can close despite out-of-order events; trade-off between completeness and latency.
