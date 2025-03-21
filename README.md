# Goal of the project
In this project, the objective was to make a review of data sets of the compbany named Looker Ecommerce to answer the following questions.
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
The project was entirely completed with SQL especially MySQL

## Data source and contents
Data was a set of seven tables as follow :


The data sets are from Kaggle. They can be downloaded through the following link:
https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce?resource=download


## SQL Data Analysis Techniques used
- Subqueries and Common Table Expressions (CTEs):
I employed subqueries and CTEs to break down problems into smaller, more manageable parts, thus simplifying queries for improved organization and readability.

- CASE Statements:
The CASE function was critical for embedding conditional logic within my queries. By categorizing and grouping data based on specific conditions, I created custom columns that enhanced the flexibility and depth of my analysis.

- Window Functions:
I utilized window functions for tasks that required comparing the values of previous years to the current data. These functions allowed me to compute metrics over specific data partitions, providing richer insights without the need for complex joins or subqueries.

- JOINs:
I combined data from multiple tables using JOINs, ensuring accurate relationships between datasets. This approach enabled a comprehensive view of the data, leading to more informed decision-making and thorough analysis.
