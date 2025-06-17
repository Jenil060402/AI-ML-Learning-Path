# LAG() and LEAD() Functions - Complete Deep Dive

## The Core Concept: Looking Backward and Forward

LAG() and LEAD() are window functions that let you **access data from other rows** without using self-joins. They're like having a time machine for your data - you can look at previous or future rows while staying in the current row.

### The Mental Model
Imagine you're looking at a spreadsheet:
- **LAG()**: "What was the value in the row above me?"
- **LEAD()**: "What's the value in the row below me?"

But unlike spreadsheet formulas, these work with proper ordering and partitioning logic.

## Function Syntax & Core Behavior

```sql
LAG(column, offset, default_value) OVER (PARTITION BY column ORDER BY column)
LEAD(column, offset, default_value) OVER (PARTITION BY column ORDER BY column)
```

### Parameters:
- **column**: The column value you want to retrieve
- **offset**: How many rows back/forward (default = 1)
- **default_value**: What to return if no row exists (default = NULL)
- **OVER()**: Defines the window scope

### Key Insight: The "Tiebreaker Principle" Applies Here Too!
Just like with ROW_NUMBER(), if your ORDER BY isn't unique, LAG/LEAD results become unpredictable. Always ensure deterministic ordering.

## Detailed Examples

### Example 1: Sales Trend Analysis
```sql
-- Sample data: daily_sales table
CREATE TABLE daily_sales (
    sale_date DATE,
    store_id INT,
    revenue DECIMAL(10,2)
);

-- Compare each day to previous day
SELECT 
    sale_date,
    store_id,
    revenue,
    LAG(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date) as prev_day_revenue,
    LEAD(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date) as next_day_revenue,
    revenue - LAG(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date) as daily_change,
    ROUND(
        (revenue - LAG(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date)) * 100.0 / 
        LAG(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date), 2
    ) as pct_change
FROM daily_sales
ORDER BY store_id, sale_date;
```

**Sample Result:**
| sale_date | store_id | revenue | prev_day_revenue | next_day_revenue | daily_change | pct_change |
|-----------|----------|---------|------------------|------------------|--------------|------------|
| 2024-01-01| 1        | 1000    | NULL             | 1200             | NULL         | NULL       |
| 2024-01-02| 1        | 1200    | 1000             | 950              | 200          | 20.00      |
| 2024-01-03| 1        | 950     | 1200             | NULL             | -250         | -20.83     |

## Popular Use Cases

### 1. **Customer Journey Analysis**
```sql
-- Track customer behavior across sessions
WITH customer_sessions AS (
    SELECT 
        customer_id,
        session_date,
        pages_viewed,
        session_duration,
        LAG(session_date) OVER (PARTITION BY customer_id ORDER BY session_date) as prev_session_date,
        LAG(pages_viewed) OVER (PARTITION BY customer_id ORDER BY session_date) as prev_pages_viewed
    FROM user_sessions
)
SELECT 
    customer_id,
    session_date,
    pages_viewed,
    prev_pages_viewed,
    pages_viewed - prev_pages_viewed as engagement_change,
    session_date - prev_session_date as days_between_sessions
FROM customer_sessions
WHERE prev_session_date IS NOT NULL;
```

### 2. **Stock Price Analysis**
```sql
-- Calculate moving differences and identify patterns
SELECT 
    ticker,
    trade_date,
    closing_price,
    LAG(closing_price, 1) OVER (PARTITION BY ticker ORDER BY trade_date) as prev_close,
    LAG(closing_price, 7) OVER (PARTITION BY ticker ORDER BY trade_date) as week_ago_close,
    closing_price - LAG(closing_price, 1) OVER (PARTITION BY ticker ORDER BY trade_date) as daily_change,
    closing_price - LAG(closing_price, 7) OVER (PARTITION BY ticker ORDER BY trade_date) as weekly_change,
    CASE 
        When closing_price > LAG(closing_price, 1) OVER (PARTITION BY ticker ORDER BY trade_date) 
        THEN 'UP'
        WHEN closing_price < LAG(closing_price, 1) OVER (PARTITION BY ticker ORDER BY trade_date) 
        THEN 'DOWN'
        ELSE 'FLAT'
    END as daily_trend
FROM stock_prices;
```

### 3. **Gap Detection in Sequences**
```sql
-- Find missing dates in a time series
WITH date_gaps AS (
    SELECT 
        event_date,
        LAG(event_date) OVER (ORDER BY event_date) as prev_date,
        event_date - LAG(event_date) OVER (ORDER BY event_date) as gap_days
    FROM events
)
SELECT 
    prev_date,
    event_date,
    gap_days
FROM date_gaps
WHERE gap_days > 1  -- More than 1 day gap
ORDER BY event_date;
```

