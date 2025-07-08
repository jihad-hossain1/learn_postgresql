# Advanced PostgreSQL Administration

## Introduction

Advanced PostgreSQL administration involves managing complex database environments, optimizing performance, implementing advanced features, and handling enterprise-level requirements. This guide covers advanced administrative topics and techniques.

## Advanced Configuration Management

### Configuration File Structure

```bash
# Main configuration files
/etc/postgresql/15/main/postgresql.conf    # Main configuration
/etc/postgresql/15/main/pg_hba.conf        # Host-based authentication
/etc/postgresql/15/main/pg_ident.conf      # User name mapping
/etc/postgresql/15/main/postgresql.auto.conf  # Auto-generated settings
```

### Advanced Configuration Parameters

```sql
-- Memory configuration
SHOW shared_buffers;                    -- Shared buffer cache
SHOW effective_cache_size;              -- OS cache estimate
SHOW work_mem;                          -- Sort/hash memory per operation
SHOW maintenance_work_mem;              -- Maintenance operation memory
SHOW max_wal_size;                      -- WAL size before checkpoint
SHOW checkpoint_completion_target;      -- Checkpoint spread time

-- Connection and resource limits
SHOW max_connections;                   -- Maximum connections
SHOW max_worker_processes;              -- Background worker processes
SHOW max_parallel_workers;              -- Parallel query workers
SHOW max_parallel_workers_per_gather;   -- Workers per gather node

-- Query planning
SHOW random_page_cost;                  -- Random page access cost
SHOW seq_page_cost;                     -- Sequential page access cost
SHOW cpu_tuple_cost;                    -- CPU processing cost per tuple
SHOW cpu_index_tuple_cost;              -- CPU cost per index tuple
SHOW cpu_operator_cost;                 -- CPU cost per operator

-- WAL and checkpointing
SHOW wal_level;                         -- WAL detail level
SHOW wal_buffers;                       -- WAL buffer size
SHOW checkpoint_timeout;                -- Maximum checkpoint interval
SHOW checkpoint_warning;                -- Checkpoint frequency warning

-- Logging configuration
SHOW log_min_duration_statement;        -- Log slow queries
SHOW log_checkpoints;                   -- Log checkpoint activity
SHOW log_lock_waits;                    -- Log lock waits
SHOW log_temp_files;                    -- Log temporary file usage
```

### Dynamic Configuration Changes

```sql
-- View configuration settings
SELECT 
    name,
    setting,
    unit,
    context,
    vartype,
    source,
    min_val,
    max_val,
    boot_val,
    reset_val,
    pending_restart
FROM pg_settings
WHERE name LIKE '%work_mem%'
ORDER BY name;

-- Change settings dynamically
SET work_mem = '256MB';                 -- Session level
ALTER SYSTEM SET work_mem = '128MB';    -- System level (requires reload)
SELECT pg_reload_conf();                -- Reload configuration

-- Database-specific settings
ALTER DATABASE mydb SET work_mem = '256MB';
ALTER DATABASE mydb SET random_page_cost = 1.1;

-- User-specific settings
ALTER USER analyst SET work_mem = '512MB';
ALTER USER app_user SET statement_timeout = '30s';

-- Function-specific settings
CREATE OR REPLACE FUNCTION heavy_calculation()
RETURNS INTEGER
SET work_mem = '1GB'
SET temp_tablespaces = 'temp_space'
AS $$
BEGIN
    -- Function implementation
    RETURN 1;
END;
$$ LANGUAGE plpgsql;
```

### Configuration Tuning Guidelines

