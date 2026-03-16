# DuckDB vs pandas for data loading

This project includes two versions of the same notebook:

| Notebook | Data loading approach |
|---|---|
| `customer_churn_revenue_risk_segmentation_clv-2.ipynb` | `pd.read_excel()` via openpyxl |
| `customer_churn_duckdb.ipynb` | DuckDB with SQL via `st_read()` |

Both produce identical DataFrames — the rest of the analysis is unchanged.

## What is DuckDB?

DuckDB is an **in-process analytical database**. Think of it as SQLite, but optimised for analytics instead of transactions. It runs inside your Python process — no server installation or configuration needed.

```python
import duckdb

con = duckdb.connect()
df = con.execute("SELECT * FROM st_read('data/customers.xlsx')").fetchdf()
```

## Why would you use it?

### 1. SQL at the source

With DuckDB you can filter, aggregate, and join **before** data enters pandas. This means you only load what you need:

```python
# pandas: loads entire file, then filters
df = pd.read_excel("data/orders.xlsx")
df = df[df["turnover"] > 1000]

# DuckDB: only matching rows leave the file scan
df = con.execute("""
    SELECT * FROM st_read('data/orders.xlsx')
    WHERE turnover > 1000
""").fetchdf()
```

### 2. Performance at scale

| | pandas + openpyxl | DuckDB |
|---|---|---|
| Processing model | Single-threaded, row-based | Multi-threaded, columnar |
| Memory usage | Loads entire file into memory | Streams and filters during scan |
| Sweet spot | Small files (< 50 MB) | Any size, especially 100 MB+ |

For the datasets in this project (~1–2 MB each) the difference is negligible. The benefit becomes clear when working with larger datasets.

### 3. No extra infrastructure

Unlike PostgreSQL or MySQL, DuckDB requires no running database server. Install it with `pip install duckdb` and you're ready to go. This makes it easy to share reproducible notebooks.

## How does it work in our notebook?

The only cells that change are the **imports** and **data loading**:

```python
# Original
import pandas as pd
customers = pd.read_excel("data/Customer_Adres_AccMng.xlsx")

# DuckDB variant
import pandas as pd
import duckdb

con = duckdb.connect()
con.install_extension("spatial")
con.load_extension("spatial")

customers = con.execute("SELECT * FROM st_read('data/Customer_Adres_AccMng.xlsx')").fetchdf()
```

The `spatial` extension gives DuckDB the ability to read Excel files via `st_read()`. The `.fetchdf()` method converts the result to a pandas DataFrame, so everything downstream works exactly the same.

## When to use what?

| Scenario | Recommendation |
|---|---|
| Quick exploration of small files | `pd.read_excel()` is fine |
| Large files or many files | DuckDB handles these more efficiently |
| You need joins/filters before loading | DuckDB lets you use SQL at the source |
| You want a persistent local database | `duckdb.connect("my_data.duckdb")` stores data on disk |
| You need a shared multi-user database | Use PostgreSQL instead |

## Further reading

- [DuckDB documentation](https://duckdb.org/docs/)
- [DuckDB Python client](https://duckdb.org/docs/api/python/overview)
- [Reading Excel files with DuckDB](https://duckdb.org/docs/guides/file_formats/excel_import)
