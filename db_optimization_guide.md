# Complete Database Optimization Guide
## From Beginner to Advanced

### Table of Contents
1. [Introduction to Database Optimization](#introduction)
2. [Understanding Database Performance](#understanding-performance)
3. [Query Optimization Fundamentals](#query-optimization)
4. [Index Optimization](#index-optimization)
5. [Database Schema Design](#schema-design)
6. [Query Execution Plans](#execution-plans)
7. [Advanced Optimization Techniques](#advanced-techniques)
8. [Performance Monitoring](#performance-monitoring)
9. [Hardware and System Optimization](#hardware-optimization)
10. [Real-World Case Studies](#case-studies)
11. [Best Practices Checklist](#best-practices)
12. [Troubleshooting Common Issues](#troubleshooting)

---

## 1. Introduction to Database Optimization {#introduction}

### What is Database Optimization?
Database optimization is the process of improving database performance by:
- Reducing query execution time
- Minimizing resource usage (CPU, memory, disk I/O)
- Improving data retrieval speed
- Enhancing overall system responsiveness

### Why Database Optimization Matters
- **User Experience**: Faster response times improve user satisfaction
- **Cost Savings**: Efficient queries reduce server resources and costs
- **Scalability**: Optimized databases handle more users and data
- **Business Impact**: Slow databases can hurt revenue and productivity

### Key Performance Metrics
- **Query Response Time**: How long a query takes to execute
- **Throughput**: Number of operations per second
- **Resource Utilization**: CPU, memory, and disk usage
- **Concurrent Users**: Number of simultaneous database connections

---

## 2. Understanding Database Performance {#understanding-performance}

### Performance Bottlenecks

#### CPU Bottlenecks
```sql
-- Example: CPU-intensive query
SELECT customer_id, 
       COUNT(*) as order_count,
       SUM(total_amount) as total_spent,
       AVG(total_amount) as avg_order_value,
       -- Complex calculations that consume CPU
       SQRT(SUM(total_amount * total_amount)) as complex_calc
FROM orders 
WHERE order_date >= '2023-01-01'
GROUP BY customer_id
HAVING COUNT(*) > 100;
```

**Problem**: Complex calculations and aggregations consume CPU resources.

**Solution**: 
```sql
-- Optimized version: Pre-calculate complex metrics
-- Create a materialized view or summary table
CREATE TABLE customer_metrics AS
SELECT customer_id,
       COUNT(*) as order_count,
       SUM(total_amount) as total_spent,
       AVG(total_amount) as avg_order_value
FROM orders 
WHERE order_date >= '2023-01-01'
GROUP BY customer_id;

-- Simple query on pre-calculated data
SELECT * FROM customer_metrics WHERE order_count > 100;
```

#### Memory Bottlenecks
```sql
-- Example: Memory-intensive query
SELECT o.*, c.*, p.*, i.*
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items i ON o.order_id = i.order_id
JOIN products p ON i.product_id = p.product_id
WHERE o.order_date >= '2023-01-01';
```

**Problem**: Joining large tables without proper filtering loads too much data into memory.

**Solution**:
```sql
-- Optimized version: Filter early and select only needed columns
SELECT o.order_id, o.order_date, o.total_amount,
       c.customer_name, c.email,
       p.product_name, i.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items i ON o.order_id = i.order_id
JOIN products p ON i.product_id = p.product_id
WHERE o.order_date >= '2023-01-01'
  AND o.total_amount > 100;  -- Additional filtering
```

#### Disk I/O Bottlenecks
```sql
-- Example: High disk I/O query
SELECT * FROM large_table
WHERE unindexed_column = 'some_value';
```

**Problem**: Full table scan on large table without index.

**Solution**:
```sql
-- Create index to reduce disk I/O
CREATE INDEX idx_unindexed_column ON large_table(unindexed_column);

-- Query now uses index seek instead of table scan
SELECT * FROM large_table
WHERE unindexed_column = 'some_value';
```

### Understanding Query Execution

#### How Databases Execute Queries
1. **Parsing**: SQL statement is parsed and validated
2. **Optimization**: Database chooses the best execution plan
3. **Execution**: Query is executed using the chosen plan
4. **Result Return**: Results are formatted and returned

#### Example: Query Execution Steps
```sql
-- Original query
SELECT c.customer_name, COUNT(o.order_id) as order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.registration_date >= '2023-01-01'
GROUP BY c.customer_id, c.customer_name
ORDER BY order_count DESC;
```

**Execution Steps**:
1. **Table Scan**: Read customers table
2. **Filter**: Apply WHERE clause on registration_date
3. **Join**: LEFT JOIN with orders table
4. **Group**: GROUP BY customer_id and customer_name
5. **Aggregate**: COUNT order_id
6. **Sort**: ORDER BY order_count DESC
7. **Return**: Send results to client

---

## 3. Query Optimization Fundamentals {#query-optimization}

### 3.1 Writing Efficient SELECT Statements

#### Avoid SELECT *
```sql
-- Bad: Selects all columns
SELECT * FROM employees WHERE department_id = 1;

-- Good: Select only needed columns
SELECT employee_id, first_name, last_name, salary 
FROM employees WHERE department_id = 1;
```

**Impact**: Reduces network traffic and memory usage by up to 80%.

#### Use Appropriate WHERE Clauses
```sql
-- Bad: Non-selective WHERE clause
SELECT * FROM orders WHERE order_date >= '2020-01-01';

-- Good: More selective WHERE clause
SELECT * FROM orders 
WHERE order_date >= '2023-01-01' 
  AND status = 'completed'
  AND total_amount > 100;
```

#### Optimize LIKE Queries
```sql
-- Bad: Leading wildcard prevents index usage
SELECT * FROM customers WHERE customer_name LIKE '%John%';

-- Good: Trailing wildcard can use index
SELECT * FROM customers WHERE customer_name LIKE 'John%';

-- Best: Use full-text search for complex text searches
SELECT * FROM customers WHERE MATCH(customer_name) AGAINST('John' IN NATURAL LANGUAGE MODE);
```

### 3.2 JOIN Optimization

#### Understanding JOIN Types and Performance

**INNER JOIN Example**:
```sql
-- Efficient INNER JOIN
SELECT c.customer_name, o.order_date, o.total_amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_type = 'premium'
  AND o.order_date >= '2023-01-01';
```

**LEFT JOIN Optimization**:
```sql
-- Bad: LEFT JOIN with WHERE on right table
SELECT c.customer_name, o.order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2023-01-01';  -- This turns it into INNER JOIN

-- Good: Move condition to JOIN clause or use INNER JOIN
SELECT c.customer_name, o.order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id 
                   AND o.order_date >= '2023-01-01';
```

#### JOIN Order Optimization
```sql
-- Database optimizes JOIN order, but you can help:

-- Good: Start with most selective table
SELECT p.product_name, s.supplier_name, c.category_name
FROM products p
JOIN suppliers s ON p.supplier_id = s.supplier_id
JOIN categories c ON p.category_id = c.category_id
WHERE p.price > 1000  -- Most selective condition first
  AND s.country = 'USA';
```

### 3.3 Subquery Optimization

#### Converting Subqueries to JOINs
```sql
-- Bad: Correlated subquery
SELECT customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
      AND o.order_date >= '2023-01-01'
);

-- Good: Equivalent JOIN (often faster)
SELECT DISTINCT c.customer_name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2023-01-01';
```

#### Optimizing IN Subqueries
```sql
-- Bad: IN with large subquery
SELECT * FROM products
WHERE category_id IN (
    SELECT category_id FROM categories 
    WHERE category_name LIKE '%electronics%'
);

-- Good: Use JOIN instead
SELECT p.*
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
WHERE c.category_name LIKE '%electronics%';
```

### 3.4 Aggregate Function Optimization

#### Efficient Grouping
```sql
-- Bad: Grouping on unindexed columns
SELECT region, COUNT(*) as customer_count
FROM customers
GROUP BY region;

-- Good: Create index on grouping column
CREATE INDEX idx_customers_region ON customers(region);

-- Now the query uses the index for grouping
SELECT region, COUNT(*) as customer_count
FROM customers
GROUP BY region;
```

#### Optimizing HAVING Clauses
```sql
-- Bad: HAVING with complex conditions
SELECT customer_id, COUNT(*) as order_count
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 10 AND SUM(total_amount) > 1000;

-- Good: Pre-filter with WHERE when possible
SELECT customer_id, COUNT(*) as order_count
FROM orders
WHERE total_amount > 50  -- Pre-filter to reduce groups
GROUP BY customer_id
HAVING COUNT(*) > 10;
```

---

## 4. Index Optimization {#index-optimization}

### 4.1 Understanding Indexes

#### What Are Indexes?
Indexes are data structures that improve query performance by providing fast access paths to table data.

**Analogy**: Like a book's index, database indexes help find information quickly without scanning every page.

#### Types of Indexes

**1. Clustered Index**
```sql
-- Primary key automatically creates clustered index
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,  -- Clustered index
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    salary DECIMAL(10,2)
);
```

**2. Non-Clustered Index**
```sql
-- Create non-clustered index on frequently queried columns
CREATE INDEX idx_employees_lastname ON employees(last_name);
CREATE INDEX idx_employees_salary ON employees(salary);
```

**3. Composite Index**
```sql
-- Multi-column index for queries filtering on multiple columns
CREATE INDEX idx_employees_dept_salary ON employees(department_id, salary);

-- This index helps queries like:
SELECT * FROM employees WHERE department_id = 1 AND salary > 50000;
```

### 4.2 Index Design Strategies

#### Single Column Indexes
```sql
-- Create indexes on frequently filtered columns
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_status ON orders(status);

-- These indexes help queries like:
SELECT * FROM orders WHERE customer_id = 123;
SELECT * FROM orders WHERE order_date >= '2023-01-01';
SELECT * FROM orders WHERE status = 'shipped';
```

#### Composite Index Design
```sql
-- Rule: Most selective column first
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- This index helps both queries:
SELECT * FROM orders WHERE status = 'pending';  -- Uses index
SELECT * FROM orders WHERE status = 'pending' AND order_date >= '2023-01-01';  -- Uses index fully
```

#### Index Column Order
```sql
-- Sample data scenario:
-- orders table with 1M records
-- status values: 'pending' (1000), 'shipped' (800K), 'delivered' (199K)
-- order_date: evenly distributed over 2 years

-- Good: More selective column first
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- This works well for:
SELECT * FROM orders WHERE status = 'pending' AND order_date >= '2023-01-01';
-- (Filters down to 1000 records first, then applies date filter)
```

### 4.3 Index Maintenance

#### Monitoring Index Usage
```sql
-- Check index usage (MySQL)
SELECT 
    table_name,
    index_name,
    column_name,
    seq_in_index,
    cardinality
FROM information_schema.statistics
WHERE table_schema = 'your_database'
ORDER BY table_name, index_name, seq_in_index;
```

#### Removing Unused Indexes
```sql
-- Find unused indexes (PostgreSQL example)
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY tablename;

-- Drop unused indexes
DROP INDEX idx_unused_column;
```

### 4.4 Index Optimization Examples

#### Example 1: E-commerce Order System
```sql
-- Sample table
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    status VARCHAR(20),
    total_amount DECIMAL(10,2),
    shipping_address_id INT
);

-- Common queries and their optimal indexes:

-- Query 1: Find orders by customer
SELECT * FROM orders WHERE customer_id = 123;
-- Index: CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Query 2: Find recent orders
SELECT * FROM orders WHERE order_date >= '2023-01-01' ORDER BY order_date DESC;
-- Index: CREATE INDEX idx_orders_date ON orders(order_date);

-- Query 3: Find pending orders for a customer
SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending';
-- Index: CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- Query 4: Find high-value orders
SELECT * FROM orders WHERE total_amount > 1000 ORDER BY total_amount DESC;
-- Index: CREATE INDEX idx_orders_amount ON orders(total_amount);
```

#### Example 2: User Activity Tracking
```sql
-- Sample table
CREATE TABLE user_activities (
    activity_id INT PRIMARY KEY,
    user_id INT,
    activity_type VARCHAR(50),
    activity_date DATETIME,
    ip_address VARCHAR(45),
    user_agent TEXT
);

-- Optimization for common queries:

-- Query: Recent activities by user
SELECT * FROM user_activities 
WHERE user_id = 123 AND activity_date >= '2023-01-01'
ORDER BY activity_date DESC;

-- Optimal index:
CREATE INDEX idx_user_activities_user_date ON user_activities(user_id, activity_date);

-- Query: Activities by type in date range
SELECT COUNT(*) FROM user_activities 
WHERE activity_type = 'login' AND activity_date >= '2023-01-01';

-- Optimal index:
CREATE INDEX idx_user_activities_type_date ON user_activities(activity_type, activity_date);
```

---

## 5. Database Schema Design {#schema-design}

### 5.1 Normalization for Performance

#### Understanding Normal Forms

**First Normal Form (1NF)**
```sql
-- Bad: Repeating groups
CREATE TABLE orders_bad (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    product1 VARCHAR(100),
    product2 VARCHAR(100),
    product3 VARCHAR(100)
);

-- Good: Atomic values
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Second Normal Form (2NF)**
```sql
-- Bad: Partial dependencies
CREATE TABLE order_items_bad (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),  -- Depends only on product_id
    product_price DECIMAL(10,2), -- Depends only on product_id
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- Good: Remove partial dependencies
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    product_price DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**Third Normal Form (3NF)**
```sql
-- Bad: Transitive dependencies
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),  -- Depends on department_id
    department_location VARCHAR(100) -- Depends on department_id
);

-- Good: Remove transitive dependencies
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    department_location VARCHAR(100)
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

### 5.2 Denormalization for Performance

#### When to Denormalize
```sql
-- Scenario: Frequently accessed customer order summary
-- Normalized approach requires expensive JOIN:
SELECT c.customer_name, COUNT(o.order_id) as order_count, SUM(o.total_amount) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name;

-- Denormalized approach: Add summary columns
ALTER TABLE customers ADD COLUMN order_count INT DEFAULT 0;
ALTER TABLE customers ADD COLUMN total_spent DECIMAL(10,2) DEFAULT 0;

-- Update summary columns with triggers or scheduled jobs
CREATE TRIGGER update_customer_summary
AFTER INSERT ON orders
FOR EACH ROW
BEGIN
    UPDATE customers 
    SET order_count = order_count + 1,
        total_spent = total_spent + NEW.total_amount
    WHERE customer_id = NEW.customer_id;
END;

-- Now the query is much faster:
SELECT customer_name, order_count, total_spent FROM customers;
```

### 5.3 Partitioning Strategies

#### Horizontal Partitioning (Sharding)
```sql
-- Partition large tables by date
CREATE TABLE orders_2023 (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    CHECK (order_date >= '2023-01-01' AND order_date < '2024-01-01')
);

CREATE TABLE orders_2024 (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    CHECK (order_date >= '2024-01-01' AND order_date < '2025-01-01')
);

-- Create a view for transparent access
CREATE VIEW orders AS
SELECT * FROM orders_2023
UNION ALL
SELECT * FROM orders_2024;
```

#### Vertical Partitioning
```sql
-- Split wide tables into frequently and rarely accessed columns
CREATE TABLE users_core (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at DATETIME
);

CREATE TABLE users_profile (
    user_id INT PRIMARY KEY,
    bio TEXT,
    profile_picture BLOB,
    last_login DATETIME,
    FOREIGN KEY (user_id) REFERENCES users_core(user_id)
);

-- Most queries only need core information
SELECT username, email FROM users_core WHERE user_id = 123;

-- Profile information loaded only when needed
SELECT * FROM users_profile WHERE user_id = 123;
```

---

## 6. Query Execution Plans {#execution-plans}

### 6.1 Understanding Execution Plans

#### What is an Execution Plan?
An execution plan shows how the database engine will execute a query, including:
- Which indexes to use
- Join methods
- Sort operations
- Estimated costs and row counts

#### Reading Execution Plans (MySQL Example)
```sql
-- Use EXPLAIN to see execution plan
EXPLAIN SELECT c.customer_name, o.order_date, o.total_amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2023-01-01'
ORDER BY o.total_amount DESC;
```

**Sample Output**:
```
+----+-------------+-------+-------+---------------+---------+---------+-------+------+-------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref   | rows | Extra       |
+----+-------------+-------+-------+---------------+---------+---------+-------+------+-------------+
|  1 | SIMPLE      | o     | range | idx_date,fk_cust | idx_date | 3    | NULL  | 1000 | Using where |
|  1 | SIMPLE      | c     | eq_ref| PRIMARY       | PRIMARY | 4       | o.customer_id | 1 | NULL |
+----+-------------+-------+-------+---------------+---------+---------+-------+------+-------------+
```

#### Key Elements to Look For:
- **type**: Access method (const, eq_ref, ref, range, index, ALL)
- **key**: Which index is used
- **rows**: Estimated number of rows examined
- **Extra**: Additional information (Using where, Using filesort, etc.)

### 6.2 Optimizing Based on Execution Plans

#### Example 1: Optimizing a Slow Query
```sql
-- Original slow query
SELECT p.product_name, c.category_name, COUNT(oi.order_id) as order_count
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_date >= '2023-01-01'
GROUP BY p.product_id, p.product_name, c.category_name
ORDER BY order_count DESC;

-- Check execution plan
EXPLAIN SELECT /* ... query above ... */;
```

**Problems Found in Plan**:
1. Full table scan on orders table
2. Using filesort for ORDER BY
3. High row count estimates

**Optimization Steps**:
```sql
-- Step 1: Create index on order_date
CREATE INDEX idx_orders_date ON orders(order_date);

-- Step 2: Create composite index for better JOIN performance
CREATE INDEX idx_order_items_product_order ON order_items(product_id, order_id);

-- Step 3: Rewrite query to be more efficient
SELECT p.product_name, c.category_name, COUNT(*) as order_count
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories c ON p.category_id = c.category_id
WHERE o.order_date >= '2023-01-01'
GROUP BY p.product_id, p.product_name, c.category_name
ORDER BY order_count DESC;
```

### 6.3 Common Execution Plan Patterns

#### Index Seek vs Index Scan vs Table Scan
```sql
-- Index Seek (Best)
SELECT * FROM customers WHERE customer_id = 123;
-- Uses: PRIMARY KEY index, examines 1 row

-- Index Scan (Good)
SELECT * FROM customers WHERE customer_id BETWEEN 100 AND 200;
-- Uses: PRIMARY KEY index, examines 100 rows

-- Table Scan (Worst)
SELECT * FROM customers WHERE customer_name LIKE '%John%';
-- No index can be used, examines all rows
```

#### Nested Loop vs Hash Join vs Merge Join
```sql
-- Query that might use different join methods
SELECT c.customer_name, o.total_amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_type = 'premium';

-- Nested Loop Join: Good for small result sets
-- Hash Join: Good for large result sets with no sorted order
-- Merge Join: Good for large result sets that are already sorted
```

---

## 7. Advanced Optimization Techniques {#advanced-techniques}

### 7.1 Query Rewriting Techniques

#### Exists vs IN vs JOIN
```sql
-- Scenario: Find customers who have placed orders

-- Method 1: EXISTS (Usually fastest for large datasets)
SELECT c.customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id
);

-- Method 2: IN (Good for small subquery results)
SELECT c.customer_name
FROM customers c
WHERE c.customer_id IN (
    SELECT DISTINCT customer_id FROM orders
);

-- Method 3: JOIN (Good when you need data from both tables)
SELECT DISTINCT c.customer_name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id;
```

#### UNION vs UNION ALL
```sql
-- UNION removes duplicates (slower)
SELECT customer_name FROM customers WHERE customer_type = 'premium'
UNION
SELECT customer_name FROM customers WHERE total_spent > 10000;

-- UNION ALL keeps duplicates (faster)
SELECT customer_name FROM customers WHERE customer_type = 'premium'
UNION ALL
SELECT customer_name FROM customers WHERE total_spent > 10000;
```

### 7.2 Materialized Views and Summary Tables

#### Creating Materialized Views
```sql
-- Create summary table for complex aggregations
CREATE TABLE monthly_sales_summary (
    year_month DATE,
    total_orders INT,
    total_revenue DECIMAL(12,2),
    avg_order_value DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Populate with complex calculation
INSERT INTO monthly_sales_summary (year_month, total_orders, total_revenue, avg_order_value)
SELECT 
    DATE_FORMAT(order_date, '%Y-%m-01') as year_month,
    COUNT(*) as total_orders,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as avg_order_value
FROM orders
WHERE order_date >= '2023-01-01'
GROUP BY DATE_FORMAT(order_date, '%Y-%m-01');

-- Fast query on summary table
SELECT * FROM monthly_sales_summary ORDER BY year_month;
```

#### Maintaining Summary Tables
```sql
-- Create procedure to refresh summary data
DELIMITER //
CREATE PROCEDURE RefreshMonthlySummary()
BEGIN
    -- Clear existing data
    DELETE FROM monthly_sales_summary;
    
    -- Repopulate with fresh data
    INSERT INTO monthly_sales_summary (year_month, total_orders, total_revenue, avg_order_value)
    SELECT 
        DATE_FORMAT(order_date, '%Y-%m-01') as year_month,
        COUNT(*) as total_orders,
        SUM(total_amount) as total_revenue,
        AVG(total_amount) as avg_order_value
    FROM orders
    WHERE order_date >= '2023-01-01'
    GROUP BY DATE_FORMAT(order_date, '%Y-%m-01');
END //
DELIMITER ;

-- Schedule to run daily
-- CREATE EVENT refresh_summary_daily
-- ON SCHEDULE EVERY 1 DAY
-- DO CALL RefreshMonthlySummary();
```

### 7.3 Query Caching Strategies

#### Application-Level Caching
```sql
-- Identify frequently executed queries for caching
SELECT 
    query_text,
    execution_count,
    avg_execution_time,
    total_execution_time
FROM query_performance_log
WHERE execution_count > 1000
ORDER BY total_execution_time DESC;

-- Cache results of expensive queries
-- Example: Product catalog query
SELECT p.product_id, p.product_name, p.price, c.category_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
WHERE p.status = 'active'
ORDER BY p.product_name;
```

### 7.4 Batch Processing Optimization

#### Efficient Batch Inserts
```sql
-- Bad: Individual inserts
INSERT INTO products (name, price, category_id) VALUES ('Product 1', 19.99, 1);
INSERT INTO products (name, price, category_id) VALUES ('Product 2', 29.99, 1);
INSERT INTO products (name, price, category_id) VALUES ('Product 3', 39.99, 2);

-- Good: Batch insert
INSERT INTO products (name, price, category_id) VALUES 
    ('Product 1', 19.99, 1),
    ('Product 2', 29.99, 1),
    ('Product 3', 39.99, 2),
    ('Product 4', 49.99, 2);
```

#### Efficient Batch Updates
```sql
-- Bad: Individual updates
UPDATE products SET price = price * 1.1 WHERE product_id = 1;
UPDATE products SET price = price * 1.1 WHERE product_id = 2;

-- Good: Batch update
UPDATE products SET price = price * 1.1 WHERE category_id = 1;

-- Even better: Use CASE for different updates
UPDATE products 
SET price = CASE 
    WHEN category_id = 1 THEN price * 1.1
    WHEN category_id = 2 THEN price * 1.05
    ELSE price
END
WHERE category_id IN (1, 2);
```

---

## 8. Performance Monitoring {#performance-monitoring}

### 8.1 Key Performance Indicators (KPIs)

#### Query Performance Metrics
```sql
-- Monitor slow queries (MySQL)
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- Log queries taking > 2 seconds

-- Analyze slow queries
SELECT 
    query_time,
    lock_time,
    rows_sent,
    rows_examined,
    sql_text
FROM mysql.slow_log
WHERE query_time > 5
ORDER BY query_time DESC;
```

#### System Performance Metrics
```sql
-- Monitor connection usage
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';

-- Monitor buffer pool usage (MySQL InnoDB)
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_total';
SHOW STATUS LIKE 'Innodb_buffer_pool_pages_free';

-- Calculate buffer pool hit ratio
SELECT 
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 
    as hit_ratio
FROM 
    (SELECT 
        VARIABLE_VALUE as Innodb_buffer_pool_reads 
     FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') as reads,
    (SELECT 
        VARIABLE_VALUE as Innodb_buffer_pool_read_requests 
     FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests') as requests;
```

### 8.2 Monitoring Tools and Techniques

#### Built-in Monitoring
```sql
-- MySQL Performance Schema
-- Enable performance schema
SET GLOBAL performance_schema = ON;

-- Top 10 slowest queries
SELECT 
    DIGEST_TEXT as query,
    COUNT_STAR as exec_count,
    AVG_TIMER_WAIT/1000000000 as avg_time_sec,
    SUM_TIMER_WAIT/1000000000 as total_time_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY AVG_TIMER_WAIT DESC
LIMIT 10;

-- Most resource-intensive queries
SELECT 
    DIGEST_TEXT as query,
    SUM_ROWS_EXAMINED as total_rows_examined,
    SUM_ROWS_SENT as total_rows_sent,
    SUM_ROWS_EXAMINED/SUM_ROWS_SENT as efficiency_ratio
FROM performance_schema.events_statements_summary_by_digest
WHERE SUM_ROWS_SENT > 0
ORDER BY SUM_ROWS_EXAMINED DESC
LIMIT 10;
```

#### Custom Monitoring Queries
```sql
-- Create monitoring views for regular checks
CREATE VIEW query_performance_summary AS
SELECT 
    DATE(created_at) as date,
    COUNT(*) as total_queries,
    AVG(execution_time) as avg_execution_time,
    MAX(execution_time) as max_execution_time,
    COUNT(CASE WHEN execution_time > 5 THEN 1 END) as slow_queries
FROM query_log
GROUP BY DATE(created_at)
ORDER BY date DESC;

-- Monitor table growth
CREATE VIEW table_growth_monitoring AS
SELECT 
    table_name,
    table_rows,
    data_length,
    index_length,
    (data_length + index_length) as total_size,
    ROUND((data_length + index_length) / 1024 / 1024, 2) as size_mb
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY total_size DESC;
```

### 8.3 Performance Alerts and Thresholds

#### Setting Up Performance Alerts
```sql
-- Create alerts table
CREATE TABLE performance_alerts (
    alert_id INT AUTO_INCREMENT PRIMARY KEY,
    alert_type VARCHAR(50),
    alert_message TEXT,
    metric_value DECIMAL(10,2),
    threshold_value DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Procedure to check and alert on slow queries
DELIMITER //
CREATE PROCEDURE CheckPerformanceAlerts()
BEGIN
    DECLARE slow_query_count INT;
    DECLARE avg_query_time DECIMAL(10,2);
    
    -- Check for slow queries in last hour
    SELECT COUNT(*), AVG(execution_time)
    INTO slow_query_count, avg_query_time
    FROM query_log
    WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 HOUR)
    AND execution_time > 5;
    
    -- Alert if too many slow queries
    IF slow_query_count > 10 THEN
        INSERT INTO performance_alerts (alert_type, alert_message, metric_value, threshold_value)
        VALUES ('slow_queries', 'High number of slow queries detected', slow_query_count, 10);
    END IF;
    
    -- Alert if average query time is too high
    IF avg_query_time > 2 THEN
        INSERT INTO performance_alerts (alert_type, alert_message, metric_value, threshold_value)
        VALUES ('avg_query_time', 'Average query time exceeds threshold', avg_query_time, 2);
    END IF;
END //
DELIMITER ;
```

---

## 9. Hardware and System Optimization {#hardware-optimization}

### 9.1 Storage Optimization

#### SSD vs HDD Considerations
```sql
-- Optimize for SSD storage
-- Configure MySQL for SSD
SET GLOBAL innodb_flush_method = 'O_DIRECT';
SET GLOBAL innodb_io_capacity = 2000;  -- Higher for SSD
SET GLOBAL innodb_io_capacity_max = 4000;

-- For traditional HDD
SET GLOBAL innodb_io_capacity = 200;   -- Lower for HDD
SET GLOBAL innodb_io_capacity_max = 400;
```

#### File System Optimization
```sql
-- Separate data and log files
-- MySQL configuration example
[mysqld]
datadir = /var/lib/mysql/data
innodb_data_home_dir = /var/lib/mysql/data
innodb_log_group_home_dir = /var/lib/mysql/logs

-- Use appropriate file systems
-- ext4 for general use
-- XFS for large files
-- ZFS for advanced features
```

### 9.2 Memory Configuration

#### Buffer Pool Optimization
```sql
-- Set buffer pool size (70-80% of available RAM for dedicated DB server)
SET GLOBAL innodb_buffer_pool_size = 8G;

-- Multiple buffer pool instances for better concurrency
SET GLOBAL innodb_buffer_pool_instances = 8;

-- Monitor buffer pool effectiveness
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
WHERE VARIABLE_NAME IN (
    'Innodb_buffer_pool_pages_total',
    'Innodb_buffer_pool_pages_free',
    'Innodb_buffer_pool_pages_data',
    'Innodb_buffer_pool_pages_dirty'
);
```

#### Query Cache Configuration
```sql
-- Enable query cache (MySQL 5.7 and earlier)
SET GLOBAL query_cache_type = 1;
SET GLOBAL query_cache_size = 256M;

-- Monitor query cache effectiveness
SHOW STATUS LIKE 'Qcache%';

-- Calculate hit ratio
SELECT 
    (Qcache_hits / (Qcache_hits + Com_select)) * 100 as hit_ratio
FROM 
    (SELECT VARIABLE_VALUE as Qcache_hits 
     FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Qcache_hits') as hits,
    (SELECT VARIABLE_VALUE as Com_select 
     FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Com_select') as selects;
```

### 9.3 CPU and Concurrency Optimization

#### Connection Pool Configuration
```sql
-- Optimize connection handling
SET GLOBAL max_connections = 200;
SET GLOBAL thread_cache_size = 50;

-- Monitor connection usage
SHOW STATUS LIKE 'Threads_created';
SHOW STATUS LIKE 'Threads_cached';
SHOW STATUS LIKE 'Threads_connected';

-- Calculate thread cache hit ratio
SELECT 
    (1 - (Threads_created / Connections)) * 100 as thread_cache_hit_ratio
FROM 
    (SELECT VARIABLE_VALUE as Threads_created 
     FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Threads_created') as created,
    (SELECT VARIABLE_VALUE as Connections 
     FROM INFORMATION_SCHEMA.GLOBAL_STATUS 
     WHERE VARIABLE_NAME = 'Connections') as conn;
```

---

## 10. Real-World Case Studies {#case-studies}

### 10.1 Case Study 1: E-commerce Order Processing

#### Problem
An e-commerce site experiencing slow order processing during peak hours.

**Original Query (Taking 15+ seconds)**:
```sql
SELECT 
    o.order_id,
    c.customer_name,
    c.email,
    o.order_date,
    o.total_amount,
    COUNT(oi.item_id) as item_count,
    GROUP_CONCAT(p.product_name) as products
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date >= '2023-01-01'
  AND o.status = 'pending'
GROUP BY o.order_id, c.customer_name, c.email, o.order_date, o.total_amount
ORDER BY o.order_date DESC
LIMIT 100;
```

#### Analysis
```sql
-- Check execution plan
EXPLAIN SELECT /* ... query above ... */;

-- Problems identified:
-- 1. Full table scan on orders table
-- 2. Inefficient GROUP_CONCAT operation
-- 3. Missing indexes on critical columns
-- 4. Large result set being sorted
```

#### Solution Implementation
```sql
-- Step 1: Create necessary indexes
CREATE INDEX idx_orders_status_date ON orders(status, order_date);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- Step 2: Optimize the query
SELECT 
    o.order_id,
    c.customer_name,
    c.email,
    o.order_date,
    o.total_amount,
    oi.item_count,
    oi.products
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN (
    SELECT 
        order_id,
        COUNT(*) as item_count,
        GROUP_CONCAT(product_name) as products
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY order_id
) oi ON o.order_id = oi.order_id
WHERE o.status = 'pending'
  AND o.order_date >= '2023-01-01'
ORDER BY o.order_date DESC
LIMIT 100;

-- Step 3: Create summary table for even better performance
CREATE TABLE order_summary (
    order_id INT PRIMARY KEY,
    item_count INT,
    products TEXT,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Populate summary table
INSERT INTO order_summary (order_id, item_count, products)
SELECT 
    oi.order_id,
    COUNT(*) as item_count,
    GROUP_CONCAT(p.product_name) as products
FROM order_items oi
JOIN products p ON oi.product_id = p.product_id
GROUP BY oi.order_id;

-- Final optimized query (executes in <1 second)
SELECT 
    o.order_id,
    c.customer_name,
    c.email,
    o.order_date,
    o.total_amount,
    os.item_count,
    os.products
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_summary os ON o.order_id = os.order_id
WHERE o.status = 'pending'
  AND o.order_date >= '2023-01-01'
ORDER BY o.order_date DESC
LIMIT 100;
```

#### Results
- **Query time**: Reduced from 15+ seconds to <1 second
- **CPU usage**: Reduced by 80%
- **Memory usage**: Reduced by 60%
- **Scalability**: Can handle 10x more concurrent users

### 10.2 Case Study 2: Analytics Dashboard

#### Problem
A business intelligence dashboard taking 5+ minutes to load daily reports.

**Original Query**:
```sql
SELECT 
    DATE(order_date) as date,
    COUNT(*) as total_orders,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as avg_order_value,
    COUNT(DISTINCT customer_id) as unique_customers,
    -- Complex calculations
    SUM(CASE WHEN total_amount > 100 THEN 1 ELSE 0 END) as high_value_orders,
    (SUM(CASE WHEN total_amount > 100 THEN 1 ELSE 0 END) / COUNT(*)) * 100 as high_value_percentage
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY DATE(order_date)
ORDER BY date DESC;
```

#### Solution
```sql
-- Create optimized daily summary table
CREATE TABLE daily_order_summary (
    summary_date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(12,2),
    avg_order_value DECIMAL(10,2),
    unique_customers INT,
    high_value_orders INT,
    high_value_percentage DECIMAL(5,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Create procedure to populate summary
DELIMITER //
CREATE PROCEDURE UpdateDailySummary(IN target_date DATE)
BEGIN
    INSERT INTO daily_order_summary (
        summary_date, total_orders, total_revenue, avg_order_value,
        unique_customers, high_value_orders, high_value_percentage
    )
    SELECT 
        DATE(order_date) as summary_date,
        COUNT(*) as total_orders,
        SUM(total_amount) as total_revenue,
        AVG(total_amount) as avg_order_value,
        COUNT(DISTINCT customer_id) as unique_customers,
        SUM(CASE WHEN total_amount > 100 THEN 1 ELSE 0 END) as high_value_orders,
        (SUM(CASE WHEN total_amount > 100 THEN 1 ELSE 0 END) / COUNT(*)) * 100 as high_value_percentage
    FROM orders
    WHERE DATE(order_date) = target_date
    GROUP BY DATE(order_date)
    ON DUPLICATE KEY UPDATE
        total_orders = VALUES(total_orders),
        total_revenue = VALUES(total_revenue),
        avg_order_value = VALUES(avg_order_value),
        unique_customers = VALUES(unique_customers),
        high_value_orders = VALUES(high_value_orders),
        high_value_percentage = VALUES(high_value_percentage);
END //
DELIMITER ;

-- Schedule daily updates
-- Run this procedure daily via cron job or scheduler

-- Optimized dashboard query (executes in <1 second)
SELECT * FROM daily_order_summary
WHERE summary_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
ORDER BY summary_date DESC;
```

#### Results
- **Load time**: Reduced from 5+ minutes to <1 second
- **Dashboard responsiveness**: 300x improvement
- **Server load**: Reduced by 95%
- **User satisfaction**: Significantly improved

### 10.3 Case Study 3: User Activity Tracking

#### Problem
A social media platform struggling with user activity queries taking too long.

**Original Query**:
```sql
SELECT 
    u.username,
    COUNT(DISTINCT p.post_id) as total_posts,
    COUNT(DISTINCT l.like_id) as total_likes_given,
    COUNT(DISTINCT c.comment_id) as total_comments,
    MAX(p.created_at) as last_post_date,
    COUNT(DISTINCT f.follower_id) as follower_count
FROM users u
LEFT JOIN posts p ON u.user_id = p.user_id
LEFT JOIN likes l ON u.user_id = l.user_id
LEFT JOIN comments c ON u.user_id = c.user_id
LEFT JOIN followers f ON u.user_id = f.following_id
WHERE u.created_at >= '2023-01-01'
GROUP BY u.user_id, u.username
HAVING total_posts > 10
ORDER BY follower_count DESC;
```

#### Solution
```sql
-- Create user activity summary table
CREATE TABLE user_activity_summary (
    user_id INT PRIMARY KEY,
    total_posts INT DEFAULT 0,
    total_likes_given INT DEFAULT 0,
    total_comments INT DEFAULT 0,
    last_post_date DATETIME,
    follower_count INT DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Create triggers to maintain summary in real-time
DELIMITER //

-- Update summary when new post is created
CREATE TRIGGER update_user_summary_post
AFTER INSERT ON posts
FOR EACH ROW
BEGIN
    INSERT INTO user_activity_summary (user_id, total_posts, last_post_date)
    VALUES (NEW.user_id, 1, NEW.created_at)
    ON DUPLICATE KEY UPDATE
        total_posts = total_posts + 1,
        last_post_date = GREATEST(last_post_date, NEW.created_at);
END //

-- Update summary when new like is given
CREATE TRIGGER update_user_summary_like
AFTER INSERT ON likes
FOR EACH ROW
BEGIN
    INSERT INTO user_activity_summary (user_id, total_likes_given)
    VALUES (NEW.user_id, 1)
    ON DUPLICATE KEY UPDATE
        total_likes_given = total_likes_given + 1;
END //

-- Update summary when new comment is made
CREATE TRIGGER update_user_summary_comment
AFTER INSERT ON comments
FOR EACH ROW
BEGIN
    INSERT INTO user_activity_summary (user_id, total_comments)
    VALUES (NEW.user_id, 1)
    ON DUPLICATE KEY UPDATE
        total_comments = total_comments + 1;
END //

-- Update summary when new follower is added
CREATE TRIGGER update_user_summary_follower
AFTER INSERT ON followers
FOR EACH ROW
BEGIN
    INSERT INTO user_activity_summary (user_id, follower_count)
    VALUES (NEW.following_id, 1)
    ON DUPLICATE KEY UPDATE
        follower_count = follower_count + 1;
END //

DELIMITER ;

-- Optimized query using summary table
SELECT 
    u.username,
    uas.total_posts,
    uas.total_likes_given,
    uas.total_comments,
    uas.last_post_date,
    uas.follower_count
FROM users u
JOIN user_activity_summary uas ON u.user_id = uas.user_id
WHERE u.created_at >= '2023-01-01'
  AND uas.total_posts > 10
ORDER BY uas.follower_count DESC;
```

#### Results
- **Query time**: Reduced from 45+ seconds to <0.5 seconds
- **Real-time updates**: Activity metrics updated instantly
- **Scalability**: Can handle millions of users efficiently
- **System load**: Reduced by 90%

---

## 11. Best Practices Checklist {#best-practices}

### 11.1 Query Design Best Practices

#### ✅ SELECT Statement Optimization
- [ ] Use specific column names instead of SELECT *
- [ ] Apply WHERE clauses to filter data early
- [ ] Use LIMIT for large result sets
- [ ] Choose appropriate JOIN types
- [ ] Avoid functions in WHERE clauses on indexed columns

#### ✅ Index Strategy
- [ ] Create indexes on frequently queried columns
- [ ] Use composite indexes for multi-column queries
- [ ] Monitor and remove unused indexes
- [ ] Consider covering indexes for read-heavy queries
- [ ] Avoid over-indexing (impacts write performance)

#### ✅ Data Types and Schema
- [ ] Use appropriate data types (smallest that fits)
- [ ] Normalize for data integrity
- [ ] Denormalize for performance when needed
- [ ] Use partitioning for very large tables
- [ ] Implement proper foreign key constraints

### 11.2 Performance Monitoring Checklist

#### ✅ Regular Monitoring Tasks
- [ ] Monitor slow query logs daily
- [ ] Check buffer pool hit ratios weekly
- [ ] Review index usage monthly
- [ ] Analyze query execution plans for critical queries
- [ ] Monitor disk space and growth trends

#### ✅ Performance Metrics to Track
- [ ] Average query response time
- [ ] Number of slow queries per day
- [ ] Database connections usage
- [ ] Memory utilization
- [ ] CPU usage during peak hours

### 11.3 Maintenance Best Practices

#### ✅ Database Maintenance
- [ ] Update table statistics regularly
- [ ] Rebuild/reorganize indexes when needed
- [ ] Monitor for table fragmentation
- [ ] Archive old data systematically
- [ ] Backup and test restore procedures

#### ✅ Security and Compliance
- [ ] Use parameterized queries to prevent SQL injection
- [ ] Implement proper user access controls
- [ ] Encrypt sensitive data at rest and in transit
- [ ] Regular security audits and updates
- [ ] Monitor for suspicious database activity

---

## 12. Troubleshooting Common Issues {#troubleshooting}

### 12.1 Slow Query Troubleshooting

#### Issue: Query Running Slowly
**Diagnosis Steps:**
```sql
-- Step 1: Check execution plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- Step 2: Check if indexes exist
SHOW INDEX FROM orders;

-- Step 3: Check table statistics
ANALYZE TABLE orders;

-- Step 4: Check for table locks
SHOW PROCESSLIST;
```

**Common Solutions:**
```sql
-- Solution 1: Add missing index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Solution 2: Update statistics
ANALYZE TABLE orders;

-- Solution 3: Rewrite query
-- Instead of:
SELECT * FROM orders WHERE YEAR(order_date) = 2023;
-- Use:
SELECT * FROM orders WHERE order_date >= '2023-01-01' AND order_date < '2024-01-01';
```

### 12.2 High CPU Usage

#### Issue: Database Server High CPU
**Diagnosis:**
```sql
-- Check running processes
SHOW FULL PROCESSLIST;

-- Find CPU-intensive queries
SELECT 
    query,
    exec_count,
    avg_timer_wait/1000000000 as avg_seconds
FROM performance_schema.events_statements_summary_by_digest
ORDER BY avg_timer_wait DESC
LIMIT 10;
```

**Solutions:**
```sql
-- Add missing indexes
CREATE INDEX idx_problem_column ON problem_table(problem_column);

-- Optimize queries
-- Replace complex subqueries with JOINs
-- Use LIMIT for large result sets
-- Add WHERE clauses to filter data
```

### 12.3 Memory Issues

#### Issue: High Memory Usage
**Diagnosis:**
```sql
-- Check buffer pool usage
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Check query cache usage
SHOW STATUS LIKE 'Qcache%';

-- Check temporary table usage
SHOW STATUS LIKE 'Created_tmp%';
```

**Solutions:**
```sql
-- Optimize buffer pool size
SET GLOBAL innodb_buffer_pool_size = 4G;  -- Adjust based on available RAM

-- Reduce temporary table creation
-- Add indexes to avoid filesort
-- Use appropriate data types
-- Limit result sets
```

### 12.4 Lock Contention

#### Issue: Deadlocks and Lock Waits
**Diagnosis:**
```sql
-- Check current locks
SELECT * FROM information_schema.innodb_locks;

-- Check lock waits
SELECT * FROM information_schema.innodb_lock_waits;

-- Check deadlock information
SHOW ENGINE INNODB STATUS;
```

**Solutions:**
```sql
-- Keep transactions short
BEGIN;
UPDATE orders SET status = 'shipped' WHERE order_id = 123;
COMMIT;

-- Access tables in consistent order
-- Transaction 1 and 2 should both access tables in same order
-- e.g., always access customers before orders

-- Use appropriate isolation levels
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

### 12.5 Storage Issues

#### Issue: Running Out of Disk Space
**Diagnosis:**
```sql
-- Check table sizes
SELECT 
    table_name,
    ROUND(((data_length + index_length) / 1024 / 1024), 2) as size_mb
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY size_mb DESC;

-- Check for large log files
SHOW VARIABLES LIKE 'innodb_log_file_size';
```

**Solutions:**
```sql
-- Archive old data
DELETE FROM orders WHERE order_date < '2022-01-01';

-- Optimize tables
OPTIMIZE TABLE orders;

-- Consider partitioning large tables
ALTER TABLE orders PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

---

## Quick Reference Guide

### Performance Optimization Priority Order
1. **Fix Missing Indexes** (Highest Impact)
2. **Optimize Query Structure** (High Impact)
3. **Update Statistics** (Medium Impact)
4. **Tune Configuration** (Medium Impact)
5. **Hardware Upgrades** (Variable Impact)

### Essential Commands for Optimization
```sql
-- Analysis Commands
EXPLAIN SELECT ...;
SHOW INDEX FROM table_name;
ANALYZE TABLE table_name;
SHOW STATUS LIKE 'pattern';

-- Optimization Commands
CREATE INDEX idx_name ON table(column);
OPTIMIZE TABLE table_name;
UPDATE table SET column = value WHERE condition;

-- Monitoring Commands
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
SHOW ENGINE INNODB STATUS;
```

### Performance Tuning Formulas
- **Buffer Pool Size**: 70-80% of available RAM
- **Max Connections**: Based on concurrent users + 20%
- **Query Cache Size**: 0-256MB (disabled in MySQL 8.0+)
- **Index Selectivity**: Aim for >95% for effective indexes

---

## Conclusion

Database optimization is an ongoing process that requires:
- **Regular monitoring** of performance metrics
- **Proactive maintenance** of indexes and statistics
- **Continuous learning** about new optimization techniques
- **Testing and validation** of changes in non-production environments

Remember: The best optimization strategy is to measure first, then optimize based on actual performance bottlenecks rather than assumptions.

This guide provides a comprehensive foundation for database optimization. Start with the basics, monitor your results, and gradually implement more advanced techniques as needed.

---

*Last Updated: 2024*
*For the most current database-specific optimization techniques, consult your database vendor's documentation.*