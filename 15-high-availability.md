# High Availability and Clustering in PostgreSQL

## Introduction

High Availability (HA) ensures that PostgreSQL databases remain accessible and operational even during hardware failures, software issues, or maintenance activities. This guide covers various HA solutions, replication methods, clustering techniques, and disaster recovery strategies.

## High Availability Concepts

### Key Metrics

- **RTO (Recovery Time Objective)**: Maximum acceptable downtime
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss
- **Availability**: Percentage of uptime (99.9% = 8.76 hours downtime/year)
- **MTTR (Mean Time To Recovery)**: Average time to restore service
- **MTBF (Mean Time Between Failures)**: Average time between system failures

### HA Architecture Patterns

```sql
-- HA configuration assessment
CREATE TABLE ha_requirements (
    requirement_id SERIAL PRIMARY KEY,
    service_tier VARCHAR(20),
    availability_target NUMERIC(5,3),
    rto_minutes INTEGER,
    rpo_minutes INTEGER,
    annual_downtime_hours NUMERIC(6,2)
);

INSERT INTO ha_requirements VALUES
(1, 'Basic', 99.0, 60, 15, 87.6),
(2, 'Standard', 99.9, 15, 5, 8.76),
(3, 'Premium', 99.95, 5, 1, 4.38),
(4, 'Enterprise', 99.99, 1, 0, 0.876);

-- Calculate downtime impact
SELECT 
    service_tier,
    availability_target || '%' as availability,
    rto_minutes || ' min' as max_recovery_time,
    rpo_minutes || ' min' as max_data_loss,
    annual_downtime_hours || ' hours' as yearly_downtime,
    ROUND(annual_downtime_hours * 60, 1) || ' minutes' as yearly_downtime_minutes
FROM ha_requirements
ORDER BY availability_target DESC;
```

## PostgreSQL Replication

### Streaming Replication Setup

#### Primary Server Configuration

```bash
# postgresql.conf on primary server
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 1GB
hot_standby = on
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
wal_sender_timeout = 60s
wal_receiver_timeout = 60s
wal_receiver_status_interval = 10s
max_standby_streaming_delay = 30s
max_standby_archive_delay = 30s
```

```bash
# pg_hba.conf on primary server
# Allow replication connections
host replication replicator 192.168.1.0/24 md5
host replication replicator 10.0.0.0/8 md5
```

```sql
-- Create replication user on primary
CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE postgres TO replicator;
```

#### Standby Server Setup

```bash
#!/bin/bash
# setup_standby.sh

PRIMARY_HOST="192.168.1.10"
STANDBY_DATA_DIR="/var/lib/postgresql/15/main"
REPLICATOR_USER="replicator"

# Stop PostgreSQL on standby
sudo systemctl stop postgresql

# Remove existing data directory
sudo rm -rf $STANDBY_DATA_DIR/*

# Create base backup from primary
sudo -u postgres pg_basebackup \
    -h $PRIMARY_HOST \
    -D $STANDBY_DATA_DIR \
    -U $REPLICATOR_USER \
    -P -v -R -W

# Create standby.signal file
sudo -u postgres touch $STANDBY_DATA_DIR/standby.signal

# Configure standby-specific settings
sudo -u postgres cat >> $STANDBY_DATA_DIR/postgresql.auto.conf << EOF
primary_conninfo = 'host=$PRIMARY_HOST port=5432 user=$REPLICATOR_USER password=secure_password application_name=standby1'
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_timeline = 'latest'
EOF

# Start PostgreSQL on standby
sudo systemctl start postgresql

echo "Standby server setup completed"
```

### Monitoring Replication

