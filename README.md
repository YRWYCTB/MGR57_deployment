# MGR57_deployment
deployment MGR for mysql5.7 
## 零、环境介绍
### 1、远程mysql-shell节点
172.18.0.150    dzst150

### 2、部署MySQL节点如下
172.18.0.152:3317    dzst152

172.18.0.151:3317    dzst151

172.18.0.160:3317    dzst160

### 3、增加TPC连接用户
create user tian identified by 'passwd';
grant all on *.* to tian with grant option;

### 4、MySQL-shell
5.7中无法通过mysqlshell远程初始化my.cnf，需要在每个实例中均安装mysql-shell

## 一、实例的配置检查
```sh
mysqlsh --log-level=8 --dba-log-sql=2
```
使用上述命令启动mysqlshell后，会在mysqlsh执行文件所在目录生成一个文件，记录mysqlshell详细调用的命令。

```sql
MySQL  172.18.0.151:3317 ssl  JS > \connect tian@172.18.0.151:3317
输入密码：
.
.
输入如下命令检查配置文件是否符合要求

MySQL  172.18.0.151:3317 ssl  JS > dba.checkInstanceConfiguration()
Validating MySQL instance at dzst151:3317 for use in an InnoDB cluster...

This instance reports its own address as dzst151:3317
Clients and other cluster members will communicate with it through this address by default. 
If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
底层调用两个sql语句
1、查询'mysql', 'sys', 'performance_schema', 'information_schema'系统库之外，没有使用Innodb及MEMORY引擎的表。
SELECT table_schema, table_name, engine  FROM information_schema.tables  
WHERE engine NOT IN ('InnoDB', 'MEMORY') 
AND table_schema NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema')
2、查询'mysql', 'sys', 'performance_schema', 'information_schema'系统库之外不创建主键或唯一索引的表
SELECT t.table_schema, t.table_name FROM information_schema.tables t     
LEFT JOIN (SELECT table_schema, table_name                
FROM information_schema.statistics                
GROUP BY table_schema, table_name, index_name                
HAVING SUM(CASE                    
WHEN non_unique = 0 AND nullable != 'YES'                    
THEN 1 ELSE 0 END) = COUNT(*)               
) puks     
ON t.table_schema = puks.table_schema         
AND t.table_name = puks.table_name WHERE puks.table_name IS NULL    
AND t.table_type = 'BASE TABLE'    
AND t.table_schema NOT IN ('mysql', 'sys', 'performance_schema', 'information_schema');

如果查询结果为空将出现如下提示。
No incompatible tables detected

#该部分将检测现存表是否与MGR兼容，如有不兼容会出现如下提示：对不包含主键或唯一索引的表会提示
WARNING: The following tables do not have a Primary Key or equivalent column: 
frc.test233, fs_share.db_data, fs_share.group_data, fs_share.limit_data, fs_share.pay_ivr_log,
fs_share.sip_presence, fs_share.sip_shared_appearance_dialogs, fs_share.sip_shared_appearance_subscriptions,
fs_share.sip_subscriptions, fs_share.tiers, healthkg.tb_top_disease, 
honeycomb_backend.sqllab_backend_20191205T205956, honeycomb_backend.sqllab_backend_20191205T210133, 
honeycomb_product_library.tb_insurance_interface_test, honeycomb_trade.sqllab_honeycomb_trade_20191129T152230, 
honeycomb_trade.tb_people_target_info_201809_test, honeycomb_trade.tb_people_target_info_201810_test, 
nbmp.nbmp2_tb_ranking_list, nbmp.sqllab_nbmp_20191205T210412, nbmp.tb_call_out_task_bak, nbmp.tb_tsr_info_bak, 
pay_trade.pay_cardbin_info, sonar.groups_users, sonar.schema_migrations, superset.bart_lines, 
superset.birth_france_by_region, superset.birth_names, superset.energy_usage, superset.flights, 
superset.long_lat, superset.multiformat_time_series, superset.paris_iris_mapping, 
superset.random_time_series, superset.wb_health_population, supply_platform.ins_mall_clause_20160815, supply_platform.test, 
tomexam3_free.tm_admin_addition, tomexam3_free.tm_paper_link_branch, tomexam3_free.tm_system_function_lib, tomexam3_free.tm_user_addition
#对于不满足上述条件及不适用Innodb引擎的表，在启动组复制之后，这类表将变为只读表。
Group Replication requires tables to use InnoDB and have a PRIMARY KEY or PRIMARY KEY Equivalent (non-null unique key). 
Tables that do not follow these requirements will be readable but not updateable when used with Group Replication. 
If your applications make updates (INSERT, UPDATE or DELETE) to these tables, 
ensure they use the InnoDB storage engine and have a PRIMARY KEY or PRIMARY KEY Equivalent.

在开启组复制的节点上对上述表中数据进行更改，则会出现如下报错，
mysql> update honeycomb_backend.sqllab_backend_20191205T205956 set id =1234 where id =123;

ERROR 3098 (HY000): The table does not comply with the requirements by an external plugin.

检查数据库配置参数是否满足MGR；
Checking instance configuration...
底层在数据中执行如下语句
show GLOBAL variables where `variable_name` in ('innodb_page_size')
show GLOBAL variables where `variable_name` in ('performance_schema')
show GLOBAL variables where `variable_name` in ('server_id')
show GLOBAL variables where `variable_name` in ('log_bin')
show GLOBAL variables where `variable_name` in ('port')
show GLOBAL variables where `variable_name` in ('slave_parallel_workers')
show GLOBAL variables where `variable_name` in ('binlog_format')
show GLOBAL variables where `variable_name` in ('log_slave_updates')
show GLOBAL variables where `variable_name` in ('enforce_gtid_consistency')
show GLOBAL variables where `variable_name` in ('gtid_mode')
show GLOBAL variables where `variable_name` in ('master_info_repository')
show GLOBAL variables where `variable_name` in ('relay_log_info_repository')
show GLOBAL variables where `variable_name` in ('binlog_checksum')
show GLOBAL variables where `variable_name` in ('transaction_write_set_extraction')
show GLOBAL variables where `variable_name` in ('report_port')
show GLOBAL variables where `variable_name` in ('slave_parallel_type')
show GLOBAL variables where `variable_name` in ('slave_preserve_commit_order')
如果参数不满足将出现如下提示：

NOTE: Some configuration options need to be fixed:
+----------------------------------+---------------+----------------+------------------------------------------------+
| Variable                         | Current Value | Required Value | Note                                           |
+----------------------------------+---------------+----------------+------------------------------------------------+
| binlog_checksum                  | CRC32         | NONE           | Update the server variable and the config file |
| enforce_gtid_consistency         | OFF           | ON             | Update the config file and restart the server  |
| gtid_mode                        | OFF           | ON             | Update the config file and restart the server  |
| master_info_repository           | FILE          | TABLE          | Update the config file and restart the server  |
| relay_log_info_repository        | FILE          | TABLE          | Update the config file and restart the server  |
| transaction_write_set_extraction | OFF           | XXHASH64       | Update the config file and restart the server  |
+----------------------------------+---------------+----------------+------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server: an option file is required.
NOTE: Please use the dba.configureInstance() command to repair these issues.

{
    "config_errors": [
        {
            "action": "server_update+config_update", 
            "current": "CRC32", 
            "option": "binlog_checksum", 
            "required": "NONE"
        }, 
        {
            "action": "config_update+restart", 
            "current": "OFF", 
            "option": "enforce_gtid_consistency", 
            "required": "ON"
        }, 
        {
            "action": "config_update+restart", 
            "current": "OFF", 
            "option": "gtid_mode", 
            "required": "ON"
        }, 
        {
            "action": "config_update+restart", 
            "current": "FILE", 
            "option": "master_info_repository", 
            "required": "TABLE"
        }, 
        {
            "action": "config_update+restart", 
            "current": "FILE", 
            "option": "relay_log_info_repository", 
            "required": "TABLE"
        }, 
        {
            "action": "config_update+restart", 
            "current": "OFF", 
            "option": "transaction_write_set_extraction", 
            "required": "XXHASH64"
        }
    ], 
    "status": "error"
}
# 对于MySQL8.0+版本的实例，执行如下命令即可在MySQL实例启动状态更改配置参数，
# 但是对于5.7如果有参数配置不兼容，根据如下提示，手动更改my.cnf中参数后，重启MySQL。
 MySQL  172.18.0.151:3317 ssl  JS > dba.configureInstance()
Configuring MySQL instance at dzst151:3317 for use in an InnoDB cluster...

This instance reports its own address as dzst151:3317
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

NOTE: Some configuration options need to be fixed:
+----------------------------------+---------------+----------------+------------------------------------------------+
| Variable                         | Current Value | Required Value | Note                                           |
+----------------------------------+---------------+----------------+------------------------------------------------+
| binlog_checksum                  | CRC32         | NONE           | Update the server variable and the config file |
| enforce_gtid_consistency         | OFF           | ON             | Update the config file and restart the server  |
| gtid_mode                        | OFF           | ON             | Update the config file and restart the server  |
| master_info_repository           | FILE          | TABLE          | Update the config file and restart the server  |
| relay_log_info_repository        | FILE          | TABLE          | Update the config file and restart the server  |
| transaction_write_set_extraction | OFF           | XXHASH64       | Update the config file and restart the server  |
+----------------------------------+---------------+----------------+------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server: an option file is required.
WARNING: Cannot update configuration file for a remote target instance.
ERROR: Unable to change MySQL configuration.
MySQL server configuration needs to be updated, but neither remote nor local configuration is possible.
Please run this command locally, in the same host as the MySQL server being configured, and pass the path to its configuration file through the mycnfPath option.
Dba.configureInstance: Unable to update configuration (RuntimeErro）
```
5.7 不能使用mysqlshell远程更改MySQL实例配置，动态持久化配置，需要手动将上述配置更新到配置文件，并重启MySQL服务。或者在每个MySQL实例上安装MySQL进行配置
```sql
binlog_checksum         =NONE
enforce_gtid_consistency=ON
gtid_mode                =ON
master_info_repository  = TABLE
relay_log_info_repository= TABLE
transaction_write_set_extraction = XXHASH64
# Disable other storage engines
disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"
```
或者在每个MySQL实例上安装MySQL进行配置，需要指定对应实例配置文件位置，如果参数需要重启生效，会有相应提示
```sql
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
配置完成，再次检查配置：    "status": "ok"表示配置无误。
```sql
MySQL  172.18.0.151:3317 ssl  JS > dba.checkInstanceConfiguration()
Validating MySQL instance at dzst151:3317 for use in an InnoDB cluster...

