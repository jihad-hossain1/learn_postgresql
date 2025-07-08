# Monitoring and Maintenance in PostgreSQL

## Introduction

Proper monitoring and maintenance are essential for keeping PostgreSQL databases running efficiently, preventing issues, and ensuring optimal performance. This guide covers monitoring tools, maintenance tasks, and best practices.

## Database Monitoring Fundamentals

### Key Metrics to Monitor

1. **Performance Metrics**
   - Query execution time
   - Throughput (transactions per second)
   - Connection count
   - Cache hit ratios

2. **Resource Utilization**
   - CPU usage
   - Memory usage
   - Disk I/O
   - Network I/O

3. **Database Health**
   - Lock contention
   - Deadlocks
   - Replication lag
   - Vacuum and analyze status

## Built-in Monitoring Views

### pg_stat_activity

```sql
-- Current database activity
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    NOW() - query_start AS query_duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- Long-running queries
SELECT 
    pid,
    usename,
    application_name,
    NOW() - query_start AS duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
AND NOW() - query_start > INTERVAL '5 minutes'
ORDER BY duration DESC;

-- Idle in transaction connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    NOW() - state_change AS idle_duration,
    query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;

-- Connection count by state
SELECT 
    state,
    COUNT(*) as connection_count
FROM pg_stat_activity
GROUP BY state
ORDER BY connection_count DESC;
```

### pg_stat_database

```sql
-- Database-level statistics
SELECT 
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2) AS cache_hit_ratio,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted,
    conflicts,
    temp_files,
    temp_bytes,
    deadlocks,
    stats_reset
FROM pg_stat_database
WHERE datname = current_database();

-- Database size and growth
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) AS database_size,
    pg_database_size(datname) AS size_bytes
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;
```

### pg_stat_user_tables

```sql
-- Table-level statistics
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_tup_hot_upd,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze,
    vacuum_count,
    autovacuum_count,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY seq_scan + idx_scan DESC;

-- Tables with high sequential scan ratio
SELECT 
    schemaname,
    tablename,
    seq_scan,
    idx_scan,
    CASE 
        WHEN seq_scan + idx_scan = 0 THEN 0
        ELSE ROUND((seq_scan::FLOAT / (seq_scan + idx_scan)) * 100, 2)
    END AS seq_scan_ratio,
    n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 100
ORDER BY seq_scan_ratio DESC;

-- Tables needing vacuum/analyze
SELECT 
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    ROUND((n_dead_tup::FLOAT / GREATEST(n_live_tup, 1)) * 100, 2) AS dead_tuple_ratio,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY dead_tuple_ratio DESC;
```

### pg_stat_user_indexes

```sql
-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND schemaname NOT IN ('information_schema', 'pg_catalog')
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index efficiency
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    CASE 
        WHEN idx_scan = 0 THEN 0
        ELSE ROUND(idx_tup_read::FLOAT / idx_scan, 2)
    END AS avg_tuples_per_scan
FROM pg_stat_user_indexes
WHERE idx_scan > 0
ORDER BY avg_tuples_per_scan DESC;
```

## Performance Monitoring

### Query Performance

```sql
-- Enable pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top queries by total time
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

-- Slowest queries by average time
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Most frequently called queries
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Queries with low cache hit ratio
SELECT 
    query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
WHERE shared_blks_hit + shared_blks_read > 1000
ORDER BY hit_percent ASC
LIMIT 10;
```

### Lock Monitoring

```sql
-- Current locks
SELECT 
    l.locktype,
    l.database,
    l.relation::regclass,
    l.page,
    l.tuple,
    l.virtualxid,
    l.transactionid,
    l.mode,
    l.granted,
    a.query,
    a.query_start,
    a.state,
    a.usename,
    a.application_name
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.granted = false
ORDER BY l.relation, l.mode;

-- Lock waits and blocking queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement,
    blocked_activity.application_name AS blocked_application,
    blocking_activity.application_name AS blocking_application
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- Deadlock information
SELECT 
    datname,
    deadlocks
FROM pg_stat_database
WHERE deadlocks > 0
ORDER BY deadlocks DESC;
```

