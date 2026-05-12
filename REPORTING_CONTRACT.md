# REPORTING_CONTRACT.md - The Immutable Reporting Constitution

**Version**: 1.0  
**Status**: 🔴 SACRED RULE - DO NOT MODIFY WITHOUT ARCHITECT APPROVAL  
**Effective Date**: 2026-05-12  
**Authority**: System Architect (15+ years enterprise)  

---

## 📜 TUYÊN NGÔN (Sacred Rule Declaration)

```
╔════════════════════════════════════════════════════════════════════════════╗
║                                                                            ║
║  ⚖️  ĐIỀU LỆ BẤT KHẢ XÂMPHẠM CỦA HỆ THỐNG BÁOO ÁO                        ║
║                                                                            ║
║  Tài liệu này là HIẾN PHÁP của hệ thống Marketing Sales Report System.   ║
║                                                                            ║
║  ❌ KHÔNG được thay đổi bất kỳ logic nào mà không có phê duyệt của:       ║
║     - Kiến trúc sư cơ sở dữ liệu                                         ║
║     - Kiến trúc sư bảo mật                                               ║
║     - Product Owner (bắt buộc)                                           ║
║                                                                            ║
║  ⚠️  Bất kỳ vi phạm nào dẫn đến:                                          ║
║     - Báo cáo sai                                                         ║
║     - Quyết định kinh doanh sai                                           ║
║     - Mất lòng tin từ quản lý                                             ║
║     - Vấn đề audit compliance                                            ║
║                                                                            ║
║  💼 CÁC NGUYÊN TẮC HẠ ĐÚNG ĐÂY LÀ LUẬT LỆ, KHÔNG PHẢI HỮU VÂN.           ║
║                                                                            ║
╚════════════════════════════════════════════════════════════════════════════╝
```

---

## 🏛️ THE THREE IMMUTABLE LAWS

### **LAW 1: CUMULATIVE SNAPSHOT**
```
Tất cả metrics trong một báo cáo slot là LŨY KẾ (CUMULATIVE) từ 00:00 
của ngày báo cáo đến thời điểm của slot đó.

  00:00 ──────────── 11:55 (Slot 1)
         ├─ ads_cost = 100
         └─ Tổng từ 00:00 đến 11:55 là 100

  00:00 ─────────────────────── 13:55 (Slot 2)
         ├─ ads_cost = 200
         └─ Tổng từ 00:00 đến 13:55 là 200 (BẢO GỒM 100 ở Slot 1)

  ⚡ NHỚ: Slot 2 không phải = Slot 1 + delta. Nó = tổng lũy kế.
```

### **LAW 2: DAILY TOTAL = LATEST REPORT ONLY**
```
Tổng doanh số của một nhân viên trong một ngày = giá trị của báo cáo 
MỚI NHẤT (highest sort_order) của người đó trong ngày đó.

  KHÔNG ĐƯỢC cộng metrics của 4 slot.
  KHÔNG ĐƯỢC lấy average của 4 slot.
  ĐÚNG: Chỉ lấy slot mới nhất (slot có sort_order cao nhất).

Ví dụ:
  Slot 1 (11:55): ads_cost = 100
  Slot 2 (13:55): ads_cost = 200
  Slot 3 (16:55): ads_cost = 350
  Slot 4 (21:00): ads_cost = 500  ← DAILY TOTAL = 500

  KHÔNG BAO GIỜ: 100 + 200 + 350 + 500 = 1,150 ❌
```

### **LAW 3: TEAM_ID IS IMMUTABLE SNAPSHOT**
```
team_id trong slot_reports được LOCK tại thời điểm submit.

Nếu employee chuyển team sau khi submit Slot 2, thì:
  - Slot 2's team_id = cũ (Team A)
  - Slot 3, 4 submit lại = mới (Team B)
  - Cả 2 team đều nhận dữ liệu từ ngày hôm đó

CỤ THỂ: team_id KHÔNG ĐƯỢC UPDATE sau INSERT.
        Nó là audit trail của tổ chức lịch sử.
```

---

## 🔍 CORE LOGIC - Latest Report Per Employee Per Day

### **Algorithm: Find Latest Report (Exact)**

