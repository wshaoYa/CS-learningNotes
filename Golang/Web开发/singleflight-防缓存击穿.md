# singleflight

本文主要介绍Go语言中的`singleflight`包，包括什么是`singleflight`以及如何使用singleflight合并请求解决缓存击穿问题。

[singleflight](https://pkg.go.dev/golang.org/x/sync/singleflight) 目前（Go1.21）还属于Go的"准"标准库，它提供了重复函数调用抑制机制，使用它可以避免同时进行相同的函数调用。第一个调用未完成时后续的重复调用会等待，当第一个调用完成时则会与它们分享结果，这样以来虽然只执行了一次函数调用但是所有调用都拿到了最终的调用结果。

## 基础示例

我们首先来看以下示例代码，在第1次调用`getData`函数没返回结果时，再次调用`getData`函数。

```go
package main

import (
	"fmt"
	"golang.org/x/sync/singleflight"
	"time"
)

func getData(id int64) string {
	fmt.Println("query...")
	time.Sleep(10 * time.Second) // 模拟一个比较耗时的操作
	return "liwenzhou.com"
}

func main() {
	g := new(singleflight.Group)

	// 第1次调用
	go func() {
		v1, _, shared := g.Do("getData", func() (interface{}, error) {
			ret := getData(1)
			return ret, nil
		})
		fmt.Printf("1st call: v1:%v, shared:%v\n", v1, shared)
	}()

	time.Sleep(2 * time.Second)

	// 第2次调用（第1次调用已开始但未结束）
	v2, _, shared := g.Do("getData", func() (interface{}, error) {
		ret := getData(1)
		return ret, nil
	})
	fmt.Printf("2nd call: v2:%v, shared:%v\n", v2, shared)
}
```

上述代码执行结果如下。

```bash
query...
1st call: v1:liwenzhou.com, shared:true
2nd call: v2:liwenzhou.com, shared:true
```

从输出可以看到`getData`函数只执行了一次（只打印了一次`query...`），但是两次调用都拿到了结果（`liwenzhou.com`）。

这就是`singleflight`包提供给我们的能力，避免了同时执行重复的函数。

## singleflight介绍

`singleflight`包中定义了一个名为`Group`的结构体类型，它表示一类工作，并形成一个命名空间，在这个命名空间中，可以使用重复抑制来执行工作单元。

```go
type Group struct {
	mu sync.Mutex       // 保护 m
	m  map[string]*call // 延迟初始化
}
```

`Group`类型有以下三个方法。

```go
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool)
```

`Do` 执行并返回给定函数的结果，确保一次只有一个给定key在执行。如果进入重复调用，重复调用方将等待原始调用方完成并会收到相同的结果。返回值`shared`表示是否给多个调用方赋值 v。

需要注意的是，使用`Do`方法时，如果第一次调用发生了阻塞，那么后续的调用也会发生阻塞。在极端场景下可能导致程序hang住。

`singleflight`包提供了`DoChan`方法，支持我们异步获取调用结果。

```go
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result
```

`DoChan` 类似于 `Do`，但不是直接返回结果而是返回一个通道，该通道将在结果准备就绪时接收结果。返回的通道将不会关闭。

其中`Result`类型定义如下。

```go
type Result struct {
	Val    interface{}
	Err    error
	Shared bool
}
```

`Result` 保存 `Do` 的结果，因此它们可以在通道上传递。

为了避免第一次调用阻塞所有调用的情况，我们可以结合使用select和`DoChan`为函数调用设置超时时间。

```go
func doChanGetData(ctx context.Context, g *singleflight.Group, id int64) (string, error) {
	ch := g.DoChan("getData", func() (interface{}, error) {
		ret := getData(id)
		return ret, nil
	})
	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case ret := <-ch:
		return ret.Val.(string), ret.Err
	}
}

func main() {
	g := new(singleflight.Group)

	// 第1次调用
	go func() {
		ctx, cancel := context.WithTimeout(context.Background(), time.Second)
		defer cancel()
		v1, err := doChanGetData(ctx, g, 1)
		fmt.Printf("v1:%v err:%v\n", v1, err)
	}()

	time.Sleep(2 * time.Second)

	// 第2次调用（第1次调用已开始但未结束）
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	v2, err := doChanGetData(ctx, g, 1)
	fmt.Printf("v2:%v err:%v\n", v2, err)
}
```

上述代码最终输出结果如下。

```bash
v1: err:context deadline exceeded
v2: err:context deadline exceeded
```

如果在某些场景下允许第一个调用失败后再次尝试调用该函数，而不希望同一时间内的多次请求都因第一个调用返回失败而失败，那么可以通过调用`Forget`方法来忘记这个key。

```go
func (g *Group) Forget(key string)
```

`Forget`告诉`singleflight`忘记一个key。将来对这个key的 `Do` 调用将调用该函数，而不是等待以前的调用完成。

例如，可以在发起调用的同时，在另外的goroutine中延迟100ms调用`Forget`方法来忘记key。

```go
func doGetData(g *singleflight.Group, id int64) (string, error) {
	v, err, _ := g.Do("getData", func() (interface{}, error) {
		go func() {
			time.Sleep(100 * time.Millisecond) // 100ms后忘记key
			g.Forget("getData")
		}()

		ret := getData(id)
		return ret, nil
	})
	return v.(string), err
}
```

## 应用场景

`singleflight`将并发调用合并成一个调用的特点决定了它非常适合用来防止缓存击穿。

下面是一段使用`singleflight`进行查询的伪代码。

```go
func getDataSingleFlight(key string) (interface{}, error) {
	v, err, _ := g.Do(key, func() (interface{}, error) {
		// 查缓存
		data, err := getDataFromCache(key)
		if err == nil {
			return data, nil
		}
		if err == errNotFound {
			// 查DB
			data, err := getDataFromDB(key)
			if err == nil {
				setCache(data) // 设置缓存
				return data, nil
			}
			return nil, err
		}
		return nil, err // 缓存出错直接返回，防止灾难传递至DB
	})

	if err != nil {
		return nil, err
	}
	return v, nil
}
```

如下图所示，在查询数据时使用`singleflight`能够避免业务高峰期缓存失效导致大量请求直接打到DB的情况，从而提高系统的可用性。

![singleflight](https://www.liwenzhou.com/images/Go/singleflight/singleflight.png)

## 总结

`singleflight`通过强制一个函数的所有后续调用等待第一个调用完成，消除了同时运行重复函数的低效性。与缓存不同，它只有在同时调用函数时才共享结果。它充当一个非常短暂的缓存，不需要手动作废或设置有效时间。