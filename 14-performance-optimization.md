# PostgreSQL Performance Optimization

## Introduction

Performance optimization in PostgreSQL involves multiple layers: query optimization, index tuning, configuration adjustments, and system-level optimizations. This comprehensive guide covers advanced techniques to maximize PostgreSQL performance.

## Performance Analysis Methodology

### Performance Baseline Establishment

```sql
-- Create performance baseline table
CREATE TABLE performance_baseline (
    baseline_id SERIAL PRIMARY KEY,
    test_name VARCHAR(100),
    query_text TEXT,
    execution_time_ms NUMERIC,
    rows_returned BIGINT,
    buffers_hit BIGINT,
    buffers_read BIGINT,
    temp_files INTEGER,
    temp_bytes BIGINT,
    baseline_date TIMESTAMP DEFAULT NOW()
);

-- Baseline performance test function
CREATE OR REPLACE FUNCTION create_performance_baseline(
    test_name TEXT,
    query_text TEXT
)
RETURNS VOID
AS $$
DECLARE
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    execution_time NUMERIC;
    explain_result JSON;
BEGIN
    -- Clear cache for consistent testing
    PERFORM pg_stat_reset();
    
    start_time := clock_timestamp();
    
    -- Execute the query
    EXECUTE query_text;
    
    end_time := clock_timestamp();
    execution_time := EXTRACT(EPOCH FROM (end_time - start_time)) * 1000;
    
    -- Get execution plan with statistics
    EXECUTE 'EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ' || query_text INTO explain_result;
    
    -- Store baseline
    INSERT INTO performance_baseline (
        test_name,
        query_text,
        execution_time_ms,
        rows_returned,
        buffers_hit,
        buffers_read
    ) VALUES (
        test_name,
        query_text,
        execution_time,
        (explain_result->0->'Plan'->>'Actual Rows')::BIGINT,
        (explain_result->0->'Plan'->'Buffers'->>'Shared Hit Blocks')::BIGINT,
        (explain_result->0->'Plan'->'Buffers'->>'Shared Read Blocks')::BIGINT
    );
END;
$$ LANGUAGE plpgsql;

-- Performance comparison function
CREATE OR REPLACE FUNCTION compare_performance(
    test_name TEXT,
    query_text TEXT
)
RETURNS TABLE(
    baseline_time NUMERIC,
    current_time NUMERIC,
    improvement_percent NUMERIC,
    baseline_buffers BIGINT,
    current_buffers BIGINT
)
AS $$
DECLARE
    baseline_record RECORD;
    current_time NUMERIC;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
BEGIN
    -- Get baseline
    SELECT * INTO baseline_record
    FROM performance_baseline
    WHERE performance_baseline.test_name = compare_performance.test_name
    ORDER BY baseline_date DESC
    LIMIT 1;
    
    -- Execute current test
    start_time := clock_timestamp();
    EXECUTE query_text;
    end_time := clock_timestamp();
    current_time := EXTRACT(EPOCH FROM (end_time - start_time)) * 1000;
    
    RETURN QUERY
    SELECT 
        baseline_record.execution_time_ms,
        current_time,
        ROUND(((baseline_record.execution_time_ms - current_time) / baseline_record.execution_time_ms) * 100, 2),
        baseline_record.buffers_hit + baseline_record.buffers_read,
        0::BIGINT; -- Current buffers would need separate tracking
END;
$$ LANGUAGE plpgsql;
```

### Performance Monitoring Setup

```sql
-- Enable performance monitoring extensions
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pg_buffercache;

-- Performance monitoring view
CREATE VIEW performance_overview AS
SELECT 
    'Query Performance' as category,
    'Top 10 Slowest Queries' as metric,
    COUNT(*)::TEXT as value
FROM (
    SELECT query
    FROM pg_stat_statements
    ORDER BY mean_exec_time DESC
    LIMIT 10
) slow_queries

UNION ALL

SELECT 
    'Cache Performance' as category,
    'Cache Hit Ratio' as metric,
    ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2)::TEXT || '%' as value
FROM pg_stat_database
WHERE datname = current_database()

UNION ALL

SELECT 
    'Connection Performance' as category,
    'Active Connections' as metric,
    COUNT(*)::TEXT as value
FROM pg_stat_activity
WHERE state = 'active'

UNION ALL

SELECT 
    'Lock Performance' as category,
    'Lock Waits' as metric,
    COUNT(*)::TEXT as value
FROM pg_locks
WHERE granted = false;

-- Performance dashboard query
SELECT * FROM performance_overview;
```

## Query Optimization Techniques

### Advanced Query Rewriting

