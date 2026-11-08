# CityCare Pharmacy: Sales, Margin and Expiry Analysis (PostgreSQL)
 
# Introduction
📊 Dive into the numbers of a pharmacy chain! Using raw point-of-sale and purchasing exports from six branches, this project cleans the data and explores 💰 which categories earn the margin, 📉 how a July price rise squeezed profit, 🩺 which chronic patients are leaving, 🚚 which suppliers charge more, and 🗑️ why medicine expires on the shelf.
 
🔍 SQL queries? Check them out here: [SQL_LOADING](/sql/)
 
# Background
CityCare Pharmacy has six branches in Lahore, Islamabad and Karachi. The owner has twelve months (2025) of raw exports from the point-of-sale and purchasing systems and wants to know where the chain makes and loses money. This project turns those messy exports into a clean PostgreSQL database and answers the owner's questions with SQL.
 
> **About the data:** this is a **synthetic dataset**, built to look like a real pharmacy POS and purchasing export, including the usual problems of such systems (an older POS in one branch, spelling variants, duplicates, missing prices). It does not describe a real company. The goal of the project is the analysis method.
 
**2025 at a glance:** 33,738 invoices, Rs 42.4 million revenue, Rs 7.74 million gross profit, 18.2% gross margin.
 
### The questions I wanted to answer through my SQL queries were:
 
1. Which categories earn the most margin, and is prescription or over-the-counter better?
2. Did the July price increase cover higher costs?
3. Are we losing chronic patients?
4. Which suppliers charge more for the same products?
5. Do large orders cause the expiry loss?
# Tools I Used
For my deep dive into the pharmacy data, I used these tools:
 
- **SQL:** The backbone of the project: cleaning, checking and analysing the data.
- **PostgreSQL:** The database system that holds the raw, clean and analysis tables.
- **Git & GitHub:** Version control and sharing the scripts, report and findings.
# Data Cleaning: Raw Tables vs Clean Tables
The six CSV files were first loaded exactly as they came (all columns as text) into `stg_` staging tables, so nothing could fail and the raw data was never changed. I then checked them, and loaded clean, typed tables with primary keys, foreign keys and checks. Every row that disappeared has a reason.
 
| Table | Raw rows | Clean rows | What changed |
|---|---:|---:|---|
| branches | 6 | 6 | Loaded as is |
| suppliers | 8 | 8 | Loaded as is |
| products | 86 | 86 | Rx flag (8 spellings) converted to true / false |
| purchases | 3,690 | 3,678 | 12 duplicates removed. 6 zero costs set to NULL. 14 impossible expiry dates set to NULL and flagged |
| sales | 70,107 | 69,898 | 209 duplicates removed. 139 missing prices filled with the median price of the product in that month. Branch 4 dates and discounts converted. 14 payment spellings mapped to 5 |
| stock_adjustments | 223 | 223 | Reasons mapped to 4 standard values. Negative quantities made positive |
| product_month_cost *(new)* | - | 1,187 | Average purchase cost of each product per month, used to calculate profit |
| product_month_price *(new)* | - | 1,032 | Median selling price of each product per month, used to fill missing prices |
| sales_detail *(new view)* | - | 69,898 | One row per sale line with revenue and gross profit |
 
Main problems found and fixed: exact duplicates, missing prices, one branch (F-10 Markaz) running an older POS with `DD/MM/YYYY` dates and discounts stored as fractions, payment methods written in many ways, zero costs, expiry dates before purchase dates, and customer returns recorded as negative quantities (kept, because they correctly reduce revenue).
 
# The Analysis
Each query aimed at one business question. Revenue = quantity x price x (1 - discount). Gross profit = revenue - quantity x cost. Here is how I approached each one:
 
### 1. Which Categories Earn the Most Margin?
To see where the profit really comes from, I grouped the sales by category and compared revenue with gross profit and margin.
 
