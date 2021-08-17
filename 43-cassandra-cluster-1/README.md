Cassandra кластер из 3 узлов в docker

Запуск кластера - выделяется 6 Гб на контейнер, иначе срабатывает oom-killer, задается 32 токена и первый узел
```
docker network create cas
docker run --name cas1 -d --network cas -m 6144M -e CASSANDRA_NUM_TOKENS=32 cassandra:3
docker run --name cas2 -d --network cas -m 6144M -e CASSANDRA_NUM_TOKENS=32 -e CASSANDRA_SEEDS=cas1 cassandra:3
```
Подождем, пока подключится 2й узел, иначе будет ошибка
```
Exception (java.lang.UnsupportedOperationException) encountered during startup: Other bootstrapping/leaving/moving nodes detected, cannot bootstrap while cassandra.consistent.rangemovement is true. Nodes detected, bootstrapping: /172.18.0.4; leaving: ; moving: ;
```
Запускаем 3й узел
```
docker run --name cas3 -d --network cas -m 6144M -e CASSANDRA_NUM_TOKENS=32 -e CASSANDRA_SEEDS=cas1 cassandra:3
```
Подождем, пока подключится 3й узел
```
# docker exec -it cas1 bash -c "nodetool status"
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address     Load       Tokens       Owns (effective)  Host ID                               Rack
UN  172.18.0.2  71.84 KiB  32           65.5%             c397bb00-40dd-42cf-b196-690e1d60dc4b  rack1
UN  172.18.0.3  71.99 KiB  32           66.7%             5e338403-b1d6-4c61-a8f2-0935bde0214b  rack1
UN  172.18.0.4  71.99 KiB  32           67.9%             5434fe49-b716-40d4-b5e7-e5a2b7831666  rack1
```
Запускаем нагрузочный тест
```
# docker exec -it cas1 bash
root@886ec1b32a68:/# /opt/cassandra/tools/bin/cassandra-stress write n=500000 cl=all -schema replication\(factor=3\)
******************** Stress Settings ********************
Command:
  Type: write
  Count: 500,000
  No Warmup: false
  Consistency Level: ALL
  Target Uncertainty: not applicable
  Key Size (bytes): 10
  Counter Increment Distibution: add=fixed(1)
Rate:
  Auto: false
  Thread Count: 200
  OpsPer Sec: 0
Population:
  Sequence: 1..500000
  Order: ARBITRARY
  Wrap: true
Insert:
  Revisits: Uniform:  min=1,max=1000000
  Visits: Fixed:  key=1
  Row Population Ratio: Ratio: divisor=1.000000;delegate=Fixed:  key=1
  Batch Type: not batching
Columns:
  Max Columns Per Key: 5
  Column Names: [C0, C1, C2, C3, C4]
  Comparator: AsciiType
  Timestamp: null
  Variable Column Count: false
  Slice: false
  Size Distribution: Fixed:  key=34
  Count Distribution: Fixed:  key=5
Errors:
  Ignore: false
  Tries: 10
Log:
  No Summary: false
  No Settings: false
  File: null
  Interval Millis: 1000
  Level: NORMAL
Mode:
  API: JAVA_DRIVER_NATIVE
  Connection Style: CQL_PREPARED
  CQL Version: CQL3
  Protocol Version: V4
  Username: null
  Password: null
  Auth Provide Class: null
  Max Pending Per Connection: 128
  Connections Per Host: 8
  Compression: NONE
Node:
  Nodes: [localhost]
  Is White List: false
  Datacenter: null
Schema:
  Keyspace: keyspace1
  Replication Strategy: org.apache.cassandra.locator.SimpleStrategy
  Replication Strategy Pptions: {replication_factor=3}
  Table Compression: null
  Table Compaction Strategy: null
  Table Compaction Strategy Options: {}
Transport:
  factory=org.apache.cassandra.thrift.TFramedTransportFactory; truststore=null; truststore-password=null; keystore=null; keystore-password=null; ssl-protocol=TLS; ssl-alg=SunX509; store-type=JKS; ssl-ciphers=TLS_RSA_WITH_AES_128_CBC_SHA,TLS_RSA_WITH_AES_256_CBC_SHA;
Port:
  Native Port: 9042
  Thrift Port: 9160
  JMX Port: 7199
Send To Daemon:
  *not set*
Graph:
  File: null
  Revision: unknown
  Title: null
  Operation: WRITE
TokenRange:
  Wrap: false
  Split Factor: 1

Connected to cluster: Test Cluster, max pending requests per connection 128, max connections per host 8
Datatacenter: datacenter1; Host: localhost/127.0.0.1; Rack: rack1
Datatacenter: datacenter1; Host: /172.18.0.4; Rack: rack1
Datatacenter: datacenter1; Host: /172.18.0.3; Rack: rack1
Created keyspaces. Sleeping 1s for propagation.
Sleeping 2s...
Warming up WRITE with 50000 iterations...
Failed to connect over JMX; not collecting these stats
Running WRITE with 200 threads for 500000 iteration
Failed to connect over JMX; not collecting these stats
type       total ops,    op/s,    pk/s,   row/s,    mean,     med,     .95,     .99,    .999,     max,   time,   stderr, errors,  gc: #,  max ms,  sum ms,  sdv ms,      mb
...
total,        500000,   19240,   19240,   19240,    10.4,     7.8,    26.5,    64.2,   103.0,   115.0,   27.7,  0.03387,      0,      0,       0,       0,       0,       0


Results:
Op rate                   :   18,020 op/s  [WRITE: 18,020 op/s]
Partition rate            :   18,020 pk/s  [WRITE: 18,020 pk/s]
Row rate                  :   18,020 row/s [WRITE: 18,020 row/s]
Latency mean              :   10.8 ms [WRITE: 10.8 ms]
Latency median            :    8.0 ms [WRITE: 8.0 ms]
Latency 95th percentile   :   28.6 ms [WRITE: 28.6 ms]
Latency 99th percentile   :   47.6 ms [WRITE: 47.6 ms]
Latency 99.9th percentile :  203.2 ms [WRITE: 203.2 ms]
Latency max               :  357.6 ms [WRITE: 357.6 ms]
Total partitions          :    500,000 [WRITE: 500,000]
Total errors              :          0 [WRITE: 0]
Total GC count            : 0
Total GC memory           : 0.000 KiB
Total GC time             :    0.0 seconds
Avg GC time               :    NaN ms
StdDev GC time            :    0.0 ms
Total operation time      : 00:00:27

END
```
Другие опции
```
docker volume create portainer_data
docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce

docker exec -it cas1 bash -c "nodetool status"
docker exec -it cas4 bash -c "nodetool status"

docker exec -it cas1 bash
docker exec -it cas4 bash

native=`docker port cas1 | grep 9042 | cut -c 21-`
thrift=`docker port cas1 | grep 9160 | cut -c 21-`
jmx=`docker port cas1 | grep 7199 | cut -c 21-`
cassandra-stress write n=1000000 -node 127.0.0.1 -port [ native=$native | thrift=$thrift | jmx=$jmx ]

/opt/cassandra/tools/bin/cassandra-stress write n=100000
/opt/cassandra/tools/bin/cassandra-stress write n=500000 cl=one -schema replication\(factor=2\)
/opt/cassandra/tools/bin/cassandra-stress write n=500000 cl=all -schema replication\(factor=3\)
```
