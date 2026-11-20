# CogHMS Backend Migration Plan
## PMS (Property Management System) — ASP.NET Core Web API + EF Core

**Scope**: This plan covers only the **core PMS** — Guest management, Rooms, Reservations,
Check-in/Check-out, Billing, Night Audit, Housekeeping, Reports, and Hotel Admin.
External systems (POS, EPABX, Door Locks, Power Automation, Tally, Fingerprint, NFC,
Signature Pads, WhatsApp, Channel Manager) are deferred to a later phase
and are documented in `bridges/future-integrations.md`.

---

## 1. Current State Summary

| Aspect | Current | Target |
|---|---|---|
| Language | VB.NET | C# (.NET 8+) |
| Framework | .NET Framework 3.5 | ASP.NET Core 8+ |
| Architecture | Monolithic WinForms app | Clean Architecture Web API |
| Data Access | Raw inline SQL via `Stp_Dynamic_Opr` SP | EF Core + LINQ |
| Database | ~80 HMS tables, `MAX(RNo)+1` IDs | Restructured schema with IDENTITY |
| Auth | Plaintext user/pass in SQL | JWT + hashed passwords |
| Config | Hardcoded SA passwords in app.config | Environment variables / appsettings |
| Reports | Crystal Reports (.rpt) | Server-side PDF generation |
| Integrations | SMS, Email | Service abstractions with adapters |

**Deferred (Phase 5+)**:
| System | Status |
|---|---|
| POS/F&B (`ILAPURAM_POs` DB) | Read-only bridge — not rebuilt |
| Tally Accounting (`BAARE2013` DB) | Tally push — deferred |
| EPABX Telephony (`Epabx` DB) | Read-only bridge — deferred |
| Door Locks (`btlock57L.dll`) | Local agent — deferred |
| Power Automation (`PC_Pro`) | Local agent — deferred |
| Fingerprint (SecuGen SDK) | Local agent — deferred |
| Signature Pad (TabletPC Ink) | Canvas-based web fallback |
| NFC Cards | Not ported (browser-native keycards) |
| WiFi Hotspot | Deferred |
| Channel Manager / OTA | Deferred |
| WhatsApp Business API | Deferred (SMS + Email are phase 1) |

---

## 2. Phased Approach

### Phase 1: Foundation (Weeks 1-3)
Schema cleanup, project scaffold, auth, core infrastructure.

### Phase 2: Core Hotel Operations (Weeks 4-8)
Guest, Room, Reservation, Check-In, Check-Out APIs.

### Phase 3: Financial & Billing (Weeks 9-12)
Billing, Split, Settlement, Night Audit, Tax, Financial Year.

### Phase 4: Supporting Modules (Weeks 13-16)
Housekeeping, Reports, Communication (SMS + Email only), Admin, Masters.

---

## 3. Phase 1: Foundation

### 3.1 Solution Structure

```
CogHMS/
├── CogHMS.sln
├── src/
│   ├── CogHMS.Api/                    # ASP.NET Core Web API
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   ├── Hubs/
│   │   │   └── RoomStatusHub.cs       # SignalR for live room status
│   │   ├── Program.cs
│   │   └── appsettings.json
│   ├── CogHMS.Application/            # Business logic, CQRS handlers
│   │   ├── Guests/
│   │   ├── Rooms/
│   │   ├── Reservations/
│   │   ├── CheckIns/
│   │   ├── Billing/
│   │   ├── NightAudit/
│   │   ├── Housekeeping/
│   │   ├── Reports/
│   │   ├── Communications/
│   │   ├── Masters/
│   │   └── Common/
│   ├── CogHMS.Domain/                 # Entities, enums, value objects
│   │   ├── Entities/
│   │   ├── Enums/
│   │   └── ValueObjects/
│   └── CogHMS.Infrastructure/         # EF Core, external services
│       ├── Persistence/
│       │   ├── Configurations/        # EF Fluent API configs per entity
│       │   └── Migrations/
│       ├── Services/                  # SMS, Email
│       └── Reports/                   # PDF generation
├── tests/
│   ├── CogHMS.Api.Tests/
│   ├── CogHMS.Application.Tests/
│   └── CogHMS.Infrastructure.Tests/
└── docs/
    ├── schema.md
    └── api-spec.md
```

