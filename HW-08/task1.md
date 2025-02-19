1. Настройте выполнение контрольной точки раз в 30 секунд.
```sql
alter system set checkpoint_timeout='30s';
```
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
```
pgbench -U postgres -T 600 locks
tps = 1801.802730
```
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
```sql
SELECT pg_current_wal_insert_lsn();
-- перед тестом 0/927FAB58
-- после теста 0/BE5BF140
SELECT '0/BE5BF140'::pg_lsn - '0/927FAB58'::pg_lsn;
--735 856 104
```
За десять минут должно было создасться 20 контрольных точек. Это примерно 36 792 805 строк на одну контрольную точку.
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```sql
--команда для сброса статистики, запущенная до теста
select pg_stat_reset_shared('bgwriter');
--вывод статистики после теста
select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 20
checkpoints_req       | 0
checkpoint_write_time | 511936
checkpoint_sync_time  | 89
buffers_checkpoint    | 46051
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 7009
buffers_backend_fsync | 0
buffers_alloc         | 7000
stats_reset           | 2025-02-18 16:36:18.734982+00
```
Из статистики видно, что было все контрольные точки были созданы по расписанию, т.е. по достижению checkpoint_timeout (checkpoints_timed=20). Это связано с тем, что объем wal файлов не достигал max_wal_size (1 gb).

Насколько точно по расписанию запускается создание контрольной точки можно посмотреть в логах.

