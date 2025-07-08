# Security and User Management in PostgreSQL

## Introduction

PostgreSQL provides a comprehensive security model with role-based access control, authentication methods, and various security features to protect your data and control access to database resources.

## Authentication vs Authorization

- **Authentication**: Verifying who the user is (login credentials)
- **Authorization**: Determining what the authenticated user can do (permissions)

## Roles and Users

In PostgreSQL, users and groups are both represented as "roles". A role can be a user (can login) or a group (cannot login but can be granted to other roles).

### Creating Roles

```sql
-- Create a basic role (cannot login by default)
CREATE ROLE developer;

-- Create a user role (can login)
CREATE ROLE john_doe WITH LOGIN PASSWORD 'secure_password';

-- Create role with multiple attributes
CREATE ROLE admin_user WITH 
    LOGIN 
    PASSWORD 'admin_password'
    SUPERUSER 
    CREATEDB 
    CREATEROLE 
    VALID UNTIL '2025-12-31';

-- Create role with connection limit
CREATE ROLE limited_user WITH 
    LOGIN 
    PASSWORD 'user_password'
    CONNECTION LIMIT 5;

-- Create role that can create databases
CREATE ROLE db_creator WITH 
    LOGIN 
    PASSWORD 'creator_password'
    CREATEDB;
```

### Role Attributes

```sql
-- View role attributes
SELECT 
    rolname,
    rolsuper,
    rolinherit,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin,
    rolconnlimit,
    rolvaliduntil
FROM pg_roles;

-- Modify role attributes
ALTER ROLE john_doe WITH PASSWORD 'new_password';
ALTER ROLE john_doe WITH SUPERUSER;
ALTER ROLE john_doe WITH NOSUPERUSER;
ALTER ROLE john_doe WITH CREATEDB;
ALTER ROLE john_doe WITH NOCREATEDB;
ALTER ROLE john_doe WITH CREATEROLE;
ALTER ROLE john_doe WITH NOCREATEROLE;
ALTER ROLE john_doe WITH LOGIN;
ALTER ROLE john_doe WITH NOLOGIN;
ALTER ROLE john_doe WITH CONNECTION LIMIT 10;
ALTER ROLE john_doe WITH VALID UNTIL '2025-12-31';

-- Rename role
ALTER ROLE old_name RENAME TO new_name;
```

### Role Membership and Inheritance

```sql
-- Create group roles
CREATE ROLE developers;
CREATE ROLE managers;
CREATE ROLE read_only_users;

-- Grant role membership
GRANT developers TO john_doe;
GRANT managers TO jane_smith;
GRANT read_only_users TO guest_user;

-- Grant multiple roles
GRANT developers, read_only_users TO contractor;

-- Grant with admin option (can grant this role to others)
GRANT developers TO team_lead WITH ADMIN OPTION;

-- Revoke role membership
REVOKE developers FROM john_doe;

-- View role memberships
SELECT 
    r.rolname AS role_name,
    m.rolname AS member_name,
    g.rolname AS grantor_name,
    a.admin_option
FROM pg_auth_members a
JOIN pg_roles r ON r.oid = a.roleid
JOIN pg_roles m ON m.oid = a.member
JOIN pg_roles g ON g.oid = a.grantor
ORDER BY r.rolname, m.rolname;

-- Check current role and inherited roles
SELECT current_user, session_user;
SELECT * FROM pg_roles WHERE pg_has_role(current_user, oid, 'member');
```

### Role Inheritance

```sql
-- Create role with inheritance (default)
CREATE ROLE parent_role WITH INHERIT;

-- Create role without inheritance
CREATE ROLE child_role WITH NOINHERIT;

-- Switch to a role during session
SET ROLE developers;
SET ROLE NONE;  -- Switch back to original role

-- Reset to session user
RESET ROLE;
```

## Database and Schema Permissions

### Database-Level Permissions

