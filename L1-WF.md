# Essilor-Luxottica Joint Promotions Management Solution- PromoPartner

## Level 1 Workflow Document

#1. SOLUTION OVERVIEW

The Essilor-Luxottica Joint Promotions Management Solution(PromoPartner) is designed to operate trade promotions from Essilor-Luxottica(EL)  head office to end customers at optical stores. The solution manages various promotion types including Value Upgrade, BOGO, Flat Discount, and Percentage Discounts.

## 1.1 Solution Components

- Backend (BCKND) - Core processing engine**
- Web Portal (WEBPRT) - Web-based management interface for Admin and KAM users.
- Mobile App (APP) - Store User and parent store user interface

## 1.2 Core Entities

- Store - Optical retail locations
- Frames - Frame product catalog
- Lenses - Lens product catalog
- Transactions - Promotion transaction records
- Users - System users (Admin, KAMs,  Store Users, Parent Store Users)
- Promotions – Promotions that will be offered on stores for end customers.

**Building Blocks – Product blocks ( frame {Field} ,  lens{Field})  etc that will be used to build conditional logics for promotions. SKU list within a set of predefined blocks.**
- Product groups – Custom built  building blocks with respect to products – frames and lens.
- Store groups  - Custom built building blocks with respects to store.

**Solution Scale :**
- Up to 5000 mobile Application users. (iOS and android)
- Daily 10 -20 transactions per user
- Approx. 20 promotions active at a time
- Lens and frames data up to 1,00,000 rows
- Single English language
- Up to 5000 stores


# 2. USER PERSONAS & ACCESS RIGHTS

## 2.1 Admin User

**Platform: Web Portal**

**Primary Responsibilities:**
- Create and manage product groups and retailer groups
- Create, activate, and deactivate promotions
- Manage master data (stores, users, frames, lenses)
- Monitor transactions and approve reconciliations
- Generate BASIC reports 

**Access Rights:**
- Full system access
- Create/edit/delete all entities
- Approve transactions
- User management
- System settings

## 2.2 KAMs User

**Platform: Web Portal**

**Primary Responsibilities:**
- KAMs are mapped to Parent Stores and have read only access for the mapped parent store.
- Views all things as Admin but only with Read Only Access.

**Access Rights:**
- Read Only access to all features
- No access to any PUT, PATCH, DELETE endpoints
- Can only view data, cannot create, edit, or delete any entities


## 2.3 Store User

**Platform: Mobile App**

**Primary Responsibilities:**
- Capture customer details and process promotion transactions
- Apply offers via customer phone verification with OTP
- Add invoice number, PID number to submit proof for Reconcile request.
- View transaction history

**Access Rights:**
- View applicable promotions for their store
- Process transactions
- Access customer verification system
- Update reconciliation data
- View own transaction history

## 2.4 Parent Store User

**Platform: Mobile App**

**Primary Responsibilities:**
- Manage mapped stores
- Create mapped store users
- Create new stores
- View store-wise dashboard
- Add invoice number, PID number to submit poof for Reconcile request.

**Access Rights:**
- View applicable promotions for their store
- Process transactions
- Access customer verification system
- Update reconciliation data
- View store transaction history
- Store selection capability
- New Store creation
- New User creation for mapped stores

# 3. ADMIN USER WORKFLOW

## 3.1 Login Process

Admin accesses web portal via solution URL
System displays input field accepting email OR phone number
User enters their identifier and clicks 'Login with OTP'
System sends same OTP to both SMS and Email via MSG91 (based on registered profile)
Admin enters OTP received via SMS or Email and clicks 'Submit'

**System validates OTP:**
- Success: Generate JWT token (RS256), store in HTTP Cookie, route to Home screen
- Failure: Display error "The OTP is incorrect. Please try again."
- Lockout: After 5 failed attempts in 10 minutes, lock account for 15 minutes

## 3.2 Home Dashboard

**Navigation: Left navigation drawer with following options:**
- Dashboard (default)
- Manage Lenses
- Manage Frames
- Manage Users
- Manage Stores
- Transactions
- Manage Promotions
- Manage Store Groups
- Manage Product Groups
- Check Promotions
- System Settings
- Profile

