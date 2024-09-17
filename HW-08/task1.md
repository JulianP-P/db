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
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 2
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
number of transactions actually processed: 691063
number of failed transactions: 0 (0.000%)
latency average = 43.408 ms
latency stddev = 57.503 ms
initial connection time = 51.094 ms
tps = 1151.708066 (without initial connection time)
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
После было произведенно несколько тестов для выяснения, какие настройки повлияли на производительность больше всего.
Во время теста применялись новые параметры из определенной группы. Все остальные параметры оставались прежними.
1) Memory Settings

При применении следующих настроек производительность практически не изменилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|shared_buffers | 1024 MB| 128 MB | Буфер для разделяемой памяти рекомендуется брать в 25% от ОЗУ |
|work_mem | 32 MB | 4 MB |  |
|maintenance_work_mem |320 MB| 64MB | |
|huge_pages | off | try | Запрашивание огромных таблиц из общей памяти отключено|
|effective_cache_size | 3 GB | 4 GB | Оценка памяти, доступной для кэширования диска, рекомендуется брать в 50-75% от ОЗУ |
|effective_io_concurrency | 100 | 1 | Число одновременных операций ввода-вывода. Параметр зависит от типа диска. При SSD стоит выбирать значения в несколько сотен|
|random_page_cost | 1.25 | 4 | Стоимость чтения одной произвольной страницы с диска. Параметр зависит от типа диска. При SSD стоит выбирать значения ближе к 1|
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 720030
number of failed transactions: 0 (0.000%)
latency average = 41.663 ms
latency stddev = 51.962 ms
initial connection time = 55.644 ms
tps = 1199.907411 (without initial connection time)
```

2) Monitoring

При применении следующих настроек производительность практически не изменилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|shared_preload_libraries |'pg_stat_statements' | - |
|track_io_timing | on | of |
|track_functions | pl | none |
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 716502
number of failed transactions: 0 (0.000%)
latency average = 41.867 ms
latency stddev = 52.727 ms
initial connection time = 56.529 ms
tps = 1194.113487 (without initial connection time)
```

3) Replication

Изменение этих метрик привело к увеличению производительности.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|wal_level |replica |
|max_wal_senders | 0 |
|synchronous_commit | off |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1619457
number of failed transactions: 0 (0.000%)
latency average = 18.520 ms
latency stddev = 23.640 ms
initial connection time = 67.204 ms
tps = 2699.069206 (without initial connection time)
```

Наибольший вклад внесла метрика synchronous_commit.
Тест, где все настройки, кроме synchronous_commit, старые.
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 1638944
number of failed transactions: 0 (0.000%)
latency average = 18.301 ms
latency stddev = 23.693 ms
initial connection time = 55.789 ms
tps = 2731.484009 (without initial connection time)
```

4) Checkpointing

Применение метрик снизило производительность

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|checkpoint_timeout | 15 min |
|checkpoint_completion_target | 0.9 |
|max_wal_size | 1024 MB |
|min_wal_size | 512 MB |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 640588
number of failed transactions: 0 (0.000%)
latency average = 46.829 ms
latency stddev = 69.050 ms
initial connection time = 50.015 ms
tps = 1067.583313 (without initial connection time)
```

5) WAL writing

Применение метрик снизило производительность

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|wal_compression | on |
|wal_buffers | -1 |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 674938
number of failed transactions: 0 (0.000%)
latency average = 44.445 ms
latency stddev = 65.378 ms
initial connection time = 54.759 ms
tps = 1124.832512 (without initial connection time)
```

6) Background writer

Применение метрик снизило производительность

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|bgwriter_delay | 200ms |
|bgwriter_lru_maxpages | 100 |
|bgwriter_lru_multiplier | 2.0 |
|bgwriter_flush_after | 0 |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 693028
number of failed transactions: 0 (0.000%)
latency average = 43.285 ms
latency stddev = 97.706 ms
initial connection time = 57.642 ms
tps = 1154.980762 (without initial connection time)
```

7) Parallel queries

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|max_worker_processes | 1 |
|max_parallel_workers_per_gather | 1 |
|max_parallel_maintenance_workers | 1 |
|max_parallel_workers | 1 |
|parallel_leader_participation | on |


```
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 711027
number of failed transactions: 0 (0.000%)
latency average = 42.191 ms
latency stddev = 53.506 ms
initial connection time = 54.557 ms
tps = 1184.850339 (without initial connection time)
```
