Физическая и логическая репликация

Физическая репликация:

    Настроить физическую репликации между двумя кластерами базы данных
    Репликация должна работать использую "слот репликации"
    Реплика должна отставать от мастера на 5 минут

Логическая репликация:

    Создать на первом кластере базу данных, таблицу и наполнить ее данными
    На нем же создать публикацию этой таблицы
    На новом кластере подписаться на эту публикацию
    Убедиться что она среплицировалась
    Добавить записи в эту таблицу на основном сервере и убедиться, что они видны на логической реплике

Стенд собирается в Vagrant на VirtualBox  
Установлен pg12, создан пользователь replicator и сделаны настройки конфигов

master - 192.168.33.10  
stb1 - 192.168.33.11  
stb2 - 192.168.33.12

На всех хостах
```
/var/lib/pgsql/12/data/postgresql.conf
listen_addresses = '*'

create user replicator with replication encrypted password '123456';
```

На master
```
/var/lib/pgsql/12/data/postgresql.conf
recovery_min_apply_delay = 5min # для отложенной репликации

/var/lib/pgsql/12/data/pg_hba.conf
host replication replicator all md5 # для физической репликации
host all replicator all md5 # для логической репликации
```

Физическая репликация

На stb1 очищаем каталог, запускаем репликацию через слот
```
# su - postgres
$ rm -rf /var/lib/pgsql/12/data/*
$ pg_basebackup -h 192.168.33.10 -U replicator -p 5432 -D /var/lib/pgsql/12/data -Fp -P -R --slot slot1 -C
Password:
24475/24475 kB (100%), 1/1 tablespace

# systemctl start postgresql-12.service

$ psql
psql (12.6)
Type "help" for help.

postgres=# \du
                                    List of roles
 Role name  |                         Attributes                         | Member of
------------+------------------------------------------------------------+-----------
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 replicator | Replication                                                | {}

postgres=# select * from pg_stat_replication;
(0 rows)
```

На master создаем таблицу, наполняем данными
```
postgres=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 7392
usesysid         | 16386
usename          | replicator
application_name | walreceiver
client_addr      | 192.168.33.11
client_hostname  |
client_port      | 44198
backend_start    | 2021-05-13 12:58:06.778442+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3000060
write_lsn        | 0/3000060
flush_lsn        | 0/3000060
replay_lsn       | 0/3000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-05-13 13:00:07.004361+00

postgres=# \c postgres
You are now connected to database "postgres" as user "postgres".

postgres=# create table t1 (n int);
CREATE TABLE

postgres=# insert into t1 values (1);
INSERT 0 1

postgres=# select * from pg_stat_replication;
-[ RECORD 1 ]----+------------------------------
pid              | 7392
usesysid         | 16386
usename          | replicator
application_name | walreceiver
client_addr      | 192.168.33.11
client_hostname  |
client_port      | 44198
backend_start    | 2021-05-13 12:58:06.778442+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/3012FD8
write_lsn        | 0/3012FD8
flush_lsn        | 0/3012FD8
replay_lsn       | 0/3012D70
write_lag        | 00:00:00.000457
flush_lag        | 00:00:00.00082
replay_lag       | 00:00:06.087892
sync_priority    | 0
sync_state       | async
reply_time       | 2021-05-13 13:00:41.480124+00

postgres=# select * from pg_replication_slots;
-[ RECORD 1 ]-------+----------
slot_name           | slot1
plugin              |
slot_type           | physical
datoid              |
database            |
temporary           | f
active              | t
active_pid          | 7392
xmin                |
catalog_xmin        |
restart_lsn         | 0/30132D8
confirmed_flush_lsn |

postgres=# select pg_is_in_recovery();
-[ RECORD 1 ]-----+--
pg_is_in_recovery | f
```

На stb1 данные появляются через 5 минут
```
postgres=# select * from t1;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1;
                      ^
.
. 5 минут
.

postgres=# select * from t1;
-[ RECORD 1 ]
n | 1

postgres=# select * from pg_stat_wal_receiver;
-[ RECORD 1 ]---------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 4331
status                | streaming
receive_start_lsn     | 0/3000000
receive_start_tli     | 1
received_lsn          | 0/30132D8
received_tli          | 1
last_msg_send_time    | 2021-05-13 13:17:31.233166+00
last_msg_receipt_time | 2021-05-13 13:17:31.232358+00
latest_end_lsn        | 0/30132D8
latest_end_time       | 2021-05-13 13:07:30.441124+00
slot_name             | slot1
sender_host           | 192.168.33.10
sender_port           | 5432
conninfo              | user=replicator password=******** dbname=replication host=192.168.33.10 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any

postgres=# select pg_is_in_recovery();
-[ RECORD 1 ]-----+--
pg_is_in_recovery | t
```

Логическая репликация

На master меняем параметр wal_level, создаем таблицу, наполняем данными, создаем публикацию
```
sed -i 's/#wal_level = replica/wal_level = logical/' /var/lib/pgsql/12/data/postgresql.conf
systemctl restart postgresql-12

postgres=# create table l1 (l int);
CREATE TABLE

postgres=# grant select on public.l1 to replicator;
GRANT

postgres=# insert into l1 values (1);
INSERT 0 1

postgres=# create publication pub1 for table l1;
CREATE PUBLICATION

postgres=# select * from pg_publication;
  oid  | pubname | pubowner | puballtables | pubinsert | pubupdate | pubdelete | pubtruncate
-------+---------+----------+--------------+-----------+-----------+-----------+-------------
 16404 | pub1    |       10 | f            | t         | t         | t         | t
(1 row)

postgres=# select * from pg_publication_tables;
 pubname | schemaname | tablename
---------+------------+-----------
 pub1    | public     | l1
(1 row)

```

На stb2 создаем таблицу, создаем публикацию, проверяем наличие данных в таблице
```
postgres=# create subscription sub1 connection 'host=192.168.33.10 port=5432 dbname=postgres user=replicator password=123456' publication pub1;
ERROR:  relation "public.l1" does not exist

postgres=# create table l1 (l int);
CREATE TABLE

postgres=# create subscription sub1 connection 'host=192.168.33.10 port=5432 dbname=postgres user=replicator password=123456' publication pub1;
NOTICE:  created replication slot "sub1" on publisher
CREATE SUBSCRIPTION

postgres=# select * from l1;
 l
---
 1
(1 row)
```

Добавляем строку на master
```
postgres=# insert into l1 values (2);
INSERT 0 1

postgres=# select * from l1;
 l
---
 1
 2
(2 rows)
```

Проверяем на stb2
```
postgres=# select * from l1;
 l
---
 1
 2
(2 rows)
```
