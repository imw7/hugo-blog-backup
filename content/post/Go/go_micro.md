---
title: go-micro
tags: [go-micro, 常用组件和技巧]
categories: [Go]
date: 2021-08-11 00:00:00
toc: true
---

本文介绍 `go-micro` 框架，该框架是微服务的基础框架。<!--more-->

## go-micro 简介与设计理念

`Go Micro` 是一个基于 `Go` 语言编写的、用于构建微服务的基础框架，提供了分布式开发所需的核心组件，包括 `RPC` 和事件驱动通信等。

它的设计哲学是「可插拔」的插件化架构，其核心专注于提供底层的接口定义和基础工具，这些底层接口可以兼容各种实现。例如 `Go Micro` 默认通过 `consul` 进行服务发现（2019年源码修改了默认使用 `mdns`），通过 `HTTP` 协议进行通信，通过 `protobuf` 和 `json` 进行编解码，以便你可以基于这些开箱提供的组件快速启动，但是如果需要的话，你也可以通过符合底层接口定义的其他组件替换默认组件，比如通过 `etcd` 或 `zookeeper` 进行服务发现，这也是插件化架构的优势所在：不需要修改任何底层代码即可实现上层组件的替换。

## go-micro 基础架构介绍

`go-micro` 之所以可以高度订制和它的框架结构是分不开的，`go-micro` 由 8 个关键的 `interface` 组成，每一个 `interface` 都可以根据自己的需求重新实现，这 8 个主要的 `interface` 也构成了 `go-micro` 的框架结构：

