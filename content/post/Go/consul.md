---
title: Consul简介
tags: [consul, docker, go-micro]
categories: [Go]
date: 2023-02-08T23:47:46+08:00
toc: true
---

写微服务项目的过程中用到了 `Consul` 作为服务发现工具，本文作为对 `Consul` 的总结。<!--more--> 

## 概念

`Consul` 是 `HashiCorp` 公司推出的开源工具，`Consul` 由 `Go` 语言开发，部署起来非常容易，只需要极少的可执行程序和配置文件，具有绿色、轻量级的特点。`Consul` 是 `分布式` 的、`可高用` 的、`可横向扩展` 的，用于实现分布式系统的服务发现与配置。

### 特点

* 服务发现（`Service Discovery`）：`Consul` 提供了通过`DNS`或者`HTTP`接口的方式来注册服务和发现服务。一些外部的服务通过`Consul`很容易的找到它所依赖的服务。
* 健康检查（`Health Checking`）：`Consul` 的 `Client` 可以提供任意数量的健康检查，即可以与给定的服务相关联（`webserver是否返回200OK`），也可以与本地节点相关联（`内存利用率是否低于90%`）。操作员可以使用这些信息来监视集群的健康状况，服务发现组件可以使用这些信息将流量从不健康的主机路由出去。
* `Key/Value`存储：应用程序可以根据自己的需要使用 `Consul` 提供的 `Key/Value` 存储。 `Consul` 提供了简单易用的 `HTTP` 接口，结合其他工具可以实现动态配置、功能标记、领袖选举等等功能。
* 安全服务通信：`Consul` 可以为服务生成和分发`TLS`证书，以建立相互的`TLS`连接。意图可用于定义允许哪些服务通信。服务分割可以很容易地进行管理，其目的是可以实时更改的，而不是使用复杂的网络拓扑和静态防火墙规则。
* 多数据中心：`Consul` 支持开箱即用的多数据中心。这意味着用户不需要担心需要建立 额外的抽象层让业务扩展到多个区域。

### Consul 架构图

