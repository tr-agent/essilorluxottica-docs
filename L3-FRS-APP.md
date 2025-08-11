## L3-FRS-APP: Functional Requirements Specification (FRS) for APP  

### 1. Purpose & Scope  
This Level-3 document specifies all functional capabilities that the PromoPartner Mobile Application (component code: **APP**) must deliver in order to fulfil the system-level behaviours defined in L1-FRS and refined by the clarifications agreed with the client. The APP targets iOS (≥ 13) and Android (≥ 8.0) phones and tablets and operates strictly online. It consumes only the public REST endpoints exposed by the Backend (**BCKND**) and never stores business data persistently on-device, other than credentials in the secure keystore / keychain.

### 2. Functional Requirements  

Each requirement is tagged with an internal identifier `APP-FR-xx` and, where applicable, cites the originating high-level provision in L1-FRS (`Implements L1-FRS-§x.y`). The listed HTTP endpoints and verbs are authoritative; any change must first be reflected in L2-LLD-IC.

### 2.1 Platform & Environment  
| ID | Requirement | Trace | Notes |
|----|-------------|-------|-------|
| APP-FR-01 | The APP SHALL run on Android 10+ and iOS 15+ phones and tablets, adapting layouts to screens ≥ 7". | L1-FRS-§6.3 | Minimum OS versions aligned with L3-LLD-APP §5 for security. |
| APP-FR-02 | The APP SHALL operate online only; any network failure surfaces a banner and a **Retry** action. | L1-HLD-§10, KD-APP C#1 | No offline queue or local DB. |


### 2.2 Authentication & Session Management  
| ID | Requirement | HTTP Endpoint(s) | Trace |
|----|-------------|------------------|-------|
| APP-FR-03 | The APP SHALL request a login OTP (both SMS and Email) for Store/Parent users via `POST /v1/auth/otp/request`. | Implements L1-FRS-§2.1.1 | Both channels use MSG91 |
| APP-FR-04 | The APP SHALL verify the OTP via `POST /v1/auth/otp/verify`; on success it MUST persist `jwt` and `refreshToken` in secure keystore / keychain only. JWT tokens are RS256-signed and passed via Authorization header. | L1-FRS-§2.1.1 |
| APP-FR-05 | The APP SHALL silently refresh the JWT 5 minutes before expiry using `POST /v1/auth/refresh`; on failure it MUST purge credentials and navigate to Login. | L1-FRS-§2.1.3 ; KD-APP C#7 |
| APP-FR-06 | The APP SHALL enforce single-device concurrency by including the last issued `refreshToken` only; any 401/403 with `UNAUTHENTICATED` code triggers forced logout. | Clarification #7 |
| APP-FR-07 | The APP SHALL enforce lockout after 5 failed OTP attempts within 10 minutes, displaying "Account locked for 15 minutes" message when `ACCOUNT_LOCKED` error is received. | L1-FRS-§2.1.1 | Backend enforces the lockout logic |


### 2.3 Store Context Switching (Parent-Store Users)  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-08 | The APP SHALL attach header `X-Store-Id: <childStoreId>` on every request when the active context ≠ parent. | KD-APP C#2 |
| APP-FR-09 | The APP SHALL provide a UI selector listing child stores fetched from `GET /v1/stores?parentId=<me>` and persist the last choice in volatile memory. | L1-FRS-§2.2.4 |


### 2.4 Customer Verification OTP  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-10 | When required by promotion rules, the APP SHALL issue `POST /v1/customer-otp/request` supporting both SMS and Email channels via MSG91 and display an OTP input dialog. | L1-FRS-§2.1.2 |
| APP-FR-11 | On successful verify (`POST /v1/customer-otp/verify`) the APP SHALL cache the returned `eligibilityToken` in RAM for the current cart session only. | L1-FRS-§2.1.2 |
| APP-FR-12 | The APP SHALL enforce customer OTP policies: 6 digits, 5-minute TTL, max 3 attempts before lockout, max 3 resends per 15 minutes with 60s cooldown. | L1-FRS-§2.1.2 | TTL and lockout enforced by backend |


### 2.5 Product Catalogue Browsing & Search  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-13 | The APP SHALL list products via paginated `GET /v1/products?page=<n>&pageSize=50` with infinite scroll. | L1-FRS-§2.3.1 ; KD-APP C#3 |
| APP-FR-14 | The APP SHALL support text search (`?search=<term>`). No barcode or QR scanning is required in v1. | Clarification #1 |
| APP-FR-15 | The APP SHALL debounce search queries (≥ 300 ms) and cancel in-flight requests when the term changes. | NFR (TBD) |


