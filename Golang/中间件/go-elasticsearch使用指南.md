# go-elasticsearch使用指南

本文是 go-elasticsearch 库的使用指南。

[go-elasticsearch](https://github.com/elastic/go-elasticsearch)是Elasticsearch 官方提供的 Go 客户端。每个 Elasticsearch 版本会有一个对应的 go-elasticsearch 版本。官方会维护最近的两个主要版本。

go-elasticsearch 提供了 Low-level 和 Fully-typed 两套API。本文以 Fully-typed API 为例介绍 go-elasticsearch 的常用方法。

> 关于 Elasticsearch 介绍和如何使用 docker 在本地搭建 Elasticsearch 环境请查看我之前的博客 [Elasticsearch简明教程](https://www.liwenzhou.com/posts/Go/elasticsearch/)。

## 安装依赖

执行以下命令安装v8版本的 go 客户端。

```bash
go get github.com/elastic/go-elasticsearch/v8@latest
```

导入依赖。

```go
import "github.com/elastic/go-elasticsearch/v8"
```

可以根据实际需求导入不同的客户端版本，也支持在一个项目中导入不同的客户端版本。

```go
import (
  elasticsearch7 "github.com/elastic/go-elasticsearch/v7"
  elasticsearch8 "github.com/elastic/go-elasticsearch/v8"
)

// ...
es7, _ := elasticsearch7.NewDefaultClient()
es8, _ := elasticsearch8.NewDefaultClient()
```

## 连接 ES

指定要连接 ES 的相关配置，并创建客户端连接。

```go
// ES 配置
cfg := elasticsearch.Config{
	Addresses: []string{
		"http://localhost:9200",
	},
}

// 创建客户端连接
client, err := elasticsearch.NewTypedClient(cfg)
if err != nil {
	fmt.Printf("elasticsearch.NewTypedClient failed, err:%v\n", err)
	return
}
```

## 操作 ES

本文接下来将以电商平台 “用户评价” 数据为例，演示 Go 语言 Elasticsearch 客户端的相关操作。

### 创建 index

创建一个名为 `my-review-1` 的 index。

```go
// createIndex 创建索引
func createIndex(client *elasticsearch.TypedClient) {
	resp, err := client.Indices.
		Create("my-review-1").
		Do(context.Background())
	if err != nil {
		fmt.Printf("create index failed, err:%v\n", err)
		return
	}
	fmt.Printf("index:%#v\n", resp.Index)
}
```

### 索引 document

定义与 document 数据对应的 `Review` 和 `Tag` 结构体，分别表示"评价"和"评价标签"。

```go
// Review 评价数据
type Review struct {
	ID          int64     `json:"id"`
	UserID      int64     `json:"userID"`
	Score       uint8     `json:"score"`
	Content     string    `json:"content"`
	Tags        []Tag     `json:"tags"`
	Status      int       `json:"status"`
	PublishTime time.Time `json:"publishDate"`
}

// Tag 评价标签
type Tag struct {
	Code  int    `json:"code"`
	Title string `json:"title"`
}
```

创建一条 document 并添加到 `my-review-1` 的 index 中。

```go
// indexDocument 索引文档
func indexDocument(client *elasticsearch.TypedClient) {
	// 定义 document 结构体对象
	d1 := Review{
		ID:      1,
		UserID:  147982601,
		Score:   5,
		Content: "这是一个好评！",
		Tags: []Tag{
			{1000, "好评"},
			{1100, "物超所值"},
			{9000, "有图"},
		},
		Status:      2,
		PublishTime: time.Now(),
	}

	// 添加文档
	resp, err := client.Index("my-review-1").
		Id(strconv.FormatInt(d1.ID, 10)).
		Document(d1).
		Do(context.Background())
	if err != nil {
		fmt.Printf("indexing document failed, err:%v\n", err)
		return
	}
	fmt.Printf("result:%#v\n", resp.Result)
}
```

### 获取 document

根据 id 获取 document。

```go
// getDocument 获取文档
func getDocument(client *elasticsearch.TypedClient, id string) {
	resp, err := client.Get("my-review-1", id).
		Do(context.Background())
	if err != nil {
		fmt.Printf("get document by id failed, err:%v\n", err)
		return
	}
	fmt.Printf("fileds:%s\n", resp.Source_)
}
```

### 检索 document

构建搜索查询可以使用结构化的查询条件。

```go
import (
	"context"
	"fmt"
	"github.com/elastic/go-elasticsearch/v8"
	"github.com/elastic/go-elasticsearch/v8/typedapi/types"
)

// searchDocument 搜索所有文档
func searchDocument(client *elasticsearch.TypedClient) {
	// 搜索文档
	resp, err := client.Search().
		Index("my-review-1").
		Query(&types.Query{
			MatchAll: &types.MatchAllQuery{},
		}).
		Do(context.Background())
	if err != nil {
		fmt.Printf("search document failed, err:%v\n", err)
		return
	}
	fmt.Printf("total: %d\n", resp.Hits.Total.Value)
	// 遍历所有结果
	for _, hit := range resp.Hits.Hits {
		fmt.Printf("%s\n", hit.Source_)
	}
}
```

下面是在 `my-review-1` 中搜索 content 包含 “好评” 的文档。

```go
import (
	"context"
	"fmt"
	"github.com/elastic/go-elasticsearch/v8"
	"github.com/elastic/go-elasticsearch/v8/typedapi/types"
)

// searchDocument2 指定条件搜索文档
func searchDocument2(client *elasticsearch.TypedClient) {
	// 搜索content中包含好评的文档
	resp, err := client.Search().
		Index("my-review-1").
		Query(&types.Query{
			MatchPhrase: map[string]types.MatchPhraseQuery{
				"content": {Query: "好评"},
			},
		}).
		Do(context.Background())
	if err != nil {
		fmt.Printf("search document failed, err:%v\n", err)
		return
	}
	fmt.Printf("total: %d\n", resp.Hits.Total.Value)
	// 遍历所有结果
	for _, hit := range resp.Hits.Hits {
		fmt.Printf("%s\n", hit.Source_)
	}
}
```

### 聚合

在 `my-review-1` 上运行一个平均值聚合，得到所有文档 score 的平均值。

```go
import (
	"context"
	"fmt"
	"github.com/elastic/go-elasticsearch/v8"
	"github.com/elastic/go-elasticsearch/v8/typedapi/core/search"
	"github.com/elastic/go-elasticsearch/v8/typedapi/some"
	"github.com/elastic/go-elasticsearch/v8/typedapi/types"
)

// aggregationDemo 聚合
func aggregationDemo(client *elasticsearch.TypedClient) {
	avgScoreAgg, err := client.Search().
		Index("my-review-1").
		Request(
			&search.Request{
				Size: some.Int(0),
				Aggregations: map[string]types.Aggregations{
					"avg_score": { // 将所有文档的 score 的平均值聚合为 avg_score
						Avg: &types.AverageAggregation{
							Field: some.String("score"),
						},
					},
				},
			},
		).Do(context.Background())
	if err != nil {
		fmt.Printf("aggregation failed, err:%v\n", err)
		return
	}
	fmt.Printf("avgScore:%#v\n", avgScoreAgg.Aggregations["avg_score"])
}
```

### 更新 document

使用新值更新文档。

```go
// updateDocument 更新文档
func updateDocument(client *elasticsearch.TypedClient) {
	// 修改后的结构体变量
	d1 := Review{
		ID:      1,
		UserID:  147982601,
		Score:   5,
		Content: "这是一个修改后的好评！", // 有修改
		Tags: []Tag{ // 有修改
			{1000, "好评"},
			{9000, "有图"},
		},
		Status:      2,
		PublishTime: time.Now(),
	}

	resp, err := client.Update("my-review-1", "1").
		Doc(d1). // 使用结构体变量更新
		Do(context.Background())
	if err != nil {
		fmt.Printf("update document failed, err:%v\n", err)
		return
	}
	fmt.Printf("result:%v\n", resp.Result)
}
```

更新可以使用结构体变量也可以使用原始JSON字符串数据。

```go
// updateDocument2 更新文档
func updateDocument2(client *elasticsearch.TypedClient) {
	// 修改后的JSON字符串
	str := `{
					"id":1,
					"userID":147982601,
					"score":5,
					"content":"这是一个二次修改后的好评！",
					"tags":[
						{
							"code":1000,
							"title":"好评"
						},
						{
							"code":9000,
							"title":"有图"
						}
					],
					"status":2,
					"publishDate":"2023-12-10T15:27:18.219385+08:00"
				}`
	// 直接使用JSON字符串更新
	resp, err := client.Update("my-review-1", "1").
		Request(&update.Request{
			Doc: json.RawMessage(str),
		}).
		Do(context.Background())
	if err != nil {
		fmt.Printf("update document failed, err:%v\n", err)
		return
	}
	fmt.Printf("result:%v\n", resp.Result)
}
```

### 删除 document

根据文档 id 删除文档。

```go
// deleteDocument 删除 document
func deleteDocument(client *elasticsearch.TypedClient) {
	resp, err := client.Delete("my-review-1", "1").
		Do(context.Background())
	if err != nil {
		fmt.Printf("delete document failed, err:%v\n", err)
		return
	}
	fmt.Printf("result:%v\n", resp.Result)
}
```

### 删除 index

删除指定的 index。

```go
// deleteIndex 删除 index
func deleteIndex(client *elasticsearch.TypedClient) {
	resp, err := client.Indices.
		Delete("my-review-1").
		Do(context.Background())
	if err != nil {
		fmt.Printf("delete document failed, err:%v\n", err)
		return
	}
	fmt.Printf("Acknowledged:%v\n", resp.Acknowledged)
}
```

## 参考资料

[elasticsearch go-api](https://www.elastic.co/guide/en/elasticsearch/client/go-api/current/getting-started-go.html)

[elasticsearch go-api examples](https://www.elastic.co/guide/en/elasticsearch/client/go-api/current/examples.html)