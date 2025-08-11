## L3-FRS-BCKND: Functional Requirements Specification (FRS) for BCKND  

### 1. Purpose & Scope  
This document specifies every externally observable function, interface, data contract, and validation rule that the Backend (BCKND) component of PromoPartner **must** implement.  
It refines the high-level requirements contained in L1-FRS and the cross-component design in L2-LLD-IC, translating them into atomic, testable backend behaviours that will be decomposed into the implementation backlog (L3-IB-BCKND).  
Only needs that are in-scope for the current single-region, ≤5 000-user deployment are covered; no speculative future-scale features are added.  
A separate Traceability Appendix (§8) maps every FR-BCKND-xxx number to its originating L1-FRS paragraph and, where applicable, to the relevant L3-KD-BCKND decision.  

---

### 2. Functional Requirements  

For traceability every requirement cites its L1-FRS source section and, where relevant, a KD-BCKND reference.

#### 2.1  Authentication & Session Management  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
| 1 | FR-BCKND-101 | Issue a login OTP (6-digit numeric) via both SMS and e-mail and enqueue separate queue jobs for each delivery channel. Rate-limit: ≤5 requests / 10 min per identity. | `POST /v1/auth/otp/request` | L1-FRS §2.1 (Login OTP) |
| 2 | FR-BCKND-102 | Verify login OTP, mark the OTP row `consumed_at = NOW()` without deleting it, and issue an RS256 JWT (TTL 60 min) + refresh token(s) – multiple concurrent refresh tokens are allowed, one per device/session. | `POST /v1/auth/otp/verify` | L1-FRS §2.1 |
| 3 | FR-BCKND-103 | Exchange a valid refresh token for a new JWT / refresh pair, rotating only the presented refresh token. | `POST /v1/auth/refresh` | L1-FRS §2.1 |
| 4 | FR-BCKND-104 | Immediately revoke all active tokens on user de-activation by incrementing `tokenVersion`. | Internal trigger executed inside the `PATCH /v1/users/{id}` handler when `active=false`. | L1-FRS §2.1, KD-BCKND-2 |
| 5 | FR-BCKND-105 | Issue customer-verification OTP (≤3/15min) via both SMS and e-mail channels, verify it, then create a **single-use opaque eligibility token** (GUID v4, TTL 15 min) stored server-side. | `POST /v1/customer-otp/request`, `POST /v1/customer-otp/verify` | L1-FRS §2.1 (Customer OTP) |
| 6 | FR-BCKND-106 | Reject any JWT whose `ver` claim differs from the current `tokenVersion`. | (middleware) | L2-LLD-IC §9 |
| 7 | FR-BCKND-107 | Enforce lockout after 5 failed login OTP attempts within 10 minutes, triggering a 15-minute lockout window. Return HTTP 429 with `ACCOUNT_LOCKED` error code. | (middleware) | L1-FRS §2.1, Key Instruction #8 |
| 8 | FR-BCKND-108 | Enforce lockout after 3 failed customer OTP attempts, triggering a 15-minute lockout window. Return HTTP 429 with `ACCOUNT_LOCKED` error code. | (middleware) | L1-FRS §2.1, Key Instruction #8 |

#### 2.2  User & Role Management  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
| 9 | FR-BCKND-201 | List users with optional `role` filter, offset pagination `page/size` (default 50, max 200). | `GET /v1/users` | L1-FRS §2.2 |
| 10 | FR-BCKND-202 | Create a user; Parent-Store Users are restricted to their own hierarchy. | `POST /v1/users` | L1-FRS §2.2 |
| 11 | FR-BCKND-203 | Update or deactivate user with optimistic concurrency (`If-Match: <etag>`). Deactivation triggers FR-BCKND-104. | `PATCH /v1/users/{id}` | L1-FRS §2.2 |
|12 | FR-BCKND-204 | Bulk create / upsert users via CSV ingest (see FR-BCKND-801). | `POST /v1/csv-jobs/users/presign`, `POST /v1/csv-jobs` | L1-FRS §2.2 |

