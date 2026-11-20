# CogHMS PMS Backend Agent Guide
## What exists, what needs building, and how to build it

This guide is for an AI coding agent that has **access to the existing ASP.NET Core API
source code** (at `hmssystemapi.cogwave.in`, project name `FrontOfficeApplication`)
and **read access to the legacy VB.NET codebase** at `CogHMS/`.

The agent's task: complete the remaining PMS backend work to match the logic in the VB
codebase, following the conventions already established by the existing API.

---

## 0. The Big Picture

### 0.1 What's Already Built — 155 Endpoints

The existing API at `hmssystemapi.cogwave.in` has:

| Module | Status | Endpoints |
|---|---|---|
| **Master CRUD** | Done | 25 masters × GET/POST/PUT/DELETE each |
| **Auth & User Mgmt** | Done | Login, user CRUD, role CRUD, access rights, operation rights, menu mgmt, password change, branch/property lists |
| **Reservation** | Done | Create reservation, booking cancellation, room availability search, chart availability, guest search, plan/tax/tariff lookups |
| **Tax Calculation** | Done | Dynamic tax engine (interstate/intrastate) |
| **File Storage** | Done | Upload/get/delete images by reference code |
| **Locations** | Done | Countries + states lookup |
| **Product Licence** | Done | Licence key management |
| **Multi-Tenancy** | Done | Property/Branch routing via `MultiTenantDatabases` table |

### 0.2 What Needs Building

| Module | Priority | Complexity |
|---|---|---|
| **Check-In** | Immediate | Medium |
| **Billing Engine** | Immediate | **Very High** |
| **Check-Out** | Immediate | High |
| **Split Billing** | High | High |
| **Settlement & Advance** | High | Medium |
| **Night Audit** | High | High |
| **Financial Year Rollover** | High | Medium |
| **Dashboard/Status** | Medium | Low |
| **Housekeeping** | Medium | Low |
| **Reports (PDF/Excel)** | Medium | Medium |
| **Communication (SMS/Email)** | Low | Low |
| **SignalR Hub** | Low | Low |

---

## 1. First — Read The Existing Code

Before writing anything, read these files in the existing API source:

```
1. Program.cs                      # Middleware pipeline, DI registrations, CORS, Swagger
2. Controllers/MasterController.cs # CRUD pattern — this is the template for all new controllers
3. Controllers/ReservationController.cs  # Complex multi-entity endpoint pattern
4. Controllers/UserAccessMasterController.cs  # Auth/login/rights pattern
5. Services/ anything               # How services are structured
6. Database/ or Data/ anything      # DbContext, entity configurations, migrations
7. Models/ or DTOs/ anything        # Request/response model conventions
8. appsettings.json                 # Connection string pattern
```

**Purpose**: You need to match the existing naming conventions, response formats,
multi-tenancy pattern, and service architecture. Do NOT invent new patterns.

### 1.1 Key Conventions (From the Live API)

Based on the Swagger docs:

| Convention | Example | Notes |
|---|---|---|
| **Endpoint naming** | `GetXxxList`, `CreateXxxDetail`, `UpdateXxxDetail`, `DeleteXxxDetail` | NON-RESTful. Verb-based. Keep this. |
| **DTO naming** | `FloorMasterRequestModel`, `CompanyMasterModel` | Singular, suffixed with `RequestModel` or `Model` |
| **Multi-tenancy** | Every entity has `PropertyCode` (nvarchar(5)) + `BranchCode` (nvarchar(5)) | Always filter by these. Always include in create/update DTOs. |
| **Audit** | Every entity has `UserCode` (int), `IPAddress` (string), `CreatedDate`, `ModifiedDate`, `ModifiedBy` | Populate on create/update. |
| **Soft delete** | `IsActive` or `IsXxxActive` boolean column | Never hard-delete. Set to false. |
| **ID column** | `INT IDENTITY(1,1)` with suffix `Id` | e.g., `FloorId`, `CompanyId`, `RoomDetailId` |
| **Code columns** | Human-readable code + auto-generated | e.g., `FloorCode = "FLR001"`, `RoomCode = "DEL"` — `GetNextMasterCode` endpoint generates these |
| **Response format** | Probably `{ data, success, message }` wrapper | Check existing response DTOs to confirm |
| **Login** | `POST /api/UserAccessMaster/BtnSubmitLogin` | Returns user + branches/properties + rights |
| **Password** | `PUT /api/UserAccessMaster/ChangeUserPassword` | Old + new password model |
| **Scheme** | `FrontOfficeApplication` API title | ASP.NET Core + SQL Server |

### 1.2 RBAC Model (New Schema — Understand This)

The existing API has a **role-based access control** system that did NOT exist in VB:

