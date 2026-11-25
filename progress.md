# Core HMS implementation progress

Last updated November 24, 2026.

## Phase and module status

| Phase | Module | Status | Evidence and remaining work |
| --- | --- | --- | --- |
| 1 · Backend and database baseline | Backend and API inventory | Verified | .NET build succeeded with zero errors. The reference matches 158 operations in the OpenAPI snapshot. |
| 1 · Backend and database baseline | Tenant SQL schema | Implemented, awaiting SQL verification | The check-in upgrade adds the master, room detail, and discount tables and two save procedures. Both procedure parameter lists match the Dapper calls. Apply the upgrade to an isolated tenant and verify it there. |
| 1 · Backend and database baseline | Authentication and permissions | In progress | The active check-in endpoint returned HTTP 401 without a token. Create check-in now checks the request branch against the JWT. Authenticated tenant isolation and permission checks remain open. |
| 2 · Front desk | Guests, rooms, tariffs, availability, and reservations | Existing, awaiting workflow verification | The backend exposes these operations. Verify representative front-desk flows against the tenant database. |
| 2 · Front desk | Check-in and active stays | In progress | Added active-stay listing and lookup, date and room validation, a scoped duplicate lookup, and a transactional room assignment guard. Reservation conversion now reconciles room counts per room type. Verify walk-ins, reservation conversions, group stays, and concurrent allocation. |
| 2 · Front desk | Stay servicing | Pending | Complete amendments, extensions, room moves, complimentary stays, and guest charges. |
| 3 · Billing and checkout | Billing, checkout, and payments | In progress | Added a room tariff and tax estimate for active stays. Full bill preview, checkout, invoice PDF, bill history, adjustments and voids, split bills, settlements, advances, refunds, transfers, and company balances remain open. Verify legacy billing parity and checkout rollback. |
| 4 · Daily hotel operations | Night audit and financial-year rollover | Pending | Implement and verify a single close per tenant and business date, recovery after failure, and rollover. |
| 4 · Daily hotel operations | Housekeeping, lost and found, and dashboards | Pending | Complete housekeeping and dashboard workflows for arrivals, departures, occupancy, room status, and revenue. |
| 5 · Reporting and HMS administration | Reports and exports | Pending | Deliver the ten priority reports in JSON, PDF, and Excel and reconcile totals with legacy output. |
| 5 · Reporting and HMS administration | Settings, permissions, master data, and backup | Pending | Close verified gaps, rehearse the upgrade, and pilot at one property. |

## Open blocker

The configured server master database is reachable. Read-only connections to BR001 and BR002 found all five check-in objects. The SH001 connection check failed, and workflow tests remain open. IIS deployment is blocked because the current Windows account cannot read IIS configuration.

## Update log

| Date | Phases | Changes and verification |
| --- | --- | --- |
| 2026-11-20 | 1, 2 | Built the backend, documented the existing API, and added branch checks, check-in validation, transactional room assignment protection, and the active-stay route. Added the [check-in SQL upgrade](../DataBase/001-HMS-CheckIn.sql). Verified the procedure parameter lists, refreshed the [API reference](hms-api-reference.md) and [OpenAPI snapshot](hms-openapi.json), and confirmed HTTP 401 without a token on the new route. |
| 2026-11-23 | 2 | Added active check-in details and fixed reservation conversions that select multiple rooms of the same type. Check-in now uses the selected primary guest for GRC and recognizes the legacy vacant-room marker. The backend built with zero errors. Updated the frontend reference and OpenAPI snapshot from the controller contract after localhost verification timed out. Tenant database verification remains open. |
| 2026-11-24 | 3 | Added a room charge estimate using stored tariffs and the existing tax service. Updated the API reference and OpenAPI snapshot from the controller contract and verified the build and catalog coverage. Tenant database verification and legacy billing parity remain open. |
| 2026-11-24 | 1, 3 | Published a Release build with IIS configuration. Verified the server master connection and check-in object presence for BR001 and BR002 using read-only queries. IIS configuration access was denied, and the SH001 connection check failed. No database upgrade was applied. |
| 2026-11-24 | 1, 2, 3 | Started the published build on localhost and confirmed that Swagger exposes 158 API operations. The countries route returned HTTP 200. Active check-in listing, check-in details, and the room charge estimate returned HTTP 401 without a token. Authenticated BR001 workflow tests require a test login. |
| 2026-11-24 | 1, 2, 3 | Tried BR001 login against the published build. An empty login request returned HTTP 400. Requests with the supplied login hit a transport error before a valid HTTP response arrived, so no JWT was obtained and authenticated tests remain open. |
| 2026-11-24 | 1, 2 | Fixed unset HTTP status codes in login responses. Re-published and verified HTTP 200 with a JWT for BR001 / ABCH1. Of 40 authenticated read checks, 37 returned HTTP 200. Tax description and property lists returned 400, and reservation tax rules returned 500. Mutation tests remain open. |
| 2026-11-24 | 2 | Fixed the SQL identity return in floor creation. Verified BR001 / ABCH1 floor creation (201), update (200), and read-back on the same disposable record. Cleanup returned 200 and a read-back confirmed the record was removed. Other mutation tests remain open. |

Update the affected module rows and add a dated log entry with each implementation change. Record what was verified, what remains open, and any change to the frontend API contract.
