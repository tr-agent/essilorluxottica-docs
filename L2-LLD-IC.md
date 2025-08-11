
## L2-LLD-IC: Inter-Component Interaction Design Document

### 1. Purpose & Scope
This Level-2 document standardises **all run-time interactions** between the three runtime components of PromoPartner—Backend (BCKND), Web Portal (WEBPRT) and Mobile App (APP)—and the supporting Azure services (PostgreSQL, Blob, Queue, MSG91).  
All REST endpoints are rooted under the immutable base path **`/v1`**; this establishes a clear versioning contract across components.  
It is the single reference for:

* Public REST contract (paths, verbs, payload skeletons, status codes)  
* Role–endpoint permission matrix consumed by API middleware  
* Cross-component authentication & token lifecycle  
* File-transfer, asynchronous messaging and database-sync strategies  
* Uniform error envelope and data-security guarantees  

Internal algorithms and in-process class diagrams are intentionally excluded and deferred to L3-LLD documents.

---

### 2 Services and Modules Landscape
The table below purposefully hides in-process implementation details (workers, helpers, etc.) and lists only externally observable runtimes.

| Code | Runtime | Responsibility (external view only) |
|------|---------|-------------------------------------|
| **BCKND** | .NET 8 container in Azure App Service | Exposes REST API, persists to PostgreSQL, pushes OTP/e-mail jobs to Queue, streams files to Blob |
| **WEBPRT** | Nuxt 3 SPA served from Azure App Service | Calls BCKND for CRUD, renders Admin/KAM UI, streams CSV & banner files to Blob |
| **APP** | Flutter 3 iOS/Android binary | Calls BCKND for promotion discovery, OTP, transaction & reconciliation |
| **PG** | Azure PostgreSQL 16 | Authoritative relational store |
| **BLOB** | Azure Blob Storage | `assets` (mutable) & `audit` (immutable) containers |
| **QUEUE** | Azure Storage Queue | Decouples OTP / e-mail dispatch |
| **MSG91** | SaaS | SMS + e-mail delivery |

---

### 3. High-Level Communication Topology
The diagram now includes direct client↔Blob traffic for SAS-based uploads/downloads and clarifies that the Notification Worker executes **in-process** inside the Backend container.

```mermaid
flowchart LR
    subgraph Clients
        WEB[WEBPRT\nNuxt SPA]
        APP[APP\nFlutter]
    end

    BCKND((Backend API\n+ In-Process Workers))
    PG[(PostgreSQL)]
    BLOB[(Blob Storage)]
    Q[Storage Queue]
    MSG[MSG91]

    WEB -- HTTPS/JSON --> BCKND
    APP -- HTTPS/JSON --> BCKND
    WEB -- HTTPS (SAS) --> BLOB
    APP -- HTTPS (SAS) --> BLOB
    BCKND -- TLS --> PG
    BCKND -- HTTPS --> BLOB
    BCKND -- enqueue --> Q
    Q -- dequeue --> BCKND
    BCKND -- HTTPS --> MSG
```

