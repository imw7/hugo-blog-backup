---
title: gRPC
tags: [grpc, protobuf, 常用组件与技巧]
categories: [Go]
date: 2021-07-21 00:00:00
toc: true
---

gRPC是Google公司基于Protobuf开发的跨语言的开源RPC框架。gRPC基于HTTP/2协议设计，可以基于一个HTTP/2链接提供多个服务，对于移动设备更加友好。本文将介绍gRPC的一些用法。<!--more-->



# gRPC入门

本节将讲述gRPC的简单用法。

## gRPC技术栈

Go语言的gRPC技术栈如图1所示：

![img](https://imw7.github.io/images/Go/grpc/grpc-go-stack.png)

*图1 gRPC技术栈*

最底层为TCP或Unix Socket协议，在此之上是HTTP/2协议的实现，然后在HTTP/2协议之上又构建了针对Go语言的gRPC核心库。应用程序通过gRPC插件生产的Stub代码和gRPC核心库通信，也可以直接和gRPC核心库通信。

## gRPC入门

如果从Protobuf的角度看，gRPC只不过是一个针对service接口生成代码的生成器。

创建hello.proto文件，定义HelloService接口：

```protobuf
syntax = "proto3"; // 使用proto3的语法

package pb; // package指令指明当前是pb包
option go_package = "../pb"; // 指明生成的go文件所放置的路径

// message 关键字定义一个新的String类型，
// 在最终生成的Go语言代码中对应一个String结构体。
message String {
	// String类型中只有一个字符串类型的value成员，
	// 该成员编码时用1编号代替名字
	string value = 1;
}

service HelloService {
	rpc Hello (String) returns (String);
}
```

使用protoc-gen-go内置的gRPC插件生成gRPC代码：

```bash
$ protoc --go_out=plugins=grpc:. hello.proto
```

gRPC插件会为服务端和客户端生成不同的接口：

```go
type HelloServiceServer interface {
    Hello(context.Context, *pb.String) (*pb.String, error)
}

type HelloServiceClient interface {
    Hello(context.Context, *pb.String, ...grpc.CallOption) (*pb.String, error)
}
```

gRPC通过context.Context参数，为每个调用方法提供了上下文支持。客户端在调用方法的时候，可以通过可选的grpc.CallOption类型的参数提供额外的上下文信息。

基于服务端的HelloServiceServer接口可以重新实现HelloService服务：

```go
type HelloServiceImpl struct{}

func (h *HelloServiceImpl) Hello(ctx context.Context, args *pb.String) (*pb.String, error) {
    reply := &pb.String{Value: "Hello, " + args.GetValue() + "!"}
    return reply, nil
}
```

gRPC服务的启动流程和标准库的RPC服务启动流程类似：

```go
func main() {
    // 构造一个gRPC服务对象
	grpcServer := grpc.NewServer()
	// 通过gRPC插件生成的RegisterHelloServiceServer函数注册HelloServiceImpl服务
	pb.RegisterHelloServiceServer(grpcServer, new(HelloServiceImpl))

	listener, err := net.Listen("tcp", ":1234")
	if err != nil {
		log.Fatal("ListenTCP error:", err)
	}
	// 在一个监听端口提供gRPC服务
	if err = grpcServer.Serve(listener); err != nil {
		log.Fatal(err)
	}
}
```

首先是通过`grpc.NewServer()`构造一个gRPC服务对象，然后通过gRPC插件生成的RegisterHelloServiceServer函数注册我们实现的HelloServiceImpl服务。然后通过`grpcServer.Serve(lis)`在一个监听端口上提供gRPC服务。

然后就可以通过客户端链接gRPC服务了：

```go
func main() {
    // 和gRPC服务建立链接
	conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {
			return
		}
	}(conn)

	// 基于已经建立的链接构造HelloServiceClient对象
	client := pb.NewHelloServiceClient(conn)
	// 返回的client其实是一个HelloServiceClient接口对象
	// 通过接口定义的方法就可以调用服务端对应的gRPC服务提供的方法
	reply, err := client.Hello(context.Background(), &pb.String{Value: "Eric"})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(reply.GetValue())
}
```

其中grpc.Dial负责和gRPC服务建立链接，然后NewHelloServiceClient函数基于已经建立的链接构造HelloServiceClient对象。返回的client其实是一个HelloServiceClient借口对象，通过接口定义的方法就可以调用服务端对应的gRPC服务提供的方法。

gRPC和标准库的RPC框架有一个区别，gRPC生成的接口并不支持异步调用。不过我们可以在多个Goroutine之间安全地共享gRPC底层的HTTP/2链接，因此可以通过在另一个Goroutine阻塞调用的方式模拟异步调用。

## gRPC流

RPC是远程函数调用，因此每次调用的函数参数和返回值不能太大，否则将严重影响每次调用的响应时间。因此传统的RPC方法调用对于上传和下载较大数据量场景并不适合。同时传统RPC模式也不适用于对时间不确定的订阅和发布模式。为此，gRPC框架针对服务器端和客户端分别提供了流特性。

服务端或客户端的单向流是双向流的特例，我们在HelloService增加一个支持双向流的Channel方法：

```protobuf
service HelloService {
	rpc Hello (String) returns (String);
	
	// 增加一个支持双向流的Channel方法
  	// 关键字stream指定启用流特性，参数部分是接收客户端参数的流，返回值是返回给客户端的流。
	rpc Channel (stream String) returns (stream String);
}
```

关键字stream指定启用流特性，参数部分是接收客户端参数的流，返回值是返回给客户端的流。

重新生成代码可以看到接口中新增加的Channel方法的定义：

```go
type HelloServiceServer interface {
	Hello(context.Context, *String) (*String, error)
	Channel(HelloService_ChannelServer) error
}
type HelloServiceClient interface {
	Hello(ctx context.Context, in *String, opts ...grpc.CallOption) (*String, error)
	Channel(ctx context.Context, opts ...grpc.CallOption) (HelloService_ChannelClient, error)
}
```

在服务端的Channel方法参数是一个新的HelloService_ChannelServer类型的参数，可以用于和客户端双向通信。客户端的Channel方法返回一个HelloService_ChannelClient类型的返回值，可以用于和服务端进行双向通信。

HelloService_ChannelServer和HelloService_ChannelClient均为接口类型：

```go
type HelloService_ChannelServer interface {
    Send(*String) error
    Recv() (*String, error)
    grpc.ServerStream
}

type HelloService_ChannelClient interface {
	Send(*String) error
	Recv() (*String, error)
	grpc.ClientStream
}
```

可以发现服务端和客户端的流辅助接口均定义了Send和Recv方法用于流数据的双向通信。

现在我们可以实现流服务了：

```go
// Channel 可以用于和客户端双向通信
// Send和Recv方法用于流数据的双向通信
func (p *HelloServiceImpl) Channel(stream pb.HelloService_ChannelServer) error {
	for { // 在循环中接收客户端发来的数据
		args, err := stream.Recv()
		if err != nil {
			if err == io.EOF { // 表示客户端流被关闭
				return nil
			}
			return nil
		}
		reply := &pb.String{Value: "hello:" + args.GetValue()}

		if err = stream.Send(reply); err != nil {
			return err
		}
	}
}
```

服务端在循环中接收客户端发来的数据，如果遇到io.EOF表示客户端流被关闭，如果函数退出表示服务端流关闭。生成返回的数据通过流发送给客户端，双向流数据的发送和接收都是完全独立的行为。需要注意的是，发送和接收的操作并不需要一一对应，用户可以根据真实场景进行组织代码。

客户端需要先调用Channel方法获取返回的流对象：

```go
stream, err := client.Channel(context.Background())
if err != nil {
    log.Fatal(err)
}
```

在客户端我们将发送和接收操作放到两个独立的Goroutine。首先是向服务端发送数据：

```go
go func() {
   for { // 向服务端发送数据
      if err = stream.Send(&pb.String{Value: "hi"}); err != nil {
         log.Fatal(err)
      }
      time.Sleep(time.Second)
   }
}()
```

然后在循环中接收服务端返回的数据：

```go
for {
   reply, err := stream.Recv() // 接收服务端返回的数据
   if err != nil {
      if err == io.EOF {
         break
      }
      log.Fatal(err)
   }
   fmt.Println(reply.GetValue())
}
```

这样就完成了完整的流接收和发送支持。

## 发布和订阅模式

在发布和订阅模式中，由调用者主动发起的发布行为类似一个普通函数调用，而被动的订阅者则类似gRPC客户端单向流中的接收者。

发布订阅是一个常见的设计模式，开源社区中已经存在很多该模式的实现。其中docker项目中提供了一个pubsub的极简实现，下面是基于pubsub包实现的本地发布订阅代码：

```go
import "github.com/moby/moby/pkg/pubsub"

func main() {
    p := pubsub.NewPublisher(100*time.Millisecond, 10)

    golang := p.SubscribeTopic(func(v interface{}) bool {
        if key, ok := v.(string); ok {
            if strings.HasPrefix(key, "golang:") {
                return true
            }
        }
        return false
    })
    docker := p.SubscribeTopic(func(v interface{}) bool {
        if key, ok := v.(string); ok {
            if strings.HasPrefix(key, "docker:") {
                return true
            }
        }
        return false
    })

    go p.Publish("hi")
    go p.Publish("golang: https://golang.org")
    go p.Publish("docker: https://www.docker.com/")
    time.Sleep(1)

    go func() {
        fmt.Println("golang topic:", <-golang)
    }()
    go func() {
        fmt.Println("docker topic:", <-docker)
    }()

    <-make(chan bool)
}
```

其中`pubsub.NewPublisher`构造一个发布对象，`p.SubscribeTopic()`可以通过函数筛选感兴趣的主题进行订阅。

现在尝试基于gRPC和pubsub包，提供一个跨网络的发布和订阅系统。首先通过Protobuf定义一个发布订阅服务接口：

```protobuf
// 发布订阅服务接口
service PubsubService {
  rpc Publish (String) returns (String); // 普通的RPC方法
  rpc Subscribe (String) returns (stream String); // 单向的流服务
}
```

其中Publish是普通的RPC方法，Subscribe则是一个单向的流服务。然后gRPC插件会为服务端和客户端生成对应的接口：

```go
type PubsubServiceServer interface {
	Publish(context.Context, *String) (*String, error)
	Subscribe(*String, PubsubService_SubscribeServer) error
}
type PubsubServiceClient interface {
	Publish(ctx context.Context, in *String, opts ...grpc.CallOption) (*String, error)
	Subscribe(ctx context.Context, in *String, opts ...grpc.CallOption) (PubsubService_SubscribeClient, error)
}

type PubsubService_SubscribeServer interface {
    Send(*String) error
    grpc.ServerStream
}
```

因为Subscribe是服务端的单向流，因此生成的HelloService_SubscribeServer接口中只有Send方法。

然后就可以实现发布和订阅服务了：

```go
type PubsubService struct {
	pub *pubsub.Publisher
}

func NewPubsubService() *PubsubService {
	return &PubsubService{
		pub: pubsub.NewPublisher(100*time.Millisecond, 10),
	}
}
```

然后是实现发布方法和订阅方法：

```go
// Publish 发布方法
func (p *PubsubService) Publish(ctx context.Context, arg *pb.String) (*pb.String, error) {
	p.pub.Publish(arg.GetValue())
	return &pb.String{}, nil
}

// Subscribe 订阅方法
func (p *PubsubService) Subscribe(arg *pb.String, stream pb.PubsubService_SubscribeServer) error {
	ch := p.pub.SubscribeTopic(func(v interface{}) bool {
		if key, ok := v.(string); ok {
			if strings.HasPrefix(key, arg.GetValue()) {
				return true
			}
		}
		return false
	})

	for v := range ch {
		if err := stream.Send(&pb.String{Value: v.(string)}); err != nil {
			return err
		}
	}

	return nil
}
```

这样就可以从客户端向服务端发布信息了：

```go
// publish_client.go 客户端: 向服务端发布信息

func main() {
	conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {
			return
		}
	}(conn)

	client := pb.NewPubsubServiceClient(conn)

	_, err = client.Publish(context.Background(), &pb.String{Value: "golang: hello Go"})
	if err != nil {
		log.Fatal(err)
	}
	_, err = client.Publish(context.Background(), &pb.String{Value: "docker: hello Docker"})
	if err != nil {
		log.Fatal(err)
	}
}
```

然后就可以在另一个服务端进行订阅信息了：

```go
// subscribe_client.go 客户端: 向另一个客户端进行订阅信息

func main() {
	conn, err := grpc.Dial("localhost:1234", grpc.WithInsecure())
	if err != nil {
		log.Fatal(err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {
			return
		}
	}(conn)

	client := pb.NewPubsubServiceClient(conn)

	var stream pb.PubsubService_SubscribeClient
	var streams []pb.PubsubService_SubscribeClient

	stream, err = client.Subscribe(context.Background(), &pb.String{Value: "golang:"})
	if err != nil {
		log.Fatal(err)
	}
	streams = append(streams, stream)

	// stream, err = client.Subscribe(context.Background(), &pb.String{Value: "docker:"})
	// if err != nil {
	// 	log.Fatal(err)
	// }
	// streams = append(streams, stream)

	for {
		for _, stream = range streams {
			reply, err := stream.Recv()
			if err != nil {
				if err == io.EOF {
					break
				}
				log.Fatal(err)
			}

			fmt.Println(reply.GetValue())
		}
	}
}
```

到此我们就基于gRPC简单实现了一个跨网络的发布和订阅服务。



# gRPC进阶

作为一个基础的RPC框架，安全和扩展是经常遇到的问题。本节将简单介绍如果对gRPC进行安全认证。然后介绍通过gRPC的截取器特性，以及如何通过截取器优雅地实现Token认证、调用跟踪以及Panic捕获等特性。最后介绍服务如何和其他Web服务共存。

## 证书认证

gRPC建立在HTTP/2协议之上，对TLS提供了很好的支持。我们前面章节中gRPC的服务都没有提供证书支持，因此客户端在链接服务器中通过`grpc.WithInsecure()`选项跳过了对服务器证书的验证。没有启用证书的gRPC服务在和客户端进行的是明文通讯，信息面临被任何第三方监听的风险。为了保障gRPC通信不被第三方监听篡改或伪造，我们可以对服务器启动TLS加密特性。

可以用以下命令为服务器和客户端分别生成私钥和证书：

```bash
$ openssl genrsa -out server.key 2048
$ openssl req -new -x509 -days 3650 \
    -subj "/C=GB/L=China/O=grpc-server/CN=server.grpc.io" \
    -key server.key -out server.crt

$ openssl genrsa -out client.key 2048
$ openssl req -new -x509 -days 3650 \
    -subj "/C=GB/L=China/O=grpc-client/CN=client.grpc.io" \
    -key client.key -out client.crt
```

以上命令将生成server.key、server.crt、client.key和client.crt四个文件。其中以.key为后缀名的是私钥文件，需要妥善保管。以.crt为后缀名是证书文件，也可以简单理解为公钥文件，并不需要秘密保存。在subj参数中的`/CN=server.grpc.io`表示服务器的名字为`server.grpc.io`，在验证服务器的证书时需要用到该信息。

有了证书之后，我们就可以在启动gRPC服务时传入证书选项参数：

```go
func main() {
    creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
    if err != nil {
        log.Fatal(err)
    }

    server := grpc.NewServer(grpc.Creds(creds))

    ...
}
```

其中credentials.NewServerTLSFromFile函数是从文件为服务器构造证书对象，然后通过grpc.Creds(creds)函数将证书包装为选项后作为参数传入grpc.NewServer函数。

在客户端基于服务器的证书和服务器名字就可以对服务器进行验证：

```go
func main() {
    creds, err := credentials.NewClientTLSFromFile("server.crt", "server.grpc.io")
    if err != nil {
        log.Fatal(err)
    }

    conn, err := grpc.Dial("localhost:5000", grpc.WithTransportCredentials(creds))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ...
}
```

其中redentials.NewClientTLSFromFile是构造客户端用的证书对象，第一个参数是服务器的证书文件，第二个参数是签发证书的服务器的名字。然后通过grpc.WithTransportCredentials(creds)将证书对象转为参数选项传人grpc.Dial函数。

以上这种方式，需要提前将服务器的证书告知客户端，这样客户端在链接服务器时才能进行对服务器证书认证。在复杂的网络环境中，服务器证书的传输本身也是一个非常危险的问题。如果在中间某个环节，服务器证书被监听或替换那么对服务器的认证也将不再可靠。

为了避免证书的传递过程中被篡改，可以通过一个安全可靠的**根证书**分别对服务器和客户端的证书进行签名。这样客户端或服务器在收到对方的证书后可以通过根证书进行验证证书的有效性。

### 根证书

根证书（root certificate）是属于根证书颁发机构（CA）的公钥证书。我们可以通过验证 CA 的签名从而信任 CA ，任何人都可以得到 CA 的证书（含公钥），用以验证它所签发的证书（客户端、服务端）。它包含公钥和密钥。

#### 生成 key

```bash
$ openssl genrsa -out ca.key 2048
```

#### 生成密钥

```bash
$ openssl req -new -x509 -days 3650 -key ca.key -out ca.pem
```

##### 填写信息

```bash
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:         
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:
```

### Server

#### 生成Key

```bash
$ openssl ecparam -genkey -name secp384r1 -out server.key
```

#### 生成CSR

```bash
$ openssl req -new -key server.key -out server.csr
```

##### 填写信息

```bash
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```

CSR 是 Cerificate Signing Request 的英文缩写，为证书请求文件。主要作用是 CA 会利用 CSR 文件进行签名使得攻击者无法伪装或篡改原有证书。

#### 基于 CA 签发

```bash
openssl x509 -req -sha256 -CA ca.pem -CAkey ca.key -CAcreateserial -days 3650 -in server.csr -out server.pem
```

签名的过程中引入了一个新的以.csr为后缀名的文件，它表示证书签名请求文件。在证书签名完成之后可以删除.csr文件。

### Client

#### 生成Key

```bash
openssl ecparam -genkey -name secp384r1 -out client.key
```

#### 生成CSR

```bash
openssl req -new -key client.key -out client.csr
```

#### 基于CA签发

```bash
openssl x509 -req -sha256 -CA ca.pem -CAkey ca.key -CAcreateserial -days 3650 -in client.csr -out client.pem
```

**注意**：至此我们生成了一堆文件，为了方便按住以下目录结构存放。

```bash
$ tree conf
conf
├── ca.key
├── ca.pem
├── ca.srl
├── client
│   ├── client.csr
│   ├── client.key
│   └── client.pem
└── server
    ├── server.csr
    ├── server.key
    └── server.pem
```

然后在客户端就可以基于CA证书对服务器进行证书验证：

```go
func main() {
    certificate, err := tls.LoadX509KeyPair("../conf/client/client.pem", "../conf/client/client.key")
	if err != nil {
		log.Fatal(err)
	}

	certPool := x509.NewCertPool()
	ca, err := ioutil.ReadFile("../conf/ca.pem")
	if err != nil {
		log.Fatal(err)
	}
	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		log.Fatal("failed to append ca certs")
	}

	creds := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{certificate},
		ServerName:   "localhost", // NOTE: this is required! 和 填写信息时候的`Common Name`相同
		RootCAs:      certPool,
	})

	conn, err := grpc.Dial(":8972", grpc.WithTransportCredentials(creds))
	if err != nil {
		log.Fatal("dialing:", err)
	}
	defer func(conn *grpc.ClientConn) {
		err := conn.Close()
		if err != nil {
			return
		}
	}(conn)

    ...
}
```

在新的客户端代码中，我们不再直接依赖服务器端证书文件。在credentials.NewTLS函数调用中，客户端通过引入一个CA根证书和服务器的名字来实现对服务器进行验证。客户端在链接服务器时会首先请求服务器的证书，然后使用CA根证书对收到的服务器端证书进行验证。

因为引入了CA根证书签名，在启动服务器时同样要配置根证书：

```go
func main() {
    certificate, err := tls.LoadX509KeyPair("../conf/server/server.pem", "../conf/server/server.key")
	if err != nil {
		log.Fatal(err)
	}

	certPool := x509.NewCertPool()
	ca, err := ioutil.ReadFile("../conf/ca.pem")
	if err != nil {
		log.Fatal(err)
	}
	if ok := certPool.AppendCertsFromPEM(ca); !ok {
		log.Fatal("failed to append certs")
	}

	creds := credentials.NewTLS(&tls.Config{
		Certificates: []tls.Certificate{certificate},
		ClientAuth:   tls.RequireAndVerifyClientCert, // NOTE: this is optional!
		ClientCAs:    certPool,
	})

	grpcServer := grpc.NewServer(grpc.Creds(creds))
    ...
}
```

服务器端同样改用credentials.NewTLS函数生成证书，通过ClientCAs选择CA根证书，并通过ClientAuth选项启用对客户端进行验证。

到此我们就实现了一个服务器和客户端进行双向证书验证的通信可靠的gRPC系统。

## Token认证

前面讲述的基于证书的认证是针对每个gRPC链接的认证。gRPC还为每个gRPC方法调用提供了认证支持，这样就基于用户Token对不同的方法访问进行权限管理。要实现对每个gRPC方法进行认证，需要实现grpc.PerRPCCredentials接口：

```go
type PerRPCCredentials interface {
    // GetRequestMetadata gets the current request metadata, refreshing
    // tokens if required. This should be called by the transport layer on
    // each request, and the data should be populated in headers or other
    // context. If a status code is returned, it will be used as the status
    // for the RPC. uri is the URI of the entry point for the request.
    // When supported by the underlying implementation, ctx can be used for
    // timeout and cancellation.
    // TODO(zhaoq): Define the set of the qualified keys instead of leaving
    // it as an arbitrary string.
    GetRequestMetadata(ctx context.Context, uri ...string) (
        map[string]string,    error,
    )
    // RequireTransportSecurity indicates whether the credentials requires
    // transport security.
    RequireTransportSecurity() bool
}
```

在GetRequestMetadata方法中返回认证需要的必要信息。RequireTransportSecurity方法表示是否要求底层使用安全链接。在真实的环境中建议必须要求底层启用安全的链接，否则认证信息有泄露和被篡改的风险。

我们可以创建一个Authentication类型，用于实现用户名和密码的认证：

```go
type Authentication struct {
    User     string
    Password string
}

// GetRequestMetadata 返回的认证信息包装user和password两个信息
func (a *Authentication) GetRequestMetadata(context.Context, ...string) (map[string]string, error) {
	return map[string]string{"user": a.User, "password": a.Password}, nil
}

// RequireTransportSecurity 方法return false表示不要求底层使用安全链接
func (a *Authentication) RequireTransportSecurity() bool {
	return false
}
```

在GetRequestMetadata方法中，我们返回地认证信息包装login和password两个信息。为了演示代码简单，RequireTransportSecurity方法表示不要求底层使用安全链接。

然后在每次请求gRPC服务时就可以将Token信息作为参数选项传人：

```go
func main() {
    auth := Authentication{
        User:    "gopher",
        Password: "password",
    }

    conn, err := grpc.Dial("localhost"+port, grpc.WithInsecure(), grpc.WithPerRPCCredentials(&auth))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    ...
}
```

通过grpc.WithPerRPCCredentials函数将Authentication对象转为grpc.Dial参数。因为这里没有启用安全链接，需要传人grpc.WithInsecure()表示忽略证书认证。

然后在gRPC服务端的每个方法中通过Authentication类型的Auth方法进行身份认证：

```go
type grpcServer struct { auth *Authentication }

func (g *grpcServer) Hello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	// 进行Auth方法认证
	if err := g.auth.Auth(ctx); err != nil {
		return nil, err
	}
	reply := &pb.HelloReply{Message: "Hello, " + in.GetName() + "!"}
	return reply, nil
}

func (a *Authentication) Auth(ctx context.Context) error {
	md, ok := metadata.FromIncomingContext(ctx) // 从ctx上下文中获取元信息
	if !ok {
		return fmt.Errorf("missing credentials")
	}
	// 取出相应的的认证信息进行认证
	var appID string
	var appKey string

	if val, ok := md["user"]; ok {
		appID = val[0]
	}
	if val, ok := md["password"]; ok {
		appKey = val[0]
	}
	// 认证失败，返回一个odes.Unauthenticated类型的错误
	if appID != "gopher" || appKey != "password" {
		return status.Errorf(codes.Unauthenticated, "invalid token")
	}

	return nil
}
```

详细地认证工作主要在Authentication.Auth方法中完成。首先通过metadata.FromIncomingContext从ctx上下文中获取元信息，然后取出相应的认证信息进行认证。如果认证失败，则返回一个codes.Unauthenticated类型地错误。

## 截取器

gRPC中的grpc.UnaryInterceptor和grpc.StreamInterceptor分别对普通方法和流方法提供了截取器的支持。我们这里简单介绍普通方法的截取器用法。

要实现普通方法的截取器，需要为grpc.UnaryInterceptor的参数实现一个函数：

```go
func filter(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
    log.Println("fileter:", info)
    return handler(ctx, req)
}
```

函数的ctx和req参数就是每个普通的RPC方法的前两个参数。第三个info参数表示当前是对应的那个gRPC方法，第四个handler参数对应当前的gRPC方法函数。上面的函数中首先是日志输出info参数，然后调用handler对应的gRPC方法函数。

要使用filter截取器函数，只需要在启动gRPC服务时作为参数输入即可：

```go
server := grpc.NewServer(grpc.UnaryInterceptor(filter))
```

然后服务器在收到每个gRPC方法调用之前，会首先输出一行日志，然后再调用对方的方法。

如果截取器函数返回了错误，那么该次gRPC方法调用将被视作失败处理。因此，我们可以在截取器中对输入的参数做一些简单的验证工作。同样，也可以对handler返回的结果做一些验证工作。截取器也非常适合前面对Token认证工作。

下面是截取器增加了对gRPC方法异常的捕获：

```go
// filter 要实现普通方法的截取器，需要实现的函数
// 函数的ctx和req参数就是每个普通的RPC方法的前两个参数
// 第三个info参数表示当前是对应的那个gRPC方法
// 第四个handler参数对应当前的gRPC方法函数
func filter(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler) (resp interface{}, err error) {
	log.Println("filter:", info)
	defer func() { // 添加对gRPC方法异常的捕获
		if r := recover(); r != nil {
			err = fmt.Errorf("panic: %v", r)
		}
	}()

	return handler(ctx, req)
}
```

不过gRPC框架中只能为每个服务设置一个截取器，因此所有的截取工作只能在一个函数中完成。开源的grpc-ecosystem项目中的go-grpc-middleware包已经基于gRPC对截取器实现了链式截取器的支持。

以下是go-grpc-middleware包中链式截取器的简单用法

```go
import "github.com/grpc-ecosystem/go-grpc-middleware"

myServer := grpc.NewServer(
    grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
        filter1, filter2, ...
    )),
    grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
        filter1, filter2, ...
    )),
)
```

## 和Web服务共存

gRPC构建在HTTP/2协议之上，因此我们可以将gRPC服务和普通的Web服务架设在同一个端口之上。

对于没有启动TLS协议的服务则需要对HTTP2/2特性做适当的调整：

```go
func main() {
    mux := http.NewServeMux()
	
	h2Handler := h2c.NewHandler(mux, &http2.Server{})
	server := &http.Server{Addr: ":8972", Handler: h2Handler}
	_ = server.ListenAndServe()
}
```

启用普通的https服务器则非常简单：

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        _, _ = fmt.Fprintln(w, "hello")
    })
    
    http.ListenAndServeTLS(":8972", "../conf/server/server.pem", "../conf/server/server.key",
        http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            mux.ServeHTTP(w, r)
            return
        }),
    )
}
```

而单独启用带证书的gRPC服务也是同样的简单：

```go
func main() {
    creds, err := credentials.NewServerTLSFromFile("../conf/server/server.pem", "../conf/server/server.key")
    if err != nil {
        log.Fatal(err)
    }

    grpcServer := grpc.NewServer(grpc.Creds(creds))

    ...
}
```

因为gRPC服务已经实现了ServeHTTP方法，可以直接作为Web路由处理对象。如果将gRPC和Web服务放在一起，会导致gRPC和Web路径的冲突，在处理时我们需要区分两类服务。

我们可以通过以下方式生成同时支持Web和gRPC协议的路由处理函数：

```go
func main() {
    ...
    
    err = http.ListenAndServeTLS(":8972", "../conf/server/server.pem", "../conf/server/server.key",
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			if r.ProtoMajor == 2 && strings.Contains(r.Header.Get("Content-Type"), "application/grpc") {
				grpcServer.ServeHTTP(w, r)
				return
			}
			mux.ServeHTTP(w, r)
			return
		}))

	if err != nil {
		log.Fatal(err)
	}
}
```

首先gRPC是建立在HTTP/2版本之上，如果HTTP不是HTTP/2协议则必然无法提供gRPC支持。同时，每个gRPC调用请求的Content-Type类型会被标注为"application/grpc"类型。

这样我们就可以在gRPC端口上同时提供Web服务了。

### 使用 cmux 让 grpc 服务和 http 服务监听同一个端口

上面的方法存在一点问题，不能同时访问同一个端口。使用[cmux](github.com/soheilhy/cmux)可以实现这一功能。但是cmux不支持TLS。

```go
// server/main.go
package main