### I/O and Buffer Monitoring

```sql
-- Buffer cache statistics
SELECT 
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name IN ('shared_buffers', 'effective_cache_size', 'work_mem', 'maintenance_work_mem');

-- Buffer usage by relation
SELECT 
    c.relname,
    pg_size_pretty(pg_relation_size(c.oid)) AS size,
    CASE 
        WHEN pg_relation_size(c.oid) = 0 THEN 0
        ELSE (pg_relation_size(c.oid) / 8192)
    END AS pages,
    COUNT(*) AS buffers_used
FROM pg_class c
JOIN pg_buffercache b ON c.relfilenode = b.relfilenode
WHERE c.relkind IN ('r', 'i')
GROUP BY c.relname, c.oid
ORDER BY buffers_used DESC
LIMIT 20;

-- I/O statistics
SELECT 
    schemaname,
    tablename,
    heap_blks_read,
    heap_blks_hit,
    CASE 
        WHEN heap_blks_read + heap_blks_hit = 0 THEN 0
        ELSE ROUND((heap_blks_hit::FLOAT / (heap_blks_read + heap_blks_hit)) * 100, 2)
    END AS heap_hit_ratio,
    idx_blks_read,
    idx_blks_hit,
    CASE 
        WHEN idx_blks_read + idx_blks_hit = 0 THEN 0
        ELSE ROUND((idx_blks_hit::FLOAT / (idx_blks_read + idx_blks_hit)) * 100, 2)
    END AS idx_hit_ratio
FROM pg_statio_user_tables
WHERE heap_blks_read + heap_blks_hit > 0
ORDER BY heap_blks_read + heap_blks_hit DESC;
```

## System Resource Monitoring

### Memory Usage

```sql
-- Memory-related settings
SELECT 
    name,
    setting,
    unit,
    context,
    short_desc
FROM pg_settings
WHERE name IN (
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'effective_cache_size',
    'max_connections',
    'temp_buffers'
)
ORDER BY name;

-- Current memory usage (requires pg_stat_statements)
SELECT 
    query,
    calls,
    total_exec_time,
    shared_blks_hit + shared_blks_read AS total_buffers,
    shared_blks_written,
    shared_blks_dirtied,
    temp_blks_read,
    temp_blks_written
FROM pg_stat_statements
WHERE temp_blks_read + temp_blks_written > 0
ORDER BY temp_blks_read + temp_blks_written DESC
LIMIT 10;
```

### Disk Usage

```sql
-- Database sizes
SELECT 
    datname AS database_name,
    pg_size_pretty(pg_database_size(datname)) AS size,
    pg_database_size(datname) AS size_bytes
FROM pg_database
WHERE datistemplate = false
ORDER BY pg_database_size(datname) DESC;

-- Table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size,
    pg_total_relation_size(schemaname||'.'||tablename) AS total_size_bytes
FROM pg_tables
WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Index sizes
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_relation_size(indexrelid) AS size_bytes
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;

-- Tablespace usage
SELECT 
    spcname AS tablespace_name,
    pg_tablespace_location(oid) AS location,
    pg_size_pretty(pg_tablespace_size(spcname)) AS size
FROM pg_tablespace;
```

## Maintenance Tasks

### VACUUM Operations

```sql
-- Manual vacuum operations
VACUUM;  -- Vacuum all tables in current database
VACUUM VERBOSE;  -- Vacuum with detailed output
VACUUM ANALYZE;  -- Vacuum and update statistics
VACUUM FULL;  -- Full vacuum (locks table, reclaims space)

-- Vacuum specific table
VACUUM employees;
VACUUM VERBOSE ANALYZE employees;

-- Check vacuum progress
SELECT 
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count,
    max_dead_tuples,
    num_dead_tuples
FROM pg_stat_progress_vacuum;

-- Autovacuum settings
SELECT 
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name LIKE 'autovacuum%'
ORDER BY name;

-- Table-specific autovacuum settings
SELECT 
    schemaname,
    tablename,
    reloptions
FROM pg_tables
WHERE reloptions IS NOT NULL;

-- Set table-specific autovacuum parameters
ALTER TABLE high_activity_table SET (
    autovacuum_vacuum_threshold = 100,
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_analyze_threshold = 50,
    autovacuum_analyze_scale_factor = 0.05
);
```

