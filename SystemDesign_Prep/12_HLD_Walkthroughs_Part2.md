# 12. Full HLD Walkthroughs, Part 2 (Cases 6–10)

Same structure as Part 1: requirements → estimates → API → data model → architecture → flows → deep dives → failures → scale-up → probes.

---

## Case 6: News Feed / Timeline (Twitter, Instagram)

### 1. Requirements
- Post text/media; follow/unfollow; home timeline (ranked or chronological); profile timeline.
- Timeline loads in < 300 ms p99; new posts appear within ~5 s for followers.
- Users follow up to 5,000 accounts; some accounts have 100M+ followers.
- Eventual consistency acceptable for timeline; posts must never be lost.

### 2. Estimates
- 300M DAU; each opens the app 5× a day, loads 1 page (20 posts) each time → 1.5B timeline reads/day ≈ **17K reads/s avg**, peak ~50K/s.
- Posts: 100M posts/day ≈ **1,160 writes/s**, peak ~4K/s.
- Fan-out: an average post goes to ~200 followers → 100M × 200 = **20B timeline inserts/day ≈ 230K inserts/s**. This is the real load driver.
- Timeline cache: 300M users × 800 post IDs × 8 B ≈ 1.9 TB. Store IDs, not posts. Reasonable for a few hundred Redis nodes.

### 3. API
```
POST /v1/posts              {text, media_ids[]}           -> 201 {post_id}
GET  /v1/timeline/home?cursor=  -> {posts[], next_cursor}
GET  /v1/users/{id}/posts?cursor=
POST /v1/follows/{user_id}  / DELETE
```
Cursor = last post's (timestamp, id) — pagination stable under new inserts.

### 4. Data model
- **posts** sharded by `author_id`: `post_id (Snowflake, time-sortable), author_id, text, created_at, media`.
- **follows** two adjacency tables: `following(user_id, followee_id)` and `followers(followee_id, user_id)`, both sharded by their first key. Writes update both (saga or outbox).
- **timeline cache**: Redis sorted set per user `timeline:{user}` → score = post_id (time-sortable), trimmed to 800.

### 5. Core decision: fan-out on write vs on read
| | Fan-out on write (push) | Fan-out on read (pull) |
|---|---|---|
| Write cost | O(followers) per post | O(1) |
| Read cost | O(1) read cache | O(following) merge at read |
| Breaks when | Celebrity with 100M followers | User follows 5,000 accounts |
| Latency | Fast reads | Slow reads |

**Hybrid (chosen):** push for normal authors (< ~10K followers); for celebrities, skip fan-out. At read time, merge the precomputed timeline with posts fetched from the few celebrities the user follows (a small pull set). Threshold is a tunable.

### 6. Architecture
```
Post Service ─► posts DB (sharded by author) ─► Kafka "post.created" (key=author)
                                             └─► Fan-out workers
                                                   ├─ author follower count < T: push post_id to each follower's timeline ZSET
                                                   └─ celebrity: skip (pulled at read)
Timeline Service (read): ZREVRANGE timeline:{user} → merge celebrity pulls → hydrate post bodies (multi-get, cache)
Post cache: Redis/memcached of post objects (hot posts)
Follow Service: follows DBs + outbox → Kafka → updates timeline (backfill last N posts of new followee)
Ranking Service (optional): scores candidates; merges with ML features
```

### 7. Flows
- **Post:** write post row → outbox event → fan-out workers push IDs into follower ZSETs in batches (pipeline, 1,000 ops per round trip).
- **Read:** get ~200 candidate IDs from ZSET → merge celebrity posts by time → fetch post bodies via multi-get from post cache (fall back to DB) → apply ranking → return page with cursor.
- **Follow:** write follow rows → backfill: copy the new followee's last 50 post IDs into the follower's timeline ZSET (async).

### 8. Deep dives
**a) Celebrity problem.** A post by a 100M-follower account would cause 100M ZSET writes. The hybrid skips it. Cost moves to read time for followers of celebrities; cap the pull set at ~20 celebrities per user.
**b) Fan-out at 230K writes/s.** Partition workers by follower shard, batch writes per Redis node, use pipelines. Lag target: p99 < 5 s. Monitor fan-out lag as an SLO.
**c) Inactive users.** Don't maintain timelines for users inactive > 30 days; build on demand when they return (pull-based rebuild). This cuts memory and write cost a lot.
**d) Cache eviction and rebuild.** If a timeline is evicted, rebuild from follows + recent posts (bounded work). Protect against stampede on a popular cold start with single-flight.
**e) Counts (likes, views).** Not synchronous. Increment via batched counters (aggregate per post per second, flush), approximate for display, exact in an offline job.
**f) Deletion and privacy.** Deleting a post must remove it from timelines: store deleted flag on post; timelines filter at hydration (lazy), and a background job prunes ZSET entries.

