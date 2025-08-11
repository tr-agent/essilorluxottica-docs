## L3-LLD-APP: Component-Specific Low-Level Design Document for APP Document
Essilor-Luxottica • PromoPartner • Mobile Component  

---

### 1. Component Context  
#### 1.1 Position in Overall Architecture  
The APP is the only end-user native client. It consumes the Backend REST API and, when instructed by the API, uploads or downloads files directly from Azure Blob via Shared-Access-Signature (SAS) URLs. All other integrations are transparently handled by the Backend.

```
**Architecture Overview**

```
Mobile Device
    ┌─────────────────────────────────┐
    │        Flutter APP              │
    │      iOS • Android              │
    └─────────────────────────────────┘
                │
                ├─ HTTPS/JSON ──→ BCKND /v1 REST
                ├─ HTTPS (SAS) ──→ Azure Blob Storage
                │
                ┌─────────────────────────────────┐
                │           Backend               │
                │  ┌─────────────────────────────┐│
                │  │    Azure PostgreSQL         ││
                │  └─────────────────────────────┘│
                │  ┌─────────────────────────────┐│
                │  │    Azure Queue              ││
                │  └─────────────────────────────┘│
                │  ┌─────────────────────────────┐│
                │  │    MSG91 SMS/E-mail        ││
                │  └─────────────────────────────┘│
                └─────────────────────────────────┘
```
```

#### 1.2 External Interaction Points  
• Synchronous: `/v1/*` REST endpoints (Auth, Promotions, Transactions, Products, Stores).  
• Asynchronous: none (queue is backend-only).  
• File Transfer: time-limited SAS URLs issued by `/v1/*/presign`.

---

### 2. Detailed Component Design  