```sql
-- Memory tuning calculation
WITH system_info AS (
    SELECT 
        setting::BIGINT * 8192 AS shared_buffers_bytes,
        (SELECT setting::BIGINT FROM pg_settings WHERE name = 'effective_cache_size') * 8192 AS effective_cache_bytes
    FROM pg_settings 
    WHERE name = 'shared_buffers'
)
SELECT 
    pg_size_pretty(shared_buffers_bytes) AS shared_buffers,
    pg_size_pretty(effective_cache_bytes) AS effective_cache_size,
    ROUND(shared_buffers_bytes::NUMERIC / (1024^3), 2) AS shared_buffers_gb,
    ROUND(effective_cache_bytes::NUMERIC / (1024^3), 2) AS effective_cache_gb
FROM system_info;

-- Recommended settings based on system memory
-- For 8GB RAM system:
-- shared_buffers = 2GB (25% of RAM)
-- effective_cache_size = 6GB (75% of RAM)
-- work_mem = 64MB (for OLTP) or 256MB+ (for analytics)
-- maintenance_work_mem = 512MB

-- For 32GB RAM system:
-- shared_buffers = 8GB
-- effective_cache_size = 24GB
-- work_mem = 128MB (OLTP) or 512MB+ (analytics)
-- maintenance_work_mem = 2GB
```

## Extensions Management

### Core Extensions

```sql
-- List available extensions
SELECT 
    name,
    default_version,
    installed_version,
    comment
FROM pg_available_extensions
ORDER BY name;

-- List installed extensions
SELECT 
    extname,
    extversion,
    nspname AS schema
FROM pg_extension e
JOIN pg_namespace n ON e.extnamespace = n.oid
ORDER BY extname;

-- Install common extensions
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS uuid-ossp;
CREATE EXTENSION IF NOT EXISTS hstore;
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gin;
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Update extension
ALTER EXTENSION pg_stat_statements UPDATE;

-- Drop extension
DROP EXTENSION IF EXISTS extension_name CASCADE;
```

### Popular Extensions

#### pg_stat_statements

```sql
-- Configure pg_stat_statements
-- Add to postgresql.conf:
-- shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.max = 10000
-- pg_stat_statements.track = all

CREATE EXTENSION pg_stat_statements;

-- Query performance analysis
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

#### PostGIS (Spatial Data)

```sql
-- Install PostGIS
CREATE EXTENSION postgis;
CREATE EXTENSION postgis_topology;

-- Create spatial table
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    geom GEOMETRY(POINT, 4326)
);

-- Create spatial index
CREATE INDEX idx_locations_geom ON locations USING GIST (geom);

-- Insert spatial data
INSERT INTO locations (name, geom) VALUES 
('New York', ST_GeomFromText('POINT(-74.006 40.7128)', 4326)),
('London', ST_GeomFromText('POINT(-0.1276 51.5074)', 4326));

-- Spatial queries
SELECT 
    name,
    ST_AsText(geom) AS coordinates,
    ST_Distance(geom, ST_GeomFromText('POINT(-74.006 40.7128)', 4326)) AS distance_from_ny
FROM locations
ORDER BY distance_from_ny;
```

#### pg_cron (Job Scheduling)

```sql
-- Install pg_cron
CREATE EXTENSION pg_cron;

-- Schedule maintenance job
SELECT cron.schedule('vacuum-tables', '0 2 * * *', 'VACUUM ANALYZE;');

-- Schedule statistics update
SELECT cron.schedule('update-stats', '0 */6 * * *', 'ANALYZE;');

-- Schedule cleanup job
SELECT cron.schedule(
    'cleanup-logs', 
    '0 1 * * 0', 
    'DELETE FROM application_logs WHERE created_at < NOW() - INTERVAL ''30 days'';'
);

-- View scheduled jobs
SELECT 
    jobid,
    schedule,
    command,
    nodename,
    nodeport,
    database,
    username,
    active
FROM cron.job;

-- View job run history
SELECT 
    jobid,
    runid,
    job_pid,
    database,
    username,
    command,
    status,
    return_message,
    start_time,
    end_time
FROM cron.job_run_details
ORDER BY start_time DESC
LIMIT 10;

-- Unschedule job
SELECT cron.unschedule('vacuum-tables');
```

## Table Partitioning

### Range Partitioning

```sql
-- Create partitioned table
CREATE TABLE sales (
    id BIGSERIAL,
    sale_date DATE NOT NULL,
    customer_id INTEGER,
    amount DECIMAL(10,2),
    product_id INTEGER
) PARTITION BY RANGE (sale_date);

