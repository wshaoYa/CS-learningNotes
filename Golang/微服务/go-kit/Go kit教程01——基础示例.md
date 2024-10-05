# Go kit教程01——基础示例

了解业务领域知识的东西。

Go kit 提供端点和传输域中间件，用于诸如速率限制、断路、负载平衡和分布式跟踪等功能ーー所有这些功能通常与你的业务域无关。

简而言之，Go kit 试图通过精心使用中间件（或装饰器）模式来强制执行严格的关注分离（**separation of concerns**）。

## 快速开始

接下来就演示如何使用 Go kit 快速实现一个微服务。

在本机上新建项目目录`addsrv1`，并在项目目录下执行`go mod init addsrv`完成项目的初始化。

### 业务逻辑

服务从业务逻辑开始写起。在 Go kit 中，我们将服务建模为一个接口。

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
	// ErrTwoZeroes  Sum方法的业务规则不能对两个0求和
	ErrTwoZeroes = errors.New("can't sum two zeroes")

	// ErrIntOverflow Sum参数越界
	ErrIntOverflow = errors.New("integer overflow")

	// ErrTwoEmptyStrings Concat方法业务规则规定参数不能是两个空字符串.
	ErrTwoEmptyStrings = errors.New("can't concat two empty strings")

	// ErrMaxSizeExceeded Concat方法的参数超出范围
	ErrMaxSizeExceeded = errors.New("result exceeds maximum size")
)

// Sum 对两个数字求和，实现AddService。
func (s addService) Sum(_ context.Context, a, b int) (int, error) {
	if a == 0 && b == 0 {
		return 0, ErrTwoZeroes
	}
	if (b > 0 && a > (math.MaxInt-b)) || (b < 0 && a < (math.MinInt-b)) {
		return 0, ErrIntOverflow
	}
	return a + b, nil
}

// Concat 连接两个字符串，实现AddService。
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

在 Go kit 中，主要的消息模式是 RPC。因此，我们接口中的每个方法都将被建模为一个远程过程调用。对于每个方法，我们定义**请求和响应**结构体，分别捕获所有的输入和输出参数。

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

Go kit 通过一个称为**endpoint**的抽象提供了许多功能。Endpoint`的定义如下：

```go
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```

它表示单个 RPC。也就是说，我们的服务接口中只有一个方法。我们将编写简单的适配器来将服务的每个方法转换为一个端点。每个适配器接受一个 AddService，并返回与其中一个方法对应的端点。

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

现在我们需要将编写的服务公开给外部世界，这样就可以调用它了。Go kit 开箱即用的支持 gRPC、Thrift或者基于HTTP的JSON。

这里我们先演示如何使用HTTP之上的JSON作为传输协议。

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

在项目目录下编译得到可执行文件并运行，服务会在本机的`8080`端口启动。我们使用curl或postman测试我们的服务。

```bash
❯ curl -XPOST -d'{"a":1,"b":2}' localhost:8080/sum
{"v":3}
❯ curl -XPOST -d'{"a":"你好","b":"qimi"}' localhost:8080/concat
{"v":"你好qimi"}
```