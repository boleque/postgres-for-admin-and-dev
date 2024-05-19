## Load Testing and Tuning PostgreSQL

### 1. Deploy a virtual machine using any convenient method

Deployed ec2 instance aws:

Ububntu Server 22.04 LTS, SSD Volume

t3.medium 2 vCPU 4GiB Memory

SSD Storage 10 GB

### 2. Install PostgreSQL 15 on it using any method

```bash

sudo  apt  update && sudo  apt  upgrade  -y  -q && sudo  sh  -c  'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget  --quiet  -O  -  https://www.postgresql.org/media/keys/ACCC4CF8.asc  |  sudo  apt-key  add  - && sudo  apt-get  update && sudo  apt  -y  install  postgresql-15

```

### 3. Configure the PostgreSQL 15 cluster for maximum performance without paying attention to possible reliability issues in case of VM crash

According to the result of testing (value of tps):

1st place (1885.424916) - fsync off
2nd place (1801.204654) - synchronous_commit = off
3rd place (699.357785) - Web Application configuration (https://pgtune.leopard.in.ua/)
4th place (693.741527) - Online Transaction Processing configuration (https://pgtune.leopard.in.ua/)
5th place (688.674347) - default cluster settings

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
**Comments:**
- significant improvement ~2.7 times compared to default settings
- no overhead on system calls because transaction log entries are not forcibly written to disk
- sacrifices reliability

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
**Comments:**
- significant improvement ~2.7 times compared to default settings
- no overhead as there is no need to wait for WAL writes to disk before returning successful transaction completion to the connected client
- data may be lost (last set of transactions)

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

**Comments:**
- improvement by 11 tps
- in web applications, there are usually many simple queries, so the work_mem value is slightly increased to 8738kB
- shared_buffers increased from 128MB to 1GB, the value is in line with general recommendations to keep it within 1/4 of RAM size (4GB in this test). Insufficient shared_buffers size would result in working data being read and written from OS cache or disk.
- min_wal_size=1GB, max_wal_size=4GB. Presumably, the logic behind choosing these parameters was that in web applications, there are no complex and long-running queries, so it's reasonable to keep the segment size within a range that won't lead to frequent checkpoints.

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
**Comments:**
- improvement by 5 tps
- min_wal_size=2GB, max_wal_size=8GB doubled compared to the web application configuration. A high min_wal_size value allows reducing I/O write operations, as the 2GB size WAL files will be efficiently reused.

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

**Comments:**
- overall, the settings provide satisfactory cluster operation, which can be attributed to the small difference in performance compared to the settings calculated by the pgtune utility.
- shared_buffers = 128MB is low; I think this is the first parameter that needs to be changed (i.e., increased) when installing a PostgreSQL cluster. A real-world, heavily loaded application will exhaust 128 MB, leading to excessive page evictions and increased IO operations.
- min_wal_size = 80MB also has a very small value, as mentioned earlier. PostgreSQL will not be able to efficiently reuse WAL files at a certain level of real-world load, leading to increased IO operations.