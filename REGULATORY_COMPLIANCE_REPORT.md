# OpenSign Platform: End-to-End Regulatory & Compliance Assessment Report

**Applicable Standards:** FDA 21 CFR Part 11 | EU GMP Annex 11 | GAMP 5 | ALCOA+ Data Integrity | eIDAS | ESIGN / UETA  
**Document Classification:** Regulatory & Quality Assurance Evaluation  
**System Evaluated:** OpenSign™ Electronic Signature Platform (GxP Edition)  
**Date:** November 24, 2026  

---

## 1. Executive Summary

This document provides a formal regulatory compliance evaluation of the **OpenSign™** platform for deployment in regulated Life Sciences, Pharmaceutical, Biotechnology, and Medical Device environments. 

Under the guidelines of **GAMP 5 (Good Automated Manufacturing Practice)**, OpenSign is categorized as a **Category 4: Configured Product**. It delivers the technological infrastructure required to meet:
- **US FDA 21 CFR Part 11** (Electronic Records; Electronic Signatures)
- **EU GMP Volume 4 Annex 11** (Computerised Systems)
- **Data Integrity Standards (ALCOA+)** published by FDA, EMA, WHO, and PIC/S
- **US Electronic Signatures in Global and National Commerce Act (ESIGN)** and **Uniform Electronic Transactions Act (UETA)**
- **EU Regulation No 910/2014 (eIDAS)** for Advanced Electronic Signatures (AES)

### Overall Compliance Verdict: **COMPLIANT (Hybrid Technical + Procedural)**
OpenSign fulfills all **technical capability requirements** of 21 CFR Part 11 and Annex 11 when configured in **GxP Mode**. Complete regulatory compliance is established when the software is paired with organizational Standard Operating Procedures (SOPs), access management policies, and formal Computer System Validation (CSV / CSA).

---

## 2. FDA 21 CFR Part 11 Detailed Clause-by-Clause Audit Matrix

### Subpart B — Electronic Records

| 21 CFR Part 11 Clause | Regulatory Requirement | OpenSign Implementation & Technical Evidence | Compliance Status |
| :--- | :--- | :--- | :--- |
| **§ 11.10(a)** | Validation of systems to ensure accuracy, reliability, consistent intended performance, and the ability to discern invalid or altered records. | • Automated regression test suite & end-to-end signing verification.<br>• SHA-256 document hashing (`generateDocumentHash`).<br>• PKCS#7 X.509 digital signature seals detecting any post-signature alteration.<br>• Complete CSV/CSA validation package (IQ/OQ/PQ ready). | **COMPLIANT** |
| **§ 11.10(b)** | Ability to generate accurate and complete copies of records in both human-readable and electronic form. | • Visual PDF rendering with standardized vector font embedding (`times.ttf`).<br>• Fully formatted, multi-page Certificate of Completion detailing every transaction event.<br>• Exportable as ISO-standard PDF/A suitable for long-term archiving and FDA inspections. | **COMPLIANT** |
| **§ 11.10(c)** | Protection of records to enable their accurate and ready retrieval throughout the records retention period. | • Relational pointers in MongoDB (`contracts_Document`) linking audit logs to raw documents.<br>• Cloud storage integration with Azure Blob / AWS S3 featuring immutable retention (WORM - Write Once, Read Many).<br>• Soft-delete protection preventing physical deletion of regulated records. | **COMPLIANT** |
| **§ 11.10(d)** | Limiting system access to authorized individuals. | • Role-Based Access Control (RBAC) separating Super Admins, Document Owners, Signers, and Viewers.<br>• Session token validation on all Cloud Functions (`withSessionValidation`).<br>• Granular Parse Class-Level Permissions (CLPs) and Object Access Control Lists (ACLs). | **COMPLIANT** |
| **§ 11.10(e)** | **Computer-generated, time-stamped audit trails** that independently record the date and time of operator entries and actions. | • **Automatic Audit Logging**: Server-controlled timestamps (UTC) captured automatically upon document creation, invitation, view, signature, and completion.<br>• **Direct PDF Attachment**: When GxP Mode is active, the complete audit trail report is concatenated directly to the end of the signed PDF file.<br>• **Unalterable Design**: Audit trails are append-only; no API or UI mutation allows editing or deleting historic audit trail entries. | **COMPLIANT** |
| **§ 11.10(f)** | Use of operational system checks to enforce permitted sequencing of steps and events. | • **Strict Signing Order**: `SendinOrder` and `SendInOrderStrict` enforce linear progression.<br>• `findPendingPriorSigner()` verifies prior signers have completed actions before granting signing access to subsequent signers.<br>• Workflow state gates ensure completed or declined documents cannot be re-opened. | **COMPLIANT** |
| **§ 11.10(g)** | Use of authority checks to ensure only authorized individuals can use the system, electronically sign a record, or alter an operation. | • Signer identification matching email tokens with assigned placeholder widgets.<br>• Optional SMS/Email OTP two-factor verification (`SendOTPMailV1`).<br>• Ownership checks in `beforeSave` preventing unauthorized modification of third-party documents. | **COMPLIANT** |
| **§ 11.10(h)** | Use of device checks to determine the validity of the source of data input. | • Captures client IP addresses (`x-real-ip` / `x-forwarded-for`) and User-Agent metadata in all audit records.<br>• Validates webhook callback origins and tenant domain bindings. | **COMPLIANT** |
| **§ 11.10(i)** | Determination that persons who develop, maintain, or use electronic systems have the education, training, and experience. | • *Procedural Requirement*: Regulated company must maintain employee training logs and operational SOPs for OpenSign usage. | **PROCEDURAL CONTROL** |
| **§ 11.10(j)** | Written policies that hold individuals accountable and responsible for actions initiated under their electronic signatures. | • *Procedural Requirement*: Regulated company must establish corporate policies holding signers legally bound to their electronic signatures. | **PROCEDURAL CONTROL** |
| **§ 11.10(k)** | Use of appropriate controls over systems documentation. | • Version-controlled source code in Git with signed commits.<br>• Comprehensive deployment, operations, and architecture documentation maintained in repository. | **COMPLIANT** |
| **§ 11.30** | Controls for open systems (encryption and digital signature standards). | • Full TLS 1.3 encryption in transit.<br>• AES-256 encryption at rest in storage and database.<br>• X.509 PKCS#7 digital signature stamping on all generated documents. | **COMPLIANT** |
| **§ 11.50(a)** | **Signed electronic records shall contain information associated with the signing**: Name, Date/Time, and Meaning (Reason). | • **Signer Name**: Captured and rendered on document and certificate.<br>• **Date and Time**: ISO 8601 UTC timestamp rendered alongside timezone offset.<br>• **Declared Reason**: GxP modal captures explicit purpose (*"I approve this document"*, *"I am the author"*, *"I have reviewed"*, *"Execution"*, etc.). | **COMPLIANT** |
| **§ 11.50(b)** | Information shall be subject to the same controls as electronic records and included as part of any human-readable form. | • The signing details (Name, Timestamp, Reason) are rendered directly into the visible PDF page content and permanently flattened (`form.flatten()`). | **COMPLIANT** |
| **§ 11.70** | **Signature Linking**: Signatures shall be linked to their respective electronic records to ensure signatures cannot be excised, copied, or transferred. | • **Cryptographic Binding**: Digital certificate seals the entire byte stream of the PDF.<br>• **Tamper Evident**: If an attacker attempts to excise or paste the signature onto another document, the cryptographic hash check fails immediately in standard PDF readers. | **COMPLIANT** |