### 4. **Funnel Analysis**
```sql
-- Track user progression through conversion funnel
WITH user_funnel AS (
    SELECT 
        user_id,
        step_name,
        step_timestamp,
        LAG(step_name) OVER (PARTITION BY user_id ORDER BY step_timestamp) as prev_step,
        LAG(step_timestamp) OVER (PARTITION BY user_id ORDER BY step_timestamp) as prev_timestamp,
        LEAD(step_name) OVER (PARTITION BY user_id ORDER BY step_timestamp) as next_step
    FROM conversion_events
)
SELECT 
    user_id,
    prev_step,
    step_name,
    next_step,
    step_timestamp - prev_timestamp as time_between_steps,
    CASE 
        WHEN next_step IS NULL THEN 'DROP_OFF'
        ELSE 'CONTINUED'
    END as funnel_outcome
FROM user_funnel;
```

## Advanced Patterns

### 1. **Running Streaks**
```sql
-- Find consecutive days with increasing sales
WITH sales_trends AS (
    SELECT 
        sale_date,
        revenue,
        LAG(revenue) OVER (ORDER BY sale_date) as prev_revenue,
        CASE 
            WHEN revenue > LAG(revenue) OVER (ORDER BY sale_date) THEN 1 
            ELSE 0 
        END as is_increase
    FROM daily_sales
),
streak_groups AS (
    SELECT 
        sale_date,
        revenue,
        is_increase,
        SUM(CASE WHEN is_increase = 0 THEN 1 ELSE 0 END) 
            OVER (ORDER BY sale_date) as streak_group
    FROM sales_trends
)
SELECT 
    streak_group,
    MIN(sale_date) as streak_start,
    MAX(sale_date) as streak_end,
    COUNT(*) as streak_length
FROM streak_groups
WHERE is_increase = 1
GROUP BY streak_group
HAVING COUNT(*) >= 3  -- Streaks of 3+ days
ORDER BY streak_length DESC;
```

### 2. **First and Last Value in Groups**
```sql
-- Compare current value to first and last in each group
SELECT 
    department,
    employee_name,
    hire_date,
    salary,
    FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date) as first_hire_salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY hire_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as last_hire_salary,
    salary - FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY hire_date) as vs_first_hire
FROM employees;
```

### 3. **Multi-Step Lookback**
```sql
-- Compare with multiple previous periods
SELECT 
    month_year,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month_year) as prev_month,
    LAG(revenue, 3) OVER (ORDER BY month_year) as three_months_ago,
    LAG(revenue, 12) OVER (ORDER BY month_year) as year_ago,
    -- Month-over-month growth
    (revenue - LAG(revenue, 1) OVER (ORDER BY month_year)) * 100.0 / 
        LAG(revenue, 1) OVER (ORDER BY month_year) as mom_growth,
    -- Year-over-year growth
    (revenue - LAG(revenue, 12) OVER (ORDER BY month_year)) * 100.0 / 
        LAG(revenue, 12) OVER (ORDER BY month_year) as yoy_growth
FROM monthly_revenue;
```

## Python Pandas Equivalents

### Basic LAG/LEAD
```python
# SQL: LAG(column) OVER (ORDER BY date)
df['prev_value'] = df.sort_values('date')['value'].shift(1)

# SQL: LEAD(column) OVER (ORDER BY date)  
df['next_value'] = df.sort_values('date')['value'].shift(-1)

# With grouping
df['prev_value'] = df.sort_values('date').groupby('customer_id')['value'].shift(1)
```

### Advanced Operations
```python
# Multiple period shifts
df = df.sort_values('date')
df['prev_month'] = df['revenue'].shift(1)
df['three_months_ago'] = df['revenue'].shift(3)
df['year_ago'] = df['revenue'].shift(12)

# Percentage changes
df['mom_pct'] = df['revenue'].pct_change() * 100
df['yoy_pct'] = df['revenue'].pct_change(periods=12) * 100

# With default values
df['prev_value'] = df['value'].shift(1).fillna(0)  # Default to 0
```

## Excel Equivalents

### Basic References
```excel
# LAG equivalent (previous row)
=OFFSET(B2,-1,0)  # Gets value from row above

# LEAD equivalent (next row)  
=OFFSET(B2,1,0)   # Gets value from row below

# With error handling
=IFERROR(OFFSET(B2,-1,0), 0)  # Default to 0 if no previous row
```

### Percentage Change
```excel
# Month-over-month growth
=(B2-B1)/B1*100

# With error handling for first row
=IF(ROW()=2, "", (B2-B1)/B1*100)
```

## Performance Considerations

### 1. **Index Strategy**
```sql
-- Create composite indexes for PARTITION BY + ORDER BY
CREATE INDEX idx_sales_store_date ON daily_sales(store_id, sale_date);
CREATE INDEX idx_events_user_timestamp ON user_events(user_id, event_timestamp);
```

