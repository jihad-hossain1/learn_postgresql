# Advanced Queries in PostgreSQL

## Introduction

Advanced SQL queries leverage PostgreSQL's powerful features to solve complex analytical problems, perform sophisticated data transformations, and optimize query performance.

## Window Functions

Window functions perform calculations across a set of table rows related to the current row, without collapsing the result set like aggregate functions.

### Basic Window Functions

```sql
-- Sample data setup
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    employee_id INTEGER,
    department VARCHAR(50),
    sale_amount DECIMAL(10,2),
    sale_date DATE,
    region VARCHAR(50)
);

INSERT INTO sales (employee_id, department, sale_amount, sale_date, region) VALUES
    (1, 'Electronics', 1500.00, '2023-01-15', 'North'),
    (2, 'Electronics', 2200.00, '2023-01-20', 'South'),
    (3, 'Clothing', 800.00, '2023-01-25', 'North'),
    (1, 'Electronics', 1800.00, '2023-02-10', 'North'),
    (4, 'Clothing', 1200.00, '2023-02-15', 'South'),
    (2, 'Electronics', 2500.00, '2023-02-20', 'South');

-- ROW_NUMBER: Assigns unique sequential numbers
SELECT 
    employee_id,
    department,
    sale_amount,
    ROW_NUMBER() OVER (ORDER BY sale_amount DESC) as overall_rank,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY sale_amount DESC) as dept_rank
FROM sales;

-- RANK and DENSE_RANK: Handle ties differently
SELECT 
    employee_id,
    sale_amount,
    RANK() OVER (ORDER BY sale_amount DESC) as rank_with_gaps,
    DENSE_RANK() OVER (ORDER BY sale_amount DESC) as dense_rank,
    ROW_NUMBER() OVER (ORDER BY sale_amount DESC) as row_num
FROM sales;
```

### Aggregate Window Functions

```sql
-- Running totals and moving averages
SELECT 
    sale_date,
    sale_amount,
    -- Running total
    SUM(sale_amount) OVER (ORDER BY sale_date) as running_total,
    -- Moving average (3-day window)
    AVG(sale_amount) OVER (
        ORDER BY sale_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3day,
    -- Cumulative count
    COUNT(*) OVER (ORDER BY sale_date) as cumulative_count
FROM sales
ORDER BY sale_date;

-- Department-wise aggregations
SELECT 
    employee_id,
    department,
    sale_amount,
    -- Total sales by department
    SUM(sale_amount) OVER (PARTITION BY department) as dept_total,
    -- Average sales by department
    AVG(sale_amount) OVER (PARTITION BY department) as dept_avg,
    -- Percentage of department total
    ROUND(
        (sale_amount / SUM(sale_amount) OVER (PARTITION BY department)) * 100, 2
    ) as pct_of_dept_total
FROM sales;
```

### LAG and LEAD Functions

```sql
-- Compare with previous and next values
SELECT 
    sale_date,
    sale_amount,
    -- Previous sale amount
    LAG(sale_amount) OVER (ORDER BY sale_date) as prev_sale,
    -- Next sale amount
    LEAD(sale_amount) OVER (ORDER BY sale_date) as next_sale,
    -- Difference from previous sale
    sale_amount - LAG(sale_amount) OVER (ORDER BY sale_date) as diff_from_prev,
    -- Growth rate
    ROUND(
        ((sale_amount - LAG(sale_amount) OVER (ORDER BY sale_date)) / 
         NULLIF(LAG(sale_amount) OVER (ORDER BY sale_date), 0)) * 100, 2
    ) as growth_rate_pct
FROM sales
ORDER BY sale_date;

-- Employee performance tracking
SELECT 
    employee_id,
    sale_date,
    sale_amount,
    LAG(sale_amount, 1) OVER (PARTITION BY employee_id ORDER BY sale_date) as prev_sale,
    LAG(sale_amount, 2) OVER (PARTITION BY employee_id ORDER BY sale_date) as two_sales_ago
FROM sales
ORDER BY employee_id, sale_date;
```

### FIRST_VALUE and LAST_VALUE