```sql
-- Example: Optimizing correlated subqueries
-- BEFORE: Correlated subquery (inefficient)
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    c.customer_id,
    c.customer_name,
    (
        SELECT COUNT(*)
        FROM orders o
        WHERE o.customer_id = c.customer_id
        AND o.order_date >= '2023-01-01'
    ) as order_count
FROM customers c
WHERE c.status = 'active';

-- AFTER: JOIN with aggregation (efficient)
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    c.customer_id,
    c.customer_name,
    COALESCE(o.order_count, 0) as order_count
FROM customers c
LEFT JOIN (
    SELECT 
        customer_id,
        COUNT(*) as order_count
    FROM orders
    WHERE order_date >= '2023-01-01'
    GROUP BY customer_id
) o ON c.customer_id = o.customer_id
WHERE c.status = 'active';

-- Example: Optimizing EXISTS vs IN
-- BEFORE: IN with subquery
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM customers c
WHERE c.customer_id IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE order_date >= '2023-01-01'
);

-- AFTER: EXISTS (often more efficient)
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
    AND o.order_date >= '2023-01-01'
);

-- Example: Window function optimization
-- BEFORE: Multiple aggregations
SELECT 
    customer_id,
    order_date,
    amount,
    (
        SELECT SUM(amount)
        FROM orders o2
        WHERE o2.customer_id = o1.customer_id
        AND o2.order_date <= o1.order_date
    ) as running_total
FROM orders o1
ORDER BY customer_id, order_date;

-- AFTER: Window function
SELECT 
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS UNBOUNDED PRECEDING
    ) as running_total
FROM orders
ORDER BY customer_id, order_date;
```

### Query Plan Optimization

```sql
-- Force specific join order
SET join_collapse_limit = 1;
SET from_collapse_limit = 1;

EXPLAIN (ANALYZE, BUFFERS)
SELECT /*+ Leading(c o p) */
    c.customer_name,
    o.order_date,
    p.product_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE c.region = 'North'
AND o.order_date >= '2023-01-01';

-- Reset join limits
RESET join_collapse_limit;
RESET from_collapse_limit;

-- Use CTEs for complex queries
WITH recent_orders AS (
    SELECT 
        customer_id,
        COUNT(*) as order_count,
        SUM(amount) as total_amount
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY customer_id
),
high_value_customers AS (
    SELECT customer_id
    FROM recent_orders
    WHERE total_amount > 1000
)
SELECT 
    c.customer_name,
    ro.order_count,
    ro.total_amount
FROM customers c
JOIN recent_orders ro ON c.customer_id = ro.customer_id
JOIN high_value_customers hvc ON c.customer_id = hvc.customer_id
ORDER BY ro.total_amount DESC;

-- Optimize LIMIT queries with ORDER BY
-- BEFORE: Inefficient for large offsets
SELECT *
FROM orders
ORDER BY order_date DESC
LIMIT 20 OFFSET 10000;

-- AFTER: Cursor-based pagination
SELECT *
FROM orders
WHERE order_date < '2023-06-15 10:30:00'  -- Last seen timestamp
ORDER BY order_date DESC
LIMIT 20;
```

### Parallel Query Optimization

```sql
-- Configure parallel query settings
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.1;
SET parallel_setup_cost = 1000;
SET min_parallel_table_scan_size = '8MB';
SET min_parallel_index_scan_size = '512kB';

-- Force parallel execution
SET force_parallel_mode = on;

EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    region,
    COUNT(*) as customer_count,
    AVG(total_orders) as avg_orders
FROM (
    SELECT 
        c.region,
        COUNT(o.order_id) as total_orders
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.region
) customer_stats
GROUP BY region;

RESET force_parallel_mode;

-- Parallel-safe custom functions
CREATE OR REPLACE FUNCTION calculate_discount(amount NUMERIC)
RETURNS NUMERIC
PARALLEL SAFE
AS $$
BEGIN
    RETURN CASE 
        WHEN amount > 1000 THEN amount * 0.1
        WHEN amount > 500 THEN amount * 0.05
        ELSE 0
    END;
END;
$$ LANGUAGE plpgsql;

-- Use parallel-safe function in queries
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    customer_id,
    SUM(amount) as total_amount,
    SUM(calculate_discount(amount)) as total_discount
FROM orders
WHERE order_date >= '2023-01-01'
GROUP BY customer_id;
```

## Index Optimization Strategies

### Advanced Index Types

```sql
-- Partial indexes for specific conditions
CREATE INDEX idx_orders_pending_recent 
ON orders (customer_id, order_date)
WHERE status = 'pending' 
AND order_date >= CURRENT_DATE - INTERVAL '30 days';

-- Expression indexes for computed values
CREATE INDEX idx_orders_total_value 
ON orders ((quantity * unit_price))
WHERE quantity * unit_price > 100;

-- Covering indexes to avoid table lookups
CREATE INDEX idx_customers_region_covering 
ON customers (region, status)
INCLUDE (customer_name, email, phone);

-- Multi-column indexes with optimal column order
-- Rule: Most selective column first, then by query frequency
CREATE INDEX idx_orders_optimized 
ON orders (status, customer_id, order_date)
WHERE status IN ('pending', 'processing');

-- Analyze index selectivity
SELECT 
    attname,
    n_distinct,
    CASE 
        WHEN n_distinct > 0 THEN n_distinct
        WHEN n_distinct < 0 THEN ABS(n_distinct) * reltuples
        ELSE 0
    END as estimated_distinct_values,
    reltuples as total_rows,
    CASE 
        WHEN reltuples > 0 THEN 
            ROUND((CASE 
                WHEN n_distinct > 0 THEN n_distinct
                WHEN n_distinct < 0 THEN ABS(n_distinct) * reltuples
                ELSE 0
            END / reltuples) * 100, 2)
        ELSE 0
    END as selectivity_percent
FROM pg_stats s
JOIN pg_class c ON s.tablename = c.relname
WHERE s.tablename = 'orders'
ORDER BY selectivity_percent DESC;
```

