# gRPC-Gateway使用指南

[gRPC-Gateway](https://github.com/grpc-ecosystem/grpc-gateway) 是一个 protoc 插件。它读取 gRPC 服务定义并生成一个反向代理服务器，该服务器将 RESTful JSON API 转换为 gRPC。此服务器根据 gRPC 定义中的自定义选项生成。

## gRPC-Gateway介绍

[gRPC-Gateway](https://github.com/grpc-ecosystem/grpc-gateway) 是一个 protoc 插件。它读取 gRPC 服务定义并生成一个反向代理服务器，该服务器将 RESTful JSON API 转换为 gRPC。此服务器根据 gRPC 定义中的自定义选项生成。

鉴于复杂的外部环境 gRPC 并不是万能的工具。在某些情况下，我们仍然希望提供传统的 HTTP/JSON API，来满足维护向后兼容性或者那些不支持 gRPC 的客户端。但是为我们的RPC服务再编写另一个服务只是为了对外提供一个 HTTP/JSON API，这是一项相当耗时和乏味的任务。

GRPC-Gateway 能帮助你同时提供 gRPC 和 RESTful 风格的 API。GRPC-Gateway 是 Google protocol buffers 编译器 protoc 的一个插件。它读取 Protobuf 服务定义并生成一个反向代理服务器，该服务器将 RESTful HTTP API 转换为 gRPC。该服务器是根据服务定义中的 google.api.http 注释生成的。

![gRPC-Gateway](https://www.liwenzhou.com/images/Go/grpc_gateway/architecture.svg)

## 基本使用示例

### 使用protobuf定义 gRPC 服务

新建一个项目`greeter`，在项目目录下执行`go mod init`命令完成go module初始化。

在项目目录下创建一个`proto/helloworld/hello_world.proto`文件，其内容如下。

```protobuf
syntax = "proto3";

package helloworld;

option go_package="github.com/Q1mi/greeter/proto/helloworld";

// 定义一个Greeter服务
service Greeter {
  // 打招呼方法
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// 定义请求的message
message HelloRequest {
  string name = 1;
}

// 定义响应的message
message HelloReply {
  string message = 1;
}
```

### 生成代码

```bash
$ protoc -I=proto \
   --go_out=proto --go_opt=paths=source_relative \
   --go-grpc_out=proto --go-grpc_opt=paths=source_relative \
   helloworld/hello_world.proto
```

*Windows执行失败的话就去掉上述命令中的`\`。*

生成pb和gRPC相关代码后，在`main`函数中注册RPC服务并启动gRPC Server。

```go
// greeter/main.go

package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"

	helloworldpb "github.com/Q1mi/greeter/proto/helloworld"
)

type server struct {
	helloworldpb.UnimplementedGreeterServer
}

func NewServer() *server {
	return &server{}
}

func (s *server) SayHello(ctx context.Context, in *helloworldpb.HelloRequest) (*helloworldpb.HelloReply, error) {
	return &helloworldpb.HelloReply{Message: in.Name + " world"}, nil
}

func main() {
	// Create a listener on TCP port
	lis, err := net.Listen("tcp", ":8080")
	if err != nil {
		log.Fatalln("Failed to listen:", err)
	}

	// 创建一个gRPC server对象
	s := grpc.NewServer()
	// 注册Greeter service到server
	helloworldpb.RegisterGreeterServer(s, &server{})
	// 启动gRPC Server
	log.Println("Serving gRPC on 0.0.0.0:8080")
	log.Fatal(s.Serve(lis))
}
```

此时的文件目录如下：

```bash
greeter
├── go.mod
├── go.sum
├── main.go
└── proto
    └── helloworld
        ├── hello_world.pb.go
        ├── hello_world.proto
        └── hello_world_grpc.pb.go
```

至此一个简单的GRPC服务就写好了。

接下来我们将介绍如何快速的为GRPC服务生成HTTP API代码。

### 将 gRPC-Gateway 注释添加到现有的proto文件

现在我们已经有了一个可以运行的 Go gRPC 服务器，接下来需要添加 gRPC-Gateway 注释。这些注释定义了 gRPC 服务如何映射到 JSON 请求和响应。使用 protocol buffers时，每个 RPC 服务必须使用 `google.api.HTTP` 注释来定义 HTTP 方法和路径。

因此，我们需要将 `google/api/http.proto` 导入到 `proto` 文件中。我们还需要添加所需的 HTTP-> gRPC 映射。在本例中，我们将 `POST /v1/example/echo` 映射到 `SayHello` RPC。

修改后的`proto/helloworld/hello_world.proto`文件，内容如下。

```protobuf
syntax = "proto3";

package helloworld;

option go_package="github.com/Q1mi/greeter/proto/helloworld";

// 导入google/api/annotations.proto
import "google/api/annotations.proto";

// 定义一个Greeter服务
service Greeter {
  // 打招呼方法
  rpc SayHello (HelloRequest) returns (HelloReply) {
    // 这里添加了google.api.http注释
    option (google.api.http) = {
      post: "/v1/example/echo"
      body: "*"
    };
  }
}

// 定义请求的message
message HelloRequest {
  string name = 1;
}

// 定义响应的message
message HelloReply {
  string message = 1;
}
```

### 生成gRPC-Gateway stubs

现在我们已经将 gRPC-Gateway 注释添加到 proto 文件中，接下来需要使用 gRPC-Gateway 生成器来生成存根。

#### 引入依赖包

在我们可以使用 `protoc` 生成存根之前，我们需要将一些依赖项复制到我们的 proto 文件结构中。将 `googleapis` 的一个子集从[官方库](https://github.com/googleapis/googleapis)复制到您的本地原型文件结构中。拷贝后的目录应该是这样的:

```bash
greeter
├── go.mod
├── go.sum
├── main.go
└── proto
    ├── google
    │   └── api
    │       ├── annotations.proto
    │       └── http.proto
    └── helloworld
        ├── hello_world.pb.go
        ├── hello_world.proto
        └── hello_world_grpc.pb.go
```

#### 安装gRPC-Gateway工具

需要安装`protoc-gen-grpc-gateway`插件来生成对应的 grpc-gateway 代码。

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@v2
```

如果不安装该插件，就无法生成grpc-gateway相关的代码。报错如下：

```bash
protoc-gen-grpc-gateway: program not found or is not executable
Please specify a program using absolute path or make sure the program is available in your PATH system variable
--grpc-gateway_out: protoc-gen-grpc-gateway: Plugin failed with status code 1.
```

#### 生成代码

现在我们需要将 gRPC-Gateway 生成器添加到 protoc 的调用命令中:

```bash
$ protoc -I=proto \
   --go_out=proto --go_opt=paths=source_relative \
   --go-grpc_out=proto --go-grpc_opt=paths=source_relative \
   --grpc-gateway_out=proto --grpc-gateway_opt=paths=source_relative \
   helloworld/hello_world.proto
```

执行上述命令应该会生成一个 `*.gw.pb.go` 文件。

### 添加HTTP Server代码

我们还需要在 `main.go` 文件中添加和启动`gRPC-Gateway mux`。按如下代码所示修改我们的`main`函数。

```go
package main

import (
	"context"
	"log"
	"net"
	"net/http"

	helloworldpb "github.com/Q1mi/greeter/proto/helloworld"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime" // 注意v2版本
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type server struct {
	helloworldpb.UnimplementedGreeterServer
}

func NewServer() *server {
	return &server{}
}

func (s *server) SayHello(ctx context.Context, in *helloworldpb.HelloRequest) (*helloworldpb.HelloReply, error) {
	return &helloworldpb.HelloReply{Message: in.Name + " world"}, nil
}

func main() {
	// Create a listener on TCP port
	lis, err := net.Listen("tcp", ":8080")
	if err != nil {
		log.Fatalln("Failed to listen:", err)
	}

	// 创建一个gRPC server对象
	s := grpc.NewServer()
	// 注册Greeter service到server
	helloworldpb.RegisterGreeterServer(s, &server{})
	// 8080端口启动gRPC Server
	log.Println("Serving gRPC on 0.0.0.0:8080")
	go func() {
		log.Fatalln(s.Serve(lis))
	}()

	// 创建一个连接到我们刚刚启动的 gRPC 服务器的客户端连接
	// gRPC-Gateway 就是通过它来代理请求（将HTTP请求转为RPC请求）
	conn, err := grpc.DialContext(
		context.Background(),
		"0.0.0.0:8080",
		grpc.WithBlock(),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		log.Fatalln("Failed to dial server:", err)
	}

	gwmux := runtime.NewServeMux()
	// 注册Greeter
	err = helloworldpb.RegisterGreeterHandler(context.Background(), gwmux, conn)
	if err != nil {
		log.Fatalln("Failed to register gateway:", err)
	}

	gwServer := &http.Server{
		Addr:    ":8090",
		Handler: gwmux,
	}
	// 8090端口提供gRPC-Gateway服务
	log.Println("Serving gRPC-Gateway on http://0.0.0.0:8090")
	log.Fatalln(gwServer.ListenAndServe())
}
```

**注意**

1. 导入的"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"是v2版本。
2. 需要使用单独的goroutine启动gRPC服务。
3. 目前DialContext函数已废除，新采用NewClient
4. 如果gRPC服务端采用了证书加密，gateway转为http时也得使用，否则连不上！

### 测试gRPC-Gateway

首先启动服务。

```bash
go run main.go
```

然后我们使用 cURL 发送 HTTP 请求:

```bash
curl -X POST -k http://localhost:8090/v1/example/echo -d '{"name": " hello"}'
```

得到响应结果。

```bash
{"message":"hello world"}
```

至此 gRPC-Gateway 的基础使用教程就结束啦，完整的示例代码可查看https://github.com/Q1mi/greeter。

### 同一个端口提供HTTP API和gRPC API

上面的程序在`8080`端口提供了gRPC API，在`8090`端口提供了HTTP API。但是在有些场景下我们可能希望由同一个端口同时提供gRPC API和HTTP API两种服务，由请求方来决定具体使用哪个协议。

下面的代码将同时在本机的`8091`端口对外提供gRPC API和HTTP API服务。

因为我们的示例中没有启用 TLS加密通信，所以这里使用`h2c`包实现对HTTP/2的支持。h2c 协议是 HTTP/2的非 TLS 版本。

```go
package main

import (
	"context"
	"log"
	"net"
	"net/http"
	"strings"

	helloworldpb "github.com/Q1mi/greeter/proto/helloworld"
	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime" // 注意v2版本
	"golang.org/x/net/http2"
	"golang.org/x/net/http2/h2c"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type server struct {
	helloworldpb.UnimplementedGreeterServer
}

func NewServer() *server {
	return &server{}
}

func (s *server) SayHello(ctx context.Context, in *helloworldpb.HelloRequest) (*helloworldpb.HelloReply, error) {
	return &helloworldpb.HelloReply{Message: in.Name + " world"}, nil
}

func main() {
	// Create a listener on TCP port
	lis, err := net.Listen("tcp", ":8091")
	if err != nil {
		log.Fatalln("Failed to listen:", err)
	}

	// 创建一个gRPC server对象
	s := grpc.NewServer()
	// 注册Greeter service到server
	helloworldpb.RegisterGreeterServer(s, &server{})

	// gRPC-Gateway mux
	gwmux := runtime.NewServeMux()
	dops := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
	err = helloworldpb.RegisterGreeterHandlerFromEndpoint(context.Background(), gwmux, "127.0.0.1:8091", dops)
	if err != nil {
		log.Fatalln("Failed to register gwmux:", err)
	}

	mux := http.NewServeMux()
	mux.Handle("/", gwmux)

	// 定义HTTP server配置
	gwServer := &http.Server{
		Addr:    "127.0.0.1:8091",
		Handler: grpcHandlerFunc(s, mux), // 请求的统一入口
	}
	log.Println("Serving on http://127.0.0.1:8091")
	log.Fatalln(gwServer.Serve(lis)) // 启动HTTP服务
}

// grpcHandlerFunc 将gRPC请求和HTTP请求分别调用不同的handler处理
func grpcHandlerFunc(grpcServer *grpc.Server, otherHandler http.Handler) http.Handler {
	return h2c.NewHandler(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.ProtoMajor == 2 && strings.Contains(r.Header.Get("Content-Type"), "application/grpc") {
			grpcServer.ServeHTTP(w, r)
		} else {
			otherHandler.ServeHTTP(w, r)
		}
	}), &http2.Server{})
}
```

将上述代码编译后运行，在`8091`端口启动。 测试gRPC API：

```bash
./greeter_client -name="hello"
```

得到响应结果。

```bash
resp:hello world
```

测试HTTP API：

```bash
curl -X POST -k http://127.0.0.1:8091/v1/example/echo -d '{"name": " hello"}'
```

得到响应结果。

```bash
{"message":"hello world"}
```