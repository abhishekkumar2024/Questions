# Q&A Bank, Behavioral Prep, and Your Resume Stories

---

## Part A: Rapid-Fire Questions (answer aloud in 60–90 seconds)

**Fundamentals**
1. What is the difference between horizontal and vertical scaling? When would you pick each?
2. Explain CAP theorem with a real example.
3. What is PACELC and how is it different from CAP?
4. SQL vs NoSQL: how do you choose?
5. What is a B-tree and why do databases use it?
6. What is an LSM tree? Where is it used?
7. Explain database indexes and the trade-off they introduce.
8. What is replication lag and how do you handle read-your-own-writes?
9. Explain sharding. What makes a good shard key?
10. What is consistent hashing and why use virtual nodes?

**Caching and messaging**
11. Cache-aside vs write-through vs write-back: when do you use each?
12. What is a cache stampede and how do you prevent it?
13. How do you handle hot keys?
14. Kafka: what guarantees does it give for ordering? Within what scope?
15. At-least-once vs exactly-once: what do you actually implement in practice?
16. How do you make a consumer idempotent?
17. What is consumer lag and what do you do when it grows?
18. What is a dead letter queue and when do messages go there?

**Reliability and distributed systems**
19. What is idempotency and where do you need it?
20. Explain the transactional outbox pattern.
21. Saga vs two-phase commit: what are the trade-offs?
22. How do you design retries? (backoff, jitter, retry budgets)
23. What is a circuit breaker and when does it open or close?
24. What is quorum? Explain R + W > N.
25. How do you detect and handle a failed node?
26. What is a leader election, and what tools provide it? (ZooKeeper, etcd, Raft)
27. Explain eventual consistency with an example users would notice.

**APIs, security, and ops**
28. REST vs gRPC vs GraphQL: when do you use each?
29. Pagination: offset vs cursor. Which and why?
30. How do you version an API?
31. JWT vs session tokens: pros and cons.
32. OAuth2 vs OIDC.
33. How do you prevent a double-charge on a retry?
34. What metrics would you monitor for this service? (RED, USE, SLOs)
35. Canary deployment vs blue-green deployment.

**Estimation and numbers**
36. How many seconds are in a day? (~86,400, use 10^5)
37. Estimate QPS for 10M DAU with 10 actions each. (~1,157 avg; peak ~3K)
38. Roughly how long does a cross-continent round trip take? (~150 ms)
39. How much storage for 1B records of 1 KB each? (1 TB)
40. When would you add a CDN? (static content, geographically spread users)

Keep answers short. If you can't answer in 90 seconds, you probably need one more pass on that topic.

---

## Part B: Common Behavioral Prompts (use the STAR format)

STAR = **Situation, Task, Action, Result**. Keep each part to 2–3 sentences. End with a number.

1. Tell me about a system you built that had to scale. What was the hardest part?
2. Describe a production incident you handled. What did you learn?
3. Tell me about a time you disagreed with a technical decision.
4. Describe a design you would change if you did it again.
5. How do you decide between build and buy?
6. Tell me about a time you had to learn something quickly.
7. How do you prioritize when you have too many tasks?
8. Why are you leaving your current role? (keep it positive and forward-looking)

**Prepare two stories from your work:**
- A **scaling story** (Kafka/Druid/Iceberg).
- An **incident or failure story** (for example: a retry storm, a bad migration, a Temporal workflow stuck in a retry loop, a data mismatch found by reconciliation). Use a real one.

---

## Part C: Turning Your Resume into System Design Stories

Your resume has strong material. Interviewers often ask "tell me more about this project" and then go into system design. Prepare each project as a 5-minute story with a diagram.

### Story 1: Telecom NPM Platform (Whiteklay Technology)

**Pitch (30 seconds):** "We ingested telecom performance data from five network vendors. I built the parsing and orchestration for 1M+ XML files per day, and Kafka streams that bring 1M+ events/day into Druid events per day into Apache Druid for near-real-time KPI dashboards."

**Architecture to draw:**
```
Vendor files (XML) → Ingestion (MinIO/S3)
    → Temporal workflows (retries, timeouts, task queues) → ETL parsers (config-driven, normalize counters)
    → Kafka (partitioned) → Druid (real-time ingestion, segments, rollup)
    → Query API (GraphQL, Node.js) → Dashboards
Batch path: Kafka → Iceberg (Parquet on S3) → compaction/dedup jobs → Druid for direct partition queries
```

**Talking points by topic:**