### ANALYZE Operations

```sql
-- Update table statistics
ANALYZE;  -- Analyze all tables
ANALYZE employees;  -- Analyze specific table
ANALYZE employees (first_name, last_name);  -- Analyze specific columns

-- Check analyze progress
SELECT 
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    sample_blks_total,
    sample_blks_scanned,
    ext_stats_total,
    ext_stats_computed,
    child_tables_total,
    child_tables_done
FROM pg_stat_progress_analyze;

-- Check statistics freshness
SELECT 
    schemaname,
    tablename,
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds,
    correlation
FROM pg_stats
WHERE tablename = 'employees'
ORDER BY attname;

-- Tables with outdated statistics
SELECT 
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_mod_since_analyze,
    n_live_tup
FROM pg_stat_user_tables
WHERE (
    last_analyze IS NULL 
    OR last_analyze < NOW() - INTERVAL '7 days'
) AND (
    last_autoanalyze IS NULL 
    OR last_autoanalyze < NOW() - INTERVAL '7 days'
)
AND n_live_tup > 1000
ORDER BY n_mod_since_analyze DESC;
```

### REINDEX Operations

```sql
-- Reindex operations
REINDEX INDEX index_name;  -- Reindex specific index
REINDEX TABLE table_name;  -- Reindex all indexes on table
REINDEX SCHEMA schema_name;  -- Reindex all indexes in schema
REINDEX DATABASE database_name;  -- Reindex entire database

-- Concurrent reindex (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY index_name;

-- Check reindex progress
SELECT 
    pid,
    datname,
    relid::regclass AS table_name,
    phase,
    lockers_total,
    lockers_done,
    current_locker_pid,
    blocks_total,
    blocks_done,
    tuples_total,
    tuples_done,
    partitions_total,
    partitions_done
FROM pg_stat_progress_create_index;

-- Find bloated indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE pg_relation_size(indexrelid) > 100 * 1024 * 1024  -- > 100MB
AND idx_scan < 100  -- Low usage
ORDER BY pg_relation_size(indexrelid) DESC;
```

## Automated Monitoring Scripts

### Health Check Script

```sql
-- Create monitoring function
CREATE OR REPLACE FUNCTION database_health_check()
RETURNS TABLE(
    check_name TEXT,
    status TEXT,
    value TEXT,
    threshold TEXT,
    recommendation TEXT
)
AS $$
BEGIN
    -- Check database size
    RETURN QUERY
    SELECT 
        'Database Size'::TEXT,
        CASE 
            WHEN pg_database_size(current_database()) > 50 * 1024^3 THEN 'WARNING'
            ELSE 'OK'
        END,
        pg_size_pretty(pg_database_size(current_database())),
        '50GB'::TEXT,
        'Monitor disk space and consider archiving old data'::TEXT;
    
    -- Check connection count
    RETURN QUERY
    SELECT 
        'Active Connections'::TEXT,
        CASE 
            WHEN COUNT(*) > 80 THEN 'WARNING'
            WHEN COUNT(*) > 50 THEN 'CAUTION'
            ELSE 'OK'
        END,
        COUNT(*)::TEXT,
        '80'::TEXT,
        'Consider connection pooling if connections are high'::TEXT
    FROM pg_stat_activity
    WHERE state = 'active';
    
    -- Check cache hit ratio
    RETURN QUERY
    SELECT 
        'Cache Hit Ratio'::TEXT,
        CASE 
            WHEN ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2) < 95 THEN 'WARNING'
            WHEN ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2) < 98 THEN 'CAUTION'
            ELSE 'OK'
        END,
        ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2)::TEXT || '%',
        '95%'::TEXT,
        'Consider increasing shared_buffers or optimizing queries'::TEXT
    FROM pg_stat_database
    WHERE datname = current_database();
    
    -- Check for long-running queries
    RETURN QUERY
    SELECT 
        'Long Running Queries'::TEXT,
        CASE 
            WHEN COUNT(*) > 5 THEN 'WARNING'
            WHEN COUNT(*) > 0 THEN 'CAUTION'
            ELSE 'OK'
        END,
        COUNT(*)::TEXT,
        '0'::TEXT,
        'Review and optimize long-running queries'::TEXT
    FROM pg_stat_activity
    WHERE state = 'active'
    AND NOW() - query_start > INTERVAL '5 minutes';
    
    -- Check for tables needing vacuum
    RETURN QUERY
    SELECT 
        'Tables Needing Vacuum'::TEXT,
        CASE 
            WHEN COUNT(*) > 10 THEN 'WARNING'
            WHEN COUNT(*) > 0 THEN 'CAUTION'
            ELSE 'OK'
        END,
        COUNT(*)::TEXT,
        '0'::TEXT,
        'Run VACUUM on tables with high dead tuple ratio'::TEXT
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 1000
    AND ROUND((n_dead_tup::FLOAT / GREATEST(n_live_tup, 1)) * 100, 2) > 20;
END;
$$ LANGUAGE plpgsql;

-- Run health check
SELECT * FROM database_health_check();
```