### 3.2 Schema Restructuring

#### Changes from legacy schema:

| Problem | Fix |
|---|---|
| `RNo` as `MAX(RNo)+1` -- race condition | `INT IDENTITY(1,1)` |
| Temp tables for reports (`Tmp_*`, `Temp_*`) | Live queries via EF, no staging tables |
| `TrDate` as string-based dates | `DATETIME2` columns |
| Duplicate columns (`IpAdd`/`IPADD`, `CUpdate`/`CUpDate`) | Normalize to single names |
| `CHAR(1)` flags everywhere (`Y`/`N`, `S`/`N`, `A`/`V`) | `BIT` or proper enums |
| `Status` columns with single-letter codes (`O`,`V`,`D`,`M`,`B`) | `TINYINT` FK to `RoomStatusTypes` |
| `FinWhereCond` -- financial year filter appended to every query | `FinYearId` FK on transaction tables |
| `CheckInNo` / `GRN` as manually formatted strings | Auto-generated from IDENTITY |
| `Loger` column pattern | `CreatedBy`, `CreatedAt`, `UpdatedBy`, `UpdatedAt` audit columns |
| Hardcoded split-bill temporary tables | Proper `SplitBill` and `SplitBillLine` schema |
| `Tbl_Room_Validater` lock table | `rowversion` concurrency |
| `Tbl_Forever_Data` calendar grid table | Computed query from live data |

#### Key entity relationships:

```
Guest ─────────────┬── Reservation
                   ├── CheckIn
                   └── Bill

RoomType ──────────┬── Room
                    └── Tariff

Room ──────────────┬── CheckIn.Slave
                    └── Housekeeping.Status

CheckIn ───────────┬── Bill
                    ├── PostCharge
                    ├── ExtraBed
                    ├── ExtraPax
                    ├── SplitBill
                    └── PlanBill

Bill ──────────────┬── Settlement
                    ├── Tax.Collected
                    └── SplitBill.Line

FinancialYear ─────┬── Bill
                    ├── Settlement
                    └── Advance
```

### 3.3 Database Connections

Only **one** primary database for the HMS core:

```
ConnectionStrings__HMSContext=Server=SANTOSHCOG;Database=ILAPURAM_POs;...
```

The legacy POS, housekeeping, EPABX, and image databases are separate bridges,
deferred to Phase 5+. For now, housekeeping data lives in the main HMS database.

### 3.4 Auth & Security

```csharp
public class AuthController : ApiController
{
    [HttpPost("login")]
    public async Task<IActionResult> Login(LoginRequest request)
    {
        var user = await _context.Users
            .FirstOrDefaultAsync(u => u.UserId == request.Username);
        
        if (user == null || !BCrypt.Verify(request.Password, user.PasswordHash))
            return Unauthorized();
        
        var claims = await GetUserClaims(user);
        var token = GenerateJwt(claims);
        
        return Ok(new { token, user.Name, user.Role });
    }
}
```

- **Password hashing**: BCrypt
- **RBAC**: Migrate `Tbl_Privilages_Master` -> JWT claims + `[Authorize]` attributes
- **Connection strings**: Environment variables / `appsettings.{Environment}.json`
- **API rate limiting**: For SMS/email endpoints

### 3.5 Core Infrastructure

**Middleware pipeline:**
```
Request -> CorrelationId -> ExceptionHandler -> Auth -> RateLimiting -> Controller
```

---

## 4. Phase 2: Core Hotel Operations

### 4.1 Guest Management API

```
GET    /api/guests                          # List/search guests
GET    /api/guests/{id}                     # Guest profile + stay history
POST   /api/guests                          # Create guest
PUT    /api/guests/{id}                     # Update guest
DELETE /api/guests/{id}                     # Soft-delete
GET    /api/guests/{id}/stays               # Stay history
```

### 4.2 Room Management API

