# Indexes and Performance in PostgreSQL

## Introduction to Indexes

Indexes are database objects that improve the speed of data retrieval operations. They work like an index in a book, providing a fast path to find specific data without scanning the entire table.

## How Indexes Work

- **B-Tree Structure**: Most common index type, organized as a balanced tree
- **Index Pages**: Store key values and pointers to table rows
- **Query Planner**: PostgreSQL's optimizer decides whether to use indexes
- **Cost-Based**: Planner estimates costs and chooses the most efficient plan

## Types of Indexes

### B-Tree Indexes (Default)

```sql
-- Create sample table
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    department_id INTEGER,
    salary DECIMAL(10,2),
    hire_date DATE,
    is_active BOOLEAN DEFAULT TRUE
);

-- Basic B-Tree index
CREATE INDEX idx_employees_last_name ON employees(last_name);

-- Composite index
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary);

-- Partial index
CREATE INDEX idx_employees_active_salary ON employees(salary) 
WHERE is_active = TRUE;

-- Unique index
CREATE UNIQUE INDEX idx_employees_email ON employees(email);

-- Expression index
CREATE INDEX idx_employees_full_name ON employees(LOWER(first_name || ' ' || last_name));
```

### Hash Indexes

```sql
-- Hash index for equality comparisons only
CREATE INDEX idx_employees_dept_hash ON employees USING HASH(department_id);

-- Good for:
-- SELECT * FROM employees WHERE department_id = 5;
-- Not good for:
-- SELECT * FROM employees WHERE department_id > 5;
```

### GIN (Generalized Inverted Index)

```sql
-- For array and full-text search
CREATE TABLE documents (
    doc_id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    tags TEXT[],
    metadata JSONB
);

-- GIN index for arrays
CREATE INDEX idx_documents_tags ON documents USING GIN(tags);

-- GIN index for JSONB
CREATE INDEX idx_documents_metadata ON documents USING GIN(metadata);

-- GIN index for full-text search
CREATE INDEX idx_documents_content_fts ON documents USING GIN(to_tsvector('english', content));

-- Example queries
-- Array contains
SELECT * FROM documents WHERE tags @> ARRAY['postgresql'];

-- JSONB contains
SELECT * FROM documents WHERE metadata @> '{"category": "database"}';

-- Full-text search
SELECT * FROM documents WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & performance');
```

### GiST (Generalized Search Tree)

```sql
-- For geometric data and full-text search
CREATE TABLE locations (
    location_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    coordinates POINT,
    area POLYGON
);

-- GiST index for geometric data
CREATE INDEX idx_locations_coordinates ON locations USING GIST(coordinates);
CREATE INDEX idx_locations_area ON locations USING GIST(area);

-- Example geometric queries
SELECT * FROM locations WHERE coordinates <-> POINT(0,0) < 10;
SELECT * FROM locations WHERE area @> POINT(1,1);
```

### BRIN (Block Range Index)

```sql
-- For very large tables with natural ordering
CREATE TABLE sensor_data (
    reading_id BIGSERIAL PRIMARY KEY,
    sensor_id INTEGER,
    reading_value DECIMAL(10,4),
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- BRIN index for timestamp (naturally ordered)
CREATE INDEX idx_sensor_data_recorded_at ON sensor_data USING BRIN(recorded_at);

-- BRIN is very small but only effective for ordered data
-- Good for time-series data, log tables, etc.
```

## Index Management

### Creating Indexes

```sql
-- Basic syntax
CREATE [UNIQUE] INDEX [CONCURRENTLY] index_name 
ON table_name [USING method] (column1 [ASC|DESC], column2, ...);

-- Create index without blocking writes (for production)
CREATE INDEX CONCURRENTLY idx_employees_hire_date ON employees(hire_date);

-- Conditional index
CREATE INDEX idx_high_salary_employees ON employees(salary) 
WHERE salary > 100000;

-- Multi-column index with different sort orders
CREATE INDEX idx_employees_dept_salary_desc ON employees(department_id ASC, salary DESC);
```

### Viewing Indexes

