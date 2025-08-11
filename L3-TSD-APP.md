## L3-TSD-APP: Technology Stack Documentation (TSD) for APP Document

### 1. Purpose & Scope

This document enumerates all technologies, frameworks, build-chain tools and third-party SDKs selected for the PromoPartner Mobile App (component code **APP**).  It refines the system-wide L1-TSD, locking concrete versions and configuration choices that apply **only** to the Flutter iOS / Android client.  
The goals are:

* Guarantee build reproducibility for the first 12-month operating period.  
* Give developers a single, authoritative reference for adding or upgrading dependencies.  
* Make security, supportability and store-listing constraints explicit.

---

### 2. Runtime & Language Versions

| Domain | Runtime | Locked Stream | Rationale |
|--------|---------|--------------|-----------|
| Mobile | Flutter SDK | 3.13.9 | Stable version with mature ecosystem support |
| Mobile | Dart | 3.1.5 | Shipped with Flutter 3.13.9; full null-safety |
| Android | minSdk / targetSdk | 28 (8.0) / 34 | Matches L3-LLD-APP: wide device reach (API 28/Android 8.0); target 34 to satisfy Play policy |
| iOS | Minimum OS | 13.0 | Matches L3-LLD-APP and current App Store review baseline |

Upgrade policy: patch & minor upgrades within the locked major stream may be applied after CI passes; major upgrades require full regression cycle and team consensus.

---

### 3. Core Framework & Library Baseline

Only dependencies common to **multiple features** are listed.  Feature-specific packages live inside the corresponding module folders and inherit the global version-pin strategy.

| Area | Package | Locked Version | Notes |
|------|---------|----------------|-------|
| App framework | flutter | 3.13.9 | Consumed via `flutter channel stable` |
| Internationalization | flutter_localizations | 3.13.9 | English-only for v1, framework for future expansion |
| State management | flutter_bloc | 8.1.3 | Authoritative per L3-KD-APP C#5; no Provider or Riverpod for state |
| State utilities | equatable | 2.0.5 | Value equality for BLoC states |
| Routing | go_router | 10.1.2 | Navigation routing with bottom nav support |
| Networking | dio | 5.3.2 | Interceptors implement auth & error mapping |
| Network monitoring | connectivity_plus | 4.0.2 | Detect online/offline status for UI blocking |
| JSON serialisation | json_annotation | 4.8.1 | JSON serialization annotations |
| JSON serialisation | json_serializable | 6.7.1 | Code-gen at build time |
| Persistence (secure) | flutter_secure_storage | 8.0.0 | AES-256; Android Keystore + iOS Keychain for JWT and refresh token storage |
| Local preferences | shared_preferences | 2.2.0 | Non-sensitive app preferences only |
| Image caching | cached_network_image | 3.2.3 | Banner assets; 5 MB RAM cap |
| SVG support | flutter_svg | 2.0.7 | For vector icons and graphics |
| UI responsive | flutter_screenutil | 5.8.4 | Responsive scaling utilities |
| UI layout | flutter_layout_grid | 2.0.1 | For responsive grid utilities per L3-LLD-APP §2.7 |
| Typography | google_fonts | 5.1.0 | Consistent font loading |
| Loading effects | shimmer | 3.0.0 | Skeleton loading states |
| Forms | flutter_form_builder | 9.1.0 | Form widget utilities |
| Form validation | form_builder_validators | 9.0.0 | Reusable validators |
| Media selection | image_picker | 1.0.4 | Future: profile pictures, receipts |
| Logging | logger | 2.0.2 | Debug only; stripped in release |
| Internationalisation | intl | 0.18.1 | English only for v1 |
| Testing | flutter_test | 3.13.9 | Core testing framework |
| Testing | mocktail | 1.0.0 | Mocking library |
| Testing | golden_toolkit | 0.15.0 | Golden tests |
| Testing | integration_test | 3.13.9 | E2E testing |
| Testing | mockito | 5.4.2 | Alternative mocking for complex scenarios |

Version numbers are exact pins in `pubspec.yaml`.

### 3.1 Excluded/Not Recommended Packages

| Package | Reason for Exclusion |
|---------|---------------------|
| flutter_riverpod | flutter_bloc is the standardized state management solution |
| hive | Online-only architecture; no local database needed |
| firebase_auth | OTP handled by backend via MSG91; no Firebase services |
| azure_* SDKs | Backend abstracts all Azure services; no direct integration |
| barcode_scan2 | Not required for current promotion flows |
| local_auth | Biometric auth not in phase 1 requirements |

---

### 4. Local Data & Security

| Concern | Technology / Setting | Detail |
|---------|----------------------|--------|
| Token storage | flutter_secure_storage (8.0.0) | Keys: `jwt`, `refreshToken`; passed via Authorization header; no PII on disk |
| App preferences | shared_preferences (2.2.0) | Non-sensitive settings only (theme, language preference) |
| Transient caching | Memory only | Lists & images cleared on app exit |
| Encryption-in-transit | TLS 1.2 | OS stack; no certificate pinning (L3-LLD-APP §5) |
| Databases | None | Online-only architecture; all data operations via Backend REST APIs |

