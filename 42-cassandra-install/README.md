Verify the version of Java installed
```
$ java -version
```
Add the Apache repository of Cassandra to the file /etc/yum.repos.d/cassandra.repo
```
[cassandra]
name=Apache Cassandra
baseurl=https://downloads.apache.org/cassandra/redhat/40x/
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://downloads.apache.org/cassandra/KEYS
```
Update the package index from sources
```
$ sudo yum update
```
Install Cassandra with YUM
```
$ sudo yum install cassandra
```
Start the Cassandra service
```
$ sudo service cassandra start
```
Monitor the progress of the startup
```
$ tail -f /var/log/cassandra/system.log
```
Check the status of Cassandra  
The status column in the output should report UN which stands for "Up/Normal"
```
$ nodetool status

Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens  Owns (effective)  Host ID                               Rack
UN  127.0.0.1  69.08 KiB  16      100.0%            3d152a33-2572-4daa-8a75-0fe21797ed94  rack1
```
Alternatively, connect to the database with:
```
$ cqlsh
```
