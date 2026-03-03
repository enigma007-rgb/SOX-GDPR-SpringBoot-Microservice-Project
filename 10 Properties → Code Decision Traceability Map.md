
# 10 Properties → Code Decision Traceability Map

Every architectural and code decision maps to at least one property.
Nothing in this codebase is accidental.

---

## 1. 🔄 Flow
| Decision | File | Why |
|---|---|---|
| `@Auditable` on service layer, not controller | `AuditAspect.java` | Audit intercepts the data flow, not HTTP flow — fires regardless of how data is accessed |
| Outbox pattern — write event in same transaction | `UserService.java` | Event flows guaranteed — no gap between business data and event publication |
| Async audit publishing with `@Async` | `AuditAspect.java` | Compliance logging does not block the business request flow |
| Kafka consumer for audit events | `AuditEventConsumer.java` | Audit data flows independently from business data — separate pipeline, separate throughput |

---

## 2. ⚖️ Consistency
| Decision | File | Why |
|---|---|---|
| `@PersonalData` in shared-compliance-lib | `PersonalData.java` | One annotation definition used by all services — no team defines PII differently |
| `EncryptedStringConverter` at persistence layer | `EncryptedStringConverter.java` | Every write encrypted — no code path can bypass it. Consistent by construction |
| GDPR-SOX conflict resolved via pseudonymization | `ErasureService.java` | One canonical rule applied everywhere: pseudonymize PII, retain financial structure |
| `store_data_at_delete: true` in Envers config | `application.yml` | Deleted record state always captured — consistent audit trail on DELETE, not just UPDATE |

---

## 3. 📦 Capacity
| Decision | File | Why |
|---|---|---|
| Audit logs to Kafka → S3 Glacier | `AuditStorageService.java` | Unbounded audit growth managed: hot (90d/DB), warm (1yr/S3), cold (7yr/Glacier) |
| `concurrency = "3"` on Kafka listener | `AuditEventConsumer.java` | Matches partition count — max throughput, no idle threads |
| Partial index on outbox_events | `V3__create_outbox.sql` | Index only covers unprocessed rows — shrinks as events are processed. Stays fast forever |
| Separate DBs per service | `docker-compose.yml` | Each service scales its DB independently — financial data volume doesn't affect user DB |

---

## 4. ⚡ Latency
| Decision | File | Why |
|---|---|---|
| JWT validated at gateway ONLY | `JwtValidationFilter.java` | One validation per request, not N validations (one per service). Saves 5-20ms × N services |
| Async audit write | `AuditAspect.java` | 0ms added to business response time from audit logging |
| `connection-timeout: 3000` HikariCP | `application.yml` | Fail fast on DB unavailability — don't queue threads for 30s burning response budget |
| `@Transactional(readOnly = true)` on reads | `UserService.java`, `SoxReportService.java` | Skips dirty checking, flushes. Reduces ORM overhead on reads |

---

## 5. 🔐 Trust
| Decision | File | Why |
|---|---|---|
| AES-256-GCM (not CBC) | `EncryptionService.java` | GCM provides authenticity (MAC) + confidentiality. Tampering with DB ciphertext is detectable |
| Random IV per encryption call | `EncryptionService.java` | IV reuse in GCM = complete confidentiality break. Per-call random IV prevents this structurally |
| 256-bit key enforced at construction | `EncryptionService.java` | Fail at startup if key is wrong size — never operate with weakened encryption |
| Creator ≠ Approver enforced in business logic | `TransactionService.java` | Defense in depth — @PreAuthorize checks role, business logic checks identity. Two independent SOD controls |
| `client-auth: need` mTLS | `application.yml` | Only services with trusted certificates can call this service. External actors rejected at TLS layer |
| REVOKE UPDATE,DELETE on _aud tables | `V1__create_users.sql` | App cannot tamper with its own audit history even if compromised — DB-level control |
| UUID primary keys not auto-increment | `User.java` | Prevents IDOR attacks — resource IDs are unguessable |

---

