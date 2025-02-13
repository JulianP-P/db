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

Используемый запрос во всех трех сеансах:
```sql
BEGIN;
SELECT pg_backend_pid();
UPDATE accounts SET amount = amount + 100 WHERE acc_no = 1;
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
Ниже рассмотрены блокировки, относящиеся к каждой отдельной транзакции.
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
- Блокировка ExclusiveLock типа transactionid (742) - блокировка настоящего номера транзакции, который появляется, как только транзакция начинает менять данные.
- Блокировка ExclusiveLock типа virtualxid - блокировка виртуального номера транзакции, котораый есть у любой транзакции.
- Блокировка RowExclusiveLock в relation accounts_pkey - блокировка индекса для первичного ключа, которая возникает при команде UPDATE. Этот же объект блокируется первой транзакцией, но уровень блокировки (RowExclusiveLock) позволяет накладывать несколько блокировок такого же уровня.
- Блокировка RowExclusiveLock в relation accounts - блокировка таблицы accounts, которая возникает при команде UPDATE. Тут такая же ситуация, как и с RowExclusiveLock accounts_pkey. Данный объект заблокирован первой транзакцией, но уровень блокировки это допускает.
- Блокировка ExclusiveLock версии строки (tuple) захватывается перед изменением строки и отпускается после внесения изменений в информационные биты. Такая же блокировка захватывалась в первой транзакции, но после внесения изменений эта блокировка была отпущена, поэтому ее не видно в pg_locks.
- После захвата блокировки tuple проверяется заблокирована ли данная строка по информационным битам. Если строка заблокирована, то номер блокирующей транзакции берется из xmax. Чтобы поймать, когда блокирующая транзакция отработает, запрошена блокировка номера блокирующей транзакции (в данном случае это 741). Это и есть блокировка ShareLock transactionid 741. Данная блокировка не была получена (granted=f).
```
   locktype    |   relation    | virtxid | xid |       mode       | granted | pid 
---------------+---------------+---------+-----+------------------+---------+-----
 transactionid |               |         | 743 | ExclusiveLock    | t       | 168
 tuple         | accounts      |         |     | ExclusiveLock    | f       | 168
 virtualxid    |               | 5/41    |     | ExclusiveLock    | t       | 168
 relation      | accounts      |         |     | RowExclusiveLock | t       | 168
 relation      | accounts_pkey |         |     | RowExclusiveLock | t       | 168
```
- Блокировки transactionid, virtualxid, relation - accounts, relation - accounts_pkey аналогичны прошлым транзакциям.
- Блокировка tuple - accounts уровня ExclusiveLock возникает по той же причине, что и во второй транзакции, но в данном случае завхватить эту блокировку уже нельзя (granted=f), так как уровень ExclusiveLock не допускает такого.

### Задание 3
Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Используемые запросы:
```sql
-- 1
BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
--
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

--2
BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 2;
--
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;

--3
BEGIN;
SELECT txid_current(), pg_backend_pid();
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 3;
--
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
```
Логи:
```
2025-02-12 12:28:29.429 UTC [35] LOG:  process 35 still waiting for ShareLock on transaction 772 after 1000.055 ms
2025-02-12 12:28:29.429 UTC [35] DETAIL:  Process holding the lock: 162. Wait queue: 35.
2025-02-12 12:28:29.429 UTC [35] CONTEXT:  while updating tuple (0,18) in relation "accounts"
2025-02-12 12:28:29.429 UTC [35] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;

2025-02-12 12:28:37.472 UTC [162] LOG:  process 162 still waiting for ShareLock on transaction 773 after 1000.370 ms
2025-02-12 12:28:37.472 UTC [162] DETAIL:  Process holding the lock: 164. Wait queue: 162.
2025-02-12 12:28:37.472 UTC [162] CONTEXT:  while updating tuple (0,3) in relation "accounts"
2025-02-12 12:28:37.472 UTC [162] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;

2025-02-12 12:28:44.036 UTC [164] LOG:  process 164 detected deadlock while waiting for ShareLock on transaction 771 after 1000.398 ms
2025-02-12 12:28:44.036 UTC [164] DETAIL:  Process holding the lock: 35. Wait queue: .
2025-02-12 12:28:44.036 UTC [164] CONTEXT:  while updating tuple (0,16) in relation "accounts"
2025-02-12 12:28:44.036 UTC [164] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
2025-02-12 12:28:44.036 UTC [164] ERROR:  deadlock detected
2025-02-12 12:28:44.036 UTC [164] DETAIL:  Process 164 waits for ShareLock on transaction 771; blocked by process 35.
        Process 35 waits for ShareLock on transaction 772; blocked by process 162.
        Process 162 waits for ShareLock on transaction 773; blocked by process 164.
        Process 164: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
        Process 35: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
        Process 162: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
2025-02-12 12:28:44.036 UTC [164] HINT:  See server log for query details.
2025-02-12 12:28:44.036 UTC [164] CONTEXT:  while updating tuple (0,16) in relation "accounts"
2025-02-12 12:28:44.036 UTC [164] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;

2025-02-12 12:28:44.036 UTC [162] LOG:  process 162 acquired ShareLock on transaction 773 after 7564.922 ms
2025-02-12 12:28:44.036 UTC [162] CONTEXT:  while updating tuple (0,3) in relation "accounts"
2025-02-12 12:28:44.036 UTC [162] STATEMENT:  UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 3;
```
По логам вполне можно разобраться, что deadlock создали три транзакции, но сами блокирующие операции не приводятся. 

В первом блоке помечается, что процесс 35 заблокирован транзакцией 772 и процессом 162. Процесс 162 заблокирован траннзакцией 773 и процессом 164. Процесс 164 заблокирован транцакцией 771 и процессом 35. На этом моменте обнаружен deadlock, поэтому транзакция 773 (процесс 164) фейлится и килляется. 

При этом в логах выведены только последние операции UPDATE, глядя на них, нельзя понять, почему произошла блокировка. В логах пишется `See server log for query details.`, но мне не удалось найти никаких доп файлов, где писалась бы более подробная информация.
### Задание 4
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Так как UPDATE - это не атомарная операция, то она блокирует строки по мере их обновления. Это происходит не мгновенно. Поэтому, если одна команда обновляет строки в одном порядке, а другая — в другом, они могут заблокировать друг друга.
