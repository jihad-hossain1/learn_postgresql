# Transactions and Concurrency in PostgreSQL

## Introduction

Transactions are fundamental to database systems, ensuring data integrity and consistency. PostgreSQL provides robust transaction support with ACID properties and sophisticated concurrency control mechanisms.

## ACID Properties

### Atomicity
All operations in a transaction succeed or fail together.

```sql
-- Example: Bank transfer (atomic operation)
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;

-- If any operation fails, entire transaction is rolled back
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    -- This will fail if account doesn't exist
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 999;
COMMIT;  -- This won't execute due to error
```

### Consistency
Database remains in a valid state before and after transaction.

```sql
-- Example with constraints ensuring consistency
CREATE TABLE accounts (
    account_id SERIAL PRIMARY KEY,
    balance DECIMAL(10,2) CHECK (balance >= 0),  -- Constraint ensures consistency
    account_type VARCHAR(20) NOT NULL
);

-- This transaction will fail due to constraint violation
BEGIN;
    UPDATE accounts SET balance = -50 WHERE account_id = 1;  -- Violates CHECK constraint
COMMIT;
```

### Isolation
Concurrent transactions don't interfere with each other.

```sql
-- Transaction 1
BEGIN;
    SELECT balance FROM accounts WHERE account_id = 1;  -- Reads 1000
    -- Other transactions can't modify this row until commit
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
COMMIT;

-- Transaction 2 (running concurrently)
BEGIN;
    SELECT balance FROM accounts WHERE account_id = 1;  -- May see different values based on isolation level
COMMIT;
```

### Durability
Committed changes persist even after system failure.

```sql
-- Once committed, changes are permanent
BEGIN;
    INSERT INTO transactions (account_id, amount, transaction_date) 
    VALUES (1, -100, NOW());
COMMIT;  -- Data is now durable
```

## Basic Transaction Control

### Transaction Commands

```sql
-- Start transaction explicitly
BEGIN;
-- or
START TRANSACTION;

-- Commit changes
COMMIT;

-- Rollback changes
ROLLBACK;

-- Savepoints for partial rollback
BEGIN;
    INSERT INTO users (name) VALUES ('John');
    SAVEPOINT sp1;
    
    INSERT INTO users (name) VALUES ('Jane');
    SAVEPOINT sp2;
    
    INSERT INTO users (name) VALUES ('Invalid Data That Causes Error');
    
    -- Rollback to savepoint
    ROLLBACK TO SAVEPOINT sp2;
    
    -- Continue with transaction
    INSERT INTO users (name) VALUES ('Bob');
COMMIT;  -- John and Bob are inserted, Jane is not
```

### Autocommit vs Explicit Transactions

```sql
-- Autocommit mode (default)
INSERT INTO users (name) VALUES ('Auto User');  -- Automatically committed

-- Explicit transaction
BEGIN;
    INSERT INTO users (name) VALUES ('Transaction User 1');
    INSERT INTO users (name) VALUES ('Transaction User 2');
COMMIT;  -- Both inserts committed together

-- Transaction with rollback
BEGIN;
    INSERT INTO users (name) VALUES ('Temp User');
    -- Decide to cancel
ROLLBACK;  -- Insert is undone
```

## Isolation Levels

PostgreSQL supports four isolation levels defined by the SQL standard.

### Read Uncommitted

```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
    SELECT balance FROM accounts WHERE account_id = 1;  -- May see uncommitted changes
COMMIT;

-- Session 2 (concurrent)
BEGIN;
    UPDATE accounts SET balance = balance + 1000 WHERE account_id = 1;
    -- Don't commit yet - Session 1 might still see this change
```

### Read Committed (Default)

```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
    SELECT balance FROM accounts WHERE account_id = 1;  -- Sees 1000
    -- Wait for Session 2 to commit
    SELECT balance FROM accounts WHERE account_id = 1;  -- Now sees 2000
COMMIT;

-- Session 2
BEGIN;
    UPDATE accounts SET balance = balance + 1000 WHERE account_id = 1;
COMMIT;  -- Now Session 1 can see the change
```