import (
	"context"
	"fmt"
	"github.com/soheilhy/cmux"
	"golang.org/x/sync/errgroup"
	"google.golang.org/grpc"
	"grpc-web/pb"
	"io/ioutil"
	"log"
	"net"
	"net/http"
)

// 使用cmux让grpc和http监听同一个端口
// 注意cmux不支持TLS

type HelloService struct{}

func (p *HelloService) Hello(_ context.Context, req *pb.String) (*pb.String, error) {
	reply := &pb.String{Value: "Hello, " + req.GetValue() + "!"}
	return reply, nil
}

func main() {
	// 创建监听
	l, err := net.Listen("tcp", ":8972")
	if err != nil {
		log.Fatal(err)
	}

	m := cmux.New(l)

	// 创建grpc监听器
	grpcListener := m.MatchWithWriters(cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"))

	// 其余的都是http监听
	httpListener := m.Match(cmux.Any())

	// 创建服务
	grpcServer := grpc.NewServer()
	pb.RegisterHelloServiceServer(grpcServer, new(HelloService))

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		file, err := ioutil.ReadFile("./index.html")
		if err != nil {
			_, err := w.Write([]byte(fmt.Sprintf("%v\n", err)))
			if err != nil {
				log.Fatal(err)
			}
		}
		_, err = w.Write(file)
		if err != nil {
			log.Fatal(err)
		}
	})
	httpServer := &http.Server{}

	// 使用error group启动所有服务
	g := errgroup.Group{}
	g.Go(func() error {
		return grpcServer.Serve(grpcListener)
	})
	g.Go(func() error {
		return httpServer.Serve(httpListener)
	})
	g.Go(func() error {
		return m.Serve()
	})

	// 等待执行，检查错误
	if err = g.Wait(); err != nil {
		log.Fatal(err)
	}
}
```



# gRPC和Protobuf扩展

目前开源社区已经围绕Protobuf和gRPC开发出众多扩展，形成了庞大的生态。本节我们将简单介绍验证器和REST接口扩展。

## 验证器

到目前为止，我们接触的全部是第三版的Protobuf语法。第二版的Protobuf有个默认值特性，可以为字符串或数值类型的成员定义默认值。

我们采用第二版的Protobuf语法创建文件：

```protobuf
syntax = "proto2";

