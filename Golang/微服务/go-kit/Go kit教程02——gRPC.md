# Go kit教程02——gRPC

本文主要介绍了如何使用 Go kit 构建基于 gRPC 的微服务，并额外补充了如何为 gRPC Server编写本地测试代码。

[上一篇](https://www.liwenzhou.com/posts/Go/go-kit-tutorial-01/)中，我们完成了一个基本的基于HTTP的 Go kit示例，本篇我们将添加代码使程序支持 gRPC 通信。

## 基于gRPC通信

想要实现基于 gRPC 的通信，首先需要定义好`proto`文件，并生成对应的 Go 代码和 gRPC 代码。

### 定义protobuf

根据`addsrv`业务的实际需要，我们定义的`proto`文件内容如下。

```protobuf
syntax = "proto3";

package pb;

option go_package="addsrv/pb";


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

执行下面的命令根据上述`proto`文件编译生成go代码（需事先安装好`protoc`和`protoc-gen-go-grpc`）。

```bash
protoc -I=pb \
   --go_out=pb --go_opt=paths=source_relative \
   --go-grpc_out=pb --go-grpc_opt=paths=source_relative \
   pb/addsrv.proto
```

此时项目目录如下：

```bash
├── go.mod
├── go.sum
├── main.go
└── pb
    ├── addsrv.pb.go
    ├── addsrv.proto
    └── addsrv_grpc.pb.go
```

> 到这里懵了？戳下面的链接先补课。
>
> - [protocol buffers使用指南](https://www.liwenzhou.com/posts/Go/protobuf/)
> - [gRPC教程](https://www.liwenzhou.com/posts/Go/gRPC/)

### grpcServer

在`main.go`中定义好`grpcServer`结构体，其内部包含`sum`和`concat`两个`grpctransport.Handler`。

```go
import grpctransport "github.com/go-kit/kit/transport/grpc"


type grpcServer struct {
	pb.UnimplementedAddServer
	sum    grpctransport.Handler
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

那么该如何得到`grpctransport.Handler`呢？与上一篇中获取`httptransport.Handler`类似。

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
	fmt.Printf("failed to listen: %v", err)
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

编写测试代码，验证`Sum`和`Concat`这两个RPC方法都工作正常。

```go
// add_test.go
package main

import (
	"context"
	"gokit_demo1/pb"
	"log"
	"net"
	"testing"

	"github.com/stretchr/testify/assert"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/test/bufconn"
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
			log.Fatalf("Server exited with error: %v", err)
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
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewAddClient(conn)

	resp, err := c.Sum(context.Background(), &pb.SumRequest{
		A: 10,
		B: 2,
	})
	assert.Nil(t, err)
	assert.NotNil(t, resp)
	assert.Equal(t, int64(12), resp.V)
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

	resp, err := c.Concat(context.Background(), &pb.ConcatRequest{
		A: "10",
		B: "2",
	})
	assert.Nil(t, err)
	assert.NotNil(t, resp)
	assert.Equal(t, "102", resp.V)
}
```

项目目录下执行下面的命令，并查看测试结果。

```bash
go test -v ./...
=== RUN   TestSum
--- PASS: TestSum (0.00s)
=== RUN   TestConcat
--- PASS: TestConcat (0.00s)
PASS
ok      addsrv     0.016s
```