**Dashboard Data Points:**
- Total Number of Opticians/stores
- Total Number of Users
- Total Number of Transactions (last 7 days)
- Total Number of Transactions (last 30 days)
- Trend Graph: Average transactions per store (last 30 days)
- Promotion wise  : Number of Retailers who availed deals,
- Tentative Promotion Spent at store and region level (Offer availed values)
- Average Transaction Value at Store , Region levels.

## 3.3 Lens Management

### 3.3.1 Lens List View

Display all lenses in database
Search and filter capabilities
Add New Lens button
Bulk Upload option

### 3.3.2 Add New Lens Form.  

- Lens Fields

| Field              | Description                   |
|--------------------|-------------------------------|
| New Code           | Unique identifier             |
| entityDescription  | Lens description              |
| Type               | Lens type category            |
| Category           | Product category              |
| Brand              | Manufacturer brand            |
| Design             | Lens design type              |
| Design_C           | Design classification         |
| Index              | Refractive index              |
| Coating_GP         | General purpose coating       |
| Coating_P          | Premium coating               |
| Coating_C          | Custom coating                |
| Material           | Lens material                 |
| Photo              | Photochromic properties       |
| Light brand P      | Light brand premium           |
| Light brand C      | Light brand custom            |
| Colour             | Lens color                    |
| Eye Code           | Eye measurement code          |
| MANUFACTURE        | Manufacturing details         |
| SRP 2025           | Suggested retail price        |




### 3.3.3 Bulk Upload Process

Click "Bulk Upload Lenses"
Upload CSV file

**System validation:**
- Existing SKU: Update attributes based on  SKU Code
- New SKU: Add new lens
- Format/Data Issues: Reject rows with issues. Insert all correct rows.
- Confirmation message upon successful upload

## 3.4 Frame Management

### 3.4.1 Frame List View

Display all frames in database
Search and filter capabilities
Add New Frame button
Bulk Upload option

### 3.4.2 Add New Frame Form. 

--Frame fields

| Column Name             | Description                 |
|-------------------------|-----------------------------|
| EAN_New                 | Unique identifier           |
| Brand                   | Frame Brand                 |
| Description             | Frame description           |
| Gender                  | Frame User gender           |
| Lens Assembly Type      | Attribute                   |
| Material                | Attribute                   |
| Frame Shape             | Attribute                   |
| Polarized               | Attribute                   |
| Photocromic             | Attribute                   |
| Lens Material           | Attribute                   |
| Brand Code              | Attribute                   |
| Brand Vertical          | Attribute                   |
| SKU Code                | SKU Code                    |
| Model Code              |                             |
| GdVal                   |                             |
| Color Code              | Color code of frame         |
| Size                    | Size of frame               |
| Focus Collection        |                             |
| IC/OC                   |                             |
| Manufacturing Details   |                             |
| Release as per SKU Master |                           |
| MRP (INR)               |                             |
| HSN Code                |                             |
| DMS SKU ID              |                             |

### 3.4.3 Bulk Upload Process

Click "Bulk Upload Lenses"
Upload CSV file

**System validation:**
- Existing SKU: Update attributes based on EAN_New
- New SKU: Add new lens
- Format/Data Issues: Reject rows with issues. Insert all correct rows.
- Confirmation message upon successful upload

## 3.5 User Management

### 3.5.1 User List View

- Display all system users - Admin, KAM , Store User , parent Store User
- Edit icon for each user
- Add New User button
- Bulk Upload Button  – Name, Registered Phone Number, Role , Status – Active/Deactive , a) Mapped Parent stores Id for KAM User, b) TR Store Id for Store User  c) Comma Seperated TR Store Id for Parent Store user.
Download CSV format Button – Downloads a CSV with upload headers.

### 3.5.2 Edit User Form

**Common Fields:**
- Name
- Registered Phone Number (10-digit)
- Role (dropdown: Store User, Parent Store User, Admin)
- Activate/Deactivate toggle

