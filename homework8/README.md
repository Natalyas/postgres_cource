# postgres_cource WB 
## Домашняя работа 8
1. Создать таблицу с продажами.
2. Реализовать функцию выбор трети года (1-4 месяц - первая треть, 5-8 - вторая и тд)
а. через case
b. * (бонуса в виде зачета дз не будет) используя математическую операцию (лучше 2+ варианта)
с. предусмотреть NULL на входе
4. Вызвать эту функцию в SELECT из таблицы с продажами, уведиться, что всё отработало
```bash
sudo su postgres
psql
CREATE DATABASE test;
\c test;
CREATE TABLE sales (price int, product text, sale_date date);
CREATE TABLE
INSERT INTO sales (price, product, sale_date)
SELECT
    (random() * 1000)::int AS price,
    substr(md5(random()::text), 1, 10) AS product,
    (CURRENT_DATE - (random() * 365)::int * INTERVAL '1 day')::date AS sale_date
FROM generate_series(1, 10000);
DROP FUNCTION if exists choose_part_year(date);
CREATE FUNCTION choose_part_year(sale_date date) RETURNS INT AS $$
DECLARE
  sale_month INT = DATE_PART('month', sale_date);
begin
   CASE 
      WHEN sale_month <=4 THEN 
         return 1 ; 
      WHEN sale_month <=8  and sale_month > 4 THEN 
         return 2 ; 
      WHEN sale_month >8 THEN 
         return 3 ; 
   END CASE; 
end;
$$ RETURNS NULL ON NULL INPUT
LANGUAGE plpgSQL;
SELECT choose_part_year('2024-01-01'::date);
1
SELECT choose_part_year('2024-05-01'::date);
2
SELECT choose_part_year('2024-09-01'::date);
3
SELECT price,  product, sale_date, choose_part_year(sale_date) FROM sales LIMIT 100;
 price |  product   | sale_date  | choose_part_year
-------+------------+------------+------------------
    67 | abccfa0588 | 2024-05-02 |                2
   989 | df3f6a914c | 2024-01-29 |                1
   935 | 7cd9adcf40 | 2024-02-20 |                1
   485 | b9e90c829e | 2024-10-12 |                3
   639 | 4865a8d7b0 | 2023-11-13 |                3
   279 | 35ec40cb66 | 2024-07-30 |                2
   930 | 7dc75b9136 | 2024-09-24 |                3
   405 | c065b31867 | 2024-09-11 |                3
   465 | 826adfb7b0 | 2024-02-19 |                1
   944 | fb33784bc2 | 2024-09-08 |                3
   959 | 35343204d2 | 2024-06-13 |                2
   596 | 51f8336b0a | 2024-01-06 |                1
   609 | e75c8936fd | 2023-11-10 |                3
   169 | b654dca5b1 | 2024-01-18 |                1
   974 | 8c657b6927 | 2024-01-16 |                1
   900 | c44fce2f98 | 2024-10-23 |                3
   509 | 8f45eb7eea | 2024-03-25 |                1
   831 | 7d6d64497d | 2024-10-01 |                3
   378 | 635f32f9aa | 2023-12-10 |                3
   489 | 4e5051d30e | 2024-05-21 |                2
   632 | 8ee8c65244 | 2024-08-26 |                2
   355 | 444baa68ee | 2024-06-26 |                2
   111 | c2248467be | 2023-12-07 |                3
   748 | f2df623a83 | 2024-03-09 |                1
   300 | 3762ada2b6 | 2024-08-06 |                2
   335 | d828e2b596 | 2024-08-18 |                2
   481 | 33e9146f85 | 2024-03-04 |                1

```
функция корректно определяет трети года