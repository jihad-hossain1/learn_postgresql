# PostgreSQL Production Deployment

## Introduction

Deploying PostgreSQL in production requires careful planning, proper configuration, security hardening, monitoring setup, and operational procedures. This guide covers best practices for production deployments, from initial setup to ongoing maintenance.

## Pre-Deployment Planning

### Requirements Assessment

```sql
-- Create requirements assessment template
CREATE TABLE deployment_requirements (
    requirement_id SERIAL PRIMARY KEY,
    category VARCHAR(50),
    requirement TEXT,
    current_value TEXT,
    target_value TEXT,
    priority VARCHAR(10),
    status VARCHAR(20),
    notes TEXT
);

INSERT INTO deployment_requirements (category, requirement, target_value, priority) VALUES
('Performance', 'Peak concurrent connections', '500', 'HIGH'),
('Performance', 'Average query response time', '< 100ms', 'HIGH'),
('Performance', 'Database size (current)', '100GB', 'MEDIUM'),
('Performance', 'Database growth rate', '10GB/month', 'MEDIUM'),
('Availability', 'Uptime requirement', '99.9%', 'HIGH'),
('Availability', 'RTO (Recovery Time Objective)', '< 5 minutes', 'HIGH'),
('Availability', 'RPO (Recovery Point Objective)', '< 1 minute', 'HIGH'),
('Security', 'Data encryption at rest', 'Required', 'HIGH'),
('Security', 'Network encryption', 'Required', 'HIGH'),
('Security', 'Access control', 'Role-based', 'HIGH'),
('Backup', 'Backup frequency', 'Daily full, hourly incremental', 'HIGH'),
('Backup', 'Backup retention', '30 days', 'MEDIUM'),
('Monitoring', 'Performance monitoring', '24/7', 'HIGH'),
('Monitoring', 'Alerting', 'Real-time', 'HIGH');

-- Requirements assessment query
SELECT 
    category,
    COUNT(*) as total_requirements,
    COUNT(*) FILTER (WHERE priority = 'HIGH') as high_priority,
    COUNT(*) FILTER (WHERE status = 'COMPLETED') as completed,
    ROUND(COUNT(*) FILTER (WHERE status = 'COMPLETED') * 100.0 / COUNT(*), 1) as completion_percentage
FROM deployment_requirements
GROUP BY category
ORDER BY high_priority DESC;
```

### Capacity Planning

```sql
-- Capacity planning calculations
CREATE OR REPLACE FUNCTION calculate_capacity_requirements(
    peak_connections INTEGER,
    avg_query_time_ms NUMERIC,
    db_size_gb NUMERIC,
    growth_rate_gb_month NUMERIC,
    planning_horizon_months INTEGER DEFAULT 12
)
RETURNS TABLE(
    component TEXT,
    current_requirement TEXT,
    projected_requirement TEXT,
    recommended_spec TEXT
)
AS $$
BEGIN
    -- CPU calculation (rough estimate)
    RETURN QUERY
    SELECT 
        'CPU Cores'::TEXT,
        CEIL(peak_connections / 50.0)::TEXT || ' cores',
        CEIL(peak_connections * 1.5 / 50.0)::TEXT || ' cores',
        'Intel Xeon or AMD EPYC, 2.5GHz+'::TEXT;
    
    -- Memory calculation
    RETURN QUERY
    SELECT 
        'Memory (RAM)'::TEXT,
        CEIL(GREATEST(db_size_gb * 0.25, 8))::TEXT || ' GB',
        CEIL(GREATEST((db_size_gb + growth_rate_gb_month * planning_horizon_months) * 0.25, 16))::TEXT || ' GB',
        'DDR4-3200 or faster, ECC recommended'::TEXT;
    
    -- Storage calculation
    RETURN QUERY
    SELECT 
        'Storage (Data)'::TEXT,
        CEIL(db_size_gb * 1.5)::TEXT || ' GB',
        CEIL((db_size_gb + growth_rate_gb_month * planning_horizon_months) * 1.5)::TEXT || ' GB',
        'NVMe SSD, 10K+ IOPS'::TEXT;
    
    -- WAL storage
    RETURN QUERY
    SELECT 
        'Storage (WAL)'::TEXT,
        CEIL(db_size_gb * 0.1)::TEXT || ' GB',
        CEIL((db_size_gb + growth_rate_gb_month * planning_horizon_months) * 0.1)::TEXT || ' GB',
        'Separate fast SSD, 15K+ IOPS'::TEXT;
    
    -- Network
    RETURN QUERY
    SELECT 
        'Network'::TEXT,
        '1 Gbps'::TEXT,
        CASE 
            WHEN peak_connections > 200 THEN '10 Gbps'
            ELSE '1 Gbps'
        END,
        'Redundant NICs, low latency'::TEXT;
END;
$$ LANGUAGE plpgsql;

-- Example capacity calculation
SELECT * FROM calculate_capacity_requirements(
    peak_connections := 500,
    avg_query_time_ms := 50,
    db_size_gb := 100,
    growth_rate_gb_month := 10,
    planning_horizon_months := 24
);
```

## Infrastructure Setup

### Server Configuration

```bash
#!/bin/bash
# production_server_setup.sh

set -e

echo "Starting PostgreSQL production server setup..."

# System updates
sudo apt-get update && sudo apt-get upgrade -y

# Install required packages
sudo apt-get install -y \
    postgresql-15 \
    postgresql-contrib-15 \
    postgresql-15-pg-stat-statements \
    postgresql-15-pgaudit \
    htop \
    iotop \
    sysstat \
    chrony \
    fail2ban \
    ufw

# Configure system limits
sudo cat >> /etc/security/limits.conf << EOF
postgres soft nofile 65536
postgres hard nofile 65536
postgres soft nproc 32768
postgres hard nproc 32768
EOF

# Configure kernel parameters
sudo cat >> /etc/sysctl.conf << EOF
# PostgreSQL optimizations
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
fs.file-max = 2097152
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.core.rmem_default = 262144
net.core.rmem_max = 16777216
net.core.wmem_default = 262144
net.core.wmem_max = 16777216
vm.swappiness = 1
vm.overcommit_memory = 2
vm.overcommit_ratio = 80
EOF

sudo sysctl -p

# Configure storage
# Create separate mount points for data, WAL, and logs
sudo mkdir -p /data/postgresql/{data,wal,logs,backups}
sudo chown -R postgres:postgres /data/postgresql
sudo chmod 700 /data/postgresql/data

# Configure fstab for optimal mount options
echo "# PostgreSQL data partition" | sudo tee -a /etc/fstab
echo "/dev/sdb1 /data/postgresql/data ext4 noatime,nodiratime,nodev,noexec,nosuid 0 2" | sudo tee -a /etc/fstab
echo "/dev/sdc1 /data/postgresql/wal ext4 noatime,nodiratime,nodev,noexec,nosuid 0 2" | sudo tee -a /etc/fstab

# Configure time synchronization
sudo systemctl enable chrony
sudo systemctl start chrony

# Configure firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 5432/tcp
sudo ufw --force enable

# Configure fail2ban for PostgreSQL
sudo cat > /etc/fail2ban/jail.local << EOF
[postgresql]
enabled = true
port = 5432
filter = postgresql
logpath = /data/postgresql/logs/postgresql-*.log
maxretry = 5
bantime = 3600
EOF

sudo systemctl enable fail2ban
sudo systemctl start fail2ban

echo "Server setup completed"
```

### PostgreSQL Installation and Configuration

