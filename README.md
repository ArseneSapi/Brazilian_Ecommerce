# Goal of the project
In this project, the objective was to make a review of data sets of the company named Brazilian_Ecommerce to answer the following questions.
## Customer satisfcation
- Average review score
- Negative review percentage (score reviews <= 2)
- Average response time for negative review
## Revenue and  product category performance
- Total revenue
- Total revenue per state
- Average order revenue  ($120.38)
- Year over year revenue and growth percentage 
- Running total revenue overtime and moving average order revenue 
- Revenue per category and percentage 
- Revenue lost and percentage 
- Revenue lost splitted into product category
## Delivery Performance
- Average delivered time
- Late delivery percentage
- Average Shipping cost per order 
- Free shipping impact
## Payment Analysis
- Total Payments
- Payment per type and percentage
- Average payments per payment type compared to global Average payments
- Instalments rate 
- Proportion of orders payments with multiple installments in terms of orders values 
## Seller Performance
- Revenue per seller(Top10)
- Average seller Response time (time btw purchase_date and carrier_delivery_date)

## Technology used 
### SQL
The project was entirely completed with SQL(MySQL)

## Data source and contents
Data was a set of seven tables as follow :
- customers (characteritics of customers who bought from company)
- order_items (items per order made by customers and their price)
- order_payments (payments made by customers)
- orders reviews (reviews left by customers)
- orders (orders made by customers)
- product_category_name_translation(from Spanish to English)
- products (characteritics of products available on internet site)
- sellers (characteritics of sellers who sold their products through internet site)
- geolocation (customers geolocation)

Data are provides by Kaggle. They can be downloaded through the following link:
https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce?resource=download

## SQL Data Analysis Techniques used
- Subqueries and Common Table Expressions (CTEs):
Subqueries and CTEs were used to break down problems into smaller, more manageable parts, thus simplifying queries for improved organization and readability.

- CASE Statements:
The CASE function was critical for embedding conditional logic within queries. By categorizing and grouping data based on specific conditions, I created custom columns that enhanced the flexibility and depth of my analysis.

- Window Functions:
Window functions were used for tasks that required comparing the values between previous years and next year. These functions allowed me to compute metrics over specific data partitions, providing richer insights without the need for joins or subqueries.

- JOINs:
I combined data from multiple tables using JOINs, ensuring accurate relationships between datasets. This approach enabled a comprehensive view of the data, leading to more informed decision-making and through analysis.


## Customer Satisfaction Analysis
- Overall customer satifaction is 4.0868 wi
```sql
SELECT 
	AVG(review_score)
FROM order_reviews
```

- Negative review percentage (score reviews <= 2 : in this data min =1 and max=5) 14.68% of reviews are negative
```sql
WITH review_stats AS (
    SELECT 
        CASE  
            WHEN review_score <= 2 THEN 'negative review'
            ELSE 'positive review'
        END AS review_type,
        COUNT(*) AS review_count,
	SUM(COUNT(*)) OVER () AS total_reviews
    FROM order_reviews
    GROUP BY 
        CASE 
            WHEN review_score <= 2 THEN 'negative review'
            ELSE 'positive review'
        END
)
SELECT
    review_type,
    review_count AS nber_of_reviews,
    ROUND((review_count * 100.0 / total_reviews), 2) AS reviewTypePerct
FROM review_stats
```

- Average response time for negative review = AVG(review_answer_timestamp - review_creation_date)
-- 2.48 days; pretty similar with overall response time (2.58 days)
```sql
SELECT
	AVG(DATEDIFF(review_answer_timestamp, review_creation_date)) AS avgDays
FROM order_reviews
WHERE review_score <=2
```

## Product Category Performance

- Total revenue
```sql
WITH valid_orders AS (
    SELECT
	order_id
    FROM orders
    WHERE order_status NOT IN ('canceled', 'unavailable')
)
SELECT
	ROUND(SUM(oi.price), 2) AS totalRevenue
FROM order_items oi
JOIN valid_orders vo USING (order_id)
```

- Total revenue per state
```sql
WITH state_rev AS (
    SELECT
        COALESCE(s.seller_state, 'unknown') AS seller_state,
        SUM(oi.price) AS state_revenu,
        SUM(SUM(oi.price)) OVER () AS total_revenu
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
    LEFT JOIN sellers s ON s.seller_id = oi.seller_id
    WHERE o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY COALESCE(s.seller_state, 'unknown')
)
SELECT
    seller_state,
    ROUND(state_revenu, 2) AS state_revenu,
    ROUND(total_revenu, 2) AS overall_rev,
    ROUND((state_revenu / total_revenu) * 100, 2) AS percentage
FROM state_rev
ORDER BY state_revenu DESC
```

