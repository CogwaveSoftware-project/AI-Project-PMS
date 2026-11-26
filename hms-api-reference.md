# CogHMS frontend API reference

This reference covers routes exposed by the local FrontOfficeApplication build. The accompanying [OpenAPI snapshot](hms-openapi.json) includes exact request schemas, nested fields, parameter types, and content types. A running instance also serves Swagger UI at `/swagger/index.html` and OpenAPI JSON at `/swagger/v1/swagger.json`.

## Authentication and tenant selection

1. Use `GET /api/UserAccessMaster/GetBranchesByMasterUser` to retrieve branches for the supplied credentials. Use HTTPS and keep the credential-bearing URL out of client logs and analytics.
2. Send the credentials and selected branch to `POST /api/UserAccessMaster/BtnSubmitLogin` using `LoginBranchRequest`. Read the JWT from `data.jwtToken`. Check `data.serialKeyExpired` and require a token before entering the authenticated UI. The API can return a success envelope without a token when the property lacks a licence.
3. Send `Authorization: Bearer <token>` on protected calls. The backend selects the tenant database from the token's `Branch_code` claim. Supply the matching branch and property where the endpoint declares those inputs. Preserve each endpoint's parameter location and naming from OpenAPI.
4. Treat HTTP 401 as an authentication failure. On HTTP 400, check for the response envelope, ASP.NET validation errors, or a plain error string.

## Response handling

Many controller actions return a JSON envelope shaped like `{"success":true,"message":"Success","data":{}}`. The shape of `data` varies by action. The current Swagger output describes requests more fully than responses because many controller actions return `IActionResult`. Confirm untyped response payloads against a test instance before binding UI fields. Use the action-specific content type for file operations.

## Front-desk calls

- Search for a guest with `GetGuestSearchList`, then check room inventory with `GetRoomAvailability` or `GetChartAvailability`. Use the reservation lookup and pricing endpoints to populate booking forms.
- Create a reservation with `CreateReservationDetail`. Retrieve its details with `GetReservationCompleteDetails`. Cancel using `BookingCancellationDetail`.
- Submit check-in with `CreateCheckInDetail`. Include a named guest, valid stay dates, and a distinct room number for each booking room. Send dates as ISO date/time values and time-of-day fields as .NET time strings such as `14:00:00`. For a reservation conversion, include the reservation number. Success returns HTTP 201 with a string in `data`. HTTP 400 can contain a plain error string. A branch code that differs from the authenticated branch returns HTTP 403.
- Fetch occupied rooms with `GetActiveCheckInList?propertyCode=...`. The branch comes from the JWT. HTTP 200 returns an envelope with one row per occupied room in `data`, including `checkInNo`, `roomNo`, `guestCode`, `arrivalDate`, and `departureDate`. An empty result returns `data: []`.
- Fetch a single active stay with `GetCheckInDetails?propertyCode=...&checkInNo=...`. It returns the same room-row format for that check-in, including multiple rows for a group stay. HTTP 404 indicates there are no active rooms for that check-in in the requested property and authenticated branch.
- For a reservation conversion, send one booking row per selected room. For each room type selected, the room count must match its unconverted reservation inventory.
- The check-in endpoints require the tenant schema from [001-HMS-CheckIn.sql](../DataBase/001-HMS-CheckIn.sql), applied after the HMS baseline.

## Route catalog

The tables list the existing controller actions by module. For each named request model, use the matching definition under `components.schemas` in the OpenAPI snapshot. The snapshot contains the development WeatherForecast route as well.

### UserAccessMaster

Login, property and branch selection, user administration, menus, and permissions.

`BtnSubmitLogin` takes a JSON object with `username`, `password`, `branchCode`, and a numeric `roleId`. Include `propertyCode` when the user has selected a property. When supplied, it must match the user's property. An invalid credential or property combination returns HTTP 401.

The three bulk permission and rights operations take a top-level JSON array of the specified model. `ChangeUserPassword` currently takes its three inputs in the query string. Use HTTPS and keep credential-bearing URLs out of logs.