```
FUNCTION fn_get_latest_report_for_employee(
  employee_id: UUID,
  report_date: DATE
) -> slot_report OR NULL

ALGORITHM:
  1. Query slot_reports WHERE:
     - user_id = employee_id
     - report_date = report_date
     - is_current = true  ← CRITICAL: Only active/current reports
     
  2. Sort by:
     - slot.sort_order DESC  ← Higher sort_order = later in day
     - submitted_at DESC NULLS LAST  ← Tiebreaker: latest submit
     
  3. LIMIT 1  ← Return EXACTLY one row
  
  4. If no row found:
     -> Return NULL (employee has NO valid report that day)
     -> Mark as MISSING in aggregation
     
  5. Return the row with all metrics
END FUNCTION
```

### **Pseudo-Code Implementation**

```sql
-- This is the CANONICAL way to query latest report per employee
SELECT sr.*
FROM slot_reports sr
JOIN report_slots rs ON sr.slot_id = rs.id
WHERE sr.user_id = ?employee_id
  AND sr.report_date = ?report_date
  AND sr.is_current = true  -- CRITICAL
ORDER BY rs.sort_order DESC, sr.submitted_at DESC NULLS LAST
LIMIT 1;
```

### **What is_current Flag?**

```
is_current = true:
  ✅ Status = submitted, approved, locked (counts toward totals)
  ✅ Not rejected, not obsoleted by resubmission

is_current = false:
  ❌ Status = rejected (kept for audit, NOT counted)
  ❌ Superseded by newer report (employee resubmitted)

RULE: At most ONE is_current = true per (user_id, report_date).
      All older versions = is_current = false (audit trail).
```

---

## 📊 AGGREGATION CHAIN (Exact Hierarchy)

### **Level 1: Individual Daily Report**
```
latest_report(employee_i, date_d) = ONE row with metrics M

M = {
  ads_cost,
  mess_count,
  data_count,
  closed_orders,
  daily_data_revenue,
  total_orders,
  total_revenue,
  recovered_revenue (derived),
  roas (derived),
  conversion_rate (derived)
}
```

### **Level 2: Team Daily Total**
```
team_daily_total(team_j, date_d) = {
  employee_1: latest_report(employee_1, date_d),
  employee_2: latest_report(employee_2, date_d),
  ...
  employee_N: latest_report(employee_N, date_d)
}

THEN:
  team_total_ads_cost = SUM(M.ads_cost for all employees)
  team_total_mess_count = SUM(M.mess_count for all employees)
  ...
  team_total_revenue = SUM(M.total_revenue for all employees)
  
  team_recovered_revenue = SUM(M.recovered_revenue for all employees)
  team_avg_roas = SUM(M.total_revenue) / SUM(M.ads_cost) if SUM(M.ads_cost) > 0
  team_avg_conversion = SUM(M.closed_orders) / SUM(M.total_orders) * 100 if SUM(M.total_orders) > 0
```

### **Level 3: Manager Daily Total**
```
manager_daily_total(manager_k, date_d) = {
  team_1: team_daily_total(team_1, date_d),
  team_2: team_daily_total(team_2, date_d),
  ...
  team_M: team_daily_total(team_M, date_d)
}

WHERE team_m is assigned to manager_k with manager_team_assignments.is_active = true

THEN:
  manager_total_ads_cost = SUM(team.ads_cost for all assigned teams)
  manager_total_revenue = SUM(team.revenue for all assigned teams)
  ...
  manager_avg_roas = SUM(all_team_revenues) / SUM(all_team_ads_costs)
```

### **Level 4: System Daily Total**
```
system_daily_total(date_d) = {
  team_1: team_daily_total(team_1, date_d),
  team_2: team_daily_total(team_2, date_d),
  ...
  team_X: team_daily_total(team_X, date_d)
}

WHERE team_x is active (team.status = 'active')

THEN:
  system_total_ads_cost = SUM(team.ads_cost for ALL teams)
  system_total_revenue = SUM(team.revenue for ALL teams)
  ...
  system_avg_roas = SUM(all_revenues) / SUM(all_ads_costs)
```

### **Visual Chain**
```
System Total (Level 4)
    │
    ├─ Team A Total (Level 2)
    │    ├─ Employee 1 Latest Report (Level 1)
    │    ├─ Employee 2 Latest Report (Level 1)
    │    └─ Employee 3 Latest Report (Level 1)
    │
    ├─ Team B Total (Level 2)
    │    ├─ Employee 4 Latest Report (Level 1)
    │    ├─ Employee 5 Latest Report (Level 1)
    │    └─ [MISSING]
    │
    └─ Team C Total (Level 2)
         └─ Employee 6 Latest Report (Level 1)

Manager Portfolio View (Level 3):
  Manager X sees: Team A + Team B (assigned teams only)
  Manager X total = Team A + Team B (NOT Team C)
```