#### 2.1 Architectural Overview  
The APP adopts a thin-client, **online-only** architecture optimised for ≤1 000 concurrent sessions and cold-start ≤3 s. When network connectivity is lost, the UI displays a blocking "Offline" banner and disables all write actions until the network is restored (L3-KD-APP C#1).

```
**Architecture Class Diagram**

```
┌─────────────────────────────────┐
│         Presentation            │
├─────────────────────────────────┤
│ + Widgets                      │
│ + Theming                      │
└─────────────────┬───────────────┘
                  │
                  ▼
┌─────────────────────────────────┐
│            State                │
├─────────────────────────────────┤
│ + Cubit<State>                 │
│ + Bloc<Event,State>            │
└─────────────────┬───────────────┘
                  │
                  ▼
┌─────────────────────────────────┐
│           Services              │
├─────────────────────────────────┤
│ + DioClient                    │
│ + AuthService                  │
│ + ProductService               │
│ + TransactionService           │
└─────────────────┬───────────────┘
                  │
                  ▼
┌─────────────────────────────────┐
│        SecureStorage            │
├─────────────────────────────────┤
│ + flutter_secure_storage       │
└─────────────────────────────────┘
```
```


```
**Services Class Diagram**

```
┌─────────────────────────────────┐
│         AuthService             │
├─────────────────────────────────┤
│ - DioClient client              │
│ - SecureStorage storage         │
├─────────────────────────────────┤
│ + requestOTP(phone, email?)    │
│ + verifyOTP(code: String)      │
│ + refreshToken()                │
│ + logout()                      │
└─────────────────┬───────────────┘
                  │
                  ├─→ DioClient
                  └─→ SecureStorage

┌─────────────────────────────────┐
│      TransactionService         │
├─────────────────────────────────┤
│ - DioClient client              │
│ - String requestUuid            │
├─────────────────────────────────┤
│ + evaluatePromotions(cart, id)  │
│ + createTransaction(...)        │
│ + listTransactions(...)         │
│ + updateReconciliation(...)     │
└─────────────────┬───────────────┘
                  │
                  └─→ DioClient

┌─────────────────────────────────┐
│         ProductService          │
├─────────────────────────────────┤
│ - DioClient client              │
├─────────────────────────────────┤
│ + searchProducts(type, query)   │
│ + getProductDetails(sku)        │
└─────────────────┬───────────────┘
                  │
                  └─→ DioClient
```
```


Layer responsibilities  
• Presentation – Flutter widgets, declarative UI, responsiveness.  
• State – `flutter_bloc 8.x` with Cubit for simple state, Bloc for multi-step flows following Clean Architecture (L3-KD-APP C#5).  
• Repository – Abstraction layer between State and Services, manages data sources and caching.
• Services – pure-Dart classes that wrap REST calls and parse DTOs.  
• SecureStorage – OS-level keystore for JWT and refresh token storage (L3-KD-APP Security-1).

#### 2.2 Module Breakdown  

| Module | Folder | Responsibility | Key Classes |
|--------|--------|----------------|-------------|
| `core` | `/lib/core` | API client, env config, error types, interceptors | `DioClient`, `ApiException`, `SASRetryInterceptor` |
| `auth` | `/lib/features/auth` | OTP login, token refresh, role guard | `AuthCubit`, `AuthService` |
| `home` | `/lib/features/home` | Landing widgets, promotion banners, Offers Tab filter UI | `HomeCubit`, `OffersCubit` |
| `transactions` | `/lib/features/transactions` | Cart build, submit, list, idempotency | `CartBloc`, `TransactionService` |
| `profile` | `/lib/features/profile` | Store switch, logout | `ProfileCubit` |
| `widgets` | `/lib/widgets` | Reusable UI (buttons, dialogs) | – |
| `routing` | `/lib/routes` | `go_router` setup w/ bottom-nav | `AppRouter` |
| `reconciliation` | `/lib/features/reconciliation` | View reconciliation status, update invoice/PID | `ReconciliationCubit` |

#### 2.3 Navigation & Screen Inventory  

Bottom navigation with 5 root routes – **Home**, **New Sale**, **History**, **Reconcile**, **Profile**. The Home tab includes an internal sub-tab or toggle to switch between the main feed view and the Offers Tab filter UI as specified in L1-SI §2.2. Inside each, stack navigation is used for multi-step flows.

```
**Navigation State Diagram**

```
                    [*]
                     │
                     ▼
                   Splash
                     │
            ┌────────┴────────┐
            │                 │
      not authenticated   has JWT
            │                 │
            ▼                 ▼
          Login             Home
            │                 │
            │ OTP verified    │
            └─────┬───────────┘
                  │
                  ▼
                 Home
                  │
                  ▼
              ┌─────────────────────────────────┐
              │           Tab Bar               │
              │  ┌─────┐  ┌─────┐  ┌─────┐    │
              │  │Home │◄─│Profile│◄─│Rec. │    │
              │  └──┬──┘  └─────┘  └──┬──┘    │
              │     │                 │        │
              │     ▼                 │        │
              │  ┌─────┐  ┌─────┐    │        │
              │  │Tx.  │─►│Hist.│───►┘        │
              │  └─────┘  └─────┘             │
              └─────────────────────────────────┘
                  │
                  ▼
                NewSale
                  │
                  ▼
             ProductSelect
                  │
                  ▼
              OfferSelect
                  │
                  ▼
             CustomerDetails
                  │
                  ▼
             CustomerVerify
                  │
                  ▼
                Success
```
```

Comprehensive screen list & primary endpoints

| Screen | Purpose | Main Endpoint | Auth |
|--------|---------|---------------|------|
| Splash | Token refresh | `/v1/auth/refresh` | optional |
| Login (OTP) | Request / verify SMS or Email OTP | `/v1/auth/otp/*` | none |
| Home | Show active promotions, toggle to Offers Tab | `/v1/promotions?status=active` | Bearer |
| Offers Tab | Filter promotions by criteria | `/v1/promotions?status=active&filters=...` | Bearer |
| Product Selection | Browse/search products via modal | `/v1/products` | Bearer |
| Offer Selection | Show applicable offers for cart | `/v1/promotions/evaluate` | Bearer |
| Customer Details | Capture customer info | - | Bearer |
| Customer Verify | Customer OTP (SMS/Email choice) | `/v1/customer-otp/*` | Bearer |
| Submit Tx | Finalise sale | `/v1/transactions` | Bearer |
| Tx List | Show past tx | `/v1/transactions` | Bearer |
| Reconciliation | View/update reconciliation status | `/v1/transactions` (GET), `/v1/transactions/{id}` (PATCH) | Bearer + X-Store-Id |
| Profile | Store switch, logout | N/A | Bearer |

*Note: For Parent Store Users, the `X-Store-Id` header is automatically injected by AuthInterceptor based on the current store context selected in Profile screen.*

#### 2.4 Network Layer & Token Handling  

*HTTP client* — `dio ^5` with three interceptors:

1. **AuthInterceptor**  
   • Adds `Authorization: Bearer <jwt>` header from secure storage.  
   • On 401, pauses outgoing queue, calls `/v1/auth/refresh`, retries original call once.  
2. **ErrorMapperInterceptor** – converts HTTP errors to typed `ApiException`.
3. **SASRetryInterceptor** – On `403 Signature expired` for Blob operations, requests fresh SAS URL and retries once (L3-KD-APP C#9).

```
**Token Refresh Sequence Diagram**

```
UI          Cubit/Bloc    DioClient    API         SecureStore
│               │            │          │              │
│  action()     │            │          │              │
│ ─────────────►│            │          │              │
│               │ /v1/resource│          │              │
│               │ ──────────►│          │              │
│               │            │ GET (jwt) │              │
│               │            │ ────────►│              │
│               │            │          401            │
│               │            │ ◄────────│              │
│               │            │ read refreshToken       │
│               │            │ ───────────────────────►│
│               │            │ POST /auth/refresh      │
│               │            │ ────────►│              │
│               │            │ 200 new jwt/refresh     │
│               │            │ ◄────────│              │
│               │            │ replay GET              │
│               │            │ ────────►│              │
│               │            │ 200 data                │
│               │            │ ◄────────│              │
│               │            │ data                    │
│               │ ◄─────────│          │              │
│               │            │          │              │
```
```

Token storage keys  

| Key              | Store                         |   Lifetime   |
|------------------|-------------------------------|--------------|
| `refreshToken`   | Secure Storage                |   12 h       |
| `jwt`            | Secure Storage                |   ≤60 min    |

#### 2.5 Token-Based Pagination Helper
All GET list calls accept `/v1/*?limit=N&nextToken=…`. `ListResponse<T>` model exposes `items` & `nextToken`. Blocs/Cubits keep last `nextToken` per list and emit `Fetching`, `Success`, `EndOfList`.

#### 2.6 Cart & Promotion Evaluation Algorithm Flow  

The cart maintains transaction idempotency using a `request_uuid` stored in RAM throughout the session. On the first `/v1/transactions` call, the Backend returns `201 Created` with transaction details. The APP retains this UUID and re-uses it on every retry until a terminal response is received (L3-KD-APP C#4).

```
**Cart Evaluation Flow**

```
                    Start
                      │
                      ▼
                  Select SKU
                      │
                      ▼
                Build cart array
                      │
                      ▼
            call /promotions/evaluate
                      │
                      ▼
              ┌─────────────────┐
              │ eligible == 0 ? │
              └─────────┬───────┘
                        │
            ┌───────────┴───────────┐
            │                       │
          Yes                     No
            │                       │
            ▼                       ▼
      Disable Checkout      Show discount + CTA
            │                       │
            │                       ▼
            │                 Generate request_uuid
            │                       │
            │                       ▼
            │              POST /transactions with UUID
            │                       │
            │                       ▼
            │                Navigate to Tx Detail
            │                       │
            └───────────────────────┘
                        │
                        ▼
                      Stop
```
```

#### 2.7 Multi-Step Sale Flow Implementation

The New Sale feature implements a 5-step wizard pattern managed by `SaleBloc` with the following states, mapping to the 4-step flow in L3-WF-APP §7 where Customer Details is a sub-step of the Customer Verification process:
**State Classes:**
- `SaleState` - Base state containing cart items, selected promotion, customer details, and current step
- `ProductSelectionState`, `OfferSelectionState`, `CustomerDetailsState`, `VerificationState`, `SuccessState` - Step-specific states
- `SaleEvent` - Base event class with subclasses: `ProductsSelected`, `OfferSelected`, `CustomerEntered`, `OTPVerified`, `TransactionCompleted`
```
**Sale Flow State Diagram**

```
                    [*]
                     │
                     ▼
              ProductSelection
                     │
              Products selected
                     │
                     ▼
               OfferSelection
                     │
               Offer selected
                     │
                     ▼
              CustomerDetails
                     │
              Details entered
                     │
                     ▼
            CustomerVerification
                     │
               Code verified
                     │
                     ▼
            TransactionSuccess
                     │
                     ▼
                    [*]
```
```

Each step maintains local state in the Bloc while progressing through the flow:
- **Step 1**: Product selection via modal overlays (L3-WF-APP §7.1)
- **Step 2**: Automatic offer calculation based on cart (L3-WF-APP §7.2)
- **Step 3**: Customer information capture with new customer detection (L3-WF-APP §7.3 - contact collection)
- **Step 4**: 6-digit OTP verification (SMS or Email based on customer preference) (L3-WF-APP §7.3 - verification)
- **Step 5**: Transaction confirmation with price breakdown (L3-WF-APP §7.4)

#### 2.8 Performance Optimisation Techniques
1. Split modules into deferred components (Login, Transactions) to reduce APK initial download size.  
2. Tree-shaking and `--split-debug-info` for release builds.  
3. Cached network images for banner assets (max 5 MB memory).  
4. `WidgetsBinding.instance.deferFirstFrame()` + shimmer skeletons maintain P95 ≤800 ms for APIs and ~2s for screen renders (L3-KD-APP Performance-1).
5. Network calls execute off the main isolate to maintain UI responsiveness.

#### 2.9 Unit & Widget Testing Plan

| Layer | Tool | Coverage Target |
|-------|------|-----------------|
| Blocs/Cubits | `flutter_test` + `mocktail` | ≥90 % |
| Services  | `http_mock_adapter` | ≥80 % |
| Routing   | `golden_toolkit` | All root screens |
| E2E (smoke) | `flutter_driver` (soon: `integration_test`) | Happy paths: login, cart, tx |
| Crash Recovery | `integration_test` | Online-only state restoration |

---

### 3. UI / UX Reference  

#### 3.1 Global Layout  
The app follows a consistent layout structure across all screens with common UI elements to ensure seamless navigation and brand consistency. Maximum viewport width is constrained to 430px for optimal mobile viewing.

```
**Layout Structure**

```
┌─────────────────────────────────────────────────┐
│              AppBar - Sticky Header             │
└─────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│           RouterOutlet - Scrollable Content     │
│                                                 │
│  ┌─────────────────────────────────────────────┐ │
│  │                                           │ │
│  │              Screen Content                │ │
│  │                                           │ │
│  └─────────────────────────────────────────────┘ │
│                                                 │
└─────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│              BottomNav – 5 tabs                │
│  [Home] [New Sale] [History] [Reconcile] [Profile] │
└─────────────────────────────────────────────────┘
```
```

Theme: 
- Primary: `#667eea` (purple/indigo)
- Success: `#27ae60` (green)  
- Error: `#e74c3c` (red)
- Text Primary: `#2c3e50`
- Text Secondary: `#7f8c8d`
- Background: `#f5f7fa`
- Card Background: `#ffffff` with box-shadow
- WCAG-AA contrast compliance
- Font scaling via `MediaQuery.textScaleFactor`

#### 3.2 Responsive Behaviour  
• Phones portrait – default.  
• Tablets landscape – 2-pane list/detail in Transactions tab. Breakpoint at 600 dp.
• Maximum content width: 430px centered on larger screens

#### 3.3 Screen-specific Layouts  

**Login Screen**  
- Centered login box with gradient background (`#667eea` to `#764ba2`)
- Company branding: "Essilor-Luxottica" title with "Store Staff App" subtitle
- One input field: phone number (10 digits)
- "Send OTP" primary button
- Terms & Conditions link at bottom

**OTP Verification Screen**  
- 6 separate input boxes for OTP digits (auto-advance on input)
- Display phone number or email address for confirmation
- "Verify & Continue" primary button
- Back navigation to login

**Home Screen (Available Offers)**  
- Sticky header with user greeting
- Toggle button to switch between "Home Feed" and "Offers Tab" views
- Search bar with icon for offer filtering
- Promotion cards with:
  - Title and validity dates
  - Type badge (BOGO, 20% OFF, BUNDLE)
  - Description text
  - "Apply Offer" action button
  - Left border accent (4px solid primary color)

**Offers Tab View**
- Filter chips for promotion types
- Date range selector
- Store group filter dropdown
- Filtered promotion cards in grid layout


**New Sale Flow (5 Steps)**  
1. **Product Selection**
   - Card-based product selectors (Frame, Lens, Add-ons)
   - Icons for each category
   - Selected state with blue background
   - Modal overlay for product search/selection

2. **Offer Selection**  
   - Grid of available offers
   - Savings amount highlighted
   - Selected state with border and background

3. **Customer Details**
   - Name and phone input fields
   - New customer badge (green) when detected
   - Selected offer summary at top

4. **Customer Verification**
   - 6-digit OTP input boxes
   - "Generate New Code" option
- 5-minute countdown timer display

5. **Transaction Success**
   - Checkmark icon animation
   - Price breakdown summary
   - Discount line item (red)
   - Total amount (green, bold)
   - Action buttons for new sale or home

**Transaction History**  
- Filter tabs (Today, This Week, This Month)
- Stats cards grid (Transaction Count, Total Sales)
- Transaction cards with:
  - Amount (green, large font)
  - Date/time stamp
  - Promotion name
  - Customer details

**Reconciliation Screen**  
- Filter tabs (Pending, Completed, All)
- Reconciliation items with:
  - Transaction ID and amount
  - Status badge (yellow for pending, green for completed)
  - Input fields for invoice and product IDs
  - Validation indicator for unique (store_id, PID, invoice_no) combination
  - Save & Submit button

**Profile Screen**  
- Avatar circle with initials
- User info section
- Store information section
- Performance metrics (This Month)
- Logout button

#### 3.4 User Interaction Flows

```
**User Login & Sale Flow**

```
                    Start
                      │
                      ▼
                  Login Screen
                      │
                      ▼
                Phone/Email Entry
                      │
                      ▼
            Request SMS/Email OTP
                      │
                      ▼
                Enter 6-Digit OTP
                      │
                      ▼
              ┌─────────────────┐
              │     Verify?     │
              └─────────┬───────┘
                        │
            ┌───────────┴───────────┐
            │                       │
          Yes                     No
            │                       │
            ▼                       ▼
          Home Screen        Return to Login
            │
            ▼
          Tap New Sale
            │
            ▼
        Select Products via Modal
            │
            ▼
        View Available Offers
            │
            ▼
          Select Best Offer
            │
            ▼
        Enter Customer Details
            │
            ▼
      Customer Verification Choice
            │
            ▼
        ┌─────────────────────────┐
        │      SMS/Email?         │
        └─────────┬───────────────┘
                  │
        ┌─────────┴───────────────┐
        │                         │
      SMS                     Email
        │                         │
        ▼                         ▼
    Enter Customer OTP      Enter Customer OTP
        │                         │
        └─────────┬───────────────┘
                  │
                  ▼
            Transaction Success
                  │
                  ▼
              View in History
                  │
                  ▼
                  Stop
```
```


```
**Reconciliation Flow**

```
                    Start
                      │
                      ▼
                Reconciliation Tab
                      │
                      ▼
            View Pending Transactions
                      │
                      ▼
                Select Transaction
                      │
                      ▼
                Enter Invoice/PID
                      │
                      ▼
              ┌─────────────────────┐
              │ Validate Uniqueness │
              └─────────┬───────────┘
                        │
            ┌───────────┴───────────┐
            │                       │
          Valid                 Duplicate
            │                       │
            ▼                       ▼
    PATCH /v1/transactions/{id}  Show Error
            │
            ▼
    Update Status to UNDER_REVIEW
            │
            ▼
            Stop
```
```


#### 3.5 Component Design System

**Input Components**
- Text inputs: 12px padding, 1px border (#ddd), 8px border-radius
- Focus state: border-color #667eea
- OTP inputs: 50x50px boxes, center-aligned text

**Button Components**
- Primary: #667eea background, white text, 14px padding
- Secondary: #ecf0f1 background, #34495e text, 8px padding  
- Outline: transparent background, #667eea border and text
- Success: #27ae60 background, white text

**Card Components**
- White background (#ffffff)
- 8-12px border-radius
- Box-shadow: 0 2px 8px rgba(0,0,0,0.1)
- 16px padding

**Badge Components**
- Small pills with colored backgrounds
- 4px horizontal, 2px vertical padding
- 4px border-radius
- 11px font size

**Navigation Components**
- Bottom nav: Fixed position, white background
- Nav items: Icon (20px) + Text (11px)
- Active state: #667eea color
- 5 equal-width tabs

#### 3.6 Endpoint Associations and Business Rules

**Login Screen**  
- Associated Endpoints: `/v1/auth/otp/request`, `/v1/auth/otp/verify`
- Data Requirements: Phone number (10 digits, Indian format), optional email address
- Validations: Phone regex `^[6-9]\d{9}$`, email RFC 5322 compliance
- Business Rules: Phone number used as identifier. Same OTP sent via SMS (always) and Email (if registered). User can verify using OTP from either channel. 5 failed attempts trigger 15-min lockout

**Product Selection Modal**
- Associated Endpoints: `/v1/products?type={frame|lens|addon}&search={query}`
- Data Requirements: Product type, optional search query
- Validations: At least one frame and one lens required
- Business Rules: Server-side product filtering, paginated results (50 items max)

**Offer Evaluation Screen**
- Associated Endpoints: `/v1/promotions/evaluate`
- Data Requirements: Cart items array, store_id
- Validations: Valid product SKUs, quantities > 0
- Business Rules: Only active promotions shown, best offer auto-highlighted

**Customer Verification**
- Associated Endpoints: `/v1/customer-otp/request`, `/v1/customer-otp/verify`
- Data Requirements: Customer phone/email, 6-digit OTP
- Validations: Valid phone/email format, OTP exactly 6 digits
- Business Rules: 5-minute OTP validity, max 5 attempts

**Transaction Submit**
- Associated Endpoints: `/v1/transactions`
- Data Requirements: Cart, promotion_id, eligibility_token, request_uuid
- Validations: Valid eligibility token, unique request_uuid
- Business Rules: Idempotent submission, status transitions managed by backend

**Reconciliation Update**
- Associated Endpoints: `/v1/transactions` (GET), `/v1/transactions/{id}` (PATCH)
- Data Requirements: Transaction ID, invoice number, PID number
- Validations: Unique combination of (store_id, PID, invoice_no)
- Business Rules: Status auto-transitions to UNDER_REVIEW on first update


---

### 4. Minimal API Specification (consumed by APP)  

**Request Login OTP**  
Purpose: Initiate SMS or Email OTP for Store/Parent User authentication. Endpoint: `/v1/auth/otp/request`. Authorization Required: None. Methods: POST.  
Accepts phone number in request body. System looks up user's registered email (if any) and sends same OTP via both SMS and Email channels through MSG91. Returns 200 on success with OTP validity period.

**Verify OTP & Issue Tokens**  
Purpose: Validate OTP code and issue JWT/refresh tokens. Endpoint: `/v1/auth/otp/verify`. Authorization Required: None. Methods: POST.  
Verifies 6-digit OTP against database, returns JWT (RS256) and refresh token on success. Enforces lockout after 5 failed attempts.

**Refresh Tokens**  
Purpose: Renew expired JWT using valid refresh token. Endpoint: `/v1/auth/refresh`. Authorization Required: Optional (expired JWT accepted). Methods: POST.  
Exchanges refresh token for new JWT/refresh pair. Called silently 5 minutes before JWT expiry.

**Fetch Active Promotions**  
Purpose: Retrieve currently active promotions for display. Endpoint: `/v1/promotions`. Authorization Required: Bearer JWT with role P or S. Methods: GET.  
Returns paginated list of active promotions with banners, rules, and validity periods. Supports query filters for status and store.

**Search Products**
Purpose: Search and filter products by type and query. Endpoint: `/v1/products`. Authorization Required: Bearer JWT with role P or S. Methods: GET.
Accepts type (frame/lens/addon) and search query parameters. Returns paginated product list with SKU, name, brand, and price.

**Evaluate Cart**  
Purpose: Calculate applicable promotions and discounts for cart items. Endpoint: `/v1/promotions/evaluate`. Authorization Required: Bearer JWT with role P or S. Methods: POST.  
Accepts cart items and store_id, returns eligible promotions with calculated discounts. Uses server-side product validation.

**Request Customer OTP**  
Purpose: Initiate customer verification OTP for promotion redemption. Endpoint: `/v1/customer-otp/request`. Authorization Required: Bearer JWT with role P or S. Methods: POST.  
Supports both SMS and Email channels via MSG91. Returns 200 with 5-minute validity period.

**Verify Customer OTP**  
Purpose: Validate customer OTP and generate eligibility token. Endpoint: `/v1/customer-otp/verify`. Authorization Required: Bearer JWT with role P or S. Methods: POST.  
Verifies OTP and returns eligibility token for transaction submission. Token valid for single transaction.

**Create Transaction**  
Purpose: Submit promotion transaction with idempotency. Endpoint: `/v1/transactions`. Authorization Required: Bearer JWT with role P or S. Methods: POST.  
Accepts cart, promotion_id, eligibility_token, and request_uuid. Returns 201 on first success, idempotent for retries.

**List Transactions**  
Purpose: Retrieve transaction history with pagination. Endpoint: `/v1/transactions`. Authorization Required: Bearer JWT with role P or S. Methods: GET.  
Returns paginated transaction list filtered by store and date range. Supports cursor-based pagination via nextToken.

**Update Transaction Reconciliation**
Purpose: Update reconciliation fields for a transaction. Endpoint: `/v1/transactions/{id}`. Authorization Required: Bearer JWT with role P or S. Methods: PATCH.
Accepts invoice_no and pid_no in request body. Validates uniqueness of (store_id, invoice_no, pid_no) combination. Auto-transitions status to UNDER_REVIEW on first update.


**Get Stores**  
Purpose: Retrieve authorized stores for switching context. Endpoint: `/v1/stores`. Authorization Required: Bearer JWT with role P. Methods: GET.  
Returns list of child stores for Parent Users. Used in Profile screen for store context switching.

**Get SAS URL for Upload**
Purpose: Generate time-limited SAS URL for direct file upload to Azure Blob. Endpoint: `/v1/csv-jobs/{entity}/presign`. Authorization Required: Bearer JWT with role A or P. Methods: POST.
Accepts file name and size in request body. Returns SAS URL valid for 15 minutes for direct blob upload.

**Register Uploaded File**
Purpose: Register previously uploaded file with backend for processing. Endpoint: `/v1/csv-jobs`. Authorization Required: Bearer JWT with role A or P. Methods: POST.
Accepts entity type and blob URL. Triggers backend processing of uploaded CSV file.


---

### 5. Security Design  

| Measure | Implementation |
|---------|----------------|
| Transport | TLS 1.2 for all public endpoints (App Service); no certificate pinning |
| Storage | JWT and refresh token stored in secure local storage (flutter_secure_storage) with AES-256 encryption, passed via Authorization headers; falls back to encrypted SharedPreferences if keychain/keystore unavailable; no plain text storage |
| Auth Flow | Both SMS and Email OTP login for Store/Parent Users, dual-channel (SMS/Email) for customer verification via MSG91 |
| Min OS | Android 8.0 (API 28), iOS 13 per L1-KD-1 |
| Force-upgrade | On every API response, `X-Min-App-Version` header triggers blocking modal when unmet |
| Accessibility | WCAG 2.1 AA labels, font scaling |
| OTP Lockout | Backend enforces lockout after 5 failed attempts within 10 minutes with 15-minute lockout window |
| OTP Storage | OTP codes stored as plaintext without encryption in PostgreSQL (accepted risk), protected only by database-level encryption |
| JWT Signing | JWT signed with RS256 algorithm |

```
**Security Authentication Flow**

```
User          APP           API
 │            │             │
 │ enters     │             │
 │ phone/email│             │
 │ ──────────►│             │
 │            │ /auth/otp/request
 │            │ ──────────►│
 │            │             │ Store plaintext OTP
 │            │             │ in PostgreSQL
 │            │             │
 │            │             │
 │            │             │
 │ /auth/otp/verify        │
 │ ───────────────────────►│
 │            │             │
 │            │ JWT (RS256) + refresh
 │            │ ◄──────────│
 │            │             │
```
```

---

### 6. Error Handling & Logging  

Error handling follows a fail-fast strategy with clear user messaging. The backend error envelope (L2-LLD-IC §6) is parsed to provide context-specific responses to users.

* Strategy: Fail-fast, surfacing backend error envelope (§L2 §6).  
* Mapping table lives in `core/error_mapper.dart`.  
* Client-side logs: `logger` → debug console; no crash reporting SDK in phase 1.  
* User messages: toast for non-blocking, blocking dialog for auth errors.
* OTP Lockout: Display "Too many attempts. Please try again after 15 minutes" when `ACCOUNT_LOCKED` error received.
* Network errors: Show blocking overlay with "No Internet Connection" message and retry button

```
**Error Handling Flow**

```
                    Start
                      │
                      ▼
                    APIError
                      │
                      ▼
                  ErrorMapper
                      │
                      ▼
                  Bloc/Cubit
                      │
                      ▼
                    UIToast
                      │
                      ▼
                     Stop
```
```

**Error Message Mapping**

| Error Code | User Message | UI Treatment |
|------------|--------------|--------------|
| `UNAUTHENTICATED` | "Session expired. Please login again." | Navigate to login |
| `RATE_LIMIT_EXCEEDED` | "Too many requests. Please wait." | Show toast, disable action |
| `ACCOUNT_LOCKED` | "Account locked for 15 minutes." | Show blocking dialog |
| `NETWORK_ERROR` | "Connection failed. Check internet." | Show retry overlay |
| `INVALID_OTP` | "Incorrect code. Please try again." | Clear input, show error |
| `FIELD_REQUIRED` | "Please fill all required fields." | Show inline validation |
| `INVALID_FORMAT` | "Invalid format. Please check your input." | Show inline validation |
| `FORBIDDEN` | "You don't have permission for this action." | Show toast |
| `STATUS_TRANSITION_INVALID` | "This action cannot be performed now." | Show toast |
| `RESOURCE_NOT_FOUND` | "Item not found." | Navigate back |
| `UNEXPECTED_ERROR` | "Something went wrong. Please try again." | Show dialog |

---

### 7. Deployment & Environment Configuration  

| Build Flavour | API Base URL | Crashlytics | Analytics |
|---------------|-------------|-------------|-----------|
| `prod` | `https://api.promopartner.in` | off | off |

CI pipeline (GitHub Actions) — `flutter test` → `flutter build appbundle --dart-define=FLAVOR=prod` → Upload to App Store / Play.

**Platform-Specific Configuration**

Android (`android/app/src/main/AndroidManifest.xml`):
- Permissions: INTERNET, USE_BIOMETRIC
- Min SDK: 28 (Android 8.0)
- Target SDK: 33 (Android 13)
- ProGuard rules for Flutter and dependencies

iOS (`ios/Runner/Info.plist`):
- Minimum iOS: 13.0
- Capabilities: Keychain Sharing
- NSAppTransportSecurity: Allow arbitrary loads = NO
- Camera usage description (future QR scanning)

---

### 8. Documentation & Coding Standards  
• Effective Dart style guide.  
• All public classes documented with `///`.  
• One widget per file.  
• File names `snake_case.dart`.  
• BLoC pattern naming: `*_bloc.dart`, `*_event.dart`, `*_state.dart`.
• Test files mirror source structure with `_test.dart` suffix
• Widget keys for testing: `Key('login_phone_input')`

---

### 9. Compliance & Regulatory  
The APP never stores PII beyond transient RAM and OS keystore; aligns with indefinite server-side retention policy (L1-HLD §5). No local backups are taken. The platform relies solely on Azure Database for PostgreSQL's built-in automated daily backups (7-day PITR) but provides no Disaster Recovery capability.

---

### 10. Internationalisation / Localisation  
Phase-1 is **English-only**; strings in `lib/l10n/intl_en.arb` to allow future expansion. Widgets wrap text with `Intl.message()`.

---

### 11. Cross-Component Interface Contract (Excerpt)  
The APP maintains strict contract adherence with the Backend API to ensure reliable communication. All requests include required headers and follow established patterns for data exchange and error handling.

* Request header `X-Client-Platform: ios|android` mandatory.  
* JWT must include `role` claim; API responds 403 when role ∉ {P,S}.  
* Store context via `X-Store-Id` header for Parent Users switching stores.
* Cursor pagination contract:  
  ```json
  {
    "data": { "items": [...], "nextToken": "abc123" },
    "meta": { "traceId": "…" }
  }
  ```  
* Versioning: breaking backend change ⇒ `/v2`; mobile app expected to consume `/v1` for ≥12 months.

---

### 12. Inter-Component Communication Standards  
Communication standards ensure consistent integration with the Backend while maintaining performance and security requirements appropriate for the ≤5,000 user scale.

• Rate-limit: API enforces 60 req/min per JWT; error code `RATE_LIMIT_EXCEEDED`.  
• All requests must carry `Accept: application/json`.  
• Timeout: 10 s connect, 30 s read; set in `DioClient`.
• SAS URL operations: 15-minute TTL, single retry on expiry.
• All timestamps displayed in IST (Indian Standard Time) per L3-KD-APP TZ-1.
• Request size limit: 1 MB for JSON payloads
• Response compression: gzip accepted via Accept-Encoding header
• CSV Upload Flow: Backend issues SAS URL, client uploads file using SAS, client calls backend to register the uploaded file.
• Keys stored in encrypted App-Service settings without Azure Key Vault.

---

### 13. Distributed Operations Requirements  
State management follows the online-only architecture with server-side persistence. The APP maintains minimal transient state to support the current user session.

State is exclusively server-side; after app crash user restarts flow. Transaction idempotency via `request_uuid` ensures no duplicate submissions even after crashes. 
Monitoring via App Store / Play dashboards only; no remote feature flags or telemetry SDKs in phase 1. No push notifications (FCM/APNS) are implemented.

**Session Recovery Patterns**
- JWT persisted in secure storage survives app restart
- Cart state lost on crash (acceptable for online-only model)
- Transaction UUID retained in memory only during active session
- Store context restored from JWT claims on app launch

---

_End of document_