| Topic | What you did | What to say about it |
|---|---|---|
| Orchestration | Temporal workflows with retry policies, timeouts, dedicated task queues | Durable execution: workflow state survives crashes; activities are retried; idempotent activities are essential |
| Kafka at 1M+/day | Streaming ingestion into Druid | Partition key choice; consumer scaling; lag monitoring; at-least-once with idempotent writes/dedupe |
| Kafka → Iceberg migration | Moved batch-heavy ingestion, cut Druid supervisor load by 40% | Trade-off: Druid is great for fresh, low-latency queries; Iceberg is cheaper for large batches and supports partition pruning and time travel |
| SQL query-generation service | Generates Druid SQL for analytics; cut report generation time by 95%; standardized queries across teams | Compiling inputs into a safe query; validation; avoiding injection; caching; consistency across teams |
| Compaction and rollup | Python jobs for compaction, dedup, rollup; up to 10x faster queries | Small-file problem; rollup trades granularity for speed; compaction schedule vs freshness |
| GraphQL API | Apollo Server, Prisma, PostgreSQL, Redis, Keycloak RBAC | N+1 problem and DataLoader; caching at the resolver; field-level authorization |

**Likely follow-ups (prepare answers):**
- "What happens if Kafka consumers fall behind?" → lag alerts, scale consumers up to partition count, check for hot partitions, backpressure.
- "How do you avoid duplicate records in Druid?" → idempotent event IDs, dedup job on the lake, and the reconciliation check you built.
- "How do you handle a vendor sending a malformed file?" → validation, DLQ or quarantine, alerting, and replay after a fix.
- "Why Temporal and not plain cron plus a queue?" → retries with state, long-running workflows, visibility, and cancellation.

**Verify before the interview:** the figures on your resume (1M+/day, 1M+ files/day, 40%, 95%, 10x, 70–80%). You must be able to explain how each was measured. If a number is an estimate, say so ("roughly", "peak") and give the calculation.

### Story 2: Distributed Cache (Your Project)

**Pitch:** "I built a sharded cache cluster in Node.js with consistent hashing using 200 virtual nodes per server, and master–slave replication with a load balancer. Adding or removing nodes causes minimal key movement."

**Draw it:**
```
Client → Load balancer/router (consistent hash ring, 200 vnodes per node)
    → Shard A (master) ↔ replica
    → Shard B (master) ↔ replica
    → Shard C (master) ↔ replica
```

**Be ready to answer:**
- Why 200 virtual nodes? → Smooths distribution; trade-off is ring memory and lookup cost (log n).
- What happens when a node is removed? → Only keys on its arc move (about 1/N). Explain how you avoided "zero invalidation" in your project (did keys move? were they re-fetched from the source?). **Be precise about what your implementation really does.**
- Replication: async or sync? → What did you choose and why? What is the failure window?
- What happens on master failure? → Promotion, client redirect, and data loss window.
- How is it different from Redis Cluster? → Hash slots (16384) vs a ring; gossip-based membership; built-in failover.
- How would you make it production-ready? → Persistence, monitoring, connection pooling, eviction policy, security, tests under failure.

### Story 3: CamelThar Travel Platform (Personal Project)

**Pitch:** "I built and launched a Next.js travel platform on Vercel with 13+ REST endpoints, JWT auth, Redis rate limiting, and a Redis-first real-time layer for chat delivery and presence without WebSockets."

**Good discussion points:**
- **Serverless constraints:** no long-lived WebSocket connections on Vercel functions. Explain why you used Redis and polling-style or SSE-style patterns, and what the trade-off is (latency vs cost and complexity).
- **Presence with sorted sets:** store user ID with a timestamp score; expire stale entries; count online users with a range query.
- **Typing indicators with TTL:** keys expire automatically, so no cleanup job is needed.
- **Atomic booking:** how you prevented race conditions (a Redis lock, or a conditional DB update). Explain the difference.
- **Rate limiting:** which algorithm, where it runs, and what happens if Redis is down (fail open or closed?).

**Interview tip:** this project shows that you can design end to end as a single engineer. Use it when they ask about a full product or a smaller-scale system.

---

## Part D: Turning Projects into a System Design Answer (the template)

Use this when asked "design X" and it matches one of your projects:
1. "I've worked on something similar: [one-line description with a number]."
2. Requirements: "Let me confirm..." (use your project's real constraints).
3. Estimation: use the real scale, then extend it ("at 10× we would...").
4. Design: draw what you built.
5. Deep dives: pick the parts where you learned the most, including what you would change.
6. Reflection: "If I rebuilt this today, I'd change X because Y."

Interviewers love the reflection step. It shows judgment, not just memory.

---

## Part E: Questions to Ask the Interviewer

- What does the team's current architecture look like at a high level?
- What's the biggest scaling challenge the team faces this year?
- How do you handle on-call and incident reviews?
- What does success look like for this role in the first 6 months?
- How do engineers here decide between building and buying a system?

Avoid asking about salary or benefits in the technical round.

---

## Part F: Final Week Checklist

- [ ] Rapid-fire questions: answered out loud, all 40, within 90 seconds each.
- [ ] Two behavioral stories (scaling and failure) written as STAR, under 2 minutes each.
- [ ] Three project stories rehearsed with a diagram each (Part C).
- [ ] Every number on your resume: you can explain how it was measured.
- [ ] Four full mock interviews done (two HLD, two LLD), each self-graded with the rubric in `00_Study_Plan.md`.
- [ ] A one-page cheat sheet made from the framework, estimation numbers, and your three stories.
