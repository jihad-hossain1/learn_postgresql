# Stored Procedures and Functions in PostgreSQL

## Introduction

PostgreSQL supports user-defined functions and stored procedures written in various languages, with PL/pgSQL being the most common. These allow you to encapsulate business logic, improve performance, and maintain data integrity.

## Function vs Procedure

- **Functions**: Return a value, can be used in SELECT statements, cannot manage transactions
- **Procedures**: Don't return values, called with CALL, can manage transactions (COMMIT/ROLLBACK)

## Basic Functions

### Simple Functions

```sql
-- Simple function returning a value
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
AS $$
BEGIN
    RETURN a + b;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT add_numbers(5, 3); -- Returns 8

-- Function with default parameters
CREATE OR REPLACE FUNCTION calculate_tax(amount DECIMAL, tax_rate DECIMAL DEFAULT 0.08)
RETURNS DECIMAL
AS $$
BEGIN
    RETURN amount * tax_rate;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT calculate_tax(100);        -- Uses default tax rate
SELECT calculate_tax(100, 0.10);  -- Uses custom tax rate

-- Function with multiple return values
CREATE OR REPLACE FUNCTION get_name_parts(full_name TEXT)
RETURNS TABLE(first_name TEXT, last_name TEXT)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        split_part(full_name, ' ', 1) as first_name,
        split_part(full_name, ' ', 2) as last_name;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_name_parts('John Doe');
```

### Functions with Complex Logic

```sql
-- Function with conditional logic
CREATE OR REPLACE FUNCTION get_discount_rate(customer_type TEXT, order_amount DECIMAL)
RETURNS DECIMAL
AS $$
DECLARE
    discount_rate DECIMAL := 0;
BEGIN
    -- VIP customers get higher discounts
    IF customer_type = 'VIP' THEN
        IF order_amount >= 1000 THEN
            discount_rate := 0.15;
        ELSIF order_amount >= 500 THEN
            discount_rate := 0.10;
        ELSE
            discount_rate := 0.05;
        END IF;
    ELSIF customer_type = 'PREMIUM' THEN
        IF order_amount >= 500 THEN
            discount_rate := 0.08;
        ELSE
            discount_rate := 0.03;
        END IF;
    ELSE
        -- Regular customers
        IF order_amount >= 200 THEN
            discount_rate := 0.02;
        END IF;
    END IF;
    
    RETURN discount_rate;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT get_discount_rate('VIP', 1200);     -- Returns 0.15
SELECT get_discount_rate('REGULAR', 150);  -- Returns 0
```

### Functions with Loops

```sql
-- Function using FOR loop
CREATE OR REPLACE FUNCTION calculate_factorial(n INTEGER)
RETURNS BIGINT
AS $$
DECLARE
    result BIGINT := 1;
    i INTEGER;
BEGIN
    IF n < 0 THEN
        RAISE EXCEPTION 'Factorial is not defined for negative numbers';
    END IF;
    
    FOR i IN 1..n LOOP
        result := result * i;
    END LOOP;
    
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- Function using WHILE loop
CREATE OR REPLACE FUNCTION fibonacci(n INTEGER)
RETURNS INTEGER
AS $$
DECLARE
    a INTEGER := 0;
    b INTEGER := 1;
    temp INTEGER;
    counter INTEGER := 0;
BEGIN
    IF n <= 0 THEN
        RETURN 0;
    ELSIF n = 1 THEN
        RETURN 1;
    END IF;
    
    WHILE counter < n - 1 LOOP
        temp := a + b;
        a := b;
        b := temp;
        counter := counter + 1;
    END LOOP;
    
    RETURN b;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT calculate_factorial(5);  -- Returns 120
SELECT fibonacci(10);           -- Returns 55
```

## Working with Data

### Functions that Query Data

