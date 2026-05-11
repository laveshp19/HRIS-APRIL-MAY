# Meeting Notes — 11th May 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Rahul** — Developer (Circle/Rights System, Ticket Module)

---

## Agenda
Code review and cleanup of the **Leave Application screen API**, **Applicable Bundle SP restructuring**, **SP consolidation** (eliminating duplicate stored procedures), and reinforcement of the **two common APIs** architecture for team-based rights.

---

## Discussion Points

### 1. Applicable Bundle SP — Scope Clarification

- **Context:** Lavesh showed the `SP_GetApplicableBundleByStaffID` as the primary data source for the Leave Application screen.
- **Senior's observation:** The Applicable Bundle SP is returning **too much data** for the Leave Application context:
  - It currently returns: Policy Bundle details, Shift timings, Leave Policy details, LWP Deduction Policy rules, Approver info, Designation, Department, etc.
  - For the **Leave Application screen**, the only data needed is: **Leave Balance** (allotted, availed, remaining per leave type) and the staff's leave application history.
  - Shift timings, LWP rules, and policy configuration details are **NOT needed** on the Leave Application screen.

- **Where all that data IS needed:** The **HRIS Master** screen (HR panel → select a staff member → view their full profile). That screen needs the complete bundle: policy details, shift, leave policies, LWP, and approver info.

- **Decision:**
  - The Applicable Bundle SP should be used **only** in the HRIS Master screen.
  - The Leave Application screen should use the **`GetLeaveApplicationData`** endpoint (which already exists in Swagger) — and that SP should return **only** leave-related data.

### 2. Unnecessary Joins in Applicable Bundle — Code Review

The senior reviewed the `SP_GetApplicableBundleByStaffID` and flagged two incorrect joins:

#### ❌ `TicketDepartmentMaster` Join — Remove
- **Issue:** The SP was joining to `TicketDepartmentMaster` to get the department name.
- **Why it's wrong:** `TicketDepartmentMaster` is for the **Ticket routing system** — it defines which department a ticket routes to and who the department owner is. It has **nothing to do** with a staff member's organizational department.
- **Senior's explanation of Ticket Department:**
  - The ticket system has departments (e.g., "IT Software"). Each department has an **owner**.
  - When a ticket is raised and **no specific assignee** is defined for that issue type, it routes to the **department owner** by default.
  - The department owner then **reassigns** the ticket to a specific team member — that's when the team dropdown and staff list are needed.
  - This is entirely separate from the Organization Department (Group → Department → Team hierarchy).
- **Action:** Remove the `TicketDepartmentMaster` join from the Applicable Bundle SP. Use `OrganizationDepartment` or the staff's own `DepartmentID` if needed.

#### ✅ `StaffDesignationMaster` Join — Keep (conditionally)
- **Used for:** Showing the Approver's designation in the UI (e.g., "Arti Soni — Team Lead").
- **Senior's note:** This is acceptable for the HRIS Master view where the full profile is displayed. However, it's **not needed** in the Leave Application context — the approver should see their team members by name only, not designation. *"Meri team ka designation toh mujhe pata hi hoga."*

### 3. Leave Application Screen — What Data Is Needed

The senior clarified the exact data requirements for the Leave Application screen:

| Data | Source | Needed? |
|------|--------|---------|
| Leave Balance (allotted, availed, remaining) | `SP_GetLeaveSummaryByStaffID` or inline in Leave App SP | ✅ Yes |
| Leave Application list (dates, status, type) | `HRISLeaveApplication` table | ✅ Yes |
| RaisedBy, ApprovedBy, ApprovedOn | `HRISLeaveApplication` table | ✅ Yes |
| Shift timings | Policy Bundle | ❌ No |
| Leave Policy details (master config) | `HRISLeavePolicyDetails` | ❌ No |
| LWP Deduction Policy rules | `HRISLWPDeductionPolicyDetailsStaffSupportStaff` | ❌ No |
| Approver name/photo/designation | Approver joins | ❌ No (not on this screen) |

- **Decision:** The `GetLeaveApplicationData` endpoint should be the **sole API** for the Leave Application screen. It already accepts `StaffID` and `AcademicYearID`. Merge the leave balance data into this endpoint so one API call fetches everything the screen needs.

### 4. SP Rename / Restructuring Plan

| Current SP | Action | New Purpose |
|------------|--------|-------------|
| `SP_GetApplicableBundleByStaffID` | **Rename** — may need to be renamed or scoped clearly | Used ONLY for HRIS Master screen (full staff profile view) |
| `GetLeaveApplicationData` (Swagger) | **Enhance** — add leave balance data, remove policy/shift/LWP data | Used for Leave Application screen |
| Old leave balance SP | **Eliminate** — consolidate into `GetLeaveApplicationData` | No longer needed as a separate call |

