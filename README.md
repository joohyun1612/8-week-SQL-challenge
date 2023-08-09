# 8-week-SQL-challenge
# 🍜 Case Study #2: Pizza Runner
<img src="https://8weeksqlchallenge.com/images/case-study-designs/2.png" alt="Image" width="500" height="520">

## 📚 Table of Contents
- [Business Task](#business-task)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Question and Solution](#question-and-solution)

Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-2/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

***

## Entity Relationship Diagram

![image](https://github.com/katiehuangx/8-Week-SQL-Challenge/assets/81607668/78099a4e-4d0e-421f-a560-b72e4321f530)

***
**Create temporary table**
CREATE TEMP TABLE customer_orders_temp AS
SELECT 
  order_id, 
  customer_id, 
  pizza_id, 
  CASE
	  WHEN exclusions IS null OR exclusions LIKE 'null' THEN ' '
	  ELSE exclusions
	  END AS exclusions,
  CASE
	  WHEN extras IS NULL or extras LIKE 'null' THEN ' '
	  ELSE extras
	  END AS extras,
	order_time
FROM pizza_runner.customer_orders;

CREATE TEMP TABLE runner_orders_temp AS
SELECT 
  order_id, 
  runner_id,  
  CASE
	  WHEN pickup_time LIKE 'null' THEN ' '
	  ELSE pickup_time
	  END AS pickup_time,
  CASE
	  WHEN distance LIKE 'null' THEN '0'
	  WHEN distance LIKE '%km' THEN TRIM('km' from distance)
	  ELSE distance 
    END AS distance,
  CASE
	  WHEN duration LIKE 'null' THEN ' '
	  WHEN duration LIKE '%mins' THEN TRIM('mins' from duration)
	  WHEN duration LIKE '%minute' THEN TRIM('minute' from duration)
	  WHEN duration LIKE '%minutes' THEN TRIM('minutes' from duration)
	  ELSE duration
	  END AS duration,
  CASE
	  WHEN cancellation IS NULL or cancellation LIKE 'null' THEN ' '
	  ELSE cancellation
	  END AS cancellation
FROM pizza_runner.runner_orders;

**1. How many pizzas were ordered?**

SELECT COUNT(*)
FROM pizza_runner.customer_orders;

**2. How many unique customer orders were made?**

SELECT COUNT(DISTINCT order_id) AS unique_customer_orders
FROM pizza_runner.customer_orders;

**3. How many successful orders were delivered by each runner?**

SELECT runner_id, COUNT (order_id) AS successful_orders
FROM pizza_runner.runner_orders
WHERE cancellation is NULL
GROUP BY runner_id;

**4. How many of each type of pizza was delivered?**

SELECT p.pizza_name, COUNT(p.pizza_name) AS amount_pizza
FROM pizza_runner.pizza_names AS p
JOIN pizza_runner.customer_orders AS c
ON p.pizza_id = c.pizza_id
JOIN pizza_runner.runner_orders AS r
ON c.order_id = r.order_id
WHERE r.cancellation IS NULL
GROUP BY pizza_name;

**5. How many Vegetarian and Meatlovers were ordered by each customer?**

SELECT c.customer_id, p.pizza_name, COUNT(p.pizza_name)
FROM pizza_runner.pizza_names AS p
JOIN pizza_runner.customer_orders AS c
ON p.pizza_id = c.pizza_id
GROUP BY customer_id, pizza_name
ORDER BY customer_id;

**6. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)**

SELECT EXTRACT(WEEK from registration_date), COUNT(runner_id) AS amount_signed_up
FROM pizza_runner.runners
GROUP BY 1;

**7. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**

WITH time_taken_cte AS

(
  
  SELECT 
    c.order_id, 
    c.order_time, 
    r.pickup_time,
  	r.runner_id,
    EXTRACT(MINUTE FROM r.pickup_time::timestamp-c.order_time::timestamp) AS pickup_minutes
  FROM pizza_runner.customer_orders AS c
  JOIN pizza_runner.runner_orders AS r
    ON c.order_id = r.order_id
  WHERE r.cancellation IS NULL
  GROUP BY c.order_id, c.order_time, r.pickup_time, r.runner_id

)

SELECT 
  runner_id, AVG(pickup_minutes) AS avg_pickup_minutes
FROM time_taken_cte
WHERE pickup_minutes > 1
GROUP BY runner_id;

**8. Is there any relationship between the number of pizzas and how long the order takes to prepare?**

WITH time_taken_cte AS

(
  SELECT 
    c.order_id,
  	COUNT(c.pizza_id) AS numberofpizza,
    c.order_time, 
    r.pickup_time,
    EXTRACT(MINUTE FROM r.pickup_time::timestamp-c.order_time::timestamp) AS pickup_minutes
  FROM pizza_runner.customer_orders AS c
  JOIN pizza_runner.runner_orders AS r
    ON c.order_id = r.order_id
  WHERE r.cancellation IS NULL
  GROUP BY c.order_id, c.order_time, r.pickup_time
)

SELECT CORR (pickup_minutes, numberofpizza)
from time_taken_cte;

**9. What is the successful delivery percentage for each runner?**

SELECT 
  runner_id, 
  ROUND(100 * SUM(
    CASE WHEN cancellation is Null THEN 0
    ELSE 1 END) / COUNT(*), 0) AS success_perc
FROM pizza_runner.runner_orders
GROUP BY runner_id
ORDER BY runner_id;

**10. What was the most commonly added extra?**

WITH toppings_cte AS (
SELECT
  pizza_id,
  REGEXP_SPLIT_TO_TABLE(toppings, '[,\s]+')::INTEGER AS topping_id
FROM pizza_runner.pizza_recipes)

SELECT 
  t.topping_id, pt.topping_name, 
  COUNT(t.topping_id) AS topping_count
FROM toppings_cte t
INNER JOIN pizza_runner.pizza_toppings pt
  ON t.topping_id = pt.topping_id
GROUP BY t.topping_id, pt.topping_name
LIMIT 1;

**11. What was the most common exclusion?**

WITH extra_count_cte AS
  (SELECT REGEXP_SPLIT_TO_TABLE(exclusions, '[,\s]+')::INTEGER AS extra_topping,
          count(*) AS purchase_counts
   FROM pizza_runner.customer_orders
   WHERE exclusions IS NOT NULL
   GROUP BY exclusions)

SELECT topping_name,
       purchase_counts
FROM extra_count_cte
INNER JOIN pizza_runner.pizza_toppings t ON extra_count_cte.extra_topping = t.topping_id
LIMIT 3;

**12. If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?**

SELECT CONCAT('$', SUM(CASE
                           WHEN pizza_id = 1 THEN 12
                           ELSE 10
                       END)) AS total_revenue
FROM pizza_runner.customer_orders
INNER JOIN pizza_runner.pizza_names USING (pizza_id)
INNER JOIN pizza_runner.runner_orders USING (order_id)
WHERE cancellation IS NULL;

**13. What if there was an additional $1 charge for any pizza extras? Add cheese is $1 extra**

SELECT CONCAT('$', topping_revenue+ pizza_revenue) AS total_revenue
FROM
  (SELECT SUM(CASE
                  WHEN pizza_id = 1 THEN 12
                  ELSE 10
              END) AS pizza_revenue,
          sum(topping_count) AS topping_revenue
   FROM
     (SELECT *,
             length(extras) - length(replace(extras, ",", ""))+1 AS topping_count
      FROM pizza_runner.customer_orders
      INNER JOIN pizza_runner.pizza_names USING (pizza_id)
      INNER JOIN pizza_runner.runner_orders USING (order_id)
      WHERE cancellation IS NULL
      ORDER BY order_id)t1) t2;

**14. The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset - generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.**

DROP TABLE IF EXISTS runner_rating;

CREATE TABLE runner_rating (order_id INTEGER, rating INTEGER, review VARCHAR(100)) ;

-- Order 6 and 9 were cancelled

INSERT INTO runner_rating

VALUES ('1', '1', 'Really bad service'),
       ('2', '1', NULL),
       ('3', '4', 'Took too long..."),
       ('4', '1','Runner was lost, delivered it AFTER an hour. Pizza arrived cold' ),
       ('5', '2', ''Good service'),
       ('7', '5', 'It was great, good service and fast'),
       ('8', '2', 'He tossed it on the doorstep, poor service'),
       ('10', '5', 'Delicious!, he delivered it sooner than expected too!');


SELECT *
FROM runner_rating;
