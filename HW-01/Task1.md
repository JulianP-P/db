### Прощина Юлия
### otus-PostgreSQL-2024-05

Домашняя работа выполнялась в docker-контейнере, версия postgre 16.3.

### 1. Создание бд
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```
### 2. Просмотр уровня изоляции
```sql
show transaction isolation level;
```
Результат команды:
```
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

### 3. Запуск транзакции с уровнем изоляции transaction_isolation
Команда в первой сессии:
```sql
insert into persons(first_name, second_name) values('sergey', 'sergeev');
```
Команда во второй сессии:
```sql
select * from persons;
```
Результат команды во второй сессии:
```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
Результат выполнения команды в сессии не виден во второй сессии, т.к. при уровне изоляции transaction_isolation не видны не зафиксированные транзакции.

Завершаю транзакцию в первой сессии:
```sql
commit;
```
Команда во второй сессии:
```sql
select * from persons;
```
Результат работы команды:
```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
