---
title: Cookie和Session
tags: [Cookie, Session, Web开发]
categories: [Go]
date: 2019-08-31 00:00:00
toc: true
---

`Cookie`和`Session`是`Web`开发绕不开的一个环节，本文介绍了`Cookie`和`Session`的原理及在`Go`语言中如何操作`Cookie`。<!--more-->

## Cookie

### Cookie的由来

`HTTP`协议是无状态的，这就存在一个问题。

无状态的意思是每次请求都是独立的，它的执行情况和结果与前面的请求和之后的请求都无直接关系，它不会受前面的请求响应情况直接影响，也不会直接影响后面的请求响应情况。

一句有意思的话来描述就是人生只如初见，对服务器来说，每次的请求都是全新的。

状态可以理解为客户端和服务器在某次会话中产生的数据，那无状态的就以为这些数据不会被保留。会话中产生的数据又是我们需要保存的，也就是说要“保持状态”。因此`Cookie`就是在这样一个场景下诞生。

### Cookie是什么

在`Internet`中，`Cookie` 实际上是指小量信息，是由 `Web` 服务器创建的，将信息存储在用户计算机上（客户端）的数据文件。一般网络用户习惯用其复数形式 `Cookies`，指某些网站为了辨别用户身份、进行 `Session` 跟踪而存储在用户本地终端上的数据，而这些数据通常会经过加密处理。

### Cookie的机制

`Cookie`是由服务器端生成，发送给`User-Agent`（一般是浏览器），浏览器会将`Cookie`的`key/value`保存到某个目录下的文本文件内，下次请求同一网站时就发送该`Cookie`给服务器（前提是浏览器设置为启用`cookie`）。`Cookie`名称和值可以由服务器端开发自己定义，这样服务器可以知道该用户是否是合法用户以及是否需要重新登录等，服务器可以设置或读取`Cookies`中包含信息，借此维护用户跟服务器会话中的状态。

总结一下`Cookie`的特点：

1. 浏览器发送请求的时候，自动把携带该站点之前存储的`Cookie`信息。
2. 服务端可以设置`Cookie`数据。
3. `Cookie`是针对单个域名的，不同域名之间的`Cookie`是独立的。
4. `Cookie`数据可以配置过期时间，过期的`Cookie`数据会被系统清除。

### 查看Cookie

我们使用`Chrome`浏览器打开一个网站，打开开发者工具查看该网站保存在我们电脑上的`Cookie`数据。

## Go操作Cookie

### Cookie

标准库`net/http`中定义了`Cookie`，它代表一个出现在`HTTP`响应头中`Set-Cookie`的值里或者`HTTP`请求头中`Cookie`的值的`HTTP cookie`。

```go
type Cookie struct {
    Name       string
    Value      string
    Path       string
    Domain     string
    Expires    time.Time
    RawExpires string
    // MaxAge=0表示未设置Max-Age属性
    // MaxAge<0表示立刻删除该cookie，等价于"Max-Age: 0"
    // MaxAge>0表示存在Max-Age属性，单位是秒
    MaxAge   int
    Secure   bool
    HttpOnly bool
    Raw      string
    Unparsed []string // 未解析的“属性-值”对的原始文本
}
```

### 设置Cookie

`net/http`中提供了如下`SetCookie`函数，它在`w`的头域中添加`Set-Cookie`头，该`HTTP`头的值为`cookie`。

```go
func SetCookie(w ResponseWriter, cookie *Cookie)
```

### 获取Cookie

`Request`对象拥有两个获取`Cookie`的方法和一个添加`Cookie`的方法：

获取`Cookie`的两种方法：

```go
// 解析并返回该请求的Cookie头设置的所有cookie
func (r *Request) Cookies() []*Cookie

// 返回请求中名为name的cookie，如果未找到该cookie会返回nil, ErrNoCookie。
func (r *Request) Cookie(name string) (*Cookie, error)
```

添加`Cookie`的方法：

```go
// AddCookie向请求中添加一个cookie。
func (r *Request) AddCookie(c *Cookie)
```

## gin框架操作Cookie

```go
import (
    "fmt"

    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()
    router.GET("/cookie", func(c *gin.Context) {
        cookie, err := c.Cookie("gin_cookie") // 获取Cookie
        if err != nil {
            cookie = "NotSet"
            // 设置Cookie
            c.SetCookie("gin_cookie", "test", 3600, "/", "localhost", false, true)
        }
        fmt.Printf("Cookie value: %s \n", cookie)
    })

    router.Run()
}
```

## Session

### Session的由来

`Cookie`虽然在一定程度上解决了“保持状态”的需求，但是由于`Cookie`本身最大支持`4096`字节，以及`Cookie`本身保存在客户端，可能被拦截或窃取，因此就需要有一种新的东西，它能支持更多的字节，并且他保存在服务器，有较高的安全性。这就是`Session`。

问题来了，基于`HTTP`协议的无状态特征，服务器根本就不知道访问者是“谁”。那么上述的`Cookie`就起到桥接的作用。

用户登陆成功之后，我们在服务端为每个用户创建一个特定的`session`和一个唯一的标识，它们一一对应。其中：

- `Session`是在服务端保存的一个数据结构，用来跟踪用户的状态，这个数据可以保存在集群、数据库、文件中；
- 唯一标识通常称为`Session ID`会写入用户的`Cookie`中。

这样该用户后续再次访问时，请求会自动携带`Cookie`数据（其中包含了`Session ID`），服务器通过该`Session ID`就能找到与之对应的`Session`数据，也就知道来的人是“谁”。

总结而言：`Cookie`弥补了`HTTP`无状态的不足，让服务器知道来的人是“谁”；但是`Cookie`以文本的形式保存在本地，自身安全性较差；所以我们就通过`Cookie`识别不同的用户，对应的在服务端为每个用户保存一个`Session`数据，该`Session`数据中能够保存具体的用户数据信息。

另外，上述所说的`Cookie`和`Session`其实是共通性的东西，不限于语言和框架。

# 练习题

1. 编写代码实现一个`gin`框架版`Session`中间件。