No SQLite or object-box database is included; the APP is intentionally **online-only** with all state management via server APIs.

---

### 5. Networking Stack

* dio 5.3.2 client with three interceptors  
  1. Auth header + silent refresh per L3-LLD-APP §2.4.  
  2. Error mapping to domain codes.  
  3. SAS retry for Blob operations on signature expiry.
* JSON payloads generated via `json_serializable 6.7.1` with `json_annotation 4.8.1`.
* Timeout defaults: 10 s connect, 30 s read.
* Network state monitoring via `connectivity_plus 4.0.2` to show offline banner.

---

### 6. UI Toolkit & Design System

* Material 3 (Flutter built-in) with custom color-scheme:  
  * Primary `#667eea` (purple/indigo)  
  * Secondary `#27ae60` (green)  
* Typography via `google_fonts 5.1.0` for consistent font rendering.
* Responsive scaling with `flutter_screenutil 5.8.4` for device adaptation.
* Adaptive layout: `MediaQuery` break-point 600 dp switches to 2-pane tablet UI.  
* Custom components live in `/lib/widgets` and reuse responsive utilities from `flutter_layout_grid` (see section 3).
* SVG icons supported via `flutter_svg 2.0.7`.
* Loading states implemented with `shimmer 3.0.0` for skeleton screens.

---

### 7. Build & Distribution Tooling

| Stage | Tool / Service | Key Configuration |
|-------|----------------|-------------------|
| Local IDEs | VS Code 1.85+, Android Studio Giraffe | Flutter/Dart plug-ins |
| Code generation | build_runner `2.4.6` | Executed in CI |
| Static analysis | `dart analyze` + `flutter_lints` | CI blocking |
| CI / CD | GitHub Actions (macOS runner) | Jobs: `tests → build-appbundle (.aab) / build-ipa → upload-to-store` |
| Signing & Store upload | Fastlane | iOS & Android lanes; credentials in GitHub Secrets |
| Artifact naming | `promopartner-<platform>-<git-sha>.<ext>` | Reproducible build id |

---

### 8. DevOps & Quality Gates

1. Dependabot weekly PRs on all Flutter/Dart deps.  
2. SAST: GitHub CodeQL (Dart) — informational only.  
3. Unit-test coverage thresholds enforced (`flutter test --coverage ≥ 85 %` global).  
4. Golden-image diff for critical screens on every PR.  
5. Manual QA executed on device matrix derived from min OS table.
6. Mock backend server used for frontend development and testing (eliminates need for http_mock_adapter).

_No Crashlytics or analytics SDK is included in v1.0; monitoring via app-store feedback only._

---

### 9. Device & OS Support Matrix

| Platform | Min | Target / Deployment | Tested Models |
|----------|-----|---------------------|---------------|
| Android | 8.0 (API 28) | 34 (Play requirement) | Samsung A10, Pixel 4a, Samsung Tab A |
| iOS | 13.0 | 17.0 (Xcode 15) | iPhone SE 2nd, iPhone 12, iPad 9th |

---

### 10. Third-Party Services Accessed Directly by APP

| Service | Purpose | Integration Mode | Notes |
|---------|---------|------------------|-------|
| Azure Blob Storage | Banner / CSV transfers | HTTPS (TLS 1.2) via **time-limited SAS URL** | All presign operations handled by Backend |
| Apple App Store / Google Play | Distribution | Fastlane | No in-app purchase SDK |
| Backend API | All business operations | HTTPS (TLS 1.2) REST/JSON | No direct MSG91, PostgreSQL, or Queue access |

(No MSG91, PostgreSQL, Azure Queue, or push notification SDKs are linked; the Backend abstracts all infrastructure services.)

---

### 11. Version-Pin & Upgrade Policy

1. All core dependencies are pinned to **exact versions** in `pubspec.yaml`.  
2. Dependabot opens PRs for patch updates; auto-merge if CI passes.  
3. Minor upgrades evaluated quarterly; major upgrades require team consensus & full regression cycle.  
4. Flutter SDK patch updates are applied within 2 weeks of stable release unless blocked by store outages.
5. No external Architecture Review Board (ARB); version decisions made by development team based on security and compatibility needs.

---

### 12. Security & Secrets Management

| Aspect | Implementation | Notes |
|--------|----------------|-------|
| JWT Storage | flutter_secure_storage | RS256-signed tokens stored securely, passed via Authorization header |
| Secrets Management | Encrypted App-Service settings | No Azure Key Vault; manual rotation during maintenance |
| OTP Security | Backend-managed | Plaintext storage in PostgreSQL (accepted risk); 5-attempt lockout enforced by backend |
| Push Notifications | None | No FCM/APNS integration; all notifications via SMS/Email through backend |

---

### 13. Summary

The APP component adopts a lean, stable Flutter 3.13.9 / Dart 3.1.5 stack with `flutter_bloc` state-management, `dio` networking and strict secure-storage rules. Version pins, CI gates and conservative OS targets meet EssilorLuxottica's requirements for a single-region, production-only deployment supporting ~5,000 store users without unnecessary architectural overhead. The online-only architecture eliminates local database complexity while JWT tokens are securely stored and transmitted via Authorization headers.

*End of document*