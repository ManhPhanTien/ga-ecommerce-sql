# E-commerce Behavior Analysis with BigQuery


All 8 queries below have been executed and validated directly in Google BigQuery. You can view the live project (queries, execution, and saved results) here:

👉 **[Open in BigQuery Console](https://console.cloud.google.com/bigquery?ws=!1m7!1m6!12m5!1m3!1sgolden-shine-472610-g7!2sus-central1!3s8401e328-64f2-4126-8ffb-c8957fd87998!2e1)**

---

## 📁 Project Overview

| | |
|---|---|
| **Dataset** | [`bigquery-public-data.google_analytics_sample.ga_sessions_2017`](https://console.cloud.google.com/marketplace/product/obfuscated-ga360-data/google-analytics-sample) — real (obfuscated) Google Analytics data from the Google Merchandise Store |
| **Tool** | Google BigQuery (Standard SQL) |
| **Time period analyzed** | January – July 2017 |
| **Focus areas** | Traffic trends, marketing channel performance, revenue attribution, purchaser behavior, cross-sell, conversion funnel |
| **Techniques used** | `UNNEST` on nested/repeated fields, CTEs (`WITH`), window-free aggregation, `_TABLE_SUFFIX` wildcard tables, `FORMAT_DATE`/`PARSE_DATE`, conditional aggregation (`COUNTIF`), self-joins via CTE, `UNION ALL` |

### Why this dataset?

The GA sample dataset stores each **session** as a row, with **hits** (page views, events, e-commerce actions) and **products** nested inside each hit as repeated `RECORD` fields. This mirrors how real analytics/event data is structured in production data warehouses — making it a realistic exercise in writing SQL against **semi-structured, nested data**, not just flat tables.

---

## 🎯 Business Questions & Key Findings

### Query 01 — Monthly Traffic & Conversion Overview (Jan–Mar 2017)

**Query:**
```sql
SELECT format_date("%Y%m",parse_date('%Y%m%d',date)) as month
      ,sum(totals.visits) as visits
      ,sum(totals.pageviews) as pageviews
      ,sum(totals.transactions) as transacitons
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix between '0101' and '0331'
GROUP BY month 
ORDER BY 1;
```

**Result:**

![Query 01 Result](images/Query_1.png)

---

### Query 02 — Bounce Rate by Traffic Source (July 2017)

**Query:**
```sql
SELECT trafficSource.`source`
      ,sum(totals.visits) as total_visits
      ,sum(totals.bounces) as total_no_of_bounces
      ,ROUND(
            100* SUM(totals.bounces)/sum(totals.visits)
            ,3) as transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY trafficSource.`source`
ORDER BY 2 DESC;
```

**Result:**

![Query 02 Result](images/Query_2.png)

---

### Query 03 — Revenue by Traffic Source, Weekly & Monthly (June 2017)

**Query:**
```sql
WITH week_out as 
      (SELECT 'week' as time_type
            ,format_date ('%Y%W',parse_date('%Y%m%d', date)) as week 
            ,trafficSource.`source`
            ,sum(product.productRevenue)/1000000 as revenue
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product
      GROUP BY week, trafficSource.`source`)

SELECT *
FROM 
      (SELECT 'month' as time_type
            ,'201706' as time 
            ,source
            ,sum(revenue) as revenue
      FROM week_out
      GROUP BY 3
      order by 4 DESC) as month_out 
UNION ALL 
SELECT * 
FROM week_out
ORDER BY revenue DESC;
```

**Result:**

![Query 03 Result](images/Query_3.png)

---

### Query 04 — Pageviews: Purchasers vs. Non-Purchasers (Jun–Jul 2017)

**Query:**
```sql
WITH purchase AS 
      (SELECT format_date('%Y%m',parse_date('%Y%m%d', date)) as month 
            ,sum(totals.pageviews) / count(distinct fullVisitorId) as avg_pageviews_purchase
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product
      WHERE (_table_suffix between '0601' and '0731')
            and totals.transactions >=1 
            and product.productRevenue is not null 
      GROUP BY month)
,non_purchase AS
      (SELECT format_date('%Y%m',parse_date('%Y%m%d', date)) as month 
            ,sum(totals.pageviews) / count(distinct fullVisitorId) as avg_pageviews_non_purchase
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product
      WHERE (_table_suffix between '0601' and '0731')
            and totals.transactions is null 
            and product.productRevenue is null 
      GROUP BY month)

SELECT *
FROM purchase
INNER JOIN non_purchase USING (month);
```

**Result:**

![Query 04 Result](images/Query_4.png)

---

### Query 05 — Avg. Transactions per Purchasing User (July 2017)

**Query:**
```sql
SELECT '201707' as month
      ,sum(totals.transactions)/count(distinct fullVisitorId) as Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
UNNEST (hits.product) product 
WHERE totals.transactions >=1 
      and product.productRevenue is not null 
GROUP BY 1;
```

**Result:**

![Query 05 Result](images/Query_5.png)

---

### Query 06 — Avg. Revenue per Session, Purchasers Only (July 2017)

**Query:**
```sql
SELECT '201707' as month 
      ,round (
            (sum(product.productRevenue)/sum(totals.visits)) / 1000000
      ,2) as avg_revenue_by_user_per_visit
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
WHERE totals.transactions is not null 
      and product.productRevenue is not null 
GROUP BY 1;
```

**Result:**

![Query 06 Result](images/Query_6.png)

---

### Query 07 — Cross-Sell Analysis: "YouTube Men's Vintage Henley" (July 2017)

**Query:**
```sql
WITH customer as
      (SELECT fullVisitorId
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
      UNNEST (hits) hits,
      UNNEST (hits.product) product
      WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
            and product.productRevenue is not null
            and totals.transactions >=1)

SELECT product.v2ProductName as other_purchased_products
      ,sum(product.productQuantity) as quantity 
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
UNNEST (hits) hits,
UNNEST (hits.product) product
WHERE product.productRevenue is not null
      and totals.transactions >=1
      and fullVisitorId in (select fullVisitorId from customer)
      and product.v2ProductName != "YouTube Men's Vintage Henley"
GROUP BY 1
ORDER BY 2 DESC;
```

**Result:**

![Query 07 Result](images/Query_7.png)

---

### Query 08 — Conversion Funnel: View → Add-to-Cart → Purchase (Jan–Mar 2017)

**Query:**
```sql
SELECT *
      ,round(100*num_addtocart/num_product_view,2) as add_to_cart_rate
      ,round(100*num_purchase/num_product_view,2) as purchase_rate
FROM
  (SELECT format_date('%Y%m', parse_date('%Y%m%d',date)) as month
        ,countif(hits.eCommerceAction.action_type = '2') as num_product_view
        ,countif (hits.eCommerceAction.action_type = '3') as num_addtocart
        ,countif (hits.eCommerceAction.action_type = '6' and product.productRevenue is not null) as num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
  UNNEST (hits) hits,
  UNNEST (hits.product) product
  WHERE _table_suffix between '0101' and '0331'
  GROUP BY month) as count_out
  ORDER BY month;
```

**Result:**

![Query 08 Result](images/Query_8.png)

---

## 🧠 Summary of Key Business Insights

1. **March 2017 converted better, not just attracted more traffic.** Transactions rose +35.5% from February to March while visits grew only +12.4%. The funnel confirms this: from January to March, add-to-cart rate climbed from 28.5% to 37.3% and purchase rate from 8.3% to 12.6%. Worth investigating what changed (promotion, seasonality, or traffic mix).
2. **Google drives volume, direct drives revenue.** In June 2017, `(direct)` traffic accounted for roughly 78% of revenue across the top three sources, even though `google` sent a larger share of visits. Traffic volume and revenue contribution are not the same thing.
3. **Engagement varies sharply by channel.** In July 2017, `(direct)` visitors bounced at 43.3%, while `youtube.com` visitors bounced at 66.7%, suggesting low-intent clicks. `reddit.com` had the lowest bounce rate (28.6%) but only 189 visits, making it a possible low-cost channel to test at scale.
4. **Heavy browsing does not mean buying.** In June and July 2017, non-purchasers viewed 2.7–3.4x more pages than purchasers. Decisive buyers appear to navigate directly to what they want, so fast, low-friction paths to checkout may matter more than maximizing content depth.
5. **Buyers come back to buy again.** Purchasing users averaged about 4.2 transactions in July 2017, so retention and remarketing to existing customers matter as much as acquisition.
6. **Clear cross-sell opportunity.** Google Sunglasses was the most common co-purchase with the YouTube Men's Vintage Henley, with 20 units sold, nearly 3x the next product. This makes it a concrete bundling candidate at checkout or on the product page.