#### 2.3  Master-Data Maintenance  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|13 | FR-BCKND-301 | List products (combined Lens + Frame) with filter query string and offset paging. | `GET /v1/products` | L1-FRS §2.3.1 |
|14 | FR-BCKND-302 | Create a single product. | `POST /v1/products` | L1-FRS §2.3.1 |
|15 | FR-BCKND-303 | Upsert (create or update) product by `sku`. | `PUT /v1/products/{sku}` | L1-FRS §2.3.1 |
|16 | FR-BCKND-304 | List stores (respecting role scope) and create store; Parent-Store Users may create children only. | `GET /v1/stores`, `POST /v1/stores` | L1-FRS §2.3.2 |

#### 2.4  Group Management  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|17 | FR-BCKND-401 | Create and manage Store Groups as simple lists of store IDs. | `/v1/store-groups` (GET, POST, PUT, DELETE) | L1-FRS §2.4 |
|18 | FR-BCKND-402 | Create and manage Product Groups as simple lists of SKUs. | `/v1/product-groups` (GET, POST, PUT, DELETE) | L1-FRS §2.4 |


#### 2.5  Promotion Lifecycle & Rule Engine  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|19 | FR-BCKND-501 | List promotions, filter by status. | `GET /v1/promotions` | L1-FRS §2.5 |
|20 | FR-BCKND-502 | Create promotion via multi-step wizard JSON; validate rule grammar (minimal v1) and benefit types (percentage, fixed amount). | `POST /v1/promotions` | L1-FRS §2.5 |
|21 | FR-BCKND-503 | Change promotion status respecting allowed transitions; membership freeze occurs on activation. Direct Draft ⇄ Paused transitions are **not** allowed. | `PATCH /v1/promotions/{id}/status` | L1-FRS §2.5 |
|22 | FR-BCKND-504 | Nightly scheduler flips SCHEDULED→ACTIVE and archives only promotions whose `endDate < NOW()` (ACTIVE→ARCHIVED). | (Scheduler worker) | L1-FRS §2.5 |

#### 2.6  Promotion Evaluation & Application  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|23 | FR-BCKND-601 | Evaluate promotions for a cart + store and return ordered list by customer savings. | `POST /v1/promotions/evaluate` | L1-FRS §2.5 |
|24 | FR-BCKND-602 | If client omits choice, auto-apply top-ranked promotion (tie-break: higher savings → earliest `startDate` → lowest `promotionId`). | (server-side in FR-BCKND-701) | L1-FRS §2.5 |

#### 2.7  Transaction Processing  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|25 | FR-BCKND-701 | Create transaction (`status=NEW`); idempotent via `request_uuid` retained 24 h. Duplicate UUID returns **201** with the original payload (OpenAPI flag `x-duplicate=true`). | `POST /v1/transactions` | L1-FRS §2.6 |
|26 | FR-BCKND-702 | List transactions with role-based scoping and pagination. | `GET /v1/transactions` | L1-FRS §2.6 |
|27 | FR-BCKND-703 | Add reconciliation data (`invoiceNo`, `pidNo`); no automatic status change. | `PATCH /v1/transactions/{id}/reconcile` | L1-FRS §2.6 |
|28 | FR-BCKND-704 | Status transition NEW→VERIFIED→COMPLETE by **Admin only**. KAM users have read-only access. | `PATCH /v1/transactions/{id}/status` | L1-HLD §10 |

#### 2.8  CSV Ingest & Bulk Jobs  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|29 | FR-BCKND-801 | Direct CSV upload and processing for files <10MB via multipart form. Synchronous response with success/error rows. | `POST /v1/csv-import/{entity}` | L1-FRS §2.3 |
|30 | FR-BCKND-802 | Reserved for Phase 2: Async processing for files ≥10MB | - | - |
|31 | FR-BCKND-803 | DEPRECATED |
|32 | FR-BCKND-804 | Internal worker updates job status (`processed/rejected`) via PATCH. | `PATCH /v1/csv-jobs/{jobId}` (internal identity) | L1-FRS §2.3 |

