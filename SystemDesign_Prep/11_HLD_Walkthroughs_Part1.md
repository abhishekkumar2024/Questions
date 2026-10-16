# 11. Full HLD Walkthroughs, Part 1 (Cases 1–5)

Each walkthrough follows the interview clock: requirements → estimates → API → data model → architecture → flows → deep dives → failures → scale-up → probes. Say the reasoning out loud; the numbers are there so you can reuse the method, not memorize the answers.

---

## Case 1: URL Shortener

### 1. Requirements (5 min)
- **Functional:** create a short code for a long URL; redirect a code to its URL; optional custom alias; optional expiry; click count.
- **Non-functional:** redirect latency p99 < 50 ms; 99.99% availability for redirects (reads matter more); codes must be unique; short codes are not guessable enough to enumerate private links (they are public by design, so state this).
- **Out of scope:** user accounts beyond API keys, link previews, malware scanning (mention as follow-up).

### 2. Estimates
- 100M new links/month → 100M / (30 × 86,400 ≈ 2.6M s) ≈ **40 writes/s**, peak ~120/s.
- Read:write 100:1 → **4,000 reads/s** average, peak ~12,000/s.
- Record ≈ 500 B (code 7 B, URL up to 2 KB, metadata). 100M × 12 × 500 B ≈ **600 GB/year**; with 5 years and replication ×3 ≈ 9 TB raw.
- Code space: base62, 7 characters → 62⁷ ≈ **3.5 × 10¹²** codes. At 1.2B/year, 7 chars last ~2,900 years. Length 7 is enough.
- Bandwidth: 12,000 redirects/s × ~300 B response ≈ 3.6 MB/s — trivial.

### 3. API
```
POST /v1/urls        {long_url, custom_alias?, expires_at?}  -> 201 {short_url, code}
GET  /{code}         -> 301 or 302 with Location header; 404 if unknown; 410 if expired
DELETE /v1/urls/{code} -> 204 (owner only)
GET  /v1/urls/{code}/stats -> {clicks, last_click}
```
**301 vs 302:** 301 lets browsers and CDNs cache the redirect permanently, so later clicks never reach you (no analytics, and you cannot change the target). 302 (or 307) keeps every click flowing through you. Pick 302 if analytics or edits matter; say so.

### 4. Data model
- **links** (KV store, partitioned by `code`): `code PK, long_url, owner_id, created_at, expires_at, is_custom`.
- **clicks**: append-only events (`code, ts, country, referrer`) → Kafka → aggregated counters.
- Why KV: access is always by code; no joins; scales by hash-partitioning on `code`.
- Why not hash the URL: MD5 of the URL then truncate gives collisions and the same URL always gets the same code (privacy leak). Use a counter-based ID instead.

### 5. Code generation
Option A (chosen): **range allocation.** A ticket service hands each app server a block of 10,000 IDs (`UPDATE counter SET next = next + 10000 RETURNING next`). Server uses IDs locally, base62-encodes them. No coordination per request; a crash wastes at most one block.
Option B: Snowflake (64-bit timestamp + machine + sequence) then base62. Longer codes (~11 chars) unless truncated.
Custom alias: insert with unique constraint on `code`; conflict → 409.

### 6. Architecture
```
Client ──► CDN/edge (optional, caches 302 only if you accept no analytics)
        ──► L7 LB ──► Redirect service (stateless, read-heavy)
                         ├─► Redis cache (code → long_url, TTL 24h, cache-aside)
                         └─► KV store (sharded by code, 3 replicas)
        ──► Write service ──► ID block allocator ──► KV store
                          └─► Kafka "clicks" ──► aggregator ──► analytics store
```
Split read and write services so the 100:1 read path scales independently.

### 7. Flows
- **Redirect:** LB → redirect svc → Redis GET → hit: return 302 (≈1 ms). Miss: KV read → SET in Redis → 302. Expired/unknown: 404/410. Click event sent async to Kafka (fire and forget with bounded buffer; losing a few clicks is acceptable, say so).
- **Create:** validate URL (scheme allowlist, length, resolvability optional) → get next ID from local block → base62 → insert (conditional on not exists for aliases) → return.