**Role-Specific  Mapping Fields:**
- Store Staff: Store (single select dropdown. Simple smart search. Parent + Child store both)
- Parent Store User: Store ( Single select from a list of Parent store only.. simple smart search WITH 3 CHARACTERS )
- Admin: No store selection

### 3.5.3 Add New User

Same form as Edit User.

### 3.5.4 User Status Management

**Activate: User can login to app**
- Deactivate: User is force signed-out and cannot login


## 3.6 Store Management

### 3.6.1 Store Operations

View store list
Add new stores
Bulk upload stores – Fields with description as mentioned below.
Edit existing stores

### 3.6.2 Store Fields

| Field Name            | Description                                                            |   Field Type   | Notes                                                                                                                   |
|-----------------------|------------------------------------------------------------------------|----------------|-------------------------------------------------------------------------------------------------------------------------|
| UniqueId              | Unique Id in our systems for each store                                |               | System generated                                                                                                        |
| Store Type            | Parent / child store                                                   | ID            | Mandatory To segregate between parent store and child store                                                             |
| Zone                  | Operational Zone                                                       | Text          | e.g. North, East, South, West                                                                                           |
| Door Type             | Type of store                                                          | Text          | ECP, EBO, DUD, etc.                                                                                                     |
| Luxottica Outlet ID   | Frames Outlet ID (Frames_Outlet_ID / SAP Code from Luxottica)          | Text/ID    |                                                                                                                         |
| Essilor Outlet ID     | Lens Unique Code (Lens Code from Essilor)                              | Text/ID    |                                                                                                                         |
| Outlet Name           | Store Name                                                             | Text         | e.g. VINAYAK ASSOCIATES                                                                                                 |
| Store Address         | Full Address with Pincode                                              | Text       |                                                                                                                         |
| City                  | City                                                                   | Text       |                                                                                                                         |
| State                 | State                                                                  | Text       |                                                                                                                         |
| GSTIN Primary         | Primary GST Registration Number                                        | Text (15 char) | Mandatory                                                                                                               |
| GSTIN Secondary       | Secondary GST Registration Number                                      | Text (15 char) | Can be blank                                                                                                            |
| PAN                   | PAN Number                                                             | Text (10 char) | Extracted from GSTIN                                                                                                    |
| Parent Customer Code  | Parent Customer Code                                                   | Text           | Useful for grouping. Mandatory. Same as System generated id for all Parent store type user (Default). For Child store type this is a mandatory field. |
| Ship To Code          | Ship To ID                                                             | Text/ID        | Key for delivery mapping                                                                                                |
| Sold To Code          | Sold To ID                                                             | Text/ID        | Key for billing mapping                                                                                                 |
| Service Type          | Direct / DUD / Others                                                  | Text       |                                                                                                                         |
| EL360                 | EL360 participation                                                    | Yes/No         | Text                                                                                                                    |
| EE                    | EE program eligibility                                                 | Yes/No         | Text                                                                                                                    |
| LIS                   | LIS program eligibility                                                | Yes/No         | Text                                                                                                                    |


### 3.6.3 : Add new store - 1. Fill all the mandatory/non mandatory fields  in the new store form and Save.

2. Check for Duplicates in Luxottica Outlet ID/Essilor Outlet Id and create if no Duplicate found.
IF duplicates are available in Essilor id/ Luxottica id – List the duplicates and an Option to Edit and save each row one by one.
Parent CUSTOMER code  SHOULD BE SAME AS Store  Unique code if the store type is Parent. ( A parent store cannot be mapped to another parent customer code).

## 3.7 Store Group Management

View existing store groups
Create new store groups based on “nested” filtering on fields :

{ 
  City
  Pin Code
  State
  Zone
  Door type
  Service type
  EL360
  EE
  LIS
}

Each group represents a set of stores

## 3.8 Product Group Management

View existing product groups
Create new product groups based on “simple” Filtering:

    **Product Type: Frames/Lenses/Combination**
    **Attributes: All Product attributes based on nested filtering Both lens.{Field} and Frame.{Fields}
    ** Filters : simple selection one or multiple.

