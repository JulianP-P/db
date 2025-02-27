Для выполнения задач использовалась таблица film из данного репозитория
https://github.com/jOOQ/sakila/tree/main
### Виды индексов. Работа с индексами и оптимизация запросов 
#### 1. Реализовать индекс для полнотекстового поиска
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

Реализовать индекс на часть таблицы или индекс
на поле с функцией
Создать индекс на несколько полей
Написать комментарии к каждому из индексов
Описать что и как делали и с какими проблемами
столкнулись
 
 
 
 
 
 update film set release_year = 2007 where film_id in (1,2,3,4,5,6,7,8,9,10);

 SELECT release_year, count(*) AS count
    FROM film
    GROUP BY release_year;
         2007 |    10
         2006 |   990


select * from pg_indexes
where tablename = 'film' \gx
schemaname | public
tablename  | film
indexname  | film_pkey
tablespace | 
indexdef   | CREATE UNIQUE INDEX film_pkey ON public.film USING btree (film_id)
-----------+-------------------------------------------------------------------------------------------
schemaname | public
tablename  | film
indexname  | film_fulltext_idx
tablespace | 
indexdef   | CREATE INDEX film_fulltext_idx ON public.film USING gist (fulltext)
-----------+-------------------------------------------------------------------------------------------
schemaname | public
tablename  | film
indexname  | idx_fk_language_id
tablespace | 
indexdef   | CREATE INDEX idx_fk_language_id ON public.film USING btree (language_id)
-----------+-------------------------------------------------------------------------------------------
schemaname | public
tablename  | film
indexname  | idx_fk_original_language_id
tablespace | 
indexdef   | CREATE INDEX idx_fk_original_language_id ON public.film USING btree (original_language_id)
-----------+-------------------------------------------------------------------------------------------
schemaname | public
tablename  | film
indexname  | idx_title
tablespace | 
indexdef   | CREATE INDEX idx_title ON public.film USING btree (title)











____
explain SELECT * FROM film where release_year = 2007;
 Seq Scan on film  (cost=0.00..74.50 rows=1 width=328)
   Filter: ((release_year)::integer = 2007)

create index idx_release_year on film(release_year) where release_year=2007;
CREATE INDEX
postgres=# explain SELECT * FROM film where release_year = 2007;
 Index Scan using idx_release_year on film  (cost=0.14..4.15 rows=1 width=328)

____
explain select * from film where rating = 'PG' and length > 90; 
 Seq Scan on film  (cost=0.00..77.00 rows=131 width=328)
   Filter: ((length > 90) AND (rating = 'PG'::mpaa_rating))

postgres=# select count(*) from film where rating='PG' and length > 90; 
   131

postgres=# create index idx_rating_length on film(rating, length);
CREATE INDEX
postgres=# explain select * from film where rating = 'PG' and length > 90; 
 Bitmap Heap Scan on film  (cost=5.62..69.58 rows=131 width=328)
   Recheck Cond: ((rating = 'PG'::mpaa_rating) AND (length > 90))
   ->  Bitmap Index Scan on idx_rating_length  (cost=0.00..5.59 rows=131 width=0)
         Index Cond: ((rating = 'PG'::mpaa_rating) AND (length > 90))
