# 07. Deep APIs, Resilience, Observability, Security, Deployment

---

## 7.1 API Design

### 7.1.1 REST
- Resources as nouns (`/orders/{id}/items`), HTTP verbs with semantics:
  - `GET` safe and idempotent; `PUT` idempotent full replace; `PATCH` partial; `DELETE` idempotent; `POST` not idempotent unless keyed.
- Status codes: 200/201/204; 400 bad input; 401 unauthenticated; 403 forbidden; 404; 409 conflict (version mismatch); 412 precondition failed (ETag); 422 semantic; 429 rate limited (+`Retry-After`); 500/503/504.
- **Pagination:**
  - Offset/limit: simple; slow on deep pages; unstable under inserts (duplicates/skips).
  - Cursor/keyset: `WHERE (created_at, id) < (?, ?) ORDER BY … LIMIT n`; stable and fast with an index.
- **Filtering, sorting:** whitelist fields; avoid arbitrary SQL.
- **Partial responses / field selection** for mobile.
- **Versioning:** URL (`/v1`), header, or content negotiation. Prefer additive changes; never remove or rename fields without a version bump.
- **Conditional requests:** `ETag`/`If-Match` for optimistic concurrency (lost update prevention).
- **Idempotency-Key header** on POST for payments and creates.
- **Long-running operations:** return `202 Accepted` with a status resource or webhook.

### 7.1.2 gRPC
- Protobuf schemas, HTTP/2, streaming (unary, server, client, bidirectional), strong typing, codegen.
- Use for internal service-to-service with low latency and high throughput. Harder to use from browsers (needs gRPC-Web).
- Schema evolution: never reuse field numbers; add optional fields; deprecate, don't remove.
- Deadlines propagate across calls (deadline budget).

### 7.1.3 GraphQL
- Client specifies fields; single endpoint; typed schema.
- Good for BFF and many client shapes; reduces over-fetching.
- **Risks:** N+1 queries (fix with DataLoader batching), expensive nested queries (limit depth, complexity analysis, persisted queries), caching is harder (no HTTP GET caching by default), authorization must be per-field.

### 7.1.4 Webhooks and events
- Push notifications to customer URLs: sign payloads (HMAC), include event ID and timestamp, retry with exponential backoff, expose delivery logs, allow replay.
- Consumers must be idempotent (events can repeat).

### 7.1.5 API gateway responsibilities
- AuthN (token validation), rate limiting, routing, TLS termination, request/response transforms, caching, metrics and logs, WAF, quota management, versioned routing.

### 7.1.6 Backward compatibility rules
- Safe changes: adding optional fields, adding endpoints, adding enum values (if clients tolerate unknown values).
- Breaking: removing/renaming fields, changing types, changing semantics, making optional required.
- Use consumer-driven contract tests (Pact) to catch breaks.

---

## 7.2 Resilience Patterns

### 7.2.1 Timeouts
- Every remote call has a timeout. Without it, a slow dependency consumes all threads.
- Derive from p99 of the dependency plus margin; propagate remaining deadline to downstream calls.

### 7.2.2 Retries
- Retry only idempotent operations, or ones with idempotency keys.
- **Exponential backoff with full jitter:** `sleep = random(0, min(cap, base * 2^attempt))`.
- **Retry budget:** cap retries to a fraction of total traffic (e.g., 10%) to avoid retry storms.
- Retry on transient errors (timeouts, 503, 429 with Retry-After), not on 400/401/403/404.
- Retries multiply load: 3 layers × 3 retries = 27× traffic. Retry at one layer, usually the edge or the caller.

### 7.2.3 Circuit breaker
- States: **Closed** (calls pass; count failures) → **Open** (fail fast for a cool-down) → **Half-open** (allow a few trial calls; success closes, failure reopens).
- Thresholds: failure rate over a sliding window (e.g., >50% over 20 calls).
- Pair with fallbacks: cached response, default value, degraded feature.

### 7.2.4 Bulkhead
- Isolate resources per dependency: separate thread pools, connection pools, or semaphores. One slow dependency can't starve others.

### 7.2.5 Load shedding and backpressure
- Reject early when overloaded (503 or 429), prioritize critical traffic.
- Queue with bounded size; drop oldest or reject newest; use adaptive concurrency limits (e.g., gradient-based limiter based on latency).
- Backpressure from consumers to producers: bounded buffers, pull-based consumption, rate-limited producers.

### 7.2.6 Graceful degradation
- Show cached data, hide non-critical widgets, reduce quality (lower image resolution), fall back to a simpler algorithm.
- Define which features are critical per product (e.g., checkout > recommendations).

### 7.2.7 Health checks, graceful shutdown, and startup
- Liveness (process is alive), readiness (ready for traffic), startup probe for slow boot.
- On SIGTERM: fail readiness, drain connections, finish in-flight work, then exit.

