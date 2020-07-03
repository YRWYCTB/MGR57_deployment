## 1、连接到MySQL实例
首次连接需要输入密码，

选择保存密码后，下次连接不再需要输入密码
```sql
MySQL  JS > \connect tian@172.18.0.151:3338
Creating a session to 'tian@172.18.0.151:3338'
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 60
Server version: 8.0.20 MySQL Community Server - GPL
No default schema selected; type \use <schema> to set one.
MySQL  172.18.0.151:3338 ssl  JS >
```

## 2、 dba相关命令分析

### 2.1 管理及创建 innodb cluseter 
dba.checkInstanceConfiguration()        查看当前MySQL实例的配置
```
 MySQL  172.18.0.151:3338 ssl  JS > dba.checkInstanceConfiguration()
Validating MySQL instance at dzst151:3338 for use in an InnoDB cluster...

This instance reports its own address as dzst151:3338
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...
Instance configuration is compatible with InnoDB cluster

The instance 'dzst151:3338' is valid to be used in an InnoDB cluster.

{
    "status": "ok"
}
```
dba.configureInstance()                 再创建innodb cluster前对MySQL参数进行更新，并持久化到配置文件
```
MySQL  172.18.0.140:3317 ssl  JS > dba.configureInstance()
Configuring local MySQL instance listening at port 3317 for use in an InnoDB cluster...

This instance reports its own address as dzst140:3317
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

NOTE: Some configuration options need to be fixed:
+-----------------------------+---------------+----------------+------------------------------------------------+
| Variable                    | Current Value | Required Value | Note                                           |
+-----------------------------+---------------+----------------+------------------------------------------------+
| binlog_checksum             | CRC32         | NONE           | Update the server variable and the config file |
| enforce_gtid_consistency    | OFF           | ON             | Update the config file and restart the server  |
| gtid_mode                   | OFF           | ON             | Update the config file and restart the server  |
| master_info_repository      | FILE          | TABLE          | Update the config file and restart the server  |
| relay_log_info_repository   | FILE          | TABLE          | Update the config file and restart the server  |
| slave_preserve_commit_order | OFF           | ON             | Update the server variable and the config file |
+-----------------------------+---------------+----------------+------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server: an option file is required.

Detecting the configuration file...
Found configuration file at standard location: /etc/my.cnf
Do you want to modify this file? [y/N]: N
Default file not found at the standard locations.
Please specify the path to the MySQL configuration file: /data/mysql/mysql3317/my3317.cnf
Do you want to perform the required configuration changes? [y/n]: y
Configuring instance...
The instance 'dzst140:3317' was configured to be used in an InnoDB cluster.
NOTE: MySQL server needs to be restarted for configuration changes to take effect.
```
dba.configureLocalInstance()            功能同上，将本实例配置持久化配置文件中，一般是再集群增减节点后执行
```
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

dba.dropMetadataSchema()                删除节点中metadata数据
```
 MySQL  172.18.0.151:3338 ssl  JS > dba.dropMetadataSchema()
Are you sure you want to remove the Metadata? [y/N]: y
ERROR: The MySQL instance at 'dzst151:3338' currently has the super_read_only system variable set to protect it from inadvertent updates from applications.
You must first unset it to be able to perform any changes to this instance.
For more information see: https://dev.mysql.com/doc/refman/en/server-system-variables.html#sysvar_super_read_only.
You must first unset it to be able to perform any changes to this instance.
For more information see:
https://dev.mysql.com/doc/refman/en/server-system-variables.html#sysvar_super_read_only.

Do you want to disable super_read_only and continue? [y/N]: y

Metadata Schema successfully removed.
```
dba.createCluster()                     创建innodb cluster 
```
var cluster=dba.createCluster("mycluster57")
A new InnoDB cluster will be created on instance '172.18.0.151:3317'.

Validating instance configuration at 172.18.0.151:3317...

This instance reports its own address as dzst151:3317

