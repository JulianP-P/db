

остановите postgres например через sudo -u postgres pg_ctlcluster 15 main stop
создайте новый диск к ВМ размером 10GB
добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)


сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/


перенесите содержимое /var/lib/postgres/15 в /mnt/data - mv /var/lib/postgresql/15/mnt/data
попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
напишите получилось или нет и почему
задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
напишите что и почему поменяли
попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 15 main start
напишите получилось или нет и почему
зайдите через через psql и проверьте содержимое ранее созданной таблицы
задание со звездочкой *: не удаляя существующий инстанс ВМ сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
### otus-PostgreSQL-2024-05-Прощина Юлия

Домашняя работа выполнялась в docker-контейнере, версия postgre 16.3.


1) Проверка, что кластер запущен
```sql
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
```sql
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
```
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
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory     Log file
16  main    5432 online postgres /mnt/data/16/main/ /var/log/postgresql/postgresql-16-main.log
```