```sql
-- List all indexes for a table
SELECT 
    indexname,
    indexdef
FROM pg_indexes 
WHERE tablename = 'employees';

-- Detailed index information
SELECT 
    schemaname,
    tablename,
    indexname,
    indexdef,
    indexsize
FROM pg_indexes 
JOIN pg_relation_size(indexname::regclass) AS indexsize ON true
WHERE tablename = 'employees';

-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_tup_read,
    idx_tup_fetch,
    idx_scan
FROM pg_stat_user_indexes 
WHERE tablename = 'employees';
```

### Dropping Indexes

```sql
-- Drop index
DROP INDEX idx_employees_last_name;

-- Drop index without blocking (for production)
DROP INDEX CONCURRENTLY idx_employees_last_name;

-- Drop index if exists
DROP INDEX IF EXISTS idx_employees_last_name;
```

## Query Performance Analysis

### EXPLAIN and EXPLAIN ANALYZE

```sql
-- Show query plan
EXPLAIN SELECT * FROM employees WHERE department_id = 5;

-- Show actual execution statistics
EXPLAIN ANALYZE SELECT * FROM employees WHERE department_id = 5;

-- More detailed output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM employees WHERE department_id = 5;

-- JSON format for programmatic analysis
EXPLAIN (ANALYZE, FORMAT JSON) 
SELECT * FROM employees WHERE department_id = 5;
```

### Understanding Query Plans

```sql
-- Sequential Scan (bad for large tables)
EXPLAIN SELECT * FROM employees WHERE first_name = 'John';
-- Result: Seq Scan on employees (cost=0.00..25.00 rows=1 width=100)

-- Index Scan (good)
CREATE INDEX idx_employees_first_name ON employees(first_name);
EXPLAIN SELECT * FROM employees WHERE first_name = 'John';
-- Result: Index Scan using idx_employees_first_name (cost=0.29..8.30 rows=1 width=100)

-- Bitmap Scan (for multiple matches)
EXPLAIN SELECT * FROM employees WHERE department_id IN (1, 2, 3);
-- Result: Bitmap Heap Scan on employees

-- Nested Loop Join
EXPLAIN SELECT e.first_name, d.name 
FROM employees e 
JOIN departments d ON e.department_id = d.department_id;

-- Hash Join (for larger datasets)
EXPLAIN SELECT e.first_name, d.name 
FROM employees e 
JOIN departments d ON e.department_id = d.department_id;
```

### Query Performance Metrics

```sql
-- Enable query statistics
SET track_activities = on;
SET track_counts = on;
SET track_io_timing = on;

-- View slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    rows
FROM pg_stat_statements 
ORDER BY total_time DESC 
LIMIT 10;

-- Current running queries
SELECT 
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
```

## Index Optimization Strategies

### Composite Index Column Order

```sql
-- Rule: Most selective columns first, then by query frequency

-- Good: department_id is more selective
CREATE INDEX idx_employees_dept_active ON employees(department_id, is_active);

-- Query can use this index efficiently
SELECT * FROM employees WHERE department_id = 5 AND is_active = TRUE;
SELECT * FROM employees WHERE department_id = 5; -- Can still use index

-- Bad: is_active is less selective
CREATE INDEX idx_employees_active_dept ON employees(is_active, department_id);
-- This query cannot use the index efficiently:
SELECT * FROM employees WHERE department_id = 5;
```

### Covering Indexes

```sql
-- Include additional columns to avoid table lookups
CREATE INDEX idx_employees_dept_covering 
ON employees(department_id) 
INCLUDE (first_name, last_name, salary);

-- This query can be satisfied entirely from the index
SELECT first_name, last_name, salary 
FROM employees 
WHERE department_id = 5;
```

### Partial Indexes

```sql
-- Index only relevant rows
CREATE INDEX idx_active_employees_salary 
ON employees(salary) 
WHERE is_active = TRUE;

-- Much smaller than full index, faster for queries on active employees
SELECT * FROM employees WHERE is_active = TRUE AND salary > 80000;

-- Index for specific conditions
CREATE INDEX idx_recent_orders 
ON orders(customer_id, order_date) 
WHERE order_date >= '2023-01-01';
```

### Expression Indexes

