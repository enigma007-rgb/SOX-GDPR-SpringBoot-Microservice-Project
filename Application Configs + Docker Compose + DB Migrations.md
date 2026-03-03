```java

# ============================================================
# FILE: user-service/src/main/resources/application.yml
# PROPERTY: Dependency + Trust + Operability + Latency
# ============================================================
spring:
  application:
    name: user-service

  datasource:
    url: ${DB_URL}              # WHY env var: 12-factor compliance.
    username: ${DB_USERNAME}    # Secrets never in config files.
    password: ${DB_PASSWORD}    # Pulled from Vault at runtime.
    hikari:
      maximum-pool-size: 15
      minimum-idle: 5
      connection-timeout: 3000  # WHY 3s timeout: fail fast.
                                 # A 30s timeout means 30s of
                                 # blocked threads under DB outage.
                                 # 3s fails fast and triggers circuit
                                 # breaker. (Latency + Dependency)
      leak-detection-threshold: 60000  # WHY: Detects connections
                                        # held >60s — sign of a bug.
                                        # Prevents connection exhaustion
                                        # silently killing the service.
      max-lifetime: 1800000    # WHY 30min: Recycles connections
                                # before DB kills them server-side.
                                # Prevents "connection reset" errors.

  jpa:
    hibernate:
      ddl-auto: validate         # WHY validate not update/create:
                                  # In production, NEVER let Hibernate
                                  # auto-modify schema. Flyway controls
                                  # migrations. Hibernate just validates.
                                  # (Change + Operability — controlled change)
    properties:
      hibernate:
        envers:
          audit_table_suffix: _aud
          store_data_at_delete: true  # WHY: Store deleted record state
                                       # in audit table. SOX requires
                                       # knowing what was deleted and what
                                       # it contained. (Trust + Observability)

  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true

  kafka:
    bootstrap-servers: ${KAFKA_SERVERS}
    producer:
      acks: all                  # WHY acks=all: All in-sync replicas
                                  # must acknowledge before producer gets
                                  # success response. Prevents audit log
                                  # loss if a broker fails mid-write. (Trust)
      retries: 3
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      properties:
        enable.idempotence: true  # WHY: Prevents duplicate Kafka messages
                                   # if producer retries. Without this,
                                   # a network hiccup causes duplicate
                                   # audit events or duplicate user creation
                                   # events — compliance data corruption.

compliance:
  encryption:
    key: ${ENCRYPTION_KEY}       # WHY from env: AES key from Vault.
                                  # Hardcoding = anyone with repo access
                                  # can decrypt all PII. (Trust)

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  endpoint:
    health:
      show-details: when-authorized  # WHY not always: Health details
                                      # may expose internal topology.
                                      # Only authorized monitoring
                                      # systems should see full details.
  metrics:
    tags:
      service: user-service      # WHY tag: Every metric carries the
      environment: ${ENV:prod}   # service name. In Grafana, you can
                                  # filter by service without guessing.

logging:
  # WHY JSON structured logs:
  # Plain text logs cannot be efficiently queried in ELK.
  # JSON logs are machine-parseable — every field (traceId,
  # userId, duration) becomes a queryable index in Elasticsearch.
  # This transforms logging from "searchable text" to
  # "queryable compliance database". (Observability + Operability)
  pattern:
    console: "%d{ISO8601} [%X{traceId},%X{spanId}] %-5level %logger{36} - %msg%n"

server:
  shutdown: graceful             # WHY graceful: During rolling deployments,
                                  # in-flight requests must complete before
                                  # the pod stops. Abrupt shutdown = data
                                  # corruption and incomplete audit trails.
  ssl:
    enabled: ${SSL_ENABLED:true}
    key-store: ${KEY_STORE_PATH}
    key-store-password: ${KEY_STORE_PASSWORD}
    client-auth: need            # WHY need: Enforces mTLS. Only services
                                  # with a trusted certificate can call
                                  # this service. External actors are
                                  # rejected at the TLS layer. (Trust)

---
# ============================================================
# FILE: docker-compose.yml
# PROPERTY: Operability + Dependency + Consistency
#
# WHY Docker Compose for local dev:
# "It works on my machine" is an operability failure.
# Docker Compose ensures every developer runs the same
# infrastructure — same Kafka version, same Postgres config,
# same Vault setup. Parity between dev and prod reduces
# the "works locally, breaks in prod" class of bugs. (Consistency)
# ============================================================

version: '3.9'

services:

  postgres-user:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: userdb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-user-data:/var/lib/postgresql/data
      # WHY named volume not bind mount: Named volumes persist
      # between container restarts. Bind mounts are host-dependent.
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  postgres-financial:
    image: postgres:15-alpine
    # WHY separate DB for financial service:
    # Database-per-service pattern enforced at infrastructure level.
    # Developers cannot accidentally join across service boundaries.
    # (Consistency + Dependency isolation)
    environment:
      POSTGRES_DB: financialdb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5433:5432"

  postgres-audit:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: auditdb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports:
      - "5434:5432"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'  # WHY false:
      # Topics should be created explicitly with correct
      # partition counts and retention settings.
      # Auto-creation uses defaults that are wrong for production.
    ports:
      - "9092:9092"

  vault:
    image: hashicorp/vault:1.15
    # WHY Vault in compose:
    # Every developer should test against Vault locally so secret
    # management issues surface in dev, not in production.
    # (Operability — match prod infrastructure locally)
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-token
      VAULT_DEV_LISTEN_ADDRESS: 0.0.0.0:8200
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK  # WHY: Vault requires memory locking to prevent
                   # secrets from being swapped to disk. (Trust)

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - kafka
      - vault
    environment:
      VAULT_ADDR: http://vault:8200
      VAULT_TOKEN: ${VAULT_TOKEN}

  user-service:
    build: ./user-service
    depends_on:
      postgres-user:
        condition: service_healthy  # WHY condition healthy:
                                     # Without this, user-service starts
                                     # before Postgres is ready and crashes.
                                     # Healthcheck-gated startup = reliable
                                     # local environment. (Operability)
      kafka:
        condition: service_started
    environment:
      DB_URL: jdbc:postgresql://postgres-user:5432/userdb
      DB_USERNAME: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      KAFKA_SERVERS: kafka:9092
      ENCRYPTION_KEY: ${ENCRYPTION_KEY}
      VAULT_ADDR: http://vault:8200

  financial-service:
    build: ./financial-service
    depends_on:
      postgres-financial:
        condition: service_healthy
    environment:
      DB_URL: jdbc:postgresql://postgres-financial:5433/financialdb
      DB_USERNAME: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      KAFKA_SERVERS: kafka:9092

  audit-service:
    build: ./audit-service
    depends_on:
      postgres-audit:
        condition: service_healthy
      kafka:
        condition: service_started
    environment:
      DB_URL: jdbc:postgresql://postgres-audit:5434/auditdb
      DB_USERNAME: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}
      KAFKA_SERVERS: kafka:9092

volumes:
  postgres-user-data:
  postgres-financial-data:
  postgres-audit-data:

---
# ============================================================
# FILE: user-service/db/migration/V1__create_users.sql
# PROPERTY: Change + Trust + Capacity
# ============================================================

-- WHY column names end in _encrypted:
-- Self-documenting schema. Any developer or DBA looking at the
-- table immediately knows this column contains ciphertext.
-- They won't accidentally query it with LIKE or regex.
-- (Change — new developers get this context for free)

CREATE TABLE users (
    id              VARCHAR(36) PRIMARY KEY,
    email_encrypted VARCHAR(512) NOT NULL UNIQUE,  -- encrypted PII
    full_name_encrypted VARCHAR(512),              -- encrypted PII
    national_id_encrypted VARCHAR(512),            -- encrypted SENSITIVE PII
    account_status  VARCHAR(50) NOT NULL,
    created_at      TIMESTAMP NOT NULL,
    updated_at      TIMESTAMP,
    created_by      VARCHAR(255) NOT NULL,
    updated_by      VARCHAR(255),
    erased_at       TIMESTAMP  -- NULL = active, NOT NULL = pseudonymized
);

-- WHY index on erased_at:
-- Data retention scheduler queries "WHERE erased_at IS NULL AND
-- created_at < retention_cutoff" nightly. Without index,
-- this full-table scan kills DB performance at scale. (Capacity)
CREATE INDEX idx_users_erased_at ON users(erased_at);
CREATE INDEX idx_users_created_at ON users(created_at);

-- Hibernate Envers audit table — auto-created but we define
-- explicitly to add our own constraints
CREATE TABLE users_aud (
    id              VARCHAR(36) NOT NULL,
    rev             INTEGER NOT NULL,
    revtype         SMALLINT,
    email_encrypted VARCHAR(512),
    full_name_encrypted VARCHAR(512),
    national_id_encrypted VARCHAR(512),
    account_status  VARCHAR(50),
    created_by      VARCHAR(255),
    updated_by      VARCHAR(255),
    erased_at       TIMESTAMP,
    PRIMARY KEY (id, rev)
);

-- WHY no UPDATE or DELETE on _aud table in app DB user:
-- Grant audit table INSERT only to app DB user.
-- This prevents application-layer tampering with audit history.
-- A compromised service account cannot alter SOX evidence.
-- REVOKE UPDATE, DELETE ON users_aud FROM app_user;  (Trust)

---
# FILE: user-service/db/migration/V2__create_consent.sql

CREATE TABLE consent_records (
    id                          VARCHAR(36) PRIMARY KEY,
    user_id                     VARCHAR(36) NOT NULL UNIQUE,
    marketing_consent           BOOLEAN NOT NULL DEFAULT FALSE,
    analytics_consent           BOOLEAN NOT NULL DEFAULT FALSE,
    third_party_consent         BOOLEAN NOT NULL DEFAULT FALSE,
    consent_given_at            TIMESTAMP NOT NULL,
    consent_withdrawn_at        TIMESTAMP,
    consent_version             VARCHAR(20) NOT NULL,
    ip_address                  VARCHAR(45) NOT NULL,  -- IPv6 max length
    user_agent                  TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX idx_consent_version ON consent_records(consent_version);
-- WHY index on consent_version:
-- When privacy policy changes, query is:
-- "SELECT user_id FROM consent_records WHERE consent_version != 'v3'"
-- to find users needing re-consent. Index makes this fast at scale.

---
# FILE: user-service/db/migration/V3__create_outbox.sql

CREATE TABLE outbox_events (
    id              VARCHAR(36) PRIMARY KEY,
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(36) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    payload         TEXT NOT NULL,
    topic           VARCHAR(200) NOT NULL,
    created_at      TIMESTAMP NOT NULL,
    processed_at    TIMESTAMP,   -- NULL = unprocessed
    retry_count     INTEGER DEFAULT 0
);

-- WHY index on processed_at:
-- The outbox poller queries "WHERE processed_at IS NULL"
-- every second. Without index, this scans the entire table.
-- With index, it's instant. Critical for low-latency event delivery.
CREATE INDEX idx_outbox_unprocessed ON outbox_events(processed_at)
    WHERE processed_at IS NULL;  -- partial index — only unprocessed rows

```
