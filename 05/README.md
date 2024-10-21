# ДЗ по теме "Логический уровень"

## 1. Создайте новый кластер PostgresSQL

```bash
root@compute-vm-2-2-10-hdd-1729433848950:~# apt-get update
root@compute-vm-2-2-10-hdd-1729433848950:~# apt install postgresql
root@compute-vm-2-2-10-hdd-1729433848950:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

## 2. Зайдите в созданный кластер под пользователем postgres и создайте новую базу данных testdb

```bash
root@compute-vm-2-2-10-hdd-1729433848950:~# sudo -u postgres psql
postgres@postgres=# create database testdb;
CREATE DATABASE
```

## 3. Зайдите в созданную базу данных под пользователем postgres и создайте новую схему testnm

```bash
postgres@postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
postgres@testdb=# create schema testnm;
CREATE SCHEMA
```

## 4. Создайте новую таблицу t1 с одной колонкой c1 типа integer и вставьте строку со значением c1=1

```bash
postgres@testdb=# create table t1(c1 integer);
CREATE TABLE
postgres@testdb=# insert into t1 values(1);
INSERT 0 1
```

## 5. Создайте новую роль readonly и дайте новой роли право на подключение к базе данных testdb

```bash
postgres@testdb=# create role readonly;
CREATE ROLE
postgres@testdb=# grant connect on database testdb to readonly;
GRANT  
```

## 6. Дайте новой роли право на использование схемы testnm

```bash
postgres@testdb=# grant usage on schema testnm to readonly;
GRANT
```

## 7. Дайте новой роли право на select для всех таблиц схемы testnm

```bash
postgres@testdb=# grant select on all tables in schema testnm to readonly;
GRANT
```

## 8. Создайте пользователя testread с паролем test123

```bash
postgres@testdb=# create user testread with password 'test123';
CREATE ROLE
```

## 9. Дайте роль readonly пользователю testread

```bash
postgres@testdb=# grant readonly to testread;
GRANT ROLE
```

## 10. Зайдите под пользователем testread в базу данных testdb

```bash
root@compute-vm-2-2-10-hdd-1729433848950:~# psql -U testread -d testdb
testread@testdb=>
```

Успешно подключился к базе данных testdb под пользователем testread, но потребовалось изменить конфиг и перезагрузить
сервер БД.

## 11. Сделайте select * from t1;

```bash
testread@testdb=> select * from t1;
ERROR:  permission denied for table t1
```

## 12. Получилось?

- Запрос вернул ошибку.
- Таблицы была создана в схеме `public`, а не в схеме `testnm`. Доступ у пользователя `testread` есть только к схеме
  `testnm`.

```bash
testread@testdb=> SELECT schemaname, tablename
FROM pg_tables
WHERE tablename = 't1';
 schemaname | tablename
------------+-----------
 public     | t1
(1 row)
```

- Таблица `t1` не создалась в схеме `testnm` потому что при создании таблицы не было указано имя схемы. По умолчанию
  таблицы создаются в схеме `public`, если не указана другая схема.
- Чтобы создать таблицу в схеме `testnm`, нужно явно указать схему при создании таблицы:

```sql
CREATE TABLE testnm.t1
(
    c1 integer
);
```

Таким образом, таблица `t1` будет создана в схеме `testnm`.

## 13. Посмотрите на список таблиц

```bash
testread@testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

## 14. Вернитесь в базу данных testdb под пользователем postgres

```bash
postgres@compute-vm-2-2-10-hdd-1729433848950:/root$ psql
psql (16.4 (Ubuntu 16.4-0ubuntu0.24.04.2))
```

## 15. Удалите таблицу t1

```bash
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# drop table t1;
DROP TABLE
```

## 16. Создайте ее заново но уже с явным указанием имени схемы testnm

```bash
testdb=# create table testnm.t1(c1 integer);
CREATE TABLE
```

## 17. Вставьте строку со значением c1=1

```bash
testdb=# insert into testnm.t1 values(1);
INSERT 0 1
```

## 18. Зайдите под пользователем testread в базу данных testdb

```bash
root@compute-vm-2-2-10-hdd-1729433848950:~# psql -U testread -d testdb -W
Password:
psql (16.4 (Ubuntu 16.4-0ubuntu0.24.04.2))
```

## 19. Сделайте select * from testnm.t1;

```bash
testread@testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

## 20. Получилось?

Нет

## 21. Есть идеи почему? если нет - смотрите шпаргалку

Потому, что доступ был предоставлен только к таблицам, которые сущестовали на момент выполнения запроса:

```sql
grant
select
on all tables in schema testnm to readonly;
```

## 22. Как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку

Этот запрос изменяет привилегии по умолчанию для всех будущих таблиц в схеме `testnm`, предоставляя роли `readonly`
право на выполнение команды `SELECT`:

```sql
ALTER
DEFAULT PRIVILEGES IN SCHEMA testnm GRANT
SELECT
ON TABLES TO readonly;
```

## 23. Сделайте select * from testnm.t1;

```bash
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

## 24. Получилось?

Нет.

## 25. Есть идеи почему? если нет - смотрите шпаргалку

- Потому что таблица `t1` была создана до изменения привилегий по умолчанию для всех будущих таблиц в схеме `testnm`.
- Нужно удалить таблицу `t1` и создать ее заново. 
- Или изменить привилегии для существующей таблицы.

## 26. Сделайте select * from testnm.t1;

```bash
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```

## 27. Получилось?

Да.

## 28. Ура!

🫶🏼

## 29. Теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);

```bash
testdb=> create table public.t2(c1 integer); insert into public.t2 values (2);
ERROR:  permission denied for schema public
LINE 1: create table t2(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
```

У моего пользователя нет прав на создание таблиц в схеме `public`.

## 30. А как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?

```bash
testdb=> \dn+
                                       List of schemas
  Name  |       Owner       |           Access privileges            |      Description
--------+-------------------+----------------------------------------+------------------------
 public | pg_database_owner | pg_database_owner=UC/pg_database_owner+| standard public schema
        |                   | =U/pg_database_owner                   |
 testnm | postgres          | postgres=UC/postgres                  +|
        |                   | readonly=U/postgres                    |
(2 rows)
```

Команда `\dn+` показывает список схем в базе данных, их владельцев и привилегии доступа. В данном случае:

- Схема `public` принадлежит роли `pg_database_owner` и имеет привилегии `UC` (Usage и Create) для роли `pg_database_owner`.
- Схема `testnm` принадлежит роли `postgres` и имеет привилегии `UC` для роли `postgres` и привилегии `U` (Usage) для роли `readonly`.

Это означает, что:
- Роль `pg_database_owner` может использовать и создавать объекты в схеме `public`.
- Роль `postgres` может использовать и создавать объекты в схеме `testnm`.
- Роль `readonly` может только использовать объекты в схеме `testnm`.

## 31. Есть идеи как убрать эти права? если нет - смотрите шпаргалку

*У меня данный сценарий не воспроизвелся.*

## 32. Если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды

_Ничего не делал._

## 33. Теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);

_Пропущено._

## 34. Расскажите что получилось и почему

_Пропущено._ 