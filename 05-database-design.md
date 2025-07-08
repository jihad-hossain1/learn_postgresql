# Database Design in PostgreSQL

## Introduction to Database Design

Database design is the process of creating a detailed data model of a database. Good design ensures data integrity, reduces redundancy, and optimizes performance.

## Database Design Process

1. **Requirements Analysis** - Understand what data needs to be stored
2. **Conceptual Design** - Create Entity-Relationship (ER) diagrams
3. **Logical Design** - Convert ER diagrams to relational schema
4. **Physical Design** - Optimize for performance and storage
5. **Implementation** - Create actual database objects
6. **Testing and Refinement** - Validate and optimize

## Entity-Relationship (ER) Modeling

### Entities and Attributes

```sql
-- Example: E-commerce System Entities

-- Customer Entity
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product Entity
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    cost DECIMAL(10,2) CHECK (cost >= 0),
    weight DECIMAL(8,3),
    dimensions JSONB, -- {"length": 10, "width": 5, "height": 3}
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Category Entity
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    parent_category_id INTEGER REFERENCES categories(category_id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Relationships

#### One-to-Many Relationships

```sql
-- Product belongs to Category (Many-to-One)
ALTER TABLE products 
ADD COLUMN category_id INTEGER REFERENCES categories(category_id);

-- Order belongs to Customer (Many-to-One)
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(customer_id),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
    total_amount DECIMAL(12,2) NOT NULL CHECK (total_amount >= 0),
    shipping_address JSONB,
    billing_address JSONB,
    notes TEXT
);
```

#### Many-to-Many Relationships

```sql
-- Order Items (Many-to-Many between Orders and Products)
CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
    discount_amount DECIMAL(10,2) DEFAULT 0 CHECK (discount_amount >= 0),
    UNIQUE(order_id, product_id)
);

-- Product Tags (Many-to-Many)
CREATE TABLE tags (
    tag_id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT
);

CREATE TABLE product_tags (
    product_id INTEGER REFERENCES products(product_id) ON DELETE CASCADE,
    tag_id INTEGER REFERENCES tags(tag_id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, tag_id)
);
```

#### One-to-One Relationships

```sql
-- Customer Profile (One-to-One with Customer)
CREATE TABLE customer_profiles (
    customer_id INTEGER PRIMARY KEY REFERENCES customers(customer_id) ON DELETE CASCADE,
    preferred_language VARCHAR(10) DEFAULT 'en',
    timezone VARCHAR(50) DEFAULT 'UTC',
    marketing_opt_in BOOLEAN DEFAULT FALSE,
    loyalty_points INTEGER DEFAULT 0,
    membership_level VARCHAR(20) DEFAULT 'bronze' 
        CHECK (membership_level IN ('bronze', 'silver', 'gold', 'platinum')),
    profile_picture_url TEXT,
    bio TEXT
);
```

## Normalization

### First Normal Form (1NF)

```sql
-- VIOLATION of 1NF - Multiple values in single column
CREATE TABLE bad_customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers VARCHAR(200)  -- "123-456-7890, 098-765-4321"
);

-- CORRECT 1NF - Atomic values
CREATE TABLE customers_1nf (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

CREATE TABLE customer_phones (
    phone_id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers_1nf(customer_id),
    phone_number VARCHAR(20),
    phone_type VARCHAR(10) CHECK (phone_type IN ('home', 'work', 'mobile'))
);
```

### Second Normal Form (2NF)

```sql
-- VIOLATION of 2NF - Partial dependency
CREATE TABLE bad_order_items (
    order_id INTEGER,
    product_id INTEGER,
    customer_name VARCHAR(100),  -- Depends only on order_id, not full key
    product_name VARCHAR(200),   -- Depends only on product_id, not full key
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id)
);

-- CORRECT 2NF - Remove partial dependencies
CREATE TABLE orders_2nf (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products_2nf (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2)
);

CREATE TABLE order_items_2nf (
    order_id INTEGER REFERENCES orders_2nf(order_id),
    product_id INTEGER REFERENCES products_2nf(product_id),
    quantity INTEGER,
    PRIMARY KEY (order_id, product_id)
);
```

### Third Normal Form (3NF)

```sql
-- VIOLATION of 3NF - Transitive dependency
CREATE TABLE bad_employees (
    employee_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER,
    department_name VARCHAR(100),  -- Depends on department_id, not employee_id
    department_budget DECIMAL(12,2) -- Depends on department_id, not employee_id
);