![img](https://blog.imw7.com/images/Go/go_micro/framework.png)

- 最顶层的 **`Service`** 接口是构建服务的主要组件，它把底层的各个包需要实现的接口，做了一次封装，包含了一系列用于初始化 `Service` 和 `Client` 的方法，使我们可以很简单的创建一个 `RPC` 服务；
- **`Client`** 是请求服务的接口，从 `Registry` 中获取 `Server` 信息，然后封装了 `Transport` 和 `Codec` 进行 `RPC` 调用，也封装了 `Brocker` 进行消息发布，默认通过 `RPC` 协议进行通信，也可以基于 `HTTP` 或 `gRPC`；
- **`Server`** 是监听服务调用的接口，也将以接收 `Broker` 推送过来的消息，需要向 `Registry` 注册自己的存在与否，以便客户端发起请求，和 `Client` 一样，默认基于 `RPC` 协议通信，也可以替换为 `HTTP` 或 `gRPC`；
- **`Broker`** 是消息发布和订阅的接口，默认实现是基于 `HTTP`，在生产环境可以替换为 `Kafka`、`RabbitMQ` 等其他组件实现；
- **`Codec`** 用于解决传输过程中的编码和解码，默认实现是 `protobuf`，也可以替换成 `json`、`mercury` 等；
- **`Registry`** 用于实现服务的注册和发现，当有新的 `Service` 发布时，需要向 `Registry` 注册，然后 `Registry` 通知客户端进行更新，`Go Micro` 默认基于 `consul` 实现服务注册与发现，当然，也可以替换成 `etcd`、`zookeeper`、`kuberpackage mainnetes` 等；
- **`Selector`** 是客户端级别的负载均衡，当有客户端向服务端发送请求时，`Selector` 根据不同的算法从 `Registery` 的主机列表中得到可用的 `Service` 节点进行通信。目前的实现有循环算法和随机算法，默认使用随机算法，另外，`Selector` 还有缓存机制，默认是本地缓存，还支持 `label`、`blacklist` 等方式；
- **`Transport`** 是服务之间通信的接口，也就是服务发送和接收的最终实现方式，默认使用 `HTTP` 同步通信，也可以支持 `TCP`、`UDP`、`NATS`、`gRPC` 等其他方式。

`Go Micro` 官方创建了一个 [Plugins](https://github.com/go-micro/plugins) 仓库，用于维护 `Go Micro` 核心接口支持的可替换插件：

| 接口        | 支持组件                                       |
| :---------- | :--------------------------------------------- |
| `Broker`    | `NATS`、`NSQ`、`RabbitMQ`、`Kafka`、`Redis` 等 |
| `Client`    | `gRPC`、`HTTP`                                 |
| `Codec`     | `BSON`、`Mercury` 等                           |
| `Registry`  | `Etcd`、`NATS`、`Kubernetes`、`Eureka` 等      |
| `Selector`  | `Label`、`Blacklist`、`Static` 等              |
| `Transport` | `NATS`、`gPRC`、`RabbitMQ`、`TCP`、`UDP`       |
| `Wrapper`   | 中间件：熔断、限流、追踪、监控                 |

各个组件接口之间的关系可以通过下图串联：

![img](https://blog.imw7.com/images/Go/go_micro/interface.jpg)

## go-micro 主要功能

**服务发现**：自动服务注册和名称解析。服务发现是微服务开发的核心。当服务 A 需要与服务 B 通话时，它需要该服务的位置。默认发现机制是多播 `DNS`（`mdns`），一种零配置系统。您可以选择使用 `SWIM` 协议为 `p2p` 网络设置八卦，或者为弹性云原生设置 `consul`。

**负载均衡**：基于服务发现构建的客户端负载均衡。一旦我们获得了服务的任意数量实例的地址，我们现在需要一种方法来决定要路由到哪个节点。我们使用随机散列负载均衡来提供跨服务的均匀分布，并在出现问题时重试不同的节点。

**消息编码**：基于内容类型的动态消息编码。客户端和服务器将使用编码器和内容类型为您无缝编码和解码 Go 类型。可以编码任何种类的消息并从不同的客户端发送。客户端和服务器默认处理此问题。这包括默认的 `protobuf` 和 `json`。

**请求响应**：基于 `RPC` 的请求响应，支持双向流。我们提供了同步通信的抽象。对服务的请求将自动解决，负载平衡，拨号和流式传输。启用 `tls` 时，默认传输为 `http/1.1` 或 `http2`。

**`Async Messaging`**：`PubSub` 是异步通信和事件驱动架构的一流公民。事件通知是微服务开发的核心模式。启用 `tls` 时，默认消息传递是点对点 `http/1.1` 或 `http2`。

**可插拔接口**：`Go Micro` 为每个分布式系统抽象使用 `Go` 接口，因此，这些接口是可插拔的，并允许 `Go Micro` 与运行时无关，可以插入任何基础技术。

## 小结

通过上述介绍，可以看到，`Go Micro` 简单轻巧、易于上手、功能强大、扩展方便，是基于 `Go` 语言进行微服务架构时非常值得推荐的一个 `RPC` 框架，基于其核心功能及插件，我们可以轻松解决之前讨论的微服务架构引入的需要解决的问题：

- 服务接口定义：通过 `Transport`、`Codec` 定义通信协议及数据编码；
- 服务发布与调用：通过 `Registry` 实现服务注册与订阅，还可以基于 `Selector` 提高系统可用性；
- 服务监控、服务治理、故障定位：通过 `Plugins Wrapper` 中间件来实现。

接下来，我们将基于 `Go Micro` 微服务框架演示如何基于 `Go` 落地微服务架构。

## 编写 go-micro HTTP服务

### go-micro安装

#### 安装protobuf

可通过访问 `protobuf` 的`github`仓库查看[最新版本](https://github.com/protocolbuffers/protobuf/releases)。

```bash
$ wget https://github.com/protocolbuffers/protobuf/releases/download/v21.4/protobuf-all-21.4.tar.gz
$ tar -zxvf protobuf-all-21.4.tar.gz
$ cd protobuf-21.4
$ ./configure
$ make && make install
```

查看安装的 `protoc` 版本：

```bash
$ protoc --version
libprotoc 3.21.4
```

注意：如果这一步遇到类似 `protoc: error while loading shared libraries: libprotoc.so.8: cannot open shared object file: No such file or directory` 的错误，可以通过以下方法解决：

```bash
$ sudo ldconfig
# 或者
$ export LD_LIBRARY_PATH=/usr/local/lib
```

#### 安装go-micro工具集

```bash
$ go get go-micro.dev/v4@latest
$ go install github.com/go-micro/cli/cmd/go-micro@latest
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
$ go install github.com/asim/go-micro/cmd/protoc-gen-micro/v4@latest
```

### 开始编写go-micro HTTP服务

`greeter` 文件夹下创建 `pb` 文件夹，在 `pb` 文件夹中写入 `greeter.proto` 文件：

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

service Greeter {
  rpc Hello (Request) returns (Response) {}
}
```

进入 `pb` 目录输入命令：

```bash
$ protoc --micro_out=. --go_out=:. greeter.proto
```

会生成 `greeter.pb.go` 和 `greeter.pb.micro.go` 两个文件。

编写 `server` 端，创建服务并注册一下 `handler`，运行服务。

`server` 端 `main.go`：

```go
package main

import (
	"go-micro.dev/v4"
	"log"
	"micro-demo/handler"
	"micro-demo/pb"
)

func main() {
	// create service
	service := micro.NewService(
		micro.Name("greeter"),
		micro.Version("latest"),
	)

	// initialise flags
	service.Init()

	// register handler
	if err := pb.RegisterGreeterHandler(service.Server(), &handler.Greeter{}); err != nil {
		log.Fatal(err)
	}

	// run service
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}
```

`handler` 包下`greeter.go`文件：

```go
package handler

import (
	"context"
	"go-micro.dev/v4/util/log"
	"micro-demo/pb"
)

type Greeter struct{}

func (g *Greeter) Hello(ctx context.Context, req *pb.Request, rsp *pb.Response) error {
	log.Info("Received Greeter.Hello request")
	rsp.Msg = "Hello, " + req.Name + "!"
	return nil
}
```

调用 `server` 服务有两种方式，一种是通过终端命令调用，另一种是通过 `client` 端代码调用。

* 终端命令调用

```bash
$ go-micro call greeter Greeter.Hello '{"name": "John"}'
```

输出结果：

```bash
{"msg":"Hello, John!"}
```

* 编写 `client` 端调用 `service` 服务

```go
package main

import (
	"context"
	"fmt"
	"go-micro.dev/v4"
	"log"
	"micro-demo/pb"
)

func main() {
	// create a new service
	service := micro.NewService()

	// parse command line flags
	service.Init()

	// use the generated client stub
	client := pb.NewGreeterService("greeter", service.Client())

	// make request
	rsp, err := client.Hello(context.Background(), &pb.Request{Name: "John"})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(rsp.Msg)
}
```

运行`go.micro.srv.HelloWorld`服务，启动客户端，运行结果如下：

```bash
Hello, John!
```

## 启动HTTP服务

使用 `go-micro` 的 `http` 创建一个可以调用接口的微服务 `HTTP`

### httpServer

这里使用 `gin` 框架结合 `go-micro` 来进行编写

首先，创建一个 `http` 目录，并在该目录下创建 `main.go`，写入代码。

```go
package main

import (
	"github.com/gin-gonic/gin"
	"go-micro.dev/v4/registry"
	"go-micro.dev/v4/web"
	"hello/http/handler"
	"log"
)

func main() {

	gin.SetMode(gin.ReleaseMode)
	router := gin.New()
	router.Use(gin.Recovery())

	// register router
	demo := handler.NewDemo()
	demo.InitRouter(router)

	// Create service
	service := web.NewService(
		web.Name("hello"),
		web.Registry(registry.NewRegistry()),
		web.Address(":8080"),
		web.Metadata(map[string]string{"protocol": "http"}),
		web.Handler(router),
	)
	
    _ = service.Init()

	// run service
	if err := service.Run(); err != nil {
		log.Println(err)
        return
	}
}
```

### 使用gin进行初始化路由

在 `http` 目录创建 `handler/handler.go`

```go
package handler

import (
	"context"
	"github.com/gin-gonic/gin"
	"go-micro-demo/hello/pb"
	"go-micro.dev/v4"
	"net/http"
)

type demo struct{}

func NewDemo() *demo {
	return &demo{}
}

func (d *demo) InitRouter(router *gin.Engine) {
	router.POST("/demo", d.demo)
}

func (d *demo) demo(c *gin.Context) {
	// create a service
	service := micro.NewService()
	service.Init()

	client := pb.NewHelloService("hello", service.Client())

	rsp, err := client.Call(context.Background(), &pb.Request{Name: "John"})
	if err != nil {
		c.JSON(http.StatusOK, gin.H{
			"code": http.StatusInternalServerError,
			"msg":  err.Error(),
		})
		return
	}

	c.JSON(http.StatusOK, gin.H{
		"code": http.StatusOK,
		"msg":  rsp.Msg,
	})
}
```

### Postman测试

在启动两个微服务之后，使用 `postman` 进行测试，调用成功并返回结果：

![Postman测试](https://blog.imw7.com/images/Go/go_micro/postman_test.png)

## 事件驱动

事件驱动架构理解起来比较简单，普通认为好的软件架构都是解耦的，微服务之间不应该相互耦合或依赖。举个例子，我们在代码中调用微服务的函数，会先通过服务发现找到微服务的地址再调用，我们的代码与该微服务有了直接性的调用交互，并不算是完全的解耦。

### 发布与订阅模式

为了理解时间驱动架构为何能使代码完全解耦，先了解事件的发布、订阅流程。微服务X完成任务x后通知消息系统说“x已完成”，它并不关心有哪些微服务正在监听这个事件、事件发生后会产生哪些影响。如果系统发生为了某个事件，随之其他微服务都要做出动作是很容易的。

举个例子，`user-service`创建了一个新用户，`email-service`要给该用户发一封注册成功的邮件，`message-service`要给网站管理员发一条用户注册的通知短信。

#### 一般实现

在`user-service`的代码中实例化另两个微服务客户端后，调用函数发邮件和短信，代码耦合度很高。如下图：

![一般实现](https://blog.imw7.com/images/Go/go_micro/normal.png)

#### 事件驱动

在事件驱动的架构下，`user-service`只需向消息系统发布一条 `topic` 为 `"user.created"` 的消息，其他两个订阅了此 `topic` 的 `service` 能知道有用户注册了，拿到用户信息后他们自行发邮件、发短信。如下图：

![发布与订阅模式](https://blog.imw7.com/images/Go/go_micro/pubsub.png)

### 代码实现

首先创建 `pubsub` 文件夹，`pb` 文件夹下`pubsub.proto`文件：

```protobuf
syntax = "proto3";

package pubsub;
option go_package = "../pb";

message Event {
  // unique id
  string id = 1;
  // unix timestamp
  int64 timestamp = 2;
  // message
  string msg = 3;
}
```

#### Publish 事件发布

`main.go`文件：

```go
package main

import (
	"context"
	"go-micro-examples/pubsub/pb"
	"go-micro.dev/v4"
	"go-micro.dev/v4/metadata"
	"go-micro.dev/v4/server"
	"go-micro.dev/v4/util/log"
)

// Sub All methods of Sub will be executed when a message is received
type Sub struct{}

// Process Method can be of any name
func (s *Sub) Process(ctx context.Context, event *pb.Event) error {
	md, _ := metadata.FromContext(ctx)
	log.Logf("[pubsub.1] received event %+v when metadata %+v\n", event, md)
	// do something with event
	return nil
}

// Alternatively a function can be used
func subEv(ctx context.Context, event *pb.Event) error {
	md, _ := metadata.FromContext(ctx)
	log.Logf("[pubsub.2] received event %+v when metadata %+v\n", event, md)
	// do something with event
	return nil
}

func main() {
	// create a service
	service := micro.NewService(
		micro.Name("go.micro.srv.pubsub"),
		micro.Version("latest"),
	)
	// initialise flags
	service.Init()

	// register subscriber
	if err := micro.RegisterSubscriber("example.topic.pubsub.1", service.Server(), &Sub{}); err != nil {
		log.Fatal(err)
	}

	// register subscriber with queue, each msg is delivered to a unique subscriber
	if err := micro.RegisterSubscriber("example.topic.pubsub.2", service.Server(), subEv, server.SubscriberQueue("queue.pubsub")); err != nil {
		log.Fatal(err)
	}

	// run service
	if err := service.Run(); err != nil {
		log.Fatal(err)
	}
}
```

运行结果如下：

```bash
$ ./pubsub 
2021-12-12 00:36:04  file=v4@v4.4.0/service.go:206 level=info Starting [service] go.micro.srv.pubsub
2021-12-12 00:36:04  file=server/rpc_server.go:820 level=info Transport [http] Listening on [::]:39189
2021-12-12 00:36:04  file=server/rpc_server.go:840 level=info Broker [http] Connected to 127.0.0.1:43507
2021-12-12 00:36:04  file=server/rpc_server.go:654 level=info Registry [mdns] Registering node: go.micro.srv.pubsub-b046fb63-518c-40e8-8d36-4a8a086b242d
2021-12-12 00:36:04  file=server/rpc_server.go:706 level=info Subscribing to topic: example.topic.pubsub.1
2021-12-12 00:36:04  file=server/rpc_server.go:706 level=info Subscribing to topic: example.topic.pubsub.2
```

可以看到，直接使用 `go-micro` 的 `NewEvent` 发布即可，成功发布 `example.topic.pubsub.1` 和 `example.topic.pubsub.2`。

#### Subscribe 事件订阅

事件既然有发布，那么当然会有订阅。`client` 文件夹下 `main.go` 文件：

```go
package main

import (
	"context"
	"fmt"
	"github.com/pborman/uuid"
	"go-micro-examples/pubsub/pb"
	"go-micro.dev/v4"
	"go-micro.dev/v4/util/log"
	"time"
)

// send events using the publisher
func sendEv(topic string, p micro.Publisher) {
	t := time.NewTimer(time.Second)

	for range t.C {
		// create new event
		ev := &pb.Event{
			Id:        uuid.NewUUID().String(),
			Timestamp: time.Now().Unix(),
			Msg:       fmt.Sprintf("Messaging you all day on %s", topic),
		}

		log.Logf("publishing %+v\n", ev)

		// publish an event
		if err := p.Publish(context.Background(), ev); err != nil {
			log.Logf("error publishing %v", err)
		}
	}
}

func main() {
	// create a service
	service := micro.NewService(
		micro.Name("go.micro.cli.pubsub"),
		micro.Version("latest"),
	)

	// initialise flags
	service.Init()

	// create publisher
	pub1 := micro.NewEvent("example.topic.pubsub.1", service.Client())
	pub2 := micro.NewEvent("example.topic.pubsub.2", service.Client())

	// pub to topic 1
	go sendEv("example.topic.pubsub.1", pub1)
	// pub to topic 2
	go sendEv("example.topic.pubsub.2", pub2)

	// block forever
	select {}
}
```

运行结果如下：

```bash
$ ./client 
2021-12-12 00:37:17  file=client/main.go:25 level=info publishing [id:"9a76de8c-5aa0-11ec-a19b-e454e8084c9f" timestamp:1639240637 msg:"Messaging you all day on example.topic.pubsub.1"]

2021-12-12 00:37:17  file=client/main.go:25 level=info publishing [id:"9a76e8e1-5aa0-11ec-a19b-e454e8084c9f" timestamp:1639240637 msg:"Messaging you all day on example.topic.pubsub.2"]
```

结果显示，成功订阅事件并调用成功。

## 注册和配置中心

下面介绍 `go-micro v4` 是如何进行配置[consul](https://www.consul.io)注册中心和操作配置中心的。

`go-micro` 框架为服务注册发现提供了标准的接口 `Registry`。只要实现这个接口就可以定制自己的服务注册和发现。不过官方已经为主流注册中心提供了官方的接口实现，大多数时候我们不需要从头写起。

### Consul安装

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

