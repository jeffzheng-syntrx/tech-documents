# 1. Overview
- What is being built/changed (5–10 lines)
- What success looks like (measurable)

# 2. Background & Current State
- Current flow/system behavior
- Current limitations (with evidence/metrics if available)
- Constraints (infra/vendor/SLA/compliance)

# 3. Problem Definition
- Functional problem
- Non-functional problem
- Observable symptoms (e.g., error rates, latency, ops pain)

# 4. Scope
## 4.1 In Scope
## 4.2 Out of Scope

# 5. Functional Scenarios
For each scenario:
## Scenario: <name>
### Normal Flow
### Alternate Flows
### Failure Flows
### Notes (idempotency/state/edge cases)

# 6. Non-Functional Requirements
- TPS (expected/peak)
- Latency (P95/P99)
- Availability target
- Data retention / TTL
- Security constraints
- Operational constraints (timeouts/retries, etc.)

# 7. High-Level Architecture
- Component responsibilities
- Data/control flow
- Sync vs async boundaries
- Include Mermaid diagrams when helpful

# 8. Detailed Technical Design
## 8.1 Component Design (packages/classes/responsibilities)
## 8.2 API Design (if applicable)
- Endpoints, request/response (use Java 18 `record` in examples)
- Error model + codes
- Idempotency strategy
## 8.3 Data Design
- Schema changes
- Index strategy
- Transaction boundaries
- Cache strategy (if any)
## 8.4 Concurrency & State Management
- Ordering guarantees
- race-condition handling
- retry strategy
## 8.5 Error Handling Strategy
- Exception taxonomy
- retryable vs non-retryable
- timeout handling

# 9. Security Considerations
- AuthN/AuthZ
- Signature validation (if external)
- PII handling and log redaction

# 10. Observability
- Structured logging + correlation IDs
- Metrics (QPS/latency/errors)
- Alerts (thresholds)

# 11. Test Strategy
- Unit test scope
- Integration test scope
- Performance test plan
- Failure injection plan
- Provide a **Scenario-to-Test coverage matrix**

# 12. Migration / Rollout Plan (if applicable)
- Backward compatibility
- Deployment approach
- Rollback plan
- Feature flags

# 13. Risks & Mitigation
- Table: Risk | Impact | Mitigation | Owner

# 14. Open Questions
- Unresolved decisions / dependencies
