
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
