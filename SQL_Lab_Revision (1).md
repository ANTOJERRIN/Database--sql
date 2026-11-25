# SQL Lab Internals: Complete Revision Guide

**Based on your COMPANY database assignment. Dialect: MySQL.** Your goal is to understand the pattern behind a query, then apply it to the question.

## Your one-hour study plan

| Time | Focus |
|---|---|
| 0–10 min | Schema, keys, CREATE TABLE and inserting records |
| 10–20 min | SELECT, WHERE, NULL, LIKE, IN and ORDER BY |
| 20–35 min | JOIN types and tracing relationships |
| 35–45 min | COUNT, SUM, AVG, GROUP BY and HAVING |
| 45–55 min | Subqueries, EXISTS, ANY, ALL and UNION |
| 55–60 min | Common mistakes and practice questions |

The worked answers to all 30 assignment questions are in section 9. Read the concepts first, then cover the answers and try writing the queries yourself.

## 1. The COMPANY schema: understand the connections

| Table | Columns | Main purpose |
|---|---|---|
| `employee` | Fname, Minit, Lname, Ssn, Bdate, Address, Sex, Salary, Super_ssn, Dno | Employee details |
| `department` | Dname, Dnumber, Mgr_ssn, Mgr_start_date | Department and its manager |
| `project` | Pname, Pnumber, Plocation, Dnum | Projects and controlling departments |
| `works_on` | Essn, Pno, Hours | Connects employees to projects |
| `dependent` | Essn, Dependent_name, Sex, Bdate, Relationship | Employees' dependents |
| `dept_locations` | Dnumber, Dlocation | Department locations shown in your notebook |

The column names below use underscores (`Super_ssn`, `Mgr_ssn`, `Dependent_name`) as in the assignment PDF. If your actual database uses `Superssn` or `Mgrssn`, adapt the names. Check using `DESC employee;`.

**Relationships to remember:**

```text
employee.Dno       = department.Dnumber
department.Mgr_ssn = employee.Ssn
employee.Super_ssn = supervisor.Ssn       (same employee table twice)
project.Dnum       = department.Dnumber
works_on.Essn      = employee.Ssn
works_on.Pno       = project.Pnumber
dependent.Essn     = employee.Ssn
dept_locations.Dnumber = department.Dnumber
```

One employee can work on multiple projects, and one project can have multiple employees. `works_on` connects them: it is a junction table.

## 2. Database, keys, constraints and data types

```sql
CREATE DATABASE company;
USE company;
SHOW TABLES;
DESC employee;             -- structure of a table
SHOW CREATE TABLE employee; -- full definition and constraints
```

Use `USE existing_database;` if the examiner already provides a database. Do not recreate an existing database unnecessarily.

| Term | Meaning | Example |
|---|---|---|
| Primary key | Uniquely identifies a row; cannot be NULL | `employee.Ssn` |
| Composite primary key | Two or more columns together identify a row | `(Essn, Pno)` in `works_on` |
| Foreign key | Refers to a key in another table, or the same table | `employee.Dno` references `department.Dnumber` |
| NOT NULL | A value is required | `Fname VARCHAR(30) NOT NULL` |
| UNIQUE | Prevents duplicate non-NULL values | `Dname VARCHAR(50) UNIQUE` |
| DEFAULT | Value supplied when a column is omitted | `Hours DECIMAL(5,1) DEFAULT 0` |
| CHECK | Restricts permitted values | `CHECK (Salary >= 0)` |

MySQL enforces `CHECK` in modern versions (8.0.16+); older versions may parse but ignore it. A nullable foreign key can be NULL; a non-NULL value must match a referenced row. A primary key is unique and non-NULL. `UNIQUE` in MySQL can allow multiple NULL values.

Useful types: `INT` for whole numbers, `DECIMAL(10,2)` for exact money values, `CHAR(9)` for a fixed-length identifier, `VARCHAR(100)` for varying text, and `DATE` for dates in `YYYY-MM-DD` form. SSNs are identifiers, so text avoids losing leading zeroes.

## 3. Create the company tables

The employee and department tables reference each other: an employee belongs to a department, while a department has a manager who is an employee. Create `department` first without its manager foreign key, create `employee`, then add the manager foreign key. This is one valid design; use the exact names and constraints your examiner specifies if they differ.

