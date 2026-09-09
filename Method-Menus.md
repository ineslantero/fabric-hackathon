# Method Menus 🧰

For each stage of the day there's more than one Fabric tool that will get the job done. Use these menus to pick the approach that fits your team's skills — and try something new where you can. ✨

Every method links straight to the Microsoft Learn docs so you can dig deeper on the spot.

---

## 📥 Ingestion

Land your raw files somewhere your team can build on. Pick **at least one** method for goal 1; try two if you want a stretch goal.

| Method | Pros | Cons | Docs |
|---|---|---|---|
| **Upload to Lakehouse Files** | Fastest way to start; zero setup. | No history, no schedule, no lineage. | [Lakehouse: upload files](https://learn.microsoft.com/fabric/data-engineering/lakehouse-overview) |
| **Data Factory pipeline — Copy activity** | Repeatable and parameterisable; 100+ source connectors. | Auth setup for private sources; connector-specific limits. | [Copy activity](https://learn.microsoft.com/fabric/data-factory/copy-data-activity) · [Pipelines overview](https://learn.microsoft.com/fabric/data-factory/pipeline-runs) |
| **Dataflow Gen2** | Low-code Power Query UX; great for Excel and JSON reshape at ingest. | Harder to version-control; slower on large data. | [Dataflow Gen2 overview](https://learn.microsoft.com/fabric/data-factory/dataflows-gen2-overview) · [Get started](https://learn.microsoft.com/fabric/data-factory/create-first-dataflow-gen2) |
| **OneLake shortcut** | Zero-copy, always-fresh view of external storage. | Needs the external store and permissions. | [OneLake shortcuts](https://learn.microsoft.com/fabric/onelake/onelake-shortcuts) |
| **Notebook (`spark.read`)** | Maximum flexibility for awkward JSON and mixed schemas. | Overkill for simple CSVs; needs code comfort. | [Fabric notebooks](https://learn.microsoft.com/fabric/data-engineering/how-to-use-notebook) · [Lakehouse and notebooks](https://learn.microsoft.com/fabric/data-engineering/lakehouse-notebook-explore) |

---

## 🧹 Transform and modelling

Turn raw files into cleaned tables and a semantic model you can report on. Pick the tool(s) that suit the shape of your data and your team's comfort zone.

| Method | Pros | Cons | Docs |
|---|---|---|---|
| **Dataflow Gen2** | Visual step-by-step; familiar Power Query UX for cleaning and casting. | Slower on large data; harder to source-control. | [Dataflow Gen2 overview](https://learn.microsoft.com/fabric/data-factory/dataflows-gen2-overview) · [Transformations reference](https://learn.microsoft.com/power-query/power-query-ui) |
| **Notebook (PySpark / Spark SQL)** | Any transform; testable, versionable; scales well. | Needs code comfort; more setup than a dataflow. | [Fabric notebooks](https://learn.microsoft.com/fabric/data-engineering/how-to-use-notebook) · [Spark SQL in Fabric](https://learn.microsoft.com/fabric/data-engineering/lakehouse-notebook-explore) |
| **SQL analytics endpoint (views)** | Fast gold layer with no data movement; T-SQL familiarity. | Read-only — no inserts or updates. | [SQL analytics endpoint](https://learn.microsoft.com/fabric/data-engineering/lakehouse-sql-analytics-endpoint) · [T-SQL surface area](https://learn.microsoft.com/fabric/data-warehouse/tsql-surface-area) |
| **Copilot in a Fabric notebook** | Generates PySpark or Spark SQL from a plain-English prompt; explains and fixes code. | Always review the code before running; quality depends on clear prompts and good column names. | [Copilot in Fabric notebooks](https://learn.microsoft.com/fabric/data-engineering/copilot-notebooks-overview) · [Chat-magics in notebooks](https://learn.microsoft.com/fabric/data-engineering/copilot-notebooks-chat-magics) |
| **Semantic model — relationships, DAX, calculation groups** | Where your star schema, measures, and RLS live; Direct Lake model is free with the Lakehouse. | Direct Lake can silently fall back to DirectQuery on unsupported types. | [Direct Lake](https://learn.microsoft.com/fabric/get-started/direct-lake-overview) · [Semantic models in Fabric](https://learn.microsoft.com/power-bi/connect-data/service-datasets-understand) · [DAX reference](https://learn.microsoft.com/dax/) |

> 💡 **Modelling tips**
> - Aim for a small **star schema**: one fact table + a few dimension tables + the shared `calendar`.
> - Mark `calendar` as the **date table** (Table tools → Mark as date table).
> - Keep raw columns but add cleaned or derived columns alongside (e.g. `ArrivalDelayMinutes`, `OverrunMinutes`, `DurationMinutes`).
> - Write a few **key measures** early — a working number lets you validate the model before you invest in visuals.

---

## 📊 Consume

Turn the model into something a stakeholder can actually use.

| Method | Pros | Cons | Docs |
|---|---|---|---|
| **Power BI interactive report** | Rich pages, visuals, slicers, drill-through; the default for a reason. | Easy to overload pages; mobile and accessibility need thought. | [Create reports in Power BI](https://learn.microsoft.com/power-bi/create-reports/) · [Visualization types](https://learn.microsoft.com/power-bi/visuals/power-bi-visualization-types-for-reports-and-q-and-a) |
| **Copilot in Power BI** | Drafts pages, suggests visuals, writes narrative summaries in seconds. | Quality depends on good column names and descriptions; always review before sharing. | [Copilot in Power BI](https://learn.microsoft.com/power-bi/create-reports/copilot-introduction) · [Enable Copilot](https://learn.microsoft.com/fabric/get-started/copilot-enable-fabric) |
| **Fabric Org App** | Curated bundle of reports and dashboards for a consumer audience. | Consumers still need appropriate licences. | [Publish an app in Power BI](https://learn.microsoft.com/power-bi/collaborate-share/service-create-distribute-apps) |
| **Embed in Teams** | Puts the report where people already work; discussion happens inline. | Best for reports you actively maintain, not one-offs. | [Power BI in Teams](https://learn.microsoft.com/power-bi/collaborate-share/service-embed-report-microsoft-teams) |
| **Publish and share a link** | Simplest way to share — a link into the workspace or app. | Check permissions and RLS before sharing broadly. | [Share reports and dashboards](https://learn.microsoft.com/power-bi/collaborate-share/service-share-dashboards) |

---

## 🔗 Cross-cutting docs

- [Microsoft Fabric documentation home](https://learn.microsoft.com/fabric/)
- [Fabric decision guide: choose a data store](https://learn.microsoft.com/fabric/fundamentals/decision-guide-data-store)
- [Fabric decision guide: copy activity, dataflow, or Spark](https://learn.microsoft.com/fabric/fundamentals/decision-guide-pipeline-dataflow-spark)
- [Microsoft Fabric roadmap](https://learn.microsoft.com/fabric/release-plan/)
