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

**Main Techniques**: Window functions, nested queries (CTEs), and data unnesting techniques.

**Goal**: Based on the dataset, I used queries to explore multiple aspects, such as bounce rate, revenue by traffic source over time, or average number of pageviews by purchaser type. 

**Details:** This is a Google Analytics public dataset within Google BigQuery. It contains information about an eCommerce company, with user sessions collected.
  
**Links to dataset info:** https://support.google.com/analytics/answer/3437719?hl=en

<div id='q1'/>

## Query 01: Calculate total visit, pageview, transaction for Jan, Feb and March 2017

```sql
SELECT 
    FORMAT_DATE("%m-%Y", PARSE_DATE("%Y%m%d", date)) AS month,
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
| month  | visits | pageviews | transactions |
|--------|--------|-----------|--------------|
| 01-2017 | 64694  | 257708    | 713          |
| 02-2017 | 62192  | 233373    | 733          |
| 03-2017 | 69931  | 259522    | 993          |

✅ March 2017 had the most transactions, while January 2017 had the highest visits.

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
| Row | source                | total_visits | total_no_of_bounces | bounce_rate |
|-----|-----------------------|--------------|---------------------|-------------|
| 1   | google                | 38400        | 19798               | 51.56      |
| 2   | (direct)              | 19891        | 8606                | 43.27      |
| 3   | youtube.com           | 6351         | 4238                | 66.73      |
| 4   | analytics.google.com  | 1972         | 1064                | 53.96      |
| 5   | Partners              | 1788         | 936                 | 52.35      |
| 6   | m.facebook.com        | 669          | 430                 | 64.28      |
| 7   | ...                   | ...          |                     |             |

✅ Google had the highest total visits, while Youtube and Facebook had the high bounce rate.

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
| Row | time_type | time   | source   | revenue    |
|-----|-----------|--------|----------|------------|
| 1   | Month     | 06-2017 | (direct) | 97333.62 |
| 2   | Week      | 24-2017 | (direct) | 30908.91 |
| 3   | Week      | 25-2017 | (direct) | 27295.32 |
| 4   | Month     | 06-2017 | google   | 18757.18 |
| 5   | Week      | 23-2017 | (direct) | 17325.68 |
| 6   | Week      | 26-2017 | (direct) | 14914.81 |
| 7   | Week      | 24-2017 | google   | 9217.17  |
| 8   | Month     | 06-2017 | dfa      | 8862.23  |
| 9   | Week      | 22-2017 | (direct) | 6888.9  |
| 10  | Week      | 26-2017 | google   | 5330.57  |
| 10  | ...       |        |          |            |

✅ Compared to June 2017, July 2017 had higher average pageviews per purchase and slightly higher average pageviews for non-purchase visitors.

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
| Month  | Avg_total_transactions_per_user |
|--------|---------------------------------|
| 201707 | 4.16                |

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
| Month  | avg_revenue_by_user_per_visit |
|--------|-------------------------------|
| 2017-07 | 43.86                         |

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
| Row | other_purchased_products                | quantity |
|-----|-----------------------------------------|----------|
| 1   | Google Sunglasses                       | 20       |
| 2   | Google Women's Vintage Hero ...         | 7        |
| 3   | SPF-15 Slim & Slender Lip Balm          | 6        |
| 4   | Google Women's Short Sleeve ...         | 4        |
| 5   | YouTube Men's Fleece Hoodie ...         | 3        |
| 6   | Google Men's Short Sleeve Bad...        | 3        |
| 7   | ...                                     |          |

✅ Customers who bought the YouTube Men's Vintage Henley favored Google sunglasses, highlighting a strong interest in casual wear and accessories across Google, or YouTube.

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
| Row | month  | num_product_view | num_addtocart | num_purchase | add_to_cart_rate | purchase_rate |
|-----|--------|------------------|---------------|--------------|------------------|---------------|
| 1   | 201701 | 25787            | 7342          | 2143         | 28.47            | 8.31          |
| 2   | 201702 | 21489            | 7360          | 2060         | 34.25            | 9.59          |
| 3   | 201703 | 23549            | 8782          | 2977         | 37.29            | 12.64         |

✅ March 2017 showed an upward trend with the highest purchase rate and number of purchases, while February 2017 had the highest add-to-cart rate, indicating improved engagement over the months.
