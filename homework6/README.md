# postgres_cource WB 
## Домашняя работа 6
Развернуть асинхронную реплику (можно использовать 1 ВМ, просто рядом кластер
развернуть и подключиться через localhost):
❖ тестируем производительность по сравнению с сингл инстансом
```bash
sudo su postgres 
cd ~
cat >> /etc/postgresql/16/main/postgresql.conf << EOL
listen_addresses = '127.0.0.1'
EOL

cat >> /etc/postgresql/16/main/pg_hba.conf << EOL
host replication replicator 127.0.0.1/32 scram-sha-256
EOL

pg_createcluster 16 main2

psql -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret#123';"

psql -c "SELECT pg_create_physical_replication_slot('test');"

cat >> ~/.pgpass << EOL
postgres4:5432:*:replicator:secret#123
EOL

chmod 0600 ~/.pgpass

pg_ctlcluster 16 main2 stop
rm -rf /var/lib/postgresql/16/main2
pg_basebackup -h localhost -p 5432 -U replicator -R -S test -D /var/lib/postgresql/16/main2
# В другом терминале делаем
psql -c 'checkpoint;'
# Тест на запись
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 56612
number of failed transactions: 0 (0.000%)
latency average = 1.413 ms
initial connection time = 10.745 ms
tps = 5663.052384 (without initial connection time)
# Тест на чтение
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 243375
number of failed transactions: 0 (0.000%)
latency average = 0.328 ms
initial connection time = 10.711 ms
tps = 24357.870487 (without initial connection time)

pg_ctlcluster 16 main2 start
psql -d thai -c "select pg_is_in_recovery();"
pg_lsclusters
Ver Cluster Port Status          Owner    Data directory               Log file
16  main    5432 online          postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 online,recovery postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log

# Тестирование с репликой на время на чтение
/usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 240986
number of failed transactions: 0 (0.000%)
latency average = 0.331 ms
initial connection time = 16.309 ms
tps = 24136.791645 (without initial connection time)
# Тестирование с репликой на время на запись

postgres@postgres-course-shumshurova:/home/shumshurova.n$ /usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -n -U postgres -p 5432 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload2.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 47570
number of failed transactions: 0 (0.000%)
latency average = 1.681 ms
initial connection time = 11.685 ms
tps = 4759.292075 (without initial connection time)
# тестирование реплики на чтение, почти как  мастер
 /usr/lib/postgresql/16/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -n -U postgres -p 5433 thai
pgbench (16.4 (Debian 16.4-1.pgdg120+2))
transaction type: /var/lib/postgresql/workload.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 4
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 234133
number of failed transactions: 0 (0.000%)
latency average = 0.341 ms
initial connection time = 11.307 ms
tps = 23436.537327 (without initial connection time)
```
Задание со * переделать:
а) под синхронную реплику
б) синхронная реплика + асинхронная каскадно снимаемая с синхронной
```bash
sudo su postgres
psql

ALTER SYSTEM SET synchronous_commit = 'on';
ALTER SYSTEM SET wal_level = replica;
SELECT pg_reload_conf();
# На реплике
psql -p 5433
 ALTER SYSTEM SET primary_conninfo = 'application_name=replica1 host=localhost port=5432 user=replicator';
 \q
pg_ctlcluster 16 replica restart
# мастер
psql
 ALTER SYSTEM SET  synchronous_standby_names = 'first 1 (replica1)';

select * from pg_stat_replication;
   pid   | usesysid |  usename   | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time
+-------------------------------
 3821280 |    25042 | replicator | replica1         | ::1         |                 |       53870 | 2024-10-21 15:52:59.837861+03 |              | streaming | 4/28000148 | 4/28000148 | 4/28000148 | 4/28000148 |           |           |            |             1 | sync       | 2024-10-21 15:56:09.912894+03
```
Таким образом мы получили синхронную реплику
далее строим каскадную репликацию
```bash
# На синхронной реплике выставляем параметры
psql -p 5433

ALTER SYSTEM SET hot_standby = on;
ALTER SYSTEM SET wal_level = replica;
ALTER SYSTEM SET hot_standby_feedback = on;
ALTER SYSTEM SET max_wal_senders = 10;
SELECT * FROM pg_create_physical_replication_slot('cascade');
\q
# Создаем кластер под каскадную репликацию и восстанавливаем из бекапа с синххронной реплики
pg_createcluster 16 cascade

pg_basebackup -h localhost -p 5433 -U replicator -R -S cascade -D /var/lib/postgresql/16/cascade
# Получили асинхронную реплику у синхронной
select * from pg_stat_replication;
   pid   | usesysid |  usename   | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn |    write_lag    |    flush_lag    |   replay_lag    | sync_priority | sync_state |          reply_time
---------+----------+------------+------------------+-------------+-----------------+-------------+-------------------------------+--------------+-----------+------------+------------+------------+------------+-----------------+-----------------+-----------------+---------------+------------+------------------------------
 3823567 |    25042 | replicator | 16/cascade       | ::1         |                 |       54182 | 2024-10-21 16:10:17.457679+03 |              | streaming | 4/28000230 | 4/28000230 | 4/28000230 | 4/28000230 | 00:00:00.000125 | 00:00:00.010558 | 00:00:00.010621 |             0 | async      | 2024-10-21 16:10:17.48413+03
```