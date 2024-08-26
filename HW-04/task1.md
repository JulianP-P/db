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