```bash
#!/bin/bash
# postgresql_production_config.sh

PG_VERSION="15"
DATA_DIR="/data/postgresql/data"
WAL_DIR="/data/postgresql/wal"
LOG_DIR="/data/postgresql/logs"
CONFIG_DIR="/etc/postgresql/$PG_VERSION/main"

# Stop PostgreSQL
sudo systemctl stop postgresql

# Initialize new data directory
sudo -u postgres /usr/lib/postgresql/$PG_VERSION/bin/initdb \
    -D "$DATA_DIR" \
    --encoding=UTF8 \
    --locale=en_US.UTF8 \
    --data-checksums \
    --auth-local=peer \
    --auth-host=md5

# Create WAL directory
sudo mkdir -p "$WAL_DIR"
sudo chown postgres:postgres "$WAL_DIR"
sudo chmod 700 "$WAL_DIR"

# Create log directory
sudo mkdir -p "$LOG_DIR"
sudo chown postgres:postgres "$LOG_DIR"

# Configure PostgreSQL
sudo cat > "$DATA_DIR/postgresql.conf" << EOF
# PostgreSQL Production Configuration

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 500
superuser_reserved_connections = 5

# Memory Settings
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 32MB
maintenance_work_mem = 1GB
max_stack_depth = 7MB
dynamic_shared_memory_type = posix

# WAL Settings
wal_level = replica
wal_buffers = 64MB
max_wal_size = 4GB
min_wal_size = 1GB
wal_compression = on
wal_log_hints = on
archive_mode = on
archive_command = 'cp %p $WAL_DIR/archive/%f'
archive_timeout = 300

# Replication Settings
max_wal_senders = 10
max_replication_slots = 10
wal_keep_size = 2GB
hot_standby = on
hot_standby_feedback = on
wal_receiver_status_interval = 10s
wal_receiver_timeout = 60s
wal_sender_timeout = 60s

# Query Planner Settings
random_page_cost = 1.1
effective_io_concurrency = 200
max_worker_processes = 16
max_parallel_workers_per_gather = 4
max_parallel_workers = 16
max_parallel_maintenance_workers = 4

# Checkpoints
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
checkpoint_warning = 30s

# Logging
logging_collector = on
log_directory = '$LOG_DIR'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0640
log_rotation_age = 1d
log_rotation_size = 100MB
log_truncate_on_rotation = on
log_min_duration_statement = 1000
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
log_error_verbosity = default
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'ddl'

# Autovacuum
autovacuum = on
autovacuum_max_workers = 6
autovacuum_naptime = 15s
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.1
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 10ms
autovacuum_vacuum_cost_limit = 1000

# Statistics
track_activities = on
track_counts = on
track_io_timing = on
track_functions = all
stats_temp_directory = '/tmp/pg_stat_tmp'

# Extensions
shared_preload_libraries = 'pg_stat_statements,pgaudit'

# Security
ssl = on
ssl_cert_file = '/etc/ssl/certs/postgresql.crt'
ssl_key_file = '/etc/ssl/private/postgresql.key'
ssl_ca_file = '/etc/ssl/certs/ca.crt'
ssl_crl_file = ''
password_encryption = scram-sha-256

# Locale
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'

# Extension Settings
pg_stat_statements.max = 10000
pg_stat_statements.track = all
pg_stat_statements.save = on

pgaudit.log = 'ddl,role,misc'
pgaudit.log_catalog = off
pgaudit.log_parameter = on
pgaudit.log_relation = on
pgaudit.log_statement_once = on
EOF

# Configure pg_hba.conf
sudo cat > "$DATA_DIR/pg_hba.conf" << EOF
# PostgreSQL Client Authentication Configuration

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     peer

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256

# Application connections
host    all             app_user        10.0.0.0/8              scram-sha-256
host    all             app_user        192.168.0.0/16          scram-sha-256

# Replication connections
host    replication     replicator      10.0.0.0/8              scram-sha-256
host    replication     replicator      192.168.0.0/16          scram-sha-256

# Monitoring connections
host    all             monitor_user    10.0.0.0/8              scram-sha-256

# Backup connections
host    all             backup_user     10.0.0.0/8              scram-sha-256

# SSL required for external connections
hostssl all             all             0.0.0.0/0               scram-sha-256
EOF

# Update systemd service
sudo cat > /etc/systemd/system/postgresql.service << EOF
[Unit]
Description=PostgreSQL database server
Documentation=man:postgres(1)
After=network.target
Wants=network.target

[Service]
Type=notify
User=postgres
ExecStart=/usr/lib/postgresql/$PG_VERSION/bin/postgres -D $DATA_DIR
ExecReload=/bin/kill -HUP \$MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=0

# Restart settings
Restart=on-failure
RestartSec=5s

# Security settings
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=$DATA_DIR $WAL_DIR $LOG_DIR /tmp

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable postgresql

echo "PostgreSQL configuration completed"
```

## Security Hardening

### SSL/TLS Configuration

```bash
#!/bin/bash
# ssl_setup.sh

SSL_DIR="/etc/ssl/postgresql"
CERT_VALIDITY_DAYS=3650

# Create SSL directory
sudo mkdir -p "$SSL_DIR"
sudo chown postgres:postgres "$SSL_DIR"
sudo chmod 700 "$SSL_DIR"

# Generate CA private key
sudo -u postgres openssl genrsa -out "$SSL_DIR/ca-key.pem" 4096

# Generate CA certificate
sudo -u postgres openssl req -new -x509 -days $CERT_VALIDITY_DAYS \
    -key "$SSL_DIR/ca-key.pem" \
    -out "$SSL_DIR/ca-cert.pem" \
    -subj "/C=US/ST=State/L=City/O=Organization/OU=Database/CN=PostgreSQL-CA"

# Generate server private key
sudo -u postgres openssl genrsa -out "$SSL_DIR/server-key.pem" 4096

# Generate server certificate request
sudo -u postgres openssl req -new \
    -key "$SSL_DIR/server-key.pem" \
    -out "$SSL_DIR/server-req.pem" \
    -subj "/C=US/ST=State/L=City/O=Organization/OU=Database/CN=postgresql.example.com"

# Generate server certificate
sudo -u postgres openssl x509 -req -days $CERT_VALIDITY_DAYS \
    -in "$SSL_DIR/server-req.pem" \
    -CA "$SSL_DIR/ca-cert.pem" \
    -CAkey "$SSL_DIR/ca-key.pem" \
    -CAcreateserial \
    -out "$SSL_DIR/server-cert.pem"

# Set proper permissions
sudo chmod 600 "$SSL_DIR/server-key.pem"
sudo chmod 644 "$SSL_DIR/server-cert.pem"
sudo chmod 644 "$SSL_DIR/ca-cert.pem"

# Create symbolic links for PostgreSQL
sudo ln -sf "$SSL_DIR/server-cert.pem" /etc/ssl/certs/postgresql.crt
sudo ln -sf "$SSL_DIR/server-key.pem" /etc/ssl/private/postgresql.key
sudo ln -sf "$SSL_DIR/ca-cert.pem" /etc/ssl/certs/ca.crt

echo "SSL certificates generated successfully"
echo "CA Certificate: $SSL_DIR/ca-cert.pem"
echo "Server Certificate: $SSL_DIR/server-cert.pem"
echo "Server Key: $SSL_DIR/server-key.pem"
```

### User and Role Management