```
GET    /api/room-types                      # List room types
GET    /api/room-types/{id}                 # Type + tariff grid
POST   /api/room-types                      # Create
PUT    /api/room-types/{id}                 # Update
DELETE /api/room-types/{id}                 # Soft-delete

GET    /api/rooms                           # List rooms (filter: floor, status, type)
GET    /api/rooms/{id}                      # Room detail
POST   /api/rooms                           # Create
PUT    /api/rooms/{id}                      # Update (status changes)
GET    /api/rooms/availability              # Date x RoomType matrix
GET    /api/rooms/status-dashboard          # Current status counts for dashboard

GET    /api/floors                          # CRUD for floors
POST   /api/floors
PUT    /api/floors/{id}

GET    /api/tariffs                         # Tariff management
POST   /api/tariffs                         # Add rate
PUT    /api/tariffs/{id}                    # Update rate
```

### 4.3 Reservation API

```
GET    /api/reservations                    # Search/list
GET    /api/reservations/{id}               # Full reservation detail
POST   /api/reservations                    # Create (with room allocation)
PUT    /api/reservations/{id}               # Amend
DELETE /api/reservations/{id}               # Cancel
POST   /api/reservations/{id}/confirm       # Confirm
POST   /api/reservations/{id}/allocate      # Allocate rooms
POST   /api/reservations/{id}/checkin       # Convert to check-in
GET    /api/reservations/{id}/instructions  # Special instructions
POST   /api/reservations/{id}/instructions  # Add instruction
```

**Key improvement**: Replace `Tbl_Forever_Data` calendar grid table with a computed view from live `ReservationSlave` + `CheckInSlave` data.

### 4.4 Check-In API

```
POST   /api/checkins                        # Create check-in
GET    /api/checkins/{id}                   # Check-in detail
PUT    /api/checkins/{id}                   # Amend stay (dates, rooms, pax)
POST   /api/checkins/{id}/extend            # Extend stay
POST   /api/checkins/{id}/room-shift        # Shift room
POST   /api/checkins/{id}/block             # Block room
POST   /api/checkins/{id}/extra-bed         # Add extra bed
POST   /api/checkins/{id}/extra-pax         # Add extra person
POST   /api/checkins/{id}/complimentary     # Mark as house guest
GET    /api/checkins/active                 # Current in-house guests
GET    /api/checkins/arrivals/{date}        # Today's arrivals
GET    /api/checkins/departures/{date}      # Today's departures
```

**Key improvement**: Merge `frmCheckIn` + `frmCheckinNew` (21,424 lines of redundant VB). Single endpoint with a `CheckInType` discriminator (WalkIn / Reservation / Group).

---

## 5. Phase 3: Financial & Billing

### 5.1 Billing API

```
GET    /api/bills                            # List/search
GET    /api/bills/{id}                       # Bill detail with line items
GET    /api/bills/{checkinId}/preview        # Preview bill (run calculation)
POST   /api/bills/{checkinId}/settle         # Final settlement
POST   /api/bills/{checkinId}/checkout       # Checkout + generate bill
PUT    /api/bills/{id}/modify                # Modify bill (with audit)
DELETE /api/bills/{id}/cancel                # Cancel/void bill

POST   /api/bills/{id}/split                 # Create split bill
GET    /api/bills/{id}/splits                # List split bills
POST   /api/bills/{id}/splits/{splitId}/settle

GET    /api/bills/outstanding                # Outstanding bills
POST   /api/bills/{id}/transfer              # Transfer to company ledger
```

**Critical extraction**: `GenerateStayingDetails()` in `RoomOperations.vb` (line ~5,305) is the core billing engine. Extract into `BillingService.CalculateBillAsync()`.

### 5.2 Tax Engine

```
GET    /api/taxes                            # Tax master list
POST   /api/taxes                            # Create tax
PUT    /api/taxes/{id}                       # Update
GET    /api/taxes/descriptions               # Tax rate descriptions
POST   /api/taxes/descriptions               # Add rate period
GET    /api/taxes/collected                  # Tax collection report
```

Centralize all tax logic into `TaxCalculationService`:

```csharp
public class TaxCalculationService
{
    public async Task<CalculatedTaxes> CalculateAsync(
        decimal baseAmount, 
        DateOnly effectiveDate, 
        TaxCategory category)
    {
        var rules = await _context.TaxDescriptions
            .Where(t => t.EffectiveDate <= effectiveDate && t.ToDate >= effectiveDate)
            .ToListAsync();
        
        var result = new CalculatedTaxes();
        foreach (var rule in rules)
            result.Add(rule.TaxName, baseAmount * rule.TaxPer / 100m);
        return result;
    }
}
```

