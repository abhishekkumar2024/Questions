# 05. Deep Distributed Systems: Replication, Partitioning, Consensus, Consistency, Transactions

---

## 5.1 Why Distributed Systems Are Hard (the fallacies)

Assume none of these are true:
1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

**Partial failure** is the core problem: a node can be slow, partitioned, or dead, and from another node these look the same. Every design decision is about how to make progress and stay correct despite that.

**Timeouts are guesses.** A timeout does not tell you whether the remote operation happened. This is why idempotency matters (Section 5.8).

---

## 5.2 Replication

### 5.2.1 Single-leader (primary–replica)
- All writes go to the leader; it streams the WAL/binlog to followers.
- **Sync replication:** leader waits for follower ack. No data loss on failover, slower writes, one slow follower stalls writes.
- **Async replication:** leader acks immediately. Fast, but failover can lose the last few writes.
- **Semi-sync:** wait for one follower.
- **Replication lag:** followers may serve stale reads. Solutions:
  - **Read-your-writes:** route the user's reads to the leader for a short window after writes, or track the write's LSN/timestamp and read from a replica that has caught up.
  - **Monotonic reads:** pin a user to one replica.
  - **Consistent prefix:** causally related writes stay in order.
- **Failover:**
  1. Detect leader failure (heartbeat timeout).
  2. Elect a new leader (most up-to-date follower).
  3. Reconfigure clients and old leader (fencing).
  - **Split brain:** two nodes think they are leader. Prevent with **fencing tokens** (monotonic token; storage rejects older tokens) and STONITH (power off the old leader).
  - **Lost writes:** async failover can drop committed-on-old-leader writes. Mitigate with sync replication or accepting the risk explicitly.

### 5.2.2 Multi-leader
- Each region has a leader; writes local; async replication between leaders.
- **Use for:** multi-region writes with low local latency; offline-capable clients (each device is a leader, e.g., calendars, notes).
- **Conflict resolution:**
  - Last write wins (LWW) by timestamp: simple, **silently loses data**.
  - Merge via application logic (e.g., union of sets).
  - CRDTs (Section 5.9).
  - Ask the user (Git-style merges).
- **Write conflicts** detected via version vectors.

### 5.2.3 Leaderless (Dynamo-style: Cassandra, Riak, Voldemort)
- Client (or coordinator) writes to N replicas; succeeds when W ack; reads from R replicas.
- **Quorum:** `R + W > N` ensures read and write sets overlap. Typical N=3, W=2, R=2.
- **Sloppy quorum + hinted handoff:** if a home replica is down, write to another node with a hint to forward later. Higher availability, weaker guarantees.
- **Read repair:** on read, detect stale replicas and update them.
- **Anti-entropy:** background process compares replicas using **Merkle trees** and syncs differences.
- **Limit:** quorum does not give linearizability on its own (races, sloppy quorums, concurrent writes).

### 5.2.4 Consensus-based replication (Raft/Paxos)
- Strongly consistent replicated state machine: etcd, ZooKeeper, Consul, CockroachDB ranges, Spanner (Paxos groups), Kafka KRaft.
- Majority commit; survives minority failures. A 5-node group tolerates 2 failures.

---

## 5.3 Partitioning (Sharding)

### 5.3.1 Strategies
| Strategy | How | Pros | Cons |
|---|---|---|---|
| Range | Keys sorted into contiguous ranges | Range scans, ordered data | Hot spots on sequential keys (timestamps) |
| Hash (modulo) | `hash(key) % N` | Even spread | Resharding moves almost everything |
| Consistent hash | Ring with vnodes | Minimal movement on change | Less range locality |
| Directory / lookup | Table maps key → shard | Flexible, move any key | Directory is a bottleneck and must be HA |
| Geo / tenant | By region or tenant | Data locality, compliance | Uneven load, cross-tenant queries hard |

### 5.3.2 Secondary indexes on partitioned data
- **Document-partitioned (local index):** each shard indexes its own docs. Writes are local; reads for a secondary attribute must **scatter-gather** to all shards.
- **Term-partitioned (global index):** index partitioned by term. Reads go to one shard; writes update index shards across the cluster (often async, so index may lag).

### 5.3.3 Rebalancing
- Fixed number of partitions (e.g., 1024) assigned to nodes; move whole partitions when nodes join. Used by Kafka, Riak, Elasticsearch.
- Dynamic splitting (range-based, e.g., HBase, Spanner) when a partition gets too big or hot.
- Avoid automatic rebalancing storms: throttle moves, and rebalance only on true membership changes.

### 5.3.4 Hot partitions and hot keys
- Detect with per-partition metrics.
- **Salting:** `key + random suffix` for write-hot keys; reads fan out to all suffixes and merge.
- **Split the entity:** e.g., per-user counters aggregated periodically.
- **Local aggregation / batching** before writing.
- **Celebrity problem:** a key with millions of followers or a viral item.