### 2.6 Cart & Promotion Evaluation  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-16 | The APP SHALL allow users to add any quantity and number of line items; client-side limits are **not** enforced. | Clarification #2 |
| APP-FR-17 | On user command **Check Promotions** the APP SHALL call `POST /v1/promotions/evaluate` with `{cart, storeId}` and render the ordered eligible list. | L1-FRS-§2.5.4 |
| APP-FR-18 | If the user makes no explicit choice, the APP SHALL apply the first promotion in the returned list in accordance with backend auto-apply behaviour. | L1-FRS-§2.5.4 |


### 2.7 Transaction Submission  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-19 | The APP SHALL POST the cart using `POST /v1/transactions` with `promotionId`, `eligibilityToken` (optional), and a UUID in `Idempotency-Key` header. | L2-LLD-IC #22 ; KD-APP C#4 |
| APP-FR-20 | On `202 Accepted` the APP SHALL retain the `Idempotency-Key` in RAM and retry on failure until a terminal 201/4xx is received. | KD-APP C#4 |
| APP-FR-21 | The APP SHALL display success screen containing `transactionId`. | L1-FRS-§2.6.1 |


### 2.8 Transaction Reconciliation  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-22 | While status = NEW or UNDER_REVIEW, the APP SHALL provide an **Add / Edit Reconciliation** screen capturing `invoiceNo`, `pidNo`, and one attachment. | Clarification #10 ; L1-FRS-§2.6.2 |
| APP-FR-23 | For the attachment the APP SHALL offer camera capture or gallery pick; JPEG, PNG and PDF files MUST be auto-compressed to ≤ 2 MB . | Clarification #3 |
| APP-FR-24 | The APP SHALL obtain a presigned SAS URL via `POST /v1/csv-jobs/{entity}/presign` (entity=`invoice`) and upload the file via HTTPS `PUT`, register the upload via `POST /v1/csv-jobs` to notify backend, then PATCH the transaction using `PATCH /v1/transactions/{id}/reconcile`. The APP SHALL also support downloading attachments via SAS URL for view-back using `GET` method. | L2-LLD-IC #26,#24 |
| APP-FR-25 | Once the backend returns status VERIFIED or later, the reconciliation UI MUST switch to read-only mode. | L1-FRS-§2.6.3 |
| APP-FR-26 | The APP SHALL enforce reconciliation uniqueness validation: display error if (store_id, invoice_no) or (store_id, pid_no) combination already exists when returned by backend. | L1-FRS-§2.6.2 | Uniqueness enforced by backend |


### 2.9 KPI Dashboard  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-27 | The Home tab SHALL display numeric KPI tiles: store count, last 7 days tx, last 30 days tx, average tx/store, promotion uptake, avg ticket size, promotion-wise retailer count. | L1-FRS-§2.7 ; Clarification #4 |
| APP-FR-28 | KPIs SHALL be fetched via `GET /v1/reports/performance?range=last7d` etc. on screen entry and on pull-to-refresh. | L2-LLD-IC #30 |


### 2.10 Role-Based UI Gating  
| ID | Requirement | Trace |
|----|-------------|-------|
| APP-FR-29 | The APP SHALL ship as a single binary; features are enabled or hidden at runtime based on role claims in the JWT (`role` = S, P). KAM and Admin users are not supported in the mobile app. | Clarification #5 |
| APP-FR-30 | Parent-Store-only features: child-store selector, child-store user creation link-out to WEBPRT. | L1-FRS-§2.2.4 |


### 2.11 Version Enforcement  
| ID | Requirement | HTTP | Trace |
|----|-------------|------|-------|
| APP-FR-31 | The APP SHALL send header `X-App-Version: <semanticVersion>` on every API call. | Clarification #9 |
| APP-FR-32 | On HTTP 426 with header `X-Min-Version`, the APP MUST show a blocking dialog and deep-link to the relevant app-store page. | Clarification #9 |


### 2.12 Error Handling & Notifications  
| ID | Requirement | Trace |
|----|-------------|-------|
| APP-FR-33 | The APP SHALL parse the canonical error envelope (`error.code`, `message`, `details`) and map known codes to localized user messages; unknown codes fall back to generic toast. | L2-LLD-IC §6 ; KD-APP L2-IC-2 |
| APP-FR-34 | A banner SHALL display when `notification_send_failures` metric indicates MSG91 outage; login and customer OTP flows become read-only. | KD-APP C#6 |
| APP-FR-35 | The APP SHALL use TLS 1.2 for all HTTPS connections to backend and Azure services. | L1-HLD-§8 | Security requirement |
| APP-FR-36 | The APP SHALL NOT implement any push notification functionality (no FCM/APNS integration). | L1-KD | All notifications via SMS/Email only |


