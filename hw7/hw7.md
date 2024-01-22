# Домашнее задание
## Механизм блокировок

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. 
```
otus=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
otus=# ALTER SYSTEM SET deadlock_timeout TO 200;
ALTER SYSTEM
otus=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
otus=# SHOW deadlock_timeout;
 deadlock_timeout 
------------------
 200ms
(1 row)
```

2. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

```
-- создадим тестовую таблицу
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
-- Запись в журнале 
2024-01-22 20:53:37.289 UTC [16772] postgres@otus LOG:  process 16772 still waiting for ShareLock on transaction 740 after 200.151 ms
2024-01-22 20:53:37.289 UTC [16772] postgres@otus DETAIL:  Process holding the lock: 16633. Wait queue: 16772.
2024-01-22 20:53:37.289 UTC [16772] postgres@otus CONTEXT:  while updating tuple (0,5) in relation "accounts"
2024-01-22 20:53:37.289 UTC [16772] postgres@otus STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2024-01-22 20:53:59.948 UTC [16772] postgres@otus LOG:  process 16772 acquired ShareLock on transaction 740 after 22858.691 ms
```

2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 

```
-- создадим представление pg_locks
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
-- транзакция теперь удерживает блокировку таблицы, собственный номер транзакции 743
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
-- на этом шаге транзакция зависает в очереди ожидания
otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
-- транзакция заблокировала таблицу accounts, собственный номер транзакции 744, блокировку типа tuple (версии строки), пытается захватить блокировку транзакции 743 - false
otus=# SELECT * FROM locks_v WHERE pid = 16772;
  pid  |   locktype    |   lockid   |       mode       | granted 
-------+---------------+------------+------------------+---------
 16772 | relation      | accounts   | RowExclusiveLock | t
 16772 | transactionid | 743        | ShareLock        | f
 16772 | transactionid | 744        | ExclusiveLock    | t
 16772 | tuple         | accounts:7 | ExclusiveLock    | t
(4 rows)

-- после коммита транзакции 1, просыпаемся. Теперь держим блокировку отношения и собственного номера транзакции 744

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
-- на этом шаге транзакция также зависает в очереди ожидания
otus=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
-- транзакция заблокировала таблицу accounts, собственный номер транзакции 745, пытается захватить блокировку типа tuple (версии строки) - false
otus=*# SELECT * FROM locks_v WHERE pid = 17180;
  pid  |   locktype    |   lockid   |       mode       | granted 
-------+---------------+------------+------------------+---------
 17180 | relation      | accounts   | RowExclusiveLock | t
 17180 | transactionid | 745        | ExclusiveLock    | t
 17180 | tuple         | accounts:7 | ExclusiveLock    | f
(3 rows)

-- после коммита транзакции 1. Теперь держим блокировку отношения и собственного номера транзакции 745 и пытаемся захватить id транзакции 744 - false

otus=*# SELECT * FROM locks_v WHERE pid = 17180;
  pid  |   locktype    |  lockid  |       mode       | granted 
-------+---------------+----------+------------------+---------
 17180 | relation      | accounts | RowExclusiveLock | t
 17180 | transactionid | 745      | ExclusiveLock    | t
 17180 | transactionid | 744      | ShareLock        | f
(3 rows)

-- после коммита транзакции 3 просыпаемся. Теперь держим блокировку отношения и собственного номера транзакции 745.

otus=# SELECT * FROM locks_v WHERE pid = 17180;
  pid  |   locktype    |  lockid  |       mode       | granted 
-------+---------------+----------+------------------+---------
 17180 | relation      | accounts | RowExclusiveLock | t
 17180 | transactionid | 745      | ExclusiveLock    | t
(2 rows)

otus=*# COMMIT;
COMMIT

```
4. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
```
-- Транзакция 1 (сессия 1)  держит блокирует строку acc_no = 1. Обновляет строку acc_no = 2
-- Транзакция 2 (сессия 2)  держит блокирует строку acc_no = 2. Обновляет строку acc_no = 3
-- Транзакция 3 (сессия 3)  держит блокирует строку acc_no = 3. Обновляет строку acc_no = 1

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

-- висим
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

-- висим
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

5. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
```
Две транзации могут взаимоблокироваться, например если Транзакция #1 обновляет строки в одном порядке (по возрастанияю), Транзакция #2 обновляет строки в другом порядке (по убыванию)
```