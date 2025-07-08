# Backup and Recovery in PostgreSQL

## Introduction

Backup and recovery are critical aspects of database administration. PostgreSQL provides multiple backup strategies and recovery options to protect your data and ensure business continuity.

## Types of Backups

### Logical Backups
Logical backups export data in a human-readable format (SQL statements).

### Physical Backups
Physical backups copy the actual database files and transaction logs.

### Comparison

| Feature | Logical Backup | Physical Backup |
|---------|----------------|------------------|
| Speed | Slower | Faster |
| Size | Smaller (compressed) | Larger |
| Portability | High | Platform-specific |
| Point-in-time Recovery | No | Yes |
| Partial Restore | Easy | Limited |
| Cross-version | Flexible | Restricted |

## Logical Backups with pg_dump

### Basic pg_dump Usage

```bash
# Backup entire database
pg_dump -h localhost -U postgres -d mydb > mydb_backup.sql

# Backup with compression
pg_dump -h localhost -U postgres -d mydb | gzip > mydb_backup.sql.gz

# Backup in custom format (recommended)
pg_dump -h localhost -U postgres -d mydb -Fc > mydb_backup.dump

# Backup in directory format
pg_dump -h localhost -U postgres -d mydb -Fd -f mydb_backup_dir

# Backup in tar format
pg_dump -h localhost -U postgres -d mydb -Ft > mydb_backup.tar
```

### Advanced pg_dump Options

```bash
# Backup specific tables
pg_dump -h localhost -U postgres -d mydb -t employees -t departments > tables_backup.sql

# Backup specific schema
pg_dump -h localhost -U postgres -d mydb -n public > public_schema.sql

# Exclude specific tables
pg_dump -h localhost -U postgres -d mydb -T audit_logs -T temp_* > mydb_no_audit.sql

# Data-only backup (no schema)
pg_dump -h localhost -U postgres -d mydb --data-only > data_only.sql

# Schema-only backup (no data)
pg_dump -h localhost -U postgres -d mydb --schema-only > schema_only.sql

# Backup with verbose output
pg_dump -h localhost -U postgres -d mydb -v -Fc > mydb_backup.dump

# Parallel backup (directory format only)
pg_dump -h localhost -U postgres -d mydb -Fd -j 4 -f mydb_parallel_backup
```

### Backup Scripts

```bash
#!/bin/bash
# backup_database.sh

# Configuration
DB_HOST="localhost"
DB_USER="postgres"
DB_NAME="mydb"
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${DATE}.dump"
LOG_FILE="${BACKUP_DIR}/backup_${DATE}.log"

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Perform backup
echo "Starting backup of $DB_NAME at $(date)" >> "$LOG_FILE"
pg_dump -h "$DB_HOST" -U "$DB_USER" -d "$DB_NAME" -Fc -v > "$BACKUP_FILE" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    echo "Backup completed successfully at $(date)" >> "$LOG_FILE"
    
    # Compress backup
    gzip "$BACKUP_FILE"
    
    # Remove backups older than 7 days
    find "$BACKUP_DIR" -name "${DB_NAME}_*.dump.gz" -mtime +7 -delete
    
    echo "Backup cleanup completed" >> "$LOG_FILE"
else
    echo "Backup failed at $(date)" >> "$LOG_FILE"
    exit 1
fi
```

### Windows Backup Script

```batch
@echo off
REM backup_database.bat

set DB_HOST=localhost
set DB_USER=postgres
set DB_NAME=mydb
set BACKUP_DIR=C:\backups\postgresql
set DATE=%date:~-4,4%%date:~-10,2%%date:~-7,2%_%time:~0,2%%time:~3,2%%time:~6,2%
set DATE=%DATE: =0%
set BACKUP_FILE=%BACKUP_DIR%\%DB_NAME%_%DATE%.dump

REM Create backup directory
if not exist "%BACKUP_DIR%" mkdir "%BACKUP_DIR%"

REM Perform backup
echo Starting backup of %DB_NAME% at %date% %time%
pg_dump -h %DB_HOST% -U %DB_USER% -d %DB_NAME% -Fc -v > "%BACKUP_FILE%"

if %errorlevel% equ 0 (
    echo Backup completed successfully
) else (
    echo Backup failed
    exit /b 1
)
```