```sql
CREATE TABLE department (
    Dnumber INT PRIMARY KEY,
    Dname VARCHAR(50) NOT NULL UNIQUE,
    Mgr_ssn CHAR(9),
    Mgr_start_date DATE
);

CREATE TABLE employee (
    Ssn CHAR(9) PRIMARY KEY,
    Fname VARCHAR(30) NOT NULL,
    Minit CHAR(1),
    Lname VARCHAR(30) NOT NULL,
    Bdate DATE,
    Address VARCHAR(100),
    Sex CHAR(1),
    Salary DECIMAL(10,2),
    Super_ssn CHAR(9),
    Dno INT,
    CONSTRAINT chk_salary CHECK (Salary >= 0),
    CONSTRAINT fk_emp_dept FOREIGN KEY (Dno)
        REFERENCES department(Dnumber),
    CONSTRAINT fk_emp_super FOREIGN KEY (Super_ssn)
        REFERENCES employee(Ssn)
);

ALTER TABLE department
    ADD CONSTRAINT fk_dept_manager FOREIGN KEY (Mgr_ssn)
    REFERENCES employee(Ssn);

CREATE TABLE project (
    Pnumber INT PRIMARY KEY,
    Pname VARCHAR(50) NOT NULL,
    Plocation VARCHAR(50),
    Dnum INT,
    CONSTRAINT fk_project_dept FOREIGN KEY (Dnum)
        REFERENCES department(Dnumber)
);

CREATE TABLE works_on (
    Essn CHAR(9),
    Pno INT,
    Hours DECIMAL(5,1),
    PRIMARY KEY (Essn, Pno),
    FOREIGN KEY (Essn) REFERENCES employee(Ssn),
    FOREIGN KEY (Pno) REFERENCES project(Pnumber)
);

CREATE TABLE dependent (
    Essn CHAR(9),
    Dependent_name VARCHAR(50),
    Sex CHAR(1),
    Bdate DATE,
    Relationship VARCHAR(30),
    PRIMARY KEY (Essn, Dependent_name),
    FOREIGN KEY (Essn) REFERENCES employee(Ssn)
);

CREATE TABLE dept_locations (
    Dnumber INT,
    Dlocation VARCHAR(50),
    PRIMARY KEY (Dnumber, Dlocation),
    FOREIGN KEY (Dnumber) REFERENCES department(Dnumber)
);
```

`Hours` is nullable because the notebook has a NULL hours value. The composite key `(Essn, Dependent_name)` assumes one employee does not have two dependents with the same name; follow the lab schema if a different key is specified.

### Inserting data when there is a circular foreign key

Insert departments with a NULL manager first. Then insert employees with NULL `Super_ssn` initially, update supervisor links after those employees exist, and finally update department managers. Insert projects, work assignments and dependents after their parent rows exist. The following illustrates the process; add the remaining rows from your supplied data in the same way.

```sql
INSERT INTO department (Dnumber, Dname, Mgr_ssn, Mgr_start_date)
VALUES (5, 'Research', NULL, NULL);

INSERT INTO employee (Ssn, Fname, Minit, Lname, Salary, Super_ssn, Dno)
VALUES ('333445555', 'Franklin', 'T', 'Wong', 40000, NULL, 5),
       ('123456789', 'John', 'B', 'Smith', 30000, NULL, 5);

UPDATE employee SET Super_ssn = '333445555' WHERE Ssn = '123456789';
UPDATE department SET Mgr_ssn = '333445555' WHERE Dnumber = 5;

INSERT INTO project (Pnumber, Pname, Plocation, Dnum)
VALUES (1, 'ProductX', 'Bellaire', 5);
INSERT INTO works_on (Essn, Pno, Hours)
VALUES ('123456789', 1, 32.5);
```

Explicit column lists prevent mistakes caused by column order. Names like `'John'` need quotes; numbers usually do not. `NULL` has no quotes. If the data is already loaded, simply inspect it with `SELECT * FROM employee;`.

### Other table and data commands

```sql
ALTER TABLE employee ADD Email VARCHAR(100);
ALTER TABLE employee MODIFY Email VARCHAR(150);
ALTER TABLE employee DROP COLUMN Email;

UPDATE employee SET Salary = 32000 WHERE Ssn = '123456789';
DELETE FROM works_on WHERE Essn = '123456789' AND Pno = 1;
```

`UPDATE` and `DELETE` without `WHERE` affect all rows. `DELETE` removes rows; `TRUNCATE TABLE` removes all rows and has MySQL-specific restrictions and implicit-commit behavior; `DROP TABLE` removes the table itself. Foreign keys may prevent deletion or dropping referenced records/tables. Run destructive examples only on a practice database.

## 4. SELECT and filtering fundamentals

```sql
SELECT * FROM employee;                    -- all columns
SELECT Fname, Lname, Salary FROM employee; -- chosen columns
SELECT DISTINCT Salary FROM employee;      -- remove duplicate result values
SELECT Fname AS first_name FROM employee;  -- rename output column
```

