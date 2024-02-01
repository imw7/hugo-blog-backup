---
title: 第三方日志库logrus使用
tags: [logrus, 常用组件和技巧]
categories: [Go]
date: 2019-08-29
toc: true
---

日志是程序中必不可少的一个环节，由于`Go`语言内置的日志库功能比较简洁，我们在实际开发中通常会选择使用第三方的日志库进行开发。文本介绍了`logrus`这个日志库的基本使用。<!--more-->

## logrus介绍

`Logrus`是`Go`的结构化`logger`，与标准库l`ogger`完全`API`兼容。

它有以下特点：

* 完全兼容标准日志库，拥有七种日志级别：`Trace`，`Debug`，`Info`，`Warning`，`Error`，`Fatal`和`Panic`。
* 可扩展的`Hook`机制，允许使用者通过Hook的方式将日志分发到任意地方，如本地文件系统，`logstash`，`elasticsearch`或者`mq`等，或者通过`Hook`定义日志内容和格式等。
* 可选的日志输出格式，内置了两种日志格式`JSONFormatter`和`TextFormatter`，还可以自定义日志格式。
* `Field`机制，通过`Field`机制进行结构化的日志记录。
* 线程安全。

## 安装

```bash
go get github.com/sirupsen/logrus
```

## 基本示例

使用`Logrus`最简单的方法是简单的包级导出日志程序：

```go
package main

import (
	log "github.com/sirupsen/logrus"
)

func main() {
	log.WithFields(log.Fields{
		"animal": "dog",
	}).Info("一只舔狗出现了")
}
```

## 进阶示例

对于更高级的用法，例如在同一应用程序记录到多个位置，我们还可以创建`logrus`的实例：

```go
package main

import (
	"github.com/sirupsen/logrus"
	"os"
)

// 创建一个新的logger实例。可以创建任意多个。
var log = logrus.New()

func main() {
	// 设置日志输出为os.Stdout
	log.Out = os.Stdout

	// 可以设置像文件等任意`io.Writer`类型作为日志输出
	// file, err := os.OpenFile("logrus.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
	// if err == nil {
	// 	log.Out = file
	// } else {
	// 	log.Info("Failed to log to file, using default stderr")
	// }

	log.WithFields(logrus.Fields{
		"animal": "dog",
		"size":   10,
	}).Info("一群舔狗出现了")
}
```

## 日志级别

`Logrus`有七个日志级别：`Trace`，`Debug`，`Info`，`Warning`，`Error`，`Fatal`和`Panic`。

```go
log.Trace("Something very low level.")
log.Debug("Useful debugging information.")
log.Info("Something noteworthy happened!")
log.Warn("You should probably take a look at this.")
log.Error("Something failed but I'm not quitting.")
// 记完日志后会调用 os.Exit(1)
log.Fatal("Bye.")
// 记完日志后会调用 panic()
log.Panic("I'm failing.")
```

### 设置日志级别

我们可以在`Logger`上设置日志记录级别，然后它只会记录具有该级别或以上级别任何内容的条目：

```go
// 会记录info及以上级别（warn, error, fatal, panic）
log.SetLevel(log.InfoLevel)
```

如果我们的程序支持`debug`或环境变量模式，设置`log.Level = logrus.DebugLevel`会很有帮助。

## 字段

`Logrus`鼓励通过日志字段进行谨慎的结构化日志记录，而不是冗长的、不可解析的错误消息。

例如，区别于使用`log.Fatalf("Failed to send event %s to topic %s with key %d")`，我们应该使用如下方式记录更容易发现的内容：

```go
log.WithFields(log.Fields{
    "event": event,
    "topic": topic,
    "key":   key,
}).Fatal("Failed to send event")
```

`WithFields`的调用是可选的。

## 默认字段

通常，将一些字段始终附加到应用程序的全部或部分的日志语句中会很有帮助。例如，我们可能希望始终在请求的上下文中记录`request_id`和`user_ip`。

区别于在每一行日志中写上`log.WithFields(log.Fields{"request_id": request_id, "user_ip": user_ip})`，我们可以像下面的示例代码一样创建一个`logrus.Entry`去传递这些字段。

