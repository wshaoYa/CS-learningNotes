# Go kit教程03——代码分层

`go`文件中，随着服务中`endpoint`数量的增加，将调用过程的每一层分隔到单独的文件中，可以提高 Go-kit 项目的可读性。

## Separation of concerns

遵循分离关注点（Separation of concerns）的设计理念，将完整的请求流程划分为`service`、`transport`和`endpoint`三层，每一层专注于实现特定的功能。

### service

`service`层负责我们业务逻辑的实现。

在项目下新建`service.go`文件，将与业务逻辑相关的代码保存至`service.go`文件中。

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

`endpoint`层负责存放我们项目中对外暴露的RPC方法。

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
	V   int    `json:"v"`
	Err string `json:"err,omitempty"`
}

type ConcatRequest struct {
	A string `json:"a"`
	B string `json:"b"`
}

type ConcatResponse struct {
	V   string `json:"v"`
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
	"net/http"

	grpctransport "github.com/go-kit/kit/transport/grpc"
	httptransport "github.com/go-kit/kit/transport/http"
	"github.com/gorilla/mux"

	"addsrv/pb"
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

// HTTP Server
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

修改后的`main.go`文件内容如下，该程序将同时对外提供HTTP API和gRPC API。

```go
// main.go

package main

import (
	"flag"
	"fmt"
	"net"
	"net/http"

	"golang.org/x/sync/errgroup"
	"google.golang.org/grpc"

	"addsrv/pb"
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
		defer httpListener.Close()
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
		defer grpcListener.Close()
		s := grpc.NewServer()
		pb.RegisterAddServer(s, NewGRPCServer(bs))
		return s.Serve(grpcListener)
	})

	if err := g.Wait(); err != nil {
		fmt.Printf("server exit with err:%v\n", err)
	}
}
```