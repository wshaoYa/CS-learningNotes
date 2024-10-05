本文介绍了gRPC中名称解析和负载均衡的设计。

在进行下面的内容之前，我们先来看一下作为gRPC Server端的`hello_server`的主要内容。

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"net"

	"hello_server/pb"

	"google.golang.org/grpc"
)

// grpc server

var port = flag.Int("port", 8972, "服务端口")

type server struct {
	pb.UnimplementedGreeterServer
	Addr string
}

// SayHello 是我们需要实现的方法
// 这个方法是我们对外提供的服务
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloResponse, error) {
	reply := fmt.Sprintf("hello %s. [from %s]", in.GetName(), s.Addr)
	return &pb.HelloResponse{Reply: reply}, nil
}

func main() {
	flag.Parse()
	addr := fmt.Sprintf("127.0.0.1:%d", *port)
	// 启动服务
	l, err := net.Listen("tcp", addr)
	if err != nil {
		fmt.Printf("failed to listen, err:%v\n", err)
		return
	}

	s := grpc.NewServer() // 创建grpc服务
	// 注册服务
	pb.RegisterGreeterServer(s, &server{Addr: addr})
	// 启动服务
	err = s.Serve(l)
	if err != nil {
		fmt.Printf("failed to serve,err:%v\n", err)
		return
	}
}
```

将服务端代码编译成可执行文件——`hello_server`，并分别在本机的`8972`和`8973`端口启动。

```bash
./hello_server -port=8972
./hello_server -port=8973
```

本文后续的内容都是编写一个`hello_client`并与上述`hello_server`进行RPC通信。

## name resolving

名称解析器（name resolver）可以看作是一个 `map[service-name][]backend-ip`。它接收一个服务名称，并返回后端的 IP 列表。gRPC中根据目标字符串中的`scheme`选择名称解析器。

### DNS解析器

gRPC中默认使用的名称解析器是 DNS，即在gRPC客户端执行`grpc.Dial`时提供域名，默认会将DNS解析出对应的IP列表返回。

使用默认DNS解析器的名称语法为：`dns:[//authority/]host[:port]`

```go
conn, err := grpc.Dial("dns:///localhost:8972",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

### consul resolver

社区里有对应不同注册中心的resolver，例如下面是使用 consul 作为注册中心的示例。其中使用了第三方的[grpc-consul-resolver](https://github.com/mbobakov/grpc-consul-resolver)库作为consul resolver。

```go
package main

import _ "github.com/mbobakov/grpc-consul-resolver"

// ...

conn, err := grpc.Dial(
		// consul服务
		"consul://192.168.1.11:8500/hello?wait=14s",
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
```

### 自定义解析器

除了使用内置和社区提供的名称解析器，我们还可以自定义一套自己的名称解析器。接下来我们就定义一个`q1miResolver`。

```go
import (
	"google.golang.org/grpc/resolver"
)

// 自定义name resolver

const (
	myScheme   = "q1mi"
	myEndpoint = "resolver.liwenzhou.com"
)

var addrs = []string{"127.0.0.1:8972", "127.0.0.1:8973"}

// q1miResolver 自定义name resolver，实现Resolver接口
type q1miResolver struct {
	target     resolver.Target
	cc         resolver.ClientConn
	addrsStore map[string][]string
}

func (r *q1miResolver) ResolveNow(o resolver.ResolveNowOptions) {
	addrStrs := r.addrsStore[r.target.Endpoint]
	addrList := make([]resolver.Address, len(addrStrs))
	for i, s := range addrStrs {
		addrList[i] = resolver.Address{Addr: s}
	}
	r.cc.UpdateState(resolver.State{Addresses: addrList})
}

func (*q1miResolver) Close() {}

// q1miResolverBuilder 需实现 Builder 接口
type q1miResolverBuilder struct{}

func (*q1miResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	r := &q1miResolver{
		target: target,
		cc:     cc,
		addrsStore: map[string][]string{
			myEndpoint: addrs,
		},
	}
	r.ResolveNow(resolver.ResolveNowOptions{})
	return r, nil
}
func (*q1miResolverBuilder) Scheme() string { return myScheme }

func init() {
	// 注册 q1miResolverBuilder
	resolver.Register(&q1miResolverBuilder{})
}
```

在gRPC客户端按如下方式发起连接。

```go
conn, err := grpc.Dial(
	"q1mi:///resolver.liwenzhou.com",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

`grpc.Dial`函数中会先根据`q1mi`这个`scheme`找到我们通过`init`函数注册的`q1miResolverBuilder`，然后调用它的`Build()`方法构建我们自定义的`q1miResolver`，并调用`ResolveNow()`方法获取到服务端地址。

也可以在客户端建立连接时通过`grpc.WithResolvers`指定使用的名称解析器，使用这种方法就不需要事先注册名称解析器了。

```go
conn, err := grpc.Dial(
	"q1mi:///resolver.liwenzhou.com",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithResolvers(&q1miResolverBuilder{}), // 指定使用q1miResolverBuilder
)
```

## 负载均衡策略

gRPC 中的负载均衡是基于每次调用的，而不是基于每个连接的。换句话说，即使所有请求来自单个客户端，我们仍然希望它们在所有服务器之间负载均衡（雨露均沾）。

gRPC-go 内置支持有 `pick_first` (默认值)和 `round_robin` 两种策略。

- `pick_first`是 gRPC 负载均衡的默认值，因此不需要设置。`pick_first` 会尝试连接取到的第一个服务端地址，如果连接成功，则将其用于所有 RPC，如果连接失败，则尝试下一个地址(并继续这样做，直到一个连接成功)。因此，所有的 RPC 将被发送到同一个后端。所有接收到的响应都显示相同的后端地址。
- `round_robin` 连接到它所看到的所有地址，并按顺序一次向每个server发送一个 RPC。例如，我们现在注册有两个server，第一个 RPC 将被发送到 server-1，第二个 RPC 将被发送到 server-2，第三个 RPC 将再次被发送到 server-1。

此外gRPC还在逐步通过一些额外的 LB 策略来支持 `xDS`。

grpc客户端通过`grpc.WithDefaultServiceConfig`来配置要使用的负载均衡策略。

```go
conn, err := grpc.Dial(
	"q1mi:///resolver.liwenzhou.com",
	grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`), // 这里设置初始策略
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

再建立连接后，发起10次`SayHello`调用。

```go
c := pb.NewGreeterClient(conn)
// 调用RPC方法
for i := 0; i < 10; i++ {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	resp, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
	if err != nil {
		fmt.Printf("c.SayHello failed, err:%v\n", err)
		return
	}
	// 拿到了RPC响应
	fmt.Printf("resp:%v\n", resp.GetReply())
}
```

从终端输出的结果会发现客户端发出的RPC请求依次由`8972`和`8973`端口的服务器处理。

```bash
./hello_client
resp:hello 七米 [from 127.0.0.1:8973]
resp:hello 七米 [from 127.0.0.1:8972]
resp:hello 七米 [from 127.0.0.1:8973]
resp:hello 七米 [from 127.0.0.1:8972]
resp:hello 七米 [from 127.0.0.1:8973]
resp:hello 七米 [from 127.0.0.1:8972]
resp:hello 七米 [from 127.0.0.1:8973]
resp:hello 七米 [from 127.0.0.1:8972]
resp:hello 七米 [from 127.0.0.1:8973]
resp:hello 七米 [from 127.0.0.1:8972]
```

## 参考链接

- https://github.com/grpc/grpc/blob/master/doc/naming.md
- https://github.com/grpc/grpc/blob/master/doc/load-balancing.md