### 8. Deep dives
**a) Cache sizing.** Power-law popularity: top 20% of links drive ~80% of clicks. Working set ≈ 20% of links accessed daily. If 1 day of new links is 3.3M × 500 B ≈ 1.6 GB, a 16 GB Redis cluster holds the hot set comfortably. Expect hit ratio > 95%.
**b) Hot link (viral).** One code gets 50,000 req/s. Redis single key can handle this on one shard (~100K ops/s), but add an in-process L1 cache with a 5 s TTL on redirect servers so the hot key never hits Redis in aggregate.
**c) Expiry.** Do not run a sweeper that scans the whole table. Store `expires_at`; check at read time (lazy deletion); a background job deletes in batches using an index on `expires_at`. Cache TTL = min(24 h, time to expiry).
**d) Analytics without slowing redirects.** Click increments are async. Aggregate in Kafka consumers into 1-minute buckets, write to a time-series or OLAP store. Redirects never wait on analytics.

### 9. Failure scenarios
| Failure | Behavior |
|---|---|
| Redis down | Fall through to KV; latency rises, availability holds; protect KV with a bulkhead/limit |
| One KV shard down | Replicas serve reads (quorum read); writes to that range fail until failover |
| ID allocator down | Servers keep using their current block; when exhausted, writes fail, reads unaffected |
| Kafka down | Click events buffered locally then dropped after limit; redirects unaffected |

### 10. Scale-up (10x traffic)
Add Redis shards and redirect replicas; split ID blocks larger; move click analytics to a sampled model if needed; regional deployment with each region serving reads from a local replica and writing to a home region.

### 11. Probes and answers
- *Two users create the same custom alias at once?* Unique constraint on `code`; one succeeds, one gets 409. No check-then-insert race.
- *How do you prevent abuse (spam links)?* Per-key rate limit on create, URL denylist, Safe Browsing check async, reporting endpoint.
- *What if the original URL changes content?* Out of our control; we store only the mapping, which is the point.

---

## Case 2: Distributed Rate Limiter

### 1. Requirements
- Limit per API key and per IP; configurable per route and per plan (e.g., 100 req/min free, 10k/min paid).
- Return 429 with `Retry-After`; overhead < 1 ms p99; works across 200 gateway instances; accuracy within ~5%.
- Fail-open if the limiter store is down (say this explicitly and justify).

### 2. Estimates
- 500K API keys active, peak 200K req/s total.
- Each request = 1 counter operation → 200K ops/s on the limiter store. A Redis cluster of 3–6 primaries handles this (each ~50–100K ops/s).
- Memory: 500K keys × ~100 B state ≈ 50 MB. Trivial.

### 3. Where to enforce
- Gateway (edge of your system): protects all services, one place to change policy. Chosen.
- Per-service middleware: needed for internal fairness; add later.
- Client SDK: advisory only; never the enforcement point.

### 4. Algorithm choice (go deeper than "token bucket")
- **Fixed window:** `INCR key:minute` with `EXPIRE`. Boundary burst: 100 requests at 0:59 and 100 at 1:00 = 200 in 2 seconds. Cheap but wrong at edges.
- **Sliding window log:** ZSET of timestamps; exact, but O(requests) memory per key. Too heavy at 100K req/min per key.
- **Sliding window counter:** `estimate = prev_count × (1 − elapsed_fraction) + curr_count`. Two counters per key. Error bounded and small (Cloudflare-style). **Chosen for paid plans with strict-ish limits.**
- **Token bucket:** tokens refill at `r` per second up to capacity `b`. Allows bursts up to `b`. **Chosen for smoothness and burst tolerance.** Store `(tokens, last_refill_ts)`.

