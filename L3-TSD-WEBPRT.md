## L3-TSD-[WEBPRT]: Technology Stack Documentation (TSD) for [WEBPRT] Document

### 1. Purpose & Scope  

This document captures every technology, framework, runtime, and external dependency required to build and operate the PromoPartner Web-Portal (component code: **WEBPRT**).  It refines the solution-wide choices recorded in L1-TSD and locks them to component-specific versions or variants so that the WEBPRT delivery team can reproduce builds, generate SBOMs, and request support without ambiguity.  All items apply **only** to WEBPRT; system-wide or backend-only stacks are intentionally omitted.

### 2. Component Runtime Model  

WEBPRT is delivered as a **fully pre-rendered static Nuxt 3 site** (`nitro preset=static`).  
At runtime the container exposes **only Nginx 1.25-alpine** which serves static assets over port 80; no Node process is present in production.  
Node 20 LTS is used **exclusively during the CI build stage**.

**Note:** This static rendering approach supersedes earlier SSR/universal mode specifications in L3-KD-WEBPRT-02, optimized for the current ≤1,000 concurrent admin session scale.


### 3. Programming Languages & Toolchains  

| Domain | Language / Tool | Locked Stream | Notes |
|--------|-----------------|---------------|-------|
| Build-time scripts | Node.js | 20.x LTS (`node:20-slim`) | Used in first Docker stage only |
| Front-end code | TypeScript | 5.1.6 | `tsconfig` extends Nuxt default |
| Stylesheets | Tailwind CSS | via @nuxtjs/tailwindcss 6.8.0 | `@tailwindcss/forms`, `@tailwindcss/typography` enabled |
| Static server | Nginx | 1.25-alpine | Final runtime image layer |
| Shell scripting | POSIX sh | Alpine 3.19 default | For container entrypoint & health-probe |

### 4. Frameworks & Client-Side Libraries  

| Category | Library | Version Pin | Usage |
|----------|---------|-------------|-------|
| SPA Framework | Nuxt | 3.5.0 | Static preset, Vue 3 underneath |
| Core UI | Vue | 3.3.4 | Composition API only |
| State Mgmt | Pinia | 2.1.6 | Global stores per entity |
| State Mgmt Integration | @pinia/nuxt | 0.4.11 | Nuxt integration for Pinia |
| Data Fetching | @tanstack/vue-query | 4.32.0 | Server state management & caching. Configure with `credentials: 'include'` in query client defaults for cookie-based auth |
| Validation | Zod | 3.21.4 | Mirrors server DTOs |
| Form Builder | @formkit/nuxt | 1.0.0 | Promotion wizard forms |
| CSS Framework | @nuxtjs/tailwindcss | 6.8.0 | Utility-first CSS |
| UI Components | @nuxt/ui | 2.8.0 | Tailwind-based components |
| Headless Components | @headlessui/vue | 1.7.14 | Accessible UI primitives |
| Select Input | @vueform/multiselect | 2.6.2 | Advanced select components |
| Charting | chart.js | 4.3.3 | Data visualization |
| Chart Integration | vue-chartjs | 5.2.0 | Vue wrapper for Chart.js |
| Icons | nuxt-icon | 0.4.2 | Icon system integration |
| Dark Mode | @nuxtjs/color-mode | 3.3.0 | Theme switching support |
| Composition Utils | @vueuse/nuxt | 10.2.1 | Nuxt integration |
| Composition Core | @vueuse/core | 10.2.1 | Vue composition utilities |
| Authentication | @sidebase/nuxt-auth | 0.5.0 | Auth module for Nuxt |
| Auth Core | @auth/core | 0.10.0 | Core auth logic |
| JWT Handling | jose | 4.14.4 | JWT verification & parsing |
| Date Handling | date-fns | 2.30.0 | Date manipulation & formatting |
| Number Formatting | numeral | 2.0.6 | Number & currency formatting |
| HTML Sanitization | dompurify | 3.0.5 | XSS prevention |
| Telemetry | @microsoft/applicationinsights-web | 3.0.6 | Azure App Insights browser telemetry |
| Azure SDK | @azure/storage-blob | 12.17.0 | SAS-based file uploads to Blob Storage |
| API Types | openapi-typescript | 6.7.5 | Type generation from OpenAPI spec |

