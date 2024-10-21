# Установка Postgres

## 1. Создать ВМ с Ubuntu 20.04/22.04 или развернуть Docker

- Создал ВМ в Яндекс Облаке: [Yandex Cloud Console](https://console.yandex.cloud)
- Выбрал образ Ubuntu, внешний IP адрес, добавил SSH ключ.

## 2. Установка Docker Engine

Установил Docker по официальной [инструкции](https://docs.docker.com/engine/install/ubuntu/):

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Проверка установки
sudo docker --version
# Вывод:
Docker version 27.3.1, build ce12230
```

## 3. Создать каталог для Postgres

```bash
mkdir -p /var/lib/postgresql/data
ls -al /var/lib/ | grep postgres
# Вывод:
drwxr-xr-x  2 root  root  4096 Oct 19 17:38 postgresql
```

## 4. Развернуть контейнер с Postgres

Создал `docker-compose.yaml` в директории `/opt/04`:

```yaml
services:
    psql:
        image: postgres:latest
        container_name: psql_client
        command: ['sleep', 'infinity']
        stdin_open: true
        tty: true
        profiles:
            - manual
  
    pg1:
        image: postgres:17
        container_name: pg1
        environment:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: postgres
        volumes:
        - /var/lib/postgresql/data:/var/lib/postgresql/data
        ports:
        - "5440:5432"
```

```bash
# Проверил, что файл docker-compose.yaml создан
root@otus:/opt/04# ls -al
total 12
drwxr-xr-x 2 root root 4096 Oct 19 17:44 .
drwxr-xr-x 4 root root 4096 Oct 19 17:40 ..
-rw-r--r-- 1 root root  601 Oct 19 17:44 docker-compose.yaml
```

```bash
# Запустить первую БД
root@otus:/opt/04# docker-compose up pg1 -d

# Визуально проверить логи
root@otus:/opt/04# docker logs -f pg1
```

## 5. Развернуть контейнер с клиентом postgres

```bash
# Запустить контейнер с клиентом
root@otus:/opt/04# docker-compose up psql -d

# Проверить, что контейнер запущен и psql клиент доступен
root@otus:/opt/04# docker exec -ti psql_client bash
root@d7b7996cae18:/# psql --help
psql is the PostgreSQL interactive terminal.

Usage:
  psql [OPTION]... [DBNAME [USERNAME]]
...
```


## 6. Подключится из контейнера с клиентом к контейнеру с сервером и сделать таблицу с парой строк

```bash
root@d7b7996cae18:/# psql -h pg1 -p 5432 postgres postgres
psql (17.0 (Debian 17.0-1.pgdg120+1))
postgres=# 
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

## 7. Подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов ЯО/места установки докера

Публичный IP адрес 84.252.132.16 взят из Яндекс Облака. Порт 5440 проброшен на порт 5432 контейнера pg1.

```bash
psql -h 84.252.132.16 -p 5440 postgres postgres
psql (16.4 (Homebrew), сервер 17.0 (Debian 17.0-1.pgdg120+1))
postgres=# select * from employees;
 id |      name      | position  |  salary
----+----------------+-----------+----------
  1 | John Doe       | Manager   | 75000.00
  2 | Jane Smith     | Developer | 68000.50
  3 | Robert Johnson | Designer  | 55000.25
(3 строки)
```
## 8. Удалить контейнер с сервером

```bash
root@otus:/opt/04# docker compose rm pg1
? Going to remove pg1 Yes
[+] Removing 1/0
 ✔ Container pg1  Removed
# Проверить, что ничего лишнего не запущено
 root@otus:/opt/04# docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS      NAMES
d7b7996cae18   postgres:latest   "docker-entrypoint.s…"   22 minutes ago   Up 22 minutes   5432/tcp   psql_client
```

## 9. Создать его заново (первый сервер)

```bash
root@otus:/opt/04# docker compose up pg1 -d
# pg1 запущен
root@otus:/opt/04# docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS                                         NAMES
3a415faac3d9   postgres:17       "docker-entrypoint.s…"   43 seconds ago   Up 42 seconds   0.0.0.0:5440->5432/tcp, [::]:5440->5432/tcp   pg1
d7b7996cae18   postgres:latest   "docker-entrypoint.s…"   23 minutes ago   Up 23 minutes   5432/tcp                                      psql_client
```

## 10. Подключится снова из контейнера с клиентом к контейнеру с сервером

```bash
root@otus:/opt/04# docker exec -ti psql_client bash
root@d7b7996cae18:/# psql -h pg1 -p 5432 postgres postgres
```

## 11. Проверить, что данные остались на месте

```bash
postgres=# select * from employees;
 id |      name      | position  |  salary
----+----------------+-----------+----------
  1 | John Doe       | Manager   | 75000.00
  2 | Jane Smith     | Developer | 68000.50
  3 | Robert Johnson | Designer  | 55000.25
(3 rows)
```

Данные доступны (не удалились).

## 12. Разные проблемы с которыми сталкивался

 - Когда в ЯО остановил и повторно запустил виртальную машину - публичный IP адрес выдался другой, не тот который был выдан до остановки Виртуальной машины.
 - Docker запущен из под root, поэтому проблем с правами доступа не возникло. Иногда Docker запускают не из под root и тогда на директорию /var/lib/postgresql/data требуется навешивать правильные пермишены.