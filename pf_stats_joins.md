## Joins, stats

### Preparatory work
![schema](https://ucarecdn.com/e4669333-8898-434f-b1a5-4fa88b39ae02/)

```

CREATE SCHEMA IF NOT EXISTS otus;

CREATE TABLE IF NOT EXISTS otus.subject (
    subject_id   BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name_subject TEXT
);

INSERT INTO otus.subject(name_subject)
VALUES ('Основы SQL'),
       ('Основы баз данных'),
       ('Физика');

CREATE TABLE IF NOT EXISTS otus.student (
    student_id   BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name_student TEXT
);

INSERT INTO otus.student(name_student)
VALUES ('Баранов Павел'),
       ('Абрамова Катя'),
       ('Семенов Иван'),
       ('Яковлева Галина');

CREATE TABLE IF NOT EXISTS otus.attempt (
    attempt_id   BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    student_id   BIGINT,
    subject_id   BIGINT,
    date_attempt DATE, 
    result       BIGINT,
    CONSTRAINT "FK_attempt_student"
            FOREIGN KEY (student_id) REFERENCES otus.student (student_id) ON DELETE CASCADE,
    CONSTRAINT "FK_attempt_subject"
            FOREIGN KEY (subject_id) REFERENCES otus.subject (subject_id) ON DELETE CASCADE        
);

INSERT INTO otus.attempt(student_id, subject_id, date_attempt, result)
VALUES (1, 2, '2020-03-23',  67),
       (3, 1, '2020-03-23', 100),
       (4, 2, '2020-03-26',   0),
       (1, 1, '2020-04-15',  33),
       (3, 1, '2020-04-15',  67),
       (4, 2, '2020-04-21', 100),
       (3, 1, '2020-05-17',  33);       

CREATE TABLE IF NOT EXISTS otus.question (
    question_id   BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name_question TEXT,
    subject_id    BIGINT,
    CONSTRAINT "FK_question_subject"
            FOREIGN KEY (subject_id) REFERENCES otus.subject (subject_id) ON DELETE CASCADE
);

INSERT INTO otus.question(name_question, subject_id)
VALUES ('Запрос на выборку начинается с ключевого слова:', 1),
       ('Условие, по которому отбираются записи, задаётся после ключевого слова:', 1),
       ('Для сортировки используется:', 1),
       ('Какой запрос выбирает все записи из таблицы student:', 1),
       ('Для внутреннего соединения таблиц используется оператор:', 1),
       ('База данных - это:', 2),
       ('Отношение - это:', 2),
       ('Концептуальная модель используется для', 2),
       ('Какой тип данных не допустим в реляционной таблице?', 2);

CREATE TABLE IF NOT EXISTS otus.answer (
    answer_id   BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name_answer TEXT,
    question_id BIGINT,
    is_correct  BOOLEAN,
    CONSTRAINT "FK_answer_question"
            FOREIGN KEY (question_id) REFERENCES otus.question
);

INSERT INTO otus.answer(name_answer, question_id, is_correct) 
VALUES ('UPDATE',   1, false),
       ('SELECT',   1, true),
       ('INSERT',   1, false),
       ('GROUP BY', 2, false),
       ('FROM',     2, false),
       ('WHERE',    2, true),
       ('SELECT',   2, false),
       ('SORT',     3, false),
       ('ORDER BY', 3, true),
       ('RANG BY',  3, false),
       ('SELECT * FROM student', 4, true),
       ('SELECT student',        4, false),
       ('INNER JOIN',     5, true),
       ('LEFT JOIN',      5, false),
       ('RIGHT JOIN',     5, false),
       ('CROSS JOIN',     5, false),
       ('совокупность данных, организованных по определённым праилам',                                 6, true),
       ('совокупность программ для хранения и обработки больших массивов информации', 6, false),
       ('строка',  7, false),
       ('столбец', 7, false), 
       ('таблица', 7, true),
       ('обобщенное представление пользователей о данных',   8, true),
       ('описание представления данных в памяти компьютера', 8, false),
       ('база данных', 8, false),
       ('file',    9, true),
       ('INT',     9, false),
       ('VARCHAR', 9, false),
       ('DATE',    9, false);
	   
CREATE TABLE IF NOT EXISTS otus.testing (
    testing_id  BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    attempt_id  BIGINT,
    question_id    BIGINT,
    answer_id   BIGINT,
    CONSTRAINT "FK_testing_attempt" 
            FOREIGN KEY (attempt_id) REFERENCES otus.attempt (attempt_id) ON DELETE CASCADE,
    CONSTRAINT "FK_testing_question"
            FOREIGN KEY (question_id) REFERENCES otus.question (question_id) ON DELETE CASCADE,
    CONSTRAINT "FK_testing_answer" 
            FOREIGN KEY (answer_id) REFERENCES otus.answer (answer_id) ON DELETE CASCADE
);

INSERT INTO otus.testing(attempt_id, question_id, answer_id)
VALUES 
       (1, 7, 19),
       (1, 6, 17),
       (2, 3, 9),
       (2, 1, 2),
       (2, 4, 11),
       (3, 6, 18),
       (3, 8, 24),
       (3, 9, 28),
       (4, 1, 2),
       (4, 5, 16),
       (4, 3, 10),
       (5, 2, 6),
       (5, 1, 2),
       (5, 4, 12),
       (6, 6, 17),
       (6, 8, 22),
       (6, 7, 21),
       (7, 1, 3),
       (7, 4, 11),
       (7, 5, 16);

```

### 1. Implement a direct join between two or more tables
```
-- students who took the “Database Fundamentals” course
SELECT name_student, date_attempt, result
FROM 
    otus.student
    inner join otus.attempt using (student_id)
    inner join otus.subject using (subject_id)
WHERE name_subject = 'Основы баз данных'
ORDER BY result desc;
```
```
                                       QUERY PLAN                                        
-----------------------------------------------------------------------------------------
 Sort  (cost=52.44..52.46 rows=6 width=44)
   Sort Key: attempt.result DESC
   ->  Nested Loop  (cost=25.23..52.37 rows=6 width=44)
         ->  Hash Join  (cost=25.07..51.12 rows=6 width=20)
               Hash Cond: (attempt.subject_id = subject.subject_id)
               ->  Seq Scan on attempt  (cost=0.00..22.70 rows=1270 width=28)
               ->  Hash  (cost=25.00..25.00 rows=6 width=8)
                     ->  Seq Scan on subject  (cost=0.00..25.00 rows=6 width=8)
                           Filter: (name_subject = 'Основы баз данных'::text)
         ->  Index Scan using student_pkey on student  (cost=0.15..0.21 rows=1 width=40)
               Index Cond: (student_id = attempt.student_id)

```

### 2. Implement a left (or right) join on two or more tables
```
-- how many attempts students made in each discipline
SELECT name_subject, count(attempt_id) as Количество, round(avg(result), 2) as Среднее
FROM
    otus.subject
    left join otus.attempt using (subject_id)
GROUP BY name_subject
ORDER BY Среднее desc;
```
```
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Sort  (cost=83.21..83.71 rows=200 width=72)
   Sort Key: (round(avg(attempt.result), 2)) DESC
   ->  HashAggregate  (cost=72.57..75.57 rows=200 width=72)
         Group Key: subject.name_subject
         ->  Hash Right Join  (cost=37.00..63.04 rows=1270 width=48)
               Hash Cond: (attempt.subject_id = subject.subject_id)
               ->  Seq Scan on attempt  (cost=0.00..22.70 rows=1270 width=24)
               ->  Hash  (cost=22.00..22.00 rows=1200 width=40)
                     ->  Seq Scan on subject  (cost=0.00..22.00 rows=1200 width=40)
(9 rows)
```

### 3. Implement a cross join of two or more tables
```
SELECT  name_student, attempt_id, subject_id, date_attempt, result
FROM otus.student 
CROSS JOIN otus.attempt;
```
```
                               QUERY PLAN                               
------------------------------------------------------------------------
 Nested Loop  (cost=0.00..19097.70 rows=1524000 width=60)
   ->  Seq Scan on attempt  (cost=0.00..22.70 rows=1270 width=28)
   ->  Materialize  (cost=0.00..28.00 rows=1200 width=32)
         ->  Seq Scan on student  (cost=0.00..22.00 rows=1200 width=32)
(4 rows)

```
### 4. Implement a full join of two or more tables
```
SELECT name_subject
FROM
    otus.subject
    full join otus.attempt using (subject_id)
WHERE attempt_id is null;
```
```
                               QUERY PLAN                               
------------------------------------------------------------------------
 Hash Full Join  (cost=37.00..63.04 rows=6 width=32)
   Hash Cond: (attempt.subject_id = subject.subject_id)
   Filter: (attempt.attempt_id IS NULL)
   ->  Seq Scan on attempt  (cost=0.00..22.70 rows=1270 width=16)
   ->  Hash  (cost=22.00..22.00 rows=1200 width=40)
         ->  Seq Scan on subject  (cost=0.00..22.00 rows=1200 width=40)
(6 rows)
```
### 5. Implement a query that will use different connection types
```
SELECT name_student, name_subject, date_attempt
FROM otus.subject INNER JOIN otus.attempt USING (subject_id)
             RIGHT JOIN otus.student USING (student_id)
ORDER BY name_student, date_attempt;
```
```
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Sort  (cost=168.86..172.04 rows=1270 width=68)
   Sort Key: student.name_student, attempt.date_attempt
   ->  Hash Right Join  (cost=74.00..103.39 rows=1270 width=68)
         Hash Cond: (attempt.student_id = student.student_id)
         ->  Hash Join  (cost=37.00..63.04 rows=1270 width=44)
               Hash Cond: (attempt.subject_id = subject.subject_id)
               ->  Seq Scan on attempt  (cost=0.00..22.70 rows=1270 width=20)
               ->  Hash  (cost=22.00..22.00 rows=1200 width=40)
                     ->  Seq Scan on subject  (cost=0.00..22.00 rows=1200 width=40)
         ->  Hash  (cost=22.00..22.00 rows=1200 width=40)
               ->  Seq Scan on student  (cost=0.00..22.00 rows=1200 width=40)
(11 rows)
```

**Conclusions:**
- Various types of joins were explored: inner, outer (left, right), full, and cross.
- Simple inner joins (without conditions) use Nested Loop.
- Left join (with grouping and sorting) optimizer uses Hash Join.
- The most complex query, involving cross joins, utilizes Nested Loop, which cannot be optimized further.
- Full join optimizer prefers executing with Hash Join.
- In the query involving different types of joins, the optimizer avoids Nested Loop and uses Hash Join in all cases.
