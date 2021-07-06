Установка MySQL InnoDB Cluster на VM Centos

Настройка ОС
```
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
setenforce 0

cat > /etc/hosts << xxx
192.168.33.10 srv1
192.168.33.11 srv2
192.168.33.12 srv3
xxx
```
Установка MySQL
```
yum -y install https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
yum -y install mysql-community-server mysql-shell
systemctl enable --now mysqld
```
SRV1 SRV2 SRV3
```
mysql -uroot -p`grep 'temporary password' /var/log/mysqld.log | awk '{print $13}'` -e"ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4\!';" --connect-expired-password

mysql -uroot -p'MyNewPass4!'

--SELECT coalesce(@@report_host, @@hostname);

create user 'cadmin'@'%' identified by 'MyNewPass4!';

GRANT ALL PRIVILEGES ON *.* TO 'cadmin'@'%';

GRANT CLONE_ADMIN, CONNECTION_ADMIN, CREATE USER, EXECUTE, FILE, GROUP_REPLICATION_ADMIN, PERSIST_RO_VARIABLES_ADMIN, PROCESS, RELOAD, REPLICATION CLIENT, REPLICATION SLAVE, REPLICATION_APPLIER, REPLICATION_SLAVE_ADMIN, ROLE_ADMIN, SELECT, SHUTDOWN, SYSTEM_VARIABLES_ADMIN ON *.* TO 'cadmin'@'%' WITH GRANT OPTION;
GRANT DELETE, INSERT, UPDATE ON mysql.* TO 'cadmin'@'%' WITH GRANT OPTION;
GRANT ALTER, ALTER ROUTINE, CREATE, CREATE ROUTINE, CREATE TEMPORARY TABLES, CREATE VIEW, DELETE, DROP, EVENT, EXECUTE, INDEX, INSERT, LOCK TABLES, REFERENCES, SHOW VIEW, TRIGGER, UPDATE ON mysql_innodb_cluster_metadata.* TO 'cadmin'@'%' WITH GRANT OPTION;
GRANT ALTER, ALTER ROUTINE, CREATE, CREATE ROUTINE, CREATE TEMPORARY TABLES, CREATE VIEW, DELETE, DROP, EVENT, EXECUTE, INDEX, INSERT, LOCK TABLES, REFERENCES, SHOW VIEW, TRIGGER, UPDATE ON mysql_innodb_cluster_metadata_bkp.* TO 'cadmin'@'%' WITH GRANT OPTION;
GRANT ALTER, ALTER ROUTINE, CREATE, CREATE ROUTINE, CREATE TEMPORARY TABLES, CREATE VIEW, DELETE, DROP, EVENT, EXECUTE, INDEX, INSERT, LOCK TABLES, REFERENCES, SHOW VIEW, TRIGGER, UPDATE ON mysql_innodb_cluster_metadata_previous.* TO 'cadmin'@'%' WITH GRANT OPTION;
```
SRV1
```
mysqlsh
// \connect cadmin@192.168.33.10:3306
// shell.connect('srv1')
// dba.configureInstance('cadmin@srv1:3306')
// dba.checkInstanceConfiguration('cadmin@192.168.33.10:3306',{password:'MyNewPass4!'});
dba.configureInstance('cadmin@192.168.33.10:3306',{password:'MyNewPass4!',interactive:false,restart:true});
```
SRV2
```
mysqlsh
dba.configureInstance('cadmin@192.168.33.11:3306',{password:'MyNewPass4!',interactive:false,restart:true});
```
SRV3
```
mysqlsh
dba.configureInstance('cadmin@192.168.33.12:3306',{password:'MyNewPass4!',interactive:false,restart:true});
```
SRV1
```
mysqlsh
\connect cadmin@192.168.33.10:3306
cluster=dba.createCluster("mycluster");
cluster.status();
cluster.addInstance("cadmin@192.168.33.11:3306",{password:'MyNewPass4!'});
cluster.addInstance("cadmin@192.168.33.12:3306",{password:'MyNewPass4!'});
cluster.status();
```
