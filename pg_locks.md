## Lock Mechanism

1. Configure the server so that information about locks held for more than 200 milliseconds is written to the message log.  
```
otus=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
otus=# ALTER SYSTEM SET lock_timeout TO 200;
ALTER SYSTEM
otus=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
otus=# SHOW lock_timeout;
 lock_timeout 
------------------
 200ms
(1 row)
```

2. Reproduce a situation where such messages appear in the log.  

```
-- create a test table
otus=# CREATE TABLE accounts( acc_no integer PRIMARY KEY, amount numeric ); 
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
CREATE TABLE
INSERT 0 3
```

```
-- session 1
BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
SELECT pg_sleep(10); 
COMMIT;
```

```
-- session 2
BEGIN; 
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
COMMIT;
```

```
-- Log entry
ubuntu@ip-172-31-19-165:~$ sudo tail -n 20 /var/log/postgresql/postgresql-15-main.log
2024-01-30 20:03:24.891 UTC [17835] postgres@otus STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2024-01-30 20:04:36.811 UTC [17835] postgres@otus ERROR:  canceling statement due to lock timeout
2024-01-30 20:04:36.811 UTC [17835] postgres@otus CONTEXT:  while updating tuple (0,1) in relation "accounts"
```

3. Simulate the scenario of updating the same row with three UPDATE commands in different sessions.  

```
-- create the pg_locks view
otus=# CREATE VIEW locks_v AS SELECT pid, locktype, CASE locktype WHEN 'relation' THEN relation::regclass::text WHEN 'transactionid' THEN transactionid::text WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text END AS lockid, mode, granted FROM pg_locks WHERE locktype in ('relation','transactionid','tuple') AND (locktype != 'relation' OR relation = 'accounts'::regclass);
CREATE VIEW
```

```
-- session 1
otus=# BEGIN;
BEGIN
otus=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
          743 |          16633
(1 row)

otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1
-- transaction now holds a lock on the table, own number transaction 743
otus=*# SELECT * FROM locks_v WHERE pid = 16633;
  pid  |   locktype    |  lockid  |       mode       | granted 
-------+---------------+----------+------------------+---------
 16633 | relation      | accounts | RowExclusiveLock | t
 16633 | transactionid | 743      | ExclusiveLock    | t
(2 rows)
otus=*# COMMIT;

```

```
-- session 2
otus=# BEGIN;
BEGIN
otus=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
          744 |          16772
(1 row)
-- at this point, the transaction is waiting in the queue
otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
-- the transaction has locked the accounts table, its own transaction id is 744, it tries to acquire the lock of transaction 743 - false
otus=# SELECT * FROM locks_v WHERE pid = 16772;
  pid  |   locktype    |   lockid   |       mode       | granted 
-------+---------------+------------+------------------+---------
 16772 | relation      | accounts   | RowExclusiveLock | t
 16772 | transactionid | 743        | ShareLock        | f
 16772 | transactionid | 744        | ExclusiveLock    | t
 16772 | tuple         | accounts:7 | ExclusiveLock    | t
(4 rows)

-- after the commit of transaction 1, we wake up. Now we hold the relation and our own transaction lock 744

otus=*# SELECT * FROM locks_v WHERE pid = 16772;
  pid  |   locktype    |  lockid  |       mode       | granted 
-------+---------------+----------+------------------+---------
 16772 | relation      | accounts | RowExclusiveLock | t
 16772 | transactionid | 744      | ExclusiveLock    | t
(2 rows)

otus=*# COMMIT;

```

