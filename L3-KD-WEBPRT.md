
## L3-KD-[WEBPRT]: Component-Specific Key Decisions for [WEBPRT]

### Purpose  
This document records all locked-in, component-specific key technical decisions for the PromoPartner Web-Portal (WEBPRT). It pulls through relevant Solution-level (L1) and Integration-level (L2) agreements and adds concrete implementation notes for the WEBPRT delivery team.

### Decision Catalogue  

- **KD-WEBPRT-01 – External Perimeter & WAF**  
  • WEBPRT will remain exposed directly via Azure App Service (Linux container) with a strict **office-IP allow-list**.  
  • No Azure Front Door or WAF is provisioned.  
  • Roaming Admin/KAM users must connect via the corporate VPN.  

- **KD-WEBPRT-02 – Hosting & Rendering Strategy**  
  • WEBPRT runs in **Nuxt 3 SSR/universal mode** inside a **Node 20** Docker image.  
  • The image is deployed to an **Azure App Service P1v2** plan pulled from Azure Container Registry (ACR).  

- **KD-WEBPRT-03 – Authentication Model & Token Storage**  
Admin/KAM users authenticate by entering either email OR phone number. System sends same OTP to both registered channels (SMS and Email) via MSG91. The 60-min access-token (JWT) is stored in Secure, HttpOnly cookie. 

- **KD-WEBPRT-04 – File-Upload Security Controls**  
  • Ships **without anti-malware scanning**; risk accepted.  
  • The portal enforces a hard MIME/extension allow-list:  
    – `text/csv` for master-data imports (≤50 MB)  
    – `image/png|jpeg|gif|webp` for banner assets (≤5 MB)  
  • Client-side validation is repeated server-side; all other types are rejected with HTTP 415.  

- **KD-WEBPRT-05 – Browser-to-Blob Upload Mechanism**  
  • WEBPRT continues to use **client-side SAS** for direct uploads.  
  • Backend issues a **write-only, single-object SAS** valid **≤10 minutes** (no IP scope).  
  • `assets` container CORS is restricted to the portal origin. Unused SAS tokens are logged and left to expire.  

- **KD-WEBPRT-06 – Promotion Rule Authoring UX**  
  • Admins author rules via a **multi-step "Rule Wizard"** that serialises to a **fixed JSON schema** stored in the shared `contracts` repo.  
  • A "Preview Eligibility" button invokes `/v1/promotions/evaluate?mode=test` to validate the draft.  
  • Free-form JSON editing is **not** offered.  
  • No hard limit on rule complexity; backend validates rule structure and logic.  

- **KD-WEBPRT-07 – Build & Deployment Pipeline**  
  • GitHub Actions: `docker build` ➜ push to **ACR** ➜ `az webapp config container set` ➜ **in-place deployment to production**.  
  • Image tags follow the short git SHA; rollback = redeploy previous image tag.  
  • No slot/swap mechanism; deployments occur during designated maintenance windows.  

- **KD-WEBPRT-08 – Client-Side State Discipline**  
  • No use of `localStorage` or `sessionStorage` for any security token or PII.  
  • All UI state lives in Pinia runtime memory; security tokens stored only in HttpOnly cookies as per KD-WEBPRT-03.  

- **KD-WEBPRT-09 – KAM User Access Restrictions**  
  • KAM users have **strictly read-only access** across all WEBPRT screens.  
  • UI enforces role-based button/action hiding; no POST/PUT/PATCH/DELETE endpoints accessible to KAM.  
  • Backend RBAC double-checks and rejects any write attempts with HTTP 403.  

- **KD-WEBPRT-10 – OTP Attempt Limits & Lockout**  
  • **Wrong OTP**: Backend enforces ≤5 failed attempts within 10 minutes, triggering 15-minute account lockout.  
  • **Resend OTP**: Limited to ≤3 resends per 15-minute window.  
  • OTPs stored as **plaintext** in PostgreSQL (no encryption); TTL enforced server-side.  
  • Both **SMS and Email OTP** channels available for all Admin/KAM users via MSG91.  

- **KD-WEBPRT-11 – TLS Configuration**  
  • All public WEBPRT endpoints enforce **TLS 1.2** minimum.  
  • Azure App Service platform handles TLS termination; no custom certificate management.  

- **KD-WEBPRT-12 – Secret Management**  
  • No Azure Key Vault usage; secrets stored as **App Service Application Settings**.  
  • JWT signing keys and MSG91 credentials managed through Azure Portal configuration.  
