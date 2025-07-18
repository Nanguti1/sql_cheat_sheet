# Complete SQL Cheat Sheet - Basic to Advanced

## Table of Contents
1. [Basic SQL Syntax](#basic-sql-syntax)
2. [Data Types](#data-types)
3. [SELECT Statements](#select-statements)
4. [Filtering Data](#filtering-data)
5. [Sorting and Grouping](#sorting-and-grouping)
6. [Joins](#joins)
7. [Subqueries](#subqueries)
8. [Data Manipulation](#data-manipulation)
9. [Database Structure](#database-structure)
10. [Advanced Functions](#advanced-functions)
11. [Window Functions](#window-functions)
12. [Common Table Expressions (CTEs)](#common-table-expressions-ctes)
13. [Indexes and Performance](#indexes-and-performance)
14. [Transactions](#transactions)
15. [Best Practices](#best-practices)

---

## Basic SQL Syntax

### General Structure
```sql
SELECT column1, column2
FROM table_name
WHERE condition
GROUP BY column
HAVING condition
ORDER BY column
LIMIT number;
```

### Case Sensitivity
- SQL keywords are case-insensitive (SELECT = select)
- Table and column names may be case-sensitive depending on the database
- String values are case-sensitive

---

## Data Types

### Numeric Types
- `INT` / `INTEGER` - Whole numbers
- `DECIMAL(p,s)` - Fixed-point numbers
- `FLOAT` - Floating-point numbers
- `DOUBLE` - Double precision floating-point

### String Types
- `VARCHAR(n)` - Variable-length string (max n characters)
- `CHAR(n)` - Fixed-length string (exactly n characters)
- `TEXT` - Large text data

### Date/Time Types
- `DATE` - Date only (YYYY-MM-DD)
- `TIME` - Time only (HH:MM:SS)
- `DATETIME` - Date and time
- `TIMESTAMP` - Date and time with timezone

### Boolean
- `BOOLEAN` - True/False values

---

## SELECT Statements

### Basic SELECT
```sql
-- Select all columns
SELECT * FROM employees;

-- Select specific columns
SELECT first_name, last_name, salary FROM employees;

-- Select with alias
SELECT first_name AS "First Name", 
       last_name AS "Last Name" 
FROM employees;

-- Select distinct values
SELECT DISTINCT department FROM employees;

-- Select with calculations
SELECT first_name, salary, salary * 12 AS annual_salary 
FROM employees;
```

### Column Aliases
```sql
SELECT 
    first_name AS fname,
    last_name AS lname,
    salary * 12 AS "Annual Salary"
FROM employees;
```

---

## Filtering Data

### WHERE Clause
```sql
-- Basic conditions
SELECT * FROM employees WHERE salary > 50000;
SELECT * FROM employees WHERE department = 'IT';
SELECT * FROM employees WHERE hire_date > '2020-01-01';

-- Multiple conditions
SELECT * FROM employees 
WHERE salary > 50000 AND department = 'IT';

SELECT * FROM employees 
WHERE department = 'IT' OR department = 'HR';

-- NOT condition
SELECT * FROM employees WHERE NOT department = 'IT';
```

### Comparison Operators
```sql
-- Equal
SELECT * FROM products WHERE price = 19.99;

-- Not equal
SELECT * FROM products WHERE price != 19.99;
SELECT * FROM products WHERE price <> 19.99;

-- Greater than, less than
SELECT * FROM products WHERE price > 20;
SELECT * FROM products WHERE price <= 50;

-- Between
SELECT * FROM products WHERE price BETWEEN 10 AND 50;

-- IN operator
SELECT * FROM employees WHERE department IN ('IT', 'HR', 'Finance');

-- NOT IN
SELECT * FROM employees WHERE department NOT IN ('IT', 'HR');
```

### Pattern Matching
```sql
-- LIKE operator
SELECT * FROM customers WHERE first_name LIKE 'J%';  -- Starts with J
SELECT * FROM customers WHERE first_name LIKE '%son'; -- Ends with son
SELECT * FROM customers WHERE first_name LIKE '%an%'; -- Contains an

-- Wildcards
-- % = any number of characters
-- _ = exactly one character
SELECT * FROM customers WHERE phone LIKE '555-___-____';

-- REGEXP/RLIKE (MySQL)
SELECT * FROM customers WHERE first_name REGEXP '^[A-M]'; -- Starts with A-M
```

### NULL Values
```sql
-- Check for NULL
SELECT * FROM employees WHERE manager_id IS NULL;

-- Check for NOT NULL
SELECT * FROM employees WHERE manager_id IS NOT NULL;

-- Handle NULL with COALESCE
SELECT first_name, COALESCE(middle_name, 'N/A') AS middle_name 
FROM employees;
```

---

## Sorting and Grouping

### ORDER BY
```sql
-- Sort ascending (default)
SELECT * FROM employees ORDER BY salary;

-- Sort descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple columns
SELECT * FROM employees ORDER BY department, salary DESC;

-- Sort by column position
SELECT first_name, last_name, salary 
FROM employees 
ORDER BY 3 DESC; -- Sort by 3rd column (salary)
```

### GROUP BY
```sql
-- Basic grouping
SELECT department, COUNT(*) as employee_count 
FROM employees 
GROUP BY department;

-- Multiple columns
SELECT department, job_title, AVG(salary) as avg_salary 
FROM employees 
GROUP BY department, job_title;

-- With WHERE (filter before grouping)
SELECT department, COUNT(*) 
FROM employees 
WHERE salary > 40000 
GROUP BY department;
```

### HAVING
```sql
-- Filter after grouping
SELECT department, COUNT(*) as employee_count 
FROM employees 
GROUP BY department 
HAVING COUNT(*) > 5;

-- Multiple conditions
SELECT department, AVG(salary) as avg_salary 
FROM employees 
GROUP BY department 
HAVING AVG(salary) > 60000 AND COUNT(*) > 3;
```

---

## Joins

### INNER JOIN
```sql
-- Basic inner join
SELECT e.first_name, e.last_name, d.department_name 
FROM employees e 
INNER JOIN departments d ON e.department_id = d.department_id;

-- Multiple joins
SELECT e.first_name, d.department_name, p.project_name 
FROM employees e 
INNER JOIN departments d ON e.department_id = d.department_id 
INNER JOIN projects p ON e.employee_id = p.employee_id;
```

### LEFT JOIN
```sql
-- Returns all records from left table
SELECT e.first_name, d.department_name 
FROM employees e 
LEFT JOIN departments d ON e.department_id = d.department_id;
```

### RIGHT JOIN
```sql
-- Returns all records from right table
SELECT e.first_name, d.department_name 
FROM employees e 
RIGHT JOIN departments d ON e.department_id = d.department_id;
```

### FULL OUTER JOIN
```sql
-- Returns all records when there's a match in either table
SELECT e.first_name, d.department_name 
FROM employees e 
FULL OUTER JOIN departments d ON e.department_id = d.department_id;
```

### CROSS JOIN
```sql
-- Cartesian product of both tables
SELECT e.first_name, d.department_name 
FROM employees e 
CROSS JOIN departments d;
```

### Self Join
```sql
-- Join table with itself
SELECT e1.first_name as Employee, e2.first_name as Manager 
FROM employees e1 
LEFT JOIN employees e2 ON e1.manager_id = e2.employee_id;
```

---

## Subqueries

### Simple Subquery
```sql
-- Subquery in WHERE clause
SELECT * FROM employees 
WHERE salary > (SELECT AVG(salary) FROM employees);

-- Subquery in SELECT clause
SELECT first_name, salary, 
       (SELECT AVG(salary) FROM employees) as avg_salary 
FROM employees;
```

### Correlated Subquery
```sql
-- Subquery references outer query
SELECT first_name, salary 
FROM employees e1 
WHERE salary > (
    SELECT AVG(salary) 
    FROM employees e2 
    WHERE e2.department_id = e1.department_id
);
```

### EXISTS
```sql
-- Check if subquery returns any rows
SELECT * FROM employees e 
WHERE EXISTS (
    SELECT 1 FROM projects p 
    WHERE p.employee_id = e.employee_id
);
```

### IN with Subquery
```sql
SELECT * FROM employees 
WHERE department_id IN (
    SELECT department_id 
    FROM departments 
    WHERE location = 'New York'
);
```

---

## Data Manipulation

### INSERT
```sql
-- Insert single row
INSERT INTO employees (first_name, last_name, salary, department_id) 
VALUES ('John', 'Doe', 55000, 1);

-- Insert multiple rows
INSERT INTO employees (first_name, last_name, salary, department_id) 
VALUES 
    ('Jane', 'Smith', 60000, 2),
    ('Bob', 'Johnson', 58000, 1),
    ('Alice', 'Williams', 62000, 3);

-- Insert from another table
INSERT INTO employees_backup 
SELECT * FROM employees WHERE department_id = 1;
```

### UPDATE
```sql
-- Update single record
UPDATE employees 
SET salary = 65000 
WHERE employee_id = 1;

-- Update multiple columns
UPDATE employees 
SET salary = salary * 1.1, last_updated = NOW() 
WHERE department_id = 1;

-- Update with JOIN
UPDATE employees e 
JOIN departments d ON e.department_id = d.department_id 
SET e.salary = e.salary * 1.05 
WHERE d.department_name = 'IT';
```

### DELETE
```sql
-- Delete specific records
DELETE FROM employees WHERE employee_id = 1;

-- Delete with condition
DELETE FROM employees WHERE salary < 30000;

-- Delete with JOIN
DELETE e FROM employees e 
JOIN departments d ON e.department_id = d.department_id 
WHERE d.department_name = 'Temp';
```

---

## Database Structure

### CREATE TABLE
```sql
-- Basic table creation
CREATE TABLE employees (
    employee_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    salary DECIMAL(10,2),
    hire_date DATE,
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- With constraints
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) CHECK (price > 0),
    category_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_category (category_id)
);
```

### ALTER TABLE
```sql
-- Add column
ALTER TABLE employees ADD COLUMN phone VARCHAR(20);

-- Modify column
ALTER TABLE employees MODIFY COLUMN phone VARCHAR(25);

-- Drop column
ALTER TABLE employees DROP COLUMN phone;

-- Add constraint
ALTER TABLE employees ADD CONSTRAINT fk_department 
FOREIGN KEY (department_id) REFERENCES departments(department_id);
```

### DROP TABLE
```sql
-- Drop table
DROP TABLE employees;

-- Drop if exists
DROP TABLE IF EXISTS employees;
```

---

## Advanced Functions

### Aggregate Functions
```sql
-- COUNT
SELECT COUNT(*) FROM employees;
SELECT COUNT(DISTINCT department_id) FROM employees;

-- SUM
SELECT SUM(salary) FROM employees;

-- AVG
SELECT AVG(salary) FROM employees;

-- MIN/MAX
SELECT MIN(salary), MAX(salary) FROM employees;

-- GROUP_CONCAT (MySQL)
SELECT department_id, GROUP_CONCAT(first_name) 
FROM employees GROUP BY department_id;
```

### String Functions
```sql
-- CONCAT
SELECT CONCAT(first_name, ' ', last_name) as full_name FROM employees;

-- SUBSTRING
SELECT SUBSTRING(first_name, 1, 3) FROM employees;

-- UPPER/LOWER
SELECT UPPER(first_name), LOWER(last_name) FROM employees;

-- LENGTH
SELECT first_name, LENGTH(first_name) FROM employees;

-- REPLACE
SELECT REPLACE(first_name, 'a', 'A') FROM employees;

-- TRIM
SELECT TRIM(first_name) FROM employees;
```

### Date Functions
```sql
-- Current date/time
SELECT NOW(), CURDATE(), CURTIME();

-- Date formatting
SELECT DATE_FORMAT(hire_date, '%Y-%m-%d') FROM employees;

-- Date arithmetic
SELECT first_name, hire_date, 
       DATEDIFF(NOW(), hire_date) as days_employed 
FROM employees;

-- Extract parts
SELECT YEAR(hire_date), MONTH(hire_date), DAY(hire_date) 
FROM employees;
```

### Conditional Functions
```sql
-- CASE statement
SELECT first_name, salary,
    CASE 
        WHEN salary > 70000 THEN 'High'
        WHEN salary > 50000 THEN 'Medium'
        ELSE 'Low'
    END as salary_grade
FROM employees;

-- IF function (MySQL)
SELECT first_name, IF(salary > 60000, 'High', 'Low') as salary_level 
FROM employees;

-- NULLIF
SELECT first_name, NULLIF(middle_name, '') FROM employees;
```

---

## Window Functions

### Basic Window Functions
```sql
-- ROW_NUMBER
SELECT first_name, salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as row_num
FROM employees;

-- RANK
SELECT first_name, salary,
    RANK() OVER (ORDER BY salary DESC) as rank
FROM employees;

-- DENSE_RANK
SELECT first_name, salary,
    DENSE_RANK() OVER (ORDER BY salary DESC) as dense_rank
FROM employees;
```

### Partition By
```sql
-- Ranking within departments
SELECT first_name, department_id, salary,
    RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) as dept_rank
FROM employees;

-- Running totals
SELECT first_name, salary,
    SUM(salary) OVER (ORDER BY hire_date) as running_total
FROM employees;
```

### Advanced Window Functions
```sql
-- LAG/LEAD
SELECT first_name, salary,
    LAG(salary, 1) OVER (ORDER BY hire_date) as prev_salary,
    LEAD(salary, 1) OVER (ORDER BY hire_date) as next_salary
FROM employees;

-- FIRST_VALUE/LAST_VALUE
SELECT first_name, salary,
    FIRST_VALUE(salary) OVER (ORDER BY hire_date) as first_salary,
    LAST_VALUE(salary) OVER (ORDER BY hire_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) as last_salary
FROM employees;
```

---

## Common Table Expressions (CTEs)

### Simple CTE
```sql
-- Basic CTE
WITH high_earners AS (
    SELECT first_name, last_name, salary 
    FROM employees 
    WHERE salary > 60000
)
SELECT * FROM high_earners ORDER BY salary DESC;
```

### Multiple CTEs
```sql
-- Multiple CTEs
WITH 
high_earners AS (
    SELECT employee_id, first_name, salary 
    FROM employees WHERE salary > 60000
),
dept_avg AS (
    SELECT department_id, AVG(salary) as avg_salary 
    FROM employees GROUP BY department_id
)
SELECT h.first_name, h.salary, d.avg_salary 
FROM high_earners h 
JOIN employees e ON h.employee_id = e.employee_id 
JOIN dept_avg d ON e.department_id = d.department_id;
```

### Recursive CTE
```sql
-- Recursive CTE for hierarchical data
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor member
    SELECT employee_id, first_name, manager_id, 1 as level
    FROM employees WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive member
    SELECT e.employee_id, e.first_name, e.manager_id, eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy ORDER BY level, first_name;
```

---

## Indexes and Performance

### Creating Indexes
```sql
-- Single column index
CREATE INDEX idx_last_name ON employees(last_name);

-- Composite index
CREATE INDEX idx_dept_salary ON employees(department_id, salary);

-- Unique index
CREATE UNIQUE INDEX idx_email ON employees(email);

-- Partial index (with condition)
CREATE INDEX idx_active_employees ON employees(department_id) 
WHERE status = 'active';
```

### Dropping Indexes
```sql
DROP INDEX idx_last_name ON employees;
```

### Query Optimization
```sql
-- Use EXPLAIN to analyze query execution
EXPLAIN SELECT * FROM employees WHERE department_id = 1;

-- Use indexes effectively
SELECT * FROM employees WHERE last_name = 'Smith'; -- Uses index
SELECT * FROM employees WHERE UPPER(last_name) = 'SMITH'; -- Doesn't use index
```

---

## Transactions

### Basic Transaction
```sql
-- Start transaction
START TRANSACTION;

-- Or
BEGIN;

-- Perform operations
UPDATE employees SET salary = salary * 1.1 WHERE department_id = 1;
INSERT INTO salary_changes (employee_id, old_salary, new_salary) VALUES (1, 50000, 55000);

-- Commit changes
COMMIT;

-- Or rollback if error
ROLLBACK;
```

### Savepoints
```sql
START TRANSACTION;

UPDATE employees SET salary = salary * 1.1 WHERE department_id = 1;

SAVEPOINT sp1;

UPDATE employees SET salary = salary * 1.05 WHERE department_id = 2;

-- Rollback to savepoint
ROLLBACK TO sp1;

COMMIT;
```

---

## Best Practices

### Query Writing
1. **Use meaningful aliases for tables and columns**
2. **Always specify column names in INSERT statements**
3. **Use LIMIT when testing queries on large datasets**
4. **Avoid SELECT * in production code**
5. **Use appropriate data types**

### Performance Tips
```sql
-- Use indexes on frequently queried columns
CREATE INDEX idx_status ON orders(status);

-- Use LIMIT for pagination
SELECT * FROM employees ORDER BY hire_date LIMIT 10 OFFSET 20;

-- Use EXISTS instead of IN for subqueries
SELECT * FROM employees e 
WHERE EXISTS (SELECT 1 FROM projects p WHERE p.employee_id = e.employee_id);

-- Use UNION ALL instead of UNION when duplicates are acceptable
SELECT first_name FROM employees 
UNION ALL 
SELECT first_name FROM contractors;
```

### Security
```sql
-- Use parameterized queries to prevent SQL injection
-- Instead of: SELECT * FROM users WHERE username = '" + username + "'
-- Use prepared statements with parameters

-- Grant minimal necessary permissions
GRANT SELECT, INSERT ON employees TO 'app_user'@'localhost';

-- Avoid storing sensitive data in plain text
-- Use proper encryption for passwords and sensitive information
```

### Maintenance
```sql
-- Regular statistics updates
ANALYZE TABLE employees;

-- Check table integrity
CHECK TABLE employees;

-- Optimize tables
OPTIMIZE TABLE employees;
```

---

## Common Patterns and Examples

### Pagination
```sql
-- MySQL/PostgreSQL
SELECT * FROM employees ORDER BY employee_id LIMIT 10 OFFSET 20;

-- SQL Server
SELECT * FROM employees ORDER BY employee_id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Find Duplicates
```sql
-- Find duplicate records
SELECT first_name, last_name, COUNT(*) 
FROM employees 
GROUP BY first_name, last_name 
HAVING COUNT(*) > 1;
```

### Delete Duplicates
```sql
-- Delete duplicates keeping the one with lowest ID
DELETE e1 FROM employees e1 
JOIN employees e2 ON e1.first_name = e2.first_name 
    AND e1.last_name = e2.last_name 
    AND e1.employee_id > e2.employee_id;
```

### Running Totals
```sql
-- Running total with window functions
SELECT employee_id, salary,
    SUM(salary) OVER (ORDER BY employee_id) as running_total
FROM employees;
```

### Pivot Data
```sql
-- Pivot example (MySQL)
SELECT 
    department_id,
    SUM(CASE WHEN YEAR(hire_date) = 2020 THEN 1 ELSE 0 END) as '2020',
    SUM(CASE WHEN YEAR(hire_date) = 2021 THEN 1 ELSE 0 END) as '2021',
    SUM(CASE WHEN YEAR(hire_date) = 2022 THEN 1 ELSE 0 END) as '2022'
FROM employees 
GROUP BY department_id;
```

---

## Quick Reference

### SQL Execution Order
1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT
6. ORDER BY
7. LIMIT

### Common Shortcuts
- `SELECT *` - Select all columns
- `CTRL+;` - Add semicolon to end of line
- `--` - Single line comment
- `/* */` - Multi-line comment

### Useful Commands
```sql
-- Show tables
SHOW TABLES;

-- Describe table structure
DESCRIBE employees;
-- or
SHOW COLUMNS FROM employees;

-- Show indexes
SHOW INDEX FROM employees;

-- Show create table statement
SHOW CREATE TABLE employees;
```

---

*This cheat sheet covers the most commonly used SQL features. Different database systems (MySQL, PostgreSQL, SQL Server, Oracle) may have slight variations in syntax and available functions.*