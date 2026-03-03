SOX & GDPR Compliant Microservice — Full Text Architecture Diagram
----

```
╔══════════════════════════════════════════════════════════════════════════════════════════╗
║              SOX & GDPR COMPLIANT JAVA SPRING BOOT MICROSERVICE ARCHITECTURE            ║
╚══════════════════════════════════════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 1 — CLIENT LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

        ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
        │   Web Browser   │   │  Mobile App     │   │ External System │
        │  (React / Vue)  │   │ (iOS / Android) │   │  (B2B Partner)  │
        └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
                 │                     │                      │
                 │   HTTPS / TLS 1.3   │                      │
                 └─────────────────────┴──────────────────────┘
                                       │
                                       ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 2 — API GATEWAY LAYER  (Single Entry Point)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

        ┌─────────────────────────────────────────────────────────────┐
        │                    API GATEWAY                              │
        │              (Spring Cloud Gateway)                         │
        │                                                             │
        │  ┌──────────────────┐   ┌──────────────────┐               │
        │  │ CorrelationId    │──▶│ JWT Validation   │               │
        │  │ Filter  [Order=0]│   │ Filter [Order=1] │               │
        │  │                  │   │                  │               │
        │  │ • Generates or   │   │ • Validates JWT  │               │
        │  │   accepts        │   │   signature      │               │
        │  │   X-Correlation  │   │ • Extracts roles │               │
        │  │   -Id header     │   │   & userId       │               │
        │  │ • Adds to        │   │ • Injects into   │               │
        │  │   response too   │   │   request header │               │
        │  └──────────────────┘   └────────┬─────────┘               │
        │                                  │                          │
        │                    ┌─────────────▼──────────────┐          │
        │                    │   Route Config              │          │
        │                    │                             │          │
        │                    │  /api/users/**  ──────────▶ user-svc  │
        │                    │  /api/finance/** ─────────▶ fin-svc   │
        │                    │  /api/audit/**  ──────────▶ audit-svc │
        │                    └─────────────────────────────┘          │
        │                                                             │
        │  [Rate Limiting]  [SSL Termination]  [Load Balancing]       │
        └──────────────────────────┬──────────────────────────────────┘
                                   │
                         mTLS (mutual TLS)
                    [Only trusted services pass]
                                   │
           ┌───────────────────────┼────────────────────────┐
           │                       │                        │
           ▼                       ▼                        ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 3 — MICROSERVICES LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────────────────┐
 │      USER SERVICE       │  │   FINANCIAL SERVICE     │  │     AUDIT SERVICE       │
 │  (GDPR Domain)          │  │   (SOX Domain)          │  │  (SOX + GDPR Domain)    │
 │                         │  │                         │  │                         │
 │  ┌───────────────────┐  │  │  ┌───────────────────┐  │  │  ┌───────────────────┐  │
 │  │  UserController   │  │  │  │ TxnController     │  │  │  │ AuditEventConsumer│  │
 │  │  DataSubject      │  │  │  │                   │  │  │  │                   │  │
 │  │  Controller       │  │  │  │ POST /transactions│  │  │  │ Kafka Listener:   │  │
 │  │                   │  │  │  │ PUT  /approve     │  │  │  │ • audit-events    │  │
 │  │ POST /users       │  │  │  │ GET  /reports     │  │  │  │ • erasure-events  │  │
 │  │ GET  /users/{id}  │  │  │  └────────┬──────────┘  │  │  └────────┬──────────┘  │
 │  │ POST /erase       │  │  │           │              │  │           │              │
 │  │ POST /consent     │  │  │  ┌────────▼──────────┐  │  │  ┌────────▼──────────┐  │
 │  └────────┬──────────┘  │  │  │ TransactionService│  │  │  │ AuditStorageService│ │
 │           │              │  │  │ SoxReportService  │  │  │  │                   │  │
 │  ┌────────▼──────────┐  │  │  │                   │  │  │  │ • Store to DB     │  │
 │  │  AuditAspect(AOP) │  │  │  │ @PreAuthorize     │  │  │  │ • Archive to S3   │  │
 │  │  @Auditable fires │  │  │  │ SOD Check:        │  │  │  │   (after 90 days) │  │
 │  │  on every data    │  │  │  │ creator≠approver  │  │  │  │ • Scrub PII on    │  │
 │  │  access method    │  │  │  │                   │  │  │  │   erasure events  │  │
 │  └────────┬──────────┘  │  │  │ @Auditable fires  │  │  │  └───────────────────┘  │
 │           │              │  │  └────────┬──────────┘  │  │                         │
 │  ┌────────▼──────────┐  │  │           │              │  └─────────────────────────┘
 │  │   UserService     │  │  │  ┌────────▼──────────┐  │
 │  │   ConsentService  │  │  │  │ FinancialTransaction│ │
 │  │   ErasureService  │  │  │  │ @Audited (Envers)  │ │
 │  │                   │  │  │  │ @Audited auto      │ │
 │  │ • Pseudonymize    │  │  │  │ tracks ALL changes  │ │
 │  │   on erasure      │  │  │  │ to _aud table      │ │
 │  │ • Granular consent│  │  │  └────────┬───────────┘ │
 │  │ • Outbox write    │  │  │           │              │
 │  └────────┬──────────┘  │  └───────────┼──────────────┘
 │           │              │              │
 │  ┌────────▼──────────┐  │  ┌───────────▼────────────┐
 │  │  OutboxPublisher  │  │  │  Financial DB          │
 │  │  (Poller/Relay)   │  │  │  (PostgreSQL)          │
 │  │                   │  │  │                        │
 │  │  Polls outbox     │  │  │ financial_transactions  │
 │  │  table every 1s   │  │  │ financial_txn_aud ◀──  │
 │  │  Publishes to     │  │  │ (Envers auto-created)  │
 │  │  Kafka reliably   │  │  │                        │
 │  └───────────────────┘  │  │ DB User: INSERT only   │
 │                          │  │ on _aud table (Trust)  │
 └──────────────────────────┘  └────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 4 — DATA LAYER  (Database Per Service)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ┌──────────────────────────┐  ┌──────────────────────────┐  ┌──────────────────────────┐
 │       USER DB            │  │     FINANCIAL DB         │  │       AUDIT DB           │
 │     (PostgreSQL)         │  │     (PostgreSQL)         │  │     (PostgreSQL)         │
 │                          │  │                          │  │                          │
 │  users                   │  │  financial_transactions  │  │  audit_log               │
 │  ├─ id (UUID)            │  │  ├─ id (UUID)            │  │  ├─ id                   │
 │  ├─ email_encrypted ◀──AES-256-GCM                     │  │  ├─ event_type           │
 │  ├─ full_name_encrypted  │  │  ├─ amount (BigDecimal)  │  │  ├─ user_id              │
 │  ├─ national_id_encrypted│  │  ├─ created_by           │  │  ├─ resource_type        │
 │  ├─ account_status       │  │  ├─ approved_by          │  │  ├─ action               │
 │  └─ erased_at            │  │  └─ status               │  │  ├─ outcome              │
 │                          │  │                          │  │  ├─ timestamp            │
 │  users_aud (Envers)      │  │  financial_txn_aud       │  │  └─ record_state_hash    │
 │  ├─ every version        │  │  ├─ every version        │  │                          │
 │  └─ who changed + when   │  │  └─ who changed + when   │  │  [Hot: 90 days]          │
 │                          │  │                          │  │  [Then → S3 Glacier]     │
 │  consent_records         │  │                          │  │                          │
 │  ├─ user_id              │  │                          │  │                          │
 │  ├─ marketing_consent    │  │                          │  │                          │
 │  ├─ consent_version      │  │                          │  │                          │
 │  └─ ip_address           │  │                          │  │                          │
 │                          │  │                          │  │                          │
 │  outbox_events           │  │                          │  │                          │
 │  ├─ event_type           │  │                          │  │                          │
 │  ├─ payload              │  │                          │  │                          │
 │  └─ processed_at (NULL=  │  │                          │  │                          │
 │      unprocessed)        │  │                          │  │                          │
 └──────────────────────────┘  └──────────────────────────┘  └──────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 5 — MESSAGING LAYER  (Event Bus)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

                    ┌────────────────────────────────────────┐
                    │              APACHE KAFKA              │
                    │                                        │
                    │  ┌────────────────┐  Partitions: 3    │
                    │  │  audit-events  │  Retention: 30d   │
                    │  └───────┬────────┘  acks: all        │
                    │          │                            │
                    │  ┌───────▼────────┐                   │
                    │  │ consent-events │                   │
                    │  └───────┬────────┘                   │
                    │          │                            │
                    │  ┌───────▼────────┐                   │
                    │  │  user-events   │                   │
                    │  └───────┬────────┘                   │
                    │          │                            │
                    │  ┌───────▼────────┐                   │
                    │  │ erasure-events │                   │
                    │  └────────────────┘                   │
                    └────────────────────────────────────────┘
                         │            │            │
                         ▼            ▼            ▼
                    audit-svc    marketing-svc  analytics-svc
                    (consumes    (respects      (respects
                     all topics)  consent)       consent)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 6 — SECURITY & SECRETS LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 ┌──────────────────────┐        ┌────────────────────────────────────────┐
 │   KEYCLOAK / OKTA    │        │         HASHICORP VAULT                │
 │   (Auth Server)      │        │         (Secrets Management)           │
 │                      │        │                                        │
 │  • Issues JWT tokens │        │  secret/user-service/                  │
 │  • RS256 signing     │        │  ├─ encryption.aes-key                 │
 │  • JWKS endpoint     │        │  ├─ db.password                        │
 │  • Manages roles:    │        │  └─ jwt.private-key                    │
 │    FINANCE_CREATOR   │        │                                        │
 │    FINANCE_APPROVER  │        │  secret/financial-service/             │
 │    AUDITOR           │        │  ├─ db.password                        │
 │    USER_ADMIN        │        │  └─ keystore.password                  │
 │                      │        │                                        │
 │  Gateway validates   │        │  • Dynamic secret rotation             │
 │  JWT using JWKS ──── │──────▶ │  • Audit every secret access          │
 │  public key          │        │  • Kubernetes auth integration         │
 └──────────────────────┘        └────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 7 — OBSERVABILITY LAYER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  All Services ──▶ Prometheus ──▶ Grafana Dashboards
       │               │              │
       │               │              ├─ Compliance Dashboard
       │               │              ├─ SOX Audit Reports
       │               │              └─ GDPR Erasure SLA Tracker
       │               │
       │               └─▶ Alerts:
       │                     • UnauthorizedFinancialAccess
       │                     • GDPRErasureSLABreached
       │                     • AuditPipelineGap
       │
       ├──▶ Logstash ──▶ Elasticsearch ──▶ Kibana
       │    (JSON logs)   (indexed)         (searchable)
       │
       └──▶ Zipkin / Jaeger  (Distributed Tracing)
            TraceId propagated across all services
            via X-Correlation-Id header

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 LAYER 8 — ARCHIVAL LAYER  (Long-term Compliance Storage)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Audit DB ──▶ [After 90 days] ──▶ S3 Standard ──▶ [After 1 year] ──▶ S3 Glacier
  (Hot)                             (Warm)                               (Cold / 7yr)

  Cost:   ████████████████          ████████                             ██
          Relational DB             S3 Standard                          S3 Glacier
          (most expensive)          (moderate)                           (cheapest)

  Query:  Instant (indexed)         Minutes (Athena)                     Hours (restore)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 COMPLETE REQUEST FLOW LEGEND
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  ──▶   Synchronous call (request/response)
  ══▶   Asynchronous event (fire and forget)
  ──│   Blocked / rejected
  ◀──   Response back to caller
  [  ]  Compliance control checkpoint

```