### 5. Atomic implementation (Redis Lua)
```lua
-- KEYS[1]=bucket key; ARGV: capacity, refill_per_ms, now_ms, cost
local data = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(data[1]) or tonumber(ARGV[1])
local ts = tonumber(data[2]) or tonumber(ARGV[3])
local now = tonumber(ARGV[3])
tokens = math.min(tonumber(ARGV[1]), tokens + (now - ts) * tonumber(ARGV[2]))
local allowed = 0
if tokens >= tonumber(ARGV[4]) then
  tokens = tokens - tonumber(ARGV[4]); allowed = 1
end
redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], 120000)
return {allowed, math.floor(tokens)}
```
Why Lua: read-modify-write must be atomic; a Lua script runs without interleaving on one Redis shard. Clock: use Redis TIME inside the script rather than app clocks, so skew between gateways cannot grant extra tokens.

### 6. Architecture
```
Request → Gateway (local L1: small in-memory bucket, sync every 100 ms)
        → Redis Cluster (hash tag {api_key} keeps a key on one shard)
        → allow → upstream service
        → deny  → 429 + X-RateLimit-* headers
Config service: limits per plan/route, pushed to gateways, cached locally
```

### 7. Deep dives
**a) Hot key.** One API key at 50K req/s on a single Redis shard is a hotspot. Options: local token pre-allocation (gateway leases 100 tokens from Redis, serves locally, renews) — trades strict accuracy for throughput. State the accuracy bound: at most (instances × lease size) overshoot.
**b) Redis failure.** Circuit breaker around the limiter; on open, fall back to local per-instance limits (divided by instance count), not unlimited. Fail-open only for public read endpoints; fail-closed for expensive endpoints.
**c) Clock.** Use server time inside Lua (see above). Never use gateway wall clocks directly.
**d) Multi-region.** Per-region budget (global limit ÷ regions) with periodic rebalancing; accept small overage. Strict global limit requires cross-region coordination, which costs latency on every request.

### 8. Failure table
| Failure | Result |
|---|---|
| Redis shard down | Keys on that shard fall back to local limits |
| Gateway crash | Its local leases expire; no double counting |
| Config push delayed | Gateways use last known limits |

### 9. Probes
- *Why not a database counter?* ~10–50x lower throughput with row locks; contention on hot keys.
- *How do you return remaining quota?* Return tokens remaining from the Lua result in headers.
- *Who is limited behind NAT?* Per-IP limits punish shared IPs; prioritize API keys, use IP limits only for anonymous traffic.

---

## Case 3: Distributed Cache

### 1. Requirements
- get/set/delete with TTL; 1 TB working data across cluster; 500K ops/s; p99 < 2 ms in-region; survives a node failure with at most a brief unavailability of its keys.
- Consistency: clients accept eventual consistency across replicas; a delete must eventually remove the key everywhere.

### 2. Estimates
- 1 TB / (50 GB usable per node) → **20 primaries**, plus replicas → 40 nodes.
- 500K ops/s / 20 primaries ≈ 25K ops/s each — comfortable for an in-memory store.
- Network: 500K × avg 1 KB ≈ 500 MB/s cluster-wide → 10 GbE NICs per node are fine.