### 5.3.5 Cross-partition operations
- Joins across shards are expensive: denormalize, or do them in a stream/batch job.
- Transactions across shards need 2PC, sagas, or a single shard design (co-locate related data by a tenant or user key).
- Choose the shard key so the common query hits **one** shard.

---

## 5.4 Consistency Models (know the spectrum)

From strongest to weakest:
1. **Strict serializability** (linearizable transactions) — Spanner, CockroachDB (with caveats).
2. **Linearizability** — every operation appears to take effect atomically at a point between its invocation and response. Like a single copy. Needed for leader election, locks, unique constraints.
3. **Sequential consistency** — all nodes see the same order, not necessarily real-time order.
4. **Causal consistency** — causally related writes are seen in order; concurrent writes may differ. Achievable with available systems.
5. **Session guarantees:** read-your-writes, monotonic reads, monotonic writes, writes-follow-reads.
6. **Eventual consistency** — replicas converge if writes stop. No bound on staleness.

**Practical mapping:**
- Money balance, inventory decrement, unique usernames → linearizable on the key (or a serializable transaction).
- Like counts, feed ordering, view counters → eventual is fine.
- User profile after update → read-your-writes.

### 5.4.1 Linearizability costs
- Requires coordination (quorum or leader round trip) on every op. Latency floor = cross-region RTT when global.
- Not a free lunch: stale reads from followers break it. Reads must go to the leader or use read-index / lease reads.

---

## 5.5 CAP and PACELC (precise version)

- **CAP:** when a network **partition** happens, a system must pick **consistency** (refuse some requests to stay correct) or **availability** (answer, possibly stale). Only during partitions.
- **PACELC:** if **P**artition → A or C; **E**lse (normal) → **L**atency or **C**onsistency.
  - Dynamo/Cassandra: PA/EL (available, low latency; eventual).
  - Spanner: PC/EC (consistent; higher latency).
  - MongoDB default configs: depends on write concern and read preference.
- **Mistakes to avoid in interviews:** saying "pick 2 of 3" as a permanent choice; calling a system "CA" for a distributed deployment (CA only makes sense for a single node).
- **Good sentence:** "Partitions are rare but latency is always there, so I optimize the normal path for latency and accept eventual consistency for non-critical reads."

---

## 5.6 Consensus, Leader Election, Coordination

### 5.6.1 The problem
Get a group of nodes to agree on a value (or a sequence of values) despite crashes and message delays. Used for: leader election, config, membership, atomic commit, lock services.

### 5.6.2 Paxos
- Proposers, acceptors, learners. Two phases: prepare/promise, accept/accepted. Requires a majority.
- Correct but hard to implement and reason about (Multi-Paxos for logs).

### 5.6.3 Raft (understandable consensus)
- **Leader election:** nodes have terms; a follower times out, becomes candidate, requests votes; majority wins. Randomized timeouts avoid split votes.
- **Log replication:** leader appends entries and sends to followers; entry is **committed** when a majority stores it; applied to the state machine in order.
- **Safety:** a candidate only wins if its log is at least as up to date as a majority.
- **Membership changes** done with joint consensus or single-server changes.
- Used by: etcd, Consul, CockroachDB (per range), TiKV, Kafka KRaft (KRaft is Raft-based).

### 5.6.4 Coordination services
- **ZooKeeper:** ephemeral znodes (session-bound), watches, sequential nodes. Used for leader election, locks, config.
- **etcd:** key-value with watch and leases; backing store of Kubernetes.
- **Typical uses:** leader election, service discovery, distributed config, distributed locks (with fencing).

### 5.6.5 Distributed locks
- A lease-based lock: holder must renew; if it pauses (GC, network), the lease expires and another holder takes over.
- **Danger:** the old holder doesn't know it lost the lock and may write. **Fencing tokens** fix this: each lock grant has an increasing token; the resource (storage) rejects writes with lower tokens.
- **Redlock critique (Kleppmann):** using multiple Redis instances with timing assumptions is unsafe under clock jumps and pauses. Use a consensus-backed lock (etcd, ZooKeeper) with fencing for correctness-critical locks; Redis locks are fine for efficiency (avoid duplicate work), not correctness.

---

## 5.7 Failure Detection and Membership

- **Heartbeats** with timeouts: simple; false positives under load or GC pauses.
- **Phi accrual failure detector:** outputs a suspicion level based on heartbeat inter-arrival statistics (Cassandra, Akka).
- **Gossip protocols:** each node periodically exchanges state with random peers. Scales O(log N) rounds to spread information (Cassandra, Consul, Dynamo membership).
- **SWIM:** gossip-based membership with indirect pings.
- **Fencing:** a node that's been declared dead must be prevented from acting (fencing tokens, epoch numbers).
- **Timeouts design:** set per call, derived from p99 latency, with a retry budget.

---

## 5.8 Idempotency and Exactly-Once

