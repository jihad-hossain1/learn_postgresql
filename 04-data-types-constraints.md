# Data Types and Constraints in PostgreSQL

## Introduction

PostgreSQL offers a rich set of data types and constraint mechanisms to ensure data integrity and optimize storage. Understanding these is crucial for effective database design.

## Numeric Data Types

### Integer Types

```sql
-- Integer types with their ranges
CREATE TABLE numeric_examples (
    small_int SMALLINT,      -- -32,768 to 32,767
    regular_int INTEGER,     -- -2,147,483,648 to 2,147,483,647
    big_int BIGINT,         -- -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807
    serial_id SERIAL,       -- Auto-incrementing integer (1 to 2,147,483,647)
    big_serial_id BIGSERIAL -- Auto-incrementing bigint
);

-- Examples
INSERT INTO numeric_examples (small_int, regular_int, big_int) VALUES
    (100, 1000000, 9000000000000000000),
    (-50, -500000, -1000000000000000);
```

### Decimal and Floating Point Types

```sql
CREATE TABLE decimal_examples (
    exact_decimal DECIMAL(10,2),    -- Exact: 10 digits total, 2 after decimal
    exact_numeric NUMERIC(15,4),    -- Same as DECIMAL
    float_real REAL,                -- 4-byte floating point
    float_double DOUBLE PRECISION,  -- 8-byte floating point
    money_amount MONEY              -- Currency amount
);

-- Examples
INSERT INTO decimal_examples VALUES
    (12345.67, 12345.6789, 123.456, 123.456789012345, '$1,234.56'),
    (99999999.99, 99999999999.9999, 1.23e10, 1.23e100, '$999,999.99');
```

## Character Data Types

```sql
CREATE TABLE text_examples (
    fixed_char CHAR(10),        -- Fixed length, padded with spaces
    variable_char VARCHAR(50),   -- Variable length with limit
    unlimited_text TEXT,         -- Unlimited length text
    single_char "char"          -- Single character (internal type)
);

-- Examples
INSERT INTO text_examples VALUES
    ('ABC', 'Hello World', 'This is a very long text that can be unlimited in length', 'A'),
    ('XYZ', 'Short', 'Another long text with special characters: àáâãäå', 'Z');

-- Character functions
SELECT 
    LENGTH(variable_char) as char_length,
    UPPER(variable_char) as uppercase,
    LOWER(variable_char) as lowercase,
    SUBSTRING(variable_char, 1, 5) as first_five_chars
FROM text_examples;
```

## Date and Time Data Types

```sql
CREATE TABLE datetime_examples (
    just_date DATE,                    -- Date only (YYYY-MM-DD)
    just_time TIME,                    -- Time only (HH:MM:SS)
    time_with_tz TIME WITH TIME ZONE,  -- Time with timezone
    timestamp_val TIMESTAMP,           -- Date and time
    timestamp_tz TIMESTAMPTZ,          -- Timestamp with timezone
    time_interval INTERVAL             -- Time interval
);

-- Examples
INSERT INTO datetime_examples VALUES
    ('2023-12-25', '14:30:00', '14:30:00+05:30', 
     '2023-12-25 14:30:00', '2023-12-25 14:30:00+05:30', '2 years 3 months 4 days'),
    ('2024-01-01', '09:15:30', '09:15:30-08:00',
     '2024-01-01 09:15:30', '2024-01-01 09:15:30-08:00', '1 hour 30 minutes');

-- Date/Time functions
SELECT 
    CURRENT_DATE as today,
    CURRENT_TIME as now_time,
    CURRENT_TIMESTAMP as now_timestamp,
    NOW() as now_function,
    EXTRACT(YEAR FROM just_date) as year_part,
    EXTRACT(MONTH FROM just_date) as month_part,
    AGE(just_date) as age_from_date
FROM datetime_examples;
```

## Boolean Data Type