### 5.3 Night Audit API

```
POST   /api/night-audit/run                  # Execute daily close
GET    /api/night-audit/status               # Current day status (open/closed)
GET    /api/night-audit/report/{date}        # Night audit report for date
GET    /api/night-audit/history              # Audit history
POST   /api/financial-year/rollover          # Financial year close + new year
```

Transactional batch — never partial-close a day:

```csharp
public async Task<NightAuditResult> RunNightAudit(DateOnly date)
{
    using var tx = await _context.Database.BeginTransactionAsync();
    
    var occupiedRooms = await _context.CheckInSlaves
        .Where(c => c.ArrivalDate <= date && c.DepartureDate >= date)
        .ToListAsync();
    
    var auditEntries = new List<NightAuditEntry>();
    foreach (var stay in occupiedRooms)
    {
        var bill = await _billingService.CalculateDailyAsync(stay, date);
        auditEntries.Add(bill.ToAuditEntry());
    }
    
    await _context.NightAuditEntries.AddRangeAsync(auditEntries);
    await _context.DayClosingStatus.AddAsync(new DayClosing { Date = date, IsClosed = true });
    
    await _context.SaveChangesAsync();
    await tx.CommitAsync();
    
    return new NightAuditResult { ProcessedRooms = auditEntries.Count };
}
```

### 5.4 Settlement & Payments API

```
GET    /api/settlements                      # List settlements
POST   /api/settlements                      # Create settlement
POST   /api/settlements/{id}/refund          # Process refund
POST   /api/settlements/{id}/transfer        # Transfer to company
GET    /api/advances                         # Advance payments
POST   /api/advances                         # Accept advance
POST   /api/advances/{id}/refund             # Refund advance
POST   /api/advances/{id}/transfer           # Transfer to another check-in
```

---

## 6. Phase 4: Supporting Modules

### 6.1 Housekeeping API

```
GET    /api/housekeeping/rooms               # Room status grid
PUT    /api/housekeeping/rooms/{roomId}       # Update cleaning status
POST   /api/housekeeping/receipts            # Post HK charges to guest folio
GET    /api/housekeeping/staff               # Steward/staff management
POST   /api/housekeeping/lost-found          # Lost & found entry
```

Note: Minibar item management deferred or handled as post charges. Legacy `HKEEPINGDB`
is external — housekeeping status data is migrated to the main HMS DB for phase 4.

### 6.2 Reports API

Crystal Reports replaced:

```
GET    /api/reports/{reportId}               # Run report (returns JSON)
GET    /api/reports/{reportId}/pdf           # Download PDF
GET    /api/reports/{reportId}/excel         # Download Excel/CSV
GET    /api/reports/available                # List available reports
```

**PDF**: Puppeteer Sharp / IronPDF. **Excel**: ClosedXML.

Priority reports (cover 90% of usage):
1. Guest bill (checkout)
2. Night audit report
3. Daily sales
4. Guest in-house list
5. House status report
6. GST returns
7. Reservation report
8. Arrival/Departure lists
9. Occupancy report
10. Company sale report

### 6.3 Communication API (SMS + Email only)

```
POST   /api/communications/sms               # Send SMS
POST   /api/communications/email             # Send email
GET    /api/communications/log                # Sent messages log
GET    /api/communications/templates          # Message templates
```

Unified interface:

```csharp
public interface IMessageSender
{
    Task<SendResult> SendAsync(MessageRequest request);
}
public class SmsSender : IMessageSender { /* ... */ }
public class EmailSender : IMessageSender { /* ... */ }
```

WhatsApp is deferred. OTP is deferred (handled as SMS if needed).

### 6.4 Admin & Configuration API

```
GET    /api/admin/users                      # User management
POST   /api/admin/users
PUT    /api/admin/users/{id}
POST   /api/admin/users/{id}/rights          # Set privileges

GET    /api/admin/settings                   # System configuration
PUT    /api/admin/settings                   # Update

POST   /api/admin/backup                     # Trigger database backup
GET    /api/admin/backup/history             # Backup history

GET    /api/admin/financial-years             # FY management
POST   /api/admin/financial-years
```

---

## 7. HMS Entity Map