```sql
-- Grant database permissions
GRANT CONNECT ON DATABASE mydb TO john_doe;
GRANT CREATE ON DATABASE mydb TO developers;
GRANT TEMPORARY ON DATABASE mydb TO temp_users;

-- Grant all database privileges
GRANT ALL PRIVILEGES ON DATABASE mydb TO admin_user;

-- Revoke database permissions
REVOKE CONNECT ON DATABASE mydb FROM guest_user;
REVOKE CREATE ON DATABASE mydb FROM developers;

-- View database permissions
SELECT 
    datname,
    datacl
FROM pg_database
WHERE datname = 'mydb';

-- Detailed database permissions
\dp  -- In psql
-- or
SELECT 
    grantee,
    privilege_type
FROM information_schema.database_privileges
WHERE catalog_name = 'mydb';
```

### Schema-Level Permissions

```sql
-- Grant schema permissions
GRANT USAGE ON SCHEMA public TO read_only_users;
GRANT CREATE ON SCHEMA public TO developers;
GRANT ALL ON SCHEMA public TO admin_user;

-- Create private schema
CREATE SCHEMA private_schema;
GRANT USAGE ON SCHEMA private_schema TO managers;
GRANT CREATE ON SCHEMA private_schema TO managers;

-- Revoke schema permissions
REVOKE CREATE ON SCHEMA public FROM developers;

-- View schema permissions
SELECT 
    schema_name,
    grantee,
    privilege_type
FROM information_schema.schema_privileges
WHERE schema_name = 'public';
```

## Table and Column Permissions

### Table-Level Permissions

```sql
-- Create sample table
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    salary DECIMAL(10,2),
    department VARCHAR(50),
    hire_date DATE
);

-- Grant table permissions
GRANT SELECT ON employees TO read_only_users;
GRANT INSERT, UPDATE, DELETE ON employees TO developers;
GRANT ALL PRIVILEGES ON employees TO managers;

-- Grant permissions on multiple tables
GRANT SELECT ON employees, departments TO read_only_users;

-- Grant permissions on all tables in schema
GRANT SELECT ON ALL TABLES IN SCHEMA public TO read_only_users;

-- Grant permissions on future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
GRANT SELECT ON TABLES TO read_only_users;

-- Revoke table permissions
REVOKE INSERT ON employees FROM developers;

-- View table permissions
\dp employees  -- In psql
-- or
SELECT 
    table_name,
    grantee,
    privilege_type,
    is_grantable
FROM information_schema.table_privileges
WHERE table_name = 'employees';
```

### Column-Level Permissions

```sql
-- Grant column-specific permissions
GRANT SELECT (first_name, last_name, email, department) ON employees TO hr_users;
GRANT UPDATE (email, department) ON employees TO hr_users;

-- Sensitive data access
GRANT SELECT (salary) ON employees TO managers;

-- Revoke column permissions
REVOKE SELECT (salary) ON employees FROM hr_users;

-- View column permissions
SELECT 
    table_name,
    column_name,
    grantee,
    privilege_type
FROM information_schema.column_privileges
WHERE table_name = 'employees';
```

### Sequence Permissions

```sql
-- Grant sequence permissions
GRANT USAGE ON SEQUENCE employees_employee_id_seq TO developers;
GRANT SELECT ON SEQUENCE employees_employee_id_seq TO read_only_users;

-- Grant on all sequences
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO developers;

-- Default privileges for sequences
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
GRANT USAGE ON SEQUENCES TO developers;
```

## Function and Procedure Permissions

```sql
-- Create sample function
CREATE OR REPLACE FUNCTION get_employee_count(dept_name TEXT)
RETURNS INTEGER
AS $$
BEGIN
    RETURN (SELECT COUNT(*) FROM employees WHERE department = dept_name);
END;
$$ LANGUAGE plpgsql;

-- Grant function execution
GRANT EXECUTE ON FUNCTION get_employee_count(TEXT) TO read_only_users;

-- Grant on all functions
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO developers;

-- Default privileges for functions
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
GRANT EXECUTE ON FUNCTIONS TO developers;

-- Security definer functions (run with creator's privileges)
CREATE OR REPLACE FUNCTION secure_salary_update(emp_id INTEGER, new_salary DECIMAL)
RETURNS BOOLEAN
SECURITY DEFINER
AS $$
BEGIN
    -- Only allow salary updates by managers
    IF NOT pg_has_role(current_user, 'managers', 'member') THEN
        RAISE EXCEPTION 'Access denied: Only managers can update salaries';
    END IF;
    
    UPDATE employees SET salary = new_salary WHERE employee_id = emp_id;
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Grant execution to all users (security handled inside function)
GRANT EXECUTE ON FUNCTION secure_salary_update(INTEGER, DECIMAL) TO PUBLIC;
```

