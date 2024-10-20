# postgres_cource WB 
## Домашняя работа 5
1. Протестировать падение производительности при использовании
pgbouncer в разных режимах: statement, transaction, session
```bash
sudo su postgres
cd ~ && wget https://storage.googleapis.com/thaibus/thai_small.tar.gz && tar -xf thai_small.tar.gz && psql < thai.sql

cat > ~/workload.sql << EOL
\set r random(1, 5000000)
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
EOL

#  Тестируем без прослойки pg_bouncer через сокет

/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres thai
# Результат initial connection time = 11.072 ms

sudo su postgres
echo "listen_addresses = '*'" | sudo tee -a /etc/postgresql/16/main/postgresql.conf
exit
```
Проверяем скорость постгрес через localhost
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432 -h localhost thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 166569
number of failed transactions: 0 (0.000%)
latency average = 0.477 ms
initial connection time = 72.818 ms
tps = 16773.082108 (without initial connection time)

Дальше ставим Pg_bouncer
pool_mode = statement
```bash
sudo DEBIAN_FRONTEND=noninteractive apt install -y pgbouncer

sudo systemctl status pgbouncer

sudo systemctl stop pgbouncer

cat > temp.cfg << EOF 
[databases]
thai = host=127.0.0.1 port=5432 dbname=thai
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
#auth_type = md5
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
admin_users = admindb
pool_mode = statement
max_client_conn = 1000
default_pool_size = 200
EOF
cat temp.cfg | sudo tee -a /etc/pgbouncer/pgbouncer.ini
sudo systemctl restart pgbouncer
sudo su postgres
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h localhost thai
Password:
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 130602
number of failed transactions: 0 (0.000%)
latency average = 0.608 ms
initial connection time = 74.284 ms
tps = 13155.772742 (without initial connection time)
exit
```
pool_mode = transaction

```bash
# Меняем файл pgbouncer.ini из под своего пользователя
sudo systemctl stop pgbouncer
sudo nano /etc/pgbouncer/pgbouncer.ini
[databases]
thai = host=127.0.0.1 port=5432 dbname=thai
[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
#auth_type = md5
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
admin_users = admindb
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 200

sudo systemctl start pgbouncer
sudo su postgres
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h localhost thai
Password:
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 120900
number of failed transactions: 0 (0.000%)
latency average = 0.657 ms
initial connection time = 76.313 ms
tps = 12180.514622 (without initial connection time)
```
Меняем pool_mode = session
```bash
sudo systemctl stop pgbouncer
sudo nano /etc/pgbouncer/pgbouncer.ini
# изменяем pool_mode = session
sudo systemctl start pgbouncer

sudo su postgres
cd ~
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 6432 -h localhost thai
Password:
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 119064
number of failed transactions: 0 (0.000%)
latency average = 0.667 ms
initial connection time = 80.852 ms
tps = 12001.534968 (without initial connection time)

```
Выводы самый быстрый режим через сокет, потом через постгрес напрямую
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432  thai
initial connection time = 13.071 ms
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432 -h localhost thai
initial connection time = 62.807 ms
потом на statement
initial connection time = 74.284 ms

затем transaction
initial connection time = 76.313 ms

с pgbouncer в режиме session
initial connection time = 74.467 ms
у меня заметной разницы в режимах нет

