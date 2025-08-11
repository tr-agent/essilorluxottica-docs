## L3-IB-WEBPRT: Component Implementation Backlog for WEBPRT  
PromoPartner · Essilor Luxottica Joint-Promotions Platform  
Version 1.0 | 2025-08-08 | Status = DRAFT (awaiting PO sign-off)

---

### Legend  
* **Type** – US = User-Story, TT = Technical Task, CH = Chore  
* **Prio** – M = Must-Have (Phase-1 scope-lock) S = Should-Have (stretch)  
* **SP** – Fibonacci story-points (1-13); granularity ≤ 8 SP preferred  
* **→ Depends on** – blocking backlog item(s)  
* **Trace** – FR-ID from L3-FRS-WEBPRT | LD-Section from L3-LLD-WEBPRT | KD-ID from L3-KD-WEBPRT | TSD-ID from L3-TSD-WEBPRT (where applicable)  

---

## 1. Foundation & CI/CD

| ID | Title | Type | Short Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-001 | Nuxt 3 static preset scaffold | TT | Generate project with `nuxi init`, set `nitro.preset = 'static'`, Tailwind & TS enabled | `npm run generate` produces `/dist` with pre-rendered routes; no SSR server bundle present | LD §1, KD-02*, Clar.#1 | — | M | 3 |
| IB-WEBPRT-002 | Nginx container image | TT | Multi-stage Dockerfile: stage 1 = build, stage 2 = `nginx:1.25-alpine`; copy `/dist` → `/usr/share/nginx/html` | `docker run` serves `/index.html` and `/login` (history-mode fallback) | LD §1 | 001 | M | 3 |
| IB-WEBPRT-003 | GitHub Actions pipeline | TT | Build static site → build/push image to ACR → deploy to **staging slot** → smoke test → slot-swap | Successful run on PR shows green check, image tagged with short SHA | KD-07 | 001,002 | M | 5 |
| IB-WEBPRT-004 | ENV wiring & secrets template | CH | `.env.example` with `API_BASE_URL`, `NUXT_PUBLIC_DEBUG`, etc. | Running `make dev` loads vars without missing-key errors | LD §12 | — | M | 1 |

* KD-02 is now aligned with the static-site decision (“static + Nginx”).

---

## 2. Authentication & Session Management

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-010 | OTP request form | US | As an Admin/KAM I can request an OTP by entering my **e-mail** | POST `/auth/otp/request` fired; toast “Code sent” on 200 | FR-01 | 001 | M | 2 |
| IB-WEBPRT-011 | OTP verification flow | US | As an Admin/KAM I can submit the 6-digit code and log in | On 200: • JWT saved in Pinia (`accessToken`) • Refresh-token handled by backend via **Secure; HttpOnly; SameSite=Strict** cookie (SPA does **not** read or write it) • Route redirects to `/dashboard` | FR-01, LD §5.1, KD-03, KD-08 | 010 | M | 3 |
| IB-WEBPRT-012 | Silent refresh scheduler | TT | Timer triggers 5 min before `exp`; it calls `POST /auth/refresh` relying solely on the HttpOnly cookie. On 200 the new JWT replaces the old one in Pinia. | Visible token countdown resets; on 401 triggers logout | FR-02, LD §7.1 | 011 | M | 3 |
| IB-WEBPRT-013 | Logout action | US | As a user I can log out from the menu | Clears Pinia; redirects to `/login`; toast shown | FR-03 | 011 | M | 1 |
| IB-WEBPRT-014 | Route guard | TT | Global Nuxt middleware verifies JWT presence & role claim on every navigation | Un-authenticated user always lands on `/login`; authenticated cannot access `/login` again | LD §3 | 011 | M | 2 |

---

## 3. API Client & Error Handling

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-020 | Typed Axios wrapper | TT | Axios instance with `baseURL`, auth header, 1-retry idempotent policy | GET retry visible in network log; POST never retried | LD §4 ApiClient | 001 | M | 3 |
| IB-WEBPRT-021 | Global error interceptor | TT | Map canonical error envelope → toast / dialog; on 5xx consecutive failures show blocking modal | Matches Error Matrix in FR §3.13 | FR-19, LD §10 | 020 | M | 2 |
| IB-WEBPRT-022 | App Insights JS SDK | CH | Load in prod builds; forward `traceId` from response header | Exceptions visible in Azure portal with component = WEBPRT | LD §10 | 020 | S | 2 |

---

## 4. Navigation & RBAC

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-030 | Side-Nav component | TT | Build collapsible Tailwind side-nav reading menu map JSON | Active route highlighted; menu collapsed persists in `localStorage` (non-PII) | FR-04, LD §4 AppShell | 014 | M | 3 |
| IB-WEBPRT-031 | Role-aware menu generation | TT | Decode `role` claim from JWT; hide DAG of routes accordingly | KAM cannot access `/settings` even via URL | FR-04 | 030 | M | 2 |
| IB-WEBPRT-032 | Breadcrumb bar | CH | Automatic from Nuxt route meta | Breadcrumb matches hierarchy for deep screens | LD §4 AppShell | 030 | S | 1 |