### Guest Domain
- `Guest` → `Tbl_Guest_Master`
- `ForeignGuest` → `Tbl_ForeignGuest`

### Room Domain
- `RoomType` → `Tbl_Room_Master`
- `Floor` → `Tbl_Floor_Master`
- `Room` → `Tbl_Room_Details`
- `RoomTariff` → `Tbl_Room_Tarrif`
- `GrandTariff` → `Tbl_Grand_Tariff`

### Reservation Domain
- `Reservation` → `Tbl_Reservation_Master`
- `ReservationSlave` → `Tbl_Reservation_Slave`
- `ReservationOther` → `Tbl_Reservation_OtherDetails`
- `ReservationCharge` → `Tbl_Reservation_Charge`
- `ReservationInstruction` → `Tbl_Special_Ins`

### Check-In Domain
- `CheckIn` → `Tbl_CheckIn_Master`
- `CheckInSlave` → `Tbl_CheckIn_Slave`
- `RoomShift` → `Tbl_RoomShifft_Master`
- `CheckInInstruction` → `Tbl_INSTRUCTION`
- `ComplimentUpdate` → `Tbl_Compliment_Update`

### Billing Domain
- `Bill` → `Tbl_NewCheckout_Master`
- `BillDay` → `Tbl_NewCheckout_Slave`
- `PostCharge` → `Tbl_PostCharge_Master`
- `PostChargeLine` → `Tbl_PostCharge_Slave`
- `ExtraBed` → `Tbl_ExtraBed_Master`
- `ExtraPax` → `Tbl_ExtraPax_Master`
- `SplitBill` → `Tbl_Split_Master`
- `SplitBillLine` → `Tbl_Split_Slave`
- `CollectedTax` → `Tbl_CollectedTax`

### Financial Domain
- `FinancialYear` → `Tbl_Financial_Master`
- `DayClosing` → `Tbl_DayClosing`
- `NightAuditEntry` → `Tbl_Night_Audit`
- `Settlement` → `Tbl_NewSettelment_Receipt`
- `Advance` → `Tbl_Advance_Receipt`
- `OutstandingBill` → `Tbl_NewOutstanding_Master`
- `Company` → `Tbl_Company_Master`
- `CompanyTariff` → `Tbl_Company_Tariff`
- `Ledger` → `Tbl_Ledger_Master`
- `LedgerGroup` → `Tbl_LedgerGroup`
- `Bank` → `Tbl_Bank_Master`

### Tax Domain
- `TaxRule` → `Tbl_Tax_Master`
- `TaxDescription` → `Tbl_Tax_Description`

### Plan/Package Domain
- `PlanMaster` → `Tbl_PlanMaster`
- `PlanBill` → `Tbl_Plan_Bill`

### Reference/Master Data
- `State` → `Tbl_State_Master`
- `Area` → `Tbl_Area_Master`
- `Reference` → `Tbl_Ref_Master`
- `Segment` → `Tbl_Segment_Master`
- `SegmentSub` → `Tbl_Segment_Slave`
- `Instruction` → `Tbl_Instruction_Master`
- `RevenueHead` → `Tbl_Revenu_Master`
- `PaymentMode` → from combo lists (create entity)

### System/Config Domain
- `User` → `Tbl_User_Master`
- `UserPrivilege` → `Tbl_Privilages_Master`
- `SystemConfig` → `Tbl_SystemConfigure`
- `HotelInfo` → `Tbl_Purcharser_Detalis`
- `SmsGateway` → `Tbl_Sms_Sender`
- `SmsLog` → `Tbl_Sms_Track`

---

## 8. Database Migration Steps

1. Add audit columns to all tables (`CreatedAt`, `UpdatedAt`, `CreatedBy`, `UpdatedBy`)
2. Add proper identity columns, populate from `RNo`, make primary key
3. Add foreign keys (`CheckInSlave.CheckInNo` -> `CheckInMaster.Id`, etc.)
4. Normalize status columns into lookup tables (`RoomStatusTypes`, `BillStatusTypes`)
5. Hash existing passwords with BCrypt (one-time migration script)
6. Drop all `Tmp_*`, `Temp_*`, `Tbl_Temp*` staging tables
7. Add `rowversion` column to high-concurrency tables (rooms, checkins)
8. Remove tables for deferred external systems from the HMS migration scope

