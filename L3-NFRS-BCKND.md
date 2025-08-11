## L3-NFRS-BCKND: Non-Functional Requirements Specification (NFRS) for BCKND  

### 1. Purpose & Scope  

This document refines the solution-wide quality attributes in L1-NFRS and tailors them to the Backend (BCKND) component of PromoPartner.  
It defines the performance envelope, security posture, reliability targets, and environmental constraints that the .NET 8 modular-monolith must honour when running on a single Azure App-Service (Linux container) instance. All figures reflect the **current, single-region, ≤5 000-user deployment**; no speculative future-scale numbers are introduced.

---

### 2. Performance Requirements  

No contractual SLOs. The following are internal guidelines for a single Azure App-Service Basic B2 instance:

| Aspect | Target |
|--------|---------|
| Daily capacity | ≤100k transactions, ≤50k OTPs |
| Concurrent users | ≤1,000 |
| API response time | <800ms P95 (best effort) |
| CSV processing | 50MB file within 2 hours |
| Resource usage | <70% CPU, <2.8GB RAM |

---

### 3. Scalability  

• Single instance only - scale vertically by upgrading Azure SKU if needed
• No horizontal scaling, read replicas, or sharding
• Cache limited to 20 active promotions
• Background jobs use PostgreSQL advisory locks to prevent conflicts
---

### 4. Security Requirements  

• **Authentication**: 6-digit OTP via SMS & Email (MSG91), stored plaintext in PostgreSQL

- Login OTP: 10-min TTL, max 5 attempts → 15-min lockout
- Customer OTP: 5-min TTL, max 3 attempts → 15-min lockout
• **Tokens**: RS256-signed JWT (60 min) + refresh token (12 hours)
- Web: HTTP-only secure cookies
- Mobile: Platform secure storage + Authorization header  
• **Transport**: TLS 1.2 minimum on all endpoints
• **Secrets**: Stored in encrypted App-Service settings (no Key Vault)
• **Data at Rest**: Azure-managed AES-256 encryption
• **Access Control**: KAM users restricted to read-only (403 on write operations)

---

### 5. Reliability & Availability Requirements  

The business accepts the risk associated with a single-region deployment; therefore the component sets only minimal reliability objectives.

| Attribute | Requirement |
|-----------|-------------|
| Deployment | In-place container replacement on the single Production App Service during a ≤5-minute maintenance window; no deployment slot/swap mechanism is provisioned (aligned with single-tenant, production-only environment). |
| Back-ups | PostgreSQL built-in automated daily backups with 7-day Point-in-Time Recovery (PITR) – same region only, no Disaster Recovery capability |
| RPO / RTO | Not contractually defined; manual restore from same region |
| Failure isolation | In-process workers protected by advisory locks and idempotency keys |
| Graceful degradation | Mobile app switches to read-only when API unreachable |

No formal SLA or monthly uptime objective is imposed; the client explicitly accepts region-wide outage risk.

---

### 6. Maintainability  

• Modular monolith architecture
• GitHub Actions CI/CD with automated tests
• Logging to Azure Application Insights (90-day retention)
• Health endpoints: `/healthz` and `/readyz`
• Basic alerting via Application Insights
---

### 7. API Standards  

• Consistent JSON envelope per L2-LLD-IC §6
• Idempotent operations return 201 with `X-Duplicate: true` header
• All responses include `traceId` for debugging
• KAM users receive 403 on write operations

### 8. Compliance & Legal  

All data-handling rules below have been reviewed and approved by EssilorLuxottica's legal team.  
• Indefinite retention of all business data and PII, per EssilorLuxottica legal sign-off (L1-NFRS §8).  
• Exception: short-lived `eligibility_tokens` are marked with `expired=true` flag every 15 min as mandated by FR-BCKND-1201 but retained indefinitely (business-approved exception to support audit trail).  
• Indian DPDP is the governing regulation; GDPR deemed out of scope.  
• Immutable audit trail written to Blob `audit/` container for every create/update/delete or status change; never purged.  
• Only MIT, Apache 2.0, or equivalent licences permitted in OSS dependencies.
• OTP rows are marked with `consumed_at` timestamp but never deleted, maintaining indefinite retention.

---

### 9. Environmental Requirements  

This section locks the runtime environment so that performance, security and cost assumptions remain valid.  

1. **Azure Region:** Central India only.  
2. **Compute:** Azure App-Service (Linux container) SKU Basic B2, base image `mcr.microsoft.com/dotnet/aspnet:8.0-bookworm` rebuilt quarterly.  
3. **Data:** Azure PostgreSQL Flexible Server (Burstable B2ms).  
4. **Messaging & Storage:** Azure Storage Queue + Blob (Hot & Cool tiers, immutable audit).  
5. **Networking:** Public HTTPS endpoints; optional IP allow-list on Web Portal only; no VNet.  
6. **3rd-Party Services:** MSG91 (SMS & e-mail) – single REST endpoint.

---

### 10. Key Dependencies  

• Azure App Service (Basic B2) - Linux container hosting
• Azure PostgreSQL Flexible Server - Data storage
• Azure Blob Storage - Files and audit logs
• Azure Storage Queue - Async messaging
• MSG91 - SMS & Email delivery
• All dependencies detailed in L1/L2 documents
---

_End of Document_