package main;

message Message {
    optional string name = 1 [default = "gopher"];
    optional int32 age = 2 [default = 10];
}
```

内置的默认值语法其实是通过Protobuf的扩展选项特性实现。在第三版的Protobuf中不再支持默认值特性，但是我们可以通过扩展选项自己模拟默认值特性。

下面是用proto3语法的扩展特性重新改写上述的proto文件：

```protobuf
syntax = "proto3";

package main;

import "google/protobuf/descriptor.proto";

extend google.protobuf.FieldOptions {
    string default_string = 50000;
    int32 default_int = 50001;
}

message Message {
    string name = 1 [(default_string) = "gopher"];
    int32 age = 2[(default_int) = 10];
}
```

其中成员后面的方括号内部的就是扩展语法。重新生成Go语言代码，里面会包含扩展选项相关的元信息：

```go
var E_DefaultString = &proto.ExtensionDesc{
    ExtendedType:  (*descriptor.FieldOptions)(nil),
    ExtensionType: (*string)(nil),
    Field:         50000,
    Name:          "main.default_string",
    Tag:           "bytes,50000,opt,name=default_string,json=defaultString",
    Filename:      "helloworld.proto",
}

var E_DefaultInt = &proto.ExtensionDesc{
    ExtendedType:  (*descriptor.FieldOptions)(nil),
    ExtensionType: (*int32)(nil),
    Field:         50001,
    Name:          "main.default_int",
    Tag:           "varint,50001,opt,name=default_int,json=defaultInt",
    Filename:      "helloworld.proto",
}
```

我们可以在运行时通过类似反射的技术解析出Message每个成员定义的扩展选项，然后从每个扩展的相关联的信息中解析出我们定义的默认值。

在开源社区中，github.com/mwitkow/go-proto-validators 已经基于Protobuf的扩展特性实现了功能较为强大的验证器功能。要使用该验证器首先需要下载其提供的代码生成插件：

```
$ go get github.com/mwitkow/go-proto-validators/protoc-gen-govalidators
```

然后基于go-proto-validators验证器的规则为Message成员增加验证规则：

```protobuf
syntax = "proto3";