### Index Maintenance and Monitoring

```sql
-- Index usage analysis
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    CASE 
        WHEN idx_scan = 0 THEN 'Unused'
        WHEN idx_scan < 100 THEN 'Low Usage'
        WHEN idx_scan < 1000 THEN 'Medium Usage'
        ELSE 'High Usage'
    END as usage_category
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index bloat detection
WITH index_bloat AS (
    SELECT 
        schemaname,
        tablename,
        indexname,
        pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
        pg_relation_size(indexrelid) as size_bytes,
        idx_scan,
        CASE 
            WHEN pg_relation_size(indexrelid) > 100 * 1024 * 1024 -- > 100MB
            AND idx_scan < 1000 THEN 'Potential Bloat'
            ELSE 'OK'
        END as bloat_status
    FROM pg_stat_user_indexes
)
SELECT *
FROM index_bloat
WHERE bloat_status = 'Potential Bloat'
ORDER BY size_bytes DESC;

-- Duplicate index detection
WITH index_columns AS (
    SELECT 
        i.indexrelid,
        i.indrelid,
        c.relname as table_name,
        ci.relname as index_name,
        array_agg(a.attname ORDER BY a.attnum) as columns
    FROM pg_index i
    JOIN pg_class c ON i.indrelid = c.oid
    JOIN pg_class ci ON i.indexrelid = ci.oid
    JOIN pg_attribute a ON a.attrelid = i.indrelid AND a.attnum = ANY(i.indkey)
    WHERE c.relkind = 'r'
    AND ci.relkind = 'i'
    GROUP BY i.indexrelid, i.indrelid, c.relname, ci.relname
)
SELECT 
    table_name,
    array_agg(index_name) as duplicate_indexes,
    columns
FROM index_columns
GROUP BY table_name, columns
HAVING COUNT(*) > 1;

-- Index rebuild recommendation
CREATE OR REPLACE FUNCTION recommend_index_maintenance()
RETURNS TABLE(
    schema_name TEXT,
    table_name TEXT,
    index_name TEXT,
    recommendation TEXT,
    reason TEXT
)
AS $$
BEGIN
    RETURN QUERY
    -- Unused large indexes
    SELECT 
        s.schemaname::TEXT,
        s.tablename::TEXT,
        s.indexname::TEXT,
        'DROP'::TEXT,
        'Large unused index'::TEXT
    FROM pg_stat_user_indexes s
    WHERE s.idx_scan = 0
    AND pg_relation_size(s.indexrelid) > 50 * 1024 * 1024  -- > 50MB
    
    UNION ALL
    
    -- Low usage indexes
    SELECT 
        s.schemaname::TEXT,
        s.tablename::TEXT,
        s.indexname::TEXT,
        'REVIEW'::TEXT,
        'Low usage index'::TEXT
    FROM pg_stat_user_indexes s
    WHERE s.idx_scan < 100
    AND pg_relation_size(s.indexrelid) > 10 * 1024 * 1024  -- > 10MB
    
    UNION ALL
    
    -- High maintenance indexes
    SELECT 
        s.schemaname::TEXT,
        s.tablename::TEXT,
        s.indexname::TEXT,
        'REINDEX'::TEXT,
        'High maintenance required'::TEXT
    FROM pg_stat_user_indexes s
    JOIN pg_stat_user_tables t ON s.relid = t.relid
    WHERE t.n_tup_upd + t.n_tup_del > s.idx_scan * 10
    AND pg_relation_size(s.indexrelid) > 100 * 1024 * 1024;  -- > 100MB
END;
$$ LANGUAGE plpgsql;

SELECT * FROM recommend_index_maintenance();
```

## Configuration Optimization

### Memory Configuration

