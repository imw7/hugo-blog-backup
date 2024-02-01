+++
title='Go kit教程'
tags=['Go', 'go-kit', '微服务']
categories=['Go']
date="2023-12-30T12:06:43+08:00"
toc=true
draft=false

+++

本文为`go-kit`的教程。<!--more-->

# 基础示例

[`go-kit`](https://gokit.io/)不能算是一个框架，可以说是一个`Go`语言开发用的工具箱/套件。

## Go kit介绍

`Go kit`是`Go`包（库）的集合，它可以帮助你构建健壮、可靠、可维护的微服务。`Go kit`通过提供成熟的模式和习惯用法来降低`Go`和微服务的风险，这些模式和习惯用法由一大群经验丰富的贡献者编写和维护，并在生产环境中得到验证。

## 架构和设计

### Go kit关键概念

使用`Go kit`构建的服务分为三层：

1. 传输层（`Transport layer`）
2. 端点层（`Endpoint layer`）
3. 服务层（`Service layer`）

请求在第1层进入服务，向下流到第3层，响应则相反。

#### Transports

传输域绑定到具体的传输协议，如`HTTP`或`gRPC`。在一个微服务可能支持一个或多个传输协议的世界中，这是非常强大的：你可以在单个微服务中支持原有的`HTTP API`和新增的`RPC`服务。

当实现`REST`式的`HTTP API`时，你的路由是在`HTTP`传输中定义的。最常见的路由定义在`HTTP`路由器函数中，如下所示：

```go
r.Methods("POST").Path("/profiles/").Handler(httptransport.NewServer{
    e.PostProfileEndpoint,
    decodePostProfileRequest,
    encodeResponse,
    options...,
})
```

#### Endpoints

端点就像控制器上的动作/处理程序；它是安全性和抗脆弱性逻辑的所在。如果实现两种传输（`HTTP`和`gRPC`），则可能有两种将请求发送到同一端点的方法。

#### Services

服务（指`Go kit`中的`service`层）是实现所有业务逻辑的地方。服务层通常将多个端点粘合在一起。在`Go kit`中，服务层通常被抽象为接口，这些接口的实现包含业务逻辑。`Go kit`服务层应该努力遵守整洁架构或六边形架构。也就是说，业务逻辑不需要了解端点（尤其是传输域）概念：你的服务层不应该关心`HTTP`头或`gRPC`错误代码。

#### Middlewares

`Go kit`试图通过使用中间件（或装饰器）模式来执行严格的关注分离（`separation of concerns`）。中间件可以包装端点或服务以添加功能，比如日志记录、速率限制、负载平衡或分布式跟踪。围绕一个端点或服务链接多个中间件是很常见的。

将所有这些概念放在一起，我们可以看到`Go kit`构建的微服务就像洋葱一样有许多层。

![onion](https://blog.imw7.com/images/Go/go_kit/onion.png)

这些层可以分组到我们的三个域中：

* 最内层的 **服务（`service`）** 域是所有内容都基于特定服务定义的地方，也是实现所有业务逻辑的地方。
* 中间 **端点（`endpoint`）** 域是将服务的每个方法抽象为通用[endpoint.Endpoint](https://pkg.go.dev/github.com/go-kit/kit/endpoint#Endpoint)以及实现安全性和抗脆弱性逻辑的位置。
* 最外层的 **传输（`transport`）** 域是端点绑定到`HTTP`或`gRPC`等具体传输的地方。

你可以通过为服务定义接口并提供具体的实现来实现核心业务逻辑。然后，编写服务中间件来提供额外的功能，比如日志记录、分析、检测——任何需要了解业务领域知识的东西。

`Go kit`提供端点和传输域中间件，用于诸如速率限制、断路、负载平衡和分布式跟踪等功能——所有这些功能通常与你的业务域无关。

简而言之，`Go kit`试图通过精心使用中间件（或装饰器）模式来强制执行严格的关注分离（**`separation of concerns`**）。

## 快速开始

接下来就演示如何使用`Go kit`快速实现一个微服务。

在本机上新建项目目录`go-kit-demo`，并在项目目录下执行`go mod init go-kit-demo`完成项目的初始化。

### 业务逻辑

服务从业务逻辑开始写起。在`Go kit`中，我们将服务建模为一个接口。

```go
// AddService 把两个东西加到一起
type AddService interface {
    Sum(ctx context.Context, a, b int) (int, error)
    Concat(ctx context.Context, a, b string) (string, error)
}
```

这个接口有一个实现。

```go
type addService struct{}

const maxLen = 10

var (
	// ErrTwoZeros Sum方法的业务规则不能对两个0求和
	ErrTwoZeros = errors.New("can't sum two zeros")

	// ErrIntOverflow Sum参数越界
	ErrIntOverflow = errors.New("integer overflow")

	// ErrTwoEmptyStrings Concat方法业务规则规定参数不能是两个空字符串
	ErrTwoEmptyStrings = errors.New("can't concat two empty strings")

	// ErrMaxSizeExceeded Concat方法的参数超出范围
	ErrMaxSizeExceeded = errors.New("result exceeds maximum size")
)

// Sum 对两个数字求和，实现AddService
func (s addService) Sum(_ context.Context, a, b int) (int, error) {
	if a == 0 && b == 0 {
		return 0, ErrTwoZeros
	}
	if (b > 0 && a > (math.MaxInt-b)) || (b < 0 && a < (math.MinInt-b)) {
		return 0, ErrIntOverflow
	}
	return a + b, nil
}

func (s addService) Concat(_ context.Context, a, b string) (string, error) {
	if a == "" && b == "" {
		return "", ErrTwoEmptyStrings
	}
	if len(a)+len(b) > maxLen {
		return "", ErrMaxSizeExceeded
	}
	return a + b, nil
}
```

### 请求和响应

在`Go kit`中，主要的消息模式是`RPC`。因此，我们接口中的每个方法都将被建模为一个远程过程调用。对于每个方法，我们定义**请求和响应**结构体，分别捕获所有的输入和输出参数。

```go
// SumRequest Sum方法的参数.
type SumRequest struct {
	A int `json:"a"`
	B int `json:"b"`
}

// SumResponse Sum方法的响应
type SumResponse struct {
	V   int    `json:"v"`
	Err string `json:"err,omitempty"`
}

// ConcatRequest Concat方法的参数.
type ConcatRequest struct {
	A string `json:"a"`
	B string `json:"b"`
}

// ConcatResponse  Concat方法的响应.
type ConcatResponse struct {
	V   string `json:"v"`
	Err string `json:"err,omitempty"`
}
```

### Endpoints

`Go kit`通过一个称为 **`endpoint`** 的抽象提供了许多功能。`Endpoint`的定义如下：

```go
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```

它表示单个`RPC`。也就是说，我们的服务接口中只有一个方法。我们将编写简单的适配器来将服务的每个方法转换为一个端点。每个适配器接受一个`AddService`，并返回与其中一个方法对应的端点。

```go
import "github.com/go-kit/kit/endpoint"

func makeSumEndpoint(svc AddService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(SumRequest)
		v, err := svc.Sum(ctx, req.A, req.B)
		if err != nil {
			return SumResponse{V: v, Err: err.Error()}, nil
		}
		return SumResponse{V: v}, nil
	}
}

func makeConcatEndpoint(svc AddService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(ConcatRequest)
		v, err := svc.Concat(ctx, req.A, req.B)
		if err != nil {
			return ConcatResponse{V: v, Err: err.Error()}, nil
		}
		return ConcatResponse{V: v}, nil
	}
}
```

### Transports

现在我们需要将编写的服务公开给外部世界，这样就可以调用它了。`Go kit`开箱即用的支持`gRPC`、`Thrift`或者基于`HTTP`的`JSON`。

这里我们先演示如何使用`HTTP`之上的`JSON`作为传输协议。

```go
import httptransport "github.com/go-kit/kit/transport/http"

func decodeSumRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request SumRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeCountRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request ConcatRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}

func main() {
	svc := addService{}

	sumHandler := httptransport.NewServer(
		makeSumEndpoint(svc),
		decodeSumRequest,
		encodeResponse,
	)

	concatHandler := httptransport.NewServer(
		makeConcatEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/sum", sumHandler)
	http.Handle("/concat", concatHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 运行

在项目目录下编译得到可执行文件并运行，服务会在本机的`8080`端口启动。我们使用`curl`或`postman`测试我们的服务。

```bash
$ curl -XPOST -d'{"a":1,"b":2}' localhost:8080/sum
{"v":3}
$ curl -XPOST -d'{"a":"你好","b":"eric"}' localhost:8080/concat
{"v":"你好eric"}
```

# gRPC

下面介绍如何使用`Go kit`构建基于`gRPC`的微服务。

## 基于gRPC通信

想要实现基于`gRPC`的通信，首先需要定义好`proto`文件，并生成对应的`Go`代码和`gRPC`代码。

### 定义protobuf

根据`addsrv`业务的实际需要，我们定义的`proto`文件内容如下。

```protobuf
syntax = "proto3";

package pb;

option go_package = "go-kit-demo/pb";

service Add {
  // Sum 对两个数字求和
  rpc Sum (SumRequest) returns (SumResponse) {}

  // Concat 方法拼接两个字符串
  rpc Concat (ConcatRequest) returns (ConcatResponse) {}
}

// Sum方法的请求参数
message SumRequest {
  int64 a = 1;
  int64 b = 2;
}

// Sum方法的响应
message SumResponse {
  int64 v = 1;
  string err = 2;
}

// Concat方法的请求参数
message ConcatRequest {
  string a = 1;
  string b = 2;
}

// Concat方法的响应
message ConcatResponse {
  string v = 1;
  string err = 2;
}
```

将上面的文件保存至项目目录下的`pb/addsrv.proto`文件中。

执行下面的命令根据上述`proto`文件编译生成`go`代码（需事先安装好`protoc`和`protoc-gen-go-grpc`）。

```bash
protoc -I=pb \
 --go_out=pb --go_opt=paths=source_relative \
 --go-grpc_out=pb --go-grpc_opt=paths=source_relative \
 pb/addsrv.proto
```

此时项目目录如下：

```bas
├── go.mod
├── go.sum
├── main.go
└── pb
    ├── addsrv_grpc.pb.go
    ├── addsrv.pb.go
    └── addsrv.proto
```

### grpcServer

在`main.go`中定义好`grpcServer`结构体，其内部包含`sum`和`concat`两个`grpctransport.Handler`。

```go
import grpctransport "github.com/go-kit/kit/transport/grpc"

type grpcServer struct {
    pb.UnimplementedAddServer
    sum	   grpctransport.Handler
    concat grpctransport.Handler
}
```

`grpctransport.Handler`本质上是一个接口类型。

```go
// Handler 应该从服务实现的gRPC绑定调用。
// 传入的请求参数和返回的响应参数都是gRPC类型，而不是用户域类型。
type Handler interface {
	ServeGRPC(ctx context.Context, request interface{}) (context.Context, interface{}, error)
}
```

那么该如何得到`grpctransport.Handler`呢？与上面获取`httptransport.Handler`类似。

我们先定义好处理请求和响应数据的编解码函数。

```go
// decodeGRPCSumRequest 将Sum方法的gRPC请求参数转为内部的SumRequest
func decodeGRPCSumRequest(_ context.Context, grpcReq interface{}) (interface{}, error) {
	req := grpcReq.(*pb.SumRequest)
	return SumRequest{A: int(req.A), B: int(req.B)}, nil
}

// decodeGRPCConcatRequest 将Concat方法的gRPC请求参数转为内部的ConcatRequest
func decodeGRPCConcatRequest(_ context.Context, grpcReq interface{}) (interface{}, error) {
	req := grpcReq.(*pb.ConcatRequest)
	return ConcatRequest{A: req.A, B: req.B}, nil
}

// encodeGRPCSumResponse 封装Sum的gRPC响应 
func encodeGRPCSumResponse(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(SumResponse)
	return &pb.SumResponse{V: int64(resp.V), Err: resp.Err}, nil
}

// encodeGRPCConcatResponse 封装Concat的gRPC响应
func encodeGRPCConcatResponse(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(ConcatResponse)
	return &pb.ConcatResponse{V: resp.V, Err: resp.Err}, nil
}
```

有了编解码的处理函数后，便可以通过`grpctransport.NewServer`得到`grpctransport.Handler`。

```go
// NewGRPCServer grpcServer构造函数
func NewGRPCServer(svc AddService) pb.AddServer {
	return &grpcServer{
		sum: grpctransport.NewServer(
			makeSumEndpoint(svc),
			decodeGRPCSumRequest,
			encodeGRPCSumResponse,
		),
		concat: grpctransport.NewServer(
			makeConcatEndpoint(svc),
			decodeGRPCConcatRequest,
			encodeGRPCConcatResponse,
		),
	}
}
```

最后再为我们的`grpcServer`实现服务。

```go
func (s *grpcServer) Sum(ctx context.Context, req *pb.SumRequest) (*pb.SumResponse, error) {
	_, rep, err := s.sum.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return rep.(*pb.SumResponse), nil
}

func (s *grpcServer) Concat(ctx context.Context, req *pb.ConcatRequest) (*pb.ConcatResponse, error) {
	_, rep, err := s.concat.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return rep.(*pb.ConcatResponse), nil
}
```

### 启动gRPC服务

```go
svc := addService{}

gs := NewGRPCServer(svc)

listener, err := net.Listen("tcp", ":8972")
if err != nil {
	fmt.Println("failed to listen:", err)
	return
}
s := grpc.NewServer()       // 创建gRPC服务器
pb.RegisterAddServer(s, gs) // 在gRPC服务端注册服务
// 启动服务
err = s.Serve(listener)
if err != nil {
	fmt.Printf("failed to serve: %v", err)
	return
}
```

## 测试

编写测试代码，验证`Sum`和`Concat`这两个`RPC`方法都正常工作。

```go
// add_test.go
package main

import (
	"context"
	"github.com/stretchr/testify/assert"
	"go-kit-demo/pb"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/test/bufconn"
	"log"
	"net"
	"testing"
)

// 使用bufconn构建测试链接，避免使用实际端口号启动服务

const bufSize = 1024 * 1024

var bufListener *bufconn.Listener

func init() {
	bufListener = bufconn.Listen(bufSize)
	s := grpc.NewServer()
	gs := NewGRPCServer(addService{})
	pb.RegisterAddServer(s, gs)
	go func() {
		if err := s.Serve(bufListener); err != nil {
			log.Fatalln("Server exited with error:", err)
		}
	}()
}

func bufDialer(context.Context, string) (net.Conn, error) {
	return bufListener.Dial()
}

func TestSum(t *testing.T) {
	conn, err := grpc.DialContext(
		context.Background(),
		"bufnet",
		grpc.WithContextDialer(bufDialer),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalln("did not connect:", err)
	}
	defer func() { _ = conn.Close() }()
	c := pb.NewAddClient(conn)

	rsp, err := c.Sum(context.Background(), &pb.SumRequest{
		A: 10,
		B: 2,
	})
	assert.Nil(t, err)
	assert.NotNil(t, rsp)
	assert.Equal(t, int64(12), rsp.V)
}

func TestConcat(t *testing.T) {
	conn, err := grpc.DialContext(
		context.Background(),
		"bufnet",
		grpc.WithContextDialer(bufDialer),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewAddClient(conn)

	rsp, err := c.Concat(context.Background(), &pb.ConcatRequest{
		A: "10",
		B: "2",
	})
	assert.Nil(t, err)
	assert.NotNil(t, rsp)
	assert.Equal(t, "102", rsp.V)
}
```

项目目录下执行下面的命令，并查看测试结果。

```bash
$ go test -v ./...
?       go-kit-demo/pb  [no test files]
=== RUN   TestSum
--- PASS: TestSum (0.00s)
=== RUN   TestConcat
--- PASS: TestConcat (0.00s)
PASS
ok      go-kit-demo     0.004s
```

# 代码分层

这一部分介绍如何使用`Go kit`编写的项目代码如何按请求进行分层，从而提升代码的可读性。

我们目前所有的项目代码都保存在`main.go`文件中，随着服务中`endpoint`数量的增加，将调用过程的每一层分隔到单独的文件中，可以提高`go-kit`项目的可读性。

## 分离关注点

遵循分离关注点（`Separation of concerns`）的设计理念，将完整的请求流程划分为`service`、`transport`和`endpoint`三层，每一层专注于实现特定的功能。

### service

`service`层负责我们业务逻辑的实现。

在项目下新建 `service.go` 文件，将与业务逻辑相关的代码保存至 `service.go` 文件中。

```go
// service.go
import (
	"context"
    "errors"
)

// AddService 列出当前服务所有RPC方法的接口类型
type AddService interface {
    Sum(ctx context.Context, a, b int) (int, error)
    Concat(ctx context.Context, a, b string) (string, error)
}

// addService 实现AddService接口
type addService struct {
    // ...
}

var (
	// ErrEmptyString 两个参数都是空字符串的错误
    ErrEmptyString = errors.New("两个参数都是空字符串")
)

// Sum 返回两个数的和
func (addService) Sum(_ context.Context, a, b int) (int, error) {
    // 业务逻辑
    return a + b, nil
}

// Concat 拼接两个字符串
func (addService) Concat(_ context.Context, a, b string) (string, error) {
    if a == "" && b == "" {
        return "", ErrEmptyString
    }
    return a + b, nil
}

// NewService 创建一个add service
func NewService() AddService {
    return &addService{}
}
```

### endpoint

`endpoint`层负责存放我们项目中对外暴露的`RPC`方法。

将以下代码存放在项目目录下的`endpoint.go`文件中。

```go
// endpoint.go

import (
	"context"
    "github.com/go-kit/kit/endpoint"
)

type SumRequest struct {
    A int `json:"a"`
    B int `json:"b"`
}

type SumResponse struct {
    V 	int    `json:"v"`
    Err string `json:"err,omitempty"`
}

type ConcatRequest struct {
    A string `json:"a"`
    B string `json:"b"`
}

type ConcatResponse struct {
    V 	int    `json:"v"`
    Err string `json:"err,omitempty"`
}

func makeSumEndpoint(srv AddService) endpoint.Endpoint {
    return func(ctx context.Context, request interface{}) (interface{}, error) {
        req := request.(SumRequest)
        v, err := srv.Sum(ctx, req.A, req.B) // 方法调用
        if err != nil {
            return SumResponse{V: v, Err: err.Error()}, nil
        }
        return SumResponse{V: v}, nil
    }
}

func makeConcatEndpoint(srv AddService) endpoint.Endpoint {
    return func(ctx context.Context, request interface{}) (interface{}, error) {
        req := request.(ConcatRequest)
        v, err := srv.Concat(ctx, req.A, req.B) // 方法调用
        if err != nil {
            return ConcatResponse{V: v, Err: err.Error()}, nil
        }
        return ConcatResponse{V: v}, nil
    }
}
```

### transport

`transport`层表示项目对外通信相关的部分，包括对外支持的协议等内容。

将项目中与网络传输相关的代码保存至项目目录下的`transport.go`文件中。

```go
// transport.go

import (
	"context"
	"encoding/json"
	grpctransport "github.com/go-kit/kit/transport/grpc"
	httptransport "github.com/go-kit/kit/transport/http"
	"github.com/gorilla/mux"
	"go-kit-demo2/pb"
	"net/http"
)

// gRPC的请求与响应
// decodeGRPCSumRequest 将Sum方法的gRPC请求参数转为内部的SumRequest
func decodeGRPCSumRequest(_ context.Context, grpcReq interface{}) (interface{}, error) {
	req := grpcReq.(*pb.SumRequest)
	return SumRequest{A: int(req.A), B: int(req.B)}, nil
}

// decodeGRPCConcatRequest 将Concat方法的gRPC请求参数转为内部的ConcatRequest
func decodeGRPCConcatRequest(_ context.Context, grpcReq interface{}) (interface{}, error) {
	req := grpcReq.(*pb.ConcatRequest)
	return ConcatRequest{A: req.A, B: req.B}, nil
}

// encodeGRPCSumResponse 封装Sum的gRPC响应
func encodeGRPCSumResponse(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(SumResponse)
	return &pb.SumResponse{V: int64(resp.V), Err: resp.Err}, nil
}

// encodeGRPCConcatResponse 封装Concat的gRPC响应
func encodeGRPCConcatResponse(_ context.Context, response interface{}) (interface{}, error) {
	resp := response.(ConcatResponse)
	return &pb.ConcatResponse{V: resp.V, Err: resp.Err}, nil
}

// gRPC
type grpcServer struct {
	pb.UnimplementedAddServer

	sum    grpctransport.Handler
	concat grpctransport.Handler
}

func (s grpcServer) Sum(ctx context.Context, req *pb.SumRequest) (*pb.SumResponse, error) {
	_, resp, err := s.sum.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return resp.(*pb.SumResponse), nil
}

func (s grpcServer) Concat(ctx context.Context, req *pb.ConcatRequest) (*pb.ConcatResponse, error) {
	_, resp, err := s.concat.ServeGRPC(ctx, req)
	if err != nil {
		return nil, err
	}
	return resp.(*pb.ConcatResponse), nil
}

// NewGRPCServer 构造函数
func NewGRPCServer(svc AddService) pb.AddServer {
	return &grpcServer{
		sum: grpctransport.NewServer(
			makeSumEndpoint(svc), // endpoint
			decodeGRPCSumRequest,
			encodeGRPCSumResponse,
		),
		concat: grpctransport.NewServer(
			makeConcatEndpoint(svc),
			decodeGRPCConcatRequest,
			encodeGRPCConcatResponse,
		),
	}
}

// HTTP
func decodeSumRequest(ctx context.Context, r *http.Request) (interface{}, error) {
	var request SumRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeConcatRequest(ctx context.Context, r *http.Request) (interface{}, error) {
	var request ConcatRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func encodeResponse(ctx context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}

// NewHTTPServer HTTP Server
func NewHTTPServer(svc AddService) http.Handler {
	sumHandler := httptransport.NewServer(
		makeSumEndpoint(svc),
		decodeSumRequest,
		encodeResponse,
	)

	concatHandler := httptransport.NewServer(
		makeConcatEndpoint(svc),
		decodeConcatRequest,
		encodeResponse,
	)
	// use github.com/gorilla/mux
	r := mux.NewRouter()
	r.Handle("/sum", sumHandler).Methods("POST")
	r.Handle("/concat", concatHandler).Methods("POST")

	// use gin
	// r := gin.Default()
	// r.POST("/sum", gin.WrapH(sumHandler))
	// r.POST("/concat", gin.WrapH(concatHandler))
	return r
}
```

### 程序入口

通过上面的示例将项目代码拆分之后，接下来可以通过以下代码将程序组织起来。

修改后的`main.go`文件内容如下，该程序将同时对外提供`HTTP API`和`gRPC API`。

```go
// main.go

package main

import (
	"flag"
	"fmt"
	"go-kit-demo2/pb"
	"golang.org/x/sync/errgroup"
	"google.golang.org/grpc"
	"net"
	"net/http"
)

var (
	httpAddr = flag.String("http-addr", ":8080", "HTTP listen address")
	grpcAddr = flag.String("grpc-addr", ":8972", "gRPC listen address")
)

func main() {
	bs := NewService()

	var g errgroup.Group

	// HTTP服务
	g.Go(func() error {
		httpListener, err := net.Listen("tcp", *httpAddr)
		if err != nil {
			fmt.Printf("http: net.Listen(tcp, %s) failed, err:%v\n", *httpAddr, err)
			return err
		}
		defer func() { _ = httpListener.Close() }()
		httpHandler := NewHTTPServer(bs)
		return http.Serve(httpListener, httpHandler)
	})

	g.Go(func() error {
		// gRPC服务
		grpcListener, err := net.Listen("tcp", *grpcAddr)
		if err != nil {
			fmt.Printf("grpc: net.Listen(tcp, %s) faield, err:%v\n", *grpcAddr, err)
			return err
		}
		defer func() { _ = grpcListener.Close() }()
		s := grpc.NewServer()
		pb.RegisterAddServer(s, NewGRPCServer(bs))
		return s.Serve(grpcListener)
	})

	if err := g.Wait(); err != nil {
		fmt.Println("server exit with err:", err)
	}
}
```

# 中间件和日志

接下来介绍`Go kit`中的中间件，并以日志中间件为例演示如何设计和实现中间件。

## 中间件

在`go-kit`中，对中间件的定义是：一个接收`Endpoint`并返回`Endpoint`的函数，具体定义如下。

```go
type Middleware func(Endpoint) Endpoint
```

在中间件接收`Endpoint`参数和返回`Endpoint`之间，你可以做任何事。

例如，下面的示例演示了如何实现一个基础的日志中间件。

```go
import (
	"context"
    "github.com/go-kit/kit/endpoint"
    "github.com/go-kit/log"
)

func loggingMiddleware(logger log.Logger) endpoint.Middleware {
    return func(next endpoint.Endpoint) endpoint.Endpoint {
        return func(ctx context.Context, request interface{}) (interface{}, error) {
            logger.log("msg", "calling endpoint")
            defer logger.Log("msg", "called endpoint")
            return next(ctx, request)
        }
    }
}
```

### transport层日志

这里想要记录`transport`层的日志信息，将先前的`NewHTTPServer`按如下方式进行改造即可。

```go
func NewHTTPServer(svc AddService, logger log.Logger) http.Handler {
	// sum
	sum := makeSumEndpoint(svc)
	// 使用loggingMiddleware为sum端点加上日志
	sum = loggingMiddleware(log.With(logger, "method", "sum"))(sum)
	sumHandler := httptransport.NewServer(
		sum,
		decodeSumRequest,
		encodeResponse,
	)

	// concat
	concat := makeConcatEndpoint(svc)
	// 使用loggingMiddleware为concat端点加上日志
	concat = loggingMiddleware(log.With(logger, "method", "concat"))(concat)
	concatHandler := httptransport.NewServer(
		concat,
		decodeConcatRequest,
		encodeResponse,
	)

	r := gin.Default()
	r.POST("/sum", gin.WrapH(sumHandler))
	r.POST("/concat", gin.WrapH(concatHandler))
	return r
}
```

在调用`NewHTTPServer`时按需传入初始化好的`logger`即可。

```go
// 初始化logger
logger := log.NewLogfmtLogger(os.Stderr)
httpHandler := NewHTTPServer(bs, logger)
```

### 应用层日志

如果要在应用程序层面添加日志，例如需要记录下详细的请求参数，那么就需要为我们的服务来定义中间件。

由于我们的 `AddService` 服务定义为接口类型，所以我们只需要定义一个新类型（把原先的实现和一个额外的`logger`包装起来）并实现这个接口即可。

先定义类型。

```go
type logMiddleware struct {
	logger log.Logger
	next   AddService
}
```

再实现接口。

```go
func (mw logMiddleware) Sum(ctx context.Context, a, b int) (res int, err error) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "sum",
			"a", a,
			"b", b,
			"output", res,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())
	res, err = mw.next.Sum(ctx, a, b)
	return
}

func (mw logMiddleware) Concat(ctx context.Context, a, b string) (res string, err error) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "sum",
			"a", a,
			"b", b,
			"output", res,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())
	res, err = mw.next.Concat(ctx, a, b)
	return
}

// NewLogMiddleware 创建一个带日志的add service
func NewLogMiddleware(logger log.Logger, svc AddService) AddService {
	return &logMiddleware{
		logger: logger,
		next:   svc,
	}
}
```

在程序入口处使用`NewLogMiddleware`来创建服务实体。

```go
logger := log.NewLogfmtLogger(os.Stderr)
bs := NewService()
bs = NewLogMiddleware(logger, bs)
```

### 集成zap日志库

上述示例默认使用的是`github.com/go-kit/log`，你也可以使用其他的日志库，例如下面示例中使用社区常用的`zap`日志库。

```go
type zapLogMiddleware struct {
	logger zap.Logger
	next   AddService
}
```

> 这里也可以将logger直接注入我们先前定义的结构体`addService`中。

当然，`go-kit`的中间件不仅仅是能用来实现日志中间件。社区里有很多的插件都是基于`go-kit`的中间件实现的。例如，限流（`ratelimit`）、熔断器（`circuitbreaker`）、指标采集（`metrics`）等。

### ratelimit

```go
import "golang.org/x/time/rate"

var (
	ErrRateLimit = errors.New("request rate limit")
)

// rateMiddleware 限流中间件
func rateMiddleware(limit *rate.Limiter) endpoint.Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(ctx context.Context, request interface{}) (interface{}, error) {
			if !limit.Allow() {
				return nil, ErrRateLimit
			}
			return next(ctx, request)
		}
	}
}
```

使用限流中间件。

```go
import "golang.org/x/time/rate"

