### otus-PostgreSQL-2024-05-Прощина Юлия

Домашняя работа выполнялась в виртуальной манише, версия postgre 16.3.


1) Проверка, что кластер запущен
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
2) Создание таблицы
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
3) Остановка кластера
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
4) Создание диска и монитрование его
```
mount | grep sdb
/dev/sdb1 on /mnt/data type ext4 (rw,relatime)
```
5) Смена владельца директории /mnt/data/ и перенос в нее сожержимого /var/lib/postgresql/16/mnt/data
```console
projs@projs-VirtualBox:/mnt$ ls -alh
итого 12K
drwxr-xr-x  3 root     root     4,0K авг 18 13:51 .
drwxr-xr-x 23 root     root     4,0K авг  4 20:02 ..
drwxr-xr-x  3 postgres postgres 4,0K авг 18 13:34 data
```
6) Перавая попытка запустить кластер
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner     Data directory              Log file
16  main    5432 down   <unknown> /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```
Содержание лога
```
2024-08-18 15:08:53.274 MSK [1112] LOG:  could not open file "postmaster.pid": Нет такого файла или каталога
2024-08-18 15:08:53.276 MSK [1112] LOG:  performing immediate shutdown because data directory lock file is invalid
2024-08-18 15:08:53.276 MSK [1112] LOG:  received immediate shutdown request
2024-08-18 15:08:53.276 MSK [1112] LOG:  could not open file "postmaster.pid": Нет такого файла или каталога
2024-08-18 15:08:53.283 MSK [1112] LOG:  database system is shut down
```
Для запуска бд необходимо скорректировать файл с конфигами postgresql.conf
```
/etc/postgresql/16/main$ vim postgresql.conf
data_directory = '/mnt/data/16/main/'        
```
7) Перезапуск бд
```
sudo systemctl start postgresql@16-main.service
```
8) Проверка запуска кластера
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory     Log file
16  main    5432 online postgres /mnt/data/16/main/ /var/log/postgresql/postgresql-16-main.log
```
9) Проверка данных
```sql
postgres=# select * from test;
 c1 
----
 1
(1 row)
```