#### 2.9  Reporting & Dashboards  

| # | Req-ID | Description | Endpoint(s) & Methods | Traceability |
|---|--------|-------------|-----------------------|--------------|
|33 | FR-BCKND-901 | Trigger on-demand "Promotion Performance" CSV for arbitrary `startDate/endDate` (range ≤365 days); always freshly generated. | `GET /v1/reports/performance` | L1-FRS §2.7 |
|34 | FR-BCKND-902 | Provide downloadable SAS URL when report is ready (303 redirect). | `GET /v1/reports/{id}/download` | L1-FRS §2.7 |
|35 | FR-BCKND-903 | Trigger on-demand "Store Leaderboard" CSV for arbitrary `startDate/endDate` (range ≤365 days); always freshly generated. | `GET /v1/reports/leaderboard` | L1-FRS §2.7 |
|36 | FR-BCKND-904 | Return dashboard KPIs: activePromotions, txToday, txLast30d, totalDiscountLast30d, activeStores. Response time <2 seconds. | `GET /v1/dashboard` | L1-FRS §2.7 |

#### 2.10  Notification & Messaging  

| # | Req-ID | Description | Mechanism | Traceability |
|---|--------|-------------|-----------|--------------|
|36 | FR-BCKND-1001 | Enqueue both SMS and e-mail jobs upon OTP creation; payload conforms to INTG3P §4 mapping. | Azure Storage Queue `otpMessage` | L1-FRS §2.8 |
|37 | FR-BCKND-1002 | Notification Worker dequeues, calls MSG91 APIs for both SMS and e-mail delivery, updates `otp.sent=true`, retries 3× with back-off, marks FAILED on expiry (15 min). | (in-process worker) | L2-LLD-INTG3P §7 |

#### 2.11  Audit & Compliance  

| # | Req-ID | Description | Storage | Traceability |
|---|--------|-------------|---------|--------------|
|38 | FR-BCKND-1101 | Persist immutable audit event `{eventId, ts, actorId, entityType, entityId, action, changedFields}` for every create/update/delete & status change. | Blob container `audit/` | L1-FRS §2.9, KD-BCKND-3 |
|39 | FR-BCKND-1102 | Exclude OTP codes, eligibility tokens and file attachments from audit payloads. | — | KD-BCKND-2 |

#### 2.12  Scheduler & House-Keeping  

| # | Req-ID | Description | Mechanism | Traceability |
|---|--------|-------------|-----------|--------------|
|40 | FR-BCKND-1201 | Mark expired `eligibility_token` rows with `expired=true` flag every 15 min (no deletion). | Scheduler worker with advisory lock | KD-BCKND-3 (indefinite retention) |
|41 | FR-BCKND-1202 | On the 28ᵗʰ each month auto-create next-month partitions for `transactions` & `otp`. | Scheduler worker | KD-BCKND-6 |
|42 | FR-BCKND-1203 | Nightly VACUUM ANALYZE and index-health report. | Scheduler worker | KD-BCKND-6 |
|43 | FR-BCKND-1204 | Execute all scheduled jobs sequentially using a global scheduler lock (`pg_advisory_lock`) to prevent concurrent execution. Jobs queued by trigger time, with partition creation having highest priority. | Scheduler coordinator | Operational requirement |

---

### 3. Component Interfaces  

1. **REST/JSON** – `/v1/**` endpoints listed above; TLS 1.2 (min); Bearer JWT.  
2. **Azure PostgreSQL** – primary relational store accessed via EF Core; TLS enforced.  
3. **Azure Blob Storage**  
   • `assets` (mutable) – CSVs, banners, invoice attachments, reports  
   • `audit` (immutable) – audit events (append-only JSON)  
