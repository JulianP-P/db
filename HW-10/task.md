
Размер таблиц

```
\d
 Схема   |          Имя          |               Тип               | Владелец |  Хранение  | Метод доступа |   Размер   |     Описание      
----------+-----------------------+---------------------------------+----------+------------+---------------+------------+-------------------
 bookings | aircrafts             | таблица                         | postgres | постоянное | heap          | 16 kB      | Самолеты
 bookings | airports              | таблица                         | postgres | постоянное | heap          | 48 kB      | Аэропорты
 bookings | boarding_passes       | таблица                         | postgres | постоянное | heap          | 33 MB      | Посадочные талоны
 bookings | bookings              | таблица                         | postgres | постоянное | heap          | 13 MB      | Бронирования
 bookings | flights               | таблица                         | postgres | постоянное | heap          | 3168 kB    | Рейсы
 bookings | flights_flight_id_seq | последовательность              | postgres | постоянное |               | 8192 bytes | 
 bookings | flights_v             | представление                   | postgres | постоянное |               | 0 bytes    | Рейсы
 bookings | routes                | материализованное представление | postgres | постоянное | heap          | 152 kB     | Маршруты
 bookings | seats                 | таблица                         | postgres | постоянное | heap          | 96 kB      | Места
 bookings | ticket_flights        | таблица                         | postgres | постоянное | heap          | 68 MB      | Перелеты
 bookings | tickets               | таблица                         | postgres | постоянное | heap          | 48 MB      | Билеты
(11 строк)
```