sum = rateMiddleware(rate.NewLimiter(1, 1))(sum)
```

### metrics

在 `Go kit`中，检测意味着使用 `metrics`包来记录关于服务运行时行为的统计信息。计算已处理作业的数量、记录请求完成后的持续时间以及跟踪正在执行的操作的数量都将被视为检测工具。

我们可以使用与日志记录相同的中间件模式。

```go
import (
	"context"
	"fmt"
	"github.com/go-kit/kit/metrics"
    "time"
)

type instrumentingMiddleware struct {
	requestCount   metrics.Counter
	requestLatency metrics.Histogram
	countResult    metrics.Histogram
	next           AddService
}

func (mw instrumentingMiddleware) Sum(ctx context.Context, a, b int) (res int, err error) {
	defer func(begin time.Time) {
		lvs := []string{"method", "sum", "error", fmt.Sprint(err != nil)}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
		mw.countResult.Observe(float64(res))
	}(time.Now())

	res, err = mw.next.Sum(ctx, a, b)
	return
}

func (mw instrumentingMiddleware) Concat(ctx context.Context, a, b string) (res string, err error) {
	defer func(begin time.Time) {
		lvs := []string{"method", "concat", "error", "false"}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
	}(time.Now())

	res, err = mw.next.Concat(ctx, a, b)
	return
}
```

添加对外接口。

```go
import (
	stdprometheus "github.com/prometheus/client_golang/prometheus"
	kitprometheus "github.com/go-kit/kit/metrics/prometheus"
	"github.com/go-kit/kit/metrics"
)

