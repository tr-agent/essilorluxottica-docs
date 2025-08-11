## L3-NFRS-WEBPRT: Non-Functional Requirements Specification (NFRS) for WEBPRT

### 1. Purpose & Scope  
This document defines the quality attributes that the PromoPartner Web-Portal component (code = WEBPRT) must satisfy in Phase-1.  It refines the solution-level L1-NFRS for portal-specific concerns and records all client clarifications that loosen or tighten those requirements.  The audience is the front-end development team, DevOps engineers and QA responsible for implementing, operating and testing WEBPRT.

### 2. References  
* L1-NFRS – solution-wide non-functional baseline  
* L1-HLD §2, §9 – hosting & deployment topology  
* L3-KD-WEBPRT – component key decisions  
* Client Clarifications #1 – #8 (see consolidated list)  
* L3-FRS-WEBPRT – functional counterpart to this specification  

### 3. General Assumptions  
1. Admin/KAM combined head-count ≤ 100 users; peak concurrency ≤ 30.  
2. WEBPRT is delivered as a Nuxt 3 Universal (SSR) app running in a single Node 20 container on an Azure App Service P1v2 plan.  
3. All business data is served through BCKND REST APIs; WEBPRT owns no persistent data store.  
4. The organisation has accepted a single-region, best-effort availability model; no formal SLAs or SLOs are mandated for WEBPRT (Clarifications #1 & #5).  

---

### 4. Non-Functional Requirements  

#### 4.1 Performance & Scalability  
Context: The user base and traffic volume are modest; therefore overly rigid web-vital targets were deemed unnecessary (Clarification #1). The following internal budgets guide development and capacity planning but are **non-contractual**.  

1. Basic performance targets (informational only, not contractual):
   • Page loads complete within 3 seconds for Admin/KAM users
   • Support up to 30 concurrent sessions on P1v2 plan
   • CSV uploads up to 50 MB supported
2. No horizontal scaling; vertical scale to P2v2 if needed 

#### 4.2 Reliability & Availability  
1. Planned downtime: WEBPRT may be offline for **≤ 5 minutes** during in-place deployment during the shared 18:00-20:00 IST maintenance window (Clarification #2).  
2. Unplanned downtime: no numeric monthly SLO; manual monitoring only (Clarification #5).  
3. Degradation strategy  
• If BCKND is unavailable, display "Service unavailable – try later" and disable all interactions

4. Recovery  
   • Container restarts automatically via App Service platform; no multi-zone fail-over.  
   • State is stateless; a cold reload restores full functionality once BCKND is available.  

#### 4.3 Security  
1. Office IP allow-list enforced
2. TLS 1.2 for all endpoints  
3. JWT/refresh tokens in HTTP-only cookies
4. No PII in browser storage
5. Basic security headers (HSTS, X-Frame-Options)

#### 4.4 Usability & Accessibility  
1. Accessibility – basic voluntary checks for colour-contrast and font-scaling aligned with L1-NFRS §7; no formal WCAG target.  
2. Consistency – UI patterns mirror those of the mobile app and adhere to EssilorLuxottica brand guidelines.  
3. Browser support – latest "evergreen" versions of Chrome, Edge and Safari.  
4. Graceful error feedback – all error states listed in FR-WEBPRT-19/-21 and the Error-Handling Matrix must surface actionable messages; stack traces must never leak to the client.  

#### 4.5 Maintainability & Supportability  
1. Codebase – single GitHub repository `promo-webprt`, main branch protected by PR review + CI unit tests.  
2. Build pipeline – GitHub Actions → Docker build → push to ACR → `az webapp config container set` for in-place deployment (KD-WEBPRT-07).  
3. Observability  
• Logs shipped to Azure Application Insights
• No WEBPRT-specific alerts needed  
4. Configuration – environment variables for: API base URL, IP allow-list, no OTP template IDs (handled by BCKND), blob container name.  

#### 4.6 Compliance & Legal  
1. Data-protection – follows the overall DPDP stance: indefinite retention justified under financial-audit exemption (L1-NFRS §8).  
2. Licensing – third-party JS libraries must be MIT/Apache 2.0 compatible; copyleft licences forbidden; CI pipeline enforces licence scanning per L1-RAD R9.  
3. Audit – WEBPRT actions are captured via BCKND audit trail; the portal itself stores no audit artefacts locally.  

#### 4.7 Environmental Constraints  
| Dimension | Requirement |
|-----------|-------------|
| Cloud Region | Azure Central India |
| Compute | 1 × App Service (Linux) Plan P1v2, Node 20 container |
| Image Source | Azure Container Registry `promo/acr/webprt:<sha>` |
| Storage | Azure Blob Hot tier container `assets` (CSV & images) |
| Network | Public endpoint, office IP allow-list, no VNet, no WAF |
| Browser | Desktop only; mobile browsers not officially supported |
| Backup/DR | Platform relies solely on Azure PostgreSQL's built-in 7-day PITR; no cross-region DR |

---

### 5. Dependencies and Requirements  
The table enumerates every upstream dependency that can influence WEBPRT's NFR compliance.  

| Dependency | Level | Impacted NFR Area | Notes / Version |
|------------|-------|-------------------|-----------------|
| BCKND REST API (`/v1/*`) | L1 / L2 | Performance, Reliability | P95 ≤ 800 ms target inherited from L1. |
| Azure App Service (Linux) | L1 | Performance, Availability | Must support in-place deployment within 5 min window. |
| Azure Blob Storage (Hot tier) | L1 | Performance, Security | Direct browser uploads via SAS; TLS enforced. |
| MSG91 SMS/Email channel | L1 | Security | OTP delivery via both SMS and email; availability outside WEBPRT control. |
| GitHub Actions + ACR | L1 | Maintainability | Pipeline artifacts, image provenance, rollback. |
| Office IP allow-list | KD-WEBPRT-01 | Security | Changes require Ops co-ordination. |
| JWT / Auth endpoints | L2-LLD-IC | Security, Reliability | Must honour RS256-signed JWT with 60-min access, 12-h refresh token stored in HTTP-only cookies. |

---

### 6. Verification & Acceptance  
QA will confirm NFR compliance using the checks below:  
1. Verify basic functionality works for 30 concurrent users
2. Confirm portal recovers after deployment
3. Test CSV upload works for 50 MB files
4. Verify error messages appear when backend unavailable
---

_End of Document_