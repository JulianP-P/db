### otus-PostgreSQL-2024-05-Прощина Юлия

поставить на нем Docker Engine
сделать каталог /var/lib/postgres
развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
развернуть контейнер с клиентом postgres
подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк
подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера
удалить контейнер с сервером
создать его заново
подключится снова из контейнера с клиентом к контейнеру с сервером
проверить, что данные остались на месте
оставляйте в ЛК ДЗ комментарии что и как вы делали и как боролись с проблемами

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
4. 
-- 3. Запускаем отдельный контейнер с клиентом в общей сети с БД: 
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres

CREATE DATABASE otus; 

-- 4. Проверяем, что подключились через отдельный контейнер:
sudo docker ps -a

sudo docker stop 5d4d5a5dbc37

sudo docker rm 5d4d5a5dbc37