// instrumentation
fieldKeys := []string{"method", "error"}
requestCount := kitprometheus.NewCounterFrom(stdprometheus.CounterOpts{
	Namespace: "my_group",
	Subsystem: "string_service",
	Name:      "request_count",
	Help:      "Number of requests received.",
}, fieldKeys)
requestLatency := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
	Namespace: "my_group",
	Subsystem: "string_service",
	Name:      "request_latency_microseconds",
	Help:      "Total duration of requests in microseconds.",
}, fieldKeys)
countResult := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
	Namespace: "my_group",
	Subsystem: "string_service",
	Name:      "count_result",
	Help:      "The result of each count method.",
}, []string{}) // no fields here

bs = instrumentingMiddleware{
	requestCount:   requestCount,
	requestLatency: requestLatency,
	countResult:    countResult,
	next:           bs,
}
```

不要忘了为`/metrics`添加路由。

```go
// 原生http注册路由
http.Handle("/metrics", promhttp.Handler())
// gin框架注册路由
r.GET("/metrics", gin.WrapH(promhttp.Handler()))
```

将程序启动后，访问http://localhost:8080/metrics就能拿到`metrics`数据了。

# 调用其他服务

接下来介绍如何使用`Go kit`作为`RPC`客户端调用其他微服务。在微服务项目中，我们的业务逻辑通常会依赖其他微服务，需要通过`RPC`调用其他微服务。`go-kit`提供传输中间件来解决出现的许多问题。

### trim_service

现在，假设`addService`服务需要调用另外一个微服务——`trim_service`来实现`Concat`方法。

`trim_service`服务是一个去除空格的服务，具体功能是接收一个字符串参数，将其中所有的空格去掉后再返回。

其中`trim.proto`内容如下：

```protobuf
syntax = "proto3";