-- Create partitions
CREATE TABLE sales_2023_q1 PARTITION OF sales
    FOR VALUES FROM ('2023-01-01') TO ('2023-04-01');

CREATE TABLE sales_2023_q2 PARTITION OF sales
    FOR VALUES FROM ('2023-04-01') TO ('2023-07-01');

CREATE TABLE sales_2023_q3 PARTITION OF sales
    FOR VALUES FROM ('2023-07-01') TO ('2023-10-01');

CREATE TABLE sales_2023_q4 PARTITION OF sales
    FOR VALUES FROM ('2023-10-01') TO ('2024-01-01');

-- Create default partition
CREATE TABLE sales_default PARTITION OF sales DEFAULT;

-- Create indexes on partitions
CREATE INDEX idx_sales_2023_q1_customer ON sales_2023_q1 (customer_id);
CREATE INDEX idx_sales_2023_q1_product ON sales_2023_q1 (product_id);

-- Insert data
INSERT INTO sales (sale_date, customer_id, amount, product_id) VALUES
('2023-02-15', 1001, 299.99, 501),
('2023-05-20', 1002, 149.50, 502),
('2023-08-10', 1003, 89.99, 503),
('2023-11-25', 1004, 199.99, 504);

-- Query with partition pruning
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM sales 
WHERE sale_date BETWEEN '2023-07-01' AND '2023-09-30';

-- View partition information
SELECT 
    schemaname,
    tablename,
    partitionbounds
FROM pg_tables 
WHERE tablename LIKE 'sales_%'
ORDER BY tablename;
```

### Hash Partitioning

```sql
-- Create hash partitioned table
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY,
    user_id INTEGER NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    data JSONB
) PARTITION BY HASH (user_id);

-- Create hash partitions
CREATE TABLE user_sessions_0 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE user_sessions_1 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE user_sessions_2 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE user_sessions_3 PARTITION OF user_sessions
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Insert test data
INSERT INTO user_sessions (session_id, user_id, data)
SELECT 
    gen_random_uuid(),
    (random() * 10000)::INTEGER,
    jsonb_build_object('ip', '192.168.1.' || (random() * 255)::INTEGER)
FROM generate_series(1, 10000);

-- Check partition distribution
SELECT 
    schemaname,
    tablename,
    n_tup_ins as inserted_rows
FROM pg_stat_user_tables 
WHERE tablename LIKE 'user_sessions_%'
ORDER BY tablename;
```

### List Partitioning

```sql
-- Create list partitioned table
CREATE TABLE orders (
    order_id BIGSERIAL,
    region VARCHAR(20) NOT NULL,
    customer_id INTEGER,
    order_date DATE,
    amount DECIMAL(10,2)
) PARTITION BY LIST (region);

-- Create list partitions
CREATE TABLE orders_north PARTITION OF orders
    FOR VALUES IN ('North', 'Northeast', 'Northwest');

CREATE TABLE orders_south PARTITION OF orders
    FOR VALUES IN ('South', 'Southeast', 'Southwest');

CREATE TABLE orders_east PARTITION OF orders
    FOR VALUES IN ('East', 'Eastern');

CREATE TABLE orders_west PARTITION OF orders
    FOR VALUES IN ('West', 'Western');

-- Insert data
INSERT INTO orders (region, customer_id, order_date, amount) VALUES
('North', 1001, '2023-06-15', 299.99),
('South', 1002, '2023-06-16', 149.50),
('East', 1003, '2023-06-17', 89.99),
('West', 1004, '2023-06-18', 199.99);
```

### Partition Management

```sql
-- Automated partition creation function
CREATE OR REPLACE FUNCTION create_monthly_partitions(
    table_name TEXT,
    start_date DATE,
    end_date DATE
)
RETURNS VOID
AS $$
DECLARE
    current_date DATE := start_date;
    next_date DATE;
    partition_name TEXT;
    sql_command TEXT;