```sql
-- First and last values in window
SELECT 
    employee_id,
    department,
    sale_amount,
    sale_date,
    -- First sale in department
    FIRST_VALUE(sale_amount) OVER (
        PARTITION BY department 
        ORDER BY sale_date 
        ROWS UNBOUNDED PRECEDING
    ) as first_dept_sale,
    -- Last sale in department
    LAST_VALUE(sale_amount) OVER (
        PARTITION BY department 
        ORDER BY sale_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as last_dept_sale,
    -- Highest sale in department
    FIRST_VALUE(sale_amount) OVER (
        PARTITION BY department 
        ORDER BY sale_amount DESC 
        ROWS UNBOUNDED PRECEDING
    ) as highest_dept_sale
FROM sales;
```

### NTILE for Quartiles and Percentiles

```sql
-- Divide data into quartiles
SELECT 
    employee_id,
    sale_amount,
    NTILE(4) OVER (ORDER BY sale_amount) as quartile,
    NTILE(10) OVER (ORDER BY sale_amount) as decile,
    -- Percentile rank
    PERCENT_RANK() OVER (ORDER BY sale_amount) as percentile_rank,
    -- Cumulative distribution
    CUME_DIST() OVER (ORDER BY sale_amount) as cumulative_dist
FROM sales;

-- Performance categories
SELECT 
    employee_id,
    sale_amount,
    CASE NTILE(3) OVER (ORDER BY sale_amount DESC)
        WHEN 1 THEN 'Top Performer'
        WHEN 2 THEN 'Average Performer'
        WHEN 3 THEN 'Needs Improvement'
    END as performance_category
FROM sales;
```

## Common Table Expressions (CTEs)

### Basic CTEs

```sql
-- Simple CTE
WITH high_sales AS (
    SELECT employee_id, department, sale_amount
    FROM sales
    WHERE sale_amount > 1500
)
SELECT 
    department,
    COUNT(*) as high_sale_count,
    AVG(sale_amount) as avg_high_sale
FROM high_sales
GROUP BY department;

-- Multiple CTEs
WITH 
dept_totals AS (
    SELECT 
        department,
        SUM(sale_amount) as total_sales,
        COUNT(*) as sale_count
    FROM sales
    GROUP BY department
),
top_departments AS (
    SELECT department, total_sales
    FROM dept_totals
    WHERE total_sales > 3000
)
SELECT 
    td.department,
    td.total_sales,
    dt.sale_count,
    ROUND(td.total_sales / dt.sale_count, 2) as avg_sale
FROM top_departments td
JOIN dept_totals dt ON td.department = dt.department;
```

### Recursive CTEs

```sql
-- Employee hierarchy
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(employee_id),
    department VARCHAR(50),
    salary DECIMAL(10,2)
);

INSERT INTO employees (name, manager_id, department, salary) VALUES
    ('CEO', NULL, 'Executive', 200000),
    ('VP Sales', 1, 'Sales', 150000),
    ('VP Engineering', 1, 'Engineering', 160000),
    ('Sales Manager', 2, 'Sales', 100000),
    ('Engineer Lead', 3, 'Engineering', 120000),
    ('Sales Rep 1', 4, 'Sales', 60000),
    ('Sales Rep 2', 4, 'Sales', 65000),
    ('Engineer 1', 5, 'Engineering', 90000),
    ('Engineer 2', 5, 'Engineering', 85000);

-- Recursive query to get organizational hierarchy
WITH RECURSIVE org_chart AS (
    -- Base case: top-level employees (no manager)
    SELECT 
        employee_id,
        name,
        manager_id,
        department,
        salary,
        0 as level,
        name as path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: employees with managers
    SELECT 
        e.employee_id,
        e.name,
        e.manager_id,
        e.department,
        e.salary,
        oc.level + 1,
        oc.path || ' -> ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT 
    REPEAT('  ', level) || name as hierarchy,
    department,
    salary,
    level
FROM org_chart
ORDER BY path;

-- Calculate team sizes
WITH RECURSIVE team_sizes AS (
    -- Base case: leaf employees (no subordinates)
    SELECT 
        employee_id,
        name,
        1 as team_size
    FROM employees e1
    WHERE NOT EXISTS (
        SELECT 1 FROM employees e2 WHERE e2.manager_id = e1.employee_id
    )
    
    UNION ALL
    
    -- Recursive case: managers
    SELECT 
        e.employee_id,
        e.name,
        1 + SUM(ts.team_size)
    FROM employees e
    JOIN team_sizes ts ON ts.employee_id IN (
        SELECT employee_id FROM employees WHERE manager_id = e.employee_id
    )
    GROUP BY e.employee_id, e.name
)
SELECT name, team_size
FROM team_sizes
ORDER BY team_size DESC;
```

