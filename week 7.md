for updating the column(member) of table sales  
```sql
set mem = 
(case when mem = "t" then "True" 
else "False" end )
```
 

## High Level Sales Analysis

**Q1: What was the total quantity sold for all products?**

```sql
select sum(qty) as total quantity from sales
```
**Q2:  What is the total generated revenue for all products before discounts?**

```sql
select sum(price*qty) as generated_revenue from sales
```
**Q3: What was the total discount amount for all products?**

```sql
select sum(discount*qty) as generated_revenue from sales
```


## Transaction Analysis
**Q1: How many unique transactions were there?**
```sql
select count(distinct(txn_id)) from sales
```
**Q2: What is the average unique products purchased in each transaction?**

```sql

select round(avg(cnt)) from (
select txn_id, count(distinct prod_id) as cnt from sales 
group by txn_id
)x
```
**Q3: What are the 25th, 50th and 75th percentile values for the revenue per transaction?**



```sql
 WITH cte_transaction_revenue AS (
  SELECT
    txn_id,
    SUM(qty * price) AS revenue
  FROM sales
  GROUP BY txn_id
)
```
The below query is for find the percentile just by putting the column name and table name 
```sql
SELECT revenue FROM                                                                    
(SELECT t.*,  @row_num :=@row_num + 1 AS row_num FROM cte_transaction_revenue as t,
    (SELECT @row_num:=0) counter ORDER BY revenue)
temp WHERE temp.row_num = ROUND (.75* @row_num);
```

Below query is to find 50th percentile
```sql
 WITH cte_transaction_revenue AS (
  SELECT
    txn_id,
    SUM(qty * price) AS revenue
  FROM sales
  GROUP BY txn_id
)
SELECT revenue FROM
(SELECT t.*,  @row_num :=@row_num + 1 AS row_num FROM cte_transaction_revenue as t,
    (SELECT @row_num:=0) counter ORDER BY revenue)
temp WHERE temp.row_num = ROUND (.50* @row_num);
```

Below query is to find 25th percentile
```sql
 WITH cte_transaction_revenue AS (
  SELECT
    txn_id,
    SUM(qty * price) AS revenue
  FROM sales
  GROUP BY txn_id
)
SELECT revenue FROM
(SELECT t.*,  @row_num :=@row_num + 1 AS row_num FROM cte_transaction_revenue as t,
    (SELECT @row_num:=0) counter ORDER BY revenue)
temp WHERE temp.row_num = ROUND (.25* @row_num);
```

**Q4: What is the average discount value per transaction?**
```sql
select round(avg(cnt)) from (
select txn_id, sum(qty*discount) as cnt from sales 
group by txn_id
)x
```

**Q5: What is the percentage split of all transactions for members vs non-members?**
```sql
SELECT 
	ROUND(100 * 
		  COUNT(DISTINCT CASE WHEN mem = true THEN txn_id END) / 
		  COUNT(DISTINCT txn_id)
		  , 2) AS member_transaction,
	(100 - ROUND(100 * 
		  COUNT(DISTINCT CASE WHEN mem = true THEN txn_id END) / 
		  COUNT(DISTINCT txn_id)
		  , 2)
	 ) AS non_member_transaction
FROM sales
```

**Q6: What is the average revenue for member transactions and non-member transactions?**

```sql

WITH cte_member_revenue AS (
  SELECT
    mem,
    txn_id,
    SUM(price * qty) AS revenue
  FROM sales
  GROUP BY 
	mem,
    txn_id
	
)
SELECT
  mem,
  ROUND(AVG(revenue), 2) AS avg_revenue
FROM cte_member_revenue
GROUP BY mem;
```

## Product Analysis
**Q1: What are the top 3 products by total revenue before discount?**

```sql
SELECT 
	product_name,
	SUM(qty * price) AS nodis_revenue
FROM sales AS a
INNER JOIN balanced_tree.product_details AS b
	ON aprod_id = b.product_id
GROUP BY details.product_name
LIMIT 3;
```