BEGIN
    WHILE current_date < end_date LOOP
        next_date := current_date + INTERVAL '1 month';
        partition_name := table_name || '_' || to_char(current_date, 'YYYY_MM');
        
        sql_command := format(
            'CREATE TABLE IF NOT EXISTS %I PARTITION OF %I FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            table_name,
            current_date,
            next_date
        );
        
        EXECUTE sql_command;
        RAISE NOTICE 'Created partition: %', partition_name;
        
        current_date := next_date;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Create partitions for the next 12 months
SELECT create_monthly_partitions('sales', '2024-01-01', '2025-01-01');

-- Drop old partitions
CREATE OR REPLACE FUNCTION drop_old_partitions(
    table_name TEXT,
    retention_months INTEGER
)
RETURNS VOID
AS $$
DECLARE
    partition_record RECORD;
    cutoff_date DATE := CURRENT_DATE - (retention_months || ' months')::INTERVAL;
BEGIN
    FOR partition_record IN
        SELECT 
            schemaname,
            tablename
        FROM pg_tables
        WHERE tablename LIKE table_name || '_%'
        AND tablename ~ '\d{4}_\d{2}$'
    LOOP
        -- Extract date from partition name and check if it's old
        IF to_date(right(partition_record.tablename, 7), 'YYYY_MM') < cutoff_date THEN
            EXECUTE format('DROP TABLE IF EXISTS %I.%I', 
                          partition_record.schemaname, 
                          partition_record.tablename);
            RAISE NOTICE 'Dropped old partition: %', partition_record.tablename;
        END IF;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Drop partitions older than 24 months
SELECT drop_old_partitions('sales', 24);
```

## Advanced Indexing Strategies

### Partial Indexes

```sql
-- Index only active records
CREATE INDEX idx_users_active_email 
ON users (email) 
WHERE status = 'active';

-- Index only recent orders
CREATE INDEX idx_orders_recent 
ON orders (customer_id, order_date) 
WHERE order_date >= CURRENT_DATE - INTERVAL '1 year';

-- Index only non-null values
CREATE INDEX idx_users_phone 
ON users (phone) 
WHERE phone IS NOT NULL;
```

### Expression Indexes

```sql
-- Index on function result
CREATE INDEX idx_users_lower_email 
ON users (lower(email));

-- Index on extracted JSON field
CREATE INDEX idx_orders_metadata_priority 
ON orders ((metadata->>'priority')) 
WHERE metadata->>'priority' IS NOT NULL;

-- Index on calculated field
CREATE INDEX idx_sales_profit_margin 
ON sales ((revenue - cost) / revenue) 
WHERE revenue > 0;

-- Use expression indexes
SELECT * FROM users WHERE lower(email) = 'john@example.com';
SELECT * FROM orders WHERE metadata->>'priority' = 'high';
```

### Covering Indexes

```sql
-- Include additional columns in index
CREATE INDEX idx_orders_customer_covering 
ON orders (customer_id, order_date) 
INCLUDE (amount, status, product_id);

-- This query can be satisfied entirely from the index
SELECT customer_id, order_date, amount, status 
FROM orders 
WHERE customer_id = 1001 
AND order_date >= '2023-01-01';
```

### Multi-Column Indexes

```sql
-- Composite index for common query patterns
CREATE INDEX idx_sales_date_customer_product 
ON sales (sale_date, customer_id, product_id);

-- Order matters - most selective column first
CREATE INDEX idx_orders_status_date 
ON orders (status, order_date) 
WHERE status IN ('pending', 'processing');

-- Analyze index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE tablename = 'sales'
ORDER BY idx_scan DESC;
```

## Advanced Query Optimization

### Query Plan Analysis

```sql
-- Detailed execution plan
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT 
    c.customer_name,
    COUNT(*) as order_count,
    SUM(o.amount) as total_amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2023-01-01'
GROUP BY c.customer_id, c.customer_name
HAVING COUNT(*) > 5
ORDER BY total_amount DESC;

