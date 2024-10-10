# Coffee Expansion SQL Project


## Objective
The goal of this project is to analyze the sales data of  Coffee, a company that has been selling its products online since January 2023, and to recommend the top three major cities in India for opening new coffee shop locations based on consumer demand and sales performance.
## ERD

   ![](https://github.com/Thojer-DC/SQL_Project_6_Coffee_Expansion/blob/main/ERD.png)


## Key Questions

**Coffee Consumers Count** 
- 1. How many people in each city are estimated to consume coffee, given that 25% of the population does?
```SQL
   SELECT 
      city_name,
      population * 0.25 AS population_by_25percent
   FROM city
   ORDER BY 2 DESC;
```


**Total Revenue from Coffee Sales** 
- 2. What is the total revenue generated from coffee sales across all cities in the last quarter of 2023?
```SQL
   SELECT
      ci.city_name,
      SUM(s.total) AS total_revenue
   FROM sales AS s
   JOIN customer AS cus
      ON cus.customer_id = s.customer_id
   JOIN city AS ci
      ON ci.city_id = cus.city_id
   WHERE 
      EXTRACT(YEAR FROM s.sale_date) = 2023
      AND
      EXTRACT(quarter FROM s.sale_date) = 4
   GROUP BY 1
   ORDER BY 2 DESC;
```


**Sales Count for Each Product**
- 3. How many units of each coffee product have been sold?
```SQL
   SELECT 
      pd.product_name,
      COUNT(*) AS sold_units
   FROM sales as s
   JOIN product AS pd
      ON pd.product_id = s.product_id
   GROUP BY 1
   ORDER BY 2 DESC;
```

**Average Sales Amount per City**
- 4. What is the average sales amount per customer in each city?
```SQL
   SELECT 
      ci.city_name,
      SUM(s.total) AS total_revenue,
      COUNT(DISTINCT s.customer_id) AS total_customers,
      ROUND((SUM(s.total) / COUNT(DISTINCT s.customer_id))::numeric, 2) as avg_sale_pr_cx

   FROM sales AS s
   JOIN customer AS cus
      ON cus.customer_id = s.customer_id
   JOIN city AS ci
      ON ci.city_id = cus.city_id
   GROUP BY 1
   ORDER BY 2 DESC;
```


**City Population and Coffee Consumers (25%)**
- 5. Provide a list of cities along with their populations and estimated coffee consumers.
```SQL
   WITH city_table AS
   (
      SELECT
         city_name,
         population,
         population * 0.25 AS est_coffee_consumers
      FROM city
   ),
   customer_table
   AS
   (
      SELECT 
         ci.city_name,
         COUNT(DISTINCT cus.customer_id) AS unique_customer
      FROM sales AS s
      JOIN customer AS cus
         ON cus.customer_id = s.customer_id
      JOIN city AS ci
         ON ci.city_id = cus.city_id
      GROUP BY 1
   )

   SELECT 
      ct.city_name,
      ct.population,
      ct.est_coffee_consumers,
      cst.unique_customer
   FROM city_table AS ct
   JOIN customer_table AS cst
      ON ct.city_name = cst.city_name;
```


**Top Selling Products by City**
- 6. What are the top 3 selling products in each city based on sales volume?
```SQL
   SELECT *
   FROM
   (
      SELECT 
         c.city_name,
         pd.product_name,
         COUNT(s.sale_id) AS total_orders,
         DENSE_RANK() OVER(PARTITION BY c.city_name ORDER BY COUNT(s.sale_id)) AS dense_rank
      FROM sales AS s
      JOIN product as pd
         ON s.product_id = pd.product_id
      JOIN customer AS cs
         ON s.customer_id = cs.customer_id
      JOIN city as c
         ON cs.city_id = c.city_id
      GROUP BY 1,2
      ORDER BY 1,3 DESC
   )
   WHERE dense_rank < 4;
```


**Customer Segmentation by City**
- 7. How many unique customers are there in each city who have purchased coffee products?
```SQL
   SELECT 
      c.city_name,
      COUNT(DISTINCT cs.customer_id) AS unique_customers
   FROM city as c
   JOIN customer as cs
      ON c.city_id = cs.city_id
   JOIN sales AS s
      ON cs.customer_id = s.customer_id
   JOIN product AS pd
      ON s.product_id = pd.product_id
   WHERE pd.product_id IN (1,2,3,4,5,6,7,8,9,10,11,12,13,14)
   GROUP BY 1;
```


**Average Sale vs Rent**
- 8. Find each city and their average sale per customer and avg rent per customer
```SQL
   SELECT
      c.city_name,
      c.estimated_rent,
      ROUND((SUM(s.total) / COUNT(DISTINCT s.customer_id))::numeric, 2) average_sale_per_cx,
      ROUND((c.estimated_rent / COUNT(DISTINCT cs.customer_id))::numeric, 2) average_rent_per_cx
   FROM sales AS s
   JOIN customer AS cs
      ON s.customer_id = cs.customer_id
   JOIN city AS c
      ON cs.city_id = c.city_id
   GROUP BY 1,2;
```


**Monthly Sales Growth** 
- 9. Sales growth rate: Calculate the percentage growth (or decline) in sales over different time periods (monthly) by each city.
```SQL
   WITH monthly_sales 
   AS
   (
      SELECT
         c.city_name,
         EXTRACT(YEAR FROM s.sale_date) AS year,
         EXTRACT(MONTH FROM s.sale_date) AS month,
         SUM(s.total) AS total_sale
      FROM sales AS s
      JOIN customer AS cs
         ON s.customer_id = cs.customer_id
      JOIN city AS c
         ON c.city_id = cs.city_id
      GROUP BY 1,2,3
      ORDER BY 1,2,3
   ),
   with_last_month_sale
   AS
   (
   SELECT 
      *,
      LAG(total_sale, 1) OVER(PARTITION BY city_name ORDER BY year, month) as last_month_sale
   FROM monthly_sales
   )

   SELECT 
      city_name,
      year,
      month,
      total_sale AS monthly_sale,
      last_month_sale,
      ROUND(
            ((total_sale / last_month_sale - 1) * 100)::numeric ,2
         )
   FROM with_last_month_sale
   WHERE last_month_sale IS NOT NULL;
```


**Market Potential Analysis**
- 10. Identify top 3 city based on highest sales, return city name, total sale, total rent, total customers, estimated coffee consumer
```SQL
   SELECT 
      c.city_name,	
      SUM(s.total) AS total_sales,
      COUNT(s.sale_id) AS total_orders,
      c.estimated_rent,
      COUNT(DISTINCT s.customer_id) AS total_customers,
      ROUND(
            (c.population *0.25 / 1000000), 2
         ) AS coffee_consumers_in_million,
      ROUND(
            (SUM(s.total) / COUNT(DISTINCT s.customer_id)):: numeric , 2
         ) AS avg_sale_per_cx,
      ROUND(
            (c.estimated_rent / COUNT(DISTINCT s.customer_id)):: numeric , 2
         ) AS avg_rent_per_cx
   FROM city as c
   JOIN customer AS cs
      ON c.city_id = cs.city_id
   JOIN sales AS s
      ON cs.customer_id = s.customer_id
   GROUP BY 1,4,6
   ORDER BY 2 DESC
   LIMIT 3;
```

 
## Insights
After analyzing the data, the recommended top three cities for new store openings are:

**City 1: Pune**  
1. Highest total revenue with 1.25 million
2. Average sale per customer is Highest with 24k
3. Low city rent eith 294

**City 2: Delhi**  
1. Highest estimated coffee consumers with 7.75 million
2. Second Highest costumers  with 68 
3. Average rent per customer is below 500

**City 3: Jaipur**  
1. Lowest average rent per customer with 156.52
2. Highest customers with 69
3. 4th Highest total revenue with 11.6k average sale per customer 

