```java
// ============================================================
// FILE: user-service/domain/User.java
// PROPERTY: Trust + Consistency + Change
//
// WHY @Audited on entity: Hibernate Envers creates a shadow
// audit table (users_aud) with every version of every row.
// This is your SOX evidence — who changed what, and when.
// Without this, you'd need to reconstruct history from logs,
// which is unreliable and rejected by SOX auditors.
//
// WHY @Convert per field: Encryption is field-specific, not
// table-wide. Fields like `accountStatus` are not PII — don't
// encrypt them. Encrypting everything wastes CPU and makes
// DB-level queries (non-PII filters) impossible.
// ============================================================
package com.compliance.user.domain;

import com.compliance.shared.annotation.*;
import com.compliance.user.converter.EncryptedStringConverter;
import jakarta.persistence.*;
import lombok.*;
import org.hibernate.envers.Audited;
import org.hibernate.envers.NotAudited;
import org.springframework.data.annotation.*;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import java.time.Instant;

@Entity
@Table(name = "users")
@Audited
@EntityListeners(AuditingEntityListener.class)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    // WHY UUID not auto-increment: Auto-increment IDs are
    // sequential and predictable — IDOR attacks become trivial.
    // UUIDs are unguessable. This is a Trust property decision.
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @PersonalData(
        classification = DataClassification.PII,
        purpose = "User identification and service communication",
        retentionDays = 730,
        erasable = true
    )
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "email_encrypted", nullable = false)
    private String email;

    @PersonalData(
        classification = DataClassification.PII,
        purpose = "Display and personalization",
        erasable = true
    )
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "full_name_encrypted")
    private String fullName;

    @PersonalData(
        classification = DataClassification.SENSITIVE_PII,
        purpose = "Identity verification — required by KYC regulation",
        erasable = false   // WHY false: KYC records may have
                           // legal hold obligations overriding GDPR erasure.
                           // Erasure service checks this flag before acting.
    )
    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "national_id_encrypted")
    private String nationalId;

    // WHY NOT annotated/encrypted: Not PII — safe to store plain,
    // safe to filter/index in DB without performance penalty.
    @Column(nullable = false)
    private String accountStatus;

    // WHY store separately from consentRecord:
    // Account creation timestamp has legal significance
    // independent of consent. Keep them decoupled.
    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    @Column(nullable = false, updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;

    // WHY nullable erasedAt:
    // Presence of erasedAt means the record was pseudonymized.
    // Downstream services check this field to skip processing
    // erased users — avoids re-processing ghost records.
    @Column
    private Instant erasedAt;

    // WHY NOT @Audited on consentRecord relation:
    // Consent has its own audit trail via ConsentRecord entity.
    // Double-auditing wastes storage and creates join complexity
    // in audit queries. Single source of truth = Consistency.
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @NotAudited
    private ConsentRecord consentRecord;
}


// ============================================================
// FILE: user-service/domain/ConsentRecord.java
// PROPERTY: Consistency + Change + Operability
//
// WHY consentVersion field: GDPR requires users consent to a
// specific version of your privacy policy. When policy changes,
// you need to know which users consented to which version, so
// you can re-request consent for affected users only.
// Without versioning, you'd have to re-consent everyone — an
// operationally catastrophic and legally unnecessary action.
// ============================================================
package com.compliance.user.domain;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "consent_records")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ConsentRecord {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String userId;

    // WHY granular consent fields instead of one boolean:
    // GDPR requires consent to be specific — blanket consent
    // ("I agree to everything") is not valid. Each purpose
    // must be independently consentable and withdrawable.
    private boolean marketingConsent;
    private boolean analyticsConsent;
    private boolean thirdPartyDataSharingConsent;

    @Column(nullable = false)
    private Instant consentGivenAt;

    private Instant consentWithdrawnAt;

    // WHY store policy version:
    // When you update privacy policy v2 → v3, you can query:
    // "SELECT * FROM consent_records WHERE consentVersion != 'v3'"
    // to find users needing re-consent. Machine-readable Change management.
    @Column(nullable = false)
    private String consentVersion;

    // WHY store IP + userAgent:
    // Proof of consent origin. In a legal dispute, you must
    // demonstrate consent was freely given by the actual user,
    // not fabricated. These fields are your evidence. (Trust)
    @Column(nullable = false)
    private String ipAddress;

    private String userAgent;
}


// ============================================================
// FILE: user-service/outbox/OutboxEvent.java
// PROPERTY: Consistency + Dependency
//
// WHY Transactional Outbox pattern:
// Problem: You save a User to DB and then publish a Kafka event.
// If Kafka is down after DB commit, the event is lost forever.
// If you publish BEFORE DB commit and the DB fails, you have a
// phantom event for a non-existent user.
//
// Solution: Write the event to an OUTBOX table in the SAME DB
// transaction as the business data. A separate poller reliably
// publishes outbox events to Kafka with retry. Now DB and Kafka
// are always consistent — no lost events, no phantom events.
// This fixes the Dependency property: Kafka being down no longer
// breaks user creation — it just delays event delivery.
// ============================================================
package com.compliance.user.outbox;

import jakarta.persistence.*;
import lombok.*;
import java.time.Instant;

@Entity
@Table(name = "outbox_events")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    // WHY aggregateType + aggregateId:
    // Standard outbox schema. Lets consumers filter by type
    // without deserializing the payload. Efficient routing.
    @Column(nullable = false)
    private String aggregateType;   // "User", "ConsentRecord"

    @Column(nullable = false)
    private String aggregateId;

    @Column(nullable = false)
    private String eventType;       // "UserCreated", "ConsentWithdrawn"

    @Column(nullable = false, columnDefinition = "TEXT")
    private String payload;         // JSON serialized event

    @Column(nullable = false)
    private String topic;           // target Kafka topic

    @Column(nullable = false)
    private Instant createdAt;

    // WHY processedAt nullable:
    // Poller queries WHERE processedAt IS NULL — unprocessed events.
    // Marking processed instead of deleting preserves event history
    // for debugging and reprocessing if needed. (Operability)
    private Instant processedAt;

    private int retryCount;
}


// ============================================================
// FILE: user-service/converter/EncryptedStringConverter.java
// PROPERTY: Trust + Latency + Consistency
//
// WHY JPA AttributeConverter:
// Encryption/decryption happens transparently at the persistence
// layer. Business logic never deals with cipher text — it always
// sees plain text. This ensures EVERY write to this column is
// encrypted, with zero chance of a developer accidentally
// bypassing it. (Trust + Consistency)
//
// WHY the converter delegates to EncryptionService:
// The converter itself has no crypto logic. EncryptionService
// is the single implementation point. When we rotate algorithms
// or keys, we change one class — not every converter. (Change)
// ============================================================
package com.compliance.user.converter;

import com.compliance.shared.encryption.EncryptionService;
import jakarta.persistence.AttributeConverter;
import jakarta.persistence.Converter;
import lombok.RequiredArgsConstructor;

@Converter
@RequiredArgsConstructor
public class EncryptedStringConverter
    implements AttributeConverter<String, String> {

    private final EncryptionService encryptionService;

    @Override
    public String convertToDatabaseColumn(String plainText) {
        return encryptionService.encrypt(plainText);
    }

    @Override
    public String convertToEntityAttribute(String cipherText) {
        return encryptionService.decrypt(cipherText);
    }
}

```
