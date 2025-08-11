## L3-KD-BCKND: Component-Specific Key Decisions for BCKND

### 1. Runtime & Deployment  
• The Backend runs as a single Linux-container Web App in **one Azure region only**. Cross-region compute fail-over is out of scope; data durability is retained through Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) which provide **no Disaster Recovery capability**. No passive standby App Service or cross-region traffic manager is provisioned, and **no slot-swap mechanism** is used.  

### 2. Data Security – OTP Handling  
• **OTP codes are stored and queued in plaintext**; protection is provided exclusively by TLS 1.2 in transit and Azure AES-256 encryption at rest. Logging middleware **redacts the `code` field** to prevent leakage to Application Insights. Backend does **not** delete OTP rows after verification;  **Both SMS and Email OTP channels** are supported for all users (Admin, KAM, Store Users, Parent Store Users) and customers via MSG91 integration.

### 3. Data-Retention Policy  
• All PII and business data are retained **for the lifetime of the system with no purge job**. Backend API therefore exposes **no delete or archive endpoints**. Cost management is handled operationally (e.g., Blob tier moves) and is outside Backend code. This retention mandate explicitly includes OTP tables, product, store and promotion data.  

### 4. Background-Worker Strategy  
• Notification Worker, CSV-Ingest and Scheduler workers execute **in-process** inside the same container as the public API. The Notification Worker is an internal BCKND process that consumes messages from Azure Storage Queue and calls MSG91 for both SMS and Email delivery. 

### 5. Secret Management  
• **Encrypted App-Service settings are the single source of truth** for all secrets (JWT RSA key, MSG91 SMS/Email API keys). **No Azure Key Vault** is introduced. Secrets are rotated manually; `dotnet user-secrets` is used only for local dev. 

### 6. Promotion Rule-Engine Safety  
• Rule records persist both JSON and **pre-compiled SQL snippets**. A CLI tool (`tools/rule-compiler`) signs SQL with an HMAC; the API executes only **read-only** snippets where `signature` matches. Faulty or unsigned snippets are rejected and the rule is marked `inactive`.  

### 9. Backup Depth  
• The 7-day PostgreSQL PITR window is accepted as sufficient. No additional logical dumps are scheduled by Backend; Operations own any extended backup strategy. The platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups with **no cross-region disaster recovery capability**.

### 10. Public API Contract Alignment  
• All Backend endpoints are fixed under base path **`/v1`**, JSON/HTTPS, and protected by RS256 JWTs. Any breaking change requires a `/v2` path and explicit contract version bump enforced by CI OpenAPI diff tests. All public endpoints enforce **TLS 1.2**. JWTs are stored in **HTTP-only cookies** for Web Portal access.

### 11. Authentication & Authorization  
• **OTP Lockout Policy**: Backend enforces lockout after **5 failed OTP attempts within 10 minutes**, triggering a **15-minute lockout window**. OTP codes are stored as **plaintext without encryption** in PostgreSQL (accepted risk).  
• **JWT Implementation**: All JWTs use **RS256 encryption algorithm**. Access tokens have 60-minute TTL, refresh tokens 12-hour TTL. Web Portal receives tokens in HTTP-only cookies; Mobile App receives in response body for secure local storage.  

### 12. Role-Based Access Control  
• **KAM users have read-only access** across all endpoints. Backend enforces this by rejecting any PUT, PATCH, or DELETE requests from KAM role with HTTP 403. Admin users have full CRUD access; Store Users have transaction-creation and limited user-management permissions.  

### 13. File Upload Handling  
• **CSV Upload Flow**: Backend issues write-only SAS URLs (10-minute TTL) for direct client uploads to Azure Blob Storage. After successful upload, clients must call `/v1/csv-jobs/<entity>/register` to notify Backend and trigger processing. The Backend CSV-Ingest worker then validates and imports the registered file.  

### 14. External Integrations  
• **No Push Notifications**: The system does **not** implement FCM/APNS push notifications. All notifications are delivered via MSG91 (SMS and Email only).  
• **MSG91 Integration**: Both Email and SMS OTPs are delivered through MSG91's unified API. The internal Notification Worker handles both channels through the same MSG91 REST endpoints.

---  
End of document  
---