---

## 🚨 REPORT STATUS HANDLING (Exact Rules)

### **Status Workflow & Aggregation Impact**

```
┌─────────────────────────────────────────────────────────────┐
│ Status      │ is_current │ Count in Agg │ Reason            │
├─────────────────────────────────────────────────────────────┤
│ draft       │ false      │ ❌ NO        │ Not submitted     │
│ submitted   │ true       │ ✅ YES       │ Official record   │
│ approved    │ true       │ ✅ YES       │ Approved official │
│ rejected    │ false      │ ❌ NO        │ Invalid, keep     │
│             │            │              │ for audit only    │
│ locked      │ true       │ ✅ YES       │ Final, immutable  │
└─────────────────────────────────────────────────────────────┘

KEY POINTS:
  ✅ submitted = FIRST official count (employee submitted)
  ✅ approved = SAME count (leader approved, same data)
  ✅ locked = FINAL state (no more changes)
  ❌ draft = NEVER counted (not submitted yet)
  ❌ rejected = Tracked for audit, NOT counted (obsolete)
```

### **Transition Rules**

```
draft → submitted (employee submits)
submitted → approved (leader approves) [is_current = true, unchanged]
submitted → rejected (leader rejects) [is_current = false, kept for audit]

CRITICAL: rejected is NOT terminal for workflow.
  Employee creates a NEW report (new draft).
  Old rejected report stays with is_current = false.
  
Aggregation NEVER includes:
  - draft reports
  - rejected reports (is_current = false)
  - Any report where is_current = false
```

---

## 🔄 AGGREGATION PSEUDOCODE (EXACT)

### **Query: Get Team Daily Total for Date D**

```python
def get_team_daily_total(team_id, report_date):
    """
    Calculate exact team daily total using LATEST report per employee.
    This is the CANONICAL algorithm.
    """
    
    # Step 1: Get all active employees in team on this date
    employees = query("""
        SELECT DISTINCT tm.user_id
        FROM team_memberships tm
        WHERE tm.team_id = ?team_id
          AND tm.is_active = true
          AND tm.start_date <= ?report_date
          AND (tm.end_date IS NULL OR tm.end_date >= ?report_date)
    """)
    
    # Step 2: For EACH employee, get LATEST report
    team_metrics = {
        'ads_cost': 0,
        'mess_count': 0,
        'data_count': 0,
        'closed_orders': 0,
        'daily_data_revenue': 0,
        'total_orders': 0,
        'total_revenue': 0,
        'recovered_revenue': 0,
        'employee_count': 0,
        'submitted_count': 0,
        'approved_count': 0,
        'rejected_count': 0,  # for audit tracking only
        'locked_count': 0,
        'missing_count': 0
    }
    
    for employee_id in employees:
        # THIS IS CRITICAL: Get exactly ONE report per employee
        latest_report = query("""
            SELECT sr.*
            FROM slot_reports sr
            JOIN report_slots rs ON sr.slot_id = rs.id
            WHERE sr.user_id = ?employee_id
              AND sr.report_date = ?report_date
              AND sr.is_current = true  -- Only active/current
            ORDER BY rs.sort_order DESC, sr.submitted_at DESC NULLS LAST
            LIMIT 1
        """)
        
        if latest_report is None:
            # Employee has NO valid report this day
            team_metrics['missing_count'] += 1
            continue
        
        # Step 3: Add metrics to team totals (NO doubling)
        team_metrics['ads_cost'] += latest_report.ads_cost
        team_metrics['mess_count'] += latest_report.mess_count
        team_metrics['data_count'] += latest_report.data_count
        team_metrics['closed_orders'] += latest_report.closed_orders
        team_metrics['daily_data_revenue'] += latest_report.daily_data_revenue
        team_metrics['total_orders'] += latest_report.total_orders
        team_metrics['total_revenue'] += latest_report.total_revenue
        team_metrics['recovered_revenue'] += latest_report.recovered_revenue
        team_metrics['employee_count'] += 1
        
        # Track status
        if latest_report.status == 'submitted':
            team_metrics['submitted_count'] += 1
        elif latest_report.status == 'approved':
            team_metrics['approved_count'] += 1
        elif latest_report.status == 'locked':
            team_metrics['locked_count'] += 1
        elif latest_report.status == 'rejected':
            team_metrics['rejected_count'] += 1
    
    # Step 4: Calculate derived metrics
    if team_metrics['ads_cost'] > 0:
        team_metrics['avg_roas'] = team_metrics['total_revenue'] / team_metrics['ads_cost']
    else:
        team_metrics['avg_roas'] = 0
    
    if team_metrics['total_orders'] > 0:
        team_metrics['avg_conversion_rate'] = (
            team_metrics['closed_orders'] / team_metrics['total_orders']
        ) * 100
    else:
        team_metrics['avg_conversion_rate'] = 0
    
    return team_metrics
```