## 6. 👁️ Observability
| Decision | File | Why |
|---|---|---|
| `@Auditable` AOP captures all access | `AuditAspect.java` | Every data access is visible — success AND failure. Invisible access = compliance blindspot |
| `recordStateHash` in AuditEvent | `AuditEventPublisher.java` | Proves what the record contained AT TIME OF ACCESS — post-hoc tampering is detectable |
| Correlation ID at gateway, propagated through | `CorrelationIdFilter.java` | Full request journey traceable across all services from single ID |
| JSON structured logs | `application.yml` | Every log field is queryable in ELK — compliance evidence is searchable, not just readable |
| Prometheus metrics with service tags | `application.yml` | Every metric labeled with service + environment — Grafana dashboards filter correctly |
| `AuditReader` SOX report API | `SoxReportService.java` | Auditors self-serve compliance evidence — no engineering needed for audit queries |

---

## 7. 💰 Economics
| Decision | File | Why |
|---|---|---|
| S3 Glacier archival after 90 days | `AuditStorageService.java` | 90-day-old audit data costs 100× less in Glacier vs. relational DB. 7-year retention becomes affordable |
| `@NotAudited` on non-SOX fields | `FinancialTransaction.java` | Only audit what SOX requires — avoid wasted audit table bloat on volatile display fields |
| Partial index on outbox unprocessed rows | `V3__create_outbox.sql` | Index stays small as events are processed — doesn't grow unboundedly with history |
| Separate consumer group per topic | `AuditEventConsumer.java` | Each topic scales independently — don't over-provision consumers for low-volume topics |

---

## 8. 🔁 Change
| Decision | File | Why |
|---|---|---|
| `@PersonalData` with `erasable`, `retentionDays`, `purpose` | `PersonalData.java` | New PII fields added to any entity carry compliance metadata automatically — no code change to erasure logic |
| `consentVersion` on ConsentRecord | `ConsentRecord.java` | Privacy policy changes trigger targeted re-consent — not blanket re-consent of all users |
| Flyway migrations, `ddl-auto: validate` | `application.yml` | Schema changes are versioned, reviewed, and applied deliberately — not auto-applied by Hibernate |
| `AuditEventPublisher` as interface | `AuditEventPublisher.java` | Swap Kafka for another broker without changing audit callsites across all services |
| Expand-contract migration pattern | (convention) | Encrypted column changes never happen in-place — always add-migrate-remove |

---

## 9. 🔗 Dependency
| Decision | File | Why |
|---|---|---|
| Transactional Outbox pattern | `OutboxEvent.java`, `UserService.java` | User creation no longer fails if Kafka is down — Kafka dependency removed from critical path |
| Consent propagated via Kafka events, not sync calls | `ConsentService.java` | Consent withdrawal doesn't require all services to be available — eventual consistency with bounded window |
| `AuditEventPublisher` interface | `AuditEventPublisher.java` | Audit infrastructure is swappable — services don't depend on specific broker implementation |
| JWT validated once at gateway | `JwtValidationFilter.java` | Downstream services don't depend on Auth Server connectivity for every request |
| Dead-letter handling on Kafka consume failure | `AuditEventConsumer.java` | Failed audit events don't block the consumer — dependency on perfect event format removed |

---

## 10. 🛠️ Operability
| Decision | File | Why |
|---|---|---|
| `server.shutdown: graceful` | `application.yml` | Rolling deployments complete in-flight requests — no data corruption, no incomplete audit trails |
| `service_healthy` condition in Docker Compose | `docker-compose.yml` | Services wait for DB to be ready — local dev matches prod startup behavior |
| `correlation-id` returned in response headers | `CorrelationIdFilter.java` | Support teams get trace ID from client error reports — log search is instant, not investigative |
| `processedAt IS NULL` partial index on outbox | `V3__create_outbox.sql` | Outbox poller query is fast forever — doesn't slow down as historical events accumulate |
| Dead-letter topic on audit consumer failure | `AuditEventConsumer.java` | Failed events don't vanish — they're available for reprocessing. No silent compliance gaps |
| `SoxReportService.generateComplianceReport()` | `SoxReportService.java` | Auditors get structured reports without engineering involvement — compliance is self-service |
| Named volumes in Docker Compose | `docker-compose.yml` | Data persists across container restarts in local dev — no data loss during development |


