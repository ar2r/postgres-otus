# Домашняя работы на тему "Индексы"

За основу взял БД: https://github.com/neondatabase-labs/postgres-sample-dbs/blob/main/employees.sql.gz

## 1. Создать индекс к какой-либо из таблиц вашей БД

```sql
create index salary_amount_index
    on employees.salary (amount);
```

## 2. Прислать текстом результат команды explain, в которой используется данный индекс

```
explain analyse select * from salary where amount = 80013;

-- Результат без индекса
Gather  (cost=1000.00..33978.04 rows=53 width=24) (actual time=1.194..108.451 rows=35 loops=1)
  Workers Planned: 2
  Workers Launched: 2
  ->  Parallel Seq Scan on salary  (cost=0.00..32972.74 rows=22 width=24) (actual time=4.636..101.823 rows=12 loops=3)
        Filter: (amount = 80013)
        Rows Removed by Filter: 948004
Planning Time: 0.138 ms
Execution Time: 108.479 ms

-- Результат с индексом
Bitmap Heap Scan on salary  (cost=4.84..208.91 rows=53 width=24) (actual time=0.191..1.986 rows=35 loops=1)
  Recheck Cond: (amount = 80013)
  Heap Blocks: exact=35
  ->  Bitmap Index Scan on salary_amount_index  (cost=0.00..4.83 rows=53 width=0) (actual time=0.160..0.160 rows=35 loops=1)
        Index Cond: (amount = 80013)
Planning Time: 0.476 ms
Execution Time: 2.057 ms
```
Добавленный индекс позволил искать в таблицы без sec scan. За счет индекса скорость выполнения запроса увеличилась почти в 100 раз.

## 3. Реализовать индекс для полнотекстового поиска

```sql
CREATE INDEX idx_employee_last_name ON employee USING GIN (to_tsvector('english', last_name));

-- Запрос, который не используется индекс выполняется за 40ms
explain analyse
select *
from employee
where lower(last_name) = lower('Swan');

-- Запрос использующий индекс
explain analyse
SELECT *
FROM employee
WHERE to_tsvector('english', last_name) @@ plainto_tsquery('Swan');

-- Результат с индксом
Bitmap Heap Scan on employee  (cost=20.93..3056.38 rows=1500 width=35) (actual time=0.101..0.714 rows=196 loops=1)
"  Recheck Cond: (to_tsvector('english'::regconfig, (last_name)::text) @@ to_tsquery('sWaN'::text))"
  Heap Blocks: exact=187
  ->  Bitmap Index Scan on idx_employee_last_name  (cost=0.00..20.56 rows=1500 width=0) (actual time=0.064..0.065 rows=196 loops=1)
"        Index Cond: (to_tsvector('english'::regconfig, (last_name)::text) @@ to_tsquery('sWaN'::text))"
Planning Time: 0.265 ms
Execution Time: 0.780 ms
```

Время выполнения запроса уменьшилось более чем в 40 раз.

## 4. Реализовать индекс на часть таблицы или индекс на поле с функцией

```sql
CREATE INDEX idx_employee_birth_date_before_2000
ON employees.employee (birth_date)
WHERE birth_date < '2000-01-01';

-- Запрос не использует индекс
explain analyse SELECT * FROM employee where birth_date = '2005-01-01 00:00:00'

Gather  (cost=1000.00..5716.36 rows=63 width=35) (actual time=13.112..14.901 rows=0 loops=1)
  Workers Planned: 1
  Workers Launched: 1
  ->  Parallel Seq Scan on employee  (cost=0.00..4710.06 rows=37 width=35) (actual time=9.490..9.491 rows=0 loops=2)
        Filter: (birth_date = '2005-01-01'::date)
        Rows Removed by Filter: 150012
Planning Time: 0.136 ms
Execution Time: 14.921 ms

-- Запрос использует индекс
explain analyse SELECT * FROM employee where birth_date = '1995-01-01 00:00:00';

Bitmap Heap Scan on employee  (cost=4.79..227.59 rows=63 width=35) (actual time=0.141..0.142 rows=0 loops=1)
  Recheck Cond: (birth_date = '1995-01-01'::date)
  ->  Bitmap Index Scan on idx_employee_birth_date_before_2000  (cost=0.00..4.77 rows=63 width=0) (actual time=0.110..0.111 rows=0 loops=1)
        Index Cond: (birth_date = '1995-01-01'::date)
Planning Time: 0.381 ms
Execution Time: 0.198 ms
```

## 5. Создать индекс на несколько полей

```sql
create index title_from_date_to_date_index
    on employees.title (from_date, to_date);
    
-- Запрос не использует индекс, т.к. в условии не используется фильтрация по полю from_date
explain analyse SELECT * FROM title where to_date > '2000-01-01';    
    
-- Запросы использующие индексы:
explain analyse SELECT * FROM title where from_date > '2000-01-01';
explain analyse SELECT * FROM title where from_date > '2000-01-01' AND to_date < '2024-01-01';
```

Порядок полей в индексе имеет значение. Индекс не будет использоваться, если в условии WHERE пропускать поля, которые указаны первыми в списке.
