# time

time 包中日历的计算采用的是公历

## **时间类型**

使用`time.Time`类型表示时间

可以通过`time.Now`函数获取当前的时间对象，然后从时间对象中可以获取到年、月、日、时、分、秒等信息。

```go
// timeDemo 时间对象的年月日时分秒
func timeDemo() {
	now := time.Now() // 获取当前时间
	fmt.Printf("current time:%v\n", now)

	year := now.Year()     // 年
	month := now.Month()   // 月
	day := now.Day()       // 日
	hour := now.Hour()     // 小时
	minute := now.Minute() // 分钟
	second := now.Second() // 秒
	fmt.Println(year, month, day, hour, minute, second)
}

current time:2023-09-30 15:26:34.5865235 +0800 CST m=+0.003144001
2023 September 30 15 26 34
```

## **Location/time zone**

使用 location 来映射具体的时区

一般为了使用方便只用东八时区的标准时即北京时间为准。

```go
// timezoneDemo 时区示例
func timezoneDemo() {
	// 中国没有夏令时，使用一个固定的8小时的UTC时差。
	// 对于很多其他国家需要考虑夏令时。
	secondsEastOfUTC := int((8 * time.Hour).Seconds())
	// FixedZone 返回始终使用给定区域名称和偏移量(UTC 以东秒)的 Location。
	beijing := time.FixedZone("Beijing Time", secondsEastOfUTC)

	// 如果当前系统有时区数据库，则可以加载一个位置得到对应的时区
	// 例如，加载纽约所在的时区
	newYork, err := time.LoadLocation("America/New_York") // UTC-05:00
	if err != nil {
		fmt.Println("load America/New_York location failed", err)
		return
	}
	fmt.Println()
	// 加载上海所在的时区
	//shanghai, err := time.LoadLocation("Asia/Shanghai") // UTC+08:00
	// 加载东京所在的时区
	//tokyo, err := time.LoadLocation("Asia/Tokyo") // UTC+09:00

	// 创建时间对象需要指定位置。常用的位置是 time.Local（当地时间） 和 time.UTC（UTC时间）。
	//timeInLocal := time.Date(2009, 1, 1, 20, 0, 0, 0, time.Local)  // 系统本地时间
	timeInUTC := time.Date(2009, 1, 1, 12, 0, 0, 0, time.UTC)
	sameTimeInBeijing := time.Date(2009, 1, 1, 20, 0, 0, 0, beijing)
	sameTimeInNewYork := time.Date(2009, 1, 1, 7, 0, 0, 0, newYork)

	// 北京时间（东八区）比UTC早8小时，所以上面两个时间看似差了8小时，但表示的是同一个时间
	timesAreEqual := timeInUTC.Equal(sameTimeInBeijing)
	fmt.Println(timesAreEqual)

	// 纽约（西五区）比UTC晚5小时，所以上面两个时间看似差了5小时，但表示的是同一个时间
	timesAreEqual = timeInUTC.Equal(sameTimeInNewYork)
	fmt.Println(timesAreEqual)
}

true
true
```

日常编码过程中使用时间对象的时候一定要注意其时区信息，由于`time.LoadLocation`依赖系统的时区数据库，在不太确定程序运行环境的情况下建议先定义时区偏移量然后使用`time.FixedZone`的方式指定时区。

## **Unix Time**

Unix Time是自1970年1月1日 00:00:00 UTC 至当前时间经过的总秒数。

```go
// timestampDemo 时间戳
func timestampDemo() {
	now := time.Now()        // 获取当前时间
	timestamp := now.Unix()  // 秒级时间戳
	milli := now.UnixMilli() // 毫秒时间戳 Go1.17+
	micro := now.UnixMicro() // 微秒时间戳 Go1.17+
	nano := now.UnixNano()   // 纳秒时间戳
	fmt.Println(timestamp, milli, micro, nano)
}

1696059821 1696059821580 1696059821580089 1696059821580089800
```

