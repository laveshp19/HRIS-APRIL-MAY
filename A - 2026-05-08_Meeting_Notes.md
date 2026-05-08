# Meeting Notes — 8th May 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)

---

## Agenda
Discussion on the **Rights & Team Visibility Architecture** — specifically how team-based rights (Group/Department/Team) and `ReportingTo`-based rights should interact when determining which staff members and teams a logged-in user can see across modules (Leave Application, Tickets, Performance, Observation, Policy Bundle, etc.).

---

## Discussion Points

### 1. Current Problem — Incorrect Rights Definition

- **Context:** Lavesh had written two SQL queries to populate staff lists:
  1. **Query 1 (Team-based):** Fetch staff members who belong to a team (e.g., TeamID = 3, "Admin Alam") on which the logged-in user has rights.
  2. **Query 2 (ReportingTo-based):** Fetch staff members who report to a specific person (e.g., Arti Soni) regardless of which team they belong to.
- **Problem identified:** The data definition was **incorrect** — staff members were reporting to Arti Soni, but the rights had been given on a **different team** than the one those staff actually belonged to.
  - Example: Arti Soni has rights on the "Admin Alam" team (TeamID 3), but some staff who report to her are in a **different** team. This means:
    - The team-based query returns only the 4 staff in Admin Alam team.
    - The ReportingTo query returns staff who report to Arti Soni but are NOT in the Admin Alam team.
    - This mismatch creates confusion — should we use team-based or ReportingTo-based logic?

### 2. How Rights Should Ideally Work — Senior's Clarification

The senior explained using the existing **Nucleus system** as reference:

#### In Nucleus (Existing Production System):
- The **Circle** system is used for rights management.
- When a staff member (e.g., the senior) is the **owner** of a sub-circle (e.g., "IT Software Team"), they can see all members of that team.
- The staff who report to the senior are the **same** staff who are in the team where she has ownership/rights.
- **There is no mismatch** — rights and reporting are aligned to the same team.

#### The Problem in the New EduSystem:
- Rights have been assigned on one team, but reporting has been defined to a different team.
- This creates a **dual source of truth** — team rights vs. ReportingTo.

### 3. Architectural Decision — Use Team-Based Rights ONLY

After detailed discussion, the senior made a clear architectural decision:

> **"We should go with the Team-based query only. Don't use ReportingTo for determining visibility."**

**Rationale:**
- If a staff member reports to someone but the **team rights aren't assigned**, that's a **rights definition error** — it means the setup is wrong, not the code.
- The system should show teams based on **which teams the logged-in user has rights on** (via `StaffRights` table with `EntityType` = TEAM/DEPARTMENT/GROUP).
- If someone reports to a user but doesn't appear in their visible teams, the fix is to **correct the rights assignment**, not to add complex dual-logic.
- This eliminates confusion: "If I see the wrong team, it means the rights system was set up wrong."

**How it works with the existing `SP_GetOwnedTeamsByStaffID`:**
```sql
-- Returns all teams where the staff has rights (through Team, Department, or Group)
SELECT OrganizationTeamID, TeamName FROM OrganizationTeam
WHERE 
  OrganizationTeamID IN (SELECT EntityID FROM StaffRights WHERE StaffID=@StaffID AND EntityType='TEAM')
  OR OrganizationDepartmentID IN (SELECT EntityID FROM StaffRights WHERE StaffID=@StaffID AND EntityType='DEPARTMENT')
  OR OrganizationGroupID IN (SELECT EntityID FROM StaffRights WHERE StaffID=@StaffID AND EntityType='GROUP')
```

### 4. Department Head Scenario — Nirav Sir Example

- **Nirav Sir** is the head of the **Operations Department**.
- The Operations Department contains ~6-7 teams: IT Software, IT Hardware, Front Desk, Day Care, Infra, etc.
- Nirav Sir has rights at the **Department level** (not individual teams).
- **Expected behavior:** When Nirav Sir logs in:
  - The team dropdown should show **all teams** under Operations: IT Software, IT Hardware, Front Desk, Infra, Day Care.
  - Selecting any team shows **all members** of that team.
  - Even though Nirav Sir's direct reports may only be the TLs, he should see **all staff** in teams under his department rights.
