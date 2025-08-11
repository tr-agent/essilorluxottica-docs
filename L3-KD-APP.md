## L3-KD-APP: Component-Specific Key Decisions for APP

* **C#1 – Online-Only Operation** (Client Clarification #1)  
  The APP remains strictly online as per L3-WF-APP §3 architectural assumptions. Every business action calls BCKND in real time; if the network is unavailable, users are shown a blocking "Offline" banner and write-actions are disabled. No local database or persistent queue is shipped.

* **C#2 – Store-Context Switching** (Client Clarification #2, L2-LLD-IC §5)  
  Active child-store context is sent on every request via `X-Store-Id`. When switching stores, Backend validates RBAC and returns a new JWT scoped to the selected `store_id` as per L3-WF-APP §10.1. While parent-store claims remain in the JWT, the token is reissued with the active store context.

* **C#3 – Product Catalogue Access** (Client Clarification #3, L2-LLD-IC Endpoints #9-11)  
  All product lookup is server-side, paginated (`pageSize ≤ 50`). The APP keeps no SKU cache; list widgets implement infinite scroll backed by `/v1/products`.

* **C#4 – Transaction Idempotency** (Client Clarification #4, L2-LLD-IC Endpoint #22)  
  On the first `/v1/transactions` call the Backend returns `201 Created` with the transaction details including a `request_uuid`. The APP stores this key in RAM and re-uses it on every retry until a terminal 201/4xx response is received, ensuring duplicate-proof submits.

* **C#5 – State-Management Standard** (Client Clarification #5)  
  Flutter implementation standardises on `flutter_bloc 8.x` (Cubit for simple, Bloc for multi-step flows) organised in Clean-Architecture layers: Presentation → Bloc → Repository → Datasource. Riverpod and other patterns are disallowed to preserve consistency.


* **C#6 – Token Lifetime & Refresh** (Client Clarification #7, L2-LLD-IC §9)  
  JWT stored in secure local storage (e.g., Flutter Secure Storage) and passed via Authorization headers. It is silently refreshed using the 12 h refresh token five minutes before expiry. On failed refresh or `UNAUTHENTICATED` error the APP purges credentials and navigates to Login.

* **C#7 – Mobile Telemetry Deferred** (Client Clarification #8)  
  No crash- or analytics-SDK is embedded at launch. Server traces combined with app-store feedback satisfy current observability needs; placeholder interfaces remain for future drop-in.

* **C#8 – Direct SAS Transfers** (Client Clarification #9, L2-LLD-IC §4.3)  
  Uploads/downloads use single-object SAS URLs with 15-minute TTL. The APP retries once on `403 Signature expired` by requesting a fresh SAS, then surfaces an error toast on failure.

* **L2-IC-1 – Uniform API Contract** (L2-LLD-IC §4 & §6)  
  All calls are JSON/HTTPS using TLS 1.2 under `/v1`, include `Authorization: Bearer <jwt>` and, when relevant, `X-Store-Id`. Responses must carry `meta.traceId`; missing IDs trigger a client-side warning log.

* **L2-IC-2 – Error Envelope Handling** (L2-LLD-IC §6)  
  Error handling follows L3-WF-APP §12.1 HTTP Status Matrix for domain-specific message mapping and fallback behaviors.

* **L1-KD-1 – Target Platforms & Build Chain** (L1-TSD §2 & §6)  
  Flutter 3.2.x (Dart 3.2.x) is the locked runtime. GitHub Actions + Fastlane produce signed `.aab`/`.ipa` artifacts targeting Android ≥ 8.0 and iOS ≥ 13.

* **Security-1 – Local Storage Rules** (L2-LLD-IC §10)  
  Mobile stores JWT and refresh token in secure local storage; Authorization header on calls. JWTs are RS256-signed, refresh token and transient idempotency keys are stored exclusively in the secure platform keystore/keychain. No credentials or PII are written to the file system or SharedPreferences.

* **Performance-1 – UI Responsiveness** (L1-NFRS §2, forthcoming)  
  Network calls execute off the main isolate. List builders use paging to maintain responsive UI as per L1-NFRS §2 guidelines (P95 ≤800 ms for APIs, ~2s for screen renders). Shimmer skeletons replace blocking spinners.

* **OTP-1 – Login Authentication** (L1-OVERVIEW §3.4, L3-WF-APP §5.1)  
  For login, Store Users and Parent Store Users enter phone number and receive both SMS and Email OTP via MSG91. 

* **OTP-2 – Customer Verification** (L1-OVERVIEW §3.4, L3-WF-APP §7.3)  
  During promotion redemption, customer verification OTP supports both SMS and Email channels based on promotion requirements. The APP provides channel selection (phone/email entry) when initiating customer OTP.

* **OTP-3 – Lockout Policy** (L1-NFRS §4, L2-LLD-IC §10)  
  Backend enforces OTP lockout after 5 failed attempts within 10 minutes, with a 15-minute lockout window. The APP displays appropriate error messages when lockout is triggered.

* **TZ-1 – Time Zone Handling** (L1-FRS)  
  All timestamps are stored and displayed using IST (Indian Standard Time) as the default timezone. No per-store timezone configuration is supported.

* **Data-1 – Reconciliation Uniqueness** (L1-FRS, L3-WF-APP §9)  
  During reconciliation, the combination of (store_id, PID number, invoice_no) must be unique. The APP enforces this at the UI level during data entry.