-- Plan analysis function
CREATE OR REPLACE FUNCTION analyze_query_plan(
    query_text TEXT
)
RETURNS TABLE(
    node_type TEXT,
    total_cost NUMERIC,
    rows_estimate BIGINT,
    width_estimate INTEGER,
    actual_time NUMERIC,
    actual_rows BIGINT,
    buffers_hit BIGINT,
    buffers_read BIGINT
)
AS $$
DECLARE
    plan_json JSONB;
    explain_query TEXT;
BEGIN
    explain_query := 'EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ' || query_text;
    EXECUTE explain_query INTO plan_json;
    
    RETURN QUERY
    SELECT 
        (node->>'Node Type')::TEXT,
        (node->>'Total Cost')::NUMERIC,
        (node->>'Plan Rows')::BIGINT,
        (node->>'Plan Width')::INTEGER,
        (node->'Actual Total Time')::NUMERIC,
        (node->'Actual Rows')::BIGINT,
        (node->'Buffers'->'Shared Hit Blocks')::BIGINT,
        (node->'Buffers'->'Shared Read Blocks')::BIGINT
    FROM jsonb_array_elements(plan_json->0->'Plan') AS node;
END;
$$ LANGUAGE plpgsql;
```

### Statistics and Histograms

```sql
-- View table statistics
SELECT 
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;

-- Adjust statistics target
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
ALTER TABLE orders ALTER COLUMN order_date SET STATISTICS 500;
ANALYZE orders;

-- Extended statistics for correlated columns
CREATE STATISTICS orders_customer_date_stats (dependencies)
ON customer_id, order_date
FROM orders;

CREATE STATISTICS orders_region_product_stats (dependencies, ndistinct)
ON region, product_id, category
FROM orders;

ANALYZE orders;

-- View extended statistics
SELECT 
    stxname,
    stxkeys,
    stxkind,
    stxndistinct,
    stxdependencies
FROM pg_statistic_ext
WHERE stxrelid = 'orders'::regclass;
```

## Connection Pooling and Load Balancing

### PgBouncer Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

# Pool settings
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
max_db_connections = 100
max_user_connections = 100

# Timeouts
server_reset_query = DISCARD ALL
server_check_delay = 30
server_check_query = SELECT 1
server_lifetime = 3600
server_idle_timeout = 600

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
min_pool_size = 5
reserve_pool_size = 5
reserve_pool_timeout = 5
```

### Connection Pool Monitoring

```sql
-- PgBouncer statistics (connect to pgbouncer admin)
-- psql -h localhost -p 6432 -U pgbouncer pgbouncer

-- Show pool status
SHOW POOLS;

-- Show client connections
SHOW CLIENTS;

-- Show server connections
SHOW SERVERS;

-- Show statistics
SHOW STATS;

-- Show configuration
SHOW CONFIG;

-- Reload configuration
RELOAD;

-- Pause/resume pools
PAUSE mydb;
RESUME mydb;
```

## Advanced Security Features

### Row Level Security (RLS)

```sql
-- Enable RLS on table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policies
CREATE POLICY orders_user_policy ON orders
    FOR ALL
    TO application_users
    USING (customer_id = current_setting('app.current_user_id')::INTEGER);

CREATE POLICY orders_manager_policy ON orders
    FOR ALL
    TO managers
    USING (region = current_setting('app.user_region'));

CREATE POLICY orders_admin_policy ON orders
    FOR ALL
    TO administrators
    USING (true);  -- Admins see everything

-- Set user context
SET app.current_user_id = '1001';
SET app.user_region = 'North';

-- Test RLS
SELECT * FROM orders;  -- Only sees allowed rows

-- Bypass RLS (for superusers)
SET row_security = off;
SELECT * FROM orders;  -- Sees all rows
SET row_security = on;
```

### Column-Level Security

