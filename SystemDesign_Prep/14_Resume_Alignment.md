# 14. Resume Alignment (based on your current PDF)

Source of truth: `Abhishek_Kumar_Backend_Engineer.pdf` (the version you shared). The older `.tex` file in this folder has different numbers (1B+ events/day, 10M+ files/day, a KPI formula engine). Use **only** the PDF numbers in interviews, and update the .tex if you keep using it.

## 1. Facts to Know Cold

| Resume line | Number to know | How to explain it (be ready) |
|---|---|---|
| Kafka → Druid, real-time dashboards | 1M+ events/day, sub-second queries | Partitioning key, consumer group size, Druid ingestion lag, query latency measurement |
| Temporal parallel workflows | 1M+ XML files/day | Workflow per file or batch, activity retries/timeouts, task queue isolation, idempotent activities |
| Kafka → Iceberg migration | Druid supervisor load down 40% | Measured how? (CPU, task count, lag). Trade-off: freshness vs cost |
| Python compaction, dedup, rollup | up to 10x faster queries | Baseline query time vs after; small-file problem; rollup granularity |
| SQL query-generation service | 95% less report generation time | Before/after timing; what was repeated work that got removed |
| Automation with Python and CI/CD | 70–80% less manual effort | Which tasks, how measured (hours per week before/after) |
| Distributed cache (Node.js) | 200 vnodes, master–slave, custom LB | Ring lookup, churn behavior, replication failure window |
| CamelThar | 13+ REST endpoints, sub-10ms cache hits, 50ms chat delivery | Measurement method (p50/p99? where measured?) |

## 2. Corrections to Make Before Any Interview
1. **"Zero cache invalidation"** (distributed cache). This is strong and easy to challenge. Say precisely what happened to keys on node add/remove (re-fetched from source, migrated, or lost until re-populated). Interviewers will ask.
2. **"Sub-second query latency"** — state the p95/p99 you measured and the query shape.
3. **"Ingesting 1M+ events/day"** is modest for Kafka. Be ready to say why the design still needed partitioning and idempotency (correctness, not raw volume). Do not oversell scale; match the number to the story.
4. **"AI-Engineered" and "Prompt-Driven Design" in the project and skills**: be ready to explain exactly what AI tools did in your work and what you designed yourself.
5. **Serverless + WebSockets:** CamelThar used Redis-based real-time without WebSockets. Expect "why not WebSockets?" and answer with the Vercel constraint and the trade-off (polling latency and load vs operational cost).

## 3. Story Map (one diagram each, 5 minutes)
- **Telecom pipeline:** Temporal orchestration → parsers → Kafka → Druid → query API; side path Kafka → Iceberg (see 03 Part C).
- **Distributed cache:** ring with 200 vnodes → shard nodes with replicas → LB routing (see 03 Part C).
- **CamelThar:** Next.js API → Redis layers (cache, rate limit, presence sorted sets, typing TTLs) → Postgres fallback (see 03 Part C).

## 4. Gaps an Interviewer May Probe (prepare an honest answer)
- Multi-region or large-scale ownership: you have not owned a multi-region system. Say so and describe how you would approach it (see 07 §7.7).
- Consensus and leader election: you used Temporal and Kafka, which rely on these; be able to explain the concepts (05 §5.6).
- On-call and incident leadership: prepare one real incident story (see 03 Part B).

## 5. Resume Keywords vs Deep Files
| Keyword | Where to study |
|---|---|
| Kafka, partitions, offsets | 06 §6.2 |
| Temporal, durable execution | 06 §6.4, 13 Case 14 |
| Druid, rollup, segments | 13 Case 12, 12 (metrics), 06 §6.5 |
| Iceberg, compaction, snapshots | 06 §6.5, 13 Case 15 |
| Consistent hashing, replication | 05 §5.2–5.3, 11 Case 3 |
| Redis sorted sets, TTL, presence | 06 §6.6.2, 12 Case 6 (ZSET timeline), 11 Case 5 |
| GraphQL, Apollo, DataLoader | 07 §7.1.3 |
| Keycloak, JWT, OAuth, RBAC | 07 §7.5 |
| Zod, rate limiting | 07 §7.3, 11 Case 2 |
| Serverless, Neon | 08 §8.1 (serverless), 04 §4.5 (isolation in serverless Postgres) |
