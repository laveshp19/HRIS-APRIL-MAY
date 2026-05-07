# Meeting Notes — 27th April 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Roly (Roni)** — Developer (Frontend / UI)

---

## Agenda
Discussion on the **HRIS Leave Module** — specifically the Leave Summary API, Leave Application API, and Policy Bundle staff setup in the SchoolERP system.

---

## Discussion Points

### 1. Leave Summary API — Data Source & Usage

- **Context:** The team needed to know where to fetch leave summary data for any given staff member.
- **Decision:** Use the existing endpoint available in the APS (Admin Panel System). Pass `StaffID` and `AcademicYearID` as parameters.
- **Leave Summary definition clarified:**
  - Shows how many leaves were **allotted** to a staff member.
  - How many have been **availed (used)**.
  - How much **balance** remains.
  - This maps to the stored procedure `SP_GetLeaveSummaryByStaffID`, which joins `HRISLeaveDetails`, `HRISLeaveMaster`, and `HRISPolicyBundle` tables.
- **Key fields returned:** `LeaveMasterID`, `LeaveName`, `LeaveAlias`, `AllottedLeaves`, `AvailedLeaves`, `PreApprovedLeaves`, `PendingLeaves`, `BalanceLeaves`.

### 2. Leave Summary vs. Leave Application — Clarification

- **Leave Summary:** Displays the balance sheet — allotted, availed, and remaining leaves per leave type (e.g., Casual Leave, Privilege Leave).
- **Leave Application:** Deals with the actual leave requests — `FromDate`, `ToDate`, `TotalDays`, status (Pending, Approved, Rejected). This is a separate API (`GetLeaveApplication`).
- **Instruction:** Both Leave Summary and Leave Application data should be **merged** in the MyHRIS view so the staff can see their balance alongside their application history.

### 3. Leave Application API — Testing Issues

- **Parameters required:** `StaffID` and `AcademicYearID`.
- **Issue observed:** When testing with a specific staff member (RP Share / Arti Sao's StaffID), no data was returned from the Leave Application endpoint.
- **Root cause identified:** The staff member had **not been assigned to a Policy Bundle** yet. Without a policy bundle assignment, no leaves are allotted, so no balance or application data exists.
- **Action item:** The staff member needs to be assigned to a policy bundle first (via the HRIS Policy Bundle setup screen) before the Leave Summary and Leave Application APIs can return data.

### 4. Policy Bundle — Staff Assignment Issues

- **Problem:** When searching for a staff member (e.g., "Arti" / "RP Share") in the HRIS Policy Bundle setup screen, the staff was **not appearing** in the search results.
- **Reason:** The staff was on a **test account**, and the system's "All Staff" list filters out test accounts by design. This was an intentional feature to prevent test data from appearing in production-like views.
- **Workaround discussed:** Navigate to the HR System Policy Bundle panel directly, and try adding the staff from there.
- **Secondary issue:** The staff member's record was also missing the **Department/Team** assignment, which is a prerequisite for the bundle assignment flow.

### 5. Staff Add to Policy Bundle — Bad Request Error

- **Error observed:** When trying to add staff to the policy bundle, a **Bad Request (400)** error was returned.
- **Root cause:** Required parameters were **not being passed** correctly in the API request.
- **Additional issue:** The developer (Lavesh) had made local changes but had **not pushed** them to the server/online environment. The live system was still running the old code.
- **Action item:** Push changes first, then retest. Verify that:
  1. Staff can be added to the policy bundle successfully.
  2. The response is correctly stored in the database.
  3. The UI reflects the data accurately.

### 6. Team Coordination — Process Guidance

- **Senior emphasized the importance of teamwork and coordination:**
  - If one team member is confused about an API's parameters or behavior, they should **sit with the other developer** and debug together.
  - No blame culture — the focus is on **"things should keep moving, nothing should get stuck."**
  - Both Lavesh (backend) and Roly (frontend) should coordinate on:
    - What parameters to pass in API calls.
    - What the expected response looks like.
    - Verifying GET, DELETE, and other operations together.
  - If the two developers cannot reach alignment on their own, they should **call the senior** for help.

---

## Action Items

| # | Task | Assigned To | Status |
|---|------|------------|--------|
| 1 | Test Leave Summary API with a valid StaffID (one that has a policy bundle assigned) | Lavesh | Pending |
| 2 | Assign test staff (RP Share / Arti Sao) to a Policy Bundle with proper Department/Team setup | Roly | Pending |
| 3 | Debug and fix the Bad Request error on "Add Staff to Policy Bundle" — ensure all parameters are passed correctly | Lavesh | Pending |
| 4 | Push local changes to the live/online environment before testing | Lavesh | Pending |
| 5 | Verify the `GetApplicableBundleByStaffID` endpoint returns correct data after bundle assignment | Lavesh / Roly | Pending |
| 6 | Coordinate together on any remaining API confusion — figure out additional required fields | Lavesh & Roly | Pending |

---

## Technical References

| Item | Reference |
|------|-----------|
| Leave Summary SP | `SP_GetLeaveSummaryByStaffID` (SchoolERPAdminDB) |
| Leave Application Table | `HRISLeaveApplication` |
| Policy Bundle Table | `HRISPolicyBundle` |
| Leave Details Table | `HRISLeaveDetails` |
| Leave Master Table | `HRISLeaveMaster` |
| Applicable Bundle API | `GetApplicableBundleByStaffID` |
| Policy Bundle Controller | `HRISPolicyBundleMasterController.cs` |

---

## Key Takeaway
> The main motive is that things should keep running — nothing should get stuck. Both developers must coordinate as a team. If confusion persists, escalate to the senior immediately.
