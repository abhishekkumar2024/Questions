# 09. Deep Question Bank with Model Answers

Use: cover the answer, say yours out loud, then compare. Each answer names the trade-off, which is what interviewers score.

---

## A. Scalability and Capacity
1. **How do you scale a stateless API?** Horizontal replicas behind an LB, autoscaling on CPU/latency/queue depth, externalize session state (Redis or JWT), idempotent endpoints, graceful shutdown.
2. **How do you scale a stateful service?** Partition by key (consistent hashing or ranges), replicate each partition (leader + followers), route via a directory or ring, rebalance by moving partitions.
3. **Vertical vs horizontal?** Vertical is simple, has hardware ceilings and single-point risk. Horizontal needs partitioning and coordination. Start vertical for simple DBs, then replicas, then shard.
4. **How do you handle a 10x traffic spike?** Confirm it's real (not a bot), autoscale, cache aggressively, shed non-critical load, queue writes, pre-warm caches, request quota increases, use CDN for static.
5. **What breaks first at scale?** Usually the database (connections, hot keys, lock contention), then fan-out, then the network. Name the first one for your design and how you'd detect it.
6. **How do you estimate a database size for 5 years?** Records/day × size × 365 × 5 × replication factor × index overhead (~1.3–2x) + growth buffer (~30%).
7. **How do you handle hot partitions?** Detect via per-shard metrics; split the key (salting), add a cache layer, separate a hot tenant onto dedicated shard, rate-limit that key.

## B. Databases and Storage
8. **Why would you choose Cassandra over Postgres?** Massive write throughput, multi-datacenter, known query patterns by partition key, tolerance for eventual consistency. Cost: no joins, limited ad hoc queries, modeling by query.
9. **When not to use NoSQL?** When you need multi-row transactions, complex joins, ad hoc analytics, or strong constraints; or when the team lacks experience and scale doesn't demand it.
10. **How does a B-tree index differ from an LSM for reads?** B-tree: predictable point and range reads, in-place updates. LSM: writes fast, reads check multiple files (mitigated by Bloom filters), background compaction.
11. **Why does a random UUID primary key hurt B-tree performance?** Random inserts scatter across pages, causing splits and cache misses. Use time-ordered IDs (UUIDv7, Snowflake, ULID).
12. **What does an index cost?** Storage, write amplification, slower inserts/updates, optimizer choices. Index for actual queries; remove unused indexes.
13. **How do you paginate a 100M-row table?** Keyset pagination on an indexed column; avoid OFFSET; return an opaque cursor.
14. **What is a covering index?** An index containing every column a query needs, so the table is never read.
15. **How do you migrate a 2 TB table without downtime?** Create the new table/column, dual write, backfill in throttled batches, verify with checksums, switch reads behind a flag, stop old writes, drop old later.
16. **How do you choose between row and column storage?** OLTP point access and updates → row. Analytics over few columns across many rows → columnar.

## C. Consistency and Transactions
17. **Strong vs eventual consistency for a shopping cart?** Eventual is acceptable with merge rules (union of items); final checkout must validate inventory strongly.
18. **What is a race condition in a booking system and how to prevent it?** Two users see the last seat free and both book. Prevent with a conditional update or unique constraint on (show, seat), or a row lock within a transaction; return conflict to the loser.
19. **Explain write skew with an example.** Two on-call doctors each check "at least one other doctor is on call," both see yes, both go off call. Snapshot isolation allows it. Fix: serializable isolation or explicit locking on a shared row.
20. **How do you make an API call exactly once?** You can't, over an unreliable network. Use an idempotency key and dedupe server-side so the effect happens once.
21. **Two-phase commit vs saga?** 2PC: atomic but blocking and tightly coupled, used within a DB cluster. Saga: eventual, compensating, more resilient, needs business-level compensation design.
22. **How do you handle a failure midway in a 5-step saga?** Compensate completed steps in reverse order, each compensation idempotent; retry transient failures; park irrecoverable cases in a manual queue with alerts.
23. **What is read-your-writes and how do you provide it in a replicated system?** Read from leader for a window, or track the write's log position and read from a replica that has reached it, or send the client a version token.

## D. Distributed Systems
24. **How does Raft elect a leader?** Terms; followers timeout (randomized) and become candidates; they request votes; majority wins; the new leader sends heartbeats. Safety requires voters to only vote for candidates with up-to-date logs.
25. **How does Kafka choose a new leader for a partition?** The controller picks a new leader from the ISR. Unclean election (out-of-sync replica) is optional and risks loss.
26. **What is a fencing token and why is it needed?** A monotonic number issued with each lock; the protected resource rejects requests with older tokens. Needed because a paused client can wake up thinking it still holds the lock.
27. **How do you detect a dead node without false positives?** Use multiple signals (heartbeats plus indirect probes), adaptive detectors (phi accrual), confirm with quorum before removing a node from membership, and fence it.
28. **What is a split brain and how do you prevent it?** Two nodes both act as leader after a partition. Prevent with quorum (majority required to act), leases, fencing tokens, STONITH.
29. **What is a Merkle tree used for?** Efficiently find differing data between replicas: compare root hashes, descend into mismatched subtrees.
30. **Why does gossip scale?** Each node talks to a few random peers per round; information spreads in O(log N) rounds with no central coordinator and constant per-node load.

