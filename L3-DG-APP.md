## L3-DG-APP: Deployment Guide (DG) for APP

### 1. Introduction & Scope

This guide describes the deployment process for the PromoPartner Mobile APP (Flutter for iOS & Android) in a **single production environment** with in-place deployment strategy. It covers:

* Required tooling and credentials
* Build and signing process  
* GitHub Actions CI/CD pipeline
* App store distribution
* Basic verification procedures

The guide is optimized for simplicity, supporting ~5,000 users with ≤1,000 concurrent sessions.

---

### 2. Prerequisites

#### 2.1 Accounts & Access

| Need | Purpose | Responsible | Reference |
|------|---------|-------------|-----------|
| GitHub repo `promopartner-app` | Source & CI | Dev Team | L1-TSD §6 - GitHub (3 repos) |
| Apple Developer Account | App Store distribution | EssilorLuxottica | L1-TSD §7 - Apple App Store |
| Google Play Console | Play Store distribution | EssilorLuxottica | L1-TSD §7 - Google Play |
| Azure Container Registry | Image storage (not used for APP) | Operations | L1-TSD §5 - ACR (Basic) |

#### 2.2 Build Environment

| Tool | Version | Where Used | Reference |
|------|---------|------------|-----------|
| Flutter SDK | 3.2.x | CI runners | L3-LLD-APP §2.1 - Flutter 3.2.x |
| Dart | 3.2.x | CI runners | L1-TSD §2 - Dart 3.2.x |
| Xcode | 14.x | macOS runner | L3-LLD-APP build requirement |
| Android SDK | API 28+ (Android 8.0) | Ubuntu runner | L3-LLD-APP §5 - Android 8.0 minimum |
| Fastlane | Latest | CI automation | L1-TSD §6 - Fastlane for mobile |


---

### 3. Secrets Management

All secrets stored as **encrypted GitHub Secrets** (per L1-TSD §6 and Key Instruction #2 - no Azure Key Vault):

| Secret Name | Description | Reference |
|-------------|-------------|-----------|
| ANDROID_KEYSTORE_B64 | Base64-encoded signing keystore | L1-TSD §6 - signing keys in GitHub secrets |
| ANDROID_KEYSTORE_PASSWORD | Keystore password | L1-TSD §6 |
| APPLE_APP_SPECIFIC_PASSWORD | App Store Connect automation | L1-TSD §6 |
| MATCH_PASSWORD | iOS certificate encryption | L1-TSD §6 |

Backend secrets (MSG91 keys, JWT signing key) stored in **encrypted App Service settings** per L1-HLD §8 and Key Instruction #2.

---

### 4. Platform Signing Configuration

#### 4.1 iOS Signing

* Use Fastlane Match for certificate management
* Certificates stored in private GitHub repository
* Production provisioning profile only

#### 4.2 Android Signing  

* Single upload keystore for Play Store
* Google Play App Signing enabled for key rotation capability
* Keystore backed up in secure team storage

---

### 5. Deployment Strategy

**Production-Only Environment** (per L1-OVERVIEW §4 and Key Instruction #1):
```
main branch → v*.*.* tag → GitHub Actions → App Stores → Production
```

* **In-place deployment** during 5-minute maintenance window (L1-HLD §9)
* No staging environment (L1-OVERVIEW - single Production tier)
* No slot-swap mechanism (Key Instruction #1)
* Force upgrade via `X-Min-App-Version` header (L3-LLD-APP §5)

---

### 6. CI/CD Pipeline Configuration

#### 6.2 Build Configuration

* Production API: `https://api.promopartner.in/v1` (L3-LLD-APP §7)
* **TLS 1.2** for all connections (Key Instruction #11)
* **No push notifications** - SMS/Email only via MSG91 (Key Instruction #4)
* **No analytics SDKs** in phase 1 (L3-LLD-APP §13)
* **Online-only mode** (L3-LLD-APP §2.1)
* JWT storage: **Secure storage** with Authorization header (Key Instruction #16)
* JWT signing: **RS256** (Key Instruction #13)

---

### 7. Deployment Steps

#### 7.1 Pre-Deployment

1. Ensure all tests pass on main branch
2. Create semantic version tag (e.g., v1.0.0)
3. Update release notes in both stores

#### 7.2 Automated Deployment

1. Push tag triggers GitHub Actions
2. CI builds and signs both platforms
3. Fastlane uploads to:
   - iOS: TestFlight for immediate production release
   - Android: Production track with staged rollout

#### 7.3 Post-Deployment

1. Verify app appears in stores (1-2 hours for iOS, 2-4 hours for Android)
2. Test core flows: Login OTP, New Transaction, Reconciliation
3. Monitor backend logs for version adoption

---

### 8. Version Management

* Backend enforces minimum version via header (L3-FRS-APP FR-31, FR-32)
* Force upgrade shows blocking dialog (L3-LLD-APP §5)
* Version sent in `X-App-Version` header (L3-FRS-APP FR-31)

---

---

### 9. Rollback Procedure

Given production-only environment, rollback is simplified:

| Platform | Action | Time |
|----------|--------|------|
| iOS | Submit previous build from TestFlight | 2-24 hours (review) |
| Android | Rollout previous APK to 100% | 2-4 hours |
| Backend | Adjust `X-Min-App-Version` if needed | Immediate |

---

### 10. Monitoring & Verification

Post-deployment checks (manual, no telemetry in phase 1):

* App Store/Play Console crash reports
* Backend API logs for mobile traffic
* Support ticket volume
* Store reviews

---

### 11. Security Considerations

Per documented security design:

* **JWT in secure storage**, passed via Authorization header (Key Instruction #16)
* **RS256-signed** JWTs (Key Instruction #13)
* **TLS 1.2** minimum (Key Instruction #11)
* **OTP plaintext** in PostgreSQL (Key Instruction #9)
* OTP lockout: **5 attempts/10 min → 15 min lockout** (Key Instruction #8)


---

### 12. Maintenance Windows

* Deployments during scheduled 5-minute maintenance windows
* Users see "Under Maintenance" message from backend
* No data loss (online-only architecture)
* Sessions resume after maintenance

---

### 13. Troubleshooting

| Issue | Resolution |
|-------|------------|
| Build fails | Check GitHub Secrets validity |
| Store rejection | Review metadata and screenshots |
| Users stuck on old version | Backend force-upgrade via header |
| High crash rate | Rollback to previous version |

---

### 14. Dependencies

* Azure Database for PostgreSQL with **7-day PITR only, no DR** (Key Instruction #5)
* **MSG91 for both SMS and Email** OTP (Key Instructions #6, #14)
* No Azure Key Vault dependency (Key Instruction #2)
* No push notification services (Key Instruction #4)

---

_End of Document_