This instance reports its own address as dzst151:3317
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...
Instance configuration is compatible with InnoDB cluster

The instance 'dzst151:3317' is valid to be used in an InnoDB cluster.

{
    "status": "ok"
}
```
对所有拟加入集群节点进行my.cnf配置检查并更新,具体参数可参考该项目中my.cnf文件

172.18.0.152:3317    dzst152

172.18.0.151:3317    dzst151

172.18.0.160:3317    dzst160

## 二、创建集群；
使用mysqlshell连接到拟作为primary node的实例，执行如下命令
```sql
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
由于MySQL版本问题，不能远程持久化MySQL group replication的参数，需要到目标节点持久配置。

集群创建完成，但是需要至少三个节点以保障单个实例failure的情况下集群可用。

dba.configureLocalInstance()

## 三、5.7增加节点
### 3.1将节点添加到集群中
```sql
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
### 3.2节点增加完成后，需要到新增节点持久化配置
```sql
 MySQL  JS > \connect tian@172.18.0.152:3317
Creating a session to 'tian@172.18.0.152:3317'
Please provide the password for 'tian@172.18.0.152:3317': *******
Save password for 'tian@172.18.0.152:3317'? [Y]es/[N]o/Ne[v]er (default No): Y
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 32
Server version: 5.7.30-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
 MySQL  172.18.0.152:3317 ssl  JS > var cluster =dba.getCluster()
 MySQL  172.18.0.152:3317 ssl  JS > cluster.status()
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "dzst151:3317", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures.", 
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
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "dzst151:3317"
}
 MySQL  172.18.0.152:3317 ssl  JS > dba.configureLocalInstance()