## Row Level Security (RLS)

### Enabling RLS

```sql
-- Enable RLS on table
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- Create policy for users to see only their own records
CREATE POLICY employee_self_policy ON employees
    FOR ALL
    TO employees_role
    USING (email = current_user || '@company.com');

-- Create policy for managers to see all records
CREATE POLICY manager_all_policy ON employees
    FOR ALL
    TO managers
    USING (true);

-- Create policy for department access
CREATE POLICY department_policy ON employees
    FOR SELECT
    TO department_heads
    USING (department = current_setting('app.user_department'));

-- Create policy with different permissions
CREATE POLICY hr_policy ON employees
    FOR SELECT
    TO hr_users
    USING (department IN ('HR', 'Legal'));

CREATE POLICY hr_update_policy ON employees
    FOR UPDATE
    TO hr_users
    USING (department IN ('HR', 'Legal'))
    WITH CHECK (department IN ('HR', 'Legal'));
```

### Advanced RLS Examples

```sql
-- Create multi-tenant table
CREATE TABLE tenant_data (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER,
    data TEXT,
    created_by TEXT DEFAULT current_user
);

-- Enable RLS
ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

-- Create tenant isolation policy
CREATE POLICY tenant_isolation ON tenant_data
    FOR ALL
    TO tenant_users
    USING (tenant_id = current_setting('app.tenant_id')::INTEGER)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::INTEGER);

-- Create admin bypass policy
CREATE POLICY admin_bypass ON tenant_data
    FOR ALL
    TO admin_users
    USING (true);

-- Time-based access policy
CREATE POLICY business_hours_policy ON sensitive_table
    FOR ALL
    TO business_users
    USING (
        EXTRACT(hour FROM now()) BETWEEN 9 AND 17
        AND EXTRACT(dow FROM now()) BETWEEN 1 AND 5
    );

-- View RLS policies
SELECT 
    schemaname,
    tablename,
    policyname,
    permissive,
    roles,
    cmd,
    qual,
    with_check
FROM pg_policies
WHERE tablename = 'employees';
```

### Managing RLS

```sql
-- Disable RLS temporarily
ALTER TABLE employees DISABLE ROW LEVEL SECURITY;

-- Re-enable RLS
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- Force RLS for table owners
ALTER TABLE employees FORCE ROW LEVEL SECURITY;

-- Drop policy
DROP POLICY employee_self_policy ON employees;

-- Alter policy
ALTER POLICY manager_all_policy ON employees
    TO managers, senior_managers;

-- Check if RLS is enabled
SELECT 
    schemaname,
    tablename,
    rowsecurity,
    forcerowsecurity
FROM pg_tables
WHERE tablename = 'employees';
```

## Authentication Methods

### pg_hba.conf Configuration

```bash
# PostgreSQL Client Authentication Configuration File
# /var/lib/postgresql/data/pg_hba.conf

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections
local   all             postgres                                peer
local   all             all                                     md5

# IPv4 local connections
host    all             all             127.0.0.1/32            md5
host    all             all             192.168.1.0/24          md5

# IPv6 local connections
host    all             all             ::1/128                 md5

# Remote connections
host    all             all             0.0.0.0/0               md5

# SSL connections only
hostssl all             all             0.0.0.0/0               md5

# No SSL connections
hostnossl all           all             192.168.1.0/24          md5

# Specific database and user
host    mydb            john_doe        192.168.1.100/32        md5

# LDAP authentication
host    all             all             192.168.1.0/24          ldap ldapserver=ldap.company.com ldapbasedn="dc=company,dc=com" ldapbinddn="cn=admin,dc=company,dc=com" ldapbindpasswd="admin_password" ldapsearchattribute="uid"

# Certificate authentication
hostssl all             all             0.0.0.0/0               cert clientcert=1

# Kerberos authentication
host    all             all             192.168.1.0/24          gss include_realm=0 krb_realm=COMPANY.COM

# RADIUS authentication
host    all             all             192.168.1.0/24          radius radiusserver=radius.company.com radiussecret="shared_secret"

# PAM authentication
host    all             all             192.168.1.0/24          pam pamservice="postgresql"
```

