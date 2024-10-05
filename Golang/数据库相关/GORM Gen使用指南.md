# GORM Gen使用指南

Gen是一个基于GORM的安全ORM框架，其主要通过代码生成方式实现GORM代码封装。使用Gen框架能够自动生成Model结构体和类型安全的CRUD代码，极大提升CRUD效率。

## Gen介绍

[Gen](https://github.com/go-gorm/gen)是由字节跳动无恒实验室与GORM作者联合研发的一个基于GORM的安全ORM框架，主要通过代码生成方式实现GORM代码封装。

Gen框架在GORM框架的基础上提供了以下能力：

- 基于原始SQL语句生成可重用的CRUD API
- 生成不使用`interface{}`的100%安全的DAO API
- 依据数据库生成遵循GORM约定的结构体Model
- 支持GORM的所有特性

简单来说，使用Gen框架后我们无需手动定义结构体Model，同时Gen框架也能帮我们生成类型安全的CRUD代码。

更多详细介绍请查看[Gen官方文档](https://gorm.io/gen/)。

此外，Facebook开源的[ent](https://github.com/ent/ent)也是社区中常用的类似框架，大家可按需选择使用。

## 如何使用Gen

Gen框架的使用非常简单，如果你熟悉GORM框架，那么你可以通过以下教程快速上手。

### 安装依赖

```bash
go get -u gorm.io/gen
```

### 快速指南

想要在项目中使用Gen框架，通常只需三步。本节将通过一个简单示例快速带大家熟悉Gen框架的使用。

首先，我们假设数据库中已经有一张`book`表，建表语句如下。

```sql
CREATE TABLE book
(
    `id`     bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
    `title`  varchar(128) NOT NULL COMMENT '书籍名称',
    `author` varchar(128) NOT NULL COMMENT '作者',
    `price`  int NOT NULL DEFAULT '0' COMMENT '价格',
    `publish_date` datetime COMMENT '出版日期',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='书籍表';
```

> 本教程演示的为先有数据表的业务场景，通常这也是比较主流的工程实现流程。

#### 定义Gen配置

配置即代码。我们通常会在项目的`cmd`目录下定义好Gen框架生成代码的配置。例如，我们的项目名称为`gen_demo`，那么我们就在`gen_demo/cmd/gen/generate.go`文件。

```go
package main

// gorm gen configure

import (
	"fmt"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"

	"gorm.io/gen"
)

const MySQLDSN = "root:root1234@tcp(127.0.0.1:13306)/db2?charset=utf8mb4&parseTime=True"

func connectDB(dsn string) *gorm.DB {
	db, err := gorm.Open(mysql.Open(dsn))
	if err != nil {
		panic(fmt.Errorf("connect db fail: %w", err))
	}
	return db
}

func main() {
	// 指定生成代码的具体相对目录(相对当前文件)，默认为：./query
	// 默认生成需要使用WithContext之后才可以查询的代码，但可以通过设置gen.WithoutContext禁用该模式
	g := gen.NewGenerator(gen.Config{
		// 默认会在 OutPath 目录生成CRUD代码，并且同目录下生成 model 包
		// 所以OutPath最终package不能设置为model，在有数据库表同步的情况下会产生冲突
		// 若一定要使用可以通过ModelPkgPath单独指定model package的名称
		OutPath: "../../dal/query",
		/* ModelPkgPath: "dal/model"*/

		// gen.WithoutContext：禁用WithContext模式
		// gen.WithDefaultQuery：生成一个全局Query对象Q
		// gen.WithQueryInterface：生成Query接口
		Mode: gen.WithDefaultQuery | gen.WithQueryInterface,
	})

	// 通常复用项目中已有的SQL连接配置db(*gorm.DB)
	// 非必需，但如果需要复用连接时的gorm.Config或需要连接数据库同步表信息则必须设置
	g.UseDB(connectDB(MySQLDSN))

	// 从连接的数据库为所有表生成Model结构体和CRUD代码
	// 也可以手动指定需要生成代码的数据表
	g.ApplyBasic(g.GenerateAllTable()...)

	// 执行并生成代码
	g.Execute()
}
```

> 为什么要放到 `cmd`目录下？👉 [Go官方模块布局说明](https://go.dev/doc/modules/layout)

#### 生成代码

进入项目下的`cmd/gen`目录下，执行以下命令。

```bash
go run generate.go
```

上述命令会在项目目录下生成`dal`目录，其中`dal/query`中是CRUD代码，`dal/model`下则是生成Model结构体。

```bash
├── cmd
│   └── gen
│       └── generate.go
├── dal
│   ├── model
│   │   └── book.gen.go
│   └── query
│       ├── book.gen.go
│       └── gen.go
├── go.mod
├── go.sum
└── main.go
```

> 我们可以在dal下新建`db.go`文件，保存如下初始化数据库连接的代码。
>
> ```go
> package dal
> 
> import (
> 	"fmt"
> 
> 	"gorm.io/driver/mysql"
> 	"gorm.io/gorm"
> )
> 
> var DB *gorm.DB
> 
> func ConnectDB(dsn string) *gorm.DB {
> 	db, err := gorm.Open(mysql.Open(dsn))
> 	if err != nil {
> 		panic(fmt.Errorf("connect db fail: %w", err))
> 	}
> 	return db
> }
> ```

注意：通常不建议直接修改Gen框架生成的代码。

#### 使用生成的代码

Gen会生成基础的查询方法，并且绑定到结构体上，我们可以在项目中使用了它们。

```go
package main

import (
	"context"
	"fmt"
	"gen_demo/dal"
	"gen_demo/dal/model"
	"gen_demo/dal/query"
	"time"
)

// gen demo

// MySQLDSN MySQL data source name
const MySQLDSN = "root:root1234@tcp(127.0.0.1:13306)/db2?charset=utf8mb4&parseTime=True"

func init() {
	dal.DB = dal.ConnectDB(MySQLDSN).Debug()
}

func main() {
	// 设置默认DB对象
	query.SetDefault(dal.DB)

	// 创建
	b1 := model.Book{
		Title:       "《七米的Go语言之路》",
		Author:      "七米",
		PublishDate: time.Date(2023, 11, 15, 0, 0, 0, 0, time.UTC),
		Price:       100,
	}
	err := query.Book.WithContext(context.Background()).Create(&b1)
	if err != nil {
		fmt.Printf("create book fail, err:%v\n", err)
		return
	}

	// 更新
	ret, err := query.Book.WithContext(context.Background()).
		Where(query.Book.ID.Eq(1)).
		Update(query.Book.Price, 200)
	if err != nil {
		fmt.Printf("update book fail, err:%v\n", err)
		return
	}
	fmt.Printf("RowsAffected:%v\n", ret.RowsAffected)

	// 查询
	book, err := query.Book.WithContext(context.Background()).First()
	// 也可以使用全局Q对象查询
	//book, err := query.Q.Book.WithContext(context.Background()).First()
	if err != nil {
		fmt.Printf("query book fail, err:%v\n", err)
		return
	}
	fmt.Printf("book:%v\n", book)

	// 删除
	ret, err = query.Book.WithContext(context.Background()).Where(query.Book.ID.Eq(1)).Delete()
	if err != nil {
		fmt.Printf("delete book fail, err:%v\n", err)
		return
	}
	fmt.Printf("RowsAffected:%v\n", ret.RowsAffected)
}
```

通过上述教程，基本即可掌握Gen框架的基本使用，大家可点击查看[Gen官方最佳实践示例代码](https://github.com/go-gorm/gen/tree/master/examples)。

### 自定义SQL查询

Gen框架使用模板注释的方法支持自定义SQL查询，我们只需要按对应规则将SQL语句注释到interface的方法上即可。Gen将对其进行解析，并为应用的结构生成查询API。

通常建议将自定义查询方法添加到`model`模块下。

#### 注释语法

Gen 为动态条件 SQL 支持提供了一些约定语法，分为三个方面：

- 返回结果
- 模板占位符
- 模板表达式

##### 返回结果

| 占位符           | 含义                                                         |
| :--------------- | :----------------------------------------------------------- |
| gen.T            | 用于返回数据的结构体，会根据生成结构体或者数据库表结构自动生成 |
| gen.M            | 表示`map[string]interface{}`,用于返回数据                    |
| gen.RowsAffected | 用于执行SQL进行更新或删除时候,用于返回影响行数               |
| error            | 返回错误（如果有）                                           |

示例

```go
// dal/model/querier.go

package model

import "gorm.io/gen"

// 通过添加注释生成自定义方法

type Querier interface {
	// SELECT * FROM @@table WHERE id=@id
	GetByID(id int) (gen.T, error) // 返回结构体和error

	// GetByIDReturnMap 根据ID查询返回map
	//
	// SELECT * FROM @@table WHERE id=@id
	GetByIDReturnMap(id int) (gen.M, error) // 返回 map 和 error

	// SELECT * FROM @@table WHERE author=@author
	GetBooksByAuthor(author string) ([]*gen.T, error) // 返回数据切片和 error
}
```

在Gen配置处（`cmd/gen/generate.go`）添加自定义方法绑定关系。

```go
// 通过ApplyInterface添加为book表添加自定义方法
g.ApplyInterface(func(model.Querier) {}, g.GenerateModel("book"))
```

重新生成代码后，即可使用自定义方法。

```go
// 使用自定义的GetBooksByAuthor方法
rets, err := query.Book.WithContext(context.Background()).GetBooksByAuthor("七米")
if err != nil {
	fmt.Printf("GetBooksByAuthor fail, err:%v\n", err)
	return
}
for i, b := range rets {
	fmt.Printf("%d:%v\n", i, b)
}
```

##### 模板占位符

|    名称    |           描述            |
| :--------: | :-----------------------: |
| `@@table`  |      转义和引用表名       |
| `@@<name>` | 从参数中转义并引用表/列名 |
| `@<name>`  |    参数中的SQL查询参数    |

示例

```go
// Filter 自定义Filter接口
type Filter interface {
  // SELECT * FROM @@table WHERE @@column=@value
  FilterWithColumn(column string, value string) (gen.T, error)
}

// 为`Book`添加 `Filter`接口
g.ApplyInterface(func(model.Filter) {}, g.GenerateModel("book"))
```

##### 模板表达式

Gen 为动态条件 SQL 提供了强大的表达式支持，目前支持以下表达式:

- `if/else`
- `where`
- `set`
- `for`

示例

```go
// Searcher 自定义接口
type Searcher interface {
	// Search 根据指定条件查询书籍
	//
	// SELECT * FROM book
	// WHERE publish_date is not null
	// {{if book != nil}}
	//   {{if book.ID > 0}}
	//     AND id = @book.ID
	//   {{else if book.Author != ""}}
	//     AND author=@book.Author
	//   {{end}}
	// {{end}}
	Search(book *gen.T) ([]*gen.T, error)
}

// 通过ApplyInterface添加为book表添加Searcher接口
g.ApplyInterface(func(model.Searcher) {}, g.GenerateModel("book"))
```

重新生成代码后，即可直接使用自定义的`Search`方法进行查询。

```go
b := &model.Book{Author: "Q1mi"}
rets, err = query.Book.WithContext(context.Background()).Search(b)
if err != nil {
	fmt.Printf("Search fail, err:%v\n", err)
	return
}
for i, b := range rets {
	fmt.Printf("%d:%v\n", i, b)
}
```

### 数据库到结构体

Gen支持根据GORM约定依据数据库生成结构体，在之前的示例中我们已经使用过类似的代码。

```go
// 根据`users`表生成对应结构体`User`
g.GenerateModel("users")

// 基于`users`表生成名为`Employee`的结构体
g.GenerateModelAs("users", "Employee")

// 在生成结构体时还可指定额外的生成选项
// gen.FieldIgnore("address")：忽略 address 字段
// gen.FieldType("id", "int64")：id字段使用 int64 类型
g.GenerateModel("users", gen.FieldIgnore("address"), gen.FieldType("id", "int64"))

// 为连接的数据库中的所有表生成对应结构体
g.GenerateAllTable()
```

#### 方法模板

当从数据库生成结构体时，还可以为它们生成事先配置的模板方法，例如：

```go
type CommonMethod struct {
    ID   int32
    Name *string
}

func (m *CommonMethod) IsEmpty() bool {
    if m == nil {
        return true
    }
    return m.ID == 0
}

func (m *CommonMethod) GetName() string {
    if m == nil || m.Name == nil {
        return ""
    }
    return *m.Name
}

// 当生成 `People` 结构体时添加 IsEmpty 方法
g.GenerateModel("people", gen.WithMethod(CommonMethod{}.IsEmpty))

// 生成`User`结构体时添加 `CommonMethod` 的所有方法
g.GenerateModel("user", gen.WithMethod(CommonMethod{}))
```

最终将生成类下面的代码。

```go
// Generated Person struct
type Person struct {
  // ...
}

func (m *Person) IsEmpty() bool {
  if m == nil {
    return true
  }
  return m.ID == 0
}


// Generated User struct
type User struct {
  // ...
}

func (m *User) IsEmpty() bool {
  if m == nil {
    return true
  }
  return m.ID == 0
}

func (m *User) GetName() string {
  if m == nil || m.Name == nil {
    return ""
  }
  return *m.Name
}
```

#### 数据映射

可以自行指定字段类型和数据库列类型之间的数据类型映射。

在某些业务场景下，这个功能非常有用，例如，我们希望将数据库中数字列在生成结构体时都定义为`int64`类型。

```go
var dataMap = map[string]func(gorm.ColumnType) (dataType string){
  // int mapping
  "int": func(columnType gorm.ColumnType) (dataType string) {
    if n, ok := columnType.Nullable(); ok && n {
      return "*int32"
    }
    return "int32"
  },

  // bool mapping
  "tinyint": func(columnType gorm.ColumnType) (dataType string) {
    ct, _ := columnType.ColumnType()
    if strings.HasPrefix(ct, "tinyint(1)") {
      return "bool"
    }
    return "byte"
  },
}

g.WithDataTypeMap(dataMap)
```

#### 从 SQL语句生成结构体

Gen 支持遵循 GORM 约定从 sql 生成结构体，具体用法如下。

```go
package main

import (
	"gorm.io/gen"
	"gorm.io/gorm"
	"gorm.io/rawsql"
)

func main() {
	g := gen.NewGenerator(gen.Config{
		OutPath: "../query",
		Mode:    gen.WithoutContext | gen.WithDefaultQuery | gen.WithQueryInterface, // generate mode
	})
	// https://github.com/go-gorm/rawsql/blob/master/tests/gen_test.go
	gormdb, _ := gorm.Open(rawsql.New(rawsql.Config{
		//SQL:      rawsql,     // create table sql
		FilePath: []string{
			//"./sql/user.sql", // create table sql file
			"./test_sql", // create table sql file directory
		},
	}))
	g.UseDB(gormdb) // reuse your gorm db

	// Generate basic type-safe DAO API for struct `model.User` following conventions

	g.ApplyBasic(
		// Generate struct `User` based on table `users`
		g.GenerateModel("users"),

		// Generate struct `Employee` based on table `users`
		g.GenerateModelAs("users", "Employee"),
	)
	g.ApplyBasic(
		// Generate structs from all tables of current database
		g.GenerateAllTable()...,
	)
	// Generate the code
	g.Execute()
}
```

关于Gen框架的更多技巧，推荐查看[官方文档](https://gorm.io/gen/)。