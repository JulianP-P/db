### otus-PostgreSQL-2024-05-Прощина Юлия

1. Создание /var/lib/postgres
```bash
mkdir /var/lib/postgres
```
2. Создание docker-сети
```sql
sudo docker network create pg-net
```
3. Создание контейнера sql-server
```bash
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
4. Создание контейнера с клиентом sql
```bash
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
5. Создание и заполнение бд внутри контейнера pg-client
```sql
CREATE DATABASE otus; 
\c otus
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
Проверка данных
```sql
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
6. Подключение к бд с ноутбука

7. Удаление контейнеров
```bash
sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                        PORTS                                       NAMES
0043f6de19d3   postgres:15   "docker-entrypoint.s…"   10 minutes ago   Up 10 minutes                 5432/tcp                                    pg-client
9440a1a232ae   postgres:15   "docker-entrypoint.s…"   12 minutes ago   Up 12 minutes                 0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

sudo docker stop 9440a1a232ae
sudo docker rm 9440a1a232ae
```
8. Создание контейнеров заново
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15


sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
9. Проверка, что данные на месте
```sql
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

Проблема возникла только на этапе создания контейнера, порт 5432 был занят, т.к. после выполнения предыдущей дз я не удалила контейнер. Проблема решилась удалением этого контейнера.
```
projs@localhost:~/db/hw2$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
efc2b5ad9eec: Pull complete 
5a5182daaaef: Pull complete 
9bd58f90d997: Pull complete 
2ef118f9f903: Pull complete 
cc5a81cd1ffc: Pull complete 
e92d4f57ba22: Pull complete 
fc2c1d24e9ca: Pull complete 
c77b1b16c09f: Pull complete 
1c1d8adbc955: Pull complete 
9154ac3ec50f: Pull complete 
24dc5078fe3b: Pull complete 
68acc6218bcf: Pull complete 
532895c2736a: Pull complete 
a08535c970e7: Pull complete 
Digest: sha256:c317f08d888548ba8d43d69608c075d4def55478c46e491c6dab2dbb862346a2
Status: Downloaded newer image for postgres:15
1e11406fb1e5c28d3dc6feb374f80626094475fde7651f718044cc2647de29af
docker: Error response from daemon: driver failed programming external connectivity on endpoint pg-server (5dd143d45f817ed6e867bcf465848482386ad9c38976026189d1a16f7ce9f17d): Bind for 0.0.0.0:5432 failed: port is already allocated.

sudo docker rm 1e11406fb1e5c28d3dc6feb374f80626094475fde7651f718044cc2647de29af
```
