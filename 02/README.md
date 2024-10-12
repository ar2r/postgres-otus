# Домашнее задание "Работа с уровнями изоляции транзакции в PostgreSQL"

### Создать новый проект в Яндекс облако или на любых ВМ, докере

Запустить Docker контейнер с Postgres 16

```bash
docker run --name otus-postgres --env=POSTGRES_PASSWORD=pwd -p 5432:5432 -d postgres:16
```

### Запустить два раза psql из под пользователя postgres

```bash
# Сессия 1
psql -h localhost -p 5432 postgres postgres

# Сессия 2
psql -h localhost -p 5432 postgres postgres
```

### Выключить auto commit

```
# Сессия 2
postgres=# \set AUTOCOMMIT off
postgres=# \echo :AUTOCOMMIT
off
```

```
# Сессия 2
postgres-# \set AUTOCOMMIT off
postgres-# \echo :AUTOCOMMIT
off
```

### В первой сессии новую таблицу и наполнить ее данными

```postgresql
# Сессия 1
postgres=#
create table persons
(
    id          serial,
    first_name  text,
    second_name text
);
CREATE TABLE
    postgres=*# insert into persons
(
    first_name,
    second_name
) values
(
    'ivan',
    'ivanov'
);
INSERT 0 1
    postgres=*#
insert into persons(first_name, second_name)
values ('petr', 'petrov');
INSERT 0 1
    postgres=*#
commit;
COMMIT
```

### Посмотреть текущий уровень изоляции

```postgresql
# Сессия 1
postgres=#
show transaction isolation level;
transaction_isolation
-----------------------
 read committed
(1 строка)
```

### Начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции

```postgresql
# Сессия 1
postgres=*#
BEGIN;
BEGIN
postgres=*#
```

```postgresql
# Сессия 2
postgres=*#
BEGIN;
BEGIN
postgres=*#
```

### В первой сессии добавить новую запись

```postgresql
# Сессия 1
postgres=*#
insert into persons(first_name, second_name)
values ('sergey', 'sergeev');
INSERT 0 1
```

### Сделать select from persons во второй сессии

```postgresql
# Сессия 2
postgres=*#
select *
from persons;
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строки)
```

- Запись созданная в сессии 1 не видна в сессии 2, потому что транзакция в сессии 1 не завершена.
- Уровень изоляции по умолчанию - `read committed` (аномалия `dirty read` отсутствует).

### Завершить первую транзакцию

```postgresql
# Сессия 1
postgres=*#
commit;
COMMIT
```

### Сделать select from persons во второй сессии

```postgresql
# Сессия 2
postgres=*#
select *
from persons;
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 строки)
```

- Новая строка видна. Транзакция в сессии 1 завершена, изменения зафиксированы.

### Завершите транзакцию во второй сессии

```postgresql
# Сессия 2
postgres=*#
commit;
COMMIT postgres=#
```

### Начать новые но уже repeatable read транзакции

```postgresql
# Сессия 1
postgres=#
set transaction isolation level repeatable read;
SET
```

```postgresql
# Сессия 2
postgres=#
set transaction isolation level repeatable read;
SET
```

### В первой сессии добавить новую запись

```postgresql
# Сессия 1
postgres=*#
insert into persons(first_name, second_name)
values ('sveta', 'svetova');
INSERT 0 1
```

### Сделать select * from persons во второй сессии

```postgresql
# Сессия 2
postgres=*#
select *
from persons;
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 строки)
```

- Результат добавления новой записи в сессии 1 не виден в сессии 2, потому что транзакция в сессии 1 не завершена.
- Уровень изоляции `repeatable read` (аномалия `phantom read` отсутствует).

### Завершить первую транзакцию

```postgresql
# Сессия 1
postgres=*#
commit;
COMMIT
```

### Сделать select * from persons во второй сессии

```postgresql
# Сессия 2
postgres=*#
select *
from persons;
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
(3 строки)
```

- Новая запись не видна, т.к. текущая транзакция не завершена.
- Уровень изоляции `repeatable read` (аномалия `phantom read` отсутствует).

### Завершить вторую транзакцию

```postgresql
# Сессия 2
postgres=*#
commit;
COMMIT
```

### Сделать select from persons во второй сессии

```postgresql
# Сессия 2
postgres=#
select *
from persons;
id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  4 | sergey     | sergeev
  5 | sveta      | svetova
(4 строки)
```

- Запись видна, т.к. предыдущая транзакция завершена и новый запрос выполняется в новой транзакции.
- Все предыдущие транзакции были завершены до выполнения этого запроса.
