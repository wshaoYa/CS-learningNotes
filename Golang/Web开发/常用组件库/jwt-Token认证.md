# 用户认证

HTTP 是⼀个⽆状态的协议，⼀次请求结束后，下次在发送服务器就不知道这个请求是谁发来的了（同⼀个 IP 不代表同⼀个⽤户），在 Web 应⽤中，⽤户的认证和鉴权是⾮常重要的⼀环，实践中有多种可⽤⽅案，并且各有千秋。

# Cookie- Session 认证模式

在 Web 应⽤发展的初期，⼤部分采⽤基于Cookie-Session 的会话管理⽅式，逻辑如下。

- 客户端 使⽤⽤户名、密码进⾏认证
- 服务端 验证⽤户名、密码正确后⽣成并存储 Session，将 SessionID 通过 Cookie 返回给客户端
- 客户端 访问需要认证的接⼝时在 Cookie 中携带 SessionID
- 服务端 通过 SessionID 查找 Session 并进⾏鉴权，返回给客户端需要的数据

<img src="https://s2.loli.net/2023/11/28/6qfbUOFXvcj5LrM.png" alt="image-20231128105735669" style="zoom: 80%;" />

基于 Session 的⽅式存在多种问题。

- 服务端需要存储 Session，并且由于 Session 需要经常快速查找，通常存储在内存或内存数据库中，同时在线⽤户较多时需要占⽤⼤量的服务器资源。
- 当需要扩展时，创建 Session 的服务器可能不是验证 Session 的服务器，所以还需要将所有Session 单独存储并共享。
- 由于客户端使⽤ Cookie 存储 SessionID，在跨域场景下需要进⾏兼容性处理，同时这种⽅式也难以防范 CSRF 攻击。

# Token 认证模式

鉴于基于 Session 的会话管理⽅式存在上述多个缺点，基于 Token 的⽆状态会话管理⽅式诞⽣了，所谓⽆状态，就是服务端可以不再存储信息，甚⾄是不再存储 Session，逻辑如下。

- 客户端 使⽤⽤户名、密码进⾏认证
- 服务端 验证⽤户名、密码正确后⽣成 Token 返回给客户端
- 客户端 保存 Token，访问需要认证的接⼝时在 URL 参数或 HTTP Header 中加⼊ Token
- 服务端 通过解码 Token 进⾏鉴权，返回给客户端需要的数据

