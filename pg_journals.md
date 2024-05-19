## Working with Journals  

1. Set up a checkpoint to be executed every 30 seconds.  
```
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
```
2. Provide a workload for 10 minutes using the pgbench utility.  
```
postgres@ip-172-31-34-121:/home/ubuntu$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.15 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.39 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.19 s, vacuum 0.05 s, primary keys 0.14 s).
```
```
postgres@ip-172-31-34-121:/home/ubuntu$ pgbench -c 8 -P 6 -T 600 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))                                                                                                                                                                                                                         
starting vacuum...end.                                                                                                                                                                                                                                             
progress: 6.0 s, 1014.5 tps, lat 7.837 ms stddev 4.634, 0 failed                                                                                                                                                                                                   
progress: 12.0 s, 907.5 tps, lat 8.821 ms stddev 7.454, 0 failed        
...
progress: 594.0 s, 791.7 tps, lat 10.103 ms stddev 5.976, 0 failed
progress: 600.0 s, 826.7 tps, lat 9.677 ms stddev 5.180, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 488198
number of failed transactions: 0 (0.000%)
latency average = 9.830 ms
latency stddev = 6.021 ms
initial connection time = 27.227 ms
tps = 813.669920 (without initial connection time)
```
3. Measure the volume of WAL files generated during this time.  
```
-- https://gist.github.com/lesovsky/4587d70f169739c01d4525027c087d14

     Uptime      | Since stats reset | Forced checkpoint ratio (%) | Minutes between checkpoints | Average write time per checkpoint (s) | Average sync time per checkpoint (s) | Total MB written | MB per checkpoint 
-----------------+-------------------+-----------------------------+-----------------------------+---------------------------------------+--------------------------------------+------------------+-------------------
 00:10:56.226604 | 00:11:45.781325   |                         4.5 |                        0.55 |                                 24.48 |                                 0.01 |            377.0 |             15.45
(1 row)

```
4. Evaluate the average amount per checkpoint.  
```
15.45 MB per checkpoint
```
5. Check the statistics: did all checkpoints execute exactly on schedule? Why did this happen?  
```

The specified interval between checkpoints was 30 seconds.

The calculated interval was 33 seconds, and the write time is checkpoint_completion_target x 30 = 0.9 * 30 = 27 seconds.

The checkpoint is written 6 seconds before the next checkpoint starts. Despite the fact that one checkpoint is running at a time, a 6-second buffer is not a reliable solution.

It is worth noting that further increasing the load, especially when decreasing the max_wal_size parameter, can lead to an unplanned checkpoint call by the server, since the allowed volume of log records will be exceeded. An unplanned checkpoint call with a margin of only 6 seconds will result in the checkpoints being "overlapped" on the timeline.

```

6. Compare tps in synchronous/asynchronous mode using the pgbench utility  
```
--- async
latency average = 0.616 ms
initial connection time = 3.639 ms
tps = 1624.534048 (without initial connection time)

-- sync
latency average = 1.726 ms
initial connection time = 3.536 ms
tps = 579.269171 (without initial connection time)

```
7. Explain your result.  
```
In asynchronous commit, response time (latency) decreased (0.616 ms compared to 1.726 ms).
Throughput (tps) increased (1624.5 compared to 579.2).

In synchronous write, the COMMIT command does not return control until the synchronization is complete, thus increasing response time and decreasing system throughput tps. With asynchronous write, changes are committed without waiting for physical disk write.
```
8. Create a new cluster with page checksum enabled.  
```
ubuntu@ip-172-31-34-233:~$ sudo /usr/lib/postgresql/15/bin/pg_checksums -D /var/lib/postgresql/15/main/ -e
Checksum operation completed
```
```
otus=# show data_checksums;
 data_checksums 
----------------
 on
(1 row)
```
9. Create a table. Insert multiple values.  
```
postgres=# CREATE DATABASE otus;
CREATE DATABASE
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov');
CREATE TABLE
INSERT 0 1
INSERT 0 1
```
10. Shut down the cluster.  
```
ubuntu@ip-172-31-34-233:~$ sudo systemctl stop postgresql@15-main
```
11. Change a couple of bytes in the table.  
```
ubuntu@ip-172-31-34-233:~$ sudo sed -i 's/@/@12345@/' /var/lib/postgresql/15/main/base/16388/16390
```
12. Turn on the cluster and make a selection from the table.  
```
ubuntu@ip-172-31-34-233:~$ sudo systemctl start postgresql@15-main
```
```
otus=# select * from persons;
WARNING:  page verification failed, calculated checksum 37049 but expected 3345
ERROR:  invalid page in block 0 of relation base/16388/16390
```
13. What happened and why?  
```
Checksum mechanism is enabled, thus protecting the primary layer of data files.
In our case, the mismatch of checksum values led to the error mentioned above.
```
14. How to ignore the error and continue working?  
```
1. Try to restore data from a backup copy (if available);
2. Try to read the corrupted page (with the risk of obtaining distorted information) using ignore_checksum_failure off.
```