# System Design Interview Prep: 8-Week Plan

**Target:** Backend / SDE-2 level system design (HLD + LLD), with a strong base for senior roles.
**Starting point (from your resume):** 2+ years backend, Kafka at 1M+ events/day (per your current PDF resume), Temporal workflows, Druid/Iceberg, GraphQL APIs, a distributed cache built from scratch, Redis-based real-time features.
**Weekly time:** about 10 hours. Weekdays 1.5 hrs, weekend 3–4 hrs.

---

## 1. How to Use These Files

| File | What it is | When to use |
|---|---|---|
| `00_Study_Plan.md` | This plan, routine, tracker | Every day |
| `01_HLD_Guide.md` | Interview framework, estimation, building blocks, 15 case studies | Weeks 1–5 |
| `02_LLD_Guide.md` | OOP principles, design patterns, 10 LLD problems with code | Weeks 6–7 |
| `03_QA_and_Your_Stories.md` | Rapid-fire Q&A, how to tell your resume projects as system design | Week 8 and the day before any interview |

---

## 2. The 8-Week Roadmap

### Week 1: Foundations (fundamentals you will be tested on)
- Read/skim: Alex Xu, *System Design Interview Vol 1*, Ch. 1–4 (framework, rate limiter, consistent hashing, key-value store).
- Learn: latency numbers, estimation, the interview framework (`01_HLD_Guide.md` Part A and B).
- Practice: 3 estimation drills (see Part C of the HLD guide).
- Goal: you can explain consistent hashing and CAP without notes.

### Week 2: Data layer
- Read: *Designing Data-Intensive Applications* (DDIA) Ch. 2, 3, 5, 6 (data models, storage engines, replication, partitioning).
- Learn: B-tree vs LSM, indexes, replication (leader/follower), sharding, SQL vs NoSQL, transactions and isolation levels.
- Practice: design a URL shortener (Case 1) end to end, timed at 45 minutes.

### Week 3: Messaging, caching, scale patterns
- Read: DDIA Ch. 8, 9, 11 (distributed issues, consistency, stream processing).
- Learn: Kafka internals (partitions, offsets, consumer groups, delivery semantics), caching strategies, idempotency, outbox, sagas.
- Practice: Cases 2 (rate limiter), 3 (distributed cache), 4 (notification system).

### Week 4: Core HLD case studies
- Practice: Cases 5 (chat), 6 (news feed), 7 (typeahead), 8 (ride sharing).
- Do 2 timed mocks with a friend or on your own (record yourself).

### Week 5: Advanced HLD
- Practice: Cases 9 (payments), 10 (web crawler), 11 (file storage), 12 (metrics system), 13 (video streaming), 14 (job scheduler), 15 (event pipeline).
- Do 2 more mocks. Target: finish the framework in time with no skipped steps.

### Week 6: LLD foundations and patterns
- Read: `02_LLD_Guide.md` Part A and B (SOLID, patterns).
- Practice: Parking lot, LRU cache, vending machine (State pattern), rate limiter LLD.