### Performance Monitoring Script

```bash
#!/bin/bash
# performance_monitor.sh

DB_HOST="localhost"
DB_USER="postgres"
DB_NAME="mydb"
LOG_FILE="/var/log/postgresql/performance_monitor.log"
ALERT_EMAIL="admin@company.com"

echo "Performance monitoring started at $(date)" >> "$LOG_FILE"

# Check for slow queries
SLOW_QUERIES=$(psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -t -c "
SELECT COUNT(*) 
FROM pg_stat_activity 
WHERE state = 'active' 
AND NOW() - query_start > INTERVAL '5 minutes';
")

if [ "$SLOW_QUERIES" -gt 0 ]; then
    echo "WARNING: $SLOW_QUERIES slow queries detected" >> "$LOG_FILE"
    
    # Get details of slow queries
    psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -c "
    SELECT 
        pid,
        usename,
        NOW() - query_start AS duration,
        LEFT(query, 100) AS query_snippet
    FROM pg_stat_activity 
    WHERE state = 'active' 
    AND NOW() - query_start > INTERVAL '5 minutes'
    ORDER BY duration DESC;
    " >> "$LOG_FILE"
fi

# Check cache hit ratio
CACHE_HIT_RATIO=$(psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -t -c "
SELECT ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2)
FROM pg_stat_database 
WHERE datname = '$DB_NAME';
")

if (( $(echo "$CACHE_HIT_RATIO < 95" | bc -l) )); then
    echo "WARNING: Cache hit ratio is $CACHE_HIT_RATIO%" >> "$LOG_FILE"
fi

# Check for lock waits
LOCK_WAITS=$(psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -t -c "
SELECT COUNT(*) 
FROM pg_locks 
WHERE granted = false;
")

if [ "$LOCK_WAITS" -gt 0 ]; then
    echo "WARNING: $LOCK_WAITS lock waits detected" >> "$LOG_FILE"
fi

# Check disk usage
DB_SIZE=$(psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -t -c "
SELECT pg_database_size('$DB_NAME');
")

DB_SIZE_GB=$((DB_SIZE / 1024 / 1024 / 1024))

if [ "$DB_SIZE_GB" -gt 50 ]; then
    echo "WARNING: Database size is ${DB_SIZE_GB}GB" >> "$LOG_FILE"
fi

echo "Performance monitoring completed at $(date)" >> "$LOG_FILE"

# Send alert if there are warnings
if grep -q "WARNING" "$LOG_FILE"; then
    tail -20 "$LOG_FILE" | mail -s "PostgreSQL Performance Alert" "$ALERT_EMAIL"
fi
```

### Maintenance Automation