**Missing from provided stack (Recommendations):**

| Category | Library | Recommendation |
|----------|---------|----------------|
| Telemetry | @microsoft/applicationinsights-web 3.x | **Required** - Needed for Azure App Insights integration per infrastructure requirements |
| Azure SDK | @azure/storage-blob 12.x | **Required** - Essential for SAS-based file uploads to Blob Storage |
| API Types | openapi-typescript 6.x | **Recommended** - Ensures type safety with backend API contract |

**Development & Testing Tools:**

| Category | Library | Version Pin | Usage |
|----------|---------|-------------|-------|
| Linting | eslint | 8.45.0 | Code quality enforcement |
| Formatting | prettier | 3.0.1 | Code formatting |
| Testing | vitest | 0.33.0 | Unit & integration testing |

**Version Strategy:** All dependencies are locked to exact versions to ensure build reproducibility. Updates are managed through Dependabot PRs after testing.

**Client-Side Storage:** JWT and refresh tokens are stored in HTTP-only secure cookies set by the backend, with no sessionStorage or localStorage usage for authentication data.

### 5. Container Image & Build Strategy  

| Stage | Base Image | Key Steps |
|-------|------------|-----------|
| builder | `node:20-slim` | `pnpm install ­--frozen-lockfile` → `pnpm run build:static` |
| runtime | `nginx:1.25-alpine` | copy `.output/public → /usr/share/nginx/html`<br/>copy `nginx.conf → /etc/nginx/conf.d/default.conf` |

Final image size ≈ 55 MB; CVE scans run in GitHub Advanced Security.

### 6. Hosting & Infrastructure Services  

| Layer | Azure Service | Config / Tier | Component-Specific Notes |
|-------|---------------|---------------|-------------------------|
| Compute | Azure App Service (Linux container) | Plan: P1v2 (1 vCPU, 3.5 GB) | Image pulled from ACR; port 80 exposed; office-IP allow-list enforced |
| Container Registry | Azure Container Registry | SKU: Basic | Holds signed WEBPRT images (`webprt:<git-sha>`) |
| DNS / TLS | App Service Managed Certs | TLS 1.2 | `portal.promopartner.xxx` cname |
| Observability | Azure Application Insights | N/A | Browser SDK auto-collects page & error telemetry |
| Storage (direct) | Azure Blob Storage | Hot tier | Used via client-side SAS for CSV & banner uploads |
| Data Tier | None directly | N/A | Indirect access to Azure Database for PostgreSQL Flexible Server via BCKND REST APIs |

WEBPRT itself does **not** own or connect to any database; all authoritative data flows through Backend APIs. The platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) but provides no Disaster Recovery capability.

### 7. DevOps Tooling & Process  

| Phase | Tool | Key Configuration |
|-------|------|-------------------|
| Source Control | GitHub (`WEBPRT` repo) | Conventional commits, branch protection |
| Package Mgmt | PNPM 8 | Monorepo workspace; lock-file committed |
| CI | GitHub Actions (ubuntu-latest) | Jobs: *lint → test → build → docker-push → SBOM* |
| Static Analysis | ESLint 8.45.0 + Prettier 3.0.1 | `airbnb-typescript` preset |
| SAST | GitHub CodeQL (JavaScript/TypeScript) | Fails pipeline on high severity |
| Container Build | `docker buildx` | Multi-platform (amd64 + arm64) |
| Deployment | Azure CLI `az webapp config container set` | In-place deployment within 5-minute maintenance window |
| Dependency Mgmt | Dependabot | Weekly PRs for npm & docker base images |
| SBOM | CycloneDX | Generated per image, uploaded to GitHub release assets |
| Secrets Management | Encrypted App-Service settings | No Azure Key Vault; manual rotation during maintenance |
| Testing Framework | Vitest 0.33.0 | Unit & integration tests |
All pipeline definitions reside in `.github/workflows/` within the WEBPRT repository.