The instance 'dzst152:3317' belongs to an InnoDB cluster.

Detecting the configuration file...
Found configuration file at standard location: /etc/my.cnf
Do you want to modify this file? [y/N]: N
Default file not found at the standard locations.
Please specify the path to the MySQL configuration file: /data/mysql/mysql3317/my3317.cnf
Persisting the cluster settings...
The instance 'dzst152:3317' was configured for use in an InnoDB cluster.

The instance cluster settings were successfully persisted.
 MySQL  172.18.0.152:3317 ssl  JS > 
```
5.7持久化配置，需要在目标节点安装mysql-shell
```sh
[root@dzst160 ~]# /usr/local/mysql-shell/bin/mysqlsh 
Logger: Tried to log to an uninitialized logger.
MySQL Shell 8.0.20

Copyright (c) 2016, 2020, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
```
```sql
 MySQL  JS > \connect tian@172.18.0.160:3317
Creating a session to 'tian@172.18.0.160:3317'
Please provide the password for 'tian@172.18.0.160:3317': *******
Save password for 'tian@172.18.0.160:3317'? [Y]es/[N]o/Ne[v]er (default No): Y
Fetching schema names for autocompletion... Press ^C to stop.
Your MySQL connection id is 25
Server version: 5.7.30-log MySQL Community Server (GPL)
No default schema selected; type \use <schema> to set one.
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

