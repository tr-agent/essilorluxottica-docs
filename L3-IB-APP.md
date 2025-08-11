## L3-IB-APP: Component Implementation Backlog for APP  

### 1. Introduction  
This backlog translates the APP-specific functional requirements (L3-FRS-APP), non-functional targets (L3-NFRS-APP), low-level design (L3-LLD-APP) and key decisions (L3-KD-APP) into granular, assignable work items.  Each item is sized for a single developer (or pair) and delivers a testable increment that moves the mobile application toward a shippable v1 suitable for the explicitly stated scale (≈5 000 users / ≤1 000 concurrent sessions). No tasks exceed the scope or complexity justified by that load.

### 2. Legend  
• **ID** – Stable reference `APP-IB-NNN`.  
• **P** – Priority: 1 = Must-have for v1 GA, 2 = Should, 3 = Could (stretch).  
• **SP** – Relative story-point estimate (Fibonacci scale).  
• **Dep.** – Direct predecessor backlog IDs or external blockers.  
• **Trace** – Requirement and design references.  

### 3. Backlog Overview by Epic  
(The recommended development order follows the epic numbering.)

| # | Epic | Goal | P | Items |
|---|------|------|---|-------|
| 0 | Foundation & CI | Scaffold, linting, CI pipeline, env handling | 1 | 001-009 |
| 1 | Core Runtime Services | HTTP client, secure storage, error mapping, version banner | 1 | 010-019 |
| 2 | Auth & Session | OTP login, refresh, single-device rule | 1 | 020-032 |
| 3 | Store Context Switch | Child-store selector & header injection | 1 | 033-037 |
| 4 | Catalogue & Search | Paginated product list with search | 1 | 038-045 |
| 5 | Cart & Promotion | Cart UI, promo evaluate, auto-apply | 1 | 046-054 |
| 6 | Transaction Submit | Idempotent POST, success screen | 1 | 055-060 |
| 7 | Reconciliation | Invoice form, image compress & SAS upload | 1 | 061-069 |
| 8 | KPI Dashboard | KPI tiles & pull-to-refresh | 2 | 070-074 |
| 9 | Role-Based UI Gating | Feature toggles by JWT role | 1 | 075-079 |
| 10 | Error & Outage UX | Canonical error mapper, MSG91 banner | 1 | 080-085 |
| 11 | I18n & Accessibility | Font scale, semantics, string files | 2 | 086-090 |
| 12 | Release Ops | App-store meta, size checks, signed builds | 1 | 091-096 |
| 99 | Cross-Cutting NFR hardening | Perf tuning, security tests | 2 | 097-100 |

---

### 4. Detailed Backlog  

#### Epic 0 – Foundation & CI  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-001 | Create Flutter 3.2 project with package structure (`core`, `features/*`, `widgets`) | Technical | 1 | 3 | – | L3-LLD §2.2 | Folder layout matches table; project builds in **dev** & **prod** flavours. |
| APP-IB-002 | Add `flutter_bloc`, `dio`, `flutter_secure_storage` to `pubspec` | Technical | 1 | 1 | 001 | L3-KD C#5, L3-LLD §2.1 | `flutter pub get` succeeds; versions locked in `pubspec.lock`. |
| APP-IB-003 | Configure analysis rules (`pedantic` + custom) and pre-commit Git hooks | Technical | 1 | 2 | 001 | L3-NFRS §8 | CI fails on any analyzer error. |
| APP-IB-004 | GitHub Actions workflow: `flutter test`, `build apk` (debug), cache pub | Technical | 1 | 2 | 003 | L3-NFRS §8, L3-LLD §7 | Pull-request builds pass; artifact attached. |
| APP-IB-005 | Enable size-check step (warn if > 75 MB AAB) | Technical | 1 | 1 | 004 | PERF-08 | Job prints size; fails if > 80 MB. |
| APP-IB-006 | Add Fastlane lanes for Play & App Store beta tracks | Technical | 1 | 3 | 004 | L3-LLD §7 | `fastlane android beta` produces signed AAB uploaded to Test track. |
| APP-IB-007 | Create `.env` handling (`dart-define`) for API base URL & flavour | Technical | 1 | 2 | 001 | L3-LLD §7 | Switching flavour changes base URL at runtime. |
| APP-IB-008 | Establish `PlatformTelemetry` stub interface | Technical | 2 | 1 | 001 | Clarif #4 | Interface compiles; no runtime SDK. |
| APP-IB-009 | README with project conventions & bootstrap steps | Doc | 2 | 1 | 001 | L3-LLD §8 | New dev can build app < 15 min following README. |

