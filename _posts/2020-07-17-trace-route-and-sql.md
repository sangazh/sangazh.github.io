---
layout: post
title: 将Gin的路由和SQL接入OpenTracing
tags:
  - golang
  - gin
  - xorm
  - jaeger
  - logrus
excerpt_separator: <!--more-->
---

这周给一个技术项目里接入了OpenTracing。

### 1. 准备工作
用docker部署Jaeger All-in-One。架构是这个样子的。
<img src=https://www.jaegertracing.io/img/architecture-v1.png />

```
$ docker run -d --name jaeger \
  -e COLLECTOR_ZIPKIN_HTTP_PORT=9411 \
  -p 5775:5775/udp \
  -p 6831:6831/udp \
  -p 6832:6832/udp \
  -p 5778:5778 \
  -p 16686:16686 \
  -p 14268:14268 \
  -p 14250:14250 \
  -p 9411:9411 \
  jaegertracing/all-in-one:1.18
```
具体端口代表哪些组件，可以参考[官方文档](https://www.jaegertracing.io/docs/1.18/getting-started/)。

其中`16686`是前端UI的端口

<!--more-->

### 2. Tracing Route - gin

Gin路由加Trace，有个Bose开源的[go-gin-opentracing](https://github.com/Bose/go-gin-opentracing)可以方便的引入中间件。

服务刚启动时，初始化一下 `GlobalTracer`

```golang
    hostName, err := os.Hostname()
	if err != nil {
		hostName = "unknown"
	}
	// initialize the global singleton for tracing...
	tracer, reporter, closer, err := ginopentracing.InitTracing(
	fmt.Sprintf("go-gin-opentracing-example::%s", hostName),
	"localhost:5775", 
	ginopentracing.WithEnableInfoLog(true))
	if err != nil {
		panic("unable to init tracing")
	}
	defer closer.Close()
	defer reporter.Close()
	opentracing.SetGlobalTracer(tracer)
```	

因为这个第三方库自己调了`logrus`写log，而我想要使用自己服务里的logger实体（后面有提到），所以这里`InitTracing`也稍微重写了，改动不大，就不放上来了。

这里`InitTracing`有几个参数：
- 服务名
- jaeger代理的地址。这个地址对应第一步启动的jaeger all-in-one的服务的地址。我们使用的是端口是 `6831`
- info log是否输出。info一般是创建了span的log。平时可禁用。
- sample probability 取样概率。默认为0，表示所有。

接下来直接`Use`该中间件即可。

```golang
// tell gin to use the middleware
	r.Use(ginopentracing.OpenTracer([]byte("api-request-")))
```

我不太满意默认创建的`Span`的`OperationName`，所以重写了中间件。把路径也拼接上了，代码成品是这样的。

```golang
var OperationPrefix = []byte("api-request-")

func OpenTracer() gin.HandlerFunc {
	return func(c *gin.Context) {
		// all before request is handled
		var span opentracing.Span
		operationName := string(OperationPrefix) + c.Request.Method + c.FullPath()
		span = ginopentracing.StartSpanWithHeader(&c.Request.Header, operationName, c.Request.Method, c.Request.URL.Path)
		defer span.Finish() // after all the other defers are completed.. finish the span

		c.Set("trace-context", span) // add the span to the context so it can be used for the duration of the request.
		c.Next()

		span.SetTag(string(ext.HTTPStatusCode), c.Writer.Status())
	}
}
```

### 3. Tracing SQL - xorm
`xorm` 使用的是 `xorm.io/xorm`这个仓库，github的xorm库太老了。

基本思路是`xorm.io/xorm/log`下有个interface `ContextLogger` ，它组合的`SQLLogger`里有两个方法`BeforeSQL`和`AfterSQL`，我们在`BeforeSQL`里创建`Span`，在`AfterSpan`里给`Span`设置一些标签和记录SQL即完成。

```
// SQLLogger represents an interface to log SQL
type SQLLogger interface {
	BeforeSQL(context LogContext) // only invoked when IsShowSQL is true
	AfterSQL(context LogContext)  // only invoked when IsShowSQL is true
}

// ContextLogger represents a logger interface with context
type ContextLogger interface {
	SQLLogger

	Debugf(format string, v ...interface{})
	Errorf(format string, v ...interface{})
	Infof(format string, v ...interface{})
	Warnf(format string, v ...interface{})

	Level() LogLevel
	SetLevel(l LogLevel)

	ShowSQL(show ...bool)
	IsShowSQL() bool
}
```

interface `ContextLogger`包括这些方法。我们要让自用的logrus的实体实现所有方法。

```
import (
	"github.com/opentracing/opentracing-go"
	"github.com/opentracing/opentracing-go/ext"
	tracerLog "github.com/opentracing/opentracing-go/log"
	core "xorm.io/xorm/log"
	"github.com/sirupsen/logrus"
)

type LogWrapper struct {
	*logrus.Logger

	span opentracing.Span
}

func Logger() *LogWrapper {
	return new(LogWrapper) //略去初始logrus的代码
}


func (l *LogWrapper) BeforeSQL(ctx core.LogContext) {
	options := []opentracing.StartSpanOption{
		opentracing.Tag{Key: string(ext.DBInstance), Value: viper.GetString("mysql.dbname")},
		opentracing.Tag{Key: string(ext.DBType), Value: "sql"},
	}
	parent, ok := ctx.Ctx.Value("trace-context").(opentracing.Span)
	if ok && parent != nil {
		options = append(options, opentracing.ChildOf(parent.Context()))
	}

	l.span, _ = opentracing.StartSpanFromContext(ctx.Ctx, "XORM SQL Execute", options...)
}

func (l *LogWrapper) AfterSQL(ctx core.LogContext) {
	defer l.span.Finish()

	// 原本的SimpleLogger里面会获取一次SessionId
	var sessionPart string
	v := ctx.Ctx.Value("__xorm_session_id")
	if key, ok := v.(string); ok {
		sessionPart = fmt.Sprintf(" [%s]", key)
		l.span.LogFields(tracerLog.String("session_id", sessionPart))
	}

	// 将Ctx中全部的信息写入到Span中
	l.span.LogFields(tracerLog.String("SQL", ctx.SQL))
	l.span.LogFields(tracerLog.Object("args", ctx.Args))

	if ctx.ExecuteTime > 0 {
		l.span.SetTag("execute_time", ctx.ExecuteTime)
		l.Infof("[SQL]%s %s %v - %v", sessionPart, ctx.SQL, ctx.Args, ctx.ExecuteTime)
	} else {
		l.Infof("[SQL]%s %s %v", sessionPart, ctx.SQL, ctx.Args)
	}
}
```

可以看到`BeforeContext`传入了一个context。接下来要做的是将`gin.Context`传下去。

有两个set context的入口：

1

```golang
	engine, err := xorm.NewEngine("mysql", conn)
	if err != nil {
		panic(err.Error())
	}
	engine.SetLogger(Logger())
	engine.SetDefaultContext(c) // c *gin.Context
	engine.Select("blabla")...

```
注意这里的`SetLogger`是上面定义过实现了 `ContextLogger`的logger实体。
而`SetDefaultContext`就可以传入`gin.context`了

2

上面的`engine.Select()`返回的其实是`Session`，一般来说query, transaction的前都会先`NewSession()`，再做接下来的工作的。

那么这个`Session`也有个方法 `Context()`可以用来传入context。所以

```
	engine, err := xorm.NewEngine("mysql", conn)
	if err != nil {
		panic(err.Error())
	}
	engine.SetLogger(Logger())
	
	session := engine.NewSession()
	session.Context(c) // c *gin.Context
	session.Select("blabla")...
```

这样就生成了以路由为父Span，与这条请求相关的SQL为子Span的多Span组合而成的一条简单的链路。

### References & Useful Links
- [Jaeger - Getting Started](https://www.jaegertracing.io/docs/1.18/getting-started/)
- [Jaeger - Github](https://github.com/jaegertracing/jaeger)
- [OpenTracing标准](https://opentracing.io/specification/)
- [Jaeger Atchitecture](https://www.jaegertracing.io/docs/1.18/architecture/)
- [jaeger-client-go](https://github.com/jaegertracing/jaeger-client-go)
- [go-gin-opentracing](https://github.com/Bose/go-gin-opentracing)
- [Golang XORM搭配OpenTracing+Jaeger链路监控让SQL执行一览无遗](https://gitee.com/avtion/xormWithTracing)
- [原生mysql如何接入Tracing](https://ceshihao.github.io/2018/11/29/tracing-db-operations/)