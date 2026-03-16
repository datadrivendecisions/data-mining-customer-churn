# DuckDB vs pandas for data loading

This project includes two versions of the same notebook:

| Notebook | Data loading approach |
|---|---|
| `customer_churn_revenue_risk_segmentation_clv-2.ipynb` | `pd.read_excel()` via openpyxl |
| `customer_churn_duckdb.ipynb` | DuckDB with SQL via `st_read()` |

Both produce identical DataFrames. The DuckDB variant also includes two additional sections not present in the original: **feature standardisation** (section 6.1b) and **feature importance analysis** (section 7.1).

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

## Feature standardisation (DuckDB notebook only)

The three features used for modelling (`total_revenue`, `active_months`, `avg_monthly_revenue`) live on very different numeric scales. `total_revenue` can range into the hundreds of thousands, while `active_months` only goes from 1 to about 28. This is a problem for distance-based models like kNN: the feature with the largest range dominates the distance calculation, effectively making the other features invisible.

The DuckDB notebook applies `StandardScaler` (section 6.1b) to transform each feature to mean = 0 and standard deviation = 1 before training. This ensures that all features contribute proportionally. For Logistic Regression, it also makes the coefficients directly comparable — you can read the size of a coefficient as the strength of that feature's influence.

```python
from sklearn.preprocessing import StandardScaler

scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)  # fit on training data only
X_test = scaler.transform(X_test)        # apply same transformation to test data
```

The original notebook does *not* include this step — comparing the two gives a clear demonstration of why preprocessing matters.

## Feature importance: what drives churn? (DuckDB notebook only)

The DuckDB notebook includes section 7.1 that investigates which features most strongly influence the churn prediction. Two methods are used:

### Logistic Regression coefficients

Logistic Regression assigns a **coefficient** (weight) to each input feature. Because the features are standardised, the coefficients are directly comparable:

- **Sign** — negative means higher values reduce churn probability, positive means they increase it
- **Magnitude** — larger absolute value means stronger influence

```python
coefs = pd.Series(log_model.coef_[0], index=feature_names)
```

This only works for models that produce coefficients (linear/logistic regression). It does not work for kNN or tree-based models.

### Permutation importance

Permutation importance is **model-agnostic** — it works with any model. The idea is simple:

1. Measure the model's accuracy on the test set
2. Randomly shuffle one feature's values — this breaks the real relationship between that feature and churn
3. Measure accuracy again — the drop tells you how important that feature was
4. Repeat for each feature

```python
from sklearn.inspection import permutation_importance

perm = permutation_importance(knn_model, X_test, y_test, n_repeats=30, random_state=42)
```

A large accuracy drop means the model relies heavily on that feature. This method is especially useful for kNN, which has no built-in feature importance.

### Key finding

The analysis reveals that **`active_months`** is the strongest predictor of churn. Customers who purchase consistently over many months are significantly less likely to churn, while customers active in only one or two months are at the highest risk.

This leads to three actionable insights:

1. **Early warning** — monitor purchasing frequency; a customer whose order pattern starts to thin out should be flagged before they disappear entirely
2. **Retention focus** — invest in consistent engagement (regular check-ins, repeat-order incentives, loyalty programs) rather than focusing solely on high-revenue accounts
3. **Onboarding** — customers who only purchase once or twice are at the highest risk; a structured follow-up process after the first order could reduce early churn

The takeaway: **it's not how much a customer spends that predicts churn, but how consistently they keep coming back.**

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
