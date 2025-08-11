## L3-CM-WEBPRT: Configuration Manual (CM) for WEBPRT  

PromoPartner – Essilor Luxottica Joint-Promotions Platform  
Component: WEBPRT (Nuxt 3 SSR Application)  
Version 1.0 • 2025-08-07  

---

### 1. Purpose  

This manual guides administrators through post-deployment configuration for WEBPRT—the web portal for Admin and KAM users. It covers:  
• Configurable parameters in Azure App Service  
• Environment-specific settings  
• Security configurations  
• Runtime reconfiguration procedures  

---

### 2. Component Summary  

WEBPRT is a Nuxt 3 SSR application:  
• Running in Node 20 Docker container  
• Deployed to Azure App Service (Linux, P1v2 plan)  
• Accessing all data through BCKND REST API  
• Using HttpOnly cookies for JWT storage  

---

### 3. Configuration Locations  

| Configuration Type | Where Configured | Who Manages |
|-------------------|------------------|-------------|
| Environment Variables | Azure App Service Settings | DevOps Team |
| IP Allow-list | Azure App Service Access Restrictions | DevOps Team |
| JWT Configuration | Environment Variables | DevOps Team |
| TLS Settings | Azure App Service (enforced) | Azure Platform |

---

### 4. Required Environment Variables  

Configure these in Azure App Service Application Settings:

| Variable | Required | Example Value | Purpose |
|----------|----------|---------------|---------|
| `API_BASE_URL` | Yes | `https://api.promopartner.xxx/v1` | Backend API endpoint (per L3-LLD-WEBPRT §11) |
| `NODE_ENV` | Yes | `production` | Node environment (per L1-TSD §3) |
| `OFFICE_CIDRS` | Yes | `123.45.0.0/16,78.90.12.0/24` | Office IP ranges (per L1-HLD §2) |
| `NUXT_PUBLIC_DEBUG` | No | `""` (empty) | Debug mode (per L3-LLD-WEBPRT §8) |

**Important**: All secrets are stored as encrypted App Service settings. No Azure Key Vault is used.

---

### 5. Security Configuration  

#### 5.1 IP Restriction  
Configure office IP allow-list in Azure App Service:
```
Azure Portal → App Service → Networking → Access Restrictions
Add rules for each OFFICE_CIDR
```

#### 5.2 JWT Configuration  
• Tokens stored in HttpOnly cookies (Secure, SameSite=Strict)  
• RS256 signing algorithm  
• No client-side token access  
• Automatic inclusion in API requests  

#### 5.3 TLS Configuration  
• Minimum TLS 1.2 enforced by Azure App Service (per Key Instruction #11)
• HSTS header with 180-day max-age (per L3-LLD-BCKND §9)  
• Managed certificates provided by Azure (per L1-TSD §5)

---

### 6. Environment-Specific Settings  

| Setting | Production |
|---------|------------|
| API_BASE_URL | `https://api.promopartner.xxx/v1` |
| NODE_ENV | `production` |
| OFFICE_CIDRS | Real office IP ranges |
| Debug Mode | Disabled |
| IP Restrictions | Enforced |

**Note**: No staging or UAT environment exists. All deployments are in-place to production.

---

### 7. Runtime Reconfiguration  

To update configuration without redeployment:

1. **Update App Service Settings**:
   - Azure Portal → App Service → Configuration → Application settings
   - Modify required values
   - Click Save

2. **Restart App Service**:
   - Settings apply after restart
   - Causes brief downtime (< 30 seconds)

3. **Verify Changes**:
   - Check application logs in Application Insights
   - Test functionality with updated settings

---

### 8. Role-Based Access Configuration  

KAM users have read-only access. This is enforced by:
• Backend API returns role in JWT claims  
• Frontend hides write controls based on role  
• Backend rejects write operations from KAM users  

No portal-side configuration needed for RBAC.

---

### 9. Integration Points  

| Integration | Configuration Required | Reference |
|-------------|----------------------|-----------|
| Backend API | `API_BASE_URL` environment variable | L3-LLD-WEBPRT §11 |
| Azure Blob Storage | Via Backend SAS URLs (no direct config) | L3-FRS-WEBPRT FR-WEBPRT-09 |
| MSG91 | Configured in Backend only | L2-LLD-INTG3P §4 |
| PostgreSQL | Configured in Backend only | L1-HLD §5 |

---

### 10. Troubleshooting Guide  

| Issue | Check | Solution |
|-------|-------|----------|
| Cannot access portal | IP restrictions | Verify office IP in allow-list |
| API calls failing | API_BASE_URL setting | Ensure correct backend URL |
| Authentication issues | Browser cookies | Clear cookies and re-login |
| 502 Bad Gateway | App Service logs | Check container startup logs |
| Missing features for KAM | JWT role claim | Verify role returned by backend |

---

### 11. Maintenance Tasks  

Monthly:
• Review and update office IP ranges if needed  
• Check App Service health metrics  

Quarterly:
• Review security settings alignment  
• Update Node base image if security patches available  

---

### 12. Deployment Notes  

• In-place deployment to production App Service (per Key Instruction #1)
• No slot-swap mechanism used (per L1-HLD §9)  
• Deployments during 5-minute maintenance windows (per L1-HLD §9)
• Container image from Azure Container Registry (per L1-TSD §5)
• Single production environment only (per L1-HLD §9)

---

End of Document