```sql
-- On primary: Check replication status
SELECT 
    client_addr,
    application_name,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;

-- On standby: Check recovery status
SELECT 
    pg_is_in_recovery() as is_standby,
    pg_last_wal_receive_lsn() as last_received,
    pg_last_wal_replay_lsn() as last_replayed,
    pg_last_xact_replay_timestamp() as last_replay_time,
    EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) as lag_seconds;

-- Replication lag monitoring function
CREATE OR REPLACE FUNCTION check_replication_lag()
RETURNS TABLE(
    standby_name TEXT,
    lag_bytes BIGINT,
    lag_seconds NUMERIC,
    status TEXT
)
AS $$
BEGIN
    IF pg_is_in_recovery() THEN
        -- This is a standby server
        RETURN QUERY
        SELECT 
            'current_standby'::TEXT,
            pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()),
            EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())),
            CASE 
                WHEN EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) > 60 THEN 'LAGGING'
                WHEN EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) > 30 THEN 'WARNING'
                ELSE 'OK'
            END;
    ELSE
        -- This is a primary server
        RETURN QUERY
        SELECT 
            r.application_name::TEXT,
            pg_wal_lsn_diff(r.sent_lsn, r.replay_lsn),
            EXTRACT(EPOCH FROM r.replay_lag),
            CASE 
                WHEN r.state != 'streaming' THEN 'DISCONNECTED'
                WHEN EXTRACT(EPOCH FROM r.replay_lag) > 60 THEN 'LAGGING'
                WHEN EXTRACT(EPOCH FROM r.replay_lag) > 30 THEN 'WARNING'
                ELSE 'OK'
            END
        FROM pg_stat_replication r;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Monitor replication health
SELECT * FROM check_replication_lag();
```

### Synchronous Replication

```bash
# postgresql.conf for synchronous replication
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
# or for all standbys: synchronous_standby_names = '*'
# or for specific priority: synchronous_standby_names = 'standby1, standby2'
```

```sql
-- Check synchronous replication status
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    sync_priority,
    replay_lag
FROM pg_stat_replication
ORDER BY sync_priority;

-- Test synchronous replication
BEGIN;
CREATE TABLE sync_test (id SERIAL, created_at TIMESTAMP DEFAULT NOW());
INSERT INTO sync_test DEFAULT VALUES;
COMMIT;

-- Verify on standby (should appear immediately)
SELECT * FROM sync_test;
```

## Automatic Failover Solutions

### Patroni Setup

#### Patroni Configuration

```yaml
# /etc/patroni/patroni.yml
scope: postgres-cluster
namespace: /db/
name: postgres-node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.1.10:8008

etcd3:
  hosts: 192.168.1.20:2379,192.168.1.21:2379,192.168.1.22:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: false
    synchronous_mode_strict: false
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_connections: 100
        max_worker_processes: 8
        wal_keep_size: 128MB
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        track_commit_timestamp: "on"
        archive_mode: "on"
        archive_timeout: 1800s
        archive_command: "cp %p /var/lib/postgresql/wal_archive/%f"
      recovery_conf:
        restore_command: "cp /var/lib/postgresql/wal_archive/%f %p"

  initdb:
  - encoding: UTF8
  - data-checksums

  pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 192.168.1.0/24 md5
  - host all all 0.0.0.0/0 md5

  users:
    admin:
      password: admin_password
      options:
        - createrole
        - createdb

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.1.10:5432
  data_dir: /var/lib/postgresql/15/main
  bin_dir: /usr/lib/postgresql/15/bin
  config_dir: /etc/postgresql/15/main
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: replicator
      password: replicator_password
    superuser:
      username: postgres
      password: postgres_password
  parameters:
    unix_socket_directories: '/var/run/postgresql'

tags:
    nofailover: false
    noloadbalance: false
    clonefrom: false
    nosync: false
```

#### Patroni Service Setup

```bash
#!/bin/bash
# install_patroni.sh

# Install dependencies
sudo apt-get update
sudo apt-get install -y python3-pip python3-dev libpq-dev

# Install Patroni
sudo pip3 install patroni[etcd3]

# Create Patroni user
sudo useradd -r -s /bin/false patroni

# Create directories
sudo mkdir -p /etc/patroni
sudo mkdir -p /var/lib/postgresql/wal_archive
sudo chown postgres:postgres /var/lib/postgresql/wal_archive

# Create systemd service
sudo cat > /etc/systemd/system/patroni.service << EOF
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
EOF

# Enable and start Patroni
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni

echo "Patroni installation completed"
```

#### Patroni Management

