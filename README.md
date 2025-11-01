# Домашнее задание к занятию «Репликация и масштабирование. Часть 1»

### Задание 1

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

*Ответить в свободной форме.*

### Решение 1

Master - это основной сервер БД, куда поступают все данные. Все изменения в данных, такие как добавление, обновление, удаление происходят на этом сервере. Slave - это вспомогательный сервер БД, который копирует все данные с Мастера. С этого сервера обычно читают данные. Slave-серверов может быть несколько.

Репликации типа master-slave наиболее распространённая схема, в которой есть один главный сервер и один или несколько подчинённых. Как написано выше в этой конфигурации Master (ведущий) — основной сервер, все операции записи (INSERT, UPDATE, DELETE) происходят только на нём. Slave же (ведомый) — копия мастера, он получает все изменения от мастера и обслуживает запросы на чтение. Master-slave обычно используют для увеличения производительности чтения - это плюс такой схемы, к минусам можно отнести - единая точка отказа: все записи идут только на master, если master выходит из строя, система не может выполнять операции записи, пока одна из slave не назначена новым master.

В конфигурации, репликаций типа master-master все серверы равноправны — каждый является одновременно и мастером, и слейвом для другого. Запросы на запись или чтение можно отправлять на любой из серверов, и изменения будут распространены на все остальные. Master-master используется для обеспечения большей отказоустойчивости но при этом требует более тонкой настройки и высокой квалификации инженера для сопровождения.

### Задание 2

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

### Решение 2

В Docker создадим структуру проекта: создаем 2 папки - master и slave. 

В папке master создаем 3 файла - Dockerfile  master.cnf  master.sql с содержимым (берем из презентации):

dockerfile

FROM mysql:8.0
COPY ./master.cnf /etc/mysql/conf.d/my.cnf
COPY ./master.sql /docker-entrypoint-initdb.d/
ENV MYSQL_ROOT_PASSWORD=rootpassword
CMD ["mysqld"]

master.cnf

[mysqld]
server-id = 1
log-bin = mysql-bin
binlog-format = ROW

master.sql

CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

В папке slave создаем 3 файла - Dockerfile  slave.cnf  slave.sql с содержимым (берем из презентации):

Dockerfile

FROM mysql:8.0
COPY ./slave.cnf /etc/mysql/conf.d/my.cnf
COPY ./slave.sql /docker-entrypoint-initdb.d/
ENV MYSQL_ROOT_PASSWORD=rootpassword
CMD ["mysqld"]

slave.cnf

[mysqld]
server-id = 2
read-only = 1

slave.sql

CHANGE REPLICATION SOURCE TO
SOURCE_HOST='mysql_master',
SOURCE_USER='repl',
SOURCE_PASSWORD='slavepass',
SOURCE_SSL=0;
START REPLICA;

Далее создаем Docker сеть:  docker network create replication  

Собираем для мастера:  docker build -t mysql_master ./master

Делаем сборку для слэйва:  docker build -t mysql_slave ./slave

Запускаем контейнеры, после проверяем их работу docker ps

<img width="1667" height="87" alt="image" src="https://github.com/user-attachments/assets/4f2248e2-2829-4870-ac30-804824d03749" />

Подключаемся к мастеру:  mysql -h 127.0.0.1 -P 3306 -u root -prootpassword

В MYSQL создаем БД, таблицу и наполняем тестовыми данными:

CREATE DATABASE test_db;

USE test_db;

CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(50));

INSERT INTO test_table VALUES (1, 'Master Record');

SELECT * FROM test_table;

После подключаемся к слэйву, уже по порту 3307:  mysql -h 127.0.0.1 -P 3307 -u root -prootpassword

Смотрим список БД, выбираем нашу базу test_db и проверяем данные в таблице:

SHOW DATABASES;

USE test_db;

SHOW TABLES;

SELECT * FROM test_table;

После на слэйве проверяем статус: SHOW REPLICA STATUS\G

<img width="1830" height="1077" alt="image" src="https://github.com/user-attachments/assets/191f9bc7-6951-4288-bb7e-2bfd9d5ca295" />


