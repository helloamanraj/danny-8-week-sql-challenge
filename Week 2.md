                            A.Pizza Metrics



**Q1: How many pizzas were ordered?**
```sql
select count(order_id) from customer_orders
```

**Q2: How many unique customer orders were made?**
```sql
select count(distinct order_id) from customer_orders
```
**Q3: How many successful orders were delivered by each runner?**

```sql
select runner_id, count(order_id) from runner_orders
where distance != 0
group by runner_id
```

**Q4: How many of each type of pizza was delivered?**

```sql
select pizza_id, count(c.order_id) from customer_orders as c
inner join runner_orders as r
on c.order_id = r.order_id
where r.distance != 0
group by pizza_id
```

**Q5: How many Vegetarian and Meatlovers were ordered by each customer?**

```sql
select customer_id, p.pizza_name, count(c.pizza_id) from customer_orders
as c
inner join pizza_names as p on c.pizza_id = p.pizza_id
group by customer_id, c.pizza_id
order by customer_id
```
**Q6: What was the maximum number of pizzas delivered in a single order?**
 
 ```sql
 select order_id, max(pizza_per_order) from (select order_id, count(pizza_id) as pizza_per_order
 from customer_orders
 group by order_id) x
 ```

***Q7: For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**

New table is created with the modification 
```sql
(create table updated_customer_orders as
select order_id, customer_id, pizza_id, nullif(exclusions, '') as ud_exclusions , nullif(extras, '') ud_extras, order_time
from customer_orders
select * from updated_customer_orders
where (ud_extras is null or ud_extras = "null") and ( ud_exclusions is null or ud_exclusions = "null")

create table updated_customer_orders_1 as
select order_id, customer_id, pizza_id, nullif(ud_exclusions, 'null') as udp_exclusions , nullif(ud_extras, 'null') udp_extras, order_time
from updated_customer_orders


select * from updated_customer_orders_1


select customer_id, pizza_id,
sum(case when udp_exclusions is not null or udp_extras is not null then 1 else 0 end ) as with_changes,
sum(case when udp_exclusions is null and udp_extras is null then 1 else 0 end) as no_changes
from updated_customer_orders_1 as a
inner join runner_orders as b
on a.order_id = b.order_id
where b.distance != 0
group by a.customer_id, pizza_id

select *, b.distance from updated_customer_orders_1 as a
inner join runner_orders as b
on a.order_id = b.order_id
where b.distance != 0
```

**Q8: How many pizzas were delivered that had both exclusions and extras?**

```sql
select customer_id,
sum(case when ( udp_exclusions is not null and  udp_extras is not null) then 1 else 0 end ) as exclu_and_extra
from updated_customer_orders_1 as a
inner join runner_orders as b
on a.order_id = b.order_id
where b.distance != 0 and sum(case when ( udp_exclusions is not null and  udp_extras is not null) then 1 else 0 end ) !=0
group by a.customer_id
```








**Q9: What was the total volume of pizzas ordered for each hour of the day?**


```sql
select hour(order_time) as week_day,  COUNT(order_id) AS total_pizzas_ordered_hourly
FROM customer_orders
group by week_day



select *, (ord_per_day/24) as ord_pizza_each_hour from (select count(order_id) as ord_per_day, day(order_time) from customer_orders
group by day(order_time)
) x
```

**Q10: What was the volume of orders for each day of the week?**

```sql
select dayname(order_time) as week_day,  COUNT(order_id) AS total_pizzas_ordered
FROM customer_orders
group by week_day
```





                     B Runner and Customer Experience
                                                     
                                                     
**Q1: How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**
```sql
select week(registration_date,5) ,count(runner_id)
from runners
group by week(registration_date,5)
```

**Q2: What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**

```sql
select avg(pickup_minutes) as Avgtime_in_minutes from (
select a.order_id,
    a.order_time,
    b.pickup_time,
TIMESTAMPDIFF(minute, a.order_time, b.pickup_time) AS pickup_minutes
from customer_orders as a
inner join runner_orders as b on a.order_id = b.order_id
where b.distance != 0
group by a.order_id
    ) x
```

**Q3: Is there any relationship between the number of pizzas and how long the order takes to prepare?**