### 5.8.1 Delivery semantics recap
- **At-most-once:** send and don't retry. May lose.
- **At-least-once:** retry until acked. May duplicate.
- **Exactly-once (effect):** at-least-once delivery + idempotent processing, or transactional atomic "process + record offset/ID".

### 5.8.2 Idempotency patterns
- **Idempotency key:** client-generated key per logical operation; server stores key → result (with TTL) and returns the stored result on retry. Must check-and-store atomically (unique constraint).
- **Natural idempotency:** `PUT` with full resource state, `SET x=5` not `x=x+1`.
- **Dedup table:** `processed_messages(message_id PK)` inserted in the same DB transaction as the business change.
- **Conditional writes:** `UPDATE ... WHERE version=?`.

### 5.8.3 Outbox pattern (publish events reliably)
1. In one local DB transaction: update business table and insert event into `outbox` table.
2. A relay (poller or CDC from the WAL via Debezium) reads outbox and publishes to Kafka, marks as sent.
3. Consumers are idempotent.
- Avoids the dual-write problem (DB commit succeeds, Kafka publish fails, or vice versa).

### 5.8.4 Inbox pattern
- Consumer records processed message IDs in the same transaction as its state change. Duplicates skipped.

---

## 5.9 Distributed Transactions

### 5.9.1 Two-phase commit (2PC)
- Coordinator asks participants to **prepare** (lock resources, write intent to log); if all vote yes, coordinator sends **commit**; else **abort**.
- **Pros:** atomic across resources.
- **Cons:** blocking. If coordinator crashes after prepare, participants are stuck holding locks until it recovers. Latency = 2 round trips + logging.
- Used inside some databases (XA transactions), Spanner (with Paxos groups), not common across microservices.

### 5.9.2 Saga
- Sequence of local transactions T1..Tn, each with compensation C1..Cn for rollback.
- **Choreography:** services react to each other's events. Decentralized; harder to follow.
- **Orchestration:** a coordinator (e.g., Temporal workflow) drives steps and compensations. Easier to reason about and monitor.
- **Trade-offs:** no isolation (intermediate states visible), compensations must be idempotent and sometimes semantic (refund, not "un-charge").
- **Design:** define the state machine, timeouts per step, retry policy, and what to do if a compensation fails (manual queue).

### 5.9.3 TCC (Try-Confirm-Cancel)
- Reserve resources in "try", finalize in "confirm" or release in "cancel". Used for payments and bookings. Requires each service to support reservation.

### 5.9.4 Choosing
- Same DB: local transaction.
- Few resources, one DB cluster: 2PC inside DB or a distributed SQL DB.
- Microservices: saga + outbox + idempotency. Accept eventual consistency and design user-visible "pending" states.

---

## 5.10 Logical Structures and Conflict Resolution

### 5.10.1 CRDTs (Conflict-free Replicated Data Types)
- Data types whose concurrent updates merge deterministically without coordination.
- **G-Counter:** grow-only counter; each node has its own slot; value = sum of max per node.
- **PN-Counter:** two G-Counters (increments and decrements).
- **G-Set / OR-Set:** add/remove sets (OR-Set uses unique tags).
- **LWW-Register:** last write wins by timestamp (simple, can lose updates).
- **Sequence CRDTs (RGA, Yjs, Automerge):** collaborative text editing.
- Use for: collaborative editing, offline-first apps, counters in multi-region systems.

### 5.10.2 Version vectors and sibling resolution
- Detect concurrency; keep siblings (Dynamo/Riak) and let the app merge, or apply LWW.

---

## 5.11 Distributed Systems Patterns Glossary

- **Leader and followers:** one coordinator, others replicate.
- **Heartbeat:** periodic liveness signal.
- **Quorum:** majority or W/R set.
- **Lease:** time-bound exclusive right; must renew.
- **Generation / epoch / term:** monotonic number to reject stale leaders.
- **Write-ahead log:** durability first.
- **Two-phase commit, saga:** atomic work across nodes.
- **Gossip:** epidemic information spread.
- **Consistent hashing:** minimal data moves on membership change.
- **Bulkhead, circuit breaker:** isolation and fail-fast (see Section 07 file).

---

## 5.12 Interview Questions on This Topic (short answers)

1. **How do you ensure only one leader?** Consensus (Raft/Paxos) with a lease, plus fencing tokens on the protected resource.
2. **Why not just use Redis locks?** Redis lock timing assumptions can allow two holders under pauses; fine for efficiency locks, not correctness. Use a consensus store with fencing for correctness.
3. **How do you handle replica lag for a user who just wrote?** Read from leader for a window, or use a version token and wait-for-version on read.
4. **What is the danger of sloppy quorums?** Reads can miss the latest write; it's availability over consistency.
5. **How do you migrate a shard key?** Dual-write or dual-read with backfill, verify with reconciliation, then cut over; keep a rollback path.