- **Implementation:** The query already handles this — `SP_GetOwnedTeamsByStaffID` resolves Department-level rights into the constituent Team-level entries.

### 5. Multiple Teams — CSV/IN Clause Handling

- **Question from Lavesh:** If a user has rights on 2+ teams, how to pass multiple TeamIDs to the staff-fetching query?
- **Answer:** Use **CSV (comma-separated values)** in the parameter and split using `IN` clause, or pass multiple TeamIDs. The dropdown should show all applicable teams, and selecting one fetches its members.

### 6. Common Functionality Across Modules

The senior emphasized that the **team-based staff visibility** should be a **common/shared functionality** used consistently across all modules:

| Module | What Shows | Notes |
|--------|-----------|-------|
| **Leave Application** | Teams in dropdown → Staff in selected team | For approving/viewing leave requests |
| **Ticket System** | Teams in dropdown → Staff in selected team | For ticket assignment/reassignment |
| **Performance Report** | Teams → Staff members for review | For filling performance evaluations |
| **Staff Observation** | Teams → Staff members to observe | For observation entries |
| **Policy Bundle (HR only)** | **All teams** (no rights filter) | HR should see all teams regardless of personal rights |

- **Key principle:** The same API/query should be reusable. Don't build page-specific logic for each module.

### 7. HR Exception — Policy Bundle Add Staff

- **Special rule for HR:** When adding staff to a Policy Bundle, the HR user should see **ALL teams** — rights-based filtering should **not** apply.
- **Reason:** HR needs to manage every staff member's policy bundle, regardless of their personal team rights.
- **This is the only exception** to the team-rights-based visibility rule.

### 8. System Accounts — Defer for Now

- **Observation:** Some entries in the staff list showed "FS Guest User" and other system accounts.
- **Senior's clarification on System Accounts:**
  - System accounts are **not actual staff** — they are helper/virtual accounts for support roles.
  - Example: **Kirit** handles the stationery store. He logs in via a system account. Tickets assigned to the "Stationery Store" account go to whoever is manning that role (currently Kirit, could be someone else tomorrow).
  - System accounts are relevant in the **Ticket system** (tickets get assigned to them) but NOT in Performance reports, observations, etc.
  - Kirit's individual performance is tracked under **Support Staff**, not under the system account.
- **Decision:** **Defer system account logic** — don't implement it in the new EduSystem right now. Leave it aside for later. It will eventually be managed via a simple **flag** (`IsSystemAccount`).
- **Note:** Testing accounts and system accounts do **not exist** in the support staff table — only in the main staff table.

### 9. Staff vs. Support Staff Filtering

- **Already handled via flag:** The existing APIs use a flag to distinguish between Staff and Support Staff data.
- **No page-specific logic needed** — use the flag-based approach consistently:
  - `PolicyFor = 1` → Staff
  - `PolicyFor = 2` → Support Staff
- Support staff do not have testing accounts or system accounts, so those filters are irrelevant for them.

### 10. Job Status — Inactive Staff Filtering

- **`JobStatus` field** is used to filter out inactive/resigned staff.
- **Rule:** Even if an inactive/resigned staff member is still technically in a team's data, they should **NOT appear** in any active staff list.
- **Implementation:** Already handled in `SP_GetMyTeamByStaffID` with the filter: `ISNULL(S.JobStatus, '') != 'Resigned'`.

### 11. Ticket System — Cross-Check Required

- **Observation:** When testing the Ticket reassignment feature on FS Pro (logged in as Arti Soni), **all teams** were showing up in the reassign dropdown — not just the ones Arti has rights on.
- **Possible reason:** Arti Soni may have been given rights on all teams in FS Pro, or the ticket system is using a different API that doesn't filter by rights.
- **Action item:** Lavesh to check:
  1. Which API is being used for the ticket reassignment team list.
  2. Whether it respects the `StaffRights` table.
  3. Cross-verify with **Rahul** (another developer) on the ticket module implementation.
- **Decision:** Schedule a joint session with Lavesh, Senior, and Rahul to review and align the ticket system's rights implementation.

### 12. Revamp Suggestion — Test in FSM Environment

