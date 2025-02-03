# Нагрузочное тестирование и тюнинг PostgreSQL
Для получения оптимальных параметров был проведен ряд тестов с разными конфигурациями. Источники конфигураций:
-  [Pgconfigurator](https://pgconfigurator.cybertec.at/)
-  [PgTune](https://pgtune.fariton.ru/)
- собственное предложение на основе рекомендаций из интернета
  
Команда для теста. Для получения наиболее корректных значений каждый тест запускался три раза. Результат - среднее значение.
```
pgbench -c 50 -j 2 -P 10 -T 300 -U postgres postgres
```
### Параметры для генерации файла с конфигами
```
# DB Version: 15
# OS Type: linux
# DB Type: oltp
# Total Memory (RAM): 16 GB
# CPUs num: 8
# Connections num: 60
# Data Storage: ssd
Number of disks: 1
How big is your database?: 1 GB
How many replicas do you need?: 0
Do you want to activate wal recycling?: No
Can you lose single transactions in case of a crash?: Yes
Are you willing to try out experimental features for better performance?: No
```


### Pgconfigurator
Конфигурационный файл, который предложил [Pgconfigurator](https://pgconfigurator.cybertec.at/)
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
```
### PgTune
Конфигурация, которую предложил [PgTune](https://pgtune.fariton.ru/)
```
# DB Version: 15
# OS Type: linux
# DB Type: oltp
# Total Memory (RAM): 16 GB
# CPUs num: 8
# Connections num: 60
# Data Storage: ssd

max_connections = 60
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 17476kB
huge_pages = off
min_wal_size = 2GB
max_wal_size = 8GB
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_parallel_maintenance_workers = 4
```
### Собственный конфиг
```
# Memory Settings
shared_buffers = '4 GB'
work_mem = '70 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '11 GB'
effective_io_concurrency = 200 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1 # speed of random disk access relative to sequential access (1.0)

```
## Сравнение
Производительность при разных конфигурациях:
|                  |По умолчанию|Pgconfigurator|PgTune  |Мое предложение|
|:-----------------|:-----------|:-------------|:-------|---------------|
|Производительность|1875        |6225       |4 GB    |4 GB           | 

После было произведенно несколько тестов для выяснения, какие настройки повлияли на производительность больше всего.
Во время теста применялись новые параметры из определенной группы. Все остальные параметры оставались по умолчанию.
### 1) Memory Settings
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'huge_pages', 'effective_cache_size', 'effective_io_concurrency', 'random_page_cost');
```
При применении следующих настроек производительность практически не изменилась.

|Настройки                |По умолчанию|Pgconfigurator|PgTune  |Мое предложение|Комментарии|
|:------------------------|:-----------|:-------------|:-------|:--------------|-----------|
|shared_buffers           |128 MB      |1024 MB       |4 GB    |4 GB           | Буфер для разделяемой памяти рекомендуется брать в 25% от ОЗУ |
|work_mem                 |4 MB        |32 MB         |17476 kB|70 MB          | Память, которая будет используется при обработке запросов. Выделяется для каждого процесса. Значение посчитано по формуле Total RAM * 0,25 / max_connections. |
|maintenance_work_mem     |64 MB       |320 MB        |1 GB    |1 GB           | Максимальный объём памяти для операций VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY |
|huge_pages               |try         |off           |off     |off            | Запрашивание огромных таблиц из общей памяти отключено. В докере для использования этой функции необходимо провести доп исследования для использования данного функционала.|
|effective_cache_size     |4 GB        |11 GB         |12 GB   |12 GB          | Оценка памяти, доступной для кэширования диска, рекомендуется брать в 50-75% от ОЗУ |
|effective_io_concurrency |1           |100           |200     |200            | Число одновременных операций ввода-вывода. Параметр зависит от типа диска. При SSD стоит выбирать значения в несколько сотен|
|random_page_cost         |4           |1.25          |1.1     |1              | Стоимость чтения одной произвольной страницы с диска. Параметр зависит от типа диска. При SSD стоит выбирать значения ближе к 1|
|**tps**                  |**1875**    |-             |-       |**1869**       | - |

### 2) Monitoring
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('shared_preload_libraries', 'track_io_timing', 'track_functions');
```

При применении следующих настроек производительность практически не изменилась.

|Настройки                |По умолчанию|Pgconfigurator      |PgTune|Мое предложение|Комментарии|
|:--------                |------------|--------------------|---- |-----------|
|shared_preload_libraries |-           |'pg_stat_statements'| -   | -  | В этом параметре задаются библиотеки, которые будут загружаться при запуске сервера. |
|track_io_timing          |of          | on                 | -   | of |Включает мониторинг времени чтения и записи блоков. |
|track_functions          |none        |pl                  | -   | none |Включает подсчёт вызовов функций и времени их выполнения. Значение pl включает отслеживание только функций на процедурном языке, а all — также функций на языках SQL и C. |
|tps                      |1875|                            | -   |  |
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
|checkpoint_completion_target | 0.9 | 0.9 | Чтобы избежать «заваливания» системы ввода/вывода при резкой интенсивной записи страниц, запись «грязных» буферов во время контрольной точки растягивается на определённый период времени. Этот период управляется параметром checkpoint_completion_target, который задаётся как часть интервала между контрольными точками.  Со значением 0.9, заданным по умолчанию, можно ожидать, что PostgreSQL завершит процедуру контрольной точки незадолго до следующей запланированной (примерно на 90% выполнения предыдущей контрольной точки).|
|max_wal_size | 1024 MB | 1024 MB | - |
|min_wal_size | 512 MB | 80 MB | - |

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
|bgwriter_delay | 200ms | 200ms | В числе специальных процессов сервера есть процесс фоновой записи, задача которого — осуществлять запись «грязных» (новых или изменённых) общих буферов на диск. Задаёт задержку между раундами активности процесса фоновой записи. Во время раунда этот процесс осуществляет запись некоторого количества загрязнённых буферов (это настраивается следующими параметрами). Затем он засыпает на время bgwriter_delay, и всё повторяется снова. | 
|bgwriter_lru_maxpages | 100 | 100 | Задаёт максимальное число буферов, которое сможет записать процесс фоновой записи за раунд активности.|
|bgwriter_lru_multiplier | 2.0 | 2.0 | Число загрязнённых буферов, записываемых в очередном раунде, зависит от того, сколько новых буферов требовалось серверным процессам в предыдущих раундах. Средняя недавняя потребность умножается на bgwriter_lru_multiplier и предполагается, что именно столько буферов потребуется на следующем раунде. |
|bgwriter_flush_after | 0 | 512 kb | Если объём всех записанных «грязных» страниц превысит заданное этим параметром значение, то Background Writer заставит ОС записать данные из своего кэша непосредственно на диск. |

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
|max_worker_processes | 1 | 8 | Максимальное число фоновых процессов, которое можно запустить в текущей системе. |
|max_parallel_workers_per_gather | 1 | 4 | Задаёт максимальное число рабочих процессов, которые могут запускаться одним узлом Gather или Gather Merge. Параллельные рабочие процессы берутся из пула процессов, контролируемого параметром max_worker_processes, в количестве, ограничиваемом значением max_parallel_workers. |
|max_parallel_maintenance_workers | 1 | 4 | Задаёт максимальное число рабочих процессов, которые могут запускаться одной служебной командой. В настоящее время параллельные процессы может использовать только CREATE INDEX при построении индекса-B-дерева и VACUUM без указания FULL. |
|max_parallel_workers | 1 | 8 | - |
|parallel_leader_participation | on | on |определяет, будет ли ведущий процесс участвовать в параллельном выполнении запроса|

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