```sql
-- Create security labels
SECURITY LABEL FOR dummy ON COLUMN customers.ssn IS 'sensitive';
SECURITY LABEL FOR dummy ON COLUMN customers.credit_card IS 'pci';

-- Create view with column masking
CREATE VIEW customers_masked AS
SELECT 
    customer_id,
    customer_name,
    email,
    CASE 
        WHEN current_user IN ('admin', 'compliance_officer') THEN ssn
        ELSE 'XXX-XX-' || RIGHT(ssn, 4)
    END AS ssn,
    CASE 
        WHEN current_user = 'admin' THEN credit_card
        ELSE 'XXXX-XXXX-XXXX-' || RIGHT(credit_card, 4)
    END AS credit_card
FROM customers;

-- Grant access to masked view
GRANT SELECT ON customers_masked TO application_users;
REVOKE ALL ON customers FROM application_users;
```

### Audit Logging

```sql
-- Create audit table
CREATE TABLE audit_log (
    audit_id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    operation VARCHAR(10),
    user_name VARCHAR(100),
    timestamp TIMESTAMP DEFAULT NOW(),
    old_values JSONB,
    new_values JSONB,
    changed_fields TEXT[]
);

-- Create audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER
AS $$
DECLARE
    old_row JSONB;
    new_row JSONB;
    changed_fields TEXT[] := ARRAY[]::TEXT[];
    field_name TEXT;
BEGIN
    -- Convert rows to JSON
    IF TG_OP = 'DELETE' THEN
        old_row := to_jsonb(OLD);
        new_row := NULL;
    ELSIF TG_OP = 'INSERT' THEN
        old_row := NULL;
        new_row := to_jsonb(NEW);
    ELSE  -- UPDATE
        old_row := to_jsonb(OLD);
        new_row := to_jsonb(NEW);
        
        -- Find changed fields
        FOR field_name IN SELECT jsonb_object_keys(new_row) LOOP
            IF old_row->field_name IS DISTINCT FROM new_row->field_name THEN
                changed_fields := array_append(changed_fields, field_name);
            END IF;
        END LOOP;
    END IF;
    
    -- Insert audit record
    INSERT INTO audit_log (
        table_name,
        operation,
        user_name,
        old_values,
        new_values,
        changed_fields
    ) VALUES (
        TG_TABLE_NAME,
        TG_OP,
        current_user,
        old_row,
        new_row,
        changed_fields
    );
    
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Create audit triggers
CREATE TRIGGER customers_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON customers
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER orders_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

-- Query audit log
SELECT 
    audit_id,
    table_name,
    operation,
    user_name,
    timestamp,
    changed_fields
FROM audit_log
WHERE table_name = 'customers'
AND timestamp >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY timestamp DESC;
```

## Advanced Backup Strategies

### Incremental Backups

```bash
#!/bin/bash
# incremental_backup.sh

BACKUP_DIR="/backup/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
FULL_BACKUP_DIR="$BACKUP_DIR/full"
INCR_BACKUP_DIR="$BACKUP_DIR/incremental"
LOG_FILE="$BACKUP_DIR/backup.log"

# Create directories
mkdir -p "$FULL_BACKUP_DIR" "$INCR_BACKUP_DIR"

echo "Starting incremental backup at $(date)" >> "$LOG_FILE"

# Check if full backup exists and is recent
LATEST_FULL=$(find "$FULL_BACKUP_DIR" -name "*.tar.gz" -mtime -7 | head -1)

if [ -z "$LATEST_FULL" ]; then
    echo "No recent full backup found, creating full backup" >> "$LOG_FILE"
    
    # Create full backup
    pg_basebackup -D "$FULL_BACKUP_DIR/full_$DATE" \
                  -Ft -z -P -v \
                  -h localhost -U postgres
    
    if [ $? -eq 0 ]; then
        echo "Full backup completed successfully" >> "$LOG_FILE"
    else
        echo "Full backup failed" >> "$LOG_FILE"
        exit 1
    fi
else
    echo "Recent full backup exists, creating incremental backup" >> "$LOG_FILE"
    
    # Create incremental backup using WAL archiving
    LAST_WAL=$(psql -h localhost -U postgres -t -c "SELECT pg_current_wal_lsn();")
    
    # Archive WAL files
    rsync -av /var/lib/postgresql/15/main/pg_wal/ "$INCR_BACKUP_DIR/wal_$DATE/"
    
    echo "Incremental backup completed. Last WAL: $LAST_WAL" >> "$LOG_FILE"
fi

# Cleanup old backups (keep 30 days)
find "$BACKUP_DIR" -type f -mtime +30 -delete

echo "Backup process completed at $(date)" >> "$LOG_FILE"
```

