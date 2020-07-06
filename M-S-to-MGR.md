# 将主从复制升级为MGR
## 0、资源情况

MySQL版本5.7.30

服务器IP

172.18.0.160

172.18.0.151

172.18.0.152

实例端口为：3317

先将现有集群拆除，移除其配置文件中关于组复制内容，并清理数据库中metedata

## 1、搭建主从复制

160 为 主库
151和152位从库

151和152中执行如下命令
```sql
CHANGE MASTER TO
MASTER_HOST='172.18.0.160',
MASTER_USER='tian',
MASTER_PASSWORD='8085782',
MASTER_PORT=3317,
MASTER_AUTO_POSITION = 1;
```
## 2、使用mysqlshell将主库配置为primary节点。

启动sysbench进行压测

## 3、使用msyqlshell创建集群,160为primary node
### 3.1 将主库配置为主节点，并开启组复制
```sql
MySQL  172.18.0.160:3317 ssl  JS > dba.createCluster("c_3317")
A new InnoDB cluster will be created on instance '172.18.0.160:3317'.

Validating instance configuration at 172.18.0.160:3317...

This instance reports its own address as dzst160:3317

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'dzst160:33171'. Use the localAddress option to override.

WARNING: Instance 'dzst160:3317' cannot persist Group Replication configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
Creating InnoDB cluster 'c_3317' on 'dzst160:3317'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
ne server failure.
```
### 3.2 应用端会出现无法写入数据库的情况

此时压测会中断，根据报错可知：

在使用mysqlshell创建集群时，在启动组复制时，会短暂的将主节点数据库设置为只读状态，启动组复制后将自动关闭只读，允许应用写入主节点。

```sh
Initializing worker threads...

Threads started!

[ 5s ] thds: 2 tps: 155.73 qps: 3118.43 (r/w/o: 2183.64/622.93/311.86) lat (ms,95%): 33.12 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 2 tps: 252.20 qps: 5045.95 (r/w/o: 3532.16/1009.39/504.39) lat (ms,95%): 21.11 err/s: 0.00 reconn/s: 0.00
[ 15s ] thds: 2 tps: 334.78 qps: 6692.32 (r/w/o: 4684.27/1338.50/669.55) lat (ms,95%): 16.12 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 2 tps: 412.81 qps: 8259.43 (r/w/o: 5782.56/1651.25/825.62) lat (ms,95%): 10.46 err/s: 0.00 reconn/s: 0.00
[ 25s ] thds: 2 tps: 455.62 qps: 9110.90 (r/w/o: 6376.35/1823.30/911.25) lat (ms,95%): 8.58 err/s: 0.00 reconn/s: 0.00
FATAL: mysql_drv_query() returned error 1290 (The MySQL server is running with the --super-read-only option so it cannot execute this statement) for query 'UPDATE sbtest8 SET k=k+1 WHERE id=50063'
FATAL: mysql_drv_query() returned error 1290 (The MySQL server is running with the --super-read-only option so it cannot execute this statement) for query 'UPDATE sbtest5 SET k=k+1 WHERE id=49853'
FATAL: `thread_run' function failed: ./sysbench-1.0.20/tests/include/oltp_legacy/oltp.lua:80: db_query() failed
Error in my_thread_global_end(): 2 threads didn't exit
```
### 3.3 sysbench压测重启后，启动组复制后将自动关闭只读，允许应用写入。
```sh
[zhaofeng.tian@l-betadb2.ops.p1 ~]$ sh semic_sysbench.sh run
sysbench 1.0.20 (using bundled LuaJIT 2.1.0-beta2)

Running the test with following options:
Number of threads: 2
Report intermediate results every 5 second(s)
Initializing random number generator from current time


Initializing worker threads...

Threads started!

[ 5s ] thds: 2 tps: 415.81 qps: 8320.62 (r/w/o: 5825.35/1663.24/832.02) lat (ms,95%): 7.43 err/s: 0.00 reconn/s: 0.00
[ 10s ] thds: 2 tps: 427.57 qps: 8553.22 (r/w/o: 5987.40/1710.68/855.14) lat (ms,95%): 6.55 err/s: 0.00 reconn/s: 0.00
[ 15s ] thds: 2 tps: 419.81 qps: 8397.54 (r/w/o: 5877.50/1680.43/839.61) lat (ms,95%): 6.55 err/s: 0.00 reconn/s: 0.00
[ 20s ] thds: 2 tps: 440.61 qps: 8809.62 (r/w/o: 6167.55/1760.84/881.22) lat (ms,95%): 5.88 err/s: 0.00 reconn/s: 0.00
[ 25s ] thds: 2 tps: 434.60 qps: 8689.71 (r/w/o: 6082.13/1738.38/869.19) lat (ms,95%): 6.09 err/s: 0.00 reconn/s: 0.00
[ 30s ] thds: 2 tps: 424.61 qps: 8494.07 (r/w/o: 5946.39/1698.45/849.23) lat (ms,95%): 6.21 err/s: 0.00 reconn/s: 0.00
```

## 4、将其他节点添加到集群中

### 4.1 查看集群中GTID变化
```sql
>show slave status\G
Retrieved_Gtid_Set: e19ada5a-a580-11ea-89f9-0242ac1200a0:1-2278
Executed_Gtid_Set: e19ada5a-a580-11ea-89f9-0242ac1200a0:1-2273
```
此时在从库执行show slave status 将出现新的gtid

```sql
>show slave status\G
Retrieved_Gtid_Set: 70352c00-bf3e-11ea-a766-0242ac1200a0:1-7096,e19ada5a-a580-11ea-89f9-0242ac1200a0:1-9667
Executed_Gtid_Set: 70352c00-bf3e-11ea-a766-0242ac1200a0:1-7096,e19ada5a-a580-11ea-89f9-0242ac1200a0:1-966
```
其中70352c00-bf3e-11ea-a766-0242ac1200a0为group_replication_group_name,70352c00-bf3e-11ea-a766-0242ac1200a0:1-7096为组复制的GTID
### 4.2 执行命令将从库添加到集群中；如果不停止主从复制将会出现如下报错，并提示需要执行'STOP SLAVE;'命令

```sql
 MySQL  172.18.0.160:3317 ssl  JS > c.addInstance("tian@172.18.0.151:3317")
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'dzst151:3317' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.

