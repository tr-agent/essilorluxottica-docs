
## L1-FRS: Functional Requirements Specification (FRS) (High-Level)

### 1. Purpose & Scope  
This document enumerates the high-level functional capabilities, external interfaces, data obligations, and validation rules for "PromoPartner", Essilor-Luxottica's joint trade-promotion management platform. It translates business goals into verifiable system behaviours that guide component-level design and implementation (captured in L2/L3 artefacts).

---

## 2. Functional Requirements  
Each subsection summarises the expected system behaviour and the main success criteria. Introductory context sentences have been added where previously missing to clarify actors, purpose, and scope.

### 2.1 Authentication & Session Management  
*Context – secure, OTP-centric access for ≈ 5 000 named users and customer OTP flows at point-of-sale.*

1. Login OTP  
   • Users request an OTP through the authentication service; both SMS and e-mail OTP options are available for all users and customers.
   • The OTP is a one-time 6-digit numeric code with a short, server-enforced validity period. Backend enforces lockout after 5 failed attempts within 10 minutes with a 15-minute lockout window.
   • A successful verification issues a time-bound session token (JWT) and a longer-lived refresh token, both stored in HTTP-only cookies; lifetimes are defined in lower-level design documents.

2. Customer-verification OTP  
   • Store and Parent-Store Users request a customer-verification OTP via both SMS and Email during promotion redemption; the backend enforces stringent validity and attempt limits.  
   • Verification returns an `eligibility_token` required for transaction creation.

3. Token Refresh & Revocation  
   • The platform provides a token-refresh service that exchanges a valid refresh token for a new JWT.  
   • Deactivating a user immediately revokes all active tokens; subsequent calls must be rejected with an unauthorised error.

### 2.2 User & Role Management  
*Context – two back-office roles (Admin, KAM) and two in-store roles (Store User, Parent Store User).*

1. The system shall support create, read, update, and deactivate operations for users.  
2. Bulk user import supports creation and upsert via CSV; invalid rows are returned to the Administrator in a reject file.  
3. Role-based access control (RBAC) is enforced per request and reflected in issued JWT claims.  
4. Parent Store Users may create Child Stores and Store Users scoped strictly to their own hierarchy.
5. KAM users have read-only access with no permissions for create, update, or delete operations.

### 2.3 Master Data Maintenance  

#### 2.3.1 Product Catalogue (Lens, Frame)  
This capability allows Administrators to keep the unified product catalogue accurate and up to date, either interactively or via scheduled data drops from upstream ERP systems.

1. Administrators can add or edit products manually via the portal or ingest them in bulk through CSV upload using SAS URLs provided by the backend.  
2. Global uniqueness is enforced on column `sku` across both product types.  
3. The bulk-ingest pipeline performs UPSERT; rows violating mandatory fields or containing duplicate SKUs within the file are rejected.

#### 2.3.2 Store Registry  
The Store Registry holds the master list of optical outlets, their hierarchical relationships, and commercial attributes used for eligibility rules and reporting.

1. CRUD operations must honour the mandatory fields defined in the Workflow document.  
2. Parent/Child classification applies; for Parent stores the `parentCustomerCode` equals the store's own unique ID.  

### 2.4 Group Management  
Groups provide reusable, attribute-based filters that simplify promotion targeting and reporting.

1. Store Groups – attribute-based Boolean expressions across the nine store attributes defined in the Workflow.  
2. Product Groups – attribute-based expressions over combined Lens/Frame attributes plus `productType`.  
3. Expressions are stored declaratively; the resolved member list is calculated at promotion activation time.

### 2.5 Promotion Lifecycle  
*Core business capability; rule engine applies up to 20 concurrent promotions.*

1. Status model = DRAFT, SCHEDULED, ACTIVE, PAUSED, ARCHIVED.  
2. Multi-step wizard captures basic information, rule set (IF/THEN), eligibility, and test data.  
3. Activation logic  
   • If `startDate > now` ⇒ SCHEDULED else ACTIVE.  
   • Daily scheduler flips SCHEDULED → ACTIVE and ACTIVE → ARCHIVED.  
4. Eligibility Evaluation  
   • The promotion-evaluation service receives store context and cart contents and returns eligible promotions ordered by estimated customer savings.  
   • Store or Parent-Store Users may manually choose any eligible promotion. Exactly one promotion may be applied per transaction.  
5. Value-Upgrade configuration uses explicit ordered lists maintained in Settings → Hierarchy.  

### 2.6 Transaction Processing  
Transactions record every promotion redemption and drive the reconciliation and settlement process.

1. Creation  
   • The API is idempotent on a client-supplied `request_uuid`.  
   • An `eligibility_token` is mandatory when customer OTP is required.  
   • Initial status = NEW.  
