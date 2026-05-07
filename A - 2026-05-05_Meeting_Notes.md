# Meeting Notes — 5th May 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Roly (Roni)** — Developer (Frontend / UI)

---

## Agenda
Discussion on the **Staff-wise Access / Permissions Management** system — specifically the UI structure, API requirements, and granular access control for Organization Groups, Departments, and Teams.

---

## Discussion Points

### 1. Staff Permission Assignment — Current State Review

- **Context:** Lavesh needed an API to support the staff permission assignment feature. The question was: "What API is needed?"
- **Senior's response:** The existing **`Staff Wise Access`** feature (in the Users section) already has an API that allows assigning roles to staff. The question was whether the existing one meets the needs or something new is required.
- **Existing flow:** In the "Users" section, when a staff member is added, roles can be assigned to them. This existing structure was identified as the foundation.

### 2. Permission Levels — Group, Department, Team

- **Three permission levels were clarified:**
  1. **Group Level (Organization Group):** Giving access at the Group level (e.g., "Administration") grants access to **all departments and teams** under that group.
  2. **Department Level:** Giving access at the Department level grants access to **all teams** within that department only.
  3. **Team Level:** Giving access at the Team level grants access to **only that specific team**.

- **The UI should support all three levels** — the admin should be able to select at whichever granularity is appropriate.

### 3. Module-Level Access Control

- **Existing structure:** The system already differentiates between:
  - **Admin / Super Admin** → sees all modules.
  - **Team Group** based access → sees only permitted modules.
- **Senior's requirement for granular control:**
  - It should be possible to give access at the **module level** — e.g., assign a staff member access to the "360 Degree Review" module specifically.
  - Within a module, the access should be further granular:
    - **View** — can see the data.
    - **Create** — can create new records.
    - **Edit** — can modify existing records.
    - **Delete** — can remove records.
  - Not all operations need to be enabled — e.g., a staff member may have **View** access only, without Create/Edit/Delete.

### 4. UI Structure — What Was Built

- **Roly had built an initial UI** for the access management screen.
- **Senior's review observations:**
  - The UI used radio buttons for Group/Department/Team selection, which was **not the right approach**.
  - The correct approach: **cascading selection** — if Group is selected, Department and Team selectors should also appear allowing further refinement.
  - The module listing was using **dummy data** from AI-generated content, not actual data from the system.
  - A proper module list API was needed.
- **Senior's reaction:** The initial UI structure had promise but needed corrections for the cascading logic and real data integration.

### 5. Hierarchical Access Design (Cascading)

The access control should follow this hierarchy:

```
Group Selected → All Departments & Teams under that Group are accessible
  └─ Optionally filter to specific Department(s)
       └─ Optionally filter to specific Team(s)

Department Selected → All Teams under that Department are accessible
  └─ Optionally filter to specific Team(s)

Team Selected → Only that specific Team is accessible
```

- **Example discussed:**
  - **Nirav Sir** is in the **Operations Team**.
  - Operations Team contains: IT Software, IT Hardware, Play (PYP & NYP).
  - Nirav Sir should only have access to **IT Software** and **IT Hardware**, not Play.
  - So the system should allow: Select Group → Select Department → **Manually select specific Teams**.

### 6. API Requirements Identified

| API | Purpose | Status |
|-----|---------|--------|
| Get All Organization Groups | Populate Group dropdown | Existing (`SP_GetAllOrganizationGroup`) |
| Get Departments by Group | Populate Department dropdown based on selected Group | Needs verification |
| Get Teams by Department | Populate Team dropdown based on selected Department | Needs verification |
| Upsert Staff Access | Save the Group/Department/Team + Module + CRUD permissions for a staff member | Needs to be built |
| Get Staff Access | Retrieve assigned permissions for a staff member | Needs to be built |
| Delete Staff Access | Remove a specific access assignment | Had an issue — was failing silently |

### 7. Delete Module — Issue Identified

- **Issue:** When trying to delete a module access entry (one that was added as an example), it was **not working correctly**.
- **Clarification:** The entry was an **Add** (new record), not an Edit. The delete operation should work on added records.
- **Action item:** Lavesh to investigate the delete functionality and fix it.

### 8. Table Structure for Permissions

- **Senior instructed:** Lavesh needs to create a **table with three sub-sections** to support the permission data:
  1. **Staff ↔ Organization Group mapping**
  2. **Staff ↔ Department mapping**
  3. **Staff ↔ Team mapping**
- Each section ties back to the module-level CRUD permissions.
- The senior offered to provide a reference/example for the table design.

### 9. Pending Work Check

- **Senior asked about pending items** on her side.
- **Lavesh confirmed:** The main pending items were the access management work and some SQL queries for observation-related features.
- **Observation module:** There were 4 pages of SQL queries related to the **Staff Observation** feature that needed to be covered. The senior asked to complete them and then she would do testing.
- **Senior noted a conflict** in the codebase — asking Lavesh to flag it if found.

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Fix the cascading logic in the Access Management UI — Group → Department → Team selection | Roly | High |
| 2 | Replace dummy/AI-generated data with real module data from APIs | Roly | High |
| 3 | Build/verify APIs: Get Departments by Group, Get Teams by Department | Lavesh | High |
| 4 | Build Upsert Staff Access API — handle Group/Department/Team + Module + CRUD permissions | Lavesh | High |
| 5 | Build Get Staff Access API — retrieve all assigned permissions for a staff member | Lavesh | High |
| 6 | Fix the Delete Module Access functionality | Lavesh | Medium |
| 7 | Design the permissions table structure (3 sub-sections: Group, Department, Team mapping) | Lavesh | High |
| 8 | Complete the 4 pages of Staff Observation SQL queries | Lavesh | Medium |
| 9 | Lavesh to create the table; Senior to review and test | Lavesh / Senior | Ongoing |

---

## Technical References

| Item | Reference |
|------|-----------|
| Organization Group SP | `SP_GetAllOrganizationGroup` (SchoolERPAdminDB) |
| Staff Observation SP | `SP_GetAllStaffObservationQuestions` |
| Staff Observation DAL | `StaffObservationQuestionDAL.cs` |
| Policy Bundle Controller | `HRISPolicyBundleMasterController.cs` |
| LWP Deduction Policy SP | `SP_UpsertHRISLWPDeductionPolicyDetails` |

---

## Permission Model Summary

```
┌─────────────────────────────────────────────────────┐
│                  Staff Access Entry                  │
├─────────────────────────────────────────────────────┤
│  Staff ID                                           │
│  ├── Organization Group (optional)                  │
│  │    ├── Department (optional refinement)           │
│  │    │    └── Team (optional refinement)            │
│  │    └── [All Departments if not specified]         │
│  ├── Department (without Group)                     │
│  │    └── Team (optional refinement)                │
│  └── Team (direct assignment)                       │
│                                                     │
│  Module Access:                                     │
│  ├── Module Name                                    │
│  │    ├── View   [✓/✗]                              │
│  │    ├── Create [✓/✗]                              │
│  │    ├── Edit   [✓/✗]                              │
│  │    └── Delete [✓/✗]                              │
│  └── ...                                            │
└─────────────────────────────────────────────────────┘
```

---

## Key Takeaway
> The access management system needs to be granular — supporting Group, Department, and Team-level access control with cascading selection, combined with module-level CRUD permissions. The table structure needs to be designed to support this hierarchy. Lavesh to create the backend tables and APIs while Roly fixes the UI cascading logic and data integration.