User creation returns HTTP 201. User updates, deletion, and password changes return HTTP 200 on success. Permission and rights saves return HTTP 201 for both inserts and updates. Read back submenu permissions through `data.menus[].subMenus`, access rights through `data.accessRights`, and operation rights through `data.operationRights`. The general submenu list groups `subMenus` by `mainMenuId`.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/UserAccessMaster/GetBranchesByMasterUser` | Username (query), Password (query) |
| `POST` | `/api/UserAccessMaster/BtnSubmitLogin` | LoginBranchRequest |
| `GET` | `/api/UserAccessMaster/GetUserMasterList` | PropertyCode (query) |
| `POST` | `/api/UserAccessMaster/CreateUserMaster` | UserMaster |
| `PUT` | `/api/UserAccessMaster/UpdateUserMaster` | UserMaster |
| `DELETE` | `/api/UserAccessMaster/DeleteUserMaster` | UserCode (query), PropertyCode (query) |
| `PUT` | `/api/UserAccessMaster/ChangeUserPassword` | UserCode (query), PropertyCode (query), NewUserPassword (query) |
| `GET` | `/api/UserAccessMaster/GetBranchesList` | None |
| `GET` | `/api/UserAccessMaster/GetPropertiesList` | None |
| `GET` | `/api/UserAccessMaster/GetRoleMasterList` | None |
| `GET` | `/api/UserAccessMaster/GetMainMenuList` | None |
| `GET` | `/api/UserAccessMaster/GetSubMenuList` | None |
| `POST` | `/api/UserAccessMaster/SaveMenuWithSubMenu` | MenuWithSubMenuModel |
| `DELETE` | `/api/UserAccessMaster/DeleteMainMenuDetail` | MainMenuId (query) |
| `DELETE` | `/api/UserAccessMaster/DeleteSubMenuDetail` | SubMenuId (query) |
| `GET` | `/api/UserAccessMaster/GetUserPermissionAccessList` | PropertyCode (query), UserCode (query), RoleId (query) |
| `POST` | `/api/UserAccessMaster/InsertUserPermissionAccessMaster` | UserPermissionAccessMaster[] |
| `GET` | `/api/UserAccessMaster/GetAccessRightsList` | None |
| `GET` | `/api/UserAccessMaster/GetOperationRightsList` | None |
| `POST` | `/api/UserAccessMaster/SaveAccessOROperationRights` | RightsMasterRequest |
| `DELETE` | `/api/UserAccessMaster/DeleteAccessRightsDetail` | AccessRightId (query) |
| `DELETE` | `/api/UserAccessMaster/DeleteOperationRightsDetail` | OperationRightId (query) |
| `GET` | `/api/UserAccessMaster/GetUserAccessRightsList` | PropertyCode (query), UserCode (query), RoleId (query) |
| `POST` | `/api/UserAccessMaster/InsertUserAccessRightsMaster` | UserAccessRightsMaster[] |
| `GET` | `/api/UserAccessMaster/GetUserOperationRightsList` | PropertyCode (query), UserCode (query), RoleId (query) |
| `POST` | `/api/UserAccessMaster/InsertUserOperationRightsMaster` | UserOperationRightsMaster[] |

### Master

Hotel reference data and master record maintenance, including rooms, tariffs, companies, financial years, and taxes.

The room detail and company tariff create and update routes take top-level JSON arrays. Use one request object per room or company tariff row. Other Master writes use the request shape listed below.

For a room-detail update, send `roomDetailId` and the existing `oldRoomNo`, `oldRoomCode`, and `oldFloorCode` along with the requested values. For payment modes, `SaveMainandSubPaymentMode` creates a main mode or appends a submode to an existing main mode. There is no edit route for an existing mode. The submode list groups its `subModes` by `mainModeId`.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/Master/GetFloorMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateFloorMasterDetail` | FloorMasterRequestModel |
| `PUT` | `/api/Master/UpdateFloorMasterDetail` | FloorMasterRequestModel |
| `DELETE` | `/api/Master/DeleteFloorMasterDetail` | FloorId (query), FloorCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetFinancialMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/SaveorUpdateFinancialMasterDetail` | FinancialMasterModel |
| `DELETE` | `/api/Master/DeleteFinancialMasterDetail` | FinId (query), PropertyCode (query) |
| `GET` | `/api/Master/GetRoomTypeMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateRoomTypeMasterDetail` | RoomTypeMasterRequestModel |
| `PUT` | `/api/Master/UpdateRoomTypeMasterDetail` | RoomTypeMasterRequestModel |
| `DELETE` | `/api/Master/DeleteRoomTypeMasterDetail` | RoomId (query), RoomCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetPlanMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreatePlanMasterDetail` | PlanMasterModel |
| `PUT` | `/api/Master/UpdatePlanMasterDetail` | PlanMasterModel |
| `DELETE` | `/api/Master/DeletePlanMasterDetail` | PlanId (query), PlanCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetRoomDetailMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateRoomDetailMasterDetail` | RoomDetailMasterRequestModel[] |
| `PUT` | `/api/Master/UpdateRoomDetailMasterDetail` | RoomDetailMasterRequestModel[] |
| `DELETE` | `/api/Master/DeleteRoomDetailMasterDetail` | RoomDetailId (query), RoomNo (query), RoomCode (query), FloorCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetPurposeMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreatePurposeMasterDetail` | PurposeMasterModel |
| `PUT` | `/api/Master/UpdatePurposeMasterDetail` | PurposeMasterModel |
| `DELETE` | `/api/Master/DeletePurposeMasterDetail` | PurposeId (query), PurposeCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetPaymentMainMasterList` | PropertyCode (query) |
| `GET` | `/api/Master/GetPaymentSubMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/SaveMainandSubPaymentMode` | PaymentModeRequestModel |
| `DELETE` | `/api/Master/DeleteMainPaymentModeDetail` | MainModeId (query), PropertyCode (query) |
| `DELETE` | `/api/Master/DeleteSubPaymentModeDetail` | SubModeId (query), PropertyCode (query) |
| `GET` | `/api/Master/GetIdentityproofMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateIdentityproofMasterDetail` | IdentityproofMasterModel |
| `PUT` | `/api/Master/UpdateIdentityproofMasterDetail` | IdentityproofMasterModel |
| `DELETE` | `/api/Master/DeleteIdentityproofMasterDetail` | IdentityproofId (query), IdentityproofCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetTaxMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateTaxMasterDetail` | TaxMasterModel |
| `PUT` | `/api/Master/UpdateTaxMasterDetail` | TaxMasterModel |
| `DELETE` | `/api/Master/DeleteTaxMasterDetail` | TaxId (query), TaxCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetTaxDescriptionMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateTaxDescriptionMasterDetail` | TaxDescriptionMasterModel |
| `PUT` | `/api/Master/UpdateTaxDescriptionMasterDetail` | TaxDescriptionMasterModel |
| `DELETE` | `/api/Master/DeleteTaxDescriptionMasterDetail` | TaxDescriptionId (query), TaxDescriptionCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetCompanyMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateCompanyMasterDetail` | CompanyMasterModel |
| `PUT` | `/api/Master/UpdateCompanyMasterDetail` | CompanyMasterModel |
| `DELETE` | `/api/Master/DeleteCompanyMasterDetail` | CompanyId (query), CompanyCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetCompanyTraiffMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateCompanyTraiffMasterDetail` | CompanyTraiffRequestModel[] |
| `PUT` | `/api/Master/UpdateCompanyTraiffMasterDetail` | CompanyTraiffRequestModel[] |
| `DELETE` | `/api/Master/DeleteCompanyTraiffMasterDetail` | CompanyTraiffId (query), RoomCode (query), CompanyCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetRevenueMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateRevenueMasterDetail` | RevenueMasterModel |
| `PUT` | `/api/Master/UpdateRevenueMasterDetail` | RevenueMasterModel |
| `DELETE` | `/api/Master/DeleteRevenueMasterDetail` | RevenueId (query), RevenueCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetStewardMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateStewardMasterDetail` | StewardMasterModel |
| `PUT` | `/api/Master/UpdateStewardMasterDetail` | StewardMasterModel |
| `DELETE` | `/api/Master/DeleteStewardMasterDetail` | StewardId (query), StewardCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetItemMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateItemMasterDetail` | ItemMasterModel |
| `PUT` | `/api/Master/UpdateItemMasterDetail` | ItemMasterModel |
| `DELETE` | `/api/Master/DeleteItemMasterDetail` | ItemId (query), ItemCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetSegmentMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateSegmentMasterDetail` | SegmentMasterModel |
| `PUT` | `/api/Master/UpdateSegmentMasterDetail` | SegmentMasterModel |
| `DELETE` | `/api/Master/DeleteSegmentMasterDetail` | SegmentId (query), SegmentCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetSubSegmentMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateSubSegmentMasterDetail` | SubSegmentMasterModel |
| `PUT` | `/api/Master/UpdateSubSegmentMasterDetail` | SubSegmentMasterModel |
| `DELETE` | `/api/Master/DeleteSubSegmentMasterDetail` | SubSegmentId (query), SubSegmentCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetGuestMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateGuestMasterDetail` | GuestMasterModel |
| `PUT` | `/api/Master/UpdateGuestMasterDetail` | GuestMasterModel |
| `DELETE` | `/api/Master/DeleteGuestMasterDetail` | GuestId (query), GuestCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetFormCMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateFormCMasterDetail` | FormCRequestModel |
| `PUT` | `/api/Master/UpdateFormCMasterDetail` | FormCRequestModel |
| `DELETE` | `/api/Master/DeleteFormCMasterDetail` | FormCId (query), GuestCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetLedgerMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateLedgerMasterDetail` | LedgerMasterModel |
| `PUT` | `/api/Master/UpdateLedgerMasterDetail` | LedgerMasterModel |
| `DELETE` | `/api/Master/DeleteLedgerMasterDetail` | LedgerId (query), LedgerCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetTaxRulesMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateTaxRulesMasterDetail` | TaxRulesMasterModel |
| `PUT` | `/api/Master/UpdateTaxRulesMasterDetail` | TaxRulesMasterModel |
| `DELETE` | `/api/Master/DeleteTaxRulesMasterDetail` | TaxRulesId (query), TaxRulesCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetDriverMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateDriverMasterDetail` | DriverMasterModel |
| `PUT` | `/api/Master/UpdateDriverMasterDetail` | DriverMasterModel |
| `DELETE` | `/api/Master/DeleteDriverMasterDetail` | DriverId (query), DriverCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetBookingSourceMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateBookingSourceMasterDetail` | BookingSourceMasterModel |
| `PUT` | `/api/Master/UpdateBookingSourceMasterDetail` | BookingSourceMasterModel |
| `DELETE` | `/api/Master/DeleteBookingSourceMasterDetail` | BookingSourceId (query), BookingSourceCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetInstructionDepartmentMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateInstructionDepartmentMasterDetail` | InstructionDepartmentMasterModel |
| `PUT` | `/api/Master/UpdateInstructionDepartmentMasterDetail` | InstructionDepartmentMasterModel |
| `DELETE` | `/api/Master/DeleteInstructionDepartmentMasterDetail` | InstructionDepartmentId (query), InstructionDepartmentCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetSalesExecutiveMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateSalesExecutiveMasterDetail` | SalesExecutiveMasterModel |
| `PUT` | `/api/Master/UpdateSalesExecutiveMasterDetail` | SalesExecutiveMasterModel |
| `DELETE` | `/api/Master/DeleteSalesExecutiveMasterDetail` | SalesExecutiveId (query), SalesExecutiveCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetReservationPayModeMasterList` | PropertyCode (query) |
| `POST` | `/api/Master/CreateReservationPayModeMasterDetail` | ReservationPayModeMasterModel |
| `PUT` | `/api/Master/UpdateReservationPayModeMasterDetail` | ReservationPayModeMasterModel |
| `DELETE` | `/api/Master/DeleteReservationPayModeMasterDetail` | ReservationPayModeId (query), ReservationPayModeCode (query), PropertyCode (query) |
| `GET` | `/api/Master/GetNextMasterCode` | TableName (query), CodeColumn (query), Prefix (query), PropertyCode (query) |

### Reservation

Guest search, availability, booking details, pricing lookups, reservation creation, and cancellation.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/Reservation/GetChartAvailability` | NoDays (query), checkInDate (query), PropertyCode (query) |
| `GET` | `/api/Reservation/GetRoomAvailability` | noDays (query), checkInDate (query), PropertyCode (query) |
| `GET` | `/api/Reservation/GetTaxDetails` | PropertyCode (query) |
| `GET` | `/api/Reservation/GetTaxRulesDetails` | PropertyCode (query) |
| `GET` | `/api/Reservation/GetPlanDetails` | PropertyCode (query) |
| `GET` | `/api/Reservation/GetRoomTraiffDetails` | PropertyCode (query) |
| `GET` | `/api/Reservation/GetCompanyTraiffDetails` | PropertyCode (query) |
| `GET` | `/api/Reservation/GetCombinedReservationDetails` | PropertyCode (query) |
| `POST` | `/api/Reservation/CreateReservationDetail` | ReservationRequestModel |
| `POST` | `/api/Reservation/BookingCancellationDetail` | BookingCancellationModel |
| `GET` | `/api/Reservation/GetGuestSearchList` | SearchingName (query), PropertyCode (query) |
| `GET` | `/api/Reservation/GetReservationCompleteDetails` | GuestCode (query), ReservationNo (query), PropertyCode (query) |