### 9. Failure table
| Failure | Behavior |
|---|---|
| Fan-out worker down | Kafka replays from offset; idempotent ZADD makes replays safe |
| Timeline cache down | Rebuild on read (slower, DB load); rate limit rebuilds |
| Post DB shard down | Posts for that author slow/unavailable; timelines still serve cached IDs |

### 10. Scale-up
More fan-out partitions; regional timelines; split timeline cache by user shard; precomputed ranking features; move celebrity list to config so thresholds adapt.

### 11. Probes
- *Why store IDs in the timeline not the post?* Post bodies are shared by many timelines; storing IDs avoids 200× duplication and makes edits/deletes single-write.
- *Ordering across fan-out workers?* Timelines sort by time-sortable post_id, so arrival order does not matter.
- *Why not one global feed table?* It would be a hot table; per-user materialization spreads load across shards.

---

## Case 7: Typeahead / Autocomplete

### 1. Requirements
- Given a prefix, return top 5–10 suggestions ranked by popularity (and optionally personalization).
- Latency p99 < 100 ms, including network; suggestions update within hours (daily freshness acceptable).
- Filter unsafe terms; handle non-English tokens; support up to 15 characters.

### 2. Estimates
- 1B searches/day ≈ 12K/s average, peak ~40K/s. Each search fires ~5 keystroke requests after client debounce → **~60K prefix requests/s peak**. Client debounce (150–200 ms) cuts this sharply; state that.
- Phrase vocabulary: 500M unique queries; average 25 chars → 12.5 GB raw; trie with compression ~ 10–30 GB. Fits in memory across a few dozen nodes.

### 3. API
`GET /v1/suggest?q=<prefix>&lang=en&limit=8` → `{suggestions: [{text, score}]}`. Cache-control with short max-age at the CDN for popular prefixes.

### 4. Data pipeline (offline)
```
Query logs → Kafka → Spark/Flink job (hourly/daily):
  normalize (case, whitespace, unicode), count per phrase, filter (blocklist, PII, low counts)
  → build top-K per prefix (K=10) for every prefix up to length 15
  → write to serving store (versioned snapshot)
```
Top-K per prefix stored at each prefix precomputed: lookup is O(prefix length) and no trie traversal at query time.

### 5. Serving architecture
```
Client (debounce, local cache of recent prefixes)
  → CDN edge (cache popular prefixes, 5–60 min)
  → Suggest service (stateless) → in-memory trie/hash map shards (partition by first 1–2 characters)
  → Snapshot swap: build new version, load in background, atomic pointer switch
```
Sharding by prefix (first 2 chars) spreads hot prefixes like "a" unevenly; mitigate by splitting heavy shards further or using a hash of the prefix.

### 6. Deep dives
**a) Data structure choice.** Trie: O(L) lookup, memory heavy. Radix tree: compressed, far less memory. Precomputed top-K per node: avoids DFS at query time. Chosen: radix tree with top-K stored per node.
**b) Freshness vs cost.** Full rebuild daily; trending terms via a streaming counter layer merged at serve time for last-hour spikes (e.g., breaking news). Merge: score = blend of historical and recent counts.
**c) Personalization.** Re-rank top-50 candidates with user history (small per-user profile in cache). Keep the base list shared.
**d) Safety.** Blocklist applied in the offline pipeline (and a runtime filter for emergent terms). Never rely on runtime only.
**e) Storage swap.** Versioned snapshots; keep last two versions for instant rollback if a bad build ships.

### 7. Failure table
| Failure | Behavior |
|---|---|
| Suggest node down | Replica in same shard; CDN serves popular prefixes |
| Bad batch build | Rollback to previous snapshot pointer |
| Pipeline late | Serve yesterday's snapshot; freshness SLO alarms |

### 8. Probes
- *Why not SQL LIKE 'prefix%'?* Index scan plus sort per keystroke; cannot rank cheaply at this scale.
- *Typos?* Fuzzy matching via edit distance (BK-tree or n-gram index) as fallback when exact prefix has no results.
- *Why debounce?* Reduces load ~5x and avoids flicker.