package pb;

option go_package="trim_service/pb";

service Trim {
  rpc TrimSpace (TrimRequest) returns (TrimResponse) {}
}


// Trim方法的请求参数
message TrimRequest {
  string s = 1;
}

// Trim方法的响应
message TrimResponse {
  string s = 1;
}
```

具体实现。

```go
// main.go

package main

import (
	"context"
	"flag"
	"fmt"
	"google.golang.org/grpc"
	"net"
	"strings"
	"trim_service/pb"
)

var port = flag.Int("port", 8975, "service port")

// trim service

type server struct {
	pb.UnimplementedTrimServer
}

// TrimSpace 去除字符串参数中的空格
func (s *server) TrimSpace(_ context.Context, req *pb.TrimRequest) (*pb.TrimResponse, error) {
	ov := req.GetS()
	v := strings.ReplaceAll(ov, " ", "")
	fmt.Printf("ov:%s v:%v\n", ov, v)
	return &pb.TrimResponse{S: v}, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		fmt.Printf("failed to listen: %v", err)
		return
	}
	s := grpc.NewServer()
	pb.RegisterTrimServer(s, &server{})
	err = s.Serve(lis)
	if err != nil {
		fmt.Printf("failed to serve: %v", err)
		return
	}
}
```

将`trim_service`服务在本地的`8975`端口启动。

### go-kit grpc client

`go-kit`提供了中间件来帮助解决项目中需要调用其他的服务。

由于我们的项目中需要调用`trim_service`服务，我们这里定义一个 `withTrimMiddleware`结构体，然后就把它当作一个`ServiceMiddleware`去实现，就像之前定义的`logMiddleware`那样。

```go
type withTrimMiddleware struct {
	next        AddService
	trimService endpoint.Endpoint // 通过它调用其他的服务
}
```

### 客户端endpoint

#### endpoint

`endpoint`层定义相关请求参数和响应参数。

```go
type trimRequest struct {
	s string
}