Для включения логгирования создания контрольных точек выполнена команда:
```sql
alter system set log_checkpoints=on;
```
```
2025-02-18 16:37:47.603 UTC [1] LOG:  parameter "log_checkpoints" changed to "on"
2025-02-18 16:38:46.664 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:39:13.067 UTC [29] LOG:  checkpoint complete: wrote 1794 buffers (10.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.393 s, sync=0.006 s, total=26.403 s; sync files=14, longest=0.003 s, average=0.001 s; distance=17847 kB, estimate=33085 kB
2025-02-18 16:39:16.070 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:39:43.077 UTC [29] LOG:  checkpoint complete: wrote 2062 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.984 s, sync=0.006 s, total=27.008 s; sync files=7, longest=0.003 s, average=0.001 s; distance=34889 kB, estimate=34889 kB
2025-02-18 16:39:46.080 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:40:13.076 UTC [29] LOG:  checkpoint complete: wrote 2150 buffers (13.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.983 s, sync=0.005 s, total=26.996 s; sync files=12, longest=0.003 s, average=0.001 s; distance=35106 kB, estimate=35106 kB
2025-02-18 16:40:16.079 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:40:43.069 UTC [29] LOG:  checkpoint complete: wrote 2058 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.980 s, sync=0.005 s, total=26.990 s; sync files=7, longest=0.003 s, average=0.001 s; distance=34973 kB, estimate=35093 kB
2025-02-18 16:40:46.069 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:41:13.081 UTC [29] LOG:  checkpoint complete: wrote 2275 buffers (13.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.993 s, sync=0.004 s, total=27.013 s; sync files=9, longest=0.003 s, average=0.001 s; distance=35091 kB, estimate=35092 kB
2025-02-18 16:41:16.084 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:41:43.068 UTC [29] LOG:  checkpoint complete: wrote 2058 buffers (12.6%); 0 WAL file(s) added, 0 removed, 3 recycled; write=26.973 s, sync=0.005 s, total=26.984 s; sync files=5, longest=0.004 s, average=0.001 s; distance=35052 kB, estimate=35088 kB
2025-02-18 16:41:46.071 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:42:13.088 UTC [29] LOG:  checkpoint complete: wrote 2291 buffers (14.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.997 s, sync=0.004 s, total=27.017 s; sync files=9, longest=0.003 s, average=0.001 s; distance=35242 kB, estimate=35242 kB
2025-02-18 16:42:16.091 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:42:43.089 UTC [29] LOG:  checkpoint complete: wrote 2061 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.989 s, sync=0.004 s, total=26.998 s; sync files=6, longest=0.003 s, average=0.001 s; distance=35218 kB, estimate=35240 kB
2025-02-18 16:42:46.092 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:43:13.082 UTC [29] LOG:  checkpoint complete: wrote 2292 buffers (14.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.981 s, sync=0.003 s, total=26.991 s; sync files=9, longest=0.003 s, average=0.001 s; distance=35343 kB, estimate=35343 kB
2025-02-18 16:43:16.085 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:43:43.068 UTC [29] LOG:  checkpoint complete: wrote 2057 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.972 s, sync=0.005 s, total=26.984 s; sync files=6, longest=0.003 s, average=0.001 s; distance=34697 kB, estimate=35278 kB
2025-02-18 16:43:46.072 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:44:13.072 UTC [29] LOG:  checkpoint complete: wrote 2271 buffers (13.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.981 s, sync=0.003 s, total=27.001 s; sync files=9, longest=0.003 s, average=0.001 s; distance=34242 kB, estimate=35175 kB
2025-02-18 16:44:16.075 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:44:43.077 UTC [29] LOG:  checkpoint complete: wrote 2059 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.974 s, sync=0.013 s, total=27.003 s; sync files=7, longest=0.010 s, average=0.002 s; distance=34910 kB, estimate=35148 kB
2025-02-18 16:44:46.081 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:45:13.087 UTC [29] LOG:  checkpoint complete: wrote 2266 buffers (13.8%); 0 WAL file(s) added, 0 removed, 3 recycled; write=26.985 s, sync=0.004 s, total=27.007 s; sync files=10, longest=0.003 s, average=0.001 s; distance=34264 kB, estimate=35060 kB
2025-02-18 16:45:16.089 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:45:43.072 UTC [29] LOG:  checkpoint complete: wrote 2054 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.972 s, sync=0.005 s, total=26.984 s; sync files=5, longest=0.002 s, average=0.001 s; distance=34790 kB, estimate=35033 kB
2025-02-18 16:45:46.076 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:46:13.080 UTC [29] LOG:  checkpoint complete: wrote 2292 buffers (14.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.986 s, sync=0.004 s, total=27.005 s; sync files=13, longest=0.003 s, average=0.001 s; distance=34944 kB, estimate=35024 kB
2025-02-18 16:46:16.083 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:46:43.084 UTC [29] LOG:  checkpoint complete: wrote 2054 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.980 s, sync=0.005 s, total=27.001 s; sync files=6, longest=0.003 s, average=0.001 s; distance=34618 kB, estimate=34983 kB
2025-02-18 16:46:46.087 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:47:13.089 UTC [29] LOG:  checkpoint complete: wrote 2971 buffers (18.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.993 s, sync=0.003 s, total=27.002 s; sync files=15, longest=0.003 s, average=0.001 s; distance=35038 kB, estimate=35038 kB
2025-02-18 16:47:16.092 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:47:43.103 UTC [29] LOG:  checkpoint complete: wrote 2061 buffers (12.6%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.990 s, sync=0.004 s, total=27.011 s; sync files=6, longest=0.002 s, average=0.001 s; distance=35117 kB, estimate=35117 kB
2025-02-18 16:47:46.106 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:48:13.087 UTC [29] LOG:  checkpoint complete: wrote 2284 buffers (13.9%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.973 s, sync=0.004 s, total=26.981 s; sync files=13, longest=0.003 s, average=0.001 s; distance=34564 kB, estimate=35062 kB
2025-02-18 16:48:16.090 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:48:43.109 UTC [29] LOG:  checkpoint complete: wrote 2055 buffers (12.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.988 s, sync=0.017 s, total=27.020 s; sync files=6, longest=0.011 s, average=0.003 s; distance=34699 kB, estimate=35025 kB
2025-02-18 16:49:46.162 UTC [29] LOG:  checkpoint starting: time
2025-02-18 16:50:13.046 UTC [29] LOG:  checkpoint complete: wrote 2923 buffers (17.8%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.870 s, sync=0.003 s, total=26.884 s; sync files=15, longest=0.001 s, average=0.001 s; distance=31244 kB, estimate=34647 kB
```
Запуск создания контрольных точек происходит ровно через 30 с.

5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

synchronous_commit = on, tps = 1801.802730


on (по умолчанию). Транзакции считаются зафиксированными только после того, как записи WAL будут записаны на диск. Это обеспечивает максимальную надёжность данных.

off. Транзакции считаются зафиксированными сразу после записи в журнал WAL, без ожидания записи на диск. Это может значительно повысить производительность, но с риском потери данных при сбое

Таким образом при synchronous_commit=off меньше накладных расходов на ожидание фиксации транзакции 


Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. 
Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 
Что и почему произошло? как проигнорировать ошибку и продолжить работу?
