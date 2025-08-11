## L3-TSD-BCKND: Technology Stack Documentation (TSD) for BCKND Document  

### 1. Purpose & Scope  
This document enumerates every technology, framework, runtime, cloud service and DevOps tool that the Backend component (**BCKND**) of PromoPartner depends on.  It is the single source of truth for developers working inside the `promopartner-backend` GitHub repository and overrides any conflicting system-wide guidance.  Choices are sized intentionally for ~5 000 daily users and ≤100 k API transactions per day; no "future hyper-scale" tooling has been introduced.  

---

### 2. Programming Languages & Runtimes  

| Layer | Language / Runtime | Locked Version | Notes |
|-------|--------------------|----------------|-------|
| Main code-base | C# 12 on .NET 8  | 8.0.x LTS | Standard build for compatibility with all required libraries. |
| Scripting / CI | Bash | GNU bash 5.x | Default on GitHub Ubuntu runners. |

---

### 3. Container Image & Operating System  

| Purpose | Image Tag | Distro | Extra Packages | Rationale |
|---------|-----------|--------|----------------|-----------|
| Build stage | `mcr.microsoft.com/dotnet/sdk:8.0-alpine` | Alpine 3.19 | none | Small footprint; musl libc is OK for pure-managed code. |
| Run stage  | `mcr.microsoft.com/dotnet/aspnet:8.0-alpine` | Alpine 3.19 | none | Matches build stage; Kestrel listens on port 80. |

No reverse-proxy (Nginx/YARP) is baked into the image; App Service provides connection-draining.  

---

### 4. Core Frameworks & Libraries  

| Category | Library | Pinned Version* | Key Use |
|----------|---------|-----------------|---------|
| Web API | ASP.NET Core Minimal APIs | 8.0.x | Lightweight routing & filters. |
| Data Access | Entity Framework Core | 8.0.x | Code-first with Npgsql provider. |
| PostgreSQL Driver | Npgsql | 8.0.* | Enables COPY & advisory locks. |
| CQRS / Mediator | MediatR | 12.1.* | Vertical-slice organisation. |
| Validation | FluentValidation | 11.* | DTO validation. |
| Auth Tokens | Microsoft.IdentityModel.Tokens.Jwt | 7.1.* | RS256 JWT issuance and validation for custom OTP auth. |
| Caching | Microsoft.Extensions.Caching.Memory | 8.0.* | In-memory cache for promotion rule expressions. |
| Azure SDK - Blob | Azure.Storage.Blobs | 12.19.* | Blob SDK for CSV and audit storage. |
| Azure SDK - Queue | Azure.Storage.Queues | 12.17.* | Queue SDK for OTP messaging. |
| Logging Sink | Microsoft.ApplicationInsights.AspNetCore | 2.22.* | Enables Serilog output to Azure Application Insights. |
| Background Cron | Cronos | 1.1.* | Monthly partition creation & promo auto-archive. |
| CSV | CsvHelper | 30.* | Bulk export/report generation. |
| Resilience | Polly | 8.* | Retry with jitter for MSG91 & Blob. |
| Logging | Serilog | 2.12.* | Structured JSON sink to App Insights. |
| Mapping | AutoMapper | 12.* | Slice-local mapping profiles. |
| Architecture tests | NetArchTest | 1.* | CI layer-violation guard. |
| Unit testing | xUnit | 2.4.* | Core test framework. |
| Integration testing | Testcontainers-for-.NET | 3.* | Spins up ephemeral PostgreSQL. |
| Integration testing | Respawn | 6.* | Database state reset for integration tests with PostgreSQL. |

\* All versions are **locked to the latest stable patch** in the indicated minor stream; upgrades occur quarterly via Dependabot PR + CI run.  

---

### 5. Data Stores & Messaging Dependencies  

| Purpose | Service | Version / Tier | Access from BCKND |
|---------|---------|----------------|-------------------|
| Primary relational DB | Azure Database for PostgreSQL Flexible Server | v16, Burstable B2ms | `Npgsql` via TLS 1.2 |
| Object storage | Azure Blob Storage | Hot & Cool tiers | `Azure.Storage.Blobs` SDK |
| Message queue | Azure Storage Queue | Platform | `Azure.Storage.Queues` SDK (OTP dispatch) |

The platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) but provides no Disaster Recovery capability.

Local development uses Dockerised PostgreSQL 16 and Azurite for blob/queue emulation (compose file lives in `/dev/docker-compose.yml`).  

---

### 6. Hosting & Runtime Environment  

| Aspect | Choice / Setting |
|--------|------------------|
| Compute | Azure App Service (Linux, container) – Plan `P1v2` |
| Scaling model | **Single instance**, vertical scale-up only |
| Environment variables | All secrets & toggles stored in encrypted App-Service settings (no Azure Key Vault) |
| Network | Public endpoint, TLS 1.2 minimum  |

---

### 7. DevOps Tooling (Component Scope)  

| Stage | Tooling | Key Configuration |
|-------|---------|-------------------|
| Source control | GitHub (`promopartner-backend`) | Branch protection, trunk-based |
| CI | GitHub Actions | `dotnet test` → `dotnet publish -c Release`  → `docker build` |
| Image registry | Azure Container Registry (Basic) | Image tag = `git-sha` |
| CD | GitHub Actions + `az webapp config container set` | **In-place deploy** to Production  environment |
| Static code analysis | GitHub CodeQL | Runs on every PR |
| SBOM & licence scan | CycloneDX plugin | Uploaded as build artifact |
| Test reports | xUnit XML → GitHub Summary | Fails pipeline if coverage < 80 % |

---

### 8. Third-Party Services Integrated  

| Service | Function | SDK / Method | Availability Strategy |
|---------|----------|--------------|-----------------------|
| MSG91 SMS | SMS dispatch for OTP/notifications | `IHttpClientFactory` + `System.Text.Json` (no external SDK) | Polly (3× retry, exp. back-off) + poison queue |
| MSG91 Email | Email dispatch for OTP/notifications | `IHttpClientFactory` + `System.Text.Json` (REST API) | Polly (3× retry, exp. back-off) + poison queue |

No other external APIs are consumed by BCKND in phase 1. No push notification services (FCM/APNS) are integrated.

---

### 9. Security & Compliance Tooling (Code-Side)  

• Transport security enforced via App Service TLS 1.2 and `HSTS` header (max-age 180 days).  
• Secrets resolved from environment variables at startup using default `Host.CreateDefaultBuilder()` providers (no Azure Key Vault integration).  
• Serilog GDPR filter redacts the `code` field of OTP payloads in logs, though OTP codes are stored as plaintext in PostgreSQL (database-level encryption only).  
• SAST (CodeQL) and SBOM generation gates every merge to `main`.  
• JWT tokens use RS256 signing with keys stored in encrypted App-Service settings.

---