---

## 5. Dashboard

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-040 | Dashboard page skeleton | TT | Route `/dashboard` loads card grid placeholders | Empty state w/ spinner, responsive at 1280 px breakpoint | FR-05 | 030 | M | 2 |
| IB-WEBPRT-041 | KPI fetch & render | TT | Consume agreed `/v1/dashboard` API (to be added in L2 contract); render 8 KPI cards; on error show toast | Snapshot renders in ≤ 600 ms avg (mock) | FR-05 | 040,020 | M | 3 |

---

## 6. User & Role Management

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-050 | UsersStore Pinia module | TT | Normalised cache w/ pagination support | `UsersStore.list` returns array size = query limit | LD §4 PiniaStores | 020 | M | 2 |
| IB-WEBPRT-051 | Users grid (Admin + KAM readonly) | US | Tabular view with filters, 25-row default page | KAM cannot see “Add User” button | FR-06 | 050,031 | M | 3 |
| IB-WEBPRT-052 | Create user modal | US | Admin enters fields; POST `/users`; optimistic row insert | Success toast; form validates mandatory fields | FR-07 | 051 | M | 3 |
| IB-WEBPRT-053 | Edit / deactivate toggle | US | Inline edit icon; PATCH `/users/{id}`; row updates without reload | Deactivated user shows grey badge | FR-07 | 051 | M | 2 |

---

## 7. Master Data – Products & Stores

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-060 | ProductsStore module | TT | Holds products list + filters | Store invalidated when JWT refreshes | LD §4 | 020 | M | 2 |
| IB-WEBPRT-061 | Products CRUD UI | US | Admin modal create/edit; PUT/POST endpoints | Validation of SKU uniqueness (surfaced from 409) | FR-08 | 060 | M | 3 |
| IB-WEBPRT-062 | Products CSV upload wizard | US | 4-step flow with presign, PUT to SAS, job register, polling | 50 MB happy-path completes **≤ 80 s on a 5 Mbps link** | FR-09, LD §5.2 | 020 | M | 5 |
| IB-WEBPRT-063 | Stores grid & CRUD | US | Mirror functionality for stores + hierarchy column | KAM grid readonly; Admin CRUD enabled | FR-10 | 060 | M | 5 |
| IB-WEBPRT-064 | Stores CSV upload flow | TT | Re-use component from 062, entity=`stores` | Same acceptance as 062 | FR-10 | 062 | M | 1 |

---

## 8. Group Management

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-070 | Rule-builder component | TT | Drag-drop UI with condition blocks, produces JSON via Zod schema | Invalid schema disables Save; live preview JSON read-only | FR-11 | 020 | M | 8 |
| IB-WEBPRT-071 | Store/Product group CRUD pages | US | Wrapper around component 070; POST endpoint call | Success toast; redirect to list screen | FR-11 | 070 | M | 3 |

---

## 9. Promotion Lifecycle

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-080 | Wizard skeleton & routing | TT | `/promotions/new` 4-step Nuxt child routes | Browser refresh stays on same step (hash) | FR-12, LD §4 PromotionWizard | 031 | M | 3 |
| IB-WEBPRT-081 | Step 1 – Basics | US | Inputs: name, type, start/end date, description | Validation: required, date ranges (VAL-04) | FR-12 | 080 | M | 3 |
| IB-WEBPRT-082 | Step 2 – Eligibility | US | Select store/product groups; preview eligibility by calling evaluate API | Shows inline percentage of eligible stores | FR-12 | 081 | M | 5 |
| IB-WEBPRT-083 | Step 3 – Benefit | US | Configure discount logic fields; JSON preview | Numeric inputs enforce VAL-05 | FR-12 | 082 | M | 3 |
| IB-WEBPRT-084 | Step 4 – Test & Review | US | Render summary; “Save draft” & “Activate” buttons | POST `/promotions` with status =DRAFT or ACTIVE; toast; redirect | FR-12 | 083 | M | 2 |
| IB-WEBPRT-085 | Edit limited fields | US | For ACTIVE/SCHEDULED, allow description & endDate only | UI forbids others; PATCH accepted (endpoint addition pending L2 contract update) | FR-13 | 080 | M | 2 |

---

## 10. Value-Upgrade List

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-090 | Settings – Value-Upgrade grid | US | Sortable list with drag handle | PATCH sends new sequence; order persists on reload | FR-14 | 031,020 | M | 3 |

---