`WHERE` filters individual rows. Comparison operators: `=`, `<>` or `!=`, `>`, `<`, `>=`, `<=`.

```sql
SELECT Fname, Lname FROM employee WHERE Dno = 5;
SELECT Fname, Lname FROM employee WHERE Sex = 'M' AND Salary > 30000;
SELECT * FROM employee WHERE Dno = 4 OR Dno = 5;
SELECT * FROM employee WHERE Dno IN (4, 5);       -- same idea as OR
SELECT * FROM employee WHERE Dno NOT IN (4, 5);
SELECT * FROM employee WHERE Salary BETWEEN 25000 AND 40000; -- inclusive
SELECT * FROM employee WHERE Address LIKE '%Houston%';
SELECT * FROM employee WHERE Fname LIKE 'J%';     -- starts with J
SELECT * FROM employee WHERE Fname LIKE '_o%';     -- second character o
SELECT * FROM employee WHERE Super_ssn IS NULL;
SELECT * FROM employee WHERE Super_ssn IS NOT NULL;
SELECT * FROM employee WHERE Bdate >= '1960-01-01' AND Bdate < '1970-01-01';
SELECT Fname, Lname, Dno FROM employee ORDER BY Dno ASC, Lname ASC;
SELECT Fname, Salary FROM employee ORDER BY Salary DESC LIMIT 3;
```

`%` means zero or more characters; `_` means exactly one. `ASC` means ascending (small to large or A–Z); `DESC` means descending. Ascending is the default. `BETWEEN` includes both endpoints. Use `IS NULL`, never `= NULL`. `YEAR(Bdate) BETWEEN 1960 AND 1969` also works in MySQL.

Parentheses clarify mixed conditions: `WHERE Dno = 5 AND (Sex = 'M' OR Salary > 40000)`. `AND` has higher precedence than `OR`.

## 5. Joins: combine related rows

Ask three questions: **Which tables contain the requested columns? How are those tables connected? Which rows must remain if there is no match?**

| Join | Result |
|---|---|
| `INNER JOIN` / `JOIN` | Only matching rows from both tables |
| `LEFT JOIN` | Every left-table row, plus matching right-table rows; missing right values become NULL |
| `RIGHT JOIN` | Every right-table row, plus matches from the left |
| `CROSS JOIN` | Every possible combination |
| Self join | A table joined to itself, using separate aliases |

### INNER JOIN: employee and department

```sql
SELECT e.Fname, e.Lname, d.Dname
FROM employee AS e
JOIN department AS d ON e.Dno = d.Dnumber;
```

`e` and `d` are short names called aliases. `ON` states how rows match. For John, `e.Dno = 5` matches the Research department's `d.Dnumber = 5`. If a column name exists in more than one table, prefix it with the alias.

### LEFT JOIN: include employees with no dependent

```sql
SELECT e.Fname, e.Lname, dep.Dependent_name
FROM employee e
LEFT JOIN dependent dep ON e.Ssn = dep.Essn;
```

An employee with three dependents appears three times; an employee with no dependent appears once with NULL in `Dependent_name`. Putting `WHERE dep.Dependent_name IS NOT NULL` afterward removes unmatched rows. To find only those with no dependent, use `WHERE dep.Essn IS NULL`.

### Self join: employee and supervisor

```sql
SELECT e.Fname AS employee_first, e.Lname AS employee_last,
       s.Fname AS supervisor_first, s.Lname AS supervisor_last
FROM employee e
LEFT JOIN employee s ON e.Super_ssn = s.Ssn;
```

`e` is the employee and `s` is the supervisor, even though both refer to `employee`. `LEFT JOIN` keeps employees without supervisors.

### CROSS JOIN

```sql
SELECT e.Ssn, d.Dname FROM employee e CROSS JOIN department d;
```

With 8 employees and 3 departments, this produces 8 × 3 = 24 combinations, regardless of actual department membership.

## 6. Aggregate functions, grouping and HAVING

An aggregate computes one value from multiple rows.

| Function | Meaning |
|---|---|
| `COUNT(*)` | Counts rows |
| `COUNT(column)` | Counts non-NULL values in that column |
| `COUNT(DISTINCT column)` | Counts distinct non-NULL values |
| `SUM(column)` | Adds non-NULL values |
| `AVG(column)` | Average of non-NULL values |
| `MIN(column)`, `MAX(column)` | Minimum and maximum non-NULL values |

```sql
SELECT COUNT(*) AS employee_count, SUM(Salary) AS total_salary,
       AVG(Salary) AS avg_salary, MIN(Salary) AS minimum_salary,
       MAX(Salary) AS maximum_salary
FROM employee;

SELECT Dno, COUNT(*) AS employee_count, AVG(Salary) AS average_salary
FROM employee
GROUP BY Dno;
```