type trimResponse struct {
	s string
}
```

使用`grpctransport.NewClient` 创建基于`gRPC client`的`endpoint`。

```go
func makeTrimEndpoint(conn *grpc.ClientConn) endpoint.Endpoint {
	return grpctransport.NewClient(
		conn,
		"pb.Trim",
		"TrimSpace",
		encodeTrimRequest,
		decodeTrimResponse,
		pb.TrimResponse{},
	).Endpoint()
}
```

此处的`endpoint`与我们之前定义的`endpoint`有所不同，它属于客户端`endpoint`。之前的`endpoint`是我们的程序直接服务（`serve`）的，而`TrimEndpoint`是用来调用外部请求（`invoke`）的。

#### transport层

transport层添加请求和响应的转换函数。

```go
// encodeTrimRequest 将内部使用的数据编码为proto
func encodeTrimRequest(_ context.Context, response interface{}) (request interface{}, err error) {
	resp := response.(trimRequest)
	return &pb.TrimRequest{S: resp.s}, nil
}

// decodeTrimResponse 解析pb消息
func decodeTrimResponse(_ context.Context, in interface{}) (interface{}, error) {
	resp := in.(*pb.TrimResponse)
	return trimResponse{s: resp.S}, nil
}
```

#### service层

service层定义包含客户端endpoint的`withTrimMiddleware`结构体，并为其实现`AddService`接口。

```go
type withTrimMiddleware struct {
	next        AddService
	trimService endpoint.Endpoint // trim 交给这个endpoint处理
}

