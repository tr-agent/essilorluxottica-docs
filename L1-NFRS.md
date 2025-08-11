## L1-NFRS: Non-Functional Requirements Specification (NFRS) (High-Level)

### 1. Purpose & Scope  
This document captures the solution-wide quality attributes—performance, security, reliability, usability, compliance, and environmental constraints—that PromoPartner must satisfy. It guides cross-component design decisions and provides a benchmark for acceptance testing. Component-specific refinements are deferred to the respective L3-NFRS artefacts.

---

### 2. Performance Requirements  
The platform must remain responsive for the forecast load of ≤ 5 000 users, ≤ 1 000 concurrent mobile sessions, and ≤ 100 000 transactions per day, while operating on a single instance of the initial Azure App-Service plan (i.e., without horizontal scale-out).

* Interactive APIs – best-effort performance; no contractually binding P95 or P99 targets are committed.   
* Bulk CSV ingest – engineered to complete a single 100 k-row (~50 MB) file in ≤ 2 h from job registration during off-peak hours.  
* Promotion evaluation – rule-engine execution must not be the dominant contributor to checkout latency; profiling results will be supplied in the L2 performance notes.  
* Concurrency – the backend must sustain at least 1 000 simultaneous JWT-authenticated requests on a single App-Service instance. Vertical scale-up to a larger SKU remains the sanctioned growth path.  
* Batch jobs – overnight schedulers (promotion-status rollovers, data exports) must finish before 06:00 IST to avoid interfering with peak store traffic.

---

### 3. Scalability  
While the current load assumptions are modest, the architecture must permit predictable vertical scaling.

* Compute – Azure App Service plan size-up is the primary scaling mechanism; no sharding, no read replicas, no micro-service decomposition in phase 1.  
* Data – PostgreSQL Flexible Server CPU/RAM tiers can be upgraded in-place; time-range partitioning keeps index growth bounded.  
* Storage – Azure Blob Cool tier for all assets, with no lifecycle or deletion policy.  
* Rule-engine caching – compiled rule cache warms on startup and is memory-resident; size limit: 20 active promotions × internal structures < 128 MB.

---

### 4. Security Requirements  
Security relies on OTP-based authentication, JWT authorisation and encryption-everywhere.

* Authentication  
  * 6-digit login OTP via both SMS and Email for all users (Admin, KAM, Store Users, and Customers).  
  * 6-digit customer-verification OTP via SMS.  
  * OTP TTL and attempt limits are environment-configurable; default values (login OTP: 10 min / 5 attempts, customer OTP: 5 min / 3 attempts) will be documented in L2-Operations.  
  * Backend enforces lockout after 5 failed attempts within 10 minutes with a 15-minute lockout window.
* Authorisation – RS256-signed JWT (default 60 min, configurable) carrying role & store claims; refresh token default ≤ 12 h, configurable via environment variable.  
  * Web Portal: JWT stored in HTTP-only cookies; Mobile App: JWT stored in secure local storage and passed via Authorization headers.
  * KAM users have read-only access with no permissions for PUT, PATCH, or DELETE endpoints.
* Transport security – TLS 1.2 on all public endpoints.  
* Data-at-rest – AES-256 server-side encryption for PostgreSQL, Blob and Queue; immutable audit container.  
  * OTP codes stored as plaintext without encryption in PostgreSQL, protected only by database-level encryption.
* Secrets – stored in encrypted App-Service settings; rotated manually during maintenance; detailed procedure lives in L2-Operations.  


---

### 5. Reliability & Availability  
Given the single-region footprint, reliability is constrained by Azure regional availability.

* Target uptime – no formal SLA; business accepts region-wide outage risk.  
* Deployment – releases are executed via in-place deployment to the single Production App Service instance during a 5-minute maintenance window during off-peak hours agreed with business users.  
* Backup – the platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day point-in-time restore). No additional cross-region replication or custom backup mechanism is provided.  
* RPO/RTO – not contractually defined; recovery relies on manual restore from same-region backups.  
* Graceful degradation – mobile app blocks write actions when offline and surfaces a clear "Connectivity lost" banner.

---

### 6. Maintainability & Supportability  
The solution must be easy to evolve by a small engineering team.

* Codebase – three independent GitHub repositories (WEBPRT, APP, BCKND) with CI/CD pipelines in GitHub Actions.  
* Modularity – .NET modular monolith; internal interfaces kept package-private to enable refactor without breaking external contracts.  
* Observability – Serilog logs and application metrics shipped to Azure Application Insights; alert rules defined in L2-Operations document.  
* Configuration – key limits, OTP TTLs, and feature flags exposed via environment variables; no run-time "edit in DB" toggles.  
* Notification delivery – internal Backend worker process consumes from Azure Storage Queue and calls MSG91 for both SMS and Email delivery.

---

### 7. Usability Requirements  
User experience must accommodate mixed technical literacy in optical stores while not burdening Admin users.

* Consistency – common navigation paradigms across Web Portal and Mobile App.  
* Responsiveness – screens should feel responsive to users on mid-range devices with 4G connectivity.  
* Accessibility – no formal WCAG target; basic colour-contrast and font-scaling checks will be performed voluntarily.  
* Supported platforms – "evergreen" desktop browsers (latest Chrome, Edge, Safari), Android 11+ and iOS 16+.  
* Error messaging – human-readable, action-oriented copy; never expose stack traces or SQL errors.

---

### 8. Compliance & Legal  
EssilorLuxottica provided written confirmation that its data-processing policy permits indefinite retention of customer PII.

* Data retention – no purge job; audit logs and transactional data stored indefinitely.  
* Audit trail – every create/update/delete or status transition is logged immutably; entries include actor, timestamp, and payload hash.  
* Licensing – all third-party libraries must be MIT, Apache 2.0 or equivalent; copyleft licences are prohibited.  

---

### 9. Environmental Constraints  
The solution is confined to a single Azure subscription and region.

* Region – Central India (Azure).  
* Compute – Linux containers on Azure App Service; no Windows hosts.  
* Network – public endpoints only; Web Portal optionally IP-allow-listed.  
* Storage – Azure Blob Storage; Hot tier for frequently accessed assets, Cool tier for aged objects, immutable audit container.  
* Messaging – Azure Storage Queue; no external message broker.  
* Third-party services – MSG91 for SMS & e-mail; Apple/Google stores for mobile distribution.  
* CSV uploads – Backend issues SAS URLs for direct client upload to Blob Storage; client registers the file/job after upload.
* Secret management – encrypted App-Service settings without Azure Key Vault.
* Push notifications – not implemented; no FCM/APNS integration.

---

### 10. Extensibility & Future-Proofing  
The architecture should tolerate moderate growth without radical redesign.

* Promotion engine – rule JSON schema versioned; backward-compatible changes preferred.  
* Database – partitioning strategy allows multi-year transaction retention before index rotation is required.  
* Scaling path – vertical scale-up first; horizontal split (read replicas or service decomposition) considered only if sustained load exceeds documented thresholds.  
* Feature toggles – new promotion types can be added via JSON without code changes as long as they fit the existing operator set.

---

_End of Document_