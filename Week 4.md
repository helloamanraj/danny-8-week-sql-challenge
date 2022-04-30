### A. Customer Nodes Exploration
**Q1: How many unique nodes are there on the Data Bank system?**
 
  ```sql
  select count(distinct(node_id)) from customer_nodes
 ```
  **Q2:  What is the number of nodes per region?**
 
 ```sql
  select count(node_id) as node_count, region_name from customer_nodes as c
  inner join regions as r on c.region_id = r.region_id
  group by region_name
  order by node_count
```
 
 **Q3: How many customers are allocated to each region?**
 
  ```sql
  select count(customer_id) as customer_count, region_name from customer_nodes as c
  inner join regions as r on c.region_id = r.region_id
  group by region_name
  order by customer_count
 ```

  **Q4: How many days on average are customers reallocated to a different node?**
 
 ```sql
  with node_diff as (
  select node_id, start_date, end_date , (end_date - start_date) as node_diff
  from customer_nodes
  group by node_id
  ),
 
  sum_diff as (
  select node_id, sum(node_diff) as sum_of_diff from node_diff
  group by node_id )
 
  select avg(sum_of_diff) from sum_diff
 ```


**Q5: What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**
 

#median
 
```sql
with region_diff as (
select region_name, start_date, end_date , (end_date - start_date) as node_diff
from customer_nodes as c
inner join regions as r on c.region_id = r.region_id
group by region_name
)
 
SELECT AVG(dd.node_diff) as median_val
FROM (
SELECT d.node_diff, @rownum:=@rownum+1 as `row_number`, @total_rows:=@rownum
  FROM region_diff d, (SELECT @rownum:=0) r
  WHERE d.node_diff is NOT NULL
  -- put some where clause here
  ORDER BY d.node_diff
) as dd
WHERE dd.row_number IN ( FLOOR((@total_rows+1)/2), FLOOR((@total_rows+2)/2) );
```


#80% percentile

```sql
SELECT * FROM
(SELECT t.*,  @row_num :=@row_num + 1 AS row_num FROM region_diff t,
    (SELECT @row_num:=0) counter ORDER BY node_diff)
temp WHERE temp.row_num = ROUND (.80* @row_num);
```



#95% percentile

```sql
SELECT * FROM
(SELECT t.*,  @row_num :=@row_num + 1 AS row_num FROM region_diff t,
    (SELECT @row_num:=0) counter ORDER BY node_diff)
temp WHERE temp.row_num = ROUND (.95* @row_num);

```










### B. Customer Transactions
***Q1. What is the unique count and total amount for each transaction type?**
```sql
select count(distinct customer_id) as customer_count, sum(txn_amount) as total_sum_amount, txn_type from customer_transactions
group by txn_type
```

**Q2. What is the average total historical deposit counts and amounts for all customers?**

```sql
select round(avg(type_count),0), round(avg(avg_amount),0) from
(select customer_id, txn_type, count(txn_type) as type_count , avg(txn_amount) as avg_amount from customer_transactions
group by customer_id, txn_type) x
where x.txn_type = 'deposit'
```

**Q3: For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**

```sql
WITH monthly_transactions AS (
  SELECT
    customer_id,
    month(txn_date) AS month_date,
    SUM(CASE WHEN txn_type = 'deposit' THEN 0 ELSE 1 END) AS deposit_count,
    SUM(CASE WHEN txn_type = 'purchase' THEN 0 ELSE 1 END) AS purchase_count,
    SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawal_count
  FROM customer_transactions
  GROUP BY customer_id, month_date
 )

SELECT
  month_date,
  COUNT(DISTINCT customer_id) AS customer_count
FROM monthly_transactions
WHERE deposit_count >= 2
  AND (purchase_count > 1 OR withdrawal_count > 1)
GROUP BY month_date
ORDER BY month_date

```

**Q4: What is the closing balance for each customer at the end of the month?**

```sql
select customer_id, closing_month, sum(x.transaction_balance) from (SELECT
    customer_id,
    last_day(txn_date) AS closing_month,
    txn_type,
    txn_amount,
    SUM(CASE WHEN txn_type = 'withdrawal' OR txn_type = 'purchase' THEN (-txn_amount)
      ELSE txn_amount END) AS transaction_balance
  FROM customer_transactions
 
  GROUP BY customer_id, txn_date, txn_type, txn_amount)x
group by x.customer_id, x.closing_month
order by x.customer_id
```
**Q5: What is the percentage of customers who increase their closing balance by more than 5% ?**
This question needs more clarifications 