```bash
#!/bin/bash
# automated_maintenance.sh

DB_HOST="localhost"
DB_USER="postgres"
DB_NAME="mydb"
LOG_FILE="/var/log/postgresql/maintenance.log"

echo "Automated maintenance started at $(date)" >> "$LOG_FILE"

# Update statistics for tables that need it
psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -c "
DO \$\$
DECLARE
    table_record RECORD;
BEGIN
    FOR table_record IN 
        SELECT schemaname, tablename
        FROM pg_stat_user_tables
        WHERE (
            last_analyze IS NULL 
            OR last_analyze < NOW() - INTERVAL '7 days'
        ) AND (
            last_autoanalyze IS NULL 
            OR last_autoanalyze < NOW() - INTERVAL '7 days'
        )
        AND n_live_tup > 1000
    LOOP
        RAISE NOTICE 'Analyzing table %.%', table_record.schemaname, table_record.tablename;
        EXECUTE format('ANALYZE %I.%I', table_record.schemaname, table_record.tablename);
    END LOOP;
END \$\$;
" >> "$LOG_FILE" 2>&1

# Vacuum tables with high dead tuple ratio
psql -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -c "
DO \$\$
DECLARE
    table_record RECORD;
BEGIN
    FOR table_record IN 
        SELECT schemaname, tablename
        FROM pg_stat_user_tables
        WHERE n_dead_tup > 1000
        AND ROUND((n_dead_tup::FLOAT / GREATEST(n_live_tup, 1)) * 100, 2) > 20
    LOOP
        RAISE NOTICE 'Vacuuming table %.%', table_record.schemaname, table_record.tablename;
        EXECUTE format('VACUUM ANALYZE %I.%I', table_record.schemaname, table_record.tablename);
    END LOOP;
END \$\$;
" >> "$LOG_FILE" 2>&1

# Clean up old log files
find /var/log/postgresql -name "*.log" -mtime +30 -delete

echo "Automated maintenance completed at $(date)" >> "$LOG_FILE"
```

## Log Analysis

### PostgreSQL Log Configuration

```bash
# Configure logging in postgresql.conf
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 100MB
log_min_messages = warning
log_min_error_statement = error
log_min_duration_statement = 1000  # Log queries > 1 second
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
log_error_verbosity = default
log_hostname = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'none'  # or 'ddl', 'mod', 'all'
```

### Log Analysis Queries

```sql
-- Create log analysis table (if using CSV logging)
CREATE TABLE postgres_log (
    log_time TIMESTAMP,
    user_name TEXT,
    database_name TEXT,
    process_id INTEGER,
    connection_from TEXT,
    session_id TEXT,
    session_line_num BIGINT,
    command_tag TEXT,
    session_start_time TIMESTAMP,
    virtual_transaction_id TEXT,
    transaction_id BIGINT,
    error_severity TEXT,
    sql_state_code TEXT,
    message TEXT,
    detail TEXT,
    hint TEXT,
    internal_query TEXT,
    internal_query_pos INTEGER,
    context TEXT,
    query TEXT,
    query_pos INTEGER,
    location TEXT,
    application_name TEXT
);

-- Load log data (example)
-- COPY postgres_log FROM '/var/log/postgresql/postgresql.csv' WITH CSV;

-- Analyze error patterns
SELECT 
    error_severity,
    sql_state_code,
    COUNT(*) as occurrence_count,
    LEFT(message, 100) as sample_message
FROM postgres_log
WHERE log_time >= CURRENT_DATE - INTERVAL '7 days'
AND error_severity IN ('ERROR', 'FATAL', 'PANIC')
GROUP BY error_severity, sql_state_code, LEFT(message, 100)
ORDER BY occurrence_count DESC;

-- Connection patterns
SELECT 
    user_name,
    database_name,
    application_name,
    connection_from,
    COUNT(*) as connection_count
FROM postgres_log
WHERE message LIKE '%connection authorized%'
AND log_time >= CURRENT_DATE - INTERVAL '1 day'
GROUP BY user_name, database_name, application_name, connection_from
ORDER BY connection_count DESC;
```

### Log Analysis Scripts