package main;

import "github.com/mwitkow/go-proto-validators/validator.proto";

message Message {
    string important_string = 1 [
        (validator.field) = {regex: "^[a-z]{2,5}$"}
    ];
    int32 age = 2 [
        (validator.field) = {int_gt: 0, int_lt: 100}
    ];
}
```

在方括弧表示的成员扩展中，validator.field表示扩展是validator包中定义的名为field扩展选项。validator.field的类型是FieldValidator结构体，在导入的validator.proto文件中定义。

所有的验证规则都由validator.proto文件中的FieldValidator定义：

```protobuf
syntax = "proto2";
package validator;

import "google/protobuf/descriptor.proto";

extend google.protobuf.FieldOptions {
    optional FieldValidator field = 65020;
}

message FieldValidator {
    // Uses a Golang RE2-syntax regex to match the field contents.
    optional string regex = 1;
    // Field value of integer strictly greater than this value.
    optional int64 int_gt = 2;
    // Field value of integer strictly smaller than this value.
    optional int64 int_lt = 3;

    // ... more ...
}
```

从FieldValidator定义的注释中我们可以看到验证器扩展的一些语法：其中regex表示用于字符串验证的正则表达式，int_gt和int_lt表示数值的范围。

然后采用以下的命令生成验证函数代码：

```
protoc  \
    --proto_path=${GOPATH}/src \
    --proto_path=${GOPATH}/src/github.com/google/protobuf/src \
    --proto_path=. \
    --govalidators_out=. --go_out=plugins=grpc:.\
    hello.proto
