# Open Inputs
1. What exact feature/change is this design for (business goal, user personas, and triggering events)?
2. What are the target traffic characteristics (steady-state TPS, peak TPS, and expected payload sizes)?
3. Which data stores and upstream/downstream systems are in scope, and what are their SLAs?
4. Are there compliance constraints (PII classes, retention mandates, audit requirements) beyond standard internal policy?
5. What rollout date and risk tolerance are expected (e.g., phased canary vs. full cutover)?

# Assumptions
- The change introduces a new backend capability named **Request Processing API** in an existing Java 18 + Spring Boot service.
- Authentication is OAuth2 JWT bearer token validation at the API gateway, with service-level authorization based on scopes.
- Primary datastore is PostgreSQL 14; Redis is available for low-latency idempotency key tracking.
- Deployment target is Kubernetes with horizontal pod autoscaling and centralized observability (Prometheus + Grafana + ELK/OpenSearch).
- The system must support 150 TPS steady state and 500 TPS peak with availability target of 99.95%.

# 1. Overview
This design adds a **Request Processing API** that accepts client requests, validates them, persists processing state, and executes asynchronous downstream handling with deterministic idempotency. The intent is to replace ad hoc request handling and reduce duplicate processing, timeout-driven failures, and operational triage load. The API exposes a synchronous acceptance response and moves heavy work to background workers through a durable queue. The design includes API contracts, persistence model, retry/dead-letter behavior, observability, and QA coverage mapping. The solution is backward compatible with existing clients by introducing a versioned endpoint and additive schema changes only. The implementation targets Java 18 and Spring Boot with clear package boundaries for API, domain, persistence, and worker modules.

Success criteria:
- P95 acceptance latency ≤ 120 ms and P99 ≤ 250 ms at 150 TPS.
- Duplicate effective processing rate < 0.1% using idempotency enforcement.
- End-to-end successful completion rate ≥ 99.9% excluding downstream 4xx business rejections.
- MTTR for production incidents reduced by 30% via actionable logs/alerts.

# 2. Background & Current State
Current behavior is synchronous request handling in a single execution path where input validation, database writes, and downstream invocation occur in one request thread. Failures in downstream dependencies frequently surface as client-facing timeouts. Duplicate client retries can trigger repeated side effects due to missing idempotency controls.

Current limitations:
- No formal idempotency key contract; retries can produce duplicate processing.
- Tight sync coupling to downstream availability causes elevated timeout/error rates during dependency degradation.
- Limited observability: logs do not consistently include request correlation IDs and state transitions.
- Manual incident triage is high due to insufficient error taxonomy and alert signal quality.

Constraints:
- Must remain compatible with existing OAuth2/JWT gateway model.
- Must use existing PostgreSQL and approved queue platform.
- SLA target for API availability is 99.95% monthly.
- PII must be encrypted at rest and redacted in logs.

# 3. Problem Definition
Functional problem:
- The system lacks deterministic request lifecycle management (accept, process, complete/fail) with idempotent behavior.

Non-functional problem:
- Synchronous dependency coupling and weak retry policies create unstable latency and elevated failure rates under partial outages.

Observable symptoms:
- Spikes in client timeout rate during downstream latency events.
- Duplicate side effects after client retries.
- High operational toil from ambiguous error logs and missing state progression metrics.

# 4. Scope
## 4.1 In Scope
- New versioned endpoint for request submission and status retrieval.
- Idempotency key support with deduplication semantics.
- Durable asynchronous processing workflow (queue + worker).
- Request state machine persisted in PostgreSQL.
- Observability instrumentation (logs/metrics/traces/alerts).
- QA plan including failure injection and performance validation.

## 4.2 Out of Scope
- Re-architecting existing gateway or identity provider.
- Changes to downstream business logic semantics.
- Multi-region active-active deployment.
- UI/portal changes.

