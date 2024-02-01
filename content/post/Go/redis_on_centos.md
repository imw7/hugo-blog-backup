---
title: CentOS与Redis
tags: [Redis, centos]
categories: [服务器]
abbrlink: about_redis
toc: true
date: 2022-01-12T12:05:23+08:00
draft: false
---

本文介绍如何在 `CentOS` 上安装、配置、启动、关闭和卸载 `Redis`。<!--more-->

## 安装

### 下载Redis安装包

本文介绍两种方法：

1. 在 `Redis` 官网下载最新版本的 [redis-6.2.6](https://download.redis.io/releases/redis-6.2.6.tar.gz)。

2. 通过命令下载：

```bash
$ wget http://download.redis.io/releases/redis-6.2.6.tar.gz
```

### 安装Redis编译的C环境

由于 `Redis` 是由 `C` 语言写成的，所以需要 `gcc` 编译工具。

```bash
$ yum install gcc-c++
```

查看 `gcc` 是否安装，可以使用：

```bash
$ rpm -q gcc
```

如果出现类似下面的输出，则说明 `gcc` 已经正确安装在 `CentOS` 服务器上了。

```bash
$ rpm -q gcc
gcc-8.5.0-7.el8.x86_64
```

### 解压安装包并进入文件夹

1. 解压安装包

```bash
$ tar -zxvf redis-6.2.6.tar.gz
```

2. 进入 `redis-6.2.6` 目录

```bash
$ cd redis-6.2.6
```

### 编译Redis

安装包中都是 `C` 语言编写的源程序，需要使用 `make` 命令进行编译。

```bash
$ make
...
Hint: It's a good idea to run 'make test' ;)
make[1]: Leaving directory '/home/imw7/pkg/redis-6.2.6/src'
```

出现类似输出结果则表示  `make` 成功。

### 安装编译后的文件到指定目录

编译好之后，需要将其安装到指定的目录，方便使用。

```bash
$ make PREFIX=/usr/local/redis install
```

**注意**：`PREFIX` 必须大写，同时会自动为我们生成 `redis` 目录，并将结果安装到此目录。

安装完成之后，在 `/usr/local/redis/bin` 目录下会生成几个文件：

```bash
bin
├── redis-benchmark
├── redis-check-aof -> redis-server
├── redis-check-rdb -> redis-server
├── redis-cli
├── redis-sentinel -> redis-server
└── redis-server
```

## 配置

### 将配置文件移动到指定位置

1. 创建目录 `etc`

```bash
$ mkdir /usr/local/redis/etc
```

2. 复制配置文件到 `etc` 目录

```bash
$ cd redis-6.2.6
$ cp redis.conf /usr/local/redis/etc
```

3. 进入 `/usr/local/redis` 修改 `redis.conf` 配置文件

```bash
$ vim redis.conf
```

### Redis配置默认必须修改

在 `vim` 中，使用 `/内容` 来搜索要找的内容，如 `/daemonize`，可快速定位要找的位置。

* 配置 `Redis` 为后台启动

```bash
daemonize no # 修改为 daemonize yes
```

* **开启外网访问**

```bash
bind 127.0.0.1 # 将这行注释掉，在最前面加“#”，即可注释
```

* 配置密码

```bash
requirepass password
```

## Redis启动

在执行命令之前，首先关闭防火墙，否则可能导致无法连接数据库。

```bash
$ systemctl stop firewalld
```

### 服务端启动

```bash
$ /usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf
```

### 查看进程

```bash
$ ps -ef | grep redis
root        1267       1  0 11:58 ?        00:00:03 /usr/local/redis/bin/redis-server *:6379
root        1857    1763  0 12:46 pts/0    00:00:00 grep --color=auto redis
```

### 客户端启动

#### 本地启动

```bash
$ cd /usr/local/redis/bin
$ ./redis-cli # 没密码
$ ./redis-cli -a password # 有密码
$ exit # 退出
```

#### 远程调用服务

```bash
$ redis-cli -h host -p port -a password  # redis-cli -h IP地址 -p 端口 -a 密码
```

#### go代码设置必须设置密码才能连接

```go
import (
	"fmt"
	"github.com/gomodule/redigo/redis"
	"log"
)

func main() {
	// 1.连接数据库
	conn, err := redis.Dial("tcp", "127.0.0.1:6379")
	if err != nil {
		log.Fatal("redis dial failed, err:", err)
	}
	defer func(conn redis.Conn) {
		err := conn.Close()
		if err != nil {
			return
		}
	}(conn)

	// 2.设置密码
	if _, err := conn.Do("AUTH", "password"); err != nil {
		log.Fatal(err)
	}

	// 3.操作数据库
	reply, err := conn.Do("set", "foo", "bar")

	// 4.回复助手类函数，确定成具体的数据类型
	str, err := redis.String(reply, err)

	fmt.Println(str, err)
}
```

## Redis关闭

第一种关闭方式（断电、非正常关闭，容易数据丢弃）：

```bash
$ ps -ef | grep -i redis # 查询PID
$ kill -9 PID
```

第二种关闭方式（正常关闭，数据保存）：

```bash
$ ./bin/redis-cli shutdown
```

## 卸载

1. 先把 `Redis` 服务关闭
2. 再把 `/usr/local/redis/bin/` 目录下的与 `Redis` 相关的文件删除即可。