```
Role_Master              AccessRights_Master          OperationRights_Master
  ├── RoleAccessRights     ├── UserAccessRights         ├── UserOperationRights
  ├── RoleOperationRights  └── AccessRightCode          └── OperationRightCode
  └── RolePermission
       └── UserPermission
```

**AccessRights** = which menus/pages a user can see (e.g., "Masters", "Reservations")
**OperationRights** = what actions they can do (e.g., "Add", "Edit", "Delete", "Print")

A Role groups these together. A User inherits from a Role, then can be individually
overridden via `UserAccessRights` and `UserOperationRights`.

When building new endpoints, add appropriate `[Authorize]` attributes.

---

## 2. The New Database Schema — What's Changed From VB

The schema in `HMSscript.sql` (57 tables in `HMS_Web_Site` database) is a **partial
redesign**, not a 1:1 port of the VB `ILAPURAM_POs` database. You need to extend it.

### 2.1 Tables Already Created (From `HMSscript.sql`)

These 57 tables EXIST in the new schema. Read their columns from the SQL file:

```
Tbl_AccessRights_Master
Tbl_Advance_Receipt          ← Replacement for Tbl_Advance_Receipt
Tbl_BookingCancellation      ← NEW — cancellation with refund calculation
Tbl_BookingSource_Master     ← Replacement for combo lists
Tbl_Branch_Master            ← NEW — multi-branch (hotel chain)
Tbl_ChannelManager           ← From VB, ported directly
Tbl_Company_Master           ← From VB, cleaned up
Tbl_Company_Traiff           ← From VB Tbl_Company_Tariff (typo in new name)
Tbl_Config_Man               ← From VB (licensing)
Tbl_Driver_Master            ← NEW — extracted from frmDriverCommisionEntry
Tbl_EInvoice_Authenticate    ← NEW — e-invoice API auth tokens
Tbl_EInvoice_Authenticate_Result ← NEW
Tbl_Financial_Master         ← From VB (financial years)
Tbl_Floor_Master             ← From VB
Tbl_ForEver_Data             ← From VB (calendar/availability grid)
Tbl_FormC_Master             ← From VB (C-Form for foreign guests)
Tbl_GRC_Bookings             ← NEW — permanent GRC (Guest Registration Card) table
Tbl_GSTType_Master           ← NEW
Tbl_Guest_Master             ← From VB, cleaned up
Tbl_Identityproof_Master     ← From VB combo lists → proper master
Tbl_Image_FileStorage        ← NEW — file storage by reference
Tbl_InstructionDepartment_Master ← NEW — extracted from Tbl_Instruction_Master
Tbl_Item_Master              ← Migrated from POS DB into HMS
Tbl_Ledger_Master            ← From VB
Tbl_MenuIndex_Master         ← NEW — menu structure for sidebar/access rights
Tbl_OperationRights_Master   ← NEW — what operations users can do
Tbl_PaymentMainMode_Master   ← NEW — configurable payment modes (Cash, Card, etc.)
Tbl_PaymentSubMode_Master    ← NEW — sub-modes (Visa, MasterCard under Card)
Tbl_Plan_Master              ← From VB Tbl_PlanMaster (renamed)
Tbl_Property_Master          ← NEW — hotel property (chain support)
Tbl_Purpose_Master           ← From VB combo lists → proper master
Tbl_Reservation_Master       ← From VB
Tbl_Reservation_Slave        ← From VB
Tbl_Reservation_Charge       ← From VB
Tbl_ReservationPayMode_Master ← From VB combo lists → proper master
Tbl_Revenue_Master           ← From VB
Tbl_Role_Master              ← NEW — role-based access control
Tbl_RoleAccessRights_Master  ← NEW
Tbl_RoleOperationRights_Master ← NEW
Tbl_RolePermission_Master    ← NEW
Tbl_Room_Master              ← From VB (room TYPES — confusingly named)
Tbl_Room_Details             ← From VB (individual rooms)
Tbl_SalesExecutive_Master    ← From VB combo lists → proper master
Tbl_Segment_Master           ← From VB
Tbl_SpecialInstruction       ← From VB Tbl_Special_Ins + Tbl_INSTRUCTION
Tbl_Steward_Master           ← Migrated from POS DB into HMS
Tbl_SubIndex_Master          ← NEW — sub-menu indices for access rights
Tbl_SubSegment_Master        ← From VB Tbl_Segment_Slave
Tbl_Tax_Master               ← From VB
Tbl_TaxDescription_Master    ← From VB Tbl_Tax_Description
Tbl_TaxDetails               ← From VB
Tbl_Taxpayer_Details         ← NEW — GST taxpayer per property
Tbl_TaxRule_Master           ← From VB
Tbl_User_Master              ← From VB, redesigned with multi-property/branch
Tbl_UserAccessRights_Master  ← NEW
Tbl_UserOperationRights_Master ← NEW
Tbl_UserPermission_Master    ← NEW
```