```sql
SELECT category,
       SUM(quantity)                                                      AS packs,
       ROUND(SUM(revenue))                                                AS revenue,
       ROUND(100.0 * SUM(revenue) / SUM(SUM(revenue)) OVER (), 1)         AS revenue_share_pct,
       ROUND(SUM(gross_profit))                                           AS gross_profit,
       ROUND(100.0 * SUM(gross_profit) / SUM(revenue), 1)                 AS margin_pct
FROM sales_detail
GROUP BY category
ORDER BY gross_profit DESC;
```
Here's the breakdown:
- **Revenue and profit rank categories differently.** Diabetes is the biggest category by revenue (Rs 7.9 million, 18.6%) but earns only an 11.6% margin.
- **Care products carry the margin.** Vitamins, Skin Care and Personal Care earn 27% to 30% margins: together 30% of revenue but 46% of the profit.
- **Prescription versus over-the-counter:** prescription medicines are 46.4% of revenue at a 14.4% margin, while over-the-counter products are 53.6% of revenue at a 21.6% margin.
| Category | Revenue (Rs) | Revenue share % | Gross profit (Rs) | Margin % |
| --- | ---: | ---: | ---: | ---: |
| Vitamins & Supplements | 4,792,617 | 11.3 | 1,276,913 | 26.6 |
| Skin Care | 4,090,703 | 9.6 | 1,214,596 | 29.7 |
| Personal Care & Devices | 3,859,841 | 9.1 | 1,075,855 | 27.9 |
| Diabetes | 7,900,726 | 18.6 | 918,159 | 11.6 |
| Baby Care | 5,763,575 | 13.6 | 776,810 | 13.5 |
| Cardiac & BP | 5,462,519 | 12.9 | 590,728 | 10.8 |
| Antibiotics | 4,006,397 | 9.4 | 586,243 | 14.6 |
| Gastro | 2,613,304 | 6.2 | 530,773 | 20.3 |
| Allergy & Cold | 2,378,663 | 5.6 | 475,859 | 20.0 |
| Pain & Fever | 1,558,085 | 3.7 | 290,279 | 18.6 |
 
*Table of revenue, profit and margin by category*
 
### 2. Did the July Price Increase Cover Higher Costs?
In July 2025 retail prices of regulated medicines rose. To check whether they rose as much as my purchase costs, I compared the same products before (January to June) and after (July to December), so a change in product mix could not fool the result.
 
```sql
WITH pp AS (
    SELECT product_id, category,
           CASE WHEN sale_month < DATE '2025-07-01' THEN 'H1' ELSE 'H2' END AS half,
           SUM(quantity * unit_price) / SUM(quantity)                       AS avg_price,
           SUM(quantity * unit_cost)  / SUM(quantity)                       AS avg_cost,
           SUM(quantity)                                                    AS units
    FROM sales_detail
    WHERE quantity > 0
    GROUP BY product_id, category, 3
),
pv AS (
    SELECT product_id, category,
           MAX(avg_price) FILTER (WHERE half = 'H1') AS p1,
           MAX(avg_price) FILTER (WHERE half = 'H2') AS p2,
           MAX(avg_cost)  FILTER (WHERE half = 'H1') AS c1,
           MAX(avg_cost)  FILTER (WHERE half = 'H2') AS c2,
           MAX(units)     FILTER (WHERE half = 'H1') AS u1,
           MAX(units)     FILTER (WHERE half = 'H2') AS u2
    FROM pp
    GROUP BY product_id, category
)
SELECT category,
       ROUND(AVG(100.0 * (p2 - p1) / p1), 1)                       AS price_change_pct,
       ROUND(AVG(100.0 * (c2 - c1) / c1), 1)                       AS cost_change_pct,
       ROUND(AVG(100.0 * (p2 - c2) / p2 - 100.0 * (p1 - c1) / p1), 1) AS margin_change_points,
       ROUND(100.0 * (SUM(u2) - SUM(u1)) / SUM(u1), 1)             AS units_change_pct
FROM pv
WHERE p1 IS NOT NULL AND p2 IS NOT NULL AND c1 IS NOT NULL AND c2 IS NOT NULL
GROUP BY category
ORDER BY margin_change_points;
```
Here's the breakdown:
- **Costs rose about twice as fast as prices.** Antibiotics: price +8.0% but cost +15.4%, a margin loss of 5.5 points. Cardiac & BP lost 5.3 points and Diabetes 3.3 points.
- **Gross margin fell** from 19.5% in January to June to 16.6% in the last quarter.
*Bar chart comparing the change in selling price and purchase cost by category after July 2025; made from the query results*
 
| Category | Price change % | Cost change % | Margin change (points) | Units change % |
| --- | ---: | ---: | ---: | ---: |
| Antibiotics | 8.0 | 15.4 | -5.5 | 3.3 |
| Cardiac & BP | 7.0 | 13.8 | -5.3 | 3.4 |
| Diabetes | 7.0 | 11.1 | -3.3 | 2.2 |
| Pain & Fever | 2.9 | 6.1 | -2.4 | 26.5 |
| Gastro | 4.0 | 6.4 | -1.8 | 1.7 |
| Skin Care | 2.0 | 4.1 | -1.4 | 6.0 |
| Allergy & Cold | 2.0 | 3.5 | -1.1 | -4.8 |
| Vitamins & Supplements | 2.0 | 3.0 | -0.7 | -0.9 |
| Baby Care | 2.0 | 2.5 | -0.4 | 6.7 |
| Personal Care & Devices | 0.0 | 0.1 | 0.0 | 9.5 |
 
*Table of price, cost, margin and unit changes by category*
 
