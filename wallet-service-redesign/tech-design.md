# 1. Problem Statement & Context

## 1.1 Background
The current wallet platform evolved around split persistence responsibilities: balances are written to Couchbase, while ledger records are persisted in MySQL through an asynchronous Kafka pipeline. This design has supported baseline wallet operations but was not built for strict financial correctness under sustained high-throughput contention (hot wallets) and multi-vendor transactional patterns.

Wallet Service v1 introduces a new internal transactional core (Java 21, Spring Boot 3.x, stateless microservice) that supports `debit`, `credit`, `reverse`, and `balance` operations with caller-supplied idempotency keys (`transaction_id`, UUIDv7 recommended). The service is designed for high TPS, horizontal scaling, and audit-grade traceability.

## 1.2 Current Architecture (As-Is)
Current write flow:
1. Wallet balance is updated in Couchbase (effective source of truth for available balance).
2. A ledger event is emitted.
3. Kafka delivers the event asynchronously.
4. Ledger entry is inserted into MySQL by downstream processing.

Current read behavior:
- Balance checks primarily depend on Couchbase state.
- Ledger queries and reconciliation depend on MySQL completeness and Kafka delivery correctness.

Architectural properties today:
- **Split source of truth:** balance correctness depends on Couchbase, transaction history depends on MySQL.
- **Asynchronous financial recording:** ledger persistence is decoupled from the balance commit.
- **Kafka dependency in correctness path:** failures or lag in Kafka materially affect financial observability and reconciliation.

## 1.3 Problems With Current Design
1. **Eventual consistency gap for money movement**
   - Balance can be committed before its corresponding ledger record exists.
   - During lag/failure windows, reads and audits can observe incomplete financial state.

2. **Split source of truth creates reconciliation risk**
   - Operational truth (balance) and accounting truth (ledger) live in different systems with different consistency guarantees.
   - Recovery and dispute resolution require expensive reconciliation workflows.

3. **Kafka in write correctness path is fragile for financial systems**
   - Broker unavailability, partition lag, consumer replay bugs, and ordering edge cases can delay or distort ledger visibility.
   - A messaging platform should support integration/eventing, not determine whether core accounting state is durable.

4. **Hot-wallet contention is under-specified**
   - High-frequency updates against the same wallet can cause race conditions, retry storms, and nondeterministic outcomes without explicit concurrency control.

5. **Auditability and determinism are insufficient**
   - Asynchronous split writes complicate deterministic replay, lineage, and regulator-grade evidence generation.

## 1.4 Goals of Wallet Service v1
1. **Single transactional authority for wallet state**
   - Remove Couchbase from wallet responsibility.
   - Establish a single durable system where balance and ledger are co-managed.

2. **Atomic financial commit semantics**
   - Commit balance mutation and immutable ledger append in the same ACID transaction.
   - Ensure no successful balance update exists without its corresponding ledger record.

3. **Immutable ledger model**
   - Enforce append-only ledger entries; reversals are compensating entries, never in-place edits.

4. **Deterministic concurrency for hot wallets**
   - Implement explicit concurrency control to preserve correctness under high-frequency same-wallet traffic.

5. **Idempotent, integration-ready API behavior**
   - Guarantee exactly-once effect per caller `transaction_id` under retries/timeouts.
   - Support multi-vendor integrations with predictable error semantics.

6. **Operationally scalable internal platform**
   - Maintain stateless service nodes with horizontal scaling.
   - Provide audit-grade logging, traceability, and observability for incident response and compliance.

## 1.5 Target Architectural Shift Summary
| Dimension | Current | Target (Wallet Service v1) |
|---|---|---|
| Balance source of truth | Couchbase | Transactional wallet store (Couchbase removed from wallet responsibility) |
| Ledger persistence | MySQL via async Kafka insert | Synchronous, in-transaction append to immutable ledger |
| Write-path dependency | Kafka required for eventual ledger correctness | Kafka deprecated in write path (may remain for downstream async consumers only) |
| Consistency model | Eventual consistency between balance and ledger | Atomic commit: balance + ledger persisted together |
| Concurrency model | Implicit/implementation-dependent | Deterministic control for hot-wallet contention |
| Idempotency | Partial/flow-dependent | Required first-class behavior keyed by caller `transaction_id` |
| Audit posture | Cross-system stitching required | Native transaction lineage with audit-grade logging |

## 1.6 Non-Goals (v1)
- Building customer-facing wallet UI or direct end-user channels.
- Supporting real-time cross-region active-active writes in v1.
- Replacing all downstream analytics/reporting pipelines immediately.
- Introducing external settlement, FX conversion, or treasury orchestration.
- Redesigning partner onboarding workflows beyond required API contract alignment.

## 1.7 Key Constraints
- **Technology baseline:** Java 21, Spring Boot 3.x, stateless microservice deployment model.
- **Operation set:** `debit`, `credit`, `reverse`, `balance` must be supported from day one.
- **Throughput profile:** high TPS transactional load, including hot-wallet traffic patterns.
- **Idempotency contract:** caller-supplied `transaction_id` is mandatory (UUIDv7 recommended).
- **Ledger invariants:** immutable append-only ledger; correction via compensating entries only.
- **Correctness invariant:** balance and ledger must commit atomically in a single transaction boundary.
- **Scalability:** horizontal service scaling without sacrificing wallet-level correctness.
- **Audit/compliance:** audit-grade structured logs with traceable request-to-ledger lineage.
- **System boundary:** internal platform service (not directly customer-facing), but must provide strong integration contracts for multiple vendor systems.
