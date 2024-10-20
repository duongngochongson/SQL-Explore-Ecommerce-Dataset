# SQL Explore Ecommerce Dataset
The project aims to explore some insights such as on user engagement and revenue trends of specific product over time for data analysts and eCommerce professionals.

## Table of Contents:
1. [Overall](#overall)
2. [Query 1](#q1)
3. [Query 2](#q2)
4. [Query 3](#q3)
5. [Query 4](#q4)
6. [Query 5](#q5)
7. [Query 6](#q6)
8. [Query 7](#q7)
9. [Query 8](#q7)

<div id='overall'/>
  
## Overall

**Platform**: Google Bigquery

**Notable Query**: Window functions, nested queries (CTEs), and data unnesting techniques.

**Goal**: Based on the dataset, I used queries to explore multiple aspects, such as bounce rate, revenue by traffic source over time, or average number of pageviews by purchaser type. 

**Details:** This is a Google Analytics public dataset within Google BigQuery. It contains information about an eCommerce company, with user sessions collected.
  
**Links to dataset info:** https://support.google.com/analytics/answer/3437719?hl=en

<div id='q1'/>

## Query 01: Calculate total visit, pageview, transaction for Jan, Feb and March 2017

```sql
SELECT 
    FORMAT_DATE("%b-%Y", PARSE_DATE("%Y%m%d", date)) AS month,
    SUM(totals.visits) AS number_of_visit,
    SUM(totals.pageviews) AS number_of_pageview,
    SUM(totals.transactions) AS number_of_transaction
FROM 
    `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE 
    _table_suffix BETWEEN '0101' AND '0331'
GROUP BY 
    month;
```

<div id='q2'/>
  
## Query 02: Bounce rate per traffic source in July 2017

```sql
SELECT 
    trafficSource.source,
    COUNT(visitNumber) AS total_visits,
    SUM(totals.bounces) AS total_no_of_bounces,
    ROUND(SUM(totals.bounces) / COUNT(visitNumber) * 100, 2) AS bounce_percent
FROM 
    `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY 
    trafficSource.source
ORDER BY 
    total_visits DESC;
```

<div id='q3'/>

## Query 3: Revenue by traffic source by week, by month in June 2017

```sql
WITH monthly_revenue AS (
    SELECT
        "month" AS time_type,
        FORMAT_DATE("%m-%Y", PARSE_DATE("%Y%m%d", date)) AS time,
        trafficSource.source AS source,
        ROUND(SUM(product.productRevenue / 1000000), 2) AS revenue
    FROM
        `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
    WHERE
        product.productRevenue IS NOT NULL
    GROUP BY
        time_type, time, source
),

weekly_revenue AS (
    SELECT
        "week" AS time_type,
        FORMAT_DATE("%W-%Y", PARSE_DATE("%Y%m%d", date)) AS time,
        trafficSource.source AS source,
        ROUND(SUM(product.productRevenue) / 1000000, 2) AS revenue
    FROM
        `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
    WHERE
        product.productRevenue IS NOT NULL
    GROUP BY
        time_type, time, source
)

SELECT * FROM monthly_revenue
UNION ALL 
SELECT * FROM weekly_revenue
ORDER BY revenue DESC;
```

<div id='q4'/>
  
## Query 04: Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.

```sql
WITH purchase_data AS (
    SELECT
        FORMAT_DATE("%m-%Y", PARSE_DATE("%Y%m%d", date)) AS month,
        ROUND(SUM(totals.pageviews) / COUNT(DISTINCT(fullVisitorId)), 2) AS avg_pageviews_purchase
    FROM
        `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
    WHERE
        totals.transactions IS NOT NULL
        AND product.productRevenue IS NOT NULL
        AND _table_suffix BETWEEN '0601' AND '0731'
    GROUP BY month
),

non_purchase_data AS (
    SELECT
        FORMAT_DATE("%m-%Y", PARSE_DATE("%Y%m%d", date)) AS month,
        ROUND(SUM(totals.pageviews) / COUNT(DISTINCT(fullVisitorId)), 2) AS avg_pageviews_non_purchase
    FROM
        `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
    WHERE
        totals.transactions IS NULL
        AND product.productRevenue IS NULL
        AND _table_suffix BETWEEN '0601' AND '0731'
    GROUP BY month
)

SELECT
    p.month,
    p.avg_pageviews_purchase,
    n.avg_pageviews_non_purchase
FROM
    purchase_data p
FULL OUTER JOIN
    non_purchase_data n ON p.month = n.month
ORDER BY p.month;
```

<div id='q5'/>
  
## Query 05: Average number of transactions per user that made a purchase in July 2017

```sql
SELECT
    "201707" AS Month,
    ROUND(SUM(totals.transactions) / COUNT(DISTINCT(fullVisitorId)), 2) AS Avg_total_transactions_per_user
FROM
    `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
    UNNEST(hits) AS hits,
    UNNEST(hits.product) AS product
WHERE
    totals.transactions >= 1
    AND product.productRevenue IS NOT NULL;
```

<div id='q6'/>

## Query 06: Average amount of money spent per session. Only include purchaser data in July 2017

```sql
SELECT 
    FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', date)) AS month,
    ROUND((SUM(product.productRevenue) / SUM(totals.visits)) / 1000000, 2) AS Avg_revenue_by_user_per_visit
FROM 
    `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`, 
    UNNEST(hits) AS hits, 
    UNNEST(hits.product) AS product
WHERE 
    product.productRevenue IS NOT NULL
    AND totals.transactions IS NOT NULL
GROUP BY 
    month;
```

<div id='q7'/>
  
## Query 07: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017.

```sql
WITH henley_cust_id AS (
    SELECT DISTINCT fullVisitorId
    FROM 
        `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
    WHERE 
        product.v2ProductName = "YouTube Men's Vintage Henley"
        AND totals.transactions >= 1
        AND product.productRevenue IS NOT NULL
)

SELECT 
    product.v2ProductName AS other_purchased_products,
    SUM(product.productQuantity) AS quantity
FROM 
    `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
    UNNEST(hits) AS hits,
    UNNEST(hits.product) AS product
WHERE 
    fullVisitorId IN (SELECT * FROM henley_cust_id)
    AND product.v2ProductName <> "YouTube Men's Vintage Henley"
    AND totals.transactions >= 1
    AND product.productRevenue IS NOT NULL
GROUP BY 
    product.v2ProductName
ORDER BY 
    quantity DESC;
```

<div id='q8'/>
  
## Query 08: Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017.

```sql
WITH cart AS (
    SELECT
        FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
        COUNT(eCommerceAction.action_type) AS num_addtocart
    FROM 
        `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits
    WHERE 
        _table_suffix BETWEEN '0101' AND '0331'
        AND eCommerceAction.action_type = '3'
    GROUP BY 
        month 
),

productview AS (
    SELECT
        FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
        COUNT(eCommerceAction.action_type) AS num_product_view
    FROM 
        `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
        UNNEST(hits) AS hits
    WHERE 
        _table_suffix BETWEEN '0101' AND '0331'
        AND eCommerceAction.action_type = '2'
    GROUP BY 
        month 
),

purchase AS (
    SELECT
        FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
        COUNT(eCommerceAction.action_type) AS num_purchase
    FROM 
        `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,   
        UNNEST(hits) AS hits,
        UNNEST(hits.product) AS product
    WHERE 
        _table_suffix BETWEEN '0101' AND '0331'
        AND eCommerceAction.action_type = '6'
        AND product.productRevenue IS NOT NULL
        AND totals.transactions IS NOT NULL
    GROUP BY 
        month
)

SELECT 
    month,
    num_product_view,
    num_addtocart,
    num_purchase,
    ROUND(num_addtocart / num_product_view * 100, 2) AS add_to_cart_rate,
    ROUND(num_purchase / num_product_view * 100, 2) AS purchase_rate
FROM 
    productview
JOIN 
    cart USING (month)
JOIN 
    purchase USING (month)
ORDER BY 
    month;
```
