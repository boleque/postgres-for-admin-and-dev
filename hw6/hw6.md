# Домашнее задание
## Работа с журналами

1. Настройте выполнение контрольной точки раз в 30 секунд.
```
postgres=# alter system set checkpoint_timeout = '30s';
ALTER SYSTEM
```
2. 10 минут c помощью утилиты pgbench подавайте нагрузку
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
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. 
```
-- https://gist.github.com/lesovsky/4587d70f169739c01d4525027c087d14

     Uptime      | Since stats reset | Forced checkpoint ratio (%) | Minutes between checkpoints | Average write time per checkpoint (s) | Average sync time per checkpoint (s) | Total MB written | MB per checkpoint 
-----------------+-------------------+-----------------------------+-----------------------------+---------------------------------------+--------------------------------------+------------------+-------------------
 00:10:56.226604 | 00:11:45.781325   |                         4.5 |                        0.55 |                                 24.48 |                                 0.01 |            377.0 |             15.45
(1 row)

```
4. Оцените, какой объем приходится в среднем на одну контрольную точку.
```
15.45 MB per checkpoint
```
5. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```
Заданный интервал между точками составил 30 секунд.
Расчетный составил 33 секунды, а время записи составляет  checkpoint_completion_target x 30 = 0.9 * 30 = 27 секунд.
Запись контрольной точки завершается за 6 секунд до начала следующей точки. Несмотря на то, что в момент времени выполняется одна контрольная точка,
запас в 6 секунд является ненадежным решением. 
Стоит отметить, что дальнейшее увеличение нагрузки, особенно при уменьшении параметра max_wal_size, может приводить к незапланированному вызову 
контрольной точки сервером, т.к. будет превышен допустимый объем журнальных записей. Внеплановый вызов точки при запасе всего 6 секунд 
приведет к тому, что точки на шкале времени будут расположены "внахлест" .
```

6. Сравните tps в синхронном/асинхронном режиме утилитой pgbench
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
7. Объясните полученный результат.
```
При асинхронной фиксации время отклика (latency) уменьшилось (0.616 ms против 1.726 ms)
Пропускная способность (tps) увеличилась (1624.5 против 579.2)

При синхронной записи команда COMMIT не возвращает управление до конца синхронизации, тем самым увеличивая время отклика 
и снижая пропускную tps системы. При асинхронной записи фиксация изменений не ждет физической записи на диск. 
```
8. Создайте новый кластер с включенной контрольной суммой страниц.
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
9. Создайте таблицу. Вставьте несколько значений. 
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
10. Выключите кластер.
```
ubuntu@ip-172-31-34-233:~$ sudo systemctl stop postgresql@15-main
```
11. Измените пару байт в таблице. 
```
ubuntu@ip-172-31-34-233:~$ sudo sed -i 's/@/@12345@/' /var/lib/postgresql/15/main/base/16388/16390
```
12. Включите кластер и сделайте выборку из таблицы. 
```
ubuntu@ip-172-31-34-233:~$ sudo systemctl start postgresql@15-main
```
```
otus=# select * from persons;
WARNING:  page verification failed, calculated checksum 37049 but expected 3345
ERROR:  invalid page in block 0 of relation base/16388/16390
```
13. Что и почему произошло?
```
Механизм контрольный сумм включен, таким образом это защищает основной слой файлов данных. 
В нашем случае несовпадение значений [контрольных сумм] привело к указанной выше ошибке.
```
14. как проигнорировать ошибку и продолжить работу?
```
Предлагаю следующие варианты решения:
1. Можно попробовать восстановить данные из резервной копии (если имеется);
2. Попробовать прочитать поврежденную страницу (с риском получить искаженную информацию) используя ignore_checksum_failure off.
```