### Point-in-Time Recovery Setup

```bash
# Configure WAL archiving in postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/postgresql/wal_archive/%f'
max_wal_senders = 3
wal_keep_size = 1GB

# Create WAL archive directory
mkdir -p /backup/postgresql/wal_archive
chown postgres:postgres /backup/postgresql/wal_archive
```

```sql
-- Create restore point
SELECT pg_create_restore_point('before_major_update');

-- Get current WAL position
SELECT pg_current_wal_lsn();

-- Switch WAL file
SELECT pg_switch_wal();
```

```bash
#!/bin/bash
# point_in_time_recovery.sh

BACKUP_DIR="/backup/postgresql"
RECOVERY_DIR="/var/lib/postgresql/15/recovery"
TARGET_TIME="2023-12-01 14:30:00"

# Stop PostgreSQL
sudo systemctl stop postgresql

# Remove current data directory
sudo rm -rf /var/lib/postgresql/15/main/*

# Restore base backup
sudo -u postgres tar -xzf "$BACKUP_DIR/full/full_20231201.tar.gz" \
                    -C /var/lib/postgresql/15/main/

# Create recovery configuration
sudo -u postgres cat > /var/lib/postgresql/15/main/postgresql.auto.conf << EOF
restore_command = 'cp /backup/postgresql/wal_archive/%f %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_action = 'promote'
EOF

# Create recovery signal file
sudo -u postgres touch /var/lib/postgresql/15/main/recovery.signal

# Start PostgreSQL
sudo systemctl start postgresql

echo "Point-in-time recovery initiated to $TARGET_TIME"
echo "Monitor logs for recovery progress"
```

## Performance Monitoring and Alerting

### Custom Monitoring Functions

```sql
-- Create monitoring schema
CREATE SCHEMA monitoring;

-- Performance metrics collection
CREATE TABLE monitoring.performance_metrics (
    metric_id BIGSERIAL PRIMARY KEY,
    metric_name VARCHAR(100),
    metric_value NUMERIC,
    metric_unit VARCHAR(20),
    collected_at TIMESTAMP DEFAULT NOW()
);

-- Collect performance metrics
CREATE OR REPLACE FUNCTION monitoring.collect_metrics()
RETURNS VOID
AS $$
DECLARE
    db_size BIGINT;
    connection_count INTEGER;
    cache_hit_ratio NUMERIC;
    slow_query_count INTEGER;
    lock_wait_count INTEGER;
BEGIN
    -- Database size
    SELECT pg_database_size(current_database()) INTO db_size;
    INSERT INTO monitoring.performance_metrics (metric_name, metric_value, metric_unit)
    VALUES ('database_size', db_size, 'bytes');
    
    -- Active connections
    SELECT COUNT(*) INTO connection_count
    FROM pg_stat_activity WHERE state = 'active';
    INSERT INTO monitoring.performance_metrics (metric_name, metric_value, metric_unit)
    VALUES ('active_connections', connection_count, 'count');
    
    -- Cache hit ratio
    SELECT ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2)
    INTO cache_hit_ratio
    FROM pg_stat_database WHERE datname = current_database();
    INSERT INTO monitoring.performance_metrics (metric_name, metric_value, metric_unit)
    VALUES ('cache_hit_ratio', cache_hit_ratio, 'percent');
    
    -- Slow queries
    SELECT COUNT(*) INTO slow_query_count
    FROM pg_stat_activity
    WHERE state = 'active'
    AND NOW() - query_start > INTERVAL '5 minutes';
    INSERT INTO monitoring.performance_metrics (metric_name, metric_value, metric_unit)
    VALUES ('slow_queries', slow_query_count, 'count');
    
    -- Lock waits
    SELECT COUNT(*) INTO lock_wait_count
    FROM pg_locks WHERE granted = false;
    INSERT INTO monitoring.performance_metrics (metric_name, metric_value, metric_unit)
    VALUES ('lock_waits', lock_wait_count, 'count');
END;
$$ LANGUAGE plpgsql;

-- Schedule metric collection (using pg_cron)
SELECT cron.schedule('collect-metrics', '*/5 * * * *', 'SELECT monitoring.collect_metrics();');

-- Performance trends query
SELECT 
    metric_name,
    DATE_TRUNC('hour', collected_at) AS hour,
    AVG(metric_value) AS avg_value,
    MIN(metric_value) AS min_value,
    MAX(metric_value) AS max_value
FROM monitoring.performance_metrics
WHERE collected_at >= NOW() - INTERVAL '24 hours'
GROUP BY metric_name, DATE_TRUNC('hour', collected_at)
ORDER BY metric_name, hour;
```