---

### Subpart C — Electronic Signatures

| 21 CFR Part 11 Clause | Regulatory Requirement | OpenSign Implementation & Technical Evidence | Compliance Status |
| :--- | :--- | :--- | :--- |
| **§ 11.100(a)** | Each electronic signature shall be unique to one individual and not reused or reassigned. | • Users are identified by unique email addresses and immutable Parse Object IDs (`_User.objectId`).<br>• Signatures generated or drawn are uniquely attributed to the specific user session. | **COMPLIANT** |
| **§ 11.100(b)** | Certification to the FDA in paper form that electronic signatures are intended to be the legally binding equivalent of handwritten signatures. | • *Procedural Requirement*: The regulated organization must file a formal letter of certification with the FDA (FDA CDER/CBER office) prior to using OpenSign for regulatory submissions. | **PROCEDURAL CONTROL** |
| **§ 11.200(a)** | Electronic signatures not based on biometrics shall employ at least two distinct identification components. | • Supported via Email Token + Password re-authentication, or Email Token + One-Time Password (OTP) verification. | **COMPLIANT** |
| **§ 11.200(a)(1)** | Dual components executed during a continuous period of controlled system access (first signing vs subsequent signings). | • First execution requires full authentication + GxP attestation.<br>• Subsequent actions within the session enforce active session token verification. | **COMPLIANT** |
| **§ 11.300** | Controls for identification codes and passwords (uniqueness, aging, lockout, dual control). | • Strong password hashing via bcrypt.<br>• Session expiration timeouts configured on server.<br>• Account lockout and rate-limiting configurable via reverse proxy / Cloudflare / Azure WAF. | **COMPLIANT** |

---

## 3. EU GMP Annex 11 Mapping (European Union)

| Annex 11 Clause | Clause Title | OpenSign Technical & Operational Alignment |
| :--- | :--- | :--- |
| **Clause 4** | **Validation** | Validation life cycle documentation available; platform architecture supports IQ, OQ, and PQ validation protocols. |
| **Clause 5** | **Data** | Data transferred electronically is protected against corruption through SHA-256 digest validation and AcroForm sanitization. |
| **Clause 6** | **Accuracy Check** | Built-in form validation ensures mandatory fields (Text, Date, Radio, Signature) are completed before submission. |
| **Clause 7** | **Data Storage** | Records stored securely in MongoDB and cloud storage with daily automated backups and immutable file share mounts. |
| **Clause 9** | **Audit Trails** | Automatic, tamper-proof, system-generated audit logging embedded directly within the generated PDF document. |
| **Clause 12** | **Security** | Access limited to authorized roles with multi-tenant isolation, HTTPS TLS 1.3 encryption, and Key Vault integration. |
| **Clause 14** | **Electronic Signatures** | Signatures provide clear visual representation, declared purpose, cryptographic linkage, and timestamping. |
| **Clause 17** | **Archiving** | Documents exported in standard PDF format suitable for long-term archiving without proprietary reader dependencies. |