4. **Azure Storage Queue** – single queue `otpMessage` (base-64 JSON payload).  
5. **MSG91** – REST API for both SMS & e-mail delivery (via Notification Worker).  

No other outbound or inbound integrations exist for BCKND in phase 1.

---

### 4. Data Requirements  

#### 4.1 Primary Tables (excerpt)

| Table | PK | Notes |
|-------|----|-------|
| `users` | `user_id` | role enum, `tokenVersion` (int) |
| `refresh_tokens` | `token_id` | `user_id`, `expires_at` |
| `otp_YYYY_MM` | `otp_id` | monthly partition; `code`, `purpose`, `sent`, `consumed_at` |
| `eligibility_tokens` | `token_id` | `user_id`, `expires_at`, `expired` (boolean) |
| `products` | `sku` | uniqueness across all product types |
| `stores` | `store_id` | parent_id for hierarchy |
| `store_groups` / `product_groups` | `group_id` | expression JSON, frozen list snapshot |
| `promotions` | `promotion_id` | status enum, start/end |
| `promotion_rules` | `rule_id` | raw JSON + signed SQL |
| `transactions_YYYY_MM` | `transaction_id` | status FSM, `request_uuid` |
| `csv_jobs` | `job_id` | entity, progress, blob_url | 

#### 4.2  Queue & Blob Schemas  
See L2-LLD-INTG3P §4 for the full JSON schema of `otpMessage`.  
Reject files follow fixed-order CSV with extra column `errorMessage`.

---

### 5. Validation Rules (supersede L1-FRS §5 where refined)

1. `sku` – unique, non-empty, ≤30 chars.  
2. Phone – `^[6-9]\d{9}$`; stored E.164 `+91xxxxxxxxxx`.  
3. E-mail – RFC 5322.  
4. OTP – exactly 6 digits; validity 10 min (login) / 5 min (customer).  
5. Eligibility Token – GUID v4, single use, TTL 15 min.  
6. CSV file – UTF-8, fixed column order, ≤50 MB.  
7. Invoice attachment – MIME `application/pdf|image/jpeg|image/png`; size ≤5 MB; stored in `assets/invoices/`; SAS URL TTL ≤24 h, HTTPS only.  
8. `request_uuid` – retained 24 h; duplicates return 201 with `x-duplicate=true`.  
9. Promotion rule – minimal grammar; compiled SQL length ≤10 000 chars.  
10. FSM – allowed transitions only; VOID not permitted.  
11. Pagination – `page≥1`, `size 1…200`; default sort `createdAt DESC`.
12. Login OTP – max 5 attempts within 10 minutes triggers 15-minute lockout; max 3 resends per 15 minutes with 60s cooldown between resends.
13. Customer OTP – max 3 attempts triggers 15-minute lockout; max 3 resends per 15 minutes with 60s cooldown between resends.
14. Reconciliation uniqueness – Both `(store_id, invoice_no)` and `(store_id, pid_no)` must be unique independently.


### 6. Assumptions & Constraints  

* Single-region Azure deployment; vertical scale-up only.  
* All PII and business data are retained indefinitely, including `eligibility_tokens` (marked with `expired` flag instead of deletion).  
* OTP rows are stored in plaintext but **never deleted** (KD-BCKND-2).  
* Secrets stored exclusively in encrypted App-Service settings.  
* No DR, payment, ERP or SSO integrations in phase 1.  

---

### 7. API Conventions  

All REST APIs conform to the canonical envelope defined in L2-LLD-IC §6.  
Special status codes:  
• 201 – resource created; idempotent re-posts on the same `request_uuid` also return 201 with header `X-Duplicate: true`.  
• 429 – rate-limit exceeded (OTP requests) or account locked.  
• 422 – validation error (CSV ingest, rule compile, etc.).  

Clients must honour the `Retry-After` header on 429 responses.

---

_End of document_