---

## Case 8: Ride Sharing (Uber / Lyft)

### 1. Requirements
- Rider requests a ride from pickup to dropoff; system matches nearest suitable driver; shows ETA and live driver location.
- Driver app sends location every 4 s; trip lifecycle with states; fare estimate and surge pricing.
- Match within 10 s p95; no driver assigned to two riders; location freshness < 10 s.
- Scale: 5M drivers online at peak (city-level hotspots), 20M rides/day.

### 2. Estimates
- Location updates: 5M drivers / 4 s = **1.25M writes/s** globally. Must be in-memory, not a database.
- Ride requests: 20M/day ≈ 230/s avg, peak ~2,000/s (concert ends, rain).
- Storage: trip records 20M × 2 KB ≈ 40 GB/day; history retention in object store/columnar.
- Per city hotspot: 100K drivers in one metro → 25K location writes/s in one region. Partition by geography.

### 3. API
```
POST /v1/rides            {pickup, dropoff, product_type}      -> {ride_id, fare_estimate, eta}
GET  /v1/rides/{id}       -> state, driver, live location (WebSocket/SSE subscription)
POST /v1/driver/location  {lat, lng, heading, speed, ts}       (UDP-ish fire-and-forget or batched HTTP)
POST /v1/driver/rides/{id}/accept | /arrive | /start | /complete
```

### 4. Geospatial index
- Partition the map into cells using **H3** (hexagons at resolution ~8 ≈ 0.7 km² each) or S2 cells.
- Driver location store: `cell_id → set(driver_id)`; `driver_id → (lat, lng, cell, ts)`. Both in memory, sharded by cell (region).
- Moving a driver across cells: remove from old cell set, add to new (atomic on the shard).
- Nearby query: fetch driver sets for the rider's cell and its k-ring neighbors (ring 1 = 7 cells, ring 2 = 19 cells), filter by distance and availability.
- Why not geohash only: boundary effects and uneven cell sizes by latitude; H3 gives uniform neighbor distances.

### 5. Architecture
```
Driver app ──location──► Location ingest (stateless; batches) ──► in-memory geo index (sharded by cell)
                                                             └──► Kafka (sampled, for analytics/ETA training)
Rider app ──request──► Trip service (state machine, DB) ──► Matching service
Matching: query nearby drivers → ETA service (routing engine on road graph) → rank → offer to best driver
Offer state: driver has 15 s to accept; on timeout, next candidate
Realtime: rider subscribes to trip channel; push driver location every 3–5 s (WebSocket gateway)
Pricing: surge computed per cell from demand (requests) and supply (available drivers) on stream aggregation
```

### 6. Flows
1. Rider requests → trip created in `REQUESTED`; fare estimated.
2. Matching finds candidates in rider's cell + ring 1; ETAs from routing engine (not straight-line).
3. Offer sent to top driver; `OFFERED` with expiry; driver accepts → conditional update `status='MATCHED' WHERE status='OFFERED' AND driver_id=?` ensures single owner.
4. Driver moves through `ARRIVED → STARTED → COMPLETED`; each transition persisted with timestamps.
5. Payment and receipt after completion (saga).

### 7. Deep dives
**a) Offer correctness (no double assignment).** One driver can receive offers for two riders simultaneously. Driver-side lock: a driver in `OFFERED` state is excluded from other matching for the offer window. The final acceptance is a conditional write on the trip row. Losers get a retry.
**b) Location write load.** 1.25M/s: keep latest position only in memory; persist a sample every 30 s to the analytics stream. Use batching (driver app sends 4 points per request every 16 s) and UDP-like best effort — losing one point is harmless.
**c) Staleness.** Each position has a timestamp; ignore positions older than 15 s for matching, and mark drivers offline if no update in 30 s.
**d) Hotspots (concert ends).** Demand spikes in one cell. Mitigations: surge pricing reduces demand; widen search radius for supply; pre-position drivers via incentives; shard by finer cell at hotspots (adaptive resolution).
**e) ETA accuracy.** Road-network routing with live traffic weights; cache routes between popular cell pairs; ML correction on historical trips.
**f) Failover.** Geo index shards are reconstructable from driver heartbeats within ~5 s, so memory-only is acceptable; trip state is in a durable DB.

### 8. Failure table
| Failure | Behavior |
|---|---|
| Geo shard down | Drivers in its cells invisible for seconds until heartbeats repopulate |
| Matching timeout | Next candidate; trip stays `REQUESTED` until limit, then cancel with retry prompt |
| Driver app offline mid-trip | Trip continues; sync when reconnected; alerts after threshold |

