```java
// ============================================================
// FILE: audit-service/consumer/AuditEventConsumer.java
// PROPERTY: Capacity + Operability + Trust
//
// WHY a dedicated Audit Service:
// If audit logging lives inside user-service or financial-service,
// those services bear the storage and query burden of all audit
// data. Separating it lets each service scale independently.
// Audit data grows without bound — it needs its own Capacity plan.
//
// WHY consume from Kafka (not direct DB write):
// 1. Decouples audit write performance from business service perf
// 2. Kafka topic acts as a buffer during audit DB slowdowns
// 3. Same audit events can be consumed by multiple systems:
//    audit DB, S3 archive, SIEM tool — without changing producers
// ============================================================
package com.compliance.audit.consumer;

import com.compliance.audit.domain.AuditLog;
import com.compliance.audit.service.AuditStorageService;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

@Slf4j
@Component
@RequiredArgsConstructor
public class AuditEventConsumer {

    private final AuditStorageService storageService;
    private final ObjectMapper objectMapper;

    // WHY separate consumer groups for each topic:
    // Each topic has independent throughput and lag characteristics.
    // Using one consumer group for all would mean slow audit
    // processing blocks fast consent event processing. (Capacity)
    @KafkaListener(
        topics = "audit-events",
        groupId = "audit-service-group",
        concurrency = "3"   // WHY 3: matches partition count.
                            // One thread per partition = max throughput
                            // without idle threads. (Economics + Capacity)
    )
    public void consumeAuditEvent(String payload,
                                   @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
                                   @Header(KafkaHeaders.OFFSET) long offset) {
        try {
            AuditLog log = objectMapper.readValue(payload, AuditLog.class);
            storageService.store(log);
        } catch (Exception e) {
            // WHY log with full context and NOT throw:
            // If we throw, Kafka will retry this message indefinitely,
            // blocking all subsequent messages on this partition.
            // Audit log parse failure should go to a dead letter topic
            // for manual investigation, not halt the pipeline. (Operability)
            log.error("Failed to process audit event — offset={}, topic={}, payload={}",
                offset, topic, payload, e);
            // In production: send to dead-letter topic for investigation
        }
    }

    @KafkaListener(topics = "erasure-events", groupId = "audit-erasure-group")
    public void consumeErasureEvent(String payload) {
        // WHY audit service listens to erasure events:
        // The audit service must record that an erasure occurred —
        // paradoxically keeping a record of the deletion for compliance.
        // But it must also scrub the ERASED user's PII from audit logs
        // where it appears — GDPR applies to audit logs too.
        storageService.processErasureInAuditLogs(payload);
    }
}


// ============================================================
// FILE: audit-service/service/AuditStorageService.java
// PROPERTY: Capacity + Economics + Operability
// ============================================================
package com.compliance.audit.service;

import com.compliance.audit.domain.AuditLog;
import com.compliance.audit.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuditStorageService {

    private final AuditLogRepository auditLogRepository;
    private final S3ArchiveClient s3ArchiveClient;     // abstracted S3 client

    @Transactional
    public void store(AuditLog auditLog) {
        // WHY store in relational DB for 90 days:
        // Recent audit logs need fast querying — daily security review,
        // incident investigation, regulatory spot checks. Relational DB
        // provides this with indexes. Beyond 90 days, queries are rare
        // (only deep audits) so cold storage is acceptable. (Economics)
        auditLogRepository.save(auditLog);
    }

    // WHY scheduled archival:
    // Left unmanaged, the audit table becomes your largest table and
    // slows down all DB operations. Automated archival keeps hot
    // storage bounded. This is Capacity management made operational.
    @Scheduled(cron = "0 0 3 * * ?")  // 3 AM daily
    @Transactional
    public void archiveOldAuditLogs() {
        Instant archiveBefore = Instant.now().minus(90, ChronoUnit.DAYS);
        List<AuditLog> toArchive = auditLogRepository
            .findByTimestampBefore(archiveBefore);

        if (toArchive.isEmpty()) return;

        // WHY archive to S3 before deleting from DB:
        // Delete-before-archive risks losing logs if S3 write fails.
        // Archive-then-delete is safe — worst case: duplicate in both.
        // Duplicates are harmless; missing audit logs are compliance failures.
        s3ArchiveClient.archiveBatch(toArchive);  // S3 Glacier — cost effective
        auditLogRepository.deleteAll(toArchive);

        log.info("Archived {} audit logs older than 90 days to S3",
            toArchive.size());
    }

    public void processErasureInAuditLogs(String erasurePayload) {
        // WHY scrub PII from audit logs on erasure:
        // Audit logs often contain user identifiers and sometimes
        // PII in payloads. GDPR erasure applies to audit logs too.
        // We replace userId references with the pseudonym without
        // deleting the audit event itself — audit trail integrity preserved.
        log.info("Processing erasure in audit logs — payload={}", erasurePayload);
        // Implementation: parse userId from payload, update audit records
        // to replace direct PII with pseudonym, preserving audit structure
    }
}


// ============================================================
// FILE: api-gateway/filter/JwtValidationFilter.java
// PROPERTY: Trust + Latency + Dependency
//
// WHY validate JWT at gateway ONLY:
// If every downstream service re-validates the JWT, you get:
// 1. Each service needs Keycloak/Auth Server connectivity (Dependency)
// 2. Each JWT validation adds 5-20ms per service (Latency compounding)
// 3. Inconsistent validation logic if services use different libraries
//
// Solution: Gateway validates once. Downstream services receive
// a verified, enriched request header with userId and roles.
// They TRUST the gateway — but only because mTLS ensures the
// request actually came from the gateway. (Trust via mTLS)
// ============================================================
package com.compliance.gateway.filter;

import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.*;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

@Slf4j
@Component
@Order(1)
public class JwtValidationFilter implements GlobalFilter {

    private final ReactiveJwtDecoder jwtDecoder;

    public JwtValidationFilter(ReactiveJwtDecoder jwtDecoder) {
        this.jwtDecoder = jwtDecoder;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String authHeader = exchange.getRequest().getHeaders()
            .getFirst("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(7);

        return jwtDecoder.decode(token)
            .flatMap(jwt -> {
                // WHY inject claims as headers to downstream services:
                // Downstream services don't re-parse the JWT.
                // They read verified, gateway-stamped headers.
                // This is safe ONLY because of mTLS between services —
                // nothing external can forge these headers.
                ServerWebExchange enriched = exchange.mutate()
                    .request(r -> r
                        .header("X-User-Id", jwt.getSubject())
                        .header("X-User-Roles",
                            jwt.getClaimAsStringList("roles").toString())
                        .header("X-Tenant-Id",
                            jwt.getClaimAsString("tenantId"))
                    )
                    .build();
                return chain.filter(enriched);
            })
            .onErrorResume(e -> {
                log.warn("JWT validation failed — {}", e.getMessage());
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            });
    }
}


// ============================================================
// FILE: api-gateway/filter/CorrelationIdFilter.java
// PROPERTY: Observability + Operability
//
// WHY generate correlation ID at gateway:
// A user request touches 4 services. When it fails in service 3,
// you need to trace the full journey in your logs.
// The correlation ID (= trace ID) ties all service logs together.
// Generated at entry point (gateway) and propagated via headers.
// Without this, production debugging is nearly impossible. (Operability)
// ============================================================
package com.compliance.gateway.filter;

import org.springframework.cloud.gateway.filter.*;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;
import java.util.UUID;

@Component
@Order(0)  // WHY Order(0): Must run before JWT filter so correlation
           // ID is available in all subsequent filter logs.
public class CorrelationIdFilter implements GlobalFilter {

    private static final String CORRELATION_HEADER = "X-Correlation-Id";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst(CORRELATION_HEADER);

        // WHY accept client-provided correlation ID:
        // Mobile clients and external systems may have their own
        // trace IDs. Accepting them enables end-to-end tracing
        // from client all the way through your microservices.
        if (correlationId == null) {
            correlationId = UUID.randomUUID().toString();
        }

        String finalCorrelationId = correlationId;
        return chain.filter(
            exchange.mutate()
                .request(r -> r.header(CORRELATION_HEADER, finalCorrelationId))
                .response(r -> {
                    // WHY add to response too:
                    // Client can log the correlation ID on errors,
                    // making support tickets contain the exact ID
                    // needed to find the logs. (Operability)
                    r.getHeaders().add(CORRELATION_HEADER, finalCorrelationId);
                    return r;
                })
                .build()
        );
    }
}

```