### Repeatable Read

```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT balance FROM accounts WHERE account_id = 1;  -- Sees 1000
    -- Even after Session 2 commits, this will still see 1000
    SELECT balance FROM accounts WHERE account_id = 1;  -- Still sees 1000
COMMIT;

-- Session 2
BEGIN;
    UPDATE accounts SET balance = balance + 1000 WHERE account_id = 1;
COMMIT;
```

### Serializable

```sql
-- Session 1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT COUNT(*) FROM accounts WHERE balance > 500;  -- Sees 5
    INSERT INTO accounts (balance) VALUES (600);
COMMIT;

-- Session 2 (concurrent)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT COUNT(*) FROM accounts WHERE balance > 500;  -- Sees 5
    INSERT INTO accounts (balance) VALUES (700);
COMMIT;  -- May fail with serialization error
```

### Setting Isolation Levels

```sql
-- For current transaction
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- For current session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Check current isolation level
SHOW transaction_isolation;

-- In transaction block
BEGIN ISOLATION LEVEL READ COMMITTED;
    -- Transaction operations
COMMIT;
```

## Locking Mechanisms

### Row-Level Locks

```sql
-- Exclusive lock (FOR UPDATE)
BEGIN;
    SELECT * FROM accounts WHERE account_id = 1 FOR UPDATE;
    -- Row is locked until transaction ends
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
COMMIT;

-- Shared lock (FOR SHARE)
BEGIN;
    SELECT * FROM accounts WHERE account_id = 1 FOR SHARE;
    -- Other sessions can read but not modify
COMMIT;

-- Lock with NOWAIT
BEGIN;
    SELECT * FROM accounts WHERE account_id = 1 FOR UPDATE NOWAIT;
    -- Fails immediately if row is already locked
COMMIT;

-- Lock with SKIP LOCKED
BEGIN;
    SELECT * FROM accounts WHERE account_id IN (1, 2, 3) FOR UPDATE SKIP LOCKED;
    -- Returns only unlocked rows
COMMIT;
```

### Table-Level Locks

```sql
-- Explicit table locking
BEGIN;
    LOCK TABLE accounts IN ACCESS EXCLUSIVE MODE;
    -- No other transactions can access the table
    ALTER TABLE accounts ADD COLUMN new_column TEXT;
COMMIT;

-- Different lock modes
BEGIN;
    LOCK TABLE accounts IN SHARE MODE;  -- Allows concurrent reads
    -- Perform read-heavy operations
COMMIT;

-- Lock multiple tables
BEGIN;
    LOCK TABLE accounts, transactions IN SHARE ROW EXCLUSIVE MODE;
    -- Perform operations on both tables
COMMIT;
```

### Advisory Locks

```sql
-- Session-level advisory locks
SELECT pg_advisory_lock(12345);  -- Acquire lock
-- Perform critical operations
SELECT pg_advisory_unlock(12345);  -- Release lock

-- Transaction-level advisory locks
BEGIN;
    SELECT pg_advisory_xact_lock(12345);  -- Auto-released at transaction end
    -- Perform operations
COMMIT;

-- Try to acquire lock without waiting
SELECT pg_try_advisory_lock(12345);  -- Returns true if acquired, false if not

-- Check if lock is held
SELECT pg_advisory_lock_held(12345);
```

## Deadlock Detection and Prevention

### Understanding Deadlocks

```sql
-- Example that can cause deadlock

-- Session 1
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    -- Now try to update account 2
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;

-- Session 2 (concurrent)
BEGIN;
    UPDATE accounts SET balance = balance - 50 WHERE account_id = 2;
    -- Now try to update account 1 - DEADLOCK!
    UPDATE accounts SET balance = balance + 50 WHERE account_id = 1;
COMMIT;
```

### Deadlock Prevention Strategies

