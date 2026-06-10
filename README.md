# 💰 Greenweez Finance — JOIN Analysis

BigQuery SQL project joining sales, product, and shipping data to calculate **operational margin** for **Greenweez**, a French organic e-commerce brand.

---

## 🎯 Objective

Greenweez's finance team needed a unified view of order profitability. This project answers:

- What is the gross margin per order after product costs?
- What is the net operational margin after shipping and logistics costs?
- How does profitability trend day by day?

---

## 🗃️ Dataset

Three tables from BigQuery (`course16` dataset):

| Table | Rows | Description |
|-------|------|-------------|
| `gwz_sales` | 1,375,630 | One row per product per order — turnover and quantity |
| `gwz_product` | — | Product catalog with purchase prices |
| `gwz_ship` | — | Per-order shipping fee and logistics costs |

Key columns in `gwz_sales`:

| Column | Description |
|--------|-------------|
| `date_date` | Sale date |
| `orders_id` | Order identifier |
| `products_id` | Product identifier |
| `turnover` | Revenue generated (€) |
| `qty` | Quantity sold |

Key columns in `gwz_product`:

| Column | Description |
|--------|-------------|
| `products_id` | Product identifier (joins to `gwz_sales`) |
| `purchase_price` | Unit purchase cost (€) |

Key columns in `gwz_ship`:

| Column | Description |
|--------|-------------|
| `orders_id` | Order identifier (joins to `gwz_orders`) |
| `shipping_fee` | Shipping revenue charged to customer (€) |
| `log_cost` | Logistics cost (€) |
| `ship_cost` | Shipping cost (€) |

---

## 🔗 Table Pipeline

```
gwz_sales + gwz_product
        ↓ LEFT JOIN on products_id
  gwz_sales_margin       ← product level, 1,375,630 rows
        ↓ GROUP BY orders_id
    gwz_orders           ← order level, 167,551 rows
        ↓ LEFT JOIN on orders_id
gwz_orders_operational   ← + shipping data + operational_margin
        ↓ GROUP BY date_date
      gwz_daily          ← daily summary, 183 rows
```

---

## 📋 Project Steps

### 1. Data Exploration
Previewed all three tables to understand structure, volume, and column types.

### 2. First JOIN — Sales + Product → `gwz_sales_margin`
Added `purchase_price` from `gwz_product` to each sales row via LEFT JOIN on `products_id`. Calculated two new columns:

| Column | Formula | Description |
|--------|---------|-------------|
| `purchase_cost` | `qty × purchase_price` | Total product cost per row |
| `margin` | `turnover − purchase_cost` | Gross margin per row |

### 3. Data Quality Checks
After every JOIN and table creation, three tests were applied:

- **PK test** — `GROUP BY + HAVING COUNT(*) >= 2` to detect duplicates
- **NULL test** — `WHERE column IS NULL` to detect unmatched JOIN rows
- **Row count check** — `COUNT(*)` to confirm no rows were lost

> All tests passed clean ✅ — no duplicates, no NULLs, no row loss.
> **RIGHT JOIN insight:** RIGHT JOIN returned 2 more rows than LEFT JOIN — meaning 2 products in `gwz_product` have never been sold.

### 4. Aggregation — `gwz_sales_margin` → `gwz_orders`
Grouped by `orders_id` to collapse 1,375,630 product-level rows into 167,551 order-level rows using `SUM()`.

### 5. Second JOIN — Orders + Ship → `gwz_orders_operational`
Added shipping data from `gwz_ship` via LEFT JOIN on `orders_id`. Calculated the final profitability metric:

| Column | Formula | Description |
|--------|---------|-------------|
| `operational_margin` | `margin + shipping_fee − ship_cost − log_cost` | Net margin after all costs |

### 6. Daily Aggregation → `gwz_daily`
Grouped by `date_date` to produce a 183-row daily summary of all financial metrics.

---

## 📊 Key Findings

- **Data integrity:** All PK, NULL, and row count tests passed at every step — clean data throughout
- **RIGHT JOIN revealed** 2 products in the catalog that have never been sold
- **Peak revenue days:** Late September 2021 — daily turnover exceeded €100K on multiple days
- **Operational margin** is consistently lower than gross margin — shipping and logistics costs have a meaningful impact on net profitability

---

## 🛠️ SQL Techniques Used

| Technique | Purpose |
|-----------|---------|
| `LEFT JOIN` | Joining tables without losing rows |
| `RIGHT JOIN` | Detecting unmatched records in the right table |
| `CREATE OR REPLACE TABLE` | Saving intermediate and final tables |
| `GROUP BY` + `SUM()` | Aggregating from product to order to daily level |
| `ROUND()` | Formatting numeric outputs |
| `COUNT(*) + HAVING` | Primary key and duplicate validation |
| `IS NULL` | Detecting unmatched JOIN rows |
| `ORDER BY` | Sorting results by date |

---

## 🗂️ Output Tables

```
course16/
├── gwz_sales_margin          ← sales enriched with purchase_price, purchase_cost, margin
├── gwz_sales_margin_pk       ← PK validation (empty = no duplicates)
├── gwz_orders                ← order-level aggregation
├── gwz_orders_pk             ← PK validation (empty = no duplicates)
├── gwz_orders_operational    ← orders + shipping data + operational_margin
└── gwz_daily                 ← daily financial summary (183 days)
```

---

## 🔧 Tools

- **Google BigQuery** — SQL engine and data warehouse
- **SQL** — All analysis done in standard SQL with BigQuery-specific functions
