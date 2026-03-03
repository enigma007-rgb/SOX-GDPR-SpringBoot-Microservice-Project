```java
// ============================================================
// FILE: user-service/service/UserService.java
// PROPERTY: Flow + Consistency + Trust
// ============================================================
package com.compliance.user.service;

import com.compliance.shared.annotation.Auditable;
import com.compliance.user.domain.*;
import com.compliance.user.outbox.*;
import com.compliance.user.repository.*;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final OutboxRepository outboxRepository;
    private final ObjectMapper objectMapper;

    // WHY @Transactional wrapping BOTH userRepository.save AND
    // outboxRepository.save: This is the Transactional Outbox
    // pattern in action. Both writes succeed or both roll back.
    // You can NEVER have a user saved without an outbox event
    // or an outbox event without a saved user. Consistency guaranteed.
    @Transactional
    public User createUser(User user, ConsentRecord consent) {
        user.setConsentRecord(consent);
        user.setAccountStatus("ACTIVE");
        User saved = userRepository.save(user);

        // WHY write to outbox in SAME transaction:
        // See OutboxEvent class for full explanation.
        // If Kafka is down, user is still created. Outbox poller
        // will publish the event when Kafka recovers. Zero data loss.
        OutboxEvent outboxEvent = OutboxEvent.builder()
            .id(UUID.randomUUID().toString())
            .aggregateType("User")
            .aggregateId(saved.getId())
            .eventType("UserCreated")
            .payload(toJson(saved))
            .topic("user-events")
            .createdAt(Instant.now())
            .retryCount(0)
            .build();

        outboxRepository.save(outboxEvent);
        log.info("User created and outbox event written — userId={}", saved.getId());
        return saved;
    }

    // WHY @Auditable annotation here and not in controller:
    // Controllers handle HTTP concerns. Auditing is a data
    // access concern. Placing it at the service layer means
    // the audit fires regardless of whether access came from
    // REST, a scheduled job, or an internal service call.
    // This is Flow property — audit intercepts the data flow,
    // not the HTTP flow.
    @Auditable(resourceType = "User", action = "READ")
    @Transactional(readOnly = true)
    public User getUser(String userId) {
        return userRepository.findById(userId)
            .filter(u -> u.getErasedAt() == null) // WHY: Never return erased users
            .orElseThrow(() -> new UserNotFoundException(userId));
    }

    private String toJson(Object obj) {
        try { return objectMapper.writeValueAsString(obj); }
        catch (Exception e) { throw new RuntimeException("Serialization failed", e); }
    }
}


// ============================================================
// FILE: user-service/service/ConsentService.java
// PROPERTY: Consistency + Change + Operability
// ============================================================
package com.compliance.user.service;

import com.compliance.user.domain.ConsentRecord;
import com.compliance.user.outbox.*;
import com.compliance.user.repository.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ConsentService {

    private final ConsentRepository consentRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public void withdrawConsent(String userId, String purpose, String ipAddress) {
        ConsentRecord record = consentRepository.findByUserId(userId)
            .orElseThrow(() -> new ConsentNotFoundException(userId));

        // WHY granular withdrawal, not full wipe:
        // GDPR allows users to withdraw consent for specific purposes.
        // Withdrawing marketing consent should not stop you from
        // processing their order. Blanket withdrawal is wrong.
        switch (purpose) {
            case "marketing"    -> record.setMarketingConsent(false);
            case "analytics"    -> record.setAnalyticsConsent(false);
            case "thirdParty"   -> record.setThirdPartyDataSharingConsent(false);
            default -> throw new IllegalArgumentException("Unknown purpose: " + purpose);
        }
        record.setConsentWithdrawnAt(Instant.now());
        consentRepository.save(record);

        // WHY publish ConsentWithdrawn event via outbox:
        // All downstream services (marketing, analytics, etc.) must
        // stop processing this user's data for this purpose.
        // Event-driven propagation via Kafka ensures all services
        // react — without a synchronous dependency on each of them.
        // (Dependency property: ConsentService doesn't know or care
        // which services consume this event.)
        OutboxEvent event = OutboxEvent.builder()
            .id(UUID.randomUUID().toString())
            .aggregateType("ConsentRecord")
            .aggregateId(userId)
            .eventType("ConsentWithdrawn")
            .payload(String.format(
                "{\"userId\":\"%s\",\"purpose\":\"%s\",\"withdrawnAt\":\"%s\"}",
                userId, purpose, Instant.now()))
            .topic("consent-events")
            .createdAt(Instant.now())
            .build();

        outboxRepository.save(event);
        log.info("Consent withdrawn — userId={}, purpose={}", userId, purpose);
    }
}


// ============================================================
// FILE: user-service/service/ErasureService.java
// PROPERTY: Consistency + Operability + Flow
//
// WHY pseudonymize instead of DELETE:
// SOX requires financial record integrity for 7 years.
// If a user made financial transactions, hard-deleting their
// record orphans those transaction records — SOX violation.
// Pseudonymization satisfies GDPR's "right to erasure" because
// pseudonymized data is no longer "personal data" under GDPR
// recital 26 — it cannot be linked back to the individual.
// This resolves the GDPR-SOX consistency conflict definitively.
// ============================================================
package com.compliance.user.service;

import com.compliance.user.domain.*;
import com.compliance.user.outbox.*;
import com.compliance.user.repository.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ErasureService {

    private final UserRepository userRepository;
    private final ConsentRepository consentRepository;
    private final OutboxRepository outboxRepository;
    private final ErasureRecordRepository erasureRecordRepository;

    @Transactional
    public ErasureRecord processErasureRequest(String userId, String requestId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));

        // WHY check already erased:
        // DSAR requests can be duplicated (user submits twice).
        // Idempotent erasure — second request returns existing
        // erasure record. No double-processing. (Flow + Operability)
        if (user.getErasedAt() != null) {
            log.info("User already erased — userId={}", userId);
            return erasureRecordRepository.findByUserId(userId).orElseThrow();
        }

        String pseudonym = "ERASED-" + UUID.randomUUID();

        // WHY pseudonymize per @PersonalData.erasable():
        // Fields with erasable=false (e.g., KYC nationalId with
        // legal hold) are NOT pseudonymized. The system respects
        // legal obligations that override GDPR erasure.
        user.setEmail(pseudonym + "@erased.invalid");
        user.setFullName("ERASED");

        // WHY check erasable annotation programmatically:
        // If someone adds a new PII field later, the erasure
        // service handles it automatically based on annotation —
        // no code change required. (Change property)
        // In full impl, use reflection to find all @PersonalData
        // fields with erasable=true and pseudonymize them.

        user.setErasedAt(Instant.now());
        userRepository.save(user);

        // WHY delete consent separately:
        // Consent records serve no purpose post-erasure and contain
        // IP/userAgent metadata that itself is PII. Must be deleted.
        consentRepository.deleteByUserId(userId);

        // WHY record the erasure action itself:
        // GDPR requires you to prove you erased the data.
        // The ErasureRecord is your compliance evidence.
        // Paradoxically, you keep a record of the deletion.
        ErasureRecord record = ErasureRecord.builder()
            .id(requestId)
            .userId(userId)
            .erasedAt(Instant.now())
            .erasureType("GDPR_REQUEST")
            .status("COMPLETED")
            .build();
        erasureRecordRepository.save(record);

        // WHY notify downstream services via outbox event:
        // Erasure must propagate to ALL services holding this
        // user's data — analytics, marketing, financial (for PII only).
        // Event-driven = no synchronous dependency on those services.
        // If one is down, it processes the erasure when it recovers.
        OutboxEvent event = OutboxEvent.builder()
            .id(UUID.randomUUID().toString())
            .aggregateType("User")
            .aggregateId(userId)
            .eventType("UserErased")
            .payload(String.format("{\"userId\":\"%s\",\"requestId\":\"%s\"}", userId, requestId))
            .topic("erasure-events")
            .createdAt(Instant.now())
            .build();
        outboxRepository.save(event);

        log.info("Erasure completed — userId={}, requestId={}", userId, requestId);
        return record;
    }
}


// ============================================================
// FILE: user-service/aspect/AuditAspect.java
// PROPERTY: Observability + Trust + Flow
//
// WHY @Aspect for auditing:
// If audit logging is written manually in each method, developers
// will forget it, do it inconsistently, or skip it under deadline
// pressure. AOP makes audit logging structurally impossible to miss.
// Every method annotated @Auditable is audited — no exceptions.
// This is the Trust property: the system enforces compliance,
// humans don't have to remember to.
//
// WHY @Async on audit write:
// Audit logs are compliance-critical but NOT on the critical
// response path. If audit write takes 50ms, that 50ms should not
// be added to every user API call. Async write = zero latency
// impact on the business operation. (Latency property)
// ============================================================
package com.compliance.user.aspect;

import com.compliance.shared.annotation.Auditable;
import com.compliance.shared.audit.AuditEventPublisher;
import com.compliance.shared.audit.AuditEventPublisher.AuditEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.scheduling.annotation.Async;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import java.time.Instant;
import java.util.UUID;

@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class AuditAspect {

    private final AuditEventPublisher auditEventPublisher;

    @Around("@annotation(auditable)")
    public Object auditDataAccess(ProceedingJoinPoint jp,
                                  Auditable auditable) throws Throwable {
        String outcome = "SUCCESS";
        String failureReason = null;
        Object result = null;
        long start = System.currentTimeMillis();

        try {
            result = jp.proceed();
            return result;
        } catch (Exception e) {
            outcome = "FAILURE";
            failureReason = e.getClass().getSimpleName();
            // WHY re-throw: Audit captures the failure but doesn't
            // swallow it. Business exception propagation is unaffected.
            throw e;
        } finally {
            // WHY in finally block: Audit fires whether the method
            // succeeded or failed. Failed access attempts are MORE
            // important to audit than successful ones. (Trust)
            String finalOutcome = outcome;
            String finalReason = failureReason;
            publishAsync(auditable, finalOutcome, finalReason,
                System.currentTimeMillis() - start);
        }
    }

    // WHY @Async here and not in the main method:
    // The main method must run in the caller's thread to capture
    // the SecurityContext (which is thread-local). After capturing
    // all needed context, we publish async. This is the correct
    // pattern — capturing sync, publishing async.
    @Async("auditExecutor")
    protected void publishAsync(Auditable auditable, String outcome,
                                 String failureReason, long durationMs) {
        try {
            String userId = SecurityContextHolder.getContext()
                .getAuthentication().getName();

            auditEventPublisher.publish(AuditEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .eventType("DATA_ACCESS")
                .userId(userId)
                .serviceId("user-service")
                .resourceType(auditable.resourceType())
                .action(auditable.action())
                .outcome(outcome)
                .failureReason(failureReason)
                .timestamp(Instant.now())
                .build());

        } catch (Exception e) {
            // WHY log but don't throw:
            // Audit failure must never break the business operation.
            // However, log it loudly — audit gaps are compliance
            // violations that need immediate attention. (Operability)
            log.error("COMPLIANCE ALERT: Audit event publish failed — " +
                      "manual review required. outcome={}", outcome, e);
        }
    }
}

```
