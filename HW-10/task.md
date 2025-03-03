otus-PostgreSQL-2024-11-Прощина Юлия

Домашняя работа выполнялась в докер контейнере, версия postgres 14.15.

# Секционирование
Исследование бд. 

Решение о партиционировании зависит о размера таблиц.
Размер таблиц:

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

Данные таблицы можно секционировать:
1. ticket_flights (По хэшу - поле ticket_no)
2. tickets (48 Mb) (По хэшу - поле ticket_no)
3. boarding_passes (33 Mb) (По хэшу - поле ticket_no)
5. bookings (13 Mb) (book_date - диапазон)

Для домашней работы будут партиционированы таблицы ticket_flights и bookings.

### Таблица ticket_flights
1. Создание таблицы и партиций
```
create table ticket_flights_cp (like ticket_flights including all) partition by hash ( ticket_no);

create table ticket_flights_0 partition of ticket_flights_cp for values with (modulus 10, remainder 0);
create table ticket_flights_1 partition of ticket_flights_cp for values with (modulus 10, remainder 1);
create table ticket_flights_2 partition of ticket_flights_cp for values with (modulus 10, remainder 2);
create table ticket_flights_3 partition of ticket_flights_cp for values with (modulus 10, remainder 3);
create table ticket_flights_4 partition of ticket_flights_cp for values with (modulus 10, remainder 4);
create table ticket_flights_5 partition of ticket_flights_cp for values with (modulus 10, remainder 5);
create table ticket_flights_6 partition of ticket_flights_cp for values with (modulus 10, remainder 6);
create table ticket_flights_7 partition of ticket_flights_cp for values with (modulus 10, remainder 7);
create table ticket_flights_8 partition of ticket_flights_cp for values with (modulus 10, remainder 8);
create table ticket_flights_9 partition of ticket_flights_cp for values with (modulus 10, remainder 9);
```
2. Проверка, что все создалось, как надо
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

--parent_schema |      parent       | child_schema |      child       
-----------------+-------------------+--------------+------------------
--bookings      | ticket_flights_cp | bookings     | ticket_flights_0
--bookings      | ticket_flights_cp | bookings     | ticket_flights_1
--bookings      | ticket_flights_cp | bookings     | ticket_flights_2
--bookings      | ticket_flights_cp | bookings     | ticket_flights_3
--bookings      | ticket_flights_cp | bookings     | ticket_flights_4
--bookings      | ticket_flights_cp | bookings     | ticket_flights_5
--bookings      | ticket_flights_cp | bookings     | ticket_flights_6
--bookings      | ticket_flights_cp | bookings     | ticket_flights_7
--bookings      | ticket_flights_cp | bookings     | ticket_flights_8
--bookings      | ticket_flights_cp | bookings     | ticket_flights_9
--(10 rows)
```
3. Копируем данные из старой таблицы:
```
insert into ticket_flights_cp select * from ticket_flights;
```
4. Сравнение производительности запроса в партицируемой таблице и в обычной:

```sql
-- Поиск случайного номера билета
-- таблица без партиций
explain analyze
select * from ticket_flights where ticket_no='0005432169756';

                                                             QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.42..16.47 rows=3 width=32) (actual time=0.099..0.100 rows=0 loops=1)
  Index Cond: (ticket_no = '0005432169756'::bpchar)
Planning Time: 0.202 ms
Execution Time: 0.129 ms
(4 rows)


-- таблица с партициями
explain analyze
select * from ticket_flights_cp where ticket_no='0005432169756';
                                                             QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
Bitmap Heap Scan on ticket_flights_0 ticket_flights_cp  (cost=4.44..15.95 rows=3 width=32) (actual time=0.049..0.051 rows=0 loops=1)
  Recheck Cond: (ticket_no = '0005432169756'::bpchar)
  ->  Bitmap Index Scan on ticket_flights_0_pkey  (cost=0.00..4.44 rows=3 width=0) (actual time=0.043..0.044 rows=0 loops=1)
        Index Cond: (ticket_no = '0005432169756'::bpchar)
Planning Time: 0.356 ms
Execution Time: 0.086 ms
(6 rows)
```
Есть небольшой выигрыш в cost (с cost=0.42..16.47 изменилось до  cost=4.44..15.95), а также небольшой выигрыш во времени (с Execution Time: 0.129 ms изменилось до Execution Time: 0.086 ms)

5. Проверка на удаление и изменение.
```
select * from ticket_flights_cp where ticket_no='0005432211371';

   ticket_no   | flight_id | fare_conditions |  amount  
---------------+-----------+-----------------+----------
 0005432211371 |     30625 | Economy         | 14000.00


UPDATE ticket_flights_cp SET amount = 599 where ticket_no='0005432211371';
select * from ticket_flights_cp where ticket_no='0005432211371';
  ticket_no   | flight_id | fare_conditions | amount 
---------------+-----------+-----------------+--------
0005432211371 |     30625 | Economy         | 599.00
(1 row)

DELETE FROM ticket_flights_cp  where ticket_no='0005432211371';
select * from ticket_flights_cp where ticket_no='0005432211371';
ticket_no | flight_id | fare_conditions | amount 
-----------+-----------+-----------------+--------
(0 rows)
```
___
### Таблица bookings
1. Анализ содержания таблицы, для того чтобы понять, какие партиции нужно создавать. В данной таблице хранится информация за несколько месяцев. Создам партицию под каждый месяц.
```
select min(book_date) from bookings limit 10;
         min           