## Advanced Aggregations

### GROUPING SETS, CUBE, and ROLLUP

```sql
-- Sample data for analysis
CREATE TABLE product_sales (
    product_id INTEGER,
    category VARCHAR(50),
    region VARCHAR(50),
    quarter INTEGER,
    sales_amount DECIMAL(12,2)
);

INSERT INTO product_sales VALUES
    (1, 'Electronics', 'North', 1, 10000),
    (1, 'Electronics', 'North', 2, 12000),
    (1, 'Electronics', 'South', 1, 8000),
    (2, 'Clothing', 'North', 1, 5000),
    (2, 'Clothing', 'South', 1, 6000),
    (3, 'Books', 'North', 1, 3000);

-- GROUPING SETS: Specify exact grouping combinations
SELECT 
    category,
    region,
    quarter,
    SUM(sales_amount) as total_sales,
    GROUPING(category) as grp_category,
    GROUPING(region) as grp_region,
    GROUPING(quarter) as grp_quarter
FROM product_sales
GROUP BY GROUPING SETS (
    (category, region),
    (category, quarter),
    (region, quarter),
    ()
)
ORDER BY category, region, quarter;

-- ROLLUP: Hierarchical aggregations
SELECT 
    category,
    region,
    SUM(sales_amount) as total_sales,
    CASE 
        WHEN GROUPING(category) = 1 THEN 'Grand Total'
        WHEN GROUPING(region) = 1 THEN 'Category Total'
        ELSE 'Region Total'
    END as aggregation_level
FROM product_sales
GROUP BY ROLLUP(category, region)
ORDER BY category, region;

-- CUBE: All possible combinations
SELECT 
    category,
    region,
    quarter,
    SUM(sales_amount) as total_sales,
    COUNT(*) as record_count
FROM product_sales
GROUP BY CUBE(category, region, quarter)
ORDER BY category, region, quarter;
```

### FILTER Clause

```sql
-- Conditional aggregations
SELECT 
    region,
    -- Total sales
    SUM(sales_amount) as total_sales,
    -- Sales only from Q1
    SUM(sales_amount) FILTER (WHERE quarter = 1) as q1_sales,
    -- Sales only from Electronics
    SUM(sales_amount) FILTER (WHERE category = 'Electronics') as electronics_sales,
    -- Count of high-value sales
    COUNT(*) FILTER (WHERE sales_amount > 8000) as high_value_count,
    -- Average of non-Electronics sales
    AVG(sales_amount) FILTER (WHERE category != 'Electronics') as non_electronics_avg
FROM product_sales
GROUP BY region;
```

## Advanced Joins

### LATERAL Joins

```sql
-- LATERAL allows subqueries to reference columns from preceding tables
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    registration_date DATE
);

CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(customer_id),
    order_date DATE,
    total_amount DECIMAL(10,2)
);

-- Get top 3 orders for each customer
SELECT 
    c.name,
    top_orders.order_date,
    top_orders.total_amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_date, total_amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY total_amount DESC
    LIMIT 3
) top_orders;

-- Calculate running statistics
SELECT 
    c.name,
    c.registration_date,
    stats.total_orders,
    stats.total_spent,
    stats.avg_order_value
FROM customers c
CROSS JOIN LATERAL (
    SELECT 
        COUNT(*) as total_orders,
        SUM(total_amount) as total_spent,
        AVG(total_amount) as avg_order_value
    FROM orders o
    WHERE o.customer_id = c.customer_id
) stats;
```

### Self Joins

```sql
-- Find employees who earn more than their manager
SELECT 
    e.name as employee,
    e.salary as employee_salary,
    m.name as manager,
    m.salary as manager_salary,
    e.salary - m.salary as salary_difference
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;

-- Find employees at the same level (same manager)
SELECT 
    e1.name as employee1,
    e2.name as employee2,
    m.name as common_manager
FROM employees e1
JOIN employees e2 ON e1.manager_id = e2.manager_id AND e1.employee_id < e2.employee_id
JOIN employees m ON e1.manager_id = m.employee_id;
```

