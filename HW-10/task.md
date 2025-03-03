
Размер таблиц

```
\d
 Схема   |          Имя          |               Тип               | Владелец |  Хранение  | Метод доступа |   Размер   |     Описание      
----------+-----------------------+---------------------------------+----------+------------+---------------+------------+-------------------
 bookings | aircrafts             | таблица                         | postgres | постоянное | heap          | 16 kB      | Самолеты
 bookings | airports              | таблица                         | postgres | постоянное | heap          | 48 kB      | Аэропорты
 bookings | boarding_passes       | таблица                         | postgres | постоянное | heap          | 33 MB      | Посадочные талоны
 bookings | bookings              | таблица                         | postgres | постоянное | heap          | 13 MB      | Бронирования
 bookings | flights               | таблица                         | postgres | постоянное | heap          | 3168 kB    | Рейсы
 bookings | flights_flight_id_seq | последовательность              | postgres | постоянное |               | 8192 bytes | 
 bookings | flights_v             | представление                   | postgres | постоянное |               | 0 bytes    | Рейсы
 bookings | routes                | материализованное представление | postgres | постоянное | heap          | 152 kB     | Маршруты
 bookings | seats                 | таблица                         | postgres | постоянное | heap          | 96 kB      | Места
 bookings | ticket_flights        | таблица                         | postgres | постоянное | heap          | 68 MB      | Перелеты
 bookings | tickets               | таблица                         | postgres | постоянное | heap          | 48 MB      | Билеты
(11 строк)
```
Наибольший размер имеют таблицы:
1. ticket_flights (68 Mb)
2. tickets (48 Mb)
3. boarding_passes (33 Mb)
4. bookings (13 Mb)