```bash
# Check cluster status
patronctl -c /etc/patroni/patroni.yml list

# Manual failover
patronctl -c /etc/patroni/patroni.yml failover

# Switchover (planned)
patronctl -c /etc/patroni/patroni.yml switchover

# Restart cluster member
patronctl -c /etc/patroni/patroni.yml restart postgres-cluster postgres-node1

# Reload configuration
patronctl -c /etc/patroni/patroni.yml reload postgres-cluster

# Show cluster configuration
patronctl -c /etc/patroni/patroni.yml show-config
```

### HAProxy Load Balancer

```bash
# /etc/haproxy/haproxy.cfg
global
    maxconn 100
    log stdout local0

defaults
    log global
    mode tcp
    retries 2
    timeout client 30m
    timeout connect 4s
    timeout server 30m
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres-write
    bind *:5000
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres-node1 192.168.1.10:5432 maxconn 100 check port 8008 httpchk GET /master
    server postgres-node2 192.168.1.11:5432 maxconn 100 check port 8008 httpchk GET /master
    server postgres-node3 192.168.1.12:5432 maxconn 100 check port 8008 httpchk GET /master

listen postgres-read
    bind *:5001
    balance roundrobin
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgres-node1 192.168.1.10:5432 maxconn 100 check port 8008 httpchk GET /replica
    server postgres-node2 192.168.1.11:5432 maxconn 100 check port 8008 httpchk GET /replica
    server postgres-node3 192.168.1.12:5432 maxconn 100 check port 8008 httpchk GET /replica
```

## Connection Pooling for HA

### PgBouncer HA Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
; Primary database (writes)
mydb_write = host=192.168.1.10 port=5000 dbname=mydb
; Read replicas (reads)
mydb_read = host=192.168.1.10 port=5001 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

; Pool settings
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
max_db_connections = 100

; Health checks
server_check_delay = 30
server_check_query = SELECT 1
server_lifetime = 3600
server_idle_timeout = 600

; Failover settings
server_reset_query = DISCARD ALL
server_reset_query_always = 0
```

### Application Connection Handling

```python
# Python application with HA connection handling
import psycopg2
from psycopg2 import pool
import time
import logging

class HADatabaseManager:
    def __init__(self):
        self.write_pool = psycopg2.pool.ThreadedConnectionPool(
            5, 20,
            host="192.168.1.10",
            port=5000,  # HAProxy write port
            database="mydb",
            user="app_user",
            password="app_password",
            application_name="app_writer"
        )
        
        self.read_pool = psycopg2.pool.ThreadedConnectionPool(
            10, 50,
            host="192.168.1.10",
            port=5001,  # HAProxy read port
            database="mydb",
            user="app_user",
            password="app_password",
            application_name="app_reader"
        )
    
    def execute_write(self, query, params=None, retries=3):
        for attempt in range(retries):
            conn = None
            try:
                conn = self.write_pool.getconn()
                with conn.cursor() as cursor:
                    cursor.execute(query, params)
                    result = cursor.fetchall() if cursor.description else None
                    conn.commit()
                    return result
            except Exception as e:
                if conn:
                    conn.rollback()
                logging.error(f"Write attempt {attempt + 1} failed: {e}")
                if attempt == retries - 1:
                    raise
                time.sleep(2 ** attempt)  # Exponential backoff
            finally:
                if conn:
                    self.write_pool.putconn(conn)
    
    def execute_read(self, query, params=None, retries=3):
        for attempt in range(retries):
            conn = None
            try:
                conn = self.read_pool.getconn()
                with conn.cursor() as cursor:
                    cursor.execute(query, params)
                    return cursor.fetchall()
            except Exception as e:
                logging.error(f"Read attempt {attempt + 1} failed: {e}")
                if attempt == retries - 1:
                    raise
                time.sleep(1)  # Short delay for reads
            finally:
                if conn:
                    self.read_pool.putconn(conn)

# Usage example
db = HADatabaseManager()

# Write operations go to primary
db.execute_write(
    "INSERT INTO orders (customer_id, amount) VALUES (%s, %s)",
    (1001, 299.99)
)

# Read operations go to replicas
orders = db.execute_read(
    "SELECT * FROM orders WHERE customer_id = %s",
    (1001,)
)
```

## Logical Replication

### Setting Up Logical Replication

```sql
-- On publisher (source) database
-- Configure postgresql.conf
-- wal_level = logical
-- max_replication_slots = 10
-- max_wal_senders = 10