# 5. Functional Scenarios
## Scenario: New request accepted and processed successfully
### Normal Flow
1. Client sends `POST /v1/requests` with idempotency key.
2. API validates schema/authz and records `ACCEPTED` state atomically.
3. API enqueues processing message and returns `202 Accepted` with `requestId`.
4. Worker consumes message, executes business logic, writes `COMPLETED` state.
5. Client polls `GET /v1/requests/{requestId}` and receives final status.

### Alternate Flows
- If the same idempotency key and identical payload arrive within TTL, API returns original `requestId` and current status.
- If request is accepted but not yet processed, status remains `IN_PROGRESS`.

### Failure Flows
- Queue publish failure after DB write: transactionally record `PENDING_ENQUEUE` and reconciliation job retries enqueue.
- Worker transient dependency failure: bounded retries with exponential backoff.
- Exhausted retries: move to DLQ and mark request `FAILED_RETRY_EXHAUSTED`.

### Notes (idempotency/state/edge cases)
- Payload hash is part of idempotency key uniqueness to prevent key reuse with modified payload.
- Status endpoint is read-only and safe for aggressive polling.
- Clock skew handled by server-side timestamps only.

## Scenario: Duplicate request with conflicting payload
### Normal Flow
1. Client reuses existing idempotency key with different payload.
2. API detects hash mismatch and rejects request.

### Alternate Flows
- None.

### Failure Flows
- DB unavailable during idempotency lookup returns `503` retryable error.

### Notes (idempotency/state/edge cases)
- Conflict response must include deterministic error code for client remediation.

## Scenario: Downstream partial outage
### Normal Flow
1. Requests continue to be accepted while queue/DB healthy.
2. Worker retries downstream calls with backoff.

### Alternate Flows
- Circuit breaker opens after failure threshold; requests remain queued until half-open probe succeeds.

### Failure Flows
- Sustained outage pushes messages to DLQ; alert triggers on DLQ depth.

### Notes (idempotency/state/edge cases)
- Preserve order only per `requestId`; global ordering is not guaranteed.

# 6. Non-Functional Requirements
- TPS: 150 expected, 500 peak for 15-minute bursts.
- Latency: API acceptance P95 ≤ 120 ms, P99 ≤ 250 ms; status query P95 ≤ 80 ms.
- Availability target: 99.95% monthly for API acceptance and status endpoints.
- Data retention/TTL: request records retained 90 days; idempotency keys retained 24 hours.
- Security constraints: OAuth2 JWT validation, scope-based authorization, TLS 1.2+, encrypted storage.
- Operational constraints:
  - API timeout: 2 seconds.
  - Worker downstream timeout: 1 second connect / 3 seconds read.
  - Retry policy: max 5 attempts, exponential backoff with jitter (200 ms to 30 s).

# 7. High-Level Architecture
Component responsibilities:
- API Service: validation, authz checks, idempotent acceptance, status reads.
- PostgreSQL: source of truth for request lifecycle and idempotency metadata.
- Message Queue: decouples ingestion from execution.
- Worker Service: executes downstream processing and updates state.
- Reconciler: re-enqueues stranded `PENDING_ENQUEUE` requests.

Data/control flow:
1. Client → API (`POST`)
2. API → DB transaction (`request`, `idempotency_record`)
3. API → Queue publish (or mark for reconciliation)
4. Worker ← Queue consume → downstream call
5. Worker → DB state update
6. Client → API (`GET` status)

Sync vs async boundaries:
- Synchronous: request validation and acceptance persistence.
- Asynchronous: business processing and downstream interactions.

```mermaid
flowchart LR
    C[Client] -->|POST /v1/requests| A[API Service]
    A --> D[(PostgreSQL)]
    A --> Q[(Message Queue)]
    Q --> W[Worker]
    W --> X[Downstream Service]
    W --> D
    C -->|GET /v1/requests/{id}| A
```

# 8. Detailed Technical Design
## 8.1 Component Design (packages/classes/responsibilities)
- `com.company.request.api`
  - `RequestController`: REST endpoints.
  - `RequestExceptionHandler`: maps exceptions to error model.