![consul_structure](https://blog.imw7.com/images/Go/consul/consul_structure.png)

首先，可以看到有两个数据中心，分别标记为 `DATACENTER 1` 和 `DATACENTER 2`。`Consul` 拥有对多个数据中心的一流支持，这是比较常见的情况。

在每个数据中心中，都有客户机和服务器。预计将有三到五台服务器。这在故障情况下的可用性和性能之间取得了平衡，因为随着添加更多的机器，一致性会逐渐变慢。但是，客户端的数量没有限制，可以很容易地扩展到数千或数万。

`Consul` 实现多个数据中心都依赖于 `gossip protocol` 协议。这样做有几个目的：首先，不需要使用服务器的地址来配置客户端，服务发现是自动完成的。其次，健康检查故障的工作不是放在服务器上，而是分布式的。这使得故障检测比单纯的心跳模式更具可伸缩性。为节点提供故障检测;如果无法访问代理，则节点可能经历了故障。

每个数据中心中的服务器都是一个筏对等集的一部分。这意味着它们一起工作来选举单个 `leader`，一个被选中的服务器有额外的职责。领导负责处理所有的查询和事务。事务还必须作为协商一致协议的一部分复制到所有对等方。由于这个需求，当非 `leader` 服务器接收到 `RPC` 请求时，它会将其转发给集群 `leader`。

### Consul 的使用场景
`Consul` 的应用场景包括服务发现、服务隔离、服务配置：

* 服务发现场景中`consul`作为注册中心，服务地址被注册到`consul`中以后，可以使用`consul`提供的`dns`、`http`接口查询，`consul`支持`health check`。

* 服务隔离场景中`consul`支持以服务为单位设置访问策略，能同时支持经典的平台和新兴的平台，支持`tls`证书分发，`service-to-service`加密。

* 服务配置场景中`consul`提供`key-value`数据存储功能，并且能将变动迅速地通知出去，借助`Consul`可以实现配置共享，需要读取配置的服务可以从`Consul`中读取到准确的配置信息。

* `Consul`可以帮助系统管理者更清晰的了解复杂系统内部的系统架构，运维人员可以将`Consul`看成一种监控软件，也可以看成一种资产（资源）管理系统。

## 安装

`Docker` 安装并启动：

```bash
$ docker run --name consul -d -p 8500:8500 -p 8300:8300 -p 8301:8301 -p 8302:8302 -p 8600:8600 consul agent -server -bootstrap-expect=1 -ui -bind=0.0.0.0 -client=0.0.0.0
```

`Ubuntu` 直接运行如下命令即可：

```bash
$ wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
$ echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
$ sudo apt update && sudo apt install consul
```

`CentOS` 安装命令：

```bash
$ sudo yum install -y yum-utils
$ sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
$ sudo yum -y install consul
```

在 `Terminal` 终端中输出命令

```bash
$ consul --version
```

出现了类似于如下的结果，则表示安装成功了。

```bash
Consul v1.10.4
Revision 7bbad6fe
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
```

在 `Terminal` 终端中输入如下命令即可启动 `consul` 了。

```bash
$ consul agent -dev
```

**注**：如果想让外网也能访问consul，可以使用如下命令：

```bash
$ consul agent -dev -client 0.0.0.0 -ui
```

通过以上命令，可以实现指定 `IP` 访问。

### Consul参数详解

| 参数                    | 作用                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `-net=host`             | `docker`参数, 使得`docker`容器越过了`net namespace`的隔离，免去手动指定端口映射的步骤 |
| `-server`               | `consul`支持以`server`或`client`的模式运行, `server`是服务发现模块的核心, `client`主要用于转发请求 |
| `-advertise`            | 将本机私有`IP`传递到`consul`                                 |
| `-retry-join`           | 指定要加入的`consul`节点地址，失败后会重试, 可多次指定不同的地址 |
| `-client`               | 指定`consul`绑定在哪个`client`地址上，这个地址可提供`HTTP`、`DNS`、`RPC`等服务，默认是`127.0.0.1` |
| `-bind`                 | 绑定服务器的`ip`地址；该地址用来在集群内部的通讯，集群内的所有节点到地址必须是可达的，默认是`0.0.0.0` |
| `allow_stale`           | 设置为`true`则表明可从`consul`集群的任一`server`节点获取`dns`信息, `false`则表明每次请求都会经过`consul`的`server leader` |
| `-bootstrap-expect`     | 数据中心中预期的服务器数。指定后，`Consul`将等待指定数量的服务器可用，然后启动群集。允许自动选举`leader`，但不能与传统 -`bootstrap` 标志一起使用, 需要在`server`模式下运行。 |
| `-data-dir`             | 数据存放的位置，用于持久化保存集群状态                       |
| `-node`                 | 群集中此节点的名称，这在群集中必须是唯一的，默认情况下是节点的主机名 |
| `-config-dir`           | 指定配置文件，当这个目录下有 `.json` 结尾的文件就会被加载    |
| `-enable-script-checks` | 检查服务是否处于活动状态，类似开启心跳                       |
| `-datacenter`           | 数据中心名称                                                 |
| `-ui`                   | 开启 `ui` 界面                                               |
| `-ip`                   | 指定`ip`，加入到已有的集群中                                 |

运行成功后，访问 http://localhost:8500 就可以到 `consul` 自带的 `UI` 界面了，如图：

![consul](https://blog.imw7.com/images/Go/go_micro/consul_init.png)

### 关键代码

`main.go`的关键代码如下：

```go
package main

import (
	"github.com/go-micro/plugins/v4/registry/consul"
    "go-micro-demo/registerConf/config"
	"go-micro-demo/registerConf/handler"
	"go-micro-demo/registerConf/pb"
	"go-micro.dev/v4"
    "go-micro.dev/v4/logger"
	"go-micro.dev/v4/registry"
)

func main() {
	// register consul
	reg := consul.NewRegistry(func(options *registry.Options) {
		options.Addrs = []string{"127.0.0.1:8500"}
	})
    
    // ...

	// create service
	srv := micro.NewService(
		micro.Name("registerConf"),
		micro.Version("latest"),
		// 注册consul中心
		micro.Registry(reg),
	)

	// register handler
	if err := pb.RegisterRegisterConfHandler(srv.Server(), &handler.RegisterConf{}); err != nil {
		logger.Fatal(err)
	}

	// run service
	if err := srv.Run(); err != nil {
		logger.Fatal(err)
	}
}
```

`go-micro v4` 提供 `plugins`，只需要引入并创建实例之后，使用 `micro.Registry` 注册即可。

运行后效果图如下：

![registerConf](https://blog.imw7.com/images/Go/go_micro/registerConf.png)

可以看出名为 `registerConf` 的服务出现在了页面上。

### 配置中心

点击 `Key/Value`创建目录`micro/config`，然后在`config`目录分别创建`mysql`、`redis`、`logger`和`server`四个`key`，如下图所示：

![config center](https://blog.imw7.com/images/Go/go_micro/config_center.png)

以其中 `mysql` 为例，输入信息：

![mysql info](https://blog.imw7.com/images/Go/go_micro/mysql_info.png)

然后在项目中的 `registerConf` 文件夹下创建 `config` 目录，并创建 `config.go`、`mysql.go` 文件，分别编写其中代码。

#### config.go

```go
package config

import (
	"github.com/go-micro/plugins/config/source/consul/v4"
	"go-micro.dev/v4/config"
	"strconv"
)

// GetConsulConfig 设置配置中心
func GetConsulConfig(host string, port int64, prefix string) (config.Config, error) {
	// 添加配置中心
	// 配置中心使用consul key/value模式
	consulSource := consul.NewSource(
		// 设置配置中心地址
		consul.WithAddress(host+":"+strconv.FormatInt(port, 10)),
		// 设置前缀，不设置默认为 /micro/config
		consul.WithPrefix(prefix),
		// 是否移除前缀，这里设置为true：表示可以不带前缀直接获取对应配置
		consul.StripPrefix(true),
	)
	// 配置初始化
	conf, err := config.NewConfig()
	if err != nil {
		return conf, err
	}
	err = conf.Load(consulSource)
	return conf, err
}
```

#### mysql.go

```go
package config

import "go-micro.dev/v4/config"

// MySQLConfig 创建结构体
type MySQLConfig struct {
	Host     string `json:"host"`
	User     string `json:"user"`
	Pwd      string `json:"pwd"`
	Database string `json:"database"`
	Port     int64  `json:"port"`
}

// GetMySQLFromConsul 获取mysql的配置
func GetMySQLFromConsul(config config.Config, path ...string) (*MySQLConfig, error) {
	mysqlConfig := &MySQLConfig{}
	// 获取配置
	if err := config.Get(path...).Scan(mysqlConfig); err != nil {
		return nil, err
	}
	return mysqlConfig, nil
}
```

### main.go

最后在主函数中插入如下代码，即可实现在启动服务之前就可获取配置中心的配置信息。

```go
func main() {	
	// register consul
    // ...
    
    // 配置中心
	consulConfig, err := config.GetConsulConfig("127.0.0.1", 8500, "/micro/config")
	if err != nil {
		logger.Fatal(err)
	}

	// MySQL配置信息
	info, err := config.GetMySQLFromConsul(consulConfig, "mysql")
	if err != nil {
		logger.Fatal(err)
	}
	logger.Info("MySQL配置信息:", info)
    
    // create service
    // ...
}
```

运行主函数之后，可以成功获取刚在 `consul` 页面输入的 `mysql` 配置信息，如图：

![get_conf](https://blog.imw7.com/images/Go/go_micro/get_conf.png)

## consul和grpc结合使用

如何将 `grpc` 服务注册到 `consul` 中呢？

`hello.proto`

```protobuf
syntax = "proto3";

package pb;
option go_package = "../pb";

message Request {
  string name = 1;
}

message Response {
  string msg = 1;
}

service HelloService {
  rpc Hello (Request) returns (Response);
}
```

服务端`server.go`

```go
package main

import (
	"context"
	"github.com/hashicorp/consul/api"
	"google.golang.org/grpc"
	"grpc-consul/pb"
	"log"
	"net"
)

// 把grpc服务注册到consul上面

type HelloService struct{}

// Hello 绑定方法，实现接口
func (h *HelloService) Hello(_ context.Context, req *pb.Request) (*pb.Response, error) {
	return &pb.Response{Msg: "Hello, " + req.Name + "!"}, nil
}

func main() {
	// 初始化consul配置，客户端服务器需要一致
	consulConfig := api.DefaultConfig()

	// 获取consul操作对象
	registry, err := api.NewClient(consulConfig)
	if err != nil {
		log.Fatal(err)
	}

	// 注册服务，服务的常规配置
	registerService := api.AgentServiceRegistration{
		ID:      "1",
		Name:    "HelloService",
		Tags:    []string{"grpc", "consul"},
		Port:    1234,
		Address: "127.0.0.1",
		Check: &api.AgentServiceCheck{
			CheckID:  "consul grpc test",
			TCP:      "127.0.0.1:1234",
			Timeout:  "5s",
			Interval: "5s",
		},
	}

	// 将服务注册到consul上
	if err := registry.Agent().ServiceRegister(&registerService); err != nil {
		log.Fatal(err)
	}

	// 设置监听
	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("listen failed, err:", err)
	}

	// 构造gRPC对象
	grpcServer := grpc.NewServer()
	// 注册服务
	pb.RegisterHelloServiceServer(grpcServer, &HelloService{})

	// 启动服务
	if err := grpcServer.Serve(listener); err != nil {
		log.Fatal(err)
	}
}
```

客户端`client.go`

```go
package main

import (
	"context"
	"fmt"
	"github.com/hashicorp/consul/api"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"grpc-consul/pb"
	"log"
	"strconv"
)

func main() {
	// 初始化consul配置，客户端服务端需要一致
	consulConfig := api.DefaultConfig()

	// 获取consul操作对象
	registerClient, err := api.NewClient(consulConfig)
	if err != nil {
		log.Fatal(err)
	}

	// 服务注销
	// if err := registerClient.Agent().ServiceDeregister("1"); err != nil {
	// 	log.Fatal(err)
	// }

	// 服务发现，从consul上获取健康的服务
	// params:
	// 	@service: 服务名。注册服务时指定该string
	// 	@tag: 外号/别名。如果有多个，任选一个
	// 	@passingOnly: 是否通过健康检查
	// 	@q: 查询参数，通常为nil
	// returns:
	// 	@ServiceEntry: 存储服务的切片
	//  @QueryMeta: 额外查询返回值，通常为nil
	//  @error: 错误信息
	serviceEntry, _, err := registerClient.Health().Service(
		"HelloService", "consul", false, &api.QueryOptions{})
	if err != nil {
		log.Fatal(err)
	}

	// 和grpc服务建立连接
	// conn, err := grpc.Dial("127.0.0.1:1234", grpc.WithTransportCredentials(insecure.NewCredentials()))
	conn, err := grpc.Dial(serviceEntry[0].Service.Address+":"+strconv.Itoa(serviceEntry[0].Service.Port),
		grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatal(err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {
			return
		}
	}(conn)

	// 初始化 gRPC 客户端
	client := pb.NewHelloServiceClient(conn)

	// 调用远程函数
	rsp, err := client.Hello(context.TODO(), &pb.Request{Name: "John"})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(rsp.Msg)
}
```