---

#### Epic 1 – Core Runtime Services  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-010 | Implement `DioClient` with base options, timeouts (10 / 30 s) | Technical | 1 | 3 | 001 | L3-LLD §2.4 | All requests include `Accept: application/json`. |
| APP-IB-011 | AuthInterceptor: Bearer header, queue pause, refresh retry | Feature | 1 | 5 | 010, 025 | APP-FR-05 | 401 triggers single refresh; original call replayed once. |
| APP-IB-012 | ErrorMapperInterceptor mapping `error.code` list | Feature | 1 | 3 | 010 | APP-FR-30, Clarif #2 | Codes in backend JSON dictionary map to enums; unknown → generic. |
| APP-IB-013 | Secure storage wrapper (`TokenStore`) | Technical | 1 | 2 | 002 | Security-1 | `setTokens`, `getTokens`, `clear()` unit-tested. |
| APP-IB-014 | IdempotencyKey service (RAM only, per cart) | Technical | 1 | 2 | 012 | C#4 | Generates v4 UUID and exposes getter/setter. |
| APP-IB-015 | Version check interceptor adds `X-App-Version` & handles 426 | Feature | 1 | 2 | 010 | APP-FR-28/29 | On 426 shows blocking dialog with deep-link stub. |
| APP-IB-016 | NetworkReachability service + global offline banner | Feature | 1 | 3 | 001 | APP-FR-02 | Banner appears ≤ 200 ms after loss; Retry button fires callback. |
| APP-IB-017 | Global theming & colour palette | UX | 1 | 2 | 001 | L3-LLD §3.1 | Primary `#004E92`, accent `#FF8200`; WCAG AA contrast verified. |
| APP-IB-018 | Centralised routing with `go_router`: Splash, Login, Tabs | Technical | 1 | 3 | 001 | L3-LLD §2.3 | Deep-links to `promopartner://tx/{id}` resolve. |
| APP-IB-019 | Unit test harness & mocking utilities (`mocktail`) | Technical | 1 | 1 | 004 | L3-LLD §2.8 | Base test runs with ≥ 90 % provider coverage target stub. |

---

#### Epic 2 – Authentication & Session  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-020 | OTP Request screen (phone/email input, validation) | Feature | 1 | 3 | 018 | APP-FR-03 | VR-01 passed; SMS or email inferred by role flag. |
| APP-IB-021 | Connect `/auth/otp/request` API & progress state | Feature | 1 | 2 | 020, 010 | APP-FR-03 | Returns masked destination; handles 429 error toast. |
| APP-IB-022 | OTP Verify screen & `/verify` integration | Feature | 1 | 3 | 020, 013 | APP-FR-04 | Success stores tokens, navigates to Splash. |
| APP-IB-023 | Splash screen with silent `/refresh` attempt | Feature | 1 | 2 | 011 | APP-FR-05 | If refresh fails → Login; else → Home. |
| APP-IB-024 | Single-device enforcement (purge on 401/403 UNAUTHENTICATED) | Feature | 1 | 2 | 011 | APP-FR-06 | Forced logout banner shown; tokens cleared. |
| APP-IB-025 | Timer-based refresh 5 min pre-expiry | Technical | 1 | 2 | 023 | APP-FR-05 | Unit test: token expiring in 6 min triggers refresh. |
| APP-IB-026 | Root/JB device detector sets `X-Device-Integrity` header | Tech/SEC | 2 | 2 | 010 | L3-NFRS §5 | Compiles on iOS & Android; root=warn only. |
| APP-IB-027 | Auth bloc & repository wiring | Technical | 1 | 2 | 022 | C#5 | State transitions: initial → requesting → codeSent → verified. |
| APP-IB-028 | Unit tests for Auth flow (request, verify, refresh) | Test | 1 | 2 | 027, 019 | L3-LLD §2.8 | 90 % line-coverage on bloc & service. |
| APP-IB-029 | UX copy & error strings for Auth | UX | 1 | 1 | 022 | APP-FR-30 | Localised in `strings.dart`. |
| APP-IB-030 | Logout action & cleanup | Feature | 1 | 1 | 023 | APP-FR-05 | Clears all RAM caches, navigates to Login. |
| APP-IB-031 | JWT role extraction utility | Technical | 1 | 1 | 013 | APP-FR-26 | Returns enum {S,P,K,A}. |
| APP-IB-032 | E2E test: happy-path login on emulator | Test | 1 | 3 | 028 | L3-LLD §2.8 | `flutter drive` completes in CI. |

