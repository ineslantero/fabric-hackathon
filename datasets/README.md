# Datasets 🗂️

Five Network Rail-inspired datasets, one per team, plus a shared `calendar.csv` at the root of this folder. All data is synthetic but modelled on the shape of real operational data.

Every dataset needs some cleaning before it's ready to use. Each section below lists the specific quirks to watch for — treat these as **hints, not an exhaustive list**. Always check types, nulls, and joins before trusting the numbers.

---

## 📅 Shared file

- **`calendar.csv`** — a continuous daily calendar. Join every fact table to this via its `Date` column, and mark it as the **date table** in your semantic model.

---

## 🚆 Rail Performance — `rail_performance/`

Train performance data: services (planned vs actual), delay events, and cancellations, joined to routes, operators, and regions.

**Files**

| File | Description |
|---|---|
| `services.csv` | One row per service. Scheduled and actual times, route, operator, cancellation flag. **Fact table.** |
| `delay_events.csv` | Delay reasons and minutes attributed to services. Many-to-one with services. |
| `cancellations.csv` | Cancellation type and reason for cancelled services. One-to-one with services when `IsCancelled = True`. |
| `routes.csv` | Route name and region. |
| `operators.csv` | Operator name. |
| `regions.csv` | Region name. |

**Watch out for** ⚠️

- **Inconsistent operator names** — at least one has leading/trailing whitespace *and* lowercase text (e.g. `"  anglia express "`). Trim, clean, and normalise casing before you join.
- **Date and time columns arrive as text** on CSV import — cast `ScheduledDeparture`, `ActualDeparture`, `ScheduledArrival`, `ActualArrival`, and `Date` to proper types.
- **`ActualArrival` may be missing** for cancelled services or where the arrival wasn't logged — decide whether to null, drop, or flag before calculating delays.
- **Delay maths can go negative** (early arrivals). Decide whether "on time = arrived ≤ scheduled" or "on time = arrived within 5 min", and be consistent.
- **`RegionRouteHint` on `services.csv` is a denormalised hint** — the authoritative region comes from `routes → regions`. Reconcile if they disagree.

---

## 🦺 Safety Incidents — `safety_incidents/`

Workplace safety incidents at stations, depots, and track sections, with corrective actions and hours worked to allow normalisation.

**Files**

| File | Description |
|---|---|
| `incidents.csv` | One row per incident. Date, time, location, type, severity, department, cause, closed date. **Fact table.** |
| `corrective_actions.csv` | Actions raised in response to incidents, with owner, due date, and completed date. Many-to-one with incidents. |
| `hours_worked.csv` | Monthly hours worked by department — the denominator for normalised incident rates. |
| `locations.csv` | Location name, type (Depot / Station / Track), region. |
| `regions.csv` | Region name. |

**Watch out for** ⚠️

- **`Date` and `Time` are separate columns** on `incidents.csv` — decide whether to combine them into a single datetime, and be careful with formatting.
- **`ClosedDate` is empty for open incidents** — filter or handle nulls before calculating `Days to Close`, otherwise your averages will be wrong.
- **Corrective actions can complete before or after the due date** — build a boolean flag for on-time closure.
- **Hours worked is monthly**, incidents are daily — aggregate incidents to a monthly grain (or roll `hours_worked` up somehow) before you compute an incident rate.
- **Location names are generic** (`Location 1`, `Location 2`) — that's the seed data, not a bug. Feel free to invent friendlier names if it helps your story.

---

## 🚧 Track Possessions — `track_possessions/`

Planned track possessions for maintenance and renewal, and how they performed against plan.

**Files**

| File | Description |
|---|---|
| `possessions.csv` | One row per possession. Planned and actual start/end, type, reason, contractor, region. **Fact table.** |
| `impacts.csv` | Services impacted, delay minutes, and cancellations per possession. One-to-one or many-to-one with possessions. |
| `contractors.csv` | Contractor name. |
| `locations.csv` | Location name and region (locations here are corridors, prefixed `PL...`). |
| `regions.csv` | Region name. |

**Watch out for** ⚠️

- **Date/time columns are text** — cast `PlannedStart`, `PlannedEnd`, `ActualStart`, `ActualEnd` to datetime before computing durations.
- **Compute overrun carefully** — actual duration can be greater or less than planned; a negative overrun means "finished early". Decide how to handle that in your KPIs.
- **`RegionID` on `possessions.csv` is a denormalised hint** — the authoritative region comes from `locations → regions`. Check whether they always agree, and choose which to trust.
- **Contractor IDs referenced in `possessions.csv` should all exist** in `contractors.csv` — quick distinct-count check before you join.
- **`impacts.csv` has zeros and small numbers** — decide whether a possession with 0 services impacted is a "clean" possession or missing data.

