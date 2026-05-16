# Meeting Notes — 15th May 2026

## Attendees
- **Senior / Team Lead (Ma'am)** — Project Lead / HRIS Module Owner
- **Lavesh** — Developer (Backend / API)
- **Danish** — Developer (Backend)
- **Ziya** — Junior Developer (to be onboarded for Student Document module)

---

## Agenda
**Database indexing strategy** for the HRIS attendance tables (`HRIS_MasterAttandenceTable` and the new `HRISLeaveDeductionRecords` table), along with task delegation for upcoming work.

---

## Discussion Points

### 1. Table Structure — Following Fountainhead Pattern

- **Context:** The senior has created two new tables for the FSM environment:
  1. **Master Attendance Table** — follows the same structure as the existing Fountainhead `HRIS_MasterAttandenceTable`.
  2. **Leave Deduction Records Table** — new table to track leave deductions when leave applications are approved.

- **Structure rationale:** The table follows the existing Fountainhead pattern with:
  - Machine Punch In/Out (raw data from biometric device)
  - Final Punch In/Out (corrected/regularized data)
  - Credit/deduction calculation fields
  - Same column layout as the existing attendance reports

- **Key design change — StaffID instead of HRISID/EmployeeCode:**
  - **Previous:** The tables used `HRISID` (employee code like "FSPL001") as the base identifier.
  - **Problem:** When an employee's code changes (e.g., due to department transfer, PSPL transition), both the Master Attendance and Leave Deduction tables needed manual ID updates.
  - **New design:** Use `StaffID` (integer, never changes) as the base.
  - **Display:** The UI will still show the Employee Code (`EmployeeID` from Staff table) — but internally all queries use `StaffID`.
  - **Benefit:** Employee code can change without affecting historical attendance or leave data.

### 2. Indexing Strategy — The Core Discussion

The meeting's primary focus was deciding the optimal indexing for these high-volume tables.

#### The Problem:
- The Fountainhead attendance table has data from **2017-2018** onwards (8+ years).
- **~750 rows inserted daily** (one per staff per day).
- No indexing currently exists → **monthly reports take 30+ seconds to load**.
- Senior: *"Ek mahine ka report nikalna hai toh bhi mujhe aadha minute toh wait karna hi pad raha hai. Woh nahi hona chahiye."*

#### Primary Filter Fields:
The team confirmed that **all queries** on these tables filter on just **three fields**:
1. **StaffID** — Which staff member
2. **RecordDate** — Which date/date range
3. **IsSupportStaff** — Staff vs. Support Staff flag

No other fields are ever used in WHERE conditions.

#### ChatGPT/AI Consultation:
- The senior had consulted ChatGPT with a simplified table structure to get indexing recommendations.
- **Recommendation received:** Create a **composite non-clustered index** on `(StaffID, RecordDate)` with `INCLUDE` for frequently selected columns.
- **Senior's approach:** Instead of two separate indexes, use **one composite index** with three fields:
  ```sql
  CREATE NONCLUSTERED INDEX IX_Attendance_StaffDate
  ON HRIS_MasterAttandenceTable (StaffID, RecordDate, IsSupportStaff)
  INCLUDE (/* frequently selected columns */)
  ```

#### Index Type Decision:
| Type | Decision | Rationale |
|------|----------|-----------|
| **Clustered Index** | Already exists (Primary Key) | One per table |
| **Non-Clustered Index** | ✅ Use this | Composite on (StaffID, RecordDate, IsSupportStaff) |
| **Index on individual columns** | ❌ Not needed | Composite covers both single and range queries |

#### INCLUDE Columns:
- The `INCLUDE` clause should contain the columns that are frequently returned in SELECT but NOT used in WHERE.
- This avoids key lookups (index covers the query entirely).
- If a new field is added later, the non-clustered index can be **ALTERed** (not dropped/recreated).

### 3. WHERE Condition — Query Writing Convention

- **Important convention established:** When writing queries against these tables, the WHERE clause should follow the index column order:
  ```sql
  -- CORRECT: Matches index order
  WHERE StaffID = @StaffID 
    AND RecordDate BETWEEN @FromDate AND @ToDate
    AND IsSupportStaff = 0

  -- ALSO FINE: SQL Server's index engine picks the right order
  WHERE RecordDate BETWEEN @FromDate AND @ToDate 
    AND StaffID = @StaffID
  ```
- **Senior clarified:** The SQL Server index engine is smart enough to pick the right index regardless of WHERE clause order. However, maintaining the convention helps readability and ensures the team is aware of the index structure.

### 4. Index Maintenance — Rebuild vs. Reorganize

A knowledge-sharing session about index maintenance:

#### Fragmentation:
- **What causes it:** Daily inserts of 750 rows + periodic updates (punch in/out corrections, regularization updates) cause **page splits** in the index.
- **Analogy explained (by Danish):** Think of the index as a notebook:
  - Pages fill up sequentially.
  - When a new record needs to go between existing pages (e.g., inserting data for Jan 5 when Jan 1-4 and Jan 8-10 are already indexed), pages split.
  - This leaves **half-empty pages** → slower reads because the engine scans more pages.

#### Rebuild vs. Reorganize:
| Operation | What It Does | When to Use | Downside |
|-----------|-------------|-------------|----------|
| **Reorganize** | Compacts existing pages in-place (online, non-blocking) | Fragmentation < 30% | Minimal impact on live operations |
| **Rebuild** | Drops and recreates the entire index | Fragmentation > 30% | **Blocks** DML operations (INSERT/UPDATE) during execution |

- **Decision:** Use **REORGANIZE** for routine maintenance. Only use REBUILD for severe fragmentation (>30%).
- **Reason:** The table has **live operations** — biometric data inserts happen throughout the day. REBUILD would block insertions.

#### Maintenance Schedule:
| Setting | Value |
|---------|-------|
| **Frequency** | Weekly (Sunday night) |
| **Method** | SQL Agent Job |
| **Logic** | Check fragmentation percentage → if <30% REORGANIZE, if >30% REBUILD |
| **Time** | Night hours (when no staff is using the system) |

### 5. Same Indexing for Leave Deduction Table

- The **same indexing pattern** should be applied to the new `HRISLeaveDeductionRecords` table.
- **Fields:** `StaffID`, `RecordDate`, `IsSupportStaff`
- **Why:** When a leave application is approved, a record is inserted into this table. The same date+staff queries will be used to check leave deductions.

### 6. Impact on Regularization

- **Senior reminded Lavesh:** When **regularization** features come in, they will run UPDATE queries on the Master Attendance table (updating Final In/Out times).
- These UPDATEs will cause additional page reshuffling in the index.
- The regularization fields are already part of the table schema — need to account for them in the INCLUDE list.

### 7. Trigger Work — Next Step for Senior

- **Senior's next task:** Build the **trigger** on the FSM database that:
  1. Receives biometric punch data from the device.
  2. Inserts raw punch data into the attendance table.
  3. Calculates credits, deductions, and work hours.
  4. Fills in Machine In/Out and Final In/Out fields.
- This follows the same pattern as the existing Fountainhead trigger (`Stode_HRIS_MasterTable`).
- **Goal:** Once the trigger is working, even without the credit/deduction display, the basic attendance data (In/Out times, total hours) should be visible for FSM staff.

### 8. Session with Chirag — Tomorrow

- **Chirag** (likely DBA or senior architect) is coming on-site tomorrow.
- The indexing strategy discussed today will be presented to him for validation.
- The team needs to have the **script ready** before he arrives.
- **Senior will share:**
  - The table scripts (both tables)
  - The ChatGPT conversation/analysis
  - The proposed index creation scripts
- **Lavesh and Danish** should also independently research:
  - Impact of `IN` clause with comma-separated StaffIDs on non-clustered index performance.
  - Alternative AI prompts for more specific indexing advice.

### 9. Crowd-Check — Multiple StaffID IN Queries

- **Senior raised a concern:** When fetching attendance for an entire team (e.g., 20 staff), the query uses `StaffID IN (1, 2, 3, ... 20)`.
- **Question:** Does a non-clustered index on StaffID handle `IN` with multiple values efficiently?
- **Action:** Research this (using AI tools) and discuss with Chirag tomorrow.
- The concern is relevant because bulk queries (entire team, entire month) are common in attendance reports.

### 10. Task Delegation — Ziya

- **Ziya** (junior developer) is being brought onto the HRIS project for a small task:
  - **Task:** Build the **Student Document Master** CRUD API.
  - **Danish** will explain the table structure.
  - **Senior** will teach the Web API pattern.
  - **Timeline:** Must be done quickly — Ziya is going on leave from Sunday.

### 11. Pending HRIS APIs

- Senior mentioned having **1-2 small APIs** to build in HRIS today.
- These are quick tasks, not related to the main indexing discussion.

---

## Action Items

| # | Task | Assigned To | Priority |
|---|------|------------|----------|
| 1 | Create composite non-clustered index script for Master Attendance table: `(StaffID, RecordDate, IsSupportStaff)` with INCLUDE columns | Senior + Lavesh | **Urgent** |
| 2 | Apply same indexing pattern to `HRISLeaveDeductionRecords` table | Senior + Lavesh | High |
| 3 | Research: Impact of `IN` clause with multiple comma-separated StaffIDs on non-clustered index | Lavesh + Danish | High |
| 4 | Share table scripts + ChatGPT analysis + index scripts via email/project folder | Senior | Today |
| 5 | Prepare for session with Chirag tomorrow — have index scripts ready | All | Urgent |
| 6 | Set up SQL Agent Job for weekly index maintenance (REORGANIZE, Sunday night) | Senior | High |
| 7 | Build biometric trigger for FSM database (same pattern as Fountainhead `Stode_HRIS_MasterTable`) | Senior | High |
| 8 | Student Document Master CRUD API — table + SP + Web API | Ziya (guided by Danish + Senior) | Medium |
| 9 | Commit table scripts and SP files to Web API project repository | Senior | Today |
| 10 | Maintain WHERE clause convention: StaffID → RecordDate → IsSupportStaff | All developers | Ongoing |

---

## Technical References

| Item | Reference |
|------|-----------|
| Master Attendance Table | `HRIS_MasterAttandenceTable` (both Fountainhead and FSM) |
| Leave Deduction Table | `HRISLeaveDeductionRecords` (new) |
| Existing Trigger | `Stode_HRIS_MasterTable` (Fountainhead/INVENTORY.MDF) |
| Index Type | Non-Clustered Composite on `(StaffID, RecordDate, IsSupportStaff)` |
| Maintenance | Weekly REORGANIZE via SQL Agent Job (Sunday night) |
| Fragmentation Threshold | <30% → REORGANIZE, >30% → REBUILD |
| Daily Insert Volume | ~750 rows/day (one per staff per day) |
| Data History | Since 2017-2018 (~8 years of data) |

---

## Indexing Decision Summary

```
┌──────────────────────────────────────────────────────────────────┐
│               HRIS Attendance Indexing Strategy                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Table: HRIS_MasterAttandenceTable                               │
│  Table: HRISLeaveDeductionRecords                                │
│                                                                  │
│  ┌─────────────────────────────────────────────┐                 │
│  │  NON-CLUSTERED COMPOSITE INDEX              │                 │
│  │  Key:     StaffID, RecordDate, IsSupportStaff│                │
│  │  Include: [frequently selected columns]      │                │
│  └─────────────────────────────────────────────┘                 │
│                                                                  │
│  ✅ Covers: Single staff + date range queries                    │
│  ✅ Covers: Bulk staff IN (...) + date range queries             │
│  ✅ Covers: Staff vs. Support Staff filtering                    │
│  ✅ Alterable: New INCLUDE fields can be added via ALTER         │
│                                                                  │
│  Maintenance:                                                    │
│  ┌─────────────────────────────────────────────┐                 │
│  │  SQL Agent Job — Every Sunday Night          │                 │
│  │  If fragmentation < 30% → REORGANIZE         │                 │
│  │  If fragmentation > 30% → REBUILD            │                 │
│  └─────────────────────────────────────────────┘                 │
│                                                                  │
│  Design Choice: StaffID (integer) over HRISID (varchar)          │
│  Reason: StaffID never changes, HRISID/EmployeeCode can change   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaway
> The 15th May meeting was a **deep database architecture session** focused on indexing the HRIS attendance tables. The key decisions: (1) use a **composite non-clustered index** on `(StaffID, RecordDate, IsSupportStaff)` for both the Master Attendance and Leave Deduction tables, (2) use **REORGANIZE** (not REBUILD) for weekly maintenance to avoid blocking live operations, (3) switch from HRISID to **StaffID** as the base identifier to avoid cascading updates when employee codes change, and (4) present the strategy to Chirag tomorrow for validation. The trigger for the FSM database is the senior's next priority.
