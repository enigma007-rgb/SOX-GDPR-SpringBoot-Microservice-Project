compliance-microservices/
├── api-gateway/
│   ├── src/main/java/com/compliance/gateway/
│   │   ├── GatewayApplication.java
│   │   ├── config/
│   │   │   ├── GatewaySecurityConfig.java
│   │   │   └── RouteConfig.java
│   │   └── filter/
│   │       ├── JwtValidationFilter.java
│   │       └── CorrelationIdFilter.java
│   └── src/main/resources/application.yml
│
├── user-service/                          # GDPR domain
│   ├── src/main/java/com/compliance/user/
│   │   ├── UserServiceApplication.java
│   │   ├── annotation/
│   │   │   ├── PersonalData.java
│   │   │   └── Auditable.java
│   │   ├── aspect/
│   │   │   └── AuditAspect.java
│   │   ├── config/
│   │   │   ├── SecurityConfig.java
│   │   │   ├── EncryptionConfig.java
│   │   │   └── AsyncConfig.java
│   │   ├── controller/
│   │   │   ├── UserController.java
│   │   │   └── DataSubjectController.java
│   │   ├── domain/
│   │   │   ├── User.java
│   │   │   ├── ConsentRecord.java
│   │   │   └── ErasureRecord.java
│   │   ├── converter/
│   │   │   └── EncryptedStringConverter.java
│   │   ├── event/
│   │   │   ├── ConsentUpdatedEvent.java
│   │   │   ├── UserErasureEvent.java
│   │   │   └── AuditEvent.java
│   │   ├── exception/
│   │   │   ├── GlobalExceptionHandler.java
│   │   │   └── ComplianceViolationException.java
│   │   ├── outbox/
│   │   │   ├── OutboxEvent.java
│   │   │   └── OutboxPublisher.java
│   │   ├── repository/
│   │   │   ├── UserRepository.java
│   │   │   ├── ConsentRepository.java
│   │   │   └── OutboxRepository.java
│   │   └── service/
│   │       ├── UserService.java
│   │       ├── ConsentService.java
│   │       ├── ErasureService.java
│   │       ├── EncryptionService.java
│   │       └── AuditService.java
│   └── src/main/resources/
│       ├── application.yml
│       └── db/migration/
│           ├── V1__create_users.sql
│           ├── V2__create_consent.sql
│           └── V3__create_outbox.sql
│
├── financial-service/                     # SOX domain
│   ├── src/main/java/com/compliance/financial/
│   │   ├── FinancialServiceApplication.java
│   │   ├── config/
│   │   │   ├── EnversConfig.java
│   │   │   └── SecurityConfig.java
│   │   ├── controller/
│   │   │   └── TransactionController.java
│   │   ├── domain/
│   │   │   └── FinancialTransaction.java
│   │   ├── repository/
│   │   │   └── TransactionRepository.java
│   │   └── service/
│   │       ├── TransactionService.java
│   │       └── SoxReportService.java
│   └── src/main/resources/application.yml
│
├── audit-service/                         # SOX + GDPR cross-cutting
│   ├── src/main/java/com/compliance/audit/
│   │   ├── AuditServiceApplication.java
│   │   ├── consumer/
│   │   │   └── AuditEventConsumer.java
│   │   ├── domain/
│   │   │   └── AuditLog.java
│   │   ├── repository/
│   │   │   └── AuditLogRepository.java
│   │   └── service/
│   │       └── AuditStorageService.java
│   └── src/main/resources/application.yml
│
├── shared-compliance-lib/                 # Shared across all services
│   └── src/main/java/com/compliance/shared/
│       ├── annotation/
│       │   └── PersonalData.java
│       ├── encryption/
│       │   └── EncryptionService.java
│       └── audit/
│           └── AuditEventPublisher.java
│
├── docker-compose.yml
└── pom.xml (parent)