## E. Messaging and Streams
31. **How do you choose a Kafka partition count?** Target throughput per partition (tens of MB/s), consumer parallelism needed, room for growth (partitions can increase but reshuffle keys), broker limits. Common: a multiple of expected consumer count.
32. **How do you handle a poison message?** Bounded retries with backoff, then DLQ with metadata; alert; replay after fix; never block the partition forever.
33. **Kafka consumer processes then crashes before commit. What happens?** Message is redelivered after rebalance; processing must be idempotent.
34. **How do you process events in order per entity across retries?** Key by entity; single consumer per partition; on retry, don't skip ahead; or use per-entity sequence numbers and buffer out-of-order.
35. **What is backpressure in a streaming system?** Consumers signal slowness through bounded buffers or pull-based consumption so producers slow down instead of overwhelming downstream.
36. **Event time vs processing time: why does it matter?** Mobile clients send delayed events; processing-time windows give wrong aggregates. Event-time windows with watermarks give correct results with bounded lateness.
37. **How do you de-duplicate a stream with at-least-once delivery?** Unique event IDs, dedupe state with TTL (window), or idempotent upserts in the sink.

## F. Caching
38. **Why delete instead of update the cache on write?** Concurrent updates can write stale values back; deletion forces the next read to load fresh data. Still possible race; mitigate with TTL and versioning.
39. **Cache hit ratio dropped from 95% to 60%. Why?** Possible causes: key change (format or version bump), eviction from memory pressure, TTL too short, traffic pattern shift, cache node restart (cold cache), or cache key too granular.
40. **How does a CDN serve personalized content?** Cache the static shell at edge; fetch personalized parts via API; or cache per-user keys only where hit rate justifies it; use `Vary` carefully.
41. **Redis vs Memcached?** Redis: rich types, persistence, replication, Lua scripting, cluster. Memcached: simpler, multi-threaded, pure cache. Pick Redis for most modern needs.

## G. APIs and Communication
42. **REST vs gRPC for internal service calls?** gRPC for performance, streaming, strong contracts, and polyglot codegen; REST for simplicity, browser access, and public APIs.
43. **How do you version an event schema?** Schema registry with compatibility modes (backward, forward, full); add optional fields with defaults; never change meaning; use new topic for breaking change.
44. **How do you design an idempotent POST for creating an order?** Client sends Idempotency-Key; server stores (key, request hash, response, status) atomically; repeat with same key returns stored response; different body with same key returns 422.
45. **Polling vs WebSocket vs SSE for a live score feed?** SSE for one-way server push with auto-reconnect and HTTP compatibility; WebSocket if the client must send often; polling at low frequency if simplicity matters more than latency.

## H. Reliability and Operations
46. **How do you set timeouts across a call chain?** Each hop's timeout is smaller than its caller's remaining budget; propagate deadlines; avoid timeouts larger than the user's patience.
47. **How do you avoid cascading failure?** Timeouts, bulkheads, circuit breakers, load shedding, bounded queues, retry budgets, and graceful degradation of non-critical features.
48. **What is an SLO and how do you pick one?** Start from user expectations and what the business needs; measure current performance; set target slightly below current reliability you can sustain; define error budget and policy.
49. **How do you do a safe deployment of a breaking change?** Expand-contract: deploy readers that support both, then writers, then remove old. Feature flags, canary, rollback plan.
50. **How do you debug high p99 latency?** Traces to find the slow span; check queue depth, GC pauses, lock contention, DB slow queries, connection pool wait time, retries and timeouts, noisy neighbors; compare per-version and per-region.
51. **What is a runbook?** Documented steps for a known alert: symptoms, checks, mitigation, escalation, rollback commands.

## I. Security
52. **How do you store passwords?** Slow salted hash (argon2id/bcrypt/scrypt), unique salt per user, pepper optional in a KMS, rate-limit logins, support MFA.
53. **Access token vs refresh token?** Access token short-lived, sent with each request; refresh token long-lived, exchanged for new access tokens, rotated and revocable.
54. **How do you prevent IDOR?** Check object ownership or permission on every request server-side; use non-guessable IDs as defense-in-depth, not as the control.
55. **How do you secure service-to-service calls?** mTLS with workload identity, short-lived certificates, authorization policies per service, secrets in a vault, network policies.