## Logical Backups with pg_dumpall

### Cluster-wide Backups

```bash
# Backup entire PostgreSQL cluster
pg_dumpall -h localhost -U postgres > cluster_backup.sql

# Backup only global objects (roles, tablespaces)
pg_dumpall -h localhost -U postgres --globals-only > globals_backup.sql

# Backup only roles
pg_dumpall -h localhost -U postgres --roles-only > roles_backup.sql

# Backup only tablespaces
pg_dumpall -h localhost -U postgres --tablespaces-only > tablespaces_backup.sql
```

### Automated Cluster Backup

```bash
#!/bin/bash
# cluster_backup.sh

DB_HOST="localhost"
DB_USER="postgres"
BACKUP_DIR="/var/backups/postgresql/cluster"
DATE=$(date +"%Y%m%d_%H%M%S")

mkdir -p "$BACKUP_DIR"

# Backup globals
pg_dumpall -h "$DB_HOST" -U "$DB_USER" --globals-only > "${BACKUP_DIR}/globals_${DATE}.sql"

# Backup each database individually
psql -h "$DB_HOST" -U "$DB_USER" -t -c "SELECT datname FROM pg_database WHERE NOT datistemplate AND datname != 'postgres'" | while read dbname; do
    if [ -n "$dbname" ]; then
        echo "Backing up database: $dbname"
        pg_dump -h "$DB_HOST" -U "$DB_USER" -d "$dbname" -Fc > "${BACKUP_DIR}/${dbname}_${DATE}.dump"
    fi
done

echo "Cluster backup completed at $(date)"
```

## Physical Backups

### File System Level Backup

```bash
# Stop PostgreSQL service
sudo systemctl stop postgresql

# Backup data directory
sudo tar -czf /backup/postgresql_data_$(date +%Y%m%d).tar.gz /var/lib/postgresql/data/

# Start PostgreSQL service
sudo systemctl start postgresql
```

### Online Physical Backup with pg_basebackup

```bash
# Basic base backup
pg_basebackup -h localhost -U postgres -D /backup/base_backup -Ft -z -P

# Base backup with WAL files
pg_basebackup -h localhost -U postgres -D /backup/base_backup -Ft -z -P -W

# Base backup in plain format
pg_basebackup -h localhost -U postgres -D /backup/base_backup -Fp -P

# Base backup with specific tablespace mapping
pg_basebackup -h localhost -U postgres -D /backup/base_backup -Ft -z -P -T /old/path=/new/path
```

### Continuous Archiving Setup

#### Configure WAL Archiving

```sql
-- Enable WAL archiving
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f';
ALTER SYSTEM SET max_wal_senders = 3;
ALTER SYSTEM SET wal_keep_segments = 32;  -- For older versions
ALTER SYSTEM SET wal_keep_size = '512MB'; -- For newer versions

-- Reload configuration
SELECT pg_reload_conf();
```

#### WAL Archive Script

```bash
#!/bin/bash
# wal_archive.sh

WAL_FILE="$1"
ARCHIVE_DIR="/var/lib/postgresql/wal_archive"
LOG_FILE="/var/log/postgresql/wal_archive.log"

# Create archive directory if it doesn't exist
mkdir -p "$ARCHIVE_DIR"

# Copy WAL file to archive
cp "$WAL_FILE" "$ARCHIVE_DIR/" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    echo "$(date): Successfully archived $WAL_FILE" >> "$LOG_FILE"
    exit 0
else
    echo "$(date): Failed to archive $WAL_FILE" >> "$LOG_FILE"
    exit 1
fi
```

#### Update PostgreSQL Configuration

```bash
# Update archive_command in postgresql.conf
archive_command = '/path/to/wal_archive.sh %p'

# Or for remote archiving
archive_command = 'rsync -a %p user@backup-server:/backup/wal_archive/%f'
```

## Restore Operations

### Restoring from pg_dump

