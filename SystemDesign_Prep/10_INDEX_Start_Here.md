# SystemDesign_Prep: Index (start here)

Read in this order.

| # | File | Covers | Time |
|---|---|---|---|
| 00 | 00_Study_Plan.md | 8-week schedule, routine, rubric, tracker | 15 min |
| 01 | 01_HLD_Guide.md | Interview framework, building blocks overview, estimation, 15 case summaries | 3 h |
| 02 | 02_LLD_Guide.md | SOLID, patterns, 10 LLD problems with code | 3 h |
| 03 | 03_QA_and_Your_Stories.md | Rapid-fire Q&A, behavioral prep, resume project stories | 1 h |
| 04 | 04_Deep_Foundations.md | Latency numbers, networking, time, storage engines, indexes, transactions and isolation | 3 h |
| 05 | 05_Deep_Distributed_Systems.md | Replication, partitioning, consistency models, CAP/PACELC, consensus, locks, failure detection, idempotency, distributed transactions, CRDTs | 4 h |
| 06 | 06_Deep_Messaging_Streaming_Caching.md | Kafka internals, stream processing, lakehouse tables, Redis internals, caching patterns, CDN | 3 h |
| 07 | 07_Deep_APIs_Resilience_Observability_Security.md | REST/gRPC/GraphQL, resilience patterns, rate limiting, SLOs, security, deployment, DR | 3 h |
| 08 | 08_Architecture_Patterns_and_Reference_Designs.md | Monolith vs microservices, event-driven, CQRS, event sourcing, cells, 13 reference architectures, DB selection | 2.5 h |
| 09 | 09_Deep_Question_Bank.md | 85 questions with model answers, scenario drills, behavioral | 2 h |
| 11 | 11_HLD_Walkthroughs_Part1.md | Full walkthroughs: URL shortener, rate limiter, distributed cache, notifications, chat | 4 h |
| 12 | 12_HLD_Walkthroughs_Part2.md | Full walkthroughs: news feed, typeahead, ride sharing, payments, web crawler | 4 h |
| 13 | 13_HLD_Walkthroughs_Part3.md | Full walkthroughs: file storage, metrics, video streaming, job scheduler, event pipeline | 4 h |
| 14 | 14_Resume_Alignment.md | Your current resume PDF: numbers to know, corrections, story map, gaps | 45 min |
| 15 | 15_LLD_Advanced.md | Chess, library, food delivery, meeting rooms, snake and ladder, multi-level cache with single-flight | 4 h |

## Coverage Map: every architecture topic and where it lives

**Foundations:** latency numbers (04 §4.1), networking (04 §4.2, 07 §7.1), time (04 §4.3, 05 §5.7)
**Storage:** engines (04 §4.4), indexes (04 §4.4.6), data modeling (04 §4.6), columnar and lakehouse (04 §4.4.5, 06 §6.5)
**Transactions:** ACID, isolation, anomalies (04 §4.5), distributed transactions and sagas (05 §5.9), outbox and inbox (05 §5.8.3–4)
**Distribution:** replication (05 §5.2), partitioning (05 §5.3), consistency models (05 §5.4), CAP and PACELC (05 §5.5), consensus (05 §5.6), membership and gossip (05 §5.7), CRDTs (05 §5.10)
**Messaging and streams:** queues vs logs (06 §6.1), Kafka (06 §6.2), other brokers (06 §6.3), stream processing and windows (06 §6.4), Lambda and Kappa (06 §6.4.4)
**Caching and CDN:** Redis internals (06 §6.6.2), eviction (06 §6.6.3), consistency patterns (06 §6.6.4), stampede and penetration (06 §6.6.5–6), CDN (06 §6.7, 04 §4.2.8)
**Load balancing and networking:** L4/L7, algorithms (04 §4.2.7), DNS (04 §4.2.2), real-time transport (04 §4.2.6)
**APIs:** REST, gRPC, GraphQL, versioning, pagination, webhooks (07 §7.1)
**Resilience:** timeouts, retries, circuit breaker, bulkhead, load shedding (07 §7.2), rate limiting (07 §7.3, 03 Case 2)
**Observability:** metrics, logs, traces, SLO, error budget, incidents (07 §7.4)
**Security:** AuthN/AuthZ, OAuth2/OIDC, JWT, mTLS, encryption, attacks (07 §7.5)
**Deployment and DR:** canary, blue-green, migrations, multi-region, cells, RTO/RPO (07 §7.6–7.7, 08 §8.1)
**Architecture styles:** monolith, microservices, event-driven, event sourcing, CQRS, hexagonal, serverless, service mesh, BFF (08 §8.1)
**Systems:** 13 reference architectures (08 §8.2); 15 full walkthroughs (11, 12, 13); case summaries (01 Part D)
**Your resume:** 14_Resume_Alignment.md (use the PDF numbers, not the older .tex)
**LLD:** principles, patterns, concurrency, 10 problems (02), 6 advanced problems and techniques (15)

## Study Method for Depth (what to do with these files)
1. Read each deep file once, slowly. Write one-line summaries in your own words.
2. For each section, answer aloud: "what, how it works, trade-off, failure mode."
3. Do the 85 questions in file 09: say your answer, then check. Mark those you missed.
4. Redo two HLD cases per week with the full framework; use the deep files to justify each choice.
5. Before the interview, re-read file 03 and your weakest deep file.
