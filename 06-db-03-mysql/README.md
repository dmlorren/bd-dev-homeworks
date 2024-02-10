# Домашнее задание к занятию 3. «MySQL» - Иванов Дмитрий (fops-13)

## Введение

Перед выполнением задания вы можете ознакомиться с 
[дополнительными материалами](https://github.com/netology-code/virt-homeworks/blob/virt-11/additional/README.md).

## Задача 1

Используя Docker, поднимите инстанс MySQL (версию 8). Данные БД сохраните в volume.
Изучите [бэкап БД](https://github.com/netology-code/virt-homeworks/tree/virt-11/06-db-03-mysql/test_data) и 
восстановитесь из него.
Перейдите в управляющую консоль `mysql` внутри контейнера.
Используя команду `\h`, получите список управляющих команд.
Найдите команду для выдачи статуса БД и **приведите в ответе** из её вывода версию сервера БД.
Подключитесь к восстановленной БД и получите список таблиц из этой БД.
**Приведите в ответе** количество записей с `price` > 300.
В следующих заданиях мы будем продолжать работу с этим контейнером.



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
mkdir netology_mysql
cd netology_mysql
``` 

3. Манифест docker-compose.yml
``` 
/home/dmlorren/netology_mysql/docker-compose.yml
version: '3.8'

services:
  mysql:
      image: mysql:8.0
      container_name: mysql
      restart: always
      ports:
        - "3306:3306"
      environment:
        MYSQL_USER: "dmlorren"
        MYSQL_ROOT_PASSWORD: "123456"
      volumes:
        - ".logs:/var/log/mysql"
        - "./backup:/backup"

``` 

4. Проверяем корректность конфигурации и запускаем контейнер:
``` 
docker compose -f docker-compose.yml config 
docker compose up -d
docker ps -a
``` 
![docker](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-03-mysql/img/docker.png)

5. Выкачиваем бэкап БД:
``` 
 wget https://github.com/netology-code/virt-homeworks/blob/virt-11/06-db-03-mysql/test_data/test_dump.sql
``` 

6. Копируем бэкап test_dump.sql в контейнер, создаём БД (пустой шаблон) и восстанавливаём в неё БД из дампа:
``` 
docker cp test_dump.sql mysql://backup
docker exec -it b9bbd73f22b1 bash
mysql -u root -p 
CREATE DATABASE test_db;
exit
mysql -u root -p test_db < backup/test_dump.sql
mysql -u root -p
show databases;
use test_db;
show tables;

На этапе восстановления БД начался сущий ад с ошибкой:

 ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '> 300' at line 1

Всё затянулось ещё на 3 дня, пришлось менять платформу с MacBook M1 с виртуалкой в Parallels Desktop на виндовый компьютер с virtual box, в винде через notepad++ удалять все комментарии в test_dump.sql и через Drag&Drop закидывать обратно в виртуалку, после этого команда по восстановлению бэкапа отработала. 

Я понимаю, что дело скорре всего в движке mysql и возможно стоило поднять в контейнере самую новую версию MySQL, но в задаче чётко указано именно версия 8. Я пробовал редактировать прямо в дебиане через редакторы nano и mcedit, но формат .sql отображался сплошным текстом.
``` 

7. Смотрим, что дамп восстановлен успешно и выводим количество записей с price > 300:
``` 
select price from orders where price > 300;
``` 
![price](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-03-mysql/img/price.png)


8. Посмотрим версию БД:
![version](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-03-mysql/img/version.png)



## Задача 2

Создайте пользователя test в БД c паролем test-pass, используя:

- плагин авторизации mysql_native_password
- срок истечения пароля — 180 дней 
- количество попыток авторизации — 3 
- максимальное количество запросов в час — 100
- аттрибуты пользователя:
    - Фамилия "Pretty"
    - Имя "James".

Предоставьте привелегии пользователю `test` на операции SELECT базы `test_db`.
    
Используя таблицу INFORMATION_SCHEMA.USER_ATTRIBUTES, получите данные по пользователю `test` и 
**приведите в ответе к задаче**.


### Решение задания 2

1. Создаем пользователя test в БД c паролем test-pass, используя плагин авторизации mysql_native_password:
``` 
create user 'test'@'localhost' identified by 'test-pass';
``` 

2. Указываем, что  срок истечения пароля составит 180 дней:
``` 
alter user 'test'@'localhost' password expire interval 180 day;
``` 

3. Далее указываем количество попыток авторизации их будет 3 (при некорректном вводе пароля 3 раза подряд учетная запись пользователя test блокируется на 1 день):
``` 
alter user 'test'@'localhost' failed_login_attempts 3 password_lock_time 1;
``` 

4. И выставляем максимальное количество запросов в час равное 100:
``` 
alter user 'test'@'localhost' with max_queries_per_hour 100;
``` 

5. Выставляем для пользователя test соответствующие атрибуты:
``` 
alter user 'test'@'localhost' attribute '{"fname":"James", "lname":"Pretty"}';
``` 

6. Предоставляем привелегии пользователю test на операции SELECT базы test_db:
``` 
grant select on test_db.* to 'test'@'localhost';
``` 

7. Проверяем его права:
``` 
show grants for 'test'@'localhost';
``` 

8. Используя таблицу INFORMATION_SCHEMA.USER_ATTRIBUTES, получаем данные по пользователю test:
```
select * from information_schema.user_attributes where user = 'test';
```

![test_user](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-03-mysql/img/test_user.png)



## Задача 3

Установите профилирование `SET profiling = 1`.
Изучите вывод профилирования команд `SHOW PROFILES;`.

Исследуйте, какой `engine` используется в таблице БД `test_db` и **приведите в ответе**.

Измените `engine` и **приведите время выполнения и запрос на изменения из профайлера в ответе**:
- на `MyISAM`,
- на `InnoDB`.



### Решение задания 3

1. Устанавливаем профилирование SET profiling = 1 (имеется ввиду, что для управления профилированием должна использоваться переменная сеанса profiling, значение которой по умолчанию равно 0 (OFF), для включения профилирования необходимо установить значение переменной profiling 1 (ON)):
```
set profiling = 1;
```

2. Проводим активность в БД, выполняем запросы, смотрим, что обращение в orders самое затратное. Движок стоит InnoDB.
```
mysql> show profiles;
+----------+------------+----------------------+
| Query_ID | Duration   | Query                |
+----------+------------+----------------------+
|        1 | 0.00066450 | SELECT DATABASE()    |
|        2 | 0.00077075 | set profiling = 1    |
|        3 | 0.00006325 | select from prders   |
|        4 | 0.00059425 | select * from prders |
|        5 | 0.00136325 | select * from orders |
+----------+------------+----------------------+

mysql> show table status;
+--------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| Name   | Engine | Version | Row_format | Rows | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time         | Update_time         | Check_time | Collation          | Checksum | Create_options | Comment |
+--------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| orders | InnoDB |      10 | Dynamic    |    5 |           3276 |       16384 |               0 |            0 |         0 |              6 | 2024-02-10 10:52:07 | 2024-02-10 10:52:07 | NULL       | utf8mb4_0900_ai_ci |     NULL |                |         |
+--------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
1 row in set (0.00 sec)
```

3. Изменим engine на MyISAM и сравним вывод запроса с InnoDB:
```
alter table orders engine = MyISAM;
select price from orders where price > 300;
show profiles;
```

- на движке MyISAM запрос выполнился быстрее и составил 0.00030000, а на InnoBD время выполнения 0.00099950.

![profiles](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-03-mysql/img/profiles.png)



## Задача 4 

Изучите файл `my.cnf` в директории /etc/mysql.

Измените его согласно ТЗ (движок InnoDB):

- скорость IO важнее сохранности данных;
- нужна компрессия таблиц для экономии места на диске;
- размер буффера с незакомиченными транзакциями 1 Мб;
- буффер кеширования 30% от ОЗУ;
- размер файла логов операций 100 Мб.

Приведите в ответе изменённый файл `my.cnf`.



### Решение задания 4

- искомый файл находится здесь:
/var/lib/docker/overlay2/5d6e72f715ceb4fcded6a1cb80cfbc6eaf73e2261a23de9574c77a687c63e300/diff/etc/my.cnf  


innodb_flush_log_at_trx_commit = 0
> Параметр как MySQL будет писать в лог на диске данные о транзакциях и имеет три значения:
> 0 - скорость, сбрасывается на диск один раз в секунду, вне зависимости от транзакций. Скорость записи возрастает, но так же растёт риск потери данных
> 1 - сохранность.
> 2 - универсальный параметр.

innodb_file_format = Barracuda
> Формат файлов Barracuda поддерживает компрессию (ROW_FORMAT=COMPRESSED). Это позволяет экономить место и снижать нагрузку на жесткие диски путём использования сжатия, однако при этом увеличивается нагрузка на процессор

innodb_log_buffer_size = 1M
> Буффера с незакомиченными транзакциями, его размер установлен в  1Мб

innodb_buffer_pool_size = 614M
> Буфер  кэширования, нужно установить в размере 30% от общего ОЗУ в нашей виртуальной машине.

innodb_log_file_size = 100M
> Данный параметр устанавливает размер лога операций. Операции сначала записываются в лог, а потом применяются к данным на диске. Чем больше этот лог, тем быстрее будут работать записи. Размер установлен в 100мб.
>
> ![compress](https://github.com/dmlorren/bd-dev-homeworks/blob/main/06-db-03-mysql/img/compress.png)