### 2.2 Tables MISSING — You Must Create These

These tables exist in the VB database but have NOT been ported yet. Create them:

```
Tbl_CheckIn_Master          ← Core — create entity
Tbl_CheckIn_Slave           ← Core — create entity
Tbl_NewCheckout_Master      ← Rename to Tbl_CheckOut_Master or Tbl_Bill_Master
Tbl_NewCheckout_Slave       ← Rename to Tbl_CheckOut_Slave or Tbl_BillDay
Tbl_Split_Master            ← Rename to Tbl_SplitBill_Master
Tbl_Split_Slave             ← Rename to Tbl_SplitBill_Slave
Tbl_Settlement_Master       ← Or reuse Tbl_Advance_Receipt for now
Tbl_DayClosing              ← Create as Tbl_DayClosing
Tbl_Night_Audit             ← Create as Tbl_NightAudit
Tbl_ExtraBed_Master         ← Create as Tbl_ExtraBed
Tbl_ExtraPax_Master         ← Create as Tbl_ExtraPax
Tbl_PostCharge_Master       ← Create as Tbl_PostCharge
Tbl_PostCharge_Slave        ← Create as Tbl_PostCharge_Line
Tbl_CollectedTax            ← Create as Tbl_CollectedTax
Tbl_Compliment_Update       ← Create as Tbl_ComplimentUpdate
Tbl_RoomShifft_Master       ← Create as Tbl_RoomShift (fix the typo)
Tbl_Ref_Master              ← Create as Tbl_Reference
Tbl_LostandFounditems       ← Create as Tbl_LostAndFound
Tbl_Sms_Track               ← Create as Tbl_SmsLog
Tbl_SMS_Balance             ← Create as Tbl_SmsBalance
Tbl_HouseKeeping            ← Housekeeping status — create as Tbl_HouseKeeping
```

**Important**: Match the existing table naming conventions — PascalCase with
underscores for multi-word prefixes. Follow the pattern used by the 57 existing
tables: `Tbl_` prefix, then PascalCase without spaces.

### 2.3 Multi-Tenancy Routing

The master database (`MasterDatabasePOS_HMS`) has a `MultiTenantDatabases` table that
routes requests to the correct property database:

```
MultiTenantDatabases:
  DBId: int IDENTITY
  BranchCode: nvarchar(50)    ← "BR001", "SH001", "SPAIR"
  DatabaseType: nvarchar(50)   ← "HMS" or "POS"
  ServerName: nvarchar(50)
  DatabaseName: nvarchar(200)
  UserName: nvarchar(100)
  Password: nvarchar(200)
  Port: int
  IsActive: bit
```

Every API request includes `PropertyCode` + `BranchCode`. The middleware/switcher
routes to the correct `HMS_Web_Site` or `Second_HMS_Web_Site` database.

**New entities**: Include `PropertyCode` + `BranchCode` columns on ALL new tables to
maintain the multi-tenancy pattern.

---

## 3. Build Plan — Exact Order

### STEP 1: Create Missing Entities + Migrations

Create entity classes for the 20 missing tables listed in Section 2.2.

For each entity:
1. Create the C# class in the existing Entities folder
2. Create EF configuration (Fluent API or data annotations) in the existing Configurations folder
3. Add `DbSet<>` to the existing DbContext
4. Create and run a migration against `HMS_Web_Site`

**Important column patterns to follow** (from existing entities):

```csharp
// Every entity should have:
public int XxxId { get; set; }                    // Primary key, IDENTITY
public string PropertyCode { get; set; }           // nvarchar(5), NOT NULL
public string BranchCode { get; set; }             // nvarchar(5), NOT NULL
public int UserCode { get; set; }                  // FK to User
public string? IpAddress { get; set; }             // nvarchar(50), nullable
public DateTime CreatedDate { get; set; }          // default GETDATE()
public DateTime? ModifiedDate { get; set; }
public int? ModifiedBy { get; set; }
```

**For the CheckIn entity specifically:**