**Q2: What is the total quantity, revenue and discount for each segment?**

 ```sql
 SELECT 
	details.segment_id,
	details.segment_name,
	SUM(qty) AS total_quantity,
	SUM(qty * price) AS total_revenue,
	SUM(qty * discount)/AS total_discount
FROM sales AS a
INNER JOIN product_details AS b
	ON a.prod_id = b..product_id
GROUP BY 
	b.segment_id,
	b.segment_name
ORDER BY total_revenue DESC
```

**Q3: What is the top selling product for each segment?**

```sql

SELECT 
	segment_id,
	segment_name,
	product_id,
	product_name,
	SUM(qty) AS product_quantity
FROM sales AS a
INNER JOIN product_details AS b
	ON a.prod_id = b.product_id
GROUP BY
	segment_id,
	segment_name,
	product_id,
	product_name
    
```

**Q4:  What is the total quantity, revenue and discount for each category?**

```sql

SELECT 
	category_id,
	category_name,
	SUM(qty) AS total_quantity,
	SUM(a.qty * a.price) AS total_revenue,
	SUM(a.qty * a.price) AS total_discount
FROM sales AS a
INNER JOIN product_details AS b
	ON a.prod_id = b.product_id
GROUP BY 
	category_id,
	category_name

```
**Q5:  What is the top selling product for each category?**

```sql
SELECT 
	category_id,
category_name,
	product_id,
	product_name,
	SUM(a.qty) AS product_quantity
FROM sales AS a
INNER JOIN product_details AS b
	ON a.prod_id = b.product_id
GROUP BY
	category_id,
	category_name,
	product_id,
	product_name

```

**Q6: What is the percentage split of revenue by product for each segment?**


```sql

WITH cte AS (
  SELECT
    segment_id,
    segment_name,
    product_id,
    product_name,
    SUM(a.qty * a.price) AS product_revenue
  FROM sales as a
  INNER JOIN product_details as b
    ON a.prod_id = b.product_id
  GROUP BY
    segment_id,
    segment_name,
    product_id,
    product_name
)
SELECT
	segment_name,
	product_name,
	ROUND(
    100 * product_revenue /
      SUM(product_revenue) OVER (
        PARTITION BY segment_id),
    	2) AS segment_product_percentage
FROM cte
ORDER BY
	segment_id,
	segment_product_percentage DESC;

```

**Q7: What is the percentage split of revenue by segment for each category?**

```sql
WITH cte AS (
  SELECT
    segment_id,
    segment_name,
    category_id,
    category_name,
    SUM(a.qty * a.price) AS product_revenue
  FROM sales as a
  INNER JOIN product_details as b
    ON a.prod_id = b.product_id
  GROUP BY
    segment_id,
    segment_name,
    category_id,
    category_name
)
SELECT
	segment_name,
	category_name,
	ROUND(
    100 * product_revenue /
      SUM(product_revenue) OVER (
        PARTITION BY category_id),
    	2) AS category_product_percentage
FROM cte
ORDER BY
	category_product_percentage DESC

```

**Q8: What is the percentage split of total revenue by category?**

```sql

SELECT 
   ROUND(100 * SUM(CASE WHEN category_id = 1 THEN (a.qty * a.price) END) / 
		 SUM(qty * a.price),
		 2) AS category_1,
   (100 - ROUND(100 * SUM(CASE WHEN category_id = 1 THEN (a.qty * a.price) END) / 
		 SUM(qty * a.price),
		 2)
	) AS category_2
FROM sales AS a
INNER JOIN product_details AS b
	ON a.prod_id = b.product_id
```
**Q9: What is the total transaction “penetration” for each product?**

```sql

WITH product_transactions AS (
  SELECT 
	DISTINCT prod_id,
    COUNT(DISTINCT txn_id) AS product_transactions
  FROM sales
  GROUP BY prod_id
),
total_transactions AS (
  SELECT
    COUNT(DISTINCT txn_id) AS total_transaction_count
  FROM sales
)
SELECT
 product_id,
  product_name,
  ROUND(
    100 * product_transactions
      / total_transaction_count,
    2
  ) AS penetration_percentage
FROM product_transactions as pt
CROSS JOIN total_transactions 
INNER JOIN product_details as pd
  ON pt.prod_id = pd.product_id
ORDER BY penetration_percentage DESC

```