---

## ⚠️ ANTI-PATTERN TEST CASE (MANDATORY)

### **Test Case: The Great Addition Mistake**

**Scenario:**
```
Employee: John (john_id)
Date: 2026-05-12 (ngày 12 tháng 5 năm 2026)
Team: Sales Team A

Reports submitted throughout the day:
```

| Slot | Time | Slot Sort Order | ads_cost | mess_count | Status | is_current |
|------|------|-----------------|----------|-----------|--------|-----------|
| 11h55 | 11:55 | 1 | 11,000,000 | 45 | submitted | false |
| 13h55 | 13:55 | 2 | 11,000,000 | 45 | submitted | false |
| 16h55 | 16:55 | 3 | 11,000,000 | 45 | submitted | false |
| 21h00 | 21:00 | 4 | 11,000,000 | 45 | approved | **true** ← CURRENT |

### **❌ WRONG CALCULATION (Anti-Pattern)**

```
Team Daily Total = 11,000,000 + 11,000,000 + 11,000,000 + 11,000,000
                 = 44,000,000 ❌ WRONG!

This assumes metrics are DELTAS, not CUMULATIVE.
This is WRONG and will cause:
  - Revenue overstatement by 4x
  - Incorrect KPI tracking
  - Wrong decision-making by management
  - Audit failure
```

### **✅ CORRECT CALCULATION (Proper)**

```
1. Identify LATEST report for John on 2026-05-12:
   - Query: slot_reports WHERE user_id = john_id AND report_date = 2026-05-12 AND is_current = true
   - Result: The LAST report (21h00, sort_order=4, status=approved, is_current=true)
   
2. Extract metrics from LATEST report only:
   - ads_cost = 11,000,000
   - mess_count = 45
   
3. Team Daily Total includes John's:
   - ads_cost = 11,000,000 ✅ CORRECT
   - mess_count = 45 ✅ CORRECT

KEY INSIGHT:
  The value 11,000,000 appears in ALL 4 slots because metrics are CUMULATIVE.
  From John's perspective:
    - 11h55: "By 11:55, I've spent 11M in ads"
    - 13h55: "By 13:55, I've spent 11M in ads" (same total, no new spend)
    - 16h55: "By 16:55, I've spent 11M in ads" (same total)
    - 21h00: "By 21:00, I've spent 11M in ads" (same total, end of day)
  
  Total ads spend for the day = 11,000,000 (NOT 44M).
```

### **Why This Matters**

```
Impact of the mistake:

Scenario 1: Team with 10 employees, each reports 11M ads spend daily
  
  ❌ WRONG approach:
    Team Daily = 10 employees × 4 slots × 11M = 440M/day (fraudulent)
    Monthly = 440M × 30 = 13.2B (claimed vs actual)
    
  ✅ CORRECT approach:
    Team Daily = 10 employees × 11M = 110M/day (accurate)
    Monthly = 110M × 30 = 3.3B (actual)
    
  Difference: 3.96x overstatement (CRITICAL FAILURE)

Consequences:
  - Budget forecasts completely wrong
  - Marketing ROI calculations impossible
  - Management makes decisions on false data
  - Audit finds massive discrepancies
  - Investor confidence destroyed
  - Career-ending mistake
```

---

## 🔄 EMPLOYEE TEAM CHANGE MID-DAY (Edge Case)

### **Scenario: Employee Transfers Between Teams During Day**

