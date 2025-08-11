## L3-CM-APP: Configuration Manual (CM) for APP

Author: Solution Architecture Team  
Version: 2.0  
Date: 2025-01-15

---

### 1. Audience & Scope

This manual enables the following personas to configure the PromoPartner Mobile App after deployment to Apple App Store / Google Play:

- Mobile App Administrator (EssilorLuxottica IT)
- DevOps engineer maintaining the GitHub Actions pipeline
- Support engineer diagnosing production issues

Covers single-region, production-only deployment for ≤5,000 users with ≤1,000 concurrent sessions.

---

### 2. Configuration Overview

| Layer | When Applied | Mechanism |
|-------|-------------|-----------|
| Compile-time build | Build time (CI or local) | `--dart-define` |
| Code signing certificates | Build time | GitHub Secrets |
| Runtime secure storage | First launch/login | Flutter Secure Storage |
| Backend-driven data | Runtime (HTTPS) | REST `/v1` APIs |

---

### 3. Configuration Entry Points

#### 3.1 Build Configuration

Single production build only (no dev/staging per Key Instruction #1).

```
flutter build <apk|appbundle|ipa> --release
```

### 3.2 Dart-Define Constants

| Key | Value | Description |
|-----|-------|-------------|
| API_BASE_URL | `https://api.promopartner.in` | Backend API endpoint (L3-LLD-APP §4) |
| APP_VERSION | `X.Y.Z` | For `X-App-Version` header (L3-FRS-APP FR-31) |

---

### 4. Component System Settings

| Setting | Level | Description | Value |
|---------|-------|-------------|-------|
| API_BASE_URL | Compile-time | REST endpoint for Backend communication | `https://api.promopartner.in` |
| APP_VERSION | Compile-time | Version sent in `X-App-Version` header | Semantic version |
| JWT_REFRESH_INTERVAL | Hard-coded | Silent refresh before JWT expiry | 55 minutes |
| REQUEST_TIMEOUT | Hard-coded | HTTP request timeout | 30 seconds |
| OTP_LENGTH | Hard-coded | OTP code length | 6 digits |
| MIN_IOS_VERSION | Build config | Minimum iOS version | 13.0 |
| MIN_ANDROID_VERSION | Build config | Minimum Android API level | 28 (Android 8.0) |

---

### 5. Parameter Definitions

Non-configurable runtime parameters for reference:

1. **request_uuid** - UUID v4 for transaction idempotency
2. **eligibility_token** - Opaque token from customer OTP verification (5-min TTL)
3. **jwt** - RS256-signed access token (60-min TTL)
4. **refreshToken** - Refresh token (12-hour TTL)
5. **X-Store-Id** - Header for Parent Store context switching
6. **invoiceNo/pidNo** - Pattern: `[A-Za-z0-9-_]{1,30}`

---

### 6. Environment-Specific Configuration

Single production environment only:

| Category | Production |
|----------|-----------|
| API endpoint | `https://api.promopartner.in` |
| TLS version | 1.2 |
| Apple signing | Distribution certificate via Fastlane |
| Google signing | Upload keystore in GitHub Secrets |
| Store listing | App Store / Play Store Production |

---

### 7. Integration Configurations

#### 7.1 Backend REST API

```
Base URL: https://api.promopartner.in
TLS Version: 1.2 (per Key Instruction #11)
Auth Header: Authorization: Bearer <JWT> (L3-LLD-APP §2.4)
Connect Timeout: 10 seconds (L3-LLD-APP §12)
Read Timeout: 30 seconds (L3-LLD-APP §12)
```

#### 7.2 Azure Blob Storage

**Based on L3-FRS-APP FR-24:**

- Direct upload/download via SAS URLs from backend
- SAS URL TTL: 15 minutes (L3-FRS-BCKND FR-29)
- Auto-retry once on 403 (L3-LLD-APP §2.4 SASRetryInterceptor)

---

### 8. Security Configurations

| Concern | Configuration |
|---------|--------------|
| Transport security | TLS 1.2 (Key Instruction #11) |
| Token storage | Flutter Secure Storage - JWT + refresh token (Key Instruction #16, L3-LLD-APP §5) |
| Auth flow | 6-digit OTP via SMS/Email → RS256 JWT (Key Instructions #13, #14) |
| OTP policies | Per Key Instruction #15 |
| JWT TTL | 60 minutes (L3-FRS-BCKND FR-2) |
| Refresh token TTL | 12 hours (L3-LLD-APP §2.4) |

---

### 9. Change Procedures

#### 9.1 Updating API Endpoint

1. Update `API_BASE_URL` in CI/CD pipeline `--dart-define`
2. Increment version in `pubspec.yaml`
3. Create release tag → triggers GitHub Actions build
4. Submit to App Store / Play Store

---

### 10. Validation Checklist

| Item | Validation | Pass Criterion | Reference |
|------|------------|----------------|-----------|
| API connectivity | `GET /v1/healthz` | `200 OK` | Standard health check |
| OTP login | Request & verify OTP | JWT received | L3-FRS-APP FR-03/04 |
| Token refresh | Wait 55 minutes | Auto-refresh succeeds | L3-FRS-APP FR-05 |
| Store context | Parent Store switches | `X-Store-Id` present | L3-FRS-APP FR-08 |
| Transaction idempotency | Submit same UUID twice | Same transaction returned | L3-FRS-APP FR-19/20 |


---

### 11. Appendix

#### 11.1 CI Build Command

```yaml
flutter build appbundle \
  --release \
  --dart-define=API_BASE_URL=https://api.promopartner.in \
  --dart-define=APP_VERSION=${{ github.ref_name }}
```

#### 11.2 Android Network Security Config

```xml
<network-security-config>
  <base-config cleartextTrafficPermitted="false">
    <trust-anchors>
      <certificates src="system"/>
    </trust-anchors>
  </base-config>
</network-security-config>
```

#### 11.3 iOS Info.plist Security

```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <false/>
</dict>
```

### 11.4 Required GitHub Secrets

- `IOS_DIST_CERT_B64` - iOS distribution certificate
- `IOS_DIST_CERT_PASSWORD` - Certificate password  
- `ANDROID_KEYSTORE_B64` - Android keystore
- `ANDROID_KEYSTORE_PASSWORD` - Keystore password
- `ANDROID_KEY_ALIAS` - Key alias
- `ANDROID_KEY_PASSWORD` - Key password

---

_End of document_