```

> windows:替换 `${GOPATH}` 为 `%GOPATH%` 即可.

以上的命令会调用protoc-gen-govalidators程序，生成一个独立的名为hello.validator.pb.go的文件：

```go
var _regex_Message_ImportantString = regexp.MustCompile("^[a-z]{2,5}$")

func (this *Message) Validate() error {
    if !_regex_Message_ImportantString.MatchString(this.ImportantString) {
        return go_proto_validators.FieldError("ImportantString", fmt.Errorf(
            `value '%v' must be a string conforming to regex "^[a-z]{2,5}$"`,
            this.ImportantString,
        ))
    }
    if !(this.Age > 0) {
        return go_proto_validators.FieldError("Age", fmt.Errorf(
            `value '%v' must be greater than '0'`, this.Age,
        ))
    }
    if !(this.Age < 100) {
        return go_proto_validators.FieldError("Age", fmt.Errorf(
            `value '%v' must be less than '100'`, this.Age,
        ))
    }
    return nil
}
```

生成的代码为Message结构体增加了一个Validate方法，用于验证该成员是否满足Protobuf中定义的条件约束。无论采用何种类型，所有的Validate方法都用相同的签名，因此可以满足相同的验证接口。

通过生成的验证函数，并结合gRPC的截取器，我们可以很容易为每个方法的输入参数和返回值进行验证。

## REST接口

gRPC服务一般用于集群内部通信，如果需要对外暴露服务一般会提供等价的REST接口。通过REST接口比较方便前端JavaScript和后端交互。开源社区中的grpc-gateway项目就实现了将gRPC服务转为REST服务的能力。

grpc-gateway的工作原理如图2所示：

![img](https://imw7.github.io/images/Go/grpc/grpc-gateway.png)



*图 2 gRPC-Gateway工作流程*

通过在Protobuf文件中添加路由相关的元信息，通过自定义的代码插件生成路由相关的处理代码，最终将REST请求转给更后端的gRPC服务处理。

路由扩展元信息也是通过Protobuf的元数据扩展用法提供：

```protobuf
syntax = "proto3";