-- Create publication
CREATE PUBLICATION orders_pub FOR TABLE orders, customers;

-- Or publish all tables
CREATE PUBLICATION all_tables_pub FOR ALL TABLES;

-- Add tables to existing publication
ALTER PUBLICATION orders_pub ADD TABLE products;

-- Remove tables from publication
ALTER PUBLICATION orders_pub DROP TABLE products;

-- View publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables WHERE pubname = 'orders_pub';
```

```sql
-- On subscriber (target) database
-- Create subscription
CREATE SUBSCRIPTION orders_sub
CONNECTION 'host=192.168.1.10 port=5432 user=replicator dbname=mydb'
PUBLICATION orders_pub;

-- View subscriptions
SELECT * FROM pg_subscription;

-- Check subscription status
SELECT 
    subname,
    pid,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time,
    latest_end_lsn,
    latest_end_time
FROM pg_stat_subscription;

-- Disable/enable subscription
ALTER SUBSCRIPTION orders_sub DISABLE;
ALTER SUBSCRIPTION orders_sub ENABLE;

-- Refresh subscription (sync schema changes)
ALTER SUBSCRIPTION orders_sub REFRESH PUBLICATION;

-- Drop subscription
DROP SUBSCRIPTION orders_sub;
```

### Logical Replication Monitoring

```sql
-- Monitor logical replication lag
CREATE OR REPLACE FUNCTION monitor_logical_replication()
RETURNS TABLE(
    subscription_name TEXT,
    publication_name TEXT,
    lag_bytes BIGINT,
    lag_seconds NUMERIC,
    status TEXT
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        s.subname::TEXT,
        array_to_string(s.subpublications, ', ')::TEXT,
        pg_wal_lsn_diff(pg_current_wal_lsn(), ss.received_lsn),
        EXTRACT(EPOCH FROM (NOW() - ss.last_msg_receipt_time)),
        CASE 
            WHEN ss.pid IS NULL THEN 'STOPPED'
            WHEN EXTRACT(EPOCH FROM (NOW() - ss.last_msg_receipt_time)) > 300 THEN 'LAGGING'
            WHEN EXTRACT(EPOCH FROM (NOW() - ss.last_msg_receipt_time)) > 60 THEN 'WARNING'
            ELSE 'OK'
        END
    FROM pg_subscription s
    LEFT JOIN pg_stat_subscription ss ON s.oid = ss.subid;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM monitor_logical_replication();

-- Check replication conflicts
SELECT 
    datname,
    confl_tablespace,
    confl_lock,
    confl_snapshot,
    confl_bufferpin,
    confl_deadlock
FROM pg_stat_database_conflicts;
```

## Multi-Master Replication

### BDR (Bi-Directional Replication) Setup

```sql
-- Install BDR extension (requires 2ndQuadrant/EDB license)
CREATE EXTENSION bdr;

-- Create BDR group
SELECT bdr.create_node_group(
    node_group_name := 'myapp_group'
);

-- Join node to group
SELECT bdr.create_node(
    node_name := 'node1',
    local_dsn := 'host=192.168.1.10 port=5432 dbname=mydb'
);

SELECT bdr.join_node_group(
    node_group_name := 'myapp_group',
    node_name := 'node1'
);

-- Add second node
SELECT bdr.create_node(
    node_name := 'node2',
    local_dsn := 'host=192.168.1.11 port=5432 dbname=mydb'
);

SELECT bdr.join_node_group(
    node_group_name := 'myapp_group',
    node_name := 'node2',
    join_using_dsn := 'host=192.168.1.10 port=5432 dbname=mydb'
);

-- Monitor BDR status
SELECT 
    node_name,
    node_group_name,
    node_kind,
    node_status
FROM bdr.node_summary;

-- Check replication status
SELECT 
    origin_name,
    received_lsn,
    last_msg_send_time,
    last_msg_receipt_time
FROM pg_stat_subscription;
```

### Conflict Resolution

```sql
-- Configure conflict resolution
SELECT bdr.alter_node_group(
    node_group_name := 'myapp_group',
    config_data := '{
        "conflict_resolution": {
            "update_update": "last_update_wins",
            "update_delete": "delete_wins",
            "insert_insert": "insert_or_error"
        }
    }'
);