此时配置文件新增如下配置
```sh
cat /data/mysql/mysql3317/my3317.cnf 
auto_increment_increment = 1
auto_increment_offset = 2
group_replication_allow_local_disjoint_gtids_join = OFF
group_replication_allow_local_lower_version_join = OFF
group_replication_auto_increment_increment = 7
group_replication_bootstrap_group = OFF
group_replication_components_stop_timeout = 31536000
group_replication_compression_threshold = 1000000
group_replication_enforce_update_everywhere_checks = OFF
group_replication_exit_state_action = READ_ONLY
group_replication_flow_control_applier_threshold = 25000
group_replication_flow_control_certifier_threshold = 25000
group_replication_flow_control_mode = QUOTA
group_replication_force_members = 
group_replication_group_name = 3ead234a-9bfc-11ea-822e-0242ac120097
group_replication_group_seeds = dzst151:33171,dzst152:33171
group_replication_gtid_assignment_block_size = 1000000
group_replication_ip_whitelist = AUTOMATIC
group_replication_local_address = dzst160:33171
group_replication_member_weight = 50
group_replication_poll_spin_loops = 0
group_replication_recovery_complete_at = TRANSACTIONS_APPLIED
group_replication_recovery_reconnect_interval = 60
group_replication_recovery_retry_count = 10
group_replication_recovery_ssl_ca = 
group_replication_recovery_ssl_capath = 
group_replication_recovery_ssl_cert = 
group_replication_recovery_ssl_cipher = 
group_replication_recovery_ssl_crl = 
group_replication_recovery_ssl_crlpath = 
group_replication_recovery_ssl_key = 
group_replication_recovery_ssl_verify_server_cert = OFF
group_replication_recovery_use_ssl = ON
group_replication_single_primary_mode = ON
group_replication_ssl_mode = REQUIRED
group_replication_start_on_boot = ON
group_replication_transaction_size_limit = 0
group_replication_unreachable_majority_timeout = 0
super_read_only = ON
```
## 五、查看成员状态
### 5.1三个节点均添加完成，查看主节点，使用如下sql
```sql
mysql> select MEMBER_HOST as primary_node ,variable_value 
from performance_schema.global_status  as a 
join performance_schema.replication_group_members  as b 
on a.variable_value=b.MEMBER_ID 
where variable_name="group_replication_primary_member";
+--------------+--------------------------------------+
| primary_node | variable_value                       |
+--------------+--------------------------------------+
| dzst151      | 90d04d68-9bf7-11ea-a947-0242ac120097 |
+--------------+--------------------------------------+
1 row in set (0.01 sec)

mysql> SELECT m.member_id, m.member_state, m.member_host, m.member_port,   
IF(g.primary_uuid = '' OR m.member_id = g.primary_uuid,   'PRIMARY', 'SECONDARY') as member_role,    
NULL as member_version, s.view_id FROM (SELECT IFNULL(variable_value, '') AS primary_uuid       
FROM performance_schema.global_status       
WHERE variable_name = 'group_replication_primary_member') g,      
performance_schema.replication_group_members m LEFT JOIN performance_schema.replication_group_member_stats s   
ON m.member_id = s.member_id     
AND s.channel_name = 'group_replication_applier' ORDER BY m.member_id;
+--------------------------------------+--------------+-------------+-------------+-------------+----------------+---------------------+
| member_id                            | member_state | member_host | member_port | member_role | member_version | view_id             |
+--------------------------------------+--------------+-------------+-------------+-------------+----------------+---------------------+
| 70d5eff5-b220-11ea-87a4-0242ac120097 | ONLINE       | dzst151     |        3336 | SECONDARY   |           NULL | 15928274861597983:3 |
| 74311bb7-b22b-11ea-8cad-0242ac120098 | ONLINE       | dzst152     |        3336 | SECONDARY   |           NULL | NULL                |
| 7af46fc2-b210-11ea-b480-0242ac1200a0 | ONLINE       | dzst160     |        3336 | PRIMARY     |           NULL | NULL                |
+--------------------------------------+--------------+-------------+-------------+-------------+----------------+---------------------+
3 rows in set (0.00 sec)
```
### 5.2查看成员状态
```sql
mysql> select * from performance_schema.replication_group_members;  
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
| group_replication_applier | 05b73559-9bf8-11ea-ace9-0242ac120098 | dzst152     |        3317 | ONLINE       |
| group_replication_applier | 90d04d68-9bf7-11ea-a947-0242ac120097 | dzst151     |        3317 | ONLINE       |
| group_replication_applier | ea0373d6-9bf6-11ea-aab0-0242ac1200a0 | dzst160     |        3317 | ONLINE       |
+---------------------------+--------------------------------------+-------------+-------------+--------------+
3 rows in set (0.00 sec)
移除一个节点
 MySQL  172.18.0.151:3317 ssl  JS > cluster.removeInstance("tian@172.18.0.160:3317")
The instance will be removed from the InnoDB cluster. Depending on the instance
being the Seed or not, the Metadata session might become invalid. If so, please
start a new session to the Metadata Storage R/W instance.

Instance '172.18.0.160:3317' is attempting to leave the cluster...
WARNING: On instance '172.18.0.160:3317' configuration cannot be persisted since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please set the 'group_replication_start_on_boot' variable to 'OFF' in the server configuration file, otherwise it might rejoin the cluster upon restart.
WARNING: Instance 'dzst151:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.
WARNING: Instance 'dzst152:3317' cannot persist configuration since MySQL version 5.7.30 does not support the SET PERSIST command (MySQL version >= 8.0.11 required). Please use the dba.configureLocalInstance() command locally to persist the changes.

The instance '172.18.0.160:3317' was successfully removed from the cluster.
```