```bash
# Restore from SQL dump
psql -h localhost -U postgres -d mydb < mydb_backup.sql

# Restore from custom format
pg_restore -h localhost -U postgres -d mydb mydb_backup.dump

# Restore with verbose output
pg_restore -h localhost -U postgres -d mydb -v mydb_backup.dump

# Restore specific tables
pg_restore -h localhost -U postgres -d mydb -t employees -t departments mydb_backup.dump

# Restore schema only
pg_restore -h localhost -U postgres -d mydb --schema-only mydb_backup.dump

# Restore data only
pg_restore -h localhost -U postgres -d mydb --data-only mydb_backup.dump

# Parallel restore
pg_restore -h localhost -U postgres -d mydb -j 4 mydb_backup.dump
```

### Creating Database for Restore

```sql
-- Create new database for restore
CREATE DATABASE mydb_restored;

-- Restore to new database
pg_restore -h localhost -U postgres -d mydb_restored mydb_backup.dump
```

### Restore Script

```bash
#!/bin/bash
# restore_database.sh

DB_HOST="localhost"
DB_USER="postgres"
DB_NAME="$1"
BACKUP_FILE="$2"
NEW_DB_NAME="${DB_NAME}_restored_$(date +%Y%m%d_%H%M%S)"

if [ -z "$DB_NAME" ] || [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <database_name> <backup_file>"
    exit 1
fi

if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file $BACKUP_FILE not found"
    exit 1
fi

echo "Creating database $NEW_DB_NAME"
psql -h "$DB_HOST" -U "$DB_USER" -c "CREATE DATABASE $NEW_DB_NAME;"

echo "Restoring from $BACKUP_FILE"
pg_restore -h "$DB_HOST" -U "$DB_USER" -d "$NEW_DB_NAME" -v "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Restore completed successfully"
    echo "Database restored as: $NEW_DB_NAME"
else
    echo "Restore failed"
    exit 1
fi
```

## Point-in-Time Recovery (PITR)

### Setting up PITR

```sql
-- Configure for PITR
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f';
ALTER SYSTEM SET max_wal_senders = 3;
ALTER SYSTEM SET checkpoint_completion_target = 0.9;

-- Reload configuration
SELECT pg_reload_conf();

-- Restart PostgreSQL to apply wal_level change
-- sudo systemctl restart postgresql
```

### Creating Base Backup for PITR

```bash
# Create base backup
pg_basebackup -h localhost -U postgres -D /backup/pitr_base -Fp -P -W

# Note the backup start time
echo "Base backup completed at: $(date)" > /backup/pitr_base/backup_info.txt
```

### Performing PITR

```bash
# Stop PostgreSQL
sudo systemctl stop postgresql

# Backup current data directory
sudo mv /var/lib/postgresql/data /var/lib/postgresql/data_old

# Restore base backup
sudo cp -R /backup/pitr_base /var/lib/postgresql/data
sudo chown -R postgres:postgres /var/lib/postgresql/data
```

### Create Recovery Configuration

```bash
# Create recovery.signal file (PostgreSQL 12+)
sudo touch /var/lib/postgresql/data/recovery.signal

# Configure recovery in postgresql.conf
sudo tee -a /var/lib/postgresql/data/postgresql.conf << EOF
# Recovery settings
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
# recovery_target_xid = '12345'
# recovery_target_lsn = '0/1234567'
# recovery_target_name = 'my_savepoint'
recovery_target_action = 'promote'
EOF
```

### Recovery Configuration Options

```bash
# Recovery to specific time
recovery_target_time = '2024-01-15 14:30:00'

# Recovery to specific transaction ID
recovery_target_xid = '12345'

# Recovery to specific LSN
recovery_target_lsn = '0/1234567'

# Recovery to named restore point
recovery_target_name = 'before_data_corruption'

# Recovery actions
recovery_target_action = 'promote'    # Promote to primary
recovery_target_action = 'pause'      # Pause recovery
recovery_target_action = 'shutdown'   # Shutdown after recovery

# Recovery timeline
recovery_target_timeline = 'latest'   # Follow latest timeline
recovery_target_timeline = '1'        # Follow specific timeline
```