-- CORRECT 3NF - Remove transitive dependencies
CREATE TABLE departments (
    department_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    budget DECIMAL(12,2)
);

CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER REFERENCES departments(department_id)
);
```

### Boyce-Codd Normal Form (BCNF)

```sql
-- Example: Student-Course-Instructor relationship
-- Assumption: Each course-instructor combination is unique,
-- but an instructor can teach multiple courses

CREATE TABLE course_instructors (
    course_id INTEGER,
    instructor_id INTEGER,
    semester VARCHAR(20),
    PRIMARY KEY (course_id, instructor_id, semester)
);

CREATE TABLE courses (
    course_id SERIAL PRIMARY KEY,
    course_name VARCHAR(100),
    credits INTEGER
);

CREATE TABLE instructors (
    instructor_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INTEGER,
    course_id INTEGER,
    instructor_id INTEGER,
    semester VARCHAR(20),
    grade CHAR(2),
    PRIMARY KEY (student_id, course_id, semester),
    FOREIGN KEY (course_id, instructor_id, semester) 
        REFERENCES course_instructors(course_id, instructor_id, semester)
);
```

## Advanced Design Patterns

### Inheritance Patterns

#### Table Per Hierarchy (Single Table Inheritance)

```sql
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    user_type VARCHAR(20) NOT NULL CHECK (user_type IN ('customer', 'employee', 'admin')),
    email VARCHAR(100) UNIQUE NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    
    -- Customer specific fields
    loyalty_points INTEGER,
    membership_level VARCHAR(20),
    
    -- Employee specific fields
    employee_id VARCHAR(20),
    department_id INTEGER,
    salary DECIMAL(10,2),
    
    -- Admin specific fields
    admin_level INTEGER,
    permissions JSONB,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### Table Per Type (Class Table Inheritance)

```sql
-- Base table
CREATE TABLE users_base (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Specific tables
CREATE TABLE customers (
    user_id INTEGER PRIMARY KEY REFERENCES users_base(user_id) ON DELETE CASCADE,
    loyalty_points INTEGER DEFAULT 0,
    membership_level VARCHAR(20) DEFAULT 'bronze'
);

CREATE TABLE employees (
    user_id INTEGER PRIMARY KEY REFERENCES users_base(user_id) ON DELETE CASCADE,
    employee_id VARCHAR(20) UNIQUE,
    department_id INTEGER,
    salary DECIMAL(10,2),
    hire_date DATE
);

CREATE TABLE admins (
    user_id INTEGER PRIMARY KEY REFERENCES users_base(user_id) ON DELETE CASCADE,
    admin_level INTEGER,
    permissions JSONB
);
```

### Temporal Data Design

```sql
-- Slowly Changing Dimensions (SCD Type 2)
CREATE TABLE customer_history (
    history_id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    address JSONB,
    valid_from TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    valid_to TIMESTAMP,
    is_current BOOLEAN DEFAULT TRUE,
    
    CONSTRAINT chk_valid_period CHECK (valid_to IS NULL OR valid_to > valid_from)
);

-- Ensure only one current record per customer
CREATE UNIQUE INDEX idx_customer_current 
ON customer_history (customer_id) 
WHERE is_current = TRUE;

-- Audit trail pattern
CREATE TABLE product_audit (
    audit_id SERIAL PRIMARY KEY,
    product_id INTEGER NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB,
    changed_by INTEGER,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Hierarchical Data

#### Adjacency List Model

```sql
CREATE TABLE categories_adjacency (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INTEGER REFERENCES categories_adjacency(category_id),
    level INTEGER,
    sort_order INTEGER
);

