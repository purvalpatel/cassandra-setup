Installation on server:
-----------------------
### Basic Installation:
```bash
apt-get update && apt-get upgrade
apt install wget sshpass ttf-mscorefonts-installer dialog s3cmd net-tools openjdk-8-jdk openjdk-11-jdk zip -y
dpkg-reconfigure tzdata
s3cmd --configure
Install Java - 14 
s3cmd get  s3://uffizio-db-backup/Server-setup/jdk-14.0.2_linux-x64_bin.deb
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/jdk-14.0.2/bin/java 1
update-alternatives --config java
```

### Download cassandra
```bash
cd /tmp/ 
s3cmd get s3://uffizio-db-backup/Server-setup/cassandra-setup/apache-cassandra-4.0.1-bin.tar.gz /tmp/
tar xzvf apache-cassandra-4.0.1-bin.tar.gz
mv apache-cassandra-4.0.1 /usr/local/apache-cassandra 
rm apache-cassandra-4.0.1-bin.tar.gz
```

### See cassandra version:
```bash
cd /usr/local/apache-cassandra/bin/
./cassandra -v
Set cassandra parameter in Cassandra 
cp /usr/local/apache-cassandra/conf/cassandra.yaml /usr/local/apache-cassandra/conf/cassandra.yaml-bak

```
### Configure parameters:

Below are the parameters which needs to change in cassandra.yaml

nano /usr/local/apache-cassandra/conf/cassandra.yaml

| Parameter | Value / Default | Notes / Description |
|-----------|----------------|-------------------|
| cluster_name | [change] | e.g. cassandra-cluster. Name should not have spaces. |
| num_tokens | 256 | Max - 256 vnodes. Number of vnodes to assign to the node. |
| allocate_tokens_for_local_replication_factor | [default] | |
| hinted_handoff_enabled | [default] | |
| max_hint_window_in_ms | [default] | |
| hinted_handoff_throttle_in_kb | [change parameter] 4096 | Increase if more bandwidth available |
| max_hints_delivery_threads | [default] | |
| hints_flush_period_in_ms | [default] | |
| max_hints_file_size_in_mb | [default] | |
| authenticator | [change parameter] PasswordAuthenticator | |
| partitioner | org.apache.cassandra.dht.Murmur3Partitioner [default] | |
| prepared_statements_cache_size_mb | [default] | |
| key_cache_save_period | [default] | |
| commitlog_segment_size_in_mb | [default] | |
| seeds | [Change] "172.31.17.60:7000" | Private IP of Cassandra server |
| concurrent_reads | [Change] [16 * number_of_drives] | |
| concurrent_writes | [Change] [8 * number_of_core] | Varies as per configuration |
| file_cache_size_in_mb | [default] | i.e. off-heap memory |
| memtable_heap_space_in_mb | [default] | Keep it commented |
| memtable_offheap_space_in_mb | [Change] 512 | ¼ of the total RAM |
| commitlog_total_space_in_mb | [Change] | Uncomment |
| listen_address | [Change] 172.31.17.60 | Change internal IP |
| native_transport_port | [default] | |
| concurrent_compactors | [Change] [number of cores] | Varies as per configuration |
| compaction_throughput_mb_per_sec | [default] | |
| trickle_fsync | [Change] true | |
| storage_port | [default] 7000 | |
| rpc_address | [change ip] 172.31.17.60 | Change internal IP |
| read_request_timeout_in_ms | [Change] 400000 | Time to wait for response |
| range_request_timeout_in_ms | [Change] 100000 | |
| write_request_timeout_in_ms | [Change] 20000 | |
| counter_write_request_timeout_in_ms | [default] | |
| truncate_request_timeout_in_ms | [default] 60000 | |
| request_timeout_in_ms | [Change] 100000 | |
| slow_query_log_timeout_in_ms | [Change] 50000 | |
| endpoint_snitch | [Change] Ec2Snitch OR SimpleSnitch | AWS server → Ec2Snitch, else default |
| auto_snapshot | [Change] false | |
| authorizer | [Change] org.apache.cassandra.auth.CassandraAuthorizer | |

### Change in cassandra-rackdc.properties (Only When the server is hosted on AWS )
1. Create backup
```
cp /usr/local/apache-cassandra/conf/cassandra-rackdc.properties /usr/local/apache-cassandra/conf/cassandra-rackdc.properties-bak
```
2. Open file.
   
nano /usr/local/apache-cassandra/conf/cassandra-rackdc.properties

Change below variable [If server running on aws]

dc=ap-south
rack=1a

### Changes in configuration, /usr/local/apache-cassandra/conf/jvm11-server.options

nano /usr/local/apache-cassandra/conf/jvm11-server.options

Comment all the below line
```
-XX:+UseConcMarkSweepGC
-XX:+CMSParallelRemarkEnabled
-XX:SurvivorRatio=8
-XX:MaxTenuringThreshold=1
-XX:CMSInitiatingOccupancyFraction=75
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSWaitDuration=10000
-XX:+CMSParallelInitialMarkEnabled
-XX:+CMSEdenChunksRecordAlways
-XX:+CMSClassUnloadingEnabled
```
Now uncomment below lines
```
-XX:+UseG1GC
-XX:+ParallelRefProcEnabled
-XX:G1RSetUpdatingPauseTimePercent=5
-XX:MaxGCPauseMillis=500
-XX:InitiatingHeapOccupancyPercent=70
-XX:ParallelGCThreads=16
-XX:ConcGCThreads=16
```

### Start cassandra
```
screen -S cassandra
cd /usr/local/apache-cassandra/bin/
./cassandra -R
```

### Login cassandra
```
cd /usr/local/apache-cassandra
./cqlsh <Internal IP> -ucassandra -pcassandra 
```

Default password: cassandra
```
cd /usr/local/apache-cassandra/bin
./cqlsh 172.31.17.60 -ucassandra -pcassandra
```
### User management
Change password of cassandra user:
```
ALTER USER cassandra WITH PASSWORD 'Ot32434jhk432';
```

### Create Keyspace (Database):

Note: Change keyspace name and replication factor. If 2 node replication then replication factor should be 2.
```
CREATE KEYSPACE <keyspacename> WITH replication = {'class':'SimpleStrategy', 'replication_factor' : 1};
```

### Create user
```
CREATE USER <username> WITH PASSWORD '<userpassword>’;
GRANT All ON KEYSPACE <Keyspacename>> TO '<username>';
ALTER USER <username> WITH PASSWORD '<userpassword>';
```

### ADD jmx authentication:

Do changes in `/usr/local/apache-cassandra/conf/cassandra-env.sh`

nano /usr/local/apache-cassandra/conf/cassandra-env.sh ( Line no - 228 )
```
LOCAL_JMX=no		[replace yes with no]
```

```
mkdir /etc/cassandra
nano /etc/cassandra/jmxremote.password
Add below variable
monitorRole QED
controlRole R&D
cassandra Cassandrauff!z!0
```

nano /etc/cassandra/jmxremote.access
```
cassandra readwrite
```
nano /usr/lib/jvm/java-11-openjdk-amd64/conf/management/jmxremote.access
```
monitorRole   readonly
controlRole   readwrite \
          	create javax.management.monitor.*,javax.management.timer.* \
          	unregister
cassandra     readwrite
```


## Restart cassandra:
```
# Start Cassandra. before that stop service if already running by killing it.
cd /usr/local/apache-cassandra/bin
screen -S cassandra
./cassandra -R
```

### Now check cassandra node status
```
cd /usr/local/apache-cassandra/bin/
./nodetool -u cassandra -pw 'localcassandra' status
```

