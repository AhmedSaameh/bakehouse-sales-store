# Bakehouse Sales Store

An end-to-end data pipeline built on **Databricks** to practice the **medallion architecture** (Bronze → Silver → Gold → Analytics). It takes a raw multi-sheet Excel sales dataset, cleans and enriches it, and produces tables and views ready for **Power BI** dashboards.

## Architecture

```
Excel (7 sheets)
      │
      ▼
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────────┐
 │  Bronze  │ → │  Silver  │ → │   Gold   │ → │  Analytics  │ → Power BI
 │ raw copy │   │ cleaned, │   │ business │   │ BI-ready    │
 │          │   │ joined,  │   │ aggregates│  │ SQL views   │
 │          │   │ enriched │   │          │   │             │
 └──────────┘   └──────────┘   └──────────┘   └─────────────┘
```

All layers live in Unity Catalog under the `workspace` catalog, one schema per layer (`bronze`, `silver`, `gold`, `analytics`).

## Pipeline stages

### 1. Bronze: raw ingestion
Reads every sheet of `Data model.xlsx` and writes each one, unchanged, to a Delta table in `workspace.bronze`. The only tweak is replacing spaces in column names with underscores so Delta accepts them.

Source sheets: fact sales, customer, date, region, product, product subcategory, and product category.

### 2. Silver: cleaning and enrichment
- Checks for **duplicates** in the fact and dimension tables and removes duplicate sales rows
- Checks for **nulls** and fills missing gender values with `Unknown`
- **Standardizes text** (trimming, case normalization) across dimensions
- Validates **referential integrity** between the fact table and dimensions
- **Joins** sales, customers, regions, dates, and the product hierarchy into one unified table (`fact_sales_unified`)
- Adds business logic to produce `fact_sales_enriched`:
  - Profit margin %, markup %, order value category, and profitability status
  - **RFM customer segmentation** (Champions, Loyal Customers, Big Spenders, At Risk, Hibernating, Lost Customers, etc.)
  - **ABC product classification** and performance status (e.g. Slow Mover)
  - Time features: quarter, month, day of week, weekend flag, season
  - Flags such as `needs_attention` and `high_value_transaction`

### 3. Gold: business aggregates
Four tables in `workspace.gold`:

| Table | Grain |
|---|---|
| `customer_summary` | One row per customer |
| `product_performance_summary` | One row per product |
| `monthly_sales_summary` | Month × product category, with month-over-month growth |
| `regional_performance_summary` | Country × region × city × product category |

### 4. Analytics: BI views
Five SQL views in `workspace.analytics`, with KPIs pre-calculated and division-by-zero protection, so Power BI doesn't have to do the heavy lifting:

| View | Grain | Highlights |
|---|---|---|
| `vw_product_performance` | Product | Revenue, profit, margin, revenue contribution %, rank |
| `vw_customer_behavior` | Customer | Lifetime value, repeat-customer flag, purchase frequency, rank |
| `vw_daily_analytics` | Day | Daily revenue and profit, avg order value, day-over-day growth |
| `vw_region_analytics` | Country / region / city | Revenue, margin, revenue per customer, regional rank |
| `vw_customer_product_breakdown` | Customer × product | Revenue share per customer and per product, rankings |

## Tech stack
- Databricks with Unity Catalog
- PySpark and Spark SQL
- Delta Lake
- Power BI (consumption layer)

## Repository contents

| Notebook | Layer |
|---|---|
| `Bronze_Layer_-_Data_Model_Ingestion.ipynb` | Bronze |
| `Silver_Layer_-_Data_Transformation.ipynb` | Silver |
| `Gold_Layer_-_Business_Aggregates.py` | Gold |
| `Analytics_Layer_-_BI_Views.py` | Analytics |

## How to run

1. Import the four notebooks into a Databricks workspace with Unity Catalog enabled.
2. Upload the source Excel file to a Unity Catalog volume and update `file_path` in the Bronze notebook (currently `/Volumes/workspace/default/data_modeling/Data model.xlsx`).
3. Run the notebooks in order: **Bronze → Silver → Gold → Analytics**.
4. Connect Power BI to the `workspace.analytics` schema and build dashboards on the views.

> The source Excel file is not included in this repository.

## Authors

Built by Ahmed Saameh
