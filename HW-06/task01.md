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
locks=# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted, pid
FROM pg_locks order by pid;
   locktype    |   relation    | virtxid | xid |       mode       | granted | pid 
---------------+---------------+---------+-----+------------------+---------+-----
 relation      | accounts_pkey |         |     | RowExclusiveLock | t       |  40
 transactionid |               |         | 741 | ExclusiveLock    | t       |  40
 virtualxid    |               | 3/13    |     | ExclusiveLock    | t       |  40
 relation      | accounts      |         |     | RowExclusiveLock | t       |  40
 transactionid |               |         | 742 | ExclusiveLock    | t       | 148
 relation      | accounts_pkey |         |     | RowExclusiveLock | t       | 148
 relation      | accounts      |         |     | RowExclusiveLock | t       | 148
 virtualxid    |               | 4/223   |     | ExclusiveLock    | t       | 148
 tuple         | accounts      |         |     | ExclusiveLock    | t       | 148
 transactionid |               |         | 741 | ShareLock        | f       | 148
 transactionid |               |         | 743 | ExclusiveLock    | t       | 168
 tuple         | accounts      |         |     | ExclusiveLock    | f       | 168
 virtualxid    |               | 5/41    |     | ExclusiveLock    | t       | 168
 relation      | accounts      |         |     | RowExclusiveLock | t       | 168
 relation      | accounts_pkey |         |     | RowExclusiveLock | t       | 168
```

```
   locktype    |   relation    | virtxid | xid |       mode       | granted | pid 
---------------+---------------+---------+-----+------------------+---------+-----
 relation      | accounts_pkey |         |     | RowExclusiveLock | t       |  40
 transactionid |               |         | 741 | ExclusiveLock    | t       |  40
 virtualxid    |               | 3/13    |     | ExclusiveLock    | t       |  40
 relation      | accounts      |         |     | RowExclusiveLock | t       |  40
```
pid = 40 - первая сессия, где выполнен update.
- Блокировка ExclusiveLock типа transactionid - блокировка настоящего номера транзакции, который появляется, как только транзакция начинает менять данные.
- Блокировка ExclusiveLock типа virtualxid - блокировка виртуального номера транзакции, котораый есть у любой транзакции.
- Блокировка RowExclusiveLock в relation accounts_pkey - блокировка индекса для первичного ключа, которая возникает при команде UPDATE.
- Блокировка RowExclusiveLock в relation accounts - блокировка таблицы accounts, которая возникает при команде UPDATE.
Все блокировки были получены (granted=t), так как этот UPDATE выполнялся первым, других блокировк не было.
```
   locktype    |   relation    | virtxid | xid |       mode       | granted | pid 
---------------+---------------+---------+-----+------------------+---------+-----
 transactionid |               |         | 742 | ExclusiveLock    | t       | 148
 relation      | accounts_pkey |         |     | RowExclusiveLock | t       | 148
 relation      | accounts      |         |     | RowExclusiveLock | t       | 148
 virtualxid    |               | 4/223   |     | ExclusiveLock    | t       | 148
 tuple         | accounts      |         |     | ExclusiveLock    | t       | 148
 transactionid |               |         | 741 | ShareLock        | f       | 148
```
pid = 148 - вторая сессия, где выполнен update.
- Блокировка ExclusiveLock типа transactionid - блокировка настоящего номера транзакции, который появляется, как только транзакция начинает менять данные.
- Блокировка ExclusiveLock типа virtualxid - блокировка виртуального номера транзакции, котораый есть у любой транзакции.
- Блокировка RowExclusiveLock в relation accounts_pkey - блокировка индекса для первичного ключа, которая возникает при команде UPDATE.
- Блокировка RowExclusiveLock в relation accounts - блокировка таблицы accounts, которая возникает при команде UPDATE.
Все блокировки были получены (granted=t), так как этот UPDATE выполнялся первым, других блокировк не было.

### Задание 3
Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?


### Задание 4
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
