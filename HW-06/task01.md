### Задание 1
Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Настройка времени блокировки, по достижению которого, будут сделаны записи в логи.
```sql
alter system set deadlock_timeout = '200ms';
SELECT pg_reload_conf();
```
Ниже воссоздана ситуация, где первая сессия блокирует вторую.
```sql
--Дейстивя в сессии 1
BEGIN;
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

-- Действия в сессии 2
BEGIN;
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;

-- через 200мс коммитим транзакцию в первой сессии
commit;
-- коммит во второй сессии
commit;
```

```
-- смена значений у deadlock_timeout
2025-02-07 11:16:36.659 UTC [73] STATEMENT:  alter system set deadlock_timeout = 200ms;
2025-02-07 11:17:22.292 UTC [1] LOG:  received SIGHUP, reloading configuration files
2025-02-07 11:17:22.293 UTC [1] LOG:  parameter "deadlock_timeout" changed to "200ms"
-- первышение deadlock_timeout
2025-02-07 11:22:52.482 UTC [73] LOG:  process 73 still waiting for ShareLock on transaction 75017284 after 200.363 ms
2025-02-07 11:22:52.482 UTC [73] DETAIL:  Process holding the lock: 70. Wait queue: 73.
2025-02-07 11:22:52.482 UTC [73] CONTEXT:  while updating tuple (0,5) in relation "accounts"
2025-02-07 11:22:52.482 UTC [73] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2025-02-07 11:22:58.876 UTC [73] LOG:  process 73 acquired ShareLock on transaction 75017284 after 6593.800 ms
2025-02-07 11:22:58.876 UTC [73] CONTEXT:  while updating tuple (0,5) in relation "accounts"
2025-02-07 11:22:58.876 UTC [73] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
### Задание 2
Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

```sql
BEGIN;
SELECT pg_backend_pid();
BEGIN
 pg_backend_pid 
----------------
             70
(1 row)

locks=*# UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
```
```sql
postgres=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pid
FROM pg_locks where pid in (70, 73, 309) order by pid;
   locktype    | relation | virtxid |   xid    |       mode       | granted | pid 
---------------+----------+---------+----------+------------------+---------+-----
 relation      | 24680    |         |          | RowExclusiveLock | t       |  70
 transactionid |          |         | 75017289 | ExclusiveLock    | t       |  70
 relation      | 24675    |         |          | RowExclusiveLock | t       |  70
 virtualxid    |          | 5/15    |          | ExclusiveLock    | t       |  70
 virtualxid    |          | 4/32    |          | ExclusiveLock    | t       |  73
 transactionid |          |         | 75017290 | ExclusiveLock    | t       |  73
 relation      | 24680    |         |          | RowExclusiveLock | t       |  73
 tuple         | 24675    |         |          | ExclusiveLock    | t       |  73
 transactionid |          |         | 75017289 | ShareLock        | f       |  73
 relation      | 24675    |         |          | RowExclusiveLock | t       |  73
 transactionid |          |         | 75017291 | ExclusiveLock    | t       | 309
 relation      | 24675    |         |          | RowExclusiveLock | t       | 309
 virtualxid    |          | 6/3     |          | ExclusiveLock    | t       | 309
 tuple         | 24675    |         |          | ExclusiveLock    | f       | 309
 relation      | 24680    |         |          | RowExclusiveLock | t       | 309
(15 rows)
```
|   locktype    | relation | virtxid |   xid    |       mode       | granted | pid |
|---------------|----------|---------|----------|------------------|---------|-----|
| relation      | 24680    |         |          | RowExclusiveLock | t       |  70| 
| transactionid |          |         | 75017289 | ExclusiveLock    | t       |  70|
| relation      | 24675    |         |          | RowExclusiveLock | t       |  70|
| virtualxid    |          | 5/15    |          | ExclusiveLock    | t       |  70|

pid = 70 - первая сессия, где выполнен update
```
|   locktype    | relation | virtxid |   xid    |       mode       | granted | pid |
|---------------|----------|---------|----------|------------------|---------|-----|
| virtualxid    |          | 4/32    |          | ExclusiveLock    | t       |  73|
| transactionid |          |         | 75017290 | ExclusiveLock    | t       |  73|
| relation      | 24680    |         |          | RowExclusiveLock | t       |  73|
| tuple         | 24675    |         |          | ExclusiveLock    | t       |  73|
| transactionid |          |         | 75017289 | ShareLock        | f       |  73|
| relation      | 24675    |         |          | RowExclusiveLock | t       |  73|

```
|   locktype    | relation | virtxid |   xid    |       mode       | granted | pid |
|---------------|----------|---------|----------|------------------|---------|-----|
| transactionid |          |         | 75017291 | ExclusiveLock    | t       | 309|
| relation      | 24675    |         |          | RowExclusiveLock | t       | 309|
| virtualxid    |          | 6/3     |          | ExclusiveLock    | t       | 309|
| tuple         | 24675    |         |          | ExclusiveLock    | f       | 309|
| relation      | 24680    |         |          | RowExclusiveLock | t       | 309|

### Задание 3
Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?


### Задание 4
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