time 包还提供了一系列将 int64 类型的时间戳转换为时间对象的方法。

```go
// timestamp2Time 将时间戳转为时间对象
func timestamp2Time() {
	// 获取北京时间所在的东八区时区对象
	secondsEastOfUTC := int((8 * time.Hour).Seconds())
	beijing := time.FixedZone("Beijing Time", secondsEastOfUTC)

	// 北京时间 2022-02-22 22:22:22.000000022 +0800 CST
	t := time.Date(2022, 02, 22, 22, 22, 22, 22, beijing)

	var (
		sec  = t.Unix()
		msec = t.UnixMilli()
		usec = t.UnixMicro()
	)

	// 将秒级时间戳转为时间对象（第二个参数为不足1秒的纳秒数）
	timeObj := time.Unix(sec, 22)
	fmt.Println(timeObj)           // 2022-02-22 22:22:22.000000022 +0800 CST
	timeObj = time.UnixMilli(msec) // 毫秒级时间戳转为时间对象
	fmt.Println(timeObj)           // 2022-02-22 22:22:22 +0800 CST
	timeObj = time.UnixMicro(usec) // 微秒级时间戳转为时间对象
	fmt.Println(timeObj)           // 2022-02-22 22:22:22 +0800 CST
}

2022-02-22 22:22:22.000000022 +0800 CST
2022-02-22 22:22:22 +0800 CST
2022-02-22 22:22:22 +0800 CST
```

## **时间间隔**

`time.Duration`是`time`包定义的一个类型，它代表两个时间点之间经过的时间，以纳秒为单位。

`time.Duration`表示一段时间间隔，可表示的最长时间段大约290年。

time 包中定义的时间间隔类型的常量如下：

```go
const (
    Nanosecond  Duration = 1
    Microsecond          = 1000 * Nanosecond
    Millisecond          = 1000 * Microsecond
    Second               = 1000 * Millisecond
    Minute               = 60 * Second
    Hour                 = 60 * Minute
)
```

其中`time.Duration`表示1纳秒，`time.Second`表示1秒。

## **时间操作**

### **Add**

```go
func (t Time) Add(d Duration) Time
```

举个例子，求一个小时之后的时间：

```go
func main() {
	now := time.Now()
	later := now.Add(time.Hour) // 当前时间加1小时后的时间
	fmt.Println(later)
}
```

### **Sub**

求两个时间之间的差值：

```go
func (t Time) Sub(u Time) Duration
```

如果结果超出了Duration可以表示的最大值/最小值，将返回最大值/最小值。

### **Equal**

判断两个时间是否相同，会考虑时区的影响，因此不同时区标准的时间也可以正确比较。

本方法和用t==u不同，这种方法还会比较地点和时区信息。

```go
func (t Time) Equal(u Time) bool
```

### **Before**

如果t代表的时间点在u之前，返回真；否则返回假。

```go
func (t Time) Before(u Time) bool
```

### **After**

如果t代表的时间点在u之后，返回真；否则返回假。

```go
func (t Time) After(u Time) bool
```

### **Since**

从t开始到现在有多少时间

```go
func Since(t Time) Duration
```

## **定时器**

使用`time.Tick(时间间隔)`来设置定时器，定时器的本质上是一个通道（channel）。

```go
func tickDemo() {
	ticker := time.Tick(time.Second) //定义一个1秒间隔的定时器
	for i := range ticker {
		fmt.Println(i)//每秒都会执行的任务
	}
}
```

## **时间格式化**

`time.Format`函数能够将一个时间对象格式化输出为指定布局的文本表示形式

需要注意的是 Go 语言中时间格式化的布局不是常见的`Y-m-d H:M:S`，而是使用 `2006-01-02 15:04:05.000`（记忆口诀为2006 1 2 3 4 5）。

其中：

- 2006：年（Y）
- 01：月（m）
- 02：日（d）
- 15：时（H）
- 04：分（M）
- 05：秒（S）

