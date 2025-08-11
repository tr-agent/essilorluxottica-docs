## L3-DG-WEBPRT: Deployment Guide for WEBPRT

---

### 1. Purpose
Step-by-step instructions for deploying the WEBPRT admin portal to Azure App Service using in-place deployment strategy. No staging slots or blue-green deployments are used.

---

### 2. Infrastructure Overview

- TLS 1.2 enforced via App Service Managed Certificate
- Office IP restrictions applied at App Service level
- No DR region or backup environment

---

### 3. Prerequisites

- Azure CLI installed and authenticated with Contributor role
- Docker installed for local builds (optional - CI handles this)
- GitHub repository access with push permissions
- Maintenance window scheduled (5 minutes downtime expected)

---

### 4. Environment Configuration

Configure these in App Service Application Settings:

- `API_BASE_URL` - Backend API endpoint (https://api.promopartner.xxx/v1)
- `OFFICE_CIDRS` - Comma-separated IP ranges for access restriction
- `NODE_ENV` - Set to "production"
- `PORT` - Set to "8080"
- `NUXT_PUBLIC_DEBUG` - Set to "" (empty) for production

Update settings using Azure Portal or CLI:

```
az webapp config appsettings set --name app-pp-webprt-prod --resource-group rg-pp-prod --settings KEY=VALUE
```

---

### 5. Deployment Process

#### 5.1 Automated Deployment (GitHub Actions)

1. Create release tag: `git tag webprt-v1.0.0 && git push origin webprt-v1.0.0`
2. GitHub Actions automatically:
   - Builds Nuxt 3 application
   - Creates Docker image with git SHA tag pattern
   - Pushes to Azure Container Registry (ACR)
   - Updates App Service using `az webapp config container set` for in-place deployment
3. Monitor deployment in GitHub Actions tab

#### 5.2 Manual Deployment (Emergency Only)

1. Build Docker image locally
2. Push to Azure Container Registry using `az acr build`
3. Update App Service container configuration
4. Restart App Service

Expected downtime: 3-5 minutes during container swap

---

### 6. Post-Deployment Verification

1. **Container Health**: Check App Service logs show "Container started successfully"
2. **HTTPS Access**: Verify portal loads at https://web.promopartner.xxx
3. **API Connection**: Attempt login to confirm backend connectivity
4. **IP Restrictions**: Verify access blocked from non-office IPs
5. **Application Insights**: Confirm telemetry flowing
6. **HttpOnly Cookies**: Verify JWT stored in HttpOnly cookies (not accessible via JavaScript)

---

### 7. Rollback Procedure

1. Identify previous working image tag from Container Registry
2. Update App Service to use previous image:

   ```
   az webapp config container set --name app-pp-webprt-prod --resource-group rg-pp-prod --docker-custom-image-name [previous-image]
   ```

3. Restart App Service
4. Verify functionality restored
5. Log incident for review

---

### 8. Maintenance Tasks

#### Quarterly

- Rotate secrets in App Service settings during quarterly maintenance window (manual process per L1-HLD)
- Review and update office IP allow-list
- Clean up old Docker images in Container Registry

#### As Needed

- Scale up to P2v2 if sustained high CPU (>70% for 30 minutes)
- Apply security patches via new deployment

---

### 9. Monitoring & Alerts

- CPU/Memory metrics via Azure Monitor
- Application errors via Application Insights
- Set alert for sustained CPU >70% (scaling trigger)
- Monitor container restart frequency

---

### 10. Important Notes

- **No staging environment** - all deployments go directly to production
- **No automated backups** - rely on Git for code recovery
- **Database backups** - PostgreSQL 7-day PITR only, no cross-region DR
- **Secrets management** - No Key Vault; use encrypted App Service settings
- **Deployment window** - Schedule during low-usage hours (typically 2-7 AM IST)

---

_End of Document_