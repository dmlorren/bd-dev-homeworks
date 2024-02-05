# Домашнее задание к занятию 2. «SQL» - Иванов Дмитрий (fops-13)


## Задача 1

Используя Docker, поднимите инстанс PostgreSQL (версию 12) c 2 volume, 
в который будут складываться данные БД и бэкапы.

Приведите получившуюся команду или docker-compose-манифест.

### Решение задания 1

1. Поднята виртуальная машина в на платформе debian. Пакет докер (включая compose) установлен согласно инструкции: https://docs.docker.com/engine/install/debian/

```
лог из хистори:
  su
  apt-get install ca-certificates curl
  install -m 0755 -d /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
  chmod a+r /etc/apt/keyrings/docker.asc
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |  tee /etc/apt/sources.list.d/docker.list > /dev/null
  apt update
  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  service docker status
  systemctl status docker
```

2. Подготовка рабочей директории:
```
mkdir netology_sql
cd netology_sql
```

3. Манифест docker-compose:
```
cat docker-compose.yml

version: '3.8'

volumes:
  data: {}
  backup: {}

services:
  postgres:
    image: postgres:12
    container_name: psql
    restart: always
    ports:
      - "0.0.0.0:5432:5432"
    volumes:
      - data:/var/lib/postgresql/data
      - backup:/media/postgresql/backup
    environment:
      TZ: "Europe/Moscow"
      POSTGRES_USER: "dmlorren"
      POSTGRES_PASSWORD: "123456"
      POSTGRES_DB: "test_db"  

```
![docker_ps](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/docker_ps.png)

4. Заходим докер-контейнрер, а затем в БД test_db (так как уже создали):
```
docker exec -it bee47eff6ae8 bash
psql -h 127.0.0.1 -U dmlorren -d test_db
```
![test_db](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/test_db.png)


## Задача 2

В БД из задачи 1: 

- создайте пользователя test-admin-user и БД test_db;
- в БД test_db создайте таблицу orders и clients (спeцификация таблиц ниже);
- предоставьте привилегии на все операции пользователю test-admin-user на таблицы БД test_db;
- создайте пользователя test-simple-user;
- предоставьте пользователю test-simple-user права на SELECT/INSERT/UPDATE/DELETE этих таблиц БД test_db.

Таблица orders:

- id (serial primary key);
- наименование (string);
- цена (integer).

Таблица clients:

- id (serial primary key);
- фамилия (string);
- страна проживания (string, index);
- заказ (foreign key orders).

Приведите:

- итоговый список БД после выполнения пунктов выше;
- описание таблиц (describe);
- SQL-запрос для выдачи списка пользователей с правами над таблицами test_db;
- список пользователей с правами над таблицами test_db.


### Решение задания 2

1. База данных у нас уже создана, теперь согласно ТЗ создадим нового пользователя.
```sql
CREATE USER "test-admin-user";
```
![user](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/user.png)

2. Создаём таблицу orders.
```sql
CREATE TABLE orders (
    id SERIAL,
    наименование VARCHAR, 
    цена INTEGER,
    PRIMARY KEY (id)
);
```

2.1 Создаём таблицу clients.
```sql
CREATE TABLE clients (
    id SERIAL,
    фамилия VARCHAR,
    "страна проживания" VARCHAR, 
    заказ INTEGER,
    PRIMARY KEY (id)
);
```

2.2 Выдаём права пользователю test-admin-user.
```sql
GRANT ALL PRIVILEGES ON TABLE orders, clients to "test-admin-user";
```
![table](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/table.png)

2.3 Создаём пользователя test-simple-user:
```sql
CREATE USER "test-simple-user";
```

2.4 Выдаём пользователю test-simple-user права на SELECT/INSERT/UPDATE/DELETE:
```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE orders, clients to "test-simple-user";
```

`Итоги:`
- итоговый список БД после выполнения пунктов выше:
![l+](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/l%2B.png)

- описание таблиц (describe):
![tables](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/tables.png)

- SQL-запрос для выдачи списка пользователей с правами над таблицами test_db:
```sql
SELECT 
    grantee, table_name, privilege_type 
FROM 
    information_schema.table_privileges 
WHERE 
    grantee in ('test-admin-user','test-simple-user')
    and table_name in ('clients','orders')
order by 
    grantee;
```
- список пользователей с правами над таблицами test_db:
![grantee](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/grantee.png)


## Задача 3

Используя SQL-синтаксис, наполните таблицы следующими тестовыми данными:

Таблица orders

|Наименование|цена|
|------------|----|
|Шоколад| 10 |
|Принтер| 3000 |
|Книга| 500 |
|Монитор| 7000|
|Гитара| 4000|

Таблица clients

|ФИО|Страна проживания|
|------------|----|
|Иванов Иван Иванович| USA |
|Петров Петр Петрович| Canada |
|Иоганн Себастьян Бах| Japan |
|Ронни Джеймс Дио| Russia|
|Ritchie Blackmore| Russia|

Используя SQL-синтаксис:
- вычислите количество записей для каждой таблицы.

Приведите в ответе:

    - запросы,
    - результаты их выполнения.


