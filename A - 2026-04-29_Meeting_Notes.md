# Meeting Notes — 29th April 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Roly (Roni)** — Developer (Frontend / UI)

---

## Agenda
Follow-up on the **Leave Summary API output cleanup**, **Leave Application API refinement**, duplicate leave data issue, **Policy Bundle controller restructuring**, and **"Enabled by HR" leave type** handling.

---

## Discussion Points

### 1. Leave Summary API — Output Cleanup

- **Current issue:** The Leave Summary API is returning **unnecessary data** along with the leave balance information — specifically, some join-related data and an approval-related output that is not needed.
- **Instructions given:**
  - **Remove** the extraneous join data that is returning irrelevant output alongside the balance.
  - The Leave Summary should return **only** the leave balance view for a given `StaffID`:
    - Leave type, allotted count, availed count, balance.
  - **No need** for the staff name or approval-related fields in the Leave Summary output.
  - The `ActualPeriod` field should remain as it is needed for **display purposes** on the UI.
- **Separate concern:** The Leave Application data (showing `FromDate`, `ToDate`, `TotalDays`, etc.) should also be **removed from the Leave Summary response** — it belongs in its own separate API response.

### 2. Duplicate Leave Data — Staff ID 108

- **Issue observed:** When passing `StaffID = 108` to the Leave Summary API, all leave types were returning **duplicate entries** (double records for each leave type).
- **Root cause hypothesis:** The staff was likely **added to a policy bundle, then deleted, and re-added** during testing. This may have resulted in duplicate `HRISLeaveDetails` records without the old data being properly cleaned up.
- **Verification approach:**
  - Testing with other staff IDs returned **correct single records**, confirming the issue is data-specific to StaffID 108.
  - Senior's assumption: The duplicates are test artifacts, not a code bug.
- **Action item:** Ask the relevant team member (Roly) to **delete StaffID 108 from the policy bundle** and re-add them cleanly. If the process is correct and data comes back without duplicates, the issue is resolved.

### 3. `GetApplicableBundleByStaffID` — Controller Relocation

- **Current state:** The `GetApplicableBundleByStaffID` endpoint was located in a separate or incorrect controller.
- **Instruction:** Move it to the **`HRISPolicyBundleMasterController`** so it is organized alongside other policy bundle-related operations.
- **Note:** This is a structural/organizational change — the endpoint logic itself remains the same.

### 4. Leave Application — Additional Fields Required

- **For the Leave Application display (e.g., in MyHRIS or HR panel), the following additional fields are needed:**
  - **`RaisedBy`** — The staff member who submitted the leave request.
  - **`ApprovedBy`** — The person who approved/rejected the leave.
  - **`ApprovedOn` (Decision Date)** — The date on which the approval/rejection decision was made.
- **Source:** These fields should already exist in the `HRISLeaveApplication` table or be derivable from the application workflow.
- **Instruction:** "Get this data first, then we can check further tasks."

### 5. "Enabled by HR" Leave Type — Policy Bundle Visibility Rule

- **Context:** There was a prior discussion about defining special leave types like **Maternity Leave** with an "Enabled by HR" flag.
- **Rule clarified:**
  - Such leave types should be **visible in the Policy Bundle definition** (i.e., when viewing a bundle, Maternity Leave should appear as one of the leave types in that bundle).
  - **However**, when staff are being **assigned** to the policy bundle (i.e., adding individual staff members), the "Enabled by HR" leave types should **NOT be shown/assigned** automatically.
  - These leaves should only **activate** for a specific staff member when HR explicitly processes/enables it for that person through a separate workflow.
  - Once HR activates it, the leave will appear in the staff member's **Applicable Bundle** (profile-level view).
- **Implementation approach:** This should be handled as a **parameter/filter** — when the staff assignment API runs, filter out leaves where the "Enabled by HR" flag is set.

### 6. Year-Wise Data Consideration

- A brief mention that the leave-related data should be **year-wise** (i.e., filtered by `AcademicYearID`), which is already the case in the existing stored procedures.

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Clean up Leave Summary API output — remove unnecessary join data and approval fields; keep only balance view and `ActualPeriod` | Lavesh | High |
| 2 | Remove Leave Application data from Leave Summary response — keep it as a separate endpoint | Lavesh | High |
| 3 | Investigate and fix duplicate records for StaffID 108 — delete from bundle, re-add cleanly, verify | Roly | Medium |
| 4 | Relocate `GetApplicableBundleByStaffID` endpoint to `HRISPolicyBundleMasterController` | Lavesh | Medium |
| 5 | Add `RaisedBy`, `ApprovedBy`, and `ApprovedOn` fields to the Leave Application API response | Lavesh | High |
| 6 | Implement "Enabled by HR" filter — hide such leave types during staff assignment to policy bundle; only show when HR explicitly activates | Lavesh | Medium |
| 7 | Ensure all leave data is filtered by `AcademicYearID` | Lavesh | Low |

---

## Technical References

| Item | Reference |
|------|-----------|
| Leave Summary SP | `SP_GetLeaveSummaryByStaffID` (SchoolERPAdminDB) |
| Applicable Bundle SP | `SP_GetApplicableBundleByStaffID` (SchoolERPAdminDB) |
| Policy Bundle Controller | `HRISPolicyBundleMasterController.cs` |
| Leave Application Table | `HRISLeaveApplication` |
| Leave Details Table | `HRISLeaveDetails` |
| Policy Bundle Table | `HRISPolicyBundle` |
| Applicable Bundle Model | `HRISApplicableBundleResponseModel.cs` |

---

## Key Takeaway
> Priority is to clean up the Leave Summary output and get the Leave Application fields (RaisedBy, ApprovedBy, ApprovedOn) working first. Once the UI is corrected and testable, further testing and feature additions can proceed. The "Enabled by HR" logic is an important architectural decision for how special leave types like Maternity Leave are managed.
