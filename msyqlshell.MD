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

dba.configureLocalInstance()            功能同上

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

c.status()     查看集群状态

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



