## Digital Analysis

**Q1: How many users are there?**
  
 ```sql 
  SELECT
  count(DISTINCT user_id) AS user_count
FROM users
 ```

 **Q2: How many cookies does each user have on average?**
 
 ```sql
 SELECT
  ROUND(AVG(cookie_id_count),0) AS avg_cookie_id
FROM (
  SELECT
    user_id,
    COUNT(cookie_id) AS cookie_id_count
  FROM users
  GROUP BY user_id)x
  ```

**3. What is the unique number of visits by all users per month?**

 ```sql
  SELECT
  MONTH(event_time),
  COUNT(DISTINCT visit_id) AS unique_visit_count
FROM events
GROUP BY MONTH(event_time)
```

**4. What is the number of events for each event type?**
```sql
SELECT
  event_type,
  COUNT(*) AS event_count
FROM events
GROUP BY event_type
ORDER BY event_type
```

**5. What is the percentage of visits which have a purchase event?**

```sql
SELECT
  100 * COUNT(DISTINCT e.visit_id)/
    (SELECT COUNT(DISTINCT visit_id) FROM clique_bait.events) AS percentage_purchase
FROM events AS e
JOIN event_identifier AS ei
  ON e.event_type = ei.event_type
WHERE ei.event_name = 'Purchase'
```


**6. What is the percentage of visits which view the checkout page but do not have a purchase event?**

```sql
select 100-((sum(purchase)/sum(checkout))*100 ) as percentage_checkout_view_with_no_purchase from (SELECT
  visit_id,
  sum(CASE WHEN event_type = 1 AND page_id = 12 THEN 1 ELSE 0 END) AS checkout,
  sum(CASE WHEN event_type = 3  THEN 1 ELSE 0 END) AS purchase
FROM events
GROUP BY visit_id
order by visit_id)x
```


**Q7: What are the top 3 pages by number of views?**


```sql
SELECT
 ph.page_name,
 COUNT(*) AS page_views
FROM events AS e
JOIN page_hierarchy AS ph
  ON e.page_id = ph.page_id
WHERE e.event_type = 1
GROUP BY ph.page_name
ORDER BY page_views DESC
LIMIT 3
```



**Q8: What is the number of views and cart adds for each product category?**

```sql
SELECT
  ph.product_category,
  SUM(CASE WHEN e.event_type = 1 THEN 1 ELSE 0 END) AS page_views,
  SUM(CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) AS cart_adds
FROM events AS e
JOIN page_hierarchy AS ph
  ON e.page_id = ph.page_id
WHERE ph.product_category IS NOT NULL
GROUP BY ph.product_category
ORDER BY page_views DESC
```


**Q9: What are the top 3 products by purchases?**

```sql

SELECT
  ph.product_category, count(*) as purchase_count
FROM events AS e
JOIN page_hierarchy AS ph
  ON e.page_id = ph.page_id
WHERE ph.product_category IS NOT NULL and e.event_type = 1
GROUP BY ph.product_category
order by purchase_count desc
```


## product funnel analysis



**How many times was each product viewed?**
**How many times was each product added to cart?**
**How many times was each product added to a cart but not purchased (abandoned)?**
**How many times was each product purchased?**



Solution for all the question in a combined query

```sql

with prod_view_cart as (
select e.visit_id, p_h.product_id,p_h.page_name as products, p_h.product_category as pc, e.event_type, e.page_id,
sum(case when event_type = 1 then 1 else 0 end) as page_view,
sum(case when event_type = 3 then 1 else 0 end) as cart_add
from events as e inner join
page_hierarchy as p_h
on e.page_id = p_h.page_id
where p_h.product_id is not null
group by e.visit_id, p_h.product_id, e.event_type, e.page_id
)
,

 purchase_table as(
 select distinct visit_id from
 prod_view_cart
 where event_type = 1
 )
 
 ,
 
 
 combined as (
 select vc.visit_id, vc.product_id, vc.event_type, vc.page_id, vc.page_view, vc.cart_add, vc.products, pc
 , case when pt.visit_id is not null then 1 else 0 end as purchase
 from prod_view_cart as vc
 left join purchase_table as pt
 on pt.visit_id = vc.visit_id
 )
 
 ,
 
 product_new_info as(
 select products, pc, sum(page_view) as views , sum(cart_add) as cart_add,
 sum(case when cart_add = 1 and purchase = 0 then 1 else 0 end) as abandoned,
 sum(case when cart_add = 1 and purchase = 1 then 1 else 0 end) as purhcase
 from combined
 group by products, pc
 )
 
 select * from product_new_info
 
```

**Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.**


In the above table remove product name then we will get the query for this



**Q1: Which product had the most views, cart adds and purchases?**
```sql
select max(views) from product_new_info
```


**Q2: Which product was most likely to be abandoned?**
 
 ```sql
select product, count(abandoned) from product_new_info
group by product
```


**Q3: Which product had the highest view to purchase percentage?**

```sql
SELECT
    product
  pc,
  ROUND(100 * purchase/views,2) AS purchase_per_view_percentage
FROM product_new_info
ORDER BY purchase_per_view_percentage DES
```
 
**Q4:What is the average conversion rate from view to cart add?**
 
 ```sql
SELECT
  ROUND(100*AVG(cart_adds/views),2) AS avg_view_to_cart_add_conversion
FROM product_info
```




**Q5:What is the average conversion rate from cart add to purchase?**
```sql
SELECT
  ROUND(100*AVG(cart_adds/views),2) AS avg_view_to_cart_add_conversion,
  ROUND(100*AVG(purchases/cart_adds),2) AS avg_cart_add_to_purchases_conversion_rate
FROM product_info
```