### Start Recovery

```bash
# Start PostgreSQL to begin recovery
sudo systemctl start postgresql

# Monitor recovery progress
sudo tail -f /var/log/postgresql/postgresql-*.log

# Check recovery status
psql -h localhost -U postgres -c "SELECT pg_is_in_recovery();"
```

### Creating Named Restore Points

```sql
-- Create named restore point
SELECT pg_create_restore_point('before_major_update');

-- List restore points (check logs)
-- Restore points are logged in PostgreSQL logs
```

## Streaming Replication for Backup

### Setting up Streaming Replication

#### Primary Server Configuration

```sql
-- Configure primary server
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET max_wal_senders = 3;
ALTER SYSTEM SET wal_keep_size = '512MB';
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f';

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl_password';

-- Reload configuration
SELECT pg_reload_conf();
```

#### Configure pg_hba.conf

```bash
# Add to pg_hba.conf
host replication replicator 192.168.1.0/24 md5
```

#### Standby Server Setup

```bash
# Create base backup for standby
pg_basebackup -h primary_server -U replicator -D /var/lib/postgresql/data -P -W

# Create standby.signal
touch /var/lib/postgresql/data/standby.signal

# Configure standby in postgresql.conf
echo "primary_conninfo = 'host=primary_server port=5432 user=replicator password=repl_password'" >> /var/lib/postgresql/data/postgresql.conf
echo "restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'" >> /var/lib/postgresql/data/postgresql.conf
```

### Monitoring Replication

```sql
-- On primary: Check replication status
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state
FROM pg_stat_replication;

-- On standby: Check recovery status
SELECT 
    pg_is_in_recovery(),
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_last_xact_replay_timestamp();

-- Check replication lag
SELECT 
    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag_seconds;
```

## Backup Strategies

### Daily Backup Strategy

```bash
#!/bin/bash
# daily_backup_strategy.sh

BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +"%Y%m%d")
DAY_OF_WEEK=$(date +"%u")  # 1=Monday, 7=Sunday
DAY_OF_MONTH=$(date +"%d")

# Daily logical backup
pg_dump -h localhost -U postgres -d mydb -Fc > "${BACKUP_DIR}/daily/mydb_${DATE}.dump"

# Weekly full backup (Sundays)
if [ "$DAY_OF_WEEK" -eq 7 ]; then
    pg_basebackup -h localhost -U postgres -D "${BACKUP_DIR}/weekly/base_${DATE}" -Ft -z -P
fi

# Monthly archive (1st of month)
if [ "$DAY_OF_MONTH" -eq "01" ]; then
    cp "${BACKUP_DIR}/daily/mydb_${DATE}.dump" "${BACKUP_DIR}/monthly/"
fi

# Cleanup old backups
find "${BACKUP_DIR}/daily" -name "*.dump" -mtime +7 -delete
find "${BACKUP_DIR}/weekly" -name "base_*" -mtime +30 -delete
find "${BACKUP_DIR}/monthly" -name "*.dump" -mtime +365 -delete
```

### Backup Rotation Script

```bash
#!/bin/bash
# backup_rotation.sh

BACKUP_DIR="/var/backups/postgresql"
DB_NAME="mydb"
DATE=$(date +"%Y%m%d_%H%M%S")
DAY_OF_WEEK=$(date +"%A")

# Create backup
BACKUP_FILE="${BACKUP_DIR}/${DAY_OF_WEEK}/${DB_NAME}_${DATE}.dump"
mkdir -p "${BACKUP_DIR}/${DAY_OF_WEEK}"

pg_dump -h localhost -U postgres -d "$DB_NAME" -Fc > "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Backup successful: $BACKUP_FILE"
    
    # Keep only the latest backup for each day
    find "${BACKUP_DIR}/${DAY_OF_WEEK}" -name "${DB_NAME}_*.dump" -type f | sort | head -n -1 | xargs rm -f
else
    echo "Backup failed"
    exit 1
fi
```

## Disaster Recovery Planning

### Recovery Time Objective (RTO) and Recovery Point Objective (RPO)