```sql
-- Sample tables
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE
);

CREATE TABLE departments (
    department_id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    budget DECIMAL(12,2)
);

-- Function to get employee count by department
CREATE OR REPLACE FUNCTION get_employee_count(dept_name TEXT)
RETURNS INTEGER
AS $$
DECLARE
    emp_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO emp_count
    FROM employees
    WHERE department = dept_name;
    
    RETURN emp_count;
END;
$$ LANGUAGE plpgsql;

-- Function returning a record type
CREATE TYPE employee_summary AS (
    full_name TEXT,
    department TEXT,
    years_employed INTEGER,
    salary_grade TEXT
);

CREATE OR REPLACE FUNCTION get_employee_summary(emp_id INTEGER)
RETURNS employee_summary
AS $$
DECLARE
    result employee_summary;
    emp_record RECORD;
BEGIN
    SELECT 
        first_name || ' ' || last_name as full_name,
        department,
        EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) as years_employed,
        salary
    INTO emp_record
    FROM employees
    WHERE employee_id = emp_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Employee with ID % not found', emp_id;
    END IF;
    
    result.full_name := emp_record.full_name;
    result.department := emp_record.department;
    result.years_employed := emp_record.years_employed;
    
    -- Determine salary grade
    IF emp_record.salary >= 100000 THEN
        result.salary_grade := 'Senior';
    ELSIF emp_record.salary >= 70000 THEN
        result.salary_grade := 'Mid-level';
    ELSE
        result.salary_grade := 'Junior';
    END IF;
    
    RETURN result;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_employee_summary(1);
```

### Functions that Return Tables

```sql
-- Function returning a table
CREATE OR REPLACE FUNCTION get_high_earners(min_salary DECIMAL)
RETURNS TABLE(
    employee_id INTEGER,
    full_name TEXT,
    department TEXT,
    salary DECIMAL,
    salary_rank INTEGER
)
AS $$
BEGIN
    RETURN QUERY
    SELECT 
        e.employee_id,
        e.first_name || ' ' || e.last_name as full_name,
        e.department,
        e.salary,
        RANK() OVER (ORDER BY e.salary DESC)::INTEGER as salary_rank
    FROM employees e
    WHERE e.salary >= min_salary
    ORDER BY e.salary DESC;
END;
$$ LANGUAGE plpgsql;

-- Function with dynamic SQL
CREATE OR REPLACE FUNCTION get_employees_by_criteria(
    sort_column TEXT DEFAULT 'last_name',
    sort_direction TEXT DEFAULT 'ASC',
    limit_count INTEGER DEFAULT 10
)
RETURNS TABLE(
    employee_id INTEGER,
    first_name TEXT,
    last_name TEXT,
    department TEXT,
    salary DECIMAL
)
AS $$
DECLARE
    query_text TEXT;
BEGIN
    -- Validate sort column
    IF sort_column NOT IN ('first_name', 'last_name', 'department', 'salary', 'hire_date') THEN
        RAISE EXCEPTION 'Invalid sort column: %', sort_column;
    END IF;
    
    -- Validate sort direction
    IF sort_direction NOT IN ('ASC', 'DESC') THEN
        RAISE EXCEPTION 'Invalid sort direction: %', sort_direction;
    END IF;
    
    query_text := format('
        SELECT employee_id, first_name, last_name, department, salary
        FROM employees
        ORDER BY %I %s
        LIMIT %s',
        sort_column, sort_direction, limit_count
    );
    
    RETURN QUERY EXECUTE query_text;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_high_earners(80000);
SELECT * FROM get_employees_by_criteria('salary', 'DESC', 5);
```

## Data Modification Functions

### Functions that Insert/Update Data