- Each group represents a set of products

## 3.9 Promotion Management

### 3.9.1 Promotion List View

Display all promotions with list including – Id, Promotion Name, Current Promotion Status, Promotion Type, Quick Action Button

**Quick Action Buttons for**
- Status change – Dropdown with Promotion status - Draft (Default)  / Active  / Scheduled/ Archived / Paused )
- Clone– which creates a similar  new promotion  with status “Draft” of the parent promotion.
- Test console - Opens up the testing framework for the Promotion selected in the same form. The Admin enters the required parameters for the promotion like lens id, frame id , store group and checks promotion correctness.

**Status fields**
- Promotion status can be -  Draft / Active  / Scheduled/ Archived / Paused 

**Status Meanings :**
- Draft : The promotion is still under development and not yet Archived or Active
- Active : Promotions which are active and can be availed at optician.
- Scheduled : Promotions that are Active but the start date is in the future. (system generated and operated Status. No Manual Intervention. System checks at the time of Active status application and assigns Scheduled if Start Date > Today)
- Archived : Promotions that is closed. . Admin can archive an existing active promotion and the promotion is no more valid. Any promotion that completes its End date automatically converts to Archived
- Paused : Admin can pause an active promotion.

**Admin Actions  :  Pause/ Archive/ Activate , Clone, Test**

**Automated Actions : . Every day the system checks All promotions Start -End date flag , Status.**
- If any Active promotions end date falls outside the range it Archives the same.
- If Any “Scheduled” promotions start date <= Today , It updates the status to “Active”.

### 3.9.2 Create New Promotion Workflow

#Step 1: Basic Information

Promotion Title
Description
Marketing Banner (PNG/JPG, max 10MB)
Start Date and End Date selection through calendar widget.

#Step 2: Promotion Type Selection
Fixed Price Offer (Lens/Frame)
Fixed Percentage Offer
Joint Promotional Offer (Lens + Frame)
BOGO
Value Upgrade Offer
Free Item Offer

#Step 3: Configuration Rules IF Conditions with Modular Building Blocks

**Building Blocks**
- Product Type (Frame/Lens)- dropdown
- Product Group – dropdown
- Purchase Quantity – Integer
- Purchase value – sum of all product price
- Lens.{field}  - Dropdown based on  lens Field.
- Frame.{field} - Dropdown based on Frame Field.
- Multiple combinations of above with logical operators
- Modularity - Equals, greater than, lower than, contains, does not contains, is not equal , one of these.
- Each If condition group can have Nested OR/AND  operators.
- Admin can apply multiple “IF Conditions group” with AND/OR operators.

**THEN Actions:**
- Apply Discount (% or Fixed Amount)
- Set Fixed Price for the Product.
- Add Free Item.
- Upgrade based on Upgrade Field selected. (From and To). From Should come from dropdown as per hierarchy and To should also come from Dropdown based on hierarchies configured in settings. MULTIPLE Action items (combining above) with % discount or fixed discount. If 100% selected then complete upgrade is free, else user can configure 50% , 60% off on upgrade etc.

**Logical Operators:**
- AND/OR combinations
- Nested conditions support
- These are Nested OR/AND  OPERATORS AVAILABLE TO APPLY A LOGIC AT MULTIPLE LEVELS


#Step 4: Eligibility Settings

- Store Group Selection AND APPLY ADDITIONAL RULES

**Additional Rules:**
- First-time customers only
- Limit per customer (specify number)
- Minimum purchase amount
- Spend Amount alert.

#Step 5: Review and Activate
- Review Summary of all settings
- Configuration rules summary
- Eligibility criteria
- Testing Framework
- Final Actions: Save as Draft / Activate
- Activate opens up an End Date selection. As start date is the moment it starts.
- If the promotion start date is in the future it automatically is updated as scheduled.
- Promotions management  process should autosave at regular intervals with no data loss. .
- Every New Promotion creates a unique incremental Promotion id and is saved as Draft by default.


### 3.9.3 : Testing console :