| Backup Method | RPO | RTO | Use Case |
|---------------|-----|-----|----------|
| Daily pg_dump | 24 hours | 2-4 hours | Small databases |
| Hourly pg_dump | 1 hour | 1-2 hours | Medium databases |
| WAL archiving | 15 minutes | 30-60 minutes | Production systems |
| Streaming replication | < 1 minute | 5-15 minutes | High availability |
| Synchronous replication | 0 | 1-5 minutes | Critical systems |

### Disaster Recovery Procedures

#### Complete System Recovery

```bash
#!/bin/bash
# disaster_recovery.sh

echo "Starting disaster recovery procedure..."

# 1. Install PostgreSQL on new server
# (Assume PostgreSQL is already installed)

# 2. Stop PostgreSQL service
sudo systemctl stop postgresql

# 3. Restore base backup
echo "Restoring base backup..."
sudo rm -rf /var/lib/postgresql/data/*
sudo tar -xzf /backup/base_backup.tar.gz -C /var/lib/postgresql/data/
sudo chown -R postgres:postgres /var/lib/postgresql/data

# 4. Configure recovery
echo "Configuring recovery..."
sudo touch /var/lib/postgresql/data/recovery.signal
sudo tee /var/lib/postgresql/data/recovery.conf << EOF
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_timeline = 'latest'
EOF

# 5. Start PostgreSQL
echo "Starting PostgreSQL..."
sudo systemctl start postgresql

# 6. Monitor recovery
echo "Monitoring recovery progress..."
while psql -h localhost -U postgres -c "SELECT pg_is_in_recovery();" | grep -q "t"; do
    echo "Recovery in progress..."
    sleep 10
done

echo "Disaster recovery completed successfully!"
```

### Testing Backup and Recovery

```bash
#!/bin/bash
# test_backup_recovery.sh

TEST_DB="test_recovery_$(date +%Y%m%d_%H%M%S)"
BACKUP_FILE="$1"

if [ -z "$BACKUP_FILE" ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

echo "Testing backup recovery with file: $BACKUP_FILE"

# Create test database
psql -h localhost -U postgres -c "CREATE DATABASE $TEST_DB;"

# Restore backup
pg_restore -h localhost -U postgres -d "$TEST_DB" "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Backup restore test PASSED"
    
    # Run basic validation queries
    echo "Running validation queries..."
    psql -h localhost -U postgres -d "$TEST_DB" -c "\dt"
    psql -h localhost -U postgres -d "$TEST_DB" -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';"
    
    # Cleanup test database
    psql -h localhost -U postgres -c "DROP DATABASE $TEST_DB;"
    echo "Test database cleaned up"
else
    echo "Backup restore test FAILED"
    exit 1
fi
```

## Monitoring and Alerting

### Backup Monitoring Queries

```sql
-- Check last backup time (requires backup logging table)
CREATE TABLE backup_log (
    backup_id SERIAL PRIMARY KEY,
    database_name VARCHAR(100),
    backup_type VARCHAR(50),
    backup_file VARCHAR(500),
    backup_size BIGINT,
    start_time TIMESTAMP,
    end_time TIMESTAMP,
    status VARCHAR(20)
);

-- Query recent backups
SELECT 
    database_name,
    backup_type,
    backup_size / 1024 / 1024 AS size_mb,
    start_time,
    end_time,
    EXTRACT(EPOCH FROM (end_time - start_time)) AS duration_seconds,
    status
FROM backup_log
WHERE start_time >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY start_time DESC;

-- Check for failed backups
SELECT 
    database_name,
    backup_type,
    start_time,
    status
FROM backup_log
WHERE status != 'SUCCESS'
AND start_time >= CURRENT_DATE - INTERVAL '1 day';
```

### WAL Archive Monitoring

```sql
-- Check WAL archiving status
SELECT 
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;

-- Check current WAL file
SELECT pg_current_wal_lsn(), pg_current_wal_file();

-- Check WAL generation rate
SELECT 
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 AS total_wal_mb,
    pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / 1024 / 1024 / 
    EXTRACT(EPOCH FROM (NOW() - pg_postmaster_start_time())) * 3600 AS wal_mb_per_hour;
```