---

#### Epic 3 – Store Context Switching  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-033 | `/stores?parentId` API client method | Technical | 1 | 2 | 010 | APP-FR-08 | Returns list sorted alphabetically. |
| APP-IB-034 | Store selector dropdown in Profile tab | Feature | 1 | 3 | 033, 031 | APP-FR-08 | Persists choice in memory; selected label visible in AppBar. |
| APP-IB-035 | Inject `X-Store-Id` header when context ≠ parent | Technical | 1 | 1 | 010, 033 | APP-FR-07 | Verified in integration test. |
| APP-IB-036 | Clear store context on logout | Maintenance | 1 | 1 | 030 | APP-FR-08 | No header sent post-logout. |
| APP-IB-037 | Unit tests for store switch provider | Test | 1 | 1 | 034, 019 | VR-none | Coverage ≥ 90 %. |

---

#### Epic 4 – Product Catalogue & Search  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-038 | `ProductService` with paginated `GET /products` | Technical | 1 | 3 | 010 | APP-FR-11 | Supports `page`, `pageSize`, `search` params. |
| APP-IB-039 | ProductList bloc with infinite scroll & debounce 300 ms | Feature | 1 | 5 | 038, 016 | APP-FR-11/13 | Cancels in-flight request on term change. |
| APP-IB-040 | Product list UI (grid phone, 2-pane tablet) | UX | 1 | 5 | 039 | L3-LLD §3.2 | Tablet ≥ 600 dp shows list + detail pane. |
| APP-IB-041 | Search field with clear & placeholder | UX | 1 | 2 | 040 | APP-FR-12 | Typing shows shimmer skeleton; VR-05 enforced (no barcode). |
| APP-IB-042 | Empty & error states (no results, 5xx) | UX | 1 | 1 | 040 | APP-FR-30 | Friendly copy displayed. |
| APP-IB-043 | Unit tests for debounce & pagination helper | Test | 1 | 2 | 039, 019 | L3-LLD §2.5 | Emits `EndOfList` when `nextToken` null. |
| APP-IB-044 | Performance check: frame build ≤ 150 ms on Moto G | Perf | 1 | 1 | 040 | PERF-04 | Profile run passes budget. |
| APP-IB-045 | Accessibility audit: product tiles | A11y | 2 | 1 | 040 | APP-FR-33 | Tiles expose semantic label “<SKU> <Price> button”. |

---

#### Epic 5 – Cart & Promotion Evaluation  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-046 | CartProvider & bloc (add, remove, qty change) | Feature | 1 | 5 | 039 | APP-FR-14 | Unlimited lines; negative qty prevented. |
| APP-IB-047 | Cart screen UI with subtotal & item list | UX | 1 | 3 | 046 | L3-LLD §2.6 | Updates reactively. |
| APP-IB-048 | `/promotions/evaluate` API client | Technical | 1 | 2 | 010 | APP-FR-15 | Accepts `{cart,storeId}`. |
| APP-IB-049 | Promotion list bottom-sheet & auto-apply | Feature | 1 | 3 | 048 | APP-FR-15/16 | If user taps none → first promo auto-applied. |
| APP-IB-050 | Disabled Checkout when no eligible promotions | Feature | 1 | 1 | 048 | L3-LLD §2.6 | UX matches flowchart. |
| APP-IB-051 | Customer OTP dialog flow (request/verify) | Feature | 1 | 3 | 020, 010 | APP-FR-09/10 | Caches `eligibilityToken` per cart in RAM. |
| APP-IB-052 | Unit tests: promotion evaluate bloc | Test | 1 | 2 | 048, 019 | PERF-05 | Mock backend returns ≤ 800 ms. |
| APP-IB-053 | UI performance: cart screen first paint ≤ 2 s | Perf | 1 | 1 | 047 | PERF-03 | Meets target on reference devices. |
| APP-IB-054 | A11y: cart controls semantic labels | A11y | 2 | 1 | 047 | APP-FR-33 | Passes TalkBack focus order. |

---

