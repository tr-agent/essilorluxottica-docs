## L3-FRS-WEBPRT: Functional Requirements Specification (FRS) for WEBPRT  

### Document Status  

Version 1.2 | 2025-08-02 | Author: Solution Architecture Team  

---

### 0. References & Traceability  

Unless stated otherwise, requirement numbers in parentheses map to the original solution-level document.  
• L1-FRS R-2xx → functional capability reference (numeric IDs to be confirmed in the upcoming L1-FRS re-numbering exercise). Example: "Implements L1-FRS-R-213" refers to Authentication requirement 3 under section 2.1.  
• L2-LLD-IC §7 → authoritative REST catalogue  
• KD-WEBPRT-## → component-specific key decisions  
• L1-WF → workflow specifications for cross-reference  

---

## 1. Purpose & Scope  

This Level-3 FRS captures every behaviour the PromoPartner Web Portal (code: WEBPRT) must exhibit in Phase-1. It is the direct input for the WEBPRT Low-Level Design (L3-LLD-WEBPRT) and Implementation Backlog (L3-IB-WEBPRT). Only Admin and KAM users access WEBPRT (Clarification #1). All functional coverage for Store / Parent-Store users resides exclusively in the mobile app (APP) and is therefore out of scope here.

---

## 2. Actors & Roles  

| Role | Portal Access | High-Level Capabilities |
|------|---------------|-------------------------|
| Admin | Yes | Full CRUD on all entities, promotion authoring, configuration & reporting |
| KAM   | Yes | Read-only access on all entities; may download (but not generate or delete) reports; **no** create / update / delete permissions across any entity |
| Parent-Store User | No | — |
| Store User | No | — |

RBAC inside WEBPRT must mirror the API permissions defined in L2-LLD-IC §8, enforcing KAM read-only access by hiding all UI controls for write actions

---

## 3. Functional Requirements  

Each requirement is tagged "FR-WEBPRT-xx" for downstream backlog linkage.  

### 3.1 Authentication & Session Management  

FR-WEBPRT-01 Login OTP (Implements L1-FRS-R-211)  
• UI collects e-mail OR phone number → POST `/v1/auth/otp/request` → success toast "Code sent".  
• • System sends same OTP to both registered SMS and Email channels via MSG91 for all users (Admin/KAM) when both channels are registered  
• OTP entry form → POST `/v1/auth/otp/verify` – on HTTP 200 the portal must:  
  a. The backend returns both `accessToken` and `refreshToken` in `Set-Cookie` headers (`Secure; HttpOnly; SameSite=Strict`) per KD-WEBPRT-03. The SPA does **not** attempt to set or modify these cookies.  
  b. Store user profile data (userId, role) in Pinia in-memory store for UI rendering.
• OTP validation: 6 digits, TTL 10 minutes, max 5 attempts within 10 minutes triggering 15-minute lockout.  

FR-WEBPRT-02 Silent Token Refresh (Implements L1-FRS-R-213)  
• A background timer calls `POST /v1/auth/refresh` 5 minutes before JWT expiry.  
• On `401` the portal forces logout and redirects to `/login`.

FR-WEBPRT-03 Single-Session Logout  
• "Logout" menu clears Pinia state, deletes any non-HttpOnly cookies (e.g., UI preferences) and relies on natural JWT/refresh-token expiry. Front-end shows "Logged out successfully".  

### 3.2 Global Navigation & Role-Aware UI  

FR-WEBPRT-04 Menu Construction  
• At bootstrap decode the role claim from the JWT (`role` claim) – no extra API call required.  
• Render menu items per mapping table below. Hidden items MUST NOT be reachable via URL typing.

| Menu Item | Admin | KAM |
|-----------|-------|-----|
| Dashboard | ✓ | ✓ |
| Users     | ✓ | Read-only |
| Manage Lenses | ✓ | Read-only |
| Manage Frames | ✓ | Read-only |
| Stores    | ✓ | Read-only |
| Groups    | ✓ | Read-only |
| Promotions| ✓ | Read-only |
| Transactions | ✓ | Read-only |
| Reports   | ✓ | ✓ |
| System Settings | ✓ | — |

### 3.3 Dashboard & KPIs  

FR-WEBPRT-05 Load Snapshot (Implements L1-FRS-R-271)  
• On `/dashboard` page-load issue `GET /v1/dashboard` (JSON payload; endpoint will be added to L2-LLD-IC §7 and OpenAPI v1.1).  
• Response contains core KPIs: `activePromotions`, `txToday`, `txLast30d`, `totalDiscountLast30d`, `activeStores`.
• No auto-polling; user must refresh page to reload.  
• Failure handling: on HTTP ≥ 400 show "Dashboard data unavailable" toast and allow manual retry.

### 3.4 User & Role Management  

FR-WEBPRT-06 User Grid (Implements L1-FRS-R-222)  
• `GET /v1/users?role=` filters users; KAM receives read-only grid.  
• Pagination page size default = 25 (applies to all grids system-wide for consistency).

FR-WEBPRT-07 Create / Edit / Deactivate (Admin only)  
• Admin actions: POST `/v1/users`, PATCH `/v1/users/{id}` supports inline deactivation toggle.  
• On success the grid row updates without full reload.  
• KAM sees read-only view with no action buttons enabled.  

### 3.5 Master Data Maintenance  

#### 3.5.1 Product Catalogue  

FR-WEBPRT-08 Manual CRUD (Implements L1-FRS-R-231)  
• Admin can add or edit a product via modal → POST/PUT endpoints.  
• KAM sees read-only grid with disabled action buttons.  

FR-WEBPRT-09 Bulk CSV Upload (Implements L1-FRS-R-232)  
• Flow:  
  1. POST `/v1/csv-jobs/products/presign` {fileName}.  
  2. Browser PUTs to SAS URL.  
  3. POST `/v1/csv-jobs` registering job.  
  4. Redirect to "Job Details" page polling `GET /v1/csv-jobs/{id}` every 10 s.  
   • Display spinner with "Processing... (elapsed time: X seconds)" during PROCESSING status.  
   • Show "Validating uploaded data..." message after 30 seconds.
• Accept only `text/csv`, ≤50 MB (KD-WEBPRT-04).  
• SAS token TTL = 15 min; portal requests a fresh URL on HTTP 403.  
• Validation errors surface via a downloadable reject file once job state = `FAILED`.  
• Give-up threshold: if status remains `PROCESSING` >10 min, show "Job is taking longer than expected— we'll e-mail you when ready."  
• KAM cannot access bulk upload functionality.  
• System processes 1 concurrent CSV job per entity type (products, stores, users) with 10,000 rows per batch.

#### 3.5.2 Store Registry  

FR-WEBPRT-10 Store CRUD and Hierarchy (Implements L1-FRS-R-241)  
• Admin may create / edit stores using POST/PUT endpoints.  
• KAM has read-only access to the same grid.  
• Same bulk CSV flow as FR-WEBPRT-09 but entity =`stores`.

#### 3.5.3 CSV Templates and Schemas

FR-WEBPRT-19A CSV Template Downloads
• Portal provides downloadable CSV templates with headers for each entity type.
• Templates accessed via "Download Template" button on each bulk upload screen.
• Templates include:
  - Products: SKU, Name, Type, Category, Price, Attributes (JSON)
  - Stores: StoreId, Name, ParentId, City, State, PinCode, Zone, DoorType, ServiceType, EL360, EE, LIS
  - Users: Phone, Email, Name, Role, StoreIds (comma-separated)
• All CSVs must be UTF-8 encoded without BOM.

### 3.6 Group Management  

FR-WEBPRT-11 Simple Group Management (Implements L1-FRS-R-251)  
• Admin can create/edit groups by selecting stores/products from a list.  
• Multi-select dropdown or checkbox list for member selection.  
• POST `/v1/store-groups` or `/v1/product-groups` with array of IDs.  
• Maximum 500 stores or 1000 products per group.

### 3.7 Promotion Lifecycle Management  

FR-WEBPRT-12 Five-Step Wizard (KD-WEBPRT-06; Implements L1-FRS-R-261; Aligns with L1-WF §3.9.2)  
Step 1 Basic Information → Step 2 Promotion Type → Step 3 Rules → Step 4 Eligibility → Step 5 Review.  
• "Save Draft" enabled at every step – POST `/v1/promotions` with body field `"status":"DRAFT"`.  
• "Preview Eligibility" button calls `POST /v1/promotions/evaluate?mode=test`.  
• "Activate" triggers promotion creation; the backend sets status per rule.  
• Failure handling: 409 (rule compile error) surfaces as modal with backend message.  
• KAM users can view promotion list and details but cannot access the creation wizard.
• When multiple promotions qualify, the system automatically applies the promotion with the highest discount amount (customer-friendly approach).

FR-WEBPRT-13 Edit ACTIVE/SCHEDULED Promotion (Implements L1-FRS-R-262)  
• Editable fields limited to `description` and `endDate` only.  
• PATCH `/v1/promotions/{id}` with allowed subset; UI prevents others.  
• Admin-only; KAM sees read-only view.

### 3.8 Value-Upgrade List Maintenance  

FR-WEBPRT-14: DEPRECATED. PELASE SKIP. 

### 3.9 Transaction Oversight  

FR-WEBPRT-15 Transaction Search Grid (Implements L1-FRS-R-263)  
• `GET /v1/transactions` with query params: `storeId`, `status`, `promotionId`, `dateFrom`, `dateTo`, `invoiceNo`, pagination.  
• Admin sees all transactions; KAM sees read-only data scoped to mapped parent stores.  
• Failure handling: on 500 show retry option.

FR-WEBPRT-16 Status Transition  
• Admin only: perform `UNDER_REVIEW → VERIFIED → SETTLEMENT_COMPLETE` via PATCH `/v1/transactions/{id}/status`.  
• KAM: Read-only view of transaction status; no transition capabilities per key instruction #7.  
• UI disables invalid transitions per FSM.  
• On 409 conflict show modal with backend message; user may refresh grid.

### 3.10 Reporting & Exports  

FR-WEBPRT-17 Canned CSV Reports (Implements L1-FRS-R-271)  
• Buttons call `GET /v1/reports/performance` (returns 202 + jobId).  
• Poll `GET /v1/reports/{id}/download` every 15 s until HTTP 303 then redirect to SAS link.  
  - Display spinner with "Generating report... (elapsed time: X seconds)" during generation.  
  - After 60 seconds, add message: "Large reports may take several minutes."
• Historical reports listed; Admin may delete obsolete reports (DELETE `/v1/reports/{id}`) – endpoint exists in L2-LLD-IC §7.  
• KAM may download reports (considered a read operation) but cannot delete them.  
• Give-up threshold 10 min; UI offers "Notify me by e-mail when ready".

### 3.11 Banner Asset Management  

FR-WEBPRT-18 Immediate Replace Flow (Implements L1-FRS-R-233)  
• "Upload Banner" → backend presigns via `/v1/upload/presign` {fileName, type:"banner"} returning SAS URL (write-only, 15 min, no IP scope).  
• Accepted MIME types: `image/png`,`image/jpeg`,`image/gif`,`image/webp` **only**.  
• After successful PUT to SAS URL, UI calls `PATCH /v1/promotions/{id}/banner` {blobUrl} for promotion-specific banner or `PATCH /v1/settings/global-banner` {blobUrl} for system-wide banner. Endpoints pending formal addition to L2-LLD-IC §7. 
• New banner visible after browser cache-bust (`?v=<timestamp>`); no CDN layer assumed.  
• Admin-only feature; not visible to KAM users.  

### 3.12 Security, Compliance & Error-Handling Helpers  

FR-WEBPRT-19 XSS / CSRF Guards  
• All API calls executed via Axios instance injecting JWT; default header `X-Requested-With: XMLHttpRequest`.  
• No backend CSRF token required because cookies are SameSite=Strict.  

FR-WEBPRT-20 Date & Time Rendering  
• All timestamps received in ISO-8601 UTC; UI converts to browser locale using `moment-time-zone` and appends UTC indicator (e.g., "2025-08-02 12:34 UTC"). Tooltip shows full ISO string.  

FR-WEBPRT-21 Accessibility & i18n  
Dont Implement

FR-WEBPRT-22 Grid Pagination Standards  
• All data grids (Users, Products, Stores, Transactions, Reports) use uniform pagination:  
  - Default page size: 25 records  
  - User-selectable options: 10, 25, 50, 100  
  - Maximum records per page: 100 (backend enforced)  
• Pagination controls display: "Showing X-Y of Z records" with Previous/Next buttons.

FR-WEBPRT-23 API Request Timeout Configuration  
• All API requests enforce a 30-second timeout.  
• Automatic retry policy:  
  - GET requests: Single retry after 2-second delay on timeout  
  - POST/PATCH/PUT/DELETE: No automatic retry (user must manually retry)  
• On timeout after retry: Display "Request timed out. Please check your connection and try again."  
• File upload operations (SAS URLs) use extended 60-second timeout with no retry.

### 3.13 Error-Handling Matrix (portal-side)  

| Scenario | User Feedback | Retry Logic |
|----------|---------------|-------------|
| OTP expired / invalid | "Code expired or incorrect" banner | User may request new OTP |
| OTP lockout (5 failures) | "Account locked for 15 minutes" | No retry until lockout expires |
| CSV upload 413 | "File too large – max 50 MB" | Block further upload |
| Network offline | "You are offline" persistent toast | Auto-retry once connection detects online |
| SAS 403 (token expired) | "Upload URL expired, generating a new one …" | Automatically requests new presign |
| API timeout (30s) | "Request timed out - retrying..." (GET) or "Request timed out - please try again" | Auto-retry GETs once, manual for others |
| Field validation failure | Inline red text with specific format requirement | No retry, user corrects input |
| Long operation (>1 min) | "Still processing... Large operations may take several minutes" | Continue polling |
---

### 3.14 Additional Portal Features

FR-WEBPRT-24 Promotion Test Console (Admin only)  
• Accessible from promotion list view via "Test" action button.  
• Input: Select products (SKUs) and store context.  
• Calls `POST /v1/promotions/evaluate?mode=test` with test payload.  
• Displays eligible promotions with calculated discounts/benefits.  
• Results are display-only; no transactions created.

FR-WEBPRT-25 System Settings - Hierarchies (Admin only)  
• Manage product/store hierarchical relationships.  
• GET `/v1/settings/hierarchies` retrieves current configuration.  
• PATCH `/v1/settings/hierarchies` updates hierarchy rules.  
• Changes affect promotion eligibility calculations.

FR-WEBPRT-26 User Profile Management  
• Available to both Admin and KAM users.  
• GET `/v1/users/me` retrieves current user profile.  
• PATCH `/v1/users/me` updates own profile (name, email, phone).  
• Password/authentication changes not supported (OTP-only system).

## 4. Component Interfaces  

| Consumer | Provider | Protocol | Endpoint(s) Used | Purpose |
|----------|----------|----------|------------------|---------|
| WEBPRT | BCKND | HTTPS/JSON | `/v1/*` listed in §3 | Business operations |
| Browser | Azure Blob | HTTPS + SAS (15 min) | PUT / GET on SAS URL | CSV & image upload/download |

---

## 5. Data Requirements  

### 5.1 Client-Side State  

• JWT – volatile Pinia memory  
• UI cache (look-ups, dropdown data) – Vue reactive store, cleared on logout  
• No persistent PII in browser storage (KD-WEBPRT-08)

### 5.2 Upload Size & SAS Limits  

| File Type | MIME | Max Size | Container | SAS TTL | Permissions |
|-----------|------|----------|-----------|---------|-------------|
| Master CSV | text/csv | 50 MB | `assets` | 15 min | write-only |
| Banner | png, jpeg, gif, webp | 5 MB | `assets` | 15 min | write-only |

### 5.3 Report Retention  

Generated CSV reports are retained 24 months, after which they are auto-deleted by a lifecycle rule (Ops SOP §4).

---

## 6. Validation Rules  

| Rule ID | Description |
|---------|-------------|
| VAL-WEBPRT-01 | Reject file uploads violating MIME/size allow-list; show "Unsupported file" toast. |
| VAL-WEBPRT-02 | Promotion Wizard field presence per step – cannot advance until required fields valid. |
| VAL-WEBPRT-03 | Group Builder live JSON must pass schema validation before "Save". |
| VAL-WEBPRT-04 | Date pickers prevent selecting endDate < startDate or > 365 days ahead. |
| VAL-WEBPRT-05 | Numeric inputs (price, discount) limited to two decimal places, min 0. |
| VAL-WEBPRT-06 | On status change PATCH failure (409/status conflict) show modal with backend message. |
| VAL-WEBPRT-07 | All interactive elements reachable via keyboard and have visible focus indicators (WCAG 2.1 AA). |
| VAL-WEBPRT-08 | Text fields: Name/Title (max 100 chars), Description (max 500 chars), alphanumeric + spaces/hyphens. |
| VAL-WEBPRT-09 | Phone: Exactly 10 digits, pattern ^[6-9]\d{9}$ for Indian mobiles. |
| VAL-WEBPRT-10 | Email: RFC 5322 format validation, max 254 chars. |
| VAL-WEBPRT-11 | Currency/Price: Positive decimal, max 2 decimal places, max value 99999999.99. |
| VAL-WEBPRT-12 | SKU/Code fields: Alphanumeric + hyphen/underscore, max 50 chars, no spaces. |

---

## 7. Assumptions & Constraints  

1. Portal served from single Azure App Service behind office IP allow-list (KD-WEBPRT-01).  
2. No WAF, SSO, or multi-language support in Phase-1.  
3. All time-critical bulk operations constrained to ≤100 k rows per file.  
No audit-trail viewer functionality in portal per design decision; audit logs accessible only via Azure Storage Explorer for Ops team.  
5. All new endpoints referenced in this document (`/v1/dashboard`, `/v1/settings/value-upgrades`, `/v1/settings/banner`) will be added to L2-LLD-IC §7 and the OpenAPI contract before implementation begins.
6. No staging or pre-production environment exists; all deployments are in-place to production during designated maintenance windows per L1-HLD §9.
7. Platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) with no cross-region disaster recovery capability; this risk is explicitly accepted by EssilorLuxottica per L1-KD and L1-NFRS §5.

---

_End of file_