### 3. Are We Losing Chronic Patients?
Patients on diabetes or heart medicine buy every month, so each one is worth a lot. I used loyalty cards to find regular patients (at least 5 different months of purchases) and checked who has not bought since 1 November.
 
```sql
WITH c AS (
    SELECT loyalty_card_id,
           COUNT(DISTINCT sale_month)                        AS months_bought,
           MAX(sale_date)                                    AS last_purchase,
           SUM(revenue) / COUNT(DISTINCT sale_month)         AS avg_monthly_spend
    FROM sales_detail
    WHERE category IN ('Diabetes', 'Cardiac & BP')
      AND loyalty_card_id IS NOT NULL
      AND quantity > 0
    GROUP BY loyalty_card_id
    HAVING COUNT(DISTINCT sale_month) >= 5
)
SELECT COUNT(*)                                                       AS chronic_patients,
       COUNT(*) FILTER (WHERE last_purchase < DATE '2025-11-01')      AS lapsed_patients,
       ROUND(100.0 * COUNT(*) FILTER (WHERE last_purchase < DATE '2025-11-01') / COUNT(*), 1)
                                                                      AS lapsed_pct,
       ROUND(SUM(avg_monthly_spend))                                  AS monthly_spend_all,
       ROUND(SUM(avg_monthly_spend) FILTER (WHERE last_purchase < DATE '2025-11-01'))
                                                                      AS monthly_spend_lost
FROM c;
```
Here's the breakdown:
- **183 regular chronic patients** were found, and **40 of them (21.9%)** have stopped buying.
- They used to spend about **Rs 75,300 a month**, roughly **Rs 0.9 million a year**.
- A pharmacist call list for these 40 patients is cheap, and winning back half of them recovers about Rs 0.45 million a year.
| Chronic patients | Lapsed patients | Lapsed % | Monthly spend of all (Rs) | Monthly spend lost (Rs) |
| --- | --- | --- | --- | --- |
| 183 | 40 | 21.9 | 387,567 | 75,313 |
 
*Table of chronic patients who stopped buying*
 
### 4. Which Suppliers Charge More for the Same Products?
For each product I found the cheapest average price among suppliers, then measured how much extra each supplier charged compared with it.
 
```sql
WITH sp AS (
    SELECT p.supplier_id, p.product_id,
           SUM(p.quantity_packs * p.unit_cost) / SUM(p.quantity_packs) AS avg_cost,
           SUM(p.quantity_packs)                                       AS packs
    FROM purchases p
    WHERE p.unit_cost IS NOT NULL AND p.supplier_id IS NOT NULL
    GROUP BY p.supplier_id, p.product_id
),
best AS (
    SELECT *, MIN(avg_cost) OVER (PARTITION BY product_id) AS best_cost
    FROM sp
)
SELECT s.supplier_name,
       COUNT(*)                                                      AS products,
       ROUND(SUM(b.packs * b.avg_cost))                              AS spend,
       ROUND(SUM(b.packs * (b.avg_cost - b.best_cost)))              AS extra_vs_cheapest,
       ROUND(100.0 * SUM(b.packs * (b.avg_cost - b.best_cost))
             / SUM(b.packs * b.avg_cost), 1)                         AS extra_pct
FROM best b
JOIN suppliers s ON s.supplier_id = b.supplier_id
GROUP BY s.supplier_name
ORDER BY extra_vs_cheapest DESC;
```
Here's the breakdown:
- **Two suppliers are about 8% dearer** for the same products: United Medicos (+7.8%, Rs 482,352 extra) and Zam Zam Distributors (+8.1%, Rs 356,328 extra).
- Together they account for about **Rs 0.84 million of the Rs 1.32 million** extra paid across all suppliers.
- Cheaper batches can also be near-expiry deals, so I treat this as a list of products to renegotiate, not an automatic switch.
| Supplier | Products | Spend (Rs) | Extra vs cheapest (Rs) | Extra % |
| --- | ---: | ---: | ---: | ---: |
| United Medicos | 35 | 6,153,478 | 482,352 | 7.8 |
| Zam Zam Distributors | 28 | 4,404,117 | 356,328 | 8.1 |
| Punjab Medical Store (Pvt) Ltd | 29 | 5,583,754 | 129,835 | 2.3 |
| NorthStar Pharma | 38 | 6,779,298 | 91,688 | 1.4 |
| Prime Care Wholesale | 32 | 5,757,216 | 84,508 | 1.5 |
| Medline Traders | 35 | 7,680,466 | 80,104 | 1.0 |
| Alpha Pharma Distributors | 30 | 4,579,277 | 55,050 | 1.2 |
| Karachi Health Supplies | 38 | 6,319,416 | 40,309 | 0.6 |
 
*Table of extra cost paid compared with the cheapest supplier*
 