### Authentication Methods Explained

- **trust**: Allow connection without password (use carefully)
- **reject**: Reject connection unconditionally
- **md5**: MD5-encrypted password authentication
- **scram-sha-256**: SCRAM-SHA-256 password authentication (recommended)
- **password**: Plain text password (not recommended)
- **peer**: Use OS user name (local connections only)
- **ident**: Use ident server to get OS user name
- **ldap**: LDAP authentication
- **cert**: SSL certificate authentication
- **gss**: GSSAPI authentication (Kerberos)
- **radius**: RADIUS authentication
- **pam**: PAM authentication

### SSL/TLS Configuration

```bash
# Generate SSL certificates
openssl req -new -x509 -days 365 -nodes -text -out server.crt -keyout server.key -subj "/CN=dbserver.company.com"
cp server.crt root.crt
chmod 600 server.key
chown postgres:postgres server.key server.crt root.crt

# Configure PostgreSQL for SSL
# In postgresql.conf:
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'
ssl_crl_file = ''
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
```

### Client Certificate Authentication

```bash
# Generate client certificate
openssl req -new -nodes -text -out client.csr -keyout client.key -subj "/CN=john_doe"
openssl x509 -req -in client.csr -text -days 365 -CA root.crt -CAkey server.key -CAcreateserial -out client.crt

# Configure pg_hba.conf for cert authentication
hostssl all john_doe 0.0.0.0/0 cert clientcert=1

# Connect with certificate
psql "host=dbserver.company.com dbname=mydb user=john_doe sslmode=require sslcert=client.crt sslkey=client.key sslrootcert=root.crt"
```

## Password Security

### Password Policies

```sql
-- Install passwordcheck extension (if available)
CREATE EXTENSION IF NOT EXISTS passwordcheck;

-- Set password encryption method
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- Create role with strong password
CREATE ROLE secure_user WITH 
    LOGIN 
    PASSWORD 'StrongP@ssw0rd123!'
    VALID UNTIL '2025-12-31';

-- Force password change on next login
ALTER ROLE user_name WITH PASSWORD 'temp_password' VALID UNTIL 'now';

-- Check password encryption method
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'secure_user';
```

### Password Management Function

```sql
-- Function to enforce password policy
CREATE OR REPLACE FUNCTION check_password_strength(password TEXT)
RETURNS BOOLEAN
AS $$
BEGIN
    -- Check minimum length
    IF length(password) < 8 THEN
        RAISE EXCEPTION 'Password must be at least 8 characters long';
    END IF;
    
    -- Check for uppercase letter
    IF password !~ '[A-Z]' THEN
        RAISE EXCEPTION 'Password must contain at least one uppercase letter';
    END IF;
    
    -- Check for lowercase letter
    IF password !~ '[a-z]' THEN
        RAISE EXCEPTION 'Password must contain at least one lowercase letter';
    END IF;
    
    -- Check for digit
    IF password !~ '[0-9]' THEN
        RAISE EXCEPTION 'Password must contain at least one digit';
    END IF;
    
    -- Check for special character
    IF password !~ '[!@#$%^&*(),.?":{}|<>]' THEN
        RAISE EXCEPTION 'Password must contain at least one special character';
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Function to create user with password validation
CREATE OR REPLACE FUNCTION create_user_with_strong_password(
    username TEXT,
    password TEXT
)
RETURNS BOOLEAN
AS $$
BEGIN
    -- Validate password
    PERFORM check_password_strength(password);
    
    -- Create user
    EXECUTE format('CREATE ROLE %I WITH LOGIN PASSWORD %L', username, password);
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT create_user_with_strong_password('new_user', 'StrongP@ssw0rd123!');
```

## Auditing and Logging

### Connection Logging

```bash
# Configure logging in postgresql.conf
log_connections = on
log_disconnections = on
log_hostname = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'all'  # or 'ddl', 'mod', 'none'
log_min_duration_statement = 1000  # Log queries taking > 1 second
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0
log_autovacuum_min_duration = 0
```

### Audit Trail Table