```
Timeline:
  - 06:00: Employee ALPHA is member of Team A
  - 11:55: ALPHA submits Slot 1 → team_id = Team A (snapshot locked)
  - 14:00: HR moves ALPHA from Team A to Team B
  - 13:55: ALPHA submits Slot 2 (was drafted before 14:00) → team_id = Team B
  - 16:55: ALPHA submits Slot 3 → team_id = Team B
  - 21:00: ALPHA submits Slot 4 → team_id = Team B
```

### **Database State After Submission**

```sql
slot_reports for ALPHA on 2026-05-12:

Slot  | Sort | team_id  | ads_cost | status    | is_current
------|------|----------|----------|-----------|------------
1     | 1    | Team A   | 100      | submitted | false
2     | 2    | Team B   | 200      | submitted | false
3     | 3    | Team B   | 300      | submitted | false
4     | 4    | Team B   | 400      | approved  | true ← LATEST

team_memberships for ALPHA:

From     | To       | team_id | is_active
---------|----------|---------|----------
01-JAN   | 14-MAY   | Team A  | false (end_date set when moved)
14-MAY   | NULL     | Team B  | true (active)
```

### **Aggregation Impact**

**Team A Daily Total for 2026-05-12:**
```
Employees: [ALPHA, BETA, GAMMA, ...]

For ALPHA:
  - Query latest report WHERE user_id=ALPHA AND report_date=2026-05-12 AND is_current=true
  - Result: Slot 4 (sort_order=4) with team_id=Team B
  - Question: Does this count toward Team A?
  
ANSWER: NO (with caveats for business logic)
  - team_id=Team B means this report is attributed to Team B
  - Historical: Slot 1 (team_id=Team A) has is_current=false, not counted
  - Result: Team A does NOT get credit for any of ALPHA's reports on this day
  
Impact on Team A daily:
  ads_cost (without ALPHA) = SUM of other team members' latest reports
```

**Team B Daily Total for 2026-05-12:**
```
Employees: [ALPHA, DELTA, EPSILON, ...]

For ALPHA:
  - Latest report: Slot 4 (sort_order=4, team_id=Team B, is_current=true)
  - ads_cost from Slot 4 = 400 (cumulative from 00:00 to 21:00)
  - This counts toward Team B
  
Impact on Team B daily:
  ads_cost = includes ALPHA's 400 + other members' latest reports
```

### **Historical View (For Auditing)**

```
Team A historical records on 2026-05-12:
  - Slot 1 from ALPHA: 100 (team_id=Team A, is_current=false after Slot 4)
  - This is kept in database for audit trail

Team B historical records on 2026-05-12:
  - Slot 2 from ALPHA: 200 (team_id=Team B, is_current=false)
  - Slot 3 from ALPHA: 300 (team_id=Team B, is_current=false)
  - Slot 4 from ALPHA: 400 (team_id=Team B, is_current=true) ← COUNTED
  
Audit benefit: Can trace "what was ALPHA's assignment when each report was made"
```

### **Business Logic Choice**

```
Option A (Current Implementation):
  - Latest report's team_id determines which team gets credit
  - Team A loses ALPHA's data when they transfer
  - Pro: Simple, clean responsibility transfer
  - Con: Earlier reports from Team A disappear from their view

Option B (Alternative - Not Recommended):
  - Allocate Slot 1 to Team A, Slots 2-4 to Team B
  - Both teams get "their" portion of the metrics
  - Pro: Honors work done for each team
  - Con: Complex, potential double-counting, reconciliation nightmare

DECISION: Use Option A
  - team_id is snapshot at submission
  - latest report determines team ownership
  - Historical records kept but not aggregated
  - Simpler, cleaner, less room for error
```

---

## 📍 REPORT STATUS CLASSIFICATION

### **Definition: MISSING vs INCOMPLETE vs COMPLETE**