```csharp
public class Tbl_CheckIn_Master
{
    public int CheckInId { get; set; }
    public string CheckInNo { get; set; }           // "CNN0000001" or "RCN0000001"
    public string GuestCode { get; set; }           // FK to Tbl_Guest_Master.GuestCode
    public string? GroupCode { get; set; }
    public int StayNo { get; set; }
    public int? GracePeriod { get; set; }            // minutes
    public string CheckType { get; set; }            // "WalkIn", "Reservation", "Group"
    public string? PackageId { get; set; }
    public string? VehicleNo { get; set; }
    public string? Purpose { get; set; }
    public int? BagCount { get; set; }
    public string? BillInstruction { get; set; }
    public string? ReservationNo { get; set; }       // FK to Reservation
    // + audit columns
}

public class Tbl_CheckIn_Slave
{
    public int CheckInSlaveId { get; set; }
    public string CheckInNo { get; set; }            // FK to CheckIn_Master
    public string GuestCode { get; set; }
    public DateTime ArrivalDate { get; set; }
    public TimeSpan? ArrivalTime { get; set; }
    public DateTime DepartureDate { get; set; }
    public TimeSpan? DepartureTime { get; set; }
    public string RoomNo { get; set; }               // FK to Tbl_Room_Details
    public string RoomCode { get; set; }
    public int AdultCount { get; set; }
    public int ChildCount { get; set; }
    public int PaxCount { get; set; }
    public decimal Tariff { get; set; }              // per-night rate
    public string? TaxCode { get; set; }
    public string? GstNo { get; set; }
    public decimal? Grand { get; set; }
    public string Status { get; set; }               // 'O', 'F', 'Q', 'U'
    public string CheckType { get; set; }
    // + audit columns
}
```

### STEP 2: Build Check-In API (8 endpoints)

These endpoints replace `frmCheckIn.vb` + `frmCheckinNew.vb` (21,424 VB lines).

Follow the existing `ReservationController` pattern for creating nested entities.

```
POST   /api/CheckIn/GetActiveCheckInList
POST   /api/CheckIn/GetCheckInDetails
POST   /api/CheckIn/CreateCheckInDetail           # Main endpoint — handles walk-in + from-reservation + group
POST   /api/CheckIn/UpdateCheckInDetail           # Amend stay
POST   /api/CheckIn/ExtendStay                     # Extend departure date
POST   /api/CheckIn/ShiftRoomDetail                # Room shift
POST   /api/CheckIn/AddExtraBedDetail
POST   /api/CheckIn/AddExtraPaxDetail
POST   /api/CheckIn/AddPostChargeDetail
POST   /api/CheckIn/MarkComplimentary               # House guest
```

**CreateCheckInDetail request shape** (follow `ReservationRequestModel` pattern):

```csharp
public class CheckInRequestModel
{
    public string? CheckInNo { get; set; }          // null for new, set for amend
    public string? ReservationNo { get; set; }       // if converting from reservation
    public string CheckType { get; set; }            // "WalkIn"/"Reservation"/"Group"
    public string? GroupCode { get; set; }
    public CheckInGuestDetail GuestDetail { get; set; }
    public List<CheckInRoomDetail> RoomDetails { get; set; }
    public CheckInPaymentDetail? PaymentDetail { get; set; }
    public List<CheckInInstructionDetail>? InstructionDetails { get; set; }
    public bool IsWalkIn { get; set; }
    public int UserCode { get; set; }
    public string IpAddress { get; set; }
    public string PropertyCode { get; set; }
    public string BranchCode { get; set; }
}
```

**VB logic to extract**:

Go to `frmCheckIn.vb`. The `_Load` and save handlers do:
1. Load room availability (call existing `GetRoomAvailability` + `GetChartAvailability`)
2. Load tax rules (call existing `GetTaxDetails` + `GetTaxRulesDetails`)
3. Load guest data (call existing `GetGuestSearchList` or create new via `CreateGuestMasterDetail`)
4. Select rooms from the availability grid
5. Insert into `Tbl_CheckIn_Master` + `Tbl_CheckIn_Slave`
6. Update `Tbl_Room_Details.Status = 'O'`
7. Process advance payment → insert into `Tbl_Advance_Receipt`

### STEP 3: Build the Billing Engine

**WARNING**: This is the hardest part. The VB function is `GenerateStayingDetails()` in
`GenClass/RoomOperations.vb` (starts around line 5,305). You already have a tax calculation
engine at `POST /api/TaxCalculation/CalculateDynamicTaxes`. Use it.

**What `GenerateStayingDetails()` does** (from VB):

For a given check-in, for EVERY day of the stay:
1. Get room tariff from `Tbl_CheckIn_Slave.Tarrif` (or shifted room tariff from `Tbl_RoomShift`)
2. If the stay has a package/plan (`Tbl_Plan_Master`), override rate with plan rate × pax
3. Call tax calculation for: tariff amount, extra bed amount, food amount
4. Add extra bed charges
5. Add extra pax charges
6. Add post charges for that day
7. Deduct advances (at total level, not per day)
8. Apply discounts (user-level discount percent, bill-level discount)
9. Calculate net amount per day
10. Handle grace period/noon checkout (if `IsNoonCheckOut`, call `NoonStaying()` variant)
11. Handle complimentary (house) guests: zero-rate tariff, tax still on extras
12. Handle round-off: ₹0.50 rounds to nearest ₹1