```sql
-- Function to create a new employee
CREATE OR REPLACE FUNCTION create_employee(
    p_first_name VARCHAR(50),
    p_last_name VARCHAR(50),
    p_department VARCHAR(50),
    p_salary DECIMAL(10,2)
)
RETURNS INTEGER
AS $$
DECLARE
    new_employee_id INTEGER;
BEGIN
    -- Validate salary
    IF p_salary <= 0 THEN
        RAISE EXCEPTION 'Salary must be positive';
    END IF;
    
    -- Insert new employee
    INSERT INTO employees (first_name, last_name, department, salary, hire_date)
    VALUES (p_first_name, p_last_name, p_department, p_salary, CURRENT_DATE)
    RETURNING employee_id INTO new_employee_id;
    
    -- Log the creation
    RAISE NOTICE 'Created employee % % with ID %', p_first_name, p_last_name, new_employee_id;
    
    RETURN new_employee_id;
END;
$$ LANGUAGE plpgsql;

-- Function to give salary raise
CREATE OR REPLACE FUNCTION give_salary_raise(
    emp_id INTEGER,
    raise_percentage DECIMAL
)
RETURNS DECIMAL
AS $$
DECLARE
    current_salary DECIMAL;
    new_salary DECIMAL;
BEGIN
    -- Get current salary
    SELECT salary INTO current_salary
    FROM employees
    WHERE employee_id = emp_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Employee with ID % not found', emp_id;
    END IF;
    
    -- Calculate new salary
    new_salary := current_salary * (1 + raise_percentage / 100);
    
    -- Update salary
    UPDATE employees
    SET salary = new_salary
    WHERE employee_id = emp_id;
    
    RAISE NOTICE 'Salary updated from % to % (%.2f%% increase)', 
                 current_salary, new_salary, raise_percentage;
    
    RETURN new_salary;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT create_employee('John', 'Doe', 'Engineering', 75000);
SELECT give_salary_raise(1, 10); -- 10% raise
```

## Stored Procedures

### Basic Procedures

```sql
-- Simple procedure
CREATE OR REPLACE PROCEDURE update_employee_department(
    emp_id INTEGER,
    new_dept VARCHAR(50)
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE employees
    SET department = new_dept
    WHERE employee_id = emp_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Employee with ID % not found', emp_id;
    END IF;
    
    COMMIT;
END;
$$;

-- Procedure with transaction management
CREATE OR REPLACE PROCEDURE process_bulk_salary_update(
    dept_name VARCHAR(50),
    raise_percentage DECIMAL
)
LANGUAGE plpgsql
AS $$
DECLARE
    affected_rows INTEGER;
BEGIN
    -- Start transaction
    BEGIN
        UPDATE employees
        SET salary = salary * (1 + raise_percentage / 100)
        WHERE department = dept_name;
        
        GET DIAGNOSTICS affected_rows = ROW_COUNT;
        
        IF affected_rows = 0 THEN
            RAISE EXCEPTION 'No employees found in department %', dept_name;
        END IF;
        
        RAISE NOTICE 'Updated salaries for % employees in % department', 
                     affected_rows, dept_name;
        
        COMMIT;
    EXCEPTION
        WHEN OTHERS THEN
            ROLLBACK;
            RAISE;
    END;
END;
$$;

-- Usage
CALL update_employee_department(1, 'Marketing');
CALL process_bulk_salary_update('Engineering', 5);
```

## Triggers and Trigger Functions

### Audit Trigger

```sql
-- Create audit table
CREATE TABLE employee_audit (
    audit_id SERIAL PRIMARY KEY,
    employee_id INTEGER,
    operation VARCHAR(10),
    old_values JSONB,
    new_values JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Trigger function for auditing
CREATE OR REPLACE FUNCTION audit_employee_changes()
RETURNS TRIGGER
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO employee_audit (employee_id, operation, new_values, changed_by)
        VALUES (NEW.employee_id, 'INSERT', to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO employee_audit (employee_id, operation, old_values, new_values, changed_by)
        VALUES (NEW.employee_id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), current_user);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO employee_audit (employee_id, operation, old_values, changed_by)
        VALUES (OLD.employee_id, 'DELETE', to_jsonb(OLD), current_user);
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER employee_audit_trigger
    AFTER INSERT OR UPDATE OR DELETE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION audit_employee_changes();
```

