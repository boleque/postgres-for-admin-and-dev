# Домашнее задание

## Нагрузочное тестирование и тюнинг PostgreSQL

  

### 1. Развернуть виртуальную машину любым удобным способом

Развернул ec2 инстанс aws:

Ububntu Server 22.04 LTS, SSD Volume

t3.medium 2 vCPU 4GiB Memory

SSD Storage 10 GB

### 2. Поставить на неё PostgreSQL 15 любым способом

```bash

sudo  apt  update && sudo  apt  upgrade  -y  -q && sudo  sh  -c  'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget  --quiet  -O  -  https://www.postgresql.org/media/keys/ACCC4CF8.asc  |  sudo  apt-key  add  - && sudo  apt-get  update && sudo  apt  -y  install  postgresql-15

```

### 3. Настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины

По результам нагрузочного тестирование, полечены следующие значения tps:

1 место (1885.424916) -  fsync off
2 место (1801.204654) - synchronous_commit = off
3 место (699.357785) - конфигурация Web Application (https://pgtune.leopard.in.ua/)
4 место (693.741527) - конфигурация Online Transaction Processing (https://pgtune.leopard.in.ua/)
5 место (688.674347) - дефолтные параметры кластера 

Далее подробнее: 
#### 3.1.  fsync off
```postgres@ip-172-31-35-20:/home/ubuntu$ pgbench -c 50 -j 2 -P 60 -T 60 postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 1885.3 tps, lat 26.444 ms stddev 22.627, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 113168
number of failed transactions: 0 (0.000%)
latency average = 26.478 ms
latency stddev = 22.707 ms
initial connection time = 135.792 ms
tps = 1885.424916 (without initial connection time)
```
**Комментарии:**
- существенный прирост ~ 2.7 раза в сравнении с дефолтными настройками
- отсутствуют накладные расходы на системные вызовы, т.к. записи в журнале транзакций не сбрасываются принудительно на диск
- жертвуем надежностью

#### 3.2.  synchronous_commit = off
```
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 1801.2 tps, lat 27.691 ms stddev 23.215, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 108121
number of failed transactions: 0 (0.000%)
latency average = 27.723 ms
latency stddev = 23.278 ms
initial connection time = 109.209 ms
tps = 1801.204654 (without initial connection time)
```
**Комментарии:**
- существенный прирост ~ 2.7 раза в сравнении с дефолтными настройками
- отсутствуют накладные расходы, т.к. нет необходимости ждать WAL записи на диск перед возвратом успешного завершения транзакции для подключенного клиента
- данные могут быть потеряны (последний набор транзакций)
- 
#### 3.3.  Web Application (https://pgtune.leopard.in.ua/)
<details>
<summary>configuration</summary>
<pre>
max_connections = 60
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 8738kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
</pre>
</details>

```
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 698.8 tps, lat 71.341 ms stddev 59.980, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41979
number of failed transactions: 0 (0.000%)
latency average = 71.411 ms
latency stddev = 60.030 ms
initial connection time = 106.875 ms
tps = 699.357785 (without initial connection time)
```

**Комментарии:**
- прирост на 11 tls 
- в веб приложениях как правило запросов много и они простые, поэтому значение work_mem несущественно увеличено до 8738kB
- shared_buffers увеличен с 128 MB до 1 GB, значение соответствует общей рекомендации в различных источниках держать в пределах 1/4 от размера RAM (в тесте это 4GB). Недостаточный размер shared_buffers приведет к тому, что рабочие данные будут писаться и читаться из кеша OS или диска.
- min_wal_size=1GB, max_wal_size=4GB. Предполагаю, что логика выбора параметров была такая - в веб приложениях нет сложных запросов, долгих транзакций, поэтому разумно держать размер сегментов в диапазоне, которых не будет приводить к частым контрольным точкам.

#### 3.4.  Online Transaction Processing (https://pgtune.leopard.in.ua/)
<details>
<summary>configuration</summary>
<pre>
max_connections = 60
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 8738kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
</pre>
</details>

```
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 693.4 tps, lat 71.902 ms stddev 60.331, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41656
number of failed transactions: 0 (0.000%)
latency average = 71.984 ms
latency stddev = 60.383 ms
initial connection time = 101.791 ms
tps = 693.741527 (without initial connection time)
```
**Комментарии:**
- прирост на 5 tls 
- min_wal_size=2GB, max_wal_size=8GB увеличены в х2 раза в сравнении с конфинурацией для веб приложений . Высокое значение min_wal_size позволяет уменьшить I/O операции записи, т.к. журнальные файлы, укладывающие в размер 2GB будут эффективно переиспользоваться. 

#### 3.5.  default
<details>
<summary>configuration</summary>
<pre>
max_connections = 100
shared_buffers = 128MB
effective_cache_size = 4GB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.9
wal_buffers = -1
default_statistics_target = 100
random_page_cost = 4.0
effective_io_concurrency = 1
work_mem = 4MB
huge_pages = try
min_wal_size = 80MB
max_wal_size = 1GB
</pre>
</details>

```
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 688.4 tps, lat 72.424 ms stddev 60.526, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41353
number of failed transactions: 0 (0.000%)
latency average = 72.508 ms
latency stddev = 60.584 ms
initial connection time = 109.338 ms
tps = 688.674347 (without initial connection time)
```

**Комментарии:**
- в целом параметры обеспечивают работу кластера на удовлетворительном уровне, что объясняется небольшой разницей в производительности с настройками, вычисленными утилитой pgtune. 
- бросается в глаза shared_buffers = 128MB занижено, думаю это первый параметр. который нужно изменить (т.е. увеличить) при установке кластера Postgresql. Реальное нагруженное приложение исчерпает 128 MB, что приведет к избыточному вытеснению страниц, расходам на IO операции.
- min_wal_size = 80MB так же имеет крайне небольшое значение, как уже было упомянуто выше, постгрес не сможет (при определенном уровне реальной нагрузки) эффективно переиспользовать журнальные файлы, что приведет к увеличение IO операций.