## J. Design Judgment (senior-level)
56. **When would you not use microservices?** Small team, early product, unclear domain boundaries, low scale, no independent deploy need. Start with a modular monolith.
57. **How do you decide build vs buy for a component?** Core differentiator and unique requirements → build; commodity (auth, payments, email, search at small scale) → buy; consider lock-in, cost at scale, team capacity.
58. **How would you migrate a monolith to services?** Identify bounded contexts, put a facade in front, extract one low-risk capability first, dual-run with comparison, move data ownership last, keep rollback path.
59. **How do you handle technical debt?** Track it, tie it to business cost, pay down in the same area as feature work, set guardrails.
60. **What would you cut to ship in 2 weeks?** Non-critical features, exotic consistency, multi-region, perfect analytics; keep correctness of money/auth and observability.

## K. Your Resume Topics (be ready for deep follow-ups)
61. **Kafka at 1M+ events/day: how did you partition?** Explain key choice, number of partitions, consumer group sizing, and what hot-partition issues you saw and how you handled them.
62. **Why migrate from Kafka to Iceberg for batch-heavy ingestion?** Druid supervisors were loaded with large batch ingestion; Iceberg gives cheap, partitioned Parquet storage and direct partition queries; trade-off is freshness, so real-time queries stayed in Druid.
63. **How did Temporal help with 1M+ files/day?** Durable workflows with per-activity retries and timeouts, task queues to isolate workloads, activities designed idempotent; visibility for stuck workflows.
64. **How did you get 95% faster report generation?** Turned formulas into optimized Druid SQL, pre-aggregated via rollup and compaction, cut repeated computation. Quantify the baseline and measurement.
65. **Consistent hashing with 200 vnodes: why?** Even distribution (law of large numbers across vnodes), smaller data moves on node churn, tunable weight per machine. Cost: ring size and lookup cost (negligible with binary search).
66. **Zero cache invalidation on node add/remove: is that accurate?** Be precise: keys that move to the new node must be re-fetched or migrated; say what your implementation actually did. Credibility depends on precision.
67. **Distributed cache replication: sync or async?** State what you did and the failure window; explain how you would choose the other option under stricter requirements.
68. **Redis-based real-time presence on serverless without WebSockets?** Presence in sorted set scored by last-seen timestamp; TTL cleanup; clients poll or use SSE; trade-off is latency and polling load vs cost and complexity of managing WebSocket servers.
69. **Keycloak with RBAC on GraphQL: how did you enforce authz?** Token claims for roles at the gateway; field-level or resolver-level checks for object ownership; never rely only on the gateway.
70. **How would you scale your ETL platform 10x?** Partition work by vendor and time, scale Temporal workers and task queues separately, Kafka partitions to match, Iceberg compaction schedules, storage tiering, and cost controls.

## L. Rapid Scenario Drills (30 seconds each)
71. Database primary crashes. First three actions? Confirm failure (not partition), trigger failover to most up-to-date replica with fencing, check for lost writes and reconcile, communicate status.
72. Latency doubled after deploy. Steps? Compare canary metrics by version, check traces, roll back if SLO burn, then diagnose.
73. Queue depth climbing. Steps? Check consumer health and throughput, downstream latency, partition skew, scale consumers, shed or pause producers if necessary.
74. Cache hit ratio falls to zero. Steps? Check cache cluster health, key format change, eviction, deployment with new key prefix; protect DB with rate limits; warm cache.
75. Duplicate charges reported. Steps? Check idempotency key handling, retry policy at all layers, webhook replays, reconciliation; fix root cause; refund and notify.
76. One region goes down. Steps? Confirm scope, shift traffic per failover runbook (DNS/global LB), check data replication lag, handle writes conflicts, communicate.
77. A hot user causes shard overload. Steps? Identify key via metrics, throttle that key, move tenant to dedicated shard, add caching, plan long-term fix.
78. Disk is full on a log broker. Steps? Free space by reducing retention or moving logs, avoid data loss by checking ISR and replication health, add capacity, prevent recurrence with alerts at 70/85%.
79. Cert expired, mTLS failing. Steps? Identify affected services, rotate certificates, verify trust chain, add expiry alerts and automated renewal.
80. Schema migration locks a table. Steps? Abort if safe, use online DDL or batched approach, have a rollback, test on staging with production-size data.

## M. Behavioral Prompts for Architects
81. Describe a design you regret. What would you do differently?
82. Describe a time you pushed back on a design or a requirement.
83. Describe a time you led a migration.
84. How do you communicate a complex design to non-engineers?
85. How do you mentor engineers on system design?

Model answer structure for 81–85: situation (1 sentence), decision and reasoning, what happened (with metric), what you learned, what you do now.