```sql
-- Index on computed values
CREATE INDEX idx_employees_full_name_lower 
ON employees(LOWER(first_name || ' ' || last_name));

-- Enables efficient case-insensitive searches
SELECT * FROM employees 
WHERE LOWER(first_name || ' ' || last_name) = 'john doe';

-- Index on date parts
CREATE INDEX idx_orders_year_month 
ON orders(EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date));

SELECT * FROM orders 
WHERE EXTRACT(YEAR FROM order_date) = 2023 
AND EXTRACT(MONTH FROM order_date) = 12;
```

## Performance Tuning

### Table Statistics

```sql
-- Update table statistics for better query planning
ANALYZE employees;

-- Analyze specific columns
ANALYZE employees(department_id, salary);

-- View table statistics
SELECT 
    schemaname,
    tablename,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables 
WHERE tablename = 'employees';
```

### Vacuum and Maintenance

```sql
-- Manual vacuum
VACUUM employees;

-- Vacuum with analyze
VACUUM ANALYZE employees;

-- Full vacuum (locks table, use carefully)
VACUUM FULL employees;

-- Reindex to rebuild indexes
REINDEX TABLE employees;
REINDEX INDEX idx_employees_last_name;
```

### Configuration Tuning

```sql
-- View current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW maintenance_work_mem;
SHOW effective_cache_size;

-- Common performance settings (in postgresql.conf)
/*
# Memory settings
shared_buffers = 256MB          # 25% of RAM
effective_cache_size = 1GB      # 75% of RAM
work_mem = 4MB                  # Per operation
maintenance_work_mem = 64MB     # For maintenance operations

# Checkpoint settings
checkpoint_completion_target = 0.9
wal_buffers = 16MB

# Query planner settings
random_page_cost = 1.1          # For SSDs
effective_io_concurrency = 200  # For SSDs
*/
```

## Advanced Performance Techniques

### Partitioning

```sql
-- Range partitioning by date
CREATE TABLE sales_data (
    sale_id SERIAL,
    sale_date DATE NOT NULL,
    customer_id INTEGER,
    amount DECIMAL(10,2),
    PRIMARY KEY (sale_id, sale_date)
) PARTITION BY RANGE (sale_date);

-- Create partitions
CREATE TABLE sales_data_2023_q1 PARTITION OF sales_data
FOR VALUES FROM ('2023-01-01') TO ('2023-04-01');

CREATE TABLE sales_data_2023_q2 PARTITION OF sales_data
FOR VALUES FROM ('2023-04-01') TO ('2023-07-01');

-- Indexes on partitions
CREATE INDEX idx_sales_2023_q1_customer ON sales_data_2023_q1(customer_id);
CREATE INDEX idx_sales_2023_q2_customer ON sales_data_2023_q2(customer_id);

-- Hash partitioning
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY,
    user_id INTEGER,
    created_at TIMESTAMP
) PARTITION BY HASH (user_id);

CREATE TABLE user_sessions_0 PARTITION OF user_sessions
FOR VALUES WITH (modulus 4, remainder 0);

CREATE TABLE user_sessions_1 PARTITION OF user_sessions
FOR VALUES WITH (modulus 4, remainder 1);
```

### Materialized Views

```sql
-- Create materialized view for expensive aggregations
CREATE MATERIALIZED VIEW monthly_sales_summary AS
SELECT 
    DATE_TRUNC('month', sale_date) as month,
    COUNT(*) as total_sales,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_sale_amount
FROM sales_data
GROUP BY DATE_TRUNC('month', sale_date)
ORDER BY month;

-- Create index on materialized view
CREATE INDEX idx_monthly_sales_month ON monthly_sales_summary(month);

-- Refresh materialized view
REFRESH MATERIALIZED VIEW monthly_sales_summary;

-- Refresh without blocking reads
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales_summary;
```

### Query Optimization Techniques

```sql
-- Use EXISTS instead of IN for better performance
-- Slow
SELECT * FROM customers 
WHERE customer_id IN (SELECT customer_id FROM orders WHERE order_date > '2023-01-01');

-- Faster
SELECT * FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id AND o.order_date > '2023-01-01');

-- Use LIMIT for large result sets
SELECT * FROM employees ORDER BY hire_date DESC LIMIT 10;

-- Use appropriate JOIN types
-- INNER JOIN when you need matching records
SELECT e.first_name, d.name
FROM employees e
INNER JOIN departments d ON e.department_id = d.department_id;

-- LEFT JOIN when you need all records from left table
SELECT e.first_name, COALESCE(d.name, 'No Department') as dept_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```