func NewServiceWithTrim(trimEndpoint endpoint.Endpoint, svc AddService) AddService {
	return &withTrimMiddleware{
		trimService: trimEndpoint,
		next:        svc,
	}
}

func (mw withTrimMiddleware) Sum(ctx context.Context, a, b int) (res int, err error) {
	return mw.Sum(ctx, a, b) // 与之前一致
}

// Concat 方法需要先发起gRPC调用外部trim service服务
func (mw withTrimMiddleware) Concat(ctx context.Context, a, b string) (res string, err error) {
	// 先调用trim服务，去除字符串中可能存在的空格
	respA, err := mw.trimService(ctx, trimRequest{s: a}) // 请求trim服务处理a
	if err != nil {
		return "", err
	}
	respB, err := mw.trimService(ctx, trimRequest{s: b}) // 请求trim服务处理b
	if err != nil {
		return "", err
	}
	trimA := respA.(trimResponse)
	trimB := respB.(trimResponse)
	return mw.next.Concat(ctx, trimA.s, trimB.s)
}
```

最后，在程序的入口处需要先初始化gRPC client，再通过`NewServiceWithTrim`创建server。

```go
// init grpc client
conn, err := grpc.Dial(*trimAddr, grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
	fmt.Printf("connect %s failed, err: %v", *trimAddr, err)
	return
}
defer conn.Close()
trimEndpoint := makeTrimEndpoint(conn)
bs = NewServiceWithTrim(trimEndpoint, bs)
```

### 测试

```bash
curl --location --request POST 'localhost:8080/concat' \
--header 'Content-Type: application/json' \
--data-raw '{
    "a":"1 0 1",
    "b":"2"
}'
```

返回结果

```bash
1012
```

# 服务发现和负载均衡

接下来介绍如何使用`Go kit`实现基于`consul`的服务发现和负载均衡。

在上一节，我们在项目中调用了其他的服务（1个实例），但实际生产环境下可能会有很多个服务实例。我们需要通过某种机制去实现服务发现和负载均衡。如果这些实例中的任何一个开始出现问题，我们希望在不影响我们自己服务的可靠性的情况下处理这个问题。

## 服务发现

### Endpoint

`Go kit`为不同的服务发现系统（`eureka`、`zookeeper`、`consul`、`etcd`等）提供适配器，`Endpointer`负责监听服务发现系统，并根据需要生成一组相同的端点。

```go
type Endpointer interface {
    Endpoints() ([]endpoint.Endpoint, error)
}
```

`Go kit`提供了工厂函数——`Factory`，它是一个将实现字符串（例如`host:port`）转换为特定端点的函数。提供多个端点的实例需要多个工厂函数。工厂函数还返回一个当实例消失并需要清理时调用的`io.Closer`。

```go
type Factory func(instance string) (endpoint.Endpoint, io.Closer, error)
```

例如，我们可以定义如下工厂函数，它会根据传入的实例地址创建一个`gRPC`客户端`endpoint`。

```go
func factory(instance string) (endpoint.Endpoint, io.Closer, error) {
	conn, err := grpc.Dial(instance, grpc.WithInsecure())
	if err != nil {
		return nil, nil, err
	}

	e := makeTrimEndpoint(conn)
	return e, conn, err
}
```

### Balancer

现在我们已经有了一组端点，我们需要从中选择一个。负载均衡器包装订阅者，并从多个端点中选择一个端点。

```go
type Balancer interface {
	Endpoint() (endpoint.Endpoint, error)
}
```

`Go Kit` 提供了一些基本的负载均衡器，如果你想要更高级的启发式方法，那么可以编写自己的负载均衡器。

下面的示例代码演示了如何使用`RoundRobin` 策略。

```go
import "github.com/go-kit/kit/sd/lb"