For `GROUP BY Dno`, SQL collects all employees with the same department number, then calculates one result row per department. Do not select an ungrouped employee name alongside grouped results unless it is properly aggregated or functionally dependent under your MySQL settings.

### WHERE vs HAVING

```sql
SELECT Dno, COUNT(*) AS employee_count, AVG(Salary) AS avg_salary
FROM employee
WHERE Salary > 25000
GROUP BY Dno
HAVING COUNT(*) > 1
ORDER BY avg_salary DESC;
```

1. `WHERE` keeps individual employees earning more than 25,000.
2. `GROUP BY` collects those remaining employees by department.
3. `HAVING` keeps groups containing more than one remaining employee.
4. `ORDER BY` sorts the result.

Written order: `SELECT → FROM/JOIN → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT`. A useful logical order is `FROM/JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT`.

### Conditional counts

```sql
SELECT Dno,
       COUNT(*) AS all_employees,
       SUM(CASE WHEN Salary > 40000 THEN 1 ELSE 0 END) AS above_40000
FROM employee
GROUP BY Dno
HAVING COUNT(*) > 5;
```

This keeps departments with more than five employees **in total**, then reports how many of those earn over 40,000. A `WHERE Salary > 40000` before grouping would change the meaning of `COUNT(*) > 5`.

When counting matches after a `LEFT JOIN`, use `COUNT(right_table.non_null_key)`: `COUNT(*)` also counts the single NULL-extended row for a left row with no match.

## 7. Subqueries and set operations

A subquery is a query inside another query. Think from the inner query outward.

```sql
-- Department number found first, then employees in that department.
SELECT Fname, Lname FROM employee
WHERE Dno = (SELECT Dnumber FROM department WHERE Dname = 'Research');
```

`=` needs a single value. Use `IN` if the subquery can return multiple values.

```sql
SELECT Fname, Lname FROM employee
WHERE Dno IN (SELECT Dnumber FROM department WHERE Dname = 'Research');

-- No matching dependent for this employee: correlated subquery.
SELECT e.Fname, e.Lname FROM employee e
WHERE NOT EXISTS (SELECT 1 FROM dependent d WHERE d.Essn = e.Ssn);

-- More than EVERY salary in department 4.
SELECT Fname, Lname, Salary FROM employee
WHERE Salary > ALL (SELECT Salary FROM employee WHERE Dno = 4);

-- More than AT LEAST ONE salary in department 4.
SELECT Fname, Lname, Salary FROM employee
WHERE Salary > ANY (SELECT Salary FROM employee WHERE Dno = 4);
```

For a nonempty set of non-NULL salaries, `> ALL` means greater than the maximum; `> ANY` means greater than the minimum. Empty sets and NULL values have special SQL semantics, so don't assume `MAX`/`MIN` are always interchangeable. `NOT IN` can return unexpected results if the subquery includes NULL; `NOT EXISTS` is often safer.

```sql
SELECT Pno AS project_no FROM works_on WHERE Essn = '123456789'
UNION
SELECT Pnumber FROM project WHERE Dnum = 5;
```

`UNION` combines compatible result columns and removes duplicates; `UNION ALL` keeps duplicates. The queries must return the same number of columns with compatible types.

## 8. How to construct a query in the lab

Example: **For each project, find its name and total hours spent by employees.**

1. Output: project name and total hours → `SELECT Pname, SUM(Hours)`.
2. Tables: names live in `project`; hours live in `works_on`.
3. Link: `project.Pnumber = works_on.Pno`.
4. “For each project” → `GROUP BY` project identifier and name.
5. Decide whether projects with no assignments must appear → choose `LEFT JOIN` if yes.

```sql
SELECT p.Pnumber, p.Pname, COALESCE(SUM(w.Hours), 0) AS total_hours
FROM project p
LEFT JOIN works_on w ON p.Pnumber = w.Pno
GROUP BY p.Pnumber, p.Pname;
```

`COALESCE(..., 0)` displays zero for a missing or all-NULL sum. This treats unknown hours as zero for display, so leave the result NULL if the distinction matters. An `INNER JOIN` shows only projects with at least one work assignment.

## 9. Complete assignment query bank: questions 1–30

The queries use the schema in section 3. They are grouped as in your assignment; read comments as the question and practice writing them without looking.

### Q1–Q7: retrieval and filtering