Based on each promotions conditions and rules and need variable need, the system creates a raw input field where the user is expected to put in Frame SKU through multiple selection of Frame.{field} and lens.{field} and check if the promotion is applicable or not.(output)

## 3.10 Transaction Management

## 3.10.1 : View all promotional transactions

- Transaction details: ID, Promotion name, Store ID/Name, Date, Promotion value, Invoice ID, PID Number**
- CSV download with filters: Date, State, City, Pin Code
- Transaction verification and status updates:  System-wide  status lifecycle “New →Under Review → Verified → Settlement Complete”
- Default initial status = “New”

## 3.10.2 : Bulk Operations

- CSV Upload Process:
- Download reconciliation template

**Fill with transaction data:**
- Transaction ID
- Invoice number
- Product IDs
- Status flags – Under Review / Verified / Settlement Complete
- Upload CSV file

**System batch processing:**
- Validates data format
- Matches transaction records

**Error Handling:**
- Invalid transaction IDs rejected
- Format errors highlighted
- Partial updates allowed
- Complete file validation required

## 3.11 Check Promotions

Admin types in the Frame code or lens code to check and sees a list of all eligible promotions for the frame/lens.



## 3.13 Settings

3.13.1 : Hierarchy Management
Configure Hierarchies for Index , coating , Material , brand and other Lens.{field} and Frames. {field}
Add new Hierarchies from Frame. Field or lens. Field
Upgrade logic is selection of next higher value within a unique set of (fields).
Configure and Create Lens.{Fields}  and Frame.{fields} that will be used as building blocks. Attributes that can be added to Master tables and be used as blocks.

## 3.14 Profile Management

View profile details
Logout functionality

# 4. STORE USER WORKFLOW

## 4.1 App Installation & Login

Download app from App Store/Play Store
Login using phone number
System sends both SMS and Email OTP via MSG91 (processed by internal Notification Worker in BCKND)
OTP verification process (OTP stored as plain text in PostgreSQL)

**Validation:**
- Unregistered number: "Sorry, This is not a registered number. Please contact your admin. admin@truereach.work “
- Success: Navigate to Home Screen

## 4.2 App Navigation

**Bottom Navigation:**
- Home
- Offers
- History
- Reconcile
- Profile

## 4.3 Home Screen

Display promotion cards with “Marketing banner”  or “Placeholder Image“ and “offer details”  as a description in an endless scrollable list of cards.
Show only promotions applicable to user's store based on Active promotions validity for user store.
Click any offer to initiate "Offer" workflow.

## 4.4  Offer Workflow

Offer  workflow is like mini POS.  Product selection to offer application. Very much like POS  -  add to cart (Next)  and checkout (Avail offer).

### 4.4.1 Step 1: Product Selection

**Staff Actions:**
- Open "Offer" screen
- Select product  - Frame or Lens

**If  Frame :**
- Frame selection from catalogue with “Nested filters” based on (brand, colour, sku code , sun/optical )

**If Lens :**
- Lens selection from catalogue with “Nested filters”  of lens fields (Type,Category,Brand,Design,Design_C,Index,Coating_GP,Coating_P,Coating_C,Material,Photo,Light brand P,Light brand C,Colour,Eye Code, Local/Intl)
- Add Another product / Next : Add Another product again goes back to product selection screen. Next goes to Offer Discovery screen.

### 4.4.3 Step 3: Offer Discovery

**System Process:**
- Analyze selected product combination

**Query eligible promotions based on:**
- Product types selected ( Or Conditions )
- Store location/group
- Current date/time
- Customer eligibility rules
- Display: Eligible offers sorted by savings amount
- Offer name and type
- Benefits to the customer with - Savings calculation
- Eligibility Criteria for the promotion.

### 4.4.4 Step 4: Customer Verification

Staff Inputs Customer name (Optional), Phone number/ Email – Check eligibility button active.
System does an Eligibility check – for Offer with: Product Details, and other eligibility criteria for the offer.
Eligibility check passed –Generate Code Button activates.
Eligibility check failed – Displays “Sorry this offer is not applicable”.  Generate Code Button Remains Inactive.
Staff generates code – Customer receives both SMS OTP and Email OTP
Customer provides code to staff
Staff enters code in app