Incremental state recovery was selected because it seems to be safely usable.

ERROR: Cannot add instance '172.18.0.151:3317' to the cluster because it has asynchronous (master-slave) replication configured and running. Please stop the slave threads by executing the query: 'STOP SLAVE;'
Cluster.addInstance: The instance '172.18.0.151:3317' is running asynchronous (master-slave) replication. (RuntimeError)
```

从库执行stop slave 后再次添加节点；
```sql

MySQL  172.18.0.160:3317 ssl  JS > c.addInstance("tian@172.18.0.151:3317")
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'dzst151:3317' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.

Incremental state recovery was selected because it seems to be safely usable.

NOTE: Group Replication will communicate with other members using 'dzst151:33171'. Use the localAddress option to override.

Validating instance configuration at 172.18.0.151:3317...

This instance reports its own address as dzst151:3317

Instance configuration is suitable.
WARNING: Instance 'dzst151:3317' cannot persist Group Replication configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
State recovery already finished for 'dzst151:3317'

WARNING: Instance 'dzst160:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
The instance '172.18.0.151:3317' was successfully added to the cluster.
```
## 5 查看集群状态
### 5.1 查看此时集群状态，此时集群不能容忍任何节点下线
```sql
 MySQL  172.18.0.160:3317 ssl  JS > c.status()
{
    "clusterName": "c_3317", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "dzst160:3317", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures.", 
        "topology": {
            "dzst151:3317": {
                "address": "dzst151:3317", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "dzst160:3317": {
                "address": "dzst160:3317", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "dzst160:3317"
```
## 6、继续添加152节点
### 6.1 停止原主从复制
```sql
> stop slave;
```
### 6.2 添加节点,在添加节点过程中，不会出现主库不可写入的情况。
```sql
MySQL  172.18.0.160:3317 ssl  JS > c.addInstance("tian@172.18.0.152:3317")
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'dzst152:3317' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.

Incremental state recovery was selected because it seems to be safely usable.

NOTE: Group Replication will communicate with other members using 'dzst152:33171'. Use the localAddress option to override.

Validating instance configuration at 172.18.0.152:3317...

This instance reports its own address as dzst152:3317

Instance configuration is suitable.
WARNING: Instance 'dzst152:3317' cannot persist Group Replication configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Incremental state recovery is now in progress.

* Waiting for distributed recovery to finish...
NOTE: 'dzst152:3317' is being recovered from 'dzst160:3317'
* Distributed recovery has finished

WARNING: Instance 'dzst160:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
WARNING: Instance 'dzst151:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
The instance '172.18.0.152:3317' was successfully added to the cluster.
```
### 6.3 查看状态,此时集群处于可以允许一个节点下线的状态
```sql
 MySQL  172.18.0.160:3317 ssl  JS > c.status()
{
    "clusterName": "c_3317", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "dzst160:3317", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "dzst151:3317": {
                "address": "dzst151:3317", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "dzst152:3317": {
                "address": "dzst152:3317", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }, 
            "dzst160:3317": {
                "address": "dzst160:3317", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "dzst160:3317"
```
## 7、持久化配置参数
### 7.1 各个节点mysqlshell中执行如下命令，将配置持久化
```
 MySQL  JS > \connect tian@172.18.0.160:3317
Creating a session to 'tian@172.18.0.160:3317'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 980345
Server version: 5.7.30-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
 MySQL  172.18.0.160:3317 ssl  JS > dba.configureLocalInstance()
The instance 'dzst160:3317' belongs to an InnoDB cluster.

Detecting the configuration file...
Found configuration file at standard location: /etc/my.cnf
Do you want to modify this file? [y/N]: N
Default file not found at the standard locations.
Please specify the path to the MySQL configuration file: /data/mysql/mysql3317/my3317.cnf
Persisting the cluster settings...
The instance 'dzst160:3317' was configured for use in an InnoDB cluster.

The instance cluster settings were successfully persisted.
```