#### Epic 6 – Transaction Submission  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-055 | `/transactions` POST with `Idempotency-Key` header | Technical | 1 | 3 | 014, 010, 046 | APP-FR-17 | 202 handling returns key header value. |
| APP-IB-056 | Retry loop until 201/4xx (back-off 2,4,8 s) | Feature | 1 | 3 | 055 | APP-FR-18 | Drops after terminal status; duplicate submits prevented. |
| APP-IB-057 | Success screen with `transactionId` & share stub | UX | 1 | 1 | 056 | APP-FR-19 | Navigate stack resets cart. |
| APP-IB-058 | Cancel/abort transaction confirmation dialog | UX | 2 | 1 | 046 | – | Aborting clears cart. |
| APP-IB-059 | Integration test: flaky-network retry | Test | 1 | 2 | 056, 019 | REL-05 | Simulated 50 % failure still yields 1 tx. |
| APP-IB-060 | Performance: idempotent submit RTT ≤ 4 s P95 | Perf | 1 | 1 | 056 | PERF-05 | JMeter script validates. |

---

#### Epic 7 – Reconciliation  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-061 | Reconciliation form (invoiceNo, pidNo, attach) | Feature | 1 | 4 | 057 | APP-FR-20 | Validation VR-02 applied. |
| APP-IB-062 | Camera & gallery pick; image compress ≤ 2 MB | Feature | 1 | 3 | 061 | APP-FR-21 | Image longest side ≤ 1280 px. |
| APP-IB-063 | `/csv-jobs/invoice/presign` POST & SAS PUT helper | Technical | 1 | 3 | 010, 062 | APP-FR-22 | Retries once on 403 “SignatureExpired”. |
| APP-IB-064 | `/transactions/{id}/reconcile` PATCH integration | Technical | 1 | 2 | 063 | APP-FR-22 | Status NEW/UNDER_REVIEW only. |
| APP-IB-065 | Read-only toggle when tx status ≥ VERIFIED | Feature | 1 | 1 | 064 | APP-FR-23 | UI switches automatically after refresh. |
| APP-IB-066 | Attachment preview & replace capability | UX | 2 | 2 | 062 | – | Tap thumbnail → full preview. |
| APP-IB-067 | Unit tests: image size enforcement | Test | 1 | 1 | 062 | VR-03 | Files > 2 MB rejected. |
| APP-IB-068 | E2E test: full reconciliation happy path | Test | 1 | 3 | 065 | L3-LLD §2.6 | Runs in CI with mocked backend. |
| APP-IB-069 | Perf: upload 2 MB JPEG ≤ 4 s P95 | Perf | 1 | 1 | 063 | PERF-06 | Meets target on 4 G network sim. |

---

#### Epic 8 – KPI Dashboard  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-070 | Implement KPI service hitting separate endpoints | Technical | 2 | 3 | 010 | Clarif #3, APP-FR-24/25 | Endpoints configurable by `range` param. |
| APP-IB-071 | Home tab KPI tiles grid & pull-to-refresh | Feature | 2 | 3 | 070 | APP-FR-24 | Shows shimmer while loading. |
| APP-IB-072 | Cache KPIs in RAM, expire after 30 min bg | Tech | 2 | 1 | 070 | Data §4 | Cache cleared on resume > 30 min. |
| APP-IB-073 | Unit tests: KPI provider & range filters | Test | 2 | 1 | 070, 019 | – | 90 % coverage. |
| APP-IB-074 | Tablet layout: two-column KPI grid | UX | 3 | 1 | 071 | L3-LLD §3.2 | Verified on 10″ simulator. |

---

#### Epic 9 – Role-Based UI Gating  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-075 | RoleGuard widget for route access | Technical | 1 | 2 | 031, 018 | APP-FR-26 | Routes blocked for disallowed roles. |
| APP-IB-076 | Hide parent-store features when role ≠ P | Feature | 1 | 1 | 075 | APP-FR-27 | Child-store selector not visible for S/K/A. |
| APP-IB-077 | Hide admin-only screens on mobile (none in v1) | Maintenance | 2 | 1 | 075 | APP-FR-26 | Placeholder guard passes tests. |
| APP-IB-078 | Smoke tests per role (S, P, K, A) | Test | 1 | 3 | 076 | – | Each role sees correct tabs. |
| APP-IB-079 | UI copy explaining hidden features | UX | 2 | 1 | 076 | NFR-Usability | Tooltip “Feature unavailable for your role”. |

---