---

## 9. Key Technical Decisions

| Decision | Choice | Rationale |
|---|---|---|
| API versioning | URL-based (`/api/v1/`) | Avoids header complexity |
| DTO mapping | Manual for complex, AutoMapper for simple CRUD | Bill/checkin DTOs are too complex for auto-mapping |
| Validation | FluentValidation | Keeps rules out of controllers |
| Logging | Serilog + Windows Event Log | Replaces `EventLogger`/`ErrorLogger` VB modules |
| Background jobs | Hangfire | Night audit, auto-backup, scheduled SMS |
| Real-time updates | SignalR | Room status dashboard |
| API docs | Swagger/Scalar | Auto-generated |
| Testing | xUnit + Moq + Testcontainers (SQL) | Billing engine is highest test priority |
| CI/CD | GitHub Actions | Build, test, deploy |
| Deployment | ASP.NET Core on IIS or Windows Service | On-prem Windows Server |

---

## 10. Deployment on Windows Server

The ASP.NET Core backend runs on the same Windows Server (IIS or Windows Service).

### IIS Hosting (Recommended)
- Install .NET Hosting Bundle + IIS ASP.NET Core Module
- `hostingModel="inprocess"` — runs inside `w3wp.exe`
- App Pool: No Managed Code, Idle Time-out 0 (24/7), recycle at 3 AM
- SignalR: Enable WebSocket Protocol in IIS

### Windows Service (Alternative)
```
dotnet publish -c Release -o C:\apps\coghms-api --self-contained -r win-x64
nssm install CogHMS-API C:\apps\coghms-api\CogHMS.Api.exe
```

### Connection Strings & Secrets
- Environment variables (machine-level) or IIS app settings
- Secrets (JWT keys, SMS passwords, SMTP credentials) NOT in `appsettings.json`
- Use Windows DPAPI or environment variables

### CORS
Configure for Next.js frontend on same network:
```csharp
policy.WithOrigins("http://localhost:3000", "http://192.168.1.10:3000", "https://coghms.local")
    .AllowAnyHeader().AllowAnyMethod().AllowCredentials();
```

---

## 11. Risk Areas

| Risk | Mitigation |
|---|---|
| Billing calculation errors | Exhaustive test cases against legacy output |
| Tax compliance (GST) | Hotel accountant validation against known bills |
| Night audit corruption | Atomic transactions, never partial-close a day |
| Split bill complexity | Unit test every combination (single/group/company/partial) |
| Data migration errors | Run old and new systems in parallel for 2 weeks |
| Deferred integrations | Keep legacy VB running alongside for POS/EPABX/door locks until bridge is ready |

---

## 12. Deliverables by Phase

### Phase 1 (Foundation)
- [ ] Solution scaffolded with 4 projects
- [ ] HMS entities with EF configurations (~45 entities)
- [ ] Auth controller with JWT
- [ ] Database migration baseline
- [ ] Swagger UI
- [ ] CI pipeline

### Phase 2 (Core Operations)
- [ ] Guest CRUD API
- [ ] Room + Floor + Tariff API
- [ ] Reservation API
- [ ] Check-In API
- [ ] Availability matrix endpoint
- [ ] Room status dashboard endpoint + SignalR

### Phase 3 (Financial)
- [ ] Billing calculation engine
- [ ] Checkout API
- [ ] Tax engine
- [ ] Split bill API
- [ ] Settlement + Advance API
- [ ] Night audit API
- [ ] Financial year management API

### Phase 4 (Supporting)
- [ ] Housekeeping API
- [ ] 10 priority reports (PDF + Excel)
- [ ] Communication API (SMS + Email)
- [ ] Admin + Config API
- [ ] Masters CRUD (companies, taxes, plans, ledgers, segments, references)

### Phase 5+ (Deferred Bridges)
- [ ] POS read-only bridge (`ILAPURAM_POs` DB)
- [ ] EPABX call-log bridge (`Epabx` DB)
- [ ] Tally push bridge (`BAARE2013` DB)
- [ ] Hardware bridge agent (fingerprint, door lock, printer)
- [ ] Channel Manager / OTA sync
- [ ] WhatsApp Business API