* All synchronous traffic = JSON over TLS 1.2 (minimum, upgrade to 1.3 when App Service GA's support).  
* All async traffic = Base-64 JSON inside Storage Queue, AES-256 at rest.  
* No other network paths exist.

---

### 4. Communication Patterns  
PromoPartner uses two primary communication paradigms: synchronous REST for interactive workflows and asynchronous messaging for non-blocking tasks such as OTP delivery and overnight schedulers.

#### 4.1 Synchronous REST
* Stateless HTTP/1.1 requests with `Authorization: Bearer <JWT>` header.  
* JSON UTF-8 bodies; field names use `camelCase`.  
* Every URI starts with `/v1` (e.g., `/v1/auth/otp/request`).  
* Breaking changes require a new base path (`/v2`) and formal change notice.

#### 4.2 Asynchronous Messaging
* BCKND enqueues `{ otpId, code, channel, recipient, purpose }` on QUEUE.  
* In-process Notification Worker pops, calls MSG91, marks row `sent=true` in DB.  
* Retry & poison-message policies handled via Polly; details live in L3-LLD-BCKND.  
A JSON Schema (`otpMessage-v1.json`) stored in the contracts repo defines the queue payload and is validated on both enqueue and dequeue.

#### 4.3 File Transfer (CSV, banners, reports)
* BCKND issues **time-limited SAS URL** (`PUT` for upload, `GET` for download).  
* Client streams file directly to Blob, then calls `/v1/csv-jobs` or `/v1/reports/{id}` to start processing or fetch status.  
* Maximum single-file size = 50 MB (L1-NFRS §2).

---

### 5. Key Interaction Sequences  
Diagrams omit internal modules and show only observable actors.

#### 5.1 Login OTP (User Authentication)

```mermaid
sequenceDiagram
    participant C as Client (WEBPRT/APP)
    participant API as BCKND
    participant DB as PostgreSQL
    participant Q as Queue
    participant MSG as MSG91

    C->>API: POST /v1/auth/otp/request { phone/email }
    API->>DB: INSERT otp(row,purpose="login")
    API->>Q: enqueue(row)
    C->>API: POST /v1/auth/otp/verify { phone/email, code }
    API->>DB: SELECT/DELETE otp
    API-->>C: 200 { jwt, refreshToken }
```

#### 5.2 Customer-Verification OTP (Eligibility Token)

```mermaid
sequenceDiagram
    participant S as Store App
    participant API as BCKND
    participant DB as PostgreSQL
    participant Q as Queue
    participant MSG as MSG91

    S->>API: POST /v1/customer-otp/request { phone }
    API->>DB: INSERT otp(row,purpose="customer")
    API->>Q: enqueue(row)
    S->>API: POST /v1/customer-otp/verify { phone, code }
    API->>DB: SELECT/DELETE otp
    API-->>S: 200 { eligibilityToken }
```

#### 5.3 Promotion Evaluation ➔ Transaction Creation

```mermaid
sequenceDiagram
    participant APP as Mobile App
    participant API as BCKND
    participant DB as PostgreSQL

    APP->>API: POST /v1/promotions/evaluate { cart, storeId }
    API->>DB: SELECT activePromotions
    API-->>APP: 200 eligiblePromotions[]

    APP->>API: POST /v1/transactions { cart, promotionId, eligibilityToken }
    API->>DB: INSERT transaction(status="NEW")
    API-->>APP: 201 { transactionId }
```

#### 5.4 CSV Ingest (SAS Variant)

```mermaid
sequenceDiagram
    participant Admin as WEBPRT
    participant API as BCKND
    participant Blob as Azure Blob
    participant DB as PostgreSQL

    Admin->>API: POST /v1/csv-jobs/stores/presign { fileName }
    API-->>Admin: 200 { uploadUrl }
    Admin->>Blob: PUT file.csv
    Admin->>API: POST /v1/csv-jobs { entity:"stores", blobUrl:"..." }
    API-->>Admin: 202 { jobId }
    API->>DB: UPSERT batched rows   %% worker integrated in API process
    API->>API: PATCH job status
    Admin->>API: GET /v1/csv-jobs/{jobId}
    API-->>Admin: 200 { status, processed, rejected }
```

---

### 6. Canonical Envelope & Status Codes  
This envelope is shared by all successful and error responses, enabling uniform client-side deserialisation and tracing.

```json
// success
{
  "data": { /* resource-specific payload */ },
  "meta": { "traceId": "f571…" }
}

// error
{
  "error": {
    "code": "SKU_DUPLICATE",
    "message": "The supplied SKU already exists.",
    "details": { "sku": "ABC123" }
  },
  "meta": { "traceId": "f571…" }
}
```

| Category | HTTP Status | Typical `code` |
|----------|-------------|----------------|
| Validation error | 400 / 422 | `FIELD_REQUIRED`, `INVALID_FORMAT` |
| Auth/Z auth | 401 / 403 | `UNAUTHENTICATED`, `FORBIDDEN` |
| Conflict / FSM | 409 | `STATUS_TRANSITION_INVALID` |
| Not found | 404 | `RESOURCE_NOT_FOUND` |
| Rate-limit | 429 | `RATE_LIMIT_EXCEEDED` |
| Server | 500 | `UNEXPECTED_ERROR` |

---


_End of document_
```
