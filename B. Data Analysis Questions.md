# 🥑 Case Study #3 - Foodie-Fi

## 🎞 Solution - B. Data Analysis Questions
````sql
SELECT customer_id,a.plan_id, plan_name, price, start_date
INTO sub_plan
FROM subscriptions a
LEFT join plans b
ON a.plan_id =b.plan_id
SELECT * FROM sub_plan
 ````
**Create a table named `sub_plan` is joined by 2 table `subcriptions` and `plans`**

### 1. How many customers has Foodie-Fi ever had?

To find the number of Foodie-Fi's unique customers, I use `DISTINCT` and wrap `COUNT` around it.

````sql
SELECT COUNT(DISTINCT customer_id) CUSTOMER_CNT
FROM sub_plan;
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227834393-5b247f3e-6c6f-4fa1-84a3-d8befaf93fcb.png)

- Foodie-Fi has 1,000 unique customers.

### 2. What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value

````sql
SELECT DATEPART(month,start_date) month,
	   DATENAME(month,start_date) month_name,
	   COUNT(*) trial_plan_cnt
FROM sub_plan
WHERE plan_id=0
GROUP BY  DATEPART(month,start_date),
	      DATENAME(month,start_date)
ORDER BY DATEPART(month,start_date);
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227834574-56a32995-7db3-4f54-96c1-c3d67e7c3651.png)

- March has the highest number of trial plans, whereas February has the lowest number of trial plans.

### 3. What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name.

Question is asking for the number of plans for start dates occurring on 1 Jan 2021 and after grouped by plan names.
- Filter plans with start_dates occurring on 2021–01–01 and after.
- Group and order results by plan.

_Note: Question calls for events occuring after 1 Jan 2021, however I ran the query for events in 2020 as well as I was curious with the year-on-year results._

````sql
SELECT plan_id,
	   plan_name as name_of_plan,
	   [2020],[2021]
FROM
(SELECT plan_id,
		plan_name,
		DATEPART(year,start_date) year_plan,
		COUNT(*) event_cnt
FROM sub_plan
GROUP BY plan_id,
		 plan_name,
	     DATEPART(year,start_date)) c
PIVOT
(Sum (event_cnt)
for year_plan in ([2020],[2021])) d
ORDER BY plan_id;
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227834724-00deacde-e83a-4eb5-afa8-0dae6b7fe857.png)

- There were 0 customer on trial plan in 2021. Does it mean that there were no new customers in 2021, or did they jumped on basic monthly plan without going through the 7-week trial?
- We should also look at the data and look at the customer proportion for 2020 and 2021.

### 4. What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

I like to write down the steps and breakdown the questions into parts.

**Steps:**
- Find the number of customers who churned.
- Find the percentage of customers who churned and round it to 1 decimal place.
- What's the churned plan_id? Filter to 4

````sql
SELECT 
	COUNT(DISTINCT customer_id) as churned_cnt,
	ROUND(COUNT(DISTINCT customer_id)*100/(SELECT COUNT(DISTINCT customer_id) FROM sub_plan),1) AS percentage
FROM sub_plan
WHERE plan_id=4;
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227834859-bd5f277b-925e-4274-a415-2a517429c0ac.png)

- There are 307 customers who have churned, which is 30.7% of Foodie-Fi customer base.

### 5. How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?

In order to identify which customer churned straight after the trial plan, I realized that customers have had `from trial to churn` lead to total `price` equal 0 

````sql
SELECT
	(SELECT COUNT (*) 
	FROM 
		(
		SELECT customer_id 
		FROM sub_plan 
		GROUP BY customer_id 
		HAVING SUM(price)=0.00)c
		) churn_straight_after_initial_free_trial,
	ROUND(100*(SELECT COUNT (*) FROM
		(SELECT customer_id
		 FROM sub_plan 
		 GROUP BY customer_id 
		 HAVING SUM(price)=0.00) c) / COUNT(DISTINCT customer_id),2)  churn_percentage
from sub_plan
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227835183-7e439497-76f6-4713-a205-19114178ff02.png)

- There are 92 customers who churned straight after the initial free trial which is at 9% of entire customer base.

### 6. What is the number and percentage of customer plans after their initial free trial?

