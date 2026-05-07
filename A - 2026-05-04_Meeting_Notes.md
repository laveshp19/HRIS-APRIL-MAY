# Meeting Notes — 4th May 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Pradeep** — Team Member (Infrastructure / Biometric)

---

## Agenda
Deep-dive knowledge transfer on the **Biometric Attendance System architecture**, the `HRIS_MasterAttendance` table structure, the **`Stode_HRIS_MasterTable` SQL trigger**, credit/deduction logic, regularization flow, leave-attendance integration, and upcoming tasks for the new system (OEDO/FSM database).

---

## Discussion Points

### 1. Biometric Machine Integration & Data Pipeline

- **Context:** The HR cell has a biometric machine connected to an **FSM (Fountainhead School Management)** database — specifically the `INVENTORY.MDF` database (`DeviceLogs` table).
- **Data flow explained:**
  1. Staff members punch in/out on the biometric machine.
  2. The punch data lands in the **`DeviceLogs`** table (in the biometric/GBay server database).
  3. A **trigger** (`Stode_HRIS_MasterTable`) is set on `DeviceLogs` that fires on `AFTER INSERT`.
  4. The trigger creates a **mirror table** in the school's INVENTORY database, then processes the punch data into the `HRIS_MasterAttandenceTable`.
- **Important note:** The senior has been the one who originally built and maintains this trigger. This is a critical piece of infrastructure that the developer (Lavesh) now owns going forward.

### 2. `HRIS_MasterAttandenceTable` — Structure Deep-Dive

The senior walked through the table structure field by field:

| Field Group | Fields | Purpose |
|------------|--------|---------|
| **Date/Day** | `Record_Date`, `Record_Day` | Date and day name of the punch record |
| **Staff Identity** | `StaffID`, `EMP_HRIS`, `ISSupportStaff` | Staff identifier, HRIS employee ID, and flag for support staff vs. teaching staff |
| **Punch Tracking** | `IN0`–`IN9`, `OUT0`–`OUT9` | Up to 10 IN punches and 10 OUT punches per day |
| **Last Counters** | `LastINcnt`, `lastOUTcnt` | Tracks which is the latest IN/OUT column populated |
| **Final Punch** | `FInalPunchIn`, `FinalPunchOut` | The effective first-in and last-out times (may differ from machine punch after regularization) |
| **Machine Punch** | `MachineFinalPunchIn`, `MachineFinalPunchOut` | Original raw machine punch times (never overwritten) |
| **Shift Info** | `StartShift`, `EndShift`, `INRealxation`, `OutRelaxation` | Shift timings and relaxation minutes, captured at first punch from Policy Bundle or Temporary Shift |
| **Late/Early Flags** | `LateIN`, `EarlyOut` | Boolean flags indicating if staff was late in or left early |
| **Vehicle & Gate** | `IsOwnVEhicle`, `isGatekeeperaEntry` | Flags for own-vehicle staff and gatekeeper entry |
| **Credit** | `creditDur` | Accumulated credit deduction in minutes for the day |
| **Regularization** | `RegularizationIN`, `RegularizationOUT`, `RegularizationID`, `IsRegularize` | Regularization times, reference ID, and status flag |
| **Leave** | `LeaveID` | Reference to leave record if applicable (though leave is managed separately) |
| **Work Time** | `TotalWorkTime` | Cumulative working minutes for the day |
| **Device** | `DeviceId` | Which biometric device recorded the punch |

### 3. Punch Processing Logic (Trigger Walkthrough)

The senior explained the complete trigger logic (`Stode_HRIS_MasterTable`):

#### First Punch (IN0):
1. On first punch of the day (`@cnt = 0`):
   - Fetch `StaffID` and `EMP_HRIS` from the inserted record.
   - Check if a **Temporary Shift** is defined for this staff on this date.
   - If yes → use temporary shift timings (`StartShift`, `EndShift`, `InTimeRelaxationInMins`, `OutTimeRelaxationInMins`).
   - If no → use the **Policy Bundle** timings from `VW_HRISPOLICYBUNDLEWISEDETAILS`.
   - Check own-vehicle list and gatekeeper entry status.
   - Calculate **credit deduction** if the punch time exceeds `StartShift + InRelaxation`.
   - Insert a new row into `HRIS_MasterAttandenceTable` with all captured data.
   - If credit deduction applies, also insert into `HRIS_LeaveDeductionRecords`.

#### Subsequent Punches (IN1–IN9, OUT0–OUT9):
- Each subsequent punch updates the corresponding `IN`/`OUT` column.
- `LastINcnt` and `lastOUTcnt` are incremented to track position.
- `FinalPunchOut` and `MachineFinalPunchOut` are updated on each OUT punch.
- For each mid-day IN punch (return from outside), the **credit deduction** (difference between OUT and next IN) is calculated and added to both `creditDur` and `HRIS_LeaveDeductionRecords`.
- A **2-minute timebound** check prevents duplicate punch processing.

