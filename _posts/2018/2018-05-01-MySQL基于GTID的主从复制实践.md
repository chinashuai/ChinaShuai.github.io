---
layout: post
title: MySQL基于GTID的主从复制实践
category: mysql
tags: [mysql]
excerpt: MySQL基于GTID的主从复制实践
keywords: mysql,gtid,主从复制
---

在开始操作基于GTID的主从复制之前，需要将MySQL的GTID开启

## 一、MySQL 5.7 在线开启和关闭GTID

在MySQL 5.7.6之后，可以在线开启GTID，需要满足两个条件
1、复制拓扑结构中，所有的数据库版本必须大于等于5.7.6
2、 `gtid_mode`必须设置为`OFF`

1、在拓扑结构中**`所有服务器`**运行以下命令：（非常重要）

```mysql
SET @@GLOBAL.ENFORCE_GTID_CONSISTENCY = WARN;

[Note] Changed ENFORCE_GTID_CONSISTENCY from OFF to WARN.
```

开启这个选项时候，让服务器在正常负载下运行一段时间，观察`err log`，如果有发现任何warning，需要通知应用进行调整，直到不出现warning。

2、在拓扑结构中**`所有服务器`**运行以下命令：

```mysql
SET @@GLOBAL.ENFORCE_GTID_CONSISTENCY = ON;

[Note] Changed ENFORCE_GTID_CONSISTENCY from WARN to ON.
```

3、在拓扑结构中**`所有服务器`**运行以下命令，所有服务器必须执行完这一步之后才能执行下一步：

```mysql
SET @@GLOBAL.GTID_MODE = OFF_PERMISSIVE;

[Note] Changed GTID_MODE from OFF to OFF_PERMISSIVE.
```

该命令的效果:服务器不产生GTID，但是能够接受不带GTID的事务，也能够接受带GTID的事务

4、在拓扑结构中**`所有服务器`**运行以下命令

```mysql
SET @@GLOBAL.GTID_MODE = ON_PERMISSIVE;
```

该命令的效果:服务器产生GTID，但是能够接受不带GTID的事务，也能够接受带GTID的事务

5、第四步之后，服务器将不产生匿名的GTID,通过以下命令查看是否还存在匿名的GTID，一定要确保该操作的结果为0（多次验证），才可以进行下一步。

```mysql
SHOW STATUS LIKE 'ONGOING_ANONYMOUS_TRANSACTION_COUNT';

```