Данные таблицы можно секционировать по полям
1. ticket_flights (ticket_no - номер билета - хэш)
2. tickets (48 Mb) (ticket_no - номер билета - хэш)
3. boarding_passes (33 Mb) (flight_id - по списку, проверила, но нет. (ticket_no - номер билета - хэш)
```sql
select count(distinct flight_id) as count from boarding_passes;
 count 
-------
 11518
(1 строка)
```
5. bookings (13 Mb) (book_date - диапазон)

Для работы будет взята таблица ticket_flights


```
create table ticket_flights_cp (like ticket_flights including all) partition by hash ( ticket_no);

create table ticket_flights_0 partition of ticket_flights_cp for values with (modulus 10, remainder 0);CREATE TABLE
demo=# create table ticket_flights_1 partition of ticket_flights_cp for values with (modulus 10, remainder 1);
CREATE TABLE
demo=# create table ticket_flights_2 partition of ticket_flights_cp for values with (modulus 10, remainder 2);
CREATE TABLE
demo=# create table ticket_flights_3 partition of ticket_flights_cp for values with (modulus 10, remainder 3);
CREATE TABLE
demo=# create table ticket_flights_4 partition of ticket_flights_cp for values with (modulus 10, remainder 4);
CREATE TABLE
demo=# create table ticket_flights_5 partition of ticket_flights_cp for values with (modulus 10, remainder 5);
CREATE TABLE
demo=# create table ticket_flights_6 partition of ticket_flights_cp for values with (modulus 10, remainder 6);
CREATE TABLE
demo=# create table ticket_flights_7 partition of ticket_flights_cp for values with (modulus 10, remainder 7);
CREATE TABLE
demo=# create table ticket_flights_8 partition of ticket_flights_cp for values with (modulus 10, remainder 8);
CREATE TABLE
demo=# create table ticket_flights_9 partition of ticket_flights_cp for values with (modulus 10, remainder 9);
```

```
SELECT
    nmsp_parent.nspname AS parent_schema,
    parent.relname      AS parent,
    nmsp_child.nspname  AS child_schema,
    child.relname       AS child
FROM pg_inherits
    JOIN pg_class parent            ON pg_inherits.inhparent = parent.oid
    JOIN pg_class child             ON pg_inherits.inhrelid   = child.oid
    JOIN pg_namespace nmsp_parent   ON nmsp_parent.oid  = parent.relnamespace
    JOIN pg_namespace nmsp_child    ON nmsp_child.oid   = child.relnamespace
WHERE parent.relname='ticket_flights_cp';
 parent_schema |      parent       | child_schema |      child       
---------------+-------------------+--------------+------------------
 bookings      | ticket_flights_cp | bookings     | ticket_flights_0
 bookings      | ticket_flights_cp | bookings     | ticket_flights_1
 bookings      | ticket_flights_cp | bookings     | ticket_flights_2
 bookings      | ticket_flights_cp | bookings     | ticket_flights_3
 bookings      | ticket_flights_cp | bookings     | ticket_flights_4
 bookings      | ticket_flights_cp | bookings     | ticket_flights_5
 bookings      | ticket_flights_cp | bookings     | ticket_flights_6
 bookings      | ticket_flights_cp | bookings     | ticket_flights_7
 bookings      | ticket_flights_cp | bookings     | ticket_flights_8
 bookings      | ticket_flights_cp | bookings     | ticket_flights_9
(10 rows)

```
```
insert into ticket_flights_cp select * from ticket_flights;
```
поиск случайного номера билета
```sql
demo=# explain analyze
select * from ticket_flights where ticket_no='0005432169756';
                                                              QUERY PLAN                                                              
--------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.42..16.47 rows=3 width=32) (actual time=0.099..0.100 rows=0 loops=1)
   Index Cond: (ticket_no = '0005432169756'::bpchar)
 Planning Time: 0.202 ms
 Execution Time: 0.129 ms
(4 rows)

demo=# explain analyze
select * from ticket_flights_cp where ticket_no='0005432169756';
                                                              QUERY PLAN                                                              
--------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on ticket_flights_0 ticket_flights_cp  (cost=4.44..15.95 rows=3 width=32) (actual time=0.049..0.051 rows=0 loops=1)
   Recheck Cond: (ticket_no = '0005432169756'::bpchar)
   ->  Bitmap Index Scan on ticket_flights_0_pkey  (cost=0.00..4.44 rows=3 width=0) (actual time=0.043..0.044 rows=0 loops=1)
         Index Cond: (ticket_no = '0005432169756'::bpchar)
 Planning Time: 0.356 ms
 Execution Time: 0.086 ms
(6 rows)
```

```
demo=# select * from ticket_flights_cp where ticket_no='0005432211371';
   ticket_no   | flight_id | fare_conditions |  amount  
---------------+-----------+-----------------+----------
 0005432211371 |     30625 | Economy         | 14000.00
(1 row)

demo=# UPDATE  * from ticket_flights_cp where ticket_no='0005432211371';
aircrafts            bookings.            pg_catalog.          ticket_flights       ticket_flights_3     ticket_flights_7     tickets              
airports             flights              pg_toast.            ticket_flights_0     ticket_flights_4     ticket_flights_8     
boarding_passes      flights_v            public.              ticket_flights_1     ticket_flights_5     ticket_flights_9     
bookings             information_schema.  seats                ticket_flights_2     ticket_flights_6     ticket_flights_cp    
demo=# UPDATE ticket_flights_cp SET  * from ticket_flights_cp where ticket_no='0005432211371';
amount           fare_conditions  flight_id        ticket_no        
demo=# UPDATE ticket_flights_cp SET amount =  * from ticket_flights_cp where ticket_no='0005432211371';

demo=# UPDATE ticket_flights_cp SET amount = 599 * from ticket_flights_cp where ticket_no='0005432211371';

demo=# UPDATE ticket_flights_cp SET amount = 599 where ticket_no='0005432211371';
UPDATE 1
demo=# select * from ticket_flights_cp where ticket_no='0005432211371';
   ticket_no   | flight_id | fare_conditions | amount 
---------------+-----------+-----------------+--------
 0005432211371 |     30625 | Economy         | 599.00
(1 row)

demo=# DELETE FROM ticket_flights_cp SET amount = 599 where ticket_no='0005432211371';
FROM
demo=# DELETE FROM ticket_flights_cp  where ticket_no='0005432211371';
USING  WHERE  
demo=# DELETE FROM ticket_flights_cp  where ticket_no='0005432211371';
DELETE 1
demo=# select * from ticket_flights_cp where ticket_no='0005432211371';
 ticket_no | flight_id | fare_conditions | amount 
-----------+-----------+-----------------+--------
(0 rows)

demo=# 
```

___
```
demo=# select min(book_date) from bookings limit 10;
          min           
------------------------
 2016-08-19 10:05:00+00
(1 строка)

demo=# select max(book_date) from bookings limit 10;
          max           
------------------------
 2016-10-13 14:00:00+00
(1 строка)

```

```
demo=# create table bookings_cp (like bookings including all) partition by range (book_date);
ERROR:  unique constraint on partitioned table must include all partitioning columns
ПОДРОБНОСТИ:  PRIMARY KEY constraint on table "bookings_cp" lacks column "book_date" which is part of the partition key.
demo=# create table bookings_cp (like bookings) partition by range (book_date);CREATE TABLE
```
```
demo=# create table bookings_2016_08 partition of bookings_cp for values from ('2016-08-01') to ('2016-09-01');
CREATE TABLE
demo=# create table bookings_2016_09 partition of bookings_cp for values from ('2016-09-01') to ('2016-10-01');
CREATE TABLE
demo=# create table bookings_2016_10 partition of bookings_cp for values from ('2016-10-01') to ('2016-11-01');
CREATE TABLE
```

```
insert INTO bookings_cp select * from bookings
bookings          bookings_2016_08  bookings_2016_10  
bookings.         bookings_2016_09  bookings_cp       
demo=# insert INTO bookings_cp select * from bookings;
INSERT 0 262788
demo=# select * from bo
boarding_passes   bookings.         bookings_2016_09  bookings_cp
bookings          bookings_2016_08  bookings_2016_10  
demo=# select * from bookings_2016_08 limit 10;
 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 000511   | 2016-08-28 23:40:00+00 |     26700.00
 0005E7   | 2016-08-31 05:25:00+00 |     28800.00
 000A39   | 2016-08-28 23:29:00+00 |     23400.00
 000B77   | 2016-08-28 22:39:00+00 |     68800.00
 000D3C   | 2016-08-30 14:23:00+00 |    173500.00
 001436   | 2016-08-29 10:10:00+00 |    101200.00
 001A6E   | 2016-08-31 14:06:00+00 |     21200.00
 001A9F   | 2016-08-29 07:29:00+00 |     55800.00
 001ED4   | 2016-08-29 06:52:00+00 |     56000.00
 00216F   | 2016-08-29 12:24:00+00 |     47200.00
(10 строк)

demo=# select * from bookings_2016_09 limit 10;
 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 00000F   | 2016-09-01 23:12:00+00 |    265700.00
 000012   | 2016-09-11 05:02:00+00 |     37900.00
 0002DB   | 2016-09-26 02:30:00+00 |    101500.00
 0002E0   | 2016-09-08 12:09:00+00 |     89600.00
 0002F3   | 2016-09-07 01:31:00+00 |     69600.00
 000352   | 2016-09-02 22:02:00+00 |    109500.00
 00044D   | 2016-09-26 20:24:00+00 |      6000.00
 00044E   | 2016-09-14 01:39:00+00 |    140100.00
 0004B0   | 2016-09-25 05:00:00+00 |     12000.00
 0004E1   | 2016-09-28 13:34:00+00 |    139300.00
(10 строк)

demo=# select * from bookings_2016_10 limit 10;
 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 000068   | 2016-10-13 10:27:00+00 |     18100.00
 000181   | 2016-10-08 09:28:00+00 |    131800.00
 0002D8   | 2016-10-05 17:40:00+00 |     23600.00
 00034E   | 2016-10-02 12:52:00+00 |     73300.00
 000374   | 2016-10-10 06:13:00+00 |    136200.00
 00053F   | 2016-10-03 23:15:00+00 |      6000.00
 0005F4   | 2016-10-06 22:14:00+00 |     95400.00
 0006F5   | 2016-10-02 18:10:00+00 |     80200.00
 000836   | 2016-10-10 19:28:00+00 |     23400.00
 000862   | 2016-10-05 10:23:00+00 |     45500.00
```

```
demo=# explain analyze                                                                        
select count(*) from bookings_cp where book_date between '2016-09-25' and '2016-10-05';
                                                                                 QUERY PLAN                                                                                  
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=4852.13..4852.14 rows=1 width=8) (actual time=10.470..12.610 rows=1 loops=1)
   ->  Gather  (cost=4851.91..4852.12 rows=2 width=8) (actual time=10.377..12.601 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=3851.91..3851.92 rows=1 width=8) (actual time=7.012..7.012 rows=1 loops=3)
               ->  Parallel Append  (cost=0.00..3794.31 rows=23040 width=0) (actual time=0.010..6.342 rows=18591 loops=3)
                     ->  Parallel Seq Scan on bookings_2016_09 bookings_cp_1  (cost=0.00..2518.14 rows=19362 width=0) (actual time=0.003..3.151 rows=11052 loops=3)
                           Filter: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
                           Rows Removed by Filter: 44184
                     ->  Parallel Seq Scan on bookings_2016_10 bookings_cp_2  (cost=0.00..1160.98 rows=13165 width=0) (actual time=0.012..3.229 rows=11308 loops=2)
                           Filter: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
                           Rows Removed by Filter: 26884
 Planning Time: 0.325 ms
 Execution Time: 12.670 ms
(14 rows)
demo=# explain analyze
select count(*) from bookings where book_date between '2016-09-25' and '2016-10-05';
                                                                              QUERY PLAN                                                                               
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=5073.84..5073.85 rows=1 width=8) (actual time=12.076..14.646 rows=1 loops=1)
   ->  Gather  (cost=5073.73..5073.84 rows=1 width=8) (actual time=11.990..14.639 rows=2 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Partial Aggregate  (cost=4073.73..4073.74 rows=1 width=8) (actual time=9.751..9.751 rows=1 loops=2)
               ->  Parallel Seq Scan on bookings  (cost=0.00..3992.72 rows=32405 width=0) (actual time=0.010..8.760 rows=27887 loops=2)
                     Filter: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
                     Rows Removed by Filter: 103507
 Planning Time: 0.562 ms
 Execution Time: 14.708 ms
(10 rows)
```
ПОсле создания индекса
```
                                                                                        QUERY PLAN                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=1881.91..1881.92 rows=1 width=8) (actual time=7.277..7.278 rows=1 loops=1)
   ->  Append  (cost=0.29..1743.61 rows=55321 width=0) (actual time=0.010..5.728 rows=55774 loops=1)
         ->  Index Only Scan using bookings_2016_09_book_date_idx on bookings_2016_09 bookings_cp_1  (cost=0.29..874.93 rows=32932 width=0) (actual time=0.009..1.961 rows=33157 loops=1)
               Index Cond: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
               Heap Fetches: 0
         ->  Index Only Scan using bookings_2016_10_book_date_idx on bookings_2016_10 bookings_cp_2  (cost=0.29..592.07 rows=22389 width=0) (actual time=0.006..1.309 rows=22617 loops=1)
               Index Cond: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
               Heap Fetches: 0
 Planning Time: 0.401 ms
 Execution Time: 7.308 ms
(10 rows)
```
```
demo=# create index book_date_old ON bookings (book_date);
CREATE INDEX
demo=# explain analyze
select count(*) from bookings where book_date between '2016-09-25' and '2016-10-05';
                                                                          QUERY PLAN                                                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=1599.92..1599.93 rows=1 width=8) (actual time=4.877..4.878 rows=1 loops=1)
   ->  Index Only Scan using book_date_old on bookings  (cost=0.42..1462.20 rows=55089 width=0) (actual time=0.131..3.233 rows=55774 loops=1)
         Index Cond: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
         Heap Fetches: 0
 Planning Time: 0.381 ms
 Execution Time: 4.920 ms
(6 rows)
```

```
demo=# select * from bookings_cp where book_ref = '000068';
 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 000068   | 2016-10-13 10:27:00+00 |     18100.00
(1 строка)

demo=# update 
aircrafts            flights              ticket_flights_3
airports             flights_v            ticket_flights_4
boarding_passes      information_schema.  ticket_flights_5
bookings             public.              ticket_flights_6
bookings.            seats                ticket_flights_7
bookings_2016_08     ticket_flights       ticket_flights_8
bookings_2016_09     ticket_flights_0     ticket_flights_9
bookings_2016_10     ticket_flights_1     ticket_flights_cp
bookings_cp          ticket_flights_2     tickets
demo=# update bookings_cp SET total_amount = 19100.0

demo=# update bookings_cp SET total_amount = '19100.0'

demo=# update bookings_cp SET total_amount = '19100.0' 

demo=# update bookings_cp SET total_amount = '19100.0' where book_ref = '000068';
UPDATE 1
demo=# select * from bookings_cp where book_ref = '000068';
 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 000068   | 2016-10-13 10:27:00+00 |     19100.00
(1 строка)

demo=# delete bookings_cp  where book_ref = '000068';
ERROR:  syntax error at or near "bookings_cp"
СТРОКА 1: delete bookings_cp  where book_ref = '000068';
                 ^
demo=# delete from bookings_cp  where book_ref = '000068';
DELETE 1
demo=# select * from bookings_cp where book_ref = '000068';
 book_ref | book_date | total_amount 
----------+-----------+--------------
(0 строк)

demo=# 
```


Всё это обычно полезно только для очень больших таблиц. Какие именно таблицы выиграют от секционирования, зависит от конкретного приложения, хотя, как правило, это следует применять для таблиц, размер которых превышает объём ОЗУ сервера.



Создание секционированной таблицы:
Преобразуйте таблицу в секционированную с выбранным типом секционирования.
Например, если вы выбрали секционирование по диапазону дат бронирования, создайте секции по месяцам или годам.
Миграция данных:
Перенесите существующие данные из исходной таблицы в секционированную структуру.
Убедитесь, что все данные правильно распределены по секциям.
Оптимизация запросов:
Проверьте, как секционирование влияет на производительность запросов. Выполните несколько выборок данных до и после секционирования для оценки времени выполнения.
Оптимизируйте запросы при необходимости (например, добавьте индексы на ключевые столбцы).

Тестирование решения:
Протестируйте секционирование, выполняя несколько запросов к секционированной таблице.
Проверьте, что операции вставки, обновления и удаления работают корректно.

Документирование:
Добавьте комментарии к коду, поясняющие выбранный тип секционирования и шаги его реализации.
Опишите, как секционирование улучшает производительность запросов и как оно может быть полезно в реальных условиях.

Критерии оценивания:
Корректность секционирования – таблица должна быть разделена логично и эффективно.
Выбор типа секционирования – обоснование выбранного типа (например, секционирование по диапазону дат рейсов или по месту отправления/прибытия).
Работоспособность решения – код должен успешно выполнять секционирование без ошибок.
Оптимизация запросов – после секционирования, запросы к таблице должны быть оптимизированы (например, быстрее выполняться для конкретных диапазонов).
Комментирование – код должен содержать поясняющие комментарии, объясняющие выбор секционирования и основные шаги.
Формат сдачи:
SQL-скрипты с реализованным секционированием.
Краткий отчет с описанием процесса и результатами тестирования.
Пример запросов и результаты до и после секционирования.