```bash
#!/bin/bash
# analyze_logs.sh

LOG_DIR="/var/log/postgresql"
REPORT_FILE="/tmp/postgresql_log_analysis.txt"

echo "PostgreSQL Log Analysis Report - $(date)" > "$REPORT_FILE"
echo "=========================================" >> "$REPORT_FILE"

# Count error types
echo "\nError Summary (last 24 hours):" >> "$REPORT_FILE"
grep "$(date -d '1 day ago' '+%Y-%m-%d')\|$(date '+%Y-%m-%d')" "$LOG_DIR"/postgresql-*.log | \
grep -E "ERROR|FATAL|PANIC" | \
cut -d':' -f4- | \
sort | uniq -c | sort -nr >> "$REPORT_FILE"

# Find slow queries
echo "\nSlow Queries (> 5 seconds):" >> "$REPORT_FILE"
grep "duration:" "$LOG_DIR"/postgresql-*.log | \
awk '$NF > 5000 {print}' | \
tail -10 >> "$REPORT_FILE"

# Connection statistics
echo "\nConnection Statistics:" >> "$REPORT_FILE"
grep "connection authorized" "$LOG_DIR"/postgresql-*.log | \
awk '{print $8}' | sort | uniq -c | sort -nr >> "$REPORT_FILE"

# Checkpoint statistics
echo "\nCheckpoint Information:" >> "$REPORT_FILE"
grep "checkpoint" "$LOG_DIR"/postgresql-*.log | tail -5 >> "$REPORT_FILE"

echo "\nReport generated at $(date)" >> "$REPORT_FILE"

# Email report
mail -s "PostgreSQL Log Analysis Report" admin@company.com < "$REPORT_FILE"
```

## Alerting and Notifications

### Alert Configuration

```sql
-- Create alerts table
CREATE TABLE monitoring_alerts (
    alert_id SERIAL PRIMARY KEY,
    alert_type VARCHAR(50),
    severity VARCHAR(20),
    message TEXT,
    threshold_value NUMERIC,
    current_value NUMERIC,
    alert_time TIMESTAMP DEFAULT NOW(),
    resolved BOOLEAN DEFAULT FALSE,
    resolved_time TIMESTAMP
);

-- Alert checking function
CREATE OR REPLACE FUNCTION check_alerts()
RETURNS VOID
AS $$
DECLARE
    connection_count INTEGER;
    cache_hit_ratio NUMERIC;
    slow_query_count INTEGER;
    lock_wait_count INTEGER;
BEGIN
    -- Check connection count
    SELECT COUNT(*) INTO connection_count
    FROM pg_stat_activity
    WHERE state = 'active';
    
    IF connection_count > 80 THEN
        INSERT INTO monitoring_alerts (alert_type, severity, message, threshold_value, current_value)
        VALUES ('HIGH_CONNECTIONS', 'WARNING', 'High number of active connections', 80, connection_count);
    END IF;
    
    -- Check cache hit ratio
    SELECT ROUND((blks_hit::FLOAT / (blks_hit + blks_read)) * 100, 2)
    INTO cache_hit_ratio
    FROM pg_stat_database
    WHERE datname = current_database();
    
    IF cache_hit_ratio < 95 THEN
        INSERT INTO monitoring_alerts (alert_type, severity, message, threshold_value, current_value)
        VALUES ('LOW_CACHE_HIT_RATIO', 'WARNING', 'Cache hit ratio below threshold', 95, cache_hit_ratio);
    END IF;
    
    -- Check for slow queries
    SELECT COUNT(*) INTO slow_query_count
    FROM pg_stat_activity
    WHERE state = 'active'
    AND NOW() - query_start > INTERVAL '5 minutes';
    
    IF slow_query_count > 0 THEN
        INSERT INTO monitoring_alerts (alert_type, severity, message, threshold_value, current_value)
        VALUES ('SLOW_QUERIES', 'CRITICAL', 'Long-running queries detected', 0, slow_query_count);
    END IF;
    
    -- Check for lock waits
    SELECT COUNT(*) INTO lock_wait_count
    FROM pg_locks
    WHERE granted = false;
    
    IF lock_wait_count > 0 THEN
        INSERT INTO monitoring_alerts (alert_type, severity, message, threshold_value, current_value)
        VALUES ('LOCK_WAITS', 'WARNING', 'Lock waits detected', 0, lock_wait_count);
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Schedule alert checking (use cron or pg_cron extension)
-- SELECT cron.schedule('check-alerts', '*/5 * * * *', 'SELECT check_alerts();');
```