- `com.company.request.service`
  - `RequestCommandService`: validate + persist + enqueue.
  - `RequestQueryService`: status retrieval.
  - `IdempotencyService`: key/hash validation and lookup.
- `com.company.request.persistence`
  - `RequestEntity`, `IdempotencyEntity`.
  - `RequestRepository`, `IdempotencyRepository`.
- `com.company.request.worker`
  - `RequestWorker`: consumes queue messages.
  - `RetryPolicyProvider`: backoff policy.
  - `DownstreamClient`: outbound integration.
- `com.company.request.reconciler`
  - `PendingEnqueueReconciler`: scheduled job for stranded messages.

## 8.2 API Design (if applicable)
Endpoints:
- `POST /v1/requests`
- `GET /v1/requests/{requestId}`

Request/response examples (Java 18 records):
```java
public record CreateRequest(
    String idempotencyKey,
    String requestType,
    Map<String, Object> payload
) {}

public record CreateResponse(
    String requestId,
    String status,
    Instant acceptedAt
) {}

public record StatusResponse(
    String requestId,
    String status,
    String failureCode,
    String failureMessage,
    Instant updatedAt
) {}
```

Error model + codes:
- `400 INVALID_REQUEST`
- `401 UNAUTHENTICATED`
- `403 UNAUTHORIZED_SCOPE`
- `409 IDEMPOTENCY_PAYLOAD_CONFLICT`
- `429 RATE_LIMITED`
- `503 DEPENDENCY_UNAVAILABLE`

Idempotency strategy:
- Require `idempotencyKey` per create call.
- Store `(tenant_id, idempotency_key, payload_hash)` unique tuple.
- Return prior response for matching tuple within 24-hour TTL.
- Reject same key with different hash using `409`.

## 8.3 Data Design
Schema changes:
- `request_record` table:
  - `request_id` (UUID PK)
  - `tenant_id` (VARCHAR)
  - `status` (ENUM)
  - `payload_json` (JSONB)
  - `failure_code`, `failure_message`
  - `created_at`, `updated_at`
- `idempotency_record` table:
  - `tenant_id`, `idempotency_key`, `payload_hash`, `request_id`, `expires_at`

Index strategy:
- Unique index: `(tenant_id, idempotency_key)`.
- Lookup index: `request_record(tenant_id, request_id)`.
- Worker scan index: `request_record(status, updated_at)` for reconciliation.

Transaction boundaries:
- Single transaction for request + idempotency insert + initial state write.
- Queue publish is out-of-transaction; failures handled via reconciliation.

Cache strategy:
- Optional Redis cache for hot status reads with 30-second TTL.
- Cache invalidation on status transition events.

## 8.4 Concurrency & State Management
- State transitions enforced by optimistic locking (`version` column).
- Allowed transitions: `ACCEPTED -> IN_PROGRESS -> COMPLETED|FAILED_*`.
- Worker ensures at-least-once processing; idempotent downstream token guards side effects.
- Retry strategy:
  - Exponential backoff with jitter.
  - Max 5 attempts before DLQ.
- Race conditions:
  - Duplicate worker delivery resolved by compare-and-set update on expected state.

## 8.5 Error Handling Strategy
Exception taxonomy:
- `ValidationException` (non-retryable, client-correctable)
- `AuthorizationException` (non-retryable)
- `DependencyTransientException` (retryable)
- `DependencyPermanentException` (non-retryable)
- `PersistenceException` (retryable for transient SQL states)

Retryable vs non-retryable:
- Retryable: network timeouts, HTTP 5xx, deadlocks.
- Non-retryable: schema validation errors, unauthorized scope, 4xx business rejects.

Timeout handling:
- API layer fast-fail at 2 seconds.
- Worker downstream timeout budgets enforced by client config and circuit breaker.