**Implementation**:

```csharp
// POST /api/Billing/GetBillPreview
public class BillPreviewRequestDto
{
    public string CheckInNo { get; set; }
    public string PropertyCode { get; set; }
    public string BranchCode { get; set; }
}

// Response
public class BillPreviewResponseDto
{
    public string CheckInNo { get; set; }
    public string GuestName { get; set; }
    public string RoomNo { get; set; }
    public DateTime CheckInDate { get; set; }
    public DateTime CheckOutDate { get; set; }
    public List<BillDayDto> Days { get; set; }
    public decimal GrandTotal { get; set; }
    public decimal AdvancePaid { get; set; }
    public decimal NetDue { get; set; }
    public Dictionary<string, decimal> Taxes { get; set; }
    public string AmountInWords { get; set; }
}

public class BillDayDto
{
    public DateTime Date { get; set; }
    public decimal Tariff { get; set; }
    public List<TaxLineDto> Taxes { get; set; }
    public decimal ExtraBed { get; set; }
    public decimal ExtraPax { get; set; }
    public decimal PostCharge { get; set; }
    public decimal Discount { get; set; }
    public decimal DayTotal { get; set; }
}
```

**How to calculate per day** (loop through each date in the stay):

```csharp
for (var date = checkInDate; date <= checkoutDate; date = date.AddDays(1))
{
    // 1. Get the slave record for this date
    var slave = slaves.FirstOrDefault(s => s.ArrivalDate <= date && s.DepartureDate >= date);
    if (slave == null) continue;
    
    var tariff = slave.Tariff;
    
    // 2. Plan/package override
    if (checkIn.PackageId != null)
    {
        var plan = await db.Tbl_Plan_Master.FindAsync(checkIn.PackageId, ct);
        tariff = plan.FoodRate ?? 0 * slave.PaxCount;
    }
    
    // 3. Tax calculation — REUSE existing TaxCalculation engine
    var taxRequest = new TaxCalculationRequestDto
    {
        TariffAmount = tariff,
        CalculationDate = date.ToString("yyyy-MM-dd"),
        IsForeignOrInterstateClient = isInterstate,
        ExtraAdult = extraAdult,
        ExtraChild = extraChild,
        DiscountValue = discountValue,
        IsDiscountPercentage = isDiscountPercent,
        PropertyCode = propertyCode,
        BranchCode = branchCode
    };
    var taxResult = await _taxService.CalculateDynamicTaxes(taxRequest);
    
    // 4. Extra bed
    var extraBed = await db.Tbl_ExtraBed
        .Where(e => e.CheckInNo == checkInNo && e.Date == date)
        .SumAsync(e => e.Total, ct);
    
    // 5. Extra pax
    var extraPax = ... similar pattern
    
    // 6. Post charges
    var postCharges = ... similar pattern
    
    // 7. Discount
    var discount = 0m;
    if (checkIn.Guest?.UserDiscountPercent > 0)
        discount = tariff * checkIn.Guest.UserDiscountPercent / 100;
    
    // 8. Day total
    var dayTotal = tariff + taxResult.TotalTax + extraBed + extraPax 
        + postCharges - discount;
    dayTotal = Math.Round(dayTotal, MidpointRounding.AwayFromZero);
    
    days.Add(new BillDayDto { Date = date, ... });
}
```

**VB reference points for billing logic**:

| Logic | VB File | Function/Area |
|---|---|---|
| Stay generation loop | `RoomOperations.vb` | `GenerateStayingDetails()` (line ~5305) |
| Plan/package rates | `RoomOperations.vb` | Plan sections in `GenerateStayingDetails()` |
| Tax for each component | `SplitGuest.vb` | `GenerateTaxRulesForHotel()` |
| Grace period/noon check-out | `NoonCheckOuts.vb` | `NoonStaying()` |
| Round-off logic | `RoomOperations.vb` | End of `GenerateStayingDetails()` |
| Amount to words | `RoomOperations.vb` | `NumToWords()` |
| Financial year filter | `CheckOutBill.vb` | `FinDataLoader()`, `FinWhereCond` |

### STEP 4: Build Check-Out API (6 endpoints)

```
POST   /api/CheckOut/GetBillList
POST   /api/CheckOut/GetBillDetails
POST   /api/CheckOut/ProcessCheckOut            # Final checkout — generates bill + updates room
POST   /api/CheckOut/VoidBillDetail
POST   /api/CheckOut/ModifyBillDetail           # Modify bill with audit trail
POST   /api/CheckOut/GetOutstandingBills
```