### 3. Partitioning
- Consistent hashing ring with **200 virtual nodes** per physical node → each node owns ~1/20 of keys with low variance (standard deviation of load shrinks as vnode count grows).
- Adding one node moves ~1/21 of keys (only keys whose ring position falls in new node's arcs). Removing one moves ~1/20 to neighbors.
- Hash function: MurmurHash3 or xxHash for speed and uniformity. Use ring lookup via sorted array + binary search: O(log(N×V)) = log(4,000) ≈ 12 comparisons.

### 4. Replication
- Each primary has 1–2 async replicas. Reads may go to replicas (eventually consistent) or primary (strong for that key).
- **Sync vs async:** async gives ~0.2–1 ms lower write latency and keeps the cache available; cost is losing the last few writes on failover. For a cache, losing a write means a miss, which is acceptable, so async is the right default. (Contrast with a primary database.)

### 5. Membership and failover
- Membership via gossip (or a coordinator like etcd storing the ring). A node missing heartbeats for T seconds is marked suspect, then dead after quorum agrees.
- Failover: promote the replica; clients refresh their ring from the coordinator. Stale client rings route to the old primary for a few seconds — acceptable for a cache; for writes that must not be lost, route through a proxy.

### 6. Eviction and memory
- Per-node `maxmemory`; LRU approximated by sampling 5–10 random keys and evicting the oldest (Redis-style). Cheap and close to true LRU.
- Track memory per key (overhead ~50–90 B per key in typical stores); small values waste memory — batch or use hashes.

### 7. Deep dives
**a) Hot key.** One key at 100K req/s overwhelms its primary. Fix: client-side local cache with 1–5 s TTL; or replicate hot key to k replicas and read round-robin.
**b) Node join with data movement.** Moving 1/21 of data while serving traffic: double-read during migration (read new owner, fall back to old), write to new owner, background copy, then drop old copy. Mention cost and that some misses are inevitable.
**c) Cache stampede on restart.** A restarted node is cold; all its keys miss to the DB. Mitigation: warm from replica snapshot, request coalescing, staggered rejoin, DB rate limit.
**d) Persistence.** Usually none (it's a cache). If needed: periodic snapshot (fork-based) plus append log; recovery time grows with data size; mention trade-off.

### 8. Failure table
| Failure | Impact |
|---|---|
| Primary down | Replica promoted; keys briefly missing → DB load spike |
| Network partition | Minority side unavailable for its keys; cache semantics tolerate this |
| Gossip storm after node flap | Dampening: require multiple missed heartbeats; back-off |

### 9. Your project mapping
You can present: consistent hashing with 200 vnodes and minimal movement on churn; master–slave replication; a load balancer routing by ring. Prepare to explain precisely how keys were handled on node removal (re-fetch vs migrate) and the failure window of your async replication. Precision is worth more than the headline number.

### 10. Probes
- *Why not Redis Cluster?* You can, and it's the production answer; building yours shows understanding. Compare: Redis Cluster uses 16,384 fixed slots and gossip; you used a ring with vnodes.
- *Strong consistency needed for a feature flag cache?* Serve from cache with version checks; the source of truth stays in the DB.

---

## Case 4: Notification System

### 1. Requirements
- Channels: push (APNs/FCM), SMS, email, in-app, webhook.
- Types: transactional (OTP, payment receipts: must arrive within seconds), marketing (bulk, can be delayed).
- Respect user preferences, quiet hours, and per-channel frequency caps.
- Deliver at least once with dedupe; track status (queued, sent, delivered, failed, opened).
- Scale: 10M/day average, bursts of 5M within 10 minutes (campaign).

### 2. Estimates
- 10M/day ÷ 86,400 ≈ **115/s average**. Campaign burst: 5M / 600 s ≈ **8,300/s**. Design for 10K/s with queues absorbing bursts.
- Storage: 10M × 1 KB log × 90 days retention ≈ 900 GB (compressed less).
- Provider limits matter more than your throughput: e.g., SMS providers throttle per account; you must pace outbound to them.

### 3. API
```
POST /v1/notifications  {user_id | recipients[], template_id, params, channels?, priority, idempotency_key}
  -> 202 {notification_id}
GET  /v1/notifications/{id} -> status per channel
POST /v1/providers/webhooks/{provider} -> delivery receipts
```
Always 202: the request is accepted, delivery is async.

### 4. Data model
- **notifications** (`id PK, user_id, template_id, params_hash, priority, created_at`), partition by user_id.
- **deliveries** (`notification_id, channel, status, attempts, provider_msg_id, updated_at`) — one row per channel.
- **preferences** (`user_id, channel, topic, enabled, quiet_hours`) — cached in Redis.
- **templates** — versioned; render at send time with version pinned.

### 5. Architecture
```
Producer → Notification API (validate, dedupe via idempotency_key in Redis SET NX with TTL)
  → Preference filter (cache) → Router (per channel)
  → Kafka topics: notif.transactional.{channel}  (high partitions, dedicated consumers)
                  notif.marketing.{channel}      (throttled consumers)
  → Channel workers (rate-limited per provider) → Provider (APNs, FCM, Twilio, SES)
  → Provider webhooks → status updater → deliveries table
  → DLQ after N attempts → alerting + replay tool
```
**Isolation:** separate topics and worker pools so a marketing blast never delays OTP.

### 6. Flows
1. API receives request; idempotency check; writes notification row in one transaction with outbox event.
2. Outbox relay publishes to Kafka.
3. Filter worker checks preferences, quiet hours (in user's timezone), caps (Redis counter per user per day).
4. Channel worker renders template, calls provider with its own idempotency key (if provider supports it), records `provider_msg_id`.
5. Provider receipt updates status; no receipt in T minutes → retry or mark unknown.

### 7. Deep dives
**a) Dedupe.** Producer retries cause duplicates: idempotency key in API. Provider retries cause duplicates: provider idempotency keys where supported; otherwise accept rare duplicates and keep them rare with timeouts that are long enough.
**b) Rate limiting outbound.** Token bucket per provider account (e.g., 500 SMS/s). Workers pause and leave messages in Kafka when out of tokens — backpressure instead of dropping.
**c) Fan-out to millions.** Do not expand recipients in the API request. Store a campaign with a segment query; a fan-out job enumerates user IDs in batches of 10K into Kafka. The API stays fast.
**d) Retries and provider outages.** Exponential backoff with jitter per channel; circuit breaker per provider; on sustained failure, route to fallback provider (e.g., second SMS vendor) if contracted.
**e) Time zones and quiet hours.** Store user timezone; schedule via delayed queue (Kafka with timestamp-based delay or a scheduler table) rather than sleeping workers.