------

Now here is the complete **step-by-step flow explanation** for every major scenario:

---

## Flow 1 — User Registration (GDPR Path)

```
Step 1 → Client sends POST /api/users with JWT in Authorization header
Step 2 → CorrelationIdFilter generates X-Correlation-Id (e.g. "abc-123")
          and stamps it on the request AND the response
Step 3 → JwtValidationFilter validates JWT signature using Keycloak's
          JWKS public key. Extracts userId + roles. Injects as headers:
          X-User-Id, X-User-Roles into the request
Step 4 → Gateway routes to user-service via mTLS
Step 5 → UserController receives the request. Reads X-User-Id from header
          (no re-validation needed — gateway already verified it)
Step 6 → UserService.createUser() is called
          → User entity is built with @PersonalData fields
          → EncryptedStringConverter intercepts at JPA layer
          → AES-256-GCM encrypts email, fullName, nationalId before DB write
          → User saved to USER DB (encrypted at rest)
          → OutboxEvent written to outbox_events table IN THE SAME TRANSACTION
          → Both succeed or both rollback — consistency guaranteed
Step 7 → AuditAspect (@Auditable AOP) fires in finally{} block
          → Captures userId, action=CREATE, outcome=SUCCESS
          → Publishes AuditEvent ASYNC to audit-service (zero latency impact)
Step 8 → OutboxPublisher polls outbox_events WHERE processed_at IS NULL
          → Publishes UserCreated event to Kafka topic "user-events"
          → Marks outbox record as processed
Step 9 → Response returns to client with X-Correlation-Id in response header
```