### Email Notification Function

```sql
-- Email notification function (requires plpython3u)
CREATE OR REPLACE FUNCTION send_alert_email(
    alert_type TEXT,
    severity TEXT,
    message TEXT,
    current_value NUMERIC
)
RETURNS VOID
AS $$
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# Email configuration
smtp_server = "smtp.company.com"
smtp_port = 587
from_email = "postgres-alerts@company.com"
to_email = "admin@company.com"
password = "email_password"

# Create message
msg = MIMEMultipart()
msg['From'] = from_email
msg['To'] = to_email
msg['Subject'] = f"PostgreSQL Alert: {alert_type} - {severity}"

body = f"""
PostgreSQL Alert Notification

Alert Type: {alert_type}
Severity: {severity}
Message: {message}
Current Value: {current_value}
Time: {plpy.execute("SELECT NOW()")[0]['now']}

Please investigate this issue promptly.
"""

msg.attach(MIMEText(body, 'plain'))

# Send email
try:
    server = smtplib.SMTP(smtp_server, smtp_port)
    server.starttls()
    server.login(from_email, password)
    text = msg.as_string()
    server.sendmail(from_email, to_email, text)
    server.quit()
    plpy.notice(f"Alert email sent for {alert_type}")
except Exception as e:
    plpy.error(f"Failed to send alert email: {str(e)}")
$$ LANGUAGE plpython3u;
```

## Best Practices

### Monitoring Best Practices

1. **Establish Baselines**
   - Monitor normal performance patterns
   - Set realistic thresholds
   - Adjust alerts based on historical data

2. **Monitor Key Metrics**
   - Response time
   - Throughput
   - Resource utilization
   - Error rates

3. **Automate Monitoring**
   - Use scheduled jobs for regular checks
   - Implement automated alerting
   - Create self-healing scripts where possible

4. **Regular Maintenance**
   - Schedule vacuum and analyze operations
   - Monitor and maintain indexes
   - Clean up old log files

### Maintenance Best Practices

1. **Preventive Maintenance**
   - Regular vacuum and analyze
   - Index maintenance
   - Statistics updates
   - Log rotation

2. **Performance Optimization**
   - Monitor slow queries
   - Optimize frequently used queries
   - Review and update indexes
   - Tune configuration parameters

3. **Capacity Planning**
   - Monitor growth trends
   - Plan for future capacity needs
   - Implement archiving strategies
   - Monitor disk space usage

## Troubleshooting Common Issues

### Performance Issues

```sql
-- Identify performance bottlenecks

-- 1. Check for blocking queries
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- 2. Check for high CPU queries
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time
FROM pg_stat_statements
WHERE calls > 100
ORDER BY total_exec_time DESC
LIMIT 10;

-- 3. Check for I/O intensive queries
SELECT 
    query,
    calls,
    shared_blks_read,
    shared_blks_hit,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
WHERE shared_blks_read > 1000
ORDER BY shared_blks_read DESC
LIMIT 10;
```

### Connection Issues

```sql
-- Check connection limits
SELECT 
    setting AS max_connections,
    COUNT(*) AS current_connections,
    COUNT(*) * 100.0 / setting::INTEGER AS connection_usage_percent
FROM pg_settings, pg_stat_activity
WHERE name = 'max_connections'
GROUP BY setting;

-- Check connection states
SELECT 
    state,
    COUNT(*) as count
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;

-- Find idle in transaction connections
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    NOW() - state_change AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;
```

## Next Steps

After mastering monitoring and maintenance:
1. Advanced Administration (13-advanced-administration.md)
2. Performance Optimization (14-performance-optimization.md)
3. High Availability and Clustering (15-high-availability.md)

---
*This is part 12 of the PostgreSQL learning series*