#### Epic 10 – Error & Outage UX  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-080 | Global error toast & inline form error mapper | Feature | 1 | 2 | 012 | APP-FR-30 | Inline field highlights for 422 validation. |
| APP-IB-081 | MSG91 outage banner polling `/system/metrics` every 60 s | Feature | 1 | 3 | 010 | Clarif #5, APP-FR-31 | If value > 0 show red banner; login disabled. |
| APP-IB-082 | Disable OTP flows when outage active | Feature | 1 | 2 | 081, 020 | APP-FR-31 | Buttons grayed out; toast on tap. |
| APP-IB-083 | Unit tests: error mapper table complete | Test | 1 | 1 | 080, 019 | Clarif #2 | All backend codes map. |
| APP-IB-084 | Integration test: outage banner path | Test | 1 | 2 | 082 | – | Banner appears within 60 s in mock. |
| APP-IB-085 | A11y: banner ARIA role “alert” | A11y | 2 | 1 | 081 | APP-FR-33 | Screen readers announce banner. |

---

#### Epic 11 – Internationalisation & Accessibility  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-086 | Centralise strings in `strings.dart` with `Intl.message` | Tech | 2 | 2 | 001 | APP-FR-32 | Build succeeds with `flutter gen-l10n`. |
| APP-IB-087 | Font scaling: verify all screens at 1.3x & 1.6x | UX | 2 | 2 | 047, 071 | APP-FR-33 | No text clipped or overlapped. |
| APP-IB-088 | Semantic labels on tappable widgets (global sweep) | A11y | 2 | 2 | 045, 054, 065 | APP-FR-33 | Accessibility scanner passes. |
| APP-IB-089 | High-contrast mode check (OS setting) | UX | 3 | 1 | 017 | NFR-Usability | Colours invert correctly. |
| APP-IB-090 | Accessibility test automation via `flutter_a11y` | Test | 3 | 2 | 088 | – | CI step fails on missing labels. |

---

#### Epic 12 – Release Ops  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-091 | Android keystore & signing config in CI | Ops | 1 | 2 | 006 | L3-LLD §7 | Play beta build signed. |
| APP-IB-092 | iOS provisioning profiles & code-sign | Ops | 1 | 2 | 006 | – | TestFlight upload succeeds. |
| APP-IB-093 | Obfuscation & split-debug-info flags in release | SEC | 1 | 1 | 004 | L3-NFRS §5 | Size reduction verified. |
| APP-IB-094 | Store-listing metadata generator script | Ops | 2 | 1 | 006 | – | Version & changelog auto-pushed. |
| APP-IB-095 | Privacy manifest file lists OTP, phone number permissions | Doc | 1 | 1 | 092 | L3-NFRS §9 | App Store Connect privacy check passes. |
| APP-IB-096 | Post-release crash log collector hook | Ops | 3 | 2 | 008 | Clarif #4 | Placeholder sends no PII. |

---

#### Epic 99 – Cross-Cutting NFR Hardening  

| ID | Title | Type | P | SP | Dep. | Trace | Acceptance Criteria |
|----|-------|------|---|----|------|-------|---------------------|
| APP-IB-097 | Performance profiling & Jank reducer sweep | Perf | 2 | 3 | 044, 053 | PERF-01-04 | No > 150 ms frame on test path. |
| APP-IB-098 | Security static-analysis (`flutter_secure_storage`, secrets) | SEC | 2 | 2 | 093 | Security-1 | OWASP Mobile Top 10 scan clean. |
| APP-IB-099 | Dependency licence audit (MIT/Apache-2 only) | Compliance | 2 | 1 | 002 | L3-NFRS §9 | Script flags any GPL. |
| APP-IB-100 | Pen-test remediation backlog | SEC | 3 | 3 | 098 | L1-HLD §8 | All high findings fixed or accepted. |

---

### 5. External & Cross-Component Dependencies  
1. **OpenAPI 3.1 spec (mobile subset)** – Blocker for items 010, 038, 048, 055, 070 (Clarification #1).  
2. **Canonical error-code JSON catalogue** – Needed for 012, 083 (Clarification #2).  
3. **KPI endpoint split** – Confirmed contract for 070-074 (Clarification #3).  
4. **System metrics endpoint** – Required for 081-084 (Clarification #5).  

---

### 6. Backlog Conformance  
All tasks:  
• Align strictly with the agreed online-only, thin-client model – no offline caching or queue beyond transient RAM.  
• Use `flutter_bloc` (v8.x) and Clean-Architecture layering per KD C#5; no Riverpod, Redux, or over-complex patterns.  
• Target performance, security and accessibility thresholds defined in L3-NFRS-APP without introducing premature scalability constructs.  
• Remain self-contained; cross-component coordination confined to documented endpoints and headers.

---

_End of Backlog_