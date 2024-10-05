# Go kit教程04——中间件和日志

`Endpoint`并返回`Endpoint`的函数，具体定义如下。

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
			logger.Log("msg", "calling endpoint")
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

由于我们的 `AddService` 服务定义为接口类型，所以我们只需要定义一个新类型（把原先的实现和一个额外的logger包装起来）并实现这个接口即可。

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

当然，go kit的中间件不仅仅是能用来实现日志中间件。社区里有很多的插件都是基于go kit的中间件实现的。例如，限流（ratelimit）、熔断器(circuitbreaker)、指标采集（metrics）等。

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

在 Go kit 中，检测意味着使用 `metrics`包来记录关于服务运行时行为的统计信息。计算已处理作业的数量、记录请求完成后的持续时间以及跟踪正在执行的操作的数量都将被视为检测工具。

我们可以使用与日志记录相同的中间件模式。

```go
import (
	"context"
	"fmt"
	"time"

	"github.com/go-kit/kit/metrics"
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