```sql
CREATE TABLE boolean_examples (
    id SERIAL PRIMARY KEY,
    is_active BOOLEAN,
    is_verified BOOL  -- BOOL is alias for BOOLEAN
);

-- Boolean values
INSERT INTO boolean_examples (is_active, is_verified) VALUES
    (TRUE, FALSE),
    ('t', 'f'),        -- Alternative representations
    ('true', 'false'),
    ('yes', 'no'),
    ('1', '0'),
    (NULL, NULL);      -- NULL is also valid

-- Boolean operations
SELECT 
    is_active,
    is_verified,
    is_active AND is_verified as both_true,
    is_active OR is_verified as either_true,
    NOT is_active as negated
FROM boolean_examples;
```

## JSON Data Types

```sql
CREATE TABLE json_examples (
    id SERIAL PRIMARY KEY,
    data JSON,          -- JSON data (stored as text)
    data_binary JSONB   -- Binary JSON (more efficient)
);

-- JSON examples
INSERT INTO json_examples (data, data_binary) VALUES
    ('{"name": "John", "age": 30, "city": "New York"}',
     '{"name": "John", "age": 30, "city": "New York"}'),
    ('{"products": [{"id": 1, "name": "Laptop"}, {"id": 2, "name": "Mouse"}]}',
     '{"products": [{"id": 1, "name": "Laptop"}, {"id": 2, "name": "Mouse"}]}');

-- JSON operations
SELECT 
    data->>'name' as name,              -- Extract as text
    data->'age' as age_json,            -- Extract as JSON
    (data->>'age')::INTEGER as age_int, -- Cast to integer
    data_binary @> '{"age": 30}' as has_age_30,  -- Contains operator
    jsonb_array_length(data_binary->'products') as product_count
FROM json_examples;
```

## Array Data Types

```sql
CREATE TABLE array_examples (
    id SERIAL PRIMARY KEY,
    tags TEXT[],                    -- Array of text
    scores INTEGER[],               -- Array of integers
    matrix INTEGER[][],             -- Multi-dimensional array
    phone_numbers VARCHAR(15)[3]    -- Fixed size array
);

-- Array examples
INSERT INTO array_examples (tags, scores, matrix, phone_numbers) VALUES
    ('{"postgresql", "database", "sql"}', 
     '{95, 87, 92}',
     '{{1,2,3},{4,5,6}}',
     '{"123-456-7890", "098-765-4321"}'),
    (ARRAY['python', 'programming'],
     ARRAY[88, 91, 85],
     ARRAY[[7,8,9],[10,11,12]],
     ARRAY['555-1234', '555-5678', '555-9012']);

-- Array operations
SELECT 
    tags[1] as first_tag,           -- Access first element (1-indexed)
    array_length(tags, 1) as tag_count,
    'database' = ANY(tags) as has_database_tag,
    scores[2:3] as middle_scores,   -- Slice array
    array_append(tags, 'new_tag') as tags_with_new
FROM array_examples;
```

## UUID Data Type

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE uuid_examples (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL,
    session_token UUID
);

-- UUID examples
INSERT INTO uuid_examples (user_id, session_token) VALUES
    (uuid_generate_v4(), uuid_generate_v4()),
    ('550e8400-e29b-41d4-a716-446655440000', uuid_generate_v4());
```

## Geometric Data Types

```sql
CREATE TABLE geometric_examples (
    id SERIAL PRIMARY KEY,
    location POINT,         -- Point (x,y)
    area_box BOX,          -- Rectangle
    path_line PATH,        -- Path
    area_circle CIRCLE,    -- Circle
    area_polygon POLYGON   -- Polygon
);

-- Geometric examples
INSERT INTO geometric_examples (location, area_box, path_line, area_circle, area_polygon) VALUES
    ('(1,2)', '((0,0),(3,3))', '((0,0),(1,1),(2,0))', '<(0,0),5>', '((0,0),(0,3),(3,3),(3,0))'),
    ('(5,10)', '((1,1),(4,4))', '[(0,0),(1,1),(2,0)]', '<(1,1),3>', '((1,1),(1,4),(4,4),(4,1))');