```sql
-- Create audit table
CREATE TABLE audit_log (
    audit_id SERIAL PRIMARY KEY,
    table_name TEXT,
    operation TEXT,
    user_name TEXT DEFAULT current_user,
    timestamp TIMESTAMP DEFAULT now(),
    old_values JSONB,
    new_values JSONB,
    client_addr INET DEFAULT inet_client_addr(),
    application_name TEXT DEFAULT current_setting('application_name')
);

-- Generic audit trigger function
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_values)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_values, new_values)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_values)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD));
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Apply audit trigger to table
CREATE TRIGGER employees_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_trigger_function();

-- Query audit log
SELECT 
    table_name,
    operation,
    user_name,
    timestamp,
    client_addr,
    application_name
FROM audit_log
WHERE timestamp >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY timestamp DESC;
```

### Login Monitoring

```sql
-- Create login tracking table
CREATE TABLE login_attempts (
    attempt_id SERIAL PRIMARY KEY,
    username TEXT,
    client_addr INET,
    success BOOLEAN,
    attempt_time TIMESTAMP DEFAULT now(),
    application_name TEXT
);

-- Function to log login attempts (requires custom authentication)
CREATE OR REPLACE FUNCTION log_login_attempt(
    p_username TEXT,
    p_success BOOLEAN,
    p_client_addr INET DEFAULT inet_client_addr(),
    p_application_name TEXT DEFAULT current_setting('application_name')
)
RETURNS VOID
AS $$
BEGIN
    INSERT INTO login_attempts (username, client_addr, success, application_name)
    VALUES (p_username, p_client_addr, p_success, p_application_name);
END;
$$ LANGUAGE plpgsql;

-- Monitor failed login attempts
SELECT 
    username,
    client_addr,
    COUNT(*) as failed_attempts,
    MAX(attempt_time) as last_attempt
FROM login_attempts
WHERE success = false
AND attempt_time >= CURRENT_DATE - INTERVAL '1 day'
GROUP BY username, client_addr
HAVING COUNT(*) >= 5
ORDER BY failed_attempts DESC;
```

## Security Best Practices

### Database Hardening

```sql
-- Remove unnecessary databases
DROP DATABASE IF EXISTS template0;
-- Note: Be very careful with this, template0 is usually needed

-- Remove public schema access
REVOKE ALL ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE mydb FROM PUBLIC;

-- Create application-specific schemas
CREATE SCHEMA app_schema;
GRANT USAGE ON SCHEMA app_schema TO app_users;
GRANT CREATE ON SCHEMA app_schema TO app_developers;

-- Set secure search path
ALTER ROLE app_user SET search_path = app_schema;

-- Disable dangerous functions for non-superusers
REVOKE EXECUTE ON FUNCTION pg_read_file(text) FROM PUBLIC;
REVOKE EXECUTE ON FUNCTION pg_ls_dir(text) FROM PUBLIC;
```

### Network Security

```bash
# Configure postgresql.conf for network security
listen_addresses = 'localhost,192.168.1.100'  # Specific IPs only
port = 5432  # Consider changing default port
max_connections = 100  # Limit connections

# SSL configuration
ssl = on
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'
ssl_prefer_server_ciphers = on
ssl_min_protocol_version = 'TLSv1.2'

# Connection security
tcp_keepalives_idle = 600
tcp_keepalives_interval = 30
tcp_keepalives_count = 3
```

### Application Security

```sql
-- Create application-specific roles
CREATE ROLE app_read_only WITH LOGIN PASSWORD 'read_password';
CREATE ROLE app_read_write WITH LOGIN PASSWORD 'write_password';
CREATE ROLE app_admin WITH LOGIN PASSWORD 'admin_password';

-- Grant minimal necessary permissions
GRANT CONNECT ON DATABASE mydb TO app_read_only, app_read_write, app_admin;
GRANT USAGE ON SCHEMA app_schema TO app_read_only, app_read_write, app_admin;

-- Read-only permissions
GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO app_read_only;
ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema 
GRANT SELECT ON TABLES TO app_read_only;

-- Read-write permissions
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app_schema TO app_read_write;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app_schema TO app_read_write;
ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema 
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_read_write;
ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema 
GRANT USAGE ON SEQUENCES TO app_read_write;

-- Admin permissions
GRANT ALL PRIVILEGES ON SCHEMA app_schema TO app_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA app_schema TO app_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA app_schema TO app_admin;
```

