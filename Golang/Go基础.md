## 开发环境

### 编译运行

#### go mod init

`go mod init 项目名` 

可对项目进行初始化，生成一个go.mod

#### go build

将源代码编译成可执行文件

`-o`参数可指定编译后可执行文件名字

#### go run

`go run xx.go`

本质上是先在临时目录编译程序然后再执行，相当于 go build + 运行

#### go install

编译源码得到可执行文件，并将文件移动至`GOPATH`的bin目录下

#### 跨平台编译

- Win编译Linux（cmd终端）

  - ```bash
    SET CGO_ENABLED=0  // 禁用CGO
    SET GOOS=linux  // 目标平台是linux
    SET GOARCH=amd64  // 目标处理器架构是amd64
    ```

  - 设置完后，`go build`即可得到linux平台的可执行文件

- Linux编译Win

  - ```bash
    CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build
    ```

### 依赖管理 (go module)

**GO111MODULE**

考虑到 Go1.16 之后 go module 已经默认开启，所以不再介绍该配置，对于刚接触Go语言的读者而言**完全没有必要了解这个历史包袱。**

~~环境变量`GO111MODULE`，通过它可以开启或关闭模块支持，它有三个可选值：`off`、`on`、`auto`，默认值是`auto`。~~

- ~~`GO111MODULE=off`禁用模块支持，编译时会从`GOPATH`和`vendor`文件夹中查找包。~~
- ~~`GO111MODULE=on`启用模块支持，编译时会忽略`GOPATH`和`vendor`文件夹，只根据 `go.mod`下载依赖。~~
- ~~`GO111MODULE=auto`，当项目在`$GOPATH/src`外且项目根目录有`go.mod`文件时，开启模块支持。~~

#### GOPROXY

设置为国内源

```bash
export GOPROXY=https://goproxy.cn
```

或

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

允许设置多个代理地址，多个地址之间需使用英文逗号 “,” 分隔。最后的 “direct” 是一个特殊指示符，用于指示 Go 回源到源地址去抓取（比如 GitHub 等）。当配置有多个代理地址时，如果第一个代理地址返回 404 或 410 错误时，Go 会自动尝试下一个代理地址，当遇见 “direct” 时触发回源，也就是回到源地址去抓取。

#### **GOPRIVATE**

在项目中引入了非公开的包（公司内部git仓库或 github 私有仓库等）。

GOPRIVATE用来告诉 go 命令哪些仓库属于私有仓库，不必通过代理服务器拉取和校验。

可以设置多个，多个地址之间使用英文逗号 “,” 分隔。我们通常会把自己公司内部的代码仓库设置到 GOPRIVATE 中，例如：

```bash
go env -w GOPRIVATE="git.mycompany.com"
```

这样在拉取以`git.mycompany.com`为路径前缀的依赖包时就能正常拉取了。

如果公司内部自建了 GOPROXY 服务，那么我们可以通过设置 `GONOPROXY=none`，允许通内部代理拉取私有仓库的包。

#### **初始化项目**

```bash
$ mkdir holiday
$ cd holiday
$ go mod init holiday
```

会自动在项目目录下创建一个`go.mod`文件

```go
module holiday

go 1.16
```

#### 下载依赖包

`holiday`项目现在需要引入一个第三方包`github.com/q1mi/hello`来实现一些必要的功能。

需要先将依赖包下载到本地同时在`go.mod`中记录依赖信息，然后才能在我们的代码中引入并使用这个包。

##### **两种方式**

###### go get 命令

执行`go get`命令可以下载依赖包，并且还可以指定下载的版本。

- 运行`go get -u`将会升级到最新的次要版本或者修订版本(x.y.z, z是修订版本号， y是次要版本号)
- 运行`go get -u=patch`将会升级到最新的修订版本
- 运行`go get package@version`将会升级到指定的版本号version

- `go get`命令手动下载

  - `-u`参数，强制更新现有依赖。

  - 默认会下载最新的发布版本

    - ```bash
      $ go get -u github.com/q1mi/hello
      ```

    - 如果依赖包没有发布任何版本则会拉取最新的提交，最终`go.mod`中的依赖信息会变成类似下面这种由默认v0.0.0的版本号和最新一次commit的时间和hash组成的版本格式：

    - ```go
      require github.com/q1mi/hello v0.0.0-20210218074646-139b0bcd549d
      ```

  - 也可以指定想要下载指定的版本号

    - ```bash
      $ go get -u github.com/q1mi/hello@v0.1.0
      ```

  - 下载某个commit对应的代码，可以直接指定commit hash，前7位即可。

    - ```bash
      $ go get github.com/q1mi/hello@2ccfadd
      ```

    - ```go
      require github.com/q1mi/hello v0.1.0 // indirect
      
      `indirect`表示该依赖包为间接依赖，说明在当前程序中的所有 import 语句中没有发现引入这个包。
      ```

###### 编辑go.mod

- 直接编辑`go.mod`文件（将依赖包和版本信息写入该文件）

  - 按照版本

    - ```go
      require github.com/q1mi/hello latest  // latest 表示最新版本 也可指定具体的版本号
      ```

    - 然后`go mod download`下载依赖包

      - ```bash
        $ go mod download
        ```

      - go.mod将变为

      - ```go
        require github.com/q1mi/hello v0.1.1
        ```

  - 按照commit

    - ```go
      require github.com/q1mi/hello 2ccfadda
      ```

    - 然后go mod download，go.mod会变为

    - ```go
      require github.com/q1mi/hello v0.1.2-0.20210219092711-2ccfaddad6a3
      ```

下载好要使用的依赖包之后，我们现在就可以在`holiday/main.go`文件中使用这个包了。

```go
package main

import (
	"fmt"

	"github.com/q1mi/hello"
)

func main() {
	fmt.Println("现在是假期时间...")

	hello.SayHi() // 调用hello包的SayHi函数
}
```

##### **依赖保存位置**

保存在 `$GOPATH/pkg/mod`目录下，每个依赖包都会带有版本号进行区分，这样就允许在本地存在同一个包的多个不同版本。

如果想清除所有本地已缓存的依赖包数据，可以执行 `go clean -modcache` 命令。

```bash
mod
├── cache
├── cloud.google.com
├── github.com
    	└──q1mi
          ├── hello@v0.0.0-20210218074646-139b0bcd549d
          ├── hello@v0.1.1
          └── hello@v0.1.0
...
```

#### 引入包

##### **本地同项目引入**

在项目内部按功能或业务划分成多个不同包。例如：

```bash
holidy
├── go.mod
├── go.sum
├── main.go
└── summer
    └── summer.go
```

其中`holiday/summer/summer.go`文件内容

```go
package summer

import "fmt"

// Diving 潜水...
func Diving() {
	fmt.Println("夏天去诗巴丹潜水...")
}
```

main中使用summer包里的函数

项目中定义的包都会以项目的导入路径为前缀。

```go
package main

import (
	"fmt"

	"holiday/summer" // 导入当前项目下的包

	"github.com/q1mi/hello" // 导入github上第三方包
)

func main() {
	fmt.Println("现在是假期时间...")
	hello.SayHi()

	summer.Diving()
}
```

##### **本地不同项目引入**

```bash
├── holiday
│   ├── go.mod
│   ├── go.sum
│   ├── main.go
│   └── summer
│       └── summer.go
└── overtime
    ├── go.mod
    └── overtime.go
```

在`holidy/go.mod`文件中正常引入`liwenzhou.com/overtime`包，然后像下面的示例那样使用`replace`语句将这个依赖替换为使用相对路径表示的本地包。

```go
module holiday

go 1.16

require github.com/q1mi/hello v0.1.1
require liwenzhou.com/overtime v0.0.0

replace liwenzhou.com/overtime  => ../overtime
```

就可以正常使用`overtime`包了

```go
package main

import (
	"fmt"

	"holiday/summer" // 导入当前项目下的包

	"liwenzhou.com/overtime" // 通过replace导入的本地包

	"github.com/q1mi/hello" // 导入github上第三方包
)

func main() {
	fmt.Println("现在是假期时间...")
	hello.SayHi()

	summer.Diving()

	overtime.Do()
}
```

我们也经常使用`replace`将项目依赖中的某个包，替换为其他版本的代码包或我们自己修改后的代码包。

#### 发布包

发布一个自己编写的代码包或者在公司内部编写一个供内部使用的公用组件

编写一个代码包并将它发布到`github.com`仓库，让它能够被全球的Go语言开发者使用。

##### **新建项目**

自己的 github 账号下新建一个项目，并把它下载到本地

```bash
$ git clone https://github.com/q1mi/hello
$ cd hello
```

##### **初始化项目**

注意！！这里定义项目的引入路径为`github.com/q1mi/hello`

```bash
$ go mod init github.com/q1mi/hello
```

##### **编写代码**

```go
package hello

import "fmt"

func SayHi() {
	fmt.Println("你好，我是七米。很高兴认识你。")
}
```

##### **push代码**

将该项目的代码 push 到仓库的远端分支，这样就对外发布了一个Go包。

其他的开发者可以通过`github.com/q1mi/hello`这个引入路径下载并使用这个包了。

##### **版本号明确**

一个设计完善的包应该包含开源许可证及文档等内容，并且我们还应该尽心维护并适时发布适当的版本。

github 上发布版本号使用git tag为代码包打上标签即可。

```bash
$ git tag -a v0.1.0 -m "release version v0.1.0"
$ git push origin v0.1.0
```

Go modules中建议使用语义化版本控制，其建议的版本号格式如下：