**System validates:**
- Code correctness
- Time validity (5-15 minutes)
- Retry limit compliance apply.
- Success – Triggers Transaction Completion
- Failure – Error : “Sorry, validation Failed”


### 4.4.5 Step 5: Transaction Completion Screen.

**Final Processing:**
- System applies selected promotion

**Calculate final pricing:**
- Original product prices – Product SKU  wise with Total
- Discount amounts
- Final payable amount
- Generate unique transaction ID
- Send confirmation SMS/Email to customer.

**Transaction Record Contains:**
- Unique transaction ID
- Timestamp
- Store and staff details
- Customer information
- Products purchased
- Promotion Id applied
- Discount amount
- Original Product Prices : Product  SKU wise with totals.

### 4.4.6 Step 6: Post-Transaction Actions

**Available Options:**
- Start new transaction
- View transaction history
- Update reconciliation data

## 4.5 Reconciliation Workflow

### 4.5.1 Step 1: Transaction Review

**Navigation:**
- Open "Reconcile" tab
- View pending transactions list

**Apply filters:**
- Date range
- Transaction status
- Customer name
- Promotion type

**Transaction Display:**
- Transaction ID
- Date and time
- Customer details
- Products sold
- Promotion applied
- Amount and discount
- Current status

### 4.5.2 Step 2: Reconciliation Entry

**Required Information per Transaction:**
- Store user/Parent store user can click on any transaction and enter these details (Alphanumeric field).
- Invoice Details (Store invoice number)
- PID Number
- If any of the Details are filled by the user the Status Changes from “New” to “Under Review”.

**Status Progression:**
- “New” →  "Under Review" → "Verified" → "Settlement Complete"

**Statuses Action Rights:**
- Initial – “New” : As soon as a transaction is created the status is set to “New”
- Under Review: as soon as the Store user enters the Invoice number and PID Number , the status gets updated to “Under Review”
- Verified:  Only Admin action only
- Settlement Complete: Admin action needed. Admin can move a Verified Status to “Settlement complete”

### 4.5.3 Step 3: Approval and Settlement

**Final Process:**
- Staff submits reconciliation data
- Goes to Admin for final approval
- Verification process initiated by Admin – Verified / Under Review
- Reconciliation marked complete - “Settlement complete”

## 4.6 History

List all previous transactions

**Filter options:**
- Date range
- Transaction status

## 4.7 Profile

**Displayed Information: Display only , no edit.**
- Name
- Phone number
- Email ID
- Store name
- Store Id
- Luxottica Id
- Essilor Id

# 5. PARENT STORE USER WORKFLOW

## 5.1 Core Functionality

**All Store User capabilities with additional features:**

## 5.2 Store Selection

Select from a list of stores already mapped through Parent Store Id.
Perform all Store User functions for  the selected store.
Can change the selected store from a dropdown of all mapped store.

## 5.3 Enhanced Profile Features : Navigation  Drawer

**Navigation  Drawer:**
- Create Store User functionality for own stores
- Create Child Store Functionality – Creates new store with Parent Customer code set as Parent User  Unique Store Id.
- Update store user phone number , email id.

# 6. KAM  USER WORKFLOW

## 6.1 Core Functionality

- Limited Admin functionality with access control on mapped parent user.
- Login
- Dashboard (same as admin but with access limited to mapped parent stores )
- Transaction management ( (same as admin but with access limited to mapped parent stores.
- User Management for mapped parent store.



# 7. SYSTEM INTEGRATION POINTS

## 7.1 SMS Integration

1. Transaction confirmation messages
2. OTP 

## 7.2 Email Integration

1. System generated emails
2. OTP 

## 7.3 File Management

1. CSV bulk upload processing
2. Banner image storage
3. Report generation and download

## 7.4 Data Validation

1. Phone number format validation
2. Email format validation
3. File format verification
4. Business rules enforcement