balancer := lb.NewRoundRobin(endpointer)
```

### 重试

重试策略包装负载均衡器，并返回可用的端点。重试策略将重试失败的请求，直到达到最大尝试或超时为止。

```go
func Retry(max int, timeout time.Duration, lb Balancer) endpoint.Endpoint
```

下面的示例代码演示了如何使用Go kit中的重试功能。

```go
import "github.com/go-kit/kit/sd/lb"

retry := lb.Retry(3, 500*time.Millisecond, balancer)
```

### 基于consul的服务发现完整示例

```go
// getTrimServiceFromConsul 基于consul的服务发现
func getTrimServiceFromConsul(consulAddr string, srvName string, tags []string, logger log.Logger) (endpoint.Endpoint, error) {
	consulConfig := api.DefaultConfig()
	consulConfig.Address = consulAddr

	consulClient, err := apiconsul.NewClient(consulConfig)
	if err != nil {
		return nil, err
	}

	sdClient := sdconsul.NewClient(consulClient)
	var passingOnly = true
	instancer := sdconsul.NewInstancer(sdClient, logger, srvName, tags, passingOnly)
	endpointer := sd.NewEndpointer(instancer, factory, logger)
	balancer := lb.NewRoundRobin(endpointer)
	retryMax := 3
	retryTimeout := 500 * time.Millisecond
	retry := lb.Retry(retryMax, retryTimeout, balancer)
	return retry, nil
}