### 8. Failure table
| Failure | Handling |
|---|---|
| Provider down | Circuit opens; queue accumulates; alert; fallback provider |
| Kafka lag on OTP topic | Dedicated consumers; lag alert pages on-call |
| Duplicate OTP sent | Acceptable for OTP if the code remains valid; prevent via dedupe window |
| Preferences service slow | Cache with TTL; default to *not* sending marketing when unknown |

### 9. Probes
- *Exactly-once notification?* Not achievable end to end; effectively-once for the user: dedupe + provider keys + idempotent user-visible effect.
- *How do you measure delivery?* Provider receipts, opens/clicks via tracking pixels/redirects, with the caveat that opens are unreliable (privacy features).

---

## Case 5: Chat Application (WhatsApp / Slack)

### 1. Requirements
- 1:1 and group chats (groups up to 1,000 members, large channels up to 100K).
- Message delivery: sent / delivered / read receipts; ordered within a conversation.
- Presence (online / last seen), typing indicators (ephemeral).
- Multi-device sync; history retention forever; offline delivery with push.
- Latency: online delivery p99 < 300 ms between two users in the same region.

### 2. Estimates
- 50M DAU × 40 messages/day = 2B messages/day ≈ **23K msgs/s average**, peak ~70K/s.
- Each message ~200 B + metadata ≈ 300 B → 600 GB/day raw → ~220 TB/year before replication and compression. Message history is the storage problem; keep recent hot data on SSD, older on cheaper tiers.
- Connections: 50M DAU, perhaps 20M concurrent sockets at peak. At ~100K connections per gateway box → 200 gateway servers.

### 3. API and protocol
- Clients open a **WebSocket** to a gateway (persistent, bidirectional).
- Frames: `SEND {client_msg_id, conv_id, body}`, `ACK {server_msg_id, seq}`, `DELIVER {conv_id, seq, body}`, `READ {conv_id, up_to_seq}`, `TYPING`, `PRESENCE`.
- REST for history pagination and media upload (pre-signed URLs).
- Long-lived connection: reconnect with exponential backoff; resume using last seen `seq` per conversation.