- **Senior's instruction:** *"Ye poori tumhari eliminate hoke isi mein ye GetLeaveApplicationData mein aani chahiye."* — All leave display data should come from this single endpoint.

### 5. Duplicate SPs — Consolidation Order

The senior and Rahul identified **two duplicate SPs** for fetching staff members by team:

| SP | Author | Description | Status |
|----|--------|-------------|--------|
| `SP_GetEmployeesByTeamIDs` | Denis (original), Modified by Lavesh | Original SP used across the system. Returns staff + support staff by CSV TeamIDs. Recently enhanced with `JobStatus` filter and designation join. | ✅ **Keep — modify this one** |
| `SP_GetStaffByTeamID` | Lavesh (new, 08-May-2026) | Newly created during the team-based rights transition. Similar logic but staff-only (no support staff), slightly different structure. | ❌ **Eliminate** |

- **Senior's decision:**
  - **Keep `SP_GetEmployeesByTeamIDs`** — it's the one already in use across the system (tickets, observation, goals, performance, etc.).
  - **Merge** the `JobStatus` filter and any additional logic from `SP_GetStaffByTeamID` into `SP_GetEmployeesByTeamIDs`.
  - **Eliminate `SP_GetStaffByTeamID`** — delete it entirely.
  - *"Ek hi logic ko do-chaar API mein kyun la rahe hain?"* — Don't duplicate the same logic across multiple SPs.
  - Update the endpoint name, DAL references, and model accordingly.

### 6. Two Common APIs — Architecture Reinforcement

The senior reiterated the core architectural decision from the May 8th meeting:

> **Only two common APIs are needed for team-based staff visibility:**

| # | API | Input | Output | Usage |
|---|-----|-------|--------|-------|
| **1** | Get My Teams | `StaffID` | List of teams where the staff has rights (via `StaffRights` + Group/Dept/Team resolution) | Populates team dropdown everywhere |
| **2** | Get Staff By TeamID(s) | `TeamID(s)` (CSV) | List of active staff members in those teams | Populates staff list after team selection |

These two APIs should be **reused everywhere**:
- Ticket reassignment
- Leave Application approval
- Regularization approval
- Staff Observation
- Staff Goals
- Performance Reports
- Any future module with team-based staff selection

### 7. Staff Permission Table — Identification

During the meeting, there was some confusion about which table stores the staff-to-group/department/team rights mapping. The team searched through multiple tables:

| Table Searched | Result |
|----------------|--------|
| `StaffPermission` | ❌ Not found / blank data |
| `StaffRights` | ❌ Not immediately visible |
| `StaffRightsMaster` | ❌ Doesn't exist |
| `RoleBasedRightsMaster` | ❌ Module-level rights, not team rights |
| `RoleModuleAccess` | ❌ Module access, not team |
| **`StaffOwnershipDetail`** | ✅ **Correct** — defines which staff is the owner of which Group/Department/Team |

- **Rahul confirmed:** The `StaffOwnershipDetail` table (and related SPs like `SP_GetStaffOwnershipDetailsByTeamIDs`, `SP_GetStaffOwnershipDetailsByDepartmentIDs`, `SP_GetStaffOwnershipDetailsByGroupIDs`) is the source of truth for which staff has rights on which organizational entities.
- **Note:** The `SP_GetOwnedTeamsByStaffID` already uses the `StaffRights` table which has `EntityType` (TEAM/DEPARTMENT/GROUP) and `EntityID`. The discussion confirmed this is the correct approach.

### 8. Where Condition — Dynamic Based on Rights Level

The senior drew out the join logic on a notepad:

```
StaffPermission → contains: StaffID, EntityType, EntityID
Example: StaffID = 2, rights on TeamID = 5

Query Pattern:
  OrganizationGroup (OG) 
    JOIN OrganizationDepartment (OD) ON OG.GroupID = OD.GroupID
    JOIN OrganizationTeam (OT) ON OD.DepartmentID = OT.DepartmentID

WHERE condition changes based on rights level:
  - If rights on TEAM    → WHERE OT.OrganizationTeamID = 5
  - If rights on GROUP   → WHERE OG.OrganizationGroupID = 3
  - If rights on DEPT    → WHERE OD.OrganizationDepartmentID = 2
```

- **Result:** Regardless of whether rights are at Group, Department, or Team level, the query always resolves to a **list of Team IDs**. Then staff members are fetched by those team IDs.

### 9. Policy Bundle Add Staff — Urgent Fix

- **Senior identified this as a blocker:** Roly (frontend) is stuck because the Policy Bundle "Add Staff" functionality isn't working correctly.
- **Action items (urgent):**
  1. Fix/finalize the SP that handles adding staff to a policy bundle.
  2. Push the corrected code immediately.
  3. Senior will **publish the changes right away** so Roly can continue her work.