### 5.3移除一个实例的原数据，实际操作是将mysql_innodb_cluster_metadata 删除
```sql
MySQL  172.18.0.160:3317 ssl  JS > dba.dropMetadataSchema()
Are you sure you want to remove the Metadata? [y/N]: y
ERROR: The MySQL instance at 'dzst160:3317' currently has the super_read_only system variable set to protect it from inadvertent updates from applications.
You must first unset it to be able to perform any changes to this instance.
For more information see: https://dev.mysql.com/doc/refman/en/server-system-variables.html#sysvar_super_read_only.
You must first unset it to be able to perform any changes to this instance.
For more information see:
https://dev.mysql.com/doc/refman/en/server-system-variables.html#sysvar_super_read_only.

Do you want to disable super_read_only and continue? [y/N]: y

Metadata Schema successfully removed.
```
### 5.4增加节点，对于已经有事务的节点，5.7实例是不支持直接加入的。
```sql
MySQL  172.18.0.151:3317 ssl  JS > cluster.addInstance("tian@172.18.0.160:3317")

WARNING: A GTID set check of the MySQL instance at 'dzst160:3317' determined that it contains transactions that do not originate from the cluster, which must be discarded before it can join the cluster.

dzst160:3317 has the following errant GTIDs that do not exist in the cluster:
ea0373d6-9bf6-11ea-aab0-0242ac1200a0:1-2

WARNING: Discarding these extra GTID events can either be done manually or by completely overwriting the state of dzst160:3317 with a physical snapshot from an existing cluster member. To use this method by default, set the 'recoveryMethod' option to 'clone'.

Having extra GTID events is not expected, and it is recommended to investigate this further and ensure that the data can be removed prior to choosing the clone recovery method.
ERROR: The target instance must be either cloned or fully provisioned before it can be added to the target cluster.
Built-in clone support is available starting with MySQL 8.0.17 and is the recommended method for provisioning instances.
Cluster.addInstance: Instance provisioning required (MYSQLSH 51153)
```
## 6 复制加速参数
在my.cnf中配置如下参数
```sh
cat my.cnf
slave_parallel_type    =LOGICAL_CLOCK
slave_parallel_workers =4
binlog_transaction_dependency_history_size = 25000
binlog_transaction_dependency_tracking =WRITESET
slave_preserve_commit_order           =1
```
MGR中secondary节点应用日志同样适用原复制通道，如上参数，配置基于WRITESET模式的并行复制

增加复制的worker线程：slave_parallel_workers=N 

N为4-8，如果lastcommitted 相等即可并行回放

为了保证回放过程中事务提交顺序和主库一致，需要开启slave_preserve_commit_order参数

5.7.30中如果不开启该参数，可能无法启动组复制

日志中会有如下报错。
[Warning] Plugin group_replication reported: 'Group Replication requires slave-preserve-commit-order to be set to ON when using more than 1 applier threads.'
## 7 MGR优化，针对大事务和从库延时进行优化
```sql
plugin_load_add                ='group_replication.so'          #msyql启动加载组复制插件，否则如下参数无法识别
group_replication_poll_spin_loops                  = 10000
group_replication_transaction_size_limit           = 150000000  #最大可允许的事务大小143M，可以根据实际情况进行调整,事务超过该值将被回滚。
group_replication_compression_threshold            = 1000000    #default 1M message compression 
group_replication_flow_control_applier_threshold   = 25000      #默认25000
group_replication_flow_control_certifier_threshold = 25000      #默认25000
group_replication_flow_control_mode                = QUOTA      #默认QUOTA
auto_increment_increment                           = 3          #集群中有多少个节点就配置成几
```