```sql
-- Memory configuration calculator
CREATE OR REPLACE FUNCTION calculate_memory_settings(
    total_ram_gb NUMERIC,
    workload_type TEXT DEFAULT 'mixed'  -- 'oltp', 'olap', 'mixed'
)
RETURNS TABLE(
    parameter TEXT,
    recommended_value TEXT,
    current_value TEXT,
    description TEXT
)
AS $$
DECLARE
    shared_buffers_gb NUMERIC;
    effective_cache_gb NUMERIC;
    work_mem_mb NUMERIC;
    maintenance_work_mem_mb NUMERIC;
BEGIN
    -- Calculate recommendations based on workload
    CASE workload_type
        WHEN 'oltp' THEN
            shared_buffers_gb := total_ram_gb * 0.25;
            effective_cache_gb := total_ram_gb * 0.75;
            work_mem_mb := GREATEST(4, total_ram_gb * 2);
            maintenance_work_mem_mb := GREATEST(64, total_ram_gb * 16);
        WHEN 'olap' THEN
            shared_buffers_gb := total_ram_gb * 0.4;
            effective_cache_gb := total_ram_gb * 0.8;
            work_mem_mb := GREATEST(64, total_ram_gb * 8);
            maintenance_work_mem_mb := GREATEST(256, total_ram_gb * 32);
        ELSE  -- mixed
            shared_buffers_gb := total_ram_gb * 0.3;
            effective_cache_gb := total_ram_gb * 0.75;
            work_mem_mb := GREATEST(16, total_ram_gb * 4);
            maintenance_work_mem_mb := GREATEST(128, total_ram_gb * 24);
    END CASE;
    
    RETURN QUERY
    SELECT 
        'shared_buffers'::TEXT,
        shared_buffers_gb::TEXT || 'GB',
        current_setting('shared_buffers'),
        'Amount of memory for shared buffer cache'::TEXT
    
    UNION ALL
    
    SELECT 
        'effective_cache_size'::TEXT,
        effective_cache_gb::TEXT || 'GB',
        current_setting('effective_cache_size'),
        'Estimate of OS cache size'::TEXT
    
    UNION ALL
    
    SELECT 
        'work_mem'::TEXT,
        work_mem_mb::TEXT || 'MB',
        current_setting('work_mem'),
        'Memory for sort and hash operations'::TEXT
    
    UNION ALL
    
    SELECT 
        'maintenance_work_mem'::TEXT,
        maintenance_work_mem_mb::TEXT || 'MB',
        current_setting('maintenance_work_mem'),
        'Memory for maintenance operations'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Get recommendations for 16GB system with OLTP workload
SELECT * FROM calculate_memory_settings(16, 'oltp');
```

### Connection and Resource Configuration

```sql
-- Connection pooling configuration analysis
WITH connection_stats AS (
    SELECT 
        COUNT(*) as total_connections,
        COUNT(*) FILTER (WHERE state = 'active') as active_connections,
        COUNT(*) FILTER (WHERE state = 'idle') as idle_connections,
        COUNT(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction,
        AVG(EXTRACT(EPOCH FROM (NOW() - backend_start))) as avg_connection_age
    FROM pg_stat_activity
    WHERE pid <> pg_backend_pid()
),
config_settings AS (
    SELECT 
        setting::INTEGER as max_connections
    FROM pg_settings
    WHERE name = 'max_connections'
)
SELECT 
    cs.total_connections,
    cs.active_connections,
    cs.idle_connections,
    cs.idle_in_transaction,
    ROUND(cs.avg_connection_age, 2) as avg_connection_age_seconds,
    cfg.max_connections,
    ROUND((cs.total_connections::FLOAT / cfg.max_connections) * 100, 2) as connection_usage_percent,
    CASE 
        WHEN cs.total_connections > cfg.max_connections * 0.8 THEN 'HIGH - Consider connection pooling'
        WHEN cs.idle_in_transaction > 10 THEN 'WARNING - Many idle in transaction'
        WHEN cs.avg_connection_age > 3600 THEN 'WARNING - Long-lived connections'
        ELSE 'OK'
    END as recommendation
FROM connection_stats cs, config_settings cfg;

-- Optimal configuration for different workloads
CREATE OR REPLACE FUNCTION get_workload_config(
    workload_type TEXT,
    concurrent_users INTEGER
)
RETURNS TABLE(
    parameter TEXT,
    value TEXT,
    explanation TEXT
)
AS $$
BEGIN
    CASE workload_type
        WHEN 'web_app' THEN
            RETURN QUERY VALUES
            ('max_connections', (concurrent_users * 2)::TEXT, 'Web apps typically need 2x user connections'),
            ('shared_buffers', '256MB', 'Moderate buffer cache for web workloads'),
            ('work_mem', '16MB', 'Small work_mem for many concurrent queries'),
            ('checkpoint_completion_target', '0.9', 'Spread checkpoints for consistent performance'),
            ('wal_buffers', '16MB', 'Adequate WAL buffering'),
            ('random_page_cost', '1.1', 'SSD-optimized random access cost');
            
        WHEN 'analytics' THEN
            RETURN QUERY VALUES
            ('max_connections', LEAST(concurrent_users, 50)::TEXT, 'Analytics needs fewer but resource-intensive connections'),
            ('shared_buffers', '2GB', 'Large buffer cache for data scanning'),
            ('work_mem', '256MB', 'Large work_mem for complex aggregations'),
            ('maintenance_work_mem', '1GB', 'Large maintenance memory for index operations'),
            ('effective_cache_size', '6GB', 'Assume large OS cache'),
            ('random_page_cost', '1.1', 'SSD-optimized for large scans');
            
        WHEN 'mixed' THEN
            RETURN QUERY VALUES
            ('max_connections', (concurrent_users * 1.5)::TEXT, 'Balanced connection count'),
            ('shared_buffers', '512MB', 'Balanced buffer cache'),
            ('work_mem', '64MB', 'Moderate work_mem'),
            ('maintenance_work_mem', '256MB', 'Moderate maintenance memory'),
            ('checkpoint_completion_target', '0.7', 'Balanced checkpoint spreading'),
            ('wal_buffers', '16MB', 'Standard WAL buffering');
    END CASE;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM get_workload_config('web_app', 100);
```

### WAL and Checkpoint Optimization