## Monitoring and Maintenance

### Index Usage Monitoring

```sql
-- Find unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes 
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find duplicate indexes
SELECT 
    pg_size_pretty(SUM(pg_relation_size(idx))::BIGINT) as size,
    (array_agg(idx))[1] as idx1, 
    (array_agg(idx))[2] as idx2
FROM (
    SELECT 
        indexrelid::regclass as idx, 
        (indrelid::regclass)::text as tablename, 
        regexp_split_to_array(indkey::text, ' ') as cols,
        array_length(regexp_split_to_array(indkey::text, ' '), 1) as ncols
    FROM pg_index
) sub
GROUP BY tablename, cols, ncols
HAVING COUNT(*) > 1;
```

### Performance Monitoring Queries

```sql
-- Table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as index_size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Index sizes
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes 
ORDER BY pg_relation_size(indexrelid) DESC;

-- Cache hit ratios
SELECT 
    'index hit rate' as name,
    (sum(idx_blks_hit)) / nullif(sum(idx_blks_hit + idx_blks_read),0) as ratio
FROM pg_statio_user_indexes
UNION ALL
SELECT 
    'table hit rate' as name,
    sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read),0) as ratio
FROM pg_statio_user_tables;
```

## Best Practices

### Index Design Guidelines

1. **Create indexes for WHERE clauses** - Index columns used in WHERE conditions
2. **Index foreign keys** - Always index foreign key columns
3. **Composite index order** - Most selective columns first
4. **Use partial indexes** - For queries with common WHERE conditions
5. **Consider covering indexes** - Include frequently selected columns
6. **Monitor index usage** - Remove unused indexes
7. **Use CONCURRENTLY** - For index creation/deletion in production

### Query Optimization Guidelines

1. **Use EXPLAIN ANALYZE** - Always analyze query performance
2. **Avoid SELECT *** - Select only needed columns
3. **Use appropriate data types** - Smaller types are faster
4. **Normalize appropriately** - Balance between normalization and performance
5. **Use LIMIT** - For large result sets
6. **Consider materialized views** - For expensive aggregations
7. **Update statistics regularly** - Keep table statistics current

### Common Anti-Patterns

```sql
-- Anti-pattern: Functions in WHERE clause
-- Slow
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH';
-- Better: Use expression index or store uppercase values

-- Anti-pattern: Leading wildcards
-- Slow
SELECT * FROM employees WHERE last_name LIKE '%smith';
-- Better: Use full-text search or trigram indexes

-- Anti-pattern: OR conditions on different columns
-- Slow
SELECT * FROM employees WHERE first_name = 'John' OR last_name = 'Smith';
-- Better: Use UNION or separate queries

-- Anti-pattern: NOT IN with NULLs
-- Problematic
SELECT * FROM employees WHERE department_id NOT IN (SELECT department_id FROM departments WHERE budget < 100000);
-- Better: Use NOT EXISTS or handle NULLs explicitly
```

## Performance Testing

```sql
-- Generate test data
INSERT INTO employees (first_name, last_name, email, department_id, salary, hire_date)
SELECT 
    'First' || i,
    'Last' || i,
    'user' || i || '@example.com',
    (i % 10) + 1,
    30000 + (random() * 70000)::INTEGER,
    '2020-01-01'::DATE + (random() * 1000)::INTEGER
FROM generate_series(1, 100000) i;

-- Benchmark queries
\timing on
SELECT COUNT(*) FROM employees WHERE department_id = 5;
SELECT COUNT(*) FROM employees WHERE salary > 80000;
SELECT * FROM employees WHERE last_name = 'Last12345';
\timing off

-- Use pgbench for load testing
-- pgbench -i -s 10 testdb  # Initialize
-- pgbench -c 10 -j 2 -t 1000 testdb  # Run benchmark
```

## Next Steps

After mastering indexes and performance:
1. Advanced Queries (07-advanced-queries.md)
2. Stored Procedures and Functions (08-functions-procedures.md)
3. Transactions and Concurrency (09-transactions-concurrency.md)

---
*This is part 6 of the PostgreSQL learning series*