![image-20230929101806906](https://s2.loli.net/2023/10/28/lQ4q9UngZhKa1ox.png)

- 主版本号：发布了不兼容的版本迭代时递增（breaking changes）。
- 次版本号：发布了功能性更新时递增。
- 修订号：发布了bug修复类更新时递增。

##### **发布新的主版本**

项目要进行与之前版本不兼容的更新时，对之前使用该包作为依赖的用户影响巨大。

因此我们需要发布一个主版本号递增的`v2`版本。在这种情况下，我们通常会修改当前包的引入路径，像下面的示例一样为引入路径添加版本后缀。

```go
module github.com/q1mi/hello/v2

go 1.16
```

把修改后的代码提交：

```bash
$ git add .
$ git commit -m "feat: SayHi现在支持给指定人打招呼啦"
$ git push
```

打好 tag 推送到远程仓库。

```bash
$ git tag -a v2.0.0 -m "release version v2.0.0"
$ git push origin v2.0.0
```

想要使用`v2`版本的代码包的用户只需按修改后的引入路径下载即可。

```bash
go get github.com/q1mi/hello/v2@v2.0.0
```

代码使用时需要注意引入路径要添加 v2 版本后缀。

```go
package main

import (
	"fmt"

	"github.com/q1mi/hello/v2" // 引入v2版本
)

func main() {
	fmt.Println("现在是假期时间...")

	hello.SayHi("张三") // v2版本的SayHi函数需要传入字符串参数
}
```

##### **废弃已发布版本**

使用`retract`声明废弃的版本。

例如我们在`hello/go.mod`文件中按如下方式声明即可对外废弃`v0.1.2`版本。

用户使用go get下载`v0.1.2`版本时就会收到提示，催促其升级到其他版本。

```go
module github.com/q1mi/hello

go 1.16

retract v0.1.2
```

#### go mod文件

记录了当前项目中所有依赖包的相关信息，声明依赖的格式如下：

```bash
require module/path v1.2.3
```

- require：声明依赖的关键字
- module/path：依赖包的引入路径
- v1.2.3：依赖包的版本号。支持以下几种格式：
  - latest：最新版本
  - v1.0.0：详细版本号
  - commit hash：指定某次commit hash

引入某些没有发布过`tag`版本标识的依赖包时，`go.mod`中记录的依赖版本信息就会出现类似`v0.0.0-20210218074646-139b0bcd549d`的格式，由版本号、commit时间和commit的hash值组成。

![image-20230929095915080](https://s2.loli.net/2023/10/28/mqWO1MT6tR23sDK.png)

##### **文件结构**

- `module`用来定义包名
- `require`用来定义依赖包及版本
- `indirect`表示间接引用
- `replace`使用国内包替换需要翻墙的包

#### go sum 文件

##### **目的**

为了防止依赖包被非法篡改，Go module 引入了`go.sum`机制来对依赖包进行校验。

详细记录了当前项目中引入的依赖包的信息及其hash 值。

`go.sum`文件内容通常是以类似下面的格式出现。

```go
<module> <version>/go.mod <hash>
```

或者

```go
<module> <version> <hash>
<module> <version>/go.mod <hash>
```

##### **分布式管理包**

不同于其他语言提供的基于中心的包管理机制，例如 npm 和 pypi等，Go并没有提供一个中央仓库来管理所有依赖包，而是采用分布式的方式来管理包。

#### 命令汇总

```bash
go mod download    下载依赖的module到本地cache（默认为$GOPATH/pkg/mod目录）
go mod edit        编辑go.mod文件
go mod graph       打印模块依赖图
go mod init        初始化当前文件夹, 创建go.mod文件
go mod tidy        增加缺少的module，删除无用的module
go mod vendor      将依赖复制到vendor下
go mod verify      校验依赖
go mod why         解释为什么需要依赖
```

#### 版本依赖

支持语义化版本号

- 比如`go get foo@v1.2.3`
- 可以跟git的分支或tag，比如`go get foo@master`
- 可以跟git提交哈希，比如`go get foo@e3702bed2`。

#### go mod edit

- 添加依赖项

  - ```bash
    go mod edit -require=golang.org/x/text
    ```

- 移除依赖项

  - ```bash
    go mod edit -droprequire=golang.org/x/text
    ```

### 导入本地包

- 同一目录下

  - 直接import

  - ```go
    import (
    	"moduledemo/mypackage"  // 导入同一项目下的mypackage包
    )
    ```

- 不同目录下

  - 在`go.mod`文件中使用`replace`指令。

  - ```go
    require "mypackage" v0.0.0
    replace "mypackage" => "../mypackage"
    ```

## 语言基础

### 变量

#### **声明**

```go
var 变量名 变量类型

var name string
var age int
var isOk bool

// 批量声明
var (
    a string
    b int
    c bool
    d float32
)
```

#### **变量初始化**

变量会被初始化成其类型的默认值

```go
var 变量名 类型 = 表达式

var name string = "Q1mi"
var age int = 18

var name, age = "Q1mi", 20
```

#### **类型自动推导**

```go
var name = "Q1mi"
var age = 18
```

#### **短变量声明（海象符）**

函数内部，使用更简略的 `:=` 方式声明并初始化变量。

注意：`:=`不能使用在函数外。

```go
func main() {
	n := 10
	m := 200 // 此处声明局部变量m
	fmt.Println(m, n)
}
```

#### **匿名变量**

忽略某个值，下划线`_`表示

不占用命名空间，不会分配内存，不存在重复声明

```go
	x, _ := foo()
	_, y := foo()
```

### 常量

恒定不变的值

定义的时候必须赋值

```go
const pi = 3.1415
const e = 2.7182

// 批量声明
const (
    pi = 3.1415
    e = 2.7182
)

// 批量声明时，若省略值，则表示和上面一行的值相同 （例子中全为100）
const (
    n1 = 100
    n2
    n3
)
```

**iota**

常量计数器

在const关键字出现时将被重置为0

const中每新增一行常量声明，将使`iota`计数一次 (iota可理解为**const语句块中的行索引**)。

**用处**：简化定义，定义枚举时有用

```go
const (
		n1 = iota //0
		n2        //1
		n3        //2
		n4        //3
	)


//使用_跳过某些值
const (
		n1 = iota //0
		n2        //1
		_
		n4        //3
	)

// 中间插队
const (
		n1 = iota //0
		n2 = 100  //100
		n3 = iota //2
		n4        //3
	)
const n5 = iota //0  const重新出现，重置为0

// 多个iota定义在一行
const (
		a, b = iota, iota 		  //0,0
		c, d                      //1,1
		e, f                      //2,2
	)
const (
		a, b = iota + 1, iota	  //1,0
		c, d                      //2,1
		e, f                      //3,2
	)
const (
		a, b = iota + 1, iota + 2 //1,2
		c, d                      //2,3
		e, f                      //3,4
	)
```

### 基本数据类型

#### **整型**

![image-20230915170949297](https://s2.loli.net/2023/10/28/OXsYuSZ96LwjzU4.png)

#### **特殊整型**

![image-20230915171029545](https://s2.loli.net/2023/10/28/vUFS413Vi96tcYI.png)

**注意：**使用时考虑`int`和`uint`可能在不同平台上的差异。

在涉及到二进制传输、读写文件的结构描述时，为了保持文件的结构不会受到不同编译目标平台字节长度的影响，不要使用`int`和 `uint`。

#### **数字字面量语法**

便于开发者以二进制、八进制或十六进制浮点数的格式定义数字

- `v := 0b00101101`， 代表二进制的 101101，相当于十进制的 45
- `v := 0o377`，代表八进制的 377，相当于十进制的 255。 
- `v := 0x1p-2`，代表十六进制的 1 除以 2²，也就是 0.25
- 允许用 `_` 分隔数字， `v := 123_456` 表示 v 的值等于 123456

```go
func main(){
	// 十进制
	var a int = 10
	fmt.Printf("%d \n", a)  // 10
	fmt.Printf("%b \n", a)  // 1010  占位符%b表示二进制
 
	// 八进制  以0开头
	var b int = 077
	fmt.Printf("%o \n", b)  // 77
 
	// 十六进制  以0x开头
	var c int = 0xff
	fmt.Printf("%x \n", c)  // ff
	fmt.Printf("%X \n", c)  // FF
}
```

#### 浮点型

支持两种浮点型数：`float32`和`float64`

```go
func main() {
        fmt.Printf("%f\n", math.Pi)
        fmt.Printf("%.2f\n", math.Pi)
}
```

#### 复数

complex64和complex128

```go
var c1 complex64
c1 = 1 + 2i
var c2 complex128
c2 = 2 + 3i
fmt.Println(c1)
fmt.Println(c2)
```

#### 布尔值

默认值为`false`

不允许将整型强制转换为布尔型.

无法参与数值运算，也无法与其他类型进行转换。

#### 字符串

内部实现使用`UTF-8`编码，**不能修改！！**

**转义符**

![image-20230915171845594](https://s2.loli.net/2023/10/28/YqDRHoMk482TCOa.png)

**多行字符串**

使用`反引号`字符

```go
s1 := `第一行
第二行
第三行
`
fmt.Println(s1)
```

**修改字符串**

需要先将其转换成`[]rune`或`[]byte`，完成后再转换为`string`。

无论哪种转换，都会重新分配内存，并复制字节数组。

```go
func changeString() {
	s1 := "big"
	// 强制类型转换
	byteS1 := []byte(s1)
	byteS1[0] = 'p'
	fmt.Println(string(byteS1))

	s2 := "白萝卜"
	runeS2 := []rune(s2)
	runeS2[0] = '红'
	fmt.Println(string(runeS2))
}
```

#### byte/rune

有以下两种

- `uint8`类型，或者叫 byte 型，代表一个`ASCII码`字符。
- `rune`类型，代表一个 `UTF-8字符`。
  - 当需要处理中文、日文或者其他复合字符时，则需要用到`rune`类型。`rune`类型实际是一个`int32`。
  - UTF8编码下一个中文汉字由3~4个字节组成，所以我们不能简单的按照字节去遍历一个包含中文的字符串
  - 一个rune字符由一个或多个byte组成。

```go
// 遍历字符串
func traversalString() {
	s := "hello沙河"
	for i := 0; i < len(s); i++ { //byte
		fmt.Printf("%v(%c) ", s[i], s[i])
	}
	fmt.Println()
	for _, r := range s { //rune
		fmt.Printf("%v(%c) ", r, r)
	}
	fmt.Println()
}

// 输出
104(h) 101(e) 108(l) 108(l) 111(o) 230(æ) 178(²) 153() 230(æ) 178(²) 179(³) 
104(h) 101(e) 108(l) 108(l) 111(o) 27801(沙) 27827(河) 
```

#### **类型转换**

只有强制类型转换，没有隐式类型转换

```go
T(表达式)  //强制类型转换的基本语法

func sqrtDemo() {
	var a, b = 3, 4
	var c int
	// math.Sqrt()接收的参数是float64类型，需要强制转换
	c = int(math.Sqrt(float64(a*a + b*b)))
	fmt.Println(c)
}
```

### 流程控制

#### if else

**基本写法**

```go
func ifDemo1() {
	score := 65
	if score >= 90 {
		fmt.Println("A")
	} else if score > 75 {
		fmt.Println("B")
	} else {
		fmt.Println("C")
	}
}
```

**if条件判断特殊写法**

作用域不同,特殊写法中声明的score只能在if判断内部使用

```go
func ifDemo2() {
	if score := 65; score >= 90 {
		fmt.Println("A")
	} else if score > 75 {
		fmt.Println("B")
	} else {
		fmt.Println("C")
	}
}
```

#### for

**基本写法**

```go
func forDemo() {
	for i := 0; i < 10; i++ {
		fmt.Println(i)
	}
}
```

**忽略初始值**

```go
func forDemo2() {
	i := 0
	for ; i < 10; i++ {
		fmt.Println(i)
	}
}
```

**类似while**

```go
func forDemo3() {
	i := 0
	for i < 10 {
		fmt.Println(i)
		i++
	}
}
```

**无限循环**

通过`break`、`goto`、`return`、`panic`语句强制退出循环。

```go
for {
    循环体语句
}
```

#### for range （键值循环）

使用`for range`遍历数组、切片、字符串、map 及通道（channel）。 

- 数组、切片、字符串返回索引和值。
- map返回键和值。
- 通道（channel）只返回通道内的值。

#### switch case

对大量的值进行条件判断。

每个`switch`只能有一个`default`分支。

```go
func switchDemo1() {
	finger := 3
	switch finger {
	case 1:
		fmt.Println("大拇指")
	case 2:
		fmt.Println("食指")
	case 3:
		fmt.Println("中指")
	case 4:
		fmt.Println("无名指")
	case 5:
		fmt.Println("小拇指")
	default:
		fmt.Println("无效的输入！")
	}
}
```

一个分支可以有多个值，多个case值中间使用英文逗号分隔。

```go
func testSwitch3() {
	switch n := 7; n {
	case 1, 3, 5, 7, 9:
		fmt.Println("奇数")
	case 2, 4, 6, 8:
		fmt.Println("偶数")
	default:
		fmt.Println(n)
	}
}
```

分支还可以使用表达式，这时候switch语句后面不需要再跟判断变量

```go
func switchDemo4() {
	age := 30
	switch {
	case age < 25:
		fmt.Println("好好学习吧")
	case age > 25 && age < 35:
		fmt.Println("好好工作吧")
	case age > 60:
		fmt.Println("好好享受吧")
	default:
		fmt.Println("活着真好")
	}
}
```

`fallthrough`语法可以执行满足条件的case的下一个case，是为了兼容C语言中的case设计的。

```go
func switchDemo5() {
	s := "a"
	switch {
	case s == "a":
		fmt.Println("a")
		fallthrough
	case s == "b":
		fmt.Println("b")
	case s == "c":
		fmt.Println("c")
	default:
		fmt.Println("...")
	}
}

// output
a
b
```

#### goto

通过标签进行代码间的无条件跳转

快速跳出循环、避免重复退出

使用`goto`语句能简化一些代码的实现过程

```go
func gotoDemo2() {
	for i := 0; i < 10; i++ {
		for j := 0; j < 10; j++ {
			if j == 2 {
				// 设置退出标签
				goto breakTag
			}
			fmt.Printf("%v-%v\n", i, j)
		}
	}
	return
	// 标签
breakTag:
	fmt.Println("结束for循环")
}
```

#### break

结束`for`、`switch`和`select`的代码块。

还可以在语句后面添加标签，表示退出某个标签对应的代码块，标签要求必须定义在对应的`for`、`switch`和 `select`的代码块上。

```go
func breakDemo1() {
BREAKDEMO1:
	for i := 0; i < 10; i++ {
		for j := 0; j < 10; j++ {
			if j == 2 {
				break BREAKDEMO1
			}
			fmt.Printf("%v-%v\n", i, j)
		}
	}
	fmt.Println("...")
}
```

#### continue

结束当前循环，开始下一次的循环迭代过程

仅限在`for`循环内使用。

在 `continue`语句后添加标签时，表示开始标签对应的循环。

```go
func continueDemo() {
forloop1:
	for i := 0; i < 5; i++ {
		// forloop2:
		for j := 0; j < 5; j++ {
			if i == 2 && j == 2 {
				continue forloop1
			}
			fmt.Printf("%v-%v\n", i, j)
		}
	}
}
```

### 数组 （array）

声明时就确定大小，使用时可以修改值，但是**大小不可变化！**

一旦定义，长度不能变。

长度必须是常量，并且**长度是数组类型的一部分。**

```go
var 数组变量名 [元素数量]T

var a [3]int
var b [4]int
a = b //不可以这样做，因为此时a和b是不同的类型
```

#### **初始化**

- 初始化列表来设置数组元素的值

  - ```go
    func main() {
    	var testArray [3]int                        //数组会初始化为int类型的零值
    	var numArray = [3]int{1, 2}                 //使用指定的初始值完成初始化
    	var cityArray = [3]string{"北京", "上海", "深圳"} //使用指定的初始值完成初始化
    	fmt.Println(testArray)                      //[0 0 0]
    	fmt.Println(numArray)                       //[1 2 0]
    	fmt.Println(cityArray)                      //[北京 上海 深圳]
    }
    ```

- 自行推断数组的长度

  - ```go
    func main() {
    	var testArray [3]int
    	var numArray = [...]int{1, 2}
    	var cityArray = [...]string{"北京", "上海", "深圳"}
    	fmt.Println(testArray)                          //[0 0 0]
    	fmt.Println(numArray)                           //[1 2]
    	fmt.Printf("type of numArray:%T\n", numArray)   //type of numArray:[2]int
    	fmt.Println(cityArray)                          //[北京 上海 深圳]
    	fmt.Printf("type of cityArray:%T\n", cityArray) //type of cityArray:[3]string
    }
    ```

- 指定索引值的方式来初始化数组

  - ```go
    func main() {
    	a := [...]int{1: 1, 3: 5}
    	fmt.Println(a)                  // [0 1 0 5]
    	fmt.Printf("type of a:%T\n", a) //type of a:[4]int
    }
    ```

#### **遍历方式**

```go
func main() {
	var a = [...]string{"北京", "上海", "深圳"}
	// 方法1：for循环遍历
	for i := 0; i < len(a); i++ {
		fmt.Println(a[i])
	}

	// 方法2：for range遍历
	for index, value := range a {
		fmt.Println(index, value)
	}
}
```

#### **多维数组**

多维数组**只有第一层**可以使用`...`来让编译器推导数组长度

```go
//支持的写法
a := [...][2]string{
	{"北京", "上海"},
	{"广州", "深圳"},
	{"成都", "重庆"},
}
//不支持多维数组的内层使用...
b := [3][...]string{
	{"北京", "上海"},
	{"广州", "深圳"},
	{"成都", "重庆"},
}
```

#### **数组是值类型**

- 支持 “==“、”!=” 操作符，因为内存总是被初始化过的。
- `[n]*T`表示指针数组，`*[n]T`表示数组指针 。

赋值和传参会复制整个数组。因此改变副本的值，不会改变本身的值。

```go
func modifyArray(x [3]int) {
	x[0] = 100
}

func modifyArray2(x [3][2]int) {
	x[2][0] = 100
}
func main() {
	a := [3]int{10, 20, 30}
	modifyArray(a) //在modify中修改的是a的副本x
	fmt.Println(a) //[10 20 30]
	b := [3][2]int{
		{1, 1},
		{1, 1},
		{1, 1},
	}
	modifyArray2(b) //在modify中修改的是b的副本x
	fmt.Println(b)  //[[1 1] [1 1] [1 1]]
}
```

### 切片 （slice）

数组有很多的局限性

切片（Slice）是一个拥有相同类型元素的可变长度的序列。它是基于数组类型做的一层封装。它非常灵活，支持自动扩容。

切片是一个引用类型，它的内部结构包含`地址`、`长度`和`容量`。切片一般用于快速地操作一块数据集合。

#### **声明**

```go
var name []T

// 例子
func main() {
	// 声明切片类型
	var a []string              //声明一个字符串切片
	var b = []int{}             //声明一个整型切片并初始化
	var c = []bool{false, true} //声明一个布尔切片并初始化
	var d = []bool{false, true} //声明一个布尔切片并初始化
	fmt.Println(a)              //[]
	fmt.Println(b)              //[]
	fmt.Println(c)              //[false true]
	fmt.Println(a == nil)       //true
	fmt.Println(b == nil)       //false
	fmt.Println(c == nil)       //false
	// fmt.Println(c == d)   //切片是引用类型，不支持直接比较，只能和nil比较
}
```

#### **长度和容量**

- `len()`函数求长度
- `cap()`函数求切片的容量

#### **切片表达式**

切片的底层就是一个数组

- 简单版

  - 可以基于数组通过切片表达式得到切片。得到的切片`长度=high-low`，容量等于得到的切片的底层数组的容量。

  - 个人理解：s的切片起始索引left到末尾元素个数即为cap。虽然s中不含有原数组的末尾一些元素，但如果再以切片s进行切片s2，则s2访问索引即为s的起始索引，并且s2的索引能够超过s的长度，访问原数组的值 **（未搞懂）**

  - ```go
    func main() {
    	a := [5]int{1, 2, 3, 4, 5}
    	s := a[1:3]  // s := a[low:high]
    	fmt.Printf("s:%v len(s):%v cap(s):%v\n", s, len(s), cap(s))
    }
    // output
    // s:[2 3] len(s):2 cap(s):4 
    
    a[2:]  // 等同于 a[2:len(a)]
    a[:3]  // 等同于 a[0:3]
    a[:]   // 等同于 a[0:len(a)]
    ```

  - ```go
    func main() {
    	a := [5]int{1, 2, 3, 4, 5}
    	s := a[1:3]  // s := a[low:high]
    	fmt.Printf("s:%v len(s):%v cap(s):%v\n", s, len(s), cap(s))
    	s2 := s[3:4]  // 索引的上限是cap(s)而不是len(s)
    	fmt.Printf("s2:%v len(s2):%v cap(s2):%v\n", s2, len(s2), cap(s2))
    }
    
    // output （未搞懂）
    s:[2 3] len(s):2 cap(s):4
    s2:[5] len(s2):1 cap(s2):1
    ```

- 完整版

  - 对于数组，指向数组的指针，或切片a(**注意不能是字符串**)支持完整切片表达式

  - 完整切片表达式需要满足的条件是`0 <= low <= high <= max <= cap(a)`，其他条件和简单切片表达式相同。

  - 简单来说：完整版的切片表达式就是自己手动限制了容量大小，如果不限制的话，默认直接取能取的最大值，也即切片的起始坐标到数组的末尾

  - ```go
    a[low : high : max]  // 结果切片的容量设置为max-low
    
    func main() {
    	a := [5]int{1, 2, 3, 4, 5}
    	t := a[1:3:5]
    	fmt.Printf("t:%v len(t):%v cap(t):%v\n", t, len(t), cap(t))
    }
    
    // output
    t:[2 3] len(t):2 cap(t):4
    ```

#### **make函数构造切片**

动态创建切片

```go
make([]T, size, cap)
```

- T:切片的元素类型
- size:切片中元素的数量
- cap:切片的容量

```go
func main() {
	a := make([]int, 2, 10)
	fmt.Println(a)      //[0 0]
	fmt.Println(len(a)) //2
	fmt.Println(cap(a)) //10
}
```

#### **切片本质**

本质是对底层数组的封装，包含了三个信息：

- **底层数组的指针** **（关键！）**
- 切片的长度（len）
- 切片的容量（cap）

![image-20230915211250486](https://s2.loli.net/2023/10/28/UpvV7JMrzNoQtCZ.png)

#### **判断为空**

始终使用`len(s) == 0`来判断，而不应该使用`s == nil`来判断。（切片的本质是指针！只要这个理解的透彻就好明白）

```go
var s1 []int         //len(s1)=0;cap(s1)=0;s1==nil
s2 := []int{}        //len(s2)=0;cap(s2)=0;s2!=nil
s3 := make([]int, 0) //len(s3)=0;cap(s3)=0;s3!=nil
```

#### **赋值拷贝**

拷贝前后两个变量共享底层数组，对一个切片的修改会影响另一个切片的内容。（切片的本质是指针！）

```go
func main() {
	s1 := make([]int, 3) //[0 0 0]
	s2 := s1             //将s1直接赋值给s2，s1和s2共用一个底层数组
	s2[0] = 100
	fmt.Println(s1) //[100 0 0]
	fmt.Println(s2) //[100 0 0]
}
```

#### **添加元素**

`append()`可以为切片动态添加元素。

- 可以一次添加一个元素
- 可以添加多个元素
- 也可以添加另一个切片中的元素（后面加…）

通过var声明的零值切片无需初始化便可直接使用`append()`函数

```go
func main(){
	var s []int
	s = append(s, 1)        // [1]
	s = append(s, 2, 3, 4)  // [1 2 3 4]
	s2 := []int{5, 6, 7}  
	s = append(s, s2...)    // [1 2 3 4 5 6 7]
}
```

#### **扩容**

切片会指向一个底层数组，容量够用就添加新增元素，不够用时则进行“扩容”（每次扩容后容量都是扩容前的2倍）。**此时该切片指向的底层数组就会更换！！**

“扩容”操作往往发生在`append()`函数调用时，所以我们通常都需要用原变量接收append函数的返回值。

```go
func main() {
	//append()添加元素和切片扩容
	var numSlice []int
	for i := 0; i < 10; i++ {
		numSlice = append(numSlice, i)
		fmt.Printf("%v  len:%d  cap:%d  ptr:%p\n", numSlice, len(numSlice), cap(numSlice), numSlice)
	}
}

[0]  len:1  cap:1  ptr:0xc0000a8000
[0 1]  len:2  cap:2  ptr:0xc0000a8040
[0 1 2]  len:3  cap:4  ptr:0xc0000b2020
[0 1 2 3]  len:4  cap:4  ptr:0xc0000b2020
[0 1 2 3 4]  len:5  cap:8  ptr:0xc0000b6000
[0 1 2 3 4 5]  len:6  cap:8  ptr:0xc0000b6000
[0 1 2 3 4 5 6]  len:7  cap:8  ptr:0xc0000b6000
[0 1 2 3 4 5 6 7]  len:8  cap:8  ptr:0xc0000b6000
[0 1 2 3 4 5 6 7 8]  len:9  cap:16  ptr:0xc0000b8000
[0 1 2 3 4 5 6 7 8 9]  len:10  cap:16  ptr:0xc0000b8000
```

#### **扩容策略**

依代码知扩容策略

- 首先判断，如果新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）就是新申请的容量（cap）。
- 否则判断，如果旧切片的长度小于1024，则最终容量(newcap)就是旧容量(old.cap)的两倍，即（newcap=doublecap），
- 否则判断，如果旧切片长度大于等于1024，则最终容量（newcap）从旧容量（old.cap）开始循环增加原来的1/4，即（newcap=old.cap,for {newcap += newcap/4}）直到最终容量（newcap）大于等于新申请的容量(cap)，即（newcap >= cap）
- 如果最终容量（cap）计算值溢出，则最终容量（cap）就是新申请容量（cap）。

需要注意的是，切片扩容还会根据切片中元素的类型不同而做不同的处理，比如`int`和`string`类型的处理方式就不一样。

```go
newcap := old.cap
doublecap := newcap + newcap
if cap > doublecap {
	newcap = cap
} else {
	if old.len < 1024 {
		newcap = doublecap
	} else {
		// Check 0 < newcap to detect overflow
		// and prevent an infinite loop.
		for 0 < newcap && newcap < cap {
			newcap += newcap / 4
		}
		// Set newcap to the requested cap when
		// the newcap calculation overflowed.
		if newcap <= 0 {
			newcap = cap
		}
	}
}
```

#### **复制切片**

切片是引用类型，赋值b:=a使得a和b指向了同一块内存地址。修改b的同时a的值也会发生变化。

`copy()`函数可以迅速地将一个切片的数据复制到另外一个切片空间中

```bash
copy(destSlice, srcSlice []T)
```

- srcSlice: 数据来源切片
- destSlice: 目标切片

```go
func main() {
	// copy()复制切片
	a := []int{1, 2, 3, 4, 5}
	c := make([]int, 5, 5)
	copy(c, a)     //使用copy()函数将切片a中的元素复制到切片c
	fmt.Println(a) //[1 2 3 4 5]
	fmt.Println(c) //[1 2 3 4 5]
	c[0] = 1000
	fmt.Println(a) //[1 2 3 4 5]
	fmt.Println(c) //[1000 2 3 4 5]
}
```

#### **删除元素**

并没有删除切片元素的专用方法，我们可以使用切片本身的特性来删除元素。 

要从切片a中删除索引为`index`的元素，操作方法是`a = append(a[:index], a[index+1:]...)`

```go
func main() {
	// 从切片中删除元素
	a := []int{30, 31, 32, 33, 34, 35, 36, 37}
	// 要删除索引为2的元素
	a = append(a[:2], a[3:]...)
	fmt.Println(a) //[30 31 33 34 35 36 37]
}
```

### map

一种无序的基于`key-value`的数据结构

map是引用类型，必须初始化才能使用

```go
map[KeyType]ValueType
```

- KeyType:表示键的类型。
- ValueType:表示键对应的值的类型。

map类型的变量默认初始值为nil，需要使用make()函数来分配内存

```go
make(map[KeyType]ValueType, [cap])
```

##### **基本使用**

```go
func main() {
	scoreMap := make(map[string]int, 8)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	fmt.Println(scoreMap)
	fmt.Println(scoreMap["小明"])
	fmt.Printf("type of a:%T\n", scoreMap)
}

map[小明:100 张三:90]
100
type of a:map[string]int
```

支持在声明的时候填充元素

```go
func main() {
	userInfo := map[string]string{
		"username": "沙河小王子",
		"password": "123456",
	}
	fmt.Println(userInfo)
}
```

##### **判断某个键是否存在**

```go
value, ok := map[key]

// 举例
func main() {
	scoreMap := make(map[string]int)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	// 如果key存在ok为true,v为对应的值；不存在ok为false,v为值类型的零值
	v, ok := scoreMap["张三"]
	if ok {
		fmt.Println(v)
	} else {
		fmt.Println("查无此人")
	}
}
```

##### **遍历**

使用`for range`遍历map

```go
func main() {
	scoreMap := make(map[string]int)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	scoreMap["娜扎"] = 60
	for k, v := range scoreMap {
		fmt.Println(k, v)
	}
}
```

只想遍历key的时候

```go
func main() {
	scoreMap := make(map[string]int)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	scoreMap["娜扎"] = 60
	for k := range scoreMap {
		fmt.Println(k)
	}
}
```

##### **删除键值对**

使用`delete()`内建函数从map中删除一组键值对

```go
delete(map, key)
```

- map:表示要删除键值对的map
- key:表示要删除的键值对的键

```go
func main(){
	scoreMap := make(map[string]int)
	scoreMap["张三"] = 90
	scoreMap["小明"] = 100
	scoreMap["娜扎"] = 60
	delete(scoreMap, "小明")//将小明:100从map中删除
	for k,v := range scoreMap{
		fmt.Println(k, v)
	}
}
```

##### **元素为map类型的切片**

```go
func main() {
	var mapSlice = make([]map[string]string, 3)
	for index, value := range mapSlice {
		fmt.Printf("index:%d value:%v\n", index, value)
	}
	fmt.Println("after init")
	// 对切片中的map元素进行初始化
	mapSlice[0] = make(map[string]string, 10)
	mapSlice[0]["name"] = "小王子"
	mapSlice[0]["password"] = "123456"
	mapSlice[0]["address"] = "沙河"
	for index, value := range mapSlice {
		fmt.Printf("index:%d value:%v\n", index, value)
	}
}

index:0 value:map[]
index:1 value:map[]
index:2 value:map[]
after init
index:0 value:map[address:沙河 name:小王子 password:123456]
index:1 value:map[]
index:2 value:map[]
```

##### **值为切片类型的map**

```go
func main() {
	var sliceMap = make(map[string][]string, 3)
	fmt.Println(sliceMap)
	fmt.Println("after init")
	key := "中国"
	value, ok := sliceMap[key]
	if !ok {
		value = make([]string, 0, 2)
	}
	value = append(value, "北京", "上海")
	sliceMap[key] = value
	fmt.Println(sliceMap)
}


map[]
after init
map[中国:[北京 上海]]
```

##### 思考题

观察下面代码，写出最终的打印结果。

```go
func main() {
	type Map map[string][]int
	m := make(Map)
	s := []int{1, 2}
	s = append(s, 3)
	fmt.Printf("%+v\n", s)
	m["q1mi"] = s
	s = append(s[:1], s[2:]...)
	fmt.Printf("%+v\n", s)
	fmt.Printf("%+v\n", m["q1mi"])
}

[1 2 3]
[1 3]
[1 3 3]
```

**原因分析**：删除切片中的元素其实是把删除元素后面的元素向前一位复制一份。切片的长度len再减一，map[“q1mi”]与slice s指针指向的位置相同，但s长度的长度少1，所以会比map[“q1mi”]少一个元素3。

### 函数

Go语言中支持函数、匿名函数和闭包，并且函数在Go语言中属于“一等公民”。

```go
func 函数名(参数)(返回值){
    函数体
}
```

- 函数名：由字母、数字、下划线组成。但函数名的第一个字母不能是数字。在同一个包内，函数名也称不能重名。
- 参数：参数由参数变量和参数变量的类型组成，多个参数之间使用`,`分隔。
- 返回值：返回值由返回值变量和其变量类型组成，也可以只写返回值的类型，多个返回值必须用`()`包裹，并用`,`分隔。
- 函数体：实现指定功能的代码块。

调用有返回值的函数时，可以不接收其返回值。

##### **参数简写**

相邻变量的类型相同，则可以省略类型

```go
func intSum(x, y int) int {
	return x + y
}
```

##### **可变参数**

参数数量不固定。通过在参数名后加`...`来标识。通常要作为函数的最后一个参数。

**本质上，**函数的可变参数是通过切片来实现的。

```go
func intSum3(x int, y ...int) int {
	fmt.Println(x, y)
	sum := x
	for _, v := range y {
		sum = sum + v
	}
	return sum
}

ret5 := intSum3(100)
ret6 := intSum3(100, 10)
ret7 := intSum3(100, 10, 20)
ret8 := intSum3(100, 10, 20, 30)
fmt.Println(ret5, ret6, ret7, ret8) //100 110 130 160
```

**返回值**

##### **多返回值**

必须用`()`将所有返回值包裹起来

```go
func calc(x, y int) (int, int) {
	sum := x + y
	sub := x - y
	return sum, sub
}
```

##### **返回值命名**

可以给返回值命名，并在函数体中直接使用这些变量，最后通过`return`关键字返回。

```go
func calc(x, y int) (sum, sub int) {
	sum = x + y
	sub = x - y
	return
}
```

**函数类型与变量**

##### **定义函数类型**

使用`type`关键字来定义一个函数类型

```go
type calculation func(int, int) int
```

凡是满足这个条件的函数都是calculation类型的函数，例如：

```go
func add(x, y int) int {
	return x + y
}

func sub(x, y int) int {
	return x - y
}
```

##### **函数类型变量**

声明函数类型的变量并且为该变量赋值：

```go
func main() {
	var c calculation               // 声明一个calculation类型的变量c
	c = add                         // 把add赋值给c
	fmt.Printf("type of c:%T\n", c) // type of c:main.calculation
	fmt.Println(c(1, 2))            // 像调用add一样调用c

	f := add                        // 将函数add赋值给变量f
	fmt.Printf("type of f:%T\n", f) // type of f:func(int, int) int
	fmt.Println(f(10, 20))          // 像调用add一样调用f
}
```

##### **函数作为参数**

```go
func add(x, y int) int {
	return x + y
}
func calc(x, y int, op func(int, int) int) int {
	return op(x, y)
}
func main() {
	ret2 := calc(10, 20, add)
	fmt.Println(ret2) //30
}
```

##### **函数作为返回值**

```go
func do(s string) (func(int, int) int, error) {
	switch s {
	case "+":
		return add, nil
	case "-":
		return sub, nil
	default:
		err := errors.New("无法识别的操作符")
		return nil, err
	}
}
```

##### **匿名函数**

没有函数名的函数，多用于实现回调函数和闭包。

```go
func(参数)(返回值){
    函数体
}
```

需要保存到某个变量或者作为立即执行函数:

```go
func main() {
	// 将匿名函数保存到变量
	add := func(x, y int) {
		fmt.Println(x + y)
	}
	add(10, 20) // 通过变量调用匿名函数

	//自执行函数：匿名函数定义完加()直接执行
	func(x, y int) {
		fmt.Println(x + y)
	}(10, 20)
}
```

##### **闭包**

一个函数和与其相关的引用环境组合而成的实体。简单来说，`闭包=函数+引用环境`。

例1：

```go
func adder() func(int) int {
	var x int
	return func(y int) int {
		x += y
		return x
	}
}
func main() {
	var f = adder()
	fmt.Println(f(10)) //10
	fmt.Println(f(20)) //30
	fmt.Println(f(30)) //60

	f1 := adder()
	fmt.Println(f1(40)) //40
	fmt.Println(f1(50)) //90
}

// `f`是函数，它引用了`x`变量，此时`f`就是一个闭包。 在`f`的生命周期内，变量`x`也一直有效。
```

例2：

```go
func makeSuffixFunc(suffix string) func(string) string {
	return func(name string) string {
		if !strings.HasSuffix(name, suffix) {
			return name + suffix
		}
		return name
	}
}

func main() {
	jpgFunc := makeSuffixFunc(".jpg")
	txtFunc := makeSuffixFunc(".txt")
	fmt.Println(jpgFunc("test")) //test.jpg
	fmt.Println(txtFunc("test")) //test.txt
}
```

例3：

```go
func calc(base int) (func(int) int, func(int) int) {
	add := func(i int) int {
		base += i
		return base
	}

	sub := func(i int) int {
		base -= i
		return base
	}
	return add, sub
}

func main() {
	f1, f2 := calc(10)
	fmt.Println(f1(1), f2(2)) //11 9
	fmt.Println(f1(3), f2(4)) //12 8
	fmt.Println(f1(5), f2(6)) //13 7
}
```

牢记`闭包=函数+引用环境`！这样就很好理解了

##### **defer**

`defer`语句会将其后面跟随的语句进行延迟处理，在`defer`归属的函数即将返回（return）时，将延迟处理的语句按`defer`定义的**逆序进行执行**

被`defer`的语句最后被执行，最后被`defer`的语句，最先被执行。（类似 堆栈）

**defer注册要延迟执行的函数时该函数所有的参数都需要确定其值！**

延迟调用的特性，使其能非常方便的处理资源释放问题。比如：资源清理、文件关闭、解锁及记录时间等。

```go
func main() {
	fmt.Println("start")
	defer fmt.Println(1)
	defer fmt.Println(2)
	defer fmt.Println(3)
	fmt.Println("end")
}

start
end
3
2
1
```

**执行时机**

`return`语句在底层并不是原子操作，它分为给返回值赋值和RET指令两步。

`defer`语句执行的时机就在返回值赋值操作后，RET指令执行前。

![image-20230916172452001](https://s2.loli.net/2023/10/28/21Z7DCPmtpsQVRA.png)

**思考题**

思考运行结果

```go
func f1() int {
	x := 5
	defer func() {
		x++
	}()
	return x
}

func f2() (x int) {
	defer func() {
		x++
	}()
	return 5
}

func f3() (y int) {
	x := 5
	defer func() {
		x++
	}()
	return x
}
func f4() (x int) {
	defer func(x int) {
		x++
	}(x)
	return 5
}
func main() {
	fmt.Println(f1())
	fmt.Println(f2())
	fmt.Println(f3())
	fmt.Println(f4())
}

5 // 将x=5 赋值给匿名返回变量ret，而后defer x=6，返回ret=5
6 // 赋值x=5，defer x=6 ，返回x=6
5 // 赋值y=x=5，defer x = 6，返回y=5
5 // defer注册要延迟执行的函数时该函数所有的参数都需要确定其值，先执行defer，x为值传递，实际值没变化仍为0，赋值x=5,返回x=5
```

**面试题**

```go
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
	x := 1
	y := 2
	defer calc("AA", x, calc("A", x, y))
	x = 10
	defer calc("BB", x, calc("B", x, y))
	y = 20
}

// defer注册要延迟执行的函数时该函数所有的参数都需要确定其值
A 1 2 3
B 10 2 12
BB 10 12 22
AA 1 3 4

```

##### **内置函数**

![image-20230916200559562](https://s2.loli.net/2023/10/28/jgC1clnEeFpYwuk.png)

##### **panic/recover**

使用`panic/recover`模式来处理错误。 `panic`可以在任何地方引发，但`recover`只有在`defer`调用的函数中有效。

```go
func funcA() {
	fmt.Println("func A")
}

func funcB() {
	panic("panic in B")
}

func funcC() {
	fmt.Println("func C")
}
func main() {
	funcA()
	funcB()
	funcC()
}


func A
panic: panic in B

goroutine 1 [running]:
main.funcB(...)
        .../code/func/main.go:12
main.main()
        .../code/func/main.go:20 +0x98
```

`panic`导致程序崩溃，异常退出。可以通过`recover`将程序恢复回来，继续往后执行。

```go
func funcA() {
	fmt.Println("func A")
}

func funcB() {
	defer func() {
		err := recover()
		//如果程序出出现了panic错误,可以通过recover恢复过来
		if err != nil {
			fmt.Println("recover in B")
		}
	}()
	panic("panic in B")
}

func funcC() {
	fmt.Println("func C")
}
func main() {
	funcA()
	funcB()
	funcC()
}

func A
recover in B
func C
```

**注意**：

1. `recover()`必须搭配`defer`使用。
2. `defer`一定要在可能引发`panic`的语句之前定义。

### 指针

Go语言中的指针不能进行偏移和运算，是安全指针。

两个关键符号：

- `&`（取地址）
- `*`（根据地址取值）

##### **指针地址和指针类型**

指针变量：保存一个数据在内存中的地址

取变量指针的语法如下：

```go
ptr := &v    // v的类型为T
```

- v:代表被取地址的变量，类型为`T`
- ptr:用于接收地址的变量，ptr的类型就为`*T`，称做T的指针类型。*代表指针。

```go
func main() {
	a := 10
	b := &a
	fmt.Printf("a:%d ptr:%p\n", a, &a) // a:10 ptr:0xc00001a078
	fmt.Printf("b:%p type:%T\n", b, b) // b:0xc00001a078 type:*int
	fmt.Println(&b)                    // 0xc00000e018
}
```

![image-20230918191638370](https://s2.loli.net/2023/10/28/3FlbgVXpcI54U9v.png)

##### **指针取值**

- `&`（取地址）
- `*`（根据地址取值）

```go
func main() {
	//指针取值
	a := 10
	b := &a // 取变量a的地址，将指针保存到b中
	fmt.Printf("type of b:%T\n", b)
	c := *b // 指针取值（根据指针去内存取值）
	fmt.Printf("type of c:%T\n", c)
	fmt.Printf("value of c:%v\n", c)
}

type of b:*int
type of c:int
value of c:10
```

变量、指针地址、指针变量、取地址、取值的相互关系和特性如下：

- 对变量进行取地址（&）操作，可以获得这个变量的指针变量。
- 指针变量的值是指针地址。
- 对指针变量进行取值（*）操作，可以获得指针变量指向的原变量的值。

```go
func modify1(x int) {
	x = 100
}

func modify2(x *int) {
	*x = 100
}

func main() {
	a := 10
	modify1(a)
	fmt.Println(a) // 10
	modify2(&a)
	fmt.Println(a) // 100
}
```

##### **new和make**

Go语言中对于引用类型的变量，在使用的时候不仅要声明它，还要为它分配内存空间，否则我们的值就没办法存储。

对于值类型的声明不需要分配内存空间，是因为它们在声明的时候已经默认分配好了内存空间。

new和make是内建的两个函数，主要用来分配内存。

###### **new**

```go
func new(Type) *Type
```

- Type表示类型，new函数只接受一个参数，这个参数是一个类型
- *Type表示类型指针，new函数返回一个指向该类型内存地址的指针。

（不太常用）得到的是一个类型的指针，并且该指针对应的值为该类型的零值。

```go
func main() {
	a := new(int)
	b := new(bool)
	fmt.Printf("%T\n", a) // *int
	fmt.Printf("%T\n", b) // *bool
	fmt.Println(*a)       // 0
	fmt.Println(*b)       // false
}	
```

按照如下方式使用内置的new函数对a进行初始化之后就可以正常对其赋值了：

```go
func main() {
	var a *int
	a = new(int)
	*a = 10
	fmt.Println(*a)
}
```

###### **make**

只用于slice、map以及channel的内存创建，它返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以就没有必要返回他们的指针了。

```go
func make(t Type, size ...IntegerType) Type
```

使用slice、map以及channel的时候，都需要使用make进行初始化

```go
func main() {
	var b map[string]int
	b = make(map[string]int, 10)
	b["沙河娜扎"] = 100
	fmt.Println(b)
}
```

**注意：**

当使用 `make` 函数时，如果不指定 `len` 和 `cap` 参数，它会分配一个默认的容量，这个容量会根据 map 类型的大小和性能进行动态调整。

Go 语言的运行时系统会自动处理 map 的扩容。

尽管不指定 `len` 和 `cap` 参数是合法的，但若提供 `len` 参数来分配更接近预期容量的 map，以减少后续的扩容操作提高效率。

###### 区别

1. 二者都是用来做内存分配的。
2. make只用于slice、map以及channel的初始化，返回的还是这三个引用类型本身；
3. 而new用于类型的内存分配，并且内存对应的值为类型零值，返回的是指向类型的指针。

### 类型别名和自定义类型

##### **自定义类型**

可以使用`type`关键字来定义自定义类型，一个全新的类型。

可以基于内置的基本类型定义，也可以通过struct定义。

```go
//将MyInt定义为int类型
type MyInt int
```

##### **类型别名**

就像一个孩子小时候有小名、乳名，上学后用学名，英语老师又会给他起英文名，但这些名字都指的是他本人。

```go
type TypeAlias = Type
```

之前见过的`rune`和`byte`就是类型别名

```go
type byte = uint8
type rune = int32
```

##### **类型定义和类型别名的区别**

类型别名只会在代码中存在，编译完成时并不会有

```go
//类型定义
type NewInt int

//类型别名
type MyInt = int

func main() {
	var a NewInt
	var b MyInt
	
	fmt.Printf("type of a:%T\n", a) //type of a:main.NewInt
	fmt.Printf("type of b:%T\n", b) //type of b:int
}
```

### 结构体

##### **定义**

提供了一种自定义数据类型，可以封装多个基本数据类型，这种数据类型叫结构体，英文名称`struct`。

通过`struct`来实现面向对象。

使用`type`和`struct`关键字来定义结构体

```go
type 类型名 struct {
    字段名 字段类型
    字段名 字段类型
    …
}
```

- 类型名：标识自定义结构体的名称，在同一个包内不能重复。
- 字段名：表示结构体字段名。结构体中的字段名必须唯一。
- 字段类型：表示结构体字段的具体类型。

##### **实例化**

实例化时，才会真正地分配内存。

结构体本身也是一种类型，我们可以像声明内置类型一样使用`var`关键字声明结构体类型。

```go
var 结构体实例 结构体类型
```

```go
type person struct {
	name string
	city string
	age  int8
}

func main() {
	var p1 person
	p1.name = "沙河娜扎"
	p1.city = "北京"
	p1.age = 18
	fmt.Printf("p1=%v\n", p1)  //p1={沙河娜扎 北京 18}
	fmt.Printf("p1=%#v\n", p1) //p1=main.person{name:"沙河娜扎", city:"北京", age:18}
}
```

通过`.`来访问结构体的字段（成员变量）

##### **地址实例化**

使用`&`对结构体进行取地址操作相当于对该结构体类型进行了一次`new`实例化操作。

Go语言中支持对结构体指针直接使用`.`来访问结构体的成员！！！

`p3.name = "七米"`其实在底层是`(*p3).name = "七米"`，这是Go语言帮我们实现的语法糖。

```go
p3 := &person{}
fmt.Printf("%T\n", p3)     //*main.person
fmt.Printf("p3=%#v\n", p3) //p3=&main.person{name:"", city:"", age:0}
p3.name = "七米"
p3.age = 30
p3.city = "成都"
fmt.Printf("p3=%#v\n", p3) //p3=&main.person{name:"七米", city:"成都", age:30}
```

##### **匿名结构体**

定义一些临时数据结构等场景

```go
func main() {
    var user struct{Name string; Age int}
    user.Name = "小王子"
    user.Age = 18
    fmt.Printf("%#v\n", user)
}
```

##### **指针类型结构体**

通过使用`new`关键字对结构体进行实例化，得到的是结构体的地址

Go语言中支持对结构体指针直接使用`.`来访问结构体的成员！！！

```go
var p2 = new(person)
p2.name = "小王子"
p2.age = 28
p2.city = "上海"
fmt.Printf("p2=%#v\n", p2) //p2=&main.person{name:"小王子", city:"上海", age:28}
```

##### **初始化**

没有初始化的结构体，其成员变量都是对应其类型的零值。

```go
type person struct {
	name string
	city string
	age  int8
}

func main() {
	var p4 person
	fmt.Printf("p4=%#v\n", p4) //p4=main.person{name:"", city:"", age:0}
}
```

##### **键值对初始化**

```go
p5 := person{
	name: "小王子",
	city: "北京",
	age:  18,
}
fmt.Printf("p5=%#v\n", p5) //p5=main.person{name:"小王子", city:"北京", age:18}

p6 := &person{
	name: "小王子",
	city: "北京",
	age:  18,
}
fmt.Printf("p6=%#v\n", p6) //p6=&main.person{name:"小王子", city:"北京", age:18}
```

没有初始值的时候，该字段可以不写。此时，没有指定的就是零值。

```go
p7 := &person{
	city: "北京",
}
fmt.Printf("p7=%#v\n", p7) //p7=&main.person{name:"", city:"北京", age:0}
```

##### **值的列表初始化**

简写，初始化的时候不写键，直接写值：

```go
p8 := &person{
	"沙河娜扎",
	"北京",
	28,
}
fmt.Printf("p8=%#v\n", p8) //p8=&main.person{name:"沙河娜扎", city:"北京", age:28}
```

需要注意：

1. 必须初始化结构体的所有字段。
2. 初始值的填充顺序必须与字段在结构体中的声明顺序一致。
3. 该方式不能和键值初始化方式混用。

##### **内存布局**

结构体占用一块连续的内存。

```go
type test struct {
	a int8
	b int8
	c int8
	d int8
}
n := test{
	1, 2, 3, 4,
}
fmt.Printf("n.a %p\n", &n.a)
fmt.Printf("n.b %p\n", &n.b)
fmt.Printf("n.c %p\n", &n.c)
fmt.Printf("n.d %p\n", &n.d)

n.a 0xc0000a0060
n.b 0xc0000a0061
n.c 0xc0000a0062
n.d 0xc0000a0063
```

##### **空结构体**

不占用空间

```go
var v struct{}
fmt.Println(unsafe.Sizeof(v))  // 0
```

##### **构造函数**

结构体没有构造函数，但可以自己实现。

结构体比较复杂的话，值拷贝性能开销会比较大，所以构造函数返回的可以是结构体指针类型。

```go
func newPerson(name, city string, age int8) *person {
	return &person{
		name: name,
		city: city,
		age:  age,
	}
}

p9 := newPerson("张三", "沙河", 90)
fmt.Printf("%#v\n", p9) //&main.person{name:"张三", city:"沙河", age:90}
```

##### **方法和接收者**

`方法（Method）`是一种作用于特定类型变量的函数。

特定类型变量叫做`接收者（Receiver）`。类似于其他语言中的`this`或者 `self`。

**方法与函数的区别**：函数不属于任何类型，方法属于特定的类型。

```go
func (接收者变量 接收者类型) 方法名(参数列表) (返回参数) {
    函数体
}
```

- 接收者变量：接收者中的参数变量名在命名时，官方建议使用接收者类型名称首字母的小写，而不是`self`、`this`之类的命名。例如，`Person`类型的接收者变量应该命名为 `p`，`Connector`类型的接收者变量应该命名为`c`等。
- 接收者类型：接收者类型和参数类似，可以是指针类型和非指针类型。
- 方法名、参数列表、返回参数：具体格式与函数定义相同。

```go
//Person 结构体
type Person struct {
	name string
	age  int8
}

//NewPerson 构造函数
func NewPerson(name string, age int8) *Person {
	return &Person{
		name: name,
		age:  age,
	}
}

//Dream Person做梦的方法
func (p Person) Dream() {
	fmt.Printf("%s的梦想是学好Go语言！\n", p.name)
}

func main() {
	p1 := NewPerson("小王子", 25)
	p1.Dream()
}
```

##### **指针类型的接收者**

接收者由一个结构体的指针组成，调用方法时修改接收者指针的任意成员变量，在方法结束后，修改都是有效的。

```go
// SetAge 设置p的年龄
// 使用指针接收者
func (p *Person) SetAge(newAge int8) {
	p.age = newAge
}

func main() {
	p1 := NewPerson("小王子", 25)
	fmt.Println(p1.age) // 25
	p1.SetAge(30)
	fmt.Println(p1.age) // 30
}
```

**什么时候应该使用指针类型接收者**

- 需要修改接收者中的值
- 接收者是拷贝代价比较大的大对象
- 保证一致性，如果有某个方法使用了指针接收者，那么其他的方法也应该使用指针接收者。

##### **值类型的接收者**

值拷贝，可以获取接收者的成员值，但修改操作只是针对副本，无法修改接收者变量本身。

```go
// SetAge2 设置p的年龄
// 使用值接收者
func (p Person) SetAge2(newAge int8) {
	p.age = newAge
}

func main() {
	p1 := NewPerson("小王子", 25)
	p1.Dream()
	fmt.Println(p1.age) // 25
	p1.SetAge2(30) // (*p1).SetAge2(30)
	fmt.Println(p1.age) // 25
}
```

##### **方法与引用数据类型的坑**

因为slice和map这两种数据类型都包含了指向底层数据的指针，因此我们在需要复制它们时要特别注意。

```go
type Person struct {
	name   string
	age    int8
	dreams []string
}

func (p *Person) SetDreams(dreams []string) {
	p.dreams = dreams
}

func main() {
	p1 := Person{name: "小王子", age: 18}
	data := []string{"吃饭", "睡觉", "打豆豆"}
	p1.SetDreams(data)

	// 你真的想要修改 p1.dreams 吗？
	data[1] = "不睡觉"
	fmt.Println(p1.dreams)  // ?
}
```

正确的做法是在方法中使用传入的slice的拷贝进行结构体赋值。

```go
func (p *Person) SetDreams(dreams []string) {
	p.dreams = make([]string, len(dreams))
	copy(p.dreams, dreams)
}
```

##### **任意类型添加方法**

接收者的类型可以是任何类型，不仅仅是结构体，任何类型都可以拥有方法。

**注意事项：** 非本地类型不能定义方法，也就是说我们不能给别的包的类型定义方法。

```go
//MyInt 将int定义为自定义MyInt类型
type MyInt int

//SayHello 为MyInt添加一个SayHello的方法
func (m MyInt) SayHello() {
	fmt.Println("Hello, 我是一个int。")
}
func main() {
	var m1 MyInt
	m1.SayHello() //Hello, 我是一个int。
	m1 = 100
	fmt.Printf("%#v  %T\n", m1, m1) //100  main.MyInt
}
```

##### **匿名字段**

结构体允许其成员字段在声明时没有字段名而只有类型

并不代表没有字段名，而是默认会采用类型名作为字段名。字段名称必须唯一，因此一个结构体中同种类型的匿名字段只能有一个。

```go
//Person 结构体Person类型
type Person struct {
	string
	int
}

func main() {
	p1 := Person{
		"小王子",
		18,
	}
	fmt.Printf("%#v\n", p1)        //main.Person{string:"北京", int:18}
	fmt.Println(p1.string, p1.int) //北京 18
}
```

##### **嵌套结构体**

```go
//Address 地址结构体
type Address struct {
	Province string
	City     string
}

//User 用户结构体
type User struct {
	Name    string
	Gender  string
	Address Address
}

func main() {
	user1 := User{
		Name:   "小王子",
		Gender: "男",
		Address: Address{
			Province: "山东",
			City:     "威海",
		},
	}
	fmt.Printf("user1=%#v\n", user1)//user1=main.User{Name:"小王子", Gender:"男", Address:main.Address{Province:"山东", City:"威海"}}
}
```

##### **字段名冲突**

嵌套结构体内部可能存在相同的字段名。在这种情况下为了避免歧义需要通过指定具体的内嵌结构体字段名。

```go
//Address 地址结构体
type Address struct {
	Province   string
	City       string
	CreateTime string
}

//Email 邮箱结构体
type Email struct {
	Account    string
	CreateTime string
}

//User 用户结构体
type User struct {
	Name   string
	Gender string
	Address
	Email
}

func main() {
	var user3 User
	user3.Name = "沙河娜扎"
	user3.Gender = "男"
	// user3.CreateTime = "2019" //ambiguous selector user3.CreateTime
	user3.Address.CreateTime = "2000" //指定Address结构体中的CreateTime
	user3.Email.CreateTime = "2000"   //指定Email结构体中的CreateTime
}
```

##### **嵌套匿名字段**

访问结构体成员时会先在结构体中查找该字段，找不到再去嵌套的匿名字段中查找。

```go
//Address 地址结构体
type Address struct {
	Province string
	City     string
}

//User 用户结构体
type User struct {
	Name    string
	Gender  string
	Address //匿名字段
}

func main() {
	var user2 User
	user2.Name = "小王子"
	user2.Gender = "男"
	user2.Address.Province = "山东"    // 匿名字段默认使用类型名作为字段名
	user2.City = "威海"                // 匿名字段可以省略
	fmt.Printf("user2=%#v\n", user2) //user2=main.User{Name:"小王子", Gender:"男", Address:main.Address{Province:"山东", City:"威海"}}
}
```

##### **“继承”**

```go
//Animal 动物
type Animal struct {
	name string
}

func (a *Animal) move() {
	fmt.Printf("%s会动！\n", a.name)
}

//Dog 狗
type Dog struct {
	Feet    int8
	*Animal //通过嵌套匿名结构体实现继承
}

func (d *Dog) wang() {
	fmt.Printf("%s会汪汪汪~\n", d.name)
}

func main() {
	d1 := &Dog{
		Feet: 4,
		Animal: &Animal{ //注意嵌套的是结构体指针
			name: "乐乐",
		},
	}
	d1.wang() //乐乐会汪汪汪~
	d1.move() //乐乐会动！
}
```

##### **字段的可见性**

大写开头表示可公开访问，小写表示私有（仅在定义当前结构体的包中可访问）。

##### **JSON序列化**

```go
//Student 学生
type Student struct {
	ID     int
	Gender string
	Name   string
}

//Class 班级
type Class struct {
	Title    string
	Students []*Student
}

func main() {
	c := &Class{
		Title:    "101",
		Students: make([]*Student, 0, 200),
	}
	for i := 0; i < 10; i++ {
		stu := &Student{
			Name:   fmt.Sprintf("stu%02d", i),
			Gender: "男",
			ID:     i,
		}
		c.Students = append(c.Students, stu)
	}
	//JSON序列化：结构体-->JSON格式的字符串
	data, err := json.Marshal(c)
	if err != nil {
		fmt.Println("json marshal failed")
		return
	}
	fmt.Printf("json:%s\n", data)
	//JSON反序列化：JSON格式的字符串-->结构体
	str := `{"Title":"101","Students":[{"ID":0,"Gender":"男","Name":"stu00"},{"ID":1,"Gender":"男","Name":"stu01"},{"ID":2,"Gender":"男","Name":"stu02"},{"ID":3,"Gender":"男","Name":"stu03"},{"ID":4,"Gender":"男","Name":"stu04"},{"ID":5,"Gender":"男","Name":"stu05"},{"ID":6,"Gender":"男","Name":"stu06"},{"ID":7,"Gender":"男","Name":"stu07"},{"ID":8,"Gender":"男","Name":"stu08"},{"ID":9,"Gender":"男","Name":"stu09"}]}`
	c1 := &Class{}
	err = json.Unmarshal([]byte(str), c1)
	if err != nil {
		fmt.Println("json unmarshal failed!")
		return
	}
	fmt.Printf("%#v\n", c1)
}
```

##### **标签（Tag）**

结构体的元信息，在运行的时候通过反射的机制读取出来。 

`Tag`在结构体字段的后方定义，由一对**反引号**包裹起来

由一个或多个键值对组成。键与值使用冒号分隔，值用双引号括起来。同一个结构体字段可以设置多个键值对tag，不同的键值对之间使用空格分隔。

```go
`key1:"value1" key2:"value2"`
```

**注意事项：** 为结构体编写`Tag`时，必须严格遵守键值对的规则。结构体标签的解析代码的容错能力很差，一旦格式写错，编译和运行时都不会提示任何错误，通过反射也无法正确取值。例如不要在key和value之间添加空格。

例如我们为`Student`结构体的每个字段定义json序列化时使用的Tag：

```go
//Student 学生
type Student struct {
	ID     int    `json:"id"` //通过指定tag实现json序列化该字段时的key
	Gender string //json序列化是默认使用字段名作为key
	name   string //私有不能被json包访问
}

func main() {
	s1 := Student{
		ID:     1,
		Gender: "男",
		name:   "沙河娜扎",
	}
	data, err := json.Marshal(s1)
	if err != nil {
		fmt.Println("json marshal failed!")
		return
	}
	fmt.Printf("json str:%s\n", data) //json str:{"id":1,"Gender":"男"}
}
```

##### **思考题**

执行结果是什么

```go
type student struct {
	name string
	age  int
}

func main() {
	m := make(map[string]*student)
	stus := []student{
		{name: "小王子", age: 18},
		{name: "娜扎", age: 23},
		{name: "大王八", age: 9000},
	}

	for _, stu := range stus {
		m[stu.name] = &stu
	}
	for k, v := range m {
		fmt.Println(k, "=>", v.name)
	}
}

小王子 => 大王八
娜扎 => 大王八
大王八 => 大王八

原因分析： for _, stu := range stus 中，遍历切片的过程中 stu 每次迭代都是使用相同的地址去接受值的（不断的更替stu变量的值，但变量始终是这一变量，地址未变）。所以打印的结果是 大王八
```

### 包

使用`包（package）`来支持代码模块化和代码复用

#### **定义包**

一个包可以简单理解为一个存放`.go`文件的文件夹。该文件夹下面的所有`.go`文件都要在非注释的第一行添加如下声明，声明该文件归属的包。

```go
package packagename
```

- package：声明包的关键字
- packagename：包名，可以不与文件夹的名称一致，不能包含 `-` 符号，最好与其实现的功能相对应。

注意：一个文件夹下面直接包含的文件只能归属一个包，同一个包的文件不能在多个文件夹下。

包名为`main`的包是应用程序的入口包，这种包编译后会得到一个可执行文件，而编译不包含`main`包的源代码则不会得到可执行文件。

#### **标识符可见性**

Go语言中是通过标识符的首字母大/小写来控制标识符的对外可见（public）/不可见（private）的。

在一个包内部只有首字母大写的标识符才是对外可见的。

一般对于首字母大写外部可访问的变量/函数，需在上方加一行注释写上 名称 + 解释用处，帮助他人理解

```go
package demo

import "fmt"

// 包级别标识符的可见性

// num 定义一个全局整型变量
// 首字母小写，对外不可见(只能在当前包内使用)
var num = 100

// Mode 定义一个常量
// 首字母大写，对外可见(可在其它包中使用)
const Mode = 1

// person 定义一个代表人的结构体
// 首字母小写，对外不可见(只能在当前包内使用)
type person struct {
	name string
	Age  int
}

// Add 返回两个整数和的函数
// 首字母大写，对外可见(可在其它包中使用)
func Add(x, y int) int {
	return x + y
}

// sayHi 打招呼的函数
// 首字母小写，对外不可见(只能在当前包内使用)
func sayHi() {
	var myName = "七米" // 函数局部变量，只能在当前函数内使用
	fmt.Println(myName)
}

//结构体中可导出字段的字段名称必须首字母大写。
type Student struct {
	Name  string // 可在包外访问的方法
	class string // 仅限包内访问的字段
}
```

#### **包的引入**

`import`关键字引入包

```go
import importname "path/to/package"
```

- importname：引入的包名，通常都省略。默认值为引入包的包名。
- path/to/package：引入包的路径名称，必须使用双引号包裹起来。

注意：

- Go语言中禁止循环导入包。
- Go语言中不允许引入包却不在代码中使用这个包的内容，如果引入了未使用的包则会触发编译错误。

同时引入多个包

```go
import "fmt"
import "net/http"
import "os"
```

批量引入

```go
import (
    "fmt"
  	"net/http"
    "os"
)
```

存在相同的包名或者想自行为某个引入的包设置一个新包名时，都需要通过`importname`指定一个在当前文件中使用的新包名。例如：

```go
import f "fmt"

f.Println("Hello world!")
```

#### **匿名引入**

设置了一个特殊`_`作为包名。目的主要是为了加载这个包，从而使得这个包中的资源得以初始化。 被匿名引入的包中的`init`函数将被执行并且仅执行一遍。

```go
import _ "github.com/go-sql-driver/mysql"
```

#### **init初始化函数**

不接收任何参数也没有任何返回值，不能在代码中主动调用它。程序启动的时候，init函数会按照它们声明的顺序自动执行。

```go
func init(){
  // ...
}
```

初始化过程是按照代码中引入的顺序来进行的

![image-20230928220336621](https://s2.loli.net/2023/10/28/6ib2ShEynRod8Zz.png)

包的初始化是先从初始化包级别变量开始的。包级别变量的初始化会先于`init`初始化函数。

```go
package main

import "fmt"

var x int8 = 10

const pi = 3.14

func init() {
  fmt.Println("x:", x)
  fmt.Println("pi:", pi)
  sayHi()
}

func sayHi() {
	fmt.Println("Hello World!")
}

func main() {
	fmt.Println("你好，世界！")
}

x: 10
pi: 3.14
Hello World!
你好，世界！
```

### 接口

接口（interface）定义了一个对象的行为规范，由具体的对象来实现规范的细节。像是一种约定——概括了一种类型应该具备哪些方法，

**接口（interface）是一种类型，一种抽象的类型！！**（遇到接口不懂的问题，默念三遍就懂了hh）

Go语言中提倡使用面向接口的编程方式实现解耦。

#### **定义**

每个接口类型由任意个方法签名组成

```go
type 接口类型名 interface{
    方法名1( 参数列表1 ) 返回值列表1
    方法名2( 参数列表2 ) 返回值列表2
    …
}
```

- 接口类型名：Go语言的接口在命名时，一般会在单词后面添加`er`，如有写操作的接口叫`Writer`，有关闭操作的接口叫`closer`等。接口名最好要能突出该接口的类型含义。
- 方法名：当方法名首字母是大写且这个接口类型名首字母也是大写时，这个方法可以被接口所在的包（package）之外的代码访问。
- 参数列表、返回值列表：参数列表和返回值列表中的参数变量名可以省略。

#### **实现接口条件**

接口就是规定了一个**需要实现的方法列表**。

一个类型只要实现了接口中规定的所有方法，那么我们就称它实现了这个接口。

例如：

```go
// Singer 接口
type Singer interface {
	Sing()
}

type Bird struct {}
// Sing Bird类型的Sing方法
func (b Bird) Sing() {
	fmt.Println("汪汪汪")
}

//这样就称为Bird实现了Singer接口。
```

#### **为什么需要接口**

通用的抽象类型

```go
package main

import "fmt"

type Sayer interface {
    Say()
}

type Cat struct{}

func (c Cat) Say() {
	fmt.Println("喵喵喵~")
}

type Dog struct{}

func (d Dog) Say() {
	fmt.Println("汪汪汪~")
}

// MakeHungry 饿肚子了...
func MakeHungry(s Sayer) {
	s.Say()
}

// 通过使用接口类型，把所有会叫的动物当成Sayer类型来处理，只要实现了Say()方法都能当成Sayer类型的变量来处理。
var c cat
MakeHungry(c)
var d dog
MakeHungry(d)
```

举例1：

电商系统中我们允许用户使用多种支付方式（支付宝支付、微信支付、银联支付等），我们的交易流程中可能不太在乎用户究竟使用什么支付方式，只要它能提供一个实现支付功能的`Pay`方法让调用方调用就可以了。

举例2：

我们需要在某个程序中添加一个将某些指标数据向外输出的功能，根据不同的需求可能要将数据输出到终端、写入到文件或者通过网络连接发送出去。在这个场景下我们可以不关注最终输出的目的地是什么，只需要它能提供一个`Write`方法让我们把内容写入就可以了。

#### **面向接口编程**

Go语言中使用隐式声明的方式实现接口。只要一个类型实现了接口中规定的所有方法，那么它就实现了这个接口。

例如：

```go
// Payer 包含支付方法的接口类型
type Payer interface {
	Pay(int64)
}

type ZhiFuBao struct {
	// 支付宝
}

// Pay 支付宝的支付方法
func (z *ZhiFuBao) Pay(amount int64) {
  fmt.Printf("使用支付宝付款：%.2f元。\n", float64(amount/100))
}

type WeChat struct {
	// 微信
}

// Pay 微信的支付方法
func (w *WeChat) Pay(amount int64) {
	fmt.Printf("使用微信付款：%.2f元。\n", float64(amount/100))
}

// Checkout 结账
func Checkout(obj Payer) {
	// 支付100元
	obj.Pay(100)
}

func main() {
	Checkout(&ZhiFuBao{}) // 之前调用支付宝支付

	Checkout(&WeChat{}) // 现在支持使用微信支付
}
```

像类似的例子在我们编程过程中会经常遇到：

- 比如一个网上商城可能使用支付宝、微信、银联等方式去在线支付，我们能不能把它们当成“支付方式”来处理呢？
- 比如三角形，四边形，圆形都能计算周长和面积，我们能不能把它们当成“图形”来处理呢？
- 比如满减券、立减券、打折券都属于电商场景下常见的优惠方式，我们能不能把它们当成“优惠券”来处理呢？

#### **接口类型变量**

一个接口类型的变量能够存储所有实现了该接口的类型变量。

例如`Dog`和`Cat`类型均实现了`Sayer`接口，此时一个`Sayer`类型的变量就能够接收`Cat`和`Dog`类型的变量。

```go
var x Sayer // 声明一个Sayer类型的变量x
a := Cat{}  // 声明一个Cat类型变量a
b := Dog{}  // 声明一个Dog类型变量b
x = a       // 可以把Cat类型变量直接赋值给x
x.Say()     // 喵喵喵
x = b       // 可以把Dog类型变量直接赋值给x
x.Say()     // 汪汪汪
```

#### **值接收者和指针接收者**

##### **值接收者**

此时实现`Mover`接口的是`Dog`类型。

使用值接收者实现接口之后，不管是结构体类型还是对应的结构体指针类型的变量都可以赋值给该接口变量。

```go
// Mover 定义一个接口类型
type Mover interface {
	Move()
}

// Dog 狗结构体类型
type Dog struct{}

// Move 使用值接收者定义Move方法实现Mover接口
func (d Dog) Move() {
	fmt.Println("狗会动")
}

var x Mover    // 声明一个Mover类型的变量x

var d1 = Dog{} // d1是Dog类型
x = d1         // 可以将d1赋值给变量x
x.Move()

var d2 = &Dog{} // d2是Dog指针类型
x = d2          // 也可以将d2赋值给变量x
x.Move()
```

##### **指针接收者**

此时实现`Mover`接口的是`*Cat`类型

```go
// Cat 猫结构体类型
type Cat struct{}

// Move 使用指针接收者定义Move方法实现Mover接口
func (c *Cat) Move() {
	fmt.Println("猫会动")
}

var c1 = &Cat{} // c1是*Cat类型
x = c1          // 可以将c1当成Mover类型
x.Move()

// 下面的代码无法通过编译。不能给将Cat类型的变量赋值给Mover接口类型的变量x。
var c2 = Cat{} // c2是Cat类型
x = c2         // 不能将c2当成Mover类型
```

**原因分析：**由于Go语言中有对指针求值的语法糖，对于值接收者实现的接口，无论使用值类型还是指针类型都没有问题。

但是我们并不总是能对一个值求址，所以对于指针接收者实现的接口要额外注意。

#### **类型和接口关系**

##### **一个类型实现多个接口**

一个类型可以同时实现多个接口，而接口间彼此独立

```go
// Sayer 接口
type Sayer interface {
	Say()
}

// Mover 接口
type Mover interface {
	Move()
}

// Dog既可以实现Sayer接口，也可以实现Mover接口。
type Dog struct {
	Name string
}

// 实现Sayer接口
func (d Dog) Say() {
	fmt.Printf("%s会叫汪汪汪\n", d.Name)
}

// 实现Mover接口
func (d Dog) Move() {
	fmt.Printf("%s会动\n", d.Name)
}

// 同一个类型实现不同的接口互相不影响使用
var d = Dog{Name: "旺财"}

var s Sayer = d
var m Mover = d

s.Say()  // 对Sayer类型调用Say方法
m.Move() // 对Mover类型调用Move方法
```

##### **多种类型实现同一接口**

不同的类型还可以实现同一接口。例如在我们的代码世界中不仅狗可以动，汽车也可以动。

```go
// 实现Mover接口
func (d Dog) Move() {
	fmt.Printf("%s会动\n", d.Name)
}

// Car 汽车结构体类型
type Car struct {
	Brand string
}

// Move Car类型实现Mover接口
func (c Car) Move() {
	fmt.Printf("%s速度70迈\n", c.Brand)
}

// 把狗和汽车当成一个会动的类型来处理，不必关注它们具体是什么，只需要调用它们的Move方法就可以了
var obj Mover

obj = Dog{Name: "旺财"}
obj.Move()

obj = Car{Brand: "宝马"}
obj.Move()


旺财会跑
宝马速度70迈
```

一个接口的所有方法，不一定需要由一个类型完全实现，接口的方法可以通过在类型中嵌入其他类型或者结构体来实现。

```go
// WashingMachine 洗衣机
type WashingMachine interface {
	wash()
	dry()
}

// 甩干器
type dryer struct{}

// 实现WashingMachine接口的dry()方法
func (d dryer) dry() {
	fmt.Println("甩一甩")
}

// 海尔洗衣机
type haier struct {
	dryer //嵌入甩干器
}

// 实现WashingMachine接口的wash()方法
func (h haier) wash() {
	fmt.Println("洗刷刷")
}
```

#### **接口组合**

接口与接口之间可以通过互相嵌套形成新的接口类型，例如Go标准库`io`源码中就有很多接口之间互相组合的示例。

这种由多个接口类型组合形成的新接口类型，同样只需要实现新接口类型中规定的所有方法就算实现了该接口类型。

```go
// src/io/io.go

type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

type Closer interface {
	Close() error
}

// ReadWriter 是组合Reader接口和Writer接口形成的新接口类型
type ReadWriter interface {
	Reader
	Writer
}

// ReadCloser 是组合Reader接口和Closer接口形成的新接口类型
type ReadCloser interface {
	Reader
	Closer
}

// WriteCloser 是组合Writer接口和Closer接口形成的新接口类型
type WriteCloser interface {
	Writer
	Closer
}
```

#### **结构体内嵌接口**

接口也可以作为结构体的一个字段，我们来看一段Go标准库`sort`源码中的示例。

```go
// src/sort/sort.go

// Interface 定义通过索引对元素排序的接口类型
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}


// reverse 结构体中嵌入了Interface接口
type reverse struct {
    Interface
}
```

通过在结构体中嵌入一个接口类型，从而让该结构体类型实现了该接口类型，并且还可以改写该接口的方法。

```go
// Less 为reverse类型添加Less方法，重写原Interface接口类型的Less方法
func (r reverse) Less(i, j int) bool {
	return r.Interface.Less(j, i)
}
```

`Interface`类型原本的`Less`方法签名为`Less(i, j int) bool`，此处重写为`r.Interface.Less(j, i)`，即通过将索引参数交换位置实现反转。

在这个示例中还有一个需要注意的地方是`reverse`结构体本身是不可导出的（结构体类型名称首字母小写），`sort.go`中通过定义一个可导出的`Reverse`函数来让使用者创建`reverse`结构体实例。

```go
func Reverse(data Interface) Interface {
	return &reverse{data}
}
```

这样做的目的是保证得到的`reverse`结构体中的`Interface`属性一定不为`nil`，否者`r.Interface.Less(j, i)`就会出现空指针panic。

#### **空接口**

##### **定义**

指没有定义任何方法的接口类型。因此任何类型都可以视为实现了空接口。也正是因为空接口类型的这个特性，空接口类型的变量可以存储任意类型的值。

```go
// Any 不包含任何方法的空接口类型
type Any interface{}

// Dog 狗结构体
type Dog struct{}

func main() {
	var x Any

	x = "你好" // 字符串型
	fmt.Printf("type:%T value:%v\n", x, x)
	x = 100 // int型
	fmt.Printf("type:%T value:%v\n", x, x)
	x = true // 布尔型
	fmt.Printf("type:%T value:%v\n", x, x)
	x = Dog{} // 结构体类型
	fmt.Printf("type:%T value:%v\n", x, x)
}


type:string value:你好
type:int value:100
type:bool value:true
type:main.Dog value:{}
```

通常我们在使用空接口类型时不必使用`type`关键字声明，可以像下面的代码一样直接使用`interface{}`。

```go
var x interface{}  // 声明一个空接口类型变量x
```

##### **作为函数的参数**

使用空接口实现可以接收任意类型的函数参数。

```go
// 空接口作为函数参数
func show(a interface{}) {
	fmt.Printf("type:%T value:%v\n", a, a)
}
```

##### **作为map的值**

使用空接口实现可以保存任意值的字典。

```go
// 空接口作为map值
var studentInfo = make(map[string]interface{})
studentInfo["name"] = "沙河娜扎"
studentInfo["age"] = 18
studentInfo["married"] = false
fmt.Println(studentInfo)

map[age:18 married:false name:沙河娜扎]
```

#### **接口值**

由于接口类型的值可以是任意一个实现了该接口的类型值，所以接口值除了需要记录具体**值**之外，还需要记录这个值属于的**类型**。

接口值由“类型”和“值”组成，鉴于这两部分会根据存入值的不同而发生变化，我们称之为接口的`动态类型`和`动态值`

![接口值示例](https://s2.loli.net/2023/10/28/jhBvxyKklR1isUJ.png)

例如：

```go
type Mover interface {
	Move()
}

type Dog struct {
	Name string
}

func (d *Dog) Move() {
	fmt.Println("狗在跑~")
}

type Car struct {
	Brand string
}

func (c *Car) Move() {
	fmt.Println("汽车在跑~")
}

var m Mover  //此时m它的类型和值部分都是nil，不能对一个空接口值调用任何方法，否则会产生panic。
fmt.Println(m == nil)  // true

m = &Dog{Name: "旺财"} // 此时，接口值m的动态类型会被设置为*Dog，动态值为结构体变量的拷贝。

var c *Car
m = c // 接口值m的动态类型为*Car，动态值为nil。
fmt.Println(m == nil) // false 此时接口变量m与nil并不相等，因为它只是动态值的部分为nil，而动态类型部分保存着对应值的类型。
```

![接口值示例](https://s2.loli.net/2023/10/28/XqZoQVEJHyYSsTP.png)

![接口值示例](https://s2.loli.net/2023/10/28/wa9WVkMZQDvLAj6.png)

![接口值示例](https://s2.loli.net/2023/10/28/gcTXKqkAzxvFPD3.png)

##### **接口值比较**

接口值是支持相互比较的，当且仅当接口值的动态类型和动态值都相等时才相等。

```go
var (
	x Mover = new(Dog)
	y Mover = new(Car)
)
fmt.Println(x == y) // false
```

有一种特殊情况需要特别注意，如果接口值保存的动态类型相同，但是这个动态类型不支持互相比较（比如切片），那么对它们相互比较时就会引发panic。

```go
var z interface{} = []int{1, 2, 3}
fmt.Println(z == z) // panic: runtime error: comparing uncomparable type []int
```

#### **类型断言**

接口值可能赋值为任意类型的值，那我们如何从接口值获取其存储的具体数据呢？

我们可以借助标准库`fmt`包的格式化打印获取到接口值的动态类型。

`fmt`包内部其实是使用反射的机制在程序运行时获取到动态类型的名称。

```go
var m Mover

m = &Dog{Name: "旺财"}
fmt.Printf("%T\n", m) // *main.Dog

m = new(Car)
fmt.Printf("%T\n", m) // *main.Car
```

想要从接口值中获取到对应的实际值需要使用类型断言，其语法格式如下。

```go
x.(T)
```

- x：表示接口类型的变量
- T：表示断言`x`可能是的类型。

返回两个参数

- 第一个参数是`x`转化为`T`类型后的变量，
- 第二个值是一个布尔值，若为`true`则表示断言成功，为`false`则表示断言失败。

```go
var n Mover = &Dog{Name: "旺财"}
v, ok := n.(*Dog)
if ok {
	fmt.Println("类型断言成功")
	v.Name = "富贵" // 变量v是*Dog类型
} else {
	fmt.Println("类型断言失败")
}
```

如果对一个接口值有多个实际类型需要判断，推荐使用`switch`语句来实现。

```go
// justifyType 对传入的空接口类型变量x进行类型断言
func justifyType(x interface{}) {
	switch v := x.(type) {
	case string:
		fmt.Printf("x is a string，value is %v\n", v)
	case int:
		fmt.Printf("x is a int is %v\n", v)
	case bool:
		fmt.Printf("x is a bool is %v\n", v)
	default:
		fmt.Println("unsupport type！")
	}
}
```

#### **验证是否实现接口**

验证某结构体是否满足特定的接口类型

```go
// 摘自gin框架routergroup.go
type IRouter interface{ ... }

type RouterGroup struct { ... }

var _ IRouter = &RouterGroup{}  // 确保RouterGroup实现了接口IRouter
```

上面的代码中也可以使用`var _ IRouter = (*RouterGroup)(nil)`进行验证。

#### **练习题**

使用接口的方式实现一个既可以往终端写日志也可以往文件写日志的简易日志库。

```go
package main

import (
   "fmt"
   "io"
   "os"
)

// 使用接口的方式实现一个既可以往终端写日志也可以往文件写日志的简易日志库。
type Logger interface {
   Info(string)
}

type FileLogger struct {
   filename string
}

func (fl *FileLogger) Info(msg string) {
   var f *os.File
   var err1 error
   if checkFileIsExist(fl.filename) { //如果文件存在
      f, err1 = os.OpenFile(fl.filename, os.O_APPEND|os.O_WRONLY, 0666) //打开文件
      fmt.Println("文件存在")
   } else {
      f, err1 = os.Create(fl.filename) //创建文件
      fmt.Println("文件不存在")
   }
   defer f.Close()
   n, err1 := io.WriteString(f, msg+"\n") //写入文件(字符串)
   if err1 != nil {
      panic(err1)
   }
   fmt.Printf("写入 %d 个字节\n", n)
}

func checkFileIsExist(filename string) bool {
   if _, err := os.Stat(filename); os.IsNotExist(err) {
      return false
   }
   return true
}

type ConsoleLogger struct {
}

func (cl *ConsoleLogger) Info(msg string) {
   fmt.Println(msg)
}

func homework() {
   var logger Logger
   fileLogger := &FileLogger{"log.txt"}
   logger = fileLogger
   logger.Info("Hello")
   logger.Info("how are you")

   consoleLogger := &ConsoleLogger{}
   logger = consoleLogger
   logger.Info("Hello")
   logger.Info("how are you")
}

func main() {
   homework()
}
```

### error

Go 语言中的错误处理与其他语言不太一样，它把错误当成一种值来处理，更强调判断错误、处理错误，而不是一股脑的 catch 捕获异常。

#### **error 接口**

使用一个名为 `error` 接口来表示错误类型。

```go
type error interface {
    Error() string
}
```

当一个函数或方法需要返回错误时，我们通常是把错误作为最后一个返回值。例如下面标准库 os 中打开文件的函数。

```go
func Open(name string) (*File, error) {
	return OpenFile(name, O_RDONLY, 0)
}
```

#### **判断是否有error**

由于 error 是一个接口类型，默认零值为`nil`。所以我们通常将调用函数返回的错误与`nil`进行比较，以此来判断函数是否返回错误。例如你会经常看到类似下面的错误判断代码。

注意：当我们使用`fmt`包打印错误时会自动调用 error 类型的 Error 方法，也就是会打印出错误的描述信息。

```go
file, err := os.Open("./xx.go")
if err != nil {
	fmt.Println("打开文件失败,err:", err)
	return
}
```

#### **创建错误**

可以根据需求自定义 error，最简单的方式是使用`errors` 包提供的`New`函数创建一个错误。

##### **errors.New**

接收一个字符串参数返回包含该字符串的错误。

```go
func New(text string) error
```

我们可以在函数返回时快速创建一个错误。

```go
func queryById(id int64) (*Info, error) {
	if id <= 0 {
		return nil, errors.New("无效的id")
	}

	// ...
}
```

或者用来定义一个错误变量，例如标准库`io.EOF`错误定义如下。

```go
var EOF = errors.New("EOF")
```

#### **fmt.Errorf**

当我们需要传入格式化的错误描述信息时，使用`fmt.Errorf`是个更好的选择。

```go
fmt.Errorf("查询数据库失败，err:%v", err)
```

但是上面的方式会丢失原有的错误类型，只拿到错误描述的文本信息。

为了不丢失函数调用的错误链，使用`fmt.Errorf`时搭配使用特殊的格式化动词`%w`，可以实现基于已有的错误再包装得到一个新的错误。

```go
fmt.Errorf("查询数据库失败，err:%w", err)
```

对于这种二次包装的错误，`errors`包中提供了以下三个方法。

```go
func Unwrap(err error) error                 // 获得err包含下一层错误
func Is(err, target error) bool              // 判断err是否包含target
func As(err error, target interface{}) bool  // 判断err是否为target类型
```

#### **错误（err）结构体类型**

可以自己定义结构体类型，实现`error`接口。

```go
// OpError 自定义结构体类型
type OpError struct {
	Op string
}

// Error OpError 类型实现error接口
func (e *OpError) Error() string {
	return fmt.Sprintf("无权执行%s操作", e.Op)
}
```

### 反射

#### **变量内在机制**

变量是分为两部分的:

- 类型信息：预先定义好的元信息。
- 值信息：程序运行过程中可动态变化的。

#### **反射介绍**

反射是指在程序运行期间对程序本身进行访问和修改的能力。可在运行时动态的获取一个变量的类型信息和值信息。

程序在编译时，变量被转换为内存地址，变量名不会被编译器写入到可执行部分。在运行程序时，程序无法获取自身的信息。

支持反射的语言可以在程序编译期间将变量的反射信息，如字段名称、类型信息、结构体信息等整合到可执行文件中，并给程序提供接口访问反射信息，这样就可以在程序运行期间获取类型的反射信息，并且有能力修改它们。

Go程序在运行期间使用reflect包访问程序的反射信息。

#### **reflect包**

Go语言的反射机制中，任何接口值都由是`一个具体类型`和`具体类型的值`两部分组成的

任意接口值在反射中都可以理解为由`reflect.Type`和`reflect.Value`两部分组成

reflect包提供了`reflect.TypeOf`和`reflect.ValueOf`两个函数来获取任意对象的Value和Type。

#### **类型**

##### **TypeOf**

`reflect.TypeOf()`函数可以获得任意值的类型对象（reflect.Type），程序通过类型对象可以访问任意值的类型信息。

```go
func reflectType(x interface{}) {
	v := reflect.TypeOf(x)
	fmt.Printf("type:%v\n", v)
}
func main() {
	var a float32 = 3.14
	reflectType(a) // type:float32
	var b int64 = 100
	reflectType(b) // type:int64
}
```

##### **type name和type kind**

反射中关于类型还划分为两种：`类型（Type）`和`种类（Kind）`。

- 类型（type）：使用type关键字构造的自定义类型、别名的实际类型等
  - 例如 people、student、dog、cat
- 种类（kind）：指底层的类型
  - 例如 slice、struct

```go
type myInt int64

func reflectType(x interface{}) {
	t := reflect.TypeOf(x)
	fmt.Printf("type:%v kind:%v\n", t.Name(), t.Kind())
}

func main() {
	var a *float32 // 指针
	var b myInt    // 自定义类型
	var c rune     // 类型别名
	reflectType(a) // type: kind:ptr
	reflectType(b) // type:myInt kind:int64
	reflectType(c) // type:int32 kind:int32

	type person struct {
		name string
		age  int
	}
	type book struct{ title string }
	var d = person{
		name: "沙河小王子",
		age:  18,
	}
	var e = book{title: "《跟小王子学Go语言》"}
	reflectType(d) // type:person kind:struct
	reflectType(e) // type:book kind:struct
}
```

Go语言的反射中像数组、切片、Map、指针等类型的变量，它们的`.Name()`都是返回`空`。

在`reflect`包中定义的Kind类型如下：

```go
type Kind uint
const (
    Invalid Kind = iota  // 非法类型
    Bool                 // 布尔型
    Int                  // 有符号整型
    Int8                 // 有符号8位整型
    Int16                // 有符号16位整型
    Int32                // 有符号32位整型
    Int64                // 有符号64位整型
    Uint                 // 无符号整型
    Uint8                // 无符号8位整型
    Uint16               // 无符号16位整型
    Uint32               // 无符号32位整型
    Uint64               // 无符号64位整型
    Uintptr              // 指针
    Float32              // 单精度浮点数
    Float64              // 双精度浮点数
    Complex64            // 64位复数类型
    Complex128           // 128位复数类型
    Array                // 数组
    Chan                 // 通道
    Func                 // 函数
    Interface            // 接口
    Map                  // 映射
    Ptr                  // 指针
    Slice                // 切片
    String               // 字符串
    Struct               // 结构体
    UnsafePointer        // 底层指针
)
```

#### **值**

##### **ValueOf**

`reflect.ValueOf()`返回的是`reflect.Value`类型，其中包含了原始值的值信息。

##### **获取值**

`reflect.Value`与原始值之间可以互相转换。

`reflect.Value`类型提供的获取原始值的方法如下：

![image-20230930221219562](https://s2.loli.net/2023/10/28/5VWxNlsREGmeZ93.png)

```go
func reflectValue(x interface{}) {
	v := reflect.ValueOf(x)
	k := v.Kind()
	switch k {
	case reflect.Int64:
		// v.Int()从反射中获取整型的原始值，然后通过int64()强制类型转换
		fmt.Printf("type is int64, value is %d\n", v.Int())
	case reflect.Float32:
		// v.Float()从反射中获取浮点型的原始值，然后通过float32()强制类型转换
		fmt.Printf("type is float32, value is %f\n", float32(v.Float()))
	case reflect.Float64:
		// v.Float()从反射中获取浮点型的原始值，然后通过float64()强制类型转换
		fmt.Printf("type is float64, value is %f\n", v.Float())
	}
}
func main() {
	var a float32 = 3.14
	var b int64 = 100
	reflectValue(a) // type is float32, value is 3.140000
	reflectValue(b) // type is int64, value is 100
	// 将int类型的原始值转换为reflect.Value类型
	c := reflect.ValueOf(10)
	fmt.Printf("type c :%T\n", c) // type c :reflect.Value
}
```

##### **设置值**

通过反射修改变量的值，需要注意函数参数传递的是值拷贝，必须传递变量地址才能修改变量值。

**Elem()**

而反射中使用专有的`Elem()`方法来获取指针对应的值。

```go
func reflectSetValue1(x interface{}) {
	v := reflect.ValueOf(x)
	if v.Kind() == reflect.Int64 {
		v.SetInt(200) //修改的是副本，reflect包会引发panic
	}
}
func reflectSetValue2(x interface{}) {
	v := reflect.ValueOf(x)
	// 反射中使用 Elem()方法获取指针对应的值
	if v.Elem().Kind() == reflect.Int64 {
		v.Elem().SetInt(200)
	}
}
func main() {
	var a int64 = 100
	// reflectSetValue1(a) //panic: reflect: reflect.Value.SetInt using unaddressable value
	reflectSetValue2(&a)
	fmt.Println(a)
}
```

##### **isNil()**

`IsNil()`报告**v持有的值是否为nil**。v持有的值的分类必须是通道、函数、接口、映射、指针、切片之一；否则IsNil函数会导致panic。

```go
func (v Value) IsNil() bool
```

##### **isValid()**

`IsValid()`返回**v是否持有一个值**。如果v是Value零值会返回假，此时v除了IsValid、String、Kind之外的方法都会导致panic。

```go
func (v Value) IsValid() bool
```

`IsNil()`常被用于判断指针是否为空；`IsValid()`常被用于判定返回值是否有效。

```go
func main() {
	// *int类型空指针
	var a *int
	fmt.Println("var a *int IsNil:", reflect.ValueOf(a).IsNil())
	// nil值
	fmt.Println("nil IsValid:", reflect.ValueOf(nil).IsValid())
	// 实例化一个匿名结构体
	b := struct{}{}
	// 尝试从结构体中查找"abc"字段
	fmt.Println("不存在的结构体成员:", reflect.ValueOf(b).FieldByName("abc").IsValid())
	// 尝试从结构体中查找"abc"方法
	fmt.Println("不存在的结构体方法:", reflect.ValueOf(b).MethodByName("abc").IsValid())
	// map
	c := map[string]int{}
	// 尝试从map中查找一个不存在的键
	fmt.Println("map中不存在的键：", reflect.ValueOf(c).MapIndex(reflect.ValueOf("娜扎")).IsValid())
}

var a *int IsNil: true
nil IsValid: false
不存在的结构体成员: false
不存在的结构体方法: false
map中不存在的键： false
```

#### **结构体反射**

##### **相关方法**

任意值通过`reflect.TypeOf()`获得反射对象信息后，如果它的类型是结构体，可以通过反射值对象（`reflect.Type`）的`NumField()`和`Field()`方法获得结构体成员的详细信息。

`reflect.Type`中与获取结构体成员相关的的方法如下表所示。

![image-20230930224221289](https://s2.loli.net/2023/10/28/GxgFvCMq1NVZJQa.png)

##### **StructField类型**

`StructField`类型用来描述结构体中的一个字段的信息。

`StructField`的定义如下：

```go
type StructField struct {
    // Name是字段的名字。PkgPath是非导出字段的包路径，对导出字段该字段为""。
    // 参见http://golang.org/ref/spec#Uniqueness_of_identifiers
    Name    string
    PkgPath string
    Type      Type      // 字段的类型
    Tag       StructTag // 字段的标签
    Offset    uintptr   // 字段在结构体中的字节偏移量
    Index     []int     // 用于Type.FieldByIndex时的索引切片
    Anonymous bool      // 是否匿名字段
}
```

##### **示例**

###### **字段信息**

使用反射得到一个结构体数据之后可以通过索引依次获取其字段信息

也可以通过字段名去获取指定的字段信息。

```go
type student struct {
	Name  string `json:"name"`
	Score int    `json:"score"`
}

func main() {
	stu1 := student{
		Name:  "小王子",
		Score: 90,
	}

	t := reflect.TypeOf(stu1)
	fmt.Println(t.Name(), t.Kind()) // student struct
	// 通过for循环遍历结构体的所有字段信息
	for i := 0; i < t.NumField(); i++ {
		field := t.Field(i)
		fmt.Printf("name:%s index:%d type:%v json tag:%v\n", field.Name, field.Index, field.Type, field.Tag.Get("json"))
	}

	// 通过字段名获取指定结构体字段信息
	if scoreField, ok := t.FieldByName("Score"); ok {
		fmt.Printf("name:%s index:%d type:%v json tag:%v\n", scoreField.Name, scoreField.Index, scoreField.Type, scoreField.Tag.Get("json"))
	}
}

student struct
name:Name index:[0] type:string json tag:name
name:Score index:[1] type:int json tag:score
name:Score index:[1] type:int json tag:score
```

###### **方法信息**

```go
// 给student添加两个方法 Study和Sleep(注意首字母大写)
func (s student) Study() string {
	msg := "好好学习，天天向上。"
	fmt.Println(msg)
	return msg
}

func (s student) Sleep() string {
	msg := "好好睡觉，快快长大。"
	fmt.Println(msg)
	return msg
}

func printMethod(x interface{}) {
	t := reflect.TypeOf(x)
	v := reflect.ValueOf(x)

	fmt.Println(t.NumMethod())
	fmt.Println(v.NumMethod())
	for i := 0; i < v.NumMethod(); i++ {
		fmt.Printf("t method name:%s\n", t.Method(i).Name)
		fmt.Printf("t method type:%s\n", t.Method(i).Type)
		fmt.Printf("v method type:%s\n", v.Method(i).Type())
		// 通过反射调用方法传递的参数必须是 []reflect.Value 类型
		var args = []reflect.Value{}
		v.Method(i).Call(args)
	}
}

2
2
t method name:Sleep
t method type:func(main.student) string
v method type:func() string
好好睡觉，快快长大。
t method name:Study
t method type:func(main.student) string
v method type:func() string
好好学习，天天向上。
```

#### **慎用反射-原因**

反射是一个强大并富有表现力的工具，能让我们写出更灵活的代码。但是反射不应该被滥用，原因有以下三个。

1. 基于反射的代码是极其脆弱的，反射中的类型错误会在真正运行的时候才会引发panic，那很可能是在代码写完的很长时间之后。
2. 大量使用反射的代码通常难以理解。
3. 反射的性能低下，基于反射实现的代码通常比正常代码运行速度慢一到两个数量级。

#### **练习题**

编写代码利用反射实现一个ini文件的解析器程序。

```go
// 编写代码利用反射实现一个ini文件的解析器程序。
// 根据ini文件的key，输出key对应的value

type MysqlConfig struct {
    Ip       string `ini:"ip"`
    User     string `ini:"user"`
    Password string `ini:"password"`
    Port     int    `ini:"port"`
}

type Config struct {
    MysqlConfig `ini:"mysqld"`
}

func loadIni(filename string, data interface{}) (err error) {
    // 1. 参数校验，参数必须是指针类型和结构体类型
    t := reflect.TypeOf(data)
    if t.Kind() != reflect.Ptr && t.Kind() != reflect.Struct {
       err = errors.New("type error should be struct ptr ")
       return err
    }

    // 2. 打开文件
    file, err := os.ReadFile(filename)
    if err != nil {
       err = fmt.Errorf("open file error %v\n", err)
       return err
    }

    // 3. 一行一行读取文件
    lineSlice := strings.Split(string(file), "\r\n")
    // 定义section切片
    var section string
    var structName string
    // 遍历slice
    for idx, line := range lineSlice {
       // 去除首尾空格
       line = strings.TrimSpace(line)
       // 如果是空行，则继续
       if len(line) == 0 {
          continue
       }
       // 3.1 如果是注释，则跳过
       if strings.HasPrefix(line, ";") || strings.HasPrefix(line, "#") {
          continue
       }

       if strings.HasPrefix(line, "[") {
          // 3.2 如果开头是[，而且是以]结尾的，且[]不为空，则被认定是section
          if line[0] == '[' && line[len(line)-1] == ']' && len(line) > 2 {
             // 将section 加入到section切片变量里
             section = line[1 : len(line)-1]
          }
          //根据字符串section去data里面根据反射找到对应的结构体
          for i := 0; i < t.Elem().NumField(); i++ {
             field := t.Elem().Field(i)
             if section == field.Tag.Get("ini") {
                structName = field.Name
             }
          }
       } else {
          // 4. 拆分键值对
          // 4.1 以等号分割，左边为key，右边为value
          if strings.Index(line, "=") == -1 || strings.HasPrefix(line, "=") {
             err = fmt.Errorf("line:%d syntax error ", idx+1)
             return
          }

          // 获取到line的key和value
          index := strings.Index(line, "=")
          key := strings.TrimSpace(line[:index])     // user
          value := strings.TrimSpace(line[index+1:]) //mysql

          // 4.2 将ini文件中的section和结构体的名字对应
          v := reflect.ValueOf(data)                 // interface的值,此时为空
          sValue := v.Elem().FieldByName(structName) // 结构体的值 { 10.0.0.1 mysql abc123 0}
          sType := sValue.Type()                     // 结构体的类型 main.MysqlConfig

          // 判断结构体的字段是否是struct类型
          if sType.Kind() != reflect.Struct {
             err = fmt.Errorf("should be struct")
             return err
          }
          // 根据structName 去data里面把嵌套的结构体取出
          // 声明字段值
          var fname string                  // 结构体的字段名
          var fieldType reflect.StructField // 结构体字段类型
          for i := 0; i < sValue.NumField(); i++ {
             fieldName := sType.Field(i)
             fieldType = fieldName
             //  遍历结构体的每一个字段，判断这个tag是不是等于key
             if fieldName.Tag.Get("ini") == key {
                // 找到对应的字段
                fname = fieldName.Name
                break
             }
          }
          // 4.3如果key == tag，and value的类型等于结构体定义的类型，则赋值
          fieldObj := sValue.FieldByName(fname)
          switch fieldType.Type.Kind() { // 判断ini文件中value的种类，字串/数字/布尔
          case reflect.String:
             fieldObj.SetString(value)
          case reflect.Int, reflect.Int16, reflect.Int32, reflect.Int64:
             var valueInt int64
             valueInt, err := strconv.ParseInt(value, 10, 64)
             if err != nil {
                return err
             }
             fieldObj.SetInt(valueInt)

          case reflect.Bool:
             var valueBool bool
             valueBool, err := strconv.ParseBool(value)
             if err != nil {
                return err
             }
             fieldObj.SetBool(valueBool)
          case reflect.Float32, reflect.Float64:
             var valueFloat float64
             valueFloat, err := strconv.ParseFloat(value, 64)
             if err != nil {
                return err
             }
             fieldObj.SetFloat(valueFloat)
          }
       }
    }
    return
}
func main() {

    var config Config
    err := loadIni("./demo.ini", &config)
    if err != nil {
       fmt.Println(err)
    }
    fmt.Println(
       config.MysqlConfig.Ip,
       config.MysqlConfig.Password,
       config.MysqlConfig.Port,
       config.MysqlConfig.User,
    )
}
```

### 并发

#### **进程、线程和协程**

进程（process）：程序在操作系统中的一次执行过程，系统进行资源分配和调度的一个独立单位。

线程（thread）：操作系统基于进程开启的轻量级进程，是操作系统调度执行的最小单位。

协程（coroutine）：非操作系统提供而是由用户自行创建和控制的用户态‘线程’，比线程更轻量级。

#### **并发模型**

常见的并发模型有以下几种：

- 线程&锁模型
- Actor模型
- CSP模型
- Fork&Join模型

Go语言中的并发程序主要是通过基于CSP（communicating sequential processes）的goroutine和channel来实现，当然也支持使用传统的多线程共享内存的并发方式。

#### **goroutine**

**介绍：**Goroutine 是 Go 语言支持并发的核心，是 Go 程序中最基本的并发执行单元。每一个 Go 程序都至少包含一个 goroutine——main goroutine，当 Go 程序启动时它会自动创建。

**大小：**一个goroutine会以一个很小的栈开始其生命周期，一般只需要2KB。

**调度方式：**区别于操作系统线程由系统内核进行调度， goroutine 是由Go运行时（runtime）负责调度。例如Go运行时会智能地将 m个goroutine 合理地分配给n个操作系统线程，实现类似m:n的调度机制，不再需要Go开发者自行在代码层面维护一个线程池。

**如何使用：**当你需要让某个任务并发执行的时候，你只需要把这个任务包装成一个函数，开启一个 goroutine 去执行这个函数就可以了，就是这么简单粗暴。

##### **go关键字**

只需要在函数或方法调用前加上`go`关键字就可以创建一个 goroutine ，从而让该函数或方法在新创建的 goroutine 中执行。

一个 goroutine 必定对应一个函数/方法，可以创建多个 goroutine 去执行相同的函数/方法。

```go
go f()  // 创建一个新的 goroutine 运行函数f
```

匿名函数也支持使用`go`关键字创建 goroutine 去执行。

```go
go func(){
  // ...
}()
```

##### **启动单个goroutine**

只需要在调用函数（普通函数和匿名函数）前加上一个`go`关键字。

**错误示例：**

```go
func main() {
	go hello() // 启动另外一个goroutine去执行hello函数
	fmt.Println("你好")
}

你好
```

只在终端打印了”你好”，并没有打印 `hello`。

**原因：**main goroutine结束return了，hello的goroutine还没进行完，因其属于main goroutine 所以老大都直接没了，小弟也直接被扼杀掉了。（ main 函数退出太快，另外一个 goroutine 中的函数还未执行完程序就退出了，导致未打印出“hello”。）

**正确示例：**

使用`sync.WaitGroup`来等待 hello goroutine 完成后再退出。

```go
// 声明全局等待组变量
var wg sync.WaitGroup

func hello() {
	fmt.Println("hello")
	wg.Done() // 告知当前goroutine完成
}

func main() {
	wg.Add(1) // 登记1个goroutine
	go hello()
	fmt.Println("你好")
	wg.Wait() // 阻塞等待登记的goroutine完成
}
```

##### **启动多个goroutine**

同样使用了`sync.WaitGroup`来实现 goroutine 的同步。

每次终端上打印数字的顺序都不一致。10个 goroutine 是并发执行的，而 goroutine 的调度是随机的。

```go
var wg sync.WaitGroup

func hello(i int) {
	defer wg.Done() // goroutine结束就登记-1
	fmt.Println("hello", i)
}
func main() {
	for i := 0; i < 10; i++ {
		wg.Add(1) // 启动一个goroutine就登记+1
		go hello(i)
	}
	wg.Wait() // 等待所有登记的goroutine都结束
}
```

##### **动态栈**

操作系统的线程一般都有固定的栈内存（通常为2MB）,而 Go 语言中的 goroutine 非常轻量级，一个 goroutine 的初始栈空间很小（一般为2KB）

所以在 Go 语言中一次创建数万个 goroutine 也是可能的。并且 goroutine 的栈不是固定的，可以根据需要动态地增大或缩小， Go 的 runtime 会自动为 goroutine 分配合适的栈空间。

##### **goroutine调度原理**

区别于操作系统内核调度操作系统线程，goroutine 的调度是Go语言运行时（runtime）层面的实现，是完全由 Go 语言本身实现的一套调度系统——go scheduler。它的作用是按照一定的规则将所有的 goroutine 调度到操作系统线程上执行。

在经历数个版本的迭代之后，目前 Go 语言的调度器采用的是 `GPM` 调度模型。

![gpm](https://s2.loli.net/2023/10/28/Ud84CWI9XSebvnH.png)

其中：

- G：表示 goroutine，每执行一次`go f()`就创建一个 G，包含要执行的函数和上下文信息。
- 全局队列（Global Queue）：存放等待运行的 G。
- P：表示 goroutine 执行所需的资源，最多有 GOMAXPROCS 个。
- P 的本地队列：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建 G 时，G 优先加入到 P 的本地队列，如果本地队列满了会批量移动部分 G 到全局队列。
- M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，当 P 的本地队列为空时，M 也会尝试从全局队列或其他 P 的本地队列获取 G。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。
- Goroutine 调度器和操作系统调度器是通过 M 结合起来的，每个 M 都代表了1个内核线程，操作系统调度器负责把内核线程分配到 CPU 的核上执行。

**优势**：

- OS线程是由OS内核来调度的， goroutine 则是由Go运行时（runtime）自己的调度器调度的，完全是在用户态下完成的， 不涉及内核态与用户态之间的频繁切换，包括内存的分配与释放，都是在用户态维护着一块大的内存池， 不直接调用系统的malloc函数（除非内存池需要改变），成本比调度OS线程低很多。
- 充分利用了多核的硬件资源，近似的把若干goroutine均分在物理线程上， 再加上本身 goroutine 的超轻量级，以上种种特性保证了 goroutine 调度方面的性能。

##### **GOMAXPROCS**

Go运行时的调度器使用`GOMAXPROCS`参数来确定需要使用多少个 OS 线程来同时执行 Go 代码。

默认值是机器上的 CPU 核心数。例如在一个 8 核心的机器上，GOMAXPROCS 默认为 8。

Go语言中可以通过`runtime.GOMAXPROCS`函数设置当前程序并发时占用的 CPU逻辑核心数。（Go1.5版本之前，默认使用的是单核心执行。Go1.5 版本之后，默认使用全部的CPU 逻辑核心数。）

#### **channel**

函数与函数间需要交换数据才能体现并发执行函数的意义。

Go语言采用的并发模型是`CSP（Communicating Sequential Processes）`，提倡**通过通信共享内存**而不是**通过共享内存而实现通信**。

如果说 goroutine 是Go程序并发的执行体，`channel`就是它们之间的连接。`channel`是可以让一个 goroutine 发送特定值到另一个 goroutine 的通信机制。

Go 语言中的通道（channel）是一种特殊的类型。通道像一个传送带或者队列，总是遵循先入先出（First In First Out）的规则，保证收发数据的顺序。每一个通道都是一个具体类型的导管，也就是声明channel的时候需要为其指定元素类型。

##### **类型**

`channel`是 Go 语言中一种特有的类型。声明通道类型变量的格式如下：

```go
var 变量名称 chan 元素类型
```

- chan：是关键字
- 元素类型：是指通道中传递元素的类型

##### **零值**

未初始化的通道类型变量其默认零值是`nil`。

```go
var ch chan int
fmt.Println(ch) // <nil>
```

##### **初始化**

声明的通道类型变量需要使用内置的`make`函数初始化之后才能使用。

```go
make(chan 元素类型, [缓冲大小])
```

- channel的缓冲大小是可选的。

##### **操作**

共有发送（send）、接收(receive）和关闭（close）三种操作。

而发送和接收操作都使用`<-`符号。

```go
ch := make(chan int)
```

###### **发送**

```go
ch <- 10 // 把10发送到ch中
```

###### **接收**

```go
x := <- ch // 从ch中接收值并赋值给变量x
<-ch       // 从ch中接收值，忽略结果
```

###### **关闭**

个人理解：只要不往channel里发送数据了，就可以关闭了。

当向通道中发送完数据时，我们可以通过`close`函数来关闭通道。

```go
close(ch)
```

**注意：**一个通道值是可以被垃圾回收掉的。通道通常由发送方执行关闭操作，并且只有在接收方明确等待通道关闭的信号时才需要执行关闭操作。它和关闭文件不一样，通常在结束操作之后关闭文件是必须要做的，但关闭通道不是必须的。

关闭后的通道有以下特点：

1. 对一个关闭的通道再发送值就会导致 panic。
2. 对一个关闭的通道进行接收会一直获取值直到通道为空。**（重点）**
3. 对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。
4. 关闭一个已经关闭的通道会导致 panic。

##### **无缓冲的通道（同步通道）**

无缓冲的通道又称为阻塞的通道。

无缓冲的通道只有在有接收方能够接收值的时候才能发送成功，否则会一直处于等待发送的阶段！！！

如果对一个无缓冲通道执行接收操作时，没有任何向通道中发送值的操作那么也会导致接收操作阻塞！！！

使用无缓冲通道进行通信将导致发送和接收的 goroutine 同步化。因此，无缓冲通道也被称为`同步通道`。

**错误示例：**

代码会阻塞在`ch <- 10`这一行代码形成死锁，因为没有goroutine去接收它

```go
func main() {
	ch := make(chan int)
	ch <- 10
	fmt.Println("发送成功")
}

fatal error: all goroutines are asleep - deadlock!
```

**正确示例：**

创建一个 goroutine 去接收值

首先无缓冲通道`ch`上的发送操作会阻塞，直到另一个 goroutine 在该通道上执行接收操作，这时数字10才能发送成功，两个 goroutine 将继续执行。

相反，如果接收操作先执行，接收方所在的 goroutine 将阻塞，直到 main goroutine 中向该通道发送数字10。

```go
func recv(c chan int) {
	ret := <-c
	fmt.Println("接收成功", ret)
}

func main() {
	ch := make(chan int)
	go recv(ch) // 创建一个 goroutine 从通道接收值
	ch <- 10
	fmt.Println("发送成功")
}
```

##### **有缓冲的通道**

在使用 make 函数初始化通道时，可以为其指定通道的容量，通道的容量表示通道中最大能存放的元素数量。

当通道内已有元素数达到最大容量后，再向通道执行发送操作就会阻塞，除非有从通道执行接收操作。

```go
func main() {
	ch := make(chan int, 1) // 创建一个容量为1的有缓冲区通道
	ch <- 10
	fmt.Println("发送成功")
}
```

可以使用内置的`len`函数获取通道内元素的数量，使用`cap`函数获取通道的容量，虽然我们很少会这么做。

##### **多返回值模式**

对一个通道执行接收操作时支持使用如下多返回值模式。

```go
value, ok := <- ch
```

- value：从通道中取出的值，如果通道被关闭则返回对应类型的零值。
- ok：通道ch关闭时返回 false，否则返回 true。

```go
func f2(ch chan int) {
	for {
		v, ok := <-ch
		if !ok {
			fmt.Println("通道已关闭")
			break
		}
		fmt.Printf("v:%#v ok:%#v\n", v, ok)
	}
}

func main() {
	ch := make(chan int, 2)
	ch <- 1
	ch <- 2
	close(ch)
	f2(ch)
}
```

##### **for range接收值**

通常我们会选择使用`for range`循环从通道中接收值，当通道被关闭后，会在通道内的所有值被接收完毕后会自动退出循环。

```go
func f3(ch chan int) {
	for v := range ch {
		fmt.Println(v)
	}
}
func main() {
	ch := make(chan int, 2)
	ch <- 1
	ch <- 2
	close(ch)
	f3(ch)
}
```

##### **判断是否为空**

**注意：**目前Go语言中并没有提供一个不对通道进行读取操作就能判断通道是否被关闭的方法。不能简单的通过`len(ch)`操作来判断通道是否被关闭。

个人理解：可通过‘多返回值模式’里的ok判断channel是否为关闭

##### **单向通道**

在不同的任务函数中对通道的使用进行限制，比如限制通道在某个函数中只能执行发送或只能执行接收操作。

Go语言中提供了**单向通道**来处理这种需要限制通道只能进行某种操作的情况。

```go
<- chan int // 只接收通道，只能接收不能发送，不允许进行close，默认通道的关闭操作应该由发送方来完成。
chan <- int // 只发送通道，只能发送不能接收
```

例如：

```go
// Producer2 返回一个接收通道
func Producer2() <-chan int {
	ch := make(chan int, 2)
	// 创建一个新的goroutine执行发送数据的任务
	go func() {
		for i := 0; i < 10; i++ {
			if i%2 == 1 {
				ch <- i
			}
		}
		close(ch) // 任务完成后关闭通道
	}()

	return ch
}

// Consumer2 参数为接收通道
func Consumer2(ch <-chan int) int {
	sum := 0
	for v := range ch {
		sum += v
	}
	return sum
}

func main() {
	ch2 := Producer2()
  
	res2 := Consumer2(ch2)
	fmt.Println(res2) // 25
}
```

##### **单向通道转换**

在函数传参及任何赋值操作中全向通道（正常通道）可以转换为单向通道，但是无法反向转换。

```go
var ch3 = make(chan int, 1)
ch3 <- 10
close(ch3)
Consumer2(ch3) // 函数传参时将ch3转为单向通道

var ch4 = make(chan int, 1)
ch4 <- 10
var ch5 <-chan int // 声明一个只接收通道ch5
ch5 = ch4          // 变量赋值时将ch4转为单向通道
<-ch5
```

##### **操作状态总结**

![img](https://s2.loli.net/2023/10/28/4OazkTJevLHgU6F.png)

**注意：**对已经关闭的通道再执行 close 也会引发 panic。

#### **select多路复用**

在某些场景下我们可能需要同时从多个通道接收数据。

Go 语言内置了`select`关键字，使用它可以同时响应多个通道的操作。

类似于之前学到的 switch 语句，它也有一系列 case 分支和一个默认的分支。每个 case 分支会对应一个通道的通信（接收或发送）过程。

```go
select {
case <-ch1:
	//...
case data := <-ch2:
	//...
case ch3 <- 10:
	//...
default:
	//默认操作
}
```

**特点：**

- 可处理一个或多个 channel 的发送/接收操作。
- 如果多个 case 同时满足，select 会**随机**选择一个执行。
- 对于没有 case 的 select 会一直阻塞，可用于阻塞 main 函数，防止退出。

```go
func main() {
	ch := make(chan int, 1)
	for i := 1; i <= 10; i++ {
		select {
		case x := <-ch:
			fmt.Println(x)
		case ch <- i:
		}
	}
}

1
3
5
7
9
```

#### **并发安全和锁**

多个 goroutine 同时操作一个资源（临界区）的情况，这种情况下就会发生`竞态问题`（数据竞态）。

##### **互斥锁**

保证同一时间只有一个 goroutine 可以访问共享资源。

Go 语言中使用`sync`包中提供的`Mutex`类型来实现互斥锁。

`sync.Mutex`提供了两个方法

![image-20231002162008582](https://s2.loli.net/2023/10/28/H2oqYkOIMlP4TBc.png)

使用互斥锁能够保证同一时间有且只有一个 goroutine 进入临界区，其他的 goroutine 则在等待锁；

当互斥锁释放后，等待的 goroutine 才可以获取锁进入临界区，多个 goroutine 同时等待一个锁时，**唤醒的策略是随机的**。

```go
var (
	x int64

	wg sync.WaitGroup // 等待组

	m sync.Mutex // 互斥锁
)

// add 对全局变量x执行5000次加1操作
func add() {
	for i := 0; i < 5000; i++ {
		m.Lock() // 修改x前加锁
		x = x + 1
		m.Unlock() // 改完解锁
	}
	wg.Done()
}

func main() {
	wg.Add(2)

	go add()
	go add()

	wg.Wait()
	fmt.Println(x)
}

10000
```

##### **读写互斥锁**

很多场景是读多写少的，当我们并发的去读取一个资源而不涉及资源修改的时候是没有必要加互斥锁的

读写锁在 Go 语言中使用`sync`包中的`RWMutex`类型。

`sync.RWMutex`提供了以下5个方法。

![image-20231002162359556](https://s2.loli.net/2023/10/28/xZb5fMcOQLursiU.png)

**读锁和写锁**

当一个 goroutine 获取到读锁之后，其他的 goroutine 如果是获取读锁会继续获得锁，如果是获取写锁就会等待；

而当一个 goroutine 获取写锁之后，其他的 goroutine 无论是获取读锁还是写锁都会等待。

**性能比较案例：**

从最终的执行结果可以看出，使用读写互斥锁在读多写少的场景下能够极大地提高程序的性能。

不过需要注意的是如果一个程序中的读操作和写操作数量级差别不大，那么读写互斥锁的优势就发挥不出来。

```go
var (
	x       int64
	wg      sync.WaitGroup
	mutex   sync.Mutex
	rwMutex sync.RWMutex
)

// writeWithLock 使用互斥锁的写操作
func writeWithLock() {
	mutex.Lock() // 加互斥锁
	x = x + 1
	time.Sleep(10 * time.Millisecond) // 假设读操作耗时10毫秒
	mutex.Unlock()                    // 解互斥锁
	wg.Done()
}

// readWithLock 使用互斥锁的读操作
func readWithLock() {
	mutex.Lock()                 // 加互斥锁
	time.Sleep(time.Millisecond) // 假设读操作耗时1毫秒
	mutex.Unlock()               // 释放互斥锁
	wg.Done()
}

// writeWithLock 使用读写互斥锁的写操作
func writeWithRWLock() {
	rwMutex.Lock() // 加写锁
	x = x + 1
	time.Sleep(10 * time.Millisecond) // 假设读操作耗时10毫秒
	rwMutex.Unlock()                  // 释放写锁
	wg.Done()
}

// readWithRWLock 使用读写互斥锁的读操作
func readWithRWLock() {
	rwMutex.RLock()              // 加读锁
	time.Sleep(time.Millisecond) // 假设读操作耗时1毫秒
	rwMutex.RUnlock()            // 释放读锁
	wg.Done()
}

func do(wf, rf func(), wc, rc int) {
	start := time.Now()
	// wc个并发写操作
	for i := 0; i < wc; i++ {
		wg.Add(1)
		go wf()
	}

	//  rc个并发读操作
	for i := 0; i < rc; i++ {
		wg.Add(1)
		go rf()
	}

	wg.Wait()
	cost := time.Since(start)
	fmt.Printf("x:%v cost:%v\n", x, cost)

}

// 使用互斥锁，10并发写，1000并发读
do(writeWithLock, readWithLock, 10, 1000) // x:10 cost:1.466500951s

// 使用读写互斥锁，10并发写，1000并发读
do(writeWithRWLock, readWithRWLock, 10, 1000) // x:10 cost:117.207592ms
```

##### **sync.WaitGroup**

使用`sync.WaitGroup`来实现并发任务的同步。 

其内部维护着一个计数器，计数器的值可以增加和减少。

- 例如当我们启动了 N 个并发任务时，就将计数器值增加N。
- 每个任务完成时通过调用 Done 方法将计数器减1。
- 通过调用 Wait 来等待并发任务执行完，当计数器值为 0 时，表示所有并发任务已经完成。

![image-20231002163121703](https://s2.loli.net/2023/10/28/UPCknQyp1VsHOqM.png)

**注意：**`sync.WaitGroup`是一个结构体，进行参数传递的时候要传递指针。

##### **sync.Once**

确保某些操作即使在高并发的场景下也只会被执行一次，例如只加载一次配置文件等。

`sync.Once`只有一个`Do`方法

```go
func (o *Once) Do(f func())
```

**注意：**如果要执行的函数`f`需要传递参数就需要搭配闭包来使用。

```go
var icons map[string]image.Image

var loadIconsOnce sync.Once

func loadIcons() {
	icons = map[string]image.Image{
		"left":  loadIcon("left.png"),
		"up":    loadIcon("up.png"),
		"right": loadIcon("right.png"),
		"down":  loadIcon("down.png"),
	}
}

// Icon 是并发安全的
func Icon(name string) image.Image {
	loadIconsOnce.Do(loadIcons)
	return icons[name]
}
```

**并发安全的单例模式**

借助`sync.Once`实现的并发安全的单例模式：

`sync.Once`其实内部包含一个互斥锁和一个布尔值，互斥锁保证布尔值和数据的安全，而布尔值用来记录初始化是否完成。这样设计就能保证初始化操作的时候是并发安全的并且初始化操作也不会被执行多次。

```go
type singleton struct {}

var instance *singleton
var once sync.Once

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{}
    })
    return instance
}
```

##### **sync.Map**

Go 语言中内置的 map 不是并发安全的，并发读写会报错

`sync`包中提供了一个开箱即用的并发安全版 map——`sync.Map`。

不用像内置的 map 一样使用 make 函数初始化就能直接使用。同时`sync.Map`内置了诸如`Store`、`Load`、`LoadOrStore`、`Delete`、`Range`等操作方法。

![image-20231002165043992](https://s2.loli.net/2023/10/28/IJTqUpg8WNOvZCM.png)

```go
// 并发安全的map
var m = sync.Map{}

func main() {
	wg := sync.WaitGroup{}
	// 对m执行20个并发的读写操作
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(n int) {
			key := strconv.Itoa(n)
			m.Store(key, n)         // 存储key-value
			value, _ := m.Load(key) // 根据key取值
			fmt.Printf("k=:%v,v:=%v\n", key, value)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

#### **原子操作**

针对整数数据类型（int32、uint32、int64、uint64）我们还可以使用原子操作来保证并发安全

通常直接使用原子操作比使用锁操作效率更高。

Go语言中原子操作由内置的标准库`sync/atomic`提供。

**atomic包**

`atomic`包提供了底层的原子级内存操作，对于同步算法的实现很有用。

**注意：**这些函数必须谨慎地保证正确使用。除了某些特殊的底层应用，使用通道或者 sync 包的函数/类型实现同步更好。

![image-20231002165329733](https://s2.loli.net/2023/10/28/2ecSXkulUOsYCKA.png)

**性能比较案例：**

比较下互斥锁和原子操作的性能。

```go

type Counter interface {
	Inc()
	Load() int64
}

// 普通版
type CommonCounter struct {
	counter int64
}

func (c *CommonCounter) Inc() {
	c.counter++
}

func (c CommonCounter) Load() int64 {
	return c.counter
}

// 互斥锁版
type MutexCounter struct {
	counter int64
	lock    sync.Mutex
}

func (m *MutexCounter) Inc() {
	m.lock.Lock()
	defer m.lock.Unlock()
	m.counter++
}

func (m *MutexCounter) Load() int64 {
	m.lock.Lock()
	defer m.lock.Unlock()
	return m.counter
}

// 原子操作版
type AtomicCounter struct {
	counter int64
}

func (a *AtomicCounter) Inc() {
	atomic.AddInt64(&a.counter, 1)
}

func (a *AtomicCounter) Load() int64 {
	return atomic.LoadInt64(&a.counter)
}

func test(c Counter) {
	var wg sync.WaitGroup
	start := time.Now()
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			c.Inc()
			wg.Done()
		}()
	}
	wg.Wait()
	end := time.Now()
	fmt.Println(c.Load(), end.Sub(start))
}

func main() {
	c1 := &CommonCounter{} // 非并发安全
	test(c1)
	c2 := &MutexCounter{} // 使用互斥锁实现并发安全
	test(c2)
	c3 := &AtomicCounter{} // 并发安全且比互斥锁效率更高
	test(c3)
}

993 1.0281ms
1000 517.6µs
1000 513.2µs
```

#### **并发错误处理**

##### **recover goroutine内的panic**

使用 recover 来会恢复程序中意想不到的 panic，而 **panic 只会触发当前 goroutine 中的 defer 操作**。

在启用 goroutine 去执行任务的场景下，如果想要 recover goroutine中可能出现的 panic 就需要在 goroutine 中使用 recover。

```go
func f2() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Printf("recover outer panic:%v\n", r)
		}
	}()
	// 开启一个goroutine执行任务
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("recover inner panic:%v\n", r)
			}
		}()
		fmt.Println("in goroutine....")
		// 只能触发当前goroutine中的defer
		panic("panic in goroutine")
	}()

	time.Sleep(time.Second)
	fmt.Println("exit")
}

in goroutine....
recover inner panic:panic in goroutine
exit
```

##### **errgroup**

当我们想要将一个任务拆分成多个子任务交给多个 goroutine 去运行，这时我们该如何获取到子任务可能返回的错误呢？

errgroup 包就是为了解决这类问题而开发的，它能为处理公共任务的子任务而开启的一组 goroutine 提供同步、error 传播和基于context 的取消功能。

errgroup 包中定义了一个 Group 类型，它包含了若干个不可导出的字段。

```go
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once
	err     error
}
```

errgroup.Group 提供了`Go`和`Wait`两个方法。

```go
func (g *Group) Go(f func() error)
```

- Go 函数会在新的 goroutine 中调用传入的函数f。
- 第一个返回非零错误的调用将取消该Group；下面的Wait方法会返回该错误

```go
func (g *Group) Wait() error
```

- Wait 会阻塞直至由上述 Go 方法调用的所有函数都返回，然后从它们**返回第一个非nil的错误（如果有）**。

例子：

子任务的 goroutine 中对`http://www.yixieqitawangzhi.com` 发起 HTTP 请求时会返回一个错误，这个错误会由 errgroup.Group 的 Wait 方法返回。

```go
// fetchUrlDemo2 使用errgroup并发获取url内容
func fetchUrlDemo2() error {
	g := new(errgroup.Group) // 创建等待组（类似sync.WaitGroup）
	var urls = []string{
		"http://pkg.go.dev",
		"http://www.liwenzhou.com",
		"http://www.yixieqitawangzhi.com",
	}
	for _, url := range urls {
		url := url // 注意此处声明新的变量,因为在for的过程中url改变了，此处声明了一个新的变量url 让下面Go函数中调用的url不会改变
		// 启动一个goroutine去获取url内容
		g.Go(func() error {
			resp, err := http.Get(url)
			if err == nil {
				fmt.Printf("获取%s成功\n", url)
				resp.Body.Close()
			}
			return err // 返回错误
		})
	}
	if err := g.Wait(); err != nil {
		// 处理可能出现的错误
		fmt.Println(err)
		return err
	}
	fmt.Println("所有goroutine均成功")
	return nil
}
```

（后续待学习补充，目前水平有限qwq）

#### **练习题**

使用 goroutine 和 channel 实现一个计算int64随机数各位数和的程序，例如生成随机数61345，计算其每个位数上的数字之和为19。

1. 开启一个 goroutine 循环生成int64类型的随机数，发送到`jobChan`
2. 开启24个 goroutine 从`jobChan`中取出随机数计算各位数的和，将结果发送到`resultChan`
3. 主 goroutine 从`resultChan`取出结果并打印到终端输出

```go
var wg sync.WaitGroup

type res struct {
    v   int64
    sum int64
}

func producer(ch1 chan<- int64) {
    defer wg.Done()
    defer close(ch1)

    for {
       v := rand.Int63()
       ch1 <- v
       time.Sleep(time.Millisecond * 300)
    }
}

func consumer(ch1 <-chan int64, ch2 chan<- *res) {
    defer wg.Done()

    for {
       var sum int64
       v := <-ch1
       vTmp := v
       for vTmp > 0 {
          sum += vTmp % 10
          vTmp /= 10
       }

       ch2 <- &res{
          v:   v,
          sum: sum,
       }
    }
}

func main() {
    jobChan := make(chan int64, 100)
    resultChan := make(chan *res, 100)

    wg.Add(1)
    go producer(jobChan)
    wg.Add(24)
    for i := 0; i < 24; i++ {
       go consumer(jobChan, resultChan)
    }

    for r := range resultChan {
       fmt.Printf("%d---->%d\n", r.v, r.sum)
    }

    wg.Wait()
}
```

### 网络编程

#### **互联网分层模型**

![osi七层模型](https://s2.loli.net/2023/10/28/3rPBXYzeb9IGnAi.png)

#### **socket编程**

`Socket`是应用层与TCP/IP协议族通信的中间软件抽象层。

在设计模式中，`Socket`其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在`Socket`后面，对用户来说只需要调用Socket规定的相关函数，让`Socket`去组织符合指定的协议数据然后进行通信。

![socket图解](https://s2.loli.net/2023/10/28/4KJRUyvfLQ6ch3N.png)

#### **TCP通信**

##### **服务端**

一个TCP服务端可以同时连接很多个客户端，Go中每建立一次链接就创建一个goroutine去处理。

TCP服务端程序的处理流程：

- 监听端口
- 接收客户端请求建立链接
- 创建goroutine处理链接

```go
// 处理函数
func process(conn net.Conn) {
	defer conn.Close() // 关闭连接
	for {
		reader := bufio.NewReader(conn)
		var buf [128]byte
		n, err := reader.Read(buf[:]) // 读取数据
		if err != nil {
			fmt.Println("read from client failed, err:", err)
			break
		}
		recvStr := string(buf[:n])
		fmt.Println("收到client端发来的数据：", recvStr)
		conn.Write([]byte(recvStr)) // 发送数据
	}
}

func main() {
	listen, err := net.Listen("tcp", "127.0.0.1:20000")
	if err != nil {
		fmt.Println("listen failed, err:", err)
		return
	}
	for {
		conn, err := listen.Accept() // 建立连接
		if err != nil {
			fmt.Println("accept failed, err:", err)
			continue
		}
		go process(conn) // 启动一个goroutine处理连接
	}
}
```

##### **客户端**

一个TCP客户端进行TCP通信的流程如下：

- 建立与服务端的链接
- 进行数据收发
- 关闭链接

```go
// 客户端
func main() {
	conn, err := net.Dial("tcp", "127.0.0.1:20000")
	if err != nil {
		fmt.Println("err :", err)
		return
	}
	defer conn.Close() // 关闭连接，客户端关闭连接后 服务端再read读取的话会EOF
	inputReader := bufio.NewReader(os.Stdin)
	for {
		input, _ := inputReader.ReadString('\n') // 读取用户输入
		inputInfo := strings.Trim(input, "\r\n")
		if strings.ToUpper(inputInfo) == "Q" { // 如果输入q就退出
			return
		}
		_, err = conn.Write([]byte(inputInfo)) // 发送数据
		if err != nil {
			return
		}
		buf := [512]byte{}
		n, err := conn.Read(buf[:])
		if err != nil {
			fmt.Println("recv failed, err:", err)
			return
		}
		fmt.Println(string(buf[:n]))
	}
}
```

##### **黏包问题**

（有待后续补充）

#### **UDP通信**

##### **服务端**

```go
// UDP server端
func main() {
	listen, err := net.ListenUDP("udp", &net.UDPAddr{
		IP:   net.IPv4(0, 0, 0, 0),
		Port: 30000,
	})
	if err != nil {
		fmt.Println("listen failed, err:", err)
		return
	}
	defer listen.Close()
	for {
		var data [1024]byte
		n, addr, err := listen.ReadFromUDP(data[:]) // 接收数据
		if err != nil {
			fmt.Println("read udp failed, err:", err)
			continue
		}
		fmt.Printf("data:%v addr:%v count:%v\n", string(data[:n]), addr, n)
		_, err = listen.WriteToUDP(data[:n], addr) // 发送数据
		if err != nil {
			fmt.Println("write to udp failed, err:", err)
			continue
		}
	}
}
```

##### **客户端**

```go
// UDP 客户端
func main() {
	socket, err := net.DialUDP("udp", nil, &net.UDPAddr{
		IP:   net.IPv4(0, 0, 0, 0),
		Port: 30000,
	})
	if err != nil {
		fmt.Println("连接服务端失败，err:", err)
		return
	}
	defer socket.Close()
	sendData := []byte("Hello server")
	_, err = socket.Write(sendData) // 发送数据
	if err != nil {
		fmt.Println("发送数据失败，err:", err)
		return
	}
	data := make([]byte, 4096)
	n, remoteAddr, err := socket.ReadFromUDP(data) // 接收数据
	if err != nil {
		fmt.Println("接收数据失败，err:", err)
		return
	}
	fmt.Printf("recv:%v addr:%v count:%v\n", string(data[:n]), remoteAddr, n)
}
```



