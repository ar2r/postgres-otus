# Домашняя работа по теме "Блокировки"

## 0. Создать таблицу с данными

```bash
  
# Подключение к базе
artur@otus:~$ sudo -u postgres psql

# Создание таблицы
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    position VARCHAR(50),
    salary NUMERIC(10, 2)
);
INSERT INTO employees (name, position, salary) 
VALUES
('John Doe', 'Manager', 75000.00),
('Jane Smith', 'Developer', 68000.50),
('Robert Johnson', 'Designer', 55000.25);
postgres=# select * from employees;
 id |      name      | position  |  salary
----+----------------+-----------+----------
  1 | John Doe       | Manager   | 75000.00
  2 | Jane Smith     | Developer | 68000.50
  3 | Robert Johnson | Designer  | 55000.25
(3 rows)
```

## 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

```bash
nano /etc/postgresql/16/main/postgresql.conf
# Добавить в конец файла строки
log_lock_waits = on
deadlock_timeout = 200ms

# Перезапуск сервера
sudo systemctl restart postgresql

# Проверка, что настройки применились
root@otus08:/home/artur# sudo -u postgres psql -c "SHOW log_lock_waits"
 log_lock_waits
----------------
 on
(1 row)

root@otus08:/home/artur# sudo -u postgres psql -c "SHOW deadlock_timeout"
 deadlock_timeout
------------------
 200ms
(1 row)
```

## 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

В трех разных сеансах выполнить SQL запросы:
```bash
BEGIN;
UPDATE employees SET salary = 80000 WHERE id = 1;
```

Результат: в первом сеансе UPDATE запросы выполнился. В остальных сеансах запросы UPDATE заблокировались и висят не выполненными.

В лог файле появились сообщения о блокировках:
```bash
2024-11-16 19:36:30.411 UTC [4600] postgres@postgres LOG:  process 4600 still waiting for ShareLock on transaction 743 after 200.079 ms
2024-11-16 19:36:30.411 UTC [4600] postgres@postgres DETAIL:  Process holding the lock: 4604. Wait queue: 4600.
2024-11-16 19:36:30.411 UTC [4600] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "employees"
2024-11-16 19:36:30.411 UTC [4600] postgres@postgres STATEMENT:  UPDATE employees SET salary = 80000 WHERE id = 1;
2024-11-16 19:36:34.284 UTC [4596] postgres@postgres LOG:  process 4596 still waiting for ExclusiveLock on tuple (0,1) of relation 16389 of database 5 after 200.109 ms
2024-11-16 19:36:34.284 UTC [4596] postgres@postgres DETAIL:  Process holding the lock: 4600. Wait queue: 4596.
2024-11-16 19:36:34.284 UTC [4596] postgres@postgres STATEMENT:  UPDATE employees SET salary = 80000 WHERE id = 1;
```

Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.
    
```bash
# Подключение к базе
artur@otus:~$ sudo -u postgres psql

# Просмотр блокировок
postgres=#  SELECT * FROM pg_locks WHERE relation = 16389;
locktype | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid  |       mode       | granted | fastpath |           waitstart
----------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+------+------------------+---------+----------+-------------------------------
relation |        5 |    16389 |      |       |            |               |         |       |          | 5/2                | 4604 | RowExclusiveLock | t       | t        |
relation |        5 |    16389 |      |       |            |               |         |       |          | 4/2                | 4600 | RowExclusiveLock | t       | t        |
relation |        5 |    16389 |      |       |            |               |         |       |          | 3/20               | 4596 | RowExclusiveLock | t       | t        |
tuple    |        5 |    16389 |    0 |     1 |            |               |         |       |          | 4/2                | 4600 | ExclusiveLock    | t       | f        |
tuple    |        5 |    16389 |    0 |     1 |            |               |         |       |          | 3/20               | 4596 | ExclusiveLock    | f       | f        | 2024-11-16 19:36:34.084168+00
(5 rows)
```

В данном случае, строки показывают, что три процесса (4604, 4600, 4596) удерживают блокировки RowExclusiveLock на таблице 16389, и два процесса (4600, 4596) удерживают блокировки ExclusiveLock на строке 0,1 таблицы 16389. Процесс 4596 ожидает предоставления блокировки.

## 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

В трех разных сеансах выполнить SQL запросы:
```bash
# Сеанс 1
BEGIN;
UPDATE employees SET salary = 80000 WHERE id = 1;

# Сеанс 2
BEGIN;
UPDATE employees SET salary = 85000 WHERE id = 2;

# Сеанс 3
BEGIN;
UPDATE employees SET salary = 90000 WHERE id = 3;

# Сеанс 1
UPDATE employees SET salary = 95000 WHERE id = 2;

# Сеанс 2
UPDATE employees SET salary = 100000 WHERE id = 3;

# Сеанс 3
UPDATE employees SET salary = 105000 WHERE id = 1;
```

Результат: все три сеанса заблокируются, так как каждая транзакция будет ждать завершения другой.

Но в данном случае, в журнале сообщений не будет информации о взаимоблокировке, так как PostgreSQL не обнаруживает взаимоблокировку. Вместо этого, транзакции будут ждать завершения друг друга.

## 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Да, могут, так как обе транзакции будут пытаться заблокировать всю таблицу целиком. Пока одна транзакция удерживает блокировку, другая транзакция будет ждать завершения первой транзакции.

