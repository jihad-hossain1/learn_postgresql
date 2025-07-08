# Basic SQL Operations in PostgreSQL

## Introduction to SQL

SQL (Structured Query Language) is the standard language for interacting with relational databases. PostgreSQL supports the SQL standard with additional extensions.

## SQL Statement Categories

- **DDL (Data Definition Language)**: CREATE, ALTER, DROP
- **DML (Data Manipulation Language)**: SELECT, INSERT, UPDATE, DELETE
- **DCL (Data Control Language)**: GRANT, REVOKE
- **TCL (Transaction Control Language)**: BEGIN, COMMIT, ROLLBACK

## Creating Databases and Tables

### Database Operations

```sql
-- Create a database
CREATE DATABASE company_db;

-- Connect to database
\c company_db

-- Drop a database (be careful!)
DROP DATABASE IF EXISTS old_database;

-- List all databases
\l
```

### Table Creation

```sql
-- Create a simple table
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    hire_date DATE DEFAULT CURRENT_DATE,
    salary DECIMAL(10,2),
    department_id INTEGER
);

-- Create table with constraints
CREATE TABLE departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    budget DECIMAL(12,2) CHECK (budget > 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create table with foreign key
CREATE TABLE projects (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    start_date DATE,
    end_date DATE,
    department_id INTEGER REFERENCES departments(id)
);
```

## Data Insertion (INSERT)

### Basic INSERT

```sql
-- Insert single row
INSERT INTO departments (name, budget) 
VALUES ('Engineering', 500000.00);

-- Insert multiple rows
INSERT INTO departments (name, budget) VALUES 
    ('Marketing', 200000.00),
    ('Sales', 300000.00),
    ('HR', 150000.00);

-- Insert with all columns specified
INSERT INTO employees (first_name, last_name, email, hire_date, salary, department_id)
VALUES ('John', 'Doe', 'john.doe@company.com', '2023-01-15', 75000.00, 1);

-- Insert with some columns (others use defaults)
INSERT INTO employees (first_name, last_name, email, salary, department_id)
VALUES 
    ('Jane', 'Smith', 'jane.smith@company.com', 80000.00, 1),
    ('Mike', 'Johnson', 'mike.johnson@company.com', 65000.00, 2),
    ('Sarah', 'Wilson', 'sarah.wilson@company.com', 70000.00, 3);
```

### INSERT with SELECT

```sql
-- Insert data from another table
INSERT INTO archived_employees 
SELECT * FROM employees 
WHERE hire_date < '2020-01-01';

-- Insert with transformation
INSERT INTO employee_summary (full_name, department, annual_salary)
SELECT 
    CONCAT(first_name, ' ', last_name),
    d.name,
    salary * 12
FROM employees e
JOIN departments d ON e.department_id = d.id;
```

## Data Retrieval (SELECT)

### Basic SELECT

```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, salary FROM employees;

-- Select with alias
SELECT 
    first_name AS "First Name",
    last_name AS "Last Name",
    salary AS "Annual Salary"
FROM employees;

-- Select with calculations
SELECT 
    first_name,
    last_name,
    salary,
    salary * 12 AS annual_salary,
    salary / 12 AS monthly_salary
FROM employees;
```

### Filtering with WHERE

```sql
-- Basic WHERE conditions
SELECT * FROM employees WHERE salary > 70000;

SELECT * FROM employees WHERE department_id = 1;

SELECT * FROM employees WHERE hire_date >= '2023-01-01';

-- Multiple conditions
SELECT * FROM employees 
WHERE salary > 60000 AND department_id = 1;

SELECT * FROM employees 
WHERE salary > 80000 OR department_id = 2;

-- Pattern matching
SELECT * FROM employees WHERE first_name LIKE 'J%';

SELECT * FROM employees WHERE email LIKE '%@company.com';

-- IN operator
SELECT * FROM employees WHERE department_id IN (1, 2, 3);

-- BETWEEN operator
SELECT * FROM employees WHERE salary BETWEEN 60000 AND 80000;

-- NULL checks
SELECT * FROM employees WHERE email IS NOT NULL;

SELECT * FROM employees WHERE department_id IS NULL;
```

### Sorting with ORDER BY

```sql
-- Sort ascending (default)
SELECT * FROM employees ORDER BY last_name;

-- Sort descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple column sorting
SELECT * FROM employees 
ORDER BY department_id ASC, salary DESC;

-- Sort by calculated column
SELECT first_name, last_name, salary * 12 AS annual_salary
FROM employees 
ORDER BY annual_salary DESC;
```

### Limiting Results

```sql
-- Get top 5 highest paid employees
SELECT * FROM employees 
ORDER BY salary DESC 
LIMIT 5;

-- Pagination with OFFSET
SELECT * FROM employees 
ORDER BY id 
LIMIT 10 OFFSET 20;  -- Skip first 20, get next 10

-- Get second page of results (page size 5)
SELECT * FROM employees 
ORDER BY last_name 
LIMIT 5 OFFSET 5;
```

## Data Updates (UPDATE)

