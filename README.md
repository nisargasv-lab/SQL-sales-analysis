# Intermediate SQL - Sales Analysis

## Overview

Analysis of customer behavior, retention, and lifetime value for an e-commerce company to improve customer retention and maximize revenue.

---

## Business Questions

1. **Customer Segmentation:** Who are our most valuable customers?
2. **Cohort Analysis:** How do different customer groups generate revenue?
3. **Retention Analysis:** Which customers haven't purchased recently?

---

# Analysis

## 1. Customer Segmentation

**🖥️ Query:** `1_customer_segmentation.sql`

### What This Analysis Does

* Categorizes customers based on total lifetime value (LTV)
* Assigns customers to **Low, Mid, and High-value** segments
* Calculates total LTV, customer count, and average LTV for each segment

### SQL Query

```sql
WITH customer_ltv AS (
    SELECT 
        customerkey,
        cleaned_name,
        SUM(net_revenue) AS total_ltv
    FROM cohort_analysis2
    GROUP BY 
        customerkey,
        cleaned_name
), 

customer_segments AS (
    SELECT 
        PERCENTILE_CONT(0.25) 
            WITHIN GROUP (ORDER BY total_ltv) AS ltv_25th_percentile,
        PERCENTILE_CONT(0.75) 
            WITHIN GROUP (ORDER BY total_ltv) AS ltv_75th_percentile
    FROM customer_ltv
), 

segment_vales AS (
    SELECT
        c.*,
        CASE 
            WHEN c.total_ltv < cs.ltv_25th_percentile 
                THEN '1-low-value'
            WHEN c.total_ltv <= cs.ltv_75th_percentile 
                THEN '2-Mid-value'
            ELSE '3-High-value'
        END AS customer_segment
    FROM 
        customer_ltv c,
        customer_segments cs
)

SELECT 
    customer_segment,
    SUM(total_ltv) AS total_ltv,
    COUNT(customerkey) AS customer_count,
    SUM(total_ltv) / COUNT(customerkey) AS avg_ltv
FROM segment_vales
GROUP BY 
    customer_segment
ORDER BY 
    customer_segment DESC;
```

### Analysis

* Calculates each customer's total lifetime value.
* Uses the **25th and 75th percentiles** to segment customers.
* Classifies customers into **Low-value, Mid-value, and High-value** segments.
* Calculates total LTV, customer count, and average LTV for each segment.

---

# 2. Customer Revenue by Cohort

**🖥️ Query:** `2_cohort_analysis.sql`

This analysis examines customer revenue from three perspectives:

1. Overall cohort revenue
2. Revenue progression based on time since first purchase
3. First-purchase revenue

---

## 2.1 Customer Revenue by Cohort — Not Adjusted for Time in Market

### SQL Query

```sql
-- Title: Customer Revenue by Cohort (NOT adjusted for time in market)

SELECT
    cohort_year,
    SUM(net_revenue) AS total_revenue,
    COUNT(DISTINCT customerkey) AS total_customers,
    SUM(net_revenue) / COUNT(DISTINCT customerkey) AS customer_revenue
FROM cohort_analysis2
GROUP BY 
    cohort_year;
```

### Analysis

* Calculates total revenue for each cohort.
* Counts the number of unique customers in each cohort.
* Calculates average revenue per customer.
* Includes all purchases made by customers in the cohort.

---

## 2.2 Customer Revenue by Cohort — Adjusted for Time in Market

### SQL Query

```sql
-- Title: Customer Revenue by Cohort (Adjusted for time in market)

WITH purchase_days AS (
    SELECT
        customerkey,
        net_revenue,
        orderdate - MIN(orderdate) OVER (
            PARTITION BY customerkey
        ) AS days_since_first_purchase
    FROM cohort_analysis2
)

SELECT
    days_since_first_purchase,
    SUM(net_revenue) AS total_revenue,
    SUM(net_revenue) / 
        (SELECT SUM(net_revenue) FROM cohort_analysis2) * 100 
        AS percentage_of_total_revenue,
    SUM(
        SUM(net_revenue) / 
        (SELECT SUM(net_revenue) FROM cohort_analysis2) * 100
    ) OVER (
        ORDER BY days_since_first_purchase
    ) AS cumulative_percentage_of_total_revenue
FROM purchase_days
GROUP BY 
    days_since_first_purchase
ORDER BY 
    days_since_first_purchase;
```

### Analysis

* Calculates the number of days since each customer's first purchase.
* Uses `MIN(orderdate) OVER (PARTITION BY customerkey)` to identify each customer's first purchase date.
* Calculates revenue generated on each day since the first purchase.
* Calculates the percentage of total revenue generated.
* Calculates the cumulative percentage of total revenue over time.

---

## 2.3 Customer Revenue by Cohort — First Purchase Only

### SQL Query

```sql
-- Title: Customer Revenue by Cohort 
-- (Adjusted for time in market) - Only First Purchase Date

SELECT
    cohort_year,
    SUM(net_revenue) AS total_revenue,
    COUNT(DISTINCT customerkey) AS total_customers,
    SUM(net_revenue) / COUNT(DISTINCT customerkey) AS customer_revenue
FROM cohort_analysis2
WHERE orderdate = first_purchasedate
GROUP BY 
    cohort_year;
```