补充：

- 如果想格式化为12小时格式，需在格式化布局中添加`PM`。
- 小数部分想保留指定位数就写0，如果想省略末尾可能的0就写 9。

```go
// formatDemo 时间格式化
func formatDemo() {
	now := time.Now()
	// 格式化的模板为 2006-01-02 15:04:05

	// 24小时制
	fmt.Println(now.Format("2006-01-02 15:04:05.000 Monday January"))
	// 12小时制
	fmt.Println(now.Format("2006-01-02 03:04:05.000 PM Mon Jan"))

	// 小数点后写0，因为有3个0所以格式化输出的结果也保留3位小数
	fmt.Println(now.Format("2006/01/02 15:04:05.000")) // 2022/02/27 00:10:42.960
	// 小数点后写9，会省略末尾可能出现的0
	fmt.Println(now.Format("2006/01/02 15:04:05.999")) // 2022/02/27 00:10:42.96

	// 只格式化时分秒部分
	fmt.Println(now.Format("15:04:05"))
	// 只格式化日期部分
	fmt.Println(now.Format("2006.01.02"))
}

2023-09-30 16:01:56.746 Saturday September
2023-09-30 04:01:56.746 PM Sat Sep
2023/09/30 16:01:56.746
2023/09/30 16:01:56.746
16:01:56
2023.09.30
```

## **解析字符串格式的时间**

从文本的时间表示中解析出时间对象

`time`包中提供了`time.Parse`和`time.ParseInLocation`两个函数。

### **time.Parse**

解析时不需要额外指定时区信息。

```go
// 在没有时区指示符的情况下，time.Parse 返回UTC时间
timeObj, err := time.Parse("2006/01/02 15:04:05", "2022/10/05 11:25:20")
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println(timeObj) // 2022-10-05 11:25:20 +0000 UTC

// 在有时区指示符的情况下，time.Parse 返回对应时区的时间表示
// RFC3339     = "2006-01-02T15:04:05Z07:00"
timeObj, err = time.Parse(time.RFC3339, "2022-10-05T11:25:20+08:00")
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println(timeObj) // 2022-10-05 11:25:20 +0800 CST

2022-10-05 11:25:20 +0000 UTC
2022-10-05 11:25:20 +0800 CST
```

### **time.ParseInLocation**

需要在解析时额外指定时区信息。

```go
// parseDemo 解析时间
func parseDemo() {
	now := time.Now()
	fmt.Println(now)
	// 加载时区
	loc, err := time.LoadLocation("Asia/Shanghai")
	if err != nil {
		fmt.Println(err)
		return
	}
	// 按照指定时区和指定格式解析字符串时间
	timeObj, err := time.ParseInLocation("2006/01/02 15:04:05", "2022/10/05 11:25:20", loc)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(timeObj)
	fmt.Println(timeObj.Sub(now))
}

2023-09-30 16:12:27.9623563 +0800 CST m=+0.004030201
2022-10-05 11:25:20 +0800 CST
-8644h47m7.9623563s
```

## **练习题**

1. 获取当前时间，格式化输出为`2017/06/19 20:30:05`格式。

   ```go
   now := time.Now()
   fmt.Println(now.Format("2006/01/02 15:04:05"))
   
   2023/09/30 16:16:16
   ```

2. 编写程序统计一段代码的执行耗时时间，单位精确到微秒。

```go
func main() {
    t1 := time.Now()
    for i := 0; i < 1e9; i++ {
       i++
    }
    t2 := time.Now()
    fmt.Printf("%d μs", t2.Sub(t1).Microseconds())
}

119133 μs
```

或

```go
func timeCost(start time.Time) {
    fmt.Printf("cost time %v μs \n", time.Since(start).Microseconds())
}

func main() {
    defer timeCost(time.Now())
    for i := 0; i < 1e9; i++ {
       i++
    }
}

cost time 120024 μs 
```

# 