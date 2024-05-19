## Replication

### 1. Create tables on VM1 for writing (test) and reading (test2).
Created EC2 instance on AWS - pg-replication-vm1 (172.31.33.195)

```
-- Create tables, test - for writing, test2 - for reading
postgres=# CREATE TABLE test as select generate_series(1, 100) as id, md5(random()::text) ::char(10) as fio;
SELECT 100
postgres=# CREATE TABLE test2 (id serial, fio varchar(10));
CREATE TABLE
```

### 2. Create a publication for table test and subscribe to the publication of table test2 on VM2.
```
-- Set wal_level to logical 
postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM

-- Restart to apply changes  
ubuntu@ip-172-31-33-195:~$ sudo systemctl restart postgresql@15-main

-- Verify that the changes are applied 
postgres=# show wal_level;
 wal_level 
-----------
 logical 

-- Create publication for table test
postgres=# CREATE PUBLICATION test_publication_vm1 FOR TABLE test;
CREATE PUBLICATION

-- Subscribe to the publication of table test2 on VM2
postgres=# CREATE SUBSCRIPTION test2_subscription_vm2 CONNECTION 'host=172.31.37.83 user=postgres dbname=postgres password=123' PUBLICATION test2_publication_vm2 WITH (copy_data = TRUE);
NOTICE:  created replication slot "test2_subscription_vm2" on publisher
CREATE SUBSCRIPTION

-- Check that data exists in table test2
postgres=# select * from test2;
 id  |    fio     
-----+------------
   1 | f5a90bd2c8
   2 | ec9e8d3c35
   3 | 9596093cca
   4 | 4bfc19a5ed
   5 | e337a2b8b0
   6 | cbf774a69a
   7 | 6998990f4f
   8 | 7114263d15
   9 | 2cde0937f6
  10 | 5d24b11cd6

-- Subscription status
postgres=# SELECT * FROM pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16417
subname               | test2_subscription_vm2
pid                   | 18838
relid                 | 
received_lsn          | 0/156C9A0
last_msg_send_time    | 2024-02-14 20:38:06.730956+00
last_msg_receipt_time | 2024-02-14 20:38:06.731177+00
latest_end_lsn        | 0/156C9A0
latest_end_time       | 2024-02-14 20:38:06.730956+00

```

### 3. Create tables on VM2 for writing (test2) and reading (test).
Created EC2 instance on AWS - pg-replication-vm2 (172.31.37.83)

```
-- Create tables, test - for reading, test2 - for writing
postgres=# CREATE TABLE test2 as select generate_series(1, 100) as id, md5(random()::text)::char(10)  as fio; 
SELECT 100
postgres=# CREATE TABLE test (id serial, fio varchar(10));
CREATE TABLE   
```