---

## 🛠️ Fleet Operations — `fleet_operations/`

Fleet availability, failures, and maintenance work orders across depots and fleet types.

**Files**

| File | Description |
|---|---|
| `availability_snapshots.csv` | Daily snapshot per fleet unit: status and hours in service. **Fact table.** |
| `failures.csv` | One row per failure event with downtime hours and delay minutes. |
| `maintenance_workorders.csv` | One row per work order, with type (CM / PM), start, end, cause. |
| `fleet_units.csv` | Fleet ID, fleet type, home depot. |
| `depots.csv` | Depot name and region. |
| `regions.csv` | Region name. |

**Watch out for** ⚠️

- **Inconsistent date formats between columns** in `maintenance_workorders.csv` — `Start` is date-only, `End` is a full timestamp with fractional seconds. Cast both carefully and think about how to compute duration.
- **Work order `Type` codes (`CM`, `PM`)** are unlabelled — decode as *Corrective Maintenance* and *Preventive Maintenance* in a small mapping table or a calculated column.
- **`HoursInService` should be between 0 and 24** per day — spot any outliers before averaging.
- **Availability % is a modelling choice** — decide whether it's based on `Status = "In service"` counts, `HoursInService` totals, or something else, and document it.
- **Failures and maintenance both cause downtime** — decide how to attribute downtime hours between the two if they overlap in time.

---

## 🏗️ Rail Infrastructure Project Performance — `rail_infrastructure_project_performance/`

Infrastructure project delivery: budget, actuals, forecast, and milestones across portfolios and regions. The story is predictive — which projects are heading for a cost overrun or milestone slippage, and where should the portfolio team look first.

**Files**

| File | Description |
|---|---|
| `projects.csv` | One row per project. Name, type, portfolio, region, sponsor, GRIP stage, planned and actual start / end dates, status. **Dim.** |
| `tasks.csv` | One row per project × task. Task name, task group, planned start and end. Bridge between projects and financials. |
| `financials.csv` | Task-grain totals: `Budget`, `ProposalEstimate`, `ActualsToDate`, `Accrual`, `FullYearForecast`, `AFC`, `CostRemaining` (£). **Fact table.** |
| `period_spend.csv` | Task × the 13 four-week accounting periods (`P01`…`P13`), in **wide** form. Needs unpivoting before it's useful. **Fact table.** |
| `milestones.csv` | Project milestones with planned and actual dates and a status. **Fact table.** |
| `portfolios.csv` | Portfolio name. |
| `regions.csv` | Region name. |

**Watch out for** ⚠️

- **`period_spend.csv` is in wide form** — 13 columns `P01`…`P13`. Unpivot to `(PeriodNumber, Amount)` before you model it. A useful sanity check: `Sum(P01..P13)` per task should equal `FullYearForecast` on `financials.csv`.
- **Project names are inconsistent** — a mix of UPPER CASE, lower case, project codes, and the occasional leading or trailing space. Trim and normalise before you slice.
- **A few `ActualsToDate` cells are text with commas** (e.g. `"86,541.22"`). Cast carefully so string values don't silently collapse to null.
- **`Budget` and `ProposalEstimate` disagree on some budgeted tasks** — the estimate was re-baselined. Neither is wrong; pick one for your headline KPI and document the choice.
- **Milestones can be `Complete` with no `ActualDate`**, and a small number of `PlannedDate` values arrive in `DD/MM/YYYY` instead of ISO — build a defensive `IsOnTime` flag rather than a naïve `ActualDate ≤ PlannedDate`.

---

## 💡 A note on modelling

These datasets are shaped for **star-schema modelling in Power BI**, but they're not all single-fact. Some teams have one fact table with a few dimensions; others have **multiple fact tables sharing conformed dimensions** (like `regions`, `calendar`, and the team's own project or fleet dim).

That's a valid — and encouraged — pattern. Microsoft's own guidance describes a star schema as *"often containing multiple fact tables, and therefore multiple stars"*. A few principles worth carrying into your model:

- **Keep every fact table at a consistent grain** — one grain per fact; different grains → different fact tables.
- **Conform your dimensions** — the same `regions` or `calendar` filters every fact.
- **Consider the fact-table type** — transaction (one row per event), periodic snapshot (state at a point in time, e.g. daily availability), or accumulating snapshot (rows updated as a workflow progresses, e.g. project milestones).

Useful references while you build:
- [Understand star schema and the importance for Power BI](https://learn.microsoft.com/power-bi/guidance/star-schema)
- [Dimensional modelling in Fabric Warehouse — fact tables](https://learn.microsoft.com/fabric/data-warehouse/dimensional-modeling-fact-tables)
