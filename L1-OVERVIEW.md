## L1-OVERVIEW: Solution Overview (High-Level)

### 1. Business Context & Overview  
PromoPartner is a single-tenant, cloud-hosted platform that enables EssilorLuxottica to:  
1. Publish and centrally govern up to 20 concurrent trade-promotion campaigns for 5,000 optical stores.  
2. Let up to 5,000 Store Users (retail staff) apply promotions via a mobile app (iOS + Android) with OTP verification.  
3. Record and reconcile up to 100,000 promotion transactions per day (10 – 20 tx/user).  
4. Maintain master data for ~100,000 SKUs (lens + frame) and hierarchical store structures through bulk CSV uploads.  
5. Support an estimated ~1,000 concurrent mobile sessions (≈ 20 % of the total user base); the exact peak-load figure will be verified during L2 capacity planning.  

Business problems addressed  
• Fragmented promotion execution across stores leads to inconsistent customer experience.  
• Manual, spreadsheet-driven reconciliation delays retailer settlement by weeks and increases error rates.  

PromoPartner standardises promotion rules, automates eligibility checks, and supplies auditable transaction data to shorten settlement cycles.  

Implementation note: When a promotion reaches its End Date the system automatically transitions its status from `Active` (or `Scheduled`) to the terminal status `Archived`. The previously described `Complete` status has been removed to avoid ambiguity.

### 2. Core Business Entities  

| Entity | Cardinality | Key Attributes |
|--------|-------------|----------------|
| Promotion | ≤ 20 active | id, type, status (enum: Draft, Active, Scheduled, Paused, Archived), IF/THEN rule set (JSON), start/end |
| Product (Lens, Frame) | ~100,000 | sku (single canonical identifier mapped from legacy “New Code” & “EAN_New” / “SKU Code”), attributes, price |
| Store | ≤ 5,000 | ids (Luxottica, Essilor), parent_id, geo attributes |
| StoreGroup | variable | attribute-based filter expression that resolves dynamically to member store_ids |
| ProductGroup | variable | attribute-based filter expression that resolves dynamically to member sku list |
| Transaction | ≤ 100,000/day | trx_id, store_id, products[], promotion_id, amounts, status |
| User | ~5,000 | phone/email, role, mapped_store_ids |

Note: During CSV ingest the fields “New Code” (lens) and “EAN_New” / “SKU Code” (frame) are transparently aliased to the unified column name `sku`. If the same `sku` already exists in the database the ingest pipeline performs an UPSERT, i.e., it updates the existing row with the new attribute values; only rows that contain conflicting mandatory-field values are rejected and reported back to the Admin user.

### 3. End-to-End Flow  

#### 3.1 Data Flow  
1. Admin uploads master-data CSV → Web Portal → File-ingest service → Azure Blob Storage.  
2. Ingest job validates schema, mandatory columns, and `sku` uniqueness within the file, then executes an UPSERT into PostgreSQL Flexible Server (time-range partitioned).
3. API layer exposes REST/JSON endpoints; all queries are routed to the primary.

#### 3.2 Product Flow  
Mobile App selects SKUs → `/promotions/evaluate` API → Rule Engine computes eligibility & discount → Transaction persisted → Confirmation returned.

#### 3.3 Money / Settlement Flow  
1. Discount applied at store (PromoPartner Store User app).  
2. Transaction status lifecycle: New → Under Review → Verified → Settlement Complete.  
3. CSV bulk status update supported for back-office teams.

#### 3.4 Control & Messaging Flow  
• OTP channels: All users (Customers, Store Users, and Admins) receive both SMS and Email OTP via MSG91.  
• Token-based authentication managed by an internal auth service.

### 4. Components  
A component is an independent GitHub repository; all services within a component live in the same repo.  

1. Web Portal (WEBPRT) – Nuxt-based SPA.  
2. Mobile App (APP) – Flutter, compiled for iOS & Android.  
3. Backend (BCKND) – .NET-based modular monolith exposing REST endpoints.  

All supporting services reside within these components only.

### 5. High-Level Architecture  
• Cloud Provider: Microsoft Azure  
• Compute Pattern: two independent Azure App Service (Linux container) plans – one for the Backend, one for the Web Portal. No slot/swap mechanism is used; all deployments are in-place to the production environment.
• Data Tier: Azure Database for PostgreSQL Flexible Server with built-in automated daily backups (7-day Point-in-Time Recovery); time-range partitioning approach will be detailed in the L2 design.  
• Storage: Azure Blob (CSV, images, exports). Hot tier for frequently accessed assets, Cool tier for aged objects, immutable audit container, no deletion. CSV uploads use SAS URLs issued by the Backend for direct client upload.

### 6. Non-Standard Elements 
1. Store-parent hierarchy management and on-device store-switch for Parent Store Users.  
2. Transaction-reconciliation CSV workflow tightly coupled to promotion-lifecycle statuses.  
3. Production-only environment.  
4. Platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) but provides no Disaster Recovery capability; risk acknowledged and accepted by EssilorLuxottica.

### 7. Deployment & Integration Strategy 
• Environment: Production (full tier).  
• CI/CD pipeline automates container build, automated unit/integration tests, and deployment to the single Production environment after sign-off (tooling to be confirmed in L2).  
• Integration Touchpoints:  
  – SMS OTP (MSG91 via internal Notification Worker process in BCKND)  
  – Email OTP (MSG91 via internal Notification Worker process in BCKND)

### 8. Technology Decision Rationale
• Container-based App Service was selected over serverless to maintain predictable warm-start latency for the .NET promotion engine.  
• Framework and tooling selections (e.g., SPA framework, CI/CD platform) are indicative and will be confirmed during the L2 Low-Level Design phase in alignment with EssilorLuxottica’s skill-sets and licensing.