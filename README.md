Danny seriously loves Japanese food so in the beginning of 2021, he decides to embark upon a risky venture and opens up a cute little restaurant that sells his 3 favourite foods: sushi, curry and ramen.

Danny’s Diner is in need of your assistance to help the restaurant stay afloat - the restaurant has captured some very basic data from their few months of operation but have no idea how to use their data to help them run the business.

Problem Statement
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. Having this deeper connection with his customers will help him deliver a better and more personalised experience for his loyal customers.

He plans on using these insights to help him decide whether he should expand the existing customer loyalty program - additionally he needs help to generate some basic datasets so his team can easily inspect the data without needing to use SQL.

Danny has provided you with a sample of his overall customer data due to privacy issues - but he hopes that these examples are enough for you to write fully functioning SQL queries to help him answer his questions!

Danny has shared with you 3 key datasets for this case study:

sales
menu
members

What is the total amount each customer spent at the restaurant?
How many days has each customer visited the restaurant?
What was the first item from the menu purchased by each customer?
What is the most purchased item on the menu and how many times was it purchased by all customers?
Which item was the most popular for each customer?
Which item was purchased first by the customer after they became a member?
Which item was purchased just before the customer became a member?
What is the total items and amount spent for each member before they became a member?
If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
**Schema (PostgreSQL v13)**

    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

---

**Query #1**

    SELECT
      	customer_id,sum(price) as total_sales
    FROM dannys_diner.menu
    inner join dannys_diner.sales using (product_id)
    group by customer_id
    order by customer_id;

| customer_id | total_sales |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

---
**Query #2**

    select customer_id,count(distinct(order_date))
    from dannys_diner.sales
    group by customer_id;

| customer_id | count |
| ----------- | ----- |
| A           | 4     |
| B           | 6     |
| C           | 2     |

---
**Query #3**

    with order_sales as
    (
    select product_name,customer_id,order_date,
    dense_rank() over  (partition by customer_id order by order_date) as rank
    FROM dannys_diner.menu
    inner join dannys_diner.sales using (product_id))
    select customer_id,product_name
    from order_sales
    where rank = 1
    group by customer_id,product_name;

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

---
**Query #4**

    select(count(product_id) )as most_purchased, product_name
    FROM dannys_diner.menu
    inner join dannys_diner.sales using (product_id)
    group by product_name
    order by most_purchased desc;

| most_purchased | product_name |
| -------------- | ------------ |
| 8              | ramen        |
| 4              | curry        |
| 3              | sushi        |

---
**Query #5**

    with most_favourite_item as
    (
    select product_name,customer_id,(count(product_id)) as order_count,
      dense_rank()over(partition by customer_id order by customer_id desc) as rank
      FROM dannys_diner.menu
    inner join dannys_diner.sales using (product_id)
    group by customer_id,product_name
    )
      select customer_id,product_name,order_count
    from most_favourite_item
    where rank = 1;

| customer_id | product_name | order_count |
| ----------- | ------------ | ----------- |
| C           | ramen        | 3           |
| B           | curry        | 2           |
| B           | sushi        | 2           |
| B           | ramen        | 2           |
| A           | curry        | 2           |
| A           | sushi        | 1           |
| A           | ramen        | 3           |

---
**Query #6**

    with member_sales_cte as
    (
    select customer_id,product_id,order_date,join_date,
      dense_rank() over (partition by customer_id order by order_date) as rank
      from dannys_diner.sales
      join dannys_diner.members using (customer_id)
      where order_date >= join_date
      )
      select product_name,order_date,customer_id
      from member_sales_cte
      join dannys_diner.menu using (product_id)
      where rank=1
      order by customer_id;

| product_name | order_date               | customer_id |
| ------------ | ------------------------ | ----------- |
| curry        | 2021-01-07T00:00:00.000Z | A           |
| sushi        | 2021-01-11T00:00:00.000Z | B           |

---
**Query #7**

    with member_sales_cte as
    (
    select customer_id,product_id,order_date,join_date,
      dense_rank() over (partition by customer_id order by order_date) as rank
      from dannys_diner.sales
      join dannys_diner.members using (customer_id)
      where order_date < join_date
      )
      select product_name,order_date,customer_id
      from member_sales_cte
      join dannys_diner.menu using (product_id)
      where rank=1;

| product_name | order_date               | customer_id |
| ------------ | ------------------------ | ----------- |
| sushi        | 2021-01-01T00:00:00.000Z | A           |
| curry        | 2021-01-01T00:00:00.000Z | B           |
| curry        | 2021-01-01T00:00:00.000Z | A           |

---
**Query #8**

    select customer_id,count(distinct(product_id)) as unique_id, sum(price) as total_sales
    from dannys_diner.sales
    join dannys_diner.menu using (product_id)
    join dannys_diner.members using (customer_id)
    where order_date < join_date
    group by  customer_id;

| customer_id | unique_id | total_sales |
| ----------- | --------- | ----------- |
| A           | 2         | 25          |
| B           | 2         | 40          |

---
**Query #9**

    WITH price_points AS
     (
     SELECT *, 
     CASE
      WHEN product_id = 1 THEN price * 20
      ELSE price * 10
      END AS points
     FROM dannys_diner.menu
     )
     SELECT s.customer_id, SUM(p.points) AS total_points
    FROM price_points AS p
    JOIN dannys_diner.sales AS s
     ON p.product_id = s.product_id
    GROUP BY s.customer_id;

| customer_id | total_points |
| ----------- | ------------ |
| B           | 940          |
| C           | 360          |
| A           | 860          |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/2rM8RAnq7h5LLDTzZiRWcd/138)
