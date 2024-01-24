# Домашнее задание
## Настройка autovacuum с учетом особеностей производительности

1. Cоздать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
```
Создан ec2 инстанс в aws
```
2. Установить на него PostgreSQL 15 с дефолтными настройками
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
```
ubuntu@ip-172-31-35-134:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
```
3. Создать БД для тестов: выполнить pgbench -i postgres
```
root@ip-172-31-35-134:/home/ubuntu# su postgres
postgres@ip-172-31-35-134:/home/ubuntu$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.10 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.53 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.14 s, vacuum 0.04 s, primary keys 0.34 s).
```
4. Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres

```
postgres@ip-172-31-44-224:/home/ubuntu$ pgbench -c 8 -P 6 -T 60 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 787.3 tps, lat 10.111 ms stddev 7.096, 0 failed
progress: 12.0 s, 831.5 tps, lat 9.607 ms stddev 5.923, 0 failed
progress: 18.0 s, 742.7 tps, lat 10.775 ms stddev 6.853, 0 failed
progress: 24.0 s, 709.8 tps, lat 11.271 ms stddev 7.340, 0 failed
progress: 30.0 s, 703.5 tps, lat 11.372 ms stddev 7.368, 0 failed
progress: 36.0 s, 733.7 tps, lat 10.906 ms stddev 6.986, 0 failed
progress: 42.0 s, 775.5 tps, lat 10.318 ms stddev 7.814, 0 failed
progress: 48.0 s, 779.3 tps, lat 10.265 ms stddev 6.526, 0 failed
progress: 54.0 s, 749.7 tps, lat 10.662 ms stddev 6.379, 0 failed
progress: 60.0 s, 769.2 tps, lat 10.404 ms stddev 6.249, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 45501
number of failed transactions: 0 (0.000%)
latency average = 10.545 ms
latency stddev = 6.882 ms
initial connection time = 21.620 ms
tps = 758.422316 (without initial connection time)
```

5. Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
```
done
```

```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```

6. Протестировать заново
```
postgres@ip-172-31-43-191:/home/ubuntu$ pgbench -c 8 -P 6 -T 60 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 696.7 tps, lat 11.364 ms stddev 7.949, 0 failed
progress: 12.0 s, 732.6 tps, lat 10.920 ms stddev 7.104, 0 failed
progress: 18.0 s, 724.3 tps, lat 11.051 ms stddev 7.604, 0 failed
progress: 24.0 s, 777.5 tps, lat 10.289 ms stddev 6.296, 0 failed
progress: 30.0 s, 789.7 tps, lat 10.130 ms stddev 5.685, 0 failed
progress: 36.0 s, 769.8 tps, lat 10.390 ms stddev 6.137, 0 failed
progress: 42.0 s, 779.3 tps, lat 10.263 ms stddev 6.348, 0 failed
progress: 48.0 s, 774.7 tps, lat 10.328 ms stddev 6.976, 0 failed
progress: 54.0 s, 800.3 tps, lat 9.994 ms stddev 5.363, 0 failed
progress: 60.0 s, 736.8 tps, lat 10.851 ms stddev 7.216, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 45498
number of failed transactions: 0 (0.000%)
latency average = 10.541 ms
latency stddev = 6.699 ms
initial connection time = 50.669 ms
tps = 758.663805 (without initial connection time)
```
7. Что изменилось и почему?

Применение настроек на используемой машине не привело к каким либо существенным изменениям.
Исключение составляет initial connection time, который изменился с 21.620 ms (до) на 50.669 ms (после). 
Связываю это с рестартом кластера, т.к. повторный прогон pgbench выдал схожее с 21.620 ms значение.

Однако, если произвести вычисление суммарно потребляемой памяти кластером, становится понятно, что память превышает имеющиеся на машине 4 GB RAM.

Память, выделенная для единицы backend process, составляет:
work_mem + maintenance_work_mem + temp_buffers = 6MB + 512MB + 8MB = 526 MB на процесс.
Клиентов 8, соответственно 526 * 8 = 4.026 GB

На shared_buffers выделяем 1GB

Таким образом приблизительный расчет дает 5 GB памяти, которую кластер будет потреблять при подключении 8 клиентов. 
Механизм swapping, существующий в ОС, безусловно решает проблему нехватки виртуальной памяти, однако возможное повышение latency остается незамеченным в рамках
прогона утилиты pgbench

8. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк
```
otus=# CREATE TABLE t1(data text);
CREATE TABLE

otus=# INSERT INTO t1 (data)
SELECT substr(md5(random()::text), 1, 10) FROM generate_series(1, 1000000);
INSERT 0 1000000
```
9. Посмотреть размер файла с таблицей
```
otus=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 42 MB
(1 row)
```
10. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
otus=# UPDATE t1 SET data=CONCAT(data, 'a');
UPDATE t1 SET data=CONCAT(data, 'b');
UPDATE t1 SET data=CONCAT(data, 'c');
UPDATE t1 SET data=CONCAT(data, 'd');
UPDATE t1 SET data=CONCAT(data, 'e');
UPDATE 1000000                                                                                     
UPDATE 1000000                                                                              
UPDATE 1000000                                                                       
UPDATE 1000000
UPDATE 1000000 
```
11. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
otus=# SELECT relname, n_live_tup, n_dead_tup, last_autovacuum 
FROM pg_stat_user_tables 
WHERE relname = 't1';
 relname | n_live_tup | n_dead_tup |        last_autovacuum        
---------+------------+------------+-------------------------------
 t1      |    1000000 |    4999894 | 2024-01-06 15:03:18.357369+00
(1 row)
otus=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 253 MB
(1 row)
```
12. Подождать некоторое время, проверяя, пришел ли автовакуум
```
otus=# SELECT relname, n_live_tup, n_dead_tup, last_autovacuum 
FROM pg_stat_user_tables 
WHERE relname = 't1';
 relname | n_live_tup | n_dead_tup |       last_autovacuum        
---------+------------+------------+------------------------------
 t1      |    1000000 |          0 | 2024-01-06 15:08:19.27044+00
(1 row)

otus=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 253 MB
(1 row)
```
13. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```
otus=# UPDATE t1 SET data=CONCAT(data, 'a');
UPDATE t1 SET data=CONCAT(data, 'b');
UPDATE t1 SET data=CONCAT(data, 'c');
UPDATE t1 SET data=CONCAT(data, 'd');
UPDATE t1 SET data=CONCAT(data, 'e');
UPDATE 1000000                                                                                     
UPDATE 1000000                                                                              
UPDATE 1000000                                                                       
UPDATE 1000000
UPDATE 1000000 
```
14. Посмотреть размер файла с таблицей
```
otus=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 253 MB
(1 row)
```
15. Отключить Автовакуум на конкретной таблице
```
otus=# ALTER TABLE t1 SET (autovacuum_enabled = false);
ALTER TABLE
```
16. 10 раз обновить все строчки и добавить к каждой строчке любой символ
```
UPDATE t1 SET data=CONCAT(data, '1');
UPDATE t1 SET data=CONCAT(data, '2');
UPDATE t1 SET data=CONCAT(data, '3');
UPDATE t1 SET data=CONCAT(data, '4');
UPDATE t1 SET data=CONCAT(data, '5');
UPDATE t1 SET data=CONCAT(data, '6');
UPDATE t1 SET data=CONCAT(data, '7');
UPDATE t1 SET data=CONCAT(data, '8');
UPDATE t1 SET data=CONCAT(data, '9');
UPDATE t1 SET data=CONCAT(data, '10');
```
17. Посмотреть размер файла с таблицей
```
test_vacuum=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 601 MB
(1 row)
```

18. Объясните полученный результат
```
Использование в PG механизма многоверсионности (tuple multi versioning) приводит к тому, что
операция UPDATE оставляет текущую версию строки (dead) и создает новую версию с обновленными данными.

1е обновление строк 5 раз подряд - 1_000_000 актуальных строк и ~ 5_000_000 мертвых строк.
Вызов AUTOVACUUM удаляет мертвые строки, однако не освобождает память, размер таблицы как до так и после AUTOVACUUM составляет 253 MB. 

2е обновление строк 5 раз подряд - размер таблицы не изменяется и составляет 253 MB, это объясняется тем, что
PG переиспользует место, занимаемое освобожденными в результате AUTOVACUUM, строками.

3е обновление строк 10 раз подряд (при отключенном AUTOVACUUM) - мертвые строки не очищаются, вставка новых версий строк, при обновлении данных, приводит к росту размера таблицы до 601 MB.

```