```sql
-- Strategy 1: Consistent ordering
CREATE OR REPLACE FUNCTION transfer_money(
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL
)
RETURNS BOOLEAN
AS $$
DECLARE
    first_account INTEGER;
    second_account INTEGER;
BEGIN
    -- Always lock accounts in order of ID to prevent deadlocks
    IF from_account < to_account THEN
        first_account := from_account;
        second_account := to_account;
    ELSE
        first_account := to_account;
        second_account := from_account;
    END IF;
    
    -- Lock in consistent order
    PERFORM balance FROM accounts WHERE account_id = first_account FOR UPDATE;
    PERFORM balance FROM accounts WHERE account_id = second_account FOR UPDATE;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Strategy 2: Use advisory locks
CREATE OR REPLACE FUNCTION safe_transfer(
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL
)
RETURNS BOOLEAN
AS $$
BEGIN
    -- Acquire advisory locks in consistent order
    IF from_account < to_account THEN
        PERFORM pg_advisory_xact_lock(from_account);
        PERFORM pg_advisory_xact_lock(to_account);
    ELSE
        PERFORM pg_advisory_xact_lock(to_account);
        PERFORM pg_advisory_xact_lock(from_account);
    END IF;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

### Deadlock Detection and Handling

```sql
-- PostgreSQL automatically detects deadlocks
-- Configure deadlock timeout
SET deadlock_timeout = '1s';  -- Check for deadlocks after 1 second

-- Handle deadlocks in application code
CREATE OR REPLACE FUNCTION retry_on_deadlock(
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL,
    max_retries INTEGER DEFAULT 3
)
RETURNS BOOLEAN
AS $$
DECLARE
    retry_count INTEGER := 0;
BEGIN
    LOOP
        BEGIN
            -- Attempt transfer
            UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
            UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
            RETURN TRUE;
        EXCEPTION
            WHEN deadlock_detected THEN
                retry_count := retry_count + 1;
                IF retry_count >= max_retries THEN
                    RAISE EXCEPTION 'Transfer failed after % retries due to deadlocks', max_retries;
                END IF;
                -- Wait a random amount before retry
                PERFORM pg_sleep(random() * 0.1);
        END;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

## Monitoring Locks and Transactions

### Current Locks

```sql
-- View current locks
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
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE a.datname = current_database()
ORDER BY l.granted, l.pid;

-- View blocking locks
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
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
```

### Transaction Information

```sql
-- Current transactions
SELECT 
    pid,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    xact_start,
    query
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
ORDER BY xact_start;

-- Long-running transactions
SELECT 
    pid,
    usename,
    application_name,
    state,
    NOW() - xact_start AS transaction_duration,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
AND NOW() - xact_start > INTERVAL '5 minutes'
ORDER BY xact_start;

-- Transaction wraparound information
SELECT 
    datname,
    age(datfrozenxid) as xid_age,
    datfrozenxid
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

## Concurrency Patterns

### Optimistic Locking

```sql
-- Add version column for optimistic locking
ALTER TABLE accounts ADD COLUMN version INTEGER DEFAULT 1;

-- Optimistic update function
CREATE OR REPLACE FUNCTION optimistic_update_balance(
    p_account_id INTEGER,
    p_new_balance DECIMAL,
    p_expected_version INTEGER
)
RETURNS BOOLEAN
AS $$
DECLARE
    rows_affected INTEGER;
BEGIN
    UPDATE accounts 
    SET balance = p_new_balance, 
        version = version + 1
    WHERE account_id = p_account_id 
    AND version = p_expected_version;
    
    GET DIAGNOSTICS rows_affected = ROW_COUNT;
    
    IF rows_affected = 0 THEN
        RAISE EXCEPTION 'Optimistic lock failed - record was modified by another transaction';
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Usage
BEGIN;
    -- Read current state
    SELECT balance, version FROM accounts WHERE account_id = 1;
    -- Returns: balance=1000, version=5
    
    -- Update with version check
    SELECT optimistic_update_balance(1, 900, 5);
COMMIT;
```

### Pessimistic Locking

```sql
-- Pessimistic locking function
CREATE OR REPLACE FUNCTION pessimistic_transfer(
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL
)
RETURNS BOOLEAN
AS $$
DECLARE
    from_balance DECIMAL;
    to_balance DECIMAL;