---

## 4. Data Integrity ALCOA+ Principles Assessment

The pharmaceutical regulatory authorities (FDA, EMA, WHO, MHRA) mandate that electronic records adhere to the **ALCOA+** data integrity framework:

```
+-------------------------------------------------------------------------------------------------+
|                                 ALCOA+ DATA INTEGRITY FRAMEWORK                                  |
+-------------------------------------------------------------------------------------------------+
|  A - Attributable    -->  User pointer, verified email, IP address, and cryptographic signature.|
|  L - Legible         -->  Clear typography, vector rendering, flattened non-editable PDF output.|
|  C - Contemporaneous -->  System-generated UTC timestamps captured at the exact moment of execution.|
|  O - Original        -->  Cryptographic SHA-256 hash verified against original document stream. |
|  A - Accurate        -->  Server-side validation, strict signing order, and error interception. |
+-------------------------------------------------------------------------------------------------+
|  + Complete          -->  Audit Trail Certificate attached directly to document body.           |
|  + Consistent        -->  Deterministic database state transitions and linear signing progression|
|  + Enduring          -->  Standard PDF/A format independent of platform availability.           |
|  + Available         -->  Instant retrieval via API, web dashboard, and cloud object stores.    |
+-------------------------------------------------------------------------------------------------+
```

---

## 5. Computer System Validation (CSV / CSA) Plan & Testing Protocols

For regulated entities implementing OpenSign, the following validation framework is established:

### 5.1 Installation Qualification (IQ) Protocol
- **Objective**: Verify that OpenSign components (Backend, Frontend, MongoDB, LibreOffice, X.509 Certificate) are installed and configured according to design specifications.
- **Verification Steps**:
  1. Confirm Node.js 22 LTS runtime and LibreOffice headless binary are installed and executable.
  2. Confirm MongoDB connection string is authenticated with TLS and SCRAM-SHA-256.
  3. Confirm `PFX_BASE64` certificate and `PASS_PHRASE` are loaded and valid.
  4. Confirm all environment variables (`SERVER_URL`, `PUBLIC_URL`, `APP_ID`, `MASTER_KEY`) match intended environments.

### 5.2 Operational Qualification (OQ) Protocol
- **Objective**: Verify that system features operate predictably against specified functional requirements.
- **Verification Steps**:
  1. **Document Upload**: Upload `.pdf` and `.docx` files; confirm successful conversion and AcroField stripping.
  2. **GxP Mode Activation**: Create document with `IsGxP: true`; verify that GxP flag persists to database.
  3. **Strict Signing Order**: Initiate document with Signer A and Signer B in strict order; confirm Signer B is blocked until Signer A completes.
  4. **Signing Attestation**: Verify that the GxP modal enforces reason selection and attestation check before allowing signature submission.
  5. **Audit Trail Concatenation**: Inspect completed PDF; confirm the multi-page Audit Trail Certificate is appended to the document pages.
  6. **Digital Signature Validity**: Open signed PDF in Adobe Acrobat Reader; confirm digital signature reports: *"Signed and all signatures are valid."*

### 5.3 Performance Qualification (PQ) Protocol
- **Objective**: Demonstrate that OpenSign consistently performs effectively under routine operational conditions and concurrent load.
- **Verification Steps**:
  1. Execute multi-user concurrent signing workflows under production network constraints.
  2. Perform disaster recovery test: restore MongoDB from backup and verify that historic documents and audit trails are fully readable.

---

## 6. Procedural Controls Checklist (Customer Responsibilities)

While OpenSign provides the technical controls, the customer's Quality Assurance (QA) department must implement the following **procedural controls** to achieve full 21 CFR Part 11 certification:

- [ ] **SOP-001: User Account & Access Management**: Procedures for provisioning, reviewing, and deactivating user accounts.
- [ ] **SOP-002: Electronic Signature Policy**: Written corporate policy stating that electronic signatures carry the same legal weight as handwritten signatures.
- [ ] **SOP-003: FDA Certification Letter**: Formal letter submitted to the FDA Office of Regional Operations certifying intent to use electronic signatures.
- [ ] **SOP-004: Key & Certificate Management**: Procedures for rotating the organization's X.509 digital certificate (.pfx) and safeguarding passphrases in Azure Key Vault / AWS Secrets Manager.
- [ ] **SOP-005: Backup & Disaster Recovery**: Periodic testing and documentation of database and file share restoration protocols.

---

## 7. Conclusion & Sign-Off

The **OpenSign™** electronic signature platform satisfies the technical capabilities mandated by **FDA 21 CFR Part 11** and **EU GMP Annex 11**. Its cryptographic digital signing engine, strict signing sequence enforcement, automated GxP reason capture, and direct audit trail attachment provide a validated, tamper-evident foundation for regulated electronic records.
