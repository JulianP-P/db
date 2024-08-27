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

```
postgres=# \c
You are now connected to database "postgres" as user "postgres".
postgres=# \c testdb 
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm
testdb-# ;
CREATE SCHEMA
testdb=# create table testnm.t1(c1 integer);
CREATE TABLE
testdb=# insert into testnm.t1 values (1)
testdb-# ;
INSERT 0 1
testdb=# select * from testnm.t1 
testdb-# ;
 c1 
----
  1
(1 row)

```
```
testdb=# create role readonly;
CREATE ROLE
testdb=# grant connect on database testdb to readonly;
GRANT
testdb=# grant usage on schema testnm to readonly;
GRANT
testdb=# grant select on all tables in schema testnm to readonly;
GRANT

testdb=# create user testread password 'test123' role readonly;
CREATE ROLE
```
На этом моменте возникла ошибка
```
testdb=> select * from testnm.t1;
ERROR:  permission denied for schema testnm
LINE 1: select * from testnm.t1;
```
Оказалосб, что правильно создавать роль следующим образом
```
testdb=# create user testread password 'test123' in role readonly;
CREATE ROLE
```
```
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
testdb=> create table testnm.t2(c1 integer);
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
```
