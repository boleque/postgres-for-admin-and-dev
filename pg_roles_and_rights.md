## Working with Databases, Users, and Permissions

1. Create a new PostgreSQL 14 cluster
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

```
2. Log in to the created cluster as postgres user
```
sudo -u postgres psql
```
3. Create a new database testdb
```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```
4. Log in to the created database as postgres user
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
5. Create a new schema testnm
```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```
6. Create a new table t1 with one column c1 of type integer
```
testdb=# CREATE TABLE t1(c1 integer);
CREATE TABLE
```
7. Insert a row with value c1=1
```
testdb=# INSERT INTO t1 values(1);
INSERT 0 1
```
8. Create a new role readonly
```
testdb=# CREATE role readonly;
CREATE ROLE
```
9. Grant the readonly role the CONNECT permission to the testdb database
```
testdb=# GRANT CONNECT on DATABASE testdb TO readonly;
GRANT
```
10. Grant the readonly role the USAGE permission on the testnm schema
 ```
 testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
 GRANT
 ```

11. Grant the readonly role the SELECT permission on all tables in the testnm schema
 ```
 testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
 GRANT
 ```

12. Create a user testread with password 'test123'
 ```
 testdb=# CREATE USER testread WITH PASSWORD 'test123';
 CREATE ROLE
 ```

13. Grant the readonly role to the user testread
 ```
 testdb=# GRANT readonly TO testread;
 GRANT ROLE
 ```

14. Log in as the user testread to the database testdb
 ```
 psql -h 127.0.0.1 -U testread -d testdb -W
 ```

15. Execute select * from t1;
 ```
 testdb=> SELECT * FROM t1;
 ERROR:  permission denied for table t1
 ```

16. Did it work?
 ```
 The query failed. 
 The table t1 was created in the public schema, not in testnm (where permissions were granted). 
 The testread user does not have SELECT privileges on the table created by the postgres user.
 ```

17. Return to the testdb database as the postgres user
18. Drop the table t1
 ```
 testdb=# DROP TABLE t1;
 DROP TABLE
 ```

19. Create the table t1 again, but explicitly specify the testnm schema
 ```
 testdb=# CREATE TABLE testnm.t1(c1 integer);
 CREATE TABLE
 ```

20. Insert a row with value c1=1
 ```
 testdb=# INSERT INTO testnm.t1 VALUES (1);
 INSERT 0 1
 ```

21. Log in as the user testread to the database testdb
22. Execute select * from testnm.t1;
 ```
 testdb=> SELECT * FROM testnm.t1;
 ERROR:  permission denied for table t1
 ```

23. Did it work?
 ```
 The query failed. 
 The SELECT permission was granted to the table t1 previously dropped. 
 No permissions were granted to the newly created table.
 ```

24. How can this be prevented from happening in the future? If you have no ideas, consult the cheat sheet
 ```
 You can use:
 ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO testread;
 This will set privileges for all future objects created (but will not resolve the issue with the current t1 table).
 ```
 ```
 Alternatively, you can re-issue:
 GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
 This will grant privileges to existing objects at the time the command is executed (allowing select from t1).
 ```

25. Execute select * from testnm.t1;
26. Did it work?
 ```
 -- after executing
 testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
 GRANT
 -- check select
 testdb=> SELECT * FROM testnm.t1;
  c1 
 ----
   1
 (1 row)
 ```

27. Now try to execute create table t2(c1 integer); insert into t2 values (2);
 ```
 testdb=# CREATE TABLE t2(c1 integer);
 CREATE TABLE
 testdb=# INSERT INTO t2 VALUES (2);
 INSERT 0 1
 ```

28. How is this possible? Nobody has permissions to create tables and insert into them under the readonly role?
 ```
 The table was created in the public schema.
 The presence of the public role (which grants all privileges) for all created users allows testread to create a table.
 ```
 ```
 testdb=> \dt public.*
         List of relations
  Schema | Name | Type  |  Owner   
 --------+------+-------+----------
  public | t2   | table | testread
 (1 row)
 ```

29. Any ideas on how to remove these permissions?
 ```
 REVOKE CREATE ON SCHEMA public FROM public;
 ```

31. Try to execute create table t3(c1 integer); insert into t3 values (2);
 ```
 testdb=# CREATE TABLE t3(c1 integer);
 ERROR:  permission denied for schema public
 LINE 1: CREATE TABLE t3(c1 integer);
 ```

32. Explain what happened and why
 ```
 The CREATE permission was revoked for the public role. 
 Thus, the testread user, which has the public role, no longer has the ability to create tables in the public schema.
 ```
