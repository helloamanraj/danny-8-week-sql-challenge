  ## Data Cleaning steps

**Q1: In a single query, perform the following operations and generate a new table in the data_mart schema named clean_weekly_sales?**

```sql
DROP TABLE IF EXISTS clean_weekly_sales;
CREATE TABLE clean_weekly_sales AS (
SELECT
  str_to_date(week_date, '%d/%m/%y') AS week_date,
  week( week_date) as week_number,
  month(week_date) AS month_number,
  year(week_date) AS calendar_year,
  region,
  platform,
  segment,
   CASE WHEN RIGHT(segment,1) = '1' THEN 'Young Adults'
    WHEN RIGHT(segment,1) = '2' THEN 'Middle Aged'
    WHEN RIGHT(segment,1) in ('3','4') THEN 'Retirees'
    ELSE 'unknown' END AS age_band,
  CASE WHEN LEFT(segment,1) = 'C' THEN 'Couples'
    WHEN LEFT(segment,1) = 'F' THEN 'Families'
    ELSE 'unknown' END AS demographic,
  transactions,
  round((sales/transactions),2) AS avg_transaction,
  sales
FROM weekly_sales
)
```

## 2.Data Exploration
                                           
                                           
**Q1: What day of the week is used for each week_date value?**

```sql
Select distinct(dayname(week_date))from clean_weekly_sales
```

**Q2: What range of week numbers are missing from the dataset?**
```sql

with cte as (
select row_number() over (order by week_number) as rw_num
from clean_weekly_sales
limit 52
),
cte_1 as (
select distinct week_number as wn from clean_weekly_sales order by week_number)

SELECT
  DISTINCT cte.rw_num as missing_week
FROM cte
LEFT JOIN cte_1
  ON cte.rw_num = cte_1.wn
WHERE cte_1.wn IS NULL

```

**Q3: How many total transactions were there for each year in the dataset?**

```sql
select year(week_date) as yea_num, sum(transactions) from clean_weekly_sales
group by yea_num
```

**Q4: What is the total sales for each region for each month?**
```sql
select sum(sales) , region , month(week_date) as mnt from clean_weekly_sales
group by  mnt, region
order by region, mnt
```

**Q5: What is the total count of transactions for each platform?**

```sql
select sum(transactions) as txn , platform from clean_weekly_sales
group by platform
```


**Q6: What is the percentage of sales for Retail vs Shopify for each month?**

```sql

select mth,
(100* max(case when platform = 'Retail' then sal_sum else null end) /
(sum(sal_sum))) as ret_perc,
( 100* max(case
when platform = 'Shopify' then sal_sum else null end) /
(sum(sal_sum))) as shop_perc
from
(select platform, month(week_date) as mth, sum(sales) as sal_sum from clean_weekly_sales
group by platform, month(week_date))
x
group by  mth
order by mth
```


**Q7: What is the percentage of sales by demographic for each year in the dataset?**

```sql

select x.year_sep,
(100*min(case when demographic = "Couples" then sales_demo else null end)/ (sum(sales_demo))) as percent_couples,
(100*max(case when demographic = "Families" then sales_demo else null end) / sum(sales_demo)) as percent_families ,
(100*max(case when demographic = "unknown" then sales_demo else null end) / sum(sales_demo))as percent_unknown

 from (select sum(sales) as sales_demo, demographic, year(week_date) as year_sep from clean_weekly_sales
group by year(week_date), demographic
) x
group by  x.year_sep
```

**Q8: Which age_band and demographic values contribute the most to Retail sales?**


```sql
with cte as (select Platform, age_band, demographic, sum(sales) as sum_dis
from clean_weekly_sales
where platform = 'Retail'
group by age_band, demographic
)


Select *,round((100*(sum_dis)/ (Select sum(sum_dis) as Total from cte)),2)  as Total from cte
order by Total desc

```

**Q9: Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?**

```sql

select year(week_date) as calendar, platform , avg(avg_transaction) as Avg_tran,
SUM(sales) / sum(transactions) AS avg_transaction_group
from clean_weekly_sales
group by year(week_date), platform
order by calendar
```


## 3.Before & After Analysis

**Using this analysis approach - answer the following questions:**

**Q1: What is the total sales for the 4 weeks before and after 2020-06-15? What is the growth or reduction rate in actual values and percentage of sales?**


```sql
#for finding the week number
SELECT
  distinct week('2020-06-15')
FROM clean_weekly_sales


WITH changes AS (
  SELECT
    week_date,
    week_number,
    sales AS total_sales
  FROM clean_weekly_sales
  WHERE (week_number BETWEEN 21 AND 28)
    AND calendar_year = 2020
  GROUP BY week_date, week_number
),
changes_2 AS (
  SELECT
    SUM(CASE WHEN week_number BETWEEN 21 AND 24 THEN total_sales END) AS before_change,
    SUM(CASE WHEN week_number BETWEEN 25 AND 28 THEN total_sales END) AS after_change
  FROM changes)

SELECT
  before_change,
  after_change,
  after_change - before_change AS variance,
  ROUND(100 * (after_change - before_change) / before_change,2) AS percentage
FROM changes_2

```


 **Q2:  What about the entire 12 weeks before and after?**
 
```sql

 WITH changes AS (
  SELECT
    week_date,
    week_number,
    SUM(sales) AS total_sales
  FROM clean_weekly_sales
  WHERE (week_number BETWEEN 13 AND 37)
    AND (calendar_year = 2020)
  GROUP BY week_date, week_number
),
changes_2 AS (
  SELECT
    SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS before_change,
    SUM(CASE WHEN week_number BETWEEN 25 AND 37 THEN total_sales END) AS after_change
  FROM changes)

SELECT
  before_change,
  after_change,
  after_change - before_change AS variance,
  ROUND(100 * (after_change - before_change) / before_change,2) AS percentage
FROM changes_2
```
**Q3: How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?**

It can be done by removing the calendar_year filter from the above query.