**ProcessCheckOut** must:
1. Call the billing preview to get the final bill
2. Insert into `Tbl_Bill` (new Tbl_NewCheckout equivalent) + `Tbl_BillDay`
3. Insert `Tbl_CollectedTax` entries
4. Insert `Tbl_Settlement` entry (payment)
5. Update `Tbl_Room_Details.Status = 'D'` (Dirty, for housekeeping)
6. Update `Tbl_CheckIn_Slave.Status = 'U'` (checked out)
7. Insert into `Tbl_ChannelManager` if needed (status update for OTA)
8. Return bill PDF

### STEP 5: Build Split Bill API (5 endpoints)

```
POST   /api/SplitBill/GetSplitBillList
POST   /api/SplitBill/CreateSplitBillDetail
POST   /api/SplitBill/UpdateSplitBillDetail
POST   /api/SplitBill/SettleSplitBill
POST   /api/SplitBill/GetSplitBillDetails
```

**VB logic**: `frmpopupsplit.vb` (4,400 lines) + `frmnewsplitbill.vb` (2,451)
Split a master bill into sub-bills per guest/company. Insert into `Tbl_SplitBill` +
`Tbl_SplitBillLine`. Each split can be settled independently.

### STEP 6: Build Settlement & Advance API (8 endpoints)

```
POST   /api/Settlement/GetSettlementList
POST   /api/Settlement/CreateSettlementDetail
POST   /api/Settlement/RefundSettlementDetail
POST   /api/Settlement/TransferToCompany

POST   /api/Advance/GetAdvanceList
POST   /api/Advance/CreateAdvanceDetail
POST   /api/Advance/RefundAdvanceDetail
POST   /api/Advance/TransferAdvanceDetail
```

Settlements already have a foundation in `Tbl_Advance_Receipt` (which acts as a general
payment receipt table). Build on it.

### STEP 7: Build Night Audit API (4 endpoints)

```
POST   /api/NightAudit/GetNightAuditStatus
POST   /api/NightAudit/RunNightAudit
POST   /api/NightAudit/GetNightAuditReport
POST   /api/NightAudit/GetNightAuditHistory
```

**VB logic**: `GenClass/NightAudit.vb` — `NightAuditGeneration()` (2,219 lines)

The night audit is a **transactional batch** that:
1. Takes a snapshot of ALL occupied rooms for the current date
2. For each room, calculates that day's charges (tariff + tax + extras)
3. Inserts into `Tbl_NightAudit` table
4. Marks `Tbl_DayClosing` for the date as closed
5. If any step fails → ROLLBACK the entire transaction

NEVER partial-close a day. This is non-negotiable.

### STEP 8: Build Financial Year Rollover

```
POST   /api/Master/RolloverFinancialYear
```

**VB logic**: `CheckOutBill.vb` — `CreateNewFinancialYear()`

Checks if the current financial year end (March 31) has passed. If yes:
1. Creates a new `Tbl_Financial_Master` row
2. Marks the previous year as closed (`CurrentStatus = false`)
3. Resets the bill counter sequence
4. The existing `SaveorUpdateFinancialMasterDetail` handles CRUD — add the rollover method

### STEP 9: Build Dashboard Endpoints

```
POST   /api/Dashboard/GetRoomStatusDashboard
POST   /api/Dashboard/GetTodayArrivals
POST   /api/Dashboard/GetTodayDepartures
POST   /api/Dashboard/GetOccupancySummary
POST   /api/Dashboard/GetRevenueSummary
```

**VB logic**: `frmCurrentStatus.vb` (7,806 lines)

Query `Tbl_Room_Details` for status counts, `Tbl_CheckIn_Slave` for today's
arrivals/departures, `Tbl_Advance_Receipt` for revenue.

### STEP 10: Build Housekeeping API (4 endpoints)

```
POST   /api/Housekeeping/GetRoomStatusGrid
POST   /api/Housekeeping/UpdateRoomStatus
POST   /api/Housekeeping/GetLostAndFoundList
POST   /api/Housekeeping/PostHousekeepingCharge
```

**VB logic**: `frmhousekeep.vb` (382 lines) + `frmHouseKeep.vb` (381 lines)

Room status cycle: Vacant → Occupied → Dirty → (cleaned) → Vacant again. The
housekeeping module tracks this and posts minibar/laundry charges to the guest folio.

### STEP 11: Build Reports API (4 endpoints)

```
POST   /api/Reports/GetReportList
POST   /api/Reports/GetReportData
POST   /api/Reports/ExportReportPdf
POST   /api/Reports/ExportReportExcel
```