```
For Team Daily Aggregate on DATE D:

MISSING Report:
  ├─ Employee has NO report for that day (is_current=true)
  ├─ Reason: Didn't submit any slot report
  ├─ Impact: Employee_count ↓, but team continues aggregation
  └─ Action: Flag in dashboard, escalate to manager

INCOMPLETE Report:
  ├─ Employee submitted SOME slots but not all 4
  ├─ Example: Only submitted Slots 1 and 2, missing Slot 3 & 4
  ├─ Aggregation: Use LATEST report submitted (Slot 2) → is_current=true
  │              Slots 3 & 4 missing → is_current=false (none exist)
  ├─ Impact: Use what we have; team still gets partial data
  └─ Action: Remind employee to complete remaining slots

COMPLETE Report:
  ├─ Employee submitted all 4 slots (Slot 4 = is_current=true)
  ├─ Status: submitted, approved, locked (all valid)
  ├─ Aggregation: Use Slot 4 metrics
  └─ Action: None; report complete

REJECTED Report:
  ├─ Employee submitted, but leader rejected
  ├─ Example: Slot 2 submitted → rejected; then new Slot 2' submitted
  ├─ Database: Old Slot 2 (is_current=false), new Slot 2' (is_current=true)
  ├─ Aggregation: Use Slot 2', ignore old Slot 2
  ├─ Impact: Audit trail shows rejection history
  └─ Action: None; new report replaces old one
```

### **Detection Query**

```sql
-- Classify each employee's report status
SELECT 
  tm.user_id,
  p.full_name,
  CASE 
    WHEN sr_latest.id IS NULL THEN 'MISSING'
    WHEN MAX(rs.sort_order) < 4 THEN 'INCOMPLETE'
    WHEN MAX(rs.sort_order) = 4 THEN 'COMPLETE'
  END AS report_classification,
  COUNT(sr.id) AS submitted_slot_count,
  MAX(rs.sort_order) AS latest_slot_order,
  sr_latest.status AS latest_status
FROM team_memberships tm
JOIN profiles p ON tm.user_id = p.id
LEFT JOIN slot_reports sr ON sr.user_id = tm.user_id 
  AND sr.report_date = ?report_date
LEFT JOIN report_slots rs ON sr.slot_id = rs.id
LEFT JOIN (
  SELECT sr.* FROM slot_reports sr
  JOIN report_slots rs ON sr.slot_id = rs.id
  WHERE sr.report_date = ?report_date
    AND sr.is_current = true
  ORDER BY rs.sort_order DESC
  LIMIT 1
) sr_latest ON sr_latest.user_id = tm.user_id
WHERE tm.team_id = ?team_id
  AND tm.is_active = true
GROUP BY tm.user_id, p.full_name, sr_latest.id, sr_latest.status
ORDER BY p.full_name;
```

### **Aggregation Rules by Classification**

```
MISSING:
  ✗ Do NOT count any metrics
  ✓ Increment missing_count
  ✓ Mark in dashboard for escalation

INCOMPLETE:
  ✓ Count LATEST report's metrics (even though <4 slots)
  ✓ Alert: "Employee only submitted Slot 2/4"
  ✓ Mark in dashboard for reminder

COMPLETE:
  ✓ Count all metrics from Slot 4 (is_current=true)
  ✓ Mark status: submitted/approved/locked
  ✓ No alert needed

REJECTED:
  ✗ Old rejected report (is_current=false) NOT counted
  ✓ New resubmitted report (is_current=true) counted
  ✓ Audit log shows both versions for compliance
```

---

## 🎯 REPORTING CONTRACT ENFORCEMENT

### **Who Must Enforce This?**

```
DATABASE LAYER:
  ✅ RLS policies prevent wrong role from accessing data
  ✅ Constraints enforce business rules
  ✅ Triggers prevent invalid transitions
  ✅ Views enforce correct aggregation logic

APPLICATION LAYER:
  ✅ Edge Functions validate submission logic
  ✅ Audit logs record all state changes
  ✅ Dashboard queries use canonical functions (fn_latest_daily_reports_for_team)
  ✅ API responses mirror database truth

HUMAN LAYER:
  ✅ Product Owner reviews changes
  ✅ Architect approves schema modifications
  ✅ QA tests anti-patterns
  ✅ Data analysts validate monthly totals
```

### **Testing Requirements**