### 2.13 Internationalisation & Accessibility  
| ID | Requirement | Trace |
|----|-------------|-------|
| APP-FR-37 | All static strings MAY be hard-coded English; no runtime localisation scaffold is required in v1. | Clarification #6 |
| APP-FR-38 | The APP SHALL provide basic accessibility: semantic labels on tappable widgets and respect OS font-scale settings. | Clarification #12 |

### 3. Component Interfaces  

#### 3.1 Backend REST API  
The APP interacts only with the following endpoints (all under `/v1`, JSON over HTTPS):

| Purpose | Method | Path | Notes |
|---------|--------|------|-------|
| Login OTP | POST | /auth/otp/request | APP-FR-03 |
| Verify OTP | POST | /auth/otp/verify | APP-FR-04 |
| Token refresh | POST | /auth/refresh | APP-FR-05 |
| Customer OTP | POST | /customer-otp/request /verify | APP-FR-10-11 |
| Product list | GET | /products | APP-FR-13 |
| Promotion evaluate | POST | /promotions/evaluate | APP-FR-17 |
| Transaction create | POST | /transactions | APP-FR-19-20 |
| Transaction reconcile | PATCH | /transactions/{id}/reconcile | APP-FR-24 |
| KPI report | GET | /reports/performance | APP-FR-27-28 |
| Presign upload | POST | /csv-jobs/invoice/presign | APP-FR-24 |
| Register upload | POST | /csv-jobs | APP-FR-24 |
| Store list | GET | /stores | APP-FR-09 |

_All requests include Authorization header and, where applicable, `X-Store-Id`, `X-App-Version`, `Idempotency-Key`._

#### 3.2 Azure Blob Storage (SAS)  
• HTTPS `PUT` upload of invoice image or PDF to SAS URL (APP-FR-24).  
• HTTPS `GET` download of transactional attachments for view-back. (APP-FR-24)

#### 3.3 Device OS Services  
1. Secure Keystore / Keychain for JWT & refresh token (APP-FR-04).  
2. Camera / Photo Gallery for attachment capture (APP-FR-23).  
3. Network reachability callback for online banner (APP-FR-02).

### 4. Data Requirements  

| Data Item | Storage Location | Lifecycle |
|-----------|------------------|-----------|
| `jwt`, `refreshToken` | Secure keystore / keychain | Cleared on logout, expiry, or 401 |
| `Idempotency-Key` | RAM | Until 201/4xx response |
| Last used `storeId` | RAM | Cleared on logout |
| Cart state | RAM | Cleared on transaction success or manual reset |
| KPI cache | RAM | Cleared when app goes to background > 30 min |

_No SKU catalogue or transaction history is cached persistently._ The platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) but provides no Disaster Recovery capability.

### 5. Validation Rules (Client-Side)  

| Rule ID | Rule | Applies To |
|---------|------|-----------|
| VR-01 | OTP code must be exactly 6 numeric characters before calling `/verify`. | APP-FR-04,11 |
| VR-02 | `invoiceNo`, `pidNo` accept `[A-Za-z0-9-_]{1,30}`; client masks illegal chars but DOES NOT block submission (server authoritative). | APP-FR-22 |
| VR-03 | Image attachment must be JPEG , PNG or PDF ≤ 2 MB after compression; otherwise show inline error. | APP-FR-23 |
| VR-04 | `X-App-Version` header must follow `major.minor.patch` semver regex. | APP-FR-31 |
| VR-05 | Reject barcode / QR usage (feature not present). | APP-FR-14 |

### 6. Assumptions & Constraints  

1. Single production environment; no sandbox endpoint selection in UI. Deployment uses in-place updates without slot-swap mechanism.
2. APP relies on backend rate-limiting headers; no client throttling logic is implemented.  
3. No third-party analytics SDK bundled at launch (KD-APP C#8).  
4. The mobile build chain is Flutter 3.2.x with `flutter_bloc` state management (KD-APP C#5).  
5. WCAG compliance is best-effort; no formal audit in scope (Clarification #12).  
6. OTP codes are stored as plaintext without encryption in PostgreSQL (accepted risk).
7. Keys and secrets are stored in encrypted App-Service settings without Azure Key Vault.
8. Login OTP has 10-minute TTL with max 5 attempts and 3 resends per 15 minutes.

---

_End of File_