### 8. Third-Party & External Services  

| Service | Purpose | Integration Mode | Availability Notes |
|---------|---------|------------------|--------------------|
| MSG91 | SMS & Email OTP delivery (Admin/KAM login) | Consumed indirectly via Backend API | No direct browser call |
| Azure Blob | File storage (CSV, images) | Browser-side SAS PUT/GET (HTTPS) via backend-issued SAS URLs | Write-only, 10-min expiry SAS |
| Azure Application Insights | Front-end telemetry | JS SDK with `connectionString` env-var | Sample rate 100 % in prod |

No push notification services (FCM/APNS) are integrated; all notifications are via SMS and Email only.

No other third-party SDKs are embedded in the WEBPRT bundle.

### 9. Versioning & Upgrade Policy (Component)  

1. Runtime images (`nginx:1.25-alpine`) are upgraded **quarterly** unless critical CVEs demand earlier action.  
2. Build-time Node 20 stream follows the L1 "N-1 LTS" rule; minor/patch bumps occur via Dependabot PRs each sprint.  
3. JavaScript/TypeScript dependencies use exact version locks; all updates require explicit Dependabot PR approval after CI validation.
4. Any cross-stream upgrade (Nuxt 3 → 4, Node 20 → 22) requires Solution Architect approval and an updated L3-TSD revision.

### 10. Security & Compliance Tooling (Component Scope)  

* CSP header emitted by Nginx (`default-src 'self'; img-src 'self' https:; style-src 'self' 'unsafe-inline'; connect-src 'self' https://api.promopartner.xxx`) – **report-only mode** for initial release to identify required exceptions before enforcement.
* `helmet-csp-report-only` Nuxt module wired at build time to ease promotion to blocking mode later.  
* SBOM published per release; licence scanning enforces MIT/Apache-2.0 whitelist.  
* Dependabot + CodeQL ensure CVE window ≤ 7 days for high severity issues.  
* JWT tokens use RS256 encryption and are stored in HTTP-only secure cookies with SameSite=Strict flag
* OTP lockout logic enforced by backend (5 failed attempts within 10 minutes triggers 15-minute lockout)

---

### 11. Databases  

WEBPRT has no direct database connections. All data access occurs indirectly through BCKND REST APIs, which interact with:

| Database | Version | Purpose | Access Pattern |
|----------|---------|---------|----------------|
| Azure PostgreSQL Flexible Server | 16 | Primary data store | Via BCKND APIs only |

Key considerations:

- OTP codes stored as plaintext without encryption in PostgreSQL (security provided by database-level encryption)
- Relies solely on Azure's built-in automated daily backups (7-day PITR)
- No Disaster Recovery capability
- WEBPRT never establishes direct database connections

### 12. Summary  

WEBPRT's technology stack is intentionally **lean and browser-centric**:

* Build with Node 20 LTS → ship static assets → run behind minimal Nginx in Azure App Service.  
* Modern, mainstream front-end libraries (Nuxt 3.5.0, Vue 3.3.4, Pinia 2.1.6, Tailwind via @nuxtjs/tailwindcss 6.8.0) locked to specific versions  
* Comprehensive UI component suite with @nuxt/ui, @headlessui/vue for accessibility, and specialized inputs
* @tanstack/vue-query for efficient server state management with proper cookie-based auth configuration
* CI/CD and security tooling mirror corporate standards while keeping image size and runtime attack surface small.  
* Authentication via HTTP-only cookies with RS256-signed JWTs, no client-side token storage
* KAM users have read-only access with no permissions for PUT, PATCH, or DELETE endpoints

This selection satisfies the current scale targets (≤ 1 000 concurrent admin sessions) and leaves clear, low-risk upgrade paths as future demand grows.

**Critical Integration Notes:**

1. Configure @tanstack/vue-query to include credentials in all requests for cookie-based authentication
2. Use date-fns for consistent date handling across the application instead of native Intl