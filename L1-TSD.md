
## L1-TSD: Technology Stack Documentation (TSD) (System-Wide) Document

### 1. Purpose & Scope  
This document enumerates all technologies, frameworks, runtimes, cloud services, DevOps tooling, and third-party providers selected for PromoPartner at the solution-wide level. It captures version pins, hosting models, and operational conventions so that every component team works against the same technical baseline. Detailed, component-specific variances reside in the respective L3-TSD artefacts.

---

### 2. Runtime & Language Versions  
The project follows an "N-1 LTS" policy to balance ecosystem stability with vendor support. Versions are locked for the first 12-month operating period and may be upgraded only through the formal change-control process.

| Domain | Language / Runtime | Locked Version Stream | Rationale |
|--------|--------------------|-----------------------|-----------|
| Backend | C# 12 on .NET 8 SDK/runtime | 8.0.x LTS | Latest LTS with 36-month support; required for native-AOT & minimal APIs |
| Web-Portal build | Node.js | 20.x LTS | Node 18 enters maintenance/EOL inside the lock window; Node 20 keeps us within active support and aligns with Nuxt 3. |
| Web-Portal front-end | TypeScript | 5.x | Ships with Nuxt 3 scaffolding; semantic version kept at minor-stream latest |
| Mobile | Dart | 3.2.x | Flutter 3 long-term stable line; "N-1" to current 3.3.x |
| Database | PostgreSQL | 16 (Azure Flexible Server) | Fits N-1 policy; avoids breaking changes introduced in v15 |
| Scripting / CI | Bash | GNU bash 5.x | Default on GitHub-hosted Ubuntu runners |

---

### 3. Framework & Library Baseline
This section lists only cross-cutting libraries that are shared by more than one component. Detailed, component-specific stacks are documented in the corresponding L3-TSDs.

#### Cross-Cutting Libraries
* Serilog 3 – structured logging sink used by Backend services
* FluentValidation.AspNetCore 11.3.0 – Request validation
* Polly 8 – retry & circuit-breaker for outbound HTTP calls

#### Backend Stack (.NET 8)
* Entity Framework Core 8.0.0 – ORM for database operations
* Npgsql.EntityFrameworkCore.PostgreSQL 8.0.0 – PostgreSQL provider
* Dapper 2.1.15 – Micro-ORM for performance-critical queries
* Swashbuckle.AspNetCore 6.5.0 – OpenAPI/Swagger documentation
* AutoMapper 12.0.1 – Object mapping
* FluentResults 3.15.2 – Result pattern implementation
* Microsoft.AspNetCore.Authentication.JwtBearer 8.0.0 – JWT token handling

#### Testing Frameworks
* xUnit 2.5.0 – Testing framework
* Moq 4.20.69 – Mocking library
* FluentAssertions 6.12.0 – Assertions

---

### 4. Data Stores  
PromoPartner is intentionally simple: a single relational store backed by object storage for large blobs and a lightweight queue for async tasks.

| Purpose | Service | Version / Tier | Notes |
|---------|---------|----------------|-------|
| Primary relational DB | Azure Database for PostgreSQL Flexible Server | PostgreSQL 16, Burstable B2ms | Time-range partitioning on `transactions`, `otp` |
| Object storage | Azure Blob Storage | Hot tier for frequently accessed assets, Cool tier for aged objects, immutable audit container | `assets` (CSV, banners) container and immutable `audit` container |
| Message queue | Azure Storage Queue | N/A (platform) | JSON payload, AES-256 encrypted at rest |

---

### 5. Hosting & Infrastructure Services  
| Layer | Azure Service | Configuration | Comment |
|-------|---------------|---------------|---------|
| Backend API | Azure App Service (Linux container) | Custom Docker image pulled from Azure Container Registry; P1v2 plan | Aligns with containerised workflow in HLD/KD; enables multi-process web + worker model. |
| Web Portal | Azure App Service (Linux container) | Custom Docker image (Node 20 base) pulled from ACR; P1v2 plan | Ensures parity with local dev and reproducible builds. |
| Container Registry | Azure Container Registry (Basic) | Mandatory; stores signed images for Backend & Portal | Central provenance & vulnerability scanning; no longer optional. |
| Static DNS / TLS | Azure App Service Managed Certificates | TLS 1.2 | Separate hostnames for API and Portal |
| Network | Public endpoints only, DDoS Basic | — | Office IP allow-list applied to Portal endpoint |

---

### 6. DevOps Toolchain & Process  
| Stage | Tool / Service | Key Steps |
|-------|----------------|-----------|
| Source Control | GitHub (3 repos – BCKND, WEBPRT, APP) | Trunk-based flow, branch protection |
| CI – Backend & Portal | GitHub Actions (ubuntu-latest runners) | `docker build` → `docker push` to ACR; image tags = `git-sha` |
| Deploy – Backend & Portal | Azure CLI (`az webapp config container set`) | In-place deployment within 5-minute window |
| Secrets | Encrypted App-Service settings | RSA JWT key, MSG91 key stored in App-Service settings; manual rotation during maintenance. |
| Mobile Build & Release | GitHub Actions macOS runners with Fastlane | Automated build, code-sign, and upload to Apple/Google stores; signing keys stored in encrypted GitHub secrets. |
| Monitoring | Azure Application Insights | Serilog sink + auto-collected APM metrics |
| Alerting | Azure Alerts (metric, log query) | Thresholds defined in L2-Operations doc |
| Audit & Compliance | Immutable Blob container | Event JSON appended in near-real-time |
| API Documentation | Swashbuckle.AspNetCore | OpenAPI documentation at /swagger | Development only |

---

### 7. Third-Party & External Services  
| Service | Purpose | Integration Mode | Availability Strategy |
|---------|---------|------------------|-----------------------|
| MSG91 | SMS & e-mail (OTP, notifications) | HTTPS REST (both SMS & Email) | Single-vendor risk accepted per client directive |
| Apple App Store / Google Play | Mobile distribution | Fastlane automated upload | No in-app purchase; APK/IPA only |
| GOV GST Validation API | Not in scope phase 1 | — | Future integration placeholder |

---

### 8. Security & Compliance Tooling – Context  
Security controls leverage native Azure features and automated checks embedded in the CI pipeline.

* Transport security: TLS 1.2 on all public endpoints.  
* Data-at-rest: Azure-managed AES-256 for PostgreSQL, Blob, Queue.  
* JWT signing key & MSG91 API key stored in encrypted App-Service settings; manual rotation during maintenance.  
* Static Application Security Testing (SAST): GitHub Advanced Security CodeQL workflows enabled for all repos.  
* Open-Source licence & SBOM: GitHub licence scanning and CycloneDX SBOM generation executed on every PR.  
* Audit events written immutably to Blob; retention = infinite.  

---