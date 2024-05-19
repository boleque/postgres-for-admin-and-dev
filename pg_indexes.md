## Working with Indexes


### Preliminary Work
```
otus=# CREATE TABLE employees (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        employee_num TEXT NOT NULL,
        first_name TEXT NOT NULL,
        last_name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
        is_active boolean
);
CREATE TABLE

otus=# INSERT INTO employees (employee_num, first_name, last_name, email, is_active)
SELECT uuid, first_name, last_name, first_name || '.' || last_name || '_' || increasing_num || '@group.com' as email, is_active
FROM (
SELECT
        s.a as increasing_num,
        uuid_in(md5(random()::text || random()::text)::cstring) as uuid,
    arrays.firstnames[s.a % ARRAY_LENGTH(arrays.firstnames,1) + 1] AS first_name,
    arrays.lastnames[s.a % ARRAY_LENGTH(arrays.lastnames,1) + 1] AS last_name,
        random() < 0.01 as is_active 
FROM generate_series(1, 100000) AS s(a)
CROSS JOIN(
    SELECT ARRAY[
    'Adam','Bill','Bob','Calvin','Donald','Dwight','Frank','Fred','George','Howard',
    'James','John','Jacob','Jack','Martin','Matthew','Max','Michael',
    'Paul','Peter','Phil','Roland','Ronald','Samuel','Steve','Theo','Warren','William',
    'Abigail','Alice','Allison','Amanda','Anne','Barbara','Betty','Carol','Cleo','Donna',
    'Jane','Jennifer','Julie','Martha','Mary','Melissa','Patty','Sarah','Simone','Susan'
    ] AS firstnames,
    ARRAY[
        'Matthews','Smith','Jones','Davis','Jacobson','Williams','Donaldson','Maxwell','Peterson','Stevens',
        'Franklin','Washington','Jefferson','Adams','Jackson','Johnson','Lincoln','Grant','Fillmore','Harding','Taft',
        'Truman','Nixon','Ford','Carter','Reagan','Bush','Clinton','Hancock'
    ] AS lastnames
) AS arrays) as tmp;
INSERT 0 100000

-- statistics for the table

otus=# SELECT attname, correlation FROM pg_stats WHERE tablename = 'employees';
   attname    |  correlation  
--------------+---------------
 id           |             1
 employee_num | 3.3436172e-05
 first_name   |   0.017357923
 last_name    |   0.037830837
 email        | -0.0028235011
 is_active    |     0.9828457
(6 rows)

```

### 1. Create an Index on a Table in Your Database. Provide the Result of the explain Command Using This Index.

```
-- the cost of the query on the non-indexed column is 2405.85, Seq Scan is used
EXPLAIN SELECT * FROM employees WHERE employee_num = '8b4ffc25-17c2-3640-6e04-640dc7954615';

otus=# EXPLAIN SELECT * FROM employees WHERE employee_num = '8b4ffc25-17c2-3640-6e04-640dc7954615';
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Seq Scan on employees  (cost=0.00..2405.85 rows=366 width=137)
   Filter: (employee_num = '8b4ffc25-17c2-3640-6e04-640dc7954615'::text)
(2 rows)

-- строим по employee_num индекс
otus=# CREATE INDEX ON employees(employee_num);
CREATE INDEX

-- повторяем запрос, стоимость запроса 8.44, используется Index Scan
otus=# EXPLAIN SELECT * FROM employees WHERE employee_num = '8b4ffc25-17c2-3640-6e04-640dc7954615';
                                         QUERY PLAN                                          
---------------------------------------------------------------------------------------------
 Index Scan using employees_employee_num_idx on employees  (cost=0.42..8.44 rows=1 width=88)
   Index Cond: (employee_num = '8b4ffc25-17c2-3640-6e04-640dc7954615'::text)
(2 rows)

```

### 2. Implement a Full-Text Search Index

