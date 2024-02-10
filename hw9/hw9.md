# Домашнее задание

## Бэкапы


### 1. Создаем ВМ/докер c ПГ.
Создан инстанс ec2 aws
### 2. Создаем БД, схему и в ней таблицу.
```
postgres=# CREATE DATABASE otus;
CREATE DATABASE
otus=# CREATE SCHEMA backups;
CREATE SCHEMA
```
### 3. Заполним таблицы автосгенерированными 100 записями.
```
otus=# CREATE TABLE backups.employees_1 as select generate_series(1, 100) as id, md5(random()::text)::char(10) as fio;
SELECT 100
otus=# SELECT * FROM backups.employees_1;
SELECT 100
 id  |    fio     
-----+------------
   1 | 9396f1306c
   2 | e0b5f95640
   3 | dfad916bd6
   4 | 8f957dc9d7
   5 | b09a1894da
   6 | 0e357c8506
   7 | 30afa68925
   8 | 37b59c5c2f
   9 | 30b3d4bb38
  10 | 524bd2ece7
  ....
 100 | a908aaa8b4
(100 rows)

```

### 4. Под линукс пользователем Postgres создадим каталог для бэкапов
```
postgres@ip-172-31-44-4:/mnt/pg_backups$ pwd
/mnt/pg_backups
```
### 5. Сделаем логический бэкап используя утилиту COPY
```
otus=# \copy backups.employees_1 to '/mnt/my_backups/employees_copy_backup.sql'
COPY 100
```
### 6. Восстановим в 2 таблицу данные из бэкапа.
```
otus=# create table backups.employees_2 (id serial, fio varchar(10));
CREATE TABLE
otus=# \copy backups.employees_2 from '/mnt/my_backups/employees_copy_backup.sql'
COPY 100
otus=# SELECT * FROM backups.employees_2;
 id  |    fio     
-----+------------
   1 | 9396f1306c
   2 | e0b5f95640
   3 | dfad916bd6
   4 | 8f957dc9d7
   6 | 0e357c8506
   7 | 30afa68925
   8 | 37b59c5c2f
   9 | 30b3d4bb38
  10 | 524bd2ece7
  ....
 100 | a908aaa8b4
(100 rows)

```
### 7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

```
postgres@ip-172-31-44-4:/$ pg_dump --dbname=otus --schema=backups --table=backups.employees_* --format=c --file=/mnt/my_backups/employees_pgdump_backup.sql
postgres@ip-172-31-44-4:/mnt/my_backups$ pg_restore -l employees_pgdump_backup.sql
;
; Archive created at 2024-02-10 18:41:48 UTC
;     dbname: otus
;     TOC Entries: 12
;     Compression: -1
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 15.5 (Ubuntu 15.5-1.pgdg22.04+1)
;     Dumped by pg_dump version: 15.5 (Ubuntu 15.5-1.pgdg22.04+1)
;
;
; Selected TOC Entries:
;
219; 1259 16402 TABLE backups employees_1 postgres
221; 1259 16406 TABLE backups employees_2 postgres
220; 1259 16405 SEQUENCE backups employees_2_id_seq postgres
3343; 0 0 SEQUENCE OWNED BY backups employees_2_id_seq postgres
3191; 2604 16409 DEFAULT backups employees_2 id postgres
3334; 0 16402 TABLE DATA backups employees_1 postgres
3336; 0 16406 TABLE DATA backups employees_2 postgres
3344; 0 0 SEQUENCE SET backups employees_2_id_seq postgres

```

### 8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!
```
postgres=# CREATE DATABASE otus_another;
CREATE DATABASE
postgres=# CREATE SCHEMA backups;
CREATE SCHEMA
```
```
postgres@ip-172-31-44-4:/mnt/my_backups$ pg_restore --dbname=otus_another --table=employees_2 employees_pgdump_backup.sql
```

```
postgres=# \c otus_another
You are now connected to database "otus_another" as user "postgres".

 id  |    fio     
-----+------------
   1 | 9396f1306c
   2 | e0b5f95640
   3 | dfad916bd6
   4 | 8f957dc9d7
   5 | b09a1894da
   6 | 0e357c8506
   7 | 30afa68925
   8 | 37b59c5c2f
   9 | 30b3d4bb38
  10 | 524bd2ece7
  ....
 100 | a908aaa8b4
(100 rows)
```

**Выводы:**
Все данные были корректно восстановленны в таблицу employees_2