### Validation Trigger

```sql
-- Trigger function for data validation
CREATE OR REPLACE FUNCTION validate_employee_data()
RETURNS TRIGGER
AS $$
BEGIN
    -- Validate salary
    IF NEW.salary <= 0 THEN
        RAISE EXCEPTION 'Salary must be positive';
    END IF;
    
    -- Validate salary increase (max 50% at once)
    IF TG_OP = 'UPDATE' AND NEW.salary > OLD.salary * 1.5 THEN
        RAISE EXCEPTION 'Salary increase cannot exceed 50%% at once';
    END IF;
    
    -- Normalize names
    NEW.first_name := INITCAP(TRIM(NEW.first_name));
    NEW.last_name := INITCAP(TRIM(NEW.last_name));
    
    -- Set hire date if not provided
    IF NEW.hire_date IS NULL THEN
        NEW.hire_date := CURRENT_DATE;
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER validate_employee_trigger
    BEFORE INSERT OR UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION validate_employee_data();
```

### Automatic Timestamp Trigger

```sql
-- Add timestamp columns to employees table
ALTER TABLE employees 
ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
ADD COLUMN updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;

-- Trigger function to update timestamp
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER
AS $$
BEGIN
    NEW.updated_at := CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger
CREATE TRIGGER update_employee_timestamp
    BEFORE UPDATE ON employees
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

## Error Handling

### Exception Handling

```sql
-- Function with comprehensive error handling
CREATE OR REPLACE FUNCTION safe_divide(numerator DECIMAL, denominator DECIMAL)
RETURNS DECIMAL
AS $$
DECLARE
    result DECIMAL;
BEGIN
    -- Check for division by zero
    IF denominator = 0 THEN
        RAISE EXCEPTION 'Division by zero is not allowed';
    END IF;
    
    result := numerator / denominator;
    RETURN result;
    
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Caught division by zero error';
        RETURN NULL;
    WHEN OTHERS THEN
        RAISE NOTICE 'Unexpected error: %', SQLERRM;
        RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Function with custom exception handling
CREATE OR REPLACE FUNCTION transfer_employee(
    emp_id INTEGER,
    new_dept VARCHAR(50)
)
RETURNS BOOLEAN
AS $$
DECLARE
    emp_exists BOOLEAN;
    dept_exists BOOLEAN;
BEGIN
    -- Check if employee exists
    SELECT EXISTS(SELECT 1 FROM employees WHERE employee_id = emp_id) INTO emp_exists;
    
    IF NOT emp_exists THEN
        RAISE EXCEPTION 'EMPLOYEE_NOT_FOUND' USING MESSAGE = 'Employee does not exist';
    END IF;
    
    -- Check if department exists
    SELECT EXISTS(SELECT 1 FROM departments WHERE name = new_dept) INTO dept_exists;
    
    IF NOT dept_exists THEN
        RAISE EXCEPTION 'DEPARTMENT_NOT_FOUND' USING MESSAGE = 'Department does not exist';
    END IF;
    
    -- Perform transfer
    UPDATE employees SET department = new_dept WHERE employee_id = emp_id;
    
    RETURN TRUE;
    
EXCEPTION
    WHEN SQLSTATE 'P0001' THEN  -- Custom exception
        CASE SQLERRM
            WHEN 'Employee does not exist' THEN
                RAISE NOTICE 'Transfer failed: Employee ID % not found', emp_id;
            WHEN 'Department does not exist' THEN
                RAISE NOTICE 'Transfer failed: Department % not found', new_dept;
        END CASE;
        RETURN FALSE;
    WHEN OTHERS THEN
        RAISE NOTICE 'Transfer failed due to unexpected error: %', SQLERRM;
        RETURN FALSE;