### 4. Credit Deduction Rules

| Scenario | Late In Credit Cut? | Explanation |
|----------|---------------------|-------------|
| Late + Own Vehicle + Gatekeeper Entry | **Yes** | Own vehicle staff arriving late with gatekeeper confirmation → credit cut |
| Late + Own Vehicle + No Gatekeeper Entry | **No** | Without gatekeeper confirmation, no deduction |
| Late + Bus (not own vehicle) | **No** | Bus staff aren't penalized (bus delays are regularized) |
| Late + Not own vehicle + Gatekeeper Entry | **No** | If not in own vehicle list, no credit cut regardless |
| Multiple punches (mid-day out/in) | **Yes** | The time difference between OUT and next IN is recorded as credit deduction |

- **Relaxation:** 3 minutes of relaxation (configurable per policy bundle). E.g., shift starts at 7:35, relaxation = 3 min → effective deadline is 7:38.
- **Cutoff threshold:** If late by more than 45 minutes (configurable `CreditCutoff`), it transitions from credit deduction to **leave quota deduction**.
- **170-minute rule:** If total credit deductions for a day exceed 170 minutes, the system switches to full-day **leave quota** deduction instead of minute-by-minute credit.
- **Day-end scheduler:** A separate scheduler runs at end of day to calculate final out-time credit deductions.

### 5. Regularization Flow & Impact

- **What is regularization?** When a staff member's recorded punch doesn't match their actual/intended work time, they submit a regularization request.
- **Common example:** Saturday has a temporary shift (7:45–3:45) but it wasn't defined in the system. Staff punches out at 3:05 (early based on default shift 4:20), so credit gets cut. They regularize to 7:15–3:15.
- **How it works:**
  1. Staff submits regularization with corrected IN/OUT times.
  2. Once approved, the `FinalPunchIn` and `FinalPunchOut` columns are updated.
  3. **`MachineFinalPunchIn`/`MachineFinalPunchOut` remain unchanged** — they always hold the raw machine data for audit trail.
  4. `RegularizationIN`, `RegularizationOUT`, `RegularizationID`, and `IsRegularize` fields are updated.
  5. Impact is on the **final punch columns only** (2 columns), not the machine punch columns.

### 6. Leave Application & Attendance Integration

- **Leave and attendance are separate:** When someone applies for leave, it doesn't directly populate the `HRIS_MasterAttandenceTable`. Leave records are stored in a separate leave deduction system.
- **Attendance Report integration:** When generating the attendance report:
  - If no punch is found for a **working day**, the system checks `HRIS_LeaveDeductionRecords` for a full-day leave.
  - If found, it displays **"L" (Leave)** along with the leave type (e.g., "PL" for Privilege Leave, "CL" for Casual Leave) using the `LeaveMasterID`.
  - Holidays are also detected and shown in the report.
- **Leave application block rule:** If a staff member has **any punch** recorded for a given day, they **cannot apply for leave** on that day. The system will reject the application with a message like "Punch found for this day, you cannot apply for leave."
  - **Edge case:** If someone punched in but left within 15–20 minutes, the backend team must **manually remove the punch**, then the staff can apply for full-day leave.

### 7. Upcoming Work — New System Trigger

- **The OEDO/FSM database** has device logs that need to be processed into the new SchoolERP system.
- **Senior will set up the basic trigger** for processing first, second, third, and fourth punches — analogous to the existing `Stode_HRIS_MasterTable` trigger but for the new system.
- **Lavesh's responsibility going forward:**
  - All additional changes and stages of the trigger will be handled by Lavesh.
  - Leave approval/rejection workflow remains Lavesh's existing assignment.
  - Senior will hand over the base trigger, then Lavesh takes ownership.

### 8. Policy Bundle — Prerequisites for Trigger

- **Critical dependency:** The trigger relies on the policy bundle being correctly set up for each staff member. Specifically:
  - `StartShift` and `EndShift` times must be accurate.
  - `InTimeRelaxationInMins` and `OutTimeRelaxationInMins` must be defined.
  - The view `VW_HRISPOLICYBUNDLEWISEDETAILS` must return correct data.
- **Instruction to Lavesh:** Verify from the **UI** that when a staff is added to a policy bundle, all shift and relaxation values are correctly stored and retrievable.
- **Current limitation:** Temporary Shift logic is not yet built in the new system. For now, only regular Policy Bundle timings will be used.
- **Approach:** Build in **chunks** — first get regular policy bundle working, then add temporary shift support later.

### 9. SQL View for Shift Data

- **Senior showed a SQL view** (`VW_HRISPOLICYBUNDLEWISEDETAILS`) used in the existing system that takes `EmployeeID` and `AcademicYear` as parameters and returns the complete policy bundle details including shift timings.
- **Instruction:** Lavesh should either **reuse an existing SP/view** that provides the same output, or create a new one. The goal is to avoid requiring multiple table joins every time shift data is needed.

### 10. HR Rights & Team Visibility

