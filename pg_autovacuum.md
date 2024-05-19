## Setting up autovacuum considering performance features

1. Create a VM instance with 2 cores, 4 GB RAM, and 10GB SSD
```
Создан ec2 инстанс в aws
```
2. Install PostgreSQL 15 with default settings
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
```
```
ubuntu@ip-172-31-35-134:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
```
3. Create a test database: execute `pgbench -i postgres`
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
4. Run `pgbench -c8 -P 6 -T 60 -U postgres postgres`

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

5. Apply PostgreSQL configuration parameters from the attached class file
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

6. Test again
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

7. What has changed and why?

Applying the settings on the machine being used did not result in any significant changes. 
The exception is the initial connection time, which changed from 21.620 ms (before) to 50.669 ms (after), which I attribute to cluster restart since subsequent pgbench runs returned a value close to 21.620 ms.

However, if you calculate the total memory consumption by the cluster, it becomes clear that the memory exceeds the available 4 GB RAM on the machine.

Memory allocated to one backend process unit is:
`work_mem + maintenance_work_mem + temp_buffers = 6MB + 512MB + 8MB = 526 MB per process.`
With 8 clients, this totals to `526 * 8 = 4.026 GB`.

Additionally, 1GB is allocated to `shared_buffers`.

Therefore, a rough calculation indicates that the cluster will consume approximately 5 GB of memory with 8 clients connected. The swapping mechanism in the OS certainly addresses the issue of virtual memory shortage, but potential latency increase remains unnoticed within the framework
The completion of the pgbench utility run.

8. Create a table with a text field and populate it with randomly or generated data with 1 million rows.

```
otus=# CREATE TABLE t1(data text);
CREATE TABLE

otus=# INSERT INTO t1 (data)
SELECT substr(md5(random()::text), 1, 10) FROM generate_series(1, 1000000);
INSERT 0 1000000
```
9. Check the size of the table file
```
otus=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 42 MB
(1 row)
```
10. Update all rows 5 times and add any character to each row.
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
11. Check the number of dead rows in the table and when autovacuum last ran
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
12. Wait for some time, checking if autovacuum has come
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
13. Update all rows 5 times and add any symbol to each row
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
14. Check the size of the table
```
otus=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 253 MB
(1 row)
```
15. Disable autovacuum on the specific table
```
otus=# ALTER TABLE t1 SET (autovacuum_enabled = false);
ALTER TABLE
```
16. Update all rows 10 times and add any symbol to each row
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
17. Check the size of the table
```
test_vacuum=# SELECT pg_size_pretty(pg_TABLE_size('t1'));
 pg_size_pretty 
----------------
 601 MB
(1 row)
```

18. Explain the result obtained
```
The use of PostgreSQL's tuple multi versioning mechanism leads to the behavior where an UPDATE operation keeps the current version of the row (dead) and creates a new version with the updated data.

1st update of rows 5 times in a row - 1,000,000 live rows and ~5,000,000 dead rows.
Running AUTOVACUUM removes the dead rows but does not free up space; hence, the table size remains 253 MB both before and after AUTOVACUUM.

2nd update of rows 5 times in a row - the table size remains unchanged at 253 MB, as PostgreSQL reuses the space occupied by the dead rows freed up by AUTOVACUUM.

3rd update of rows 10 times in a row (with autovacuum disabled) - dead rows are not cleaned up, and inserting new versions of rows as data is updated causes the table size to grow to 601 MB.


```
