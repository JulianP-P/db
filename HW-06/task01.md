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
