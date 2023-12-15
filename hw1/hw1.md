# Домашнее задание
## Работа с уровнями изоляции транзакции в PostgreSQL

Подготовительные работы
```sql
-- создаем новую БД
CREATE DATABASE otus;

-- подключаемся
\c otus

-- создаем и наполняем таблицу persons
create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); 

-- отключаем автокоммит
\set AUTOCOMMIT OFF

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
Проверяем, что глобальный уровень изоляции транзакций **read committed**:
```sql
otus=*# show transaction isolation level;
transaction_isolation 
+-----------------------------+
 read committed
(1 row)

```

#Т1
```sql
begin;
insert into persons(first_name, second_name) values('sergey', 'sergeev');

```
#Т2
```sql
select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)

```

- **ВОПРОС**: видите ли вы новую запись и если да то почему?
- **ОТВЕТ**: Новая запись отсутствует, т.к. в рамках T1 не был произведен commit

#Т1
```sql
commit;
```

#Т2
```sql
select * from persons
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

#T2
```sql
commit;
```

- **ВОПРОС**: видите ли вы новую запись и если да то почему?
- **ОТВЕТ**: Новая запись есть, т.к. в рамках T1 был произведен commit

```sql
begin;
alter database otus set default_transaction_isolation to 'repeatable read';
commit;
```

#T1
```sql
begin;
insert into persons(first_name, second_name) values('sveta', 'svetova');
```

#T2
```sql
begin;
select * from persons;

 id | first_name | second_name 
----+-------------+-------------
  1 | ivan         | ivanov
  2 | petr          | petrov
  3 | sergey     | sergeev
(3 rows)
```

- **ВОПРОС**: видите ли вы новую запись и если да то почему?
- **ОТВЕТ**: нет, мы видим состояние таблицы на момент начала T2

#T1
```sql
commit;
```

#T2
```sql
select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

- **ВОПРОС**: видите ли вы новую запись и если да то почему?
- **ОТВЕТ**: нет, мы видим состояние таблицы на момент начала T2

#T2
```sql
commit;
```

#T2
```sql
begin;
select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

- **ВОПРОС**: видите ли вы новую запись и если да то почему?
- **ОТВЕТ**: начинаем новую транзацию на чтение, получаем актуальные изменения с добавленным person sveta, т.к. T1 завершилась с COMMIT