```sql
-- Production user and role setup

-- Create roles for different access levels
CREATE ROLE readonly_role;
CREATE ROLE readwrite_role;
CREATE ROLE admin_role;
CREATE ROLE backup_role;
CREATE ROLE monitor_role;
CREATE ROLE replication_role REPLICATION;

-- Grant permissions to roles
-- Readonly role
GRANT CONNECT ON DATABASE production_db TO readonly_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly_role;

-- Readwrite role
GRANT readonly_role TO readwrite_role;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readwrite_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT INSERT, UPDATE, DELETE ON TABLES TO readwrite_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO readwrite_role;

-- Admin role
GRANT readwrite_role TO admin_role;
GRANT CREATE ON SCHEMA public TO admin_role;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO admin_role;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO admin_role;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO admin_role;

-- Backup role
GRANT CONNECT ON DATABASE production_db TO backup_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO backup_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO backup_role;

-- Monitor role
GRANT CONNECT ON DATABASE production_db TO monitor_role;
GRANT SELECT ON pg_stat_database TO monitor_role;
GRANT SELECT ON pg_stat_user_tables TO monitor_role;
GRANT SELECT ON pg_stat_user_indexes TO monitor_role;
GRANT SELECT ON pg_stat_activity TO monitor_role;
GRANT SELECT ON pg_stat_replication TO monitor_role;
GRANT EXECUTE ON FUNCTION pg_stat_file(text) TO monitor_role;
GRANT EXECUTE ON FUNCTION pg_ls_dir(text) TO monitor_role;

-- Create application users
CREATE USER app_user WITH PASSWORD 'secure_app_password';
GRANT readwrite_role TO app_user;

CREATE USER app_readonly WITH PASSWORD 'secure_readonly_password';
GRANT readonly_role TO app_readonly;

CREATE USER backup_user WITH PASSWORD 'secure_backup_password';
GRANT backup_role TO backup_user;

CREATE USER monitor_user WITH PASSWORD 'secure_monitor_password';
GRANT monitor_role TO monitor_user;

CREATE USER replicator WITH PASSWORD 'secure_replication_password';
GRANT replication_role TO replicator;

-- Create admin user
CREATE USER db_admin WITH PASSWORD 'secure_admin_password' CREATEDB CREATEROLE;
GRANT admin_role TO db_admin;

-- Set password policies
ALTER ROLE app_user VALID UNTIL '2025-12-31';
ALTER ROLE app_readonly VALID UNTIL '2025-12-31';
ALTER ROLE backup_user VALID UNTIL '2025-12-31';
ALTER ROLE monitor_user VALID UNTIL '2025-12-31';
ALTER ROLE replicator VALID UNTIL '2025-12-31';

-- Create password rotation function
CREATE OR REPLACE FUNCTION rotate_user_passwords()
RETURNS TABLE(username TEXT, new_password TEXT)
AS $$
DECLARE
    user_record RECORD;
    new_pwd TEXT;
BEGIN
    FOR user_record IN 
        SELECT rolname FROM pg_roles 
        WHERE rolname IN ('app_user', 'app_readonly', 'backup_user', 'monitor_user')
    LOOP
        -- Generate new password (in production, use a secure password generator)
        new_pwd := 'new_' || user_record.rolname || '_' || extract(epoch from now())::text;
        
        -- Update password
        EXECUTE format('ALTER USER %I WITH PASSWORD %L', user_record.rolname, new_pwd);
        
        -- Return username and new password
        username := user_record.rolname;
        new_password := new_pwd;
        RETURN NEXT;
    END LOOP;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Create audit table for user activities
CREATE TABLE user_audit_log (
    log_id SERIAL PRIMARY KEY,
    username TEXT,
    action TEXT,
    object_name TEXT,
    timestamp TIMESTAMP DEFAULT NOW(),
    client_addr INET,
    application_name TEXT
);

-- Create audit trigger function
CREATE OR REPLACE FUNCTION audit_user_activity()
RETURNS event_trigger
AS $$
BEGIN
    INSERT INTO user_audit_log (username, action, object_name, client_addr, application_name)
    VALUES (
        current_user,
        tg_tag,
        coalesce(tg_object_identity, 'unknown'),
        inet_client_addr(),
        current_setting('application_name', true)
    );
END;
$$ LANGUAGE plpgsql;

-- Create event trigger for DDL auditing
CREATE EVENT TRIGGER audit_ddl_commands
    ON ddl_command_end
    EXECUTE FUNCTION audit_user_activity();
```

## Monitoring and Alerting

### Comprehensive Monitoring Setup

```sql
-- Create monitoring schema
CREATE SCHEMA monitoring;

-- Database health monitoring
CREATE OR REPLACE FUNCTION monitoring.database_health_check()
RETURNS TABLE(
    metric_name TEXT,
    current_value NUMERIC,
    threshold_warning NUMERIC,
    threshold_critical NUMERIC,
    status TEXT,
    message TEXT
)
AS $$
BEGIN
    -- Connection count
    RETURN QUERY
    SELECT 
        'active_connections'::TEXT,
        COUNT(*)::NUMERIC,
        400::NUMERIC,
        450::NUMERIC,
        CASE 
            WHEN COUNT(*) >= 450 THEN 'CRITICAL'
            WHEN COUNT(*) >= 400 THEN 'WARNING'
            ELSE 'OK'
        END,
        'Active connections: ' || COUNT(*)
    FROM pg_stat_activity
    WHERE state = 'active';
    
    -- Database size
    RETURN QUERY
    SELECT 
        'database_size_gb'::TEXT,
        ROUND(pg_database_size(current_database()) / 1024.0^3, 2),
        100::NUMERIC,
        150::NUMERIC,
        CASE 
            WHEN pg_database_size(current_database()) / 1024.0^3 >= 150 THEN 'CRITICAL'
            WHEN pg_database_size(current_database()) / 1024.0^3 >= 100 THEN 'WARNING'
            ELSE 'OK'
        END,
        'Database size: ' || ROUND(pg_database_size(current_database()) / 1024.0^3, 2) || ' GB';
    
    -- Replication lag (if standby)
    IF pg_is_in_recovery() THEN
        RETURN QUERY
        SELECT 
            'replication_lag_seconds'::TEXT,
            EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())),
            60::NUMERIC,
            300::NUMERIC,
            CASE 
                WHEN EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) >= 300 THEN 'CRITICAL'
                WHEN EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) >= 60 THEN 'WARNING'
                ELSE 'OK'
            END,
            'Replication lag: ' || ROUND(EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())), 2) || ' seconds';
    END IF;
    
    -- Long running queries
    RETURN QUERY
    SELECT 
        'long_running_queries'::TEXT,
        COUNT(*)::NUMERIC,
        5::NUMERIC,
        10::NUMERIC,
        CASE 
            WHEN COUNT(*) >= 10 THEN 'CRITICAL'
            WHEN COUNT(*) >= 5 THEN 'WARNING'
            ELSE 'OK'
        END,
        'Long running queries (>5min): ' || COUNT(*)
    FROM pg_stat_activity
    WHERE state = 'active'
      AND query_start < NOW() - INTERVAL '5 minutes'
      AND pid <> pg_backend_pid();
    
    -- Blocked queries
    RETURN QUERY
    SELECT 
        'blocked_queries'::TEXT,
        COUNT(*)::NUMERIC,
        1::NUMERIC,
        5::NUMERIC,
        CASE 
            WHEN COUNT(*) >= 5 THEN 'CRITICAL'
            WHEN COUNT(*) >= 1 THEN 'WARNING'
            ELSE 'OK'
        END,
        'Blocked queries: ' || COUNT(*)
    FROM pg_stat_activity
    WHERE wait_event_type = 'Lock'
      AND state = 'active';
    
    -- Disk space (approximate)
    RETURN QUERY
    SELECT 
        'disk_usage_percent'::TEXT,
        85::NUMERIC, -- This would need to be calculated from actual disk usage
        80::NUMERIC,
        90::NUMERIC,
        CASE 
            WHEN 85 >= 90 THEN 'CRITICAL'
            WHEN 85 >= 80 THEN 'WARNING'
            ELSE 'OK'
        END,
        'Disk usage: 85%' -- This would be dynamic in real implementation
    ;
END;
$$ LANGUAGE plpgsql;

-- Performance monitoring
CREATE OR REPLACE FUNCTION monitoring.performance_metrics()
RETURNS TABLE(
    metric_name TEXT,
    value NUMERIC,
    unit TEXT,
    description TEXT
)
AS $$
BEGIN
    -- Query performance
    RETURN QUERY
    SELECT 
        'avg_query_time_ms'::TEXT,
        ROUND(AVG(mean_exec_time), 2),
        'milliseconds'::TEXT,
        'Average query execution time'
    FROM pg_stat_statements
    WHERE calls > 10;
    
    -- Cache hit ratio
    RETURN QUERY
    SELECT 
        'cache_hit_ratio'::TEXT,
        ROUND(
            100.0 * SUM(blks_hit) / NULLIF(SUM(blks_hit + blks_read), 0), 2
        ),
        'percent'::TEXT,
        'Buffer cache hit ratio'
    FROM pg_stat_database;
    
    -- Index usage
    RETURN QUERY
    SELECT 
        'index_usage_ratio'::TEXT,
        ROUND(
            100.0 * SUM(idx_scan) / NULLIF(SUM(seq_scan + idx_scan), 0), 2
        ),
        'percent'::TEXT,
        'Index usage ratio'
    FROM pg_stat_user_tables;
    
    -- Transactions per second
    RETURN QUERY
    SELECT 
        'transactions_per_second'::TEXT,
        ROUND(
            (SUM(xact_commit + xact_rollback))::NUMERIC / 
            EXTRACT(EPOCH FROM (NOW() - stats_reset)), 2
        ),
        'tps'::TEXT,
        'Transactions per second'
    FROM pg_stat_database
    WHERE datname = current_database();
    
    -- Checkpoint frequency
    RETURN QUERY
    SELECT 
        'checkpoints_per_hour'::TEXT,
        ROUND(
            (checkpoints_timed + checkpoints_req)::NUMERIC * 3600 / 
            EXTRACT(EPOCH FROM (NOW() - stats_reset)), 2
        ),
        'per_hour'::TEXT,
        'Checkpoint frequency'
    FROM pg_stat_bgwriter;
END;
$$ LANGUAGE plpgsql;

-- Create monitoring tables
CREATE TABLE monitoring.health_check_history (
    check_id SERIAL PRIMARY KEY,
    check_time TIMESTAMP DEFAULT NOW(),
    metric_name TEXT,
    current_value NUMERIC,
    status TEXT,
    message TEXT
);

CREATE TABLE monitoring.performance_history (
    metric_id SERIAL PRIMARY KEY,
    metric_time TIMESTAMP DEFAULT NOW(),
    metric_name TEXT,
    value NUMERIC,
    unit TEXT
);

-- Automated monitoring data collection
CREATE OR REPLACE FUNCTION monitoring.collect_metrics()
RETURNS VOID
AS $$
BEGIN
    -- Collect health check data
    INSERT INTO monitoring.health_check_history (metric_name, current_value, status, message)
    SELECT metric_name, current_value, status, message
    FROM monitoring.database_health_check();
    
    -- Collect performance data
    INSERT INTO monitoring.performance_history (metric_name, value, unit)
    SELECT metric_name, value, unit
    FROM monitoring.performance_metrics();
    
    -- Clean up old data (keep 30 days)
    DELETE FROM monitoring.health_check_history 
    WHERE check_time < NOW() - INTERVAL '30 days';
    
    DELETE FROM monitoring.performance_history 
    WHERE metric_time < NOW() - INTERVAL '30 days';
END;
$$ LANGUAGE plpgsql;
```