```sql
-- 1. John B. Smith's birth date and address
SELECT Bdate, Address FROM employee
WHERE Fname = 'John' AND Minit = 'B' AND Lname = 'Smith';

-- 2. Employees in department 5
SELECT * FROM employee WHERE Dno = 5;

-- 3. Male employees earning more than 30,000
SELECT Fname, Minit, Lname FROM employee
WHERE Sex = 'M' AND Salary > 30000;

-- 4. Distinct salaries
SELECT DISTINCT Salary FROM employee;

-- 5. Address contains Houston, TX; match exact spacing in your data
SELECT * FROM employee WHERE Address LIKE '%Houston, TX%';

-- 6. Employees born in the 1960s
SELECT * FROM employee
WHERE Bdate >= '1960-01-01' AND Bdate < '1970-01-01';

-- 7. Employees without a supervisor
SELECT Fname, Minit, Lname FROM employee WHERE Super_ssn IS NULL;
```

### Q8–Q15: joins

```sql
-- 8. Names and addresses of employees in Research
SELECT e.Fname, e.Minit, e.Lname, e.Address
FROM employee e JOIN department d ON e.Dno = d.Dnumber
WHERE d.Dname = 'Research';

-- 9. Stafford projects: project number, department, manager's surname
SELECT p.Pnumber, p.Dnum, e.Lname
FROM project p
JOIN department d ON p.Dnum = d.Dnumber
JOIN employee e ON d.Mgr_ssn = e.Ssn
WHERE p.Plocation = 'Stafford';

-- 10. Each employee and their immediate supervisor
SELECT e.Fname AS employee_first, e.Lname AS employee_last,
       s.Fname AS supervisor_first, s.Lname AS supervisor_last
FROM employee e LEFT JOIN employee s ON e.Super_ssn = s.Ssn;

-- 11. Every employee SSN paired with every department
SELECT e.Ssn, d.Dname FROM employee e CROSS JOIN department d;

-- 12. Employee names and the projects they work on
SELECT e.Fname, e.Lname, p.Pname
FROM employee e
JOIN works_on w ON e.Ssn = w.Essn
JOIN project p ON w.Pno = p.Pnumber;

-- 13. Employees working on project 10, 20 or 30
SELECT DISTINCT e.Fname, e.Minit, e.Lname
FROM employee e JOIN works_on w ON e.Ssn = w.Essn
WHERE w.Pno IN (10, 20, 30);

-- 14. Department managers with at least one dependent
SELECT DISTINCT e.Fname, e.Minit, e.Lname
FROM employee e
JOIN department d ON e.Ssn = d.Mgr_ssn
JOIN dependent dep ON e.Ssn = dep.Essn;

-- 15. Employee and dependent names, including employees with none
SELECT e.Fname, e.Minit, e.Lname, dep.Dependent_name
FROM employee e LEFT JOIN dependent dep ON e.Ssn = dep.Essn;
```

Q12 returns employees with a project assignment. If the examiner explicitly requires employees with no project too, use `LEFT JOIN` for both joins. For Q13 and Q14, `DISTINCT` avoids repeated names from multiple matching rows; different people with identical names could still collapse into one result.

### Q16–Q22: UNION and subqueries

```sql
-- 16. Projects involving a Smith as worker or controlling department manager
SELECT w.Pno AS project_number
FROM works_on w JOIN employee e ON w.Essn = e.Ssn
WHERE e.Lname = 'Smith'
UNION
SELECT p.Pnumber
FROM project p JOIN department d ON p.Dnum = d.Dnumber
JOIN employee e ON d.Mgr_ssn = e.Ssn
WHERE e.Lname = 'Smith';

-- 17. No dependents using NOT IN (Essn is NOT NULL in our schema)
SELECT Fname, Lname FROM employee
WHERE Ssn NOT IN (SELECT Essn FROM dependent WHERE Essn IS NOT NULL);

-- 18. No dependents using NOT EXISTS
SELECT e.Fname, e.Lname FROM employee e
WHERE NOT EXISTS (SELECT 1 FROM dependent d WHERE d.Essn = e.Ssn);

-- 19. Research employees using a nested query
SELECT Fname, Lname FROM employee
WHERE Dno = (SELECT Dnumber FROM department WHERE Dname = 'Research');

-- 20. Salary more than every employee in department 4
SELECT Fname, Lname, Salary FROM employee
WHERE Salary > ALL (SELECT Salary FROM employee WHERE Dno = 4);

-- 21. Salary more than at least one employee in department 4
SELECT Fname, Lname, Salary FROM employee
WHERE Salary > ANY (SELECT Salary FROM employee WHERE Dno = 4);

-- 22. Employee with a dependent of the same first name and sex
SELECT DISTINCT e.Fname, e.Lname
FROM employee e JOIN dependent d ON e.Ssn = d.Essn
WHERE e.Fname = d.Dependent_name AND e.Sex = d.Sex;
```

