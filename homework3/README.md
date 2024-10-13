# postgres_cource WB 
## Домашняя работа 3

1.Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1 млн строк
```console
CREATE TABLE my_table (
    text_field TEXT
);

INSERT INTO my_table (text_field)
SELECT md5(random()::text)
FROM generate_series(1, 1000000);
```
2. Посмотреть размер файла с таблицей
```console
 select pg_total_relation_size('my_table');
 pg_total_relation_size
------------------------
               68329472
```
3. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```console
UPDATE my_table SET text_field = text_field || 'a';
UPDATE 1000000
test=# UPDATE my_table SET text_field = text_field || 'b';
UPDATE 1000000
test=# UPDATE my_table SET text_field = text_field || 'c';
UPDATE 1000000
test=# UPDATE my_table SET text_field = text_field || 'd';
UPDATE 1000000
test=# UPDATE 
my_table SET text_field = text_field || 'f';
UPDATE 1000000
```
4. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил
автовакуум
```console
SELECT relname AS table_name, n_dead_tup AS dead_tuples
FROM pg_stat_user_tables
WHERE relname = 'my_table';
 table_name | dead_tuples
------------+-------------
 my_table   |           0

SELECT relname AS table_name, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'my_table';
 table_name |        last_autovacuum
------------+-------------------------------
 my_table   | 2024-10-13 15:53:47.749629+03
```
5. Подождать некоторое время, проверяя, пришел ли автовакуум
автовакуум приходил  после обновления 
```console
2024-10-13 15:53:47.749629+03
```
6. 5 раз обновить все строчки и добавить к каждой строчке любой символ
```console
 CREATE OR REPLACE FUNCTION update_text_field_random()
RETURNS void AS $$
DECLARE
    i INTEGER;
    random_char CHAR;
BEGIN
    FOR i IN 1..5 LOOP
        Select substr(md5(random()::text), 1, 1) INTO random_char;
        UPDATE my_table SET text_field = text_field || random_char;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
select update_text_field_random();
```
7. Посмотреть размер файла с таблицей
```console
select pg_total_relation_size('my_table');
 pg_total_relation_size
------------------------
              434642944
```
8. Отключить Автовакуум на конкретной таблице
```console
ALTER TABLE my_table SET (autovacuum_enabled = false);
9. 10 раз обновить все строчки и добавить к каждой строчке любой символ
```console
CREATE OR REPLACE FUNCTION update_text_field_random()
RETURNS void AS $$
DECLARE
    i INTEGER;
    random_char CHAR;
BEGIN
    FOR i IN 1..10 LOOP
        Select substr(md5(random()::text), 1, 1) INTO random_char;
        UPDATE my_table SET text_field = text_field || random_char;
        RAISE NOTICE 'обновляем % раз', i;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
select update_text_field_random();
```
10. Посмотреть размер файла с таблицей
```console
select pg_total_relation_size('my_table');
 pg_total_relation_size
------------------------
             1787150336
```
11. Объясните полученный результат
размер вырос в несколько раз так какво первых каждая строчка после отключения автовакуума повторена по 10 раз и так же вырос размер строк которые мы храним, я на 10 раз больше обновила когда писала функцию, но после этого делала вакуум, и затем снова обновила
```console
SELECT relname AS table_name, n_dead_tup AS dead_tuples
FROM pg_stat_user_tables
WHERE relname = 'my_table';
 table_name | dead_tuples
------------+-------------
 my_table   |    10000000
```
12. Не забудьте включить автовакуум)
```console
ALTER TABLE my_table SET (autovacuum_enabled = true);
```