- **Senior's exact words:** *"Jisko correct karna hai woh karke do. Main turant tumhe publish kar dungi. Roly atki hui hai wahan pe. Uska kaam nahi ho raha hai."*

### 10. Endpoint Cleanup Required

Based on the SP consolidation, the following endpoint changes are needed:

| Layer | Change |
|-------|--------|
| **SP** | Eliminate `SP_GetStaffByTeamID`. Keep and enhance `SP_GetEmployeesByTeamIDs`. |
| **DAL** | Update references from the eliminated SP to the kept one. |
| **Controller/Endpoint** | Rename the endpoint if necessary to match the consolidated SP name. |
| **Model** | Ensure the response model matches the consolidated SP output. |
| **SP** | Remove `TicketDepartmentMaster` join from `SP_GetApplicableBundleByStaffID`. |
| **SP** | Scope `SP_GetApplicableBundleByStaffID` to HRIS Master only. |
| **SP** | Enhance `GetLeaveApplicationData` to include leave balance. |

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Remove `TicketDepartmentMaster` join from `SP_GetApplicableBundleByStaffID` | Lavesh | High |
| 2 | Remove Leave Policy, LWP, and Shift data from the Leave Application screen API — keep only leave balance + application list | Lavesh | High |
| 3 | Enhance `GetLeaveApplicationData` SP to include leave balance data (merge from old balance SP) | Lavesh | High |
| 4 | Eliminate `SP_GetStaffByTeamID` — merge `JobStatus` filter into `SP_GetEmployeesByTeamIDs` | Lavesh | High |
| 5 | Fix Policy Bundle "Add Staff" SP — push immediately so Roly can unblock | Lavesh | **Urgent** |
| 6 | Senior to publish the corrected SP to live environment immediately after Lavesh delivers | Senior | Urgent |
| 7 | Rename/scope `SP_GetApplicableBundleByStaffID` for HRIS Master screen only | Lavesh | Medium |
| 8 | Update DAL and endpoint references for the eliminated SP | Lavesh | Medium |
| 9 | Ensure only 2 common APIs are used across all modules for team-based staff visibility | Lavesh | Ongoing |

---

## Technical References

| Item | Reference |
|------|-----------|
| Applicable Bundle SP | `SP_GetApplicableBundleByStaffID` (SchoolERPDB) |
| Employees by Team SP (KEEP) | `SP_GetEmployeesByTeamIDs` (SchoolERPDB) — original, widely used |
| Staff by Team SP (ELIMINATE) | `SP_GetStaffByTeamID` (SchoolERPDB) — newly created, to be removed |
| Owned Teams SP | `SP_GetOwnedTeamsByStaffID` (SchoolERPDB) |
| Leave Summary SP | `SP_GetLeaveSummaryByStaffID` (SchoolERPAdminDB) |
| Staff Rights Table | `StaffRights` — EntityType (TEAM/DEPARTMENT/GROUP) + EntityID |
| Staff Ownership Table | `StaffOwnershipDetail` — defines Group/Dept/Team ownership |
| Organization Hierarchy | `OrganizationGroup` → `OrganizationDepartment` → `OrganizationTeam` |
| Ticket Department Table | `TicketDepartmentMaster` — ONLY for ticket routing, not org structure |

---

## SP Consolidation Summary

```
BEFORE (Fragmented):
  ├── SP_GetApplicableBundleByStaffID  → Returns EVERYTHING (overkill for most screens)
  ├── SP_GetStaffByTeamID              → New, duplicate of existing
  ├── SP_GetEmployeesByTeamIDs         → Original, widely used
  ├── SP_GetLeaveSummaryByStaffID      → Separate leave balance
  └── GetLeaveApplicationData          → Exists but underutilized

AFTER (Consolidated):
  ├── SP_GetApplicableBundleByStaffID  → HRIS Master screen ONLY (full profile)
  │     └── Remove: TicketDepartmentMaster join
  ├── SP_GetEmployeesByTeamIDs         → SINGLE SP for fetching staff by teams
  │     └── Merge: JobStatus filter, designation join from eliminated SP
  ├── SP_GetOwnedTeamsByStaffID        → SINGLE SP for "which teams do I have rights on"
  └── GetLeaveApplicationData          → Leave Application screen (balance + applications)
        └── Merge: Leave balance data from old SP
  
  ❌ ELIMINATED: SP_GetStaffByTeamID
  ❌ ELIMINATED: Old separate leave balance SP
```

---

## Key Takeaway
> The main theme is **consolidation and clarity** — stop creating duplicate SPs for similar functionality. Two common APIs (Get My Teams + Get Staff By TeamIDs) should be reused everywhere. The Leave Application screen should ONLY get leave data, not the full policy bundle. The TicketDepartmentMaster has nothing to do with organizational structure — don't confuse ticket routing departments with staff departments. Fix the Policy Bundle Add Staff SP urgently to unblock Roly.