### Alerting System

```python
#!/usr/bin/env python3
# postgresql_alerting.py

import psycopg2
import smtplib
import json
import time
import logging
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from datetime import datetime, timedelta
import requests

class PostgreSQLAlerting:
    def __init__(self, config_file):
        with open(config_file, 'r') as f:
            self.config = json.load(f)
        
        self.setup_logging()
        self.alert_history = {}
    
    def setup_logging(self):
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('/var/log/postgresql/alerting.log'),
                logging.StreamHandler()
            ]
        )
        self.logger = logging.getLogger(__name__)
    
    def connect_database(self):
        try:
            conn = psycopg2.connect(
                host=self.config['database']['host'],
                port=self.config['database']['port'],
                database=self.config['database']['name'],
                user=self.config['database']['user'],
                password=self.config['database']['password']
            )
            return conn
        except Exception as e:
            self.logger.error(f"Database connection failed: {e}")
            return None
    
    def check_health_metrics(self):
        conn = self.connect_database()
        if not conn:
            return []
        
        try:
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM monitoring.database_health_check()")
            results = cursor.fetchall()
            
            alerts = []
            for row in results:
                metric_name, current_value, threshold_warning, threshold_critical, status, message = row
                
                if status in ['WARNING', 'CRITICAL']:
                    alert = {
                        'metric': metric_name,
                        'value': current_value,
                        'status': status,
                        'message': message,
                        'timestamp': datetime.now()
                    }
                    alerts.append(alert)
            
            return alerts
        
        except Exception as e:
            self.logger.error(f"Health check failed: {e}")
            return []
        finally:
            conn.close()
    
    def should_send_alert(self, alert):
        """Implement alert throttling to prevent spam"""
        metric = alert['metric']
        status = alert['status']
        now = datetime.now()
        
        # Check if we've sent this alert recently
        key = f"{metric}_{status}"
        if key in self.alert_history:
            last_sent = self.alert_history[key]
            
            # For critical alerts, send every 15 minutes
            # For warnings, send every hour
            throttle_minutes = 15 if status == 'CRITICAL' else 60
            
            if now - last_sent < timedelta(minutes=throttle_minutes):
                return False
        
        self.alert_history[key] = now
        return True
    
    def send_email_alert(self, alerts):
        if not alerts:
            return
        
        try:
            smtp_config = self.config['email']
            
            msg = MIMEMultipart()
            msg['From'] = smtp_config['from']
            msg['To'] = ', '.join(smtp_config['to'])
            msg['Subject'] = f"PostgreSQL Alert - {len(alerts)} issues detected"
            
            # Create email body
            body = "PostgreSQL Database Alerts\n\n"
            body += f"Server: {self.config['database']['host']}\n"
            body += f"Database: {self.config['database']['name']}\n"
            body += f"Timestamp: {datetime.now()}\n\n"
            
            critical_count = sum(1 for alert in alerts if alert['status'] == 'CRITICAL')
            warning_count = sum(1 for alert in alerts if alert['status'] == 'WARNING')
            
            body += f"Summary: {critical_count} Critical, {warning_count} Warning\n\n"
            
            for alert in alerts:
                body += f"[{alert['status']}] {alert['metric']}\n"
                body += f"  Value: {alert['value']}\n"
                body += f"  Message: {alert['message']}\n\n"
            
            msg.attach(MIMEText(body, 'plain'))
            
            # Send email
            server = smtplib.SMTP(smtp_config['smtp_server'], smtp_config['smtp_port'])
            if smtp_config.get('use_tls'):
                server.starttls()
            if smtp_config.get('username'):
                server.login(smtp_config['username'], smtp_config['password'])
            
            server.send_message(msg)
            server.quit()
            
            self.logger.info(f"Email alert sent for {len(alerts)} issues")
        
        except Exception as e:
            self.logger.error(f"Failed to send email alert: {e}")
    
    def send_slack_alert(self, alerts):
        if not alerts or 'slack' not in self.config:
            return
        
        try:
            webhook_url = self.config['slack']['webhook_url']
            
            critical_count = sum(1 for alert in alerts if alert['status'] == 'CRITICAL')
            warning_count = sum(1 for alert in alerts if alert['status'] == 'WARNING')
            
            color = 'danger' if critical_count > 0 else 'warning'
            
            payload = {
                'username': 'PostgreSQL Monitor',
                'icon_emoji': ':warning:',
                'attachments': [{
                    'color': color,
                    'title': f'PostgreSQL Database Alert - {self.config["database"]["host"]}',
                    'text': f'{critical_count} Critical, {warning_count} Warning alerts',
                    'fields': []
                }]
            }
            
            for alert in alerts[:5]:  # Limit to first 5 alerts
                payload['attachments'][0]['fields'].append({
                    'title': f"[{alert['status']}] {alert['metric']}",
                    'value': alert['message'],
                    'short': False
                })
            
            if len(alerts) > 5:
                payload['attachments'][0]['fields'].append({
                    'title': 'Additional Alerts',
                    'value': f'... and {len(alerts) - 5} more alerts',
                    'short': False
                })
            
            response = requests.post(webhook_url, json=payload)
            response.raise_for_status()
            
            self.logger.info(f"Slack alert sent for {len(alerts)} issues")
        
        except Exception as e:
            self.logger.error(f"Failed to send Slack alert: {e}")
    
    def run_monitoring_cycle(self):
        self.logger.info("Starting monitoring cycle")
        
        # Check health metrics
        alerts = self.check_health_metrics()
        
        # Filter alerts that should be sent
        alerts_to_send = [alert for alert in alerts if self.should_send_alert(alert)]
        
        if alerts_to_send:
            self.logger.warning(f"Found {len(alerts_to_send)} alerts to send")
            
            # Send notifications
            self.send_email_alert(alerts_to_send)
            self.send_slack_alert(alerts_to_send)
        else:
            self.logger.info("No alerts to send")
    
    def run_continuous_monitoring(self):
        self.logger.info("Starting continuous monitoring")
        
        while True:
            try:
                self.run_monitoring_cycle()
                time.sleep(self.config.get('check_interval', 300))  # Default 5 minutes
            except KeyboardInterrupt:
                self.logger.info("Monitoring stopped by user")
                break
            except Exception as e:
                self.logger.error(f"Monitoring cycle failed: {e}")
                time.sleep(60)  # Wait 1 minute before retrying

if __name__ == "__main__":
    # Example configuration file (alerting_config.json)
    config_example = {
        "database": {
            "host": "localhost",
            "port": 5432,
            "name": "production_db",
            "user": "monitor_user",
            "password": "monitor_password"
        },
        "email": {
            "smtp_server": "smtp.gmail.com",
            "smtp_port": 587,
            "use_tls": True,
            "username": "alerts@company.com",
            "password": "email_password",
            "from": "alerts@company.com",
            "to": ["dba@company.com", "ops@company.com"]
        },
        "slack": {
            "webhook_url": "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
        },
        "check_interval": 300
    }
    
    alerting = PostgreSQLAlerting('alerting_config.json')
    alerting.run_continuous_monitoring()
```