-- Create conflict resolution triggers
CREATE OR REPLACE FUNCTION resolve_order_conflicts()
RETURNS TRIGGER
AS $$
BEGIN
    -- Custom conflict resolution logic
    IF TG_OP = 'UPDATE' THEN
        -- Use highest order amount in case of conflict
        IF NEW.amount > OLD.amount THEN
            RETURN NEW;
        ELSE
            RETURN OLD;
        END IF;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER order_conflict_resolution
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION resolve_order_conflicts();

-- Monitor conflicts
SELECT 
    nspname,
    relname,
    conflict_type,
    conflict_resolution,
    conflict_time
FROM bdr.conflict_history
WHERE conflict_time >= NOW() - INTERVAL '24 hours'
ORDER BY conflict_time DESC;
```

## Disaster Recovery

### Backup Strategy for HA

```bash
#!/bin/bash
# ha_backup_strategy.sh

BACKUP_DIR="/backup/postgresql"
S3_BUCKET="my-postgres-backups"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)

# Function to backup from primary
backup_primary() {
    echo "Starting backup from primary server..."
    
    # Create base backup
    pg_basebackup -D "$BACKUP_DIR/base_$DATE" \
                  -Ft -z -P -v \
                  -h primary.example.com \
                  -U backup_user
    
    if [ $? -eq 0 ]; then
        echo "Base backup completed successfully"
        
        # Upload to S3
        aws s3 cp "$BACKUP_DIR/base_$DATE.tar.gz" \
                  "s3://$S3_BUCKET/base_backups/"
        
        # Create backup metadata
        cat > "$BACKUP_DIR/backup_$DATE.info" << EOF
Backup Date: $DATE
Backup Type: Base Backup
Source: Primary Server
WAL Position: $(psql -h primary.example.com -U backup_user -t -c "SELECT pg_current_wal_lsn();")
Database Size: $(du -sh $BACKUP_DIR/base_$DATE)
EOF
        
    else
        echo "Base backup failed"
        exit 1
    fi
}

# Function to backup WAL files
backup_wal() {
    echo "Archiving WAL files..."
    
    # Sync WAL archive to S3
    aws s3 sync /var/lib/postgresql/wal_archive/ \
                "s3://$S3_BUCKET/wal_archive/" \
                --delete
    
    # Clean up old local WAL files
    find /var/lib/postgresql/wal_archive/ -type f -mtime +7 -delete
}

# Function to test backup integrity
test_backup() {
    local backup_file="$1"
    echo "Testing backup integrity: $backup_file"
    
    # Extract and verify
    mkdir -p "/tmp/backup_test_$DATE"
    tar -xzf "$backup_file" -C "/tmp/backup_test_$DATE"
    
    # Start test instance
    pg_ctl -D "/tmp/backup_test_$DATE" -l "/tmp/backup_test_$DATE/test.log" start
    
    if [ $? -eq 0 ]; then
        echo "Backup test successful"
        pg_ctl -D "/tmp/backup_test_$DATE" stop
    else
        echo "Backup test failed"
    fi
    
    # Cleanup
    rm -rf "/tmp/backup_test_$DATE"
}

# Function to cleanup old backups
cleanup_backups() {
    echo "Cleaning up old backups..."
    
    # Remove local backups older than retention period
    find "$BACKUP_DIR" -name "base_*" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR" -name "backup_*.info" -mtime +$RETENTION_DAYS -delete
    
    # Remove old S3 backups
    aws s3 ls "s3://$S3_BUCKET/base_backups/" | \
    while read -r line; do
        backup_date=$(echo $line | awk '{print $1" "$2}')
        backup_file=$(echo $line | awk '{print $4}')
        
        if [[ $(date -d "$backup_date" +%s) -lt $(date -d "$RETENTION_DAYS days ago" +%s) ]]; then
            aws s3 rm "s3://$S3_BUCKET/base_backups/$backup_file"
        fi
    done
}

# Main execution
echo "Starting HA backup process at $(date)"

backup_primary
backup_wal
test_backup "$BACKUP_DIR/base_$DATE.tar.gz"
cleanup_backups

echo "HA backup process completed at $(date)"
```

### Point-in-Time Recovery for HA

```bash
#!/bin/bash
# ha_pitr_recovery.sh

