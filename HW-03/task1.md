```sql
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

```sql
postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
postgres=# \q
```
```sql
postgres=# select * from test;
 c1 
----
 1
(1 row)
```

```
mount | grep sdb
/dev/sdb1 on /mnt/data type ext4 (rw,relatime)
```

```
projs@projs-VirtualBox:/mnt$ ls -alh
итого 12K
drwxr-xr-x  3 root     root     4,0K авг 18 13:51 .
drwxr-xr-x 23 root     root     4,0K авг  4 20:02 ..
drwxr-xr-x  3 postgres postgres 4,0K авг 18 13:34 data

```

```sql
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
16  main    5432 down   <unknown> /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

```

```
2024-08-18 15:08:53.274 MSK [1112] LOG:  could not open file "postmaster.pid": Нет такого файла или каталога
2024-08-18 15:08:53.276 MSK [1112] LOG:  performing immediate shutdown because data directory lock file is invalid
2024-08-18 15:08:53.276 MSK [1112] LOG:  received immediate shutdown request
2024-08-18 15:08:53.276 MSK [1112] LOG:  could not open file "postmaster.pid": Нет такого файла или каталога
2024-08-18 15:08:53.283 MSK [1112] LOG:  database system is shut down
```

```
/etc/postgresql/16/main$ vim postgresql.conf
data_directory = '/mnt/data/16/main/'           # use data in another directory
```

```
sudo systemctl start postgresql@16-main.service
```

```sql
postgres=# select * from test;
 c1 
----
 1
(1 row)
```
```
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory     Log file
16  main    5432 online postgres /mnt/data/16/main/ /var/log/postgresql/postgresql-16-main.log
```