- Average order revenue  ($120.38)
```sql
SELECT
	ROUND(AVG(oi.price), 2) AS totalRevenu
FROM order_items oi
JOIN orders o ON o.order_id=oi.order_id
WHERE o.order_status NOT IN('canceled', 'unavailable')
```

- Year over year revenue and growth percentage (more than 13000% growth between 2016 and 2017
```sql
SELECT
	*,
	ROUND(CurrentYearRev - COALESCE(PreviousYearRev, 0), 1)  AS YoY_Change,
	ROUND(((CurrentYearRev - COALESCE(PreviousYearRev, 0)) / PreviousYearRev )*100,1) AS YoY_perc

FROM (
	SELECT
		YEAR(o.order_purchase_timestamp) AS yr,
		ROUND(SUM(oi.price), 1) AS CurrentYearRev,
		ROUND(LAG(SUM(oi.price))  OVER (ORDER BY YEAR(o.order_purchase_timestamp)), 1) AS PreviousYearRev
	FROM order_items oi
	JOIN orders o on o.order_id=oi.order_id
		AND o.order_status not in('canceled', 'unavailable')
	GROUP BY YEAR(o.order_purchase_timestamp) 
    )t
```

- Running total revenue overtime and moving average order revenue 
```sql
WITH yearly_stats AS (
    SELECT
        YEAR(o.order_purchase_timestamp) AS year_order,
        SUM(oi.price) AS year_revenue,
        AVG(oi.price) AS avg_price
    FROM order_items oi
    JOIN orders o 
        ON o.order_id = oi.order_id
        AND o.order_status NOT IN ('canceled', 'unavailable')
    GROUP BY year_order
)
SELECT
    year_order,
    ROUND(year_revenue, 0) AS year_revenue,
    ROUND(SUM(year_revenue) OVER (ORDER BY year_order), 0) AS running_total_revenue,
    ROUND(AVG(avg_price) OVER (ORDER BY year_order ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), 0) AS moving_average_price
FROM yearly_stats
ORDER BY year_order
```

- Revenue per category and percentage (The top 10 product category drives 63% of Total revenue
```sql
WITH category_revenue AS (
    SELECT
        pn.product_category_name_english AS product_category_name,
        SUM(oi.price) AS category_revenue,
        SUM(SUM(oi.price)) OVER () AS total_revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
        AND o.order_status NOT IN ('canceled', 'unavailable')
    JOIN products p ON p.product_id = oi.product_id
    JOIN product_category_name_translation pn 
        ON pn.product_category_name = p.product_category_name
    GROUP BY pn.product_category_name_english
)
SELECT
    product_category_name,
    ROUND(category_revenue, 2) AS RevPerCategory,
    ROUND((category_revenue / total_revenue) * 100, 1) AS RevCategoryPerc
FROM category_revenue
ORDER BY RevPerCategory DESC
LIMIT 10
```

- Revenue lost and percentage (it represent 0.72% of revene projected=revenue without cancellation or unavailability)
```sql
WITH revenue_summary AS (
    SELECT
        SUM(CASE
		WHEN o.order_status IN ('canceled', 'unavailable') THEN oi.price ELSE 0 
            END) AS lost_revenue,
        SUM(oi.price) AS total_projected_revenue
    FROM order_items oi
    LEFT JOIN orders o ON oi.order_id = o.order_id
)
SELECT
    ROUND(lost_revenue, 0) AS lostRevenue,
    ROUND(total_projected_revenue, 0) AS projectedRevenue,
    ROUND((lost_revenue / NULLIF(total_projected_revenue, 0)) * 100,2) AS LostPercentage
FROM revenue_summary
```

- Revenue lost splitted into product category
```sql
WITH category_revenue AS (
    SELECT
        pn.product_category_name_english AS product_category_name,
        SUM(oi.price) AS category_revenue,
        SUM(SUM(oi.price)) OVER () AS total_lost_revenue
    FROM order_items oi
    JOIN orders o ON o.order_id = oi.order_id
        AND o.order_status IN ('canceled', 'unavailable')
    JOIN products p ON p.product_id = oi.product_id
    JOIN product_category_name_translation pn 
        ON pn.product_category_name = p.product_category_name
    GROUP BY pn.product_category_name_english
)
SELECT
    product_category_name,
    ROUND(category_revenue, 2) AS LostRevPerCategory,
    ROUND(
        (category_revenue / NULLIF(total_lost_revenue, 0)) * 100, 1) AS LostRevPercentage
FROM category_revenue
WHERE total_lost_revenue > 0  -- Prevent division by zero
ORDER BY LostRevPerCategory DESC
LIMIT 10
``` 

## Delivery Performance
- Average delivered time

```sql
SELECT
	AVG(DATEDIFF(order_delivered_customer_date, order_purchase_timestamp)) AS AvgdeliveredTime
FROM orders
WHERE order_status='delivered'
```

- Late delivery percentage (there are 7826 delivered =8.11% of total deliveries)
```sql
WITH delivery_stats AS (
    SELECT
        SUM(CASE WHEN order_delivered_customer_date > order_estimated_delivery_date THEN 1 ELSE 0 
            END) AS late_deliveries,
        COUNT(*) AS total_deliveries
    FROM orders
    WHERE order_status = 'delivered'
)
SELECT
    late_deliveries,
    total_deliveries,
    ROUND((late_deliveries * 100.0 / NULLIF(total_deliveries, 0)),2) AS late_delivery_percentage
FROM delivery_stats
```

- Average Shipping cost per order ($19.99 = 16.65% of average revenue)

```sql
SELECT
	ROUND(AVG(i.freight_value),2) AvgFreightValue
FROM orders o
JOIN order_items i ON i.order_id=o.order_id
WHERE o.order_status NOT IN ('canceled', 'unavailable')
```

- Free shipping impact (0.35% of orders are shipping cost free)
```sql
WITH shipping_stats AS (
    SELECT
        COUNT(DISTINCT CASE WHEN i.freight_value = 0 THEN i.order_id END) AS free_ship_orders,
        COUNT(DISTINCT i.order_id) AS total_orders
    FROM order_items i
    JOIN orders o ON i.order_id = o.order_id
        AND o.order_status NOT IN ('canceled', 'unavailable')
)
SELECT
    free_ship_orders,
    total_orders,
    ROUND((free_ship_orders * 100.0 / NULLIF(total_orders, 0)),2) AS free_ship_percentage
FROM shipping_stats
```

## Payment Analysis
- Total Payments
```sql
SELECT 
	ROUND(SUM(payment_value), 2) totalPayments
FROM order_payments
```

- Payment per type and percentage(Credit cart is the most payment rype with 78.34% of payments)
```sql
SELECT
    payment_type,
    ROUND(SUM(payment_value), 0) AS total_payments,
    ROUND((SUM(payment_value) * 100.0 / NULLIF(SUM(SUM(payment_value)) OVER (), 0)),2) AS total_payments_percentage
FROM order_payments
GROUP BY payment_type
ORDER BY total_payments DESC
```

- Average payments per payment type compared to global Average payments
```sql
SELECT
    payment_type,
    ROUND(avg_payment_type, 0) AS AvgPaymentType,
    ROUND(global_avg_payment, 2) AS GlobalAvgPayment,
    ROUND(avg_payment_type - global_avg_payment, 2) AS AvgDiffPerPaymentType
FROM (
    SELECT
        payment_type,
        AVG(payment_value) AS avg_payment_type,
        AVG(AVG(payment_value)) OVER () AS global_avg_payment
    FROM order_payments
    GROUP BY payment_type
     )t
ORDER BY AvgPaymentType DESC
```

- Instalments rate (49.42% payments with instalments more than 1)
```sql
SELECT
	ROUND((COUNT(payment_installments)/(SELECT COUNT(payment_installments) FROM order_payments ))*100,2) installmentRate
FROM (
	SELECT
		order_id,
		payment_installments
	FROM order_payments 
	WHERE payment_installments >1
     )t
```

- Proportion of orders payments with multiple installments in terms of orders values (62.35%)
```sql
WITH revenue_summary AS (
    SELECT
        SUM(oi.price) AS total_revenue,
        SUM(CASE WHEN p.payment_installments > 1 THEN oi.price ELSE 0 END) AS installment_revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
        AND o.order_status NOT IN ('canceled', 'unavailable')
    LEFT JOIN order_payments p ON oi.order_id = p.order_id
)
SELECT
    ROUND((installment_revenue / NULLIF(total_revenue, 0)) * 100, 2) AS installment_revenue_percentage
FROM revenue_summary
```

## Seller Performance
- Revenue per seller(Top10)
```sql
SELECT 
	seller_id,
	sellerrev,
	rank_nber
FROM (
	SELECT 
		i.seller_id,
		ROUND(SUM(i.price),2) sellerRev,
		ROW_NUMBER() OVER(ORDER BY SUM(i.price) DESC)  Rank_nber
	FROM order_items i
	JOIN orders o ON  i.order_id=o.order_id
		AND o.order_status NOT IN('canceled', 'unavailable')
	GROUP BY i.seller_id
)t
WHERE Rank_nber<=10
```

- Average seller Response time (time btw purchase_date and carrier_delivery_date) = 3.22 days
```sql
SELECT 
	AVG(DATEDIFF(order_delivered_carrier_date, order_purchase_timestamp)) avgdays
FROM orders
WHERE order_status NOT IN ('canceled', 'unavailable')
```

- Shipper delivery performance = 9.28 days
```sql
SELECT 
	AVG(DATEDIFF(order_delivered_customer_date, order_delivered_carrier_date)) avgdays
FROM orders
WHERE order_status NOT IN ('canceled', 'unavailable')
```
## Main facts
1- Customer Satisfaction
The company’s overall customer satisfaction score is 4.086 , which falls slightly below the e-commerce industry standard of 4.2 .

2- Revenue Growth

Revenue surged by 13,500% year-over-year (YOY) between 2016 and 2017, driven by only two months of operational activity in 2016.
A more sustainable growth rate of 20% YOY was achieved between 2017 and 2018.
Pricing Trends
The average selling price has declined annually: $129 → $125 → $123 . This trend is partly attributed to limited sales data from the two-month operational period in 2016.

3- Product Performance
The top 10 product categories account for 63.3% of total revenue , highlighting a significant concentration of sales in key segments.

4- Delivery Performance

Average delivery time : 12.49 days (includes seller response time + shipper delivery).
8.11% of deliveries are delayed, indicating room for improvement in logistics efficiency.
Shipping Costs

Average shipping cost per order: $19.99 , representing 16.65% of average revenue per order .
Less than 1% of orders qualify for free shipping, suggesting an opportunity to explore promotional strategies.

5- Payment Methods

78.35% of payments are processed via credit card, which also has the highest average transaction value compared to other payment methods.
Payment plans represent 49.42% of all payments and are used for 63% of total orders , indicating their popularity among customers.

6- Operational Metrics

Seller response time (time from order placement to carrier pickup): 3.22 days .
Shipper delivery performance (carrier transit time): 9.28 days .

## Recommendations

1- Improve Customer Satisfaction
- Conduct surveys to identify pain points like delivery delays, pricing concerns.
- Address late deliveries by optimizing logistics
- Introduce loyalty programs to enhance satisfaction. It could be rewards for repeat customers, early access to sales
  
2. Sustain Revenue Growth
- Focus on top-performing categories : Invest in marketing and inventory for the 10 categories driving 63.3% of revenue.
- Expand product offerings in adjacent categories to reduce reliance on a few segments.
- Leverage payment plans : Promote flexible payment options (used in 63% of orders) to attract budget-conscious customers.
  
3. Address Pricing Trends
- Analyze price elasticity : Determine if price reductions are impacting margins or driving volume.
- Introduce bundled offers (discounts for purchasing multiple items) to offset price declining.
- Highlight value-added services (Extended warranties, free tech support) to justify pricing.
  
4. Optimize Delivery Performance
- Reduce seller response time (3.22 days): one (1) day would be a good target
- Automate order confirmations and inventory tracking.
- Partner with faster carriers or regional fulfillment centers.

5- Improve shipper delivery time (9.28 days):
- Negotiate SLAs with carriers for faster transit.
- Offer tiered shipping options (Expedited delivery for higher-margin products).

6- Reduce late deliveries (8.11%):
- Implement real-time tracking for customers.
- Penalize underperforming carriers or diversify logistics partners.

6. Reevaluate Shipping Costs & Promotions
- Introduce free shipping thresholds ("Free shipping on orders over $X") to increase average order value.
- Negotiate bulk shipping rates with carriers to lower costs as order volume grows.
- Test limited-time free shipping promotions to gauge impact on conversion rates.

8. Capitalize on Payment Preferences
- Optimize credit card processing : Ensure seamless checkout experiences (Saved payment details, fraud protection).
- Expand payment plan options : Partner with fintech companies to offer "buy now, pay later" solutions.
- Monitor payment plan risks : Track default rates to avoid cash flow issues.

9. Operational Efficiency
- Automate workflows : Use tools to streamline order processing and reduce seller response time.
- Benchmark against competitors : Compare delivery times, shipping costs, and pricing to identify gaps.
Invest in data analytics : Track trends in real time to make proactive decisions (Dynamic pricing, inventory forecasting).