```sql
-- Update single column
UPDATE employees 
SET salary = 85000 
WHERE id = 1;

-- Update multiple columns
UPDATE employees 
SET 
    salary = salary * 1.1,  -- 10% raise
    email = 'new.email@company.com'
WHERE department_id = 1;

-- Update with calculations
UPDATE employees 
SET salary = salary + 5000 
WHERE hire_date < '2022-01-01';

-- Update with subquery
UPDATE employees 
SET department_id = (
    SELECT id FROM departments WHERE name = 'Engineering'
)
WHERE first_name = 'John' AND last_name = 'Doe';

-- Conditional update
UPDATE employees 
SET salary = CASE 
    WHEN salary < 60000 THEN salary * 1.15
    WHEN salary < 80000 THEN salary * 1.10
    ELSE salary * 1.05
END;
```

## Data Deletion (DELETE)

```sql
-- Delete specific rows
DELETE FROM employees WHERE id = 5;

-- Delete with conditions
DELETE FROM employees WHERE salary < 50000;

-- Delete with subquery
DELETE FROM employees 
WHERE department_id IN (
    SELECT id FROM departments WHERE budget < 100000
);

-- Delete all rows (be very careful!)
DELETE FROM employees;  -- This deletes ALL data!

-- Safer way to delete all (with confirmation)
TRUNCATE TABLE employees;  -- Faster for large tables
```

## Joins

### INNER JOIN

```sql
-- Basic INNER JOIN
SELECT 
    e.first_name,
    e.last_name,
    e.salary,
    d.name AS department_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- Multiple table join
SELECT 
    e.first_name,
    e.last_name,
    d.name AS department,
    p.name AS project
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
INNER JOIN projects p ON d.id = p.department_id;
```

### LEFT JOIN

```sql
-- Include all employees, even those without departments
SELECT 
    e.first_name,
    e.last_name,
    COALESCE(d.name, 'No Department') AS department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

### RIGHT JOIN

```sql
-- Include all departments, even those without employees
SELECT 
    COALESCE(e.first_name, 'No Employee') AS first_name,
    d.name AS department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

### FULL OUTER JOIN

```sql
-- Include all employees and all departments
SELECT 
    COALESCE(e.first_name, 'No Employee') AS first_name,
    COALESCE(d.name, 'No Department') AS department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

## Aggregate Functions

```sql
-- Basic aggregates
SELECT 
    COUNT(*) AS total_employees,
    AVG(salary) AS average_salary,
    MIN(salary) AS minimum_salary,
    MAX(salary) AS maximum_salary,
    SUM(salary) AS total_payroll
FROM employees;

-- Group by department
SELECT 
    d.name AS department,
    COUNT(e.id) AS employee_count,
    AVG(e.salary) AS avg_salary,
    MAX(e.salary) AS max_salary
FROM employees e
JOIN departments d ON e.department_id = d.id
GROUP BY d.id, d.name
ORDER BY avg_salary DESC;

-- Having clause (filter groups)
SELECT 
    d.name AS department,
    COUNT(e.id) AS employee_count,
    AVG(e.salary) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.id
GROUP BY d.id, d.name
HAVING COUNT(e.id) > 2 AND AVG(e.salary) > 65000;
```

## Subqueries

```sql
-- Scalar subquery
SELECT first_name, last_name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- IN subquery
SELECT first_name, last_name
FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE budget > 200000
);

-- EXISTS subquery
SELECT d.name
FROM departments d
WHERE EXISTS (
    SELECT 1 FROM employees e WHERE e.department_id = d.id
);

-- Correlated subquery
SELECT 
    e1.first_name,
    e1.last_name,
    e1.salary,
    (
        SELECT COUNT(*) 
        FROM employees e2 
        WHERE e2.department_id = e1.department_id 
        AND e2.salary > e1.salary
    ) AS employees_with_higher_salary
FROM employees e1;
```

## Common Table Expressions (CTEs)

```sql
-- Basic CTE
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 70000
)
SELECT 
    he.first_name,
    he.last_name,
    d.name AS department
FROM high_earners he
JOIN departments d ON he.department_id = d.id;

-- Multiple CTEs
WITH 
dept_stats AS (
    SELECT 
        department_id,
        COUNT(*) AS emp_count,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),
high_performing_depts AS (
    SELECT department_id
    FROM dept_stats
    WHERE emp_count > 2 AND avg_salary > 65000
)
SELECT 
    e.first_name,
    e.last_name,
    e.salary
FROM employees e
JOIN high_performing_depts hpd ON e.department_id = hpd.department_id;
```

## Practical Examples

### Employee Management Queries

```sql
-- Find employees hired in the last 30 days
SELECT first_name, last_name, hire_date
FROM employees
WHERE hire_date >= CURRENT_DATE - INTERVAL '30 days';

-- Calculate years of service
SELECT 
    first_name,
    last_name,
    hire_date,
    EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date)) AS years_of_service
FROM employees;

-- Find duplicate emails
SELECT email, COUNT(*)
FROM employees
GROUP BY email
HAVING COUNT(*) > 1;

-- Rank employees by salary within department
SELECT 
    first_name,
    last_name,
    salary,
    department_id,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as salary_rank
FROM employees;
```

## Best Practices

1. **Always use WHERE clauses** when updating or deleting
2. **Use transactions** for multiple related operations
3. **Index frequently queried columns**
4. **Use LIMIT** when testing queries on large tables
5. **Avoid SELECT *** in production code
6. **Use meaningful aliases** for better readability
7. **Comment complex queries**
8. **Test queries on sample data** first

## Next Steps

After mastering basic SQL operations:
1. Data Types and Constraints (04-data-types-constraints.md)
2. Database Design (05-database-design.md)
3. Indexes and Performance (06-indexes-performance.md)

---
*This is part 3 of the PostgreSQL learning series*