-- Geometric operations
SELECT 
    location,
    location[0] as x_coordinate,
    location[1] as y_coordinate,
    area_box @> POINT(1,1) as box_contains_point,
    area(area_circle) as circle_area
FROM geometric_examples;
```

## Network Address Types

```sql
CREATE TABLE network_examples (
    id SERIAL PRIMARY KEY,
    ip_address INET,        -- IPv4 or IPv6 address
    network_cidr CIDR,      -- Network address
    mac_address MACADDR     -- MAC address
);

-- Network examples
INSERT INTO network_examples (ip_address, network_cidr, mac_address) VALUES
    ('192.168.1.1', '192.168.1.0/24', '08:00:2b:01:02:03'),
    ('2001:db8::1', '2001:db8::/32', '08-00-2b-01-02-03');

-- Network operations
SELECT 
    ip_address,
    host(ip_address) as host_part,
    network(ip_address) as network_part,
    broadcast(network_cidr) as broadcast_address
FROM network_examples;
```

## Constraints

### Primary Key Constraints

```sql
-- Single column primary key
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL
);

-- Composite primary key
CREATE TABLE order_items (
    order_id INTEGER,
    product_id INTEGER,
    quantity INTEGER NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Named primary key constraint
CREATE TABLE products (
    id INTEGER,
    name VARCHAR(100) NOT NULL,
    CONSTRAINT pk_products PRIMARY KEY (id)
);
```

### Foreign Key Constraints

```sql
-- Basic foreign key
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    order_date DATE DEFAULT CURRENT_DATE
);

-- Foreign key with actions
CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    street VARCHAR(200),
    city VARCHAR(100),
    FOREIGN KEY (user_id) REFERENCES users(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE
);

-- Named foreign key constraint
CREATE TABLE reviews (
    id SERIAL PRIMARY KEY,
    product_id INTEGER,
    user_id INTEGER,
    rating INTEGER,
    CONSTRAINT fk_reviews_product 
        FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_reviews_user 
        FOREIGN KEY (user_id) REFERENCES users(id)
);
```

### Unique Constraints

```sql
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE,              -- Column constraint
    employee_code VARCHAR(20),
    department_id INTEGER,
    UNIQUE (employee_code, department_id)   -- Table constraint
);

-- Named unique constraint
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    parent_id INTEGER,
    CONSTRAINT uk_category_name_parent UNIQUE (name, parent_id)
);
```

### Check Constraints

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) CHECK (price > 0),
    discount_percent INTEGER CHECK (discount_percent >= 0 AND discount_percent <= 100),
    category VARCHAR(50) CHECK (category IN ('Electronics', 'Clothing', 'Books', 'Home')),
    stock_quantity INTEGER DEFAULT 0,
    CONSTRAINT chk_stock_non_negative CHECK (stock_quantity >= 0)
);

-- Complex check constraints
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    start_date DATE,
    end_date DATE,
    max_participants INTEGER,
    current_participants INTEGER DEFAULT 0,
    CONSTRAINT chk_date_order CHECK (end_date >= start_date),
    CONSTRAINT chk_participants CHECK (current_participants <= max_participants)
);
```

### Not Null Constraints

```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    phone VARCHAR(20),  -- This can be NULL
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## Default Values

```sql
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    author_id INTEGER NOT NULL,
    status VARCHAR(20) DEFAULT 'draft',
    view_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_published BOOLEAN DEFAULT FALSE,
    tags TEXT[] DEFAULT '{}',  -- Empty array
    metadata JSONB DEFAULT '{}' -- Empty JSON object
);

-- Function-based defaults
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id INTEGER NOT NULL,
    expires_at TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL '24 hours'),
    created_at TIMESTAMP DEFAULT NOW()
);
```

## Custom Data Types

### Enumerated Types

```sql
-- Create enum type
CREATE TYPE order_status AS ENUM (
    'pending', 'processing', 'shipped', 'delivered', 'cancelled'
);