## 11. Transactions Oversight

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-100 | TransactionsStore module | TT | Fetch list with filters & pagination; reuse table component | 500-response triggers retry banner | FR-15 | 020 | M | 3 |
| IB-WEBPRT-101 | Status transition dialog | US | Admin only: dropdown to change FSM state | PATCH `/transactions/{id}/status`; invalid transition disabled | FR-16 | 100 | M | 3 |

---

## 12. Reporting & Exports

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-110 | Reports page | US | List historical downloads; “Generate” buttons | GET returns 202; polling logic & 303 redirect; give-up after 10 min offers e-mail notify | FR-17 | 020 | M | 5 |


---

## 13. Banner Asset Management

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-120 | Upload banner flow | US | File input → presign → PUT to SAS → Persist banner reference via **endpoint to-be-defined in updated L2 contract** | Accepts only image MIME list; after success appends `?v=timestamp` to `<img>` src | FR-18, LD §5.2 | 020 | M | 3 |

---

## 14. Accessibility & i18n

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-130 | WCAG focus & keyboard sweep | CH | Ensure all interactive elements reachable via Tab; visible focus style | Axe CLI reports zero critical WCAG 2.1 AA violations | FR-21, VAL-07 | 041-120 | M | 3 |
| IB-WEBPRT-131 | Centralised copy strings | TT | Extract literal strings → `/src/constants/messages.ts` | Mutation of JSON entry updates UI without rebuild in dev | FR-21 | 001 | S | 2 |

---

## 15. Security Hardening

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-140 | CSP header (report-only) | TT | Configure Nginx `Content-Security-Policy-Report-Only` default | Header present in prod, absent in dev; report URI points to App Insights | NFR §4.3-5 | 002 | S | 2 |
| IB-WEBPRT-141 | Referrer & HSTS headers | TT | Add `Strict-Transport-Security`, `Referrer-Policy`, `X-Frame-Options` | OWASP ZAP pass | NFR §4.3-4 | 002 | M | 1 |

---

## 16. Observability & Ops

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-150 | Build-time Sentry DSN switch removed | CH | Remove legacy Sentry lib; rely solely on App Insights | No Sentry JS loaded in network tab | KD-07 | 001 | S | 1 |
| IB-WEBPRT-151 | Health-probe endpoint | TT | Expose `/ping.html` static file for App Service health checks | Azure shows container Healthy; file returns 200 | Ops SOP | 002 | M | 1 |

---

## 17. Test Automation

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-160 | Unit test harness | CH | Vitest configured; 80 % statement coverage gate | CI fails < 80 % | Dev policy | 001 | M | 2 |
| IB-WEBPRT-161 | Cypress smoke suite | TT | Login, dashboard load, CSV upload stubbed | Runs in pipeline after deploy-to-staging | FR-01-05 | 003 | M | 5 |

---

## 18. Documentation & Handover

| ID | Title | Type | Description | Acceptance Criteria | Trace | → Depends on | Prio | SP |
|----|-------|------|-------------|---------------------|-------|-------------|------|----|
| IB-WEBPRT-170 | README update | CH | Cover local dev, lint, build, docker run instructions | New engineer can start dev in ≤ 15 min | KD-07 | 001 | M | 1 |
| IB-WEBPRT-171 | Admin user manual draft | CH | Markdown with screenshots of all screens | Reviewed by PO; moved to Confluence | FRS all | 041-120 | S | 3 |

---

### 2. Cross-Item Dependencies Graph (excerpt)

```
IB-001 ─┬─► 002 ─► 003
        └─► 020 ─► {040,050,060,070}
020 ─► 021 ─► 041
014 ─► 030 ─► 031
030 ─► {040…120}
062 ─► 064
080 ─► 081 ─► 082 ─► 083 ─► 084
```

---

### 3. Release-Cut Candidate (Minimum Increment)

To ship a functional but minimal Phase-1 slice (M-items only) the following backlog IDs must be completed:  
`001-004,010-014,020-022,030-032,040-041,050-053,060-063,070-071,080-085,090,100-101,110,120,130,141,150-151,160-161,170`.  
All **S-priority** items may follow in a patch release.

---

### 4. Traceability Matrix (sample)

| Backlog ID | FRS-Req | LLD-Section | KD | TSD | Notes |
|------------|---------|-------------|----|-----|-------|
| 011 | FR-WEBPRT-01 | §5.1 | KD-03 | — | Refresh-token handled via HttpOnly cookie |
| 062 | FR-WEBPRT-09 | §5.2 | KD-05 | TSD-UPL-01 | CSV 50 MB flow |
| 120 | FR-WEBPRT-18 | §5.2 | KD-04 | — | Image MIME allow-list |

(Full CSV traceability sheet stored at `docs/traceability/ib-webprt.xlsx`.)

---

### 5. Estimation Summary  

  
Total Must-Have SP = 84.  Detailed scheduling is managed in the separate project-planning tool.

---

_End of backlog file_