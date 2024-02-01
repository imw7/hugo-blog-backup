---
title: Go语言操作Redis
tags: [Redis, 数据库]
categories: [Go]
date: 2019-07-27 00:00:00
toc: true
---

在项目开发中`Redis`的使用也比较频繁，本文介绍了`Go`语言中`go-redis`库的基本使用。<!--more-->

# Redis介绍

`Redis`是一个开源的内存数据库，`Redis`提供了多种不同类型的数据结构，很多业务场景下的问题都可以很自然地映射到这些数据结构上。除此之外，通过复制、持久化和客户端分片等特性，我们可以很方便地将`Redis`扩展成一个能够包含数百`GB`数据、每秒处理上百万次请求的系统。

## Redis支持的数据结构

`Redis`支持诸如字符串（`strings`）、哈希（`hashes`）、列表（`lists`）、集合（`sets`）、带范围查询的排序集合（`sorted sets`）、位图（`bitmaps`）、`hyperloglogs`、带半径查询和流的地理空间索引等数据结构（`geospatial indexes`）。

## Redis应用场景

- 缓存系统，减轻主数据库（`MySQL`）的压力。
- 计数场景，比如微博、抖音中的关注数和粉丝数。
- 热门排行榜，需要排序的场景特别适合使用`ZSET`。
- 利用`LIST`可以实现队列的功能。

## 准备Redis环境

### Linux源码安装

从下载地址：https://redis.io/download，下载最新稳定版本 `Redis`。

以 `6.2.6` 版本为例，下载并安装：

```bash
$ wget http://download.redis.io/releases/redis-6.2.6.tar.gz
$ tar xzf redis-6.2.6.tar.gz
$ cd redis-6.2.6
$ make
```

完成上述步骤后，在 `src` 目录中会出现编译好的`redis-server` 和 `redis-cli` 等可执行文件。启动`redis` 服务的命令：

```bash
$ src/redis-server
```

用编译好的`Redis`客户端 `redis-cli`和 `Redis` 服务端 `redis-server` 进行交互：

```bash
$ src/redis-cli
redis> set foo bar
OK
redis> get foo
"bar"
```

### Ubuntu安装

可以通过`PPA`安装，将 `redislabs/redis`包仓库添加到`apt` 索引中，更新并且安装。

```bash
$ sudo add-apt-repository ppa:redislabs/redis
$ sudo apt-get update
$ sudo apt-get install redis
```

### Docker安装

`docker`启动一个名为`redis626`的`6.2.6版本`的`redis server`示例：

```bash
docker run --name redis626 -p 6379:6379 -d redis:6.2.6
```

**注意**：此处的版本、容器名和端口号请根据自己需要设置。

启动一个`redis-cli`连接上面的`redis server`:

```bash
docker run -it --network host --rm redis:6.2.6 redis-cli
```

## Redis基本使用

### 修改配置文件

```bash
$ vim /etc/redis/redis.conf
```

 将 `bind 127.0.0.1 -::1` 修改为当前主机 `IP` 地址，如：`192.168.6.108`。

### 端口

`Redis`默认端口为`6379`，当然可以通过配置文件修改默认端口。

### 开启Redis

```bash
$ redis-server
```

验证`Redis`的`ip`和`port`：

```bash
$ ps xau | grep redis
```

### 连接Redis

通过`redis-cli`来实现数据库的连接：

```bash
$ redis-cli -h 127.0.0.1 -p 6379
```

或者直接：

```bash
$ redis-cli
```

也是可以的。

### 添加数据

添加一条数据可以使用 `set key value` 命令来实现，如：

```bash
redis> set foo bar
OK
```

### 获取数据

想要获取数据可以通过 `get key` 命令来实现，如：

```bash
redis> get foo
"bar"
```

### 查看所有

```bash
redis> keys *
```

### 删除所有

```bash
redis> flushall
```

# go-redis库

## 安装

区别于另一个比较常用的`Go`语言`redis` 的`client`库：[redigo](https://github.com/gomodule/redigo)，我们这里采用https://github.com/go-redis/redis连接`Redis`数据库并进行操作，因为`go-redis`支持连接哨兵及集群模式的`Redis`。

使用以下命令下载并安装:

```bash
go get -u github.com/go-redis/redis
```

## 连接

### 普通连接

```go
// 声明一个全局的rdb变量
var rdb *redis.Client

// 初始化连接
func initClient() (err error) {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})

	_, err = rdb.Ping().Result()
	if err != nil {
		return err
	}
	return nil
}
```

## V8新版本相关

最新版本的`go-redis`库的相关命令都需要传递`context.Context`参数，例如：

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/go-redis/redis/v8" // 注意导入的是新版本
)