```
-- session 3
otus=# BEGIN;
BEGIN
otus=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
          745 |          17180
(1 row)
-- at this point, the transaction is also waiting in the queue
otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
-- the transaction has locked the accounts table, its own transaction id is 745, it tries to acquire the tuple lock - false
otus=*# SELECT * FROM locks_v WHERE pid = 17180;
  pid  |   locktype    |   lockid   |       mode       | granted 
-------+---------------+------------+------------------+---------
 17180 | relation      | accounts   | RowExclusiveLock | t
 17180 | transactionid | 745        | ExclusiveLock    | t
 17180 | tuple         | accounts:7 | ExclusiveLock    | f
(3 rows)

-- after the commit of transaction 1. Now we hold the relation and our own transaction lock 745 and try to acquire transaction id 744 - false

otus=*# SELECT * FROM locks_v WHERE pid = 17180;
  pid  |   locktype    |  lockid  |       mode       | granted 
-------+---------------+----------+------------------+---------
 17180 | relation      | accounts | RowExclusiveLock | t
 17180 | transactionid | 745      | ExclusiveLock    | t
 17180 | transactionid | 744      | ShareLock        | f
(3 rows)

-- after the commit of transaction 2 we wake up. Now we hold the relation and our own transaction lock 745.

otus=# SELECT * FROM locks_v WHERE pid = 17180;
  pid  |   locktype    |  lockid  |       mode       | granted 
-------+---------------+----------+------------------+---------
 17180 | relation      | accounts | RowExclusiveLock | t
 17180 | transactionid | 745      | ExclusiveLock    | t
(2 rows)

otus=*# COMMIT;
COMMIT

```
4. Reproduce a deadlock situation with three transactions. Can you understand the situation post-factum by studying the log messages?  
```
-- Transaction 1 (session 1) holds and locks row acc_no = 1. Updates row acc_no = 2.
-- Transaction 2 (session 2) holds and locks row acc_no = 2. Updates row acc_no = 3.
-- Transaction 3 (session 3) holds and locks row acc_no = 3. Updates row acc_no = 1.

otus=*# select * from accounts;                                                                                                                                                               
 acc_no | amount                                                                                                                                                                              
--------+---------                                                                                                                                                                            
      1 | 1000.00                                                                                                                                                                             
      2 | 2000.00                                                                                                                                                                             
      3 | 3000.00                                                                                                                                                                             
(3 rows) 
```

```
-- session 1.
otus=# BEGIN;
BEGIN
otus=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
          752 |          16633
(1 row)
otus=*# SELECT amount FROM accounts WHERE acc_no = 1 FOR UPDATE;
 amount  
---------
 1000.00
(1 row)

-- busy
otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

```

```
-- session 2. 
otus=# BEGIN;
BEGIN
otus=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
          753 |          16772
(1 row)
otus=*# SELECT amount FROM accounts WHERE acc_no = 2 FOR UPDATE;
 amount  
---------
 2000.00
(1 row)

-- busy
otus=*# otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;

```

```
-- session 3 
otus=# BEGIN;
BEGIN
otus=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid 
--------------+----------------
          754 |          17180
(1 row)
otus=*# SELECT amount FROM accounts WHERE acc_no = 3 FOR UPDATE;
 amount  
---------
 3000.00
(1 row)

otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
ERROR:  deadlock detected
DETAIL:  Process 17180 waits for ShareLock on transaction 752; blocked by process 16633.
Process 16633 waits for ShareLock on transaction 753; blocked by process 16772.
Process 16772 waits for ShareLock on transaction 754; blocked by process 17180.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"

```

```
In logs - deadlock detected
Detalization:
1. Process 17180 wants a ShareLock on transaction 752, which is held by process 16633.
2. Process 16633 wants a ShareLock on transaction 753, which is held by process 16772.
3. Process 16772 wants a ShareLock on transaction 754, which is held by process 17180.
```

```
2024-01-22 22:16:12.353 UTC [16772] postgres@otus STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
2024-01-22 22:16:19.921 UTC [17180] postgres@otus LOG:  process 17180 detected deadlock while waiting for ShareLock on transaction 752 after 200.180 ms
2024-01-22 22:16:19.921 UTC [17180] postgres@otus DETAIL:  Process holding the lock: 16633. Wait queue: .
2024-01-22 22:16:19.921 UTC [17180] postgres@otus CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-01-22 22:16:19.921 UTC [17180] postgres@otus STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2024-01-22 22:16:19.921 UTC [17180] postgres@otus ERROR:  deadlock detected
2024-01-22 22:16:19.921 UTC [17180] postgres@otus DETAIL:  Process 17180 waits for ShareLock on transaction 752; blocked by process 16633.
        Process 16633 waits for ShareLock on transaction 753; blocked by process 16772.
        Process 16772 waits for ShareLock on transaction 754; blocked by process 17180.
        Process 17180: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
        Process 16633: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
        Process 16772: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
2024-01-22 22:16:19.921 UTC [17180] postgres@otus HINT:  See server log for query details.
2024-01-22 22:16:19.921 UTC [17180] postgres@otus CONTEXT:  while updating tuple (0,1) in relation "accounts"
2024-01-22 22:16:19.921 UTC [17180] postgres@otus STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2024-01-22 22:16:19.921 UTC [16772] postgres@otus LOG:  process 16772 acquired ShareLock on transaction 754 after 7768.154 ms
2024-01-22 22:16:19.921 UTC [16772] postgres@otus CONTEXT:  while updating tuple (0,3) in relation "accounts"
2024-01-22 22:16:19.921 UTC [16772] postgres@otus STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
```

5. Can two transactions executing a single UPDATE command on the same table (without where) block each other?  
```
Two transactions can deadlock each other:  
- Transaction #1 updates rows in one order (ascending).  
- Transaction #2 updates rows in a different order (descending).  
```