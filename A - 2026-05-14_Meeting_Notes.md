# Meeting Notes — 14th May 2026

## Attendees
- **Senior / Team Lead (Ma'am / Sonal Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Danish** — Developer (Backend)
- **Rahul** — Developer (Circle/Rights System, Ticket Module)
- **Rohini (Roly)** — Frontend Developer (on vacation — referenced)

---

## Agenda
Live **Policy Bundle review session** on FSM environment — walking through Shift Policy Master, Leave Policy Master, LWP Deduction configuration, and the Applicable Bundle Staff view. Identifying bugs, UI improvements, and clarifying field-level semantics.

---

## Discussion Points

### 1. Target Deadline — 10 June

- **Senior announced:** All open items must be resolved by **10 June**.
- **Minimum goal:** At least 25 staff members should have visible policy data — shift times, attendance, and basic leave info.
- **Prerequisite:** Policy Bundles must be correctly configured and staff must be properly assigned before this can happen.
- **Action:** Lavesh and Danish to sit together and streamline the Policy Bundle setup workflow.

### 2. Shift Policy Master — UI Review

The team reviewed the Shift Policy Master screen together on the FSM environment:

#### Fields Reviewed:
| Field | Status | Notes |
|-------|--------|-------|
| **Shift Name** | ✅ Working | Displays correctly |
| **Description** | ⚠️ UI Issue | Text field too small — **increase the description field size** |
| **Start Time / End Time** | ✅ Working | Using dropdown with predefined time slots (e.g., 6:10 AM to 4:50 PM) |
| **Multiple Days** | ⚠️ Unclear | Checkbox exists but no one remembers the exact use case. Possibly for night shifts spanning two calendar days. **Decision:** Leave it for now — someone from the business team will clarify soon |
| **In-Time Relaxation** | ✅ Defined | Minutes of grace allowed for late arrival (e.g., 3 mins) |
| **Out-Time Relaxation** | ✅ Defined | Minutes of grace allowed for early departure |
| **Each Credit Size** | ⚠️ Needs tooltip | The label "Credit Size" is unclear to end users |
| **Cutoff Time** | ⚠️ Needs clarification | Possibly the threshold beyond which late arrival rules change (e.g., >25 mins late → only 1 credit instead of per-minute) |

#### Info Tooltip Requirement:
- **All technical fields** (Credit Size, Cutoff Time, Relaxation Time) must have an **info icon (ℹ️)** that shows a descriptive tooltip on hover.
- **Credit Size tooltip text (draft):** *"This value defines the number of minutes each credit is made of. Default value is 80 minutes. Zero means no credit."*
- **Cutoff Time:** Confirm with business team. Likely means: *"If an employee arrives more than X minutes late, the credit calculation changes to a flat deduction instead of per-minute."*

### 3. Leave Master — "HR Enable" Feature Deep Dive

This was a lengthy discussion about the `IsUserApply` / `HR Enable` flag in the Leave Master.

#### What is "HR Enable"?
- Some leave types should **NOT** be directly visible to all staff on their leave application screen.
- Example: **Maternity Leave** — exists in the policy bundle (which applies to all staff), but only women should see/use it.
- Example: **Admin Saturday** — only applicable to admin staff, not teachers.
- Example: **Diwali Leave** — a vacation leave that is given by the school, not applied by the user.

#### How It Works:
1. A leave type (e.g., Maternity Leave) is added to the **Leave Policy** within a Policy Bundle.
2. The Policy Bundle is applied to **all staff** in the organization.
3. If `HR Enable = ON` for that leave type:
   - The leave does **NOT** appear in the staff's leave application dropdown by default.
   - **HR must explicitly enable** that leave for specific staff members.
   - Staff can then see and apply for that leave.
4. If `HR Enable = OFF`:
   - The leave is visible to **all staff** who have the policy bundle.
   - Any staff member can apply for it directly.

#### Where Does HR Enable This?
- Inside the **Policy Bundle → Applicable Staff** section.
- HR selects specific staff members and enables specific leave types for them.
- **Frontend implementation not yet seen** — Lavesh to investigate the checkbox/toggle mechanism with Suman (who had worked on this feature earlier).

#### Key Takeaway:
> HR Enable is a **visibility gate** — it hides certain leaves from staff until HR explicitly assigns them to specific individuals. This is not about preventing leave application; it's about controlling which leaves appear in a user's dropdown.

### 4. Leave Policy — Duplicate Record Detection

- **Bug found:** When creating a new Leave Policy, the system allowed creating a **duplicate** with the same name as an existing one.
- **Fix required:** Add a **unique name validation** — if `PolicyBundleName` already exists, block creation and show an error.
- **Similar to how Shift Policy** should also prevent duplicates.

### 5. Leave Policy — Auto-Deduction Priority System

Detailed discussion on how leave deduction priority works when a staff member is absent without applying for leave:

#### Concept:
- When an employee is **absent** and hasn't applied for leave, the system auto-deducts from their leave balance.
- The deduction follows a **priority order** defined in the Leave Policy.

#### How Priority Works:
1. If **Auto-Deduction is ON** for Casual Leave (CL) with Priority = 1:
   - Absent days are first deducted from CL.
2. If CL is exhausted, the system moves to the next priority (e.g., Priority 2 = Paid Leave).
3. If all prioritized leaves are exhausted → **LWP** (Leave Without Pay) kicks in.

#### UI Improvement — Priority Dropdown Instead of Toggle:
| Current | Proposed |
|---------|----------|
| Toggle button (ON/OFF) + separate priority assignment | **Single dropdown** with values 0-10 |
| Two-step process: Enable first, then set priority | One-step: Select priority directly (0 = not applicable) |

- **Decision:** Replace the toggle+priority combination with a **single dropdown** (0 to 5 or 0 to 10).
- **Zero (0)** = This leave type is NOT part of auto-deduction.
- **1-10** = Priority order for auto-deduction.

### 6. LWP (Leave Without Pay) — Deduction Tiers

The senior explained the LWP deduction slab system:

| From (Days) | To (Days) | Deduction Multiplier | Example (₹30,000/month salary) |
|-------------|-----------|---------------------|-------------------------------|
| 1 | 6 | 1x (normal per-day) | ₹1,000/day |
| 7 | 12 | 1.25x | ₹1,250/day |
| 13+ | — | 1.5x | ₹1,500/day |

- **Key insight:** LWP deduction is **progressive** — the more unauthorized leaves taken, the higher the per-day deduction.
- **Edge case discussed:** If someone takes the entire month off on LWP, their deduction could exceed their salary (e.g., ₹45,000 deduction on ₹30,000 salary → net salary = ₹0 or negative).
- **This is already handled** in the `HRISLWPDeductionPolicyDetailsStaffSupportStaff` table with `FromVal`, `ToVal`, and `DeductionVal` columns.

### 7. Credit Leave (CRL) — New Leave Type

- **CRL (Credit Leave)** was identified as a new/existing leave type in the Leave Master.
- **Concept:** Related to the credit/deduction system — likely leave earned through accumulated attendance credits.
- **Details:** Paid vs. unpaid is already handled via a flag in the Leave Master. CRL follows the same pattern.
- **Action:** Document the CRL logic and confirm with business team.

### 8. Policy Bundle — Team Assignment Issue (FSM)

- **Bug found:** When viewing the Applicable Staff list in the Policy Bundle on FSM, the **Team** column was empty for support staff.
- **Root cause:** The SP was not deployed to the FSM environment. The API and SP exist but the SP wasn't pushed to FSM.
- **Lavesh confirmed:** SP exists in EDU, needs to be deployed to FSM.
- **Discussion about OrganizationTeam synonym:**
  - The `OrganizationTeam` table is in the Admin database (shared across campuses).
  - It should be accessible via synonym in both EDU and FSM.
  - If teams show in EDU but not in FSM, it's likely a synonym or SP deployment issue.

### 9. Policy Bundle — Applicable Staff View Enhancements

#### Shift Policy Display Format:
| Current | Proposed |
|---------|----------|
| Only shift time displayed (e.g., "6:10 AM - 4:50 PM") | **Shift Policy Name + time in brackets** (e.g., "Admin Second Year Shift (6:10 AM - 4:50 PM)") |

- **Rationale:** Consistent with how Leave Policy and LWP Policy display their names.

#### Multiple Team Selection:
- **Requirement:** When adding staff to a Policy Bundle, allow **multiple team selection**.
- **Use case:** Support staff may span multiple teams (Kitchen, Bus Driver, Delivery, etc.) — HR needs to add staff from multiple teams at once.
- **Similar to:** Student/Staff Observation module where multiple selection is already implemented.

### 10. Leave Count Editability — Applicable Bundle View

- **Debate:** Should the `NoOfLeaves` (leave count) be editable in the Applicable Bundle individual staff view?
- **Senior's clarification:**
  - Leave **name** and **alias**: Should **NOT** be editable — they come from the Leave Master and are fixed.
  - Leave **count** (`NoOfLeaves`): Should be editable for individual staff — some employees may have customized allocations.
  - LWP `FromVal`, `ToVal`, `DeductionVal`: **Editable** per staff.
- **Bug found:** Rohini had made the leave name also editable (misunderstanding during a previous discussion). **Fix:** Make leave name/alias read-only; only the count should be editable.

### 11. Toast Message — Auto-Dismiss

- **UI Bug:** Success/error toast messages currently stay on screen and block interaction.
- **Fix:** Toast messages should **auto-dismiss after 2-3 seconds** or have a **close (×) icon**.
- **Note:** *"Toast message should slide out automatically after a few seconds."*

### 12. Credits Calculation — Quick Math

The team did a quick calculation during the session to verify credit values:
- Staff: **23 credits** (each credit = 90 minutes) → Total = 23 × 90 = **2,070 minutes** → **34.5 hours**
- Support Staff: **20 credits** × 18 = **360 minutes** → **170 max take-home**
- **Credit Min Limit:** The maximum credits a staff member can accumulate in a year.

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Increase description field size in Shift Policy Master | Lavesh/Roly | Medium |
| 2 | Add info (ℹ️) tooltips for Credit Size, Cutoff Time, Relaxation fields | Lavesh/Roly | Medium |
| 3 | Confirm Cutoff Time meaning with business team | Senior | Medium |
| 4 | Add duplicate name validation for Leave Policy and Shift Policy creation | Lavesh | High |
| 5 | Replace auto-deduction toggle+priority with single dropdown (0-10) | Lavesh | High |
| 6 | Investigate HR Enable workflow — where/how HR enables specific leaves for staff | Lavesh | High |
| 7 | Deploy missing SP to FSM for team visibility in Policy Bundle | Lavesh | Urgent |
| 8 | Update Shift Policy display format: Name + Time in brackets | Lavesh | Medium |
| 9 | Implement multiple team selection when adding staff to Policy Bundle | Lavesh | Medium |
| 10 | Make leave name/alias read-only in Applicable Bundle individual view (only count editable) | Lavesh/Roly | High |
| 11 | Toast messages should auto-dismiss after 2-3 seconds | Roly/Akshita | Medium |
| 12 | Document CRL (Credit Leave) logic | Lavesh | Low |
| 13 | **Target: All open items resolved by 10 June** | All | Critical |

---

## Technical References

| Item | Reference |
|------|-----------|
| Shift Policy Master | `HRISShiftPolicy`, `HRISShiftMaster` tables |
| Leave Master | `HRISLeaveMaster` — contains `IsUserApply`, `IsPaid`, `IsHREnabled` flags |
| Leave Policy | `HRISLeavePolicy` → `HRISLeavePolicyDetails` |
| LWP Deduction | `HRISLWPDeductionPolicy` → `HRISLWPDeductionPolicyDetailsStaffSupportStaff` |
| Policy Bundle | `HRISPolicyBundleMaster` → `HRISPolicyBundle` (staff assignments) |
| Applicable Bundle SP | `SP_GetApplicableBundleByStaffID` |
| Organization Team | `OrganizationTeam` (Admin DB, accessed via synonym) |

---

## Key Takeaway
> This was a comprehensive **live review session** of the Policy Bundle system. The major findings were: (1) HR Enable is a visibility gate for leave types — needs frontend investigation, (2) auto-deduction priority should be simplified to a single dropdown, (3) leave names must be read-only in the individual staff view, (4) missing SP deployment to FSM is blocking team visibility, and (5) all items must be closed by 10 June.