var rdb *redis.Client

// 初始化连接
func initClient() (err error) {
	rdb = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",  // no password set
		DB:       0,   // use default DB
		PoolSize: 100, // 连接池大小
	})

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	_, err = rdb.Ping(ctx).Result()
	return err
}

func V8Example() {
	ctx := context.Background()
	if err := initClient(); err != nil {
		return
	}

	err := rdb.Set(ctx, "key", "value", 0).Err()
	if err != nil {
		panic(err)
	}

	val, err := rdb.Get(ctx, "key").Result()
	if err != nil {
		panic(err)
	}
	fmt.Println("key", val)

	val2, err := rdb.Get(ctx, "key2").Result()
	if err == redis.Nil {
		fmt.Println("key2 does not exist")
	} else if err != nil {
		panic(err)
	} else {
		fmt.Println("key2", val2)
	}
	// Output: key value
	// key2 does not exist
}
```

### 连接Redis哨兵模式

```go
func initClient()(err error){
	rdb := redis.NewFailoverClient(&redis.FailoverOptions{
		MasterName:    "master",
		SentinelAddrs: []string{"x.x.x.x:26379", "xx.xx.xx.xx:26379", "xxx.xxx.xxx.xxx:26379"},
	})
	_, err = rdb.Ping().Result()
	if err != nil {
		return err
	}
	return nil
}
```

### 连接Redis集群

```go
func initClient()(err error){
	rdb := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs: []string{":7000", ":7001", ":7002", ":7003", ":7004", ":7005"},
	})
	_, err = rdb.Ping().Result()
	if err != nil {
		return err
	}
	return nil
}
```

## 基本使用

### set/get示例

```go
func redisExample() {
	ctx := context.Background()
	err := rdb.Set(ctx, "score", 100, 0).Err()
	if err != nil {
		fmt.Println("set score failed, err:", err)
		return
	}

	val, err := rdb.Get(ctx, "score").Result()
	if err != nil {
		fmt.Println("get score failed, err:", err)
		return
	}
	fmt.Println("score", val)

	val2, err := rdb.Get(ctx, "name").Result()
	if err == redis.Nil {
		fmt.Println("name does not exist")
	} else if err != nil {
		fmt.Println("get name failed, err:", err)
		return
	} else {
		fmt.Println("name", val2)
	}
}
```

### zset示例

```go
func redisExample2() {
	ctx := context.Background()
	zsetKey := "language_rank"
	languages := []*redis.Z{
		{Score: 90.0, Member: "Golang"},
		{Score: 98.0, Member: "Java"},
		{Score: 95.0, Member: "Python"},
		{Score: 97.0, Member: "JavaScript"},
		{Score: 99.0, Member: "C/C++"},
	}
	// ZADD 用于将一个或多个成员元素及其分数值加入到有序集当中
	num, err := rdb.ZAdd(ctx, zsetKey, languages...).Result()
	if err != nil {
		fmt.Println("zadd failed, err:", err)
		return
	}
	fmt.Printf("zadd %d succ.\n", num)

	// 把Golang的分数加10
	newScore, err := rdb.ZIncrBy(ctx, zsetKey, 10.0, "Golang").Result()
	if err != nil {
		fmt.Println("zincrby failed, err:", err)
		return
	}
	fmt.Printf("Golang's score is %.1f now.\n", newScore)

	// 取分数最高的3个
	ret, err := rdb.ZRevRangeWithScores(ctx, zsetKey, 0, 2).Result()
	if err != nil {
		fmt.Println("zrevrange failed, err:", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}

	// 取95~100分的
	op := &redis.ZRangeBy{
		Min: "95",
		Max: "100",
	}
	ret, err = rdb.ZRangeByScoreWithScores(ctx, zsetKey, op).Result()
	if err != nil {
		fmt.Println("zrangebyscore failed, err:", err)
		return
	}
	for _, z := range ret {
		fmt.Println(z.Member, z.Score)
	}
}
```

输出结果如下：

```bash
$ ./redis_demo 
zadd 0 succ.
Golang's score is 100.0 now.
Golang 100
C/C++ 99
Java 98
JavaScript 97
Java 98
C/C++ 99
Golang 100
```

### Pipeline

`Pipeline` 主要是一种网络优化。它本质上意味着客户端缓冲一堆命令并一次性将它们发送到服务器。这些命令不能保证在事务中执行。这样做的好处是节省了每个命令的网络往返时间（`RTT`）。

`Pipeline` 基本示例如下：

```go
pipe := rdb.Pipeline()

incr := pipe.Incr("pipeline_counter")
pipe.Expire("pipeline_counter", time.Hour)

_, err := pipe.Exec()
fmt.Println(incr.Val(), err)
```

上面的代码相当于将以下两个命令一次发给redis server端执行，与不使用`Pipeline`相比能减少一次RTT。

```bash
INCR pipeline_counter
EXPIRE pipeline_counts 3600
```

也可以使用`Pipelined`：

```go
var incr *redis.IntCmd
_, err := rdb.Pipelined(func(pipe redis.Pipeliner) error {
	incr = pipe.Incr("pipelined_counter")
	pipe.Expire("pipelined_counter", time.Hour)
	return nil
})
fmt.Println(incr.Val(), err)
```

在某些场景下，当我们有多条命令要执行时，就可以考虑使用`pipeline`来优化。

### 事务

`Redis`是单线程的，因此单个命令始终是原子的，但是来自不同客户端的两个给定命令可以依次执行，例如在它们之间交替执行。但是，`Multi/exec`能够确保在`multi/exec`两个语句之间的命令之间没有其他客户端正在执行命令。

在这种场景我们需要使用`TxPipeline`。`TxPipeline`总体上类似于上面的`Pipeline`，但是它内部会使用`MULTI/EXEC`包裹排队的命令。例如：

```go
pipe := rdb.TxPipeline()

incr := pipe.Incr("tx_pipeline_counter")
pipe.Expire("tx_pipeline_counter", time.Hour)

_, err := pipe.Exec()
fmt.Println(incr.Val(), err)
```

上面代码相当于在一个RTT下执行了下面的redis命令：

```bash
MULTI
INCR pipeline_counter
EXPIRE pipeline_counts 3600
EXEC
```

还有一个与上文类似的`TxPipelined`方法，使用方法如下：

```go
var incr *redis.IntCmd
_, err := rdb.TxPipelined(func(pipe redis.Pipeliner) error {
	incr = pipe.Incr("tx_pipelined_counter")
	pipe.Expire("tx_pipelined_counter", time.Hour)
	return nil
})
fmt.Println(incr.Val(), err)
```

### Watch

在某些场景下，我们除了要使用`MULTI/EXEC`命令外，还需要配合使用`WATCH`命令。在用户使用`WATCH`命令监视某个键之后，直到该用户执行`EXEC`命令的这段时间里，如果有其他用户抢先对被监视的键进行了替换、更新、删除等操作，那么当用户尝试执行`EXEC`的时候，事务将失败并返回一个错误，用户可以根据这个错误选择重试事务或者放弃事务。

```go
Watch(fn func(*Tx) error, keys ...string) error
```

`Watch`方法接收一个函数和一个或多个`key`作为参数。基本使用示例如下：

```go
// 监视watch_count的值，并在值不变的前提下将其值+1
key := "watch_count"
err = client.Watch(func(tx *redis.Tx) error {
	n, err := tx.Get(key).Int()
	if err != nil && err != redis.Nil {
		return err
	}
	_, err = tx.Pipelined(func(pipe redis.Pipeliner) error {
		pipe.Set(key, n+1, 0)
		return nil
	})
	return err
}, key)
```

最后看一个官方文档中使用`GET`和`SET`命令以事务方式递增`Key`的值的示例：

```go
const routineCount = 100

increment := func(key string) error {
	txf := func(tx *redis.Tx) error {
		// 获得当前值或零值
		n, err := tx.Get(key).Int()
		if err != nil && err != redis.Nil {
			return err
		}

		// 实际操作（乐观锁定中的本地操作）
		n++

		// 仅在监视的Key保持不变的情况下运行
		_, err = tx.Pipelined(func(pipe redis.Pipeliner) error {
			// pipe 处理错误情况
			pipe.Set(key, n, 0)
			return nil
		})
		return err
	}

	for retries := routineCount; retries > 0; retries-- {
		err := rdb.Watch(txf, key)
		if err != redis.TxFailedErr {
			return err
		}
		// 乐观锁丢失
	}
	return errors.New("increment reached maximum number of retries")
}

var wg sync.WaitGroup
wg.Add(routineCount)
for i := 0; i < routineCount; i++ {
	go func() {
		defer wg.Done()

		if err := increment("counter3"); err != nil {
			fmt.Println("increment error:", err)
		}
	}()
}
wg.Wait()

n, err := rdb.Get("counter3").Int()
fmt.Println("ended with", n, err)
```

更多详情请查阅[文档](https://godoc.org/github.com/go-redis/redis)。