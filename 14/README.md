# Домашняя работа на тему "Секционирование"

Для запуска локальной БД использую Docker:

```yaml
volumes:
  postgres-data:
    driver: local

services:
  pg1:
    image: postgres:17
    container_name: pg1
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    volumes:
      - postgres-data:/var/lib/postgresql
    ports:
      - "5432:5432"
```

Разворачиваю базу данных из бэкапа:

```bash
wget https://edu.postgrespro.ru/demo_medium.zip
unzip demo_medium.zip
psql -h localhost -p 5432 -U postgres -d postgres -f demo_medium.sql -c 'alter database demo set search_path to bookings'
```

Находим самую большую таблицу:

```sql
SELECT table_name,
       pg_size_pretty(pg_total_relation_size('"' || table_schema || '"."' || table_name || '"')) AS size
FROM information_schema.tables
WHERE table_schema = 'bookings';
```

```
boarding_passes,264 MB
ticket_flights,246 MB
tickets,134 MB
bookings,43 MB
flights,10192 kB
seats,144 kB
airports,64 kB
aircrafts,32 kB
flights_v,0 bytes
```

Изучаю запрос, которые получает все билеты определенного рейса:

```sql
explain analyse
select *
from ticket_flights
where flight_id = 38986;

-- Результат explain
Gather
    (cost=1000.00..33021.71 rows =83 width=32)
    (actual time =100.509..145.915 rows =48 loops=1)
    Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on ticket_flights  (cost=0.00..32013.41 rows=35 width=32) (actual time=87.701..130.221 rows=16 loops=3)
        Filter: (flight_id = 38986)
        Rows Removed by Filter: 786762
Planning Time: 0.234 ms
Execution Time: 145.973 ms
```

Создает пустую таблицу для секционирования с полями аналогичными ticket_flights:

```sql
create table ticket_flights_pt
(
    like ticket_flights including all
) PARTITION BY HASH (flight_id);
```

Создаем таблицы для секционирования:

```sql
CREATE TABLE ticket_flights_0 PARTITION OF ticket_flights_pt FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE ticket_flights_1 PARTITION OF ticket_flights_pt FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE ticket_flights_2 PARTITION OF ticket_flights_pt FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE ticket_flights_3 PARTITION OF ticket_flights_pt FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Скопируюем данные из ticket_flights в ticket_flights_pt:

```sql
INSERT INTO ticket_flights_pt
SELECT *
FROM ticket_flights;
-- 2,360,335 rows affected in 9 s 501 ms
```

Переименовываем ticket_flights в ticket_flights_old:

```sql
alter table ticket_flights
    rename to ticket_flights_bak;
alter table ticket_flights_pt
    rename to ticket_flights;
```

Повторяем запрос:

```sql
explain analyse
select *
from ticket_flights
where flight_id = 38986;

-- Результат explain
Gather
    (cost=1000.00..9132.92 rows =63 width=32)
    (actual time =18.430..34.698 rows =48 loops=1)
    Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on ticket_flights_2 ticket_flights  (cost=0.00..8126.62 rows=26 width=32) (actual time=13.121..25.884 rows=16 loops=3)
        Filter: (flight_id = 38986)
        Rows Removed by Filter: 200024
Planning Time: 0.654 ms
Execution Time: 34.731 ms
```

## Выводы:

- Секционирование позволяет ускорить запросы, так как уменьшается количество данных, которые нужно просматривать.
- Секционирование позволяет распределить данные по разным таблицам, что упрощает их обслуживание.
- В таблицы меньшего размера быстрее происходит вставка данных.