**VB logic**: `Report Forms/frmreportman.vb` (10,987 lines)

Define reports as config: `{ reportId, name, sqlQuery, columns[], params[] }`.
The `GetReportData` endpoint runs the query and returns JSON for the frontend DataTable.
PDF and Excel are server-side generation.

Priority reports from legacy Crystal Reports:
1. Guest bill (`MyCheckOutBill`)
2. Night audit report
3. Daily sales
4. Guest in-house list
5. House status
6. GST returns
7. Reservation report
8. Arrival/Departure list
9. Occupancy report
10. Company sale report

### STEP 12: Build Communication API (4 endpoints)

```
POST   /api/Communication/SendSms
POST   /api/Communication/SendEmail
POST   /api/Communication/GetSmsBalance
POST   /api/Communication/GetMessageLog
```

**VB logic**: `GenClass/ElectronicMail.vb` (1,724 lines)

SMS and Email only. WhatsApp is deferred. Use the `IMessageSender` interface pattern.
SMS config from `Tbl_Config_Man` or a new `Tbl_SmsConfig` table.

### STEP 13: Build SignalR Hub

Add a `RoomStatusHub` for live room status updates:

```csharp
public class RoomStatusHub : Hub
{
    public async Task SendRoomStatusUpdate(string roomNo, string status)
    {
        await Clients.All.SendAsync("RoomStatusChanged", new { roomNo, status });
    }
}
```

Wire it in `Program.cs` and trigger from check-in, check-out, and housekeeping operations.

---

## 4. How to Read the VB Code (Quick Reference)

When you need to find specific logic in the legacy VB:

| Logic | VB File | Function/Area |
|---|---|---|
| Login + app bootstrap | `frmuserlog.vb` | `_Load`, `btnlogin_Click` |
| Server/DB connection setup | `GenClass/GeneralFunctions.vb` | `GetDBConnection()` |
| Financial year loading | `GenClass/CheckOutBill.vb` | `FinDataLoader()` |
| Financial year rollover | `GenClass/CheckOutBill.vb` | `CreateNewFinancialYear()` |
| Room availability matrix | `GenClass/RoomOperations.vb` | `GetRoomAllocation()`, `CheckCurrentRoom()` |
| Room status dashboard | `frmCurrentStatus.vb` | `_Load`, timer handlers |
| Reservation create | `frmreservation.vb` | `_Load`, save handlers |
| Reservation amend/cancel | `frmamendbooking.vb` | Various handlers |
| Check-in create | `frmCheckIn.vb` / `frmCheckinNew.vb` | `_Load`, save handlers |
| Check-in amendments | `frmGuestInfo.vb` | Various handlers |
| Checkout flow | `frmcheckout.vb` | `ShowGridData()`, button handlers |
| **Bill calculation** | **`GenClass/RoomOperations.vb`** | **`GenerateStayingDetails()` (line ~5305)** |
| Tax calculation | `GenClass/SplitGuest.vb` | `GenerateTaxRulesForHotel()` |
| Split bill | `frmpopupsplit.vb` | Various handlers |
| Settlement | `frmFinalSetlement.vb` | Save handlers |
| Advance payment | `frmadvanced.vb` | Save handlers |
| Night audit | `GenClass/NightAudit.vb` | `NightAuditGeneration()` |
| Day status check | `GenClass/NightAudit.vb` | `GetDayStatus()` |
| Noon checkout logic | `GenClass/NoonCheckOuts.vb` | `NoonStaying()` |
| SMS sending | `GenClass/ElectronicMail.vb` | `SmsAmc()`, various `Case` blocks |
| Email sending | `GenClass/ElectronicMail.vb` | `SendYourMail()` |
| Report queries | `Report Forms/frmreportman.vb` | `btnfilter_Click()` |
| Bill print | `Report Forms/frmbillprint.vb` | `btnprint_Click()` |
| Menu/role config | `CogMenu.vb` | `ToolBarShow()` |
| System config | `frmSystemconfigure.vb` | Key-value read/write |
| Database backup | `frmbackup.vb` | Backup commands |
| Housekeeping | `frmhousekeep.vb` | `GenerateItemGrids()` |
| Guest entry/search | `frmguestentry.vb` | `_Load`, `BtnSearchGuest()` |
| Room master | `frmroomsmast.vb` | `_Load`, save handlers |
| Tax master | `frmtaxmaster.vb` | CRUD handlers |
| Plan/package | `frmPlanMaster.vb` | CRUD handlers |
| Company master | `frmcompmast.vb` | CRUD handlers |
| Amount to words | `GenClass/RoomOperations.vb` | `NumToWords()` — port this function directly |
| Message templates | `SMSTEMPLATE.txt` | 16 SMS templates |

