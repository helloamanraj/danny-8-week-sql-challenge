**Q1: What is the total amount each customer spent at the restaurant?**

```sql
  
Select distinct customer_id, sum(price) from sales as s
inner join menu as m on s.product_id = m.product_id
group by customer_id
```

**Q2:How many days has each customer visited the restaurant?**
```sql
select customer_id, count(distinct order_date) as days_visited from sales
group by customer_id
```

**Q3: What was the first item from the menu purchased by each customer?**


```sql 
select x.* from
(select customer_id, order_date, Product_name, dense_rank() over (partition by customer_id order by order_date) as rnk from sales as s inner join menu as m on s.product_id = m.product_id) x
where rnk= 1
group by customer_id, product_name
```



**Q4: What is the most purchased item on the menu and how many times was it purchased by all customers?**

```sql
select product_name, max(purchase_count) from (select product_name, count(s.product_id) as purchase_count from sales as s inner join menu as m on s.product_id = m.product_id
group by product_name ) x
```
**Q5: Which item was the most popular for each customer?**

```sql
select y.* from (select customer_id, product_name, dense_rank() over (partition by customer_id order by purchase_count desc) as rnk,
purchase_count
from (select customer_id, product_name, count(s.product_id) as purchase_count from sales as s inner join menu as m on s.product_id = m.product_id
group by customer_id, product_name ) x)y
where rnk =1
```
**Q6: Which item was purchased first by the customer after they became a member?**

```sql
with cte as (
select s.customer_id, s.product_id, m.join_date, s.order_date, dense_rank() over(partition by customer_id order by order_date) as rnk from sales as s inner join members as m
on s.customer_id = m.customer_id
where s.order_date >= m.join_date)

select customer_id, product_name, order_date from cte as c inner join menu as m
on c.product_id = m.product_id
where rnk = 1
```

**Q7: Which item was purchased just before the customer became a member?**

```sql
with cte as (
select s.customer_id, s.product_id, m.join_date, s.order_date, dense_rank() over(partition by customer_id order by order_date) as rnk from sales as s inner join members as m
on s.customer_id = m.customer_id
where s.order_date <  m.join_date)

select customer_id, product_name, order_date from cte as c inner join menu as m
on c.product_id = m.product_id
where rnk = 1
```
**Q8: What is the total items and amount spent for each member before they became a member?**
```sql
select s.customer_id, sum(price) from sales as s inner join menu as m
on s.product_id = m.product_id
inner join members as mem on s.customer_id = mem.customer_id
where s.order_date < mem.join_date
group by customer_id
order by sum(price) desc
```

**Q9: If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?**
```sql
select customer_id, sum(points) from (select customer_id,
case when product_name = "sushi" then price*20
else price*10 end as points
from sales as s
inner join menu as m
on s.product_id = m.product_id)
x
group by customer_id
```
**Q10: In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?**

```sql

with cte as (
select *,  date_add(join_date, interval 6 day) as new_date
from members
where join_date <= "2021-01-31")

select s.customer_id,
sum(case when s.product_id =1 then m.price*20
when
 s.order_date between c.join_date and c.new_date then 20*m.price
else 10*m.price end) as earn_points
from cte as c
inner join sales as s on s.customer_id = c.customer_id
inner join menu as m on s.product_id = m.product_id
where s.order_date < "2021-01-31"
group by s.customer_id

```

 
 
 



-------------------------------

Bonus Question
Q1:
```sql
select s.customer_id, s.order_date, m.product_name, m.price,
case when mm.join_date > s.order_date then "N"
when mm.join_date <= s.order_date then "Y"

end as member

from sales as s
left join menu as m on s.product_id = m.product_id
left join members as mm on s.customer_id = mm.customer_id
```

Q2:

```sql
WITH summary_cte AS
(
   SELECT s.customer_id, s.order_date, m.product_name, m.price,
      CASE
      WHEN mm.join_date > s.order_date THEN 'N'
      WHEN mm.join_date <= s.order_date THEN 'Y'
      ELSE 'N' END AS member
   FROM sales AS s
   LEFT JOIN menu AS m
      ON s.product_id = m.product_id
   LEFT JOIN members AS mm
      ON s.customer_id = mm.customer_id
)

SELECT *, CASE
   WHEN member = 'N' then NULL
   ELSE
      RANK () OVER(PARTITION BY customer_id, member
      ORDER BY order_date) END AS ranking
FROM summary_cte;
```