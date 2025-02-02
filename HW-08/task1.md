### Нагрузочное тестирование и тюнинг PostgreSQL
Команда для теста
```
pgbench -c 50 -j 2 -P 10 -T 300 -U postgres postgres
```
Конфигурационный файл, который предложил https://pgconfigurator.cybertec.at/
```
# Connectivity
max_connections = 60
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 1 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 4 # speed of random disk access relative to sequential access (1.0)

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
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_maintenance_workers = 4
max_parallel_workers = 8
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 1
wal_recycle = off
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
number of transactions actually processed: 1133736
number of failed transactions: 0 (0.000%)
latency average = 26.458 ms
latency stddev = 36.175 ms
initial connection time = 63.400 ms
tps = 1889.665834 (without initial connection time)
```
После тюнинга
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 1896283
number of failed transactions: 0 (0.000%)
latency average = 7.908 ms
latency stddev = 10.697 ms
initial connection time = 63.404 ms
tps = 6321.807651 (without initial connection time)
```
После было произведенно несколько тестов для выяснения, какие настройки повлияли на производительность больше всего.
Во время теста применялись новые параметры из определенной группы. Все остальные параметры оставались прежними.
### 1) Memory Settings
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'huge_pages', 'effective_cache_size', 'effective_io_concurrency', 'random_page_cost');
```
При применении следующих настроек производительность практически не изменилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|shared_buffers | 1024 MB| 128 MB | Буфер для разделяемой памяти рекомендуется брать в 25% от ОЗУ |
|work_mem | 32 MB | 4 MB | Память, которая будет используется при обработке запросов. Выделяется для каждого процесса. |
|maintenance_work_mem |320 MB| 64 MB | Максимальный объём памяти для операций VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY |
|huge_pages | off | try | Запрашивание огромных таблиц из общей памяти отключено|
|effective_cache_size | 11 GB | 4 GB | Оценка памяти, доступной для кэширования диска, рекомендуется брать в 50-75% от ОЗУ |
|effective_io_concurrency | 1 | 1 | Число одновременных операций ввода-вывода. Параметр зависит от типа диска. При SSD стоит выбирать значения в несколько сотен|
|random_page_cost | 4 | 4 | Стоимость чтения одной произвольной страницы с диска. Параметр зависит от типа диска. При SSD стоит выбирать значения ближе к 1|
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 558208
number of failed transactions: 0 (0.000%)
latency average = 26.867 ms
latency stddev = 36.133 ms
initial connection time = 62.015 ms
tps = 1860.886103 (without initial connection time)
```

### 2) Monitoring
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('shared_preload_libraries', 'track_io_timing', 'track_functions');
```

При применении следующих настроек производительность практически не изменилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|shared_preload_libraries |'pg_stat_statements' | - | В этом параметре задаются библиотеки, которые будут загружаться при запуске сервера. |
|track_io_timing | on | of | Включает мониторинг времени чтения и записи блоков. |
|track_functions | pl | none | Включает подсчёт вызовов функций и времени их выполнения. Значение pl включает отслеживание только функций на процедурном языке, а all — также функций на языках SQL и C. |
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 554214
number of failed transactions: 0 (0.000%)
latency average = 27.060 ms
latency stddev = 37.255 ms
initial connection time = 58.711 ms
tps = 1847.563321 (without initial connection time)
```

### 3) Replication
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('wal_level', 'max_wal_senders', 'synchronous_commit');
```
Изменение этих метрик привело к увеличению производительности.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|wal_level |replica | replica | Параметр определяет, как много информации записывается в WAL. Возможные значения replica, minimal, logical. Со значением replica в журнал записываются данные, необходимые для поддержки архивирования WAL и репликации, включая запросы только на чтение на ведомом сервере. |
|max_wal_senders | 0 | 10 | Задаёт максимально допустимое число одновременных подключений ведомых серверов или клиентов потокового копирования. |
|synchronous_commit | off | on | Параметр, который определяет, когда транзакции считаются зафиксированными и в какой момент клиент получает подтверждение об этом. Транзакции считаются зафиксированными только после того, как записи WAL будут записаны на диск.  Транзакции считаются зафиксированными сразу после записи в журнал WAL, без ожидания записи на диск. Другие возможные значения: remote_write, local, remote_apply |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 1927988
number of failed transactions: 0 (0.000%)
latency average = 7.778 ms
latency stddev = 10.661 ms
initial connection time = 61.670 ms
tps = 6427.607412 (without initial connection time)
```

**Наибольший вклад внесла метрика synchronous_commit.**
Тест, где все настройки, кроме synchronous_commit, старые.
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 1955794
number of failed transactions: 0 (0.000%)
latency average = 7.667 ms
latency stddev = 10.567 ms
initial connection time = 62.401 ms
tps = 6520.263292 (without initial connection time)
```

### 4) Checkpointing
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('checkpoint_timeout', 'checkpoint_completion_target', 'max_wal_size', 'min_wal_size');
```
При применении следующих настроек производительность практически не изменилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|checkpoint_timeout | 15 min | 5 min | Параметр, который устанавливает максимальное время между автоматическими контрольными точками в WAL |
|checkpoint_completion_target | 0.9 | 0.9 |
|max_wal_size | 1024 MB | 1024 MB |
|min_wal_size | 512 MB | 80 MB |

```
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 567613
number of failed transactions: 0 (0.000%)
latency average = 26.421 ms
latency stddev = 35.287 ms
initial connection time = 59.801 ms
tps = 1892.268002 (without initial connection time)
```

### 5) WAL writing
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('wal_compression', 'wal_buffers');
```
При применении следующих настроек производительность немного повысилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|wal_compression | on | off |
|wal_buffers | -1 | 4 MB |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 701818
number of failed transactions: 0 (0.000%)
latency average = 21.370 ms
latency stddev = 24.492 ms
initial connection time = 42.333 ms
tps = 2339.531194 (without initial connection time)
```

### 6) Background writer
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('bgwriter_delay', 'bgwriter_lru_maxpages', 'bgwriter_lru_multiplier', 'bgwriter_flush_after');
```

При применении следующих настроек производительность немного повысилась.

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|bgwriter_delay | 200ms | 200ms
|bgwriter_lru_maxpages | 100 | 100 |
|bgwriter_lru_multiplier | 2.0 | 2.0 |
|bgwriter_flush_after | 0 | 512 kb |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 700070
number of failed transactions: 0 (0.000%)
latency average = 21.424 ms
latency stddev = 24.676 ms
initial connection time = 44.246 ms
tps = 2333.728822 (without initial connection time)
```

### 7) Parallel queries
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('max_worker_processes', 'max_parallel_workers_per_gather', 'max_parallel_maintenance_workers', 'max_parallel_workers', 'parallel_leader_participation');
```

|Настройки|Новое значение|Старое значение |Комментарии|
|---------|--------------|----------------|-----------|
|max_worker_processes | 1 | 8 |
|max_parallel_workers_per_gather | 1 | 4 |
|max_parallel_maintenance_workers | 1 | 4 |
|max_parallel_workers | 1 | 8 |
|parallel_leader_participation | on | on |

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 700750
number of failed transactions: 0 (0.000%)
latency average = 21.403 ms
latency stddev = 24.287 ms
initial connection time = 44.512 ms
tps = 2335.972923 (without initial connection time)
```
**Наибольшее значение на производительность оказал параметр synchronous_commit.**
