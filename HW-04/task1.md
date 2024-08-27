создайте новый кластер PostgresSQL 14
зайдите в созданный кластер под пользователем postgres
создайте новую базу данных testdb
зайдите в созданную базу данных под пользователем postgres
создайте новую схему testnm
создайте новую таблицу t1 с одной колонкой c1 типа integer
вставьте строку со значением c1=1
создайте новую роль readonly
дайте новой роли право на подключение к базе данных testdb
дайте новой роли право на использование схемы testnm
дайте новой роли право на select для всех таблиц схемы testnm
создайте пользователя testread с паролем test123
дайте роль readonly пользователю testread
зайдите под пользователем testread в базу данных testdb
сделайте select * from t1;
получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)
напишите что именно произошло в тексте домашнего задания
у вас есть идеи почему? ведь права то дали?
посмотрите на список таблиц
подсказка в шпаргалке под пунктом 20
а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)
вернитесь в базу данных testdb под пользователем postgres
удалите таблицу t1
создайте ее заново но уже с явным указанием имени схемы testnm
вставьте строку со значением c1=1
зайдите под пользователем testread в базу данных testdb
сделайте select * from testnm.t1;
получилось?
есть идеи почему? если нет - смотрите шпаргалку
как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
сделайте select * from testnm.t1;
получилось?
есть идеи почему? если нет - смотрите шпаргалку
сделайте select * from testnm.t1;
получилось?
ура!
теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);
а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?
есть идеи как убрать эти права? если нет - смотрите шпаргалку
если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды
теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
расскажите что получилось и почему


### otus-PostgreSQL-2024-05-Прощина Юлия
#### ДЗ Работа с базами данных, пользователями и правами
Домашняя работа выполнялась в виртуальной манише, версия postgre 16.3.

1) Создание бд testdb под пользователем postgres
```
postgres=# select datname from pg_database;
  datname  
-----------
 postgres
 template1
 template0
(3 rows)

postgres=# create database testdb;
CREATE DATABASE
postgres=# select datname from pg_database;
  datname  
-----------
 postgres
 template1
 template0
 testdb
(4 rows)
```
2) Создание схемы testnm, таблицы t1 и заполнение ее
```
postgres=# \c
You are now connected to database "postgres" as user "postgres".
postgres=# \c testdb 
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm;
CREATE SCHEMA
testdb=# create table testnm.t1(c1 integer);
CREATE TABLE
testdb=# insert into testnm.t1 values (1);
INSERT 0 1
testdb=# select * from testnm.t1;
 c1 
----
  1
(1 row)
```
3) Создание роли readonly
```
testdb=# create role readonly;
CREATE ROLE
testdb=# grant connect on database testdb to readonly;
GRANT
testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```
4) Создание user testread. Так как все задания я выполняла сама, без опоры на шпаргалку, то в этом пункте я совершила ошибку.
```
testdb=# create user testread password 'test123' role readonly;
CREATE ROLE
```
На этом моменте возникла ошибка
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for schema testnm
LINE 1: select * from testnm.t1;
```
Оказалось, что правильно создавать роль следующим образом
```
testdb=# create user testread password 'test123' in role readonly;
CREATE ROLE
```
5) Нельзя создать таблицу ни в схеме public, ни в схеме testnm. Как я понимаю, это связано с версией postgres -  16.3. В этой версии убрана автоматическая выдача прав CREATE TABLE в схеме public.
```
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
testdb=> create table testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
```
