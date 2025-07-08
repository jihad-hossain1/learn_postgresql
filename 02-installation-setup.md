# PostgreSQL Installation and Setup

## Prerequisites

- Administrative privileges on your system
- At least 1GB of free disk space
- 512MB RAM minimum (2GB+ recommended)

## Installation by Operating System

### Windows Installation

#### Method 1: Official Installer
1. Visit [postgresql.org/download/windows](https://www.postgresql.org/download/windows/)
2. Download the installer for your Windows version
3. Run the installer as administrator
4. Follow the installation wizard:
   - Choose installation directory (default: `C:\Program Files\PostgreSQL\15`)
   - Select components (PostgreSQL Server, pgAdmin, Command Line Tools)
   - Set data directory (default: `C:\Program Files\PostgreSQL\15\data`)
   - Set password for postgres user (remember this!)
   - Set port number (default: 5432)
   - Choose locale (default is usually fine)

#### Method 2: Using Chocolatey
```powershell
# Install Chocolatey first if not installed
choco install postgresql
```

### macOS Installation

#### Method 1: Homebrew (Recommended)
```bash
# Install Homebrew if not installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install PostgreSQL
brew install postgresql@15

# Start PostgreSQL service
brew services start postgresql@15

# Create a database
createdb mydatabase
```

#### Method 2: Postgres.app
1. Download from [postgresapp.com](https://postgresapp.com/)
2. Drag to Applications folder
3. Launch the app
4. Click "Initialize" to create a new server

### Linux Installation

#### Ubuntu/Debian
```bash
# Update package list
sudo apt update

# Install PostgreSQL and additional utilities
sudo apt install postgresql postgresql-contrib

# Start and enable PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Check status
sudo systemctl status postgresql
```

#### CentOS/RHEL/Fedora
```bash
# Install PostgreSQL
sudo dnf install postgresql postgresql-server postgresql-contrib

# Initialize database
sudo postgresql-setup --initdb

# Start and enable service
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

## Initial Configuration

### Setting Up the postgres User

#### Linux/macOS
```bash
# Switch to postgres user
sudo -i -u postgres

# Access PostgreSQL prompt
psql

# Set password for postgres user
\password postgres

# Exit
\q
exit
```

#### Windows
The password is set during installation.

### Creating Your First Database

```sql
-- Connect to PostgreSQL
psql -U postgres -h localhost

-- Create a new database
CREATE DATABASE myapp;

-- Create a new user
CREATE USER myuser WITH PASSWORD 'mypassword';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE myapp TO myuser;

-- Connect to the new database
\c myapp
```

## Configuration Files

### postgresql.conf
Main configuration file located in data directory:

```conf
# Connection settings
listen_addresses = 'localhost'  # or '*' for all interfaces
port = 5432
max_connections = 100

# Memory settings
shared_buffers = 128MB
effective_cache_size = 4GB
work_mem = 4MB

# Logging
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
```

### pg_hba.conf
Client authentication configuration:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
local   all             all                                     md5
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

## Essential Tools

### Command Line Tools

- **psql**: Interactive terminal for PostgreSQL
- **createdb**: Create databases
- **dropdb**: Delete databases
- **pg_dump**: Backup databases
- **pg_restore**: Restore databases

### GUI Tools

1. **pgAdmin**: Web-based administration tool
   - Included with PostgreSQL installation
   - Access via browser: http://localhost:5050

2. **DBeaver**: Universal database tool
   - Free community edition available
   - Cross-platform

3. **DataGrip**: JetBrains IDE for databases
   - Commercial product
   - Advanced features

## Basic psql Commands

```sql
-- Connect to database
psql -U username -d database_name -h hostname

-- List databases
\l

-- Connect to database
\c database_name

-- List tables
\dt

-- Describe table
\d table_name

-- List users
\du

-- Show current database
SELECT current_database();

-- Show current user
SELECT current_user;

-- Exit
\q
```

## Environment Variables

```bash
# Set environment variables (Linux/macOS)
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=myapp
export PGUSER=myuser
export PGPASSWORD=mypassword

# Windows (Command Prompt)
set PGHOST=localhost
set PGPORT=5432
set PGDATABASE=myapp
set PGUSER=myuser
set PGPASSWORD=mypassword
```

## Troubleshooting Common Issues

### Connection Issues
```bash
# Check if PostgreSQL is running
sudo systemctl status postgresql  # Linux
brew services list | grep postgresql  # macOS

# Check port availability
netstat -an | grep 5432
```

### Permission Issues
```sql
-- Grant connection permission
GRANT CONNECT ON DATABASE myapp TO myuser;

-- Grant schema usage
GRANT USAGE ON SCHEMA public TO myuser;

-- Grant table permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO myuser;
```

### Reset postgres Password
```bash
# Linux - edit pg_hba.conf to use 'trust' method temporarily
sudo nano /etc/postgresql/15/main/pg_hba.conf
# Change 'md5' to 'trust' for local connections
# Restart PostgreSQL
sudo systemctl restart postgresql
# Connect and change password
psql -U postgres
\password postgres
# Revert pg_hba.conf changes and restart
```

## Next Steps

After completing installation and setup:
1. Basic SQL Operations (03-basic-sql-operations.md)
2. Data Types and Constraints (04-data-types-constraints.md)
3. Database Design (05-database-design.md)

---
*This is part 2 of the PostgreSQL learning series*