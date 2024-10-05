# Go操作Kafka之kafka-go

Kafka是一种高吞吐量的分布式发布订阅消息系统，本文介绍了如何使用kafka-go这个库实现Go语言与kafka的交互。

Go社区中目前有三个比较常用的kafka客户端库 , 它们各有特点。

首先是[IBM/sarama](https://github.com/IBM/sarama)（这个库已经由Shopify转给了IBM），之前我写过一篇使用[sarama操作Kafka](https://www.liwenzhou.com/posts/Go/kafka)的教程，相较于sarama， [kafka-go](https://github.com/segmentio/kafka-go) 更简单、更易用。

[segmentio/kafka-go](https://github.com/segmentio/kafka-go) 是纯Go实现，提供了与kafka交互的低级别和高级别两套API，同时也支持Context。

此外社区中另一个比较常用的[confluentinc/confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)，它是一个基于cgo的[librdkafka](https://github.com/edenhill/librdkafka)包装，在项目中使用它会引入对C库的依赖。

## 准备Kafka环境

这里推荐使用Docker Compose快速搭建一套本地开发环境。

以下`docker-compose.yml`文件用来搭建一套单节点zookeeper和单节点kafka环境，并且在`8080`端口提供`kafka-ui`管理界面。

```yaml
version: '2.1'

services:
  zoo1:
    image: confluentinc/cp-zookeeper:7.3.2
    hostname: zoo1
    container_name: zoo1
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_SERVERS: zoo1:2888:3888

  kafka1:
    image: confluentinc/cp-kafka:7.3.2
    hostname: kafka1
    container_name: kafka1
    ports:
      - "9092:9092"
      - "29092:29092"
      - "9999:9999"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka1:19092,EXTERNAL://${DOCKER_HOST_IP:-127.0.0.1}:9092,DOCKER://host.docker.internal:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,DOCKER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
      KAFKA_ZOOKEEPER_CONNECT: "zoo1:2181"
      KAFKA_BROKER_ID: 1
      KAFKA_LOG4J_LOGGERS: "kafka.controller=INFO,kafka.producer.async.DefaultEventHandler=INFO,state.change.logger=INFO"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_JMX_PORT: 9999
      KAFKA_JMX_HOSTNAME: ${DOCKER_HOST_IP:-127.0.0.1}
      KAFKA_AUTHORIZER_CLASS_NAME: kafka.security.authorizer.AclAuthorizer
      KAFKA_ALLOW_EVERYONE_IF_NO_ACL_FOUND: "true"
    depends_on:
      - zoo1
  kafka-ui:
    container_name: kafka-ui
    image: provectuslabs/kafka-ui:latest
    ports:
      - 8080:8080
    depends_on:
      - kafka1
    environment:
      DYNAMIC_CONFIG_ENABLED: "TRUE"
```

> 参考资料
>
> - [conduktor/kafka-stack-docker-compose](https://github.com/conduktor/kafka-stack-docker-compose)
> - [provectus/kafka-ui](https://github.com/provectus/kafka-ui)

将上述`docker-compose.yml`文件在本地保存，在同一目录下执行以下命令启动容器。

```bash
docker-compose up -d
```

容器启动后，使用浏览器打开[127.0.0.1:8080](http://127.0.0.1:8080/) 即可看到如下`kafka-ui`界面。

![kafka-ui](https://www.liwenzhou.com/images/Go/kafka-go/kafka-ui_01.png)

点击页面右侧的“Configure new cluster”按钮，配置kafka服务连接信息。

![kafka-ui](https://www.liwenzhou.com/images/Go/kafka-go/kafka-ui_02.png)

填写完信息后，点击页面下方的“Submit”按钮提交即可。

![kafka-ui](https://www.liwenzhou.com/images/Go/kafka-go/kafka-ui_03.png)

## 安装kafka-go

执行以下命令下载 `kafka-go`依赖。

```bash
go get github.com/segmentio/kafka-go
```

> 注意：kafka-go 需要 Go 1.15或更高版本。

## kafka-go使用指南

`kafka-go` 提供了两套与Kafka交互的API。

- 低级别（ low-level）：基于与 Kafka 服务器的原始网络连接实现。
- 高级别（high-level）：对于常用读写操作封装了一套更易用的API。

通常建议直接使用高级别的交互API。

### Connection

[Conn](https://pkg.go.dev/github.com/segmentio/kafka-go#Conn) 类型是 `kafka-go` 包的核心。它代表与 Kafka broker之间的连接。基于它实现了一套与Kafka交互的低级别 API。

#### 发送消息

下面是连接至Kafka之后，使用Conn发送消息的代码示例。

```go
// writeByConn 基于Conn发送消息
func writeByConn() {
	topic := "my-topic"
	partition := 0

	// 连接至Kafka集群的Leader节点
	conn, err := kafka.DialLeader(context.Background(), "tcp", "localhost:9092", topic, partition)
	if err != nil {
		log.Fatal("failed to dial leader:", err)
	}

	// 设置发送消息的超时时间
	conn.SetWriteDeadline(time.Now().Add(10 * time.Second))

	// 发送消息
	_, err = conn.WriteMessages(
		kafka.Message{Value: []byte("one!")},
		kafka.Message{Value: []byte("two!")},
		kafka.Message{Value: []byte("three!")},
	)
	if err != nil {
		log.Fatal("failed to write messages:", err)
	}

	// 关闭连接
	if err := conn.Close(); err != nil {
		log.Fatal("failed to close writer:", err)
	}
}
```

#### 消费消息

```go
// readByConn 连接至kafka后接收消息
func readByConn() {
	// 指定要连接的topic和partition
	topic := "my-topic"
	partition := 0

	// 连接至Kafka的leader节点
	conn, err := kafka.DialLeader(context.Background(), "tcp", "localhost:9092", topic, partition)
	if err != nil {
		log.Fatal("failed to dial leader:", err)
	}

	// 设置读取超时时间
	conn.SetReadDeadline(time.Now().Add(10 * time.Second))
	// 读取一批消息，得到的batch是一系列消息的迭代器
	batch := conn.ReadBatch(10e3, 1e6) // fetch 10KB min, 1MB max

	// 遍历读取消息
	b := make([]byte, 10e3) // 10KB max per message
	for {
		n, err := batch.Read(b)
		if err != nil {
			break
		}
		fmt.Println(string(b[:n]))
	}

	// 关闭batch
	if err := batch.Close(); err != nil {
		log.Fatal("failed to close batch:", err)
	}

	// 关闭连接
	if err := conn.Close(); err != nil {
		log.Fatal("failed to close connection:", err)
	}
}
```

使用`batch.Read`更高效一些，但是需要根据消息长度选择合适的buffer（上述代码中的b），如果传入的buffer太小（消息装不下）就会返回`io.ErrShortBuffer`错误。

如果不考虑内存分配的效率问题，也可以按以下代码使用`batch.ReadMessage`读取消息。

```go
for {
  msg, err := batch.ReadMessage()
  if err != nil {
    break
  }
  fmt.Println(string(msg.Value))
}
```

#### 创建topic

当Kafka关闭自动创建topic的设置时，可按如下方式创建topic。

```go
// createTopicByConn 创建topic
func createTopicByConn() {
	// 指定要创建的topic名称
	topic := "my-topic"

	// 连接至任意kafka节点
	conn, err := kafka.Dial("tcp", "localhost:9092")
	if err != nil {
		panic(err.Error())
	}
	defer conn.Close()

	// 获取当前控制节点信息
	controller, err := conn.Controller()
	if err != nil {
		panic(err.Error())
	}
	var controllerConn *kafka.Conn
	// 连接至leader节点
	controllerConn, err = kafka.Dial("tcp", net.JoinHostPort(controller.Host, strconv.Itoa(controller.Port)))
	if err != nil {
		panic(err.Error())
	}
	defer controllerConn.Close()

	topicConfigs := []kafka.TopicConfig{
		{
			Topic:             topic,
			NumPartitions:     1,
			ReplicationFactor: 1,
		},
	}

	// 创建topic
	err = controllerConn.CreateTopics(topicConfigs...)
	if err != nil {
		panic(err.Error())
	}
}
```

#### 通过非leader节点连接leader节点

下面的示例代码演示了如何通过已有的非leader节点的Conn，连接至 leader节点。

```go
conn, err := kafka.Dial("tcp", "localhost:9092")
if err != nil {
    panic(err.Error())
}
defer conn.Close()
// 获取当前控制节点信息
controller, err := conn.Controller()
if err != nil {
    panic(err.Error())
}
var connLeader *kafka.Conn
connLeader, err = kafka.Dial("tcp", net.JoinHostPort(controller.Host, strconv.Itoa(controller.Port)))
if err != nil {
    panic(err.Error())
}
defer connLeader.Close()
```

#### 获取topic列表

```go
conn, err := kafka.Dial("tcp", "localhost:9092")
if err != nil {
    panic(err.Error())
}
defer conn.Close()

partitions, err := conn.ReadPartitions()
if err != nil {
    panic(err.Error())
}

m := map[string]struct{}{}
// 遍历所有分区取topic
for _, p := range partitions {
    m[p.Topic] = struct{}{}
}
for k := range m {
    fmt.Println(k)
}
```

### Reader

`Reader`是由 `kafka-go` 包提供的另一个概念，对于从单个主题-分区（topic-partition）消费消息这种典型场景，使用它能够简化代码。Reader 还实现了自动重连和偏移量管理，并支持使用 Context 支持异步取消和超时的 API。

**注意：** 当进程退出时，必须在 `Reader` 上调用 `Close()` 。Kafka服务器需要一个优雅的断开连接来阻止它继续尝试向已连接的客户端发送消息。如果进程使用 SIGINT (shell 中的 Ctrl-C)或 SIGTERM (如 docker stop 或 kubernetes start)终止，那么下面给出的示例不会调用 `Close()`。当同一topic上有新Reader连接时，可能导致延迟(例如，新进程启动或新容器运行)。在这种场景下应使用`signal.Notify`处理程序在进程关闭时关闭Reader。

#### 消费消息

下面的代码演示了如何使用Reader连接至Kafka消费消息。

```go
// readByReader 通过Reader接收消息
func readByReader() {
	// 创建Reader
	r := kafka.NewReader(kafka.ReaderConfig{
		Brokers:   []string{"localhost:9092", "localhost:9093", "localhost:9094"},
		Topic:     "topic-A",
		Partition: 0,
		MaxBytes:  10e6, // 10MB
	})
	r.SetOffset(42) // 设置Offset

	// 接收消息
	for {
		m, err := r.ReadMessage(context.Background())
		if err != nil {
			break
		}
		fmt.Printf("message at offset %d: %s = %s\n", m.Offset, string(m.Key), string(m.Value))
	}

	// 程序退出前关闭Reader
	if err := r.Close(); err != nil {
		log.Fatal("failed to close reader:", err)
	}
}
```

#### 消费者组

`kafka-go`支持消费者组，包括broker管理的offset。要启用消费者组，只需在 `ReaderConfig` 中指定 `GroupID`。

使用消费者组时，ReadMessage 会自动提交偏移量。

```go
// 创建一个reader，指定GroupID，从 topic-A 消费消息
r := kafka.NewReader(kafka.ReaderConfig{
	Brokers:  []string{"localhost:9092", "localhost:9093", "localhost:9094"},
	GroupID:  "consumer-group-id", // 指定消费者组id
	Topic:    "topic-A",
	MaxBytes: 10e6, // 10MB
})

// 接收消息
for {
	m, err := r.ReadMessage(context.Background())
	if err != nil {
		break
	}
	fmt.Printf("message at topic/partition/offset %v/%v/%v: %s = %s\n", m.Topic, m.Partition, m.Offset, string(m.Key), string(m.Value))
}

// 程序退出前关闭Reader
if err := r.Close(); err != nil {
	log.Fatal("failed to close reader:", err)
}
```

在使用消费者组时会有以下限制：

- `(*Reader).SetOffset` 当设置了GroupID时会返回错误
- `(*Reader).Offset` 当设置了GroupID时会永远返回 `-1`
- `(*Reader).Lag` 当设置了GroupID时会永远返回 `-1`
- `(*Reader).ReadLag` 当设置了GroupID时会返回错误
- `(*Reader).Stats` 当设置了GroupID时会返回一个`-1`的分区

#### 显式提交

`kafka-go` 也支持显式提交。当需要显式提交时不要调用 `ReadMessage`，而是调用 `FetchMessage`获取消息，然后调用 `CommitMessages` 显式提交。

```go
ctx := context.Background()
for {
    // 获取消息
    m, err := r.FetchMessage(ctx)
    if err != nil {
        break
    }
    // 处理消息
    fmt.Printf("message at topic/partition/offset %v/%v/%v: %s = %s\n", m.Topic, m.Partition, m.Offset, string(m.Key), string(m.Value))
    // 显式提交
    if err := r.CommitMessages(ctx, m); err != nil {
        log.Fatal("failed to commit messages:", err)
    }
}
```

在消费者组中提交消息时，具有给定主题/分区的最大偏移量的消息确定该分区的提交偏移量的值。例如，如果通过调用 `FetchMessage` 获取了单个分区的偏移量为 1、2 和 3 的消息，则使用偏移量为3的消息调用 `CommitMessages` 也将导致该分区的偏移量为 1 和 2 的消息被提交。

#### 管理提交间隔

默认情况下，调用`CommitMessages`将同步向Kafka提交偏移量。为了提高性能，可以在ReaderConfig中设置CommitInterval来定期向Kafka提交偏移。

```go
// 创建一个reader从 topic-A 消费消息
r := kafka.NewReader(kafka.ReaderConfig{
    Brokers:        []string{"localhost:9092", "localhost:9093", "localhost:9094"},
    GroupID:        "consumer-group-id",
    Topic:          "topic-A",
    MaxBytes:       10e6, // 10MB
    CommitInterval: time.Second, // 每秒刷新一次提交给 Kafka
})
```

### Writer

向Kafka发送消息，除了使用基于`Conn`的低级API，`kafka-go`包还提供了更高级别的 Writer 类型。大多数情况下使用`Writer`即可满足条件，它支持以下特性。

- 对错误进行自动重试和重新连接。
- 在可用分区之间可配置的消息分布。
- 向Kafka同步或异步写入消息。
- 使用Context的异步取消。
- 关闭时清除挂起的消息以支持正常关闭。
- 在发布消息之前自动创建不存在的topic。

#### 发送消息

```go
// 创建一个writer 向topic-A发送消息
w := &kafka.Writer{
	Addr:         kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
	Topic:        "topic-A",
	Balancer:     &kafka.LeastBytes{}, // 指定分区的balancer模式为最小字节分布
	RequiredAcks: kafka.RequireAll,    // ack模式
	Async:        true,                // 异步
}

err := w.WriteMessages(context.Background(),
	kafka.Message{
		Key:   []byte("Key-A"),
		Value: []byte("Hello World!"),
	},
	kafka.Message{
		Key:   []byte("Key-B"),
		Value: []byte("One!"),
	},
	kafka.Message{
		Key:   []byte("Key-C"),
		Value: []byte("Two!"),
	},
)
if err != nil {
    log.Fatal("failed to write messages:", err)
}

if err := w.Close(); err != nil {
    log.Fatal("failed to close writer:", err)
}
```

#### 创建不存在的topic

如果给Writer配置了`AllowAutoTopicCreation:true`，那么当发送消息至某个不存在的topic时，则会自动创建topic。

```go
w := &Writer{
    Addr:                   kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
    Topic:                  "topic-A",
    AllowAutoTopicCreation: true,  // 自动创建topic
}

messages := []kafka.Message{
    {
        Key:   []byte("Key-A"),
        Value: []byte("Hello World!"),
    },
    {
        Key:   []byte("Key-B"),
        Value: []byte("One!"),
    },
    {
        Key:   []byte("Key-C"),
        Value: []byte("Two!"),
    },
}

var err error
const retries = 3
// 重试3次
for i := 0; i < retries; i++ {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    err = w.WriteMessages(ctx, messages...)
    if errors.Is(err, LeaderNotAvailable) || errors.Is(err, context.DeadlineExceeded) {
        time.Sleep(time.Millisecond * 250)
        continue
    }

    if err != nil {
        log.Fatalf("unexpected error %v", err)
    }
    break
}

// 关闭Writer
if err := w.Close(); err != nil {
    log.Fatal("failed to close writer:", err)
}
```

#### 写入多个topic

通常，`WriterConfig.Topic`用于初始化单个topic的Writer。通过去掉WriterConfig中的Topic配置，分别设置每条消息的`message.topic`，可以实现将消息发送至多个topic。

```go
w := &kafka.Writer{
	Addr:     kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
    // 注意: 当此处不设置Topic时,后续的每条消息都需要指定Topic
	Balancer: &kafka.LeastBytes{},
}

err := w.WriteMessages(context.Background(),
    // 注意: 每条消息都需要指定一个 Topic, 否则就会报错
	kafka.Message{
        Topic: "topic-A",
		Key:   []byte("Key-A"),
		Value: []byte("Hello World!"),
	},
	kafka.Message{
        Topic: "topic-B",
		Key:   []byte("Key-B"),
		Value: []byte("One!"),
	},
	kafka.Message{
        Topic: "topic-C",
		Key:   []byte("Key-C"),
		Value: []byte("Two!"),
	},
)
if err != nil {
    log.Fatal("failed to write messages:", err)
}

if err := w.Close(); err != nil {
    log.Fatal("failed to close writer:", err)
}
```

注意：Writer中的Topic和Message中的Topic是互斥的，同一时刻有且只能设置一处。

### 其他配置

#### TLS

对于基本的 Conn 类型或在 Reader/Writer 配置中，可以在Dialer中设置TLS选项。如果 TLS 字段为空，则它将不启用TLS 连接。

注意：不在Conn/Reder/Writer上配置TLS，连接到启用TLS的Kafka集群，可能会出现`io.ErrUnexpectedEOF`错误。

##### Connection

```go
dialer := &kafka.Dialer{
    Timeout:   10 * time.Second,
    DualStack: true,
    TLS:       &tls.Config{...tls config...},  // 指定TLS配置
}

conn, err := dialer.DialContext(ctx, "tcp", "localhost:9093")
```

##### Reader

```go
dialer := &kafka.Dialer{
    Timeout:   10 * time.Second,
    DualStack: true,
    TLS:       &tls.Config{...tls config...},  // 指定TLS配置
}

r := kafka.NewReader(kafka.ReaderConfig{
    Brokers:        []string{"localhost:9092", "localhost:9093", "localhost:9094"},
    GroupID:        "consumer-group-id",
    Topic:          "topic-A",
    Dialer:         dialer,
})
```

##### Writer

创建`Writer`时可以按如下方式指定TLS配置。

```go
w := kafka.Writer{
    Addr: kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"), 
    Topic:   "topic-A",
    Balancer: &kafka.Hash{},
    Transport: &kafka.Transport{
        TLS: &tls.Config{},  // 指定TLS配置
      },
    }
```

#### SASL

可以在`Dialer`上指定一个选项以使用SASL身份验证。`Dialer`可以直接用来打开一个 Conn，也可以通过它们各自的配置传递给一个 `Reader` 或 `Writer`。如果 `SASLMechanism`字段为 nil，则不会使用 SASL 进行身份验证。

##### SASL 身份验证类型

###### 明文

```go
mechanism := plain.Mechanism{
    Username: "username",
    Password: "password",
}
```

###### SCRAM

```go
mechanism, err := scram.Mechanism(scram.SHA512, "username", "password")
if err != nil {
    panic(err)
}
```

##### Connection

```go
mechanism, err := scram.Mechanism(scram.SHA512, "username", "password")
if err != nil {
    panic(err)
}

dialer := &kafka.Dialer{
    Timeout:       10 * time.Second,
    DualStack:     true,
    SASLMechanism: mechanism,
}

conn, err := dialer.DialContext(ctx, "tcp", "localhost:9093")
```

##### Reader

```go
mechanism, err := scram.Mechanism(scram.SHA512, "username", "password")
if err != nil {
    panic(err)
}

dialer := &kafka.Dialer{
    Timeout:       10 * time.Second,
    DualStack:     true,
    SASLMechanism: mechanism,
}

r := kafka.NewReader(kafka.ReaderConfig{
    Brokers:        []string{"localhost:9092","localhost:9093", "localhost:9094"},
    GroupID:        "consumer-group-id",
    Topic:          "topic-A",
    Dialer:         dialer,
})
```

##### Writer

```go
mechanism, err := scram.Mechanism(scram.SHA512, "username", "password")
if err != nil {
    panic(err)
}

// Transport 负责管理连接池和其他资源,
// 通常最好的使用方式是创建后在应用程序中共享使用它们。
sharedTransport := &kafka.Transport{
    SASL: mechanism,
}

w := kafka.Writer{
	Addr:      kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
	Topic:     "topic-A",
	Balancer:  &kafka.Hash{},
	Transport: sharedTransport,
}
```

##### Client

```go
mechanism, err := scram.Mechanism(scram.SHA512, "username", "password")
if err != nil {
    panic(err)
}

// Transport 负责管理连接池和其他资源,
// 通常最好的使用方式是创建后在应用程序中共享使用它们。
sharedTransport := &kafka.Transport{
    SASL: mechanism,
}

client := &kafka.Client{
    Addr:      kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
    Timeout:   10 * time.Second,
    Transport: sharedTransport,
}
```

#### Balancer

`kafka-go`实现了多种负载均衡策略。特别是当你从其他Kafka库迁移过来时，你可以按如下说明选择合适的Balancer实现。

##### Sarama

如果从 sarama 切换过来，并且需要/希望使用相同的算法进行消息分区，则可以使用`kafka.Hash`或`kafka.ReferenceHash`。

- `kafka.Hash` = `sarama.NewHashPartitioner`
- `kafka.ReferenceHash` = `sarama.NewReferenceHashPartitioner`

```go
w := &kafka.Writer{
	Addr:     kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
	Topic:    "topic-A",
	Balancer: &kafka.Hash{},
}
```

##### librdkafka和confluent-kafka-go

`kafka.CRC32Balancer`与`librdkafka`默认的`consistent_random`策略表现一致。

```go
w := &kafka.Writer{
	Addr:     kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
	Topic:    "topic-A",
	Balancer: kafka.CRC32Balancer{},
}
```

##### Java

使用`kafka.Murmur2Balancer`可以获得与默认Java客户端相同的策略。

```go
w := &kafka.Writer{
	Addr:     kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
	Topic:    "topic-A",
	Balancer: kafka.Murmur2Balancer{},
}
```

#### Compression

可以通过设置`Compression`字段在Writer上启用压缩：

```go
w := &kafka.Writer{
	Addr:        kafka.TCP("localhost:9092", "localhost:9093", "localhost:9094"),
	Topic:       "topic-A",
	Compression: kafka.Snappy,
}
```

`Reader` 将通过检查消息属性来确定消费的消息是否被压缩。

#### Logging

想要记录Reader/Writer类型的操作，可以在创建时配置日志记录器。

kafka-go中的`Logger`是一个接口类型。

```go
type Logger interface {
	Printf(string, ...interface{})
}
```

并且提供了一个`LoggerFunc`类型，帮我们实现了`Logger`接口。

```go
type LoggerFunc func(string, ...interface{})

func (f LoggerFunc) Printf(msg string, args ...interface{}) { f(msg, args...) }
```

##### Reader

借助`kafka.LoggerFunc`我们可以自定义一个`Logger`。

```go
// 自定义一个Logger
func logf(msg string, a ...interface{}) {
	fmt.Printf(msg, a...)
	fmt.Println()
}

r := kafka.NewReader(kafka.ReaderConfig{
	Brokers:     []string{"localhost:9092", "localhost:9093", "localhost:9094"},
	Topic:       "q1mi-topic",
	Partition:   0,
	Logger:      kafka.LoggerFunc(logf),
	ErrorLogger: kafka.LoggerFunc(logf),
})
```

##### Writer

也可以直接使用第三方日志库，例如下面示例代码中使用了zap日志库。

```go
w := &kafka.Writer{
	Addr:        kafka.TCP("localhost:9092"),
	Topic:       "q1mi-topic",
	Logger:      kafka.LoggerFunc(zap.NewExample().Sugar().Infof),
	ErrorLogger: kafka.LoggerFunc(zap.NewExample().Sugar().Errorf),
}
```