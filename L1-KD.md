## L1-KD: Key Technical Decisions  

### Disaster-Recovery & Backup  
- The platform runs in a single Azure region with **no secondary (DR) region**. Azure Database for PostgreSQL Flexible Server’s mandatory daily automated backups (7-day retention) remain enabled; EssilorLuxottica accepts that these backups support only same-region point-in-time recovery and do not provide cross-region disaster recovery.  
 

### OTP Storage & Verification  
- PostgreSQL stores the **plaintext OTP code** together with metadata (channel, recipient, expiry_ts). Both SMS and Email OTPs are supported for all users and customers, delivered via MSG91.
- The same plaintext value is also carried in the encrypted Azure Storage-Queue message that is consumed by an internal Backend worker process that calls MSG91 for both SMS and Email OTP delivery.

### Production-Only Environment & Release Pipeline  
- CI/CD deploys the container image **in-place to the single Production App Service instance** during a designated 5-minute maintenance window. Automated health-checks run post-restart. No pre-production environment or slot-swap mechanism is provisioned.  

### Blob Storage Retention Strategy  
- **Audit logs** are written to an immutable Blob container with indefinite retention.  
- **CSVs and marketing banners** are stored in a separate, non-immutable container; no lifecycle or deletion policy is applied.  

### Global SKU Uniqueness  
- Catalogue tables enforce a **single-column primary key on `sku`**, ensuring global uniqueness across lens and frame products. Ingest and API paths perform UPSERT operations strictly on this key; if an incoming CSV row contains a `sku` that already exists, the row updates the existing record regardless of product type.  

### Mobile App Connectivity Model  
- The mobile APP remains **strictly online-only**; no local queuing or offline transaction mode is implemented.  
- When connectivity is lost the UI shows a blocking banner and disables write actions until the network is restored.  

### Scaling Path for Backend & Database  
- The solution retains a **single .NET modular monolith** and one Azure PostgreSQL Flexible Server; no read replicas are planned.  
- Vertical scaling is the sole growth mechanism; service or data-tier decomposition will be re-evaluated only if sustained load exceeds the documented thresholds.  

###   

###   

### Promotion Rules Engine (Summary)  
- Promotion rules are defined declaratively in JSON. Detailed compilation and caching behaviour is documented in the dedicated L3-KD.