```
Unit Tests:
  ✅ fn_latest_daily_reports_for_team() returns exactly 1 row per employee
  ✅ Anti-pattern test case: ads_cost should be 11M, not 44M
  ✅ Team aggregation sums correctly
  ✅ Manager aggregation sums correctly
  ✅ System aggregation sums correctly

Integration Tests:
  ✅ Employee submits 4 slots → team total = Slot 4 only
  ✅ Employee transfers team mid-day → latest team_id used
  ✅ Employee resubmits (rejected) → new submission replaces old
  ✅ Missing report → not counted, marked MISSING

Edge Case Tests:
  ✅ All employees missing reports → team_count = 0, totals = 0
  ✅ Mix of submitted/approved/locked → all counted identically
  ✅ Exactly 3 slots submitted (no 4) → use Slot 3 as latest
  ✅ Concurrent submissions (same slot_id) → timestamp decides

Data Validation Tests:
  ✅ Monthly total = SUM(daily latest reports)
  ✅ Team total ≤ System total
  ✅ Manager portfolio ≤ System total
  ✅ recovered_revenue = total_revenue - daily_data_revenue (always)
  ✅ No negative metrics
  ✅ ROAS > 0 only if ads_cost > 0
```

---

## 📋 CANONICAL QUERIES (Copy-Paste Safe)

### **Query 1: Get Latest Report Per Employee Per Day**
```sql
SELECT sr.*, p.full_name, t.name as team_name, rs.slot_name, rs.sort_order
FROM slot_reports sr
JOIN profiles p ON sr.user_id = p.id
JOIN teams t ON sr.team_id = t.id
JOIN report_slots rs ON sr.slot_id = rs.id
WHERE sr.user_id = $1
  AND sr.report_date = $2
  AND sr.is_current = true
ORDER BY rs.sort_order DESC, sr.submitted_at DESC NULLS LAST
LIMIT 1;
```

### **Query 2: Team Daily Total (All Metrics)**
```sql
WITH latest_reports AS (
  SELECT sr.*, rs.sort_order
  FROM slot_reports sr
  JOIN report_slots rs ON sr.slot_id = rs.id
  WHERE sr.report_date = $1
    AND sr.is_current = true
  ORDER BY sr.user_id, rs.sort_order DESC, sr.submitted_at DESC NULLS LAST
),
deduplicated AS (
  SELECT DISTINCT ON (user_id) * FROM latest_reports
)
SELECT 
  $2 as team_id,
  $1 as report_date,
  COUNT(*) as employee_count,
  COUNT(*) FILTER (WHERE status = 'submitted') as submitted_count,
  COUNT(*) FILTER (WHERE status = 'approved') as approved_count,
  COUNT(*) FILTER (WHERE status = 'locked') as locked_count,
  SUM(ads_cost) as total_ads_cost,
  SUM(mess_count) as total_mess_count,
  SUM(data_count) as total_data_count,
  SUM(closed_orders) as total_closed_orders,
  SUM(daily_data_revenue) as total_daily_data_revenue,
  SUM(total_orders) as total_orders,
  SUM(total_revenue) as total_revenue,
  SUM(recovered_revenue) as total_recovered_revenue,
  CASE 
    WHEN SUM(ads_cost) > 0 THEN SUM(total_revenue) / SUM(ads_cost)
    ELSE 0 
  END as avg_roas,
  CASE 
    WHEN SUM(total_orders) > 0 THEN (SUM(closed_orders)::NUMERIC / SUM(total_orders)) * 100
    ELSE 0 
  END as avg_conversion_rate
FROM deduplicated
WHERE team_id = $2
GROUP BY team_id, report_date;
```

### **Query 3: Manager Daily Portfolio**
```sql
WITH team_assignments AS (
  SELECT team_id 
  FROM manager_team_assignments 
  WHERE manager_id = $1 AND is_active = true
)
SELECT 
  mta.team_id,
  t.name as team_name,
  fn_team_daily_aggregate(mta.team_id, $2).*
FROM manager_team_assignments mta
JOIN teams t ON mta.team_id = t.id
WHERE mta.manager_id = $1 
  AND mta.is_active = true
ORDER BY t.name;
```

### **Query 4: Identify Missing/Incomplete Reports**
```sql
WITH latest_reports AS (
  SELECT 
    sr.user_id,
    MAX(rs.sort_order) as latest_slot_order
  FROM slot_reports sr
  JOIN report_slots rs ON sr.slot_id = rs.id
  WHERE sr.report_date = $1
    AND sr.is_current = true
  GROUP BY sr.user_id
)
SELECT 
  tm.user_id,
  p.full_name,
  CASE 
    WHEN lr.user_id IS NULL THEN 'MISSING'
    WHEN lr.latest_slot_order < 4 THEN 'INCOMPLETE'
    ELSE 'COMPLETE'
  END as classification,
  COALESCE(lr.latest_slot_order, 0) as submitted_slots,
  sr.status
FROM team_memberships tm
JOIN profiles p ON tm.user_id = p.id
LEFT JOIN latest_reports lr ON tm.user_id = lr.user_id
LEFT JOIN slot_reports sr ON sr.user_id = tm.user_id 
  AND sr.report_date = $1 
  AND sr.is_current = true
LEFT JOIN report_slots rs ON sr.slot_id = rs.id
WHERE tm.team_id = $2
  AND tm.is_active = true
ORDER BY p.full_name;
```