```sql
-- WAL configuration analysis
WITH wal_stats AS (
    SELECT 
        pg_current_wal_lsn() as current_lsn,
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 as wal_mb_generated,
        setting::INTEGER as max_wal_size_mb
    FROM pg_settings
    WHERE name = 'max_wal_size'
),
checkpoint_stats AS (
    SELECT 
        checkpoints_timed,
        checkpoints_req,
        checkpoint_write_time,
        checkpoint_sync_time,
        buffers_checkpoint,
        buffers_clean,
        buffers_backend
    FROM pg_stat_bgwriter
)
SELECT 
    ws.wal_mb_generated,
    ws.max_wal_size_mb,
    cs.checkpoints_timed,
    cs.checkpoints_req,
    ROUND((cs.checkpoints_req::FLOAT / (cs.checkpoints_timed + cs.checkpoints_req)) * 100, 2) as forced_checkpoint_percent,
    cs.checkpoint_write_time,
    cs.checkpoint_sync_time,
    CASE 
        WHEN cs.checkpoints_req > cs.checkpoints_timed THEN 'Increase max_wal_size'
        WHEN cs.checkpoint_write_time > 60000 THEN 'Increase checkpoint_completion_target'
        ELSE 'OK'
    END as recommendation
FROM wal_stats ws, checkpoint_stats cs;

-- Optimal WAL settings calculator
CREATE OR REPLACE FUNCTION calculate_wal_settings(
    write_workload TEXT,  -- 'light', 'moderate', 'heavy'
    storage_type TEXT     -- 'hdd', 'ssd', 'nvme'
)
RETURNS TABLE(
    parameter TEXT,
    recommended_value TEXT,
    current_value TEXT
)
AS $$
DECLARE
    wal_buffers_mb INTEGER;
    max_wal_size_mb INTEGER;
    checkpoint_target NUMERIC;
BEGIN
    -- Calculate based on workload and storage
    CASE write_workload
        WHEN 'light' THEN
            wal_buffers_mb := 16;
            max_wal_size_mb := 1024;  -- 1GB
        WHEN 'moderate' THEN
            wal_buffers_mb := 32;
            max_wal_size_mb := 4096;  -- 4GB
        WHEN 'heavy' THEN
            wal_buffers_mb := 64;
            max_wal_size_mb := 8192;  -- 8GB
    END CASE;
    
    -- Adjust for storage type
    CASE storage_type
        WHEN 'hdd' THEN
            checkpoint_target := 0.9;  -- Spread checkpoints more
        WHEN 'ssd' THEN
            checkpoint_target := 0.7;  -- Balanced
        WHEN 'nvme' THEN
            checkpoint_target := 0.5;  -- Can handle more frequent checkpoints
    END CASE;
    
    RETURN QUERY
    SELECT 
        'wal_buffers'::TEXT,
        wal_buffers_mb::TEXT || 'MB',
        current_setting('wal_buffers')
    
    UNION ALL
    
    SELECT 
        'max_wal_size'::TEXT,
        max_wal_size_mb::TEXT || 'MB',
        current_setting('max_wal_size')
    
    UNION ALL
    
    SELECT 
        'checkpoint_completion_target'::TEXT,
        checkpoint_target::TEXT,
        current_setting('checkpoint_completion_target');
END;
$$ LANGUAGE plpgsql;

SELECT * FROM calculate_wal_settings('moderate', 'ssd');
```

## System-Level Optimization

### Operating System Tuning

```bash
#!/bin/bash
# postgresql_os_tuning.sh

echo "PostgreSQL Operating System Tuning Script"
echo "=========================================="

# Check current settings
echo "Current OS Settings:"
echo "Shared Memory: $(cat /proc/sys/kernel/shmmax)"
echo "Semaphores: $(cat /proc/sys/kernel/sem)"
echo "File Descriptors: $(ulimit -n)"
echo "Virtual Memory: $(cat /proc/sys/vm/overcommit_memory)"

# Recommended sysctl settings for PostgreSQL
cat > /etc/sysctl.d/99-postgresql.conf << EOF
# PostgreSQL Kernel Parameters

# Shared Memory
kernel.shmmax = 68719476736  # 64GB
kernel.shmall = 4294967296   # 16TB

# Semaphores (SEMMSL, SEMMNS, SEMOPM, SEMMNI)
kernel.sem = 250 32000 100 128

# Virtual Memory
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
vm.swappiness = 1
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10
vm.dirty_expire_centisecs = 6000
vm.dirty_writeback_centisecs = 500

# Network
net.core.rmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_default = 262144
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 65536 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# File System
fs.file-max = 2097152
EOF

# Apply settings
sysctl -p /etc/sysctl.d/99-postgresql.conf

# Set limits for postgres user
cat > /etc/security/limits.d/99-postgresql.conf << EOF
postgres soft nofile 65536
postgres hard nofile 65536
postgres soft nproc 8192
postgres hard nproc 8192
EOF

# I/O Scheduler optimization for SSDs
echo "Optimizing I/O scheduler for SSDs..."
for disk in /sys/block/sd*; do
    if [ -f "$disk/queue/scheduler" ]; then
        echo "noop" > "$disk/queue/scheduler"
        echo "Set noop scheduler for $(basename $disk)"
    fi
done

# Transparent Huge Pages (disable for PostgreSQL)
echo "Disabling Transparent Huge Pages..."
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Add to startup script
cat > /etc/systemd/system/disable-thp.service << EOF
[Unit]
Description=Disable Transparent Huge Pages
DefaultDependencies=false
After=sysinit.target local-fs.target
Before=postgresql.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF

systemctl enable disable-thp.service

echo "OS tuning completed. Reboot recommended."
```