# 9. Security Considerations
- AuthN/AuthZ:
  - Validate JWT signature and expiration at gateway.
  - Enforce service-level scopes: `requests:create`, `requests:read`.
- Signature validation (if external):
  - For webhook-style downstream callbacks, require HMAC SHA-256 signature verification.
- PII handling and log redaction:
  - Encrypt PII columns with KMS-managed keys.
  - Redact payload fields marked sensitive in structured logs.
  - Prohibit raw payload logging at `INFO` level.

# 10. Observability
- Structured logging + correlation IDs:
  - Include `traceId`, `requestId`, `idempotencyKey`, `tenantId`, `statusTransition`.
- Metrics:
  - `request_accept_qps`
  - `request_accept_latency_ms` (P50/P95/P99)
  - `worker_processing_latency_ms`
  - `worker_retry_count`
  - `dlq_depth`
  - `idempotency_conflict_rate`
- Alerts:
  - P95 acceptance latency > 120 ms for 10 min.
  - Error rate > 1% over 5 min.
  - DLQ depth > 100 for 15 min.
  - Reconciliation backlog age > 10 min.

# 11. Test Strategy
- Unit test scope:
  - Validation rules, idempotency matching/conflict logic, state transition guards, exception mapping.
- Integration test scope:
  - API + DB transaction semantics, queue publish/reconciliation, worker retry + DLQ behavior.
- Performance test plan:
  - Load at 150 TPS for 60 min and burst at 500 TPS for 15 min; verify latency/error SLOs.
- Failure injection plan:
  - Inject downstream 5xx/timeouts, DB failover events, queue publish interruptions.

Scenario-to-Test coverage matrix:

| Scenario | Unit Tests | Integration Tests | Performance Tests | Failure Injection |
|---|---|---|---|---|
| New request accepted and processed successfully | Validation + state transition tests | End-to-end accept/process/status flow | Baseline TPS/latency test | N/A |
| Duplicate request with conflicting payload | Idempotency hash conflict logic | API returns 409 with deterministic code | Conflict behavior under load | DB lookup failure simulation |
| Downstream partial outage | Retry classifier + backoff policy | Retries, circuit breaker, DLQ transition | Throughput under degraded dependency | Inject 5xx + timeout faults |

# 12. Migration / Rollout Plan (if applicable)
- Backward compatibility:
  - Introduce `/v1` endpoint without altering existing endpoints.
  - Additive DB migrations with nullable new columns and online index creation.
- Deployment approach:
  - Phase 1: deploy code with feature flag off.
  - Phase 2: enable for 5% tenants (canary).
  - Phase 3: ramp to 50%, then 100% after SLO confirmation.
- Rollback plan:
  - Disable feature flag immediately.
  - Stop worker consumers.
  - Keep accepted records for replay after recovery.
- Feature flags:
  - `request.processing.enabled`
  - `request.reconciler.enabled`

# 13. Risks & Mitigation
| Risk | Impact | Mitigation | Owner |
|---|---|---|---|
| Queue backlog growth during downstream outage | Increased completion latency, potential SLA miss | Auto-scale workers, enforce DLQ thresholds, outage runbook | SRE |
| Idempotency key misuse by clients | Elevated 409/confusion, support load | Publish client contract + SDK helper, observability dashboard for conflicts | API Team |
| Reconciler misconfiguration | Stranded accepted requests | Heartbeat metric + alert, integration tests for reconciliation | Backend Team |
| DB hot partition on tenant-heavy workloads | Latency degradation | Composite indexing, query plan review, partitioning strategy if growth exceeds threshold | Data Platform |

# 14. Open Questions
- Should status retrieval support push callbacks/webhooks in addition to polling?
- Is Redis approved as mandatory dependency for idempotency acceleration, or should DB-only remain default?
- What tenant-level rate limits should be enforced at gateway vs service layer?
- What is the required retention for failed payload forensic data beyond 90 days?
- Do we need strict per-tenant ordering guarantees for subsets of request types?
