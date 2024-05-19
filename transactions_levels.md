## Working with transaction isolation levels PostgreSQL


Preparations  
```sql
-- create a new database
CREATE DATABASE otus;

-- connect to it
\c otus

-- create and populate the persons table
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 

-- disable autocommit
\set AUTOCOMMIT OFF

```

Check that the data is in place  
```sql
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```
Check that the global transaction isolation level is **read committed**  
```sql
otus=*# show transaction isolation level;
transaction_isolation 
+-----------------------------+
 read committed
(1 row)

```

#Т1
```sql
begin;
insert into persons(first_name, second_name) values('sergey', 'sergeev');

```
#Т2
```sql
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```
-- **Question**: Do you see the new record and if yes, why?
-- **Answer**: The new record is absent because T1 has not committed yet.

#Т1
```sql
commit;
```

#Т2
```sql
select * from persons
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

#T2
```sql
commit;
```

- **Question**: Do you see the new record and if yes, why?
- **Answer**: The new record is present because T1 has committed.

```sql
begin;
alter database otus set default_transaction_isolation to 'repeatable read';
commit;
```

#T1
```sql
begin;
insert into persons(first_name, second_name) values('sveta', 'svetova');
```

#T2
```sql
begin;
select * from persons;

 id | first_name | second_name 
----+-------------+-------------
  1 | ivan         | ivanov
  2 | petr          | petrov
  3 | sergey     | sergeev
(3 rows)
```

- **Question**: Do you see the new record and if yes, why?
- **Answer**: No, we see the state of the table at the beginning of T2.

#T1
```sql
commit;
```

#T2
```sql
select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

- **Question**: Do you see the new record and if yes, why?
- **Answer**: No, we see the state of the table at the beginning of T2.

#T2
```sql
commit;
```

#T2
```sql
begin;
select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

- **Question**: Do you see the new record and if yes, why?
- **Answer**: Starting a new transaction for reading, we get the current changes with the added person 'sveta' since T1 has committed.