---

## Flow 2 — Financial Transaction (SOX Path)

```
Step 1  → FINANCE_CREATOR calls POST /api/finance/transactions
Step 2  → Gateway validates JWT, confirms role = FINANCE_CREATOR
Step 3  → TransactionController calls TransactionService.createTransaction()
Step 4  → @PreAuthorize("hasRole('FINANCE_CREATOR')") passes
Step 5  → Transaction saved with status = PENDING_APPROVAL
           createdBy = current user's ID from SecurityContext
           Amount stored as BigDecimal (no floating point rounding risk)
Step 6  → Hibernate Envers intercepts the INSERT
           → Writes version 1 to financial_transactions_aud table automatically
           → Captures: who, what, when — tamper-proof SOX evidence
Step 7  → @Auditable AOP fires → AuditEvent published async to Kafka

--- [Later — different person] ---

Step 8  → FINANCE_APPROVER calls PUT /api/finance/transactions/{id}/approve
Step 9  → TransactionService checks: approver == createdBy?
           → YES → throws SoxViolationException (logged as COMPLIANCE ALERT)
           → NO  → proceeds to approval
Step 10 → Transaction status updated to APPROVED
           approvedBy + approvedAt stamped on record
Step 11 → Envers writes version 2 to _aud table
           → Full history: created by X at T1, approved by Y at T2
Step 12 → @Auditable AOP fires → AuditEvent with action=APPROVE published
```