### Q23–Q30: aggregates, grouping and sorting

```sql
-- 23. Sum, maximum, minimum and average salary
SELECT SUM(Salary) AS total_salary, MAX(Salary) AS maximum_salary,
       MIN(Salary) AS minimum_salary, AVG(Salary) AS average_salary
FROM employee;

-- 24. Number of employees
SELECT COUNT(*) AS total_employees FROM employee;

-- 25. Employee count and average salary per department represented in employee
SELECT Dno, COUNT(*) AS total_employees, AVG(Salary) AS average_salary
FROM employee GROUP BY Dno;

-- 26. Project number, name and total hours, including projects with no workers
SELECT p.Pnumber, p.Pname, COALESCE(SUM(w_hours.Hours), 0) AS total_hours
FROM project p LEFT JOIN works_on w_hours ON p.Pnumber = w_hours.Pno
GROUP BY p.Pnumber, p.Pname;

-- 27. Departments with >5 total staff: count staff earning >40,000
SELECT Dno, SUM(CASE WHEN Salary > 40000 THEN 1 ELSE 0 END) AS high_earners
FROM employee GROUP BY Dno HAVING COUNT(*) > 5;

-- 28. Number of distinct non-NULL supervisors
SELECT COUNT(DISTINCT Super_ssn) AS total_supervisors FROM employee;

-- 29. Departments whose total salary exceeds 100,000
SELECT Dno, SUM(Salary) AS total_salary
FROM employee GROUP BY Dno HAVING SUM(Salary) > 100000;

-- 30. Department ascending, then surname alphabetically
SELECT Fname, Lname, Dno FROM employee
ORDER BY Dno ASC, Lname ASC;
```

For Q26, `SUM` ignores NULL hours; `COALESCE` displays zero if no non-NULL hours exist. Q30's assignment wording says “dynamically,” but your handwritten solution uses `ASC`; use `DESC` if the examiner specifically asks for descending order.

For Q25, starting from `employee` excludes departments with zero employees. To include them, start with `department`, use a `LEFT JOIN`, and count `e.Ssn` rather than `COUNT(*)`.

## 10. Common exam mistakes to avoid

1. **`NULL`:** use `IS NULL` / `IS NOT NULL`, not `= NULL`.
2. **Strings:** write `Dname = 'Research'`, not `Dname = Research`.
3. **Join condition:** connect the related key columns, such as `e.Dno = d.Dnumber`; an incorrect match changes the result.
4. **Ambiguous columns:** qualify repeated names using aliases, for example `e.Ssn`.
5. **Row vs group:** `WHERE` filters rows; `HAVING` filters aggregate groups.
6. **Aggregate in WHERE:** use `HAVING COUNT(*) > 5`, not `WHERE COUNT(*) > 5`.
7. **Parentheses:** functions use `SUM(Salary)`, `COUNT(*)`, `AVG(Salary)`.
8. **Duplicates:** joins can produce multiple rows per employee; use `DISTINCT` only when the question requires unique output.
9. **Missing matches:** use `LEFT JOIN` when all rows from the left must be retained. Conditions on the right table in `WHERE` can remove unmatched rows.
10. **Column and table names:** use `DESC` and `SHOW TABLES`; your notebook and database may use different spelling or case.
11. **Keys and insertion:** insert referenced parent rows first. Handle the employee–department circular reference using nullable manager/supervisor values, then update them.
12. **`NOT IN` and NULL:** if the subquery might return NULL, filter it out or use `NOT EXISTS`.
13. **Semicolons:** end SQL statements with `;`. Dates use `'YYYY-MM-DD'`.
14. **Don't blindly rely on handwritten totals:** results depend on the exact inserted data. Run the query to verify.

## 11. Five quick practice prompts

Try these yourself, then compare with the query bank:

1. Show all names in department 5. Which clause filters individual employees? (Q2)
2. Show every employee and their supervisor, including someone without one. Which table appears twice? (Q10)
3. Count employees and calculate average salary for each department. Which column defines the groups? (Q25)
4. Show departments where total salaries exceed 100,000. Why is `HAVING` needed? (Q29)
5. Find employees without dependents. How does the subquery check each employee? (Q18)

**Final mental check:** identify output columns → identify tables → link keys → filter rows → group if needed → filter groups → sort.

---

## 12. Any MySQL database: start-to-finish lab workflow

Use this as a reusable checklist for COMPANY, COLLEGE, SALES, or any other schema. **Names in angle brackets are placeholders:** replace them with actual names and remove `< >`. Run only the steps requested or needed for the given lab database. These commands are for MySQL; other DBMSs use some different commands.