func factory(instance string) (endpoint.Endpoint, io.Closer, error) {
	conn, err := grpc.Dial(instance, grpc.WithInsecure())
	if err != nil {
		return nil, nil, err
	}

	e := makeTrimEndpoint(conn)
	return e, conn, err
}
```

将上一部分的`main`函数中获取`trimEndpoint`的方式改为从`consul`获取。

```go
// main.go

// 基于 consul 对 trim_service 做服务发现
trimEndpoint, err := getTrimServiceFromConsul("localhost:8500", "trim_service", nil, logger)
if err != nil {
	fmt.Printf("connect %s failed, err: %v", *trimAddr, err)
	return
}
```

### trim服务

对 `trim` 服务做适当改造，以支持在程序启动后将当前服务实例注册到`consul`，以及程序退出时从`consul`注销服务。

```go
package main

import (
	"context"
	"flag"
	"fmt"
    apiconsul "github.com/hashicorp/consul/api"
	"google.golang.org/grpc"
	"net"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"trim_service/pb"	
)

const serviceName = "trim_service"

var (
	port       = flag.Int("port", 8975, "service port")
	consulAddr = flag.String("consul", "localhost:8500", "consul address")
)

// trim service

type server struct {
	pb.UnimplementedTrimServer
}

// TrimSpace 去除字符串参数中的空格
func (s *server) TrimSpace(_ context.Context, req *pb.TrimRequest) (*pb.TrimResponse, error) {
	ov := req.GetS()
	v := strings.ReplaceAll(ov, " ", "")
	fmt.Printf("ov:%s v:%v\n", ov, v)
	return &pb.TrimResponse{S: v}, nil
}

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		fmt.Printf("failed to listen: %v", err)
		return
	}
	s := grpc.NewServer()
	pb.RegisterTrimServer(s, &server{})

	// 服务注册
	cc, err := NewConsulClient(*consulAddr)
	if err != nil {
		fmt.Printf("failed to NewConsulClient: %v", err)
		return
	}
	ipInfo, err := getOutboundIP()
	if err != nil {
		fmt.Printf("getOutboundIP failed, err:%v\n", err)
		return
	}
	if err := cc.RegisterService(serviceName, ipInfo.String(), *port); err != nil {
		fmt.Printf("regToConsul failed, err:%v\n", err)
		return
	}
	go func() {
		if err := s.Serve(lis); err != nil {
			fmt.Printf("failed to serve: %v", err)
			return
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
	<-quit
	// 退出时注销服务
	cc.Deregister(fmt.Sprintf("%s-%s-%d", serviceName, ipInfo.String(), *port))
}

// consul reg&de
type consulClient struct {
	client *apiconsul.Client
}

// NewConsulClient 新建consulClient
func NewConsulClient(consulAddr string) (*consulClient, error) {
	cfg := apiconsul.DefaultConfig()
	cfg.Address = consulAddr
	client, err := apiconsul.NewClient(cfg)
	if err != nil {
		return nil, err
	}
	return &consulClient{client}, nil
}

// RegisterService 服务注册
func (c *consulClient) RegisterService(serviceName, ip string, port int) error {
	srv := &apiconsul.AgentServiceRegistration{
		ID:      fmt.Sprintf("%s-%s-%d", serviceName, ip, port), // 服务唯一ID
		Name:    serviceName,                                    // 服务名称
		Tags:    []string{"q1mi", "trim"},                       // 为服务打标签
		Address: ip,
		Port:    port,
	}
	return c.client.Agent().ServiceRegister(srv)
}

// Deregister 注销服务
func (c *consulClient) Deregister(serviceID string) error {
	return c.client.Agent().ServiceDeregister(serviceID)
}

// getOutboundIP 获取本机的出口IP
func getOutboundIP() (net.IP, error) {
	conn, err := net.Dial("udp", "8.8.8.8:80")
	if err != nil {
		return nil, err
	}
	defer conn.Close()
	localAddr := conn.LocalAddr().(*net.UDPAddr)
	return localAddr.IP, nil
}
```