## Backup and Recovery

### Production Backup Strategy

```bash
#!/bin/bash
# production_backup.sh

set -e

# Configuration
BACKUP_DIR="/backup/postgresql"
S3_BUCKET="company-postgres-backups"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/postgresql/backup.log"
DB_HOST="localhost"
DB_PORT="5432"
DB_USER="backup_user"
DB_NAME="production_db"

# Logging function
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

# Error handling
error_exit() {
    log "ERROR: $1"
    exit 1
}

# Create backup directories
mkdir -p "$BACKUP_DIR"/{full,incremental,wal,logs}

# Function to perform full backup
full_backup() {
    log "Starting full backup..."
    
    local backup_file="$BACKUP_DIR/full/full_backup_$DATE.sql.gz"
    local backup_info="$BACKUP_DIR/full/full_backup_$DATE.info"
    
    # Get WAL position before backup
    local wal_start=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT pg_current_wal_lsn();" | tr -d ' ')
    
    # Perform backup
    pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
        --verbose --format=custom --compress=9 \
        --file="${backup_file%.gz}" || error_exit "Full backup failed"
    
    # Compress backup
    gzip "${backup_file%.gz}" || error_exit "Backup compression failed"
    
    # Get WAL position after backup
    local wal_end=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -t -c "SELECT pg_current_wal_lsn();" | tr -d ' ')
    
    # Create backup info file
    cat > "$backup_info" << EOF
Backup Type: Full
Backup Date: $DATE
Database: $DB_NAME
WAL Start: $wal_start
WAL End: $wal_end
Backup File: $backup_file
File Size: $(du -h "$backup_file" | cut -f1)
EOF
    
    # Upload to S3
    aws s3 cp "$backup_file" "s3://$S3_BUCKET/full/" || log "WARNING: S3 upload failed"
    aws s3 cp "$backup_info" "s3://$S3_BUCKET/full/" || log "WARNING: S3 info upload failed"
    
    log "Full backup completed: $backup_file"
}

# Function to perform incremental backup
incremental_backup() {
    log "Starting incremental backup..."
    
    local backup_file="$BACKUP_DIR/incremental/incremental_backup_$DATE.sql.gz"
    local backup_info="$BACKUP_DIR/incremental/incremental_backup_$DATE.info"
    
    # Find last backup timestamp
    local last_backup=$(find "$BACKUP_DIR/full" "$BACKUP_DIR/incremental" -name "*.info" -exec grep "Backup Date:" {} \; | sort | tail -1 | cut -d' ' -f3)
    
    if [ -z "$last_backup" ]; then
        log "No previous backup found, performing full backup instead"
        full_backup
        return
    fi
    
    # Convert timestamp to PostgreSQL format
    local since_date=$(date -d "${last_backup:0:8} ${last_backup:9:2}:${last_backup:11:2}:${last_backup:13:2}" '+%Y-%m-%d %H:%M:%S')
    
    # Perform incremental backup (simplified - in practice, use WAL-based incremental)
    pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
        --verbose --format=custom --compress=9 \
        --file="${backup_file%.gz}" || error_exit "Incremental backup failed"
    
    gzip "${backup_file%.gz}" || error_exit "Backup compression failed"
    
    # Create backup info file
    cat > "$backup_info" << EOF
Backup Type: Incremental
Backup Date: $DATE
Database: $DB_NAME
Since: $since_date
Backup File: $backup_file
File Size: $(du -h "$backup_file" | cut -f1)
EOF
    
    # Upload to S3
    aws s3 cp "$backup_file" "s3://$S3_BUCKET/incremental/" || log "WARNING: S3 upload failed"
    aws s3 cp "$backup_info" "s3://$S3_BUCKET/incremental/" || log "WARNING: S3 info upload failed"
    
    log "Incremental backup completed: $backup_file"
}

# Function to backup WAL files
wal_backup() {
    log "Backing up WAL files..."
    
    local wal_archive_dir="/data/postgresql/wal/archive"
    
    if [ -d "$wal_archive_dir" ]; then
        # Sync WAL files to S3
        aws s3 sync "$wal_archive_dir/" "s3://$S3_BUCKET/wal/" --delete || log "WARNING: WAL S3 sync failed"
        
        # Clean up old local WAL files (keep 7 days)
        find "$wal_archive_dir" -type f -mtime +7 -delete
        
        log "WAL backup completed"
    else
        log "WARNING: WAL archive directory not found"
    fi
}

# Function to test backup integrity
test_backup() {
    local backup_file="$1"
    log "Testing backup integrity: $(basename "$backup_file")"
    
    # Create temporary test database
    local test_db="test_restore_$(date +%s)"
    
    createdb -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$test_db" || error_exit "Failed to create test database"
    
    # Restore backup to test database
    if zcat "$backup_file" | pg_restore -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$test_db" --verbose; then
        log "✓ Backup integrity test passed"
        
        # Run basic queries to verify data
        local table_count=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$test_db" -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';" | tr -d ' ')
        log "✓ Restored $table_count tables"
    else
        log "✗ Backup integrity test failed"
    fi
    
    # Clean up test database
    dropdb -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" "$test_db" || log "WARNING: Failed to drop test database"
}

# Function to clean up old backups
cleanup_backups() {
    log "Cleaning up old backups..."
    
    # Remove local backups older than retention period
    find "$BACKUP_DIR/full" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR/incremental" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR/full" -name "*.info" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR/incremental" -name "*.info" -mtime +$RETENTION_DAYS -delete
    
    # Clean up S3 backups (keep based on retention policy)
    local cutoff_date=$(date -d "$RETENTION_DAYS days ago" +%Y%m%d)
    
    aws s3 ls "s3://$S3_BUCKET/full/" | while read -r line; do
        local file_date=$(echo "$line" | awk '{print $4}' | grep -o '[0-9]\{8\}')
        if [ "$file_date" -lt "$cutoff_date" ]; then
            local file_name=$(echo "$line" | awk '{print $4}')
            aws s3 rm "s3://$S3_BUCKET/full/$file_name" || log "WARNING: Failed to delete S3 file $file_name"
        fi
    done
    
    log "Cleanup completed"
}

# Function to send backup report
send_backup_report() {
    local status="$1"
    local message="$2"
    
    # Create backup report
    local report_file="/tmp/backup_report_$DATE.txt"
    
    cat > "$report_file" << EOF
PostgreSQL Backup Report
========================

Date: $(date)
Server: $(hostname)
Database: $DB_NAME
Status: $status

$message

Recent Backups:
$(ls -la "$BACKUP_DIR/full/" | tail -5)

Disk Usage:
$(df -h "$BACKUP_DIR")

S3 Backup Status:
$(aws s3 ls "s3://$S3_BUCKET/" --recursive | tail -10)
EOF
    
    # Send email report (if configured)
    if command -v mail >/dev/null 2>&1; then
        mail -s "PostgreSQL Backup Report - $status" dba@company.com < "$report_file"
    fi
    
    rm -f "$report_file"
}

# Main execution
log "Starting backup process"

# Determine backup type based on day of week
if [ "$(date +%u)" -eq 7 ]; then
    # Sunday - full backup
    BACKUP_TYPE="full"
else
    # Other days - incremental backup
    BACKUP_TYPE="incremental"
fi

trap 'send_backup_report "FAILED" "Backup process failed. Check logs for details."' ERR

# Perform backup
if [ "$BACKUP_TYPE" = "full" ]; then
    full_backup
    
    # Test the backup
    latest_backup=$(ls -t "$BACKUP_DIR/full/"*.sql.gz | head -1)
    test_backup "$latest_backup"
else
    incremental_backup
fi

# Always backup WAL files
wal_backup

# Cleanup old backups
cleanup_backups

# Send success report
send_backup_report "SUCCESS" "Backup completed successfully. Type: $BACKUP_TYPE"

log "Backup process completed successfully"
```

