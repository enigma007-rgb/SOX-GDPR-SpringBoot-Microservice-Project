SOX & GDPR Compliant Microservice — Full Text Architecture Diagram
----

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