END;
$$ LANGUAGE plpgsql;
```

## Advanced Function Features

### Variadic Functions

```sql
-- Function with variable number of arguments
CREATE OR REPLACE FUNCTION calculate_average(VARIADIC numbers DECIMAL[])
RETURNS DECIMAL
AS $$
DECLARE
    total DECIMAL := 0;
    count INTEGER := 0;
    num DECIMAL;
BEGIN
    IF array_length(numbers, 1) IS NULL THEN
        RAISE EXCEPTION 'At least one number is required';
    END IF;
    
    FOREACH num IN ARRAY numbers LOOP
        total := total + num;
        count := count + 1;
    END LOOP;
    
    RETURN total / count;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT calculate_average(10, 20, 30, 40);  -- Returns 25
SELECT calculate_average(VARIADIC ARRAY[1, 2, 3, 4, 5]);  -- Returns 3
```

### Functions with OUT Parameters

```sql
-- Function with OUT parameters
CREATE OR REPLACE FUNCTION get_employee_stats(
    dept_name VARCHAR(50),
    OUT total_employees INTEGER,
    OUT avg_salary DECIMAL,
    OUT min_salary DECIMAL,
    OUT max_salary DECIMAL
)
AS $$
BEGIN
    SELECT 
        COUNT(*),
        AVG(salary),
        MIN(salary),
        MAX(salary)
    INTO total_employees, avg_salary, min_salary, max_salary
    FROM employees
    WHERE department = dept_name;
    
    IF total_employees = 0 THEN
        RAISE EXCEPTION 'No employees found in department %', dept_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT * FROM get_employee_stats('Engineering');
```

### Polymorphic Functions

```sql
-- Function that works with any data type
CREATE OR REPLACE FUNCTION first_element(anyarray)
RETURNS anyelement
AS $$
BEGIN
    RETURN $1[1];
END;
$$ LANGUAGE plpgsql;

-- Usage
SELECT first_element(ARRAY[1, 2, 3]);           -- Returns 1
SELECT first_element(ARRAY['a', 'b', 'c']);     -- Returns 'a'
SELECT first_element(ARRAY[1.5, 2.5, 3.5]);     -- Returns 1.5
```

## Performance Considerations

### Function Volatility

```sql
-- IMMUTABLE: Always returns same result for same input
CREATE OR REPLACE FUNCTION calculate_circle_area(radius DECIMAL)
RETURNS DECIMAL
IMMUTABLE
AS $$
BEGIN
    RETURN PI() * radius * radius;
END;
$$ LANGUAGE plpgsql;

-- STABLE: Same result within a single statement
CREATE OR REPLACE FUNCTION get_current_date_string()
RETURNS TEXT
STABLE
AS $$
BEGIN
    RETURN CURRENT_DATE::TEXT;
END;
$$ LANGUAGE plpgsql;

-- VOLATILE: Result can change (default)
CREATE OR REPLACE FUNCTION get_random_employee()
RETURNS employees
VOLATILE
AS $$
DECLARE
    result employees;
BEGIN
    SELECT * INTO result
    FROM employees
    ORDER BY RANDOM()
    LIMIT 1;
    
    RETURN result;
END;
$$ LANGUAGE plpgsql;
```

### Caching and Optimization

```sql
-- Function with caching using temporary table
CREATE OR REPLACE FUNCTION get_department_summary_cached()
RETURNS TABLE(
    department TEXT,
    employee_count BIGINT,
    avg_salary DECIMAL,
    total_payroll DECIMAL
)
AS $$
BEGIN
    -- Check if cache table exists and is recent
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.tables 
        WHERE table_name = 'dept_summary_cache'
    ) OR (
        SELECT EXTRACT(EPOCH FROM (NOW() - created_at)) 
        FROM dept_summary_cache 
        LIMIT 1
    ) > 3600 THEN  -- Cache for 1 hour
        
        -- Recreate cache
        DROP TABLE IF EXISTS dept_summary_cache;
        
        CREATE TEMP TABLE dept_summary_cache AS
        SELECT 
            e.department,
            COUNT(*) as employee_count,
            AVG(e.salary) as avg_salary,
            SUM(e.salary) as total_payroll,
            NOW() as created_at
        FROM employees e
        GROUP BY e.department;
    END IF;
    
    RETURN QUERY
    SELECT 
        dsc.department,
        dsc.employee_count,
        dsc.avg_salary,
        dsc.total_payroll
    FROM dept_summary_cache dsc;