---

## 5. Conventions Checklist (Must Follow)

- [ ] Every new table has: `Id` (IDENTITY), `PropertyCode`, `BranchCode`, `UserCode`, `IpAddress`, `CreatedDate`, `ModifiedDate`, `ModifiedBy`
- [ ] Every endpoint uses `POST` (not `GET`) for queries that need PropertyCode/BranchCode
- [ ] DTOs are singular and suffixed: `XxxRequestModel`, `XxxDto`
- [ ] Response wrapped in `{ data, success, message }` if that's the existing pattern
- [ ] Soft deletes only — set `IsActive = false`, never DELETE rows
- [ ] All money uses `decimal`, never `float`/`double`
- [ ] All dates use `DateTime` or `DateOnly`
- [ ] All tax calculations reuse the existing `TaxCalculation/CalculateDynamicTaxes` endpoint
- [ ] All guest lookups reuse the existing `Reservation/GetGuestSearchList`
- [ ] All room lookups reuse the existing `Reservation/GetRoomAvailability`
- [ ] Night audit is a single atomic transaction — never partial-close a day
- [ ] Use async/await for all DB operations
- [ ] Pass `CancellationToken` through the call chain

---

## 6. What NOT to Port

| VB Pattern | Why Not | What to Do |
|---|---|---|
| `Stp_Dynamic_Opr` (dynamic SQL gateway) | SQL injection | EF Core LINQ or parameterized raw SQL |
| `MAX(RNo) + 1` | Race condition | `IDENTITY` column |
| `Module` global variables | Untraceable state | DI-injected scoped services |
| `Tmp_*` / `Temp_*` staging tables | Disk I/O, concurrency | In-memory computation from live data |
| Crystal Reports `.rpt` | Proprietary, not server-side | HTML templates or SQL → PDF/Excel |
| Registry-based config | Not portable | appsettings.json / environment variables |
| Hardcoded `sa/cog` | Security nightmare | Environment variables with restricted DB users |
| Inline SQL string concatenation | SQL injection | Parameterized queries |
| `Try/Catch` with empty catch | Silent failures | Proper exception handling with logging |
| `MsgBox` calls | Desktop-only | HTTP status codes with { success, message } response |

---

## 7. Testing Priorities

1. **Billing engine** — test every tax scenario. Compare output against the legacy VB
   for the same input data. This is where real money is calculated.
2. **Night audit** — verify atomic rollback on failure. Verify duplicate run prevention.
3. **Check-in/Check-out** — verify room status transitions. Verify reservation conversion.
4. **Split bill** — verify split amounts sum to the total original bill.

Use the existing `HMS_web_site` test DB for integration tests.

---

## 8. Summary — Your Task List

```
Phase 1: Schema
  □ Create 20 missing entity classes
  □ Create EF configurations for each
  □ Run migrations

Phase 2: Check-In
  □ CreateCheckInDetail (walk-in + from-reservation + group)
  □ GetActiveCheckInList, GetCheckInDetails
  □ UpdateCheckInDetail, ExtendStay
  □ ShiftRoomDetail
  □ AddExtraBed, AddExtraPax, AddPostCharge
  □ MarkComplimentary

Phase 3: Billing
  □ GetBillPreview (the billing engine)
  □ AmountInWords helper

Phase 4: Check-Out
  □ GetBillList, GetBillDetails
  □ ProcessCheckOut
  □ VoidBillDetail, ModifyBillDetail
  □ GetOutstandingBills

Phase 5: Split Bill
  □ CreateSplitBill, UpdateSplitBill
  □ SettleSplitBill, GetSplitBillDetails

Phase 6: Settlement & Advance
  □ Settlement CRUD + refund + transfer
  □ Advance CRUD (or extend existing Advance_Receipt)

Phase 7: Night Audit
  □ GetNightAuditStatus
  □ RunNightAudit (atomic transaction)
  □ GetNightAuditReport, GetNightAuditHistory

Phase 8: Financial Year
  □ RolloverFinancialYear

Phase 9: Dashboard
  □ GetRoomStatusDashboard
  □ GetTodayArrivals/Departures
  □ GetOccupancySummary, GetRevenueSummary

Phase 10: Housekeeping
  □ GetRoomStatusGrid, UpdateRoomStatus
  □ GetLostAndFoundList, PostHKCharge

Phase 11: Reports
  □ GetReportList, GetReportData
  □ ExportPdf, ExportExcel

Phase 12: Communication
  □ SendSms, SendEmail
  □ GetSmsBalance, GetMessageLog

Phase 13: SignalR
  □ RoomStatusHub
  □ Wire into check-in/check-out/housekeeping
```