![image-20231128105921737](https://s2.loli.net/2023/11/28/AhEXdz5CpZI8mrM.png)

基于 Token 的会话管理⽅式有效解决了基于 Session 的会话管理⽅式带来的问题。

- 服务端不需要存储和⽤户鉴权有关的信息，鉴权信息会被加密到 Token 中，服务端只需要读取Token 中包含的鉴权信息即可
- 避免了共享 Session 导致的不易扩展问题
- 不需要依赖 Cookie，有效避免 Cookie 带来的 CSRF 攻击问题
- 使⽤ CORS 可以快速解决跨域问题

# JWT

## 介绍

JWT全称JSON Web Token是一种跨域认证解决方案，属于一个开放的标准，它规定了一种Token 实现方式，目前多用于前后端分离项目和 OAuth2.0 业务场景下。

JWT 是 JSON Web Token 的缩写，是为了在⽹络应⽤环境间传递声明⽽执⾏的⼀种基于JSON的开放标
准（(RFC 7519)。JWT 本身没有定义任何技术实现，它只是定义了⼀种基于 Token 的会话管理的规则，
涵盖 Token 需要包含的标准内容和 Token 的⽣成过程，特别适⽤于分布式站点的单点登录（SSO）场
景。

⼀个 JWT Token 就像这样：

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoyODAxODcyNzQ4ODMyMzU4NSwiZ
XhwIjoxNTk0NTQwMjkxLCJpc3MiOiJibHVlYmVsbCJ9.lk_ZrAtYGCeZhK3iupHxP1kgjBJzQTVTtX
0iZYFx9wU
```

它是由` . `分隔的三部分组成，这三部分依次是：

- 头部（Header）
- 负载（Payload）
- 签名（Signature）

头部和负载以 JSON 形式存在，这就是 JWT 中的 JSON，三部分的内容都分别单独经过了 Base64 编

码，以 `.` 拼接成⼀个 JWT Token。

![image-20231203101917056](https://s2.loli.net/2023/12/03/W2Rse1dTkCatY78.png)

### **Header**

JWT 的 Header 中存储了所使⽤的加密算法和 Token 类型。

```json
{
 "alg": "HS256",
 "typ": "JWT"
}
```

### Payload

Payload 表示负载，也是⼀个 JSON 对象，JWT 规定了7个官⽅字段供选⽤

```
iss (issuer)：签发⼈
exp (expiration time)：过期时间
sub (subject)：主题
aud (audience)：受众
nbf (Not Before)：⽣效时间
iat (Issued At)：签发时间
jti (JWT ID)：编号
```

除了官⽅字段，开发者也可以⾃⼰指定字段和内容，例如下⾯的内容。

```json
{
 "sub": "1234567890",
 "name": "John Doe",
 "admin": true
}
```

注意，JWT 默认是不加密的，任何⼈都可以读到，所以不要把秘密信息放在这个部分。这个 JSON 对象
也要使⽤ Base64URL 算法转成字符串。

### Signature

Signature 部分是对前两部分的签名，防⽌数据篡改。

⾸先，需要指定⼀个密钥（secret）。这个密钥只有服务器才知道，不能泄露给⽤户。然后，使⽤
Header ⾥⾯指定的签名算法（默认是 HMAC SHA256），按照下⾯的公式产⽣签名。

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload),secret)
```

## 为什么需要JWT？

在之前的一些web项目中，我们通常使用的是`Cookie-Session`模式实现用户认证。相关流程大致如下：

1. 用户在浏览器端填写用户名和密码，并发送给服务端
2. 服务端对用户名和密码校验通过后会生成一份保存当前用户相关信息的session数据和一个与之对应的标识（通常称为session_id）
3. 服务端返回响应时将上一步的session_id写入用户浏览器的Cookie
4. 后续用户来自该浏览器的每次请求都会自动携带包含session_id的Cookie
5. 服务端通过请求中的session_id就能找到之前保存的该用户那份session数据，从而获取该用户的相关信息。

这种方案依赖于客户端（浏览器）保存 Cookie，并且需要在服务端存储用户的session数据。

在移动互联网时代，我们的用户可能使用浏览器也可能使用APP来访问我们的服务，我们的web应用可能是前后端分开部署在不同的端口，有时候我们还需要支持第三方登录，这下`Cookie-Session`的模式就有些力不从心了。

JWT就是一种基于Token的轻量级认证模式，服务端认证通过后，会生成一个JSON对象，经过签名后得到一个Token（令牌）再发回给用户，用户后续请求只需要带上这个Token，服务端解密之后就能获取该用户的相关信息了。

想要了解JWT的原理，推荐大家阅读：[阮一峰的JWT入门教程](https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)

## 优缺点

JWT 拥有基于 Token 的会话管理⽅式所拥有的⼀切优势，不依赖 Cookie，使得其可以防⽌ CSRF 攻
击，也能在禁⽤ Cookie 的浏览器环境中正常运⾏

⽽ JWT 的最⼤优势是服务端不再需要存储 Session，使得服务端认证鉴权业务可以⽅便扩展，避免存储
Session 所需要引⼊的 Redis 等组件，降低了系统架构复杂度。但这也是 JWT 最⼤的劣势，由于有效期
存储在 Token 中，JWT Token ⼀旦签发，就会在有效期内⼀直可⽤，⽆法在服务端废⽌，当⽤户进⾏登
出操作，只能依赖客户端删除掉本地存储的 JWT Token，如果需要禁⽤⽤户，单纯使⽤ JWT 就⽆法做到
了。

## 安装

我们使用 Go 语言社区中的 jwt 相关库来构建我们的应用，例如：https://github.com/golang-jwt/jwt。

```bash
go get -u github.com/golang-jwt/jwt/v5
```

本文将使用这个库来实现我们生成JWT和解析JWT的功能。

## 使用

### 默认Claim

如果我们直接使用JWT中默认的字段，没有其他定制化的需求则可以直接使用这个包中的和方法快速生成和解析token。

```go
// 用于签名的字符串
var mySigningKey = []byte("liwenzhou.com")

// GenRegisteredClaims 使用默认声明创建jwt
func GenRegisteredClaims() (string, error) {
	// 创建 Claims
	claims := &jwt.RegisteredClaims{
		ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Hour * 24)), // 过期时间
		Issuer:    "qimi",                                             // 签发人
	}
	// 生成token对象
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	// 生成签名字符串
	return token.SignedString(mySigningKey)
}

// ParseRegisteredClaims 解析jwt
func ValidateRegisteredClaims(tokenString string) bool {
	// 解析token
	token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
		return mySigningKey, nil
	})
	if err != nil { // 解析token失败
		return false
	}
	return token.Valid
}
```

### 自定义Claims

我们需要定制自己的需求来决定JWT中保存哪些数据，比如我们规定在JWT中要存储`username`信息，那么我们就定义一个`MyClaims`结构体如下：

```go
// CustomClaims 自定义声明类型 并内嵌jwt.RegisteredClaims
// jwt包自带的jwt.RegisteredClaims只包含了官方字段
// 假设我们这里需要额外记录一个username字段，所以要自定义结构体
// 如果想要保存更多信息，都可以添加到这个结构体中
type CustomClaims struct {
	// 可根据需要自行添加字段
	Username             string `json:"username"`
	jwt.RegisteredClaims        // 内嵌标准的声明
}
```

然后我们定义JWT的过期时间，这里以24小时为例：

```go
const TokenExpireDuration = time.Hour * 24
```

接下来还需要定义一个用于签名的字符串：

```go
// CustomSecret 用于加盐的字符串
var CustomSecret = []byte("夏天夏天悄悄过去")
```

### 生成JWT

我们可以根据自己的业务需要封装一个生成 token 的函数。

```go
// GenToken 生成JWT
func GenToken(username string) (string, error) {
	// 创建一个我们自己的声明
	claims := CustomClaims{
		username, // 自定义字段
		jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(TokenExpireDuration)),
			Issuer:    "my-project", // 签发人
		},
	}
	// 使用指定的签名方法创建签名对象
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	// 使用指定的secret签名并获得完整的编码后的字符串token
	return token.SignedString(CustomSecret)
}
```

### 解析JWT

根据给定的 JWT 字符串，解析出数据。

```go
// ParseToken 解析JWT
func ParseToken(tokenString string) (*CustomClaims, error) {
	// 解析token
	// 如果是自定义Claim结构体则需要使用 ParseWithClaims 方法
	token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(token *jwt.Token) (i interface{}, err error) {
		// 直接使用标准的Claim则可以直接使用Parse方法
		//token, err := jwt.Parse(tokenString, func(token *jwt.Token) (i interface{}, err error) {
		return CustomSecret, nil
	})
	if err != nil {
		return nil, err
	}
	// 对token对象中的Claim进行类型断言
	if claims, ok := token.Claims.(*CustomClaims); ok && token.Valid { // 校验token
		return claims, nil
	}
	return nil, errors.New("invalid token")
}
```

## gin框架使用JWT

首先我们注册一条路由`/auth`，对外提供获取Token的渠道：

```go
r.POST("/auth", authHandler)
```

我们的`authHandler`定义如下：

```go
func authHandler(c *gin.Context) {
	// 用户发送用户名和密码过来
	var user UserInfo
	err := c.ShouldBind(&user)
	if err != nil {
		c.JSON(http.StatusOK, gin.H{
			"code": 2001,
			"msg":  "无效的参数",
		})
		return
	}
	// 校验用户名和密码是否正确
	if user.Username == "q1mi" && user.Password == "q1mi123" {
		// 生成Token
		tokenString, _ := GenToken(user.Username)
		c.JSON(http.StatusOK, gin.H{
			"code": 2000,
			"msg":  "success",
			"data": gin.H{"token": tokenString},
		})
		return
	}
	c.JSON(http.StatusOK, gin.H{
		"code": 2002,
		"msg":  "鉴权失败",
	})
	return
}
```

用户通过上面的接口获取Token之后，后续就会携带着Token再来请求我们的其他接口，这个时候就需要对这些请求的Token进行校验操作了，很显然我们应该实现一个检验Token的中间件，具体实现如下：

```go
// JWTAuthMiddleware 基于JWT的认证中间件
func JWTAuthMiddleware() func(c *gin.Context) {
	return func(c *gin.Context) {
		// 客户端携带Token有三种方式 1.放在请求头 2.放在请求体 3.放在URI
		// 这里假设Token放在Header的Authorization中，并使用Bearer开头
		// 这里的具体实现方式要依据你的实际业务情况决定
		authHeader := c.Request.Header.Get("Authorization")
		if authHeader == "" {
			c.JSON(http.StatusOK, gin.H{
				"code": 2003,
				"msg":  "请求头中auth为空",
			})
			c.Abort()
			return
		}
		// 按空格分割
		parts := strings.SplitN(authHeader, " ", 2)
		if !(len(parts) == 2 && parts[0] == "Bearer") {
			c.JSON(http.StatusOK, gin.H{
				"code": 2004,
				"msg":  "请求头中auth格式有误",
			})
			c.Abort()
			return
		}
		// parts[1]是获取到的tokenString，我们使用之前定义好的解析JWT的函数来解析它
		mc, err := ParseToken(parts[1])
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"code": 2005,
				"msg":  "无效的Token",
			})
			c.Abort()
			return
		}
		// 将当前请求的username信息保存到请求的上下文c上
		c.Set("username", mc.Username)
		c.Next() // 后续的处理函数可以用过c.Get("username")来获取当前请求的用户信息
	}
}
```

注册一个`/home`路由，发个请求验证一下吧。

```go
r.GET("/home", JWTAuthMiddleware(), homeHandler)

func homeHandler(c *gin.Context) {
	username := c.MustGet("username").(string)
	c.JSON(http.StatusOK, gin.H{
		"code": 2000,
		"msg":  "success",
		"data": gin.H{"username": username},
	})
}
```

如果不想自己实现上述功能，你也可以使用Github上别人封装好的包，比如https://github.com/appleboy/gin-jwt。

## refresh token

在某些业务场景下，我们可能还需要使用refresh token。

这里可以参考 [RFC 6749 OAuth2.0中关于refresh token的介绍](https://datatracker.ietf.org/doc/html/rfc6749#section-1.5)

### **基于jwt实现认证实践**

前⾯讲的 Token，都是 Access Token，也就是访问资源接⼝时所需要的 Token，还有另外⼀种
Token，Refresh Token，通常情况下，Refresh Token 的有效期会⽐较⻓，⽽ Access Token 的有效期
⽐较短，当 Access Token 由于过期⽽失效时，使⽤ Refresh Token 就可以获取到新的 Access Token，
如果 Refresh Token 也失效了，⽤户就只能重新登录了。

在 JWT 的实践中，引⼊ Refresh Token，将会话管理流程改进如下。

- 客户端使⽤⽤户名密码进⾏认证
- 服务端⽣成有效时间较短的 Access Token（例如 10 分钟），和有效时间较⻓的 Refresh Token（例如 7 天）
- 客户端访问需要认证的接⼝时，携带 Access Token
- 如果 Access Token 没有过期，服务端鉴权后返回给客户端需要的数据
- 如果携带 Access Token 访问需要认证的接⼝时鉴权失败（例如返回 401 错误），则客户端使⽤Refresh Token 向刷新接⼝申请新的 Access Token
- 如果 Refresh Token 没有过期，服务端向客户端下发新的 Access Token
- 客户端使⽤新的 Access Token 访问需要认证的接⼝

![image-20231203102401995](https://s2.loli.net/2023/12/03/i2Uh5w8j1WOcJKH.png)

后端需要对外提供⼀个刷新Token的接⼝，前端需要实现⼀个当Access Token过期时⾃动请求刷新
Token接⼝获取新Access Token的拦截器

### 实现

#### **GenToken**

```go
// GenToken ⽣成access token 和 refresh token
func GenToken(userID int64) (aToken, rToken string, err error) {
	// 创建⼀个我们⾃⼰的声明
	c := MyClaims{
		userID, // ⾃定义字段
		jwt.StandardClaims{
			ExpiresAt: time.Now().Add(TokenExpireDuration).Unix(), // 过期时间
			Issuer:    "bluebell",                                 // 签发⼈
		},
	}
	// 加密并获得完整的编码后的字符串token
	aToken, err = jwt.NewWithClaims(jwt.SigningMethodHS256,
		c).SignedString(mySecret)
	// refresh token 不需要存任何⾃定义数据
	rToken, err = jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.StandardClaims{
		ExpiresAt: time.Now().Add(time.Second * 30).Unix(), // 过期时间
		Issuer:    "bluebell",                              // 签发⼈
	}).SignedString(mySecret)
	// 使⽤指定的secret签名并获得完整的编码后的字符串token
	return
}
```

#### RefreshToken

```go
// RefreshToken 刷新AccessToken
func RefreshToken(aToken, rToken string) (newAToken, newRToken string, err error) {
	// refresh token⽆效直接返回
	if _, err = jwt.Parse(rToken, keyFunc); err != nil {
		return
	}
	// 从旧access token中解析出claims数据
	var claims MyClaims
	_, err = jwt.ParseWithClaims(aToken, &claims, keyFunc)
	v, _ := err.(*jwt.ValidationError)
	// 当access token是过期错误 并且 refresh token没有过期时就创建⼀个新的access token
	if v.Errors == jwt.ValidationErrorExpired {
		return GenToken(claims.UserID)
	}
	return
}
```



# 单设备登录

## 方式1

首次登录时，db中将当前userID的token值进行映射绑定存储到db中。

同一时间其他设备登录此同一账号时（token有效前提下），若发现此请求的token与库中此userID对应的token不同，则说明有蹊跷 ovo，后续再进行系列操作等...**（待后续学习补充）**

![image-20231201205342766](https://s2.loli.net/2023/12/01/uQxnpty4f7AiwPo.png)







