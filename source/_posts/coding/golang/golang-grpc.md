---
title: GRPC入门和实践
date: 2019-08-28 23:15:43
tags: Grpc
category: 编程开发
---

# gPRC
首先，先阐述一个误区，很多人以为gRPC只能go语言使用，以为这个g代表的就是go，其实并不是，这个g应该理解成Google，这个rpc框架是Google出品，不过Go对这个框架的支持确实非常好，看一下官网的介绍：
>gRPC is a modern open source high performance RPC framework that can run in any environment. It can efficiently connect services in and across data centers with pluggable support for load balancing, tracing, health checking and authentication. It is also applicable in last mile of distributed computing to connect devices, mobile applications and browsers to backend services.

详细的介绍可以参考[官网](https://grpc.io)（grpc.io）,简单说，gRPC是一个开源的高性能rpc框架。

说到rpc，很多搞微服务的都喜欢用，特别是Java领域，rpc全称 Remote Procedure Call，翻译过来叫远程过程调用，这个翻译并不是特别好理解。

<!--more-->

举个例子，假设你写了一个算法，非常牛逼，你想把这个算法给别人用，你会咋办？

首先，得确定这个调用方在哪里？如果这个调用方都在一个项目里面，那我们只需要写个函数，告诉别人函数名字就行了:
```go
package lib

import "fmt"

func Run() {
	fmt.Println("something very NB")
}
```
但是现实是，这个调用方不是同一个项目的，代码不在一起，是其它项目需要用，咋办呢？

有人说，把代码copy给别人，比较low，而且有时候代码要保密。

有人说，使用http服务，写个接口出来，扔一个API文档，这个方案完全可以，但是不是今天的主角。

或许，我们也可以使用rpc通信。

## Golang RPC
不少语言都有自己的rpc框架，比如PHP有phprpc和yar，但是这些rpc框架局限在这个语言，无法做到跨语言之间的调用，而Go也是类似，Go标准库自带的rpc有好几种，默认采用Gob编码，只能在Go语言之间使用,还有一种jsonrpc，采用的是json编码，如果你需要跨语言的话，最好采用gRPC。

Go RPC的函数只有符合下面的条件才能被远程访问：
- 函数必须是导出的(首字母大写)
- 必须有两个参数，并且是导出类型或者内建类型
- 第二个参数必须是指针类型的
- 函数还要有一个返回值 error

下面看一个简单例子：
### 入参出参
我们首先单独定义了需要被远程调用的方法，以及方法的入参和出参，后面的服务端和客户端都会用到：
```go
package golang_rpc

import "log"

type Add struct {
}

func (a *Add) Plus(request Request, response *Response) error {
	response.Result = request.A + request.B
	log.Printf("Add...%d + %d", request.A, request.B)
	return nil
}

type Request struct {
	A int
	B int
}

type Response struct {
	Result int
}
```

### Server端
这里使用的http协议，其实还有一种tcp的用法，主要作用是注册rpc服务，开启服务。
```go
package main

import (
	. "gRPC/golang-rpc"
	"log"
	"net/http"
	"net/rpc"
)

func main() {
	add := new(Add)
	_ = rpc.Register(add)
	rpc.HandleHTTP()
	log.Println("rpc server started at port 8888")
	if err := http.ListenAndServe(":8888", nil); err != nil {
		panic(err)
	}
}
```

### Client端
客户端根据定义的入参结构体拼装好请求参数，调用rpc
```go
package main

import (
	. "gRPC/golang-rpc"
	"log"
	"net/rpc"
)

func main() {
	dial, err := rpc.DialHTTP("tcp", ":8888")
	if err != nil {
		panic(err)
	}
	args := Request{
		A: 1,
		B: 2,
	}
	var response = Response{}
	err = dial.Call("Add.Plus", args, &response)
	if err != nil {
		panic(err)
	}
	log.Printf("a = %d, b= %d, result = %d", args.A, args.B, response.Result)
}
```
这只是展示了Go rpc的一种用法，Go rpc的除了支持tcp之外，还可以使用json，也就是jsonrpc，其编码方式是使用json而不是默认的Gob。


## RPC vs HTTP
我所参与项目大部分都是基于http，很少使用rpc，原因之一就是因为http特别成熟，文本协议，简单易用，支持广泛，而且其它支持比如负载均衡，流量控制都非常好用。

本质上，这个2种通信方式都可以实现远程过程调用，也就说把数据从一个地方传输到另一个地方（经过处理再返回回来）。当然也有人说http也是rpc的一种实现形式，这些概念性的东西这里就不争论了。

但是rpc确实有一些优点，其中最主要的就是传输效率高，因为http是文本协议，而rpc数据协议往往是二进制。


## gRPC
gRPC相比于其它rpc语言，目前发展迅速，不仅仅支持多语言（Go、Java、Python、JS），目前也支持Web端，意味着可以在某种程度上替代http了。

先不过多介绍太多理论的东西，这里先结合实际代码来看，默认情况下，gRPC使用Protobuf作为 Interface Definition Language（IDL），所谓IDL就是接口定义语言，说的通俗点就是描述这个服务的结构包括请求参数和响应结果。

这里说到的Protobuf又是什么东西呢？

>Protobuf(Google Protocol Buffers)是Google提供一个具有高效的协议数据交换格式工具库(类似Json)，但相比于Json，Protobuf有更高的转化效率，时间效率和空间效率都是JSON的3-5倍。

下面，咱们先看一个demo，先写个helloWorld，gRPC的写法比起http服务确实复杂很多，我们不仅仅要写server端，还要写client端，而http服务的client端一般都有现成的工具（浏览器、curl），但gRPC的client必须是一对一定制化的，需根据IDL生成。

1. Go的运行环境咱就不说了，目前gRPC要求Go版本在1.6以上
2. 安装gRPC: go get -u google.golang.org/grpc
3. 安装Protobuf v3 compiler，我的Ubuntu系统是自带这个，如果不带的话可以使用apt安装，其它系统可以参考[github](https://github.com/protocolbuffers/Protobuf)
4. 安装go的Protobuf插件： go get -u github.com/golang/Protobuf/protoc-gen-go

这个IDL文件并不是Go的语法，只是Protobuf的描述语法，大概的意思相信大部分都能看懂，service 是用来定义服务，然后还定义了请求和响应的参数类型，详细的用法可以参考Protobuf的[官方文档](https://developers.google.com/protocol-buffers/docs/proto3#simple)。

项目的整理结构如下：
```
├── client
│   └── client.go
├── go.mod
├── go.sum
├── proto
│   ├── hello.pb.go
│   └── hello.proto
└── server.go
```
切换到终端，在proto目录下执行```protoc --go_out=plugins=grpc:. *.proto```命令生成一个pb.go文件，这是一个go语法的文件，里面的东西非常多，我们真正用到的就是这个。

下面完成server端的开发：
```go
package main

import (
	"context"
	"fmt"
	pb "gRPC/proto"
	"google.golang.org/grpc"
	"log"
	"net"
)

type HelloService struct{}

func (s *HelloService) Hello(ctx context.Context, r *pb.HelloRequest) (*pb.HelloResponse, error) {
	fmt.Println("new request...")
	return &pb.HelloResponse{Response: r.GetRequest() + " Server"}, nil
}

const PORT = "8080"

func main() {
	server := grpc.NewServer()
	pb.RegisterHelloServiceServer(server, &HelloService{})

	listen, err := net.Listen("tcp", ":"+PORT)
	if err != nil {
		log.Fatalf("net.Listen err: %v", err)
	}
	_ = server.Serve(listen)
}
```
server端的主要作用是实现服务定义的接口，然后把服务注册到rpc server里面，最后启动服务等待请求的到来，和http服务有点类似。

虽然服务启动了，但是这时候无法像像http一样使用浏览器或者其它工具去访问，我们必须使用特定的客户端来访问服务，下面是客户端的代码：
```go
package main

import (
	"context"
	pb "gRPC/proto"
	"google.golang.org/grpc"
	"log"
)

const PORT = "8080"

func main() {
	conn, err := grpc.Dial(":"+PORT, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("grpc.Dial err: %v", err)
	}
	defer conn.Close()

	client := pb.NewHelloServiceClient(conn)
	resp, err := client.Hello(context.Background(), &pb.HelloRequest{
		Request: "Hello gRPC",
	})
	if err != nil {
		log.Fatalf("client.Search err: %v", err)
	}

	log.Printf("resp: %s", resp.GetResponse())
}
```

最后，先启动server，然后运行client。

有人可能会说，废了这么大劲，到最后结果和http服务有啥区别？我使用http服务分分钟钟搞定的事情，gRPC还需要定义这个那个...但是gRPC的功能不止这些。

## 流式请求
上面的demo只是一个simple模型，类似于http的request和response模型，但是gRPC还支持流式请求，其交互模型包括：
1. 服务端流。客户端发出一个请求，服务端返回一个响应流
2. 客户端流。客户端发出一个请求流，服务端返回一个响应
3. 双向流。客户端和服务端可以互相通信，类似websocket一样

具体的应用场景可以结合业务需求来定，这里demo就不展示了，官方有非常详细的example，其实大部分时候还是使用simple模型比较多。

## 应用场景
目前gRPC已经支持移动端和Web，如果拿来替代http也可行，但是http很容易调试和测试，而gRPC则很难，而且http的通用性更广泛，如果是对外提供的公开API，非http莫属。

目前来说gPRC比较适合用在一些对性能要求高而且比较稳定的场景，比如项目内部微服务之间的通信，这也是大多数rpc框架的主要应用场景。


