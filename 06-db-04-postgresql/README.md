# Домашнее задание к занятию 4. «PostgreSQL» - Иванов Дмитрий (fops-13)

## Задача 1

Используя Docker, поднимите инстанс PostgreSQL (версию 13). Данные БД сохраните в volume.
Подключитесь к БД PostgreSQL, используя `psql`.
Воспользуйтесь командой `\?` для вывода подсказки по имеющимся в `psql` управляющим командам.

**Найдите и приведите** управляющие команды для:

- вывода списка БД,
- подключения к БД,
- вывода списка таблиц,
- вывода описания содержимого таблиц,
- выхода из psql.

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
mkdir netology_psql
cd netology_psql
``` 

3. Манифест docker-compose.yml
``` 
/home/dmlorren/netology_psql/docker-compose.yml 
version: '3.8'

volumes:
  data: {}
  backup: {}

services:
  postgres:
    image: postgres:13
    container_name: psql
    ports:
      - "0.0.0.0:5432:5432"
    volumes:
      - data:/var/lib/postgresql/data
      - backup:/media/postgresql/backup
    environment:
      POSTGRES_USER: "admin"
      POSTGRES_PASSWORD: "admin"
``` 

4. Поднимаем контейнер, убеждаемся, что запущен и работает:
``` 
docker-compose up -d
docker ps -a
``` 
![docker_ps](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-04-postgresql/img/docker_ps.png)

5. Заходим в контейнер, подключаемся к БД postgres, выполняем управляющий команды:
``` 
docker exec -it c9f90f0ae11d bash
psql -h 127.0.0.1 -U admin
\l
\c postgres
\dt
# в базе отсутствуют таблицы, поэтому вывод пустой
show * from table_name;
\q
``` 
![psql_bd](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-04-postgresql/img/psql_bd.png)


## Задача 2

Используя `psql`, создайте БД `test_database`.
Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/virt-11/06-db-04-postgresql/test_data).
Восстановите бэкап БД в `test_database`.
Перейдите в управляющую консоль `psql` внутри контейнера.
Подключитесь к восстановленной БД и проведите операцию ANALYZE для сбора статистики по таблице.
Используя таблицу [pg_stats](https://postgrespro.ru/docs/postgresql/12/view-pg-stats), найдите столбец таблицы `orders` 
с наибольшим средним значением размера элементов в байтах.

**Приведите в ответе** команду, которую вы использовали для вычисления, и полученный результат.


### Решение задания 2

- В этом задании мне снова пришлось использовать костыли,чтобы выполнить восстановление БД: переезжать на винду, редактировать бекап базы в vscode/notepad++ удалив комментарии: 
``` 
-- Dumped from database version 13.0 (Debian 13.0-1.pgdg100+1)
-- Dumped by pg_dump version 13.0 (Debian 13.0-1.pgdg100+1)
``` 
- пробовал восстановить через pg_restore (что логично, так как дамп сделан через pg_dump у нас в этих строках так и написано):
``` 
> root@69aacac29576:/# pg_restore -U admin -d test_database < /tmp/Test_dump.sql 
> pg_restore: error: input file appears to be a text format dump. Please use psql.
``` 
- окей, пробуем через psql и тут ловлю ошибку, которую абсолютно не могу победить:
``` 
root@c9f90f0ae11d:/# psql -h localhost -U admin -d test_database < /opt/test_dump.sql 
ERROR:  syntax error at or near "{"
LINE 1: {"payload":{"allShortcutsEnabled":false,"fileTree":{"06-db-0...
        ^
``` 
- перепробывал все доступные варианты синтаксиса psql указанные на stackoverflow, но цивилизованного варианта восстановить дамп так и не нашёл, поэтому костыль.


1. Создаём тестовую БД:
``` 
create database test_database;

admin=# \l
                               List of databases
     Name      | Owner | Encoding |  Collate   |   Ctype    | Access privileges 
---------------+-------+----------+------------+------------+-------------------
 admin         | admin | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres      | admin | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0     | admin | UTF8     | en_US.utf8 | en_US.utf8 | =c/admin         +
               |       |          |            |            | admin=CTc/admin
 template1     | admin | UTF8     | en_US.utf8 | en_US.utf8 | =c/admin         +
               |       |          |            |            | admin=CTc/admin
 test_database | admin | UTF8     | en_US.utf8 | en_US.utf8 | 
(5 rows)


\q
``` 

2. Выкачиваем дамп БД, не в контейнер, а в VM, где у нас этот контейнер и поднят, далее сразу восстанавливаем базу (редактируем в винде):
``` 
wget https://github.com/netology-code/virt-homeworks/blob/virt-11/06-db-04-postgresql/test_data/test_dump.sql

psql -U admin -d test_database < /tmp/Test_dump.sql
``` 

3. Подключаемся к нашей тестовой БД:
``` 
psql -U admin -d test_database
``` 

4. Выводим список существующих в базе таблиц:
``` 
test_database=# \dt
        List of relations
 Schema |  Name  | Type  | Owner 
--------+--------+-------+-------
 public | orders | table | admin
(1 row)
``` 

5. Выполняем команду ANALYZE:
``` 
analyze verbose public.orders;
``` 
> Эта команда выполняет анализ статистики для таблицы "public.orders" в базе данных Postgres. Анализ статистики помогает оптимизировать выполнение SQL > запросов, так как база данных использует эти статистики для создания планов выполнения запросов. 

> Команда "analyze" с ключевым словом "verbose" запускает более подробный анализ, который может включать в себя сбор информации о различных аспектах  таблицы, таких как количество строк, распределение значений в столбцах, и другие статистические данные. 

> После выполнения данной команды, база данных будет обновлять статистику для таблицы "public.orders", что поможет ей принимать более оптимальные > решения о том, как выполнять запросы к этой таблице.

``` 
test_database=# analyze verbose public.orders;
INFO:  analyzing "public.orders"
INFO:  "orders": scanned 1 of 1 pages, containing 8 live rows and 0 dead rows; 8 rows in sample, 8 estimated total rows
ANALYZE
``` 


6. Теперь в таблице orders мы ищем столбец в котором содержится наибольшее среднее значение:
``` 
select avg_width from pg_stats where tablename='orders';
``` 
> Эта команда выполняет запрос к системной таблице pg_stats в базе данных Postgres. 

> 1. Сначала команда выбирает столбец 'avg_width' из таблицы pg_stats, который содержит статистику по средней ширине значений в столбцах таблиц базы данных.

> 2. Затем команда использует условие "where tablename='orders'", что означает, что она ищет информацию о средней ширине значений для столбцов таблицы "orders".

> 3. После выполнения этой команды, она вернет среднюю ширину значений в столбцах таблицы "orders". Эта информация может быть полезна для оптимизации запросов к данной таблице, так как она поможет базе данных лучше понимать структуру данных и принимать правильные решения о выполнении запросов.
``` 
test_database=# select avg_width from pg_stats where tablename='orders';
 avg_width 
-----------
         4
        16
         4
(3 rows)
``` 
- Видим, что во втором столбце средняя ширина составляет 16 байт, выведем всю таблицу целиком:
``` 
select * from orders;
``` 
- Второй столбец - это описание продукции, по количеству символов оно самое ёмкое.
``` 
test_database=# select * from orders;
 id |        title         | price 
----+----------------------+-------
  1 | War and peace        |   100
  2 | My little database   |   500
  3 | Adventure psql time  |   300
  4 | Server gravity falls |   300
  5 | Log gossips          |   123
  6 | WAL never lies       |   900
  7 | Me and my bash-pet   |   499
  8 | Dbiezdmin            |   501
(8 rows)
``` 


## Задача 3

Архитектор и администратор БД выяснили, что ваша таблица orders разрослась до невиданных размеров и
поиск по ней занимает долгое время. Вам как успешному выпускнику курсов DevOps в Нетологии предложили
провести разбиение таблицы на 2: шардировать на orders_1 - price>499 и orders_2 - price<=499.
Предложите SQL-транзакцию для проведения этой операции.
Можно ли было изначально исключить ручное разбиение при проектировании таблицы orders?


### Решение задания 3

1. Сделаем всё в 4 итерации:
``` 
1.1 создадм таблицу orders_1;
1.2 создадм таблицу orders_2;
1.3 заполним orders_1 данными из orders (оригинальной);
1.4 заполним orders_2 данными из orders (оригинальной);
``` 
- создаём таблицы:
``` 
create table orders_1 (
id bigint not null,
title text not null,
price bigint not null);
``` 

``` 
create table orders_2 (
id bigint not null,
title text not null,
price bigint not null);
``` 

`Подробное пояснение:`
> Эта команда выполняет создание новой таблицы "orders_1" в базе данных Postgres. 
> 1. Команда "create table" используется для создания новой таблицы в базе данных.
> 2. "orders_1" - это название новой таблицы, которая будет создана.
> 3. Далее, в скобках, указываются столбцы таблицы и их атрибуты:
>   - "id bigint not null" - создается столбец "id" с типом данных bigint (целое число большой диапазон), а также указывается, что значение этого столбца не может быть пустым (not null).
>   
>   - "title text not null" - создается столбец "title" с типом данных text (текстовая строка переменной длины), и также указывается, что значение этого столбца не может быть пустым.
>   
>   - "price bigint not null" - создается столбец "price" с типом данных bigint и, аналогично, значение этого столбца не может быть пустым.
>
>4. Таким образом, команда создает новую таблицу "orders_1" с тремя столбцами: "id", "title" и "price". Все эти столбцы обязательно должны содержать > значения, и их тип данных задан соответствующим образом для хранения определенных типов информации (идентификатор заказа, название товара, цена). 
>
>После выполнения этой команды, новая таблица будет создана в базе данных и будет доступна для записи и чтения данных.

- заполняем новые таблицы:
``` 
insert into orders_1
select * from orders
where price > 499;
``` 

``` 
insert into orders_2
select * from orders
where price <= 499;
``` 

`Подробное пояснение:`

> Эта команда выполняет операцию вставки данных из одной таблицы в другую в базе данных Postgres. Давайте подробно разберем каждую часть команды:
> 1. "insert into orders_1" - это указание на то, что данные будут вставляться в таблицу "orders_1".
>
> 2. "select * from orders" - это подзапрос, который выбирает (читает) все столбцы и строки из таблицы "orders".
> 
> 3. "where price > 499" - это условие, которое указывает, что только строки из таблицы "orders", у которых значение в столбце "price" больше 499, будут вставлены в таблицу "orders_1".

> Итак, данная команда выполняет следующее:
> - Выбирает все строки и столбцы из таблицы "orders".
> - Фильтрует только те строки, где значение столбца "price" больше 499.
> - Вставляет отфильтрованные строки в таблицу "orders_1".
>
> Таким образом, после выполнения этой команды, в таблицу "orders_1" будут добавлены данные из таблицы "orders", удовлетворяющие условию "price > 499".


2. Данные импортированы, теперь проверим:
```
select * from orders_1;
select * from orders_2;
```
![shard](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-04-postgresql/img/shard.png)
- в оригинальной таблице содержится 8 строк, на скриншоте видим, что всё шардировано верно, 3 строки в orders_1 и 5 строк в orders_2

3. Так как в задании жалобы на разросшеюся таблицу, то старую можно удалить:
```
drop table orders;
```

4. На вопрос можно ли было изначально исключить ручное разбиение при проектировании таблицы orders, ответить предполагаю так:
```
При планировании можно было применить сегментацию таблиц. Т.е. мы тем самым смогли бы разделить данных на несколько физических или логических частей (масташтабировать) называемых partiitions (разделениями). Каждое разделение может содержать часть данных, относящихся к определённому условию или значению ключа, что позволяет упростить управление данными и ускорить выполнение запросов. В каждой партиции (разделении) мы могли бы проводить архивирование или удаление данных.

Для разделения таблиц на сегменты в PostgreSQL можно использовать такие методы как:
Range Partitioning (разделение по диапазону), 
List Partitioning (разделение по списку), 
Hash Partitioning (разделение по хэшу) и другие. 
Каждый метод имеет свои особенности и применимость в зависимости от требований и характеристик данных.
```

## Задача 4

Используя утилиту `pg_dump`, создайте бекап БД `test_database`.
Как бы вы доработали бэкап-файл, чтобы добавить уникальность значения столбца `title` для таблиц `test_database`?


### Решение задания 4

1. Создаём бекап из под докера:
```
root@69aacac29576:/# pg_dump -U admin test_database > /opt/test_database_super_puper_backup.sql

# проверяем наличие
root@69aacac29576:/# ls -lah /opt/
total 12K
drwxr-xr-x 1 root root 4.0K Feb 13 18:13 .
drwxr-xr-x 1 root root 4.0K Feb 12 21:58 ..
-rw-r--r-- 1 root root 1.5K Feb 13 18:13 test_database_super_puper_backup.sql

# смотрим, что внутри
root@69aacac29576:/# cat /opt/test_database_super_puper_backup.sql 
--
-- PostgreSQL database dump
--

-- Dumped from database version 13.13 (Debian 13.13-1.pgdg120+1)
-- Dumped by pg_dump version 13.13 (Debian 13.13-1.pgdg120+1)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: orders_1; Type: TABLE; Schema: public; Owner: admin
--

CREATE TABLE public.orders_1 (
    id bigint NOT NULL,
    title text NOT NULL,
    price bigint NOT NULL
);


ALTER TABLE public.orders_1 OWNER TO admin;

--
-- Name: orders_2; Type: TABLE; Schema: public; Owner: admin
--

CREATE TABLE public.orders_2 (
    id bigint NOT NULL,
    title text NOT NULL,
    price bigint NOT NULL
);


ALTER TABLE public.orders_2 OWNER TO admin;

--
-- Data for Name: orders_1; Type: TABLE DATA; Schema: public; Owner: admin
--

COPY public.orders_1 (id, title, price) FROM stdin;
2	My little database	500
6	WAL never lies	900
8	Dbiezdmin	501
\.


--
-- Data for Name: orders_2; Type: TABLE DATA; Schema: public; Owner: admin
--

COPY public.orders_2 (id, title, price) FROM stdin;
1	War and peace	100
3	Adventure psql time	300
4	Server gravity falls	300
5	Log gossips	123
7	Me and my bash-pet	499
\.


--
-- PostgreSQL database dump complete
--
```

2. Чтобы добавить уникальность значения столбца `title` для таблиц `test_database` нам нужен индекс!

- на этапе создания таблиц можно добавить unique:
```
create table orders_1 (
id bigint not null,
title text not null, unique,
price bigint not null);


create table orders_2 (
id bigint not null,
title text not null, unique,
price bigint not null);
```
- либо уже к существующим таблицам добавить:
```
alter table orders_1 add  constraint constraintname unique (title);
alter table orders_2 add  constraint constraintname unique (title);
```

`Объяснение:`
>Указанная выше команда в PostgreSQL выполняет добавление уникального ограничения (constraint) на столбец title таблицы orders_1. Уникальное ограничение гарантирует, что значения в указанном столбце будут уникальными для каждой строки в таблице, то есть нельзя будет добавить две строки с одинаковым значением в столбце title.
>
>В данном случае:
>- alter table orders_1 указывает, что мы хотим изменить структуру таблицы orders_1.
>- add constraint constraintname добавляет новое ограничение на таблицу с указанным именем constraintname. Имя ограничения может быть выбрано произвольно и должно быть уникальным в пределах таблицы.
>- unique (title) определяет тип ограничения как уникальное, что означает, что все значения в столбце title должны быть уникальными.
>
>После выполнения этой команды, PostgreSQL проверит наличие дублирующихся значений в столбце title и не допустит добавления строки с уже существующим значением, если ограничение будет нарушено. Это поможет обеспечить целостность данных и предотвратить дублирование >информации в указанном столбце таблицы orders_1.
>
>Уникальные ограничения часто используются для обеспечения уникальности значений в ключевых полях, идентификаторах или других важных столбцах в базе данных. Они помогают избежать ошибок при добавлении или изменении данных и повышают качество и надежность работы с базой >данных.