### 9. Probes
- *Why not a PostGIS query on every request?* Millions of updates/s with row locks; in-memory cell index is far cheaper.
- *How do you prevent a rider being matched twice?* Trip-level conditional state transition.
- *How do you compute ETA?* Routing service on road graph; precomputed contraction hierarchies for fast queries.

---

## Case 9: Payment System

### 1. Requirements
- Accept card payments through an external PSP; authorize, capture, refund, partial refund.
- No double charge under any retry; every money movement recorded in a double-entry ledger.
- Reconcile with PSP daily; PCI scope kept minimal (card data never touches our servers — tokenized by PSP).
- Availability 99.99% for checkout; correctness above availability for money.

### 2. Estimates
- 1M payments/day ≈ 12/s avg, peak ~150/s (flash sale). Low throughput; correctness dominates.
- Ledger rows: each payment ~6 entries → 6M rows/day ≈ 2.2B/year. Fine for a sharded relational DB.
- PSP latency 1–3 s per call; our timeouts must allow for it, with async completion via webhooks.

### 3. API
```
POST /v1/payments     Idempotency-Key: <uuid>   {amount_minor, currency, customer_token, order_id}
  -> 202 {payment_id, status: PROCESSING}   (or 201 final if sync)
GET  /v1/payments/{id}
POST /v1/payments/{id}/refunds   Idempotency-Key   {amount_minor}
POST /webhooks/psp              (signed; event_id dedupe)
```
Amounts in integer minor units (paise/cents). Never floats.

### 4. Data model
- **payments**: `id, order_id, amount, currency, status, version, psp_ref, created_at`. Status transitions guarded by `version`.
- **idempotency_keys**: `(merchant_id, key) PK, request_hash, response, status, created_at, ttl`.
- **ledger_entries**: append-only `(entry_id, txn_id, account, debit, credit, currency, ts)`; invariant: per transaction, debits = credits.
- **accounts**: merchant receivable, platform revenue, PSP clearing, customer funds.
- **psp_events**: raw webhook payloads with event_id unique.

### 5. State machine
```
CREATED → PROCESSING → AUTHORIZED → CAPTURED → (REFUNDED partially or fully)
                    └→ FAILED
AUTHORIZED → VOIDED (before capture)
```
Only the orchestrator moves states; each move is a conditional update on `version`, and emits an event via outbox.

### 6. Flows
1. Check idempotency key: insert `(key, PROCESSING)` in unique table. Existing key with same hash → return stored response; different hash → 422.
2. Create payment `CREATED`; outbox event.
3. Call PSP with PSP idempotency key = our payment ID (PSPs support this). Timeout → status stays `PROCESSING`; do not retry blindly without the same key.
4. PSP response (sync or webhook) → transition to `AUTHORIZED`/`FAILED`; write ledger entries in the same transaction as the state change.
5. Capture → ledger: move from PSP clearing to merchant receivable.
6. Daily reconciliation: compare our captured set with PSP settlement file; mismatches → queue for investigation.

### 7. Deep dives
**a) Exactly-once effect.** Retries at three layers (client, our API, our call to PSP) all carry the same key; uniqueness constraints enforce single effect. Webhook duplicates are dropped by event_id.
**b) Unknown outcome.** PSP timeout = unknown. Never mark failed. Resolve by querying PSP status with our payment ID (reconciliation/poller) before any retry or customer message.
**c) Ledger correctness.** Each transaction is one DB transaction with balanced entries; a periodic job asserts sum of debits = sum of credits globally and per currency.
**d) Concurrency on refunds.** Two concurrent refunds that together exceed the captured amount: lock the payment row (`SELECT … FOR UPDATE`) or conditional update on `refunded_total + amount <= captured`.
**e) Multi-currency.** Store original currency and exchange rate snapshot; never convert at display time for accounting.
**f) PCI scope.** Hosted fields or PSP SDK tokenize cards in the browser; our services only see tokens. Shrinks compliance surface dramatically.
**g) Availability.** Synchronous auth in checkout; if PSP is down, fail fast with clear message; failover to a secondary PSP only with the same idempotency key guarantees and reconciliation coverage.