---

## ⚖️ VIOLATION CONSEQUENCES

```
IF a developer or team violates this contract:

🔴 Level 1 Violation: Wrong aggregation logic
   ├─ Example: SUM all 4 slots instead of latest only
   ├─ Consequence: Reports are wrong by 4x
   ├─ Action: Code review rejection, mandatory rewrite
   └─ Punishment: Architecture review required

🔴 Level 2 Violation: Incorrect status handling
   ├─ Example: Counting rejected reports
   ├─ Consequence: Numbers don't match audit trail
   ├─ Action: Immediate revert to production backup
   └─ Punishment: Incident postmortem, code freeze

🔴 Level 3 Violation: Updating team_id after submission
   ├─ Example: Changing team_id after INSERT
   ├─ Consequence: Historical records lost, audit trail broken
   ├─ Action: Database restore, revert all changes
   └─ Punishment: Investigation, potential suspension

🔴 Level 4 Violation: Adding business logic without architect approval
   ├─ Example: New aggregation method without review
   ├─ Consequence: Entire system integrity compromised
   ├─ Action: Emergency war room, complete audit
   └─ Punishment: Escalation to CTO, career impact

FINAL SANCTION: Any breach is treated as data integrity incident.
                Full audit trail. Potential regulatory implications.
                Not a joke. Not flexible.
```

---

## 🔐 AMENDMENT PROCESS

```
To modify this contract:

1. PROPOSAL
   ├─ Technical Design Document (TDD)
   ├─ Impact analysis (which queries change)
   ├─ Test cases (before + after)
   ├─ Rollback procedure (required)
   └─ Business approval from Product Owner

2. REVIEW
   ├─ Database Architect review (mandatory)
   ├─ Security Architect review (mandatory)
   ├─ QA team review (mandatory)
   ├─ At least 2 reviewers must approve
   └─ Any rejection requires new proposal

3. APPROVAL
   ├─ Product Owner final sign-off
   ├─ Architecture board approval
   ├─ Legal/Compliance review (if data sensitive)
   └─ Version bump (this document)

4. IMPLEMENTATION
   ├─ Staged rollout (dev → staging → production)
   ├─ Backward compatibility tests (if applicable)
   ├─ Continuous monitoring post-deployment
   └─ Rollback plan on standby

5. DOCUMENTATION
   ├─ Update DATABASE_BLUEPRINT.md
   ├─ Update API documentation
   ├─ Update test cases
   ├─ Changelog entry with full rationale
   └─ Team training if needed

NO EXCEPTION. NO SHORTCUT. NO "JUST ONE CHANGE".
```

---

## ✅ SIGN-OFF (Digital)

```
This document is APPROVED and EFFECTIVE as of 2026-05-12.

By reading this document, you ACKNOWLEDGE:
  ✅ I understand the 3 Immutable Laws
  ✅ I understand cumulative metrics are NOT deltas
  ✅ I understand daily total = latest report only
  ✅ I understand team_id is immutable snapshot
  ✅ I will use canonical queries only
  ✅ I will test anti-pattern cases
  ✅ I will NOT modify this without approval
  ✅ I accept consequences of violations

System Architect: Database Architect (15+ years)
Product Owner: [To be signed by PO]
Effective Date: 2026-05-12
Version: 1.0

REVIEWED AND APPROVED BY:
  [ ] Database Architect
  [ ] Security Architect
  [ ] Product Owner
  [ ] QA Lead
  [ ] DevOps Lead

This contract is binding on all code written for this system.
```

---

**END OF REPORTING_CONTRACT.md**

**Status**: 🔒 LOCKED - SACRED DOCUMENT  
**Next Review**: 2026-08-12 (90 days)  
**Escalation**: Breach = Incident Management Protocol

---
