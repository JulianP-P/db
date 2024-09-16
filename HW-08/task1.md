### Нагрузочное тестирование и тюнинг PostgreSQL
Команда для теста
```
pgbench -c 50 -j 2 -P 10 -T 600 -U postgres postgres
```
Конфигурационный файл, который предложил https://pgconfigurator.cybertec.at/
```
# Connectivity
max_connections = 60

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = off

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries:
max_worker_processes = 1
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 1
parallel_leader_participation = on
```
Производительность до тюнинга
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 590428
number of failed transactions: 0 (0.000%)
latency average = 50.808 ms
latency stddev = 65.690 ms
initial connection time = 59.221 ms
tps = 983.983951 (without initial connection time)
```
После тюнинга
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1566987
number of failed transactions: 0 (0.000%)
latency average = 19.143 ms
latency stddev = 24.210 ms
initial connection time = 54.629 ms
tps = 2611.540042 (without initial connection time)
```
После было произведенно несколько тестов для выяснения, какие настройки повлияли на производительность больше всего

После тюнинга synchronous_commit = on
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 686589
number of failed transactions: 0 (0.000%)
latency average = 43.691 ms
latency stddev = 56.320 ms
initial connection time = 65.759 ms
tps = 1144.255503 (without initial connection time)
```
synchronous_commit off
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1512182
number of failed transactions: 0 (0.000%)
latency average = 19.836 ms
latency stddev = 24.993 ms
initial connection time = 57.819 ms
tps = 2520.230310 (without initial connection time)
```