## Performance Tuning

### Production Performance Configuration

```sql
-- Performance tuning queries and functions

-- Analyze query performance
CREATE OR REPLACE FUNCTION analyze_query_performance()
RETURNS TABLE(
    query_text TEXT,
    calls BIGINT,
    total_time_ms NUMERIC,
    avg_time_ms NUMERIC,
    rows_per_call NUMERIC,
    cache_hit_ratio NUMERIC
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        LEFT(s.query, 100) || CASE WHEN LENGTH(s.query) > 100 THEN '...' ELSE '' END,
        s.calls,
        ROUND(s.total_exec_time, 2),
        ROUND(s.mean_exec_time, 2),
        ROUND(s.rows::NUMERIC / s.calls, 2),
        ROUND(
            100.0 * s.shared_blks_hit / 
            NULLIF(s.shared_blks_hit + s.shared_blks_read, 0), 2
        )
    FROM pg_stat_statements s
    WHERE s.calls > 10
    ORDER BY s.total_exec_time DESC
    LIMIT 20;
END;
$$ LANGUAGE plpgsql;

-- Identify slow queries
CREATE OR REPLACE FUNCTION identify_slow_queries(min_duration_ms NUMERIC DEFAULT 1000)
RETURNS TABLE(
    query_text TEXT,
    avg_duration_ms NUMERIC,
    calls BIGINT,
    total_duration_ms NUMERIC,
    optimization_priority TEXT
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        LEFT(s.query, 200) || CASE WHEN LENGTH(s.query) > 200 THEN '...' ELSE '' END,
        ROUND(s.mean_exec_time, 2),
        s.calls,
        ROUND(s.total_exec_time, 2),
        CASE 
            WHEN s.mean_exec_time > 10000 THEN 'CRITICAL'
            WHEN s.mean_exec_time > 5000 THEN 'HIGH'
            WHEN s.mean_exec_time > 1000 THEN 'MEDIUM'
            ELSE 'LOW'
        END
    FROM pg_stat_statements s
    WHERE s.mean_exec_time >= min_duration_ms
      AND s.calls > 5
    ORDER BY s.mean_exec_time DESC;
END;
$$ LANGUAGE plpgsql;

-- Index usage analysis
CREATE OR REPLACE FUNCTION analyze_index_usage()
RETURNS TABLE(
    schema_name TEXT,
    table_name TEXT,
    index_name TEXT,
    index_scans BIGINT,
    table_scans BIGINT,
    index_size TEXT,
    usage_ratio NUMERIC,
    recommendation TEXT
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        schemaname::TEXT,
        tablename::TEXT,
        indexname::TEXT,
        idx_scan,
        seq_scan,
        pg_size_pretty(pg_relation_size(indexrelid)),
        ROUND(
            100.0 * idx_scan / NULLIF(idx_scan + seq_scan, 0), 2
        ),
        CASE 
            WHEN idx_scan = 0 THEN 'Consider dropping - unused'
            WHEN idx_scan < seq_scan / 10 THEN 'Low usage - review'
            WHEN idx_scan > seq_scan * 2 THEN 'Well used'
            ELSE 'Normal usage'
        END
    FROM pg_stat_user_indexes psi
    JOIN pg_stat_user_tables pst ON psi.relid = pst.relid
    ORDER BY idx_scan DESC;
END;
$$ LANGUAGE plpgsql;

-- Table maintenance analysis
CREATE OR REPLACE FUNCTION analyze_table_maintenance()
RETURNS TABLE(
    schema_name TEXT,
    table_name TEXT,
    table_size TEXT,
    dead_tuples BIGINT,
    live_tuples BIGINT,
    dead_ratio NUMERIC,
    last_vacuum TIMESTAMP,
    last_analyze TIMESTAMP,
    maintenance_needed TEXT
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        schemaname::TEXT,
        tablename::TEXT,
        pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)),
        n_dead_tup,
        n_live_tup,
        ROUND(
            100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2
        ),
        last_vacuum,
        last_analyze,
        CASE 
            WHEN n_dead_tup > n_live_tup * 0.2 THEN 'VACUUM NEEDED'
            WHEN last_vacuum < NOW() - INTERVAL '7 days' THEN 'VACUUM OVERDUE'
            WHEN last_analyze < NOW() - INTERVAL '3 days' THEN 'ANALYZE NEEDED'
            ELSE 'OK'
        END
    FROM pg_stat_user_tables
    ORDER BY n_dead_tup DESC;
END;
$$ LANGUAGE plpgsql;

-- Connection analysis
CREATE OR REPLACE FUNCTION analyze_connections()
RETURNS TABLE(
    state TEXT,
    count BIGINT,
    avg_duration INTERVAL,
    max_duration INTERVAL
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        COALESCE(a.state, 'unknown')::TEXT,
        COUNT(*),
        AVG(NOW() - a.state_change),
        MAX(NOW() - a.state_change)
    FROM pg_stat_activity a
    WHERE a.pid <> pg_backend_pid()
    GROUP BY a.state
    ORDER BY COUNT(*) DESC;
END;
$$ LANGUAGE plpgsql;

-- Run performance analysis
SELECT 'Query Performance' as analysis_type;
SELECT * FROM analyze_query_performance();

SELECT 'Slow Queries' as analysis_type;
SELECT * FROM identify_slow_queries(500);

SELECT 'Index Usage' as analysis_type;
SELECT * FROM analyze_index_usage();

SELECT 'Table Maintenance' as analysis_type;
SELECT * FROM analyze_table_maintenance();

SELECT 'Connection Analysis' as analysis_type;
SELECT * FROM analyze_connections();
```

## Deployment Automation

### Infrastructure as Code

```yaml
# docker-compose.yml for PostgreSQL production
version: '3.8'

services:
  postgresql-primary:
    image: postgres:15
    container_name: postgres-primary
    environment:
      POSTGRES_DB: production_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_INITDB_ARGS: "--data-checksums --encoding=UTF8"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - postgres_wal:/var/lib/postgresql/wal
      - ./config/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./config/pg_hba.conf:/etc/postgresql/pg_hba.conf
      - ./ssl:/etc/ssl/postgresql
      - ./scripts:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    command: >
      postgres
      -c config_file=/etc/postgresql/postgresql.conf
      -c hba_file=/etc/postgresql/pg_hba.conf
    networks:
      - postgres_network
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  postgresql-standby:
    image: postgres:15
    container_name: postgres-standby
    environment:
      POSTGRES_DB: production_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGUSER: postgres
    volumes:
      - postgres_standby_data:/var/lib/postgresql/data
      - ./ssl:/etc/ssl/postgresql
    ports:
      - "5433:5432"
    depends_on:
      - postgresql-primary
    networks:
      - postgres_network
    restart: unless-stopped
    command: >
      bash -c '
      until pg_basebackup -h postgresql-primary -D /var/lib/postgresql/data -U postgres -v -P -W;
      do
        echo "Waiting for primary to be ready..."
        sleep 5
      done
      echo "standby_mode = on" >> /var/lib/postgresql/data/recovery.conf
      echo "primary_conninfo = ''host=postgresql-primary port=5432 user=postgres''" >> /var/lib/postgresql/data/recovery.conf
      postgres
      '

  pgbouncer:
    image: pgbouncer/pgbouncer:latest
    container_name: pgbouncer
    environment:
      DATABASES_HOST: postgresql-primary
      DATABASES_PORT: 5432
      DATABASES_USER: postgres
      DATABASES_PASSWORD: ${POSTGRES_PASSWORD}
      DATABASES_DBNAME: production_db
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 25
    ports:
      - "6432:6432"
    depends_on:
      - postgresql-primary
    networks:
      - postgres_network
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - postgres_network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    networks:
      - postgres_network
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/postgresql/data
  postgres_standby_data:
    driver: local
  postgres_wal:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/postgresql/wal
  prometheus_data:
  grafana_data:

networks:
  postgres_network:
    driver: bridge
```