BEGIN
    -- Lock both accounts immediately
    SELECT balance INTO from_balance 
    FROM accounts 
    WHERE account_id = from_account 
    FOR UPDATE;
    
    SELECT balance INTO to_balance 
    FROM accounts 
    WHERE account_id = to_account 
    FOR UPDATE;
    
    -- Check sufficient funds
    IF from_balance < amount THEN
        RAISE EXCEPTION 'Insufficient funds';
    END IF;
    
    -- Perform transfer
    UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

### Queue Processing Pattern

```sql
-- Job queue table
CREATE TABLE job_queue (
    job_id SERIAL PRIMARY KEY,
    job_type VARCHAR(50),
    payload JSONB,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW(),
    processed_at TIMESTAMP,
    worker_id VARCHAR(50)
);

-- Safe job processing function
CREATE OR REPLACE FUNCTION process_next_job(worker_name VARCHAR(50))
RETURNS TABLE(
    job_id INTEGER,
    job_type VARCHAR(50),
    payload JSONB
)
AS $$
DECLARE
    selected_job RECORD;
BEGIN
    -- Atomically claim a job
    UPDATE job_queue 
    SET status = 'processing',
        worker_id = worker_name,
        processed_at = NOW()
    WHERE job_id = (
        SELECT jq.job_id
        FROM job_queue jq
        WHERE jq.status = 'pending'
        ORDER BY jq.created_at
        FOR UPDATE SKIP LOCKED
        LIMIT 1
    )
    RETURNING job_queue.job_id, job_queue.job_type, job_queue.payload
    INTO selected_job;
    
    IF selected_job.job_id IS NOT NULL THEN
        RETURN QUERY
        SELECT selected_job.job_id, selected_job.job_type, selected_job.payload;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Mark job as completed
CREATE OR REPLACE FUNCTION complete_job(p_job_id INTEGER)
RETURNS BOOLEAN
AS $$
BEGIN
    UPDATE job_queue 
    SET status = 'completed'
    WHERE job_id = p_job_id;
    
    RETURN FOUND;
END;
$$ LANGUAGE plpgsql;
```

## Performance Tuning

### Connection Pooling

```sql
-- Configure connection limits
ALTER SYSTEM SET max_connections = 200;
ALTER SYSTEM SET shared_buffers = '256MB';

-- Monitor connection usage
SELECT 
    count(*) as total_connections,
    count(*) FILTER (WHERE state = 'active') as active_connections,
    count(*) FILTER (WHERE state = 'idle') as idle_connections,
    count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction
FROM pg_stat_activity;
```

### Lock Monitoring and Tuning

```sql
-- Configure lock timeouts
SET lock_timeout = '30s';  -- Timeout for acquiring locks
SET statement_timeout = '60s';  -- Timeout for statement execution

-- Monitor lock waits
SELECT 
    schemaname,
    tablename,
    attname,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_tup_hot_upd
FROM pg_stat_user_tables
WHERE n_tup_upd > 1000
ORDER BY n_tup_upd DESC;
```

### Transaction Log Management

```sql
-- Configure WAL settings
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET max_wal_size = '2GB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;

-- Monitor WAL usage
SELECT 
    pg_current_wal_lsn(),
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 AS wal_mb;

-- Check checkpoint activity
SELECT 
    checkpoints_timed,
    checkpoints_req,
    checkpoint_write_time,
    checkpoint_sync_time
FROM pg_stat_bgwriter;
```

## Best Practices

### Transaction Design

1. **Keep transactions short** - Minimize lock duration
2. **Consistent lock ordering** - Prevent deadlocks
3. **Use appropriate isolation levels** - Balance consistency and performance
4. **Handle exceptions properly** - Always clean up resources
5. **Use savepoints for complex operations** - Allow partial rollback

### Concurrency Best Practices

