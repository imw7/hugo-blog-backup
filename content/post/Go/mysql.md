---
title: '在CentOS7.x上安装MySQL8.0'
tags: [centos, mysql]
categories: [服务器]
date: "2022-01-13T15:31:46+08:00"
toc: true
abbrlink: centos_mysql
draft: false
---

本文介绍在`CentOS`服务器上安装`MySQL8.0`，以及重置`MySQL8.0`的密码。<!--more-->

## 安装和配置MySQL

### 下载yum源的安装包

```bash
$ yum install -y https://repo.mysql.com//mysql80-community-release-el7-5.noarch.rpm
```

### 安装

```bash
$ yum install -y mysql-community-server
```

### 启动服务

```bash
$ service mysqld start
```

### 查看状态

```bash
$ service mysqld status
```

出现以下相似内容，则说明`MySQL`安装和启动正确。

```bash
Redirecting to /bin/systemctl status mysqld.service
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2022-01-18 15:57:24 CST; 25min ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 2639 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 2710 (mysqld)
   Status: "Server is operational"
   CGroup: /system.slice/mysqld.service
           └─2710 /usr/sbin/mysqld

Jan 18 15:57:14 centos systemd[1]: Starting MySQL Server...
Jan 18 15:57:24 centos systemd[1]: Started MySQL Server.
```

### 查看初始密码

```bash
$ grep 'temporary password' /var/log/mysqld.log
```

会出现类似的输出：

```bash
2022-01-18T07:57:18.914129Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: =6CVwAogkz>F
```

### 登陆

```bash
$ mysql -uroot -p
```

在 `Enter password:` 后面填写初始密码，回车之后即可登陆。

登陆之后，必须修改密码才能对数据库进行进一步的操作，密码必须要有数字、大小写字母和特殊符号。

```mysql
mysql> ALTER user 'root'@'localhost' IDENTIFIED by 'My_passw0rd';
```

输入`exit`之后，重新登陆验证密码是否修改成功。

### 远程连接

创建远程访问用户，一次执行以下命令：

```mysql
mysql> CREATE user 'root'@'%' IDENTIFIED with mysql_native_password by 'My_passw0rd';
Query OK, 0 rows affected (0.01 sec)

mysql> grant all privileges on *.* to 'root'@'%' with grant option;
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```

## 另外一种配置方法

### 配置文件MySQL免密码登陆

在终端输入以下命令，打开 `my.cnf` 文件，编辑 `MySQL` 的配置文件：

```bash
$ vim /etc/my.cnf
```

在 `[mysqld]` 下第一行添加 `skip-grant-tables`，实现免密码登陆：

```mysql
...
[mysqld]
skip-grant-tables
port            = 3306
socket          = /tmp/mysql.sock
...
```

然后输入 `:wq` 保存退出。

**注意**：除了免密码登陆之外，还可以通过命令查看 `MySQL` 初始密码。

```bash
$ grep 'temporary password' /var/log/mysqld.log
```

### 重启MySQL服务

输入以下命令即可重启服务：

```bash
$ service mysqld restart
```

### 免密码登陆到MySQL上

```bash
$ mysql -uroot -p
```

提示输入密码时，直接敲回车就可以进入数据库了。

### 选择MySQL数据库

使用 `mysql` 数据库：

```mysql
mysql> use mysql;
```

因为 `mysql` 数据库中存储了一张 `MySQL` 用户的 `user`  表。

### 当前root用户的相关信息

在 `mysql` 数据库的 `user` 表中查看当前 `root` 用户的相关信息：

```mysql
mysql> select host, user, authentication_string, plugin from user;
```

执行完上面的命令后会显示一个表格：

| host      | user             | authentication_string | plugin                |
| --------- | ---------------- | --------------------- | --------------------- |
| localhost | mysql.infoschema | $A$005$...USED        | caching_sha2_password |
| localhost | mysql.session    | $A$005$...USED        | caching_sha2_password |
| localhost | mysql.sys        | $A$005$...USED        | caching_sha2_password |
| localhost | root             | *2470C0C06DEE42...    | mysql_native_password |

表格中有以下信息：

`host`：允许用户登陆的 `ip` 的「位置」，`%` 表示可以远程连接；

`user`：当前数据库的用户名；

`authentication_string`：用户密码（在 `mysql 5.7.9` 以后废弃了 `password` 字段和 `password()` 函数）；

`plugin`：密码加密方式；

### 将默认的root密码置空

```mysql
mysql> use mysql;
mysql> update user set authentication_string='' where user='root';
```

### 退出MySQL命令行

```mysql
mysql> quit;
```

### 修改 /etc/my.cnf 文件

```bash
$ vim /etc/my.cnf
```

注销或删除`[mysqld]` 下的 `skip-grant-tables` 并保存退出。

### 重启MySQL服务

```bash
$ service mysqld restart
```

### 重新登陆到MySQL上

```bash
$ mysql -uroot -p
```

提示输入密码时直接敲回车，因为刚才已经将密码置为空了。

### 使用ALTER修改root用户密码

```mysql
ALTER user 'root'@'localhost' IDENTIFIED BY 'my_password';
```

其中，`my_password` 为设置的新密码。

执行完之后如果提示 `OK` 的话，就代表修改成功了，至此重置密码也就算是完成了。
