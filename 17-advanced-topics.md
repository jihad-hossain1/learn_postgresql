# PostgreSQL Advanced Topics and Extensions

*Part 17 of the PostgreSQL Learning Series*

## Table of Contents
1. [Advanced Data Types](#advanced-data-types)
2. [PostgreSQL Extensions](#postgresql-extensions)
3. [Custom Data Types](#custom-data-types)
4. [Advanced Indexing](#advanced-indexing)
5. [Parallel Processing](#parallel-processing)
6. [Foreign Data Wrappers](#foreign-data-wrappers)
7. [Event Triggers](#event-triggers)
8. [Advanced PL/pgSQL](#advanced-plpgsql)
9. [PostgreSQL Internals](#postgresql-internals)
10. [Performance Tuning Deep Dive](#performance-tuning-deep-dive)

## Advanced Data Types

### Range Types

```sql
-- Create a table with range types
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_number INTEGER,
    reservation_period TSRANGE,
    price_range NUMRANGE
);

-- Insert data with ranges
INSERT INTO reservations (room_number, reservation_period, price_range) VALUES
(101, '[2024-01-15 14:00, 2024-01-17 12:00)', '[100.00, 150.00]'),
(102, '[2024-01-16 15:00, 2024-01-18 11:00)', '[120.00, 180.00]'),
(103, '[2024-01-17 16:00, 2024-01-19 10:00)', '[90.00, 130.00]');

-- Query overlapping reservations
SELECT r1.room_number, r2.room_number, 
       r1.reservation_period, r2.reservation_period
FROM reservations r1, reservations r2
WHERE r1.id < r2.id 
  AND r1.reservation_period && r2.reservation_period;

-- Find available time slots
SELECT room_number,
       reservation_period,
       lag(upper(reservation_period), 1, '-infinity'::timestamp) 
           OVER (PARTITION BY room_number ORDER BY reservation_period) AS prev_end,
       lower(reservation_period) AS current_start
FROM reservations
ORDER BY room_number, reservation_period;

-- Range operations
SELECT 
    '[1,10]'::int4range + '[5,15]'::int4range AS union_range,
    '[1,10]'::int4range * '[5,15]'::int4range AS intersection,
    '[1,10]'::int4range - '[5,15]'::int4range AS difference;

-- Custom range type
CREATE TYPE float8_range AS RANGE (
    subtype = float8,
    subtype_diff = float8mi
);

-- Use custom range
SELECT '[1.5, 3.7)'::float8_range @> 2.5;
```

### Array Operations

```sql
-- Advanced array operations
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    tags TEXT[],
    ratings INTEGER[],
    features JSONB
);

INSERT INTO products (name, tags, ratings, features) VALUES
('Laptop', ARRAY['electronics', 'computer', 'portable'], ARRAY[5,4,5,3,4], 
 '{"screen_size": 15.6, "weight": 2.1, "battery_life": 8}'),
('Phone', ARRAY['electronics', 'mobile', 'communication'], ARRAY[4,5,4,4,3],
 '{"screen_size": 6.1, "weight": 0.18, "battery_life": 24}'),
('Tablet', ARRAY['electronics', 'portable', 'touch'], ARRAY[4,4,5,4,4],
 '{"screen_size": 10.9, "weight": 0.47, "battery_life": 12}');

-- Array aggregation and analysis
SELECT 
    name,
    array_length(tags, 1) as tag_count,
    array_length(ratings, 1) as rating_count,
    ROUND(AVG(rating), 2) as avg_rating,
    array_agg(rating ORDER BY rating DESC) as sorted_ratings
FROM products, unnest(ratings) as rating
GROUP BY id, name, tags, ratings;

-- Array search and filtering
SELECT name, tags
FROM products
WHERE tags @> ARRAY['electronics']  -- Contains
  AND tags && ARRAY['portable', 'mobile'];  -- Overlaps

-- Array manipulation functions
SELECT 
    name,
    tags,
    array_prepend('new', tags) as with_new_tag,
    array_append(tags, 'featured') as with_featured,
    array_cat(tags, ARRAY['sale', 'discount']) as with_sale_tags,
    array_remove(tags, 'electronics') as without_electronics
FROM products;

-- Multi-dimensional arrays
CREATE TABLE game_board (
    id SERIAL PRIMARY KEY,
    board INTEGER[][],
    player_moves POINT[]
);

INSERT INTO game_board (board, player_moves) VALUES
(ARRAY[[1,0,1],[0,1,0],[1,0,1]], ARRAY[POINT(1,1), POINT(2,2), POINT(0,0)]);

SELECT 
    board[1:2][1:2] as top_left_quadrant,
    array_dims(board) as dimensions,
    cardinality(player_moves) as move_count
FROM game_board;
```

### JSONB Advanced Operations

```sql
-- Advanced JSONB operations
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    username TEXT,
    profile JSONB,
    settings JSONB,
    activity_log JSONB[]
);

INSERT INTO user_profiles (username, profile, settings, activity_log) VALUES
('alice', 
 '{
   "name": "Alice Johnson",
   "age": 28,
   "location": {"city": "New York", "country": "USA"},
   "interests": ["photography", "travel", "cooking"],
   "social": {"twitter": "@alice_j", "linkedin": "alice-johnson"}
 }',
 '{
   "theme": "dark",
   "notifications": {"email": true, "push": false, "sms": true},
   "privacy": {"profile_public": false, "show_activity": true}
 }',
 ARRAY[
   '{"action": "login", "timestamp": "2024-01-15T10:30:00Z", "ip": "192.168.1.100"}',
   '{"action": "profile_update", "timestamp": "2024-01-15T11:15:00Z", "changes": ["location"]}'
 ]::JSONB[]);

-- JSONB path operations
SELECT 
    username,
    profile -> 'name' as name,
    profile -> 'location' ->> 'city' as city,
    profile #> '{social,twitter}' as twitter,
    jsonb_array_length(profile -> 'interests') as interest_count
FROM user_profiles;

-- JSONB aggregation
SELECT 
    jsonb_object_agg(username, profile -> 'location') as user_locations,
    jsonb_agg(profile -> 'interests') as all_interests
FROM user_profiles;

-- JSONB path queries (PostgreSQL 12+)
SELECT 
    username,
    jsonb_path_query_array(profile, '$.interests[*]') as interests,
    jsonb_path_exists(profile, '$.social.twitter') as has_twitter
FROM user_profiles;

-- Complex JSONB operations
WITH interest_analysis AS (
    SELECT 
        username,
        jsonb_array_elements_text(profile -> 'interests') as interest
    FROM user_profiles
)
SELECT 
    interest,
    COUNT(*) as user_count,
    array_agg(username) as users
FROM interest_analysis
GROUP BY interest
ORDER BY user_count DESC;

-- JSONB updates and modifications
UPDATE user_profiles 
SET profile = jsonb_set(
    profile, 
    '{location,timezone}', 
    '"EST"'::jsonb
)
WHERE username = 'alice';

-- Remove JSONB keys
UPDATE user_profiles 
SET profile = profile - 'age'
WHERE username = 'alice';

-- Merge JSONB objects
UPDATE user_profiles 
SET settings = settings || '{"language": "en", "currency": "USD"}'::jsonb
WHERE username = 'alice';
```

## PostgreSQL Extensions

### PostGIS for Spatial Data

```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

-- Create spatial table
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location GEOMETRY(POINT, 4326),
    area GEOMETRY(POLYGON, 4326)
);

-- Insert spatial data
INSERT INTO locations (name, location, area) VALUES
('Central Park', 
 ST_SetSRID(ST_MakePoint(-73.9654, 40.7829), 4326),
 ST_SetSRID(ST_MakePolygon(ST_GeomFromText(
   'LINESTRING(-73.9807 40.7681, -73.9481 40.7681, -73.9481 40.7978, -73.9807 40.7978, -73.9807 40.7681)'
 )), 4326)),
('Times Square',
 ST_SetSRID(ST_MakePoint(-73.9857, 40.7589), 4326),
 NULL);

-- Spatial queries
SELECT 
    name,
    ST_AsText(location) as coordinates,
    ST_X(location) as longitude,
    ST_Y(location) as latitude
FROM locations;

-- Distance calculations
SELECT 
    l1.name as from_location,
    l2.name as to_location,
    ROUND(ST_Distance(l1.location, l2.location)::numeric, 6) as distance_degrees,
    ROUND(ST_DistanceSphere(l1.location, l2.location)::numeric, 2) as distance_meters
FROM locations l1, locations l2
WHERE l1.id < l2.id;

-- Spatial indexing
CREATE INDEX idx_locations_geom ON locations USING GIST (location);
CREATE INDEX idx_locations_area ON locations USING GIST (area);

-- Find nearby locations
SELECT 
    name,
    ST_Distance(location, ST_SetSRID(ST_MakePoint(-73.9857, 40.7589), 4326)) as distance
FROM locations
WHERE ST_DWithin(location, ST_SetSRID(ST_MakePoint(-73.9857, 40.7589), 4326), 0.01)
ORDER BY distance;
```

### pg_stat_statements for Query Analysis

```sql
-- Enable pg_stat_statements (requires restart)
-- Add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Reset statistics
SELECT pg_stat_statements_reset();

-- Analyze query performance
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

-- Find queries with high I/O
SELECT 
    query,
    calls,
    shared_blks_read + shared_blks_written as total_io,
    shared_blks_read,
    shared_blks_written,
    shared_blks_dirtied
FROM pg_stat_statements
WHERE shared_blks_read + shared_blks_written > 0
ORDER BY total_io DESC
LIMIT 10;

-- Queries with low cache hit ratio
SELECT 
    query,
    calls,
    shared_blks_hit,
    shared_blks_read,
    ROUND(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_percent
FROM pg_stat_statements
WHERE shared_blks_read > 0
ORDER BY hit_percent ASC
LIMIT 10;
```

### pg_cron for Scheduled Tasks

```sql
-- Enable pg_cron extension
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Schedule daily maintenance
SELECT cron.schedule('daily-maintenance', '0 2 * * *', 'CALL perform_daily_maintenance();');

-- Schedule hourly statistics update
SELECT cron.schedule('hourly-stats', '0 * * * *', 'REFRESH MATERIALIZED VIEW hourly_stats;');

-- Schedule weekly backup
SELECT cron.schedule('weekly-backup', '0 3 * * 0', 'SELECT backup_database();');

-- View scheduled jobs
SELECT * FROM cron.job;

-- View job run history
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;

-- Unschedule a job
SELECT cron.unschedule('daily-maintenance');

-- Create maintenance procedure
CREATE OR REPLACE PROCEDURE perform_daily_maintenance()
LANGUAGE plpgsql
AS $$
BEGIN
    -- Vacuum and analyze all tables
    PERFORM vacuum_and_analyze_all_tables();
    
    -- Update table statistics
    ANALYZE;
    
    -- Clean up old log entries
    DELETE FROM application_logs WHERE created_at < NOW() - INTERVAL '30 days';
    
    -- Reindex if needed
    PERFORM reindex_if_needed();
    
    -- Log maintenance completion
    INSERT INTO maintenance_log (operation, completed_at) 
    VALUES ('daily_maintenance', NOW());
END;
$$;
```

### pgcrypto for Encryption

```sql
-- Enable pgcrypto extension
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Create table with encrypted data
CREATE TABLE secure_users (
    id SERIAL PRIMARY KEY,
    username TEXT UNIQUE,
    email TEXT,
    password_hash TEXT,
    ssn_encrypted BYTEA,
    credit_card_encrypted BYTEA,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert user with encrypted data
INSERT INTO secure_users (username, email, password_hash, ssn_encrypted, credit_card_encrypted)
VALUES (
    'john_doe',
    'john@example.com',
    crypt('user_password', gen_salt('bf', 8)),  -- Blowfish hash
    pgp_sym_encrypt('123-45-6789', 'encryption_key'),  -- Symmetric encryption
    pgp_sym_encrypt('4532-1234-5678-9012', 'encryption_key')
);

-- Verify password
SELECT 
    username,
    password_hash = crypt('user_password', password_hash) as password_valid
FROM secure_users
WHERE username = 'john_doe';

-- Decrypt sensitive data (in application, not in logs!)
SELECT 
    username,
    email,
    pgp_sym_decrypt(ssn_encrypted, 'encryption_key') as ssn,
    LEFT(pgp_sym_decrypt(credit_card_encrypted, 'encryption_key'), 4) || '-****-****-' || 
    RIGHT(pgp_sym_decrypt(credit_card_encrypted, 'encryption_key'), 4) as masked_card
FROM secure_users
WHERE username = 'john_doe';

-- Generate random data
SELECT 
    gen_random_uuid() as uuid,
    encode(gen_random_bytes(16), 'hex') as random_hex,
    gen_salt('md5') as md5_salt,
    gen_salt('bf', 10) as blowfish_salt;

-- Hash functions
SELECT 
    digest('Hello World', 'sha256') as sha256_hash,
    digest('Hello World', 'md5') as md5_hash,
    hmac('Hello World', 'secret_key', 'sha256') as hmac_sha256;
```

## Custom Data Types

### Creating Composite Types

```sql
-- Create composite types
CREATE TYPE address AS (
    street TEXT,
    city TEXT,
    state TEXT,
    zip_code TEXT,
    country TEXT
);

CREATE TYPE contact_info AS (
    email TEXT,
    phone TEXT,
    address address
);

-- Create table using composite types
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name TEXT,
    contact contact_info,
    billing_address address,
    shipping_address address
);

-- Insert data with composite types
INSERT INTO customers (name, contact, billing_address, shipping_address) VALUES
('John Smith',
 ROW('john@example.com', '555-1234', 
     ROW('123 Main St', 'New York', 'NY', '10001', 'USA')::address)::contact_info,
 ROW('123 Main St', 'New York', 'NY', '10001', 'USA')::address,
 ROW('456 Oak Ave', 'Boston', 'MA', '02101', 'USA')::address);

-- Query composite type fields
SELECT 
    name,
    (contact).email,
    (contact).phone,
    ((contact).address).city as contact_city,
    (billing_address).city as billing_city,
    (shipping_address).city as shipping_city
FROM customers;

-- Update composite type fields
UPDATE customers 
SET contact.email = 'john.smith@newdomain.com'
WHERE id = 1;

-- Functions for composite types
CREATE OR REPLACE FUNCTION format_address(addr address)
RETURNS TEXT AS $$
BEGIN
    RETURN addr.street || ', ' || addr.city || ', ' || 
           addr.state || ' ' || addr.zip_code || ', ' || addr.country;
END;
$$ LANGUAGE plpgsql;

SELECT 
    name,
    format_address(billing_address) as formatted_billing,
    format_address(shipping_address) as formatted_shipping
FROM customers;
```

### Creating Enum Types

```sql
-- Create enum types
CREATE TYPE order_status AS ENUM (
    'pending', 'processing', 'shipped', 'delivered', 'cancelled'
);

CREATE TYPE priority_level AS ENUM (
    'low', 'medium', 'high', 'urgent'
);

-- Create table with enum types
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    status order_status DEFAULT 'pending',
    priority priority_level DEFAULT 'medium',
    order_date TIMESTAMP DEFAULT NOW(),
    total_amount DECIMAL(10,2)
);

-- Insert data with enums
INSERT INTO orders (customer_id, status, priority, total_amount) VALUES
(1, 'processing', 'high', 299.99),
(2, 'shipped', 'medium', 149.50),
(3, 'pending', 'urgent', 599.00);

-- Query with enum comparisons
SELECT 
    id,
    status,
    priority,
    total_amount,
    CASE 
        WHEN priority = 'urgent' THEN 'Process immediately'
        WHEN priority = 'high' THEN 'Process within 2 hours'
        WHEN priority = 'medium' THEN 'Process within 24 hours'
        ELSE 'Process when convenient'
    END as processing_instruction
FROM orders
WHERE status IN ('pending', 'processing')
ORDER BY priority DESC, order_date;

-- Enum ordering
SELECT 
    unnest(enum_range(NULL::order_status)) as status,
    generate_series(1, array_length(enum_range(NULL::order_status), 1)) as sort_order;

-- Add new enum value
ALTER TYPE order_status ADD VALUE 'returned' AFTER 'delivered';

-- Functions with enums
CREATE OR REPLACE FUNCTION get_next_status(current_status order_status)
RETURNS order_status AS $$
BEGIN
    RETURN CASE current_status
        WHEN 'pending' THEN 'processing'
        WHEN 'processing' THEN 'shipped'
        WHEN 'shipped' THEN 'delivered'
        ELSE current_status
    END;
END;
$$ LANGUAGE plpgsql;

SELECT 
    id,
    status,
    get_next_status(status) as next_status
FROM orders;
```

## Advanced Indexing

### Partial Indexes

```sql
-- Create partial indexes for better performance
CREATE TABLE user_sessions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    session_token TEXT,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP
);

-- Partial index on active sessions only
CREATE INDEX idx_active_sessions 
ON user_sessions (user_id, created_at) 
WHERE is_active = TRUE;

-- Partial index on recent sessions
CREATE INDEX idx_recent_sessions 
ON user_sessions (user_id) 
WHERE created_at > NOW() - INTERVAL '30 days';

-- Partial index on expired sessions for cleanup
CREATE INDEX idx_expired_sessions 
ON user_sessions (expires_at) 
WHERE is_active = TRUE AND expires_at < NOW();

-- Test partial index usage
EXPLAIN (ANALYZE, BUFFERS) 
SELECT * FROM user_sessions 
WHERE user_id = 123 AND is_active = TRUE
ORDER BY created_at DESC;
```

### Expression Indexes

```sql
-- Create expression indexes
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    description TEXT,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Index on lowercase name for case-insensitive searches
CREATE INDEX idx_products_name_lower 
ON products (LOWER(name));

-- Index on date part for date-based queries
CREATE INDEX idx_products_created_date 
ON products (DATE(created_at));

-- Index on calculated field
CREATE INDEX idx_products_price_category 
ON products (
    CASE 
        WHEN price < 50 THEN 'budget'
        WHEN price < 200 THEN 'mid-range'
        ELSE 'premium'
    END
);

-- Functional index for full-text search
CREATE INDEX idx_products_search 
ON products USING gin(to_tsvector('english', name || ' ' || description));

-- Test expression index usage
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products 
WHERE LOWER(name) = 'laptop';

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products 
WHERE to_tsvector('english', name || ' ' || description) @@ to_tsquery('laptop & computer');
```

### Covering Indexes (INCLUDE)

```sql
-- Create covering indexes (PostgreSQL 11+)
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    total_price DECIMAL(10,2)
);

-- Covering index includes non-key columns
CREATE INDEX idx_order_items_covering 
ON order_items (order_id, product_id) 
INCLUDE (quantity, unit_price, total_price);

-- This query can be satisfied entirely from the index
EXPLAIN (ANALYZE, BUFFERS)
SELECT product_id, quantity, unit_price, total_price
FROM order_items
WHERE order_id = 123;

-- Unique covering index
CREATE UNIQUE INDEX idx_order_items_unique_covering 
ON order_items (order_id, product_id) 
INCLUDE (quantity);
```

## Parallel Processing

### Parallel Query Configuration

```sql
-- Check parallel query settings
SELECT name, setting, unit, context 
FROM pg_settings 
WHERE name LIKE '%parallel%'
ORDER BY name;

-- Configure parallel processing
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.1;
SET parallel_setup_cost = 1000.0;
SET min_parallel_table_scan_size = '8MB';
SET min_parallel_index_scan_size = '512kB';

-- Create large table for testing
CREATE TABLE large_sales (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    product_id INTEGER,
    sale_date DATE,
    amount DECIMAL(10,2),
    region TEXT
);

-- Insert test data
INSERT INTO large_sales (customer_id, product_id, sale_date, amount, region)
SELECT 
    (random() * 10000)::INTEGER + 1,
    (random() * 1000)::INTEGER + 1,
    '2023-01-01'::DATE + (random() * 365)::INTEGER,
    (random() * 1000 + 10)::DECIMAL(10,2),
    CASE (random() * 4)::INTEGER
        WHEN 0 THEN 'North'
        WHEN 1 THEN 'South'
        WHEN 2 THEN 'East'
        ELSE 'West'
    END
FROM generate_series(1, 1000000);

-- Analyze table for better statistics
ANALYZE large_sales;

-- Test parallel aggregation
EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    region,
    COUNT(*) as sale_count,
    SUM(amount) as total_amount,
    AVG(amount) as avg_amount
FROM large_sales
GROUP BY region;

-- Test parallel join
CREATE TABLE customers AS
SELECT 
    id,
    'Customer ' || id as name,
    CASE (random() * 3)::INTEGER
        WHEN 0 THEN 'Bronze'
        WHEN 1 THEN 'Silver'
        ELSE 'Gold'
    END as tier
FROM generate_series(1, 10000) id;

EXPLAIN (ANALYZE, BUFFERS)
SELECT 
    c.tier,
    COUNT(*) as customer_count,
    SUM(s.amount) as total_sales
FROM large_sales s
JOIN customers c ON s.customer_id = c.id
GROUP BY c.tier;
```

### Parallel Index Creation

```sql
-- Enable parallel index creation
SET maintenance_work_mem = '1GB';
SET max_parallel_maintenance_workers = 4;

-- Create indexes in parallel
CREATE INDEX CONCURRENTLY idx_large_sales_customer_parallel 
ON large_sales (customer_id);

CREATE INDEX CONCURRENTLY idx_large_sales_date_amount_parallel 
ON large_sales (sale_date, amount);

-- Monitor index creation progress
SELECT 
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query
FROM pg_stat_activity
WHERE query LIKE '%CREATE INDEX%';
```

## Foreign Data Wrappers

### postgres_fdw for Remote PostgreSQL

```sql
-- Enable postgres_fdw extension
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

-- Create foreign server
CREATE SERVER remote_postgres
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote-server.example.com', port '5432', dbname 'remote_db');

-- Create user mapping
CREATE USER MAPPING FOR current_user
SERVER remote_postgres
OPTIONS (user 'remote_user', password 'remote_password');

-- Create foreign table
CREATE FOREIGN TABLE remote_orders (
    id INTEGER,
    customer_id INTEGER,
    order_date DATE,
    total_amount DECIMAL(10,2)
)
SERVER remote_postgres
OPTIONS (schema_name 'public', table_name 'orders');

-- Query foreign table
SELECT COUNT(*), SUM(total_amount)
FROM remote_orders
WHERE order_date >= '2024-01-01';

-- Join local and remote data
SELECT 
    l.customer_id,
    l.name,
    COUNT(r.id) as order_count,
    SUM(r.total_amount) as total_spent
FROM customers l
LEFT JOIN remote_orders r ON l.id = r.customer_id
GROUP BY l.customer_id, l.name
ORDER BY total_spent DESC NULLS LAST;

-- Import foreign schema
IMPORT FOREIGN SCHEMA public
LIMIT TO (products, categories)
FROM SERVER remote_postgres
INTO public;
```

### file_fdw for CSV Files

```sql
-- Enable file_fdw extension
CREATE EXTENSION IF NOT EXISTS file_fdw;

-- Create foreign server for files
CREATE SERVER file_server
FOREIGN DATA WRAPPER file_fdw;

-- Create foreign table for CSV file
CREATE FOREIGN TABLE csv_sales (
    sale_id INTEGER,
    customer_name TEXT,
    product_name TEXT,
    sale_date DATE,
    amount DECIMAL(10,2)
)
SERVER file_server
OPTIONS (
    filename '/path/to/sales_data.csv',
    format 'csv',
    header 'true',
    delimiter ','
);

-- Query CSV data
SELECT 
    product_name,
    COUNT(*) as sales_count,
    SUM(amount) as total_revenue
FROM csv_sales
WHERE sale_date >= '2024-01-01'
GROUP BY product_name
ORDER BY total_revenue DESC;

-- Import CSV data into regular table
CREATE TABLE imported_sales AS
SELECT * FROM csv_sales;
```

## Event Triggers

### DDL Event Triggers

```sql
-- Create audit table for DDL changes
CREATE TABLE ddl_audit (
    id SERIAL PRIMARY KEY,
    event_type TEXT,
    object_type TEXT,
    object_name TEXT,
    command_tag TEXT,
    user_name TEXT,
    database_name TEXT,
    schema_name TEXT,
    executed_at TIMESTAMP DEFAULT NOW(),
    command_text TEXT
);

-- Create event trigger function
CREATE OR REPLACE FUNCTION audit_ddl_commands()
RETURNS event_trigger AS $$
DECLARE
    obj record;
BEGIN
    -- Log DDL command
    INSERT INTO ddl_audit (
        event_type,
        command_tag,
        user_name,
        database_name,
        executed_at,
        command_text
    ) VALUES (
        TG_EVENT,
        TG_TAG,
        current_user,
        current_database(),
        NOW(),
        current_query()
    );
    
    -- Log affected objects
    FOR obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        UPDATE ddl_audit 
        SET 
            object_type = obj.object_type,
            object_name = obj.object_identity,
            schema_name = obj.schema_name
        WHERE id = currval('ddl_audit_id_seq');
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Create event trigger
CREATE EVENT TRIGGER audit_ddl_trigger
ON ddl_command_end
EXECUTE FUNCTION audit_ddl_commands();

-- Test the trigger
CREATE TABLE test_table (id SERIAL, name TEXT);
ALTER TABLE test_table ADD COLUMN email TEXT;
DROP TABLE test_table;

-- View audit log
SELECT 
    event_type,
    command_tag,
    object_type,
    object_name,
    user_name,
    executed_at
FROM ddl_audit
ORDER BY executed_at DESC;
```

### Table Rewrite Event Trigger

```sql
-- Create function to prevent table rewrites
CREATE OR REPLACE FUNCTION prevent_table_rewrites()
RETURNS event_trigger AS $$
DECLARE
    obj record;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_table_rewrite_oid()
    LOOP
        RAISE EXCEPTION 'Table rewrite blocked for table: %', obj::regclass;
    END LOOP;
END;
$$ LANGUAGE plpgsql;

-- Create event trigger for table rewrites
CREATE EVENT TRIGGER prevent_rewrites
ON table_rewrite
EXECUTE FUNCTION prevent_table_rewrites();

-- This will be blocked
-- ALTER TABLE large_sales ALTER COLUMN amount TYPE NUMERIC(12,2);

-- Disable the trigger when needed
ALTER EVENT TRIGGER prevent_rewrites DISABLE;
```

## Next Steps

After mastering advanced topics:
1. PostgreSQL Ecosystem and Tools (18-ecosystem-tools.md)
2. Case Studies and Real-world Examples (19-case-studies.md)
3. PostgreSQL Certification and Career Path (20-certification-career.md)

---
*This is part 17 of the PostgreSQL learning series*