### 5. Do Large Orders Cause the Expiry Loss?
Medicines worth **Rs 1.09 million** expired in 2025 (3.0% of everything bought). To find the cause, I measured how many months of sales each product's average order covers, and compared that with the expiry loss.
 
```sql
WITH monthly AS (
    SELECT branch_id, product_id,
           SUM(quantity) FILTER (WHERE quantity > 0) / 12.0 AS packs_per_month
    FROM sales
    GROUP BY branch_id, product_id
),
cover AS (
    SELECT p.product_id, AVG(p.quantity_packs / m.packs_per_month) AS months_per_order
    FROM purchases p
    JOIN monthly m ON m.branch_id = p.branch_id AND m.product_id = p.product_id
    WHERE p.purchase_date >= DATE '2025-01-01'
    GROUP BY p.product_id
),
lost AS (
    SELECT a.product_id, SUM(a.quantity * p.unit_cost) AS expired_cost
    FROM stock_adjustments a
    JOIN purchases p ON p.branch_id = a.branch_id AND p.product_id = a.product_id
                    AND p.batch_no = a.batch_no
    WHERE a.reason = 'Expired' AND p.unit_cost IS NOT NULL
    GROUP BY a.product_id
)
SELECT CASE WHEN c.months_per_order > 3.5 THEN 'Orders cover more than 3.5 months'
            ELSE 'Orders cover 3.5 months or less' END                   AS order_size_group,
       COUNT(*)                                                           AS products,
       ROUND(SUM(COALESCE(l.expired_cost, 0)))                            AS expired_cost,
       ROUND(100.0 * SUM(COALESCE(l.expired_cost, 0))
             / SUM(SUM(COALESCE(l.expired_cost, 0))) OVER (), 1)          AS pct_of_loss
FROM cover c
LEFT JOIN lost l ON l.product_id = c.product_id
GROUP BY 1
ORDER BY expired_cost DESC;
```
Here's the breakdown:
- **12 products (14% of the range) cause 67.9% of the expiry loss.** Their orders cover more than 3.5 months of sales.
- The worst are slow, expensive products such as formula milk, insulin and newer diabetes tablets, ordered 6 to 7 months at a time.
- The loss comes from **order size**, not from yearly volume, so the fix is smaller orders for slow items.
| Order size group | Products | Expired cost (Rs) | % of loss |
| --- | ---: | ---: | ---: |
| Orders cover more than 3.5 months | 12 | 739,743 | 67.9 |
| Orders cover 3.5 months or less | 74 | 350,487 | 32.1 |
 
*Table of expiry loss by order size group*
 
| Product | Category | Months of sales per order | Expired cost (Rs) |
| --- | --- | ---: | ---: |
| Ensure Vanilla 400g | Vitamins & Supplements | 6.1 | 130,456 |
| Insulin Glargine Pen 3ml | Diabetes | 6.3 | 129,116 |
| Januvia 100mg 28s | Diabetes | 5.7 | 112,439 |
| Galvus 50mg 28s | Diabetes | 7.0 | 96,222 |
| Nan Pro 1 400g | Baby Care | 6.8 | 95,904 |
 
*Table of the five products with the biggest expiry loss (from the companion query in the same step)*
 
# What I Learned
 
Throughout this project, I practised the full workflow of a data analyst in SQL:
 
- **🧹 Cleaning real-world data:** Loaded raw exports into staging tables, measured every problem first, then cleaned with `CASE`, `NULLIF`, `TO_TIMESTAMP` and `PERCENTILE_DISC`, and reconciled every removed row.
- **🧩 Complex query crafting:** Used CTEs, `CROSS JOIN LATERAL`, window functions (`LAG`, `SUM() OVER`) and conditional aggregation with `FILTER` to answer multi-step questions.
- **🛡️ Data integrity:** Built typed tables with primary keys, foreign keys and `CHECK` rules, so bad data cannot come back.
- **💡 Business thinking:** Turned results into decisions, and learned that a good number needs a fair comparison: the same products before and after, order size instead of yearly volume, and regular patients instead of occasional buyers.
# Conclusions
 
### Insights
From the analysis, several insights emerged:
 
1. **Margin squeeze:** after July, purchase costs rose about twice as fast as regulated prices, and gross margin fell from 19.5% to 16.6%.
2. **Where the margin is:** over-the-counter and care products earn the margin; prescription medicines bring the revenue at a low margin.
3. **Expiry is a buying problem:** 12 slow, expensive products ordered in large batches cause 68% of the Rs 1.09 million expiry loss.
4. **Supplier prices differ:** two suppliers cost about 8% more for the same products, worth up to Rs 0.84 million a year.
5. **Patients are leaving:** 21.9% of regular chronic patients have stopped buying, about Rs 0.9 million of yearly sales.
