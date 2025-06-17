# ROW_NUMBER(), RANK(), DENSE_RANK() - Complete Deep Dive5

## The Core Concept: What Are These Functions?

These are **window functions** that assign numbers to rows based on some ordering criteria. Think of them as different ways to "number" your data rows, each with specific rules about how they handle ties (duplicate values).

### The Mental Model
Imagine you're ranking students in a class by their test scores:
- **ROW_NUMBER()**: Every student gets a unique number, even if they have the same score
- **RANK()**: Students with the same score get the same rank, but we skip numbers afterward
- **DENSE_RANK()**: Students with the same score get the same rank, but we DON'T skip numbers

## Function Syntax & Behavior

```sql
ROW_NUMBER() OVER (PARTITION BY column ORDER BY column)
RANK() OVER (PARTITION BY column ORDER BY column) 
DENSE_RANK() OVER (PARTITION BY column ORDER BY column)
```

### Key Components:
- **OVER()**: Defines the window (scope of rows to consider)
- **PARTITION BY**: Divides data into groups (optional)
- **ORDER BY**: Defines the ranking criteria (required)

## Detailed Breakdown

### ROW_NUMBER()
**What it does**: Assigns a unique sequential integer to each row
**Ties handling**: No ties - every row gets a different number
**Use when**: You need unique identifiers or want to pick exactly N rows

### RANK()
**What it does**: Assigns the same rank to rows with identical values
**Ties handling**: Gaps in sequence (1, 2, 2, 4, 5...)
**Use when**: Traditional ranking like sports leaderboards

### DENSE_RANK()
**What it does**: Same rank for identical values, but no gaps
**Ties handling**: No gaps in sequence (1, 2, 2, 3, 4...)
**Use when**: You want consecutive ranking numbers

## Real-World Examples

### Example 1: Employee Salary Ranking
```sql
-- Sample data: employees table
CREATE TABLE employees (
    emp_id INT,
    name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10,2)
);

-- Ranking queries
SELECT 
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num,
    RANK() OVER (ORDER BY salary DESC) as rank_pos,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank_pos
FROM employees;
```

**Result with sample data:**
| Name | Department | Salary | ROW_NUMBER | RANK | DENSE_RANK |
|------|------------|--------|------------|------|------------|
| Alice | Engineering | 95000 | 1 | 1 | 1 |
| Bob | Engineering | 90000 | 2 | 2 | 2 |
| Carol | Marketing | 90000 | 3 | 2 | 2 |
| David | Sales | 85000 | 4 | 4 | 3 |

### Example 2: Top N Per Group
```sql
-- Get top 3 highest paid employees per department
WITH ranked_employees AS (
    SELECT 
        name,
        department,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
)
SELECT * FROM ranked_employees WHERE rn <= 3;
```

## Popular Use Cases

### 1. **Pagination & Data Sampling**
```sql
-- Get rows 11-20 for pagination
WITH numbered_rows AS (
    SELECT *, ROW_NUMBER() OVER (ORDER BY created_date) as rn
    FROM products
)
SELECT * FROM numbered_rows WHERE rn BETWEEN 11 AND 20;
```

### 2. **Removing Duplicates**
```sql
-- Remove duplicate customers, keeping the latest record
WITH dedupe AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY updated_date DESC) as rn
    FROM customers
)
SELECT * FROM dedupe WHERE rn = 1;
```

### 3. **Finding Top/Bottom N**
```sql
-- Top 5 products by revenue in each category
WITH product_ranks AS (
    SELECT 
        product_name,
        category,
        revenue,
        DENSE_RANK() OVER (PARTITION BY category ORDER BY revenue DESC) as rank_pos
    FROM product_sales
)
SELECT * FROM product_ranks WHERE rank_pos <= 5;
```

### 4. **Percentile Analysis**
```sql
-- Divide customers into quartiles by spending
SELECT 
    customer_id,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent) as quartile,
    DENSE_RANK() OVER (ORDER BY total_spent DESC) as spending_rank
FROM customer_spending;
```

## Python Pandas Equivalents

### ROW_NUMBER()
```python
# SQL: ROW_NUMBER() OVER (ORDER BY salary DESC)
df['row_num'] = df.sort_values('salary', ascending=False).reset_index(drop=True).index + 1

# With grouping
df['row_num'] = df.sort_values('salary', ascending=False).groupby('department').cumcount() + 1
```