```
-- Implement a query to search for employees with the name Bill, the cost of the query was 4772.00
otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE first_name LIKE 'Bill%';
                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on employees  (cost=0.00..4772.00 rows=2063 width=42)
   Filter: (first_name ~~ 'Bill%'::text)
(2 rows)


-- Add a new column to the employees table, which will contain the normalized form of the email address

otus=# ALTER TABLE employees ADD COLUMN email_vector tsvector;
ALTER TABLE

-- Populate the column 
otus=# UPDATE employees SET email_vector = to_tsvector('english', email);
UPDATE 100000


-- Create a GIN index for the email_vector column

otus=# CREATE INDEX idx_emails_vector ON employees USING gin(email_vector);
CREATE INDEX

-- Analyze the query, the cost of the query was 3187.81
otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE email_vector @@ to_tsquery('english', 'bill:*');
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=39.50..3187.81 rows=2000 width=42)
   Recheck Cond: (email_vector @@ '''bill'':*'::tsquery)
   ->  Bitmap Index Scan on idx_emails_vector  (cost=0.00..39.00 rows=2000 width=0)
         Index Cond: (email_vector @@ '''bill'':*'::tsquery)
(4 rows)
```

### 3. Implement an Index on Part of a Table or a Functional Index
```
-- Implement a query to search for names starting with 'a' in lower case, analyze and get the cost of 5272.00
otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE lower(substr(first_name, 1, 1)) = 'a';
                          QUERY PLAN                           
---------------------------------------------------------------
 Seq Scan on employees  (cost=0.00..5272.00 rows=500 width=42)
   Filter: (lower(substr(first_name, 1, 1)) = 'a'::text)
(2 rows)
-- Create a functional index
otus=# CREATE INDEX idx_first_name_first_letter_lower ON employees ((lower(substr(first_name, 1, 1))));
CREATE INDEX

-- Analyze the query, the cost of the query was 1374.76
otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE lower(substr(first_name, 1, 1)) = 'a';
                                            QUERY PLAN                                            
--------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=8.17..1374.76 rows=500 width=42)
   Recheck Cond: (lower(substr(first_name, 1, 1)) = 'a'::text)
   ->  Bitmap Index Scan on idx_first_name_first_letter_lower  (cost=0.00..8.04 rows=500 width=0)
         Index Cond: (lower(substr(first_name, 1, 1)) = 'a'::text)
(4 rows)

```
### 4. Create an Index on Multiple Fields

```
-- Implement a query to search for employees with the name Bill and the last name Smith, analyze the cost, which is 5022.00
otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE first_name = 'Bill' AND last_name = 'Smith';
                               QUERY PLAN                                
-------------------------------------------------------------------------
 Seq Scan on employees  (cost=0.00..5022.00 rows=70 width=42)
   Filter: ((first_name = 'Bill'::text) AND (last_name = 'Smith'::text))
(2 rows)

-- Create an index on 2 columns: first_name, last_name

otus=# CREATE INDEX idx_first_last_name ON employees(first_name, last_name);
CREATE INDEX

-- Analyze the original query, the cost is 256.45

otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE first_name = 'Bill' AND last_name = 'Smith';
                                    QUERY PLAN                                     
-----------------------------------------------------------------------------------
 Bitmap Heap Scan on employees  (cost=5.01..256.45 rows=70 width=42)
   Recheck Cond: ((first_name = 'Bill'::text) AND (last_name = 'Smith'::text))
   ->  Bitmap Index Scan on idx_first_last_name  (cost=0.00..4.99 rows=70 width=0)
         Index Cond: ((first_name = 'Bill'::text) AND (last_name = 'Smith'::text))
(4 rows)


-- The cost of a query only by the last_name column is 4772.00

otus=# EXPLAIN SELECT first_name, last_name, email FROM employees WHERE last_name = 'Smith';
                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on employees  (cost=0.00..4772.00 rows=3390 width=42)
   Filter: (last_name = 'Smith'::text)
(2 rows)

```
**Conclusions:**
- The index on the low cardinality field employee_num (correlation 3.3436172e-05) reduced the query cost from 2405.85 to 8.44.
- Using a GIN index for searching by email reduced the cost from 4772.00 to 3187.81, which is approximately a 34% reduction.
- The functional index for searching by the first letter of the name in lower case reduced the query cost from 5272.00 to 1374.76.
- The composite index on first_name and last_name reduced the query cost from 5022.00 to 256.45. It's worth noting that querying only by last_name leads to a sequential scan with a cost of 4772.00. There's a mention in literature that using filtering not in the order defined in the index creation statement also doesn't reduce the query cost, but this couldn't be confirmed. Additionally, a Bitmap Heap Scan is used with a cost of 256.45.