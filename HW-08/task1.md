# Нагрузочное тестирование и тюнинг PostgreSQL
Для получения оптимальных параметров был проведен ряд тестов с разными конфигурациями. Источники конфигураций:
- конфигурация, которая идет по умолчанию
-  [Pgconfigurator](https://pgconfigurator.cybertec.at/)
-  [PgTune](https://pgtune.fariton.ru/)
- собственное предложение на основе рекомендаций из интернета
  
Команда для теста:
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
# Connections num: 100
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
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '11 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = on

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

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
# Connections num: 100
# Data Storage: ssd

max_connections = 100
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 10485kB
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
# Connectivity
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '4 GB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '11 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = on

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

# Parallel queries:
max_worker_processes = 8
max_parallel_workers_per_gather = 4
max_parallel_maintenance_workers = 4
max_parallel_workers = 8
parallel_leader_participation = on

```
## Сравнение
Производительность при разных конфигурациях:
|                  |По умолчанию|Pgconfigurator|PgTune  |Мое предложение|
|:-----------------|:-----------|:-------------|:-------|---------------|
|Производительность|2338        |6190          |2335    |4 GB           | 

После было произведенно несколько тестов для выяснения, какие настройки повлияли на производительность больше всего.
Во время теста применялись новые параметры из определенной группы. Все остальные параметры оставались по умолчанию.
### 1) Memory Settings
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('shared_buffers', 'work_mem', 'maintenance_work_mem', 'huge_pages', 'effective_cache_size', 'effective_io_concurrency', 'random_page_cost');
```
При применении следующих настроек производительность не изменилась.

|Настройки                |По умолчанию|Pgconfigurator|PgTune  |Мое предложение|Комментарии|
|:------------------------|:-----------|:-------------|:-------|:--------------|-----------|
|shared_buffers           |128 MB      |1024 MB       |4 GB    |4 GB           | Буфер для разделяемой памяти рекомендуется брать в 25% от ОЗУ |
|work_mem                 |4 MB        |32 MB         |10485 kB|41 MB          | Память, которая будет используется при обработке запросов. Выделяется для каждого процесса. Значение посчитано по формуле Total RAM * 0,25 / max_connections. |
|maintenance_work_mem     |64 MB       |320 MB        |1 GB    |64 MB          | Максимальный объём памяти для операций VACUUM, CREATE INDEX и ALTER TABLE ADD FOREIGN KEY. Объем данных в бд небольшой, данные команды выполняются редко, поэтому взято значение по умолчанию. 
|huge_pages               |try         |off           |off     |off            | Запрашивание огромных таблиц из общей памяти отключено. В докере для использования этой функции необходимо провести доп исследования для использования данного функционала.|
|effective_cache_size     |4 GB        |11 GB         |12 GB   |12 GB          | Оценка памяти, доступной для кэширования диска, рекомендуется брать в 50-75% от ОЗУ |
|effective_io_concurrency |1           |100           |200     |200            | Число одновременных операций ввода-вывода. Параметр зависит от типа диска. При SSD стоит выбирать значения в несколько сотен|
|random_page_cost         |4           |1.25          |1.1     |1              | Стоимость чтения одной произвольной страницы с диска. Параметр зависит от типа диска. При SSD стоит выбирать значения ближе к 1|
|**tps**                  |2338        |2337          |2334    |2339           | - |

### 2) Monitoring
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('shared_preload_libraries', 'track_io_timing', 'track_functions');
```
PgTune не предложил никаких настроек для этого блока. По моему мнению, самые оптимальные параметры, те, которые предложены по умолчанию, т.к. мониторинг и загрузка библиотек создает дополнительную нагрузку. Поэтому в таблицу сравнения попали парметры по умолчанию и параметры от Pgconfigurator.
При применении следующих настроек производительность не изменилась.

|Настройки                |По умолчанию|Pgconfigurator      |Комментарии|
|:--------                |:-----------|:-------------------|------|
|shared_preload_libraries |-           |'pg_stat_statements'|В этом параметре задаются библиотеки, которые будут загружаться при запуске сервера. |
|track_io_timing          |of          |on                  |Включает мониторинг времени чтения и записи блоков. |
|track_functions          |none        |pl                  |Включает подсчёт вызовов функций и времени их выполнения. Значение pl включает отслеживание только функций на процедурном языке, а all — также функций на языках SQL и C. |
|**tps**                  |2338        |2318                | |

### 3) Replication
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('wal_level', 'max_wal_senders', 'synchronous_commit');
```
Изменение этих метрик привело к увеличению производительности.

|Настройки         |По умолчанию|Pgconfigurator|Мое предложение|Комментарии|
|:-----------------|:-----------|:-------------|:--------------|-----------|
|wal_level         |replica     |replica       |minimal        |Параметр определяет, как много информации записывается в WAL. Возможные значения replica, minimal, logical. Со значением replica в журнал записываются данные, необходимые для поддержки архивирования WAL и репликации, включая запросы только на чтение на ведомом сервере. Minimal оставляет только информацию, необходимую для восстановления после сбоя или аварийного отключения. Так как в данном случае не предусмотрена репликация, то wal_level был понижен до минимального уровня.|
|max_wal_senders   |10          |0             |0              |Задаёт максимально допустимое число одновременных подключений ведомых серверов или клиентов потокового копирования. Задан 0, т.к. ведомые сервера отсутствуют. |
|synchronous_commit|on          |off           |off            |Параметр, который определяет, когда транзакции считаются зафиксированными и в какой момент клиент получает подтверждение об этом. on - Транзакции считаются зафиксированными только после того, как записи WAL будут записаны на диск.  off - Транзакции считаются зафиксированными сразу после записи в журнал WAL, без ожидания записи на диск. Другие возможные значения: remote_write, local, remote_apply. Так как в задании предлагается принебречь возможной потерей данных, то был выбран параметр off. |
|**tps**           |2338        |7436          |7422           | |

**Наибольший вклад внесла метрика synchronous_commit.**

### 4) Checkpointing
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('checkpoint_timeout', 'checkpoint_completion_target', 'max_wal_size', 'min_wal_size');
```
При применении следующих настроек производительность не изменилась.

|Настройки                   |По умолчанию|Pgconfigurator|PgTune |Мое предложение|Комментарии|
|:---------------------------|:-----------|:-------------|:------|:--------------|:----|
|checkpoint_timeout          |5 min       |15 min        |-      |30 min         |Параметр, который устанавливает максимальное время между автоматическими контрольными точками в WAL. Количество контрольных точек влияет на производительность, поэтому для бОльшой производителньости это число можно взять побольше. При этом при сбое восстановление будет занимать больше времени.|
|checkpoint_completion_target|0.9         |0.9           |0.9    |0.9            |Чтобы избежать «заваливания» системы ввода/вывода при резкой интенсивной записи страниц, запись «грязных» буферов во время контрольной точки растягивается на определённый период времени. Этот период управляется параметром checkpoint_completion_target, который задаётся как часть интервала между контрольными точками.  Со значением 0.9, заданным по умолчанию, можно ожидать, что PostgreSQL завершит процедуру контрольной точки незадолго до следующей запланированной (примерно на 90% выполнения предыдущей контрольной точки).|
|max_wal_size                |1024 MB     |1024 MB       |8 GB   |8 GB           |Увеличение max_wal_size может уменьшить количество контрольных точек и, как следствие, снизить задержку при записи. Однако это увеличивает время восстановления после сбоя. Для уменьшении кол-во контрольных точек и для уменьшения кол-ва операций записи на диск. |
|min_wal_size                |80 MB       |512 MB        |2 GB   |2 GB           | min_wal_size установлен на достаточно высоком значении, чтобы уменьшить количество операций записи на диск. |
|**tps**                     |2338        |2337          |2332   | -             | - |

### 5) WAL writing
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('wal_compression', 'wal_buffers');
```
По моему мнению, самые оптимальные параметры, те, которые предложены по умолчанию.
При применении следующих настроек производительность не изменилась.

|Настройки      |По умолчанию|Pgconfigurator|PgTune|Комментарии|
|:--------      |:-----------|:-------------|:-----|:----------|
|wal_compression|off         |on            |-     |Сжатие оказывает дполнительную нагрузку на процессор. Сжатие нужно для уменьшение wal файла, чтобы улучшить репликацию. В данной задаче репликация не используется.|
|wal_buffers    |4 MB        |-1            |16 MB |При большом количестве одновременных подключений, то более высокое значение может повысить производительность. В данном тесте используется 50 коннектов, что не так много.|
|**tps**        |2338        |2335          |2326  ||

### 6) Parallel queries
```sql
select name, setting, unit, sourcefile from pg_settings where name in ('max_worker_processes', 'max_parallel_workers_per_gather', 'max_parallel_maintenance_workers', 'max_parallel_workers', 'parallel_leader_participation');
```
По моему мнению, самые оптимальные параметры, те, которые предложены по умолчанию
Pgconfigurator и PgTune совпадают.
|Настройки                       |По умолчанию|Pgconfigurator и PgTune|Комментарии|
|:-------------------------------|:-----------|:-------------|:-----|
|max_worker_processes            |8           |8             |Максимальное число фоновых процессов, которое можно запустить в текущей системе. |
|max_parallel_workers_per_gather |2           |4             |Задаёт максимальное число рабочих процессов, которые могут запускаться одним узлом Gather или Gather Merge. Параллельные рабочие процессы берутся из пула процессов, контролируемого параметром max_worker_processes, в количестве, ограничиваемом значением max_parallel_workers. |
|max_parallel_maintenance_workers|2           |4             |Задаёт максимальное число рабочих процессов, которые могут запускаться одной служебной командой. В настоящее время параллельные процессы может использовать только CREATE INDEX при построении индекса-B-дерева и VACUUM без указания FULL. |
|max_parallel_workers            |8           |8             |- |
|**tps**                         |2338        |2337          ||


**Наибольшее значение на производительность оказал параметр synchronous_commit.**