Instance configuration is suitable.
NOTE: Group Replication will communicate with other members using 'dzst151:33171'. Use the localAddress option to override.

WARNING: Instance 'dzst151:3317' cannot persist Group Replication configuration 
since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required).
 Please use the dba.configureLocalInstance() command locally to persist the changes.
Creating InnoDB cluster 'mycluster57' on 'dzst151:3317'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.
```
dba.getCluster()                        获取innodb cluster信息 ， var c = dba.getCluster()


### 2.2 管理创建复制集

dba.configureReplicaSetInstance()       在创建复制集前，对MySQL实例参数进行配置

dba.createReplicaSet()                  创建MySQL复制集，（主从模式）

dba.getReplicaSet()                     获取复制集信息 

## 3、innodb cluster 管理命令

首先获取集群信息

var c = dba.getCluster()

对于cluster有如下命令

c.addInstance()增加实例，需要至少传入一个变量
```
MySQL  172.18.0.151:3317 ssl  JS > cluster.addInstance("tian@172.18.0.160:3317")
Please provide the password for 'tian@172.18.0.160:3317': *******
Save password for 'tian@172.18.0.160:3317'? [Y]es/[N]o/Ne[v]er (default No): Y

NOTE: The target instance 'dzst160:3317' has not been pre-provisioned (GTID set is empty). The Shell is unable to decide whether incremental state recovery can correctly provision it.
The safest and most convenient way to provision a new instance is through automatic clone provisioning, which will completely overwrite the state of 'dzst160:3317' with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental state recovery may be safely used if you are sure all updates ever executed in the cluster were done with GTIDs enabled, there are no purged transactions and the new instance contains the same GTID set as the cluster or a subset of it. To use this method by default, set the 'recoveryMethod' option to 'incremental'.


Please select a recovery method [I]ncremental recovery/[A]bort (default Incremental recovery): 
NOTE: Group Replication will communicate with other members using 'dzst160:33171'. Use the localAddress option to override.

Validating instance configuration at 172.18.0.160:3317...

This instance reports its own address as dzst160:3317

Instance configuration is suitable.
WARNING: Instance 'dzst160:3317' cannot persist Group Replication configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
State recovery already finished for 'dzst160:3317'

WARNING: Instance 'dzst151:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
WARNING: Instance 'dzst152:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
The instance '172.18.0.160:3317' was successfully added to the cluster.
```
c.status()     查看集群状态
```
 MySQL  172.18.0.160:3317 ssl  JS > var cluster = dba.getCluster()
 MySQL  172.18.0.160:3317 ssl  JS > cluster.status()
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "dzst151:3317", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "dzst151:3317": {
                "address": "dzst151:3317", 
                "mode": "R/W", 
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
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "dzst151:3317"
}
```
c.removeInstance() 删除集群中某个节点

c.rejoinInstance() 节点意外下线，实例启动但是未能成功加入集群，执行该命令，将节点重新加入

c.dissolve()       将集群解散
```
MySQL  172.18.0.151:3338 ssl  JS > c.dissolve()
The cluster still has the following registered instances:
{
    "clusterName": "cluster3338", 
    "defaultReplicaSet": {
        "name": "default", 
        "topology": [
            {
                "address": "dzst151:3338", 
                "label": "dzst151:3338", 
                "role": "HA"
            }
        ], 
        "topologyMode": "Single-Primary"
    }
}
WARNING: You are about to dissolve the whole cluster and lose the high availability features provided by it. This operation cannot be reverted. All members will be removed from the cluster and replication will be stopped, internal recovery user accounts and the cluster metadata will be dropped. User data will be maintained intact in all instances.

Are you sure you want to dissolve the cluster? [y/N]: y

Instance 'dzst151:3338' is attempting to leave the cluster...

The cluster was successfully dissolved.
Replication was disabled but user data was left intact.
```

c.setPrimaryInstance() 将某个节点提升为主节点，前提该节点为从节点。

c.getName()        获取当前集群名称