### Решение задания 3

- Добавляем тестовые данные в orders:
```sql
INSERT INTO orders VALUES (1, 'Шоколад', 10), (2, 'Принтер', 3000), (3, 'Книга', 500), (4, 'Монитор', 7000), (5, 'Гитара', 4000);

SELECT * FROM orders;
```
![orders](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/orders.png)

- Добавляем тестовые данные в clients:
```sql
INSERT INTO clients VALUES (1, 'Иванов Иван Иванович', 'USA'), (2, 'Петров Петр Петрович', 'Canada'), (3, 'Иоганн Себастьян Бах', 'Japan'), (4, 'Ронни Джеймс Дио', 'Russia'), (5, 'Ritchie Blackmore', 'Russia');

SELECT * FROM clients;
```
![clients](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/clients.png)

- Высчитываем количество записей для каждой таблицы:
```sql
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM clients;
```


## Задача 4

Часть пользователей из таблицы clients решили оформить заказы из таблицы orders.
Используя foreign keys, свяжите записи из таблиц, согласно таблице:

|ФИО|Заказ|
|------------|----|
|Иванов Иван Иванович| Книга |
|Петров Петр Петрович| Монитор |
|Иоганн Себастьян Бах| Гитара |

Приведите SQL-запросы для выполнения этих операций.
Приведите SQL-запрос для выдачи всех пользователей, которые совершили заказ, а также вывод этого запроса.
Подсказка: используйте директиву `UPDATE`.

### Решение задания 4

- Список запросов:
```sql
UPDATE clients SET заказ=3 WHERE id=1;
UPDATE clients SET заказ=4 WHERE id=2;
UPDATE clients SET заказ=5 WHERE id=3;

SELECT * FROM clients;
SELECT * FROM clients WHERE заказ IS NOT NULL;
```
![update](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/update.png)


## Задача 5

Получите полную информацию по выполнению запроса выдачи всех пользователей из задачи 4 
(используя директиву EXPLAIN).
Приведите получившийся результат и объясните, что значат полученные значения.

### Решение задания 5

- Команда по выводу анализа запроса будет иметь вид:
```sql
EXPLAIN (FORMAT JSON) SELECT * FROM clients WHERE заказ IS NOT NULL;
```

![explain](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/explain.png)

```
 [                                        +
   {                                      +
     "Plan": {                            +
       "Node Type": "Seq Scan",           +
       "Parallel Aware": false,           +
       "Relation Name": "clients",        +
       "Alias": "clients",                +
       "Startup Cost": 0.00,              +
       "Total Cost": 18.10,               +
       "Plan Rows": 806,                  +
       "Plan Width": 72,                  +
       "Filter": "(\"заказ\" IS NOT NULL)"+
     }                                    +
   }                                      +
 ]
(1 row)
```

> Seq Scan - показывает, что при выполнении запроса последовательно считывается каждая запись таблицы.
> Rows - примерное количество строк, возвращаемых на каждом этапе плана запроса.
> Width - предполагаемый средний размер строк в байтах.
> Startup Cost, Total Cost - ориентировочная стоимость выполнения запроса состоящая из двух частей: стартовая стоимость и общая стоимость. 
> Чем ниже стоимость, тем более эффективным будет запрос.
> Relation Name - имя таблицы над котороый производится анализ.
> Filter - каждая запись сравнивается с условием "заказ" IS NOT NULL, если условие выполняется, запись вводится в результат, если нет, то >  > отбрасывается.


## Задача 6

Создайте бэкап БД test_db и поместите его в volume, предназначенный для бэкапов (см. задачу 1).
Остановите контейнер с PostgreSQL, но не удаляйте volumes.
Поднимите новый пустой контейнер с PostgreSQL.
Восстановите БД test_db в новом контейнере.
Приведите список операций, который вы применяли для бэкапа данных и восстановления. 


### Решение задания 6

1. Создаём бэкап:

```
pg_dump -U dmlorren test_db > /media/postgresql/backup/test_db.backup
root@dd23429f1884:/# ls -lah /media/postgresql/backup/
total 16K
drwxr-xr-x 2 root root 4.0K Feb  5 01:26 .
drwxr-xr-x 3 root root 4.0K Feb  3 22:36 ..
-rw-r--r-- 1 root root 4.1K Feb  5 01:26 test_db.backup
```

2. Стопаем контейнер:

```
docker ps -a
docker stop dd23429f1884
```
![docker](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/docker.png)

3. Поднимаем новый контейнер с постгрес и восстанавливаем БД:

```
docker pull postgres
docker run --name postgres123 -e POSTGRES_PASSWORD=123456 -d postgres
docker cp psql://media/postgresql/backup/test_db.backup backup123
docker cp backup123 postgres123:/home/

# заходим в контейнер
docker exec -it 98933f5012dc bash

# в самом поднятом контейнере
psql -U postgres -f /home/backup123
# заходим в БД и проверем данные в таблицах
psql -U postgres
psql -U -d template0
select * from clients;
select * from orders;

```
![restore](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-02-sql/img/restore.png)