### CheckIn

Front-desk check-in and active-stay queries.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `POST` | `/api/CheckIn/CreateCheckInDetail` | CheckInRequestModel |
| `GET` | `/api/CheckIn/GetActiveCheckInList` | propertyCode (query) |
| `GET` | `/api/CheckIn/GetCheckInDetails` | propertyCode (query), checkInNo (query) |

### TaxCalculation

Dynamic tax calculation for the supplied amounts and effective date.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `POST` | `/api/TaxCalculation/CalculateDynamicTaxes` | TaxCalculationRequestDto |

### FileStorage

Image upload, lookup, retrieval, and deletion.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/FileStorage/GetFileStorageList` | TableCode (query), PropertyCode (query) |
| `GET` | `/api/FileStorage/GetFileStorageById` | FileId (query), TableCode (query), PropertyCode (query) |
| `POST` | `/api/FileStorage/UploadImageFile` | Multipart form |
| `DELETE` | `/api/FileStorage/DeleteImageFileDetail` | FileId (query), TableCode (query), PropertyCode (query) |

### Locations

Country and state lookups.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/Locations/countries` | None |
| `GET` | `/api/Locations/states/{countryCode}` | countryCode (path) |

### GST

GST verification.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/GST/GSTVerification` | GSTIn (query), PropertyCode (query) |

### ProductLicence

Existing licensing endpoints.

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/ProductLicence/GetProductLicenceKey` | branchcode (query) |
| `POST` | `/api/ProductLicence/SaveProductLicenceKey` | InsertProductLicenceModel |
| `POST` | `/api/ProductLicence/decode` | EncodedProductkey (query) |

