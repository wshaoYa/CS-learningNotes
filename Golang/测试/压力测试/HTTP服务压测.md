在项目正式上线之前，我们通常需要通过压测来评估当前系统能够支撑的请求量、排查可能存在的隐藏bug，同时了解了程序的实际处理能力能够帮我们更好的匹配项目的实际需求，节约资源成本。

# HTTP服务压力测试工具

在项目正式上线之前，我们通常需要通过压测来评估当前系统能够支撑的请求量、排查可能存在的隐藏bug，同时了解了程序的实际处理能力能够帮我们更好的匹配项目的实际需求，节约资源成本。

## 压测相关术语

- 响应时间(RT) ：指系统对请求作出响应的时间.
- 吞吐量(Throughput) ：指系统在单位时间内处理请求的数量
- QPS每秒查询率(Query Per Second) ：“每秒查询率”，是一台服务器每秒能够响应的查询次数，是对一个特定的查询服务器在规定时间内所处理流量多少的衡量标准。
- TPS(TransactionPerSecond)：每秒钟系统能够处理的交易或事务的数量
- 并发连接数：某个时刻服务器所接受的请求总数

## 压力测试工具

### ab

ab全称Apache Bench，是Apache自带的性能测试工具。使用这个工具，只须指定同时连接数、请求数以及URL，即可测试网站或网站程序的性能。

通过ab发送请求模拟多个访问者同时对某一URL地址进行访问,可以得到每秒传送字节数、每秒处理请求数、每请求处理时间等统计数据。

命令格式：

```bash
ab [options] [http://]hostname[:port]/path
```

常用参数如下：

```bash
-n requests 总请求数
-c concurrency 一次产生的请求数，可以理解为并发数
-t timelimit 测试所进行的最大秒数, 可以当做请求的超时时间
-p postfile 包含了需要POST的数据的文件
-T content-type POST数据所使用的Content-type头信息
```

更多参数请查看[官方文档](http://httpd.apache.org/docs/2.2/programs/ab.html)。

例如测试某个GET请求接口：

```bash
ab -n 10000 -c 100 -t 10 "http://127.0.0.1:8080/api/v1/posts?size=10"
```

测试POST请求接口：

```bash
ab -n 10000 -c 100 -t 10 -p post.json -T "application/json" "http://127.0.0.1:8080/api/v1/post"
```

### wrk

[wrk](https://github.com/wg/wrk)是一款开源的HTTP性能测试工具，它和上面提到的`ab`同属于HTTP性能测试工具，它比`ab`功能更加强大，可以通过编写lua脚本来支持更加复杂的测试场景。

Mac下安装：

```bash
brew install wrk
```

常用命令参数：

```bash
-c --conections：保持的连接数
-d --duration：压测持续时间(s)
-t --threads：使用的线程总数
-s --script：加载lua脚本
-H --header：在请求头部添加一些参数
--latency 打印详细的延迟统计信息
--timeout 请求的最大超时时间(s)
```

使用示例：

```bash
wrk -t8 -c100 -d30s --latency http://127.0.0.1:8080/api/v1/posts?size=10
```

输出结果：

```bash
Running 30s test @ http://127.0.0.1:8080/api/v1/posts?size=10
  8 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    14.55ms    2.02ms  31.59ms   76.70%
    Req/Sec   828.16     85.69     0.97k    60.46%
  Latency Distribution
     50%   14.44ms
     75%   15.76ms
     90%   16.63ms
     99%   21.07ms
  198091 requests in 30.05s, 29.66MB read
Requests/sec:   6592.29
Transfer/sec:      0.99MB
```

### go-wrk（ver1）

[go-wrk](https://github.com/adjust/go-wrk)是Go语言版本的`wrk`，Windows同学可以使用它来测试，使用如下命令来安装`go-wrk`：

```bash
go get github.com/adeven/go-wrk
```

使用方法同`wrk`类似，基本格式如下：

```bash
go-wrk [flags] url
```

常用的参数：

```bash
-H="User-Agent: go-wrk 0.1 bechmark\nContent-Type: text/html;": 由'\n'分隔的请求头
-c=100: 使用的最大连接数
-k=true: 是否禁用keep-alives
-i=false: if TLS security checks are disabled
-m="GET": HTTP请求方法
-n=1000: 请求总数
-t=1: 使用的线程数
-b="" HTTP请求体
-s="" 如果指定，它将计算响应中包含搜索到的字符串s的频率
```

执行测试：

```bash
go-wrk -t=8 -c=100 -n=10000 "http://127.0.0.1:8080/api/v1/posts?size=10"
```

输出结果：

```bash
==========================BENCHMARK==========================
URL:                            http://127.0.0.1:8080/api/v1/posts?size=10

Used Connections:               100
Used Threads:                   8
Total number of calls:          10000

===========================TIMINGS===========================
Total time passed:              2.74s
Avg time per request:           27.11ms
Requests per second:            3644.53
Median time per request:        26.88ms
99th percentile time:           39.16ms
Slowest time for request:       45.00ms

=============================DATA=============================
Total response body sizes:              340000
Avg response body per request:          34.00 Byte
Transfer rate per second:               123914.11 Byte/s (0.12 MByte/s)
==========================RESPONSES==========================
20X Responses:          10000   (100.00%)
30X Responses:          0       (0.00%)
40X Responses:          0       (0.00%)
50X Responses:          0       (0.00%)
Errors:                 0       (0.00%)
```

### go-wrk（ver2）

与上面的go-wrk同名了，用起来也挺简易好用的

github地址：[tsliwowicz/go-wrk: go-wrk - a HTTP benchmarking tool based in spirit on the excellent wrk tool (https://github.com/wg/wrk)](https://github.com/tsliwowicz/go-wrk)

**使用示例**

![image-20231216200300375](https://s2.loli.net/2023/12/16/Vi6EKqPloCbTXzn.png)