---

## Flow 3 — Audit Event Pipeline

```
Step 1 → AuditAspect publishes AuditEvent to Kafka topic "audit-events"
          using @Async — caller's thread is NOT blocked
Step 2 → AuditEventConsumer in audit-service consumes the event
          concurrency=3 (one thread per partition for max throughput)
Step 3 → AuditStorageService stores AuditLog to AUDIT DB (hot storage)
          Record includes: traceId, userId, resourceType, action,
          outcome, timestamp, recordStateHash
Step 4 → [Nightly at 3 AM] AuditStorageService.archiveOldAuditLogs() runs
          → Finds records older than 90 days
          → Archives to S3 FIRST (archive-then-delete, never delete-then-archive)
          → Deletes from DB after S3 confirm
          → Audit DB stays bounded in size forever
Step 5 → After 1 year in S3 Standard → lifecycle rule moves to S3 Glacier
          → 7-year SOX retention satisfied at minimal cost
```

---

## Flow 4 — GDPR Erasure Request

```
Step 1 → User calls POST /api/users/{id}/erase (Right to be Forgotten)
Step 2 → ErasureService checks: is user already erased?
          → YES → return existing ErasureRecord (idempotent)
          → NO  → proceed
Step 3 → Check each @PersonalData field's erasable flag:
          → erasable=true  (email, fullName) → pseudonymize to "ERASED-<UUID>"
          → erasable=false (nationalId/KYC)  → skip — legal hold overrides GDPR
Step 4 → user.erasedAt = now() stamped — all future reads filter this out
Step 5 → ConsentRecord deleted (IP address + userAgent are themselves PII)
Step 6 → ErasureRecord saved — proof that erasure was performed (compliance evidence)
Step 7 → OutboxEvent "UserErased" written to outbox IN SAME TRANSACTION
Step 8 → OutboxPublisher sends to Kafka "erasure-events" topic
Step 9 → audit-service consumes erasure event
          → Scrubs userId references in audit logs → replaces with pseudonym
          → Audit trail structure preserved, PII removed
Step 10 → marketing-service, analytics-service consume erasure event
           → Remove user from all marketing lists and analytics profiles
           → No synchronous dependency — if they're down, they process on recovery
```

---

## Flow 5 — Consent Withdrawal

```
Step 1 → User calls POST /api/users/{id}/consent/withdraw
          with body: { "purpose": "marketing" }
Step 2 → ConsentService.withdrawConsent() called
Step 3 → Only the specific purpose field is set to false
          (marketing_consent = false, others unchanged — GDPR granularity)
Step 4 → consent_withdrawn_at stamped on ConsentRecord
Step 5 → OutboxEvent "ConsentWithdrawn" written in SAME TRANSACTION as consent update
Step 6 → Kafka publishes to "consent-events" topic
Step 7 → marketing-service consumes event → stops all marketing to this user
Step 8 → analytics-service consumes event → stops analytics tracking
          Each service acts independently — ConsentService has zero
          knowledge of who consumes the event (Dependency isolation)
```

---

## Flow 6 — Incident Investigation (Observability Path)

```
Step 1 → Alert fires in Prometheus/Grafana:
          "UnauthorizedFinancialAccess" — 6 FAILURE audit events in 5 minutes
Step 2 → Engineer takes the X-Correlation-Id from the alert
Step 3 → Searches Kibana with traceId = X-Correlation-Id
          → Sees full request journey across gateway → financial-service
          → Exact SQL queries, method calls, exception messages
Step 4 → Opens Jaeger/Zipkin with same traceId
          → Visual trace shows which service call was slow or failed
Step 5 → Calls SoxReportService.getTransactionHistory(txnId)
          → Envers returns full version history of the transaction
          → Sees every change: who, what field, old value, new value, when
Step 6 → Root cause identified and documented
          → Entire investigation done with the Correlation ID
          → No manual log scraping across 4 services
```

---

The key thing to notice across all 6 flows: **every compliance control is invisible to the business logic.** Controllers don't call audit services. Services don't manage encryption. The business code is clean — the compliance framework wraps around it via AOP, JPA converters, Kafka events, and Envers — enforced by the architecture, not by developer discipline.