RECOVERY_TARGET_TIME="$1"
BACKUP_DIR="/backup/postgresql"
RECOVERY_DIR="/var/lib/postgresql/recovery"
S3_BUCKET="my-postgres-backups"

if [ -z "$RECOVERY_TARGET_TIME" ]; then
    echo "Usage: $0 'YYYY-MM-DD HH:MM:SS'"
    exit 1
fi

echo "Starting Point-in-Time Recovery to: $RECOVERY_TARGET_TIME"

# Find appropriate base backup
find_base_backup() {
    echo "Finding appropriate base backup..."
    
    # List available backups from S3
    aws s3 ls "s3://$S3_BUCKET/base_backups/" | \
    sort -r | \
    while read -r line; do
        backup_file=$(echo $line | awk '{print $4}')
        backup_date=$(echo $backup_file | sed 's/base_\(.*\)\.tar\.gz/\1/')
        
        # Convert backup date to timestamp
        backup_timestamp=$(date -d "${backup_date:0:8} ${backup_date:9:2}:${backup_date:11:2}:${backup_date:13:2}" +%s)
        target_timestamp=$(date -d "$RECOVERY_TARGET_TIME" +%s)
        
        if [ $backup_timestamp -le $target_timestamp ]; then
            echo "Selected backup: $backup_file"
            echo $backup_file
            return 0
        fi
    done
}

# Download and restore base backup
restore_base_backup() {
    local backup_file="$1"
    
    echo "Downloading base backup: $backup_file"
    aws s3 cp "s3://$S3_BUCKET/base_backups/$backup_file" "$BACKUP_DIR/"
    
    echo "Extracting base backup..."
    mkdir -p "$RECOVERY_DIR"
    tar -xzf "$BACKUP_DIR/$backup_file" -C "$RECOVERY_DIR"
    
    # Set permissions
    chown -R postgres:postgres "$RECOVERY_DIR"
    chmod 700 "$RECOVERY_DIR"
}

# Setup recovery configuration
setup_recovery() {
    echo "Setting up recovery configuration..."
    
    # Download WAL files from S3
    mkdir -p "$RECOVERY_DIR/wal_archive"
    aws s3 sync "s3://$S3_BUCKET/wal_archive/" "$RECOVERY_DIR/wal_archive/"
    
    # Create recovery configuration
    cat > "$RECOVERY_DIR/postgresql.auto.conf" << EOF
restore_command = 'cp $RECOVERY_DIR/wal_archive/%f %p'
recovery_target_time = '$RECOVERY_TARGET_TIME'
recovery_target_action = 'promote'
EOF
    
    # Create recovery signal
    touch "$RECOVERY_DIR/recovery.signal"
}

# Start recovery process
start_recovery() {
    echo "Starting recovery process..."
    
    # Start PostgreSQL in recovery mode
    sudo -u postgres pg_ctl -D "$RECOVERY_DIR" \
                            -l "$RECOVERY_DIR/recovery.log" \
                            start
    
    # Monitor recovery progress
    echo "Monitoring recovery progress..."
    while [ -f "$RECOVERY_DIR/recovery.signal" ]; do
        echo "Recovery in progress..."
        sleep 10
    done
    
    echo "Recovery completed successfully"
    echo "Database is now available at: $RECOVERY_DIR"
}

# Main execution
BASE_BACKUP=$(find_base_backup)
if [ -z "$BASE_BACKUP" ]; then
    echo "No suitable base backup found"
    exit 1
fi

restore_base_backup "$BASE_BACKUP"
setup_recovery
start_recovery