### Step 1 — Read the question and map the schema

Before typing, note each table, its columns and data types, its primary key, and which columns refer to other tables. Identify composite keys and whether a column can be NULL. For each query, underline the required output, tables, conditions, and words such as **each**, **total**, **including all**, and **at least one**.

Example mapping: `student(dept_id)` is a child foreign key referring to `department(dept_id)`, the parent primary key. Create the parent table before the child table, and insert parent rows first. If two tables refer to each other, use the staged approach in section 3.

### Step 2 — Connect and choose your database

In a GUI, open the provided MySQL connection. On a command line, if your lab uses the MySQL client:

```bash
mysql -u <username> -p
```

Enter the password when prompted. This is a **terminal command**, not a SQL statement. Once connected:

```sql
SHOW DATABASES;
SELECT DATABASE();               -- current database; NULL if none selected
USE <database_name>;
SHOW TABLES;
```

If the examiner asks you to create a new database and it does not already exist:

```sql
CREATE DATABASE <database_name>;
USE <database_name>;
```

If the database already exists, use its actual name. `CREATE DATABASE IF NOT EXISTS <database_name>;` avoids an error when it exists, but does not clear or reset existing data. Database/table name case sensitivity can depend on the server's operating system and settings.

### Step 3 — Inspect what already exists

```sql
SHOW TABLES;
DESCRIBE <table_name>;           -- DESC <table_name>; also works
SHOW CREATE TABLE <table_name>;  -- shows keys and constraints
SELECT * FROM <table_name> LIMIT 10;
SELECT COUNT(*) FROM <table_name>;
```

Write down the **actual** spelling of column names before writing joins. Check whether tables and records are already provided; do not create duplicates.

### Step 4 — Create tables and constraints if required

Generic syntax and a runnable small example:

```sql
CREATE TABLE <table_name> (
    <column_name> <data_type> <column_constraints>,
    <other_column> <data_type>,
    PRIMARY KEY (<key_column>),
    FOREIGN KEY (<foreign_column>) REFERENCES <parent_table>(<parent_key>)
);
```

```sql
CREATE TABLE department_demo (
    dept_id INT PRIMARY KEY,
    dept_name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE student_demo (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(50) NOT NULL,
    age INT CHECK (age >= 0),
    city VARCHAR(40) DEFAULT 'Bengaluru',
    dept_id INT,
    CONSTRAINT fk_student_demo_dept FOREIGN KEY (dept_id)
        REFERENCES department_demo(dept_id)
);
```

`PRIMARY KEY` identifies a row, `FOREIGN KEY` links tables, `NOT NULL` makes a value compulsory, `UNIQUE` prevents duplicate non-NULL values, `DEFAULT` supplies a value when omitted, and `CHECK` defines a rule. As noted earlier, MySQL versions before 8.0.16 may not enforce CHECK constraints. Data types of corresponding foreign and referenced keys must be compatible; use matching definitions. For a composite key: `PRIMARY KEY (student_id, subject_id)`.

If asked to add a constraint later:

```sql
ALTER TABLE <child_table>
ADD CONSTRAINT <constraint_name>
FOREIGN KEY (<child_column>) REFERENCES <parent_table>(<parent_column>);
```

Existing rows must satisfy the foreign key when you add it. Verify structure using `DESC` and `SHOW CREATE TABLE`.

### Step 5 — Insert the given data and verify it

```sql
INSERT INTO department_demo (dept_id, dept_name)
VALUES (1, 'CSE'), (2, 'ECE');

INSERT INTO student_demo (student_id, student_name, age, city, dept_id)
VALUES (101, 'Anu', 20, 'Bengaluru', 1),
       (102, 'Ravi', 21, 'Mysuru', 2);

SELECT * FROM department_demo;
SELECT * FROM student_demo;
SELECT COUNT(*) AS row_count FROM student_demo;
```

The number and order of values must match the listed columns. Text and dates use single quotes; `NULL` is unquoted. For dates use `'2026-11-25'`. Insert parent records before referencing them from child tables. `DEFAULT` is used when the column is omitted, or when the `DEFAULT` keyword is supplied.

### Step 6 — Use the query template

```sql
SELECT [DISTINCT] <columns_or_aggregates>
FROM <table> [alias]
[JOIN <other_table> [alias] ON <join_condition>]
[WHERE <row_condition>]
[GROUP BY <grouping_columns>]
[HAVING <group_condition>]
[ORDER BY <column> ASC | DESC]
[LIMIT <number>];
```

