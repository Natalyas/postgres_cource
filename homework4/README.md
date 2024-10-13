# postgres_cource WB 
## Домашняя работа 4
1. Создать таблицу accounts(id integer, amount numeric);
CREATE TABLE IF NOT EXISTS accounts (
    id integer,
    amount numeric
);
2. Добавить несколько записей и подключившись через 2 терминала добиться ситуации взаимоблокировки (deadlock).
INSERT INTO accounts (id,amount) VALUES (1,100), (2,200), (3,300);
первый терминал часть 1
test=# begin;
BEGIN
test=*# UPDATE accounts SET amount = amount + 1 WHERE id = 1;
UPDATE 1
второй терминал часть 1
test=# begin;
BEGIN
test=*# UPDATE accounts SET amount = amount + 1 WHERE id = 2;
UPDATE 1
test=*# UPDATE accounts SET amount = amount + 1 WHERE id = 1;
UPDATE 1


первый терминал часть 2
test=*# UPDATE accounts SET amount = amount + 1 WHERE id = 2;
ERROR:  deadlock detected
DETAIL:  Process 18028 waits for ShareLock on transaction 10936; blocked by process 17965.
Process 17965 waits for ShareLock on transaction 10937; blocked by process 18028.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "accounts"
3. Посмотреть логи и убедиться, что информация о дедлоке туда попала.

2024-10-13 16:48:10.283 MSK [18024] postgres@postgres STATEMENT:  UPDATE accounts SET amount = amount + 1 WHERE id = 1;
2024-10-13 16:49:20.269 MSK [18028] postgres@test ERROR:  deadlock detected
2024-10-13 16:49:20.269 MSK [18028] postgres@test DETAIL:  Process 18028 waits for ShareLock on transaction 10936; blocked by process 17965.
        Process 17965 waits for ShareLock on transaction 10937; blocked by process 18028.
        Process 18028: UPDATE accounts SET amount = amount + 1 WHERE id = 2;
        Process 17965: UPDATE accounts SET amount = amount + 1 WHERE id = 1;