-- Query to get all descendants (requires recursive CTE)
WITH RECURSIVE category_tree AS (
    -- Base case: root categories
    SELECT category_id, name, parent_id, 0 as depth, ARRAY[category_id] as path
    FROM categories_adjacency 
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: children
    SELECT c.category_id, c.name, c.parent_id, ct.depth + 1, ct.path || c.category_id
    FROM categories_adjacency c
    JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT * FROM category_tree ORDER BY path;
```

#### Nested Set Model

```sql
CREATE TABLE categories_nested (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    lft INTEGER NOT NULL,
    rgt INTEGER NOT NULL,
    level INTEGER NOT NULL,
    
    CONSTRAINT chk_nested_set CHECK (rgt > lft)
);

-- Query to get all descendants (much faster)
SELECT child.*
FROM categories_nested parent
JOIN categories_nested child ON child.lft BETWEEN parent.lft AND parent.rgt
WHERE parent.category_id = 1;
```

#### Materialized Path

```sql
CREATE TABLE categories_path (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    path VARCHAR(500) NOT NULL,  -- e.g., '/1/3/7/'
    level INTEGER NOT NULL
);

CREATE INDEX idx_categories_path ON categories_path USING GIST (path gist_trgm_ops);

-- Query descendants
SELECT * FROM categories_path 
WHERE path LIKE '/1/3/%';
```

## Schema Design Best Practices

### Naming Conventions

```sql
-- Table names: plural, lowercase with underscores
CREATE TABLE customer_orders (
    -- Primary keys: table_name + _id
    customer_order_id SERIAL PRIMARY KEY,
    
    -- Foreign keys: referenced_table + _id
    customer_id INTEGER REFERENCES customers(customer_id),
    
    -- Boolean columns: is_, has_, can_, should_
    is_paid BOOLEAN DEFAULT FALSE,
    has_discount BOOLEAN DEFAULT FALSE,
    
    -- Date/time columns: descriptive + _at or _date
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    shipped_date DATE,
    
    -- Constraints: type_table_column(s)
    CONSTRAINT chk_customer_orders_total CHECK (total_amount >= 0),
    CONSTRAINT uk_customer_orders_number UNIQUE (order_number)
);

-- Indexes: idx_table_column(s)
CREATE INDEX idx_customer_orders_customer_id ON customer_orders(customer_id);
CREATE INDEX idx_customer_orders_created_at ON customer_orders(created_at);
```

### Schema Organization

```sql
-- Create schemas for organization
CREATE SCHEMA sales;
CREATE SCHEMA inventory;
CREATE SCHEMA hr;
CREATE SCHEMA audit;

-- Move tables to appropriate schemas
CREATE TABLE sales.orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INTEGER,
    total_amount DECIMAL(12,2)
);

CREATE TABLE inventory.products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    stock_quantity INTEGER
);

CREATE TABLE hr.employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

-- Set search path
SET search_path = sales, inventory, public;
```

### Data Integrity Patterns

```sql
-- Soft delete pattern
CREATE TABLE products_soft_delete (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2),
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP,
    deleted_by INTEGER
);

-- Partial unique index for non-deleted records
CREATE UNIQUE INDEX idx_products_name_active 
ON products_soft_delete (name) 
WHERE is_deleted = FALSE;

-- Optimistic locking pattern
CREATE TABLE documents (
    document_id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    version INTEGER DEFAULT 1,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Update with version check
-- UPDATE documents 
-- SET content = 'new content', version = version + 1, updated_at = CURRENT_TIMESTAMP
-- WHERE document_id = 1 AND version = 3;
```

### Performance Considerations

```sql
-- Partitioning large tables
CREATE TABLE sales_data (
    sale_id SERIAL,
    sale_date DATE NOT NULL,
    customer_id INTEGER,
    amount DECIMAL(10,2),
    PRIMARY KEY (sale_id, sale_date)
) PARTITION BY RANGE (sale_date);

-- Create partitions
CREATE TABLE sales_data_2023 PARTITION OF sales_data
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE sales_data_2024 PARTITION OF sales_data
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Denormalization for read performance
CREATE TABLE order_summary (
    order_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(100),  -- Denormalized from customers table
    customer_email VARCHAR(100), -- Denormalized from customers table
    total_amount DECIMAL(12,2),
    item_count INTEGER,
    order_date TIMESTAMP
);
```

## Design Patterns for Specific Use Cases

### E-commerce Schema

```sql
-- Complete e-commerce schema example
CREATE SCHEMA ecommerce;
SET search_path = ecommerce, public;

-- Users and authentication
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Product catalog
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE,
    parent_id INTEGER REFERENCES categories(category_id)
);

CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    cost DECIMAL(10,2),
    sku VARCHAR(50) UNIQUE,
    category_id INTEGER REFERENCES categories(category_id),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Inventory management
