## L3-NFRS-APP: Non-Functional Requirements Specification (NFRS) for APP

### 1. Purpose & Scope

This Level-3 NFRS refines the solution-wide quality attributes in **L1-NFRS** for the mobile application component (**APP**). It establishes concrete, testable targets for performance, scalability, security, reliability, usability, compliance and environmental constraints that must be met by the Flutter code-base and its delivery pipeline. All figures explicitly align with the 5 000-user / 1 000-concurrent-session scale committed by the client and avoid any over-engineering beyond that scope.

### 2. Referenced Documents

• L1-NFRS – Non-Functional Requirements (solution-wide)  
• L1-HLD – High-Level Design  
• L3-KD-APP – Component Key Decisions  
• L3-FRS-APP – Functional Requirements  
• Consolidated Client Clarifications #1-#11 (2024-05-28)

---

### 3. Performance Requirements

The APP is strictly online; end-to-end responsiveness therefore depends on both client code and backend latency.

| ID | Metric | Target | Notes |
|----|--------|--------|-------|
| PERF-01 | Cold-start to first frame | ≤ 3 s | Measured on mid-range devices |
| PERF-02 | Screen render after API call | ≤ 2 s | Including network latency |
| PERF-03 | Image upload (2 MB) | ≤ 5 s | Via SAS URL on 4G |

### 4. Scalability & Concurrency

1. Tested with **1 000 simultaneous logged-in devices**, each issuing a peak of **4 requests/min**.  
2. The HTTP client pool SHALL allow at least **4 parallel connections** per host to avoid head-of-line blocking.  
3. Idempotency logic (L3-KD-APP C#4) MUST guarantee duplicate-free writes under retry storms (≥ 10 retries/sec).  
4. No client-side caching of catalogue or rule data is introduced; any future scale increase will require backend-centric optimisation, not app redesign.

### 5. Security Requirements

The security posture follows the "encryption-everywhere, no local business data" principle.

| Area | Requirement |
|------|-------------|
| Transport | All traffic >= **TLS 1.2**; system trusts OS root store, **no certificate pinning** (Clarification #5). |
| AuthN | 6-digit OTP (SMS and Email for all users); `/auth/otp/verify` issues RS256-signed JWT (60 min) + refresh token (12 h). |
| AuthZ | JWT carries `role`, `store` claims; client attaches `X-Store-Id` on context switch. |
| Secrets | JWT, refresh token, idempotency key stored only in Secure Keystore / Keychain; JWT passed via Authorization header; never written to filesystem. |
| Root/JB devices | Detect at launch; **warn and allow** (Clarification #6) while tagging session header `X-Device-Integrity: degraded`. |
| Code Hardening | Build with `flutter build --obfuscate --split-debug-info` and Android R8 iOS bit-code stripping; **no** Play Integrity / App Attest checks Phase-1 (Clarification #7). |
| Logging | Debug builds may persist sandbox logs; Release builds emit in-memory logs only and redact PII (Clarification #8). |
| Data at rest on device | None (except credentials) ─ fulfills DPDP "reasonable security" expectation. |
| OTP Storage | OTP codes stored as plaintext without encryption in PostgreSQL backend (accepted risk), protected only by database-level encryption. |

### 6. Reliability & Availability

| ID | Metric | Target |
|----|--------|--------|
| REL-01 | API error-handling | All 4xx/5xx errors display user-friendly messages |
| REL-02 | Network loss handling | Show offline banner and disable write actions |
| REL-03 | Transaction integrity | Idempotent submission via request_uuid |

### 7. Usability & Accessibility

1. Layouts adaptive from **4.7″ to 10″** screens; minimum touch target **48 dp**.  
2. Text honours OS font-scale; colour contrast ≥ 4.5:1 for primary actions.  
3. VoiceOver / TalkBack focus order defined on interactive widgets; no formal WCAG audit in scope (client accepted).  
4. Error messages human-readable, action-oriented; no stack traces.  
5. Mandatory update flow: if backend returns **HTTP 426** the app compares `X-Min-Version`; builds older than **N-2** versions show a blocking update dialog (Clarification #9).

### 8. Maintainability & Supportability

• Standardised on flutter_bloc 8.x and Clean-Architecture layers (KD-APP C#5).
• Mandatory CI lints: dart analyze, flutter test, and obfuscation flag presence.
• All user-visible strings centralised in strings.dart to ease future expansion.

### 9. Compliance & Legal

1. **DPDP (India)** – phone number & OTP data processed under "specified purpose" exemption; no on-device persistence.  
2. **App-store privacy manifests**: declare phone number,  OTP messages, sandbox debug logs only (Clarification #11).  
3. Third-party licences bundled in *About* screen; only MIT / Apache 2.0 permitted (L1-NFRS §8).  
4. No GDPR obligations Phase-1 (client statement L1-NFRS §8).

### 10. Environmental Constraints

| Aspect | Constraint |
|--------|------------|
| Devices | Android 11+ (API 30), iOS 16+; 2 GB RAM minimum. |
| Network | Designed for 4 G; no explicit 3 G performance guarantee (Clarification #10). |
| OS Services | camera & gallery intents; secure storage APIs. |
| Build Chain | Flutter 3.2.x, Dart 3.2.x, GitHub Actions CI → Fastlane release channels. |
| Key Storage | Keys stored in encrypted App-Service settings without Azure Key Vault. |

### 11. Dependencies and Requirements

1. **Backend (BCKND)** – REST / JSON `/v1` endpoints, JWT semantics, idempotency key contract.  
2. **MSG91** – SMS and Email gateway for OTP delivery; outages propagate read-only banner (KD-APP C#6).  
3. **Azure Blob Storage (SAS)** – Backend issues SAS URL, client uploads file using SAS, client calls backend to register the uploaded file.  
4. **L1-NFRS** baseline: no formal latency SLOs, single-region availability, vertical scale-up.  
5. **L2-LLD-IC** – header conventions (`X-Store-Id`, `X-App-Version`, `meta.traceId`), error-envelope schema.  
6. **Mobile OS trust store** – TLS root CA validation; no custom CA bundles.
7. **OTP Policies** – Login OTP: 10-min TTL, max 5 attempts within 10 minutes, 15-minute lockout, 3 resends per 15 minutes with 60s cooldown. Customer OTP: 5-min TTL, max 3 attempts, 15-minute lockout, 3 resends per 15 minutes with 60s cooldown.
8. **Role Access** – Store Users (S) and Parent Store Users (P) only; KAM users have web-only access with no mobile app support.

---

### 12. Verification & Acceptance

Basic smoke testing will verify:

- App starts within 3 seconds on mid-range devices
- All error scenarios display appropriate messages
- Transactions cannot be duplicated
- App functions correctly on Android 11+ and iOS 16+

---

*End of Document*
