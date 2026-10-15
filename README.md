# Zepto E-Commerce & Inventory Analysis

End-to-end analysis of a Zepto quick-commerce product catalogue. The project uses **PostgreSQL** for data cleaning and business-question analysis, and **Power BI** for an interactive inventory dashboard.

![Dashboard preview](images/dashboard.png)
<!-- TODO: add a screenshot of your Power BI page at images/dashboard.png -->

---

## Table of contents
- [Project overview](#project-overview)
- [Key findings](#key-findings)
- [Dataset](#dataset)
- [Tech stack](#tech-stack)
- [Repository structure](#repository-structure)
- [How to reproduce](#how-to-reproduce)
- [SQL analysis](#sql-analysis)
- [Power BI dashboard](#power-bi-dashboard)
- [Data quality notes](#data-quality-notes)
- [Recommendations](#recommendations)
- [Future work](#future-work)
- [Author](#author)

---

## Project overview
Quick-commerce platforms live or die by pricing and stock availability. This project answers questions such as:

- Which products and categories carry the deepest discounts?
- Where are stock-outs concentrated, and are high-value items affected?
- How much inventory value and weight sits in each category?
- Which products give the best price per gram?

The full write-up is in [`PROJECT_REPORT.md`](PROJECT_REPORT.md).

## Key findings
| Metric | Result |
|---|---|
| Rows after cleaning | **3,731** (1 row with MRP = 0 removed) |
| Categories | **14** |
| Out-of-stock rate | **12.1%** (453 of 3,731 listings) |
| Average MRP / selling price | **₹156.84 / ₹141.97** |
| Average discount | **7.6%** (31.5% of listings have no discount at all) |
| Highest average discount | Fruits & Vegetables (**15.5%**), then Meats, Fish & Eggs (**11.0%**) |
| Highest out-of-stock rate | Biscuits (**28.6%**), then Dairy, Bread & Batter and Beverages (**21.7%**) |
| Out-of-stock items are cheaper | Avg MRP ₹92.8 vs ₹165.7 for in-stock items |
| Weight mix | 90.9% of items under 1 kg, 7.9% medium, 1.2% bulk |

> **Important:** the raw file contains only **1,800 distinct products** once category is ignored; many products are listed under several categories. See [Data quality notes](#data-quality-notes).

## Dataset
File: `data/zepto_v2.csv` (UTF-8, comma separated, 3,732 data rows, 9 columns)

| Column | Description |
|---|---|
| `Category` | Product category (14 distinct) |
| `name` | Product name |
| `mrp` | Maximum retail price, **in paise** (converted to rupees in the SQL cleaning step) |
| `discountPercent` | Discount on MRP, in percent |
| `availableQuantity` | Units available in stock (0 to 6) |
| `discountedSellingPrice` | Selling price after discount, **in paise** |
| `weightInGms` | Pack weight in grams |
| `outOfStock` | `TRUE` if the item is out of stock |
| `quantity` | Pack quantity / size attribute of the listing |

## Tech stack
- **PostgreSQL**: data cleaning and analysis (`E-Commerce_Inventory_Data_Analysis.sql`)
- **Power BI Desktop**: dashboard (`Zepto_E-_Commerce_And_Inventory_Analysis_Dashboard.pbix`)
- **Python / pandas** (optional): used to cross-check the SQL results

## Repository structure
```
.
├── README.md
├── PROJECT_REPORT.md
├── data/
│   └── zepto_v2.csv
├── sql/
│   └── E-Commerce_Inventory_Data_Analysis.sql
├── powerbi/
│   └── Zepto_E-_Commerce_And_Inventory_Analysis_Dashboard.pbix
└── images/
    └── dashboard.png
```

## How to reproduce
1. **Create the table and run the setup section** of `sql/E-Commerce_Inventory_Data_Analysis.sql` (the `drop table` / `create table` block).
2. **Load the CSV** from `psql`:
   ```sql
   \copy zepto(category,name,mrp,discountPercent,availableQuantity,discountedSellingPrice,weightInGms,outOfStock,quantity)
   FROM 'data/zepto_v2.csv' WITH (FORMAT csv, HEADER true, ENCODING 'UTF8');
   ```
3. **Run the cleaning steps**: delete rows with `mrp = 0`, then convert paise to rupees.
   > Run the paise-to-rupees `UPDATE` **only once**; running it again divides prices by 100 a second time.
4. **Run the business questions** (Q1 to Q8) in the same script.
5. **Open the dashboard**: open the `.pbix` in Power BI Desktop, then *Transform data → Data source settings* and point it at your PostgreSQL server and the `public.zepto` table. Refresh.

## SQL analysis
| # | Question | Technique |
|---|---|---|
| Q1 | Top 10 products by discount percentage | `ORDER BY`, `LIMIT`, `DISTINCT` |
| Q2 | High-MRP (> ₹300) products that are out of stock | filtering |
| Q3 | Estimated inventory value per category | `SUM(price × quantity)`, `GROUP BY` |
| Q4 | MRP > ₹500 with discount < 10% | multi-condition filter |
| Q5 | Top 5 categories by average discount | `AVG`, `ROUND` |
| Q6 | Price per gram for products ≥ 100 g | calculated column |
| Q7 | Weight buckets: low / medium / bulk | `CASE WHEN` |
| Q8 | Total inventory weight per category | `SUM(weight × quantity)` |

Data exploration covers row counts, null checks, distinct categories, in-stock vs out-of-stock counts and duplicate product names.

## Power BI dashboard
A single dark-themed page titled **Zepto E-Commerce and Inventory Analysis Dashboard**.

- **KPI cards:** Total Products, Out of Stock Products, Out of Stock Rate, Average Price
- **Out of stock products by category** (clustered bar)
- **Top products by quantity** (bar chart)
- **Category share of quantity** (donut chart)
- **Quantity vs product count** (clustered column chart)
- **Stock status by category matrix** with Total Products and Avg Price (pivot table)
- **Product detail table** with category, name, quantity and stock status
- **Slicers:** category list and an in-stock / out-of-stock slicer

<!-- TODO: paste your DAX measures here, e.g. Total Products, Out of Stock Products, Out of Stock Rate, Avg Price -->

## Data quality notes
- **Cross-category duplication.** 3,731 rows reduce to 1,800 unique products when `Category` is ignored; 1,281 of those appear under more than one category (up to 7). Several categories are exact copies of each other (for example *Munchies* and *Cooking Essentials*), so category totals in Q3 and Q8 are repeated. Treat category-level revenue and weight as indicative.
- **"Revenue" is potential value**, calculated from selling price × available stock, not actual sales.
- **Q6 precision.** Rounding to 2 decimals makes many cheap items tie at ₹0.02 per gram. Use price per kg or 4 decimals for finer ranking.
- **Stock fields are consistent:** no item shows zero stock while marked in stock, or positive stock while marked out of stock.

## Recommendations
1. Restock **Biscuits, Dairy and Beverages** first, since they have the highest stock-out rates.
2. Review **high-MRP stock-outs** such as Patanjali Cow's Ghee and Aashirvaad Atta, which are costly for customers to miss.
3. Extend promotions to the **31.5% of listings with no discount**, especially premium oil and ghee jars priced over ₹500.
4. De-duplicate the product feed before building category-level forecasts.

## Future work
- Add sales or order data to replace "potential revenue" with real revenue
- Track stock status over time to measure how long items stay out of stock
- Build a de-duplicated product dimension table for a star-schema model
- Publish the dashboard to Power BI Service and link it here

## Author
**[Your Name]** · [LinkedIn](https://www.linkedin.com/) · [Email](mailto:you@example.com)

*Dataset source: [add the Kaggle / original source link here]*