package pb;
option go_package = "../pb";

import "google/api/annotations.proto";

message StringMessage {
  string value = 1;
}

service RestService {
  rpc Get(StringMessage) returns (StringMessage) {
    option (google.api.http) = {
      get: "/get/{value}"
    };
  }
  rpc Post(StringMessage) returns (StringMessage) {
    option (google.api.http) = {
      post: "/post"
      body: "*"
    };
  }
}
```

我们首先为gRPC定义了Get和Post方法，然后通过元扩展语法在对应的方法后添加路由信息。其中“/get/{value}”路径对应的是Get方法，`{value}`部分对应参数中的value成员，结果通过json格式返回。Post方法对应“/post”路径，body中包含json格式的请求信息。

然后通过以下命令安装protoc-gen-grpc-gateway插件：

```bash
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
```

再通过插件生成grpc-gateway必须的路由处理代码：

```bash
$ protoc -I/usr/local/include -I. \
    -I$GOPATH/src \
    -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
    --grpc-gateway_out=. --go_out=plugins=grpc:.\
    hello.proto
```

> windows:替换 `${GOPATH}$` 为 `%GOPATH%` 即可.

***注意***：这一步可能会出现错误类似：

1. 使用上面的protoc语句生成代码时出现：

```bash
Import “google/api/annotations.proto” was not found or had errors.
```

出现这个错误，需要从[这个](https://github.com/zh-and/grpc-gateway)链接下载压缩文件，解压后将其中的`third_party`文件夹复制到`${GOPATH}$/src/github.com/grpc-ecosystem/grpc-gateway`路径下。

2. 使用插件生成代码时出现错误：

```bash
--go_out: protoc-gen-go: plugins are not supported; use 'protoc --go-grpc_out=...' to generate gRPC
```

这样的错误，老版的`protoc-gen-go`不支持，使用下面的语句安装新版本:

```bash
go get github.com/golang/protobuf/protoc-gen-go
```

执行完成后，会在项目中新生成`hello.pb.go`和`hello.pb.gw.go`两个文件。

插件会为`RestService`服务生成对应的`RegisterRestServiceHandleFromEndpoint`函数：

```go
func RegisterRestServiceHandlerFromEndpoint(
    ctx context.Context, mux *runtime.ServeMux, 
    endpoint string, opts []grpc.DialOption) (err error) {
    ...
}
```

`RegisterRestServiceHandleFromEndpoint`函数用于将定义了REST接口的请求转发到真正的gRPC服务。注册路由处理函数之后就可以启动Web服务了。

```go
func main() {
    ctx := context.Background()
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    mux := runtime.NewServeMux()

    err := RegisterRestServiceHandlerFromEndpoint(
        ctx, mux, "localhost:5000",
        []grpc.DialOption{grpc.WithInsecure()},
    )
    if err != nil {
        log.Fatal(err)
    }

    http.ListenAndServe(":8080", mux)
}
```

启动grpc服务，端口5000

```go
type RestServiceImpl struct{}

