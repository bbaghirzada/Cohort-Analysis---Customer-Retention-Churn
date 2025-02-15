# Cohort Analysis for Subscription Retention and Churn

## Overview
This project focuses on analyzing **subscription retention** and **churn rates** for a fictional company using cohort analysis. The goal is to understand customer behavior over time, identify trends in retention and churn, and provide actionable insights to improve customer retention strategies.

The analysis is based on subscription data from the provided turing college dataset. The results are visualized in an Excel file (`cohort_analysis.xlsx`) and generated using SQL queries.

---

## Key Features
- **Cohort Analysis**: Tracks customer retention and churn rates over a 14-week period.
- **Retention Metrics**: Measures the percentage of customers who continue their subscriptions over time.
- **Churn Metrics**: Identifies the percentage of customers who cancel their subscriptions each week.
- **Visualization**: Results are presented in an Excel file for easy interpretation.

---

## Files Included
1. **`cohort_analysis.xlsx`**:
   - Contains the results of the cohort analysis, including:
     - **Retention Rates**: Weekly retention rates for each cohort.
     - **Churn Rates**: Weekly churn rates for each cohort.
     - **Active Users**: Number of active users per week for each cohort.

---
### Query for Cohort Analysis for Retention and Churn
```sql
WITH date_range AS (
    SELECT
        DATE('2021-02-07') - INTERVAL 14 WEEK AS start_date,
        DATE('2021-02-07') AS end_date
),

subscriptions AS (
    SELECT 
        user_pseudo_id,
        DATE_TRUNC(subscription_start, WEEK) AS subscription_week,
        DATE_TRUNC(IFNULL(subscription_end, CURRENT_DATE()), WEEK) AS end_week
    FROM `tc-da-1.turing_data_analytics.subscriptions`
    WHERE subscription_start >= (SELECT start_date FROM date_range)
        AND subscription_start < (SELECT end_date FROM date_range)
),

cohort_data AS (
    SELECT
        subscription_week,
        user_pseudo_id,
        subscription_week AS cohort_week,
        0 AS week_num
    FROM subscriptions
    UNION ALL
    SELECT
        s.subscription_week,
        s.user_pseudo_id,
        DATE_ADD(s.subscription_week, INTERVAL seq WEEK) AS cohort_week,
        seq AS week_num
    FROM subscriptions s
    CROSS JOIN UNNEST(GENERATE_ARRAY(1, 4)) AS seq
    WHERE DATE_ADD(s.subscription_week, INTERVAL seq WEEK) <= s.end_week
),

cohort_counts AS (
    SELECT
        subscription_week,
        week_num,
        COUNT(DISTINCT user_pseudo_id) AS active_users
    FROM cohort_data
    GROUP BY subscription_week, week_num
),

cohort_metrics AS (
    SELECT
        subscription_week,
        week_num,
        active_users,
        LAG(active_users) OVER (PARTITION BY subscription_week ORDER BY week_num) - active_users AS churned_users,
        ROUND(active_users / NULLIF(LAG(active_users) OVER (PARTITION BY subscription_week ORDER BY week_num), 0), 2) AS retention_rate,
        ROUND((LAG(active_users) OVER (PARTITION BY subscription_week ORDER BY week_num) - active_users) / NULLIF(MAX(active_users) OVER (PARTITION BY subscription_week ORDER BY week_num), 0), 2) AS churn_rate
    FROM cohort_counts
)

SELECT
    subscription_week,
    MAX(CASE WHEN week_num = 0 THEN active_users ELSE 0 END) AS start_subscriptions,
    MAX(CASE WHEN week_num = 1 THEN active_users ELSE 0 END) AS week_1,
    MAX(CASE WHEN week_num = 2 THEN active_users ELSE 0 END) AS week_2,
    MAX(CASE WHEN week_num = 3 THEN active_users ELSE 0 END) AS week_3,
    MAX(CASE WHEN week_num = 4 THEN active_users ELSE 0 END) AS week_4,
    MAX(CASE WHEN week_num = 1 THEN churned_users ELSE 0 END) AS churned_week_1,
    MAX(CASE WHEN week_num = 2 THEN churned_users ELSE 0 END) AS churned_week_2,
    MAX(CASE WHEN week_num = 3 THEN churned_users ELSE 0 END) AS churned_week_3,
    MAX(CASE WHEN week_num = 4 THEN churned_users ELSE 0 END) AS churned_week_4,
    ROUND(MAX(CASE WHEN week_num = 1 THEN churned_users ELSE 0 END) / NULLIF(MAX(CASE WHEN week_num = 0 THEN active_users ELSE 0 END), 0), 3) AS churn_rate_week_1,
    ROUND(MAX(CASE WHEN week_num = 2 THEN churned_users ELSE 0 END) / NULLIF(MAX(CASE WHEN week_num = 0 THEN active_users ELSE 0 END), 0), 3) AS churn_rate_week_2,
    ROUND(MAX(CASE WHEN week_num = 3 THEN churned_users ELSE 0 END) / NULLIF(MAX(CASE WHEN week_num = 0 THEN active_users ELSE 0 END), 0), 3) AS churn_rate_week_3,
    ROUND(MAX(CASE WHEN week_num = 4 THEN churned_users ELSE 0 END) / NULLIF(MAX(CASE WHEN week_num = 0 THEN active_users ELSE 0 END), 0), 3) AS churn_rate_week_4
FROM cohort_metrics
GROUP BY subscription_week
ORDER BY subscription_week;
```

---

## Key Insights

### Retention Trends
- **High Initial Retention**: Most customers remain active in the first week after subscribing.
- **Gradual Decline**: Retention rates decrease over time, with the most significant drop occurring in the first few weeks.

### Churn Trends
- **Early Churn**: The highest churn rates are observed in the first week, indicating that many customers cancel their subscriptions shortly after signing up.
- **Stabilization**: Churn rates stabilize after the third week, suggesting that customers who stay beyond this period are more likely to remain subscribed.

### Recommendations
- **Improve Onboarding**: Focus on improving the onboarding process to reduce early churn.
- **Engage Customers**: Implement engagement strategies (e.g., personalized emails, discounts) to retain customers beyond the first few weeks.

---

## Tools and Technologies
- **SQL**: For data extraction and analysis.
- **Excel**: For visualizing and presenting the results.
- **BigQuery**: For querying the dataset.