## Array and JSON Operations

### Array Functions

```sql
-- Working with arrays
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    tags TEXT[],
    ratings INTEGER[],
    metadata JSONB
);

INSERT INTO products (name, tags, ratings, metadata) VALUES
    ('Laptop', ARRAY['electronics', 'computer', 'portable'], ARRAY[5, 4, 5, 3, 4], 
     '{"brand": "TechCorp", "warranty": 2, "features": ["SSD", "16GB RAM"]}'),
    ('Smartphone', ARRAY['electronics', 'mobile', 'communication'], ARRAY[4, 5, 4, 4, 3],
     '{"brand": "PhoneCorp", "warranty": 1, "features": ["5G", "128GB"]}');

-- Array operations
SELECT 
    name,
    tags,
    -- Array length
    array_length(tags, 1) as tag_count,
    -- Check if array contains element
    'electronics' = ANY(tags) as is_electronics,
    -- Array to string
    array_to_string(tags, ', ') as tags_string,
    -- Array aggregations
    array_length(ratings, 1) as rating_count,
    (SELECT AVG(unnest) FROM unnest(ratings)) as avg_rating,
    (SELECT MAX(unnest) FROM unnest(ratings)) as max_rating
FROM products;

-- Array aggregation
SELECT 
    -- Aggregate arrays
    array_agg(name) as all_products,
    array_agg(DISTINCT tags[1]) as unique_first_tags,
    -- Aggregate ratings into single array
    array_agg(ratings) as all_ratings_arrays
FROM products;
```

### JSON/JSONB Operations

```sql
-- JSON extraction and manipulation
SELECT 
    name,
    -- Extract JSON values
    metadata->>'brand' as brand,
    metadata->'warranty' as warranty_json,
    (metadata->>'warranty')::INTEGER as warranty_years,
    -- Extract array elements
    metadata->'features' as features,
    metadata->'features'->>0 as first_feature,
    -- Check if key exists
    metadata ? 'warranty' as has_warranty,
    -- Check if contains value
    metadata @> '{"brand": "TechCorp"}' as is_techcorp
FROM products;

-- JSON aggregation
SELECT 
    json_agg(json_build_object(
        'name', name,
        'brand', metadata->>'brand',
        'avg_rating', (SELECT AVG(unnest) FROM unnest(ratings))
    )) as products_summary
FROM products;

-- Update JSON fields
UPDATE products 
SET metadata = metadata || '{"updated_at": "2023-12-01"}'
WHERE product_id = 1;

-- Remove JSON keys
UPDATE products 
SET metadata = metadata - 'updated_at'
WHERE product_id = 1;
```

## Text Search and Pattern Matching

### Full-Text Search

```sql
-- Create table for full-text search
CREATE TABLE articles (
    article_id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    author VARCHAR(100),
    published_date DATE,
    search_vector tsvector
);

-- Insert sample data
INSERT INTO articles (title, content, author, published_date) VALUES
    ('PostgreSQL Performance Tuning', 'Learn how to optimize PostgreSQL queries for better performance...', 'John Doe', '2023-01-15'),
    ('Advanced SQL Techniques', 'Explore window functions, CTEs, and other advanced SQL features...', 'Jane Smith', '2023-02-20'),
    ('Database Design Principles', 'Best practices for designing efficient and scalable databases...', 'Bob Johnson', '2023-03-10');

-- Update search vector
UPDATE articles 
SET search_vector = to_tsvector('english', title || ' ' || content);

-- Create GIN index for full-text search
CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);

-- Full-text search queries
SELECT 
    title,
    author,
    ts_rank(search_vector, to_tsquery('english', 'PostgreSQL & performance')) as rank
FROM articles
WHERE search_vector @@ to_tsquery('english', 'PostgreSQL & performance')
ORDER BY rank DESC;

-- Highlight search terms
SELECT 
    title,
    ts_headline('english', content, to_tsquery('english', 'SQL | database'), 
                'MaxWords=20, MinWords=5') as highlighted_content
FROM articles
WHERE search_vector @@ to_tsquery('english', 'SQL | database');
```

### Pattern Matching