2. Reconciliation  
   • Store or Parent-Store Users supply `invoiceNo`, `pidNo` via the reconciliation interface.  
   • On first entry the status auto-moves NEW → UNDER_REVIEW.  
3. Admin Status Control  
   • UNDER_REVIEW → VERIFIED when the Admin validates the reconciliation fields.  
   • VERIFIED → SETTLEMENT_COMPLETE is a manual Admin action or a bulk CSV update.  
4. The finite-state machine prevents backward transitions; invalid requests must be rejected with a conflict error.

### 2.7 Reporting & Dashboards  
Dashboards provide at-a-glance operational insight, while downloadable reports support deeper analysis and offline reconciliation.

1. Dashboard displays core KPIs:
   - Active promotions count
   - Transactions today
   - Total transactions (last 30 days)  
   - Total discount given (last 30 days)
   - Active stores count
2. Reports are generated on demand, delivered as CSV, and stored indefinitely in object storage.

### 2.8 Notification & Messaging  
The platform dispatches OTPs and system notifications through MSG91 for both SMS and e-mail delivery. Resilience mechanisms such as retry policies and circuit-breakers shall be implemented; specific thresholds are defined in the L2 Operational Design.

### 2.9 Audit & Compliance  
Every state-changing action is recorded to support financial audit, fraud detection, and regulatory compliance.

1. Every create/update/delete action on key entities, status change, and login/logout is logged as an immutable audit event.  
2. Audit events are written to append-only object storage; retention is indefinite.  
3. Customer phone/e-mail data is retained indefinitely in line with the client's approved policy.

---

## 3. System Interfaces  

| Interface | Direction | Purpose | Key Operations |
|-----------|-----------|---------|----------------|
| REST API | Web Portal & Mobile App ⇆ Backend | Core business functions | Authentication, catalogue, promotion, transaction, reporting |
| Object Storage Service | Web Portal → Backend → Object Storage | Bulk file ingest & retrieval | Backend provides SAS URL for upload, then client registers uploaded file|
| Message Queue Service | Backend → Internal Queue → Backend Notification Worker Process | Asynchronous dispatch of OTP & e-mail jobs | Enqueue / Dequeue messages | 
| SMS / E-mail Gateway | Backend Notification Worker Process → MSG91 | Delivery of OTP and system notifications | Send SMS / Send E-mail via MSG91 REST API |

---

## 4. Data Requirements  

### 4.1 Master Entities & Volumes  
| Entity | Max Volume | Critical Attributes |
|--------|------------|---------------------|
| Product (Lens, Frame) | 100 k | `sku*`, attributes, price |
| Store | 5 k | `storeId*`, type, geo |
| Promotion | ≤ 20 active | `promotionId*`, rule JSON, status, dates |
| User | 5 k | phone/e-mail, role, mappedStoreIds |
| Transaction | 100 k / day | `transactionId*`, promotionId, status FSM |
\* denotes primary key.

### 4.2 Storage & Retention  
1. Relational data are held in the primary transactional database; high-volume tables such as `transactions` and `otp` are partitioned by time to maintain performance.  
2. Blobs—CSV, banners, reports, audit logs—are stored in object storage tiers appropriate to access patterns; no automatic deletion policy is applied.  
3. The platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) with no disaster recovery capabilities.

---

## 5. Validation Rules  

| Area | Rule |
|------|------|
| SKU | `sku` must be unique globally (Lens + Frame). |
| Phone | 10-digit numeric; E.164 formatting on output. |
| E-mail | RFC 5322 compliance. |
| OTP | 6 digits; validity period and attempt limits enforced server-side. |
| Invoice / PID | `[A-Za-z0-9-_]{1,30}`; (storeId, invoiceNo) must be unique AND (storeId, pidNo) must be unique |
| Promotion Rule | Saved rule must compile. |
| Date Ranges | `startDate ≤ endDate`. |
| Status Transitions | Must follow defined FSM; invalid transition → conflict error. |
| File Upload | CSV files must conform to the agreed schema and the size limits defined in L2-NFR documentation. |
| OTP Storage | OTP codes are stored as plaintext without encryption in PostgreSQL (accepted risk). | 
---

## 6. Assumptions & Constraints  

1. Single-region deployment; no cross-region disaster-recovery.  
2. Online-only mobile experience; write actions are blocked when offline.  
3. Peak concurrent load capped at ≈ 1 000 mobile sessions and 100 k transactions/day; vertical scaling is the only growth path.  
4. No payment gateway, ERP, or SSO integration is required for phase 1.  
5. No push notifications (FCM/APNS) are implemented; all notifications are via SMS and email only.
---

_End of Document_