## L1-RAD: Risk Assessment Document (RAD)  Document

### 1. Purpose & Scope
This Risk Assessment Document enumerates solution-level risks that could materially affect the successful delivery and operation of PromoPartner.  It considers only the current, explicitly-approved architecture (single Azure region, vertical-scaling path, in-place production deployment, MSG91 as sole messaging gateway).  Component-specific issues will be handled in L3 risk logs.

### 2. Risk-Rating Methodology
Risks are scored on two orthogonal three-point scales:

| Scale | Likelihood (L) | Impact (I) |
|-------|---------------|------------|
| Low   | Unlikely (<20 %) within 12 m | Limited service degradation or <4 h recoverable data loss |
| Medium| Credible (20–60 %)          | Noticeable customer/business impact; manual recovery ≤ 24 h |
| High  | Probable (>60 %) or inevitable | Prolonged outage, legal breach, or irreversible data loss |

Risk Exposure (R) = L × I.

### 3. Overall Risk Heat-Map (Snapshot)
<add>The heat-map offers a one-glance visual of where each identified risk sits on the Likelihood/Impact matrix so that stakeholders can focus mitigation effort on the most critical quadrants.</add>

| Impact → / Likelihood ↓ | Low | Medium | High |
|-------------------------|-----|--------|------|
| Low                     | **R10** |         |      |
| Medium                  | **R9** | **R2**, **R4**, **R6**, **R7** | **R5** |
| High                    |       | **R3** | **R1**, **R8** |

Legend: R# references detailed register entries below.

### 4. Detailed Risk Register
<add>The register provides a drill-down view of each risk, its qualitative score, nominated owner, and the agreed mitigation or contingency plan.</add>

| ID | Category | Description | L | I | R | Owner | Mitigation / Contingency |
|----|----------|-------------|---|---|---|-------|--------------------------|
| R1 | Availability | Single-region deployment with no DR; region-wide Azure outage halts service & blocks OTP delivery. | H | H | **High** | Ops Lead | a) Document manual rebuild run-book (≤72 h target). b) Quarterly restore drill. |
| R2 | Data Integrity | Direct UPSERT of bulk CSVs can overwrite master data if the file is mis-mapped; no preview/rollback. | M | M | **Medium** | Data Lead | a) Enforce mandatory “dry-run” flag in ingest API that counts affected rows before commit. b) Store every accepted file + ingest log for post-incident diff & restore. |
| R3 | Security / Compliance | Indefinite PII retention combined with plaintext OTP storage increases breach impact; regulatory stance may change. | H | M | **High** | Security Lead | a) Obtain and archive legal memo; schedule annual legal review. b) <edit>Hash (SHA-256) or blank out OTP codes after 24 h while retaining metadata for audit, instead of hard-deleting rows.</edit> |
| R4 | Performance | Promotion rule engine mis-configuration could cause >100 ms CPU spikes and raise API latency above UX guideline. | M | M | **Medium** | Dev Lead | a) Reject rules whose compiled tree >10 k chars (already in FRS). b) Add 150 ms execution timeout with circuit-breaker to fall back to “no promotion available”. |
| R5 | 3rd-Party Dependency | MSG91 becomes a single point of failure for all OTP and e-mail traffic; no fallback gateway. | M | H | **High** | Ops Lead | a) Implement sliding-window retry with exponential back-off. b) Queue unsent messages for 24 h. <delete>c) Manual playbook to switch to voice-call OTP if outage > 2 h.</delete> |
| R6 | Release Process | <edit>Production-only in-place deployment; a failed release could take the entire platform offline if rollback is delayed.</edit> | M | M | **Medium** | DevOps Lead | a) Embed smoke-test script (<30 s) in pipeline; auto-rollback on failure. b) Maintain last-known-good container tag for instant re-deploy. |
| R7 | Scalability | Vertical-only scaling path may saturate under unexpected growth (>1 000 concurrent sessions or 200 k tx/day). | M | M | **Medium** | Architecture Lead | a) Monitor CPU/RAM with autoscale alerts at 70 % threshold. b) Keep documented roadmap for read-replica + service split if peak load doubles. |
| R8 | Algorithmic Complexity | **Value-Upgrade** logic depends on mutable hierarchy edited live; conflicting edits can produce unintended discounts. | H | H | **High** | Dev Lead | a) Snapshot hierarchy JSON at promotion activation; changes require “clone & re-activate”. b) Raise admin alert when hierarchy referenced by ACTIVE promotion is modified. |
| R9 | Legal / Licensing | Third-party libraries with restrictive licences could slip into the codebase and violate corporate policy. | M | L | **Medium** | Dev Lead | a) Enable licence-scanner in CI pipeline; block PR merge on copyleft findings. |
| R10| Operational | Absence of MSG91 throughput guarantees could throttle OTP delivery during campaign peaks (~500 k OTP/day). | L | L | **Low** | Ops Lead | a) Negotiate written 100 TPS quota; monitor SMS queue depth; alert at 5 min backlog. |

### 5. High-Complexity Functional Areas
The following five atomic features contain the densest algorithmic logic and therefore carry elevated implementation risk:

1. Promotion Rule Compilation & Evaluation  
   • Transforms nested IF/THEN JSON into executable SQL-like predicates, caches them in-memory, and must return results < 50 ms.  
2. Dynamic Store & Product Group Resolution  
   • Attribute-based Boolean expressions resolved on-the-fly at promotion activation; impacts both eligibility and reporting views.  
3. Value-Upgrade Path Determination  
   • Requires ordered lattice traversal of lens/frame attributes to find the “next higher” SKU; multiple upgrade tiers per promotion.  
4. CSV Ingest with Idempotent UPSERT  
   • Streamed parsing, schema validation, and conflict-handling for up to 100 k rows without staging tables.  
5. Scheduled Status Rollover Job  
   • Daily cron that atomically flips hundreds of promotions (SCHEDULED → ACTIVE, ACTIVE → ARCHIVED) while preventing race conditions with manual edits.

### 6. Complex 3rd-Party Integrations
<add>This section lists external services whose behaviour or contractual constraints can materially affect PromoPartner’s reliability, security, or time-to-market.</add>

1. **MSG91 SMS & E-Mail Gateway** – OTP throughput, template whitelisting, DLT registration, TLS trust-store management.  
2. **Apple App Store / Google Play** – Release signing, phased roll-out, and mandatory privacy manifest updates.  
3. **Azure PostgreSQL Flexible Server** – Time-range partitioning, query-store tuning, and burstable SKU autoscaling nuances.

### 7. <edit>Accepted Residual Risks After Mitigation</edit>
The client has explicitly accepted the following residual exposure after the mitigations detailed in §4:

* Single-region service outage (R1) once the manual rebuild playbook is in place.  
* Plaintext OTP storage for ≤ 24 h until hashing job runs (part of R3).  
* No secondary SMS gateway beyond retries/queueing (R5).  
* Production-only in-place deployment pipeline (R6) despite automated smoke-test & rollback safeguards.

### 8. Next Review
The Solution Architect will revisit this RAD at the end of L2 design or upon any material change to scope, infrastructure, or legal context—whichever occurs first.

---
_End of Document_