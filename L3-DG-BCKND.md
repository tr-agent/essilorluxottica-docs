## L3-DG-BCKND: Deployment Guide (DG) for BCKND

1. Purpose
This guide provides just‑enough, production‑ready steps to package, release, configure, verify, and (if needed) roll back the BCKND (.NET 8) API. It matches the approved architecture:

- One Azure App Service (Linux container), single instance (vertical scale only), autoscale disabled
- In-place container update (no deployment slots)
- EF Core forward-only migrations applied on startup
- All Azure resources pre-provisioned, single region (Central India)
- Secrets in App Service settings (no Key Vault)

2. Deployment Topology (Recap) – updated

- App Service Plan: PremiumV2 P1v2, Linux; single instance; no slots. Ref: L1-TSD §5 (Hosting & Infrastructure), L1-HLD §9 (Deployment Architecture)
- Backend App: Azure App Service container (BCKND). Ref: L1-HLD §5, §9
- Azure Container Registry: Basic SKU. Ref: L1-TSD §5
- PostgreSQL: Azure Database for PostgreSQL Flexible Server v16 (B2ms). Ref: L1-HLD §5, L1-TSD §4
- Storage: Azure Blob (assets + immutable audit) and Azure Storage Queue. Ref: L1-HLD §5, L1-TSD §4
- Monitoring: Application Insights. Ref: L1-TSD §6, §8

3. Prerequisites

- Azure identity with Contributor on app-bcknd and AcrPush on acr-promopartner
- GitHub repo promo/bcknd, protected main branch
- For manual deploys only: Docker 25+, .NET 8 SDK, Azure CLI 2.61+
- Egress to: Azure RM, ACR, *.blob.core.windows.net,*.queue.core.windows.net, api.msg91.com
- 5-minute maintenance window agreed with business

4. Release Artifacts

- Container image: acr-promopartner.azurecr.io/bcknd:{gitSha}
- OpenAPI: promopartner-contracts/openapi/promopartner-v1.yaml
- EF migrations compiled into image (forward-only)

5. CI/CD Pipeline (Automated Path) – notes aligned

- Build and push Docker image with GitHub Actions; deploy via az webapp config container set (in-place, no slot-swap). Ref: L1-TSD §6–§7, L1-HLD §9
- Single production environment only. Ref: L1-HLD §9, §10

6. Manual Deployment (fallback)

- Build and push
  - git checkout main && git pull
  - docker build -t acr-promopartner.azurecr.io/bcknd:manual-$(git rev-parse --short HEAD) .
  - az acr login -n acr-promopartner
  - docker push acr-promopartner.azurecr.io/bcknd:manual-<sha>
- Point App Service to the new image
  - az webapp config container set --name app-bcknd --resource-group RG-promo-core --docker-custom-image-name acr-promopartner.azurecr.io/bcknd:manual-<sha> --docker-registry-server-url <https://acr-promopartner.azurecr.io>
  - az webapp restart --name app-bcknd --resource-group RG-promo-core
- Tail logs
  - az webapp log tail --name app-bcknd --resource-group RG-promo-core

7. Configuration Reference – updated keys and mapping

- Database
  - PG__ConnectionString, PG__MaxPoolSize=30. Ref: L3-LLD-BCKND §8 (pool size), L1-HLD §5
- JWT (RS256)
  - JWT signing key stored in App Service settings (private RSA key PEM). Ref: L1-HLD §8, L3-LLD-BCKND §9
- Storage (Blob)
  - assets container, audit container (immutable). Ref: L1-HLD §5, L1-TSD §4
- Queue
  - QUEUE__OtpQueueName=otpMessage. Ref: L3-FRS-BCKND §3 (Component Interfaces)
- MSG91 (separate keys per channel and DLT specifics)
  - MSG91_SMS_AUTHKEY, MSG91_EMAIL_AUTHKEY. Ref: L2-LLD-INTG3P §4.1, §4.2, §5
  - MSG91_SMS_SENDER_ID, MSG91_SMS_OTP_TEMPLATE_ID (DLT). Ref: L2-LLD-INTG3P §3, §4.1 (senderId/template_id)
- Schedulers
  - SCHED__CronPartition="0 0 1 **", SCHED__CronPromoAutoArchive="0 0** *". Ref: L3-LLD-BCKND §3 (Scheduler), §6.1 notes; L3-FRS-BCKND §2.12
- Observability
  - APPLICATIONINSIGHTS_CONNECTION_STRING. Ref: L1-TSD §6, §8

8. Verification Checklist – simplified

- App Service state is Running. Ref: L1-HLD §9 (single prod app, in-place)
- OTP smoke test: POST /v1/auth/otp/request and /v1/auth/otp/verify succeed (SMS/Email via MSG91). Ref: L2-LLD-IC §5.1, L2-LLD-INTG3P §4.1–§4.2, L3-LLD-BCKND §6.1
- Queue processing visible (otpMessage dequeues, deliveries logged in App Insights). Ref: L2-LLD-IC §4.2, §5.1; L3-LLD-BCKND §3 (Notification Worker)
- Application Insights receiving traces. Ref: L1-TSD §6, §8

9. Rollback Procedures – clarified

- Code rollback only (repoint to previous image tag); no slot/swap. Ref: L1-HLD §9 (in-place), L1-TSD §6–§7
- Do not run down-migrations; schema change reversal is out of scope. Ref: forward-only noted in DG purpose; deployment/operations model in L3-LLD-BCKND §11

10. Security & Compliance Notes – aligned

- Secrets in App Service settings (encrypted at rest); no Key Vault. Ref: L1-HLD §8, §10; L1-TSD §6, §8; L3-LLD-BCKND §9
- TLS 1.2 on public endpoints; API is Internet-exposed, Portal may be IP-allow-listed. Ref: L1-HLD §2 (ingress), §8 (security), §10 (constraints)
- Immutable audit container for append-only audit JSON. Ref: L1-HLD §5 (Audit Log), L1-FRS §2.9, L1-TSD §4

11. Change Control & Versioning

- Merge to main = production release
- Image tag = git-sha; record in GitHub Release notes
- Post release summary in agreed change-log channel

12. Troubleshooting (quick)

- 502/boot loop: az webapp log tail; if immediate failure, rollback image tag
- Bad image tag or setting: revert tag and fix app setting
- EF migration timeout: temporarily scale up PostgreSQL compute; restart app
- MSG91 errors: verify MSG91__* settings, see Notification Worker logs, check MSG91 status page
- Queue backlog: check queue depth in Azure Storage; confirm worker is pulling

13. Appendix: One-time app settings (optional helpers)

- Configure health check path (once):
  - az webapp update -g RG-promo-core -n app-bcknd --set healthCheckPath="/healthz"
- Disable autoscale, pin instances to 1:
  - Ensure App Service plan scale-out = 1 instance (portal)