Question is asking for number and percentage of customers who converted to becoming paid customer after the trial. 

**Steps:**
- Find customer's next plan which is located in the next row using `LEAD()`. Run the `next_plan_cte` separately to view the next plan results and understand how `LEAD()` works.
- Filter for `non-null next_plan`. Why? Because a next_plan with null values means that the customer has churned. 
- Filter for `plan_id = 0` as every customer has to start from the trial plan at 0.

````sql
-- To retrieve next plan's start date located in the next row based on current row
WITH next_plan_cte as	
	(SELECT customer_id, plan_id, 
	LEAD(plan_id, 1) OVER(PARTITION BY customer_id ORDER BY plan_id) as next_plan
	FROM sub_plan)
SELECT next_plan,
	   COUNT(*) convertions,
	   ROUND(100*CAST(COUNT(*) as float)/(SELECT COUNT(DISTINCT customer_id) FROM sub_plan),2) convertion_percentage
FROM next_plan_cte
WHERE next_plan IS NOT NULL AND plan_id=0
GROUP BY next_plan
ORDER BY next_plan;
````
**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227835413-36d9d73e-9862-4fff-9457-fdb95031420c.png)

- More than 80% of customers are on paid plans with small 3.7% on plan 3 (pro annual $199). Foodie-Fi has to strategize on their customer acquisition who would be willing to spend more.

### 7. What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

````sql
-- To retrieve next plan's start date located in the next row based on current row
WITH next_plan AS(
SELECT 
  customer_id, 
  plan_id, 
  start_date,
  LEAD(start_date, 1) OVER(PARTITION BY customer_id ORDER BY start_date) as next_date
FROM sub_plan
WHERE start_date <= '2020-12-31'),
customer_breakdown AS (
  SELECT plan_id, COUNT(DISTINCT customer_id) AS customers
    FROM next_plan
    WHERE (next_date IS NOT NULL AND (start_date < '2020-12-31' AND next_date > '2020-12-31'))
      OR (next_date IS NULL AND start_date < '2020-12-31')
    GROUP BY plan_id)
SELECT plan_id, customers, 
  ROUND(100 * CAST(customers AS FLOAT) / (
    SELECT COUNT(DISTINCT customer_id) 
    FROM sub_plan),1) AS percentage
FROM customer_breakdown
GROUP BY plan_id, customers
ORDER BY plan_id
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227835517-b491b316-9c5a-4a70-a7da-413ffe9149fb.png)

### 8. How many customers have upgraded to an annual plan in 2020?

````sql
SELECT COUNT(DISTINCT customer_id) as annual_plan_cnt
FROM sub_plan
WHERE YEAR(start_date)=2020 and plan_id=3
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227835601-8c351b44-6e0f-4326-b98a-8c4d0af0bad0.png)

- 195 customers upgraded to an annual plan in 2020.

### 9. How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?

````sql
WITH trial_date_cte AS 
(
	SELECT customer_id, start_date as trial_date
	FROM sub_plan
	WHERE plan_id = 0 or plan_id=3
),
annual_date_cte as
(
	SELECT customer_id, start_date as annual_date
	FROM sub_plan
	WHERE plan_id = 3
)
SELECT avg(datediff(day,trial_date,annual_date)) as avg_days
FROM trial_date_cte a 
JOIN annual_date_cte b
ON a.customer_id = b.customer_id
WHERE trial_date != annual_date;;
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227835705-4a0010c3-f722-4e9c-b61b-a610f39e5d21.png)

- On average, it takes 104 days for a customer to upragde to an annual plan from the day they join Foodie-Fi.

### 10. Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)


### 11. How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

````sql
-- To retrieve next plan's start date located in the next row based on current row
SELECT count(*) downgraded_customder_cnt
FROM
(SELECT customer_id, plan_id,
		LEAD(plan_id,1) OVER (partition by customer_id order by plan_id) next_plan
FROM sub_plan
WHERE YEAR(start_date)=2020) downgrade
WHERE plan_id=2 AND next_plan=1;
````

**Answer:**

![image](https://user-images.githubusercontent.com/125182638/227835802-f27edc1a-7b55-4f0f-ba6d-ad5683c873ad.png)

- No customer has downgrade from pro monthly to basic monthly in 2020.