echo "Point-in-Time Recovery completed"
```

## Monitoring and Alerting

### HA Monitoring Dashboard

```sql
-- Create HA monitoring views
CREATE VIEW ha_cluster_status AS
WITH replication_status AS (
    SELECT 
        CASE 
            WHEN pg_is_in_recovery() THEN 'STANDBY'
            ELSE 'PRIMARY'
        END as node_role,
        CASE 
            WHEN pg_is_in_recovery() THEN
                EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp()))
            ELSE 0
        END as replication_lag_seconds,
        pg_is_in_recovery() as is_standby
),
connection_status AS (
    SELECT 
        COUNT(*) as total_connections,
        COUNT(*) FILTER (WHERE state = 'active') as active_connections,
        COUNT(*) FILTER (WHERE state = 'idle in transaction') as idle_in_transaction
    FROM pg_stat_activity
    WHERE pid <> pg_backend_pid()
),
wal_status AS (
    SELECT 
        pg_current_wal_lsn() as current_wal_lsn,
        pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 as wal_mb_generated
)
SELECT 
    rs.node_role,
    rs.replication_lag_seconds,
    cs.total_connections,
    cs.active_connections,
    cs.idle_in_transaction,
    ws.current_wal_lsn,
    ws.wal_mb_generated,
    NOW() as status_time
FROM replication_status rs, connection_status cs, wal_status ws;

-- HA health check function
CREATE OR REPLACE FUNCTION ha_health_check()
RETURNS TABLE(
    component TEXT,
    status TEXT,
    message TEXT,
    last_check TIMESTAMP
)
AS $$
BEGIN
    -- Check if this is primary or standby
    IF pg_is_in_recovery() THEN
        -- Standby checks
        RETURN QUERY
        SELECT 
            'Replication Lag'::TEXT,
            CASE 
                WHEN EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) > 300 THEN 'CRITICAL'
                WHEN EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) > 60 THEN 'WARNING'
                ELSE 'OK'
            END,
            'Lag: ' || ROUND(EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())), 2) || ' seconds',
            NOW();
        
        RETURN QUERY
        SELECT 
            'WAL Receiver'::TEXT,
            CASE 
                WHEN pg_last_wal_receive_lsn() IS NULL THEN 'CRITICAL'
                ELSE 'OK'
            END,
            'Last received: ' || COALESCE(pg_last_wal_receive_lsn()::TEXT, 'None'),
            NOW();
    ELSE
        -- Primary checks
        RETURN QUERY
        SELECT 
            'Replication Slots'::TEXT,
            CASE 
                WHEN COUNT(*) = 0 THEN 'WARNING'
                WHEN COUNT(*) FILTER (WHERE active = false) > 0 THEN 'WARNING'
                ELSE 'OK'
            END,
            'Active slots: ' || COUNT(*) FILTER (WHERE active = true)::TEXT || 
            ', Inactive slots: ' || COUNT(*) FILTER (WHERE active = false)::TEXT,
            NOW()
        FROM pg_replication_slots;
        
        RETURN QUERY
        SELECT 
            'WAL Senders'::TEXT,
            CASE 
                WHEN COUNT(*) = 0 THEN 'WARNING'
                WHEN COUNT(*) FILTER (WHERE state != 'streaming') > 0 THEN 'WARNING'
                ELSE 'OK'
            END,
            'Streaming: ' || COUNT(*) FILTER (WHERE state = 'streaming')::TEXT ||
            ', Other: ' || COUNT(*) FILTER (WHERE state != 'streaming')::TEXT,
            NOW()
        FROM pg_stat_replication;
    END IF;
    
    -- Common checks
    RETURN QUERY
    SELECT 
        'Database Connections'::TEXT,
        CASE 
            WHEN COUNT(*) > 80 THEN 'WARNING'
            WHEN COUNT(*) > 50 THEN 'CAUTION'
            ELSE 'OK'
        END,
        'Active connections: ' || COUNT(*)::TEXT,
        NOW()
    FROM pg_stat_activity
    WHERE state = 'active';
END;
$$ LANGUAGE plpgsql;

-- Run health check
SELECT * FROM ha_health_check();
```

### Automated Failover Testing

```bash
#!/bin/bash
# ha_failover_test.sh

PRIMARY_HOST="192.168.1.10"
STANDBY_HOST="192.168.1.11"
TEST_DB="test_failover"
TEST_USER="test_user"
LOG_FILE="/var/log/postgresql/failover_test.log"

log() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

# Test database connectivity
test_connectivity() {
    local host="$1"
    local role="$2"
    
    log "Testing connectivity to $role ($host)..."
    
    if psql -h "$host" -U "$TEST_USER" -d "$TEST_DB" -c "SELECT 1;" >/dev/null 2>&1; then
        log "✓ Connection to $role successful"
        return 0
    else
        log "✗ Connection to $role failed"
        return 1
    fi
}

