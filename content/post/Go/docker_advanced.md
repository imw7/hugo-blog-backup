---
title: Docker进阶指南
tags: [Docker, 常用组件和技巧]
categories: [Docker]
abbrlink: docker_advanced
date: 2022-06-25T13:39:22+08:00
toc: true
---

本文是 `Docker` 的进阶篇，通过本文能够更好的对 `Docker` 的高级功能有一个清楚的认识。<!--more-->

## Docker复杂安装详说

### 安装MySQL主从复制

#### 主从复制

##### 主从复制的含义

在 `MySQL` 多服务器的架构中，至少要有一个主节点「`master`」，跟主节点相对的，我们把它叫做从节点「`slave`」。主从复制，就是把主节点的数据复制到一个或者多个从节点。主服务器和从服务器可以在不同的 `IP` 上，通过远程连接来同步数据，这个是异步的过程。

##### 主从复制的形式

1、一主一从/一主多从

​       ![mysql1](https://blog.imw7.com/images/Go/docker/mysql1.png)

2、多主一从

​        ![mysql2](https://blog.imw7.com/images/Go/docker/mysql2.png) 

3、双主复制

​      ![mysql3](https://blog.imw7.com/images/Go/docker/mysql3.png)

4、级联复制

![mysql4](https://blog.imw7.com/images/Go/docker/mysql4.png) 

##### 主从复制的用途

**数据备份**：把数据复制到不同的机器上，以免单台服务器发生故障时数据丢失。

**读写分离**：让主库负责写，从库负责读，从而提高读写的并发度。

**高可用 `HA`**：当节点故障时，自动转移到其他节点，提高可用性。

**扩展**：结合负载的机制，均摊所有的应用访问请求，降低单机 `IO`。

##### 主从复制原理

- **主从复制配置**

1、主库开启 `binlog`，设置 `server-id`

2、在主库创建具有复制权限的用户，允许从库连接

```mysql
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.8.147' IDENTIFIED BY '123456';
mysql> FLUSH PRIVILEGES;
```

3、从库 `/etc/my.cnf` 配置，重启数据库

```ini
server-id=2 
log-bin=mysql-bin 
relay-log=mysql-relay-bin 
read-only=1 
log-slave-updates=1
```

`log-slave-updates` 决定了在从 `binlog` 读取数据时，是否记录 `binlog`，实现双主和级联的关键。

4、在从库执行

```mysql
mysql> stop slave; 
mysql> change master to master_host='192.168.8.146', master_user='repl', master_password='123456', master_log_file='mysql-bin.000001', 
master_log_pos=4; 
mysql> start slave;
```

5、查看同步状态

```mysql
mysql> SHOW SLAVE STATUS \G;
```

以下为正常：

![binlog6](https://blog.imw7.com/images/Go/docker/binlog6.png)

 

主从复制原理这里面涉及到几个线程：

![binlog7](https://blog.imw7.com/images/Go/docker/binlog7.png)

- 1、`slave` 服务器执行 `start slave`，开启主从复制开关， `slave` 服务器的 `IO` 线程请求从 `master` 服务器读取 `binlog`（如果该线程追赶上了主库，会进入睡眠状态）。
- 2、`master` 服务器创建 `Log Dump` 线程，把 `binlog` 发送给 `slave` 服务器。`slave` 服务器把读取到的 `binlog` 日志内容写入中继日志 `relay log`（会记录位置信息，以便下次继续读取）。
- 3、`slave` 服务器的 `SQL` 线程会实时检测 `relay log` 中新增的日志内容，把 `relay log` 解析成 `SQL` 语句，并执行。

#### 主从搭建步骤

1、指定 `MySQL` 版本，新建主服务器容器实例 `3307`：

```bash
docker run -p 3307:3306 --name master --restart always \
-v /data/mysql/master/log:/var/log/mysql \
-v /data/mysql/master/data:/var/lib/mysql \
-v /data/mysql/master/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7
```

使用 `MySQL` 最新版，新建主服务器容器实例 `3309`：

```bash
docker run -p 3309:3306 --name source --restart always \
-v /data/mysql/source/log:/var/log/mysql \
-v /data/mysql/source/data:/var/lib/mysql \
-v /data/mysql/source/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql
```

2、在 `/data/mysql/master/conf` 或 `data/mysql/source/conf` 新建 `my.cnf`：

```bash
cd /data/mysql/master/conf 
# 或 cd /data/mysql/source/conf
vim my.cnf
```

将以下内容写入`my.cnf`文件中：

```ini
[client]
# 设置默认编码为utf8
default_character_set=utf8
[mysqld]
####collation_server和character_set_server两个变量只用来为create database命令提供默认值####
# 服务器字符集
collation_server=utf8_general_ci
# 服务器指定的默认编码格式
character_set_server=utf8
# 设置server_id，同一局域网中需要唯一
server_id=101
# 指定不需要同步的数据库名称
binlog-ignore-db=mysql
# 开启二进制日志功能
log-bin=mysql-bin
# 设置二进制日志使用内存大小（事务）
binlog_cache_size=1M
# 设置使用的二进制日志格式（mixed,statement,row）
binlog_format=mixed
# 二进制日志过期清理时间。默认值为0，表示不自动清理。
expire_logs_days=7
# 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制终端。
# 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
####mysql8.0必须添加以下内容，否则docker启动mysql8.0后会秒退####
# secure_file_priv 为 NULL 时，表示限制mysqld不允许导入或导出。
# secure_file_priv 为 /tmp 时，表示限制mysqld只能在/tmp目录中执行导入导出，其他目录不能执行。
# secure_file_priv 没有值时，表示不限制mysqld在任意目录的导入导出。
secure_file_priv=''
# 身份验证插件
default_authentication_plugin=mysql_native_password
```

**注**：使用 `vim` 粘贴时，需要先按 `i` 进入插入模式，然后再粘贴，否则会有粘贴不全的情况。**不能有错误**，否则重启会失败。

3、修改完配置后重启 `master/source` 实例

```bash
docker restart master
# docker restart source
```

4、进入 `master/source` 容器

```bash
docker exec -it master /bin/bash
# docker exec -it source /bin/bash
mysql -uroot -proot
```

> **注意**：`mysql8.0 `以后的版本，如果通过上面的命令无法进入数据库。可以做如下的操作来修改密码。
>
> 进入 `srouce` 容器之后，直接输入以下命令：
>
> ```bash
> mysql
> ```
>
> 即可进入容器内的 `MySQL` 数据库。然后修改密码：
>
> ```mysql
> #使用mysql库
> mysql> use mysql;
> #清空密码
> mysql> UPDATE user SET authentication_string='' WHERE user='root';
> #修改访问主机以及密码等，设置为所有主机可访问
> mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
> #刷新权限
> mysql> flush privileges;
> ```
>
> 如果修改密码失败，出现了类似以下的错误：
>
> ```bash
> ERROR 1396 (HY000): Operation ALTER USER failed for 'root'@'%'
> ```
>
> 可以通过：
> 
> ```mysql
> mysql> select user, host from user;
> ```
> 
> 查看具体的 `root` 用户对应的 `host`，再根据响应的值做出调整。

5、`master/source` 容器实例内创建数据同步用户

`master`：

```mysql
mysql> CREATE USER 'slave'@'%' IDENTIFIED WITH mysql_native_password BY 'Root@123456';
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* to 'slave'@'%';
```

`source`：

```mysql
mysql> CREATE USER 'replica'@'%' IDENTIFIED WITH mysql_native_password BY 'Root@123456';
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* to 'replica'@'%';
```

6、指定`MySQL`版本，新建从服务器容器实例 `3308`：

```bash
docker run -p 3308:3306 --name slave --restart always \
-v /data/mysql/slave/log:/var/log/mysql \
-v /data/mysql/slave/data:/var/lib/mysql \
-v /data/mysql/slave/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7
```

使用`MySQL`最新版本，新建从服务器容器实例 `3310`：

```bash
docker run -p 3310:3306 --name replica --restart always \
-v /data/mysql/replica/log:/var/log/mysql \
-v /data/mysql/replica/data:/var/lib/mysql \
-v /data/mysql/replica/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql
```

7、在 `/data/mysql/slave/conf` 或 `data/mysql/replica/conf` 新建 `my.cnf`，并写入以下内容：

```ini
[client]
# 设置默认编码为utf8
default_character_set=utf8
[mysqld]
####collation_server和character_set_server两个变量只用来为create database命令提供默认值####
# 服务器字符集
collation_server=utf8_general_ci
# 服务器指定的默认编码格式
character_set_server=utf8
# 设置server_id，同一局域网中需要唯一
server_id=102
# 指定不需要同步的数据库名称
binlog-ignore-db=mysql
# 开启二进制日志功能，以备Slave作为其它数据库实例的Master时使用
log-bin=mysql-slave1-bin
# 设置二进制日志使用内存大小（事务）
binlog_cache_size=1M
# 设置使用的二进制日志格式（mixed,statement,row）
binlog_format=mixed
# 二进制日志过期清理时间。默认值为0，表示不自动清理。
expire_logs_days=7
# 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制终端。
# 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
# relay_log配置中继日志
relay_log=mysql-relay-bin
# log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1
# slave设置为只读（具有super权限的用户除外）
read_only=1
####mysql8.0必须添加以下内容，否则docker启动mysql8.0后会秒退####
# secure_file_priv 为 NULL 时，表示限制mysqld不允许导入或导出。
# secure_file_priv 为 /tmp 时，表示限制mysqld只能在/tmp目录中执行导入导出，其他目录不能执行。
# secure_file_priv 没有值时，表示不限制mysqld在任意目录的导入导出。
secure_file_priv=''
# 身份验证插件
default_authentication_plugin=mysql_native_password
```

8、修改完配置后重启 `slave` 实例

```bash
docker restart slave
```

9、在主数据库中查看主从同步状态

```mysql
mysql> show master status;
```

10、进入 `slave` 容器

```bash
docker exec -it slave /bin/bash
mysql -uroot -proot
```

11、在从数据库中配置主从复制

如果 `mysql` 是 `8.0.23` 之前的版本，执行如下 `SQL`：

```mysql
mysql> CHANGE MASTER TO MASTER_HOST='172.30.108.127', MASTER_USER='slave', MASTER_PASSWORD='Root@123456', MASTER_PORT=3307, MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=617, MASTER_CONNECT_RETRY=30;
```

 `8.0.23` 之后的语法：

```mysql
mysql> CHANGE REPLICATION SOURCE TO SOURCE_HOST='172.30.108.127', SOURCE_USER='replica', SOURCE_PASSWORD='Root@123456', SOURCE_PORT=3310, SOURCE_LOG_FILE='mysql-bin.000001', SOURCE_LOG_POS=617, SOURCE_CONNECT_RETRY=30;
```

主从复制命令参数说明：

`master_host`：主数据库的 `IP` 地址；

`master_port`：主数据库的运行端口；

`master_user`：在主数据库创建的用于同步数据的用户账号；

`master_password`：在主数据库创建的用于同步数据的用户密码；

`master_log_file`：指定从数据库要复制数据的日志文件，通过查看主数据的状态，获取 `File` 参数；

`master_log_pos`：指定从数据库从哪个位置开始复制数据，通过查看主数据的状态，获取 `Position` 参数；

`master_connect_retry`：连接失败重试的时间间隔，单位为秒。

12、在从数据库中查看主从同步状态

```mysql
mysql> show slave status \G;
```

主要看 `Slave_IO_Running` 和 `Slave_SQL_Running`，结果均为 `No`，说明还没开始。

```mysql
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 172.30.108.127
                  Master_User: slave
                  Master_Port: 3307
                Connect_Retry: 30
              Master_Log_File: mall-mysql-bin.000001
          Read_Master_Log_Pos: 617
               Relay_Log_File: mall-mysql-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mall-mysql-bin.000001
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 617
              Relay_Log_Space: 154
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 0
                  Master_UUID: 
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: 
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

> **注意**：如果是 `MySQL8.0` 以上版本，出现类似 `Last_IO_Error: error connecting to > master 'slave@192.168.3.6:3309' - retry-time: 30  retries: 3 message:  Authentication plugin 'caching_sha2_password'  reported error: Authentication requires secure connection.` 错误的解决办法：
> 
> 查看主库：
> 
> ```mysql
> mysql> SELECT plugin FROM `user` where user='root';
> +-----------------------+
> | plugin                |
> +-----------------------+
> | mysql_native_password |
> | caching_sha2_password |
> +-----------------------+
> ```
>
> 原来是主库 `root` 的 `plugin` 是 `caching_sha2_password` 导致连接不上，修改为  `mysql_native_password` 即可解决。
>
> ```mysql
> mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root'; 
> ```
> 
> 其中，`root` 可以改为`slave`， `%` 可以改为 `localhost`，根据实际情况而定。

13、在从数据库中开启主从同步

```mysql
mysql> start replica; #8.0.22之后
mysql> start slave; #8.0.22之前
```

14、查看从数据库状态发现已经同步

```mysql
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.30.108.127
                  Master_User: slave
                  Master_Port: 3307
                Connect_Retry: 30
              Master_Log_File: mall-mysql-bin.000001
          Read_Master_Log_Pos: 617
               Relay_Log_File: mall-mysql-relay-bin.000002
                Relay_Log_Pos: 325
        Relay_Master_Log_File: mall-mysql-bin.000001
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
          Exec_Master_Log_Pos: 617
              Relay_Log_Space: 537
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
             Master_Server_Id: 101
                  Master_UUID: 9fac5357-f698-11ec-8dca-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
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
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)
```

再次查看 `Slave_IO_Running` 和 `Slave_SQL_Running` 的状态，均为 `Yes`。说明已经同步完成。

15、主从复制测试

① 主机新建库-使用库-新建表-插入数据，OK

```mysql
mysql> create database db01;
Query OK, 1 row affected (0.00 sec)

mysql> use db01;
Database changed
mysql> create table t1 (id int, name varchar(20));
Query OK, 0 rows affected (0.04 sec)

mysql> insert into t1 values(1, 'z3');
Query OK, 1 row affected (0.02 sec)

mysql> select * from t1;
+------+------+
| id   | name |
+------+------+
|    1 | z3   |
+------+------+
1 row in set (0.00 sec)
```

② 从机使用库-查看记录，OK

```mysql
mysql> use db01;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from t1;
+------+------+
| id   | name |
+------+------+
|    1 | z3   |
+------+------+
1 row in set (0.00 sec)
```

**扩展**：如果使用 `Navicat` 连接 `mysql ` 时出现 `1045 - Access denied for user ‘root‘@‘localhost‘ (using password: YES)` 错误，可以通过以下方法解决：

- 通过命令行修改用户的密码和加密方式解决

```mysql
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
```

`mysql8.0`的新特性，`caching_sha2_password` 密码加密方式。

- 通过修改配置文件解决

以前版本的 `mysql` 密码加密使用的是 `mysql_native_password`，新添加的用户密码默认使用的 `caching_sha2_password`。如果在以前 `mysql` 基础上升级的，就必须确保用户使用的密码加密使用的是 `mysql_native_password`。

如果还要使用以前的密码加密方式，就修改文件配置文件 `my.cnf` 即可，如下：

```ini
[mysqld]
default_authentication_plugin=mysql_native_password
```

### 安装Redis集群

> cluster(集群)模式-docker版
>
> 哈希槽分区进行亿级数据存储

#### 面试题

1~2亿条数据需要缓存，请问如何设计这个存储案例？

**回答**：单机单台100%不可能，肯定是分布式存储，用 `redis` 如何落地？

上述问题阿里P6~P7工程案例和场景设计类必考题目，一般业界有3种解决方案：

##### 哈希取余分区

![hash_partitioning](https://blog.imw7.com/images/Go/docker/hash_partitioning.jpg)

2亿条记录就是2亿个 `k-v`，单机不行必须要分布式多机，假设有3台机器构成一个集群，用户每次读写操作都是根据公式：`hash(key) % N个机器台数`，计算出哈希值，用来决定数据映射到哪一个节点上。

**优点**：简单粗暴，直接有效，只需要预估好数据规划好节点，例如3台、8台、10台，就能保证一段时间的数据支撑。使用Hash算法让固定的一部分请求落到同一台服务器上，这样每台服务器固定处理一部分请求（并维护这些请求的信息），起到负载均衡+分而治之的作用。

**缺点**：原来规划好的节点，进行扩容或者缩容就比较麻烦了。不管扩容、每次数据变动导致节点有变动，映射关系需要重新进行计算，在服务器个数固定不变时没有问题，如果需要弹性扩容或故障停机的情况下，原来的取模公式就会发生变化：`Hash(key)/3`会变成`Hash(key)/?`。此时地址经过取余运算的结果将发生很大变化，根据公式获取的服务器也会变得不可控。

某个 `redis` 机器宕机了，由于台数数量变化，会导致hash取余全部数据重新洗牌。

##### 一致性哈希算法分区

1、是什么

一致性哈希算法在1997年由麻省理工学院提出，设计目标是解决分布式缓存数据**变动和映射问题**。某个机器宕机了，分母数量改变了，自然取余数就不行了。

2、能干嘛

提出一致性 `Hash` 解决方案。目的是当服务器个数发生变动时，尽量减少影响客户端到服务端的映射关系。

3、**三步骤**

step 1：算法构建一致性哈希环

一致性哈希算法必然有一个 `hash` 函数并按照算法产生 `hash` 值，这个算法的所有可能哈希值会构成一个全量集，这个集合可以成为一个 `hash` 空间 [0,$2^{32}−1$]，这个是一个线性空间，但是在算法钟，我们通过适当的逻辑控制将它首尾相连(0=$2^{32}$)，这样让它逻辑上形成了一个环形空间。

它也是按照使用取模的方法，前面介绍的节点取模法是对节点（服务器）的数量进行取模。而一致性 `Hash` 算法是对$2^{32}$取模，简单来说，**一致性 `Hash` 算法将整个哈希值空间组织成一个虚拟的圆环**，如假设某哈希函数H的值空间为$[0,2^{32}−1]$（即哈希值是一个32位无符号整形），整个哈希环如下图：整个空间**按顺时针方向组织**，圆环的正上方的点代表0，0点右侧的第一个点代表1，以此类推，2、3、4.……直到 $2^{32}−1$，也就是说0点左侧的第一个点代表 $2^{32}−1$，0 和 $2^{32}−1$ 在零点钟方向重合，我们把这个由 $2^{32}$ 个点组成的圆环称为 `Hash` 环。

![hash_ring1](https://blog.imw7.com/images/Go/docker/hash_ring1.jpg)

step 2：服务器 `IP` 节点映射

将集群中各个 `IP` 节点映射到环上的某一个位置。

将各个服务器使用 `Hash` 进行一个哈希，具体可以选择服务器的 `IP` 或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置。假如4个节点`Node A`、`Node B`、`Node C`、`Node D`，经过 `IP` 地址的**哈希函数**计算「`hash(ip)`」，使用 `IP` 地址哈希后在环空间的位置如下：

![hash_ring2](https://blog.imw7.com/images/Go/docker/hash_ring2.jpg)

step 3：`key` 落到服务器的落键规则

当需要存储一个 `k-v` 键值对时，首先计算 `key` 的 `hash` 值，`hash(key)`，将这个 `key` 使用相同的函数 `Hash` 计算出哈希值并确定此数据在环上的位置，**从此位置沿环顺时针”行走”**，第一台遇到的服务器就是其应该定位到的服务器，并将该键值对存储在该节点上。

如有 `Object A`、`Object B`、`Object C`、`Object D` 四个数据对象，经过哈希计算后，在环空间上的位置如下：根据一致性 `Hash` 算法，数据A会被定位到 `Node A`上，B被定位到 `Node B` 上，`C` 被定位到 `Node C` 上，`D` 被定位到 `Node D` 上。

![hash_ring3](https://blog.imw7.com/images/Go/docker/hash_ring3.jpg)

4、优点

①一致性哈希算法的**容错性**

假如 `Node C` 宕机，可以看出此时对象 `A`、`B`、`D` 不会受到影响，只有 `C` 对象被重新定位到 `Node D`。一般的，在一致性 `Hash` 算法中，如果一台服务器不可用，则**受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间的数据**，其它不会受到影响。简单说，就是 `C` 挂了，受到影响的只是 `B`、`C` 之间的数据，并且这些数据会转移到 `D` 进行存储。

![hash_ring4](https://blog.imw7.com/images/Go/docker/hash_ring4.jpg)

②一致性哈希算法的**扩展性**

数据量增加了，需要增加一台节点 `Node X`，`X` 的位置在 `A` 和 `B` 之间，那收到影响的也就是 `A` 到 `X` 之间的数据，重新把 `A` 到 `X` 的数据录入到 `X` 上即可，不会导致 `hash` 取余全部数据重新洗牌。

![hash_ring5](https://blog.imw7.com/images/Go/docker/hash_ring5.jpg)

5、缺点

一致性哈希算法的**数据倾斜**问题：

一致性 `Hash` 算法在服务**节点太少时**，容易因为节点分布不均匀而造成**数据倾斜**（被缓存的对象大部分集中缓存在某一台服务器上）问题，例如系统中只有两台服务器：

![hash_ring6](https://blog.imw7.com/images/Go/docker/hash_ring6.jpg)

6、小总结

为了在节点数目发生改变时尽可能少的迁移数据。将所有的存储节点排列在首尾相连的 `Hash` 环上，每个 `key` 在计算 `Hash` 后会顺时针找到临近的存储节点存放。而当有节点加入或退出时仅影响该节点在 `Hash` 环上**顺时针相邻的后续节点**。

**优点**：加入和删除节点只影响哈希环中顺时针方向的相邻的节点，对其他节点无影响。

**缺点**：数据的分布和节点的位置有关，因为这些节点不是均匀的分布在哈希环上的，所以数据在进行存储时达不到均匀分布的效果。

##### 哈希槽分区

1、是什么

①为什么会出现

一致性哈希算法的**数据倾斜**问题。

哈希槽实质就是一个数组，数组 $[0,2^{14}−1]$ 形成 `hash slot` 空间。

②能干什么

解决均匀分配的问题，**在数据和节点之间又加入了一层，把这层称为哈希槽 「`slot`」，用于管理数据和节点之间的关系**，现在就相当于节点上放的是槽，槽里放的是数据。

![slot1](https://blog.imw7.com/images/Go/docker/slot1.png)

槽解决的是粒度问题，相当于把粒度变大了，这样便于数据移动。

哈希解决的是映射问题，使用 `key` 的哈希值来计算所在的槽，便于数据分配。

③多少个 `hash` 槽

一个集群只能有 `16384` 个槽，编号`0-16383` 「$[0,2^{14}−1]$ 」。这些槽会分配给集群中的所有主节点，分配策略没有要求。可以指定哪些编号的槽分配给哪个主节点。集群会记录节点和槽的对应关系。解决了节点和槽的关系后，接下来就需要对 `key` 求哈希值，然后对 `16384` 取余，余数是几，`key` 就落入对应的槽里。`slot = CRC16(key) % 16384`。以槽为单位移动数据，因为槽的数目是固定的，处理起来比较容易，这样数据移动问题就解决了。

2、哈希槽计算

`Redis` 集群中内置了 `16384` 个哈希槽，`Redis` 会根据节点数量大致均等的将哈希槽映射到不同的节点。当需要在 `Redis` 集群中放置一个 `key-value` 时，`Redis` 先对 `key` 使用 `crc16` 算法算出一个结果，然后把结果对 `16384` 求余数，这样每个 `key` 都会对应一个编号在 `0-16383` 之间的哈希槽，也就是映射到某个节点上。如下代码，`key` 之`A`、`B` 在 `Node2`，`key` 之 `C` 落在 `Node3` 上。

![slot2](https://blog.imw7.com/images/Go/docker/slot2.jpg)

### 3主3从Redis集群扩缩容配置案例架构说明

![redis_cluster](https://blog.imw7.com/images/Go/docker/redis_cluster.png)

### 步骤

#### 3主3从Redis集群配置

1、关闭防火墙+启动 `Docker` 后台服务

```bash
systemctl start docker
```

2、新建6个 `Docker` 容器实例

```bash
docker run -d --name redis-node-1 --restart always --net host --privileged=true -v /data/redis/share/redis-node-1:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6381
docker run -d --name redis-node-2 --restart always --net host --privileged=true -v /data/redis/share/redis-node-2:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6382
docker run -d --name redis-node-3 --restart always --net host --privileged=true -v /data/redis/share/redis-node-3:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6383
docker run -d --name redis-node-4 --restart always --net host --privileged=true -v /data/redis/share/redis-node-4:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6384
docker run -d --name redis-node-5 --restart always --net host --privileged=true -v /data/redis/share/redis-node-5:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6385
docker run -d --name redis-node-6 --restart always --net host --privileged=true -v /data/redis/share/redis-node-6:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6386
```

如果运行成功，效果如下：

```bash
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS         PORTS     NAMES
b28d2108811b   redis:6.0.8   "docker-entrypoint.s…"   4 seconds ago   Up 4 seconds             redis-node-6
24c1a37b99a5   redis:6.0.8   "docker-entrypoint.s…"   7 seconds ago   Up 7 seconds             redis-node-5
daa39c009d24   redis:6.0.8   "docker-entrypoint.s…"   7 seconds ago   Up 7 seconds             redis-node-4
e8ed44b0a6c0   redis:6.0.8   "docker-entrypoint.s…"   8 seconds ago   Up 7 seconds             redis-node-3
1525ab8ecfaf   redis:6.0.8   "docker-entrypoint.s…"   8 seconds ago   Up 8 seconds             redis-node-2
dcafc91c1562   redis:6.0.8   "docker-entrypoint.s…"   8 seconds ago   Up 8 seconds             redis-node-1
```

命令分步解释：

- `docker run`：创建并运行 `Docker` 容器实例
- `--name redis-node-6`：容器名字
- `--net host`：使用宿主机的 `IP` 和端口，默认
- `--privileged=true`：获取宿主机 `root` 用户权限
- `-v /data/redis/share/redis-node-6:/data`：容器卷，宿主机地址：`Docker` 内部地址
- `redis:6.0.8`：`Redis` 镜像和版本号
- `--cluster-enabled yes`：开启 `Redis` 集群
- `--appendonly yes`：开启持久化
- `--port 6386`：`Redis` 端口号

3、进入容器 `redis-node-1` 并为6台机器构建集群关系

step 1：进入容器

```bash
docker exec -it redis-node-1 /bin/bash
```

step 2：构建主从关系

**注意**：进入 `Docker` 容器后才能执行下一个命令，且注意自己的真实 `IP` 地址

```bash
root@centos:/data# redis-cli --cluster create 172.30.108.127:6381 172.30.108.127:6382 172.30.108.127:6383 172.30.108.127:6384 172.30.108.127:6385 172.30.108.127:6386 --cluster-replicas 1
```

其中 `--cluster-replicas 1` 表示为每个 `master` 创建一个 `slave` 节点。

```bash
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.30.108.127:6385 to 172.30.108.127:6381
Adding replica 172.30.108.127:6386 to 172.30.108.127:6382
Adding replica 172.30.108.127:6384 to 172.30.108.127:6383
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

如果一切OK，3主3从搞定。

4、链接进入 `6381` 作为切入点，**查看集群状态**

`cluster info`：

```bash
root@centos:/data# redis-cli -p 6381
127.0.0.1:6381> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:508
cluster_stats_messages_pong_sent:514
cluster_stats_messages_sent:1022
cluster_stats_messages_ping_received:509
cluster_stats_messages_pong_received:508
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:1022
```

`cluster nodes`：

```bash
127.0.0.1:6381> cluster nodes
2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381@16381 myself,master - 0 1656663477000 1 connected 0-5460
8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383@16383 master - 0 1656663478032 3 connected 10923-16383
f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385@16385 slave 806df04787c07b9a7cc12eba80c80b529b03b3b2 0 1656663479034 2 connected
68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384@16384 slave 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 0 1656663481038 1 connected
a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386@16386 slave 8fee34d62a334219c66c151f21aab77ba3b467ee 0 1656663480036 3 connected
806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382@16382 master - 0 1656663479000 2 connected 5461-10922
```

#### 主从容错切换迁移案例

1、数据读写存储

step 1：启动6机构成的集群并通过 `exec` 进入

```bash
docker exec -it redis-node-1 /bin/bash
```

step 2：对 `6381` 新增两个 `key`

```bash
root@centos:/data# redis-cli -p 6381
127.0.0.1:6381> keys *
(empty array)
127.0.0.1:6381> set k1 v1
(error) MOVED 12706 172.30.108.127:6383
127.0.0.1:6381> set k2 v2
OK
127.0.0.1:6381>
```

step 3：防止路由失效加参数 `-c` 并新增两个 `key`

```bash
root@centos:/data# redis-cli -p 6381 -c
127.0.0.1:6381> FLUSHALL
OK
127.0.0.1:6381> set k1 v1
-> Redirected to slot [12706] located at 172.30.108.127:6383
OK
172.30.108.127:6383> set k2 v2
-> Redirected to slot [449] located at 172.30.108.127:6381
OK
172.30.108.127:6381>
```

step 4：查看集群信息

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 1 keys | 5461 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 5461 slots | 1 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 5462 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

2、容错切换迁移

step 1：主 `6381` 和从机切换，先停止主机 `6381`

先进入主机 `redis-node-1` 查看集群信息：

```bash
$ docker exec -it redis-node-1 /bin/bash
root@centos:/data# redis-cli -p 6381 -c
127.0.0.1:6381> cluster nodes
2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381@16381 myself,master - 0 1656781342000 1 connected 0-5460
8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383@16383 master - 0 1656781341000 3 connected 10923-16383
f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385@16385 slave 806df04787c07b9a7cc12eba80c80b529b03b3b2 0 1656781342000 2 connected
68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384@16384 slave 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 0 1656781343850 1 connected
a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386@16386 slave 8fee34d62a334219c66c151f21aab77ba3b467ee 0 1656781343000 3 connected
806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382@16382 master - 0 1656781344853 2 connected 5461-10922
127.0.0.1:6381> 
```

然后把主机 `redis-node-1` 停止：

```bash
docker stop redis-node-1
```

step 2：再次查看集群信息

```bash
$ docker exec -it redis-node-2 /bin/bash
root@centos:/data# redis-cli -p 6382 -c
127.0.0.1:6382> cluster nodes
2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381@16381 master,fail - 1656781408103 1656781405086 1 disconnected
f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385@16385 slave 806df04787c07b9a7cc12eba80c80b529b03b3b2 0 1656781471312 2 connected
8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383@16383 master - 0 1656781472314 3 connected 10923-16383
806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382@16382 myself,master - 0 1656781471000 2 connected 5461-10922
a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386@16386 slave 8fee34d62a334219c66c151f21aab77ba3b467ee 0 1656781473318 3 connected
68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384@16384 master - 0 1656781472000 7 connected 0-5460
127.0.0.1:6382> 
```

`6381` 宕机了，`6384` 上位成为了新的 `master` 节点。

**备注**：本次是 `6381` 为主机下面挂载从机 `6384`。每次主机挂载的从机可能不同，以实际情况为准。

一台主机宕机，数据不会受到影响：

```bash
127.0.0.1:6382> get k1
-> Redirected to slot [12706] located at 172.30.108.127:6383
"v1"
172.30.108.127:6383> get k2
-> Redirected to slot [449] located at 172.30.108.127:6384
"v2"
172.30.108.127:6384> get k3
(nil)
172.30.108.127:6384> get k4
-> Redirected to slot [8455] located at 172.30.108.127:6382
(nil)
172.30.108.127:6382> 
```

step 3：先还原之前的3主3从

①先启动 `6381`

```bash
docker start redis-node-1
```

②再停 `6384`

```bash
docker stop redis-node-4
```

③再启 `6384`

```bash
docker start redis-node-4
```

**注意**：主从机分配情况以实际情况为准

step 4：查看集群状态

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 1 keys | 5461 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 5461 slots | 1 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 5462 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### 主从扩容案例

1、新建 `6387`、`6388` 两个节点+新建后启动+查看是否8节点

新建并启动 `6387`：

```bash
docker run -d --name redis-node-7 --net host --privileged=true -v /data/redis/share/redis-node-7:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6387
```

新建并启动 `6388`：

```bash
docker run -d --name redis-node-8 --net host --privileged=true -v /data/redis/share/redis-node-8:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6388
```

查看是否8节点：

```bash
docker ps
```

2、进入 `6387` 容器实例内部

```bash
docker exec -it redis-node-7 /bin/bash
```

3、将新增的 `6387` 节点(空槽号)作为 `master` 节点加入原集群

```bash
redis-cli --cluster add-node 172.30.108.127:6387 172.30.108.127:6381
```

`6387` 就是将要作为 `master` 的新增节点。

`6381` 就是原来集群节点里面的领路人，相当于 `6387` 拜拜 `6381` 的码头从而找到组织加入集群。

```bash
[root@centos ~]# docker exec -it redis-node-7 /bin/bash
root@centos:/data# redis-cli --cluster add-node 172.30.108.127:6387 172.30.108.127:6381
>>> Adding node 172.30.108.127:6387 to cluster 172.30.108.127:6381
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 172.30.108.127:6387 to make it join the cluster.
[OK] New node added correctly.
```

4、检查集群情况第1次

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 1 keys | 5461 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 5461 slots | 1 slaves.
172.30.108.127:6387 (b8fb6518...) -> 0 keys | 0 slots | 0 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 5462 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots: (0 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

5、重新分派槽号

```bash
redis-cli --cluster reshard IP:port
```

实际操作：

```bash
root@centos:/data# redis-cli --cluster reshard 172.30.108.127:6381
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots: (0 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096
What is the receiving node ID? b8fb6518d6d9190252a4bece85cf71deeffe406e
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all

Ready to move 4096 slots.
  Source nodes:
    M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
       slots:[0-5460] (5461 slots) master
       1 additional replica(s)
    M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
       slots:[10923-16383] (5461 slots) master
       1 additional replica(s)
    M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
       slots:[5461-10922] (5462 slots) master
       1 additional replica(s)
  Destination node:
    M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
       slots: (0 slots) master
  Resharding plan:
    Moving slot 5461 from 806df04787c07b9a7cc12eba80c80b529b03b3b2
    Moving slot 5462 from 806df04787c07b9a7cc12eba80c80b529b03b3b2
    ...
Do you want to proceed with the proposed reshard plan (yes/no)? yes
Moving slot 5461 from 172.30.108.127:6382 to 172.30.108.127:6387: 
...
```

关于`How many slots do you want to move (from 1 to 16384)?` 这里应该填写的是4096，怎么来的呢？只需要将 `16384/master台数` 所得到的数字填入即可。

6、检查集群情况第2次

```bash
redis-cli --cluster check IP:port
```

实际操作：

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 0 keys | 4096 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 4096 slots | 1 slaves.
172.30.108.127:6387 (b8fb6518...) -> 1 keys | 4096 slots | 0 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

**槽号分派说明**

为什么 `6387` 是3个新的区间，以前的还是连续的？

重新分配成本太高，所以前面3家各自匀出来一部分，从 `6381/6382/6383` 三个旧节点分别匀出 1364 个槽位给新节点 `6387`。

7、为主节点 `6387` 分配从节点 `6388`

```bash
redis-cli --cluster add-node ip:新slave端口 ip:新master端口 --cluster-slave --cluster-master-id 新主机节点ID
```

实际操作：

```bash
root@centos:/data# redis-cli --cluster add-node 172.30.108.127:6388 172.30.108.127:6387 --cluster-slave --cluster-master-id b8fb6518d6d9190252a4bece85cf71deeffe406e #真实的6387的id
>>> Adding node 172.30.108.127:6388 to cluster 172.30.108.127:6387
>>> Performing Cluster Check (using node 172.30.108.127:6387)
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
>>> Send CLUSTER MEET to node 172.30.108.127:6388 to make it join the cluster.
Waiting for the cluster to join

>>> Configure node as replica of 172.30.108.127:6387.
[OK] New node added correctly.
```

8、检查集群情况第3次

```bash
[root@centos ~]# docker exec -it redis-node-1 /bin/bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 0 keys | 4096 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 4096 slots | 1 slaves.
172.30.108.127:6387 (b8fb6518...) -> 1 keys | 4096 slots | 1 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: c5ad1043832242c4a08ce7fb268e44a6299d34e7 172.30.108.127:6388
   slots: (0 slots) slave
   replicates b8fb6518d6d9190252a4bece85cf71deeffe406e
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
   1 additional replica(s)
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### 主从缩容案例

目的：`6387` 和 `6388` 下线

1、检查集群情况1获得 `6388` 节点 `ID`

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6382
...
S: c5ad1043832242c4a08ce7fb268e44a6299d34e7 172.30.108.127:6388
   slots: (0 slots) slave
   replicates b8fb6518d6d9190252a4bece85cf71deeffe406e
...
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

2、将 `6388` 删除

从集群中将4号从节点 `6388` 删除

```bash
redis-cli --cluster del-node ip:从机端口 从机6388节点ID
```

实际操作：

```bash
root@centos:/data# redis-cli --cluster del-node 172.30.108.127:6388 c5ad1043832242c4a08ce7fb268e44a6299d34e7
>>> Removing node c5ad1043832242c4a08ce7fb268e44a6299d34e7 from cluster 172.30.108.127:6388
>>> Sending CLUSTER FORGET messages to the cluster...
>>> Sending CLUSTER RESET SOFT to the deleted node.
```

检查一下发现，`6388` 被删除了，只剩下7台机器了：

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6382
172.30.108.127:6382 (806df047...) -> 0 keys | 4096 slots | 1 slaves.
172.30.108.127:6381 (2e6bd5c5...) -> 0 keys | 4096 slots | 1 slaves.
172.30.108.127:6387 (b8fb6518...) -> 1 keys | 4096 slots | 0 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6382)
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

4、将 `6387` 的槽号清空，重新分配，本例将清出来的槽号都给 `6381`

```bash
root@centos:/data# redis-cli --cluster reshard 172.30.108.127:6381
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[1365-5460] (4096 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 4096
What is the receiving node ID? 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb #6381的节点id，由它接手空出来的槽号
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: b8fb6518d6d9190252a4bece85cf71deeffe406e #6387的节点id，要被删除的那个
Source node #2: done

Ready to move 4096 slots.
  Source nodes:
    M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
       slots:[0-1364],[5461-6826],[10923-12287] (4096 slots) master
  Destination node:
    M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
       slots:[1365-5460] (4096 slots) master
       1 additional replica(s)
  Resharding plan:
    Moving slot 0 from b8fb6518d6d9190252a4bece85cf71deeffe406e
    Moving slot 1 from b8fb6518d6d9190252a4bece85cf71deeffe406e
    Moving slot 2 from b8fb6518d6d9190252a4bece85cf71deeffe406e
    Moving slot 3 from b8fb6518d6d9190252a4bece85cf71deeffe406e
    Moving slot 4 from b8fb6518d6d9190252a4bece85cf71deeffe406e
	...
Do you want to proceed with the proposed reshard plan (yes/no)? yes
...
```

5、检查集群情况2

```bash
redis-cli --cluster check 172.30.108.127:6381
```

4096 个槽位都指给 `6381`，它变成了8192个槽位。相当于全部都给 `6381` 了，不然要输入3次。

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 1 keys | 8192 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 4096 slots | 1 slaves.
172.30.108.127:6387 (b8fb6518...) -> 0 keys | 0 slots | 0 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 4 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-6826],[10923-12287] (8192 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
M: b8fb6518d6d9190252a4bece85cf71deeffe406e 172.30.108.127:6387
   slots: (0 slots) master
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

6、将 `6387` 删除

```bash
redis-cli --cluster del-node ip:从机端口6387节点ID
```

实际操作：

```bash
root@centos:/data# redis-cli --cluster del-node 172.30.108.127:6387 b8fb6518d6d9190252a4bece85cf71deeffe406e
>>> Removing node b8fb6518d6d9190252a4bece85cf71deeffe406e from cluster 172.30.108.127:6387
>>> Sending CLUSTER FORGET messages to the cluster...
>>> Sending CLUSTER RESET SOFT to the deleted node.
```

7、检查集群情况3

```bash
root@centos:/data# redis-cli --cluster check 172.30.108.127:6381
172.30.108.127:6381 (2e6bd5c5...) -> 1 keys | 8192 slots | 1 slaves.
172.30.108.127:6383 (8fee34d6...) -> 1 keys | 4096 slots | 1 slaves.
172.30.108.127:6382 (806df047...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 2 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 172.30.108.127:6381)
M: 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb 172.30.108.127:6381
   slots:[0-6826],[10923-12287] (8192 slots) master
   1 additional replica(s)
M: 8fee34d62a334219c66c151f21aab77ba3b467ee 172.30.108.127:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: a81198d45c07dc3580c9de56255aee219b6b7662 172.30.108.127:6386
   slots: (0 slots) slave
   replicates 8fee34d62a334219c66c151f21aab77ba3b467ee
S: 68549a4f4b658eb7d443629eac61aea4feec1f37 172.30.108.127:6384
   slots: (0 slots) slave
   replicates 2e6bd5c5bd1bea86e906931f02ac7f9fffb03ceb
M: 806df04787c07b9a7cc12eba80c80b529b03b3b2 172.30.108.127:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: f50ca2949b33d267a3993a41fcae65de8f9370cc 172.30.108.127:6385
   slots: (0 slots) slave
   replicates 806df04787c07b9a7cc12eba80c80b529b03b3b2
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## Dockerfile解析

### 是什么

`Dockerfile` 是用来构建 `Docker` 镜像的文本文件，是由一条条构建镜像所需的指令和参数构成的脚本。

#### 概述

![dockerfile](https://blog.imw7.com/images/Go/docker/dockerfile.png)

#### 官网

https://docs.docker.com/engine/reference/builder/

#### 构建三步骤

step1：编写 `Dockerfile` 文件

step2：`docker build` 命令构建镜像

step3：`docker run` 依镜像运行容器实例

### Dockerfile构建过程解析

#### Dockerfile内容基础知识

1、每条保留字指令都**必须为大写字母**且后面要跟随至少一个参数

2、指令按照从上到下，顺序执行

3、`#` 表示注释

4、每条指令都会创建一个新的镜像层并对镜像进行提交

#### Docker执行Dockerfile的大致流程

step1：`Docker` 从基础镜像运行一个容器

step2：执行一条指令并对容器作出修改

step3：执行类似 `docker commit` 的操作提交一个新的镜像层

step4：`Docker` 再基于刚提交的镜像运行一个新容器

step5：执行 `Dockerfile` 中的下一条指令直到所有指令都执行完成

#### 小总结

从应用软件的角度来看，`Dockerfile`、`Docker` 镜像与 `Docker` 容器分别代表软件的三个不同阶段，

- `Dockerfile` 是软件的原材料
- `Docker` 镜像是软件的交付品
- `Docker` 容器则可以认为是软件镜像的运行态，也即依照镜像运行的容器实例

`Dockerfile` 面向开发，`Docker` 镜像成为交付标准，`Docker` 容器则涉及部署与运维，三者缺一不可，合力充当 `Docker` 体系的基石。

![dockerfile1](https://blog.imw7.com/images/Go/docker/dockerfile1.png)

1. `Dockerfile`，需要定义一个 `Dockerfile`，`Dockerfile` 定义了进程需要的一切东西。`Dockerfile` 涉及的内容包括执行代码或者是文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程和内核进程(当应用进程需要和系统服务和内核进程打交道，这时需要考虑如何设计 `namespace` 的权限控制)等等；
2. `Docker` 镜像，在用 `Dockerfile` 定义一个文件之后，`docker build` 时会产生一个 `Docker` 镜像，当运行`Docker` 镜像时会真正开始提供服务；
3. `Docker` 容器，容器是直接提供服务的。

### Dockerfile常用保留字指令

#### 常用保留字

**注**：参考 `tomcat8` 的 `Dockerfile` [入门](https://github.com/docker-library/tomcat)。

##### FROM

基础镜像，当前新镜像是基于哪个镜像的，指定一个已经存在的镜像作为模板，第一条必须是 `FROM`。

##### MAINTAINER

镜像维护者的姓名和邮箱地址。

##### RUN

1、容器构建时需要运行的命令。

2、两种格式：

- `shell` 格式：

```dockerfile
RUN <命令行命令>
# <命令行命令> 等同于，在终端操作的 shell 命令。
```

- `exec` 格式：

```dockerfile
RUN ["可执行文件", "参数1", "参数2"]
# 例如：
# RUN ["./test.php", "dev", "offline"] 等价于 RUN ./test.php dev offline
```

3、`RUN` 是在 `docker build` 时运行。

##### EXPOSE

当前容器对外暴露出的端口。

##### WORKDIR

指定在创建容器后，终端默认登陆进来的工作目录，一个落脚点。

##### USER

指定该镜像以什么样的用户去执行，如果不指定，默认是 `root`。

##### ENV

用来在构建镜像过程中设置环境变量。

```dockerfile
ENV MY_PATH /usr/mytest
```

这个环境变量可以在后续的任何 `RUN` 指令中使用，这就如同在命令前面指定了环境变量前缀一样；也可以在其它指令中直接使用这些环境变量。比如：

```dockerfile
WORKDIR $MY_PATH
```

##### ADD

将宿主机目录下的文件拷贝进镜像且会自动处理 `URL` 和解压 `tar` 压缩包。

##### COPY

类似 `ADD`，拷贝文件和目录到镜像中。

将从构建上下文目录中<源路径>的文件/目录复制到新的一层的镜像内的<目标路径>位置。

```dockerfile
COPY src dest
COPY ["src", "dest"]
# <src源路径>：源文件或者源目录
# <dest目标路径>：容器内的指定路径，该路径不用事先建好，路径如果不存在，会自动创建。
```

##### VOLUME

容器数据卷，用于数据保存和持久化工作。

##### CMD

1、指定容器**启动后**要干的事情

`CMD` 容器启动命令，`CMD` 指令的格式和 `RUN` 相似，也是两种格式：

- `shell` 格式：`CMD <命令>`
- `exec` 格式：`CMD ["可执行文件", "参数1", "参数2"...]`
- 参数列表格式：`CMD ["参数1", "参数2"...]`，在指定了 `ENTRYPOINT` 指令后，用 `CMD` 指定具体的参数。

2、注意

`Dockerfile` 中可以有多个 `CMD` 指令，**但只有最后一个生效，`CMD` 会被 `docker run` 之后的参数替换**。

参考官网 `Tomcat` 的 `Dockerfile` 演示：

官网最后一行命令：

```dockerfile
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

演示自己的覆盖操作：

```bash
docker run -it -p 8080:8080 billygoo/tomcat8-jdk8
```

这样能够正常访问Tomcat首页，如果是这样：

```bash
docker run -it -p 8080:8080 billygoo/tomcat8-jdk8 /bin/bash
```

则会导致Tomcat首页无法访问，因为 `CMD ["catalina.sh", "run"]` 指令被 `/bin/bash` 给替换了。

3、它和前面`RUN`命令的区别

- `CMD` 是在 `docker run` 时运行。
- `RUN` 是在 `docker build` 时运行。

##### ENTRYPOINT

也是用来指定一个容器启动时要运行的命令。类似于 `CMD` 指令，**但是 `ENTRYPOINT` 不会被 `docker run` 后面的命令覆盖**，而且这些命令行参数**会被当作参数送给 `ENTRYPOINT` 指令指定的程序**。

1、命令格式和案例说明

命令格式：

```dockerfile
ENTRYPOINT ["<executeable>","<param1>","<param2>",...]
```

`ENTRYPOINT` 可以和 `CMD` 一起用，一般是**变参**才会使用 `CMD`，这里的 `CMD` 等于是在给 `ENTRYPOINT` 传参。

当指定了 `ENTRYPOINT` 后，`CMD` 的含义就发生了变化，不再是直接运行其命令而是将 `CMD` 的内容作为参数传递给 `ENTRYPOINT` 指令，他两个组合会变成 `<ENTRYPOINT> "<CMD>"`

案例如下：

假设已通过 `Dockerfile` 构建了 `nginx:test` 镜像：

```dockerfile
FROM nginx

ENTRYPOINT ["nginx", "-c"] # 定参
CMD ["/etc/nginx/nginx.conf"] # 变参
```

| 是否传参         | 按照Dockerfile编写执行         | 传参运行                                     |
| :--------------- | :----------------------------- | :------------------------------------------- |
| Docker命令       | docker run nginx:test          | docker run nginx:test -c /etc/nginx/new.conf |
| 衍生出的实际命令 | nginx -c /etc/nginx/nginx.conf | nginx -c /etc/nginx/new.conf                 |

2、优点

在执行 `docker run` 的时候可以通过 `--entrypoint` 来指定 `ENTRYPOINT` 运行所需的参数。

3、注意

如果 `Dockerfile` 中存在多个 `ENTRYPOINT` 指令，仅最后一个生效。

#### 小总结

| BUILD         | Both    | RUN        |
| ------------- | ------- | ---------- |
| FROM          | WORKDIR | CMD        |
| MAINTAINER    | USER    | ENV        |
| COPY          |         | EXPOSE     |
| ADD           |         | VOLUME     |
| RUN           |         | ENTRYPOINT |
| ONBUILD       |         |            |
| .dockerignore |         |            |

### Docker部署Go Web应用

#### 准备代码

```go
package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, _ *http.Request) {
	_, err := w.Write([]byte("Hello, 世界！"))
	if err != nil {
		return
	}
}

func main() {
	http.HandleFunc("/", hello)
	server := &http.Server{
		Addr: ":8080",
	}
	fmt.Println("server startup...")
	if err := server.ListenAndServe(); err != nil {
		fmt.Printf("server startup failed, err:%v\n", err)
	}
}
```

上面的代码通过 `8080` 端口对外提供服务，返回一个字符串响应：`Hello, 世界！`。

#### 创建Docker镜像

##### 创建go镜像

想用最新版本的`Go`语言，于是自己搭建了 `go:alpine` 镜像。

新建`Dockerfile`文件，下载`go1.21.5.linux-amd64.tar.gz`，并将它们放在同一个文件夹下。

将以下内容写入 `Dockerfile` 中：

```dockerfile
FROM alpine

LABEL author="ashley<hugew7@163.com>"

RUN mkdir /root/Go
WORKDIR /root/Go

ADD go1.21.5.linux-amd64.tar.gz /usr/local

ENV GO111MODULE=on \
    GOPROXY=https://goproxy.cn,direct

# 配置go环境变量
ENV GOROOT=/usr/local/go \
    GOPATH=/root/Go \
    PATH=/usr/local/go/bin

EXPOSE 80

CMD echo "success----------ok" \
    echo "/bin/sh"
```

##### 构建go镜像

在相同目录下执行：

```bash
docker build -t go:alpine .
```

下面开始构建`Go Web`镜像，并搭建。

##### 创建Web镜像

在上面的项目中，新建`Dockerfile`文件。写入：

```dockerfile
FROM go:alpine #自己构建的go镜像
LABEL authors="ashley<hugew7@163.com>"

WORKDIR /app

COPY . .

EXPOSE 8080

RUN go build -o hello .

CMD ["/app/hello"]
```

##### 构建Web镜像

在项目目录下，执行下面的命令创建镜像，指定镜像名称为 `hello`：

```bash
docker build -t hello .
```

现在已经准备好了镜像，但是目前它什么也没做。我们接下来要做的是运行我们的镜像，以便它能够处理我们的请求。运行中的镜像称为容器。

执行下面的命令来运行镜像：

```bash
docker run -p 8080：8080 hello
```

标志位`-p`用来定义端口绑定。由于容器中的应用程序在端口8888上运行，我们将其绑定到主机端口也是8888。如果要绑定到另一个端口，则可以使用`-p $HOST_PORT:8080`。例如`-p 5000:8080`。

现在就可以测试下我们的web程序是否工作正常，打开浏览器输入`http://127.0.0.1:8080`就能看到我们事先定义的响应内容如下：

```bash
Hello, 世界！
```

### 分段式构建

我们的`Go`程序编译之后会得到一个可执行的二进制文件，其实在最终的镜像中是不需要`Go`编译器的，也就是说我们只需要一个运行最终二进制文件的容器即可。

`Docker`的最佳实践之一是通过仅保留二进制文件来减小镜像大小，为此，我们将使用一种称为多阶段构建的技术，这意味着我们将通过多个步骤构建镜像。

```dockerfile
FROM go:alpine AS builder #自己构建的go镜像
LABEL authors="ashley<hugew7@163.com>"

# 移动到工作目录：/app
WORKDIR /app

# 将代码复制到容器中
COPY . .

EXPOSE 8081

# 将代码编译成二进制可执行文件 hello
RUN go build -o hello2 .

# 接下来创建一个小镜像
FROM scratch

#从builder镜像中把 /dist/hello 拷贝到当前目录
COPY --from=builder /app/hello2 /

# 需要运行的命令
ENTRYPOINT ["/hello2"]
```

使用这种技术，我们剥离了使用`go:alpine`作为编译镜像来编译得到二进制可执行文件的过程，并基于`scratch`生成一个简单的、非常小的新镜像。我们将二进制文件从命名为`builder`的第一个镜像中复制到新创建的`scratch`镜像中。

```bash
$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
hello2       latest    7c2d373df216   4 minutes ago   6.71MB
hello        latest    e9f464d9adc3   19 hours ago    289MB
```

可以发现，使用了 `scratch` 的镜像大小非常小的。

有关`scratch`镜像的更多信息，请查看https://hub.docker.com/_/scratch。

### 自定义镜像

#### 要求

`Centos7` 镜像具备 `vim` + `ifconfig` + [`jdk8`](https://www.oracle.com/java/technologies/downloads/#java8)

#### 编写

准备编写 `Dockerfile` 文件，**大写字母D**。在本地下载 `jdk8`，宿主机中创建 `myfile` 文件夹，并将 `jdk8` 传到文件夹下，并在该文件夹创建 `Dockerfile` 文件。

```bash
[root@centos myfile]# ls
Dockerfile  jdk-8u333-linux-x64.tar.gz
```

将以下内容写入 `Dockerfile` 中：

```dockerfile
FROM centos:7
MAINTAINER ashley<hugew7@163.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH

#安装vim编辑器
RUN yum -y install vim
#安装ifconfig命令查看网络IP
RUN yum -y install net-tools
#安装java8及lib库
RUN yum -y install glibc.i686
RUN mkdir /usr/local/java
#ADD 是相对路径jar，把jdk-8u333-linux-x64.tar.gz添加到容器中，安装包必须要和Dockerfile文件在同一位置
ADD jdk-8u333-linux-x64.tar.gz /usr/local/java/
#配置java环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_333
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH

EXPOSE 80

CMD echo $MYPATH
CMD echo "success------------ok"
CMD /bin/bash
```

#### 构建

```bash
docker build -t 新镜像名字:TAG .
```

**注意：上面 `TAG` 后面有个空格，有个点**

```bash
docker build -t centosjava8:1.5 .
```

#### 运行

```bash
docker run -it 新镜像名字:TAG
```

本次：

```bash
docker run -it centosjava8:1.5
```

#### 再体会下 `UnionFS`（联合文件系统）

`UnionFS` 是一种分层、轻量级并且高性能的文件系统，它支持**对文件系统的修改作为一次提交来一层层的叠加**，同时可以将不同且目录挂载到同一个虚拟文件系统下「`unite several directories into a single virtual filesystem`」。`Union` 文件系统是 `Docker` 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

特性：一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录。

### 虚悬镜像

#### 是什么

仓库名、标签都是 `<none>` 的镜像，俗称 `dangling image`。

`Dockerfile` 写一个：

1、`vim Dockerfile`

```dockerfile
FROM ubuntu
CMD echo 'action is success'
```

2、`docker build .`

```bash
$ pwd
/myfile/test
$ docker build .
...
Successfully built c43dea5c5227
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED             SIZE
<none>        <none>    c43dea5c5227   30 seconds ago      72.8MB
```

#### 查看

```bash
$ docker image ls -f dangling=true
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
<none>       <none>    c43dea5c5227   8 minutes ago   72.8MB
```

#### 删除

```bash
docker image prune
```

虚悬镜像已经失去存在价值，可以删除。

```bash
$ docker image ls -f dangling=true
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
<none>       <none>    c43dea5c5227   10 minutes ago   72.8MB
$ docker image prune
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Deleted Images:
deleted: sha256:c43dea5c52272d6d85df1762759e1b78bc6148b99a6492327afc2e3d67e11f30

Total reclaimed space: 0B
$ docker image ls -f dangling=true
REPOSITORY   TAG       IMAGE ID   CREATED   SIZE
```

### 使用Docker中的Go编译运行本地程序

以上面的`go:alpine`镜像为例，用它来运行本地的一个简单程序。

`main.go`：

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

想要用 `go:alpine` 镜像来执行上面的程序，可以执行下面的命令：

```bash
docker run -it --rm -v "$PWD":/myapp -w /myapp go:alpine go run main.go
```

**注**：

`-it`：表示启动一个交互式的容器，即启动一个`shell`。

`--rm`：表示容器退出时，自动删除该容器。

`-v "$PWD":/myapp`：表示将本地当前的工作目录映射到容器的`/myapp`目录。

`-w /myapp`：表示设置容器的工作目录为`/myapp`目录。

`go:alpine`：表示使用`go:alpine`镜像来启动容器。

`go run main.go`：表示直接运行`main.go`。此部分可换成`/bin/sh`（使用`sh`）、`go build`（编译程序）等等其他操作。

输出结果：

```bash
Hello, 世界
```

表示，使用`Docker`中的`go`镜像来编译运行本地的`go`程序是成功的。

**Tips**：

因为`docker run -it --rm -v "$PWD":/myapp -w /myapp go:alpine go`，这段命令是不会变的，所以可以通过`alias`来简化，打开`~/.bashrc`添加：

```bash
alias go='docker run -it --rm -v "$PWD":/myapp -w /myapp go:alpine go'
```

即通过命令把 `docker run -it --rm -v "$PWD":/myapp -w /myapp go:alpine go` 替换为 `go`。

使命令生效更改：

```bash
source ~/.bashrc
```

重启`ssh`，输入 `alias` 查看命令是否生效：

```bash
[root@centos ~]# alias
alias cp='cp -i'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias go='docker run -it --rm -v "$PWD":/myapp -w /myapp go:alpine go' # 看到这个表示命令已经生效
alias grep='grep --color=auto'
alias l.='ls -d .* --color=auto'
alias ll='ls -l --color=auto'
alias ls='ls --color=auto'
alias mv='mv -i'
alias rm='rm -i'
alias which='alias | /usr/bin/which --tty-only --read-alias --show-dot --show-tilde'
```

这样就可以直接用下面的命令执行程序的编译或运行了：

```bash
go run main.go
```

结果：

```bash
Hello, 世界
```

### 小总结

![dockerfile](https://blog.imw7.com/images/Go/docker/dockerfile.png)

## Docker微服务实战



## Docker网络

### 是什么

#### Docker不启动，默认网络情况

1、`ens33`

```bash
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.173.128  netmask 255.255.255.0  broadcast 172.16.173.255
        inet6 fe80::5440:c6ca:eb51:dae4  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:af:e5:6d  txqueuelen 1000  (Ethernet)
        RX packets 118  bytes 13777 (13.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 107  bytes 11996 (11.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

2、`lo`

`localhost`的简写，本地地址。一般为：`127.0.0.1`。

```bash
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

3、`virbr0`

在 `CentOS7` 的安装过程中如果**有选择相关虚拟化的服务安装系统后**，启动网卡时会发现有一个以网桥连接的私网地址的 `virbr0` 网卡（`virbr0` 网卡：它还有一个固定的默认 `IP` 地址 `192.168.122.1`），是做虚拟机网桥的使用的，其作用是为连接其上的虚拟网卡提供 `NAT` 访问外网的功能。

之前安装 `Linux`，勾选安装系统的时候附带了 `libvirt` 服务才会生成，如果不需要可直接卸载：

```bash
$ yum remove libvirt-libs.x86_64
```

#### Docker启动后，网络情况

会产生一个名为 `docker0` 的虚拟网桥

```bash
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:c7:7f:fe:86  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

查看 `Docker` 网络模式命令：

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
c991f0b97e5e   bridge    bridge    local
8e6b4651280a   host      host      local
612c09bd5f08   none      null      local
```

默认创建3大网络模式。

### 常用基础命令

#### All命令

```bash
$ docker network --help

Usage:  docker network COMMAND

Manage networks

Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  prune       Remove all unused networks
  rm          Remove one or more networks

Run 'docker network COMMAND --help' for more information on a command.
```

#### 查看网络

```bash
docker network ls
```

#### 查看网络源数据

```bash
docker network inspect XXX网络名字
```

#### 删除网络

```bash
docker network rm XXX网络名字
```

#### 案例

```bash
$ docker network create aa_network
1410ae30c25c2e21fe5a69538f531c3e2541d3cb0047f843f608daa8f395ebaf
$ docker network ls
NETWORK ID     NAME         DRIVER    SCOPE
1410ae30c25c   aa_network   bridge    local
c991f0b97e5e   bridge       bridge    local
8e6b4651280a   host         host      local
612c09bd5f08   none         null      local
$ docker network rm aa_network
aa_network
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
c991f0b97e5e   bridge    bridge    local
8e6b4651280a   host      host      local
612c09bd5f08   none      null      local
```

### 能干嘛

1、容器间的互联和通信以及端口映射

2、容器 `IP` 变动时可以通过服务名直接网络通信而不受影响

### 网络模式

#### 总体介绍

| 网络模式  | 简介                                                         |
| :-------- | :----------------------------------------------------------- |
| bridge    | 为每一个容器分配、设置IP等，并将容器连接到一个`docker0`虚拟网桥，默认为该模式。 |
| host      | 容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用容器宿主机的IP和端口。 |
| none      | 容器有独立的 `Network namespace`，但并没有对其进行任何网络设置，如分配 `veth pair` 和网桥连接，IP等。 |
| container | 新创建的容器不会创建自己的网卡和配置自己的IP，而是和一个指定的容器共享IP、端口范围等。 |

- `bridge` 模式：使用 `--network bridge` 指定，默认使用 `docker0`
- `host` 模式：使用 `--network host` 指定
- `none` 模式：使用 `--network none` 指定
- `container` 模式：使用 `--network container:NAME 或 容器ID` 指定

#### 容器实例内默认网络IP生产规则

##### 说明

1、先启动两个 `ubuntu` 容器实例

```bash
$ docker run -it --name u1 ubuntu
root@c5fdfdfe9104:/# #按ctrl+p+q退出
$ docker run -it --name u2 ubuntu
root@0d892e6ef374:/# 
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS          PORTS     NAMES
0d892e6ef374   ubuntu    "bash"    6 seconds ago    Up 5 seconds              u2
c5fdfdfe9104   ubuntu    "bash"    18 seconds ago   Up 17 seconds             u1
```

2、`docker inspect NAME|ID`

```bash
$ docker inspect u1 | tail -n 20
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "c991f0b97e5e9d423f6632ef08c11577c01770d134da992b7c047a7dc45c9764",
                    "EndpointID": "752acb1d865c080b978b7417c051a4054285f4c4af84a694e9364c1febdac007",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
$ docker inspect u2 | tail -n 20
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "c991f0b97e5e9d423f6632ef08c11577c01770d134da992b7c047a7dc45c9764",
                    "EndpointID": "c1be3f3a34a65f9305c4ab7fc45a229b5b5ffede7e0eb123adee30f463509089",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

3、关闭 `u2` 实例，新建 `u3`，查看 `ip` 变化

```bash
$ docker stop u2
u2
$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS              PORTS     NAMES
3858717e3f07   ubuntu    "bash"    2 minutes ago    Up About a minute             u3
c5fdfdfe9104   ubuntu    "bash"    11 minutes ago   Up 11 minutes                 u1
$ docker run -it --name u3 ubuntu
root@3858717e3f07:/# #按ctrl+p+q退出
$ docker inspect u3 | tail -n 20
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "c991f0b97e5e9d423f6632ef08c11577c01770d134da992b7c047a7dc45c9764",
                    "EndpointID": "9d67500fdd88e7d14f073d03b41648d065c1c419fedcd549e4e42174a1e377ac",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

##### 结论

**`Docker` 容器内部的 `IP` 是有可能会发生改变的**。

#### 案例说明

##### bridge

1、是什么

`Docker` 服务默认会创建一个 `docker0` 网桥（其上有一个 `docker0` 内部接口），该桥接网络的名称为 `docker0`，它在**内核层**连通了其他的物理或虚拟网卡，这就将所有容器和本地主机都放到**同一个物理网络**。`Docker` 默认指定 `docker0` 接口的 `IP` 地址和子网掩码，**让主机和容器之间可以通过网桥相互通信**。

```bash
#查看bridge网络的详细信息，并通过grep获取名称项
$ docker network inspect bridge | grep name
            "com.docker.network.bridge.name": "docker0",
$ ifconfig | grep docker
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
```

2、案例

①说明

1. `Docker` 使用 `Linux桥接`，在宿主机虚拟一个 `Docker` 容器网桥(`docker0`)，`Docker` 启动一个容器时会根据 `Docker` 网桥的网段分配给容器一个 `IP` 地址，称为`Container-IP`，同时 `Docker` 网桥是每个容器的默认网关。因为在同一宿主机内的容器都接入同一个网桥，这样容器之间就能够通过容器的`Container-IP`直接通信。

2. `docker run` 时，如果没有指定 `network` ，默认使用的网桥模式就是 `bridge`，使用的就是 `docker0`。在宿主机 `ifconfig`，就可以看到 `docker0` 和自己 `create` 的 `network` 的 `eth0`，`eth1`，`eth2`……代表网卡一，网卡二，网卡三……，`lo`代表 `127.0.0.1`，即`localhost`，`inet addr` 用来表示网卡的 `IP` 地址。

3. 网桥 `docker0` 创建一对对等虚拟设备接口一个叫 `veth`，另一个叫 `eth0`，成对匹配。

   - 整个宿主机的网桥模式都是 `docker0`，类似一个交换机有一堆接口，每个接口叫 `veth`，在本地主机和容器内分别创建一个虚拟接口，并让他们彼此连通（这样一对接口叫 `veth pair`）；
   - 每个容器实例内部也有一块网卡，每个接口叫 `eht0`；
   - `docker0` 上面的每个 `veth` 匹配某个容器实例内部的 `eth0`，两两配对，一一匹配。

通过上述，将宿主机上的所有容器都连接到这个内部网络上，两个容器在同一个网络下，会从这个网关下各自拿到分配的 `ip`，此时两个容器的网络是互通的。

![bridge](https://blog.imw7.com/images/Go/docker/bridge.png) 

②代码

```bash
docker run -d -p 8081:8080 --name tomcat81 billygoo/tomcat8-jdk8
docker run -d -p 8082:8080 --name tomcat82 billygoo/tomcat8-jdk8
```

③两两匹配验证

宿主机：

```bash
$ ip addr | tail -n 8
5: vethf9ef600@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 9a:10:64:15:ef:58 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::9810:64ff:fe15:ef58/64 scope link 
       valid_lft forever preferred_lft forever
7: veth2fbe32f@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default 
    link/ether 0e:15:26:b7:b4:91 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::c15:26ff:feb7:b491/64 scope link 
       valid_lft forever preferred_lft forever
```

`tomcat81`：

```bash
$ docker exec -it tomcat81 bash
oot@3fdde15b6173:/usr/local/tomcat# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

`tomcat82`：

```bash
$ docker exec -it tomcat82 bash
root@eec5bc95985b:/usr/local/tomcat# ip addr
root@455ddfebfa4e:/usr/local/tomcat# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

##### host

1、是什么

直接使用宿主机的 `IP` 地址与外界进行通信，不再需要额外进行 `NAT` 转接。

2、案例

①说明

容器将**不会获得**一个独立的 `Network Namespace`，而是和宿主机共用一个`Network Namespace`。**容器将不会虚拟出自己的网卡而是使用宿主机的IP和端口**。

![host](https://blog.imw7.com/images/Go/docker/host.png)

  

②代码

**警告**：

```bash
$ docker run -d -p 8083:8080 --network host --name tomcat83 billygoo/tomcat8-jdk8
WARNING: Published ports are discarded when using host network mode
ee04c5a61a805f7564a9327bdd0486af91d1292b8d4262d8d794fbe6eaa0d367
$ docker ps
CONTAINER ID   IMAGE                   COMMAND            CREATED        STATUS          PORTS      NAMES
ee04c5a61a80   billygoo/tomcat8-jdk8   "catalina.sh run"  3 seconds age  Up 2 seconds               tomcat83
```

1. 问题：`Docker` 启动时总是遇见上面的警告
2. 原因：`Docker` 启动时指定 `--network=host` 或 `-net=host`，如果还指定了 `-p` 映射端口，那这个时候就会有此警告。并且通过 `-p` 设置的参数将不会起到任何作用，端口号会以主机端口号为主，重复时则递增。
3. 解决：解决的办法就是使用 `Docker` 的其他网络模式，例如 `--network=bridge`，这样就可以解决问题，或者直接无视……

**正确**：

```bash
docker run -d --network host --name tomcat83 billygoo/tomcat8-jdk8
```

**注意**：没有之前的配对显示了，看容器实例内部：

```bash
$ docker inspect tomcat83 | tail -n 20
            "Networks": {
                "host": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "bf93337b605d54c375263979dda34234ddbeff5f39c17ca921cbefad3dd8acab",
                    "EndpointID": "ddaea4ae78633f41b13e5ca4cdc652f8142a766109d1584bafc4a23205583d62",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

可以看到，`host` 网络模式下 `Gateway` 和 `IPAddress` 都为空。全部使用的是宿主机的。

**问题**：没有设置 `-p` 的端口映射了，如何访问启动的 `tomcat83`？

使用 `http://宿主机IP:8080` 即可访问 `tomcat83`。

在 `CentOS` 里面用默认的火狐浏览器访问容器内的 `tomcat83` 看到访问成功，因为此时容器的 `IP` 借用主机的，所以容器共享宿主机网络 `IP`，这样的好处是外部主机与容器可以直接通信。

##### none

1、是什么

禁用网络功能，只有 `lo` 标识（就是 `127.0.0.1` 表示本地回环）。

在 `none` 模式下，并不为 `Docker` 容器进行任何网络配置。也就是说，这个 `Docker` 容器没有网卡、`IP`、路由等信息，只有一个`lo`。需要自己为 `Docker` 容器添加网卡、配置 `IP` 等。

2、案例

```bash
$ docker run -d -p 8084:8080 --network none --name tomcat84 billygoo/tomcat8-jdk8
7933d4080506dd4ace2f060d4a002ef6d78cb1cfe74c402e478f13dd67975c6c
$ docker ps
CONTAINER ID   IMAGE                   COMMAND             CREATED          STATUS          PORTS          NAMES
7933d4080506   billygoo/tomcat8-jdk8   "catalina.sh run"   2 seconds ago    Up 2 seconds                   tomcat84
$ docker inspect tomcat84 | tail -n 20
            "Networks": {
                "none": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "a287a6fe57a0f10ae3e407ca9841daff6d868dbd8cef80b54a452b73022df043",
                    "EndpointID": "3a628a91ded6737c235012e015e5b2f355153fe37f03069dc4dc2d3944353a9b",
                    "Gateway": "",
                    "IPAddress": "",
                    "IPPrefixLen": 0,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "",
                    "DriverOpts": null
                }
            }
        }
    }
]
$ docker exec -it tomcat84 bash
root@7933d4080506:/usr/local/tomcat# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

##### container

1、是什么

`container` 网络模式，新建的容器和已经存在的一个容器共享一个网络 `IP` 配置而不是和宿主机共享，新创建的容器不会创建自己的网卡，配置自己的 `IP`，而是和一个指定的容器共享 `IP`、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。

![container](https://blog.imw7.com/images/Go/docker/container.png)

 

2、案例

```bash
docker run -d -p 8085:8080 --name tomcat85 billygoo/tomcat8-jdk8
docker run -d -p 8086:8080 --network container:tomcat85 --name tomcat86 billygoo/tomcat8-jdk8
```

运行结果：

```bash
$ docker run -d -p 8086:8080 --network container:tomcat85 --name tomcat86 billygoo/tomcat8-jdk8
docker: Error response from daemon: conflicting options: port publishing and the container type network mode.
See 'docker run --help'.
```

相当于`tomcat86` 和 `tomcat85` 共用同一个 `IP` 同一个端口，导致端口冲突。

3、案例2

`Alpine` 操作系统是一个面向安全的轻型 `Linux` 发行版。`Alpine Linux` 是一款独立的、非商用的通用 `Linux` 发行版，专为追求安全性、简单性和资源效率的用户而设计。可能很多人没听说过这个 `Linux` 发行版本，但是经常用 `Docker` 的朋友可能都用过，因为它以小、简单、安全而著称，所以作为基础镜像是非常好的一个选择，可谓是麻雀虽小但五脏俱全，镜像非常小巧，不到 `6M` 的大小，所以特别适合容器打包。

命令：

```bash
docker run -it --name alpine1 alpine /bin/sh
docker run -it --network container:alpine1 --name alpine2 alpine /bin/sh
```

运行结果，验证共用搭桥：

`alpine1`：

```bash
$ docker run -it --name alpine1 alpine /bin/sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

`alpine2`：

```bash
$ docker run -it --network container:alpine1 --name alpine2 alpine /bin/sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

假如此时关闭 `apline1`，再看看 `alpine2`：

```bash
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```

##### 自定义网络

1、过时的link

![links](https://blog.imw7.com/images/Go/docker/links.png)

2、是什么

3、案例

① before

1. 案例

```bash
docker run -d -p 8081:8080 --name tomcat81 billygoo/tomcat8-jdk8
docker run -d -p 8082:8080 --name tomcat82 billygoo/tomcat8-jdk8
```

上述成功启动并用 `docker exec` 进入各自容器实例内部。

2. 问题

- 按照 `IP` 地址 `ping` 是可以的

  - 查看 `IP` 地址：
  
  ```bash
  $ docker exec -it tomcat81 bash
  root@7273c8b7579b:/usr/local/tomcat# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  4: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  ```
  
  ```bash
  $ docker exec -it tomcat82 bash
  root@2d1a3d035761:/usr/local/tomcat# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  6: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  ```
  
  - 在 `tomcat81` 中 `ping` `tomcat82`：
  
  ```bash
  root@7273c8b7579b:/usr/local/tomcat# ping 172.17.0.3
  PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
  64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.077 ms
  64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.102 ms
  64 bytes from 172.17.0.3: icmp_seq=3 ttl=64 time=0.109 ms
  64 bytes from 172.17.0.3: icmp_seq=4 ttl=64 time=0.103 ms
  ^C
  --- 172.17.0.3 ping statistics ---
  4 packets transmitted, 4 received, 0% packet loss, time 3001ms
  rtt min/avg/max/mdev = 0.077/0.097/0.109/0.017 ms
  ```
  
  - 在 `tomcat82` 中 `ping` `tomcat81`：
  
  ```bash
  root@2d1a3d035761:/usr/local/tomcat# ping 172.17.0.2
  PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
  64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.079 ms
  64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.113 ms
  64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.178 ms
  64 bytes from 172.17.0.2: icmp_seq=4 ttl=64 time=0.114 ms
  ^C
  --- 172.17.0.2 ping statistics ---
  4 packets transmitted, 4 received, 0% packet loss, time 2999ms
  rtt min/avg/max/mdev = 0.079/0.121/0.178/0.035 ms
  ```

* 按照服务名 `ping` 结果？
  * 在 `tomcat81` 中按照服务名 `ping` `tomcat82`：

  ```bash
  root@7273c8b7579b:/usr/local/tomcat# ping tomcat82
  ping: tomcat82: Name or service not known
  ```

  * 在 `tomcat82` 中按照服务名 `ping` `tomcat81`：

  ```bash
  root@2d1a3d035761:/usr/local/tomcat# ping tomcat81
  ping: tomcat81: Name or service not known
  ```

以上使用服务名是 `ping` 不通的。

② after

1. 案例

​	自定义桥接网络，自定义网络默认使用的桥接网络 `bridge`。

- 新建自定义网络

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
2e29d6fa39aa   bridge    bridge    local
8e6b4651280a   host      host      local
612c09bd5f08   none      null      local
$ docker network create my_network
a996f674abd8e93b6f197e0dac501d2668c7b5390e105a83a1d1e40d07cc82ac
$ docker network ls
NETWORK ID     NAME         DRIVER    SCOPE
2e29d6fa39aa   bridge       bridge    local
8e6b4651280a   host         host      local
a996f674abd8   my_network   bridge    local
612c09bd5f08   none         null      local
```

- 新建容器加入上一步新建的自定义网络

```bash
$ docker run -d -p 8081:8080 --network my_network --name tomcat81 billygoo/tomcat8-jdk8
$ docker run -d -p 8082:8080 --network my_network --name tomcat82 billygoo/tomcat8-jdk8
```

- 互相 `ping` 测试

  - 进入容器查看 `IP` 地址：

  `tomcat81`：
  
  ```bash
  $ docker exec -it tomcat81 bash
  root@1a5a5a7ce19c:/usr/local/tomcat# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  9: eth0@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  ```
  
  `tomcat82`：
  
  ```bash
  $ docker exec -it tomcat82 bash
  root@6a3eb013f735:/usr/local/tomcat# ip addr
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
      link/ether 02:42:ac:12:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
      inet 172.18.0.3/16 brd 172.18.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  ```
  
  - `IP` 地址和服务名均可互相 `ping` 通：

  在 `tomcat81` 中 `ping` `tomcat82`：
  
  ```bash
  root@1a5a5a7ce19c:/usr/local/tomcat# ping 172.18.0.3
  PING 172.18.0.3 (172.18.0.3) 56(84) bytes of data.
  64 bytes from 172.18.0.3: icmp_seq=1 ttl=64 time=2.02 ms
  64 bytes from 172.18.0.3: icmp_seq=2 ttl=64 time=0.070 ms
  64 bytes from 172.18.0.3: icmp_seq=3 ttl=64 time=0.112 ms
  ^C
  --- 172.18.0.3 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2001ms
  rtt min/avg/max/mdev = 0.070/0.735/2.024/0.911 ms
  root@1a5a5a7ce19c:/usr/local/tomcat# ping tomcat82
  PING tomcat82 (172.18.0.3) 56(84) bytes of data.
  64 bytes from tomcat82.my_network (172.18.0.3): icmp_seq=1 ttl=64 time=0.066 ms
  64 bytes from tomcat82.my_network (172.18.0.3): icmp_seq=2 ttl=64 time=0.114 ms
  64 bytes from tomcat82.my_network (172.18.0.3): icmp_seq=3 ttl=64 time=0.104 ms
  ^C
  --- tomcat82 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2003ms
  rtt min/avg/max/mdev = 0.066/0.094/0.114/0.023 ms
  ```
  
  在 `tomcat82` 中 `ping` `tomcat81`：
  
  ```bash
  root@6a3eb013f735:/usr/local/tomcat# ping 172.18.0.2
  PING 172.18.0.2 (172.18.0.2) 56(84) bytes of data.
  64 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.099 ms
  64 bytes from 172.18.0.2: icmp_seq=2 ttl=64 time=0.112 ms
  64 bytes from 172.18.0.2: icmp_seq=3 ttl=64 time=0.110 ms
  ^C
  --- 172.18.0.2 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2001ms
  rtt min/avg/max/mdev = 0.099/0.107/0.112/0.005 ms
  root@6a3eb013f735:/usr/local/tomcat# ping tomcat81
  PING tomcat81 (172.18.0.2) 56(84) bytes of data.
  64 bytes from tomcat81.my_network (172.18.0.2): icmp_seq=1 ttl=64 time=0.062 ms
  64 bytes from tomcat81.my_network (172.18.0.2): icmp_seq=2 ttl=64 time=0.206 ms
  64 bytes from tomcat81.my_network (172.18.0.2): icmp_seq=3 ttl=64 time=0.132 ms
  ^C
  --- tomcat81 ping statistics ---
  3 packets transmitted, 3 received, 0% packet loss, time 2002ms
  rtt min/avg/max/mdev = 0.062/0.133/0.206/0.059 ms
  ```

2. 问题结论

​	自定义网络本身就维护好了主机名和 `ip` 的对应关系（`ip` 和域名都能通）。

### Docker平台架构图解

#### 整体说明

从其架构和运行流程来看，`Docker` 是一个 `C/S` 模式的架构，后端是一个松耦合架构，众多模型各司其职。

`Docker` 运行的基本流程为：

1、用户是使用 `Docker Client` 与 `Docker Daemon` 建立通信，并发送请求给后者。

2、`Docker Daemon` 作为 `Docker` 架构中的主体部分，首先提供 `Docker Server` 的功能使其可以接受 `Docker Client` 的请求。

3、`Docker Engine` 执行 `Docker` 内部的一系列工作，每一项工作都是以一个 `Job` 的形式的存在。

4、`Job` 的运行过程中，当需要容器镜像时，则从 `Docker Registry` 中下载镜像，并通过镜像管理驱动 `Graph driver` 将下载镜像以 `Graph` 的形式存储。

5、当需要为 `Docker` 创建网络环境时，通过网络管理驱动 `Network driver` 创建并配置 `Docker` 容器网络环境。

6、当需要限制 `Docker` 容器运行资源或执行用户指令等操作时，则通过 `Execdriver` 来完成。

7、`Libcontainer` 是一项独立的容器管理包，`Network driver` 以及 `Exec driver` 都是通过 `Libcontainer` 来实现具体对容器进行的操作。

#### 整体架构

![docker_structure](https://blog.imw7.com/images/Go/docker/docker_structure.jpg)  

## Docker-compose容器编排

### 简介

#### 概念

`Docker-Compose` 是 `Docker` 官方的开源项目，负责实现对 `Docker` 容器集群的快速编排。

`Compose` 是 `Docker` 公司推出的一个工具软件，可以管理多个 `Docker` 容器组成一个应用。你需要定义一个 `YAML` 格式的配置文件 `docker-compose.yaml`。写好多个容器之间的调用关系，然后，只需要一个命令，就能同时启动/关闭这些容器。

#### 作用

`docker` 建议每一个容器中只运行一个服务，因为 `docker` 容器本身占用资源很少，所以最好是将每个服务单独的分割开来。但是这样有面临一个问题：

如果需要同时部署好多个服务，难道要每个服务单独写 `Dockerfile`，然后再构建镜像，构建容器吗？这样太麻烦。所以 `docker` 官方给提供了 `docker-compose` 多服务部署的工具。

例如要实现一个 `Web` 微服务项目，除了 `Web` 服务容器本身，往往还需要再加上后端的数据库 `mysql` 服务容器，`redis` 服务器，注册中心 `eureka`，甚至还包括负载均衡容器等等……

`Compose` 允许用户通过一个单独的 `docker-compose.yml` 模板文件（`YAML`格式）来定义一组相关联的应用容器为一个项目（`project`）。

可以很容易地用一个配置文件定义一个多容器的应用，然后使用一条指令安装这个应用的所有依赖，完成构建。`Docker-Compose` 解决了容器与容器之间如何管理编排的问题。

#### 安装

官网API：https://docs.docker.com/compose/compose-file/compose-file-v3/

下载：https://docs.docker.com/compose/install/

##### 安装步骤

① 设置镜像仓库

第一次安装需要设置镜像仓库。

* `Ubuntu` 和 `Debian` ，执行：

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

* 基于`RPM`的发行版，执行：

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

② 安装

* `Ubuntu` 和 `Debian` ，执行：

```bash
sudo apt-get update
sudo apt-get install docker-compose-plugin
```

* 基于`RPM`的发行版，执行：

```bash
sudo yum update
sudo yum install docker-compose-plugin
```

检查`docker-compose`是否安装成功：

```bash
$ docker compose --version
Docker Compose version v2.21.0
```

出现版本说明安装成功。

##### 卸载步骤

* `Ubuntu` 和 `Debian` ，执行：

```bash
sudo apt-get remove docker-compose-plugin
```

* 基于`RPM`的发行版，执行：

```bash
sudo yum remove docker-compose-plugin
```

### Compose核心概念

#### 一文件

```
docker-compose.yml
```

#### 两要素

1. 服务（`service`）：一个个应用容器实例，比如订单微服务、库存微服务、`MySQL` 容器、`Nginx` 容器或者 `Redis` 容器。
2. 工程（`project`）：由一组关联的应用容器组成的一个**完整业务单元**，在 `docker-compose.yml` 文件中定义。

### Compose使用的三步骤

1. 编写 `Dockerfile` 定义各个微服务应用并构建出对应的镜像文件
2. 使用 `docker-compose.yml` 定义一个完整业务单元，安排好整体应用中的各个容器服务。
3. 最后，执行 `docker-compose up` 命令来启动并运行整个应用程序，完成一键部署上线。

### Compose常用命令

| 常用命令                            | 解释                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| docker compose -h                   | 查看帮助                                                     |
| docker compose up                   | 启动所有docker-compose服务                                   |
| docker compose up -d                | 启动所有docker-compose服务并后台运行                         |
| docker compose down                 | 停止并删除容器、网络、卷、镜像                               |
| docker compose exec yml里面的服务id | 进入容器实例内部 docker compose exec docker-compose.yml文件中写的服务id /bin/bash |
| docker compose ps                   | 展示当前docker-compose编排过的运行的所有容器                 |
| docker compose top                  | 展示当前docker-compose编排过的容器进程                       |
| docker compose logs yml里面的服务id | 查看容器输出日志                                             |
| docker compose config               | 检查配置                                                     |
| docker compose config -q            | 检查配置，有问题才有输出                                     |
| docker compose restart              | 重启服务                                                     |
| docker compose start                | 启动服务                                                     |
| docker compose stop                 | 停止服务                                                     |

### Compose编排微服务

```yaml
version: "3"

services:
  microService:
    image: ashley_docker:1.6
    container_name: ms01
    ports:
      - "6001:6001"
    volumes:
      - /app/microService:/data
    networks:
      - my_net
    depends_on:
      - redis
      - mysql

  redis:
    image: redis:6.0.8
    ports:
      - "6379:6379"
    volumes:
      - /app/redis/redis.conf:/etc/redis/redis.conf
      - /app/redis/data:/data
    networks:
      - my_net
    command: redis-server /etc/redis/redis.conf

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
      MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
      MYSQL_DATABASE: 'db2022'
      MYSQL_USER: 'ashley'
      MYSQL_PASSWORD: 'password'
    ports:
      - "3306:3306"
    volumes:
      - /app/mysql/db:/var/lib/mysql
      - /app/mysql/conf/my.cnf:/etc/my.cnf
      - /app/mysql/init:/docker-entrypoint-initdb.d
    networks:
      - my_net
    command: --default-authentication-plugin=mysql_native_password #解决外部无法访问

networks:
  my_net:
```

## Docker轻量级可视化工具Portainer

### 是什么

`Portainer` 是一款轻量级的应用，它提供了图形化界面，用于方便地管理 `Docker` 环境，包括单机环境和集群环境。

### 安装

#### 官网

- https://www.portainer.io/
- https://docs.portainer.io/start/install/server/docker/linux

#### 步骤

1. `docker` 命令安装

首先，创建用于 `Portainer Server` 存储数据库的卷：

```bash
docker volume create portainer_data
```

然后，下载安装 `Portainer Server` 容器：

```bash
docker run -d -p 8000:8000 -p 9000:9000 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

1. 第一次登陆需创建 `admin`，访问地址：`xxx.xxx.xxx.xxx:9000`

```http
http://192.168.3.6:9000
```

![portainer-be-server-setup](https://blog.imw7.com/images/Go/docker/portainer-be-server-setup.png)   

1. 设置 `admin` 用户和密码后首次登陆

![portainer-install-setup-wizard](https://blog.imw7.com/images/Go/docker/portainer-install-setup-wizard.png)    

1. 选择 `Get Started` 选项卡后本地 `docker` 详细信息展示

![portainer_local](https://blog.imw7.com/images/Go/docker/portainer_local.png)  

1. 上一步的图形展示，能想得起对应命令吗？

```bash
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          12        2         4.168GB   3.577GB (85%)
Containers      3         3         1.087kB   0B (0%)
Local Volumes   35        1         1.482GB   1.481GB (99%)
Build Cache     0         0         0B        0B
```

### 登陆并演示介绍常用操作案例

#### 安装Nginx

第1步：

![portainer_nginx](https://blog.imw7.com/images/Go/docker/portainer_nginx.png)   

第2步：

![portainer_nginx2](https://blog.imw7.com/images/Go/docker/portainer_nginx2.png)   

第3步：

![portainer_nginx3](https://blog.imw7.com/images/Go/docker/portainer_nginx3.png)   

## Docker容器监控之cAdvisor+InfluxDB+Grafana

### 原生命令

#### 操作

```bash
$ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                                  NAMES
c6ea2ba401f3   redis:6.0.8   "docker-entrypoint.s…"   38 seconds ago   Up 37 seconds   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp              confident_davinci
3c83a749a7d5   mysql:5.7     "docker-entrypoint.s…"   2 minutes ago    Up 2 minutes    0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp   laughing_ardinghelli
```

`docker stats` 命令的结果：

```bash
CONTAINER ID   NAME                   CPU %     MEM USAGE / LIMIT    MEM %     NET I/O       BLOCK I/O     PIDS
c6ea2ba401f3   confident_davinci      0.00%     1000KiB / 7.472GiB   0.01%     4.06kB / 0B   279kB / 0B    1
3c83a749a7d5   laughing_ardinghelli   0.00%     832KiB / 7.472GiB    0.01%     4.28kB / 0B   8.19kB / 0B   1
```

#### 问题

通过 `docker stats` 命令可以很方便的看到当前宿主机上所有容器的 `CPU`，内存以及网络流量等数据，一般小公司够用了。

但是，`docker stats` 统计结果只能是当前宿主机的全部容器，数据资料是实时的，没有地方存储、没有健康指标过线预警等功能。

### 是什么

#### 容器监控3剑客

`cAdvisor` 监控收集 + `InfluxDB` 存储数据 + `Grafana` 展示图标

![cig](https://blog.imw7.com/images/Go/docker/cig.png)

   

##### cAdvisor

`cAdvisor` 是一个容器资源监控工具，包括容器的内存，`CPU`，网络 `IO`，磁盘 `IO` 等监控，同时提供了一个 `WEB` 页面用于查看容器的实时运行状态。`cAdvisor` 默认存储2分钟的数据，而且只是针对单物理机。不过，`cAdvisor` 提供了很多数据集成接口，支持 `InfluxDB`，`Redis`，`Kafka`，`Elasticsearch` 等集成，可以加上对应配置将监控数据发往这些数据库存储起来。

`cAdvisor` 功能主要有两点：

- 展示 `Host` 和容器两个层次的监控数据。
- 展示历史变化数据。

##### InfluxDB

`InfluxDB` 是用 `Go` 语言编写的一个开源分布式时序、事件和指标数据库，无需外部依赖。

`cAdvisor` 默认只在本机保存最近2分钟的数据，为了持久化存储数据和统一收集展示监控数据，需要将数据存储到 `InfluxDB` 中。`InfluxDB` 是一个时序数据库，专门用于存储时序相关数据，很适合存储 `cAdvisor` 的数据。而且，`cAdvisor` 本身已经提供了 `InfluxDB` 的集成方法，启动容器时指定配置即可。

`InfluxDB` 主要功能：

- 基于时间序列，支持与时间有关的相关函数（如最大、最小、求和等）；
- 可度量性：可以实时对大量数据进行计算；
- 基于事件：支持任意的事件数据。

##### Grafana

`Grafana` 是一个开源的数据监控分析可视化平台，支持多种数据源配置（支持的数据源包括 `InfluxDB`，`MySQL`，`Elasticsearch`，`OpenTSDB`，`Graphite`等）和丰富的插件及模板功能，支持图标权限控制和报警。

`Grafana` 主要特性：

- 灵活丰富的图形化选项
- 可以混合多种风格
- 支持白天和夜间模式
- 多个数据源

##### 总结

![cig_summary](https://blog.imw7.com/images/Go/docker/cig_summary.png)   

### compose容器编排

#### 新建目录

```bash
$ cd /data/dockerfile/
$ mkdir cig
$ cd cig
$ pwd
/data/dockerfile/cig
```

#### 新建3件套组合的 docker-compose.yml

```yml
version: '3.1'

volumes:
  grafana_data: {}

services:
  influxdb:
    image: tutum/influxdb:0.9
    restart: always
    environment:
      - PRE_CREATE_DB=cadvisor
    ports:
      - "8083:8083"
      - "8086:8086"
    volumes:
      - ./data/influxdb:/data

  cadvisor:
    image: google/cadvisor
    links:
      - influxdb:influxsrv
    command: -storage_driver=influxdb -storage_driver_db=cadvisor -storage_driver_host=influxsrv:8086
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro

  grafana:
    user: "104"
    image: grafana/grafana
    restart: always
    links:
      - influxdb:influxsrv
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - HTTP_USER=admin
      - HTTP_PASS=admin
      - INFLUXDB_HOST=influxsrv
      - INFLUXDB_PORT=8086
      - INFLUXDB_NAME=cadvisor
      - INFLUXDB_USER=root
      - INFLUXDB_PASS=root
```

#### 启动docker-compose文件

```bash
$ docker compose up -d
...
[+] Running 3/3
 ✔ Container cig-influxdb-1  Started                                              0.3s 
 ✔ Container cig-grafana-1   Started                                              0.4s 
 ✔ Container cig-cadvisor-1  Started                                              0.5s
```

#### 查看三个服务容器是否启动

```bash
$ docker ps
CONTAINER ID   IMAGE                 COMMAND                  CREATED        STATUS         PORTS                                                                                            NAMES
cbc0cbdf3ed3   grafana/grafana      "/run.sh"                2 minutes ago   Up 2 minutes   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp                                                  cig-grafana-1
b54a3397a0ce   google/cadvisor      "/usr/bin/cadvisor -…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp                                                  cig-cadvisor-1
6d0afcf4350d   tutum/influxdb:0.9   "/run.sh"                2 minutes ago   Up 2 minutes   0.0.0.0:8083->8083/tcp, :::8083->8083/tcp, 0.0.0.0:8086->8086/tcp, :::8086->8086/tcp       cig-influxdb-1
```

#### 测试

##### 浏览 `cAdvisor` **收集**服务，`http://ip:8080/`

第一次访问慢，请稍等。`cAdvisor` 也有基础的图形展现功能，这里主要用它来做数据采集。

![cadvisor](https://blog.imw7.com/images/Go/docker/cadvisor.png)   

##### 浏览 `InfluxDB` **存储**服务，`http://ip:8083/`

##### 浏览 `Grafana` **展现**服务，`http://ip:3000/`

用 `ip:3000` 的方式访问，默认账户密码均为 `admin`。

![grafana_login](https://blog.imw7.com/images/Go/docker/grafana_login.png) 

配置步骤：

1、配置数据源

![grafana1](https://blog.imw7.com/images/Go/docker/grafana1.png)

2、选择 `InfluxDB` 数据源

![grafana2](https://blog.imw7.com/images/Go/docker/grafana2.png)   

3、配置细节

① 填入 `Name` 和 `URL`：

![grafana3](https://blog.imw7.com/images/Go/docker/grafana3.png)   

② 填入数据，点击 `Save & test` 按钮：

![grafana4](https://blog.imw7.com/images/Go/docker/grafana4.png)  

4、配置面板

① 创建仪表盘

![grafana5](https://blog.imw7.com/images/Go/docker/grafana5.png)  

② 选择新增面盘

![grafana6](https://blog.imw7.com/images/Go/docker/grafana6.png)   

③ 选择仪表盘样式

![grafana7](https://blog.imw7.com/images/Go/docker/grafana7.png)  

④ 选择 `Graph (old)` 作为仪表盘样式

![grafana8](https://blog.imw7.com/images/Go/docker/grafana8.png)   

⑤ 填写相应的数据

![grafana9](https://blog.imw7.com/images/Go/docker/grafana9.png)   

⑥ 完成

![grafana10](https://blog.imw7.com/images/Go/docker/grafana10.png)   

到这里 `cAdvisor` + `InfluxDB` + `Grafana` 容器监控系统就部署完成了。