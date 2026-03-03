```
// ============================================================
// FILE: shared-compliance-lib/annotation/PersonalData.java
// PROPERTY: Consistency + Change
//
// WHY: Without a shared annotation, each team defines PII
// differently. Some fields get encrypted, others don't.
// Consistency breaks silently — no runtime error, just an
// unencrypted SSN in your database discovered during an audit.
// Change: CI/CD gate reflects on this annotation to catch
// any new PII field added without proper tagging at build time.
// ============================================================
package com.compliance.shared.annotation;

import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface PersonalData {

    DataClassification classification() default DataClassification.PII;

    // WHY purpose: GDPR Article 5 requires data minimization and
    // purpose limitation. Purpose must be documented per field,
    // not just per entity. This makes it machine-readable.
    String purpose();

    // WHY erasable: Some PII (e.g., name on a financial record)
    // cannot be erased due to SOX. Marking it false here signals
    // the erasure service to pseudonymize rather than null it.
    boolean erasable() default true;

    int retentionDays() default 365;
}


// ============================================================
// FILE: shared-compliance-lib/annotation/DataClassification.java
// ============================================================
package com.compliance.shared.annotation;

public enum DataClassification {
    SENSITIVE_PII,  // SSN, passport — encrypt + strict access
    PII,            // name, email — encrypt + audit access
    INTERNAL,       // order IDs — audit access, no encryption
    PUBLIC          // catalog data — no restrictions
}


// ============================================================
// FILE: shared-compliance-lib/annotation/Auditable.java
// PROPERTY: Observability + Trust (SOX)
//
// WHY: AOP-driven auditing means no developer can "forget"
// to add audit logging. The annotation declaratively says
// "this method accesses sensitive data" and the aspect
// handles the rest — consistently, without human error.
// ============================================================
package com.compliance.shared.annotation;

import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Auditable {
    String resourceType();
    String action();  // READ, CREATE, UPDATE, DELETE, EXPORT
}


// ============================================================
// FILE: shared-compliance-lib/encryption/EncryptionService.java
// PROPERTY: Trust + Dependency + Latency
//
// WHY AES-256-GCM: GCM mode provides both confidentiality AND
// authenticity (built-in MAC). If data is tampered with in DB,
// decryption fails — giving us tamper detection for free.
// AES-CBC does NOT provide this — a critical difference for
// SOX and GDPR data integrity requirements.
//
// WHY random IV per encryption: Reusing IVs with the same key
// in GCM mode is catastrophic — it breaks confidentiality
// completely. Per-call random IV prevents this.
//
// WHY Dependency: Key is injected — impl can swap to Vault-backed
// encryption-as-a-service without changing callers.
// ============================================================
package com.compliance.shared.encryption;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import javax.crypto.Cipher;
import javax.crypto.spec.*;
import java.nio.ByteBuffer;
import java.security.SecureRandom;
import java.util.Base64;

@Slf4j
@Service
public class EncryptionService {

    private final byte[] keyBytes;

    // WHY constructor injection of key bytes:
    // Eagerly validates key on startup. A misconfigured key
    // fails at boot, not at runtime when a user's SSN is
    // about to be stored. Fail-fast = safer system.
    public EncryptionService(@Value("${compliance.encryption.key}") String base64Key) {
        this.keyBytes = Base64.getDecoder().decode(base64Key);
        if (keyBytes.length != 32) {
            // WHY throw at construction: 256-bit key is non-negotiable.
            // A 128-bit key silently weakens security. Enforce here.
            throw new IllegalStateException(
                "Encryption key must be 256-bit (32 bytes). " +
                "Got: " + keyBytes.length + " bytes. " +
                "Compliance requirement violated at startup."
            );
        }
    }

    public String encrypt(String plainText) {
        if (plainText == null) return null;
        try {
            // WHY 12-byte IV: NIST recommends 96-bit (12-byte) IV
            // for GCM. Larger IVs require additional processing.
            byte[] iv = new byte[12];
            new SecureRandom().nextBytes(iv);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE,
                new SecretKeySpec(keyBytes, "AES"),
                new GCMParameterSpec(128, iv));   // 128-bit auth tag

            byte[] encrypted = cipher.doFinal(plainText.getBytes("UTF-8"));

            // WHY prepend IV to ciphertext:
            // IV is not secret — it just must be unique per encryption.
            // Storing it with the ciphertext means we never need a
            // separate IV store, eliminating a dependency and
            // simplifying key rotation.
            ByteBuffer buf = ByteBuffer.allocate(iv.length + encrypted.length);
            buf.put(iv);
            buf.put(encrypted);
            return Base64.getEncoder().encodeToString(buf.array());
        } catch (Exception e) {
            // WHY not log plainText in error: Never log PII data,
            // even in error paths. This is a GDPR requirement and
            // also prevents sensitive data from appearing in your
            // log aggregation system (Observability risk).
            log.error("Encryption failed for field — data NOT stored");
            throw new EncryptionException("Field encryption failed", e);
        }
    }

    public String decrypt(String cipherText) {
        if (cipherText == null) return null;
        try {
            byte[] decoded = Base64.getDecoder().decode(cipherText);
            ByteBuffer buf = ByteBuffer.wrap(decoded);

            byte[] iv = new byte[12];
            buf.get(iv);
            byte[] encrypted = new byte[buf.remaining()];
            buf.get(encrypted);

            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE,
                new SecretKeySpec(keyBytes, "AES"),
                new GCMParameterSpec(128, iv));

            return new String(cipher.doFinal(encrypted), "UTF-8");
        } catch (Exception e) {
            log.error("Decryption failed — possible data tampering detected");
            // WHY re-throw as domain exception: Upper layers need
            // to distinguish between "data not found" and "data tampered"
            // for compliance incident logging.
            throw new EncryptionException("Field decryption failed — integrity check failed", e);
        }
    }
}


// ============================================================
// FILE: shared-compliance-lib/audit/AuditEventPublisher.java
// PROPERTY: Consistency + Dependency + Flow
//
// WHY interface: All 3 services use the same AuditEventPublisher
// contract. If we later switch from Kafka to a different broker,
// we swap the implementation — no service code changes.
// This is Dependency isolation applied to compliance infrastructure.
//
// WHY async: Writing audit logs synchronously on the hot path
// means a slow Kafka broker adds latency to every user request.
// Audit logs are critical but not on the critical response path.
// ============================================================
package com.compliance.shared.audit;

import lombok.Builder;
import lombok.Data;
import java.time.Instant;

public interface AuditEventPublisher {
    void publish(AuditEvent event);

    @Data
    @Builder
    class AuditEvent {
        private String eventId;        // UUID — idempotency key
        private String traceId;        // links to distributed trace
        private String eventType;      // DATA_ACCESS, CONSENT_CHANGE, ERASURE
        private String userId;         // who triggered the action
        private String serviceId;      // which service
        private String resourceType;   // User, FinancialTransaction
        private String resourceId;     // which specific record
        private String action;         // READ, UPDATE, DELETE
        private String outcome;        // SUCCESS, FAILURE
        private String failureReason;
        private Instant timestamp;
        private String ipAddress;
        private String sessionId;
        // WHY recordStateHash: SOX requires you can prove what the
        // record state was AT THE TIME of access. Hash lets auditors
        // verify no post-hoc tampering without storing full snapshots.
        private String recordStateHash;
    }
}
```