### 2. **Window Frame Optimization**
```sql
-- Default frame is usually efficient
LAG(revenue) OVER (PARTITION BY store_id ORDER BY sale_date)

-- Avoid unnecessary frame specifications
-- This is redundant for LAG/LEAD:
LAG(revenue) OVER (
    PARTITION BY store_id 
    ORDER BY sale_date 
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- Unnecessary!
)
```

### 3. **Reusing Window Definitions**
```sql
-- Instead of repeating the OVER clause:
SELECT 
    sale_date,
    revenue,
    LAG(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date) as prev_day,
    LAG(revenue, 7) OVER (PARTITION BY store_id ORDER BY sale_date) as week_ago,
    LEAD(revenue, 1) OVER (PARTITION BY store_id ORDER BY sale_date) as next_day
FROM daily_sales;

-- Use WINDOW clause:
SELECT 
    sale_date,
    revenue,
    LAG(revenue, 1) OVER w as prev_day,
    LAG(revenue, 7) OVER w as week_ago,
    LEAD(revenue, 1) OVER w as next_day
FROM daily_sales
WINDOW w AS (PARTITION BY store_id ORDER BY sale_date);
```

## Common Interview Questions

### Q1: "Calculate month-over-month growth rate"
```sql
WITH monthly_growth AS (
    SELECT 
        month_year,
        revenue,
        LAG(revenue) OVER (ORDER BY month_year) as prev_month_revenue
    FROM monthly_sales
)
SELECT 
    month_year,
    revenue,
    prev_month_revenue,
    ROUND(
        (revenue - prev_month_revenue) * 100.0 / prev_month_revenue, 2
    ) as growth_rate_pct
FROM monthly_growth
WHERE prev_month_revenue IS NOT NULL;
```

### Q2: "Find the longest streak of increasing values"
```sql
WITH value_changes AS (
    SELECT 
        date_col,
        value,
        CASE 
            WHEN value > LAG(value) OVER (ORDER BY date_col) THEN 1 
            ELSE 0 
        END as is_increase
    FROM time_series
),
streak_groups AS (
    SELECT 
        date_col,
        value,
        is_increase,
        SUM(CASE WHEN is_increase = 0 THEN 1 ELSE 0 END) 
            OVER (ORDER BY date_col) as group_id
    FROM value_changes
)
SELECT 
    group_id,
    COUNT(*) as streak_length,
    MIN(date_col) as streak_start,
    MAX(date_col) as streak_end
FROM streak_groups
WHERE is_increase = 1
GROUP BY group_id
ORDER BY streak_length DESC
LIMIT 1;
```

### Q3: "Identify customer churn (no activity for 30+ days)"
```sql
WITH customer_activity AS (
    SELECT 
        customer_id,
        activity_date,
        LEAD(activity_date) OVER (
            PARTITION BY customer_id 
            ORDER BY activity_date
        ) as next_activity_date
    FROM customer_events
)
SELECT 
    customer_id,
    activity_date as last_activity,
    next_activity_date,
    next_activity_date - activity_date as days_inactive
FROM customer_activity
WHERE next_activity_date - activity_date > 30
   OR next_activity_date IS NULL;  -- No future activity
```

## When to Use LAG vs LEAD

| Scenario | Use Function | Example |
|----------|--------------|---------|
| Trend analysis | LAG() | Compare to previous period |
| Growth calculations | LAG() | Current vs previous value |
| Forecasting validation | LEAD() | Compare prediction to actual |
| Pipeline analysis | LEAD() | What happens next? |
| Cohort analysis | Both | Entry and exit behaviors |
| Time series gaps | LAG() | Find missing periods |

## Key Mental Models

1. **LAG() = "What happened before?"** - Perfect for trend analysis, growth calculations
2. **LEAD() = "What happens next?"** - Great for predicting outcomes, validating forecasts
3. **Both preserve row context** - Unlike self-joins, you stay in the current row while accessing others
4. **Offset parameter is powerful** - Look back/forward multiple periods easily
5. **Default values handle boundaries** - Control what happens at start/end of partitions

## The "Tiebreaker" Connection

Just like with ROW_NUMBER(), deterministic ordering is crucial:
```sql
-- ❌ Unpredictable if multiple records per day
LAG(revenue) OVER (PARTITION BY store_id ORDER BY sale_date)

-- ✅ Predictable with tiebreaker
LAG(revenue) OVER (PARTITION BY store_id ORDER BY sale_date, transaction_id)
```

## Key Takeaways

1. **LAG/LEAD eliminate complex self-joins** for accessing neighboring rows
2. **Essential for time series analysis** - trends, growth, patterns
3. **Offset parameter** lets you look multiple periods back/forward
4. **Default values** handle edge cases gracefully
5. **Performance depends on proper indexing** of partition and order columns
6. **Always ensure deterministic ordering** to avoid unpredictable results
7. **Combine with other window functions** for powerful analytics

These functions turn complex temporal analysis into simple, readable queries. Master them, and you'll solve most time-based data problems elegantly!