![服务器将不产生匿名的GTID](https://upload-images.jianshu.io/upload_images/2710833-55964338d3e9ef86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



6、确保第五步操作之前的所有binlog都已经被其他服务器应用了，因为匿名的GTID必须确保已经复制应用成功，才可以进行下一步操作。

7、如果你需要用binlog来实现一个闪回操作，需要确保你已经不需要那些不包含GTID的binlog了，否则需要等那些binlog过期，才可以进行下一步操作

8、在**`所有的服务器`**上执行：

```mysql
SET @@GLOBAL.GTID_MODE = ON;
```

该命令的效果:服务器产生GTID，且只能接受带GTID的事务。

> ***这里有个很关键的地方：如果master的 `gtid_mode` 为 `on`，那么他的所有slave都只能接受带GTID的事务，所以必须等待所有的slave 都已经接受且应用了不带gtid的事务，才可以将master的 `gtid_mode` 设置为 `on`***

![GTID已经打开](https://upload-images.jianshu.io/upload_images/2710833-b1d8efac48b506d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


9、添加以下内容到 `my.cnf` 中

```
#GTID#
enforce_gtid_consistency       = on
gtid_mode                      = on
```

10、开启slave的auto position，如果你使用的是多源复制，需要针对每个channel启用auto position
```mysql
STOP SLAVE [FOR CHANNEL 'channel'];
CHANGE MASTER TO MASTER_AUTO_POSITION = 1 [FOR CHANNEL 'channel'];
START SLAVE [FOR CHANNEL 'channel'];
```
## 二、在线关闭GTID

在线关闭和在线开启是类似的，只是步奏相反
1、关闭slave的auto position，如果你使用的是多源复制，需要针对每个channel关闭auto position
```mysql
STOP SLAVE [FOR CHANNEL 'channel'];
CHANGE MASTER TO MASTER_AUTO_POSITION = 0, MASTER_LOG_FILE = file, \
MASTER_LOG_POS = position [FOR CHANNEL 'channel'];
START SLAVE [FOR CHANNEL 'channel'];
```
2、在 `所有的服务器` 上执行：
```mysql
SET @@GLOBAL.GTID_MODE = ON_PERMISSIVE;
```
3、在 `所有的服务器` 上执行：
```mysql
SET @@GLOBAL.GTID_MODE = OFF_PERMISSIVE;
```
4、在 `所有的服务器` 上等待 @@GLOBAL.GTID_OWNED 的值是一个空字符串为止。
```mysql
SELECT @@GLOBAL.GTID_OWNED;
```
5、等待第三步操作时候，master上的binlog中的日志都已经被slave应用完毕。

6、如果你需要用binlog来实现一个闪回操作，需要确保你已经不需要那些包含GTID的binlog了，否则需要等那些binlog过期，才可以进行下一步操作

7、在 `所有的服务器` 上执行：
```mysql
SET @@GLOBAL.GTID_MODE = OFF;
```
8、在 `所有的服务器` 上执行：
```mysql
SET @@GLOBAL.GTID_MODE = OFF;
SET @@GLOBAL.ENFORCE_GTID_CONSISTENCY = OFF;
```
9、删除以下配置
```
#GTID#
enforce_gtid_consistency       = on
gtid_mode                      = on
```
## 三、启用 MySQL 并行复制

MySQL 5.7 的并行复制建立在组提交的基础上，所有在主库上能够完成 Prepared 的语句表示没有数据冲突，就可以在 Slave 节点并行复制。

#### 1.配置Master

Master:
```mysql
root@localhost [tpcc]>set global binlog_group_commit_sync_delay=10;

root@localhost [tpcc]>show global variables like '%group_commit%';
+-----------------------------------------+-------+
| Variable_name                           | Value |
+-----------------------------------------+-------+
| binlog_group_commit_sync_delay          | 0  |
| binlog_group_commit_sync_no_delay_count | 10    |
+-----------------------------------------+-------+
2 rows in set (0.00 sec)
```
#### 2.配置slave

Slave:
在配置文件中添加：
```
# slave
slave-parallel-type=LOGICAL_CLOCK
slave-parallel-workers=4
master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log_recovery=ON
```


#### 3.确认配置

```mysql
root@localhost [(none)]>show variables like 'slave_parallel_%';
+------------------------+---------------+
| Variable_name          | Value         |
+------------------------+---------------+
| slave_parallel_type    | LOGICAL_CLOCK |
| slave_parallel_workers | 4             |
+------------------------+---------------+
2 rows in set (0.00 sec)
```
#### 4.配置主从复制关系
master：
先用 `mysqldump` 所有的数据信息，不清楚 `mysqldump` 如何使用的可以查看文章 [主库已有数据时，如何进行主备复制？](https://www.jianshu.com/p/ff4a81bed841)

```shell
# mysqldump -A -B -uroot -p > /tmp/230.sql 
```

slave：

```mysql
$ mysql -uroot -p < /tmp/230.sql 
root@localhost [(none)]>CHANGE MASTER TO 
   MASTER_HOST='192.168.xxx.xxx',
   MASTER_USER='name',
   MASTER_PASSWORD='password',
   MASTER_PORT=3306,
   MASTER_AUTO_POSITION=1;

start slave;Query OK, 0 rows affected, 2 warnings (0.49 sec)

root@localhost [(none)]>
root@localhost [(none)]>start slave;
```
查看复制状态：
```mysql
root@localhost [(none)]>show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.199.230
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 154
               Relay_Log_File: relay-bin.000002
                Relay_Log_Pos: 367
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 568
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 2303306
                  Master_UUID: b5a3240c-8946-11e7-bf07-d067e528dfb8
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```
#### 5.检查 Worker 线程的状态
当前的 Slave 的 SQL 线程为 Coordinator（协调器），执行 Relay log 日志的线程为 Worker(当前的 SQL 线程不仅起到协调器的作用，同时也可以重放 Relay log 中主库提交的事务)。

我们上面设置的线程数是 4 ，从库就能看到 4 个 Coordinator（协调器）进程。
```mysql
root@localhost [(none)]>show processlist;
+----+-------------+-----------+------+---------+------+--------------------------------------------------------+------------------+
| Id | User        | Host      | db   | Command | Time | State                                                  | Info             |
+----+-------------+-----------+------+---------+------+--------------------------------------------------------+------------------+
| 20 | root        | localhost | NULL | Query   |    0 | starting                                               | show processlist |
| 21 | system user |           | NULL | Connect |   19 | Waiting for master to send event                       | NULL             |
| 22 | system user |           | NULL | Connect |   19 | Slave has read all relay log; waiting for more updates | NULL             |
| 23 | system user |           | NULL | Connect |   19 | Waiting for an event from Coordinator                  | NULL             |
| 24 | system user |           | NULL | Connect |   19 | Waiting for an event from Coordinator                  | NULL             |
| 25 | system user |           | NULL | Connect |   19 | Waiting for an event from Coordinator                  | NULL             |
| 26 | system user |           | NULL | Connect |   19 | Waiting for an event from Coordinator                  | NULL             |
+----+-------------+-----------+------+---------+------+--------------------------------------------------------+------------------+
7 rows in set (0.00 sec)
```
#### 6.MySQL 并行复制监控
复制的监控依然可以通过 `SHOW SLAVE STATUS\G` ，但是 MySQL 5.7 在performance_schema 架构下多了以下这些元数据表，用户可以更细力度的进行监控：
```mysql
root@localhost [(none)]>use performance_schema;
Database changed
root@localhost [performance_schema]>show tables like 'replication%';
+---------------------------------------------+
| Tables_in_performance_schema (replication%) |
+---------------------------------------------+
| replication_applier_configuration           |
| replication_applier_status                  |
| replication_applier_status_by_coordinator   |
| replication_applier_status_by_worker        |
| replication_connection_configuration        |
| replication_connection_status               |
| replication_group_member_stats              |
| replication_group_members                   |
+---------------------------------------------+
8 rows in set (0.00 sec)

```






