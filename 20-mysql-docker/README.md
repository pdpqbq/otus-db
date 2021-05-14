Создаем базу данных MySQL в докере

Стартовый репозиторий https://github.com/erlong15/otus-mysql-docker

    прописать sql скрипт для создания своей БД в init.sql
    проверить запуск и работу контейнера
    прописать кастомный конфиг - настроить innodb_buffer_pool и другие параметры по желанию
    протестить сисбенчем - результат теста приложить в README

Контейнер запускается на хосте, Vagrant-файл создает ВМ для клиента mysql

Запуск контейнера
```
docker-compose up

[Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.25'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
```

Пересоздание контейнера после изменения конифгов
```
sudo docker-compose rm -f
sudo docker volume rm 16-mysql-docker_data
```

Подключение к базе
```
mysql -h 192.168.33.1 -u root -p12345
#docker-compose exec otusdb mysql -u root -p12345 otus
#mysql -u root -p12345 --port=3309 --protocol=tcp otus
```

Создание базы в стартовом скрипте
```
CREATE database myotus;
USE myotus;
create table t (n int);
insert into t values (1);
```

Вывод значения параметра
```
MySQL [(none)]> SELECT @@innodb_buffer_pool_size;
+---------------------------+
| @@innodb_buffer_pool_size |
+---------------------------+
|                 134217728 |
+---------------------------+

MySQL [(none)]> show variables like 'innodb_buffer_pool_size';
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
```

Изменение параметра через конфиг my.cnf
```
innodb_buffer_pool_size=4294967296
```

Изменение параметра через консоль
```
MySQL [(none)]> SET GLOBAL innodb_buffer_pool_size=4831838208;
Query OK, 0 rows affected, 1 warning (0.00 sec)

MySQL [(none)]> show warnings
    -> ;
+---------+------+-----------------------------------------------------------------+
| Level   | Code | Message                                                         |
+---------+------+-----------------------------------------------------------------+
| Warning | 1292 | Truncated incorrect innodb_buffer_pool_size value: '4831838208' |
+---------+------+-----------------------------------------------------------------+
1 row in set (0.00 sec)

MySQL [(none)]> SET GLOBAL innodb_buffer_pool_size=6442450944;
Query OK, 0 rows affected (0.01 sec)

MySQL [(none)]> show variables like 'innodb_buffer_pool_size';
+-------------------------+------------+
| Variable_name           | Value      |
+-------------------------+------------+
| innodb_buffer_pool_size | 6442450944 |
+-------------------------+------------+
1 row in set (0.00 sec)
```

Тесты
```
sysbench /usr/share/sysbench/oltp_read_write.lua --threads=2 --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --mysql-port=3306 --tables=3 --table-size=100000 prepare

sysbench /usr/share/sysbench/oltp_read_write.lua --threads=2 --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --tables=3 --mysql-port=3306 run

sysbench /usr/share/sysbench/oltp_read_write.lua --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --mysql-port=3306 --tables=3 cleanup
```

InnoDB
```
show table status\G;
*************************** 1. row ***************************
           Name: sbtest1
         Engine: InnoDB
.
.
.

SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES where table_name like 'sbt%';
+------------+--------+
| TABLE_NAME | ENGINE |
+------------+--------+
| sbtest1    | InnoDB |
| sbtest2    | InnoDB |
| sbtest3    | InnoDB |
+------------+--------+
```

```
$ sysbench /usr/share/sysbench/oltp_read_write.lua --threads=2 --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --mysql-port=3306 --tables=3 --table-size=100000 prepare
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Initializing worker threads...

Creating table 'sbtest1'...
Creating table 'sbtest2'...
Inserting 100000 records into 'sbtest1'
Inserting 100000 records into 'sbtest2'
Creating a secondary index on 'sbtest1'...
Creating a secondary index on 'sbtest2'...
Creating table 'sbtest3'...
Inserting 100000 records into 'sbtest3'
Creating a secondary index on 'sbtest3'...

$ sysbench /usr/share/sysbench/oltp_read_write.lua --threads=2 --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --tables=3 --mysql-port=3306 run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 2
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            26152
        write:                           7472
        other:                           3736
        total:                           37360
    transactions:                        1868   (186.66 per sec.)
    queries:                             37360  (3733.16 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0061s
    total number of events:              1868

Latency (ms):
         min:                                    6.53
         avg:                                   10.70
         max:                                  102.58
         95th percentile:                       15.55
         sum:                                19992.90

Threads fairness:
    events (avg/stddev):           934.0000/0.00
    execution time (avg/stddev):   9.9964/0.00
```

MyISAM
```
ALTER TABLE sbtest1 ENGINE = MyISAM;
ALTER TABLE sbtest2 ENGINE = MyISAM;
ALTER TABLE sbtest3 ENGINE = MyISAM;

SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES where table_name like 'sbt%';
+------------+--------+
| TABLE_NAME | ENGINE |
+------------+--------+
| sbtest1    | MyISAM |
| sbtest2    | MyISAM |
| sbtest3    | MyISAM |
+------------+--------+
```

```
$ sysbench /usr/share/sysbench/oltp_read_write.lua --threads=2 --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --tables=3 --mysql-port=3306 run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 2
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            12796
        write:                           3656
        other:                           1828
        total:                           18280
    transactions:                        914    (91.23 per sec.)
    queries:                             18280  (1824.59 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0160s
    total number of events:              914

Latency (ms):
         min:                                   12.44
         avg:                                   21.90
         max:                                  705.51
         95th percentile:                       25.74
         sum:                                20015.97

Threads fairness:
    events (avg/stddev):           457.0000/1.00
    execution time (avg/stddev):   10.0080/0.00
```

BLACKHOLE
```
ALTER TABLE sbtest1 ENGINE = blackhole;
ALTER TABLE sbtest2 ENGINE = blackhole;
ALTER TABLE sbtest3 ENGINE = blackhole;

SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES where table_name like 'sbt%';
+------------+-----------+
| TABLE_NAME | ENGINE    |
+------------+-----------+
| sbtest1    | BLACKHOLE |
| sbtest2    | BLACKHOLE |
| sbtest3    | BLACKHOLE |
+------------+-----------+
```

```
$ sysbench /usr/share/sysbench/oltp_read_write.lua --threads=2 --mysql-host=192.168.33.1 --mysql-db=myotus --mysql-user=root --mysql-password=12345 --tables=3 --mysql-port=3306 run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 2
Initializing random number generator from current time


Initializing worker threads...

Threads started!

SQL statistics:
    queries performed:
        read:                            48706
        write:                           3479
        other:                           17395
        total:                           69580
    transactions:                        3479   (347.68 per sec.)
    queries:                             69580  (6953.66 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          10.0046s
    total number of events:              3479

Latency (ms):
         min:                                    3.63
         avg:                                    5.74
         max:                                  123.08
         95th percentile:                        9.73
         sum:                                19983.51

Threads fairness:
    events (avg/stddev):           1739.5000/2.50
    execution time (avg/stddev):   9.9918/0.00
```
