# Домашнее задание
## Установка и настройка PostgteSQL в контейнере Docker

Запустил контейнер с БД в EC2 инстансе aws
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Запустил отдельный контейнер с клиентом и подключился к  pg-server
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
```
Работа с базой данных из клиента **pg-client**
```sql
-- создаем новую БД
CREATE DATABASE otus;

-- подключаемся
\c otus

-- создаем и наполняем таблицу persons
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 

```
Проверяем что данные на месте:
```sql
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```
Удалил контейнер с сервером
```
sudo docker rm -f pg-server
```
Запустил контейнер заново 
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
Попробовал подключиться к БД с рабочей машины
```
psql -p 5432 -U postgres -h ec2-13-49-66-130.eu-north-1.compue.amazonaws.com -d postgres -W
Password: 
psql: error: connection to server at "ec2-13-49-66-130.eu-north-1.compute.amazonaws.com" (13.49.66.130), port 5432 failed: fe_sendauth: no password supplied
```
Решил проблему с подключением следующим образом:
- разрешил inbound траффик на порт 5432 для security group aws
- в pg_hba.conf IPv4 local connections заменен с  127.0.0.1/32 на 0.0.0.0/0

Подключение к БД с рабочей машины прошло, далее убедидся что данные в БД остались, несмотря на удаление контейнера:
```sql
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```
