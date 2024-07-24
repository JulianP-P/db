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