### SQL Injection Prevention

```sql
-- Use parameterized queries (application level)
-- BAD: SELECT * FROM users WHERE id = ' + user_input + ';
-- GOOD: SELECT * FROM users WHERE id = $1;

-- Use SECURITY DEFINER functions for complex operations
CREATE OR REPLACE FUNCTION safe_user_lookup(user_id INTEGER)
RETURNS TABLE(id INTEGER, name TEXT, email TEXT)
SECURITY DEFINER
AS $$
BEGIN
    -- Validate input
    IF user_id IS NULL OR user_id <= 0 THEN
        RAISE EXCEPTION 'Invalid user ID';
    END IF;
    
    RETURN QUERY
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.id = user_id;
END;
$$ LANGUAGE plpgsql;

-- Revoke direct table access, grant function access only
REVOKE ALL ON users FROM app_users;
GRANT EXECUTE ON FUNCTION safe_user_lookup(INTEGER) TO app_users;
```

## Monitoring Security

### Security Monitoring Queries

```sql
-- Check for users with dangerous privileges
SELECT 
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin
FROM pg_roles
WHERE rolsuper = true OR rolcreaterole = true
ORDER BY rolname;

-- Check for tables without RLS when it should be enabled
SELECT 
    schemaname,
    tablename,
    rowsecurity
FROM pg_tables
WHERE schemaname NOT IN ('information_schema', 'pg_catalog')
AND rowsecurity = false;

-- Check for overly permissive grants
SELECT 
    table_schema,
    table_name,
    grantee,
    privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'PUBLIC'
AND table_schema NOT IN ('information_schema', 'pg_catalog');

-- Check for users with no password expiration
SELECT 
    rolname,
    rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
AND (rolvaliduntil IS NULL OR rolvaliduntil > CURRENT_DATE + INTERVAL '1 year');

-- Check for unused roles
SELECT 
    r.rolname,
    r.rolcanlogin,
    COALESCE(s.last_login, 'Never') as last_login
FROM pg_roles r
LEFT JOIN (
    SELECT 
        usename,
        MAX(backend_start) as last_login
    FROM pg_stat_activity
    GROUP BY usename
) s ON r.rolname = s.usename
WHERE r.rolcanlogin = true
ORDER BY r.rolname;
```

### Security Alerts

```sql
-- Function to check for security violations
CREATE OR REPLACE FUNCTION security_check()
RETURNS TABLE(
    check_type TEXT,
    severity TEXT,
    description TEXT,
    recommendation TEXT
)
AS $$
BEGIN
    -- Check for superusers
    RETURN QUERY
    SELECT 
        'SUPERUSER_CHECK'::TEXT,
        'HIGH'::TEXT,
        'Found ' || COUNT(*)::TEXT || ' superuser accounts'::TEXT,
        'Review and minimize superuser accounts'::TEXT
    FROM pg_roles
    WHERE rolsuper = true AND rolname != 'postgres'
    HAVING COUNT(*) > 0;
    
    -- Check for password-less accounts
    RETURN QUERY
    SELECT 
        'PASSWORD_CHECK'::TEXT,
        'CRITICAL'::TEXT,
        'Found ' || COUNT(*)::TEXT || ' accounts without passwords'::TEXT,
        'Set passwords for all login accounts'::TEXT
    FROM pg_authid
    WHERE rolcanlogin = true AND rolpassword IS NULL
    HAVING COUNT(*) > 0;
    
    -- Check for public schema permissions
    RETURN QUERY
    SELECT 
        'PUBLIC_SCHEMA_CHECK'::TEXT,
        'MEDIUM'::TEXT,
        'Public schema has broad permissions'::TEXT,
        'Review and restrict public schema permissions'::TEXT
    WHERE EXISTS (
        SELECT 1 FROM information_schema.schema_privileges
        WHERE schema_name = 'public' AND grantee = 'PUBLIC'
    );
END;
$$ LANGUAGE plpgsql;

-- Run security check
SELECT * FROM security_check();
```

## Compliance and Regulations

### GDPR Compliance

