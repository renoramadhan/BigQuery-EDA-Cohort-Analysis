# TheLook Ecommerce: Using BigQuery to do Exploratory Data Analysis (EDA) & Create Cohort Retention Analysis.

## A. About the dataset
TheLook is a fictitious eCommerce clothing site developed by the Looker team. The dataset contains information about customers, products, orders, logistics, web events and digital marketing campaigns. The contents of this dataset are synthetic, and are provided to industry practitioners for the purpose of product discovery, testing, and evaluation. (<a href="https://console.cloud.google.com/marketplace/product/bigquery-public-data/thelook-ecommerce?q=search&referrer=search&project=sincere-torch-350709" target="Data Source">Data Source</a>)

## B. Create Entity Relationship Diagram (ERD)
To understand the data well & simplify the join table process, which will be done later, it's a good idea to make an Entity Relationship Diagram first, as shown in the following figure.

![ERD_THELOOK_ECOMMERCE](https://user-images.githubusercontent.com/39979466/211189672-30b1e1cf-03a8-4b04-a3e9-5e297d033007.png)


## C. Exploratory Data Analysis (EDA)
### 1. Get the number of unique users, number of orders, and total sale price per status and month (Jan 2019 until Aug 2022)

```
SELECT
       DATE_TRUNC(DATE(created_at), MONTH) AS month_year,
       status AS status,
       COUNT(DISTINCT user_id) AS total_unique_users,
       COUNT(DISTINCT order_id) AS total_orders,
       SUM(sale_price) AS total_sales_price
FROM `bigquery-public-data.thelook_ecommerce.order_items`
WHERE created_at BETWEEN '2019-01-01' AND '2022-09-01'
GROUP BY 1,2
ORDER BY 2,1
```

Below is the table result

<img width="544" alt="01" src="https://user-images.githubusercontent.com/39979466/211228411-e1c09616-d511-4eb5-8b2a-70e67f279ef0.png">

### 2. Get frequencies, average order value and total number of unique users where status is complete grouped by month (Jan 2019 until Aug 2022)

```
SELECT 
       DATE_TRUNC(DATE(shipped_at), MONTH) AS month_year,
       ROUND(COUNT(DISTINCT order_id)/COUNT(DISTINCT user_id),2) AS frequency,
       ROUND(SUM(sale_price)/COUNT(DISTINCT order_id),2) AS aov,
       COUNT(DISTINCT(user_id)) AS number_of_unique_users
FROM `bigquery-public-data.thelook_ecommerce.order_items` 
WHERE shipped_at BETWEEN '2019-01-01' AND '2022-09-01' AND status = 'Complete'
GROUP BY 1
ORDER BY 1
```

Below is the table result

<img width="409" alt="02" src="https://user-images.githubusercontent.com/39979466/211228739-7bfa3a73-9c8b-44e7-813e-757c95be2ff4.png">

### 3. Find the user id, email, first and last name of users whose status is refunded on Aug 22

```
SELECT DISTINCT (users_.id) AS user_id,
       users_.email AS email,
       users_.first_name,
       users_.last_name
FROM `bigquery-public-data.thelook_ecommerce.orders` AS orders_
LEFT JOIN `bigquery-public-data.thelook_ecommerce.users` AS users_
ON orders_.user_id = users_.id
WHERE orders_.returned_at BETWEEN '2022-08-01' AND '2022-09-01' 
      AND orders_.status = 'Returned'
ORDER BY 3
```

Below is the table result

<img width="570" alt="03" src="https://user-images.githubusercontent.com/39979466/211229114-a2a3cc1f-d4d0-4d28-9421-a62d0c0756b9.png">

### 4. Get the top 5 least and most profitable product over all time

```
WITH
  most AS (
  SELECT
    c.product_id AS product_id,
    c.product_name AS product_name,
    c.sale_price AS sale_price,
    c.cost AS cost,
    c.profit AS profit
  FROM (
    SELECT
      DENSE_RANK() OVER(ORDER BY a.profit) AS b,
      a.product_id,
      a.product_name,
      a.sale_price,
      a.cost,
      a.profit
    FROM (
      SELECT
        DISTINCT z.id AS product_id,
        z.name AS product_name,
        orderitems.sale_price AS sale_price,
        z.cost AS cost,
        SAFE_SUBTRACT(orderitems.sale_price, z.cost) AS profit,
      FROM
        `bigquery-public-data.thelook_ecommerce.products` AS z
      LEFT JOIN
        `bigquery-public-data.thelook_ecommerce.order_items` AS orderitems
      ON
        z.id = orderitems.product_id) AS a
    ORDER BY
      b DESC ) AS c
  LIMIT
    5 ),
  least_ AS (
  SELECT
    f.product_id AS product_id,
    f.product_name AS product_name,
    f.sale_price AS sale_price,
    f.cost AS cost,
    f.profit AS profit
  FROM (
    SELECT
      DENSE_RANK() OVER(ORDER BY d.profit) AS e,
      d.product_id,
      d.product_name,
      d.sale_price,
      d.cost,
      d.profit
    FROM (
      SELECT
        DISTINCT y.id AS product_id,
        y.name AS product_name,
        orderitems.sale_price AS sale_price,
        y.cost AS cost,
        SAFE_SUBTRACT(orderitems.sale_price, cost) AS profit,
      FROM
        `bigquery-public-data.thelook_ecommerce.products` AS y
      LEFT JOIN
        `bigquery-public-data.thelook_ecommerce.order_items` AS orderitems
      ON
        y.id = orderitems.product_id
      WHERE
        orderitems.sale_price IS NOT NULL) AS d
    ORDER BY
      e ASC ) AS f
  LIMIT
    5 )
  SELECT
    *
  FROM
    most
  UNION ALL
  SELECT
    *
  FROM
    least_
```

Below is the table result

<img width="634" alt="04" src="https://user-images.githubusercontent.com/39979466/211230514-a77bc735-018e-4c95-b2a5-e7c045577520.png">


### 5. Get Month to Date of total profit in each product categories of past 3 months (current date 15 Aug 2022), breakdown by date

```
WITH
  b AS (
  WITH
    a AS (
    SELECT
      DATE_TRUNC(DATE(created_at),day) AS date_,
      EXTRACT(YEAR
      FROM
        created_at) AS year,
      EXTRACT(MONTH
      FROM
        created_at) AS month,
      EXTRACT(DAY
      FROM
        created_at) AS day,
      SAFE_SUBTRACT(retail_price, cost) AS profit,
      products.category AS product_category
    FROM
      `bigquery-public-data.thelook_ecommerce.order_items` orderitems
    INNER JOIN
      `bigquery-public-data.thelook_ecommerce.products` products
    ON
      orderitems.product_id = products.id
      AND created_at >= '2022-06-01 00:00:00 UTC'
      AND created_at <='2022-08-15 23:59:59 UTC'
    GROUP BY
      date_,
      year,
      month,
      day,
      product_category,
      profit )
  SELECT
    a.date_ AS Date,
    a.year,
    a.month,
    a.day,
    a.product_category AS Product_Categories,
    SUM(a.profit) AS Profit
  FROM
    a
  WHERE
    a.day <= 15
  GROUP BY
    a.date_,
    a.year,
    a.month,
    a.day,
    a.product_category
  ORDER BY
    a.date_,
    year,
    month,
    day,
    a.product_category)
SELECT
  Date,
  b.Product_Categories,
  round(b.profit) as profit,
  round(SUM(Profit) OVER(PARTITION BY product_categories, month ORDER BY date)) AS profit_cumulative
FROM
  b
```

Below is the table result

<img width="367" alt="05" src="https://user-images.githubusercontent.com/39979466/211230791-28e853e8-92ac-4a3f-acfb-04add287134b.png">

### 6. Find monthly growth of inventory in percentage breakdown by product categories, ordered by time descendingly. After analyzing the monthly growth, is there any interesting insight that we can get? (Jan 2019 until Apr 2022)

```
WITH
  register_inventory AS (
  SELECT
     DATE_TRUNC(DATE(created_at), MONTH) AS month_year,
    EXTRACT(YEAR
    FROM
      created_at) AS year,
    EXTRACT(MONTH
    FROM
      created_at) AS month,
    COUNT(DISTINCT id) AS total_registered_inventory,
    product_category AS categories
  FROM
    `bigquery-public-data.thelook_ecommerce.inventory_items`
  WHERE
    created_at BETWEEN '2019-01-01 00:00:00 UTC'
    AND '2022-04-01 23:59:59 UTC'
  GROUP BY
    month_year,
    year,
    month,
    product_category
  ORDER BY
    month_year,
    year,
    month),
  add_prev_inventory_register AS (
  SELECT
    *,
    LAG(total_registered_inventory) OVER(PARTITION BY categories ORDER BY year, month) AS prev_inventory_registered,
  FROM
    register_inventory)
SELECT
  month_year,
  categories,
  ROUND((total_registered_inventory - prev_inventory_registered)/prev_inventory_registered,2) AS growth
FROM
  add_prev_inventory_register
ORDER BY
  categories,
  year DESC,
  month DESC
```

Below is the table result

<img width="314" alt="06" src="https://user-images.githubusercontent.com/39979466/211231043-f62f5f90-6d3a-4f0f-a0f6-cdc2c0f67ff9.png">

### 7. Create monthly retention cohorts (the groups, or cohorts, can be defined based upon the date that a user completely purchased a product) and then how many of them (%) coming back for the following months in 2022. After analyzing the retention cohort, is there any interesting insight that we can get?

```
  #QUESTION 7
  WITH
  cohort_items AS (
  SELECT
    user_id,
    MIN(DATE_TRUNC(DATE(created_at), month)) AS cohort_month,
  FROM
    `bigquery-public-data.thelook_ecommerce.orders`
  GROUP BY
    user_id
  ORDER BY
    cohort_month ),
  user_activity AS (
  SELECT
    orders_table.user_id,
    DATE_DIFF(DATE(DATE_TRUNC(created_at,month)),cohort.cohort_month,month) AS month_number
  FROM
    `bigquery-public-data.thelook_ecommerce.orders` AS orders_table
  LEFT JOIN
    cohort_items AS cohort
  ON
    orders_table.user_id = cohort.user_id
  -- WHERE
  --   DATE(created_at) BETWEEN '2022-01-01'
  --   AND '2022-12-31'
    -- EXTRACT(year
    -- FROM
    --   created_at) IN (2022)
  WHERE EXTRACT(YEAR FROM cohort.cohort_month) IN (2022)
    AND EXTRACT(YEAR FROM DATE_TRUNC(orders_table.created_at, YEAR)) IN (2022)
    -- GROUP BY 1,2
  GROUP BY
    user_id,
    month_number
  ORDER BY
    month_number),
  cohort_size AS (
  SELECT
    cohort_month,
    COUNT(cohort_month) AS num_users
  FROM
    cohort_items
  GROUP BY
    cohort_month
  ORDER BY
    cohort_month),
  retention_table AS (
  SELECT
    cohort_month,
    COUNT(CASE
        WHEN month_number =1 THEN cohort_month
    END
      ) AS M1,
    COUNT (CASE
        WHEN month_number =2 THEN cohort_month
    END
      ) AS M2,
    COUNT (CASE
        WHEN month_number=3 THEN cohort_month
    END
      ) AS M3,
    COUNT (CASE
        WHEN month_number=4 THEN cohort_month
    END
      ) AS M4,
    COUNT (CASE
        WHEN month_number=5 THEN cohort_month
    END
      ) AS M5,
    COUNT (CASE
        WHEN month_number=6 THEN cohort_month
    END
      ) AS M6
  FROM
    cohort_items AS cohortitems
  LEFT JOIN
    user_activity AS useractivity
  ON
    useractivity.user_id= cohortitems.user_id
  GROUP BY
    cohort_month
  ORDER BY
    cohort_month)
SELECT
  retentiontable.cohort_month AS Month,
  cohortsize.num_users AS M,
  M1,
  M2,
  M3,
  M4,
  M5,
  M6
FROM
  retention_table AS retentiontable
LEFT JOIN
  cohort_size AS cohortsize
ON
  retentiontable.cohort_month = cohortsize.cohort_month
WHERE EXTRACT (YEAR FROM retentiontable.cohort_month) IN (2022)
ORDER BY
  month
```

Below is the table result

<img width="643" alt="07" src="https://user-images.githubusercontent.com/39979466/211235045-f5f58c6f-3719-4a70-9b5b-a62c2f625d3f.png">

After that, I exported the query results to a spreadsheet to further analyze the cohort percentage for each month number. The results can be seen below.

<img width="1098" alt="08" src="https://user-images.githubusercontent.com/39979466/211237522-a4459558-2881-49cf-8cc1-fbbcb94e1edd.png">

<img width="719" alt="09" src="https://user-images.githubusercontent.com/39979466/211240312-d1398330-8df5-4ed1-aa66-f731f8fc7c08.png">

**Insights** :
1. TheLook E-Commerce needs to maintain the retention rate. It can be seen that the average has fallen from 7% to 3%

2. The highest retention rate was found in M1 in November 2022, which was 17%.

3. There is a relatively high increase in M1 (the first month the user returns to purchase after the first purchase), namely from 8% to 17% from August to November 2022

**Recommendation** :
1. TheLook Ecommerce may think about developing a marketing and advertising plan to attract new users, in order to lower the churn rate.

2. The company can create discounts, promo events through their digital marketing campaign.