Square brackets mean optional parts; do not type the brackets. Replace the `ASC | DESC` example with either `ASC` or `DESC`. Build in this order:

1. Find output columns and the tables that contain them.
2. Add the correct `JOIN ... ON ...` for related tables.
3. Add `WHERE` for conditions on individual rows.
4. Add `GROUP BY` when the question asks for a result **per** department, project, or other category.
5. Add `HAVING` for a condition on an aggregated group.
6. Add `ORDER BY` and `LIMIT` only if requested.

### Step 7 — Syntax patterns to remember

```sql
-- Filter, list of values, range, pattern, missing value
SELECT * FROM <table> WHERE <column> = <value>;
SELECT * FROM <table> WHERE <column> IN (<value1>, <value2>);
SELECT * FROM <table> WHERE <column> BETWEEN <low> AND <high>;
SELECT * FROM <table> WHERE <text_column> LIKE '%<text>%';
SELECT * FROM <table> WHERE <column> IS NULL;
SELECT DISTINCT <column> FROM <table>;

-- Join matched rows; change to LEFT JOIN to retain all left rows
SELECT a.<column>, b.<column>
FROM <table_a> a JOIN <table_b> b ON a.<key> = b.<key>;

-- Aggregate, then group and filter groups
SELECT <group_column>, COUNT(*) AS total, AVG(<numeric_column>) AS average_value
FROM <table>
WHERE <row_condition>
GROUP BY <group_column>
HAVING COUNT(*) > <number>
ORDER BY total DESC;

-- Query within a query
SELECT <columns> FROM <table>
WHERE <column> IN (SELECT <column> FROM <other_table> WHERE <condition>);
```

`BETWEEN` includes both bounds. `%` matches any number of characters; `_` matches one. `COUNT(*)` counts rows; `COUNT(column)` ignores NULL. `WHERE` filters before grouping, `HAVING` filters after grouping. For `IN` string values, quote each value: `IN ('CSE', 'ECE')`.

### Step 8 — Modify records or structure only when requested

```sql
UPDATE <table> SET <column> = <value> WHERE <condition>;
DELETE FROM <table> WHERE <condition>;
ALTER TABLE <table> ADD <column> <data_type>;
ALTER TABLE <table> MODIFY <column> <new_data_type>;
ALTER TABLE <table> DROP COLUMN <column>;
```

Preview the rows first with `SELECT * FROM <table> WHERE <condition>;`. Without `WHERE`, `UPDATE` or `DELETE` affects every row. An `ALTER`, `DROP`, or `TRUNCATE` can change structure or remove data; use them only when the task asks. Constraints can block deletes and structural changes.

### Step 9 — Check results and finish

1. Check the query runs and returns the requested columns, correct row count and expected conditions. An empty result is valid if no row matches.
2. Check joins for repeated rows or missing unmatched rows; check whether `DISTINCT` is appropriate.
3. For aggregates, verify whether the question means all rows or only filtered rows; check `NULL` and `WHERE` versus `HAVING`.
4. Check ordering, spelling, aliases, brackets, quotes and the final semicolon.
5. Save the SQL script or record outputs **if your lab requires it**; follow the examiner's submission instructions. In a GUI, use its save/export controls; the exact steps depend on the tool.

### Quick error diagnosis

| Error or symptom | What to check |
|---|---|
| No database selected | Run `USE <database_name>;` |
| Table doesn't exist | Run `SHOW TABLES;`; check database, spelling and case |
| Unknown column | Run `DESC <table>;`; check spelling, alias and table |
| Column is ambiguous | Prefix it: `a.id`, `b.id` |
| Duplicate entry | Check primary/unique key values already present |
| Foreign key constraint fails | Insert the parent first; check matching key value, types and dependencies |
| Syntax error | Check commas, quotes, parentheses, clause order and semicolon |
| Empty results | Inspect input rows, join keys, filter values, text spacing and NULL |
| Too many rows | Check the `ON` condition and whether the relationship has multiple matches |
| Invalid group query | Group every selected non-aggregate column as required; inspect `GROUP BY` and `HAVING` |

### Compact transaction reference

If asked to demonstrate transactions on transactional tables such as InnoDB:

```sql
START TRANSACTION;
UPDATE student_demo SET city = 'Mysuru' WHERE student_id = 101;
-- Choose one:
ROLLBACK;  -- undo the uncommitted update
-- COMMIT; -- instead, permanently keep the transaction changes
```

`COMMIT` and `ROLLBACK` are alternatives here; do not run both expecting the rollback to undo an already committed update. MySQL DDL statements such as `CREATE TABLE` and `ALTER TABLE` can cause implicit commits, so do not assume they can be rolled back like the update above.