### Storage Optimization

```sql
-- Storage performance analysis
CREATE OR REPLACE FUNCTION analyze_storage_performance()
RETURNS TABLE(
    metric TEXT,
    value TEXT,
    recommendation TEXT
)
AS $$
DECLARE
    total_reads BIGINT;
    total_writes BIGINT;
    cache_hit_ratio NUMERIC;
    temp_file_usage BIGINT;
BEGIN
    -- Get I/O statistics
    SELECT 
        SUM(heap_blks_read + idx_blks_read),
        SUM(heap_blks_hit + idx_blks_hit)
    INTO total_reads, total_writes
    FROM pg_statio_user_tables;
    
    -- Calculate cache hit ratio
    cache_hit_ratio := (total_writes::FLOAT / (total_reads + total_writes)) * 100;
    
    -- Get temp file usage
    SELECT SUM(temp_bytes) INTO temp_file_usage
    FROM pg_stat_database;
    
    RETURN QUERY
    SELECT 
        'Cache Hit Ratio'::TEXT,
        ROUND(cache_hit_ratio, 2)::TEXT || '%',
        CASE 
            WHEN cache_hit_ratio < 95 THEN 'Increase shared_buffers or add more RAM'
            WHEN cache_hit_ratio < 98 THEN 'Consider tuning work_mem'
            ELSE 'Good'
        END
    
    UNION ALL
    
    SELECT 
        'Temp File Usage'::TEXT,
        pg_size_pretty(temp_file_usage),
        CASE 
            WHEN temp_file_usage > 1024*1024*1024 THEN 'Increase work_mem to reduce temp file usage'
            ELSE 'Acceptable'
        END;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM analyze_storage_performance();

-- Tablespace optimization
CREATE TABLESPACE fast_ssd LOCATION '/mnt/fast_ssd';
CREATE TABLESPACE slow_hdd LOCATION '/mnt/slow_hdd';

-- Move frequently accessed tables to fast storage
ALTER TABLE orders SET TABLESPACE fast_ssd;
ALTER TABLE customers SET TABLESPACE fast_ssd;

-- Move archive tables to slower storage
ALTER TABLE order_history SET TABLESPACE slow_hdd;
ALTER TABLE audit_log SET TABLESPACE slow_hdd;

-- Create indexes on fast storage
CREATE INDEX CONCURRENTLY idx_orders_fast 
ON orders (customer_id, order_date) 
TABLESPACE fast_ssd;
```

## Application-Level Optimization

### Connection Management

```python
# Python connection pooling example
import psycopg2
from psycopg2 import pool
import threading
import time

class DatabaseManager:
    def __init__(self, min_conn=5, max_conn=20):
        self.connection_pool = psycopg2.pool.ThreadedConnectionPool(
            min_conn, max_conn,
            host="localhost",
            database="mydb",
            user="postgres",
            password="password",
            # Connection optimization
            options="-c default_transaction_isolation=read_committed "
                   "-c statement_timeout=30000 "
                   "-c idle_in_transaction_session_timeout=60000"
        )
        self.local = threading.local()
    
    def get_connection(self):
        if not hasattr(self.local, 'connection'):
            self.local.connection = self.connection_pool.getconn()
        return self.local.connection
    
    def return_connection(self):
        if hasattr(self.local, 'connection'):
            self.connection_pool.putconn(self.local.connection)
            del self.local.connection
    
    def execute_query(self, query, params=None):
        conn = self.get_connection()
        try:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                if cursor.description:
                    return cursor.fetchall()
                conn.commit()
        except Exception as e:
            conn.rollback()
            raise e
        finally:
            self.return_connection()

# Usage example
db = DatabaseManager()

# Optimized query with prepared statement
result = db.execute_query(
    "SELECT * FROM orders WHERE customer_id = %s AND order_date >= %s",
    (1001, '2023-01-01')
)
```

### Query Optimization Patterns