### RANK()
```python
# SQL: RANK() OVER (ORDER BY salary DESC)
df['rank'] = df['salary'].rank(method='min', ascending=False)

# With grouping
df['rank'] = df.groupby('department')['salary'].rank(method='min', ascending=False)
```

### DENSE_RANK()
```python
# SQL: DENSE_RANK() OVER (ORDER BY salary DESC)
df['dense_rank'] = df['salary'].rank(method='dense', ascending=False)

# With grouping
df['dense_rank'] = df.groupby('department')['salary'].rank(method='dense', ascending=False)
```

## Excel Equivalents

### ROW_NUMBER()
```excel
=ROW() - 1  # Simple sequential numbering
=COUNTIF($A$1:A1, A1)  # For grouped numbering
```

### RANK()
```excel
=RANK(B2, $B$2:$B$10, 0)  # 0 for descending order
```

### DENSE_RANK()
Excel doesn't have a direct equivalent, but you can use:
```excel
=RANK(B2, $B$2:$B$10, 0) - COUNTIF($B$2:B1, ">"&B2)
```

## Performance Considerations

### 1. **Index Strategy**
- Create indexes on columns used in ORDER BY
- Composite indexes for PARTITION BY + ORDER BY

### 2. **Memory Usage**
- Window functions can be memory-intensive
- Consider limiting result sets with WHERE clauses

### 3. **Query Optimization**
```sql
-- GOOD: Filter first, then rank
WITH filtered_data AS (
    SELECT * FROM large_table WHERE date >= '2024-01-01'
)
SELECT *, ROW_NUMBER() OVER (ORDER BY amount DESC) as rn
FROM filtered_data;

-- AVOID: Ranking all data then filtering
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY amount DESC) as rn
    FROM large_table
) WHERE date >= '2024-01-01';
```

## Common Interview Questions

### Q1: "Get the 2nd highest salary in each department"
```sql
WITH salary_ranks AS (
    SELECT 
        emp_id, 
        department, 
        salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank_pos
    FROM employees
)
SELECT * FROM salary_ranks WHERE rank_pos = 2;
```

### Q2: "Find duplicate records"
```sql
WITH duplicate_check AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_date) as rn
    FROM users
)
SELECT * FROM duplicate_check WHERE rn > 1;
```

### Q3: "Calculate running totals with rankings"
```sql
SELECT 
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) as running_total,
    ROW_NUMBER() OVER (ORDER BY order_date) as day_number,
    RANK() OVER (ORDER BY amount DESC) as amount_rank
FROM daily_sales;
```

## Advanced Patterns

### 1. **Gap and Island Analysis**
```sql
-- Find consecutive sequences
WITH gaps AS (
    SELECT 
        id,
        id - ROW_NUMBER() OVER (ORDER BY id) as grp
    FROM sequence_table
)
SELECT 
    MIN(id) as start_id,
    MAX(id) as end_id,
    COUNT(*) as sequence_length
FROM gaps
GROUP BY grp;
```

### 2. **Median Calculation**
```sql
WITH ranked_values AS (
    SELECT 
        value,
        ROW_NUMBER() OVER (ORDER BY value) as rn,
        COUNT(*) OVER () as total_count
    FROM measurements
)
SELECT AVG(value) as median
FROM ranked_values
WHERE rn IN ((total_count + 1) / 2, (total_count + 2) / 2);
```

## When to Use Which Function

| Scenario | Use Function | Why |
|----------|--------------|-----|
| Pagination | ROW_NUMBER() | Need unique sequential numbers |
| Top N per group | ROW_NUMBER() | Guarantees exactly N rows |
| Sports rankings | RANK() | Traditional ranking with gaps |
| Grade distributions | DENSE_RANK() | Consecutive ranking levels |
| Duplicate removal | ROW_NUMBER() | Unique identifier for each row |
| Percentile analysis | RANK() or DENSE_RANK() | Depending on gap preference |

## Key Takeaways

1. **ROW_NUMBER()** is your go-to for unique sequential numbering
2. **RANK()** mimics traditional ranking systems (think Olympics)
3. **DENSE_RANK()** provides consecutive numbering without gaps
4. All three require ORDER BY clause
5. PARTITION BY creates separate ranking groups
6. Performance depends heavily on indexing strategy
7. These functions are essential for data analysis and extremely common in interviews

Remember: Master these three functions, and you'll solve 80% of ranking-related problems in SQL!