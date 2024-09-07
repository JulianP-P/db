Изменение параметров
| Название параметра | Новые настройки | Старые настройки |
|--------------------|:---------------:|:----------------:|
|max_connections     |      40         |        100       | 
|shared_buffers      |      1GB        |       128 MB     | 
|effective_cache_size|      3GB        |       4 GB       | 
|maintenance_work_mem|      512MB      |       64MB       |
|checkpoint_completion_target| 0.9     |        0.9       |
|wal_buffers         |     16MB        |        -1        |
|default_statistics_target|  500       |        100       |
|random_page_cost    |      4          |        4.0       |
|effective_io_concurrency |   2        |        1         |
|work_mem            |     6553kB      |        4MB       |
|min_wal_size        |       4GB       |    |
|max_wal_size        |      16GB       |        1 GB      |

Команда для теста
```
pgbench -c20 -P 6 -T 180 -U postgres postgres
```

| тест с новыми настройками | тест со старыми настройками |
|---------------------------|-----------------------------|
| transaction type: <builtin: TPC-B (sort of)> | transaction type: <builtin: TPC-B (sort of)> | 
| scaling factor: 1 | scaling factor: 1 |
| query mode: simple | query mode: simple |
| number of clients: 20 | number of clients: 20 |
| number of threads: 1 | number of threads: 1 |
| maximum number of tries: 1 | maximum number of tries: 1 |
| duration: 180 s | duration: 180 s | 
| number of transactions actually processed: 227999 | number of transactions actually processed: 229560 |
| number of failed transactions: 0 (0.000%) | number of failed transactions: 0 (0.000%) |
| latency average = 15.786 ms | latency average = 15.679 ms |
| latency stddev = 15.524 ms | latency stddev = 15.335 ms |
| initial connection time = 41.338 ms | initial connection time = 38.796 ms |
| tps = 1266.674193 (without initial connection time) | tps = 1275.339540 (without initial connection time) |


1) Создание таблицы и заполнение ее
```
postgres=# CREATE TABLE student(
 id serial,
 fio char(100)
);
CREATE TABLE
postgres=# INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,1000000);
INSERT 0 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 135 MB
(1 row)
```
2) Обновление полей таблицы. Из-за того, что в короткое время было обнолено большое количество полей, автовакуум не успел отработать и таблица распухла в 6 раз (5 апдейтов + изначальный размер таблицы).
```
postgres=# UPDATE student SET fio = 'name1';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name12';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name123';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name1234';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name12345';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 808 MB
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% | last_autovacuum 
---------+------------+------------+--------+-----------------
 student |    1000000 |    4999944 |    499 | 
(1 row)

```
3) После отработки автовакуума размер таблицы не поменялся, т.к. произошла фрагментация памяти.
```
postgres=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1000000 |          0 |      0 | 2024-09-07 18:02:13.323372+03
(1 row)

postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 808 MB
(1 row)
```
4) Новые записи заняли фрагментированную память, поэтому размер таблицы не увеличился
```
postgres=# UPDATE student SET fio = 'name123456';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name1234567';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name12345678';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name123456789';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name1234567890';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 808 MB
(1 row)

postgres=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1000000 |    2999826 |    299 | 2024-09-07 18:06:12.056018+03
(1 row)
```
5)Отключение автовакуума и обновление полей таблицы. Размер таблицы увеличился в 11 раз по сравнению с начальным.
```
postgres=# alter table student set (autovacuum_enabled = off);
postgres=# UPDATE student SET fio = 'name12345678901';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name123456789012';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name1234567890123';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name12345678901234';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name123456789012345';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name1234567890123456';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name12345678901234567';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name123456789012345678';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name1234567890123456789';
UPDATE 1000000
postgres=# UPDATE student SET fio = 'name12345678901234567890';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 1482 MB
(1 row)
```
6) После включение автовакуума размер таблицы не изменился из-за фрагментации.
```
postgres=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1003098 |          0 |      0 | 2024-09-07 18:12:50.228346+03
(1 row)

postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 1482 MB
(1 row)
```
7) После запуска вакуум фул размер таблицы вернулся к первоначальному.
```
postgres=# vacuum full;
VACUUM
postgres=# SELECT relname, n_live_tup, n_dead_tup,
trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 student |    1003098 |          0 |      0 | 2024-09-07 18:12:50.228346+03
(1 row)

postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty 
----------------
 135 MB
(1 row)
```
