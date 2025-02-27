otus-PostgreSQL-2024-11-Прощина Юлия

Домашняя работа выполнялась в докер контейнере, версия postgres 14.15.

Для выполнения задач использовалась таблица film из данного репозитория

https://github.com/jOOQ/sakila/tree/main

### Виды индексов. Работа с индексами и оптимизация запросов 
#### 1. Создать индекс к какой-либо из таблиц вашей БД. Прислать текстом результат команды explain, в которой используется данный индекс
Поиск фильмов с продолжительностью 60 минут.
```sql
 explain select * from film where length = 60;
 Seq Scan on film  (cost=0.00..74.50 rows=8 width=440)
   Filter: (length = 60)
```
Создание индекса
```sql
create index idx_lenth on film(length);
```
Поиск фильмов после создание индекса.
```sql
explain select * from film where length = 60;
 Bitmap Heap Scan on film  (cost=4.34..27.82 rows=8 width=440)
   Recheck Cond: (length = 60)
   ->  Bitmap Index Scan on idx_lenth  (cost=0.00..4.33 rows=8 width=0)
         Index Cond: (length = 60)
```

#### 2. Реализовать индекс для полнотекстового поиска
Поиск будем производить по полю description. Будем искать фильмы с собаками. 
Изначальный тип поля - text.

Пробуем "наивный" способ поиска: 
```sql
EXPLAIN SELECT * FROM film
    WHERE description like '%Dog%' \gx
QUERY PLAN | Seq Scan on film  (cost=0.00..68.72 rows=41 width=390)
-----------+-------------------------------------------------------
QUERY PLAN |   Filter: (description ~~ '%Dog%'::text)
```
Изменю изначальный тип поля на TSVECTOR для полнотекстового поиска:
```sql
alter table film
alter column description type TSVECTOR
 USING description::tsvector;
```
Создание индекса
```sql
create index idx_description on film using gin(description);
```
Поиск после создания индекса
```sql
 explain SELECT * FROM film
    WHERE description @@ 'Dog';
 Bitmap Heap Scan on film  (cost=8.04..23.84 rows=5 width=328)
   Recheck Cond: (description @@ '''Dog'''::tsquery)
   ->  Bitmap Index Scan on idx_description  (cost=0.00..8.04 rows=5 width=0)
         Index Cond: (description @@ '''Dog'''::tsquery)
```
Цена запроса изменилась с cost=0.00..68.72 до cost=8.04..23.84.

#### 3. Реализовать индекс на часть таблицы
В изначальной таблице все фильмы были 2006 года. Изменю данные, чтобы было несколько значений с 2007 годом. Тогда будет смысл в создании индекса на часть полей.
```sql
update film
    set release_year = 2007
    where film_id in (1,2,3,4,5,6,7,8,9,10);
```
Проверим количество полей с разным значением
```sql
 SELECT release_year, count(*) AS count
    FROM film
    GROUP BY release_year;
         2007 |    10
         2006 |   990
```
Прогоним запрос до создания индекса
```sql
explain SELECT * FROM film where release_year = 2007;
 Seq Scan on film  (cost=0.00..74.50 rows=1 width=328)
   Filter: ((release_year)::integer = 2007)
```
Создание индекса
```sql
create index idx_release_year on film(release_year) where release_year=2007;
```
Запрос после создание индекса
```sql
explain SELECT * FROM film where release_year = 2007;
 Index Scan using idx_release_year on film  (cost=0.14..4.15 rows=1 width=328)
```
Цена запроса изменилась с cost=0.00..74.50 до cost=0.14..4.15.

#### 4. Создать индекс на несколько полей
Запрос до создания индекса
```sql
explain select * from film where rating = 'PG' and length > 90; 
 Seq Scan on film  (cost=0.00..77.00 rows=131 width=328)
   Filter: ((length > 90) AND (rating = 'PG'::mpaa_rating))
```
Создание индекса. Так как по полю rating будет проверка по равенству, то это поле идет первым.
```sql
create index idx_rating_length on film(rating, length);
```
```
explain select * from film where rating = 'PG' and length > 90; 
 Bitmap Heap Scan on film  (cost=5.62..69.58 rows=131 width=328)
   Recheck Cond: ((rating = 'PG'::mpaa_rating) AND (length > 90))
   ->  Bitmap Index Scan on idx_rating_length  (cost=0.00..5.59 rows=131 width=0)
         Index Cond: ((rating = 'PG'::mpaa_rating) AND (length > 90))
```
Цена запроса изменилась с cost=0.00..77.00 до cost=5.62..69.58.

Такая маленькая разница скорее всего связана с небольшой кардинальностью.
```sql
SELECT rating, count(*) AS count
    FROM film
    GROUP BY rating;
 NC-17  |   210
 PG-13  |   223
 PG     |   194
 R      |   195
 G      |   178
```