## Best Practices

### Configuration Management

1. **Version Control Configuration**
   - Store configuration files in version control
   - Use configuration management tools
   - Document all changes

2. **Environment Consistency**
   - Maintain consistent settings across environments
   - Use automated deployment
   - Test configuration changes

3. **Performance Tuning**
   - Monitor before and after changes
   - Make incremental adjustments
   - Document performance impacts

### Security Best Practices

1. **Access Control**
   - Implement least privilege principle
   - Use role-based access control
   - Regular access reviews

2. **Data Protection**
   - Encrypt sensitive data
   - Implement audit logging
   - Use row-level security

3. **Network Security**
   - Use SSL/TLS connections
   - Implement firewall rules
   - Monitor connection attempts

### Maintenance Best Practices

1. **Automated Maintenance**
   - Schedule regular maintenance tasks
   - Monitor maintenance job results
   - Implement alerting for failures

2. **Capacity Planning**
   - Monitor growth trends
   - Plan for future capacity
   - Implement archiving strategies

3. **Disaster Recovery**
   - Test backup and recovery procedures
   - Document recovery processes
   - Maintain off-site backups

## Troubleshooting Advanced Issues

### Performance Debugging

```sql
-- Identify resource bottlenecks
WITH resource_usage AS (
    SELECT 
        'CPU' as resource_type,
        SUM(total_exec_time) as total_usage
    FROM pg_stat_statements
    UNION ALL
    SELECT 
        'I/O' as resource_type,
        SUM(shared_blks_read + shared_blks_written) as total_usage
    FROM pg_stat_statements
    UNION ALL
    SELECT 
        'Memory' as resource_type,
        SUM(temp_blks_read + temp_blks_written) as total_usage
    FROM pg_stat_statements
)
SELECT 
    resource_type,
    total_usage,
    ROUND(total_usage * 100.0 / SUM(total_usage) OVER (), 2) as percentage
FROM resource_usage
ORDER BY total_usage DESC;

-- Find queries causing high I/O
SELECT 
    query,
    calls,
    shared_blks_read,
    shared_blks_written,
    shared_blks_dirtied,
    temp_blks_read,
    temp_blks_written
FROM pg_stat_statements
WHERE shared_blks_read + shared_blks_written > 10000
ORDER BY shared_blks_read + shared_blks_written DESC
LIMIT 10;
```

### Connection Issues

```sql
-- Analyze connection patterns
SELECT 
    usename,
    application_name,
    client_addr,
    state,
    COUNT(*) as connection_count,
    AVG(EXTRACT(EPOCH FROM (NOW() - backend_start))) as avg_connection_age
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY usename, application_name, client_addr, state
ORDER BY connection_count DESC;

-- Find connection leaks
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    backend_start,
    NOW() - backend_start as connection_age,
    state,
    query
FROM pg_stat_activity
WHERE NOW() - backend_start > INTERVAL '1 hour'
AND state = 'idle'
ORDER BY backend_start;
```

## Next Steps

After mastering advanced administration:
1. Performance Optimization (14-performance-optimization.md)
2. High Availability and Clustering (15-high-availability.md)
3. PostgreSQL in Production (16-production-deployment.md)

---
*This is part 13 of the PostgreSQL learning series*