```sql
-- Regular expressions
SELECT 
    name,
    -- Check if matches pattern
    name ~ '^[A-Z][a-z]+' as starts_with_capital,
    -- Extract pattern
    (regexp_match(name, '([A-Z][a-z]+)'))[1] as first_word,
    -- Replace pattern
    regexp_replace(name, '\s+', '_', 'g') as name_with_underscores,
    -- Split by pattern
    regexp_split_to_array(name, '\s+') as name_parts
FROM (
    VALUES ('John Doe'), ('jane smith'), ('Bob Johnson')
) AS people(name);

-- Similarity search using trigrams
CREATE EXTENSION IF NOT EXISTS pg_trgm;

SELECT 
    name,
    similarity(name, 'Jon Do') as similarity_score
FROM (
    VALUES ('John Doe'), ('Jane Smith'), ('Jon Davis'), ('Joan Doe')
) AS people(name)
WHERE similarity(name, 'Jon Do') > 0.3
ORDER BY similarity_score DESC;
```

## Date and Time Operations

### Advanced Date Functions

```sql
-- Date arithmetic and functions
SELECT 
    -- Current date/time
    CURRENT_DATE as today,
    CURRENT_TIME as now_time,
    CURRENT_TIMESTAMP as now_timestamp,
    NOW() as now_function,
    
    -- Date arithmetic
    CURRENT_DATE + INTERVAL '30 days' as thirty_days_later,
    CURRENT_DATE - INTERVAL '1 year' as one_year_ago,
    
    -- Extract parts
    EXTRACT(YEAR FROM CURRENT_DATE) as current_year,
    EXTRACT(MONTH FROM CURRENT_DATE) as current_month,
    EXTRACT(DOW FROM CURRENT_DATE) as day_of_week,
    EXTRACT(DOY FROM CURRENT_DATE) as day_of_year,
    
    -- Date truncation
    DATE_TRUNC('month', CURRENT_DATE) as first_of_month,
    DATE_TRUNC('week', CURRENT_DATE) as start_of_week,
    
    -- Age calculation
    AGE(CURRENT_DATE, '1990-01-01') as age_from_1990;

-- Generate date series
SELECT 
    date_series as date,
    EXTRACT(DOW FROM date_series) as day_of_week,
    TO_CHAR(date_series, 'Day') as day_name,
    TO_CHAR(date_series, 'Month') as month_name
FROM generate_series(
    '2023-01-01'::DATE,
    '2023-01-31'::DATE,
    '1 day'::INTERVAL
) as date_series
WHERE EXTRACT(DOW FROM date_series) IN (1, 7); -- Mondays and Sundays
```

### Time Zone Handling

```sql
-- Working with time zones
SELECT 
    -- Current timestamp in different zones
    NOW() as local_time,
    NOW() AT TIME ZONE 'UTC' as utc_time,
    NOW() AT TIME ZONE 'America/New_York' as ny_time,
    NOW() AT TIME ZONE 'Asia/Tokyo' as tokyo_time,
    
    -- Convert between zones
    '2023-06-15 14:30:00'::TIMESTAMP AT TIME ZONE 'America/Los_Angeles' AT TIME ZONE 'UTC' as la_to_utc;

-- Time zone functions
SELECT 
    name,
    abbrev,
    utc_offset
FROM pg_timezone_names
WHERE name LIKE '%New_York%' OR name LIKE '%Tokyo%'
ORDER BY name;
```

## Performance Optimization Techniques

### Query Optimization

```sql
-- Use EXISTS instead of IN for better performance
-- Instead of:
-- SELECT * FROM customers WHERE customer_id IN (SELECT customer_id FROM orders);

-- Use:
SELECT DISTINCT c.*
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- Use UNION ALL instead of UNION when duplicates are not an issue
SELECT name, 'customer' as type FROM customers
UNION ALL
SELECT name, 'employee' as type FROM employees;

-- Use LIMIT for large result sets
SELECT *
FROM large_table
ORDER BY created_date DESC
LIMIT 100;

-- Use appropriate JOIN types
-- INNER JOIN for required relationships
SELECT c.name, o.order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- LEFT JOIN when you need all records from left table
SELECT c.name, COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

### Subquery Optimization

```sql
-- Correlated vs Non-correlated subqueries
-- Correlated (can be slow)
SELECT *
FROM employees e1
WHERE salary > (
    SELECT AVG(salary)
    FROM employees e2
    WHERE e2.department = e1.department
);

