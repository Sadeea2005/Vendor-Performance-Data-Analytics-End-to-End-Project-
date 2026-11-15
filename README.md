# Vendor Performance Analysis

An end-to-end data analytics project that turns raw retail/wholesale purchase, sales and invoice data into actionable vendor and inventory insights. It covers the full workflow: **CSV ingestion → SQL aggregation → data cleaning → exploratory analysis → statistical testing → Power BI dashboard → written report.**

---

## Table of Contents

- [Business Problem](#business-problem)
- [Project Workflow](#project-workflow)
- [Repository Structure](#repository-structure)
- [Tech Stack](#tech-stack)
- [Data](#data)
- [Getting Started](#getting-started)
- [Methodology](#methodology)
- [Key Findings](#key-findings)
- [Recommendations](#recommendations)
- [Power BI Dashboard](#power-bi-dashboard)
- [Limitations](#limitations)

---

## Business Problem

Effective inventory and sales management are critical for profitability in retail and wholesale. Companies must avoid losses caused by inefficient pricing, poor inventory turnover and over-dependence on a few vendors. This project aims to:

1. Identify underperforming brands that need promotional or pricing adjustments.
2. Determine the top vendors contributing to sales and gross profit.
3. Analyze the impact of bulk purchasing on unit costs.
4. Assess inventory turnover to reduce holding costs and improve efficiency.
5. Investigate the profitability variance between high-performing and low-performing vendors.

---

## Project Workflow

```
Raw CSVs (data/)
      │  ingestion_db.py
      ▼
SQLite database (inventory.db)
      │  get_vendor_summary.py   (SQL joins + cleaning + feature engineering)
      ▼
vendor_sales_summary table  ──►  vendor_sales_summary.csv
      │
      ├──► Exploratory_Data_Analysis.ipynb   (understand tables, build the summary query)
      ├──► Vendor_Performance_Analysis.ipynb (EDA, research questions, hypothesis test)
      ├──► vendor_performance.pbix           (interactive dashboard)
      └──► Vendor_Performance_Report.pdf     (written report)
```

---

## Repository Structure

```
├── data/                                # Raw source CSVs (not tracked, see "Data")
├── logs/                                # Log files written by the scripts
├── ingestion_db.py                      # Loads every CSV in data/ into SQLite
├── get_vendor_summary.py                # Builds and cleans the vendor_sales_summary table
├── Exploratory_Data_Analysis.ipynb      # Database exploration + summary-table design
├── Vendor_Performance_Analysis.ipynb    # Main analysis and statistical testing
├── vendor_sales_summary.csv             # Final aggregated dataset (10,692 rows × 18 columns)
├── vendor_performance.pbix              # Power BI dashboard
├── Vendor_Performance_Report.pdf        # Full written report with charts
└── README.md
```

---

## Tech Stack

| Area | Tools |
|---|---|
| Language | Python 3 |
| Data handling | pandas, NumPy |
| Database | SQLite, SQLAlchemy, SQL (CTEs, joins, aggregations) |
| Visualization | Matplotlib, Seaborn, Power BI |
| Statistics | SciPy (t-confidence intervals, Welch's t-test) |
| Environment | Jupyter Notebook, Python `logging` |

---

## Data

The source data is a set of CSV files, each loaded into SQLite as a table named after its file:

| Table | Description |
|---|---|
| `purchases` | Purchase transactions: vendor, brand, quantity, dollars, purchase price |
| `purchase_prices` | Product-wise actual price and volume per brand |
| `vendor_invoice` | Invoice-level quantity, dollars and **freight** per vendor and PO number |
| `sales` | Sales transactions: quantity, sales dollars, sales price, excise tax |
| `begin_inventory` / `end_inventory` | Start/end-of-year inventory (excluded, not relevant to vendor behavior) |

The raw CSVs are not included in this repository because of their size. Place them in a `data/` folder before running the pipeline.

### Final dataset: `vendor_sales_summary`

One row per **vendor + brand + purchase price** combination (10,692 rows, 128 vendors).

| Column | Description |
|---|---|
| `VendorNumber`, `VendorName` | Vendor identifiers |
| `Brand`, `Description` | Brand ID and product name |
| `PurchasePrice`, `ActualPrice`, `Volume` | Unit cost, retail price, bottle volume |
| `TotalPurchaseQuantity`, `TotalPurchaseDollars` | Aggregated purchases |
| `TotalSalesQuantity`, `TotalSalesDollars`, `TotalSalesPrice`, `TotalExciseTax` | Aggregated sales |
| `FreightCost` | Total freight per vendor |
| `GrossProfit` | `TotalSalesDollars − TotalPurchaseDollars` |
| `ProfitMargin` | `GrossProfit / TotalSalesDollars × 100` |
| `StockTurnover` | `TotalSalesQuantity / TotalPurchaseQuantity` |
| `SalesToPurchaseRatio` | `TotalSalesDollars / TotalPurchaseDollars` |

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/<your-repo-name>.git
cd <your-repo-name>
```

### 2. Install dependencies

```bash
pip install pandas numpy matplotlib seaborn scipy sqlalchemy jupyter
```

### 3. Prepare folders

```bash
mkdir -p data logs
# copy the raw CSV files into data/
```

> The `logs/` folder must exist before running the scripts, since they write their log files there.

### 4. Run the pipeline

```bash
# Step 1: load all CSVs from data/ into inventory.db
python ingestion_db.py

# Step 2: build, clean and store the vendor_sales_summary table
python get_vendor_summary.py
```

### 5. Explore the analysis

```bash
jupyter notebook
```

Open `Exploratory_Data_Analysis.ipynb` first, then `Vendor_Performance_Analysis.ipynb`. To work from the final dataset without rebuilding the database, load `vendor_sales_summary.csv` directly.

---

## Methodology

### 1. Ingestion (`ingestion_db.py`)
Reads every `.csv` in `data/`, writes each to SQLite using the filename as the table name, and logs progress and total run time.

### 2. Aggregation and cleaning (`get_vendor_summary.py`)
Uses three CTEs (`FreightSummary`, `PurchaseSummary`, `SalesSummary`) joined into a single vendor-brand summary, so heavy joins run once instead of on every analysis or dashboard refresh. It then:
- converts `Volume` to float,
- fills missing values (unsold products) with 0,
- strips whitespace from `VendorName` and `Description`,
- engineers `GrossProfit`, `ProfitMargin`, `StockTurnover` and `SalesToPurchaseRatio`.

### 3. Exploratory data analysis
Summary statistics, distributions, boxplots and a correlation heatmap. Records were filtered to remove inconsistencies before the analysis:
- `GrossProfit > 0`
- `ProfitMargin > 0`
- `TotalSalesQuantity > 0`

### 4. Analysis and statistical testing
- Quantile-based segmentation (bottom 15% sales / top 15% margin) to flag brands for promotion or pricing changes.
- Pareto and donut charts for vendor purchase concentration.
- Tercile binning (`pd.qcut`) of purchase quantity to measure bulk-purchase unit-cost savings.
- Unsold inventory value: `(PurchaseQty − SalesQty) × PurchasePrice`.
- 95% confidence intervals and a Welch two-sample t-test comparing profit margins of top-quartile and bottom-quartile vendors by sales.

---

## Key Findings

| # | Question | Finding |
|---|---|---|
| 1 | Which brands need promotion or pricing changes? | **198 brands** combine low sales with high profit margins, so there is room to grow volume without hurting profitability. |
| 2 | How concentrated is purchasing? | The **top 10 vendors account for 65.69%** of total purchases (Diageo North America alone is 16.3%); the rest contribute 34.31%. This is a supplier-dependency risk. |
| 3 | Does bulk buying cut unit cost? | Yes. Average unit purchase price: **Small $39.06, Medium $15.49, Large $10.78**, roughly a 72% reduction from small to large orders. |
| 4 | How much capital is tied up in unsold stock? | **$2.71M** in unsold inventory. The largest contributors are Diageo North America ($722K), Jim Beam Brands ($555K) and Pernod Ricard USA ($471K). |
| 5 | Do top and low vendors differ in margin? | Low-sales vendors earn higher margins (mean **41.55%**, 95% CI 40.48–42.62) than top-sales vendors (mean **31.17%**, 95% CI 30.74–31.61). |
| 6 | Is that difference statistically significant? | Yes. The null hypothesis is rejected at the 5% level: the two groups operate under different profitability models. |

**Other observations from EDA**
- Purchase quantity and sales quantity are almost perfectly correlated (0.999).
- Purchase price has almost no correlation with sales dollars or gross profit.
- Faster stock turnover does not translate into higher profitability (weak negative correlation with gross profit and margin).
- Some products were purchased but never sold, and some records show negative gross profit (minimum around −$52K).

Full charts and tables are in [`Vendor_Performance_Report.pdf`](Vendor_Performance_Report.pdf).

---

## Recommendations

- **Re-price low-sales, high-margin brands** to lift volume without sacrificing profitability.
- **Diversify vendor partnerships** to reduce dependency on a few suppliers.
- **Leverage bulk-purchase discounts** while managing inventory levels.
- **Optimize slow-moving inventory** by adjusting purchase quantities, running clearance sales or revising storage strategy.
- **Improve marketing and distribution** for low-performing vendors to raise sales volume while protecting margins.

---

## Power BI Dashboard

`vendor_performance.pbix` is a single-page **Vendor Performance Dashboard** built on `vendor_sales_summary`. It includes:

- KPI cards
- Purchase contribution % (donut chart)
- Top vendors by sales (bar chart)
- Top brands by sales (bar chart)
- Low-performing vendors (funnel chart)
- Low-performing brands (scatter chart)

Open it with [Power BI Desktop](https://powerbi.microsoft.com/desktop/) (Windows).

---

## Limitations

- Results are descriptive and based on a single period of data; correlations do not imply causation.
- Records with zero sales, non-positive gross profit or non-positive margin were excluded from the main analysis, so the findings describe profitable, sold products only.
- Freight cost is aggregated at the vendor level and repeated on each vendor-brand row, so summing it across rows would overstate total freight.
- Top vs. low vendors are defined by sales quartiles at the vendor-brand row level.