CREATE TYPE priority_level AS ENUM ('low', 'medium', 'high', 'urgent');

-- Use enum in table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    status order_status DEFAULT 'pending',
    priority priority_level DEFAULT 'medium',
    total_amount DECIMAL(10,2)
);

-- Insert with enum values
INSERT INTO orders (customer_id, status, priority, total_amount) VALUES
    (1, 'processing', 'high', 299.99),
    (2, 'pending', 'low', 49.99);
```

### Composite Types

```sql
-- Create composite type
CREATE TYPE address_type AS (
    street VARCHAR(200),
    city VARCHAR(100),
    state VARCHAR(50),
    zip_code VARCHAR(20),
    country VARCHAR(50)
);

-- Use composite type
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    headquarters address_type,
    billing_address address_type
);

-- Insert composite data
INSERT INTO companies (name, headquarters, billing_address) VALUES
    ('Tech Corp', 
     ROW('123 Tech St', 'San Francisco', 'CA', '94105', 'USA'),
     ROW('456 Billing Ave', 'New York', 'NY', '10001', 'USA'));

-- Query composite fields
SELECT 
    name,
    (headquarters).city as hq_city,
    (headquarters).state as hq_state
FROM companies;
```

### Domain Types

```sql
-- Create domain with constraints
CREATE DOMAIN email_address AS VARCHAR(320)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE DOMAIN positive_integer AS INTEGER
    CHECK (VALUE > 0);

CREATE DOMAIN percentage AS DECIMAL(5,2)
    CHECK (VALUE >= 0 AND VALUE <= 100);

-- Use domains
CREATE TABLE newsletter_subscribers (
    id SERIAL PRIMARY KEY,
    email email_address NOT NULL UNIQUE,
    age positive_integer,
    engagement_rate percentage
);
```

## Constraint Management

### Adding Constraints

```sql
-- Add constraints to existing table
ALTER TABLE products 
ADD CONSTRAINT chk_price_positive CHECK (price > 0);

ALTER TABLE orders 
ADD CONSTRAINT fk_orders_customer 
FOREIGN KEY (customer_id) REFERENCES customers(id);

ALTER TABLE users 
ADD CONSTRAINT uk_users_email UNIQUE (email);
```

### Dropping Constraints

```sql
-- Drop constraints
ALTER TABLE products DROP CONSTRAINT chk_price_positive;

ALTER TABLE orders DROP CONSTRAINT fk_orders_customer;

ALTER TABLE users DROP CONSTRAINT uk_users_email;
```

### Viewing Constraints

```sql
-- View table constraints
SELECT 
    tc.constraint_name,
    tc.constraint_type,
    tc.table_name,
    kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu 
    ON tc.constraint_name = kcu.constraint_name
WHERE tc.table_name = 'products';

-- View check constraints
SELECT 
    constraint_name,
    check_clause
FROM information_schema.check_constraints
WHERE constraint_schema = 'public';
```

## Best Practices

1. **Choose appropriate data types** to optimize storage and performance
2. **Use constraints** to enforce data integrity at the database level
3. **Name constraints explicitly** for better maintainability
4. **Use domains** for commonly used constrained types
5. **Consider using JSONB** over JSON for better performance
6. **Use arrays judiciously** - normalize when relationships are complex
7. **Set appropriate default values** to simplify application logic
8. **Use UUIDs** for distributed systems or when exposing IDs publicly

## Common Pitfalls

1. **Using VARCHAR without length** - use TEXT instead
2. **Not considering timezone** - use TIMESTAMPTZ for timestamps
3. **Overusing JSON** - normalize structured data when possible
4. **Ignoring constraint naming** - makes debugging difficult
5. **Using CHAR for variable-length data** - wastes space
6. **Not validating data types** - can lead to application errors

## Next Steps

After understanding data types and constraints:
1. Database Design (05-database-design.md)
2. Indexes and Performance (06-indexes-performance.md)
3. Advanced Queries (07-advanced-queries.md)

---
*This is part 4 of the PostgreSQL learning series*