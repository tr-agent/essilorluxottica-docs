## L3-CM-BCKND: Configuration Manual (CM) for BCKND

### 1. Purpose
This manual provides configuration instructions for operators managing the PromoPartner Backend (BCKND) in production. It covers required settings, acceptable values, and change procedures for the single-instance Azure App Service deployment.

---

### 2. Configuration Locations

| Location | Purpose | Change Method |
|----------|---------|---------------|
| Azure App Service Settings | Secrets, connection strings, feature flags | Azure Portal / CLI |
| `appsettings.json` (in container) | Fixed business rules (OTP TTLs, rate limits) | Code change → rebuild |
| App Service Scale/Restart | Apply configuration changes | Azure Portal / CLI |

All secrets are stored in encrypted App Service settings. No Azure Key Vault is used.

---

### 3. Required Configuration

#### 3.1 Core Settings
| Setting | Example Value | Purpose |
|---------|---------------|---------|
| `ASPNETCORE_ENVIRONMENT` | `Production` | Controls logging verbosity |
| `APPINSIGHTS_CONNECTION_STRING` | `InstrumentationKey=...` | Optional telemetry |

#### 3.2 Database
| Setting | Example | Notes |
|---------|---------|-------|
| `PG__ConnectionString` | `Host=pg-promo.postgres.database.azure.com;Database=promopartner;Username=...;Password=...;SSL Mode=Require` | Must include SSL Mode=Require |
| `PG__MaxPoolSize` | `30` | Match B2ms PostgreSQL limits |

#### 3.3 Storage
| Setting | Example | Purpose |
|---------|---------|---------|
| `AZURE_STORAGE_CONNECTION_STRING` | `DefaultEndpointsProtocol=https;AccountName=...` | Blob and Queue access |
| `BLOB__AssetsContainer` | `assets` | CSV, images, reports |
| `BLOB__AuditContainer` | `audit` | Immutable audit logs |
| `QUEUE__OtpName` | `otp-messages` | OTP delivery queue |

#### 3.4 Authentication (JWT RS256)
| Setting | Format | Notes |
|---------|--------|-------|
| `JWT__RsaKeyPem` | Base64-encoded PEM | RS256 private key per L3-LLD-BCKND §6.1 |
| `JWT__RsaPreviousKeyPem` | Base64-encoded PEM | Previous key for rotation per L3-LLD-BCKND §9 |


#### 3.5 MSG91 Integration
| Setting | Example | Purpose |
|---------|---------|---------|
| `MSG91_SMS_AUTHKEY` | `32-char-key` | Per L2-LLD-INTG3P §4.1 |
| `MSG91_EMAIL_AUTHKEY` | `32-char-key` | Per L2-LLD-INTG3P §4.2 |
| `MSG91_SMS_SENDER_ID` | `ESSLUX` | Per L2-LLD-INTG3P §6.1 |
| `MSG91_SMS_OTP_TEMPLATE_ID` | `19-char-id` | Per L2-LLD-INTG3P §6.1 |
| `MSG91_EMAIL_TEMPLATE_ID` | `template-id` | Per L2-LLD-INTG3P §4.2 |

#### 3.6 OTP Configuration (Fixed in appsettings.json)
```json
{
  "OTP": {
    "LoginTTLMinutes": 10,
    "LoginMaxAttempts": 5,
    "LoginLockoutMinutes": 15,
    "LoginResendLimit": 3,
    "LoginResendCooldownSeconds": 60,
    "CustomerTTLMinutes": 5,
    "CustomerMaxAttempts": 3,
    "CustomerLockoutMinutes": 15
  }
}
```

#### 3.7 Rate Limiting
| Setting | Default | Purpose |
|---------|---------|---------|
| `RATELIMIT__LoginOtpPer10Min` | `5` | Per L3-FRS-BCKND FR-BCKND-101 |
| `RATELIMIT__CustomerOtpPer15Min` | `3` | Per L3-FRS-BCKND FR-BCKND-105 |
| `RATELIMIT__ApiPerMinute` | `60` | Per L3-LLD-APP §12 |

#### 3.8 CSV Processing
| Setting | Default | Purpose |
|---------|---------|---------|
| `CSV__MaxFileSizeMB` | `50` | Per L1-NFRS §2 and L3-FRS-BCKND FR-BCKND-801 |
| `CSV__BatchSize` | `1000` | Per L3-FRS-BCKND FR-BCKND-802 |
| `CSV__SasUrlTTLMinutes` | `15` | Per L3-LLD-BCKND §6.4 |

---

### 4. Operational Procedures

#### 4.1 Initial Setup
1. Create Azure resources (App Service, PostgreSQL, Storage Account)
2. Set all required configuration in App Service Settings
3. Ensure PostgreSQL firewall allows App Service outbound IPs
4. Create blob containers (`assets`, `audit`) and queue (`otp-messages`)
5. Deploy container image and restart App Service

#### 4.2 Configuration Changes
1. Update setting in Azure Portal or CLI
2. Save changes
3. Restart App Service (20-30 second downtime)
4. Verify via health endpoint: `GET /healthz`

#### 4.3 Secret Rotation (Zero Downtime)
**JWT Key Rotation:**
1. Generate new RSA key pair
2. Add `JWT__RsaPrivateKeyNew` with new key
3. Add `JWT__RsaPrivateKeyOld` with current key
4. Update `JWT__RsaPrivateKey` to new key value
5. Restart App Service
6. Wait 65 minutes (token TTL + buffer)
7. Remove `JWT__RsaPrivateKeyOld`

**MSG91 Key Rotation:**
1. Obtain new API keys from MSG91
2. Update `MSG91_SMS_AUTHKEY` and `MSG91_EMAIL_AUTHKEY`
3. Restart App Service
4. Test OTP flow immediately

---

### 5. Health Checks

| Endpoint | Expected Response | Validates |
|----------|------------------|-----------|
| `/healthz` | `{"status":"Healthy"}` | App running |
| `/healthz/ready` | `{"database":"OK","storage":"OK","queue":"OK"}` | Dependencies connected |

---

### 6. Backup & Recovery

- **Database**: Azure PostgreSQL automatic daily backups (7-day PITR)
- **Blob Storage**: No automated backups; audit container is immutable
- **Configuration**: Export App Service settings before changes
- **No cross-region DR capability** (accepted by client)

---

### 7. Monitoring Thresholds

| Metric | Warning | Critical | Action |
|--------|---------|----------|--------|
| OTP Queue Depth | >1000 | >5000 | Per L2-LLD-INTG3P §8 |
| DB Connection Pool | >25 | >28 | Per L3-LLD-BCKND §8 (30 max) |
| Failed OTP Sends | >100/5min | >500/5min | Per L2-LLD-INTG3P §8 |

---

### 8. Environment Variables Quick Reference

```bash
# Export current settings
az webapp config appsettings list \
  --resource-group rg-promopartner \
  --name app-bcknd \
  --output json > settings-backup.json

# Update single setting
az webapp config appsettings set \
  --resource-group rg-promopartner \
  --name app-bcknd \
  --settings "SETTING_NAME=value"

# Restart after changes
az webapp restart \
  --resource-group rg-promopartner \
  --name app-bcknd
```

---

*End of Configuration Manual*