### Прощина Юлия
### otus-PostgreSQL-2024-05

Домашняя работа выполнялась в docker-контейнере, версия postgre 16.3.

### 1. Создание бд
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
```