### 4. Create a publication for table test2 and subscribe to the publication of table test1 on VM1.
```
-- Set wal_level to logical 
postgres=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM

-- Restart to apply changes 
ubuntu@ip-172-31-37-83:~$ sudo systemctl restart postgresql@15-main

-- Verify that the changes are applied 
postgres=# show wal_level;
 wal_level 
-----------
 logical 

-- Create publication for table test2
postgres=# CREATE PUBLICATION test2_publication_vm2 FOR TABLE test2;
CREATE PUBLICATION

-- Subscribe to the publication of table test1 on VM1
postgres=# CREATE SUBSCRIPTION test_subscription_vm1 CONNECTION 'host=172.31.33.195 user=postgres dbname=postgres password=123' PUBLICATION test_publication_vm1 WITH (copy_data = TRUE);
NOTICE:  created replication slot "test_subscription_vm1" on publisher
CREATE SUBSCRIPTION

-- Check that data exists in table test
postgres=# select * from test;
 id  |    fio     
-----+------------
   1 | f5a90bd2c8
   2 | ec9e8d3c35
   3 | 9596093cca
   4 | 4bfc19a5ed
   5 | e337a2b8b0
   6 | cbf774a69a
   7 | 6998990f4f
   8 | 7114263d15
   9 | 2cde0937f6
  10 | 5d24b11cd6

-- Subscription status
postgres=# SELECT * FROM pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16399
subname               | test_subscription_vm1
pid                   | 18537
relid                 | 
received_lsn          | 0/1576D58
last_msg_send_time    | 2024-02-14 20:38:19.225892+00
last_msg_receipt_time | 2024-02-14 20:38:19.225997+00
latest_end_lsn        | 0/1576D58
latest_end_time       | 2024-02-14 20:38:19.225892+00
```
### Use VM3 as a read replica and for backups (subscribe to tables from VM1 and VM2).
Created EC2 instance on AWS - pg-replication-vm3 (172.31.41.188)
```
-- Create tables
postgres=# CREATE TABLE test (id serial, fio varchar(10));
CREATE TABLE
postgres=# CREATE TABLE test2 (id serial, fio varchar(10));  
CREATE TABLE

-- Subscribe to publications of table test2 on VM1, and test1 on VM2

postgres=# CREATE SUBSCRIPTION test_subscription_vm1 CONNECTION 'host=172.31.37.83 user=postgres dbname=postgres password=123' PUBLICATION test2_publication_vm2 WITH (copy_data = TRUE);
NOTICE:  created replication slot "test_subscription_vm1" on publisher
CREATE SUBSCRIPTION

postgres=# CREATE SUBSCRIPTION test_subscription_vm2 CONNECTION 'host=172.31.33.195 user=postgres dbname=postgres password=123' PUBLICATION test_publication_vm1 WITH (copy_data = TRUE);
NOTICE:  created replication slot "test_subscription_vm2" on publisher
CREATE SUBSCRIPTION

-- Verify that data is present
postgres=# select * from test;
 id  |    fio     
-----+------------
   1 | f5a90bd2c8
   2 | ec9e8d3c35
   3 | 9596093cca
   4 | 4bfc19a5ed
   5 | e337a2b8b0
   6 | cbf774a69a
   7 | 6998990f4f
   8 | 7114263d15
   9 | 2cde0937f6
  10 | 5d24b11cd6

postgres=# select * from test2;
 id  |    fio     
-----+------------
   1 | a2e3d71dba
   2 | 74af4b99c6
   3 | bf3aaed38b
   4 | a614791263
   5 | 8302cc25a0
   6 | f4fc8a309e
   7 | ca9d2f5e48
   8 | a85ce16ddc
   9 | 74fdf4f4dc
  10 | a5a5eb6c0f

-- статус подписки
postgres=# SELECT * FROM pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16398
subname               | test_subscription_vm1
pid                   | 18506
relid                 | 
received_lsn          | 0/156C9A0
last_msg_send_time    | 2024-02-14 20:39:06.799575+00
last_msg_receipt_time | 2024-02-14 20:39:06.799668+00
latest_end_lsn        | 0/156C9A0
latest_end_time       | 2024-02-14 20:39:06.799575+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16399
subname               | test_subscription_vm2
pid                   | 18508
relid                 | 
received_lsn          | 0/1576D58
last_msg_send_time    | 2024-02-14 20:39:19.293625+00
last_msg_receipt_time | 2024-02-14 20:39:19.293711+00
latest_end_lsn        | 0/1576D58
latest_end_time       | 2024-02-14 20:39:19.293625+00

```
### 6. Additional Preparatory Steps
To ensure communication between clusters, passwords were set for vm1 and vm2. The PostgreSQL server on vm1 and vm2 listens on all addresses (listen_addresses='*'). Separate IP addresses were configured for each machine in pg_hba.conf to enable connections within the subscriptions:
```
-- pg-replication-vm1
ubuntu@ip-172-31-33-195:~$ sudo nano /etc/postgresql/15/main/pg_hba.conf
# vm2
host    postgres        postgres        172.31.37.83/32         trust
# vm3
host    postgres        postgres        172.31.41.188/32        trust

-- pg-replication-vm2
ubuntu@ip-172-31-37-83:~$ sudo nano /etc/postgresql/15/main/pg_hba.conf
# vm3
host    postgres        postgres        172.31.41.188/32        trust
# vm1
host    postgres        postgres        172.31.33.195/32        trust
```