### Backup Validation Script

```bash
#!/bin/bash
# validate_backups.sh

BACKUP_DIR="/var/backups/postgresql"
LOG_FILE="/var/log/postgresql/backup_validation.log"
ERROR_COUNT=0

echo "Starting backup validation at $(date)" >> "$LOG_FILE"

# Check if backup files exist
for backup_file in "$BACKUP_DIR"/*.dump; do
    if [ -f "$backup_file" ]; then
        echo "Validating $backup_file" >> "$LOG_FILE"
        
        # Check if file is readable
        if pg_restore --list "$backup_file" > /dev/null 2>&1; then
            echo "✓ $backup_file is valid" >> "$LOG_FILE"
        else
            echo "✗ $backup_file is corrupted" >> "$LOG_FILE"
            ERROR_COUNT=$((ERROR_COUNT + 1))
        fi
    fi
done

# Check WAL archive
WAL_ARCHIVE_DIR="/var/lib/postgresql/wal_archive"
WAL_COUNT=$(find "$WAL_ARCHIVE_DIR" -name "*.wal" -mtime -1 | wc -l)

if [ "$WAL_COUNT" -gt 0 ]; then
    echo "✓ WAL archiving is working ($WAL_COUNT files in last 24h)" >> "$LOG_FILE"
else
    echo "✗ No WAL files archived in last 24 hours" >> "$LOG_FILE"
    ERROR_COUNT=$((ERROR_COUNT + 1))
fi

echo "Backup validation completed with $ERROR_COUNT errors" >> "$LOG_FILE"

if [ "$ERROR_COUNT" -gt 0 ]; then
    # Send alert (example with mail)
    echo "Backup validation failed with $ERROR_COUNT errors" | mail -s "PostgreSQL Backup Alert" admin@company.com
    exit 1
fi
```

## Best Practices

### Backup Best Practices

1. **Regular Testing** - Test restore procedures regularly
2. **Multiple Locations** - Store backups in multiple locations
3. **Encryption** - Encrypt sensitive backup data
4. **Compression** - Use compression to save storage space
5. **Monitoring** - Monitor backup success/failure
6. **Documentation** - Document recovery procedures
7. **Automation** - Automate backup processes
8. **Retention Policy** - Implement appropriate retention policies

### Security Considerations

```bash
# Encrypt backups
pg_dump -h localhost -U postgres -d mydb | gpg --cipher-algo AES256 --compress-algo 1 --symmetric --output mydb_backup.dump.gpg

# Decrypt backup
gpg --decrypt mydb_backup.dump.gpg | psql -h localhost -U postgres -d mydb_restored

# Secure backup storage permissions
chmod 600 /var/backups/postgresql/*.dump
chown postgres:postgres /var/backups/postgresql/*.dump
```

### Performance Optimization

```bash
# Use parallel processing
pg_dump -j 4 -Fd -f backup_directory mydb
pg_restore -j 4 -d mydb_restored backup_directory

# Optimize compression
pg_dump -Z 6 mydb > mydb_backup.sql.gz  # Compression level 1-9

# Use custom format for flexibility
pg_dump -Fc mydb > mydb_backup.dump
```

## Troubleshooting Common Issues

### Backup Issues

```bash
# Check disk space
df -h /var/backups/postgresql

# Check permissions
ls -la /var/backups/postgresql

# Test database connectivity
pg_isready -h localhost -U postgres

# Check PostgreSQL logs
tail -f /var/log/postgresql/postgresql-*.log
```

### Recovery Issues

```sql
-- Check recovery status
SELECT pg_is_in_recovery();

-- Check last replayed transaction
SELECT pg_last_xact_replay_timestamp();

-- Check recovery conflicts
SELECT * FROM pg_stat_database_conflicts;
```

## Next Steps

After mastering backup and recovery:
1. Security and User Management (11-security-users.md)
2. Monitoring and Maintenance (12-monitoring-maintenance.md)
3. Advanced Administration (13-advanced-administration.md)

---
*This is part 10 of the PostgreSQL learning series*