func (r *RestServiceImpl) Get(ctx context.Context, message *StringMessage) (*StringMessage, error) {
    return &StringMessage{Value: "Get:" + message.Value}, nil
}

func (r *RestServiceImpl) Post(ctx context.Context, message *StringMessage) (*StringMessage, error) {
    return &StringMessage{Value: "Post:" + message.Value}, nil
}
func main() {
    grpcServer := grpc.NewServer()
    RegisterRestServiceServer(grpcServer, new(RestServiceImpl))
    lis, _ := net.Listen("tcp", ":5000")
    grpcServer.Serve(lis)
}
```

首先通过runtime.NewServeMux()函数创建路由处理器，然后通过RegisterRestServiceHandlerFromEndpoint函数将RestService服务相关的REST接口中转到后面的gRPC服务。grpc-gateway提供的runtime.ServeMux类也实现了http.Handler接口，因此可以和标准库中的相关函数配合使用。

当gRPC和REST服务全部启动之后，就可以用curl请求REST服务了：

```bash
$ curl localhost:8080/get/gopher
{"value":"Get: gopher"}

$ curl localhost:8080/post -X POST --data '{"value":"grpc"}'
{"value":"Post: grpc"}
```

在对外公布REST接口时，我们一般还会提供一个Swagger格式的文件用于描述这个接口规范。

```bash
$ go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger

$ protoc -I. \
  -I$GOPATH/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
  --swagger_out=. \
  hello.proto
```

然后会生成一个hello.swagger.json文件。这样的话就可以通过swagger-ui这个项目，在网页中提供REST接口的文档和测试等功能。