### 7.2.8 Chaos and game days
- Inject failures (kill nodes, add latency, partition networks) in staging and controlled production. Verify timeouts, retries, failover, and alerts behave as designed.

---

## 7.3 Rate Limiting and Quotas (depth beyond HLD guide)

- Limit dimensions: per user, per API key, per IP, per endpoint, per tenant (fairness).
- **Global vs local:** local per-node limits are cheap but sum to more than the global limit; global needs shared state (Redis) with latency cost. Hybrid: local token buckets with periodic sync.
- **Multi-tenant fairness:** weighted fair queuing; per-tenant queues.
- **Quota vs rate:** quotas are long-window totals (monthly); rate limits are short-window (per second).
- Communicate limits: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`.

---

## 7.4 Observability

### 7.4.1 Three pillars plus
- **Metrics:** numeric time series (counters, gauges, histograms). Cheap; aggregatable; use for alerts and dashboards.
- **Logs:** discrete events with context. Structured (JSON) with request ID, user/tenant, latency, error. Expensive at scale; sample high-volume logs.
- **Traces:** a request's path across services as spans with timing and parent-child links (OpenTelemetry standard). Essential for microservice latency debugging.
- **Profiles:** CPU/memory profiles in production (continuous profiling).

### 7.4.2 Metric types and what to measure
- **RED** for request-driven services: Rate, Errors, Duration (p50, p95, p99).
- **USE** for resources: Utilization, Saturation (queue depth), Errors.
- **Golden signals:** latency, traffic, errors, saturation.
- **Histograms over averages:** averages hide tails. Use percentiles from histograms; be careful averaging percentiles across hosts (aggregate histogram buckets instead).
- **Cardinality:** labels like `user_id` explode storage. Use bounded labels (route template, not raw path).

### 7.4.3 SLI, SLO, SLA, error budgets
- **SLI:** measured indicator (fraction of requests < 300 ms and successful).
- **SLO:** target (99.9% over 30 days).
- **SLA:** contract with consequences.
- **Error budget:** 1 − SLO. With 99.9%, about 43 minutes of full downtime per 30 days. Spend on releases and experiments; freeze when exhausted.
- **Burn-rate alerts:** alert when budget is being consumed faster than sustainable (multi-window, e.g., 14× over 1 h and 6× over 6 h).

### 7.4.4 Alerting philosophy
- Alert on symptoms users feel (SLO burn, error rate, latency), not every cause (CPU 80%).
- Every page should be actionable and have a runbook.
- Reduce noise: deduplicate, group, route by ownership.

### 7.4.5 Distributed tracing details
- Propagate `traceparent` (W3C Trace Context) through HTTP headers and message headers (Kafka record headers) to link async hops.
- Sampling: head-based (decide at start) vs tail-based (decide after seeing errors/latency).

### 7.4.6 Incident management
- Severity levels, incident commander, communication cadence, status page.
- Postmortem: blameless, timeline, root cause, contributing factors, action items with owners.

---

## 7.5 Security

### 7.5.1 Authentication and authorization
- **AuthN:** password (bcrypt/argon2/scrypt with per-user salt, never plaintext or fast hashes), MFA, passkeys/WebAuthn, SSO via SAML/OIDC.
- **OAuth 2.0:** delegated authorization. Flows: authorization code with PKCE (for web/mobile), client credentials (service-to-service). Implicit flow deprecated.
- **OpenID Connect (OIDC):** identity layer on OAuth2; ID token (JWT) says who the user is.
- **Access tokens:** short-lived (minutes), refresh tokens long-lived and rotated, revocable.
- **JWT:** signed (JWS) claims (`sub`, `exp`, `aud`, `iss`, roles). Validate signature, `exp`, `aud`, `iss`; prefer asymmetric keys (RS256/ES256); don't put secrets in payload (only base64 encoded). Revocation is hard for stateless tokens; mitigate with short TTL or a denylist.
- **Session cookies:** HttpOnly, Secure, SameSite; server-side session store allows instant revocation.
- **AuthZ models:** RBAC (roles), ABAC (attributes), ReBAC (relationships, Zanzibar/OpenFGA style). Enforce at service boundary, not only at gateway.
- **Keycloak** (your resume): OIDC/OAuth2 provider with realms, roles, and client scopes.

### 7.5.2 Service-to-service security
- **mTLS:** both sides present certificates; identity by SPIFFE IDs (service mesh like Istio/Linkerd).
- **Zero trust:** verify every request; no implicit trust from the internal network.
- **Secrets management:** Vault, cloud KMS/Secrets Manager; rotate; never in code or images.

### 7.5.3 Data protection
- **In transit:** TLS 1.2+ (prefer 1.3).
- **At rest:** AES-256 with envelope encryption (data keys encrypted by a KMS master key).
- **PII:** minimize collection, tokenize, restrict access, audit.
- **Compliance:** GDPR (right to erasure → data deletion pipelines and backups policy), PCI DSS (don't store card data; use tokenization via processor), HIPAA.

### 7.5.4 Common attacks and defenses
| Attack | Defense |
|---|---|
| SQL injection | Parameterized queries, ORM, least-privilege DB user |
| XSS | Output encoding, CSP header, HttpOnly cookies |
| CSRF | SameSite cookies, CSRF tokens, custom headers |
| SSRF | Allowlist outbound hosts, block metadata IPs |
| Broken access control (IDOR) | Check ownership on every object access; don't trust IDs |
| DDoS | Edge scrubbing, CDN, rate limits, SYN cookies, autoscaling buffers |
| Credential stuffing | Rate limits, MFA, breached-password checks |
| Replay attacks | Nonces, timestamps, idempotency keys |
| Supply chain | Dependency scanning, signed images, SBOM |

### 7.5.5 Payload signing and webhooks
- HMAC-SHA256 over body with shared secret and timestamp; reject old timestamps (replay window).

---

## 7.6 Deployment and Release Engineering

- **Immutable artifacts:** build once, promote through environments.
- **Blue-green:** two full environments; switch traffic. Instant rollback; double cost during switch.
- **Canary:** route 1% → 10% → 50% → 100% with automated metric checks (error rate, latency) and auto rollback.
- **Rolling update:** replace instances in batches; needs readiness and backward-compatible versions running side by side.
- **Feature flags:** decouple deploy from release; kill switches; percentage rollouts; cleanup flags after launch.
- **Database migrations:** expand → migrate → contract. Never deploy code that needs a schema change without the schema being ready first. Avoid long locks (online DDL, batched backfills).
- **Config:** externalized, versioned, hot-reloadable where safe.
- **Infrastructure as code:** Terraform, Helm, GitOps (Argo CD).

---

## 7.7 Multi-Region and Disaster Recovery

- **RTO:** how long to recover. **RPO:** how much data loss is acceptable.
- **Backup and restore:** cheapest, highest RTO/RPO. Test restores.
- **Pilot light / warm standby / hot active-active:** increasing cost and decreasing RTO.
- **Active-passive:** one region serves; failover on incident. Simpler, DNS switch or LB switch.
- **Active-active:** all regions serve; requires conflict handling (partitioned ownership by user/tenant, CRDTs, or global DB like Spanner/CockroachDB). Use **geo-partitioning**: each user's home region owns their data.
- **Cell-based architecture:** split the system into independent cells, each serving a subset of users. Blast radius limited to one cell.
- **Data residency:** keep EU data in EU; route accordingly.
- **Global data:** strongly consistent global writes are expensive (cross-region quorum latency). Keep most writes regional.

---

## 7.8 Cost Awareness (say this in senior interviews)

- Biggest levers: storage tiering (hot/warm/cold; S3 Glacier), caching (reduce DB/compute), right-sizing, autoscaling, reserved/spot capacity, compression, data retention policies, egress costs (CDN and cross-region transfer are often the hidden cost).
- Trade-off framing: "This design costs ~X per month; I'd reduce it by Y by moving Z to cold storage."

---

## 7.9 Testing for Distributed Systems

- Unit tests for logic; integration tests with real dependencies (Testcontainers).
- Contract tests between services (Pact).
- Load testing (k6, Gatling) against realistic traffic shapes; find the knee of the latency curve.
- Fault injection (Toxiproxy, Chaos Mesh, Gremlin).
- Property-based and model-based testing for state machines.
- Jepsen-style testing for consistency claims (understand the concept; it checks linearizability under partitions).

---

## 7.10 Interview Questions (short answers)

1. **How do you prevent a retry storm?** Retry budget, exponential backoff with jitter, circuit breakers, retry only at one layer, respect Retry-After.
2. **Access token is stolen. What now?** Short TTL limits window; revoke refresh token; use token binding or DPoP; detect anomalies; force re-auth for sensitive actions.
3. **Design an SLO for checkout.** SLI: fraction of checkout requests succeeding under 1 s. SLO: 99.9% over 28 days. Error budget ~40 min. Alert on burn rates.
4. **Canary shows higher p99 but same error rate. Action?** Investigate latency regression via traces; compare by version; roll back if p99 breaches SLO.
5. **How do you migrate a column type on a live table?** Expand (add new column), dual write, backfill in batches, switch reads, stop writes to old, contract (drop old). Keep rollback.
6. **Active-active across regions for a payment system?** Usually not for the ledger. Use regional ownership per account, async replication for read models, and a strongly consistent store for balances.
