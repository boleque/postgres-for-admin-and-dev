# Домашнее задание
## Работа с базами данных, пользователями и правами

1. создайте новый кластер PostgresSQL 14
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-14

```
2. зайдите в созданный кластер под пользователем postgres
```
sudo -u postgres psql
```
3. создайте новую базу данных testdb
```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
```
4. зайдите в созданную базу данных под пользователем postgres
```
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
```
5. создайте новую схему testnm
```
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
```
6. создайте новую таблицу t1 с одной колонкой c1 типа integer
```
testdb=# CREATE TABLE t1(c1 integer);
CREATE TABLE
```
7. вставьте строку со значением c1=1
```
testdb=# INSERT INTO t1 values(1);
INSERT 0 1
```
8. создайте новую роль readonly
```
testdb=# CREATE role readonly;
CREATE ROLE
```
9. дайте новой роли право на подключение к базе данных testdb
```
testdb=# GRANT CONNECT on DATABASE testdb TO readonly;
GRANT
```
10. дайте новой роли право на использование схемы testnm
```
testdb=# GRANT USAGE on SCHEMA testnm to readonly;
GRANT
```
11. дайте новой роли право на select для всех таблиц схемы testnm
```
testdb=# GRANT SELECT on all TABLEs in SCHEMA testnm TO readonly;
GRANT
```
12. создайте пользователя testread с паролем test123
```
testdb=# CREATE USER testread with password 'test123';
CREATE ROLE
```
13. дайте роль readonly пользователю testread
```
testdb=# GRANT readonly TO testread;
GRANT ROLE
```
14. зайдите под пользователем testread в базу данных testdb
```
psql -h 127.0.0.1 -U testread -d testdb -W
```
15. сделайте select * from t1;
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```

16. получилось?
```
Выборка не прошла. 
Таблица t1 была создана в схеме public, a не testnm (на которую и были выданы права). 
Привилегий на чтение таблицы, созданной пользователей postgres, у пользователя testdb нет.
```
```
testdb=> \dt public.*
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=> \dt testnm.*
Did not find any relation named "testnm.*".
```

17. вернитесь в базу данных testdb под пользователем postgres
18. удалите таблицу t1
```
testdb=# DROP table t1;
DROP TABLE
```
19. создайте ее заново но уже с явным указанием имени схемы testnm
```
testdb=# CREATE TABLE testnm.t1(c1 integer);
CREATE TABLE
```
20. вставьте строку со значением c1=1
```
testdb=# INSERT INTO testnm.t1 values(1);
INSERT 0 1
```
21. зайдите под пользователем testread в базу данных testdb
22. сделайте select * from testnm.t1;
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
23. получилось?
```
Выборка не прошла. 
Право на select было выдано на табоицу t1 ранее удаленную. 
На пересозданную таблицу какие либо права отсутствуют.
```
24. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
```
Можно использовать:
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO testread;
Позволит установить привилегии для всех объектов созданных в будущем (но в данный момент проблему выборки из t1 не решит)
```
```
Другой вариант повторно вызвать:
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
Позволит установить привилегии на существующие, в момент вызова команды, объекты (позволит провести выборку из t1)
```
25. сделайте select * from testnm.t1;
26. получилось?
```
-- после вызова
testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
-- проверям select
testdb=> select * from testnm.t1;
 c1 
----
  1
(1 row)
```
27. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
```
testdb=# create table t2(c1 integer);
CREATE TABLE
testdb=# insert into t2 values (2);
INSERT 0 1
```
28. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
```
Таблица была создана в схеме public.
А наличие роли public (grant на все операции) для всех созданных пользователей, позволяет testread создавать таблицу
```
```
testdb=> \dt public.*
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t2   | table | testread
(1 row)
```
29. есть идеи как убрать эти права?
```
REVOKE CREATE on SCHEMA public FROM public; 
```
31. попробуйте выполнить команду create table t3(c1 integer); insert into t3 values (2);
```
testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
```
32. расскажите что получилось и почему
```
Была отозвана привилегия на создание (CREATE) таблиц для роли public. 
Таким образом пользователь testread, имеющий роль public, более не имеет возможности создавать таблицы для схемы public.
```