### Week 7: LLD problems and concurrency
- Practice: elevator, logger, Splitwise, movie ticket booking (concurrency), notification (Observer), tic-tac-toe.
- Write the code in Python or Java (whichever you're fastest in), without an IDE's autocomplete.

### Week 8: Mocks, weak spots, your stories
- Do 4 full mocks (2 HLD, 2 LLD), each with a 1-page written self-review.
- Prepare your resume project stories using `03_QA_and_Your_Stories.md`.
- Rehearse the 20 rapid-fire questions out loud.

---

## 3. Daily Routine (90 minutes)

| Block | Time | Activity |
|---|---|---|
| Warm-up | 10 min | Review yesterday's rapid-fire questions (no notes) |
| Learn | 30 min | Read one concept or chapter; write 5–10 bullets in your own words |
| Practice | 40 min | Timed problem (45 min budget) or LLD coding |
| Review | 10 min | Compare with the reference; list 3 gaps |

**Rules:**
1. Draw on paper or a whiteboard, never only in your head. Talk out loud as if an interviewer is listening.
2. Time every practice. Use a timer and stop at the limit.
3. Never read the solution before attempting the problem for at least 20 minutes.
4. Every week, redo one old problem from scratch without notes.

---

## 4. Self-Grading Rubric (use after each mock)

Score each 1–5:

| Dimension | What a 5 looks like |
|---|---|
| Requirements | Asked 4–6 clarifying questions; stated functional and non-functional requirements; agreed scope out loud |
| Estimation | Computed QPS (average and peak), storage per year, bandwidth; used the numbers to make a decision |
| API and data model | Clear endpoints with inputs and outputs; chose storage with a reason |
| High-level design | Clean diagram; each component has a job; data flow for the main use case is clear |
| Deep dives | Went deep on 2+ bottlenecks (scaling, hot keys, failures, consistency) with trade-offs |
| Trade-offs | Named alternatives and why you chose one; said what you gave up |
| Communication | Structured; checked in with the interviewer; managed time |

**Passing bar for SDE-2:** average 3.5+ with no dimension below 3.
**Senior bar:** average 4+ with strong deep dives and proactive failure-mode thinking.

---

## 5. Progress Tracker

Copy this table into a note and fill it in.

| # | Topic | Type | Date | Time taken | Self-score (avg) | Weak spot |
|---|---|---|---|---|---|---|
| 1 | URL shortener | HLD | | | | |
| 2 | Rate limiter | HLD | | | | |
| 3 | Distributed cache | HLD | | | | |
| 4 | Notification system | HLD | | | | |
| 5 | Chat app | HLD | | | | |
| 6 | News feed | HLD | | | | |
| 7 | Typeahead | HLD | | | | |
| 8 | Ride sharing | HLD | | | | |
| 9 | Payment system | HLD | | | | |
| 10 | Web crawler | HLD | | | | |
| 11 | File storage (Dropbox) | HLD | | | | |
| 12 | Metrics / monitoring | HLD | | | | |
| 13 | Video streaming | HLD | | | | |
| 14 | Job scheduler | HLD | | | | |
| 15 | Event pipeline | HLD | | | | |
| 16 | Parking lot | LLD | | | | |
| 17 | LRU cache | LLD | | | | |
| 18 | Vending machine | LLD | | | | |
| 19 | Rate limiter | LLD | | | | |
| 20 | Elevator | LLD | | | | |
| 21 | Logger | LLD | | | | |
| 22 | Splitwise | LLD | | | | |
| 23 | Movie booking | LLD | | | | |
| 24 | Notification (Observer) | LLD | | | | |
| 25 | Tic-tac-toe | LLD | | | | |

---

## 6. Resources (pick one main per area; don't collect too many)

**Core reading (choose these):**
- *Designing Data-Intensive Applications* (Martin Kleppmann): the single best book for depth. Read it in weeks 2–3.
- *System Design Interview* Vol 1 and 2 (Alex Xu): good for structure and common questions.

**Practice:**
- Hello Interview (hellointerview.com): free, structured HLD and LLD walkthroughs.
- ByteByteGo: short visual explainers.
- The "system-design-primer" GitHub repo: reference only.

**LLD:**
- Refactoring Guru (design patterns, with Python and Java examples).
- *Head First Design Patterns*: optional, for intuition.

**Check the current content of these resources yourself before relying on them; they update often.**

---

## 7. Interview Day Checklist

- [ ] Re-read your resume project stories (`03_QA_and_Your_Stories.md`, Part C).
- [ ] Revise the estimation cheat sheet (HLD guide, Part C).
- [ ] Have a 4-step framework in mind: Requirements → Estimates → API/Data → High-level → Deep dives.
- [ ] Ask the interviewer at the start: "Should I focus on the core flow or go broad?"
- [ ] Watch the clock: at 35 min you should be in deep dives.
- [ ] End with a summary: what you built, what you'd improve next, and the main trade-off you made.