- **HR panel visibility rule:**
  - HR panel should only be visible to **HR team members**, **Admin**, and **Super Admin**.
  - Regular TLs (Team Leads) or staff should **not** see the HR panel.
  - In the existing Nucleus system, Lavesh was given **administrator rights** for development purposes — this is why he can see the HR panel.
- **HR team visibility:**
  - When an HR member logs in, they should see **all teams** (i.e., `GetAllTeams` without any filter parameter).
  - A non-HR member should not see the HR panel at all.
  - The `Role and Manager` section in the Nucleus panel has the HR team definition — this will be referenced for rights management.
- **Action item:** Lavesh to document the HR-rights management requirements from both the **UI side** and **API side**. Discussion to continue the next day.

### 11. Git Merge Conflict — Induction Pages

- **Context:** There was a merge conflict related to some pages (induction-related changes).
- **Issue:** The senior had pushed changes, but Lavesh had not merged them. This resulted in conflicts when trying to sync.
- **Resolution approach:**
  - The senior had a backup of the changes.
  - Decided to **ignore/untrack** the conflicting files for now.
  - Focus was shifted to the induction work — only the induction-related changes were needed immediately.
  - The merge conflict resolution was deferred to be handled properly later.
- **Related issue:** Some resignation-related changes were also pending — 2–3 resignation records had been modified by someone. This was noted but deferred.

### 12. Roly's Work Review — Scheduled for Next Day

- **Senior mentioned** that Roly should be part of the next day's morning meeting.
- **Purpose:** Review how much work Roly has completed, assess coordination level, and ensure both developers are on the same page.

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Review the `Stode_HRIS_MasterTable` trigger thoroughly — understand every punch scenario | Lavesh | High |
| 2 | Verify Policy Bundle data accuracy from UI — ensure shift timings, relaxation values are correctly stored | Lavesh | High |
| 3 | Create/reuse a SQL View that returns staff shift details (StartShift, EndShift, Relaxation) by EmployeeID and AcademicYear | Lavesh | High |
| 4 | Document HR-rights management requirements (UI + API) for discussion | Lavesh | Medium |
| 5 | Senior to set up the basic new-system trigger (first 4 punches) on the OEDO/FSM database | Senior | High |
| 6 | Continue Leave Application approval/rejection workflow | Lavesh | Ongoing |
| 7 | Resolve Git merge conflicts from induction pages properly | Lavesh / Senior | Low |
| 8 | Morning meeting with Roly to review work progress | All | Next Day |

---

## Technical References

| Item | Reference |
|------|-----------|
| Attendance Trigger | `Stode_HRIS_MasterTable` (Trigger on `DeviceLogs` table in `INVENTORY.MDF`) |
| Master Attendance Table | `HRIS_MasterAttandenceTable` (INVENTORY.MDF) |
| Leave Deduction Records | `HRIS_LeaveDeductionRecords` |
| Policy Bundle View | `VW_HRISPOLICYBUNDLEWISEDETAILS` |
| Policy Bundle View (Support) | `VW_HRISPOLICYBUNDLEWISEDETAILS_SupportStaff` |
| Leave Balance Summary | `VW_GET_LEAVEBALANCE_SUMMERY` (Function) |
| Temp Shift Table | `HRISTempShiftPolicy` |
| Gatekeeper Entry Table | `HRIS-GateKeeperEntry` |
| Own Vehicle List | `StaffOwnVehicle` |
| Device Logs | `DeviceLogs` (INVENTORY.MDF) |
| Shift Master | `hrisShiftMaster` |
| Policy Bundle Controller | `HRISPolicyBundleMasterController.cs` |
| Applicable Bundle SP | `SP_GetApplicableBundleByStaffID` |

---

## Key Architectural Insights

### Machine Punch vs. Final Punch (Dual-Column Design)
The system maintains **two sets of IN/OUT columns** for audit purposes:
- **`MachineFinalPunchIn` / `MachineFinalPunchOut`:** Raw biometric data — never overwritten.
- **`FinalPunchIn` / `FinalPunchOut`:** Effective times — updated when regularization is approved.

This design allows the system to always trace back to the original machine data for any dispute resolution.

### Credit Deduction Hierarchy
```
Late arrival → Within cutoff (45 min) → Credit deduction (in minutes)
Late arrival → Beyond cutoff → Full credit size deduction
Total day deductions > 170 min → Switch to Leave Quota deduction
```

### Temporary Shift Override
```
First punch → Check Temporary Shift for date
  ├─ If found → Override StartShift, EndShift, Relaxation values
  └─ If not found → Use Policy Bundle values
Second+ punches → No re-check (already captured on first punch)
```

---

## Key Takeaway
> This was a comprehensive knowledge transfer session. The biometric attendance trigger is the backbone of the HRIS attendance system, and Lavesh now has ownership of all future changes. The immediate priorities are: (1) verify policy bundle data accuracy from UI, (2) create the shift details view, and (3) prepare for the new-system trigger setup.