```sql
-- Batch processing optimization
CREATE OR REPLACE FUNCTION process_orders_batch(
    batch_size INTEGER DEFAULT 1000
)
RETURNS INTEGER
AS $$
DECLARE
    processed_count INTEGER := 0;
    batch_count INTEGER;
BEGIN
    LOOP
        -- Process batch
        WITH batch_orders AS (
            SELECT order_id
            FROM orders
            WHERE status = 'pending'
            ORDER BY order_date
            LIMIT batch_size
            FOR UPDATE SKIP LOCKED
        )
        UPDATE orders
        SET 
            status = 'processing',
            processed_at = NOW()
        FROM batch_orders
        WHERE orders.order_id = batch_orders.order_id;
        
        GET DIAGNOSTICS batch_count = ROW_COUNT;
        processed_count := processed_count + batch_count;
        
        -- Exit if no more rows to process
        EXIT WHEN batch_count = 0;
        
        -- Commit batch
        COMMIT;
        
        -- Small delay to prevent overwhelming the system
        PERFORM pg_sleep(0.1);
    END LOOP;
    
    RETURN processed_count;
END;
$$ LANGUAGE plpgsql;

-- Efficient pagination
CREATE OR REPLACE FUNCTION get_orders_page(
    cursor_id BIGINT DEFAULT NULL,
    page_size INTEGER DEFAULT 20
)
RETURNS TABLE(
    order_id BIGINT,
    customer_id INTEGER,
    order_date DATE,
    amount DECIMAL,
    next_cursor BIGINT
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        o.order_id,
        o.customer_id,
        o.order_date,
        o.amount,
        LEAD(o.order_id) OVER (ORDER BY o.order_id) as next_cursor
    FROM orders o
    WHERE (cursor_id IS NULL OR o.order_id > cursor_id)
    ORDER BY o.order_id
    LIMIT page_size;
END;
$$ LANGUAGE plpgsql;

-- Efficient aggregation with materialized views
CREATE MATERIALIZED VIEW customer_order_summary AS
SELECT 
    customer_id,
    COUNT(*) as total_orders,
    SUM(amount) as total_amount,
    AVG(amount) as avg_order_value,
    MAX(order_date) as last_order_date,
    MIN(order_date) as first_order_date
FROM orders
GROUP BY customer_id;

CREATE UNIQUE INDEX idx_customer_summary_id 
ON customer_order_summary (customer_id);

-- Refresh materialized view efficiently
CREATE OR REPLACE FUNCTION refresh_customer_summary()
RETURNS VOID
AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY customer_order_summary;
END;
$$ LANGUAGE plpgsql;

-- Schedule refresh
SELECT cron.schedule('refresh-summary', '0 */6 * * *', 'SELECT refresh_customer_summary();');
```

## Performance Testing and Benchmarking

### pgbench Configuration

```bash
#!/bin/bash
# postgresql_benchmark.sh

DB_NAME="benchmark_db"
DB_USER="postgres"
SCALE_FACTOR=100  # Adjust based on your needs
CLIENTS=50
THREADS=4
DURATION=300  # 5 minutes

echo "PostgreSQL Performance Benchmark"
echo "================================"

# Initialize benchmark database
echo "Initializing benchmark database..."
pgbench -i -s $SCALE_FACTOR -d $DB_NAME -U $DB_USER

# Run standard benchmark
echo "Running standard TPC-B benchmark..."
pgbench -c $CLIENTS -j $THREADS -T $DURATION -P 10 -r $DB_NAME -U $DB_USER

# Custom benchmark script
cat > custom_benchmark.sql << EOF
\set customer_id random(1, 100000)
\set amount random(10, 1000)
BEGIN;
INSERT INTO orders (customer_id, order_date, amount, status) 
VALUES (:customer_id, CURRENT_DATE, :amount, 'pending');
SELECT COUNT(*) FROM orders WHERE customer_id = :customer_id;
COMMIT;
EOF

# Run custom benchmark
echo "Running custom benchmark..."
pgbench -c $CLIENTS -j $THREADS -T $DURATION -f custom_benchmark.sql $DB_NAME -U $DB_USER

# Cleanup
rm custom_benchmark.sql

echo "Benchmark completed"
```

### Performance Regression Testing

```sql
-- Performance regression test framework
CREATE TABLE performance_tests (
    test_id SERIAL PRIMARY KEY,
    test_name VARCHAR(100),
    test_query TEXT,
    expected_max_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE performance_results (
    result_id SERIAL PRIMARY KEY,
    test_id INTEGER REFERENCES performance_tests(test_id),
    execution_time_ms NUMERIC,
    rows_returned BIGINT,
    buffers_hit BIGINT,
    buffers_read BIGINT,
    test_date TIMESTAMP DEFAULT NOW(),
    passed BOOLEAN
);

-- Add performance tests
INSERT INTO performance_tests (test_name, test_query, expected_max_time_ms) VALUES
('Customer Order Count', 'SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id', 1000),
('Recent Orders', 'SELECT * FROM orders WHERE order_date >= CURRENT_DATE - INTERVAL ''30 days''', 500),
('Customer Search', 'SELECT * FROM customers WHERE customer_name ILIKE ''%smith%''', 200);

-- Performance test runner
CREATE OR REPLACE FUNCTION run_performance_tests()
RETURNS TABLE(
    test_name TEXT,
    execution_time_ms NUMERIC,
    expected_max_ms INTEGER,
    passed BOOLEAN,
    performance_ratio NUMERIC
)
AS $$
DECLARE
    test_record RECORD;
    start_time TIMESTAMP;
    end_time TIMESTAMP;
    execution_time NUMERIC;
    test_passed BOOLEAN;
BEGIN
    FOR test_record IN SELECT * FROM performance_tests ORDER BY test_id LOOP
        -- Execute test query
        start_time := clock_timestamp();
        EXECUTE test_record.test_query;
        end_time := clock_timestamp();
        
        execution_time := EXTRACT(EPOCH FROM (end_time - start_time)) * 1000;
        test_passed := execution_time <= test_record.expected_max_time_ms;
        
        -- Store result
        INSERT INTO performance_results (
            test_id, 
            execution_time_ms, 
            passed
        ) VALUES (
            test_record.test_id,
            execution_time,
            test_passed
        );
        
        -- Return result
        RETURN QUERY
        SELECT 
            test_record.test_name::TEXT,
            execution_time,
            test_record.expected_max_time_ms,
            test_passed,
            ROUND(execution_time / test_record.expected_max_time_ms, 2);
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Run tests
SELECT * FROM run_performance_tests();

-- Performance trend analysis
SELECT 
    pt.test_name,
    DATE_TRUNC('day', pr.test_date) as test_day,
    AVG(pr.execution_time_ms) as avg_execution_time,
    MIN(pr.execution_time_ms) as min_execution_time,
    MAX(pr.execution_time_ms) as max_execution_time,
    COUNT(*) as test_runs
FROM performance_results pr
JOIN performance_tests pt ON pr.test_id = pt.test_id
WHERE pr.test_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY pt.test_name, DATE_TRUNC('day', pr.test_date)
ORDER BY pt.test_name, test_day;
```

