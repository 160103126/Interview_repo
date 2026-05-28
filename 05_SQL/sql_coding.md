# SQL Coding Patterns — Interview Questions

> Essential SQL queries and patterns for interviews. Standard ANSI SQL used (compatible with PostgreSQL/MySQL).

---

## Table of Contents

- [🟢 Simple (Fundamentals)](#-simple-fundamentals)
- [🟡 Medium (Intermediate)](#-medium-intermediate)
- [🔴 Hard (Advanced / MAANG-level)](#-hard-advanced--maang-level)

---

## 🟢 Simple (Fundamentals)

### Q1: Find the second highest salary.

**Schema:** `Employee (id, salary)`

**Answer:**

```sql
-- Method 1: Using MAX() and Subquery
SELECT MAX(salary) AS SecondHighestSalary
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);

-- Method 2: Using ORDER BY and OFFSET (Database specific)
-- PostgreSQL / MySQL:
SELECT salary AS SecondHighestSalary
FROM Employee
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

---

### Q2: Find duplicate emails.

**Schema:** `Person (id, email)`

**Answer:**

```sql
SELECT email
FROM Person
GROUP BY email
HAVING COUNT(email) > 1;
```

---

### Q3: Customers who never order.

**Schema:** `Customers (id, name)`, `Orders (id, customerId)`

**Answer:**

```sql
-- Method 1: LEFT JOIN
SELECT c.name AS Customers
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customerId
WHERE o.id IS NULL;

-- Method 2: NOT IN
SELECT name AS Customers
FROM Customers
WHERE id NOT IN (SELECT customerId FROM Orders);

-- Method 3: NOT EXISTS (Often fastest)
SELECT name AS Customers
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1 FROM Orders o WHERE o.customerId = c.id
);
```

---

### Q4: Delete duplicate emails.

**Schema:** `Person (id, email)`
*Task: Delete all duplicate emails, keeping only one unique email with the smallest id.*

**Answer:**

```sql
DELETE p1
FROM Person p1, Person p2
WHERE p1.email = p2.email 
  AND p1.id > p2.id;
```

---

## 🟡 Medium (Intermediate)

### Q5: Nth highest salary.

**Schema:** `Employee (id, salary)`

**Answer:**

```sql
-- Using Window Functions (DENSE_RANK)
WITH RankedSalaries AS (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) as rnk
    FROM Employee
)
SELECT salary AS NthHighestSalary
FROM RankedSalaries
WHERE rnk = :N
LIMIT 1;
```

---

### Q6: Department Highest Salary.

**Schema:** `Employee (id, name, salary, departmentId)`, `Department (id, name)`

**Answer:**

```sql
WITH RankedEmployees AS (
    SELECT 
        d.name AS Department,
        e.name AS Employee,
        e.salary AS Salary,
        RANK() OVER (PARTITION BY e.departmentId ORDER BY e.salary DESC) as rnk
    FROM Employee e
    JOIN Department d ON e.departmentId = d.id
)
SELECT Department, Employee, Salary
FROM RankedEmployees
WHERE rnk = 1;
```

---

### Q7: Rank Scores (without gaps).

**Schema:** `Scores (id, score)`
*Task: Rank scores from highest to lowest. If there is a tie, they should have the same rank. The next rank should be the next consecutive integer (no gaps).*

**Answer:**

```sql
SELECT 
    score,
    DENSE_RANK() OVER (ORDER BY score DESC) as "rank"
FROM Scores;
```

---

### Q8: Find consecutive available seats in a cinema.

**Schema:** `Cinema (seat_id, free)` — where `free` is 1 (free) or 0 (occupied).
*Task: Find all consecutive available seats (2 or more).*

**Answer:**

```sql
SELECT DISTINCT c1.seat_id
FROM Cinema c1 JOIN Cinema c2
  ON ABS(c1.seat_id - c2.seat_id) = 1
  AND c1.free = 1 AND c2.free = 1
ORDER BY c1.seat_id;

-- Alternative using Window Functions (LAG/LEAD)
WITH SeatStatus AS (
    SELECT 
        seat_id,
        free,
        LAG(free) OVER (ORDER BY seat_id) as prev_free,
        LEAD(free) OVER (ORDER BY seat_id) as next_free
    FROM Cinema
)
SELECT seat_id
FROM SeatStatus
WHERE free = 1 AND (prev_free = 1 OR next_free = 1);
```

---

## 🔴 Hard (Advanced / MAANG-level)

### Q9: Active Users (Rolling Window).

**Schema:** `Logins (id, login_date)`
*Task: Find users who logged in for 5 or more consecutive days.*

**Answer:**

```sql
WITH DistinctLogins AS (
    SELECT DISTINCT id, login_date 
    FROM Logins
),
RankedLogins AS (
    SELECT 
        id,
        login_date,
        -- Subtracting the row number (in days) from the date groups consecutive dates!
        login_date - INTERVAL '1 day' * ROW_NUMBER() OVER (PARTITION BY id ORDER BY login_date) as date_group
    FROM DistinctLogins
)
SELECT DISTINCT id 
FROM RankedLogins
GROUP BY id, date_group
HAVING COUNT(*) >= 5;
```

---

### Q10: Human Traffic of Stadium (3+ consecutive rows meeting a condition).

**Schema:** `Stadium (id, visit_date, people)`
*Task: Display records with 3 or more consecutive rows where people >= 100.*

**Answer:**

```sql
WITH GroupedStadium AS (
    SELECT 
        id, 
        visit_date, 
        people,
        id - ROW_NUMBER() OVER (ORDER BY id) as grp
    FROM Stadium
    WHERE people >= 100
),
CountedGroups AS (
    SELECT 
        id, visit_date, people,
        COUNT(*) OVER (PARTITION BY grp) as grp_count
    FROM GroupedStadium
)
SELECT id, visit_date, people
FROM CountedGroups
WHERE grp_count >= 3
ORDER BY visit_date;
```

---

### Q11: Department Top Three Salaries.

**Schema:** `Employee (id, name, salary, departmentId)`, `Department (id, name)`

**Answer:**

```sql
WITH RankedSalaries AS (
    SELECT 
        d.name AS Department,
        e.name AS Employee,
        e.salary AS Salary,
        DENSE_RANK() OVER (PARTITION BY e.departmentId ORDER BY e.salary DESC) as rnk
    FROM Employee e
    JOIN Department d ON e.departmentId = d.id
)
SELECT Department, Employee, Salary
FROM RankedSalaries
WHERE rnk <= 3;
```

---

### Q12: Finding User Retention (Month-over-Month).

**Schema:** `UserActions (user_id, action_date, action_type)`
*Task: Calculate month-over-month retention rate. Retention is defined as a user performing an action in month N and also in month N+1.*

**Answer:**

```sql
WITH MonthlyActiveUsers AS (
    SELECT DISTINCT 
        user_id, 
        DATE_TRUNC('month', action_date) AS activity_month
    FROM UserActions
),
RetentionData AS (
    SELECT 
        m1.activity_month AS current_month,
        COUNT(DISTINCT m1.user_id) AS current_users,
        COUNT(DISTINCT m2.user_id) AS retained_users
    FROM MonthlyActiveUsers m1
    LEFT JOIN MonthlyActiveUsers m2 
        ON m1.user_id = m2.user_id 
        AND m1.activity_month + INTERVAL '1 month' = m2.activity_month
    GROUP BY m1.activity_month
)
SELECT 
    current_month,
    current_users,
    retained_users,
    ROUND((retained_users::decimal / current_users) * 100, 2) AS retention_rate
FROM RetentionData
ORDER BY current_month;
```

---

*End of SQL Coding — 12 problems covering aggregation, window functions, gap-and-island problems, and date manipulation.*