END;
$$ LANGUAGE plpgsql;
```

## Function Management

### Viewing Functions

```sql
-- List all user-defined functions
SELECT 
    n.nspname as schema_name,
    p.proname as function_name,
    pg_catalog.pg_get_function_result(p.oid) as return_type,
    pg_catalog.pg_get_function_arguments(p.oid) as arguments,
    CASE p.provolatile
        WHEN 'i' THEN 'IMMUTABLE'
        WHEN 's' THEN 'STABLE'
        WHEN 'v' THEN 'VOLATILE'
    END as volatility
FROM pg_catalog.pg_proc p
JOIN pg_catalog.pg_namespace n ON n.oid = p.pronamespace
WHERE n.nspname NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
AND p.prokind = 'f'  -- Functions only
ORDER BY schema_name, function_name;

-- List all procedures
SELECT 
    n.nspname as schema_name,
    p.proname as procedure_name,
    pg_catalog.pg_get_function_arguments(p.oid) as arguments
FROM pg_catalog.pg_proc p
JOIN pg_catalog.pg_namespace n ON n.oid = p.pronamespace
WHERE n.nspname NOT IN ('information_schema', 'pg_catalog', 'pg_toast')
AND p.prokind = 'p'  -- Procedures only
ORDER BY schema_name, procedure_name;

-- View function definition
SELECT pg_get_functiondef(oid)
FROM pg_proc
WHERE proname = 'your_function_name';
```

### Dropping Functions

```sql
-- Drop function (must specify parameter types if overloaded)
DROP FUNCTION IF EXISTS add_numbers(INTEGER, INTEGER);

-- Drop procedure
DROP PROCEDURE IF EXISTS update_employee_department(INTEGER, VARCHAR);

-- Drop function with CASCADE (removes dependent objects)
DROP FUNCTION calculate_tax(DECIMAL, DECIMAL) CASCADE;
```

## Best Practices

1. **Use appropriate volatility** - Mark functions as IMMUTABLE or STABLE when possible
2. **Handle exceptions properly** - Always include error handling for production functions
3. **Use meaningful names** - Function names should clearly indicate their purpose
4. **Document parameters** - Use comments to explain complex parameters
5. **Validate inputs** - Always validate function parameters
6. **Use transactions wisely** - Only procedures can manage transactions
7. **Consider security** - Use SECURITY DEFINER carefully
8. **Test thoroughly** - Write comprehensive tests for your functions
9. **Monitor performance** - Profile functions that process large datasets
10. **Version control** - Keep function definitions in version control

## Security Considerations

```sql
-- Function with SECURITY DEFINER (runs with creator's privileges)
CREATE OR REPLACE FUNCTION get_sensitive_data(user_id INTEGER)
RETURNS TABLE(data TEXT)
SECURITY DEFINER
AS $$
BEGIN
    -- Only allow users to see their own data
    IF user_id != current_setting('app.current_user_id')::INTEGER THEN
        RAISE EXCEPTION 'Access denied';
    END IF;
    
    RETURN QUERY
    SELECT sensitive_column
    FROM sensitive_table
    WHERE owner_id = user_id;
END;
$$ LANGUAGE plpgsql;

-- Grant execute permission
GRANT EXECUTE ON FUNCTION get_sensitive_data(INTEGER) TO app_users;
```

## Next Steps

After mastering functions and procedures:
1. Transactions and Concurrency (09-transactions-concurrency.md)
2. Backup and Recovery (10-backup-recovery.md)
3. Security and User Management (11-security-users.md)

---
*This is part 8 of the PostgreSQL learning series*