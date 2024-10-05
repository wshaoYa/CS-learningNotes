# Go kit教程05——调用其他服务

一个字符串参数，将其中所有的空格去掉后再返回。

其中`trim.proto`内容如下：

```proto
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

Go-kit提供了中间件来帮助解决项目中需要调用其他的服务。

由于我们的项目中需要调用`trim_service`服务，我们这里定义一个 `withTrimMiddleware`结构体，然后就把它当作一个`ServiceMiddleware`去实现，就像之前定义的`logMiddleware`那样。

```go
type withTrimMiddleware struct {
	next        AddService
	trimService endpoint.Endpoint // 通过它调用其他的服务
}
```

### 客户端endpoint

#### endpoint

endpoint层定义相关请求参数和响应参数。

```go
type trimRequest struct {
	s string
}

type trimResponse struct {
	s string
}
```

使用`grpctransport.NewClient` 创建基于gRPC client的endpoint。

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

此处的endpoint与我们之前定义的endpoint有所不同，它属于客户端endpoint。之前的endpoint是我们的程序直接服务（serve）的，而`TrimEndpoint`是用来调用外部请求（invoke）的。

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

copy

返回结果

```bash
1012
```