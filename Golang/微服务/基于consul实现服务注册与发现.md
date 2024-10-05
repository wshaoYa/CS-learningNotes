# 基于consul实现服务注册与发现


而`datacenter-deploy-secure` 目录中提供了一个由3个server和1个client组成的集群环境。

#### 管理后台

使用浏览器打开 [http:127.0.0.1:8500](http://127.0.0.1:8500/) 就可以看到Consul的管理界面了。

![consul ui](https://www.liwenzhou.com/images/Go/consul/consul01.png)

## Agent HTTP API

请查阅完整的[Agent HTTP API文档](https://www.consul.io/api-docs/agent/service)。

### 服务注册与服务发现

这里只列出与服务注册和服务发现相关的API。

查询服务

| Method | Path              | Produces           |
| :----- | :---------------- | :----------------- |
| `GET`  | `/agent/services` | `application/json` |

注册服务

| Method | Path                      | Produces           |
| :----- | :------------------------ | :----------------- |
| `PUT`  | `/agent/service/register` | `application/json` |

注销服务

| Method | Path                                    | Produces           |
| :----- | :-------------------------------------- | :----------------- |
| `PUT`  | `/agent/service/deregister/:service_id` | `application/json` |

**注意：** 实际Path路径前需要加`v1`版本前缀。

## Go SDK

在代码中导入官方 api 包

```go
import "github.com/hashicorp/consul/api"
```

### 连接consul

定义一个`consul`结构体，保存consul client对象。

```go
// consul 定义一个consul结构体，其内部有一个`*api.Client`字段。
type consul struct {
	client *api.Client
}

// NewConsul 连接至consul服务返回一个consul对象
func NewConsul(addr string) (*consul, error) {
	cfg := api.DefaultConfig()
	cfg.Address = addr
	c, err := api.NewClient(cfg)
	if err != nil {
		return nil, err
	}
	return &consul{c}, nil
}
```

### 获取本机出口IP

这里补充一个获取本机出口IP的方法。

```go
// GetOutboundIP 获取本机的出口IP
func GetOutboundIP() (net.IP, error) {
	conn, err := net.Dial("udp", "8.8.8.8:80")
	if err != nil {
		return nil, err
	}
	defer conn.Close()
	localAddr := conn.LocalAddr().(*net.UDPAddr)
	return localAddr.IP, nil
}
```

### 服务注册

将我们 grpc 服务注册到 consul。

```go
// RegisterService 将gRPC服务注册到consul
func (c *consul) RegisterService(serviceName string, ip string, port int) error {
	srv := &api.AgentServiceRegistration{
		ID:      fmt.Sprintf("%s-%s-%d", serviceName, ip, port), // 服务唯一ID
		Name:    serviceName,                                    // 服务名称
		Tags:    []string{"q1mi", "hello"},                      // 为服务打标签
		Address: ip,
		Port:    port,
	}
	return c.client.Agent().ServiceRegister(srv)
}
```

### 健康检查

consul支持为服务配置相应的健康检查。

#### gRPC服务支持健康检查

gRPC 服务要支持健康检查需要先导入相关依赖包。

```go
import "google.golang.org/grpc/health"
import healthpb "google.golang.org/grpc/health/grpc_health_v1"
```

向gRPC服务注册健康检查服务。

```go
s := grpc.NewServer() // 创建gRPC服务器
healthcheck := health.NewServer()
healthpb.RegisterHealthServer(s, healthcheck)
```

#### consul添加健康检查

在向 consul 注册我们的 gRPC 服务时，可以指定健康检查的配置。

```go
// RegisterService 将gRPC服务注册到consul
func (c *consul) RegisterService(serviceName string, ip string, port int) error {
	// 健康检查
	check := &api.AgentServiceCheck{
		GRPC:                           fmt.Sprintf("%s:%d", ip, port), // 这里一定是外部可以访问的地址
		Timeout:                        "10s",  // 超时时间
		Interval:                       "10s",  // 运行检查的频率
		// 指定时间后自动注销不健康的服务节点
		// 最小超时时间为1分钟，收获不健康服务的进程每30秒运行一次，因此触发注销的时间可能略长于配置的超时时间。
		DeregisterCriticalServiceAfter: "1m",
	}
	srv := &api.AgentServiceRegistration{
		ID:      fmt.Sprintf("%s-%s-%d", serviceName, ip, port), // 服务唯一ID
		Name:    serviceName,                                    // 服务名称
		Tags:    []string{"q1mi", "hello"},                      // 为服务打标签
		Address: ip,
		Port:    port,
		Check:   check,
	}
	return c.client.Agent().ServiceRegister(srv)
}
```

### 服务发现

在服务端程序将服务注册到 consul 之后，客户端程序可以查询服务名称下的服务实例（机器），具体支持的filter策略可查看[文档](https://www.consul.io/api-docs/agent/service#filtering)。

```go
// 连接consul
cc, err := api.NewClient(api.DefaultConfig())
if err != nil {
	fmt.Printf("api.NewClient failed, err:%v\n", err)
	return
}
// 返回的是一个 map[string]*api.AgentService
// 其中key是服务ID，值是注册的服务信息
serviceMap, err := cc.Agent().ServicesWithFilter("Service==`hello`")
if err != nil {
	fmt.Printf("query service from consul failed, err:%v\n", err)
	return
}
// 选一个服务机（这里选最后一个）
var addr string
for k, v := range serviceMap {
	fmt.Printf("%s:%#v\n", k, v)
	addr = v.Address + ":" + strconv.Itoa(v.Port)
}

// 建立RPC连接
conn, err := grpc.Dial(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
	log.Fatalf("grpc.Dial failed,err:%v", err)
	return
}
defer conn.Close()
```

**注意**：这里只是演示如何通过 api 查询 consul ，实际的服务发现过程并不需要这么写。

#### consul resolver

gRPC支持自定义resolver，借助第三方的[grpc-consul-resolver](https://github.com/mbobakov/grpc-consul-resolver)库，我们可以更便捷的实现基于consul的服务发现。

在代码中按如下方式匿名导入`github.com/mbobakov/grpc-consul-resolver`包。

```go
import _ "github.com/mbobakov/grpc-consul-resolver"
```

在`grpc.Dial`时直接使用类似 `consul://[user:password@]127.0.0.127:8555/my-service?[healthy=]&[wait=]&[near=]&[insecure=]&[limit=]&[tag=]&[token=]`的连接字符串来指定连接目标。

目前支持的参数：

|        Name        |        格式        |                             介绍                             |
| :----------------: | :----------------: | :----------------------------------------------------------: |
|        tag         |       string       |                         根据标签筛选                         |
|      healthy       |     true/false     |         只返回通过所有健康检查的端点。默认值：false          |
|        wait        | time.ParseDuration | 监控变更的等待时间。在这个时间段内，端点将强制刷新。默认值：继承agent的配置 |
|      insecure      |     true/false     |          允许与consul进行不安全的通信。默认值：true          |
|        near        |       string       | 按响应持续时间对端点排序。可与“limit”参数有效结合。默认值："_agent" |
|       limit        |        int         |               限制服务的端点数。默认值：无限制               |
|      timeout       | time.ParseDuration |                 Http-client超时。默认值：60s                 |
|    max-backoff     | time.ParseDuration | 重新连接到consul的最大后退时间。重连从10ms开始，成倍增长，直到到max-backoff。默认值：1s |
|       token        |       string       |                         Consul token                         |
|         dc         |       string       |                     consul数据中心。可选                     |
|    allow-stale     |     true/false     | 允许agent返回过期读的结果 https://developer.hashicorp.com/consul/api-docs/features/consistency#stale |
| require-consistent |     true/false     |    强制读取完全一致。这比较昂贵，但可以防止执行过期读操作    |

例如下面的示例中，使用`consul://localhost:8500/hello?healthy=true`即可快速实现查询hello服务健康节点的服务发现过程。

```go
conn, err := grpc.Dial(
	// consul服务
	"consul://localhost:8500/hello?healthy=true",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
defer conn.Close()

c := pb.NewGreeterClient(conn)
resp, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
if err != nil {
	fmt.Printf("c.SayHello failed, err:%v\n", err)
	return
}
fmt.Printf("resp:%v\n", resp.GetReply())
```

### 注销服务

将服务从 consul 注销可以使用`ServiceDeregister`函数。

```go
// Deregister 注销服务
func (c *consul) Deregister(serviceID string) error {
	return c.client.Agent().ServiceDeregister(serviceID)
}
```

在现实场景中我们通常需要在某个server节点退出时将当前节点从 consul 注销。 这里搭配`chan os.Signal`实现程序退出前执行注销服务操作。

```go
lis, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
if err != nil {
	fmt.Printf("failed to listen: %v", err)
	return
}
s := grpc.NewServer() // 创建gRPC服务器
// 开启健康检查
healthcheck := health.NewServer()
healthpb.RegisterHealthServer(s, healthcheck)
pb.RegisterGreeterServer(s, &server{}) // 在gRPC服务端注册服务
// 注册服务
consul, err := NewConsul("127.0.0.1:8500")
if err != nil {
	log.Fatalf("NewConsul failed, err:%v\n", err)
}
ipObj, _ := GetOutboundIP()
ip := ipObj.String()
consul.RegisterService("hello", ip, port)

// 启动服务
go func() {
	err = s.Serve(lis)
	if err != nil {
		log.Printf("failed to serve: %v", err)
		return
	}
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
<-quit
// 退出时注销服务
serviceId := fmt.Sprintf("%s-%s-%d", "hello", ip, port)
consul.Deregister(serviceId)
```

### 负载均衡

gRPC中使用`grpc.WithDefaultServiceConfig`来指定使用的负载均衡策略。

> 网上很多介绍gRPC负载均衡的旧资料还在使用已经废弃且被 v1.46 移除的[WithBalancerName](https://github.com/grpc/grpc-go/blob/431ea809a7676e1da8d09c33ae0d31fcba85f1ff/dialoptions.go#L208)。

下面的示例演示了 gRPC 客户端使用 consul 作为注册中心，`round_robin`作为负载均衡策略建立 gRPC 连接的示例。

```go
package main

import _ "github.com/mbobakov/grpc-consul-resolver"

// ...

conn, err := grpc.Dial(
		// consul服务
		"consul://127.0.0.1:8500/hello?healthy=true",
		// 指定round_robin策略
		grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy": "round_robin"}`),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
```

### consul 服务注册与服务发现总结

### ![基于consul的服务注册与服务发现流程](https://www.liwenzhou.com/images/Go/consul/consul02.png)