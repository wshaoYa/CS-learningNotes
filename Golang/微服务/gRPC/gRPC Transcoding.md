# gRPC Transcoding

本文介绍了如何使用 `.proto` 文件中的注释来指定从 HTTP/JSON 到 gRPC 的数据转换。

# gRPC Transcoding

> 可以使用两种方式指定数据应如何从 HTTP/JSON 转换为 gRPC：使用 `.proto` 文件中的直接注释，以及在属于 gRPC API 配置文件的 YAML 中配置。本文只介绍在`.proto` 文件中添加注释的方式。

gRPC 转码（RPC Transcoding）是一种用于在 gRPC 方法和一个或多个 HTTP REST 端点之间进行映射的功能。 它允许开发人员构建一个同时支持 gRPC API 和 REST API 的 API 服务。许多系统，包括[Google APIs](https://github.com/googleapis/googleapis)，[Cloud Endpoints](https://cloud.google.com/endpoints)，[gRPC Gateway](https://github.com/grpc-ecosystem/grpc-gateway)和[Envoy](https://github.com/envoyproxy/envoy) 代理都支持此功能并将其用于大规模生产服务。

`HttpRule` 定义了 gRPC/REST 映射的模式。 映射指定 gRPC 请求消息的不同部分如何映射到 URL 路径、URL 查询参数和 HTTP 请求正文。 它还控制 gRPC 响应消息如何映射到 HTTP 响应正文。 `HttpRule` 通常被指定为 gRPC 方法上的 `google.api.http` 注释。

每个映射指定一个 URL 路径模板和一个 HTTP 方法。 路径模板可以引用 gRPC 请求消息中的一个或多个字段，只要每个字段是原始（非消息）类型的非重复字段即可。 路径模板控制请求消息的字段如何映射到 URL 路径。

例如：

```protobuf
service Messaging {
  rpc GetMessage(GetMessageRequest) returns (Message) {
    option (google.api.http) = {
        get: "/v1/{name=messages/*}"
    };
  }
}
message GetMessageRequest {
  string name = 1; // Mapped to URL path.
}
message Message {
  string text = 1; // The resource content.
}
```

这使得 HTTP REST 到 gRPC 的映射如下所示:

|           HTTP            |                 gRPC                  |
| :-----------------------: | :-----------------------------------: |
| `GET /v1/messages/123456` | `GetMessage(name: "messages/123456")` |

如果没有 HTTP 请求正文，则请求消息中没有被路径模板绑定的任何字段都会自动成为 HTTP 查询参数。

例如：

```protobuf
service Messaging {
    rpc GetMessage(GetMessageRequest) returns (Message) {
    option (google.api.http) = {
        get:"/v1/messages/{message_id}"
    };
    }
}
message GetMessageRequest {
    message SubMessage {
    string subfield = 1;
    }
    string message_id = 1; // Mapped to URL path.
    int64 revision = 2;    // Mapped to URL query parameter `revision`.
    SubMessage sub = 3;    // Mapped to URL query parameter `sub.subfield`.
}
```

这使得 HTTP JSON 到 RPC 的映射如下所示。

|                         HTTP                          |                             gRPC                             |
| :---------------------------------------------------: | :----------------------------------------------------------: |
| `GET /v1/messages/123456?revision=2&sub.subfield=foo` | `GetMessage(message_id: "123456" revision: 2 sub: SubMessage(subfield:"foo"))` |

请注意，映射到 URL 查询参数的字段必须具有原始类型或重复的原始类型或非重复的消息类型。 在重复类型的情况下，参数可以在URL中重复为`...?param=A&param=B`。在消息类型的情况下，消息的每个字段都映射到一个单独的参数，例如 作为`...?foo.a=A&foo.b=B&foo.c=C`。

对于允许请求正文（request body）的 HTTP 方法，`body`字段指定映射关系。下面是一个资源集合message的更新方法。

```protobuf
service Messaging {
    rpc UpdateMessage(UpdateMessageRequest) returns (Message) {
    option (google.api.http) = {
        patch: "/v1/messages/{message_id}"
        body: "message"
    };
    }
}
message UpdateMessageRequest {
    string message_id = 1; // mapped to the URL
    Message message = 2;   // mapped to the body
}
```

此时 HTTP JSON 到 RPC 映射如下，其中请求正文（request body）中 JSON 的表示由 protos JSON 编码决定：

|                     HTTP                      |                             gRPC                             |
| :-------------------------------------------: | :----------------------------------------------------------: |
| `PATCH /v1/messages/123456 { "text": "Hi!" }` | `UpdateMessage(message_id:"123456" message { text: "Hi!" })` |

特殊名称 `*` 可用于主体映射来定义不受路径模板绑定的每个字段都应映射到请求正文。更新方法可以替换为以下定义。

```protobuf
service Messaging {
    rpc UpdateMessage(Message) returns (Message) {
    option (google.api.http) = {
        patch: "/v1/messages/{message_id}"
        body: "*"
    };
    }
}
message Message {
    string message_id = 1;
    string text = 2;
}
```

此时 HTTP JSON 到 RPC 的映射:

|                     HTTP                      |                       gRPC                       |
| :-------------------------------------------: | :----------------------------------------------: |
| `PATCH /v1/messages/123456 { "text": "Hi!" }` | `UpdateMessage(message_id:"123456" text: "Hi!")` |

请注意，在正文映射中使用 `*` 时，不可能有 HTTP 参数，因为所有不受路径绑定的字段都在正文中结束。这使得该选项在定义 REST API 时很少在实践中使用。 `*` 的常见用法是在根本不使用 URL 来传输数据的自定义方法中。

可以使用 `additional_bindings` 选项为一个 RPC 定义多个 HTTP 方法。例如：

```protobuf
service Messaging {
    rpc GetMessage(GetMessageRequest) returns (Message) {
    option (google.api.http) = {
        get: "/v1/messages/{message_id}"
        additional_bindings {
        get: "/v1/users/{user_id}/messages/{message_id}"
        }
    };
    }
}
message GetMessageRequest {
    string message_id = 1;
    string user_id = 2;
}
```

这启用了以下两种可选的 HTTP JSON 到 RPC 映射：

|                HTTP                |                      gRPC                       |
| :--------------------------------: | :---------------------------------------------: |
|     `GET /v1/messages/123456`      |       `GetMessage(message_id: "123456")`        |
| `GET /v1/users/me/messages/123456` | `GetMessage(user_id: "me" message_id:"123456")` |

## HTTP映射的规则

1. 叶请求字段（请求消息中的递归扩展嵌套消息）分为三类
   - 由路径模板引用的字段。它们通过 URL 路径传递。
   - `[HttpRule.body][google.api.HttpRule.body]` 引用的字段。 它们通过 HTTP 请求正文传递。
   - 所有其他字段都是通过 URL 查询参数传递的，参数名称是请求消息中的字段路径。 一个重复的字段可以表示为同名的多个查询参数。
2. 如果 `[HttpRule.body][google.api.HttpRule.body]` 为“*”，则没有 URL 查询参数，所有字段都通过 URL 路径和 HTTP 请求正文传递。
3. 如果 `[HttpRule.body][google.api.HttpRule.body]` 省略，则没有 HTTP 请求正文，所有字段都通过 URL 路径和 URL 查询参数传递。

### 路径模板（path template）语法

```
Template = "/" Segments [ Verb ] ;
Segments = Segment { "/" Segment } ;
Segment  = "*" | "**" | LITERAL | Variable ;
Variable = "{" FieldPath [ "=" Segments ] "}" ;
FieldPath = IDENT { "." IDENT } ;
Verb     = ":" LITERAL ;
```

语法 `*` 匹配单个 URL 路径段。 语法 `**` 匹配零个或多个 URL 路径段，它们必须是 URL 路径的最后一部分，除了 `Verb`。

语法 `Variable` 匹配其模板指定的部分 URL 路径。 变量模板不得包含其他变量。 如果变量匹配单个路径段，则可以省略其模板，例如 `{var}` 等价于 `{var=*}`。

语法 `LITERAL` 匹配 URL 路径中的文字文本。 如果 `LITERAL` 包含任何保留字符，则此类字符应在匹配之前进行URL编码。

如果一个变量只包含一个路径段，例如 `"{var}"` 或 `"{var=*}"`，当这样的变量在客户端扩展为 URL 路径时，除了 `[-_.~0-9a-zA-Z]` 之外的所有字符都是URL编码的。 服务器端进行反向解码。

如果一个变量包含多个路径段，例如`"{var=foo/*}"`或`"{var=**}"`，当这样的变量在客户端展开为URL路径时，所有字符 除了 `[-_.~/0-9a-zA-Z]` 是URL编码的。 服务器端进行反向解码，除了 “%2F” 和 “%2f”(`/`) 保持不变。

## 注意事项

1. 路径变量**不得**引用任何repeated或mapped的字段，因为客户端库无法处理此类变量扩展。
2. 路径变量**不得**捕获前导“/”字符。 原因是最常见的用例“{var}”没有捕获前导“/”字符。 为了一致性，所有路径变量必须共享相同的行为。
3. 不能将重复的消息字段（repeated message）映射到 URL 查询参数，因为没有客户端库可以支持如此复杂的映射。
4. 如果 API 需要为请求或响应正文使用 JSON 数组，它可以将请求或响应正文映射到repeated字段。 但是，某些 gRPC 转码实现可能不支持此功能。

|         字段          |                              值                              |
| :-------------------: | :----------------------------------------------------------: |
|       selector        | string选择应用此规则的方法。有关语法详细信息，请参阅[selector](https://cloud.google.com/endpoints/docs/grpc-service-config/reference/rpc/google.api#google.api.DocumentationRule.FIELDS.string.google.api.DocumentationRule.selector)。 |
|         body          | string请求字段的名称，其值映射到 HTTP 请求正文，或*用于将路径模式未捕获的所有请求字段映射到 HTTP 正文，或因没有任何 HTTP 请求正文而省略。注意：引用字段必须出现在请求消息类型的顶层。 |
|     response_body     | string可选的。其值映射到 HTTP 响应正文的响应字段的名称。省略时，整个响应消息将用作 HTTP 响应正文。注意：引用字段必须出现在响应消息类型的顶层。 |
| additional_bindings[] | [HttpRule](https://cloud.google.com/endpoints/docs/grpc-service-config/reference/rpc/google.api#google.api.HttpRule)选择器的附加 HTTP 绑定。嵌套绑定本身不能包含additional_bindings字段（也就是说，嵌套可能只有一层深） |
|   allow_half_duplex   | bool当此标志设置为 true 时，将允许 HTTP 请求调用半双工流方法。 |
|    联合字段模式。     | 确定 URL 模式是否与此规则匹配。此模式可以与任何 {get\|put\|post\|delete\|patch} 方法一起使用。可以使用“自定义”字段定义自定义方法。pattern只能是以下之一： |
|          get          |    string映射到 HTTP GET。用于列出和获取有关资源的信息。     |
|          put          |            string映射到 HTTP PUT。用于替换资源。             |
|         post          |       string映射到 HTTP POST。用于创建资源或执行操作。       |
|        delete         |           string映射到 HTTP DELETE。用于删除资源。           |
|         patch         |           string映射到 HTTP PATCH。用于更新资源。            |
|        custom         | [CustomHttpPattern](https://cloud.google.com/endpoints/docs/grpc-service-config/reference/rpc/google.api#google.api.CustomHttpPattern) 自定义模式用于指定未包含在pattern字段中的 HTTP 方法，例如 HEAD，或“*”以使该规则未指定 HTTP 方法。通配符规则对于向 Web (HTML) 客户端提供内容的服务很有用。 |

## 示例

转码涉及将 HTTP/JSON 请求及其参数映射到 gRPC 方法及其参数和返回类型。因此，尽管可以将 HTTP/JSON 请求映射到任意 API 方法，但如果你以面向资源的方式设计 gRPC API 的结构（就像传统 HTTP REST API 一样），则有助于实现映射。换句话说，你可设计 API 服务，让其使用少量标准方法，并与操作该服务的资源和资源集合（本身是一种资源类型）的 GET 和 PUT 等 HTTP 谓词相对应。这些标准方法包括 `List`、`Get`、`Create`、`Update` 和 `Delete`。

我们在这里使用一个包含书架（Shelves）和图书（book）的 Bookstore 系统作为示例，Bookstore 具有“图书”资源的“书架”集合，用户可以执行 `List`、`Get`、`Create` 或 `Delete` 方法。我们将演示如何在`.proto`文件中为具体的方法编写gRPC HTTP 映射注释。

### 映射List方法

Bookstore 中有一个列出所有书架的`ListShelves`方法，该`List` 方法及其响应类型在 `.proto` 文件中的定义如下：

```protobuf
  // 返回书店中的书架列表
  rpc ListShelves(google.protobuf.Empty) returns (ListShelvesResponse) {
    // 定义 HTTP 映射关系。
    //   curl http://DOMAIN_NAME/v1/shelves
    option (google.api.http) = { get: "/v1/shelves" };
  }
...
message ListShelvesResponse {
  // 书店中的书架
  repeated Shelf shelves = 1;
}
```

其中：

- `option (google.api.http)` 指定此方法是一个 gRPC HTTP 映射注释。
- `get` 指定此方法映射到 HTTP `GET` 请求。
- `"/v1/shelves"` 是 `GET` 请求在调用该方法时使用的网址路径模板（附加到服务的网域）。网址路径也称为资源路径，因为它通常用于指定你要使用的“对象”或资源。在本例中是指我们的 Bookstore 的所有书架资源。

### 映射 Get 方法

Bookstore 的 `GetShelf` 方法及其请求和响应类型是在 `.proto` 文件中定义的：

```protobuf
// 返回指定的书架
rpc GetShelf(GetShelfRequest) returns (Shelf) {
// 返回第一个书架:
// curl http://DOMAIN_NAME/v1/shelves/1
option (google.api.http) = { get: "/v1/shelves/{shelf}" };
}

...
// GetShelf 方法的请求消息
message GetShelfRequest {
// 要检索的书架资源的 ID。
int64 shelf = 1;
}
...
// 一个书架资源
message Shelf {
// 唯一的书架 id.
int64 id = 1;
// 书架的主题(小说、诗歌等)。
string theme = 2;
}
```

其中：

- `option (google.api.http)` 指定此方法是一个 gRPC HTTP 映射注释。
- `get` 指定此方法映射到 HTTP `GET` 请求。
- 如前所述，`"/v1/shelves/{shelf}"` 是请求的网址路径，但它先指定 `/v1/shelves/`，然后指定 `{shelf}`。`{shelf}` 中的任何内容都是此方法的 `GetShelfRequest` 参数中 `shelf` 的值。

如果客户端通过向网址 `http://mydomain/v1/shelves/4` 发送 `GET` 来调用此方法，则会创建一个 `shelf` 值为 `4` 的 `GetShelfRequest`，然后通过该请求调用 gRPC 方法 `GetShelf()`。然后，gRPC 后端返回所请求的 ID 为 `4` 的 `Shelf`，再将其转换为 JSON 格式并返回给客户端。

此方法只需要客户端 `shelf` 提供一个请求字段值，该值是你在采用花括号“捕获型”表示法的网址路径模板中指定的。如有必要，你还可以捕获网址的多个部分，以识别请求的资源。例如，`GetBook` 方法需要客户端在网址中同时指定书架 ID 和图书 ID：

```protobuf
// 返回一本指定的图书
rpc GetBook(GetBookRequest) returns (Book) {
// 从第二个书架上拿第一本书：
//   curl http://DOMAIN_NAME/v1/shelves/2/books/1
option (google.api.http) = { get: "/v1/shelves/{shelf}/books/{book}" };
}
...
// GetBook 方法的请求消息。
message GetBookRequest {
// 要从中检索图书的书架 ID。
int64 shelf = 1;
// 要检索的图书的 ID。
int64 book = 2;
}
```

除了将文字和捕获型括号用于字段值之外，网址路径模板还可使用通配符指示应捕获该网址部分中的全部内容。上述示例中使用的 `{shelf}` 表示法实际上是 `{shelf=*}` 的快捷方式。

### 映射 Create 方法

Bookstore 的 `CreateShelf` 方法映射到 HTTP `POST`。

```protobuf
  // 在书店里创建一个新的书架。
  rpc CreateShelf(CreateShelfRequest) returns (Shelf) {
    // Client example:
    //   curl -d '{"theme":"Music"}' http://DOMAIN_NAME/v1/shelves
    option (google.api.http) = {
      post: "/v1/shelves"
      body: "shelf"
    };
  }
...
// CreateShelf 方法的请求消息。
message CreateShelfRequest {
  // 要创建的书架资源。
  Shelf shelf = 1;
}
...
// 一个书架资源。
message Shelf {
  // 唯一的书架id。
  int64 id = 1;
  // 书架上的主题(小说、诗歌等)。
  string theme = 2;
}
```

其中：

- `option (google.api.http)` 指定此方法是一个 gRPC HTTP 映射注释。
- `post` 指定此方法映射到 HTTP `POST` 请求。
- 如前所述，`"/v1/shelves"` 是请求的网址路径。
- `body: "shelf"` 在 HTTP 请求正文中用来以 JSON 格式指定你要添加的资源。

例如，如果客户端按如下所示的方式调用该方法：

```bash
curl -d '{"theme":"Music"}' http://DOMAIN_NAME/v1/shelves
```

这将使用 JSON 正文为 `CreateShelfRequest` 创建一个主题为 `"Music"` 的 `Shelf` 值，然后调用 gRPC `CreateShelf()` 方法。请注意，客户端不提供 `Shelf` 的 `id` 值。Bookstore 的书架 ID 由相关服务在创建新书架时提供。你应在 API 文档中为服务用户提供此类信息。

### 在body中使用通配符

在body映射中，可以使用特殊名称 `*` 来指示不受路径模板约束的每个字段应该映射到请求正文。此方式支持以下备用的 `CreateShelf` 方法定义。

```protobuf
  // 在书店里创建一个新的书架。
  rpc CreateShelf(CreateShelfRequest) returns (Shelf) {
    // Client example:
    //   curl -d '{"shelf_theme":"Music", "shelf_size": 20}' http://DOMAIN_NAME/v1/shelves/123
    option (google.api.http) = {
      post: "/v1/shelves/{shelf_id}"
      body: "*"
    };
  }
...
// CreateShelf 方法的请求消息。
message CreateShelfRequest {
  // 一个唯一的书架id。
  int64 shelf_id = 1;
  // 书架上的主题(小说、诗歌等)。
  string shelf_theme = 2;
  // 架子的大小
  int64 shelf_size = 3;
}
```

其中：

- `option (google.api.http)` 指定此方法是一个 gRPC HTTP 映射注释。
- `post` 指定此方法映射到 HTTP `POST` 请求。
- `"/v1/shelves/{shelf_id}"` 是该请求的网址路径。`{shelf_id}`中的内容就是 `shelf_id` 字段在`CreateShelfRequest`中的值。
- `body: "*"` 在 HTTP 请求正文中用于指定此示例中除 `shelf_id` 之外的所有剩余请求字段，这些字段是 `shelf_theme` 和 `shelf_size`。对于 JSON 正文中具有这两个名称的任何字段，其值都将在 `CreateShelfRequest` 的相应字段中使用。

例如，如果客户端按如下方式调用该方法：

```bash
curl -d '{"shelf_theme":"Music", "shelf_size": 20}' http://DOMAIN_NAME/v1/shelves/123
```

这将使用 JSON 正文和路径模板来创建 `CreateShelfRequest{shelf_id: 123 shelf_theme: "Music" shelf_size: 20}`，然后使用该模板调用 gRPC `CreateShelf()` 方法。

参考资料

1. https://cloud.google.com/endpoints/docs/grpc-service-config/reference/rpc/google.api#google.api.HttpRule
2. https://cloud.google.com/endpoints/docs/grpc/transcoding