------------------------
2016-08-19 10:05:00+00
(1 строка)

select max(book_date) from bookings limit 10;
         max           
------------------------
2016-10-13 14:00:00+00
(1 строка)

```
2. Создание таблицы и партиции.
При создании таблицы возникла ошибка.
```
create table bookings_cp (like bookings including all) partition by range (book_date);
ERROR:  unique constraint on partitioned table must include all partitioning columns
ПОДРОБНОСТИ:  PRIMARY KEY constraint on table "bookings_cp" lacks column "book_date" which is part of the partition key.
```
Данную ошибку удалось обойти с помощью данной команды (копирование информации без создания индексов)
```
create table bookings_cp (like bookings) partition by range (book_date);
```
Создание партиций
```
create table bookings_2016_08 partition of bookings_cp for values from ('2016-08-01') to ('2016-09-01');
create table bookings_2016_09 partition of bookings_cp for values from ('2016-09-01') to ('2016-10-01');
create table bookings_2016_10 partition of bookings_cp for values from ('2016-10-01') to ('2016-11-01');
```
3. Копирование информации и проверка корректности распределения по партициям:
``` 
insert INTO bookings_cp select * from bookings;
 
select * from bookings_2016_08 limit 3;
book_ref |       book_date        | total_amount 
----------+------------------------+--------------
000511   | 2016-08-28 23:40:00+00 |     26700.00
0005E7   | 2016-08-31 05:25:00+00 |     28800.00
000A39   | 2016-08-28 23:29:00+00 |     23400.00

select * from bookings_2016_09 limit 3;
book_ref |       book_date        | total_amount 
----------+------------------------+--------------
00000F   | 2016-09-01 23:12:00+00 |    265700.00
000012   | 2016-09-11 05:02:00+00 |     37900.00
0002DB   | 2016-09-26 02:30:00+00 |    101500.00

select * from bookings_2016_10 limit 3;
book_ref |       book_date        | total_amount 
----------+------------------------+--------------
000068   | 2016-10-13 10:27:00+00 |     18100.00
000181   | 2016-10-08 09:28:00+00 |    131800.00
0002D8   | 2016-10-05 17:40:00+00 |     23600.00
```
4. Сравнение производительности запроса на таблице с партициями и без партиций
```
-- таблица с партициями
explain analyze                                                                        
select count(*) from bookings_cp where book_date between '2016-09-25' and '2016-10-05';
                                                                                QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
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


-- таблица без партиций
explain analyze
select count(*) from bookings where book_date between '2016-09-25' and '2016-10-05';
                                                                             QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
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
Цена запроса к таблице с партициями ощутимо меньше (cost=4852.13..4852.14). Цена запроса к таблице без партиций cost=5073.84..5073.85.

Теперь попробуем повесить индекс на поле book_date и сравнить производительность.
```
create index book_date_old ON bookings (book_date);

-- таблица с партициями
explain analyze                                                                        
select count(*) from bookings_cp where book_date between '2016-09-25' and '2016-10-05';
                                                                                       QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
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


-- таблица без партиций
explain analyze
select count(*) from bookings where book_date between '2016-09-25' and '2016-10-05';
                                                                         QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
Aggregate  (cost=1599.92..1599.93 rows=1 width=8) (actual time=4.877..4.878 rows=1 loops=1)
  ->  Index Only Scan using book_date_old on bookings  (cost=0.42..1462.20 rows=55089 width=0) (actual time=0.131..3.233 rows=55774 loops=1)
        Index Cond: ((book_date >= '2016-09-25 00:00:00+00'::timestamp with time zone) AND (book_date <= '2016-10-05 00:00:00+00'::timestamp with time zone))
        Heap Fetches: 0
Planning Time: 0.381 ms
Execution Time: 4.920 ms
(6 rows)
```
С добавлением индексов производительность таблицы с партициями падает (цена запроса к тадлице с партициями cost=1881.91..1881.92, цена запроса к таблице без партиций cost=1599.92..1599.93). Возможно, такой эффект связан с небольшим кол-вом записей в таблице, и при бОльшем количестве записей таблица с партициями начнет выиграывать в производительности. 
В документации сказано:`Всё это обычно полезно только для очень больших таблиц. Какие именно таблицы выиграют от секционирования, зависит от конкретного приложения, хотя, как правило, это следует применять для таблиц, размер которых превышает объём ОЗУ сервера.`

5. Проверка на удаление и изменение.
```
select * from bookings_cp where book_ref = '000068';
book_ref |       book_date        | total_amount 
----------+------------------------+--------------
000068   | 2016-10-13 10:27:00+00 |     18100.00
(1 строка)

update bookings_cp SET total_amount = '19100.0' where book_ref = '000068';
select * from bookings_cp where book_ref = '000068';
book_ref |       book_date        | total_amount 
----------+------------------------+--------------
000068   | 2016-10-13 10:27:00+00 |     19100.00
(1 строка)

delete from bookings_cp  where book_ref = '000068';
select * from bookings_cp where book_ref = '000068';
book_ref | book_date | total_amount 
----------+-----------+--------------
(0 строк)
```