### Analysis

* Filters the data to include only the customer's first purchase.
* Calculates total first-purchase revenue by cohort.
* Counts customers in each cohort.
* Calculates average first-purchase revenue per customer.

---

# 3. Customer Retention

**🖥️ Query:** `3_retention_analysis.sql`

### What This Analysis Does

* Identifies customers at risk of churning.
* Analyzes each customer's last purchase.
* Uses `ROW_NUMBER()` to identify the most recent purchase.
* Classifies customers as **Active** or **Churned**.
* Uses a **6-month inactivity period** as the churn threshold.
* Calculates customer counts and status percentages by cohort.

### SQL Query

```sql
WITH customer_last_purchase AS (
    SELECT  
        customerkey,
        cleaned_name,
        orderdate,
        cohort_year,
        ROW_NUMBER() OVER (
            PARTITION BY customerkey 
            ORDER BY orderdate DESC
        ) AS rn,
        first_purchasedate
    FROM cohort_analysis2
), 

churned_customers AS (
    SELECT 
        customerkey,
        cohort_year,
        cleaned_name,
        orderdate AS last_purchase_date,
        CASE 
            WHEN orderdate < 
                (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months' 
                THEN 'churned'
            ELSE 'Active'
        END AS customer_status
    FROM customer_last_purchase 
    WHERE rn = 1
    AND first_purchasedate < 
        (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months' 
)

SELECT 
    customer_status,
    cohort_year,
    COUNT(customerkey) AS num_customers,
    SUM(COUNT(customerkey)) OVER (
        PARTITION BY cohort_year
    ) AS total_customers,
    ROUND(
        COUNT(customerkey) / 
        SUM(COUNT(customerkey)) OVER (
            PARTITION BY cohort_year
        ), 
        2
    ) AS status_percentage
FROM churned_customers
GROUP BY 
    cohort_year, 
    customer_status;
```

### Analysis

* Uses `ROW_NUMBER()` to identify the latest purchase for each customer.
* Compares the latest purchase date with the latest order date in the sales data.
* Classifies customers as **Active** or **Churned** based on six months of inactivity.
* Calculates the number and percentage of customers in each status by cohort.

---

# Key SQL Concepts Used

| SQL Concept         | Purpose                                                      |
| ------------------- | ------------------------------------------------------------ |
| `WITH`              | Creates Common Table Expressions (CTEs)                      |
| `SELECT`            | Retrieves and calculates required data                       |
| `SUM()`             | Calculates revenue and lifetime value                        |
| `COUNT(DISTINCT)`   | Counts unique customers                                      |
| `GROUP BY`          | Groups data by customer, cohort, or days                     |
| `WHERE`             | Filters records                                              |
| `CASE`              | Creates customer segments and customer status                |
| `PERCENTILE_CONT()` | Calculates LTV percentiles                                   |
| `MIN()`             | Finds the first purchase date                                |
| `ROW_NUMBER()`      | Identifies the most recent purchase                          |
| `PARTITION BY`      | Performs calculations separately for each customer or cohort |
| `OVER()`            | Applies window functions                                     |
| Subqueries          | Calculates overall revenue and latest order dates            |
| `INTERVAL`          | Defines the six-month churn period                           |
| `ORDER BY`          | Sorts results and calculates cumulative revenue              |

---

# Strategic Analysis

## 1. Customer Value Optimization

* Segment customers based on lifetime value.
* Identify high-value customers for retention and loyalty initiatives.
* Develop targeted strategies for mid-value and low-value customers.

## 2. Cohort Performance

* Compare revenue generated by different customer cohorts.
* Measure how revenue accumulates after the first purchase.
* Analyze first-purchase revenue separately from later purchases.
* Understand customer revenue progression based on time since the first purchase.

## 3. Retention & Churn Prevention

* Identify customers who have not purchased within the defined **6-month period**.
* Compare active and churned customers across cohorts.
* Use customer purchase history to identify customers who may require re-engagement.

---

# Conclusion

This project demonstrates the use of SQL to analyze:

* Customer segmentation
* Lifetime value
* Cohort revenue
* Customer purchasing behavior
* Customer retention
* Churn analysis

The analysis combines:

**CTEs + Aggregations + Window Functions + Subqueries + Cohort Analysis + Customer Segmentation + Churn Analysis**

to transform transactional customer data into meaningful business insights.

---

# Technical Details

| Category           | Details                                                               |
| ------------------ | --------------------------------------------------------------------- |
| **Database**       | PostgreSQL                                                            |
| **Analysis Tools** | PostgreSQL, DBeaver                                                   |
| **Visualization**  | No charts created; analysis presented through SQL queries and results |

---

# Future Improvements

The project can be further extended by:

* [ ] Adding Power BI visualizations
* [ ] Creating a customer retention dashboard
* [ ] Analyzing monthly revenue trends
* [ ] Investigating customer purchase frequency
* [ ] Identifying high-value customers at risk of churn
* [ ] Creating additional customer behavior segments

---

# Author

**Nethra**

*Data Analytics Portfolio Project*
