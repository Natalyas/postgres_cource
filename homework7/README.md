# postgres_cource WB 
## Домашняя работа 7
1. Развернуть ВМ (Linux) с PostgreSQL
2. Залить Тайские перевозки
https://github.com/aeuge/postgres16book/tree/main/database
3. Проверить скорость выполнения сложного запроса (приложен в конце файла скриптов)
4. Навесить индексы на внешние ключ
5. Проверить, помогли ли индексы на внешние ключи ускориться
```bash
sudo su postgres
cd ~ && wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql
psql
\c thai
\timing

WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,  
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
Time: 2908.973 ms (00:02.909)

CREATE INDEX CONCURRENTLY book_seat_fkbus ON book.seat(fkbus);
CREATE INDEX CONCURRENTLY book_tickets_fkfide ON book.tickets(fkride);
CREATE INDEX CONCURRENTLY book_ride_fkschedule ON book.ride(fkschedule);
CREATE INDEX CONCURRENTLY book_schedule_fkroute ON book.schedule(fkroute);
CREATE INDEX CONCURRENTLY book_ride_fkbus ON book.ride(fkbus);
WITH all_place AS (
    SELECT count(s.id) as all_place, s.fkbus as fkbus
    FROM book.seat s
    group by s.fkbus
),
order_place AS (
    SELECT count(t.id) as order_place, t.fkride
    FROM book.tickets t
    group by t.fkride
)
SELECT r.id, r.startdate as depart_date, bs.city || ', ' || bs.name as busstation,
      t.order_place, st.all_place
FROM book.ride r
JOIN book.schedule as s
      on r.fkschedule = s.id
JOIN book.busroute br
      on s.fkroute = br.id
JOIN book.busstation bs
      on br.fkbusstationfrom = bs.id
JOIN order_place t
      on t.fkride = r.id
JOIN all_place st
      on r.fkbus = st.fkbus
GROUP BY r.id, r.startdate, bs.city || ', ' || bs.name, t.order_place,st.all_place
ORDER BY r.startdate
limit 10;
 id | depart_date |       busstation       | order_place | all_place
----+-------------+------------------------+-------------+-----------
  2 | 2000-01-01  | Bankgkok, Suvarnabhumi |        1056 |        40
  3 | 2000-01-01  | Bankgkok, Suvarnabhumi |         996 |        40
  4 | 2000-01-01  | Bankgkok, Eastern      |        1117 |        40
  5 | 2000-01-01  | Bankgkok, Eastern      |        1114 |        40
  6 | 2000-01-01  | Bankgkok, Eastern      |        1044 |        40
  7 | 2000-01-01  | Bankgkok, Chatuchak    |        1088 |        40
  8 | 2000-01-01  | Bankgkok, Chatuchak    |        1016 |        40
  9 | 2000-01-01  | Bankgkok, Chatuchak    |        1066 |        40
 10 | 2000-01-01  | Bankgkok, Suvarnabhumi |        1053 |        40
  1 | 2000-01-01  | Bankgkok, Suvarnabhumi |        1039 |        40
(10 rows)

Time: 1335.242 ms (00:01.335)
```
Создав индексы по вторичному ключу получили заметное ускорение