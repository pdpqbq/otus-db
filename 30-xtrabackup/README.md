Востановить таблицу из бэкапа xtrabackup

Цель:  
Восстановить таблицу otus.articles из сжатого и шифрованного бэкапа

Файл бэкапа backup.xbstream.gz.des3 был выполнен с помощью команды xtrabackup --databases='otus' --backup --stream=xbstream | gzip - | openssl des3 -salt -k "password" > backup.xbstream.gz.des3  
Дамп структуры базы otus - otus-db.dmp

Vagrantfile создает ВМ с MySQL и xtrabackup и копирует исходные файлы в каталог /tmp

Установка пароля, создание базы, загрузка структуры
```
mysql -uroot -p`grep 'temporary password' /var/log/mysqld.log | awk '{print $13}'` -e"ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4\!';" --connect-expired-password
mysql -uroot -p'MyNewPass4!' -e"create database otus;"
mysql otus -uroot -p'MyNewPass4!' < /tmp/otus_db-4560-3521f1.dmp

mysql> show tables;
+------------------+
| Tables_in_otus   |
+------------------+
| Python_Employee  |
| articles         |
| bin_test         |
| myset            |
| otus_test        |
| otus_test_myisam |
| python_employee  |
| sbtest1          |
| shirts           |
| t1               |
| t_uuid           |
| time_test        |
| time_test2       |
| time_test3       |
+------------------+
14 rows in set (0.01 sec)

mysql> select * from articles;
Empty set (0.00 sec)
```
Отключение тейблспейса
```
# set foreign_key_checks=0;
mysql> alter table articles discard tablespace;
Query OK, 0 rows affected (0.01 sec)
```
Расшифрование, распаковка, извлечение из стрима, копирование бэкапа тейблспейса в каталог БД
```
cd /tmp
openssl des3 -salt -k "password" -d -in /tmp/backup.xbstream.gz-4560-0d8b3a.des3 -out /tmp/backup.xbstream.gz
gzip -d backup.xbstream.gz
mkdir restore
cd restore
xbstream -x < ../backup.xbstream
cp otus/articles.ibd /var/lib/mysql/otus/
chown mysql:mysql /var/lib/mysql/otus/articles.ibd
```
Подключение тейблспейса и провека
```
mysql> alter table articles import tablespace;
Query OK, 0 rows affected, 2 warnings (0.06 sec)

mysql> select * from articles;
+----+-----------------------+------------------------------------------+
| id | title                 | body                                     |
+----+-----------------------+------------------------------------------+
|  1 | MySQL Tutorial        | DBMS stands for DataBase ...             |
|  2 | How To Use MySQL Well | After you went through a ...             |
|  3 | Optimizing MySQL      | In this tutorial we will show ...        |
|  4 | 1001 MySQL Tricks     | 1. Never run mysqld as root. 2. ...      |
|  5 | MySQL vs. YourSQL     | In the following database comparison ... |
|  6 | MySQL Security        | When configured properly, MySQL ...      |
|  7 | Oracle configuration  | Beginning guide                          |
|  8 | RDBMS course          | Trash and hell                           |
|  9 | RDBMS course2         | Trash and hell2                          |
| 10 | RDBMS course3         | Trash and hell3                          |
| 11 | RDBMS course10        | Trash and hell10                         |
+----+-----------------------+------------------------------------------+
11 rows in set (0.00 sec)
```