```sql
-- Data anonymization function
CREATE OR REPLACE FUNCTION anonymize_personal_data()
RETURNS VOID
AS $$
BEGIN
    -- Anonymize email addresses
    UPDATE users 
    SET email = 'anonymized_' || user_id || '@example.com'
    WHERE consent_withdrawn = true;
    
    -- Anonymize names
    UPDATE users 
    SET first_name = 'Anonymous',
        last_name = 'User'
    WHERE consent_withdrawn = true;
    
    -- Log anonymization
    INSERT INTO audit_log (table_name, operation, new_values)
    VALUES ('users', 'ANONYMIZE', jsonb_build_object('anonymized_count', 
        (SELECT COUNT(*) FROM users WHERE consent_withdrawn = true)));
END;
$$ LANGUAGE plpgsql;

-- Data retention policy
CREATE OR REPLACE FUNCTION apply_data_retention()
RETURNS VOID
AS $$
BEGIN
    -- Delete old audit logs (keep for 7 years)
    DELETE FROM audit_log 
    WHERE timestamp < CURRENT_DATE - INTERVAL '7 years';
    
    -- Archive old user data
    INSERT INTO users_archive 
    SELECT * FROM users 
    WHERE last_login < CURRENT_DATE - INTERVAL '3 years';
    
    DELETE FROM users 
    WHERE last_login < CURRENT_DATE - INTERVAL '3 years';
END;
$$ LANGUAGE plpgsql;
```

### SOX Compliance

```sql
-- Segregation of duties
CREATE ROLE financial_read_only;
CREATE ROLE financial_data_entry;
CREATE ROLE financial_approver;
CREATE ROLE financial_auditor;

-- Financial data access controls
GRANT SELECT ON financial_transactions TO financial_read_only;
GRANT INSERT ON financial_transactions TO financial_data_entry;
GRANT UPDATE (approved_by, approval_date) ON financial_transactions TO financial_approver;
GRANT SELECT ON ALL TABLES IN SCHEMA financial TO financial_auditor;

-- Immutable audit trail
CREATE TABLE financial_audit (
    audit_id SERIAL PRIMARY KEY,
    transaction_id INTEGER,
    operation TEXT,
    user_name TEXT,
    timestamp TIMESTAMP DEFAULT now(),
    old_values JSONB,
    new_values JSONB
);

-- Prevent modifications to audit table
REVOKE ALL ON financial_audit FROM PUBLIC;
GRANT SELECT ON financial_audit TO financial_auditor;
```

## Troubleshooting Security Issues

### Common Security Problems

```sql
-- Check for locked accounts
SELECT 
    rolname,
    rolcanlogin,
    rolvaliduntil
FROM pg_roles
WHERE rolcanlogin = true
AND rolvaliduntil < now();

-- Check for connection issues
SELECT 
    datname,
    usename,
    client_addr,
    state,
    query_start,
    query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;

-- Check authentication failures in logs
-- grep "authentication failed" /var/log/postgresql/postgresql-*.log

-- Check SSL connection status
SELECT 
    pid,
    usename,
    client_addr,
    ssl,
    ssl_version,
    ssl_cipher
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE ssl = true;
```

### Emergency Procedures

```sql
-- Emergency: Disable a compromised account
ALTER ROLE compromised_user WITH NOLOGIN;

-- Emergency: Revoke all permissions from a role
REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM compromised_role;
REVOKE ALL PRIVILEGES ON SCHEMA public FROM compromised_role;
REVOKE CONNECT ON DATABASE mydb FROM compromised_role;

-- Emergency: Kill all connections from a specific IP
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE client_addr = '192.168.1.100';

-- Emergency: Change all user passwords
DO $$
DECLARE
    user_record RECORD;
BEGIN
    FOR user_record IN 
        SELECT rolname FROM pg_roles WHERE rolcanlogin = true AND rolname != 'postgres'
    LOOP
        EXECUTE format('ALTER ROLE %I WITH PASSWORD %L', 
                      user_record.rolname, 
                      'TempPassword_' || extract(epoch from now())::text);
    END LOOP;
END $$;
```

## Next Steps

After mastering security and user management:
1. Monitoring and Maintenance (12-monitoring-maintenance.md)
2. Advanced Administration (13-advanced-administration.md)
3. Performance Optimization (14-performance-optimization.md)

---
*This is part 11 of the PostgreSQL learning series*