```go
requestLogger := log.WithFields(log.Fields{"request_id": request_id, "user_ip": user_ip})
requestLogger.Info("something happened on that request") // will log request_id and user_ip
requestLogger.Warn("something not great happened")
```

## 日志条目

除了使用`WithField`或`WithFields`添加的字段外，一些字段会自动添加到所有日志记录中：

* `time`：记录日志时的时间戳
* `msg`：记录的日志信息
* `level`：记录的日志级别

## Hooks

我们可以添加日志级别的钩子（`Hook`）。例如，向异常跟踪服务发送`Error`、`Fatal`和`Panic`信息到`StatsD`或同时将日志发送到多个位置，例如`syslog`。

`Logrus`配有内置钩子。在`init`中添加这些内置钩子或我们自定义的钩子：

```go
import (
	log "github.com/sirupsen/logrus"
	logrussyslog "github.com/sirupsen/logrus/hooks/syslog"
	"gopkg.in/gemnasium/logrus-airbrake-hook.v2" // the package is named "airbrake"
	"log/syslog"
)

func init() {

	// Use the Airbrake hook to report errors that have Error severity or above to
	// an exception tracker. You can create custom hooks, see the Hooks section.
	log.AddHook(airbrake.NewHook(123, "xyz", "production"))

	hook, err := logrussyslog.NewSyslogHook("udp", "localhost:514", syslog.LOG_INFO, "")
	if err != nil {
		log.Error("Unable to connect to local syslog daemon")
	} else {
		log.AddHook(hook)
	}
}
```

注：`Syslog`钩子还支持连接到本地`syslog`（例如: “`/dev/log”` or “`/var/run/syslog`” or “`/var/run/log`”)。有关详细信息，请查看[syslog hook README](https://github.com/sirupsen/logrus/blob/master/hooks/syslog/README.md)。

## 格式化

`logrus`内置以下两种日志格式化程序：

`logrus.TextFormatter` `logrus.JSONFormatter`

还支持一些第三方的格式化程序，详见logrus项目首页。

## 记录函数名

如果希望将调用的函数名添加为字段，请通过以下方式设置：

```go
log.SetReportCaller(true)
```

这会将调用者添加为"`method`"，如下所示：

```bash
{"animal":"penguin","level":"fatal","method":"github.com/sirupsen/arcticcreatures.migrate","msg":"a penguin swims by","time":"2014-03-10 19:57:38.562543129 -0400 EDT"}
```

**注意：** 开启这个模式会增加性能开销。

## 线程安全

默认的`logger`在并发写的时候是被`mutex`保护的，比如当同时调用`hook`和写`log`时`mutex`就会被请求，有另外一种情况，文件时以`appending mode`打开的，此时的并发操作就是安全的，可以用`logger.SetNoLock()`来关闭它。

## gin框架使用logrus

```go
// a gin with logrus demo

var log = logrus.New()

func init() {
	// Log as JSON instead of the default ASCII formatter.
	log.Formatter = &logrus.JSONFormatter{}
	// Output to stdout instead of the default stderr
	// Can be any io.Writer, see below for File example
	f, _ := os.Create("./gin.log")
	log.Out = f
	gin.SetMode(gin.ReleaseMode)
	gin.DefaultWriter = log.Out
	// Only log the warning severity or above.
	log.Level = logrus.InfoLevel
}

func main() {
	// 创建一个默认的路由引擎
	r := gin.Default()
	// GET：请求方式；/hello：请求的路径
	// 当客户端以GET方法请求/hello路径时，会执行后面的匿名函数
	r.GET("/hello", func(c *gin.Context) {
		log.WithFields(logrus.Fields{
			"animal": "walrus",
			"size":   10,
		}).Warn("A group of walrus emerges from the ocean")
		// c.JSON：返回JSON格式的数据
		c.JSON(200, gin.H{
			"message": "Hello world!",
		})
	})
	// 启动HTTP服务，默认在0.0.0.0:8080启动服务
	r.Run()
}
```

