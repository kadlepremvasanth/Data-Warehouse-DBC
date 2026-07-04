# Data-Warehouse-DBC
# Orders Data Warehouse — Databricks SQL Project

My first end-to-end data warehousing project, built entirely in **Databricks SQL**. It takes raw order data, moves it through a layered pipeline, and models it into a **star schema** — with a second module demonstrating **incremental loading** and **Slowly Changing Dimensions (SCD Type 1)**.

---

##  What This Project Covers

-  Multi-layer data pipeline (Staging → Transformation → Core)
- Star schema design (Fact + Dimension tables)
-  Surrogate keys generated with `ROW_NUMBER()`
-  Incremental data loading
-  Slowly Changing Dimension (SCD) Type 1 using `MERGE INTO`
---

##  Pipeline Layers

### 1. Staging Layer
Raw source data is copied as-is into a staging table, so the source system is never touched directly.

```sql
CREATE OR REPLACE TABLE ordersDwh.stg_sales AS
SELECT * FROM sales_new_scd.orders;
```

### 2. Transformation Layer
A view cleans the staged data (e.g. removing rows with missing `Quantity`) before it's used downstream.

```sql
CREATE VIEW ordersDwh.trans_sales AS
SELECT * FROM ordersdwh.stg_sales
WHERE Quantity IS NOT NULL;
```

### 3. Core Layer — Star Schema
Dimension tables are built from distinct values in the transformed data, each with a generated surrogate key (`ROW_NUMBER()` over the natural key). The **Fact table** then joins all dimensions on their business keys to pull in the surrogate keys.

| Table | Type | Grain |
|---|---|---|
| `DimCustomers` | Dimension | One row per customer |
| `DimProducts` | Dimension | One row per product |
| `DimRegion` | Dimension | One row per region/country |
| `DimDate` | Dimension | One row per order date |
| `FactSales` | Fact | One row per order line |

Example — building `DimCustomers`:

```sql
CREATE OR REPLACE VIEW ordersDWH.view_DimCustomers AS
SELECT T.*, ROW_NUMBER() OVER (ORDER BY T.CustomerID) AS DimCustomersKey
FROM (
  SELECT DISTINCT CustomerID, CustomerName, CustomerEmail
  FROM ordersDWH.trans_sales
) AS T;

INSERT INTO ordersdwh.DimCustomers
SELECT * FROM ordersdwh.view_DimCustomers;
```

The same pattern is repeated for `DimProducts`, `DimRegion`, and `DimDate`.

Final fact table load — joins every dimension to attach surrogate keys to each order:

```sql
SELECT
  F.OrderID, F.Quantity, F.UnitPrice, F.TotalAmount,
  DC.DimCustomersKey, DP.DimProductsKey, DR.DimRegionKey, DD.DimDateKey
FROM ordersDWH.trans_sales F
LEFT JOIN ordersDWH.DimCustomers DC ON F.CustomerID = DC.CustomerID
LEFT JOIN ordersDWH.DimProducts  DP ON F.ProductID  = DP.ProductID
LEFT JOIN ordersDWH.DimRegion    DR ON F.Country    = DR.Country
LEFT JOIN ordersDWH.DimDate      DD ON F.OrderDate  = DD.OrderDate;
```

---

##  Incremental Loading & SCD Type 1

A second module (`sales_new_scd1` database) simulates new orders arriving over time and shows how to merge changes into a dimension table **without reloading everything from scratch**.

1. **New data arrives** with updates to existing products (e.g. a product name change) and brand-new products.
2. A view isolates only the new/changed rows:

```sql
CREATE OR REPLACE VIEW sales_new_scd1.view_DimProducts AS
SELECT DISTINCT ProductID, ProductName, ProductCategory
FROM sales_new_scd1.orders
WHERE OrderDate > '2024-02-10';
```

3. `MERGE INTO` applies **SCD Type 1** logic — matched records get overwritten with the latest values (no history kept), and new records are inserted:

```sql
MERGE INTO sales_new_scd1.DimProducts AS trg
USING sales_new_scd1.view_DimProducts AS src
ON trg.ProductID = src.ProductID
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

This keeps the dimension table current with a single incremental pass instead of a full reload.

---

##  Databases & Tables

| Database | Purpose |
|---|---|
| `ordersDwh` | Main warehouse: staging, transformation, and star schema |
| `sales_new_scd1` | Sandbox for incremental load + SCD Type 1 practice |

---

##  Tech Stack

- **Platform:** Databricks (SQL notebook)
- **Language:** Spark SQL
- **Concepts:** Data warehousing, dimensional modeling (star schema), surrogate keys, incremental loading, SCD Type 1

---

##  How to Run

1. Import the `.dbc` notebook into your Databricks workspace (**Workspace → Import**).
2. Run the cells top to bottom — they create the databases, staging table, transformation view, dimension tables, and fact table in order.
3. Explore the `sales_new_scd1` section separately to see the incremental load / `MERGE` demo.

---

##  What I Learned

Building this project helped me get hands-on with core data warehousing concepts: layered pipelines (staging → transform → core), designing a star schema with fact and dimension tables, generating surrogate keys, and handling incremental updates with SCD Type 1 merges — the fundamentals behind how real-world data warehouses stay both accurate and up to date.

---

##  License

This project is open source and available for learning purposes.