- Since the current data in FS Pro has too many staff and complex rights definitions, the senior suggested:
  - Test in the **FSM environment** where there are fewer staff members.
  - Or **revamp** the rights definition data to be correct (align reporting and team assignments).
- Lavesh agreed to revamp the test data to validate the team-based approach properly.

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Adopt team-based rights query (`SP_GetOwnedTeamsByStaffID`) as the single source for staff visibility across all modules | Lavesh | High |
| 2 | Remove/deprecate ReportingTo-based logic for determining visible staff lists | Lavesh | High |
| 3 | Implement common API: Given a StaffID, return all teams they have rights on → given a TeamID, return all active members | Lavesh | High |
| 4 | Add HR exception: Policy Bundle staff-add screen should show ALL teams regardless of rights | Lavesh | Medium |
| 5 | Filter out inactive/resigned staff from all team member lists (`JobStatus` check) | Lavesh | High |
| 6 | Defer system account logic — do not include in current implementation | Lavesh | Deferred |
| 7 | Check Ticket reassignment module — verify which API is used and whether it respects `StaffRights` | Lavesh | Medium |
| 8 | Schedule joint session with Rahul to align ticket module rights implementation | Lavesh + Senior | Medium |
| 9 | Revamp test data / test in FSM environment to validate team-based approach | Lavesh | Medium |
| 10 | Ensure CSV/multiple TeamID support for staff who have rights on multiple teams | Lavesh | Medium |

---

## Technical References

| Item | Reference |
|------|-----------|
| Team Rights SP | `SP_GetOwnedTeamsByStaffID` (SchoolERPDB) |
| My Team SP | `SP_GetMyTeamByStaffID` (SchoolERPDB) — uses `ReportingToID` (to be deprecated for rights purposes) |
| Staff Rights Table | `StaffRights` — columns: `StaffID`, `EntityID`, `EntityType` (TEAM / DEPARTMENT / GROUP) |
| Organization Team Table | `OrganizationTeam` — links to `OrganizationDepartmentID` and `OrganizationGroupID` |
| Applicable Bundle SP | `SP_GetApplicableBundleByStaffID` — includes `TeamName` via `OrganizationTeam` join |
| Policy Bundle Staff SP | `SP_GetPolicyBundleStaff` — uses `PolicyFor` flag for Staff vs. Support Staff |
| Nucleus Circle System | `VirtualCircleOwners` (INVENTORY.MDF) — legacy rights model via circle ownership |

---

## Rights Architecture Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Staff Visibility Architecture                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Step 1: Determine visible TEAMS for logged-in user                │
│  ┌──────────────────────────────────────────────────┐               │
│  │  StaffRights table (EntityType + EntityID)       │               │
│  │  ├─ EntityType = 'GROUP'     → All Departments   │               │
│  │  │                             → All Teams        │               │
│  │  ├─ EntityType = 'DEPARTMENT'→ All Teams in Dept  │               │
│  │  └─ EntityType = 'TEAM'     → Specific Team only │               │
│  └──────────────────────────────────────────────────┘               │
│                          │                                          │
│  Step 2: Populate team dropdown with resolved teams                │
│                          │                                          │
│  Step 3: On team selection, fetch active members                   │
│          (JobStatus != 'Resigned')                                 │
│                          │                                          │
│  ┌──────────────────────────────────────────────────┐               │
│  │  EXCEPTION: HR / Admin users in Policy Bundle    │               │
│  │  screen → Show ALL teams (bypass rights filter)  │               │
│  └──────────────────────────────────────────────────┘               │
│                                                                     │
│  ✗ DO NOT use ReportingToID for visibility                         │
│  ✗ DO NOT show inactive/resigned staff                             │
│  ✗ DO NOT show system accounts (deferred)                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaway
> The fundamental architectural decision is: **Use team-based rights (via `StaffRights` table) as the single source of truth** for determining which staff members a user can see. Do NOT use `ReportingTo` for visibility — if a reporting relationship doesn't match the team rights, it means the rights definition needs to be corrected, not the code. This should be a common, shared functionality across all modules (Leave, Tickets, Performance, Observation) with the only exception being HR users in the Policy Bundle screen who see all teams.