# Test write operations
test_write() {
    local host="$1"
    local test_id=$(date +%s)
    
    log "Testing write operations on $host..."
    
    if psql -h "$host" -U "$TEST_USER" -d "$TEST_DB" -c \
        "INSERT INTO failover_test (test_id, test_time) VALUES ($test_id, NOW());" >/dev/null 2>&1; then
        log "✓ Write operation successful (test_id: $test_id)"
        return 0
    else
        log "✗ Write operation failed"
        return 1
    fi
}

# Test read operations
test_read() {
    local host="$1"
    
    log "Testing read operations on $host..."
    
    local count=$(psql -h "$host" -U "$TEST_USER" -d "$TEST_DB" -t -c \
        "SELECT COUNT(*) FROM failover_test;" 2>/dev/null | tr -d ' ')
    
    if [ -n "$count" ] && [ "$count" -gt 0 ]; then
        log "✓ Read operation successful (records: $count)"
        return 0
    else
        log "✗ Read operation failed"
        return 1
    fi
}

# Simulate failover
simulate_failover() {
    log "Simulating failover scenario..."
    
    # Stop primary database
    log "Stopping primary database..."
    ssh "$PRIMARY_HOST" "sudo systemctl stop postgresql"
    
    # Wait for failover detection
    log "Waiting for failover detection..."
    sleep 30
    
    # Test if standby became primary
    if test_write "$STANDBY_HOST"; then
        log "✓ Failover successful - standby is now accepting writes"
    else
        log "✗ Failover failed - standby is not accepting writes"
        return 1
    fi
}

# Restore original setup
restore_setup() {
    log "Restoring original setup..."
    
    # Start original primary as standby
    ssh "$PRIMARY_HOST" "sudo systemctl start postgresql"
    
    # Wait for synchronization
    sleep 60
    
    # Perform switchback if needed
    log "Setup restored"
}

# Main test execution
log "Starting HA failover test"

# Initial connectivity tests
test_connectivity "$PRIMARY_HOST" "primary" || exit 1
test_connectivity "$STANDBY_HOST" "standby" || exit 1

# Test normal operations
test_write "$PRIMARY_HOST" || exit 1
test_read "$STANDBY_HOST" || exit 1

# Simulate and test failover
simulate_failover || exit 1

# Test operations after failover
test_write "$STANDBY_HOST" || exit 1
test_read "$STANDBY_HOST" || exit 1

# Restore setup
restore_setup

log "HA failover test completed successfully"
```

## Best Practices

### HA Design Principles

1. **Eliminate Single Points of Failure**
   - Redundant hardware and network paths
   - Multiple data centers or availability zones
   - Load balancer redundancy

2. **Automate Failover Processes**
   - Use proven tools like Patroni or Pacemaker
   - Implement health checks and monitoring
   - Test failover procedures regularly

3. **Monitor Everything**
   - Replication lag and status
   - System resources and performance
   - Application connectivity and response times

4. **Plan for Different Failure Scenarios**
   - Hardware failures
   - Network partitions
   - Data corruption
   - Human errors

### Performance Considerations

1. **Synchronous vs Asynchronous Replication**
   - Synchronous: Zero data loss, higher latency
   - Asynchronous: Potential data loss, lower latency
   - Choose based on RPO requirements

2. **Network Optimization**
   - Dedicated replication network
   - Optimize network latency and bandwidth
   - Use compression for WAN replication

3. **Resource Planning**
   - Size standby servers appropriately
   - Plan for read workload distribution
   - Consider connection pooling overhead

### Security in HA Environments

1. **Secure Replication Connections**
   - Use SSL/TLS for replication
   - Implement certificate-based authentication
   - Restrict network access

2. **Access Control**
   - Separate replication and application users
   - Implement role-based access control
   - Regular security audits

3. **Data Protection**
   - Encrypt data at rest and in transit
   - Secure backup storage
   - Implement audit logging

## Next Steps

After mastering high availability:
1. PostgreSQL in Production (16-production-deployment.md)
2. Advanced Topics and Extensions (17-advanced-topics.md)
3. PostgreSQL Ecosystem and Tools (18-ecosystem-tools.md)

---
*This is part 15 of the PostgreSQL learning series*