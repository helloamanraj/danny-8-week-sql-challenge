### A. Customer Journey##
**Q1: Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.**

  ```sql 
  select distinct a.Customer_id,  b.plan_name, a.start_date, b.plan_id
  from subscriptions as a
  inner join plans as b
  on a.plan_id = b.plan_id
  where a.customer_id in  (1,2,11,13,15,16,18,19)
  ```


### Data Analysis

** Q1: How many customers has Foodie-Fi ever had? **

```sql select count(distinct customer_id) from subscriptions
```

**Q2: What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value?**

```sql
select monthname(start_date), month(start_date), count(*) from plans as a
inner join subscriptions as b on a.plan_id = b.plan_id
where b.plan_id = 0
group by monthname(start_date) , month(start_date)
order by month(start_date)
```

** Q3: What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name.**

```sql select count(year(a.start_date)) as event_count, year(a.start_date) as in_year , b.plan_name from subscriptions as a
inner join plans as b on b.plan_id = a.plan_id
where year(a.start_date) > 2020
group by b.plan_name, year(a.start_date) 
```


**Q4: What is the customer count and percentage of customers who have churned rounded to 1 decimal place?**
 
sol 1:

```sql select y.*, round((307/(
    SELECT COUNT(DISTINCT customer_id)
    FROM subscriptions)* 100),1) as percentage from (select x.* from (select count(a.customer_id) as total, b.plan_name from subscriptions as a
inner join plans as b on a.plan_id = b.plan_id
group by b.plan_name ) x
where x.plan_name  = 'churn' ) y
```

sol 2:

```sql  
SELECT
  COUNT(*) AS churn_count,
  ROUND(100 * COUNT(*) / (
    SELECT COUNT(DISTINCT customer_id)
    FROM subscriptions),1) AS churn_percentage
FROM subscriptions s
WHERE plan_id = 4
```

**Q5: How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?**

```sql select count(*) as chrun_count, round((count(*) / (select count(distinct customer_id) from subscriptions)*100),0) as churn_percentage
from (

 select a.plan_id, a.customer_id, b.plan_name , row_number() over (partition by a.customer_id order by a.plan_id) as rw_num
 from subscriptions as a
 inner join plans as b on b.plan_id = a.plan_id
 )x
 where x.plan_id = 4 and x.rw_num = 2
 ```
**Q6: What is the number and percentage of customer plans after their initial free trial?**
sol 1:
```sql
select x.next_cte, count(*) as conversion, round((100*(count(*)/ (select count(distinct customer_id) from subscriptions))),1) as conversion_percent from (
select customer_id, plan_id, lead(plan_id,1) over ( partition by customer_id order by plan_id) as next_cte from subscriptions
) x
where x.next_cte is not null and plan_id = 0
group by x.next_cte
```
sol 2:
```sql
WITH next_plan_cte AS (
SELECT
  customer_id,
  plan_id,
  LEAD(plan_id, 1) OVER( -- Offset by 1 to retrieve the immediate row's value below
    PARTITION BY customer_id
    ORDER BY plan_id) as next_plan
FROM subscriptions)

SELECT
  next_plan,
  COUNT(*) AS conversions,
  ROUND(100 * COUNT(*) / (
    SELECT COUNT(DISTINCT customer_id)
    FROM subscriptions),1) AS conversion_percentage
FROM next_plan_cte
WHERE next_plan IS NOT NULL
  AND plan_id = 0
GROUP BY next_plan
ORDER BY next_plan;
``` 

**Q7: What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?**

```sql select b.plan_name , count(distinct a.customer_id)
from subscriptions as a
inner join plans as b on a.plan_id = b.plan_id
where a.start_date = '2020-12-31'
group by b.plan_name
```
**Q8: How many customers have upgraded to an annual plan in 2020?**

 ```sql select count(distinct customer_id) from subscriptions
 where plan_id = 3 and start_date < '2020-12-31' 
 ```
 
 
 **Q9: How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?**
  
  ```sql with trail_plan as  (
 select customer_id, start_date as trai_date
 from subscriptions
 where plan_id = 0),
 
annual_plan as (
 select customer_id , start_date as annual_date
 from subscriptions  
 where plan_id = 3)
 
 

 select * from (select avg(annual_date - trai_date) as count_ from trail_plan as a
 join annual_plan as b on a.customer_id = b.customer_id
)x
 order by count_
 ```
 
 
 
 
 
 **Q 10: Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)**
 
 
 ```sql
 WITH TEMP_A AS
(select customer_id,start_date from subscriptions where plan_id=3),
TEMP_B AS
(SELECT CUSTOMER_ID, START_DATE FROM SUBSCRIPTIONS WHERE PLAN_ID=0),
temp_c as (SELECT TEMP_A.customer_id, datediff(TEMP_A.START_DATE,TEMP_B.START_DATE) as diff_of_days FROM TEMP_A
INNER JOIN TEMP_B ON TEMP_A.CUSTOMER_ID=TEMP_B.CUSTOMER_ID)
select concat_ws('-',30 * floor(diff_of_days / 30) ,30 * (floor(diff_of_days / 30) + 1), "days") as range_days, count(customer_id)
from temp_c
group by 1
order by range_days
```
**Q11: How many customers downgraded from a pro monthly to a basic monthly plan in 2020?**

```sql
WITH next_plan_cte AS (
 SELECT
  customer_id,
  plan_id,
  start_date,
  LEAD(plan_id, 1) OVER(PARTITION BY customer_id ORDER BY plan_id) as next_plan
FROM subscriptions
)

SELECT
  COUNT(*) AS downgraded
FROM next_plan_cte
WHERE start_date <= '2020-12-31'
  AND plan_id = 2
  AND next_plan = 1;
  ```