```sql
select pizza_count, avg(prep_time_minutes) from
(
select a.order_id, count(a.pizza_id) as pizza_count, TIMESTAMPDIFF(minute, a.order_time, b.pickup_time) as prep_time_minutes
from customer_orders as a inner join runner_orders as b
 on a.order_id = b.order_id
 where b.distance != 0
group by a.order_id
) x
where x.prep_time_minutes > 1
group by x.pizza_count
```
**Q4: What was the average distance travelled for each customer?**

```sql
select x.customer_id,  avg(dis) from (
select a.customer_id, b.distance as dis from customer_orders as a
inner join runner_orders
as b
on a.order_id = b.order_id
where distance != 0
group by customer_id
)x
group by x.customer_id
```
**Q5: What was the difference between the longest and shortest delivery times for all orders?**

```sql
select y.mx- y.mn as Max_min_time_diff from (select x.order_id, max(x.duration) as mx, min(x.duration) as mn
from (select order_id, duration
from runner_orders
where duration != 'null'
)
x
)
y
```

**Q6: What was the average speed for each runner for each delivery and do you notice any trend for these values?**

```sql
select x.*, (distance / hour) as avg_speed from
(select runner_id, distance, round((duration/60),1) as hour from
runner_orders
where distance != 'null'
) x
```

**Q7: What is the successful delivery percentage for each runner?**


```sql
select runner_id,
(100*sum(case when distance = 0 then 0
else 1 end)/ count(*)) as SUCC_PERC
from runner_orders
group by runner_id

```
```sql
select weekday(order_time), count(order_id) as total_pizzas_ordered
from customer_orders
group by weekday(order_time)

```



                                    C. ingredient optimisation


**Q1: What are the standard ingredients for each pizza?**


New table is created for creatinf the transpose 
```sql
create temporary table numbers as (
  select 1 as n
  union select 2 as n
  union select 3 as n
  union select 4 as n
  union select 5 as n
  union select 6 as n
  union select 7 as n
  union select 8 as n
  union select 9 as n
  union select 10 as n
)
```
#Make the table into 1NF form

```sql
create table updated_pizza_recipes as
select
  pizza_id,
  substring_index(
    substring_index(toppings, ',', n),
    ',',
    -1
  ) as separated_topping
from pizza_recipes
join numbers
  on char_length(toppings)
    - char_length(replace(toppings, ',', ''))
    >= n - 1
order by pizza_id    




select * from updated_pizza_recipes

SELECT *
FROM updated_pizza_recipes
WHERE separated_topping IN (
    SELECT separated_topping
    FROM updated_pizza_recipes
    GROUP BY separated_topping
    HAVING COUNT(distinct pizza_id) > 1
)

```



***Q2: What was the most commonly added extra?**

```sql
(select u.separated_topping, p.topping_name from updated_pizza_recipes as u
inner join pizza_toppings as p on u.separated_topping = p.topping_id
where u.separated_topping in u.pizza_id) as x

select a.separated_topping, count(a.separated_topping) from updated_pizza_recipes as a
right join updated_customer_orders_1 as b on a.separated_topping = b.udp_extras


select separated_topping, count(separated_topping) from updated_pizza_recipes
where separated_topping in (1,5,4)
group by separated_topping

(select u.udp_extras, p.topping_name from updated_customer_orders_1 as u
left join pizza_toppings as p on u.udp_extras = p.topping_id
where u.udp_extras is not null
)
```


***Q3: What was the most common exclusion?**

```sql
select customer_id, udp_exclusions from updated_customer_orders_1
where udp_exclusions is not null
```

**Q4: Generate an order item for each record in the customers_orders table in the format of one of the following:**


```sql
select a.*, pizza_name from updated_customer_orders_1 as
a inner join pizza_names as b on a.pizza_id = b.pizza_id

```
Q5:






**Q6: What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?**

```sql
select * from updated_pizza_recipes


select a.*, b.separated_topping, count(a.order_id) as most_imp_ingredient from updated_customer_orders_1 as a
inner join updated_pizza_recipes as b on a.pizza_id = b.pizza_id
group by b.separated_topping


select a.*, b.separated_topping from updated_customer_orders_1 as a
inner join updated_pizza_recipes as b on a.pizza_id = b.pizza_id
inner join runner_orders as c on a.order_id = c.order_id
where c.distance != 0
order by a.order_id
```