### Deployment Scripts

```bash
#!/bin/bash
# deploy_production.sh

set -e

ENVIRONMENT="${1:-production}"
VERSION="${2:-latest}"
CONFIG_DIR="./config"
SSL_DIR="./ssl"
SCRIPTS_DIR="./scripts"

echo "Deploying PostgreSQL to $ENVIRONMENT environment (version: $VERSION)"

# Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(staging|production)$ ]]; then
    echo "Error: Environment must be 'staging' or 'production'"
    exit 1
fi

# Load environment variables
if [ -f ".env.$ENVIRONMENT" ]; then
    source ".env.$ENVIRONMENT"
else
    echo "Error: Environment file .env.$ENVIRONMENT not found"
    exit 1
fi

# Validate required environment variables
required_vars=("POSTGRES_PASSWORD" "GRAFANA_PASSWORD")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "Error: Required environment variable $var is not set"
        exit 1
    fi
done

# Create necessary directories
sudo mkdir -p /data/postgresql/{data,wal,logs,backups}
sudo chown -R 999:999 /data/postgresql  # PostgreSQL container user

# Validate configuration files
echo "Validating configuration files..."
if [ ! -f "$CONFIG_DIR/postgresql.conf" ]; then
    echo "Error: postgresql.conf not found in $CONFIG_DIR"
    exit 1
fi

if [ ! -f "$CONFIG_DIR/pg_hba.conf" ]; then
    echo "Error: pg_hba.conf not found in $CONFIG_DIR"
    exit 1
fi

# Validate SSL certificates
echo "Validating SSL certificates..."
if [ ! -f "$SSL_DIR/server-cert.pem" ] || [ ! -f "$SSL_DIR/server-key.pem" ]; then
    echo "Error: SSL certificates not found in $SSL_DIR"
    exit 1
fi

# Test SSL certificate validity
openssl x509 -in "$SSL_DIR/server-cert.pem" -noout -checkend 86400 || {
    echo "Warning: SSL certificate expires within 24 hours"
}

# Backup existing data (if upgrading)
if [ "$ENVIRONMENT" = "production" ] && docker ps | grep -q postgres-primary; then
    echo "Creating backup before deployment..."
    ./scripts/backup_before_deploy.sh
fi

# Pull latest images
echo "Pulling Docker images..."
docker-compose pull

# Deploy services
echo "Deploying services..."
docker-compose -f docker-compose.yml -f "docker-compose.$ENVIRONMENT.yml" up -d

# Wait for PostgreSQL to be ready
echo "Waiting for PostgreSQL to be ready..."
for i in {1..30}; do
    if docker exec postgres-primary pg_isready -U postgres; then
        echo "PostgreSQL is ready"
        break
    fi
    echo "Waiting... ($i/30)"
    sleep 10
done

# Run post-deployment scripts
echo "Running post-deployment scripts..."
for script in "$SCRIPTS_DIR"/post-deploy-*.sql; do
    if [ -f "$script" ]; then
        echo "Executing $(basename "$script")..."
        docker exec -i postgres-primary psql -U postgres -d production_db < "$script"
    fi
done

# Verify deployment
echo "Verifying deployment..."
./scripts/verify_deployment.sh

# Update monitoring
echo "Updating monitoring configuration..."
docker-compose restart prometheus grafana

echo "Deployment completed successfully!"
echo "Services:"
echo "  PostgreSQL Primary: localhost:5432"
echo "  PostgreSQL Standby: localhost:5433"
echo "  PgBouncer: localhost:6432"
echo "  Prometheus: http://localhost:9090"
echo "  Grafana: http://localhost:3000"
```

### Health Check Scripts

```bash
#!/bin/bash
# verify_deployment.sh

set -e

echo "Verifying PostgreSQL deployment..."

# Test primary database connection
echo "Testing primary database connection..."
if docker exec postgres-primary psql -U postgres -d production_db -c "SELECT 1;" >/dev/null 2>&1; then
    echo "✓ Primary database connection successful"
else
    echo "✗ Primary database connection failed"
    exit 1
fi

# Test standby database connection
echo "Testing standby database connection..."
if docker exec postgres-standby psql -U postgres -d production_db -c "SELECT 1;" >/dev/null 2>&1; then
    echo "✓ Standby database connection successful"
else
    echo "✗ Standby database connection failed"
    exit 1
fi

# Test replication
echo "Testing replication..."
test_table="deployment_test_$(date +%s)"
docker exec postgres-primary psql -U postgres -d production_db -c "CREATE TABLE $test_table (id SERIAL, created_at TIMESTAMP DEFAULT NOW());"
docker exec postgres-primary psql -U postgres -d production_db -c "INSERT INTO $test_table DEFAULT VALUES;"

# Wait for replication
sleep 5

if docker exec postgres-standby psql -U postgres -d production_db -c "SELECT COUNT(*) FROM $test_table;" | grep -q "1"; then
    echo "✓ Replication working correctly"
else
    echo "✗ Replication not working"
    exit 1
fi

# Cleanup test table
docker exec postgres-primary psql -U postgres -d production_db -c "DROP TABLE $test_table;"

# Test PgBouncer
echo "Testing PgBouncer connection..."
if docker exec pgbouncer psql -h localhost -p 6432 -U postgres -d production_db -c "SELECT 1;" >/dev/null 2>&1; then
    echo "✓ PgBouncer connection successful"
else
    echo "✗ PgBouncer connection failed"
    exit 1
fi

# Test SSL connections
echo "Testing SSL connections..."
if docker exec postgres-primary psql "sslmode=require host=localhost user=postgres dbname=production_db" -c "SELECT 1;" >/dev/null 2>&1; then
    echo "✓ SSL connection successful"
else
    echo "✗ SSL connection failed"
    exit 1
fi

# Check monitoring services
echo "Checking monitoring services..."
if curl -s http://localhost:9090/-/healthy >/dev/null; then
    echo "✓ Prometheus is healthy"
else
    echo "✗ Prometheus health check failed"
fi

if curl -s http://localhost:3000/api/health >/dev/null; then
    echo "✓ Grafana is healthy"
else
    echo "✗ Grafana health check failed"
fi

# Check database performance
echo "Checking database performance..."
response_time=$(docker exec postgres-primary psql -U postgres -d production_db -c "\timing on" -c "SELECT COUNT(*) FROM information_schema.tables;" 2>&1 | grep "Time:" | awk '{print $2}')
echo "Query response time: $response_time"

echo "Deployment verification completed successfully!"
```

## Operational Procedures

### Daily Operations Checklist