### 4. Data model
- **messages** partitioned by `conv_id`, clustered by `seq` (wide-column like Cassandra/HBase or sharded SQL). Row: `conv_id, seq, msg_id, sender_id, body, created_at`.
- **sequence numbers:** per conversation, monotonic. Assign with a per-conversation counter on the conversation's owner partition (single writer per conversation ⇒ no conflicts). Gaps detected client-side trigger a sync.
- **user_inbox / cursors:** `user_id, conv_id, last_delivered_seq, last_read_seq`.
- **connection registry:** Redis `user:{id}` → `{gateway_id, device_ids}` with TTL refreshed by heartbeat.

### 5. Architecture
```
Clients ⇄ WebSocket Gateways (stateful; ~100K conn each)
    │  registry: Redis user→gateway
    ▼
Chat Service (stateless) ── assigns seq via conversation owner (partition by conv_id)
    ├─► Message store (partitioned by conv_id)
    ├─► Kafka "messages" (partition by conv_id; ordering preserved)
    │       └─► Delivery router: look up recipient gateways → push via gateway RPC/pub-sub
    │       └─► Push service: APNs/FCM for offline devices
    └─► Presence service (Redis with TTL heartbeats)
Media service: upload to object store; thumbnails async
```

### 6. Send flow (step by step)
1. Client A sends `SEND` with `client_msg_id` (UUID generated on device) over its socket.
2. Gateway forwards to Chat Service (RPC).
3. Chat Service checks membership (cached), assigns `seq` from conversation counter, writes to message store. Idempotent on `(conv_id, client_msg_id)`.
4. Chat Service returns `ACK` to A with `seq` (A shows "sent").
5. Event published to Kafka keyed by `conv_id`.
6. Delivery router looks up each recipient's gateway in Redis; sends `DELIVER`. Recipient's client sends `ACK`; server marks "delivered" and notifies sender.
7. Offline recipient: no registry entry → push notification; on next connect, client syncs from its cursor.

### 7. Deep dives
**a) Ordering.** Single writer per conversation (partition by `conv_id`) gives total order per conversation without global locks. Cross-conversation order is not guaranteed and not needed.
**b) Group fan-out.** Group of 1,000: fan-out on write to 1,000 inboxes is fine. Channel of 100K members: don't write 100K copies; readers pull from the conversation log by cursor, and the router pushes only "new message available" to online members.
**c) Presence at scale.** Broadcasting every online/offline change to every contact is quadratic. Solutions: only subscribe clients to presence of people currently visible on screen; coarse "last seen" updated every few minutes; heartbeat TTL in Redis.
**d) Gateway failure.** Socket drops; clients reconnect to another gateway; registry updated; client resumes with last `seq`. Messages are persisted before ACK, so nothing is lost even if the push was in flight.
**e) Multi-device.** Each device has its own cursor and socket. Read state per device; the sender sees "read" when any recipient device reads (define the rule explicitly).
**f) End-to-end encryption.** Server stores ciphertext and per-device keys; group encryption uses sender keys. Mention that search and server-side features become harder.

### 8. Failure table
| Failure | Behavior |
|---|---|
| Gateway crash | Clients reconnect; resume by cursor; no loss |
| Chat service crash mid-send | Client retries with same `client_msg_id`; idempotent write |
| Kafka lag | Delivery delayed; persistence unaffected; users see "sent" not "delivered" |
| Redis registry lost | Fall back to "push to everyone" for a window, or query gateways; rebuild from heartbeats |

### 9. Probes
- *How do you show messages in order when two arrive simultaneously from different users?* Order by server-assigned `seq` in the conversation; clients never reorder by device time.
- *Why not poll?* Polling every second for 50M users is 50M req/s of mostly empty responses. WebSocket costs memory per connection but eliminates that load.
- *Where is the single point of failure for a conversation?* Its owner partition; mitigate with replicas and fast failover; short unavailability for sends, reads still served from replicas.