### Billing

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `POST` | `/api/Billing/GetRoomChargeEstimate` | RoomChargeEstimateRequest |

Send `propertyCode`, `checkInNo`, and `isForeignOrInterstateClient` in the JSON body. The JWT supplies the branch. HTTP 200 returns room-by-date lines and tariff, tax, and estimated totals in `data`. HTTP 404 indicates that the stay has no active rooms in the requested property and branch.

Use this endpoint for an indicative room tariff and tax display. It uses the recorded tariff and scheduled stay dates, charging arrival through the night before departure, with one charge for a same-day stay. The estimate excludes discounts, extras, posted charges, advances, package and room-move adjustments, and legacy checkout-time rules. The frontend should keep settlement and final bill preview pending.

### Dashboard

| Method | Route | Request model or parameters |
| --- | --- | --- |
| `GET` | `/api/Dashboard/GetRoomStatusDashboard` | propertyCode (query, required, up to 5 characters) |
| `GET` | `/api/Dashboard/GetScheduledRoomMovements` | propertyCode and businessDate (query, required) |

Send a bearer token. The server uses its branch claim and the requested property to count active primary room records. HTTP 200 returns `totalRooms`, `occupiedRooms`, `vacantRooms`, `occupancyPercent`, and `statusCounts` in `data`. Each status row contains `statusCode` and `roomCount`. The backend counts `O` and `F` as occupied and `V` as vacant. Occupancy is occupied rooms divided by all active primary rooms, expressed as a percentage. An empty inventory returns zero totals and an empty `statusCounts` array.

For scheduled movements, send `businessDate` as `YYYY-MM-DD`. HTTP 200 returns `businessDate`, `arrivals`, and `departures` in `data`. Arrival rows contain `reservationNo`, `roomCode`, and `roomCount` for confirmed, unconverted inventory. Departure rows contain `checkInNo` and `roomNo` for occupied rooms due out on the selected date. Lists are empty when no rows qualify. A missing date returns HTTP 400.

## HMS workflows pending APIs

Checkout, bill preview and history, split bills, standalone settlement and advance operations, night audit, financial-year rollover, housekeeping, revenue dashboards, report export, and backup administration have no dedicated routes in this snapshot.