```sql
-- Daily operations monitoring queries

-- 1. Check database health
SELECT 
    'Database Health' as check_type,
    CASE 
        WHEN pg_is_in_recovery() THEN 'STANDBY'
        ELSE 'PRIMARY'
    END as role,
    pg_database_size(current_database()) / 1024^3 as size_gb,
    (SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active') as active_connections,
    (SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'idle in transaction') as idle_in_transaction;

-- 2. Check replication status (run on primary)
SELECT 
    'Replication Status' as check_type,
    application_name,
    client_addr,
    state,
    sync_state,
    EXTRACT(EPOCH FROM replay_lag) as replay_lag_seconds
FROM pg_stat_replication;

-- 3. Check for long-running queries
SELECT 
    'Long Running Queries' as check_type,
    pid,
    usename,
    application_name,
    client_addr,
    state,
    EXTRACT(EPOCH FROM (NOW() - query_start)) as duration_seconds,
    LEFT(query, 100) as query_preview
FROM pg_stat_activity
WHERE state = 'active'
  AND query_start < NOW() - INTERVAL '5 minutes'
  AND pid <> pg_backend_pid()
ORDER BY query_start;

-- 4. Check for blocked queries
SELECT 
    'Blocked Queries' as check_type,
    blocked_locks.pid as blocked_pid,
    blocked_activity.usename as blocked_user,
    blocking_locks.pid as blocking_pid,
    blocking_activity.usename as blocking_user,
    blocked_activity.query as blocked_query,
    blocking_activity.query as blocking_query
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

-- 5. Check table maintenance needs
SELECT 
    'Maintenance Needed' as check_type,
    schemaname,
    tablename,
    n_dead_tup,
    n_live_tup,
    ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_tuple_ratio,
    last_vacuum,
    last_analyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
   OR last_vacuum < NOW() - INTERVAL '7 days'
   OR last_analyze < NOW() - INTERVAL '3 days'
ORDER BY n_dead_tup DESC;

-- 6. Check disk space usage
SELECT 
    'Disk Usage' as check_type,
    datname,
    pg_size_pretty(pg_database_size(datname)) as database_size,
    ROUND(100.0 * pg_database_size(datname) / 
          (SELECT SUM(pg_database_size(datname)) FROM pg_database), 2) as percentage
FROM pg_database
WHERE datname NOT IN ('template0', 'template1', 'postgres')
ORDER BY pg_database_size(datname) DESC;
```

### Incident Response Procedures

```bash
#!/bin/bash
# incident_response.sh

INCIDENT_TYPE="$1"
SEVERITY="$2"
LOG_FILE="/var/log/postgresql/incidents.log"

log_incident() {
    echo "$(date): [INCIDENT] [$SEVERITY] $1" | tee -a "$LOG_FILE"
}

case "$INCIDENT_TYPE" in
    "high_connections")
        log_incident "High connection count detected"
        
        # Get current connection info
        docker exec postgres-primary psql -U postgres -d production_db -c "
            SELECT state, COUNT(*) 
            FROM pg_stat_activity 
            GROUP BY state 
            ORDER BY COUNT(*) DESC;
        "
        
        # Kill idle connections if critical
        if [ "$SEVERITY" = "critical" ]; then
            log_incident "Killing idle connections"
            docker exec postgres-primary psql -U postgres -d production_db -c "
                SELECT pg_terminate_backend(pid)
                FROM pg_stat_activity
                WHERE state = 'idle'
                  AND state_change < NOW() - INTERVAL '1 hour'
                  AND pid <> pg_backend_pid();
            "
        fi
        ;;
        
    "replication_lag")
        log_incident "Replication lag detected"
        
        # Check replication status
        docker exec postgres-primary psql -U postgres -d production_db -c "
            SELECT 
                application_name,
                client_addr,
                state,
                replay_lag,
                sync_state
            FROM pg_stat_replication;
        "
        
        # Check standby status
        docker exec postgres-standby psql -U postgres -d production_db -c "
            SELECT 
                pg_is_in_recovery(),
                pg_last_wal_receive_lsn(),
                pg_last_wal_replay_lsn(),
                pg_last_xact_replay_timestamp();
        "
        ;;
        
    "disk_space")
        log_incident "Disk space issue detected"
        
        # Check disk usage
        df -h /data/postgresql
        
        # Clean up old WAL files if critical
        if [ "$SEVERITY" = "critical" ]; then
            log_incident "Cleaning up old WAL files"
            find /data/postgresql/wal/archive -type f -mtime +1 -delete
        fi
        ;;
        
    "slow_queries")
        log_incident "Slow queries detected"
        
        # Get slow query information
        docker exec postgres-primary psql -U postgres -d production_db -c "
            SELECT 
                pid,
                usename,
                application_name,
                client_addr,
                EXTRACT(EPOCH FROM (NOW() - query_start)) as duration_seconds,
                LEFT(query, 200) as query_preview
            FROM pg_stat_activity
            WHERE state = 'active'
              AND query_start < NOW() - INTERVAL '5 minutes'
              AND pid <> pg_backend_pid()
            ORDER BY query_start;
        "
        
        # Kill extremely long queries if critical
        if [ "$SEVERITY" = "critical" ]; then
            log_incident "Killing queries running longer than 30 minutes"
            docker exec postgres-primary psql -U postgres -d production_db -c "
                SELECT pg_terminate_backend(pid)
                FROM pg_stat_activity
                WHERE state = 'active'
                  AND query_start < NOW() - INTERVAL '30 minutes'
                  AND pid <> pg_backend_pid();
            "
        fi
        ;;
        
    "failover")
        log_incident "Initiating failover procedure"
        
        # Stop primary (if accessible)
        docker stop postgres-primary || true
        
        # Promote standby
        docker exec postgres-standby pg_ctl promote -D /var/lib/postgresql/data
        
        # Update connection routing
        # This would typically involve updating load balancer configuration
        log_incident "Failover completed - standby promoted to primary"
        ;;
        
    *)
        echo "Usage: $0 {high_connections|replication_lag|disk_space|slow_queries|failover} {warning|critical}"
        exit 1
        ;;
esac

log_incident "Incident response completed for $INCIDENT_TYPE"
```

## Best Practices Summary

### Security Best Practices

1. **Authentication and Authorization**
   - Use strong passwords and regular rotation
   - Implement role-based access control
   - Enable SSL/TLS for all connections
   - Use certificate-based authentication where possible

2. **Network Security**
   - Restrict network access using firewalls
   - Use VPNs for remote access
   - Implement network segmentation
   - Monitor network traffic

3. **Data Protection**
   - Enable data checksums
   - Implement encryption at rest
   - Regular security audits
   - Secure backup storage

### Performance Best Practices

1. **Configuration Optimization**
   - Tune memory settings based on workload
   - Optimize checkpoint and WAL settings
   - Configure appropriate connection limits
   - Enable query plan caching

2. **Monitoring and Maintenance**
   - Regular VACUUM and ANALYZE
   - Monitor query performance
   - Track index usage
   - Implement automated alerting

3. **Capacity Planning**
   - Monitor growth trends
   - Plan for peak loads
   - Implement connection pooling
   - Consider read replicas for scaling

### Operational Best Practices

1. **Backup and Recovery**
   - Automated backup procedures
   - Regular recovery testing
   - Offsite backup storage
   - Document recovery procedures

2. **Change Management**
   - Version control for configurations
   - Staged deployment process
   - Rollback procedures
   - Change documentation

3. **Incident Management**
   - Defined escalation procedures
   - Automated incident detection
   - Post-incident reviews
   - Knowledge base maintenance

## Troubleshooting Common Issues

### Connection Issues

```sql
-- Check connection limits
SELECT 
    setting as max_connections,
    (SELECT COUNT(*) FROM pg_stat_activity) as current_connections,
    ROUND(100.0 * (SELECT COUNT(*) FROM pg_stat_activity) / setting::numeric, 2) as usage_percentage
FROM pg_settings 
WHERE name = 'max_connections';

-- Find connection sources
SELECT 
    client_addr,
    application_name,
    state,
    COUNT(*) as connection_count
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY client_addr, application_name, state
ORDER BY connection_count DESC;
```

### Performance Issues

```sql
-- Find resource-intensive queries
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Check for missing indexes
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / seq_scan as avg_seq_tup_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 10;
```

### Storage Issues

```sql
-- Check table and index sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as index_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Check WAL usage
SELECT 
    pg_current_wal_lsn(),
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 as wal_mb_generated;
```

## Next Steps

After mastering production deployment:
1. Advanced Topics and Extensions (17-advanced-topics.md)
2. PostgreSQL Ecosystem and Tools (18-ecosystem-tools.md)
3. Case Studies and Real-world Examples (19-case-studies.md)

---
*This is part 16 of the PostgreSQL learning series*