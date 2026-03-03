```java
// ============================================================
// FILE: financial-service/domain/FinancialTransaction.java
// PROPERTY: Trust + Observability + Change
//
// WHY @Audited at entity level:
// Hibernate Envers intercepts every INSERT/UPDATE/DELETE at the
// persistence layer and writes a version to `financial_transactions_aud`.
// This happens even if a developer bypasses the service layer.
// It's a Trust property control — compliance enforcement at the
// infrastructure layer, not just the application layer.
//
// WHY @NotAudited on specific fields:
// Auditing every field has storage cost. Volatile non-SOX fields
// like display values or cache fields would bloat the audit table
// with meaningless versions. (Economics + Capacity)
// ============================================================
package com.compliance.financial.domain;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.envers.*;
import org.springframework.data.annotation.*;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;
import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(name = "financial_transactions")
@Audited
@EntityListeners(AuditingEntityListener.class)
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class FinancialTransaction {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false, precision = 19, scale = 4)
    // WHY BigDecimal not double:
    // Floating point (double) has rounding errors. For financial
    // amounts, 0.1 + 0.2 != 0.3 in floating point arithmetic.
    // SOX requires financial accuracy — BigDecimal is mandatory.
    private BigDecimal amount;

    @Column(nullable = false, length = 3)
    private String currency;     // ISO 4217 currency code

    @Column(nullable = false)
    private String transactionType;  // DEBIT, CREDIT, TRANSFER

    @Column(nullable = false)
    private String status;           // PENDING_APPROVAL, APPROVED, REJECTED

    @Column(nullable = false)
    private String userId;           // pseudonymized reference only

    // WHY store createdBy/approvedBy as separate columns:
    // SOX Segregation of Duties requires proof that creator != approver.
    // Envers tracks changes but we need this duality queryable directly.
    @CreatedBy
    @Column(nullable = false, updatable = false)
    private String createdBy;

    @Column
    private String approvedBy;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @Column
    private Instant approvedAt;

    @Column
    private String rejectionReason;

    // WHY NOT @Audited here:
    // This is a computed display field, not a SOX-material field.
    // Auditing it would create a version entry on every UI refresh.
    @NotAudited
    @Transient
    private String displayLabel;
}


// ============================================================
// FILE: financial-service/service/TransactionService.java
// PROPERTY: Trust + Consistency + Observability
//
// WHY Segregation of Duties enforced in service code:
// Spring Security's @PreAuthorize is the first line of defense,
// but we also enforce SOD in business logic. Why both?
// @PreAuthorize can be misconfigured. Business logic check is
// the safety net. Two independent controls = defense in depth.
// (Trust property — never rely on a single enforcement point)
// ============================================================
package com.compliance.financial.service;

import com.compliance.financial.domain.FinancialTransaction;
import com.compliance.financial.repository.TransactionRepository;
import com.compliance.shared.annotation.Auditable;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.Instant;

@Slf4j
@Service
@RequiredArgsConstructor
public class TransactionService {

    private final TransactionRepository transactionRepository;

    @PreAuthorize("hasRole('FINANCE_CREATOR')")
    @Auditable(resourceType = "FinancialTransaction", action = "CREATE")
    @Transactional
    public FinancialTransaction createTransaction(FinancialTransaction txn) {
        String creator = currentUser();
        txn.setCreatedBy(creator);
        txn.setStatus("PENDING_APPROVAL");

        // WHY status starts as PENDING_APPROVAL and not APPROVED:
        // SOX dual-control principle. No transaction is self-approved.
        // Every financial record must pass through a second person.
        // The status machine enforces this flow structurally.
        FinancialTransaction saved = transactionRepository.save(txn);
        log.info("Transaction created — id={}, creator={}, amount={}",
            saved.getId(), creator, saved.getAmount());
        return saved;
    }

    @PreAuthorize("hasRole('FINANCE_APPROVER')")
    @Auditable(resourceType = "FinancialTransaction", action = "APPROVE")
    @Transactional
    public FinancialTransaction approveTransaction(String txnId) {
        FinancialTransaction txn = transactionRepository.findById(txnId)
            .orElseThrow(() -> new TransactionNotFoundException(txnId));

        String approver = currentUser();

        // WHY check creator != approver in code, not just in @PreAuthorize:
        // @PreAuthorize checks the role, not the identity relative to the
        // record. This business logic check is the SOX-specific control.
        // A user with FINANCE_APPROVER role must still not approve their
        // own transaction. This is Segregation of Duties enforcement.
        if (txn.getCreatedBy().equals(approver)) {
            log.error("SOX VIOLATION ATTEMPT — userId={} tried to approve " +
                      "their own transaction txnId={}", approver, txnId);
            throw new SoxViolationException(
                "Creator cannot approve their own transaction. " +
                "SOX Segregation of Duties violation prevented."
            );
        }

        if (!"PENDING_APPROVAL".equals(txn.getStatus())) {
            throw new InvalidTransactionStateException(
                "Transaction " + txnId + " is not in PENDING_APPROVAL state"
            );
        }

        txn.setApprovedBy(approver);
        txn.setApprovedAt(Instant.now());
        txn.setStatus("APPROVED");

        FinancialTransaction approved = transactionRepository.save(txn);
        log.info("Transaction approved — id={}, approver={}", txnId, approver);
        return approved;
    }

    @PreAuthorize("hasAnyRole('FINANCE_CREATOR','FINANCE_APPROVER','AUDITOR')")
    @Auditable(resourceType = "FinancialTransaction", action = "READ")
    @Transactional(readOnly = true)
    public FinancialTransaction getTransaction(String txnId) {
        return transactionRepository.findById(txnId)
            .orElseThrow(() -> new TransactionNotFoundException(txnId));
    }

    private String currentUser() {
        return SecurityContextHolder.getContext()
            .getAuthentication().getName();
    }
}


// ============================================================
// FILE: financial-service/service/SoxReportService.java
// PROPERTY: Operability + Observability
//
// WHY a dedicated SOX report service:
// SOX auditors will ask: "Show me all access to financial records
// between Jan 1 and Mar 31 by non-admin users." Without a
// pre-built query interface, this becomes a multi-day engineering
// task during an audit — a catastrophic operability failure.
// This service makes audit evidence self-service. (Operability)
// ============================================================
package com.compliance.financial.service;

import com.compliance.financial.domain.FinancialTransaction;
import com.compliance.financial.repository.TransactionRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.hibernate.envers.*;
import org.hibernate.envers.query.*;
import java.time.Instant;
import java.util.List;

@Service
@RequiredArgsConstructor
public class SoxReportService {

    private final AuditReader auditReader;     // Hibernate Envers audit reader
    private final TransactionRepository transactionRepository;

    // WHY AUDITOR role can read but not modify:
    // SOX requires auditors have read access to all financial data
    // but zero ability to modify it. Role separation at the method
    // level enforces this — auditors literally cannot call the
    // approve/reject methods due to @PreAuthorize.
    @PreAuthorize("hasRole('AUDITOR')")
    @Transactional(readOnly = true)
    public List<Object[]> getTransactionHistory(String txnId) {
        // WHY use Envers AuditReader here:
        // This returns every version of the transaction — including
        // who changed it, when, and what it looked like before.
        // This is the SOX evidence trail: immutable, complete, DB-backed.
        AuditQuery query = auditReader.createQuery()
            .forRevisionsOfEntity(FinancialTransaction.class, false, true)
            .add(AuditEntity.id().eq(txnId))
            .addOrder(AuditEntity.revisionNumber().asc());

        return query.getResultList();
    }

    @PreAuthorize("hasRole('AUDITOR')")
    @Transactional(readOnly = true)
    public List<FinancialTransaction> getPendingApprovalTransactions() {
        // WHY expose this specifically:
        // SOX requires dual control. Transactions sitting in
        // PENDING_APPROVAL for too long indicate a control failure.
        // Auditors must be able to identify and escalate these.
        return transactionRepository.findByStatus("PENDING_APPROVAL");
    }

    @PreAuthorize("hasRole('AUDITOR')")
    @Transactional(readOnly = true)
    public SoxComplianceReport generateComplianceReport(Instant from, Instant to) {
        // WHY pre-built report and not raw query:
        // Auditors are not engineers. They need structured reports,
        // not SQL access. This method assembles the report from
        // multiple data sources and returns a compliance-ready structure.
        long totalTransactions = transactionRepository.countByCreatedAtBetween(from, to);
        long selfApprovalAttempts = transactionRepository
            .countSelfApprovalAttempts(from, to);  // should always be 0
        long pendingOver24h = transactionRepository
            .countPendingApprovalOlderThan(Instant.now().minusSeconds(86400));

        return SoxComplianceReport.builder()
            .reportPeriodFrom(from)
            .reportPeriodTo(to)
            .totalTransactions(totalTransactions)
            .selfApprovalAttempts(selfApprovalAttempts)  // non-zero = SOX violation
            .pendingApprovalOver24h(pendingOver24h)
            .generatedAt(Instant.now())
            .build();
    }
}

```