## Monitoring and Alerting

### Performance Monitoring Dashboard

```sql
-- Real-time performance dashboard
CREATE VIEW performance_dashboard AS
WITH current_activity AS (
    SELECT 
        COUNT(*) as total_connections,
        COUNT(*) FILTER (WHERE state = 'active') as active_connections,
        COUNT(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction,
        AVG(EXTRACT(EPOCH FROM (NOW() - query_start))) FILTER (WHERE state = 'active') as avg_query_time
    FROM pg_stat_activity
    WHERE pid <> pg_backend_pid()
),
cache_stats AS (
    SELECT 
        ROUND((blks_hit::FLOAT / NULLIF(blks_hit + blks_read, 0)) * 100, 2) as cache_hit_ratio
    FROM pg_stat_database
    WHERE datname = current_database()
),
lock_stats AS (
    SELECT 
        COUNT(*) as lock_waits
    FROM pg_locks
    WHERE granted = false
),
checkpoint_stats AS (
    SELECT 
        checkpoints_timed + checkpoints_req as total_checkpoints,
        ROUND((checkpoints_req::FLOAT / NULLIF(checkpoints_timed + checkpoints_req, 0)) * 100, 2) as forced_checkpoint_ratio
    FROM pg_stat_bgwriter
)
SELECT 
    'Connections' as category,
    jsonb_build_object(
        'total', ca.total_connections,
        'active', ca.active_connections,
        'idle_in_transaction', ca.idle_in_transaction,
        'avg_query_time_seconds', ROUND(ca.avg_query_time, 2)
    ) as metrics
FROM current_activity ca

UNION ALL

SELECT 
    'Cache Performance' as category,
    jsonb_build_object(
        'hit_ratio_percent', cs.cache_hit_ratio
    ) as metrics
FROM cache_stats cs

UNION ALL

SELECT 
    'Locks' as category,
    jsonb_build_object(
        'waiting_locks', ls.lock_waits
    ) as metrics
FROM lock_stats ls

UNION ALL

SELECT 
    'Checkpoints' as category,
    jsonb_build_object(
        'total_checkpoints', cps.total_checkpoints,
        'forced_ratio_percent', cps.forced_checkpoint_ratio
    ) as metrics
FROM checkpoint_stats cps;

-- Query the dashboard
SELECT 
    category,
    jsonb_pretty(metrics) as performance_metrics
FROM performance_dashboard;
```

## Best Practices Summary

### Query Optimization
1. **Use EXPLAIN ANALYZE** for all performance-critical queries
2. **Optimize WHERE clauses** with proper indexing
3. **Avoid SELECT *** in production queries
4. **Use appropriate JOIN types** and order
5. **Leverage window functions** instead of correlated subqueries
6. **Implement proper pagination** for large result sets

### Index Strategy
1. **Create indexes** for frequently queried columns
2. **Use composite indexes** for multi-column queries
3. **Consider partial indexes** for filtered queries
4. **Monitor index usage** and remove unused indexes
5. **Use covering indexes** to avoid table lookups

### Configuration Tuning
1. **Size shared_buffers** appropriately (25-40% of RAM)
2. **Set work_mem** based on workload and concurrency
3. **Configure checkpoint settings** for your storage
4. **Tune parallel query settings** for analytical workloads
5. **Monitor and adjust** based on actual usage patterns

### System Optimization
1. **Use appropriate storage** (SSD for performance-critical data)
2. **Configure OS parameters** for PostgreSQL
3. **Implement connection pooling** for high-concurrency applications
4. **Monitor system resources** (CPU, memory, I/O)
5. **Plan for capacity growth** and scaling

## Next Steps

After mastering performance optimization:
1. High Availability and Clustering (15-high-availability.md)
2. PostgreSQL in Production (16-production-deployment.md)
3. Advanced Topics and Extensions (17-advanced-topics.md)

---
*This is part 14 of the PostgreSQL learning series*