-- Better: Use window functions
SELECT *
FROM (
    SELECT *,
           AVG(salary) OVER (PARTITION BY department) as dept_avg
    FROM employees
) e
WHERE salary > dept_avg;

-- Use CTEs for complex logic
WITH dept_averages AS (
    SELECT department, AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.*
FROM employees e
JOIN dept_averages da ON e.department = da.department
WHERE e.salary > da.avg_salary;
```

## Advanced Analytics Examples

### Cohort Analysis

```sql
-- Customer cohort analysis
WITH customer_cohorts AS (
    SELECT 
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) as cohort_month
    FROM orders
    GROUP BY customer_id
),
order_periods AS (
    SELECT 
        o.customer_id,
        cc.cohort_month,
        DATE_TRUNC('month', o.order_date) as order_month,
        (EXTRACT(YEAR FROM o.order_date) - EXTRACT(YEAR FROM cc.cohort_month)) * 12 +
        (EXTRACT(MONTH FROM o.order_date) - EXTRACT(MONTH FROM cc.cohort_month)) as period_number
    FROM orders o
    JOIN customer_cohorts cc ON o.customer_id = cc.customer_id
)
SELECT 
    cohort_month,
    period_number,
    COUNT(DISTINCT customer_id) as customers,
    SUM(COUNT(DISTINCT customer_id)) OVER (PARTITION BY cohort_month ORDER BY period_number) as cumulative_customers
FROM order_periods
GROUP BY cohort_month, period_number
ORDER BY cohort_month, period_number;
```

### RFM Analysis (Recency, Frequency, Monetary)

```sql
WITH customer_rfm AS (
    SELECT 
        customer_id,
        -- Recency: days since last order
        CURRENT_DATE - MAX(order_date) as recency_days,
        -- Frequency: number of orders
        COUNT(*) as frequency,
        -- Monetary: total spent
        SUM(total_amount) as monetary_value
    FROM orders
    GROUP BY customer_id
),
rfm_scores AS (
    SELECT 
        customer_id,
        recency_days,
        frequency,
        monetary_value,
        -- Score from 1-5 (5 is best)
        NTILE(5) OVER (ORDER BY recency_days DESC) as recency_score,
        NTILE(5) OVER (ORDER BY frequency) as frequency_score,
        NTILE(5) OVER (ORDER BY monetary_value) as monetary_score
    FROM customer_rfm
)
SELECT 
    customer_id,
    recency_score,
    frequency_score,
    monetary_score,
    CASE 
        WHEN recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4 THEN 'Champions'
        WHEN recency_score >= 3 AND frequency_score >= 3 AND monetary_score >= 3 THEN 'Loyal Customers'
        WHEN recency_score >= 3 AND frequency_score <= 2 THEN 'Potential Loyalists'
        WHEN recency_score <= 2 AND frequency_score >= 3 THEN 'At Risk'
        WHEN recency_score <= 2 AND frequency_score <= 2 THEN 'Lost Customers'
        ELSE 'Others'
    END as customer_segment
FROM rfm_scores
ORDER BY recency_score DESC, frequency_score DESC, monetary_score DESC;
```

## Best Practices for Advanced Queries

1. **Use CTEs for readability** - Break complex queries into logical steps
2. **Leverage window functions** - Avoid self-joins when possible
3. **Choose appropriate aggregation methods** - ROLLUP, CUBE, or GROUPING SETS
4. **Optimize subqueries** - Consider CTEs or JOINs as alternatives
5. **Use LATERAL joins** - For correlated subqueries that return multiple rows
6. **Index appropriately** - Support your complex queries with proper indexes
7. **Test performance** - Use EXPLAIN ANALYZE for complex queries
8. **Consider materialized views** - For expensive analytical queries

## Next Steps

After mastering advanced queries:
1. Stored Procedures and Functions (08-functions-procedures.md)
2. Transactions and Concurrency (09-transactions-concurrency.md)
3. Backup and Recovery (10-backup-recovery.md)

---
*This is part 7 of the PostgreSQL learning series*