### 8. Failure table
| Failure | Behavior |
|---|---|
| Client retries after timeout | Idempotency key returns the same payment result |
| Webhook arrives before sync response | State machine accepts out-of-order if version allows; otherwise buffer |
| DB failover during ledger write | Transaction atomic; either all entries or none; PSP reconciliation catches mismatches |

### 9. Probes
- *Why not Kafka for payments?* You can use it for events, but the source of truth is a transactional DB with constraints; Kafka alone cannot enforce "one captured payment per order."
- *Why not eventual consistency for balances?* Balances gate money movement; strong consistency on those rows is required.
- *Settlement mismatch of ₹1 found?* Investigate via PSP refs; never adjust ledger without an auditable correction entry.

---

## Case 10: Web Crawler

### 1. Requirements
- Crawl the web (say 5B pages) with a refresh policy: important/news pages daily, others monthly.
- Respect robots.txt and per-host politeness (e.g., max 1 request/s per host by default).
- Deduplicate content; avoid crawler traps; store raw HTML and extracted links.
- Throughput: 500M pages/day ≈ **5,800 pages/s** average (state assumption; the number is the scale of the design).

### 2. Estimates
- 5B pages × 100 KB average compressed raw ≈ **500 TB** per full snapshot; keep last 2–3 snapshots and deltas.
- Fetchers: each takes ~1 s per page including network/parsing → 5,800 concurrent fetches in flight → ~600 fetcher workers with 10 connections each.
- URL frontier: ~50B URLs known; need disk-backed storage, not memory alone.

### 3. Architecture
```
Seeds ─► URL Frontier
           ├─ Priority queues (importance score, freshness due time)
           └─ Per-host queues (politeness: next allowed fetch time per host)
        ─► Fetcher pool (DNS cache, robots.txt cache, HTTP keep-alive, timeouts, max size)
        ─► Parser (extract text, links, canonical URL, meta)
        ─► Dedupe: content fingerprint (SimHash) lookup
        ─► Link extractor → normalize URL → seen-filter (Bloom filter + persistent store) → back to frontier
        ─► Storage: raw object store (WARC files), index pipeline (Kafka → indexer)
```

### 4. Frontier design
- **Politeness:** each host has a queue and a `next_allowed_at`. Scheduler pops hosts whose time has arrived; a host with 10M URLs cannot starve others.
- **Priority:** importance (PageRank-like score) + freshness deadline. Combine into a scheduling key.
- **Storage:** frontier backed by sharded disk stores (e.g., RocksDB per shard) keyed by host; in-memory only for active hosts.

### 5. Deep dives
**a) Duplicate URLs.** Normalize: lowercase host, remove default ports, sort query params, remove tracking params (utm_*), strip fragments, resolve relative paths. Seen-check: Bloom filter for fast negative answers (no false negatives) with persistent backing for exactness. Sizing: 50B URLs at 1% false positive ≈ 10 bits/URL ≈ 62 GB — shard it.
**b) Duplicate content.** Exact hash catches byte-identical; SimHash (64-bit) catches near duplicates: Hamming distance ≤ 3 means near-duplicate. Index SimHash blocks for lookup.
**c) Traps.** Infinite calendars, session IDs in URLs, generated pages. Limits: max depth per host, max URLs per host per day, max URL length, pattern detection (same template repeated thousands of times).
**d) robots.txt and legal.** Cache robots per host (TTL ~24 h); respect Crawl-delay; honor meta robots and noindex; keep opt-out handling.
**e) DNS.** Local DNS cache with TTL; the resolver is often the bottleneck at scale — run your own caching resolver.
**f) Fault tolerance.** Checkpoint frontier state; fetch jobs are idempotent (re-fetch is harmless); a crashed fetcher's leases expire and jobs return to the queue.
**g) Freshness.** Estimate change rate per page from history; schedule re-crawl by expected change interval (adaptive), not fixed period.

### 6. Failure table
| Failure | Behavior |
|---|---|
| Host returns 5xx storms | Exponential backoff per host; circuit per host |
| Fetcher crash | Lease expires; another fetcher takes the URL |
| Frontier shard lost | Rebuild from replicas or re-seed from link graph |

### 7. Probes
- *Why Bloom filter instead of a hash set?* Memory: a hash set of 50B URLs is terabytes; a Bloom filter is an order of magnitude smaller, with an acceptable false-positive rate (a few URLs skipped).
- *How do you avoid hammering one site?* Per-host queues with time-gated scheduling.
- *Is the crawler legal everywhere?* Respect robots, terms, and local law; the design includes opt-outs and rate limits.