```sql
-- Good: Short transaction
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
    INSERT INTO transaction_log (account_id, amount) VALUES (1, -100);
COMMIT;

-- Bad: Long transaction with user interaction
-- DON'T DO THIS
BEGIN;
    SELECT * FROM accounts WHERE account_id = 1 FOR UPDATE;
    -- Wait for user input (locks held too long)
    -- ... user thinks for 5 minutes ...
    UPDATE accounts SET balance = new_balance WHERE account_id = 1;
COMMIT;

-- Good: Optimistic approach
DO $$
DECLARE
    current_version INTEGER;
    new_balance DECIMAL;
BEGIN
    -- Read without locking
    SELECT version, balance INTO current_version, new_balance 
    FROM accounts WHERE account_id = 1;
    
    -- Calculate new balance
    new_balance := new_balance - 100;
    
    -- Quick update with version check
    BEGIN
        UPDATE accounts 
        SET balance = new_balance, version = version + 1
        WHERE account_id = 1 AND version = current_version;
        
        IF NOT FOUND THEN
            RAISE EXCEPTION 'Account was modified by another transaction';
        END IF;
    END;
END $$;
```

### Error Handling

```sql
-- Comprehensive error handling
CREATE OR REPLACE FUNCTION robust_transfer(
    from_account INTEGER,
    to_account INTEGER,
    amount DECIMAL
)
RETURNS BOOLEAN
AS $$
DECLARE
    from_balance DECIMAL;
BEGIN
    -- Validate inputs
    IF amount <= 0 THEN
        RAISE EXCEPTION 'Transfer amount must be positive';
    END IF;
    
    IF from_account = to_account THEN
        RAISE EXCEPTION 'Cannot transfer to same account';
    END IF;
    
    BEGIN
        -- Check source account balance
        SELECT balance INTO STRICT from_balance
        FROM accounts 
        WHERE account_id = from_account
        FOR UPDATE;
        
        IF from_balance < amount THEN
            RAISE EXCEPTION 'Insufficient funds: % available, % requested', 
                           from_balance, amount;
        END IF;
        
        -- Verify destination account exists
        PERFORM 1 FROM accounts WHERE account_id = to_account FOR UPDATE;
        
        -- Perform transfer
        UPDATE accounts SET balance = balance - amount WHERE account_id = from_account;
        UPDATE accounts SET balance = balance + amount WHERE account_id = to_account;
        
        -- Log transaction
        INSERT INTO transfer_log (from_account, to_account, amount, transfer_date)
        VALUES (from_account, to_account, amount, NOW());
        
        RETURN TRUE;
        
    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            RAISE EXCEPTION 'Account not found';
        WHEN TOO_MANY_ROWS THEN
            RAISE EXCEPTION 'Multiple accounts found - data integrity issue';
        WHEN deadlock_detected THEN
            RAISE EXCEPTION 'Transfer failed due to deadlock - please retry';
        WHEN OTHERS THEN
            RAISE EXCEPTION 'Transfer failed: %', SQLERRM;
    END;
END;
$$ LANGUAGE plpgsql;
```

## Troubleshooting Common Issues

### Identifying Lock Contention

```sql
-- Find tables with high lock contention
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins + n_tup_upd + n_tup_del as total_writes
FROM pg_stat_user_tables
WHERE n_tup_ins + n_tup_upd + n_tup_del > 1000
ORDER BY total_writes DESC;

-- Find slow queries that might be holding locks
SELECT 
    pid,
    usename,
    application_name,
    state,
    NOW() - query_start AS query_duration,
    NOW() - xact_start AS transaction_duration,
    query
FROM pg_stat_activity
WHERE state = 'active'
AND NOW() - query_start > INTERVAL '1 minute'
ORDER BY query_start;
```

### Resolving Deadlocks

```sql
-- Enable deadlock logging
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '1s';
ALTER SYSTEM SET log_statement = 'all';  -- For debugging only

-- Check PostgreSQL logs for deadlock details
-- Look for messages like:
-- "DETAIL: Process 12345 waits for ShareLock on transaction 67890"
```

## Next Steps

After mastering transactions and concurrency:
1. Backup and Recovery (10-backup-recovery.md)
2. Security and User Management (11-security-users.md)
3. Monitoring and Maintenance (12-monitoring-maintenance.md)

---
*This is part 9 of the PostgreSQL learning series*