# 分布式ID生成器

## 分布式ID的特点

- **全局唯⼀性**：不能出现有重复的ID标识，这是基本要求。
- **递增性**：确保⽣成ID对于⽤户或业务是递增的。
- **⾼可⽤性**：确保任何时候都能⽣成正确的ID。
- **⾼性能性**：在⾼并发的环境下依然表现良好。

不仅仅是⽤于⽤户ID，实际互联⽹中有很多场景需要能够⽣成类似MySQL⾃增ID这样不断增⼤，同时⼜
不会重复的id。以⽀持业务中的⾼并发场景。

⽐较典型的场景有：电商促销时短时间内会有⼤量的订单涌⼊到系统，⽐如每秒10w+；明星出轨时微博短时间内会产⽣⼤量的相关微博转发和评论消息。在这些业务场景下将数据插⼊数据库之前，我们需要给这些订单和消息先分配⼀个唯⼀ID，然后再保存到数据库中。对这个id的要求是希望其中能带有⼀些时间信息，这样即使我们后端的系统对消息进⾏了分库分表，也能够以时间顺序对这些消息进⾏排序。

## snowflake算法介绍

雪花算法，它是Twitter开源的由64位整数组成分布式ID，性能较⾼，并且在单机上递增。

<img src="https://s2.loli.net/2023/11/28/ZUuovHJAe1IkP2p.png" alt="image-20231128101914347" style="zoom: 50%;" />

1. **第⼀位** 占⽤1bit，其值始终是0，没有实际作⽤。

2. **时间戳** 占⽤41bit，单位为毫秒，总共可以容纳约69年的时间。当然，我们的时间毫秒计数不会真的从1970年开始记，那样我们的系统跑到 2039/9/7 23:47:35 就不能⽤了，所以这⾥的时间戳只是相对于某个时间的增量，⽐如我们的系统上线是2020-07-01，那么我们完全可以把这个timestamp当作是从2020-07-01 00:00:00.000 的偏移量。

3. **⼯作机器id** 占⽤10bit，其中⾼位5bit是数据中⼼ID，低位5bit是⼯作节点ID，最多可以容纳1024个节点。
4. **序列号** 占⽤12bit，⽤来记录同毫秒内产⽣的不同id。每个节点每毫秒0开始不断累加，最多可以累加到4095，同⼀毫秒⼀共可以产⽣4096个ID。

SnowFlake算法在同⼀毫秒内最多可以⽣成多少个全局唯⼀ID呢？

同一毫秒的ID数量 **= 1024 X 4096 = 4194304**

## snowflake的Go实现

**https://github.com/bwmarrin/snowflake**

[github.com/bwmarrin/snowflake]()是⼀个相当轻量化的snowflflake的Go实现。

<img src="https://s2.loli.net/2023/11/28/YdhJcm7SkTKxeX6.png" alt="image-20231128102232563"  />

```go
package snowflake

import (
    sf "github.com/bwmarrin/snowflake"
    "time"
)

var node *sf.Node

// Init 初始化雪花算法node
func Init(startTime string, machineID int64) (err error) {
    // 解析时间格式
    var st time.Time
    if st, err = time.Parse("2006-01-02", startTime); err != nil {
       return
    }

    // 设置起始时间（毫秒级别）
    sf.Epoch = st.UnixNano() / 1000_000 // 纳秒 转换为 毫秒

    // 在特定机器节点node上产生ID（分布式使用背景）
    node, err = sf.NewNode(machineID)
    return
}

// GenID 生成ID
func GenID() int64 {
    return node.Generate().Int64()
}
```

## sonyflake

 https://github.com/sony/sonyflake

sonyflake是Sony公司的⼀个开源项⽬，基本思路和snowflake差不多，不过位分配上稍有不同

![image-20231128102632362](C:/Users/sw/AppData/Roaming/Typora/typora-user-images/image-20231128102632362.png)

这⾥的时间只⽤了39个bit，但时间的单位变成了10ms，所以理论上⽐41位表示的时间还要久(174年)。Sequence ID 和之前的定义⼀致， Machine ID 其实就是节点id。 sonyflake 库有以下配置参数：

```go
type Settings struct {
	StartTime      time.Time
	MachineID      func() (uint16, error)
	CheckMachineID func(uint16) bool
}
```

其中：

- `StartTime` 选项和我们之前的 `Epoch` 差不多，如果不设置的话，默认是从 `2014-09-01 00:00:00 +0000 UTC` 开始。
- `MachineID` 可以由⽤户⾃定义的函数，如果⽤户不定义的话，会默认将本机IP的低16位作为 `machine id` 
- `CheckMachineID` 是由⽤户提供的检查 `MachineID` 是否冲突的函数。

使⽤示例：

```go
import (
    "fmt"
    "github.com/sony/sonyflake"
    "time"
)

var (
    sonyFlake     *sonyflake.Sonyflake
    sonyMachineID uint16
)

func getMachineID() (uint16, error) {
    return sonyMachineID, nil
}

// Init 需传⼊当前的机器ID
func Init(startTime string, machineId uint16) (err error) {
    sonyMachineID = machineId
    var st time.Time
    st, err = time.Parse("2006-01-02", startTime)
    if err != nil {
       return err
    }
    settings := sonyflake.Settings{
       StartTime: st,
       MachineID: getMachineID,
    }
    sonyFlake = sonyflake.NewSonyflake(settings)
    return
}

// GenID ⽣成id
func GenID() (id uint64, err error) {
    if sonyFlake == nil {
       err = fmt.Errorf("snoy flake not inited")
       return
    }
    id, err = sonyFlake.NextID()
    return
}
func main() {
    if err := Init("2020-07-01", 1); err != nil {
       fmt.Printf("Init failed, err:%v\n", err)
       return
    }
    id, _ := GenID()
    fmt.Println(id)
}
```



