# Журналы
## Домашнее задание
1. Настройте выполнение контрольной точки раз в 30 секунд.
```
checkpoint_timeout = 30s     
```
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
```
hw6=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 2/75178040         | 2/75178040
(1 row)

hw6=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 47
checkpoints_req       | 5
checkpoint_write_time | 1560897
checkpoint_sync_time  | 471
buffers_checkpoint    | 137381
buffers_clean         | 131926
maxwritten_clean      | 1299
buffers_backend       | 375006
buffers_backend_fsync | 0
buffers_alloc         | 353888
stats_reset           | 2024-12-24 16:26:51.779821+03
```
```
postgres@mgt-annenkova-test-02:~$  pgbench -c8 -T 600 -U postgres hw6
pgbench (15.10 (Ubuntu 15.10-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 928734
number of failed transactions: 0 (0.000%)
latency average = 5.168 ms
initial connection time = 20.173 ms
tps = 1547.882655 (without initial connection time)
```
```
postgres=# select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 2/A0F3CAF0         | 2/A0F3CAF0
(1 row)

postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 74
checkpoints_req       | 5
checkpoint_write_time | 2100037
checkpoint_sync_time  | 593
buffers_checkpoint    | 182725
buffers_clean         | 131926
maxwritten_clean      | 1299
buffers_backend       | 380997
buffers_backend_fsync | 0
buffers_alloc         | 359869
stats_reset           | 2024-12-24 16:26:51.779821+03

postgres=# select pg_size_pretty(('2/A0F3CAF0'::pg_lsn - '2/75178040'::pg_lsn) / 27 ) for_one_checkpoint;
 for_one_checkpoint
--------------------
 26 MB
(1 row)
```
> checkpoints_req не изменился => контрольные точки выполнялись точно по расписанию
5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
```
postgres@mgt-annenkova-test-02:~$  pgbench -c8 -T 600 -U postgres hw6
pgbench (15.10 (Ubuntu 15.10-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2187751
number of failed transactions: 0 (0.000%)
latency average = 2.194 ms
initial connection time = 31.861 ms
tps = 3646.372343 (without initial connection time)
```
> В синхронном режиме каждое подтверждение транзакции ждет записи данных в WAL, что увеличивает задержки
6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?
```
postgres@mgt-annenkova-test-02:~$ pg_createcluster 15 main --start -- -k
Creating new PostgreSQL cluster 15/main ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/main --auth-local peer --auth-host scram-sha-256 --no-instructions -k
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
```
postgres=# CREATE TABLE test_table(id SERIAL PRIMARY KEY, data TEXT);
CREATE TABLE
postgres=# INSERT INTO test_table(data) VALUES ('test1'), ('test2');
INSERT 0 2
```
```
s_annenkova@mgt-annenkova-test-02:~$ sudo systemctl stop postgresql             s_annenkova@mgt-annenkova-test-02:~$ sudo dd if=/dev/zero of=/var/lib/postgresql/15/main/base/5/16398 oflag=dsync conv=notrunc bs=1 count=16
16+0 records in
16+0 records out
16 bytes copied, 0.003793 s, 4.2 kB/s
s_annenkova@mgt-annenkova-test-02:~$ sudo systemctl start postgresql    
```
> При select запросе появляется ошибка, так как не совпадают контрольные суммы. исправить можно с помощью VACUUM FULL
