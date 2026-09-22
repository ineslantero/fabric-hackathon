# Infrastructure Projects — Beginner Step-by-Step Walkthrough

> **Purpose:** a fully guided, click-by-click path through the Infrastructure Projects dataset for someone who has **never opened a Fabric workspace before**. Every step explains what the thing is, why it exists, and where to click.
>
> **What you'll build by the end:**
> - A Fabric **workspace** with a **lakehouse** holding cleaned, modelled data.
> - A **Direct Lake semantic model** — a star schema with 3 fact tables and 4 dimension tables.
> - A **Power BI report** with 3 pages, one of which is drafted by **Copilot**.
> - Optionally, a **pipeline** that runs the whole thing on a schedule and pings **Teams** if anything fails.

---

## Contents

1. [What you'll build (architecture)](#1-what-youll-build-architecture)
2. [Create your workspace and lakehouse](#2-create-your-workspace-and-lakehouse)
3. [Download the CSV files from GitHub](#3-download-the-csv-files-from-github)
4. [Upload the files to the lakehouse](#4-upload-the-files-to-the-lakehouse)
5. [Transform with a Dataflow Gen2](#5-transform-with-a-dataflow-gen2)
6. [Unpivot period spend with a notebook (PySpark)](#6-unpivot-period-spend-with-a-notebook-pyspark)
7. [Build the semantic model](#7-build-the-semantic-model)
8. [Build the Power BI report — with Copilot](#8-build-the-power-bi-report--with-copilot)
9. *(Optional)* [Automate with a pipeline + Teams notification](#9-optional-automate-with-a-pipeline--teams-notification)
10. *(Optional)* [Use Copilot inside the notebook](#10-optional-use-copilot-inside-the-notebook)
11. *(Optional)* [Row-level security by Region](#11-optional-row-level-security-by-region)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. What you'll build (architecture)

Before you touch anything, here's the shape of what we're building:

```
GitHub CSVs (8 files)
        │
        ▼  (Upload)
┌─────────────────────────────┐
│ Lakehouse: LH_Projects      │
│ ┌─────────────────────────┐ │
│ │ Files/raw/              │ │  ← where the raw CSVs land
│ └────────────┬────────────┘ │
│              │              │
│  Dataflow    │ Notebook     │
│  Gen2        │ (PySpark)    │
│              ▼              │
│ ┌─────────────────────────┐ │
│ │ Tables (Delta)          │ │  ← cleaned, star-schema shape
│ │  • dim_project          │ │
│ │  • dim_task             │ │
│ │  • dim_portfolio        │ │
│ │  • dim_region           │ │
│ │  • dim_calendar         │ │
│ │  • fact_financials      │ │
│ │  • fact_period_spend    │ │  ← unpivoted by the notebook
│ │  • fact_milestones      │ │
│ └────────────┬────────────┘ │
└──────────────┼──────────────┘
               │  Direct Lake
               ▼
    Semantic model (SM_Projects)
               │
               ▼
      Power BI report (3 pages)
```

**Terminology in one sentence each:**

- **Workspace** — a folder for all the Fabric items you build. Everyone in your team shares it.
- **Lakehouse** — a storage container inside a workspace. It has two areas: **Files** (any raw file) and **Tables** (Delta tables, which are queryable like SQL tables).
- **Delta table** — a table stored on OneLake using the Delta Lake format. Fast, versioned, and query-able by SQL, Spark, and Power BI.
- **Dataflow Gen2** — a low-code data transformation tool. Think "Excel Power Query, but in the cloud, feeding your lakehouse."
- **Notebook** — a cell-based coding environment. We'll use it once, for one transformation that's easier in code than in the Dataflow UI.
- **Semantic model** — the layer that sits between your Delta tables and Power BI. It holds relationships, measures, and formatting. Reports read from it.
- **Direct Lake** — a Power BI storage mode where the semantic model reads directly from Delta tables in OneLake, with no import step. Fastest option for Fabric.

---

## 2. Create your workspace and lakehouse

### 2.1 Open Fabric

1. Go to [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com) in your browser.
2. Sign in with your work account.

You'll land on the Fabric home page. The left-hand navigation shows **Home**, **Create**, **Browse**, **OneLake**, **Workspaces**, and so on.

### 2.2 Create a workspace (if you don't already have one)

If your organiser has already assigned you a workspace, **skip to 2.3**.

1. Left navigation → **Workspaces** → top-right **+ New workspace**.
2. **Name:** `Team 5 - Infrastructure Projects` (or whatever you like).
3. **License mode:** pick one attached to a **Fabric capacity** (usually **Fabric capacity** or **Trial capacity**). If you don't see a Fabric option, ask your organiser — you can't do this hack on a Pro-only workspace.
4. Click **Apply**. The workspace opens automatically.

### 2.3 Create a lakehouse

1. Inside the workspace, click **+ New item** (top-left, or the big tile if the workspace is empty).
2. In the search box, type `Lakehouse`. Click the **Lakehouse** tile.
3. **Name:** `LH_Projects`. Click **Create**.

Fabric drops you into the **Lakehouse explorer** view. You see two folders in the left pane:

- **Tables** — Delta tables live here.
- **Files** — any raw file (CSV, Parquet, JSON, images) lives here.

Both are empty for now. That's fine.

---

## 3. Download the CSV files from GitHub

Everything for this exercise is in a public GitHub repo.

### 3.1 Open the dataset folder

1. In a new browser tab, go to **https://github.com/ineslantero/fabric-hackathon**.
2. Click the **`datasets`** folder.
3. Click **`infrastructure_projects`**.

You'll see 7 CSV files:

- `financials.csv`
- `milestones.csv`
- `period_spend.csv`
- `portfolios.csv`
- `projects.csv`
- `regions.csv`
- `tasks.csv`

### 3.2 Download each file

For each of the 7 files:

1. Click the file name to open it.
2. On the file page, look at the top-right of the file preview toolbar — click the **Download raw file** button (down-arrow icon; hover to see the tooltip).
3. Save into a folder on your machine — call it `projects-data`.

### 3.3 Grab the shared calendar

1. Back on the repo, click **`datasets`** (breadcrumb at the top).
2. Click **`calendar.csv`**.
3. Same as before — **Download raw file** → save into `projects-data`.

You should now have **8 files** in your `projects-data` folder. Check the count before moving on.

---

## 4. Upload the files to the lakehouse

### 4.1 Open the Files pane

1. Go back to the Fabric tab.
2. Open `LH_Projects` (from the workspace or the left nav → Workspaces).
3. In the left explorer pane, click **Files** to select it.

### 4.2 Create a subfolder for the raw files

Good habit — keep raw uploads separate from anything derived.

1. Right-click **Files** → **New subfolder**.
2. Name it `raw`. Click **Create**.

### 4.3 Upload

1. Click into the new `raw` folder (single-click it in the tree).
2. Click **... (More options)** next to `raw` → **Upload → Upload files**.
3. In the file picker, navigate to your `projects-data` folder.
4. Select **all 8 CSVs** (Ctrl+A on the folder, or drag-select).
5. Click **Open**. A progress toast appears in the top-right.
6. Wait for all 8 to finish. Refresh the folder if needed (right-click `raw` → **Refresh**).

You should now see 8 files inside `Files/raw/`. Click any file → **Preview** to sanity-check the contents.

> 📌 **What just happened.** You've placed 8 CSVs into OneLake (Fabric's storage). They're not tables yet — they're just files. Next we'll turn them into Delta tables.

---

## 5. Transform with a Dataflow Gen2

We'll use Dataflow Gen2 to bring **7 of the 8 files** into Delta tables. The 8th (`period_spend.csv`) needs unpivoting, which is easier in a notebook — we'll do that in Step 6.

### 5.1 Create a new dataflow

1. From your workspace, click **+ New item**.
2. Search for **Dataflow Gen2** → click the tile.
3. **Name:** `DF_Projects_Bronze`.
4. Click **Create**.

The Power Query editor opens. Empty canvas.

### 5.2 Get data from OneLake

1. Click **Get data** (top-left) → **Get data from another source**.
2. In the **Choose data source** dialog, pick **OneLake catalog**.
3. Sign in if prompted — accept the defaults.
4. In the OneLake catalog, find and select your lakehouse `LH_Projects` → click **Connect**.
5. The Navigator opens showing the lakehouse contents. Expand **Files → raw**.
6. In the **Files** section (not Tables), tick **7 of the 8 CSVs:** everything except `period_spend.csv`.
7. Click **Create**.

Power Query loads all 7 as separate queries in the left-hand **Queries** pane.

### 5.3 Promote headers and set types

Power Query loads each CSV with generic `Column1, Column2, …` names. We need to tell it the first row is the header, then fix data types where they matter.

For **each** of the 7 queries:

1. Click the query name in the left pane.
2. **Home tab → Transform → Use first row as headers**.

That's the base step. Now the per-query fixes below.

> 📌 **Note on Detect data type:** you'll probably see *"Didn't detect any better data types"* on most queries — that's because Power Query already inferred sensible types from the CSV. Skip the button and only apply the specific fixes below.

#### `calendar`

- No fixes needed beyond promoting the header. Types are already fine.

#### `regions`

- No fixes needed.

#### `portfolios`

- No fixes needed.

#### `projects`

This has real casing and whitespace dirt to clean up.

1. Click the `ProjectName` column.
2. **Transform tab → Text column group → Format → Trim** — removes leading and trailing spaces (e.g., `"Reading Depot Improvement "` becomes `"Reading Depot Improvement"`).
3. Same column, **Format → Capitalize Each Word** — normalises mixed casing (`"scotland s&c renewals cp7 p1"` becomes `"Scotland S&C Renewals Cp7 P1"`). Not perfect but consistent.
4. Check the date columns (`PlannedStartDate`, `PlannedEndDate`, `ActualStartDate`, `ActualEndDate`) — Power Query should already have typed them as **Date**. Nulls in `ActualEndDate` for projects that haven't finished are expected.

#### `tasks`

- No fixes needed beyond the header promotion.

#### `milestones`

- No fixes needed beyond the header promotion. `PlannedDate` and `ActualDate` come through as Date already.

#### `financials`

- No fixes needed beyond the header promotion. All numeric columns come through as Decimal.

### 5.4 Rename the queries

Before setting a default destination, rename each query so the resulting Delta tables end up with the names we want (the default destination auto-maps query name → table name).

For each query, right-click its name in the **Queries** pane → **Rename** → apply the new name:

| Original query name | Rename to |
|---|---|
| `calendar` | `dim_calendar` |
| `regions` | `dim_region` |
| `portfolios` | `dim_portfolio` |
| `projects` | `dim_project` |
| `tasks` | `dim_task` |
| `milestones` | `fact_milestones` |
| `financials` | `fact_financials` |

### 5.5 Set the default data destination

1. **Home tab → Default data destination → Add**.
2. In the OneLake catalog panel, pick your lakehouse `LH_Projects` → **Connect**.
3. A dialog appears listing all 7 queries. **Make sure all 7 are ticked**, then click **Bind selected queries**.
4. Table names are set automatically from the query names (that's what we renamed them for).

Every query now has a small lakehouse icon at the bottom, confirming the destination.

> 💡 Hover the info icon in the Table name column — you'll see it says *"Automatic"*. That means the table name follows the query name, so if you need to change a table name later, rename the query and the destination follows.

### 5.6 Publish and run

1. **Home tab → Save, Run and Close**.
2. You're returned to the workspace. The dataflow starts running automatically.
3. Wait for it to finish — depending on capacity, 2–5 minutes.
4. Check the **Last refreshed** column next to `DF_Projects_Bronze` in the workspace list to confirm it just ran.

### 5.7 Verify

1. Open `LH_Projects`.
2. In the **Tables** section on the left, you should now see 7 tables (they may be under a `dbo` schema — expand it).
3. Click each one — you should see rows in the preview.

> 📌 **What just happened.** You've turned 7 CSVs into 7 Delta tables. The Dataflow will re-run whenever you trigger it (manually or on a schedule) — pointing at the same source files.

---

## 6. Unpivot period spend with a notebook (PySpark)

`period_spend.csv` has 13 columns `P01` … `P13` — one per four-week accounting period. For a proper star schema, we want **one row per (task × period)** instead. That's a transformation called **unpivot** (or *melt*). It's a one-liner in PySpark.

> 💡 **Why a notebook and not Power Query?** You *could* do this in a Dataflow — Power Query has an **Unpivot columns** button on the Transform tab, and the result would be the same. We're using a notebook here to give you exposure to the code-first path in Fabric. Both patterns are on the Method Menus — pick whichever fits your team when you use this at work.

### 6.1 Take a look at the wide file

Before we transform, let's see what we're starting from.

1. Open `LH_Projects`.
2. In the left explorer, expand **Files → raw** → click `period_spend.csv`.
3. Top-right of the preview → toggle from **File view** to **Table view**.
4. Notice the shape: two key columns (`TaskID`, `ProjectID`) followed by 13 amount columns `P01` through `P13`. This is what "wide form" means. We'll turn it into three columns: `TaskID`, `PeriodNumber`, `Amount`.

### 6.2 Create a notebook

1. From your workspace → **+ New item** → search **Notebook** → **Notebook**.
2. **Name it** `NB_Unpivot_PeriodSpend` (rename via the header if it defaults to *Notebook 1*).
3. Once open, on the left side you'll see the **Lakehouse explorer** pane. Click **+ Sources** → **Existing lakehouse** → tick `LH_Projects` → **Add**.

Now the notebook is wired to your lakehouse — you can reference `Files/` and `Tables/` directly.

### 6.3 The unpivot code

In the first cell (should be a PySpark cell by default; if not, change the language dropdown top-right of the cell to **PySpark (Python)**):

```python
from pyspark.sql.functions import expr, col, regexp_extract

# 1. Read the wide-form CSV from lakehouse Files
period_wide = (
    spark.read
    .option("header", "true")
    .option("inferSchema", "true")
    .csv("Files/raw/period_spend.csv")
)

# 2. Unpivot the P01..P13 columns to (PeriodCode, Amount)
period_cols = [f"P{i:02d}" for i in range(1, 14)]
stack_expr = "stack(13, " + ", ".join([f"'{c}', {c}" for c in period_cols]) + ") as (PeriodCode, Amount)"

period_long = (
    period_wide
    .selectExpr("TaskID", "ProjectID", stack_expr)
    .withColumn("PeriodNumber", regexp_extract("PeriodCode", r"(\d+)", 1).cast("int"))
    .select("TaskID", "ProjectID", "PeriodNumber", "Amount")
)

# 3. Sanity check
print(f"Rows before: {period_wide.count()}")
print(f"Rows after:  {period_long.count()}")
period_long.show(5)
```

Click **▶ Run cell** (top-left of the cell, or Shift+Enter).

**Expected output:**

```
Rows before: 486
Rows after:  6318          # 486 × 13
+---------+----------+-------------+------+
|   TaskID| ProjectID|PeriodNumber|Amount|
+---------+----------+-------------+------+
|TK000001 |    PJ0001|            1|   0.0|
|TK000001 |    PJ0001|            2|   0.0|
...
```

> 💡 If the row count isn't exactly 13× the input, something's off — usually a typo in the `stack_expr` string.

### 6.4 Save to a Delta table

In a **new cell** (click `+ Code` below):

```python
(
    period_long.write
    .mode("overwrite")
    .format("delta")
    .saveAsTable("fact_period_spend")
)
```

Run it. Takes ~15 seconds.

### 6.5 Verify the invariant

The dataset is designed so that `Sum(P01..P13) == FullYearForecast` for every task. Let's confirm the unpivot preserved that:

```python
from pyspark.sql.functions import sum as _sum, abs as _abs

per = spark.table("fact_period_spend")
fin = spark.table("fact_financials")

check = (
    per.groupBy("TaskID").agg(_sum("Amount").alias("SumPeriods"))
       .join(fin.select("TaskID", "FullYearForecast"), on="TaskID")
       .withColumn("Diff", _abs(col("SumPeriods") - col("FullYearForecast")))
)

bad = check.filter("Diff > 0.05").count()
print(f"Rows where Sum(P01..P13) != FullYearForecast: {bad}")
```

Expected: **`0`** — perfect reconciliation.

### 6.6 Confirm in the lakehouse

1. Left pane → refresh the Tables list (right-click `LH_Projects` → **Refresh**).
2. `fact_period_spend` should now appear alongside the 7 tables from Step 5. Click it to preview.

You now have **8 Delta tables** — the full star schema is loaded.

---

## 7. Build the semantic model

The semantic model is the layer Power BI reads from. We'll create it with Direct Lake, which means the report reads straight from your Delta tables — no import.

### 7.1 Create the semantic model

1. Open `LH_Projects`.
2. Top ribbon → **New semantic model**.
3. In the panel that slides out:
   - **Name:** `SM_Projects`
   - **Workspace:** your workspace
   - **Tables:** tick **all 8**.
4. Click **Confirm**.

Fabric creates the semantic model in **Direct Lake on OneLake** mode and opens it in the model editor.

### 7.2 Create relationships

You're in the model view (canvas with tables shown as boxes). We need to wire the star schema up.

Drag from the "many" column to the "one" column in each of these:

| From (many side) | To (one side) |
|---|---|
| `dim_project[PortfolioID]` | `dim_portfolio[PortfolioID]` |
| `dim_project[RegionID]` | `dim_region[RegionID]` |
| `dim_task[ProjectID]` | `dim_project[ProjectID]` |
| `fact_financials[TaskID]` | `dim_task[TaskID]` |
| `fact_period_spend[TaskID]` | `dim_task[TaskID]` |
| `fact_milestones[ProjectID]` | `dim_project[ProjectID]` |
| `fact_milestones[ActualDate]` | `dim_calendar[Date]` |
| `dim_project[PlannedStartDate]` | `dim_calendar[Date]` |

For each drag, a dialog appears — check:
- **Cardinality:** Many to one (*:1)
- **Cross-filter direction:** Single
- **Make this relationship active:** ticked

Click **OK**.

### 7.3 Mark the calendar as a date table

1. Click `dim_calendar` in the table list.
2. In the **Properties** pane on the right → look for **Mark as date table** → tick it.
3. Choose `Date` as the date column.

Without this step, Power BI's time intelligence functions won't work.

### 7.4 Add measures

Measures are the DAX formulas that make your report calculate anything interesting. Create these one by one:

1. Right-click `fact_financials` in the table list → **New measure**.
2. Paste one of the DAX snippets below.
3. Set the format (in the ribbon that appears on selection): currency for £, percentage for %.
4. Press **✓** to save. Repeat for each measure.

**On `fact_financials`:**

```dax
Total Budget = SUM(fact_financials[Budget])
```

```dax
Total AFC = SUM(fact_financials[AFC])
```

```dax
Total Actuals = SUM(fact_financials[ActualsToDate])
```

```dax
AFC Variance = [Total AFC] - [Total Budget]
```

```dax
AFC Variance % = DIVIDE([AFC Variance], [Total Budget])
```

```dax
% Spent = DIVIDE([Total Actuals], [Total Budget])
```

**On `fact_period_spend`:**

```dax
Period Spend = SUM(fact_period_spend[Amount])
```

**On `fact_milestones`:**

```dax
Milestones = COUNTROWS(fact_milestones)
```

```dax
Milestones Complete =
CALCULATE([Milestones], fact_milestones[Status] = "Complete")
```

```dax
Milestones At Risk =
CALCULATE([Milestones], fact_milestones[Status] IN {"At Risk", "Slipped"})
```

```dax
% Milestones Complete =
DIVIDE([Milestones Complete], [Milestones])
```

**Format hints:**
- £ measures: Currency, 0 decimals, £ symbol.
- % measures: Percentage, 1 decimal.

### 7.5 Sanity check

Add a **card visual** to the model view (or use the Data pane) to verify:

- `Total Budget` should be around **£379M**.
- `Total AFC` should be around **£125M** (yes — many projects only have partial forecast; that's real).
- `% Milestones Complete` should be around **43%**.

If numbers are wildly off, check that relationships are active and that measures reference the right table.

Click **Save** (top-left).

---

## 8. Build the Power BI report — with Copilot

### 8.1 Start a new report

1. From the semantic model view, top ribbon → **New report** → **Auto-create**? Choose **Blank report** instead — we want control.
2. Or: workspace → `SM_Projects` (semantic model) → **... → Create report → From scratch**.

A blank canvas opens with all your tables and measures in the **Data** pane on the right.

### 8.2 Page 1 — Portfolio KPIs (build manually)

Rename the page **KPIs** (right-click the tab at the bottom).

Add visuals from the **Visualizations** pane on the right:

1. **5 Card visuals** across the top:
   - `Total Budget`
   - `Total AFC`
   - `AFC Variance`
   - `% Spent`
   - `% Milestones Complete`

2. **Slicer** — top of the page:
   - Field: `dim_portfolio[PortfolioName]`
   - Style: Dropdown.

3. **Clustered column chart:**
   - X: `dim_portfolio[PortfolioName]`
   - Y: `Total Budget` and `Total AFC` (two series)
   - Sort descending by `Total Budget`.

4. **Line chart:**
   - X: `dim_calendar[Date]`
   - Y: `Period Spend`
   - Small multiples: `dim_portfolio[PortfolioName]` (optional).

5. **Bar chart:**
   - Y: `dim_region[Region]`
   - X: `AFC Variance`
   - Data colour → conditional format → red for negative, green for positive.

Save the report as `Rpt_InfrastructureProjects`.

### 8.3 Page 2 — Milestone tracker (build manually)

Rename the page **Milestones**.

1. **2 Cards:**
   - `Milestones Complete`
   - `Milestones At Risk`

2. **Donut chart:**
   - Legend: `fact_milestones[Status]`
   - Values: `Milestones`

3. **Table:**
   - Columns: `dim_project[ProjectName]`, `fact_milestones[MilestoneName]`, `fact_milestones[PlannedDate]`, `fact_milestones[ActualDate]`, `fact_milestones[Status]`
   - Filter to `Status IN {At Risk, Slipped}`.

4. **Matrix:**
   - Rows: `dim_portfolio[PortfolioName]`
   - Columns: `fact_milestones[Status]`
   - Values: `Milestones`

### 8.4 Page 3 — Let Copilot draft it

This is where we use **Copilot in Power BI** to save time on the "cost analysis" page.

1. Bottom-left of the canvas → **+ (new page)**. Rename it **Cost Analysis**.
2. Top ribbon → **Copilot** button (looks like a sparkle icon). If you don't see it, look under **... (More options)** → **Copilot**.
3. Copilot pane opens on the right.
4. Click **Create a report page** → give it a prompt:

   ```
   Create a cost analysis page. Show total AFC and total actuals by
   portfolio. Include a line chart of Period Spend across the year
   split by PortfolioName. Add a table listing the top 10 projects
   with the biggest AFC variance from budget, showing ProjectName,
   Total Budget, Total AFC, and AFC Variance %.
   ```

5. Copilot generates the visuals. Review each one:
   - Does the chart show what you expect?
   - Does the visual use your measures rather than raw columns? (e.g., it should use `[AFC Variance %]`, not sum a raw column.)
6. **Adjust anything wrong** — Copilot's first draft is rarely perfect. Common tweaks: fix the sort order, change chart type, remove extra columns.
7. **Save** the report.

> 💡 **What just happened.** Copilot read your semantic model — the tables, measures, and column names you set up in Step 7 — and generated a page from your prompt. This is exactly why the quality of your semantic model (good column names, descriptive measures) matters: it's what Copilot sees.

### 8.5 Also try Copilot for a narrative summary

Add a **fourth visual** to Page 1 (KPIs):

1. Visualizations pane → **Narrative visual** (looks like a paragraph icon; may be labelled "Narrative").
2. In the narrative pane on the right, click **Copilot** → it drafts a summary of the KPIs in plain English.
3. Edit if you want a specific tone or emphasis.

Save.

**You're done with the core build.** You have a working end-to-end solution: files → clean tables → semantic model → interactive report.

---

## 9. *(Optional)* Automate with a pipeline + Teams notification

Once you've built everything by hand, you can wire the whole thing up so it runs on a schedule. Adds ~20 min.

### 9.1 Create a pipeline

1. Workspace → **+ New item** → **Data pipeline** → name it `PL_Projects_Refresh`.
2. **Blank canvas** with an activities pane at the top.

### 9.2 Add a Dataflow activity

1. Top ribbon → **Activities → Dataflow**.
2. Drag it onto the canvas.
3. Click the activity → **Settings** tab.
4. **Dataflow:** pick `DF_Projects_Bronze` from your workspace.

### 9.3 Add a Notebook activity

1. Ribbon → **Activities → Notebook**.
2. Drag it onto the canvas to the right of the Dataflow activity.
3. Click it → **Settings**.
4. **Notebook:** pick `NB_Unpivot_PeriodSpend`.
5. **Wire the dependency:** drag from the green tick on the right of the Dataflow activity to the Notebook activity. This means "run the notebook only if the dataflow succeeded".

### 9.4 Add a Teams notification on failure

1. Ribbon → **Activities → Teams**.
2. Drag it onto the canvas to the right of the Notebook activity.
3. **Wire it to the failure path:** drag from the **red X** on the right of the Notebook activity to the Teams activity. Repeat from the Dataflow activity's red X, so either failure triggers it.
4. Click the Teams activity → **Settings**.
5. Sign in with your Teams account → pick a **team and channel** (or DM yourself).
6. **Message body:**

   ```
   ⚠️ Projects refresh failed at @{utcnow()}.
   Pipeline: @{pipeline().Pipeline}
   Run ID: @{pipeline().RunId}
   ```

7. **Save** the pipeline.

### 9.5 Trigger and test

1. Ribbon → **Run**. Watch the activities light up green.
2. Then break something on purpose to test the failure path: for example, temporarily rename a source CSV in the lakehouse Files pane, run again. The Teams activity should trigger.

### 9.6 (Bonus) Schedule it

1. Pipeline → **Schedule** icon in the ribbon.
2. Turn on scheduled runs → daily at 06:00.

---

## 10. *(Optional)* Use Copilot inside the notebook

Adds ~10 min. Copilot in Fabric notebooks can generate PySpark from natural language.

1. Open `NB_Unpivot_PeriodSpend`.
2. Top-right of the notebook → **Copilot** button.
3. Add a **new cell** below your existing code.
4. In the cell, click the **Copilot** hint icon (or press the shortcut shown).
5. Try a prompt:

   ```
   Calculate the top 5 tasks by Amount from fact_period_spend
   and join to dim_task to include TaskName and TaskGroup.
   ```

6. Copilot suggests code. Read it before running (important). Run it — check the output makes sense.

Try a few more:
- *"Compute the AFC variance per project and show projects with variance greater than £1M."*
- *"For projects with 'Complete' milestones missing an ActualDate, list ProjectID and MilestoneID."*

The point isn't to keep the code — it's to see how Copilot can accelerate exploration once your data is in Delta tables.

---

## 11. *(Optional)* Row-level security by Region

Adds ~10 min. Restrict what a user sees in the report based on their `Region`.

### 11.1 In the semantic model

1. Open `SM_Projects` → **Model view**.
2. Ribbon → **Modeling → Manage roles**.
3. Click **+ New** → name the role `Anglia`.
4. Under **Tables**, click `dim_region` → **+ Add filter** → write DAX filter:

   ```dax
   [Region] = "Anglia"
   ```

5. Save. Repeat for a couple more regions (e.g., `Scotland`, `Wales & Western`).
6. To test: ribbon → **View as** → tick your role → view the report. You should now only see one region's data.

### 11.2 Assign to a user

1. Publish the report to a workspace (already done if you saved from the web).
2. Semantic model → **... → Security** → pick a role → add a team member's email → save.

Log in as them and confirm they see only their region.

---

## 12. Troubleshooting

**Dataflow refresh fails with "Type mismatch"**
- Usually one of the destination table names got mistyped. Open the Dataflow, check that each query is named exactly as in the mapping in Step 5.4, then re-check the default destination binding under Home → Default data destination.

**Notebook cell hangs on "Waiting for Spark session"**
- First run of the day can take ~30 seconds to start the Spark session. Subsequent cells are near-instant.

**Semantic model refresh says "Column not found"**
- You added a measure that references a column that got renamed in the dataflow. Open the measure and update the reference, or re-add the measure.

**Copilot says "I'm not able to help with that"**
- Try being more specific. Include column names and table names in your prompt. Copilot works best when it can pattern-match to your actual model.

**Direct Lake fallback to DirectQuery warning**
- Fine to ignore for the hack — this happens if your model has a type Direct Lake doesn't support. Query performance is still good.

**Teams activity fails to send**
- Check the connection: pipeline → Teams activity → **Settings** → **Test connection**. If it fails, re-authenticate.

**"Insufficient capacity" error**
- Fabric capacity is being used elsewhere. Wait 2–3 min and try again, or ping your organiser.

---

## What you've built

- ✅ 8 CSVs cleaned and loaded into Delta tables.
- ✅ A dataflow doing the low-code transforms.
- ✅ A notebook doing the code-first unpivot.
- ✅ A Direct Lake semantic model with 8 tables, correct relationships, and 11 measures.
- ✅ A Power BI report with 3 pages, one drafted by Copilot.
- *(Optional)* A pipeline that runs it all and alerts on failure.

That's a full modern data platform, end-to-end, in one workspace. Nice work.

---

**Next steps if you have time:**
- Extend the report to show milestone slippage trends over time.
- Turn on incremental refresh in the dataflow so it only picks up new rows.
- Publish the report as a **Fabric Org App** so stakeholders can view it in a curated bundle.
- Set up a **Power BI data alert** on `AFC Variance %` so you get notified when a portfolio crosses -10%.