CREATE TABLE inventory (
    product_id INTEGER PRIMARY KEY REFERENCES products(product_id),
    quantity_on_hand INTEGER DEFAULT 0,
    quantity_reserved INTEGER DEFAULT 0,
    reorder_level INTEGER DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Shopping cart
CREATE TABLE cart_items (
    user_id INTEGER REFERENCES users(user_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, product_id)
);

-- Orders
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(user_id),
    status VARCHAR(20) DEFAULT 'pending',
    subtotal DECIMAL(12,2),
    tax_amount DECIMAL(12,2),
    shipping_amount DECIMAL(12,2),
    total_amount DECIMAL(12,2),
    shipping_address JSONB,
    billing_address JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id),
    product_id INTEGER REFERENCES products(product_id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL
);
```

### Blog/CMS Schema

```sql
CREATE SCHEMA blog;
SET search_path = blog, public;

-- Authors
CREATE TABLE authors (
    author_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    bio TEXT,
    avatar_url TEXT
);

-- Categories and tags
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT
);

CREATE TABLE tags (
    tag_id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL
);

-- Posts
CREATE TABLE posts (
    post_id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    excerpt TEXT,
    content TEXT,
    author_id INTEGER REFERENCES authors(author_id),
    category_id INTEGER REFERENCES categories(category_id),
    status VARCHAR(20) DEFAULT 'draft' 
        CHECK (status IN ('draft', 'published', 'archived')),
    featured_image_url TEXT,
    published_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Post tags (many-to-many)
CREATE TABLE post_tags (
    post_id INTEGER REFERENCES posts(post_id) ON DELETE CASCADE,
    tag_id INTEGER REFERENCES tags(tag_id) ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- Comments
CREATE TABLE comments (
    comment_id SERIAL PRIMARY KEY,
    post_id INTEGER REFERENCES posts(post_id) ON DELETE CASCADE,
    parent_id INTEGER REFERENCES comments(comment_id),
    author_name VARCHAR(100),
    author_email VARCHAR(100),
    content TEXT NOT NULL,
    is_approved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Database Documentation

```sql
-- Add comments to document schema
COMMENT ON TABLE products IS 'Product catalog containing all sellable items';
COMMENT ON COLUMN products.sku IS 'Stock Keeping Unit - unique identifier for inventory';
COMMENT ON COLUMN products.price IS 'Selling price in USD';
COMMENT ON COLUMN products.cost IS 'Cost of goods sold in USD';

COMMENT ON TABLE orders IS 'Customer orders with status tracking';
COMMENT ON COLUMN orders.status IS 'Order status: pending, processing, shipped, delivered, cancelled';

-- View comments
SELECT 
    t.table_name,
    t.table_comment,
    c.column_name,
    c.column_comment
FROM information_schema.tables t
LEFT JOIN information_schema.columns c ON t.table_name = c.table_name
WHERE t.table_schema = 'public' AND t.table_comment IS NOT NULL;
```

## Migration Strategies

```sql
-- Version your schema changes
CREATE TABLE schema_migrations (
    version VARCHAR(20) PRIMARY KEY,
    description TEXT,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example migration script
-- Migration: 001_create_users_table.sql
BEGIN;

CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO schema_migrations (version, description) 
VALUES ('001', 'Create users table');

COMMIT;

-- Migration: 002_add_user_profile_fields.sql
BEGIN;

ALTER TABLE users 
ADD COLUMN first_name VARCHAR(50),
ADD COLUMN last_name VARCHAR(50),
ADD COLUMN phone VARCHAR(20);

INSERT INTO schema_migrations (version, description) 
VALUES ('002', 'Add user profile fields');

COMMIT;
```

## Design Review Checklist

### Data Integrity
- [ ] All tables have primary keys
- [ ] Foreign key relationships are properly defined
- [ ] Appropriate constraints are in place
- [ ] Data types are suitable for the data
- [ ] NULL/NOT NULL constraints are appropriate

### Performance
- [ ] Indexes are created for frequently queried columns
- [ ] Large tables are considered for partitioning
- [ ] Denormalization is used where appropriate
- [ ] Query patterns are considered in design

### Maintainability
- [ ] Consistent naming conventions
- [ ] Tables and columns are documented
- [ ] Schema is organized logically
- [ ] Migration strategy is in place

### Scalability
- [ ] Design can handle expected data growth
- [ ] Partitioning strategy for large tables
- [ ] Archival strategy for historical data
- [ ] Read replicas considered for read-heavy workloads

## Next Steps

After mastering database design:
1. Indexes and Performance (06-indexes-performance.md)
2. Advanced Queries (07-advanced-queries.md)
3. Stored Procedures and Functions (08-functions-procedures.md)

---
*This is part 5 of the PostgreSQL learning series*