---
title: Golang微服务最佳实践
date: 2019-07-24T15:16:41+08:00
lastmod: 2019-08-28T15:16:41+08:00
draft: false
categories: ["golang"]
tags: ["golang", "dependency-inject","unit-test","microservice","grpc"]
description: Golang项目示例
---

本文通过一个完整的项目的示例，从Golang项目的结构、分层思想、依赖注入、错误处理、单元测试、服务治理、框架选择等方面介绍Go语言项目的开发经验.
<!--more-->

## 目的

1. 提供一个完整的go语言项目示例
2. 介绍Go开发应该遵守的规范
3. 介绍编程思想在go语言中的应用
4. 服务治理相关的框架、中间件介绍
5. 自动化生成监控和报警

## 示例项目

Github源码[go-project-sample](http://github.com/sdgmf/go-project-sample)

项目分为products、details、ratings、reviews四个微服务,依赖关系如下.

![dependency](/images/goproject_dep.jpg)

## 快速开始

```bash
    git clone https://github.com/sdgmf/go-project-sample.git
    cd go-project-sample
    make docker-compose
```

* **访问接口**： http://localhost:8080/product/1
* **consul**: http://localhost:8500/
* **grafana**: http://localhost:3000/ 
* **jaeger**: http://localhost:16686/search
* **Prometheus**: http://localhost:9090/graph 

## 截图

Grafana Dashboard,可以自动生成!

![dashboard](/images/grafana_dashboard.jpg)

![dashboard1](/images/grafana_dashboard1.jpg)

调用链跟踪

![jaeger](/images/jaeger.jpg)

![jaeger](/images/jaeger1.jpg)

## 包结构

关于golang项目的包结构，Dave Chaney博客《[Five suggestions for setting up a Go project](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project)》里讨论了package和command的包设计建议，还有一个社区普遍认可的包结构规范[project-layout](https://github.com/golang-standards/project-layout)。在这两个两篇文章的知道下，结合常见的互联网微服务项目，我又细化了如下的项目结构。

```bash
.
├── api
│   └── proto
├── build
│   ├── details
│   ├── products
│   ├── ratings
│   └── reviews
├── cmd
│   ├── details
│   ├── products
│   ├── ratings
│   └── reviews
├── configs
├── deployments
├── dist
├── internal
│   ├── app
│   │   ├── details
│   │   │   ├── controllers
│   │   │   ├── grpcservers
│   │   │   ├── repositorys
│   │   │   └── services
│   │   ├── products
│   │   │   ├── controllers
│   │   │   ├── grpcclients
│   │   │   └── services
│   │   ├── ratings
│   │   │   ├── controllers
│   │   │   ├── grpcservers
│   │   │   ├── repositorys
│   │   │   └── services
│   │   └── reviews
│   │       ├── controllers
│   │       ├── grpcservers
│   │       ├── repositorys
│   │       └── services
│   └── pkg
│       ├── app
│       ├── config
│       ├── consul
│       ├── database
│       ├── jaeger
│       ├── log
│       ├── models
│       ├── transports
│       │   ├── grpc
│       │   └── http
│       │       └── middlewares
│       │           └── ginprom
│       └── utils
│           └── netutil
├── mocks
└── scripts

```

### /cmd

同[project-layout](https://github.com/golang-standards/project-layout)

> "该项目的main方法。 每个应用程序的目录名称应与您要拥有的可执行文件的名称相匹配（例如，/cmd/myapp）。 不要在应用程序目录中放入大量代码。 如果您认为代码可以导入并在其他项目中使用，那么它应该存在于/ pkg目录中。 如果代码不可重用或者您不希望其他人重用它，请将该代码放在/ internal目录中。 你会惊讶于别人会做什么，所以要明确你的意图！ 通常有一个小的main函数可以从/ internal和/ pkg目录中导入和调用代码，而不是其他任何东西。"

### /internal/pkg

同[project-layout](https://github.com/golang-standards/project-layout)

> "私有应用程序和库代码。 这是您不希望其他人在其应用程序或库中导入的代码。 将您的实际应用程序代码放在/internal/app目录（例如/internal/app/myapp）和/internal/ pkg目录中这些应用程序共享的代码（例如/internal/pkg/myprivlib）。"

内部的包采用平铺的方式。

### /internal/pkg/config

加载配置文件,或者从配置中心获取配置和监听配置变动。

### /internal/pkg/database

数据库连接初始化和ORM框架初始化配置。

### /internal/pkg/models

结构体定义。

### /internal/pkg/transport

http/gpc 传输层

### /internal/app/products

应用内部代码

### /internal/app/products/controllers

MVC控制层

### /internal/app/products/services

领域逻辑层

### /internal/app/products/repositorys

存储层

### /internal/app/products/grpcclients

grpc client

### /internal/app/details/grpcservers

grpc servers

### /mocks

mockery 生成的mock实现

### /api

OpenAPI/Swagger规范，JSON模式文件，协议定义文件等。

### /grafana

生成grafana dashboard 用到的脚本

### /scripts

sql、部署脚本等

### /build

Dockerfile、docker-compose

### /deployment

docker-compose/kubernetes等配置

## 分层

MVC、领域模型、ORM 这些都是通过把特定职责的代码拆分到不同的层次对象里，在Java里这些分层概念在各种框架里都有体现(如SSH,SSM等常用框架组合)，并且早已形成了默认的规约，是否还适用go语言吗？答案是肯定的。Martin Fowler在《[企业应用架构模式](https://book.douban.com/subject/1230559/)》就阐述过分层带来的各种好处。

1. 便代码复用,提高代码可维护性.如service的代码可被http协议和grpc协议复用，如果增加thrift协议的接口也很方便。
2. 层次清晰，代码可读性更高。
3. 方便单元测试,单元测试往往因为依赖持久的存储而无法进行,如果持久化代码抽取到单独的对象里，这就变的很简单了.

## 依赖注入

Java 程序员都很熟悉依赖注入和控制翻转这种思想，Spring正式基于依赖注入的思想开发。依赖注入的好处是解耦，对象的组装交给容器来控制(选择需要的实现类、是否单例和初始化).基于依赖注入可以很方便的实现单元测试和提高代码可维护性。

关于Golang依赖注入的讨论《[Dependency Injection in Go](https://blog.drewolson.org/dependency-injection-in-go)》,Golang依赖注入的Package有 Uber的[dig](https://github.com/uber-go/dig),[fx](https://github.com/uber-go/fx),facebook 的 [inject](https://github.com/facebookarchive/inject),google的[wire](https://github.com/google/wire)。dig、fx和inject都是基于反射实现，wire是通过代码生成实现，代码生成的方式是显式的。

本示例通过wire来完成依赖注入. 编写wire.go,wire会根据wire.go生成代码。

``` go
// +build wireinject

package main

import (
    "github.com/google/wire"
    "github.com/zlgwzy/go-project-sample/cmd/app"
    "github.com/zlgwzy/go-project-sample/internal/pkg/config"
    "github.com/zlgwzy/go-project-sample/internal/pkg/database"
    "github.com/zlgwzy/go-project-sample/internal/pkg/log"
    "github.com/zlgwzy/go-project-sample/internal/pkg/services"
    "github.com/zlgwzy/go-project-sample/internal/pkg/repositorys"
    "github.com/zlgwzy/go-project-sample/internal/pkg/transport/http"
    "github.com/zlgwzy/go-project-sample/internal/pkg/transport/grpc"
)

var providerSet = wire.NewSet(
    log.ProviderSet,
    config.ProviderSet,
    database.ProviderSet,
    services.ProviderSet,
    repositorys.ProviderSet,
    http.ProviderSet,
    grpc.ProviderSet,
    app.ProviderSet,
)

func CreateApp(cf string) (*app.App, error) {
    panic(wire.Build(providerSet))
}

```

生成代码

```bash
go get github.com/google/wire/cmd/wire

wire ./...
```

生成后的代码在wire_gen.go

``` go
// Code generated by Wire. DO NOT EDIT.

//go:generate wire
//+build !wireinject

package main

import (
    "github.com/google/wire"
    "github.com/zlgwzy/go-project-sample/cmd/app"
    "github.com/zlgwzy/go-project-sample/internal/pkg/config"
    "github.com/zlgwzy/go-project-sample/internal/pkg/database"
    "github.com/zlgwzy/go-project-sample/internal/pkg/log"
    "github.com/zlgwzy/go-project-sample/internal/pkg/services"
    "github.com/zlgwzy/go-project-sample/internal/pkg/repositorys"
    "github.com/zlgwzy/go-project-sample/internal/pkg/transport/grpc"
    "github.com/zlgwzy/go-project-sample/internal/app/proxy/grpcservers"
    "github.com/zlgwzy/go-project-sample/internal/pkg/transport/http"
    "github.com/zlgwzy/go-project-sample/internal/app/proxy/controllers"
)

// Injectors from wire.go:

func CreateApp(cf string) (*app.App, error) {
    viper, err := config.New(cf)
    if err != nil {
        return nil, err
    }
    options, err := log.NewOptions(viper)
    if err != nil {
        return nil, err
    }
    logger, err := log.New(options)
    if err != nil {
        return nil, err
    }
    httpOptions, err := http.NewOptions(viper)
    if err != nil {
        return nil, err
    }
    databaseOptions, err := database.NewOptions(viper, logger)
    if err != nil {
        return nil, err
    }
    db, err := database.New(databaseOptions)
    if err != nil {
        return nil, err
    }
    productsRepository := repositorys.NewMysqlProductsRepository(logger, db)
    productsService := services.NewProductService(logger, productsRepository)
    productsController := controllers.NewProductsController(logger, productsService)
    initControllers := controllers.CreateInitControllersFn(productsController)
    engine := http.NewRouter(httpOptions, initControllers)
    server, err := http.New(httpOptions, logger, engine)
    if err != nil {
        return nil, err
    }
    grpcOptions, err := grpc.NewOptions(viper)
    if err != nil {
        return nil, err
    }
    productsServer, err := grpcservers.NewProductsServer(logger, productsService)
    if err != nil {
        return nil, err
    }
    initServers := grpcservers.CreateInitServersFn(productsServer)
    grpcServer, err := grpc.New(grpcOptions, logger, initServers)
    if err != nil {
        return nil, err
    }
    appApp, err := app.New(logger, server, grpcServer)
    if err != nil {
        return nil, err
    }
    return appApp, nil
}

// wire.go:

var providerSet = wire.NewSet(log.ProviderSet, config.ProviderSet, database.ProviderSet, services.ProviderSet, repositorys.ProviderSet, http.ProviderSet, grpc.ProviderSet, app.ProviderSet)

```

## 面向接口编程

多态和单元测试必须,比较好理解不再解释。

## 显式编程

Golang的开发推崇这样一种显式编程的思想，显式的初始化、方法调用和错误处理.

1. 尽可能不要使用包级别的全局变量.
2. 尽量不要使用init函数，初始化操作可以在main函数中调用，这样方便阅读代码和控制初始化顺序。
3. 函数都要返回错误，用if err != nil 显式的处理错误.
4. 依赖的参数让调用者去控制（控制翻转的思想),可以看下节依赖注入。

几个大佬都讨论过这个问题，博士Peter的《[A theory of modern Go](http://peter.bourgon.org/blog/2017/06/09/theory-of-modern-go.html)》认为魔法代码的核心是”no package level vars; no func init“.单这也不是绝对。
Dave Cheny在《[go-without-package-scoped-variables](https://dave.cheney.net/2017/06/11/go-without-package-scoped-variables)》做了更详细的说明.

## 打印日志

使用比较多的两个日志库，[logrush](https://github.com/sirupsen/logrus)和[zap](https://github.com/uber-go/zap),个人更喜欢zap。

初始化logger,通过viper加载日志相关配置,lumberjack负责日志切割。

``` go

// Options is log configration struct
type Options struct {
     Filename   string
     MaxSize    int
     MaxBackups int
     MaxAge     int
     Level      string
     Stdout     bool
}

func NewOptions(v *viper.Viper) (*Options, error) {
     var (
          err error
          o   = new(Options)
     )
     if err = v.UnmarshalKey("log", o); err != nil {
          return nil, err
     }

     return o, err
}

// New for init zap log library
func New(o *Options) (*zap.Logger, error) {
     var (
          err    error
          level  = zap.NewAtomicLevel()
          logger *zap.Logger
     )

     err = level.UnmarshalText([]byte(o.Level))
     if err != nil {
          return nil, err
     }

     fw := zapcore.AddSync(&lumberjack.Logger{
          Filename:   o.Filename,
          MaxSize:    o.MaxSize, // megabytes
          MaxBackups: o.MaxBackups,
          MaxAge:     o.MaxAge, // days
     })

     cw := zapcore.Lock(os.Stdout)

     // file core 采用jsonEncoder
     cores := make([]zapcore.Core, 0, 2)
     je := zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())
     cores = append(cores, zapcore.NewCore(je, fw, level))

     // stdout core 采用 ConsoleEncoder
     if o.Stdout {
          ce := zapcore.NewConsoleEncoder(zap.NewDevelopmentEncoderConfig())
          cores = append(cores, zapcore.NewCore(ce, cw, level))
     }

     core := zapcore.NewTee(cores...)
     logger = zap.New(core)

     zap.ReplaceGlobals(logger)

     return logger, err
}


```

logger应该作为私有变量，这样可以统一添加对象的标示。

``` go

type Object struct {
    logger *zap.Logger
}

// 统一添加标示
func NewObject(logger *zap.Logger){
    return &Object{
        logger:  logger.With(zap.String("type","Object"))
    }

}
```

## 错误处理

错误处理还是看Dave Cheny的博客《[Stack traces and the errors package](https://dave.cheney.net/2016/06/12/stack-traces-and-the-errors-package)》,《[Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)》。

1. 使用类型判断错误。
2. 包装错误，记录错误的上下文。
3. 使用 pakcage [errors](github.com/pkg/errors)
4. 只处理一次错误，处理错误意味着检查错误值并做出决定。

### 错误日志

```go
logger.Error("get product by id error", zap.Error(err))
```

``` json

{
    "level":"error",
    "ts":1564056905.4602501,
    "msg":"get product by id error",
    "error":"product service get product error: get product error[id=2]: record not found",
    "errorVerbose":"record not found get product error[id=2]
github.com/zlgwzy/go-project-sample/internal/pkg/repositorys.(*MysqlProductsRepository).Get
/Users/xxx/code/go/go-project-sample/internal/pkg/repositorys/products.go:29
github.com/zlgwzy/go-project-sample/internal/pkg/services.(*DefaultProductsService).Get
/Users/xxx/code/go/go-project-sample/internal/pkg/services/products.go:27
github.com/zlgwzy/go-project-sample/internal/app/proxy/controllers.(*ProductsController).Get
/Users/xxx/code/go/go-project-sample/internal/app/proxy/controllers/products.go:30
github.com/gin-gonic/gin.(*Context).Next
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-gonic/gin.RecoveryWithWriter.func1
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/recovery.go:83
github.com/gin-gonic/gin.(*Context).Next
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-gonic/gin.(*Engine).handleHTTPRequest
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/gin.go:389
github.com/gin-gonic/gin.(*Engine).ServeHTTP
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/gin.go:351
net/http.serverHandler.ServeHTTP
/usr/local/Cellar/go/1.12.6/libexec/src/net/http/server.go:2774
net/http.(*conn).serve
/usr/local/Cellar/go/1.12.6/libexec/src/net/http/server.go:1878
runtime.goexit
/usr/local/Cellar/go/1.12.6/libexec/src/runtime/asm_amd64.s:1337
product service get product error
github.com/zlgwzy/go-project-sample/internal/pkg/services.(*DefaultProductsService).Get
/Users/xxx/code/go/go-project-sample/internal/pkg/services/products.go:28
github.com/zlgwzy/go-project-sample/internal/app/proxy/controllers.(*ProductsController).Get
/Users/xxx/code/go/go-project-sample/internal/app/proxy/controllers/products.go:30
github.com/gin-gonic/gin.(*Context).Next
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-gonic/gin.RecoveryWithWriter.func1
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/recovery.go:83
github.com/gin-gonic/gin.(*Context).Next
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-gonic/gin.(*Engine).handleHTTPRequest
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/gin.go:389
github.com/gin-gonic/gin.(*Engine).ServeHTTP
/Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/gin.go:351
net/http.serverHandler.ServeHTTP
/usr/local/Cellar/go/1.12.6/libexec/src/net/http/server.go:2774
net/http.(*conn).serve
/usr/local/Cellar/go/1.12.6/libexec/src/net/http/server.go:1878
runtime.goexit
/usr/local/Cellar/go/1.12.6/libexec/src/runtime/asm_amd64.s:1337"
}

```

### 接口中返回错误

gin 的使用方式:

```go
func Handler(c *gin.Context) {
    err := //
    c.String(http.StatusInternalServerError, "%+v", err)
}


```

curl http://localhost:8080/product/5 输出:

```txt
rpc error: code = Unknown desc = details grpc service get detail error: detail service get detail error: get product error[id=5]: record not found
get rating error
github.com/sdgmf/go-project-sample/internal/app/products/services.(*DefaultProductsService).Get
     /Users/xxx/code/go/go-project-sample/internal/app/products/services/products.go:50
github.com/sdgmf/go-project-sample/internal/app/products/controllers.(*ProductsController).Get
     /Users/xxx/code/go/go-project-sample/internal/app/products/controllers/products.go:30
github.com/gin-gonic/gin.(*Context).Next
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/opentracing-contrib/go-gin/ginhttp.Middleware.func4
     /Users/xxx/go/pkg/mod/github.com/opentracing-contrib/go-gin@v0.0.0-20190301172248-2e18f8b9c7d4/ginhttp/server.go:99
github.com/gin-gonic/gin.(*Context).Next
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/sdgmf/go-project-sample/internal/pkg/transports/http/middlewares/ginprom.(*GinPrometheus).Middleware.func1
     /Users/xxx/code/go/go-project-sample/internal/pkg/transports/http/middlewares/ginprom/ginprom.go:105
github.com/gin-gonic/gin.(*Context).Next
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-contrib/zap.RecoveryWithZap.func1
     /Users/xxx/go/pkg/mod/github.com/gin-contrib/zap@v0.0.0-20190528085758-3cc18cd8fce3/zap.go:109
github.com/gin-gonic/gin.(*Context).Next
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-contrib/zap.Ginzap.func1
     /Users/xxx/go/pkg/mod/github.com/gin-contrib/zap@v0.0.0-20190528085758-3cc18cd8fce3/zap.go:32
github.com/gin-gonic/gin.(*Context).Next
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-gonic/gin.RecoveryWithWriter.func1
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/recovery.go:83
github.com/gin-gonic/gin.(*Context).Next
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/context.go:124
github.com/gin-gonic/gin.(*Engine).handleHTTPRequest
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/gin.go:389
github.com/gin-gonic/gin.(*Engine).ServeHTTP
     /Users/xxx/go/pkg/mod/github.com/gin-gonic/gin@v1.4.0/gin.go:351
net/http.serverHandler.ServeHTTP
     /usr/local/Cellar/go/1.12.6/libexec/src/net/http/server.go:2774
net/http.(*conn).serve
     /usr/local/Cellar/go/1.12.6/libexec/src/net/http/server.go:1878
runtime.goexit
     /usr/local/Cellar/go/1.12.6/libexec/src/runtime/asm_amd64.s:1337
```

## 添加监控

### Prometheus

作为新一代的监控框架，Prometheus 具有以下特点：

* 强大的多维度数据模型：
* 时间序列数据通过 metric 名和键值对来区分。
* 所有的 metrics 都可以设置任意的多维标签。
* 数据模型更随意，不需要刻意设置为以点分隔的字符串。
* 可以对数据模型进行聚合，切割和切片操作。
* 支持双精度浮点类型，标签可以设为全 unicode。
* 灵活而强大的查询语句（PromQL）：在同一个查询语句，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作。
* 易于管理： Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储。
* 高效：平均每个采样点仅占 3.5 bytes，且一个 Prometheus server 可以处理数百万的 metrics。
* 使用 pull 模式采集时间序列数据，这样不仅有利于本机测试而且可以避免有问题的服务器推送坏的 metrics。
* 可以采用 push gateway 的方式把时间序列数据推送至 Prometheus server 端。
* 可以通过服务发现或者静态配置去获取监控的 targets。
* 有多种可视化图形界面。
* 易于伸缩

### Go基础监控

```go
import (
    "github.com/opentracing-contrib/go-gin/ginhttp"
    "github.com/gin-gonic/gin"
)
r := gin.New()

r.GET("/metrics", gin.WrapH(promhttp.Handler()))

```

### http监控

创建internal/pkg/transports/http/middlewares/ginprom/ginprom.go

```go
package ginprom

import (
     "strconv"
     "sync"
     "time"

     "github.com/gin-gonic/gin"
     "github.com/prometheus/client_golang/prometheus"
)

const (
     metricsPath = "/metrics"
     faviconPath = "/favicon.ico"
)

var (
     httpHistogram = prometheus.NewHistogramVec(prometheus.HistogramOpts{
          Namespace: "http_server",
          Name:      "requests_seconds",
          Help:      "Histogram of response latency (seconds) of http handlers.",
     }, []string{"method", "code", "uri"})
)

func init() {
     prometheus.MustRegister(httpHistogram)
}

type handlerPath struct {
     sync.Map
}

func (hp *handlerPath) get(handler string) string {
     v, ok := hp.Load(handler)
     if !ok {
          return ""
     }
     return v.(string)
}

func (hp *handlerPath) set(ri gin.RouteInfo) {
     hp.Store(ri.Handler, ri.Path)
}

// GinPrometheus  struct
type GinPrometheus struct {
     engine   *gin.Engine
     ignored map[string]bool
     pathMap *handlerPath
     updated bool
}

// Option 可配置参数
type Option func(*GinPrometheus)

// Ignore 添加忽略的路径
func Ignore(path ...string) Option {
     return func(gp *GinPrometheus) {
          for _, p := range path {
               gp.ignored[p] = true
          }
     }
}

// New 构造器
func New(e *gin.Engine, options ...Option) *GinPrometheus {
     // 参数验证
     if e == nil {
          return nil
     }
     gp := &GinPrometheus{
          engine: e,
          ignored: map[string]bool{
               metricsPath: true,
               faviconPath: true,
          },
          pathMap: &handlerPath{},
     }
     for _, o := range options {
          o(gp)
     }
     return gp
}

func (gp *GinPrometheus) updatePath() {
     gp.updated = true
     for _, ri := range gp.engine.Routes() {
          gp.pathMap.set(ri)
     }
}

// Middleware 返回中间件
func (gp *GinPrometheus) Middleware() gin.HandlerFunc {
     return func(c *gin.Context) {
          if !gp.updated {
               gp.updatePath()
          }
          // 把不需要的过滤掉
          if gp.ignored[c.Request.URL.String()] == true {
               c.Next()
               return
          }
          start := time.Now()

          c.Next()

          httpHistogram.WithLabelValues(
               c.Request.Method,
               strconv.Itoa(c.Writer.Status()),
               gp.pathMap.get(c.HandlerName()),
          ).Observe(time.Since(start).Seconds())
     }
}

```

在internal/pkg/transports/http/http.go添加:

```go
r.Use(ginprom.New(r).Middleware()) // 添加prometheus 监控
```

### grpc监控

在server添加：

```go
          gs = grpc.NewServer(
               grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
                    grpc_prometheus.StreamServerInterceptor,
               )),
               grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
                    grpc_prometheus.UnaryServerInterceptor,
               )),
          )
```

在client添加：

```go
     grpc_prometheus.EnableClientHandlingTimeHistogram()
     o.GrpcDialOptions = append(o.GrpcDialOptions,
          grpc.WithInsecure(),
          grpc.WithUnaryInterceptor(grpc_middleware.ChainUnaryClient(
               grpc_prometheus.UnaryClientInterceptor,
          ),
          grpc.WithStreamInterceptor(grpc_middleware.ChainStreamClient(
               grpc_prometheus.StreamClientInterceptor,
          ),
     )
```

### 添加dashbord

可以通过自动生成dashboard，可以集成到自己公司的CICD系统中，上线后dashboard就有了,下面介绍如何通过jsonnet生成dashboard

* 安装[jsonnet](https://jsonnet.org/)
* 下载[graonnet-lib](https://github.com/grafana/grafonnet-lib.git)

创建 grafana/dashboard.jsonnet

```jsonnet
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local row = grafana.row;
local singlestat = grafana.singlestat;
local prometheus = grafana.prometheus;
local graphPanel = grafana.graphPanel;
local template = grafana.template;
local row = grafana.row;

local app = std.extVar('app');

local baseUp() = singlestat.new(
  'Number of instances',
  datasource='Prometheus',
  span=2,
  valueName='current',
  transparent=true,
).addTarget(
  prometheus.target(
    'sum(up{app="' + app + '"})', instant=true
  )
);

local baseGrpcQPS() = singlestat.new(
  'Number of grpc request  per seconds',
  datasource='Prometheus',
  span=2,
  valueName='current',
  transparent=true,
).addTarget(
  prometheus.target(
    'sum(rate(grpc_server_handled_total{app="' + app + '",grpc_type="unary"}[1m]))',
     instant=true
  )
);

local baseGrpcError() = singlestat.new(
  'Percentage of grpc error request',
  format='percent',
  datasource='Prometheus',
  span=2,
  valueName='current',
  transparent=true,
).addTarget(
  prometheus.target(
    'sum(rate(grpc_server_handled_total{app="' + app + '",grpc_type="unary",grpc_code!="OK"}[1m])) /sum(rate(grpc_server_started_total{app="' + app + '",grpc_type="unary"}[1m])) * 100.0',
    instant=true
  )
);

local baseHttpQPS() = singlestat.new(
  'Number of http request  per seconds',
  datasource='Prometheus',
  span=2,
  valueName='current',
  transparent=true,
).addTarget(
  prometheus.target(
    'sum(rate(http_server_requests_seconds_count{app="' + app + '"}[1m]))',
     instant=true
  )
);

local baseHttpError() = singlestat.new(
  'Percentage of http error request',
  datasource='Prometheus',
  format='percent',
  span=2,
  valueName='current',
  transparent=true,
).addTarget(
  prometheus.target(
    'sum(rate(http_server_requests_seconds_count{app="' + app + '",code!="200"}[1m])) /sum(rate(http_server_requests_seconds_count{app="' + app + '"}[1m])) * 100.0',
    instant=true
  )
);

local goState(metric, description=null, format='none') = graphPanel.new(
  metric,
  span=6,
  fill=0,
  min=0,
  legend_values=true,
  legend_min=false,
  legend_max=true,
  legend_current=true,
  legend_total=false,
  legend_avg=false,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  description=description,
).addTarget(
  prometheus.target(
    metric + '{app="' + app + '"}',
    datasource='Prometheus',
    legendFormat='{{instance}}'
  )
);




local grpcQPS(kind='server', groups=['grpc_code']) = graphPanel.new(
  //title='grpc_' + kind + '_qps_' + std.join(',', groups),
  title='Number of grpc ' + kind + ' request  per seconds group by (' + std.join(',', groups) + ')',
  description='Number of grpc ' + kind + ' request per seconds  group by (' + std.join(',', groups) + ')',
  legend_values=true,
  legend_max=true,
  legend_current=true,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  span=6,
  fill=0,
  min=0,
).addTarget(
  prometheus.target(
    'sum(rate(grpc_' + kind + '_handled_total{app="' + app + '",grpc_type="unary"}[1m])) by (' + std.join(',', groups) + ')',
    datasource='Prometheus',
    legendFormat='{{' + std.join('}}.{{', groups) + '}}'
  )
);


local grpcErrorPercentage(kind='server', groups=['instance']) = graphPanel.new(
  //title='grpc_' + kind + '_error_percentage_' + std.join(',', groups),
  title='Percentage of grpc ' + kind + ' error request group by (' + std.join(',', groups) + ')',
  description='Percentage of grpc ' + kind + ' error request group by (' + std.join(',', groups) + ')',
  format='percent',
  legend_values=true,
  legend_max=true,
  legend_current=true,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  span=6,
  fill=0,
  min=0,
).addTarget(
  prometheus.target(
    'sum(rate(grpc_'+kind+'_handled_total{app="' + app + '",grpc_type="unary",grpc_code!="OK"}[1m])) by (' + std.join(',', groups) + ')/sum(rate(grpc_'+kind+'_started_total{app="' + app + '",grpc_type="unary"}[1m])) by (' + std.join(',', groups) + ')* 100.0',
    datasource='Prometheus',
    legendFormat='{{' + std.join('}}.{{', groups) + '}}'
  )
);


local grpcLatency(kind='server', groups=['instance'], quantile='0.99') = graphPanel.new(
  title='Latency of grpc ' + kind + ' request group by (' + std.join(',', groups) + ')',
  description='Latency of grpc ' + kind + ' request group by (' + std.join(',', groups) + ')',
  format='ms',
  legend_values=true,
  legend_max=true,
  legend_current=true,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  span=6,
  fill=0,
  min=0,
).addTarget(
  prometheus.target(
    '1000 * histogram_quantile(' + quantile + ',sum(rate(grpc_' + kind + '_handling_seconds_bucket{app="' + app + '",grpc_type="unary"}[1m])) by (' + std.join(',', groups) + ',le))',
    datasource='Prometheus',
    legendFormat='{{' + std.join('}}.{{', groups) + '}}'
  )
);




local httpQPS(kind='server', groups=['grpc_code']) = graphPanel.new(
  title='Number of http' + kind + ' request group by (' + std.join(',', groups) + ') per seconds',
  description='Number of http' + kind + ' request group by (' + std.join(',', groups) + ') per seconds',
  legend_values=true,
  legend_max=true,
  legend_current=true,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  span=6,
  fill=0,
  min=0,
).addTarget(
  prometheus.target(
    'sum(rate(http_server_requests_seconds_count{app="' + app + '"}[1m])) by (' + std.join(',', groups) + ')',
    datasource='Prometheus',
    legendFormat='{{' + std.join('}}.{{', groups) + '}}'
  )
);


local httpErrorPercentage(groups=['instance']) = graphPanel.new(
  //title='grpc_' + kind + '_error_percentage_' + std.join(',', groups),
  title='Percentage of http error request group by (' + std.join(',', groups) + ') ',
  description='Percentage of http error request group by (' + std.join(',', groups) + ')',
  format='percent',
  legend_values=true,
  legend_max=true,
  legend_current=true,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  span=6,
  fill=0,
  min=0,
).addTarget(
  prometheus.target(
    'sum(rate(http_server_requests_seconds_count{app="' + app + '",status!="200"}[1m])) by (' + std.join(',', groups) + ')/sum(rate(http_server_requests_seconds_count{app="' + app + '"}[1m])) by (' + std.join(',', groups) + ')* 100.0',
    datasource='Prometheus',
    legendFormat='{{' + std.join('}}.{{', groups) + '}}'
  )
);

local httpLatency(groups=['instance'], quantile='0.99') = graphPanel.new(
  title='Latency of http request group by (' + std.join(',', groups) + ')',
  description='Latency of http request group by (' + std.join(',', groups) + ')',
  format='ms',
  legend_values=true,
  legend_max=true,
  legend_current=true,
  legend_alignAsTable=true,
  legend_rightSide=true,
  transparent=true,
  span=6,
  fill=0,
  min=0,
).addTarget(
  prometheus.target(
    '1000 * histogram_quantile(' + quantile + ',sum(rate(http_server_requests_seconds_bucket{app="' + app + '"}[1m])) by (' + std.join(',', groups) + ',le))',
    datasource='Prometheus',
    legendFormat='{{' + std.join('}}.{{', groups) + '}}'
  )
);


dashboard.new(app, schemaVersion=16, tags=['go'], editable=true, uid=app)
.addPanel(row.new(title='Base', collapse=true)
          .addPanel(baseUp(), gridPos={ x: 0, y: 0, w: 4, h: 10 })
          .addPanel(baseGrpcQPS(), gridPos={x: 4, y: 0, w: 4, h: 10 })
          .addPanel(baseGrpcError(), gridPos={x: 8, y: 0, w: 4, h: 10 })
          .addPanel(baseHttpQPS(), gridPos={x: 12, y: 0, w: 4, h: 10 })
          .addPanel(baseHttpError(), gridPos={x: 16, y: 0, w: 4, h: 10 })
          ,{  })
.addPanel(row.new(title='Go', collapse=true)
          .addPanel(goState('go_goroutines', 'Number of goroutines that currently exist'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_alloc_bytes', 'Number of bytes allocated and still in use'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_alloc_bytes_total', 'Total number of bytes allocated, even if freed'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_buck_hash_sys_bytes', 'Number of bytes used by the profiling bucket hash table'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_frees_total', 'Total number of frees'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_gc_cpu_fraction', "The fraction of this program's available CPU time used by the GC since the program started."), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_gc_sys_bytes', 'Number of bytes used for garbage collection system metadata'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_heap_alloc_bytes', 'Number of heap bytes allocated and still in use'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_heap_idle_bytes', 'Number of heap bytes waiting to be used'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_heap_inuse_bytes', 'Number of heap bytes that are in use'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_heap_objects', 'Number of allocated objects'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_heap_released_bytes', 'Number of heap bytes released to OS'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_heap_sys_bytes', 'Number of heap bytes obtained from system'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_last_gc_time_seconds', 'Number of seconds since 1970 of last garbage collection'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_lookups_total', 'Total number of pointer lookups'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_mallocs_total', 'Total number of mallocs'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_mcache_inuse_bytes', 'Number of bytes in use by mcache structures'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_mcache_sys_bytes', 'Number of bytes used for mcache structures obtained from system'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_mspan_inuse_bytes', 'Number of bytes in use by mspan structures'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_mspan_sys_bytes', 'Number of bytes used for mspan structures obtained from system'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_next_gc_bytes', 'Number of heap bytes when next garbage collection will take place'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_other_sys_bytes', 'Number of bytes used for other system allocations'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_stack_inuse_bytes', 'Number of bytes in use by the stack allocator'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_stack_sys_bytes', 'Number of bytes obtained from system for stack allocator'), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(goState('go_memstats_sys_bytes', 'Number of bytes obtained from system'), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})

.addPanel(row.new(title='Grpc Server request rate', collapse=true)
          .addPanel(grpcQPS('server', ['grpc_code']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcQPS('server', ['instance']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcQPS('server', ['grpc_service']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcQPS('server', ['grpc_service', 'grpc_method']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Grpc Server request error percentage', collapse=true)
          .addPanel(grpcErrorPercentage('server', ['grpc_service']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcErrorPercentage('server', ['grpc_service', 'grpc_method']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcErrorPercentage('server', ['instance']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Grpc server 99%-tile Latency of requests', collapse=true)
          .addPanel(grpcLatency('server', ['grpc_code'], 0.99), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('server', ['instance'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('server', ['grpc_service'], 0.99), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('server', ['grpc_service', 'grpc_method'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Grpc server 90%-tile Latency of requests', collapse=true)
        .addPanel(grpcLatency('server', ['grpc_code'], 0.90), gridPos={ x: 0, y: 0, w: 12, h: 10 })
        .addPanel(grpcLatency('server', ['instance'], 0.90), gridPos={ x: 12, y: 0, w: 12, h: 10 })
        .addPanel(grpcLatency('server', ['grpc_service'], 0.90), gridPos={ x: 0, y: 0, w: 12, h: 10 })
        .addPanel(grpcLatency('server', ['grpc_service', 'grpc_method'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
        , {})
.addPanel(row.new(title='Grpc client request rate', collapse=true)
          .addPanel(grpcQPS('client', ['grpc_code']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcQPS('client', ['instance']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcQPS('client', ['grpc_service']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcQPS('client', ['grpc_service', 'grpc_method']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Grpc client request error percentage', collapse=true)
          .addPanel(grpcErrorPercentage('client', ['grpc_service']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcErrorPercentage('client', ['grpc_service', 'grpc_method']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcErrorPercentage('client', ['instance']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Grpc client 99%-tile Latency of requests', collapse=true)
          .addPanel(grpcLatency('client', ['grpc_code'], 0.99), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('client', ['instance'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('client', ['grpc_service'], 0.99), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('client', ['grpc_service', 'grpc_method'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Grpc client 90%-tile Latency of requests', collapse=true)
          .addPanel(grpcLatency('client', ['grpc_code'], 0.90), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('client', ['instance'], 0.90), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('client', ['grpc_service'], 0.90), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(grpcLatency('client', ['grpc_service', 'grpc_method'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Http server request rate', collapse=true)
          .addPanel(httpQPS( ['grpc_code']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpQPS( ['instance']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(httpQPS( ['grpc_service']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpQPS( ['grpc_service', 'grpc_method']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Http server request error percentage', collapse=true)
          .addPanel(httpErrorPercentage( ['instance']), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpErrorPercentage( ['method','uri']), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Http server 99%-tile Latency of requests', collapse=true)
          .addPanel(httpLatency( ['grpc_code'], 0.99), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpLatency( ['instance'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(httpLatency( ['grpc_service'], 0.99), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpLatency( ['grpc_service', 'grpc_method'], 0.99), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
.addPanel(row.new(title='Http server 90%-tile Latency of requests', collapse=true)
          .addPanel(httpLatency( ['grpc_code'], 0.90), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpLatency( ['instance'], 0.90), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          .addPanel(httpLatency( ['grpc_service'], 0.90), gridPos={ x: 0, y: 0, w: 12, h: 10 })
          .addPanel(httpLatency( ['grpc_service', 'grpc_method'], 0.90), gridPos={ x: 12, y: 0, w: 12, h: 10 })
          , {})
```

创建grafana/dashboard-api.jsonnet

```jsonnet
local dash = import './dashboard.jsonnet';

{
  dashboard: dash,
  folderId: 0,
  overwrite: false,
}
```

生成jsonnet配置

``` bash
jsonnet -J ./grafana/grafonnet-lib  -o ./grafana/dashboards-api/$$app-api.json  --ext-str app=$$app  ./grafana/dashboard-api.jsonnet ;

```

调研grafana api

``` bash
curl -X DELETE --user admin:admin  -H "Content-Type: application/json" 'http://localhost:3000/api/dashboards/db/$$app'
curl -x POST --user admin:admin  -H "Content-Type: application/json" --data-binary "@./grafana/dashboards-api/$$app-api.json" http://localhost:3000/api/dashboards/db 
```



### 生成alermanager 告警

TODO

## 调用链跟踪

### Jaeger

[Jaeger](https://github.com/jaegertracing/jaeger) 是Uber开源的基于[Opentracing](https://opentracing.io/) 的一个实现，类似于zipkin。

创建internal/pkg/jaeger/jaeger.go

```go
package jaeger

import (
     "github.com/google/wire"
     "github.com/opentracing/opentracing-go"
     "github.com/pkg/errors"
     "github.com/spf13/viper"
     "github.com/uber/jaeger-client-go/config"
     "github.com/uber/jaeger-lib/metrics/prometheus"
     "go.uber.org/zap"
)

func NewConfiguration(v *viper.Viper, logger *zap.Logger) (*config.Configuration, error) {
     var (
          err error
          c   = new(config.Configuration)
     )

     if err = v.UnmarshalKey("jaeger", c); err != nil {
          return nil, errors.Wrap(err, "unmarshal jaeger configuration error")
     }

     logger.Info("load jaeger configuration success")

     return c, nil
}

func New(c *config.Configuration) (opentracing.Tracer, error) {

     metricsFactory := prometheus.New()
     tracer, _, err := c.NewTracer(config.Metrics(metricsFactory))

     if err != nil {
          return nil, errors.Wrap(err, "create jaeger tracer error")
     }

     return tracer, nil
}

var ProviderSet = wire.NewSet(New, NewConfiguration)

```

### Grpc

修改internal/pkg/transports/grpc/server.go

```go
          gs = grpc.NewServer(
               grpc.StreamInterceptor(grpc_middleware.ChainStreamServer(
                    otgrpc.OpenTracingStreamServerInterceptor(tracer),
               )),
               grpc.UnaryInterceptor(grpc_middleware.ChainUnaryServer(
                    otgrpc.OpenTracingServerInterceptor(tracer),
               )),
          )
```

修改internal/pkg/transports/grpc/client.go

```go
     conn, err := grpc.DialContext(ctx, target, grpc.WithInsecure(),
          grpc.WithUnaryInterceptor(grpc_middleware.ChainUnaryClient(
               otgrpc.OpenTracingClientInterceptor(tracer)),
          ),
          grpc.WithStreamInterceptor(grpc_middleware.ChainStreamClient(
               otgrpc.OpenTracingStreamClientInterceptor(tracer)),
          ),)

```

### Gin

修改internal/pkg/transports/http/http.go

```go
import "github.com/opentracing-contrib/go-gin/ginhttp"

r.Use(ginhttp.Middleware(tracer))
```

## 单元测试

### 存储层测试

添加repositorys/wire.go 创建要测试的对象，会根据ProviderSet注入合适的依赖。

```go
// +build wireinject

package repositorys

import (
     "github.com/google/wire"
     "github.com/sdgmf/go-project-sample/internal/pkg/config"
     "github.com/sdgmf/go-project-sample/internal/pkg/database"
     "github.com/sdgmf/go-project-sample/internal/pkg/log"
)



var testProviderSet = wire.NewSet(
     log.ProviderSet,
     config.ProviderSet,
     database.ProviderSet,
     ProviderSet,
)

func CreateDetailRepository(f string) (DetailsRepository, error) {
     panic(wire.Build(testProviderSet))
}


```

添加repositorys/products_test.go,这里采用表格驱动的方法进行测试,存储层测试会依赖数据库。

``` go
package repositorys

import (
     "flag"
     "github.com/stretchr/testify/assert"
     "testing"
)

var configFile = flag.String("f", "details.yml", "set config file which viper will loading.")

func TestDetailsRepository_Get(t *testing.T) {
     flag.Parse()

     sto, err := CreateDetailRepository(*configFile)
     if err != nil {
          t.Fatalf("create product Repository error,%+v", err)
     }

     tests := []struct {
          name     string
          id       uint64
          expected bool
     }{
          {"id=1", 1, true},
          {"id=2", 2, true},
          {"id=3", 3, true},
     }

     for _, test := range tests {
          t.Run(test.name, func(t *testing.T) {
               _, err := sto.Get(test.id)

               if test.expected {
                    assert.NoError(t, err )
               }else {
                    assert.Error(t, err)
               }
          })
     }
}

```

运行测试

``` bash
go test -v ./internal/app/details/repositorys -f $(pwd)/configs/details.yml

=== RUN   TestDetailsRepository_Get
use config file -> /Users/xxx/code/go/go-project-sample/configs/details.yml
=== RUN   TestDetailsRepository_Get/id=1
=== RUN   TestDetailsRepository_Get/id=2
=== RUN   TestDetailsRepository_Get/id=3
--- PASS: TestDetailsRepository_Get (0.11s)
    --- PASS: TestDetailsRepository_Get/id=1 (0.00s)
    --- PASS: TestDetailsRepository_Get/id=2 (0.00s)
    --- PASS: TestDetailsRepository_Get/id=3 (0.00s)
PASS
ok      github.com/sdgmf/go-project-sample/internal/app/details/repositorys     0.128s

```

### 逻辑层测试

通过mockery自动生成mock对象.

```go
    mockery --all
```

添加internal/app/details/services/wire.go

```go
// +build wireinject

package services

import (
     "github.com/google/wire"
     "github.com/sdgmf/go-project-sample/internal/pkg/config"
     "github.com/sdgmf/go-project-sample/internal/pkg/database"
     "github.com/sdgmf/go-project-sample/internal/pkg/log"
     "github.com/sdgmf/go-project-sample/internal/app/details/repositorys"
)

var testProviderSet = wire.NewSet(
     log.ProviderSet,
     config.ProviderSet,
     database.ProviderSet,
     ProviderSet,
)

func CreateDetailsService(cf string, sto repositorys.DetailsRepository) (DetailsService, error) {
     panic(wire.Build(testProviderSet))
}


```

存储层使用生成的MockProductsRepository，可以直接在用例中定义Mock方法的返回值。

创建 services/details_test.go

```go
package services

import (
     "flag"
     "github.com/sdgmf/go-project-sample/internal/pkg/models"
     "github.com/sdgmf/go-project-sample/mocks"
     "github.com/stretchr/testify/assert"
     "github.com/stretchr/testify/mock"
     "testing"
)

var configFile = flag.String("f", "details.yml", "set config file which viper will loading.")

func TestDetailsRepository_Get(t *testing.T) {
     flag.Parse()

     sto := new(mocks.DetailsRepository)

     sto.On("Get", mock.AnythingOfType("uint64")).Return(func(ID uint64) (p *models.Detail) {
          return &models.Detail{
               ID: ID,
          }
     }, func(ID uint64) error {
          return nil
     })

     svc, err := CreateDetailsService(*configFile, sto)
     if err != nil {
          t.Fatalf("create product serviceerror,%+v", err)
     }

     // 表格驱动测试
     tests := []struct {
          name     string
          id       uint64
          expected uint64
     }{
          {"1+1", 1, 1},
          {"2+3", 2, 2},
          {"4+5", 3, 3},
     }

     for _, test := range tests {
          t.Run(test.name, func(t *testing.T) {
               p, err := svc.Get(test.id)
               if err != nil {
                    t.Fatalf("product service get proudct error,%+v", err)
               }

               assert.Equal(t, test.expected, p.ID)
          })
     }
}

```

### 控制层测试

添加controllers/details_test.go,利用httptest进行测试

```go
package controllers

import (
     "encoding/json"
     "flag"
     "fmt"
     "github.com/gin-gonic/gin"
     "github.com/sdgmf/go-project-sample/internal/pkg/models"
     "github.com/sdgmf/go-project-sample/mocks"
     "github.com/stretchr/testify/assert"
     "github.com/stretchr/testify/mock"
     "io/ioutil"
     "net/http/httptest"
     "testing"
)

var r *gin.Engine
var configFile = flag.String("f", "details.yml", "set config file which viper will loading.")

func setup() {
     r = gin.New()
}

func TestDetailsController_Get(t *testing.T) {
     flag.Parse()
     setup()

     sto := new(mocks.DetailsRepository)

     sto.On("Get", mock.AnythingOfType("uint64")).Return(func(ID uint64) (p *models.Detail) {
          return &models.Detail{
               ID: ID,
          }
     }, func(ID uint64) error {
          return nil
     })

     c, err := CreateDetailsController(*configFile, sto)
     if err != nil {
          t.Fatalf("create product serviceerror,%+v", err)
     }

     r.GET("/proto/:id", c.Get)

     tests := []struct {
          name     string
          id       uint64
          expected uint64
     }{
          {"1", 1, 1},
          {"2", 2, 2},
          {"3", 3, 3},
     }

     for _, test := range tests {
          t.Run(test.name, func(t *testing.T) {
               uri := fmt.Sprintf("/proto/%d", test.id)
               // 构造get请求
               req := httptest.NewRequest("GET", uri, nil)
               // 初始化响应
               w := httptest.NewRecorder()

               // 调用相应的controller接口
               r.ServeHTTP(w, req)

               // 提取响应
               rs := w.Result()
               defer func() {
                    _ = rs.Body.Close()
               }()

               // 读取响应body
               body, _ := ioutil.ReadAll(rs.Body)
               p := new(models.Detail)
               err := json.Unmarshal(body, p)
               if err != nil {
                    t.Errorf("unmarshal response body error:%v", err)
               }

               assert.Equal(t, test.expected, p.ID)
          })
     }

}


```

### grpc测试

#### 测试Server

添加grpcservers/details_test.go

```go
package grpcservers

import (
     "context"
     "flag"
     "github.com/sdgmf/go-project-sample/api/proto"
     "github.com/sdgmf/go-project-sample/internal/pkg/models"
     "github.com/sdgmf/go-project-sample/mocks"
     "github.com/stretchr/testify/assert"
     "github.com/stretchr/testify/mock"
     "testing"
)

var configFile = flag.String("f", "details.yml", "set config file which viper will loading.")

func TestDetailsService_Get(t *testing.T) {
     flag.Parse()

     service := new(mocks.DetailsService)

     service.On("Get", mock.AnythingOfType("uint64")).Return(func(ID uint64) (p *models.Detail) {
          return &models.Detail{
               ID: ID,
          }
     }, func(ID uint64) error {
          return nil
     })

     server, err := CreateDetailsServer(*configFile, service)
     if err != nil {
          t.Fatalf("create product server error,%+v", err)
     }

     // 表格驱动测试
     tests := []struct {
          name     string
          id       uint64
          expected uint64
     }{
          {"1+1", 1, 1},
          {"2+3", 2, 2},
          {"4+5", 3, 3},
     }

     for _, test := range tests {
          t.Run(test.name, func(t *testing.T) {
               req := &proto.GetDetailRequest{
                    Id: test.id,
               }
               p, err := server.Get(context.Background(), req)
               if err != nil {
                    t.Fatalf("product service get proudct error,%+v", err)
               }

               assert.Equal(t, test.expected, p.Id)
          })
     }

}

```

#### mock grpc client

/internal/app/products/services/products_test.go:

```go
package services

import (
     "context"
     "flag"
     "github.com/golang/protobuf/ptypes"
     "github.com/sdgmf/go-project-sample/api/proto"
     "github.com/sdgmf/go-project-sample/mocks"
     "github.com/stretchr/testify/assert"
     "github.com/stretchr/testify/mock"
     "google.golang.org/grpc"
     "testing"
)

var configFile = flag.String("f", "products.yml", "set config file which viper will loading.")

func TestDefaultProductsService_Get(t *testing.T) {
     flag.Parse()

     detailsCli := new(mocks.DetailsClient)
     detailsCli.On("Get", mock.Anything, mock.Anything).
          Return(func(ctx context.Context, req *proto.GetDetailRequest, cos ...grpc.CallOption) *proto.Detail {
               return &proto.Detail{
                    Id:          req.Id,
                    CreatedTime: ptypes.TimestampNow(),
               }
          }, func(ctx context.Context, req *proto.GetDetailRequest, cos ...grpc.CallOption) error {
               return nil
          })

     ratingsCli := new(mocks.RatingsClient)

     ratingsCli.On("Get", context.Background(), mock.AnythingOfType("*proto.GetRatingRequest")).
          Return(func(ctx context.Context, req *proto.GetRatingRequest, cos ...grpc.CallOption) *proto.Rating {
               return &proto.Rating{
                    Id:          req.ProductID,
                    UpdatedTime: ptypes.TimestampNow(),
               }
          }, func(ctx context.Context, req *proto.GetRatingRequest, cos ...grpc.CallOption) error {
               return nil
          })

     reviewsCli := new(mocks.ReviewsClient)

     reviewsCli.On("Query", context.Background(), mock.AnythingOfType("*proto.QueryReviewsRequest")).
          Return(func(ctx context.Context, req *proto.QueryReviewsRequest, cos ...grpc.CallOption) *proto.QueryReviewsResponse {
               return &proto.QueryReviewsResponse{
                    Reviews: []*proto.Review{
                         &proto.Review{
                              Id:          req.ProductID,
                              CreatedTime: ptypes.TimestampNow(),
                         },
                    },
               }
          }, func(ctx context.Context, req *proto.QueryReviewsRequest, cos ...grpc.CallOption) error {
               return nil
          })

     svc, err := CreateProductsService(*configFile, detailsCli, ratingsCli, reviewsCli)
     if err != nil {
          t.Fatalf("create product service error,%+v", err)
     }

     // 表格驱动测试
     tests := []struct {
          name     string
          id       uint64
          expected bool
     }{
          {"id=1", 1, true},
     }

     for _, test := range tests {
          t.Run(test.name, func(t *testing.T) {
               _, err := svc.Get(context.Background(), test.id)

               if test.expected {
                    assert.NoError(t, err)
               } else {
                    assert.Error(t, err)
               }
          })
     }
}

```

## Makefile

编写Makefile

``` makefile
apps = 'products' 'details' 'ratings' 'reviews'
.PHONY: run
run: proto wire
     for app in $(apps) ;\
     do \
           go run ./cmd/$$app -f configs/$$app.yml  & \
     done
.PHONY: wire
wire:
     wire ./...
.PHONY: test
test: mock
     for app in $(apps) ;\
     do \
          go test -v ./internal/app/$$app/... -f `pwd`/configs/$$app.yml -covermode=count -coverprofile=dist/cover-$$app.out ;\
     done
.PHONY: build
build:
     for app in $(apps) ;\
     do \
          GOOS=linux GOARCH="amd64" go build -o dist/$$app-linux-amd64 ./cmd/$$app/; \
          GOOS=darwin GOARCH="amd64" go build -o dist/$$app-darwin-amd64 ./cmd/$$app/; \
     done
.PHONY: cover
cover: test
     for app in $(apps) ;\
     do \
          go tool cover -html=dist/cover-$$app.out; \
     done
.PHONY: mock
mock:
     mockery --all
.PHONY: lint
lint:
     golint ./...
.PHONY: proto
proto:
     protoc -I api/proto ./api/proto/* --go_out=plugins=grpc:api/proto
.PHONY: dash
dash: # create grafana dashboard
      for app in $(apps) ;\
      do \
           jsonnet -J ./grafana/grafonnet-lib   -o ./grafana/dashboards/$$app.json  --ext-str app=$$app ./grafana/dashboard.jsonnet ;\
      done
.PHONY: pubdash
pubdash:
      for app in $(apps) ;\
      do \
           jsonnet -J ./grafana/grafonnet-lib  -o ./grafana/dashboards-api/$$app-api.json  --ext-str app=$$app  ./grafana/dashboard-api.jsonnet ; \
           curl -X DELETE --user admin:admin  -H "Content-Type: application/json" 'http://localhost:3000/api/dashboards/db/$$app'; \
           curl -x POST --user admin:admin  -H "Content-Type: application/json" --data-binary "@./grafana/dashboards-api/$$app-api.json" http://localhost:3000/api/dashboards/db ; \
      done
.PHONY: docker
docker-compose: build dash
     docker-compose -f deployments/docker-compose.yml up --build -d
all: lint cover docker

```

1. make run 运行项目
2. make wire 生成依赖注入的代码
3. make mock 生成mock对象
4. make test 运行单元测试
5. cover 查看测试用例覆盖度
6. make build 编译代码
7. make lint 静态代码检查
8. make proto 生成grpc代码
9. make docker-compse 启动所有的服务和依赖的中间件，all-in-one

## 框架或库

1. [Gin](https://github.com/gin-gonic/gin) MVC库
2. [gorm](https://github.com/jinzhu/gorm) ORM库
3. [viper](https://github.com/spf13/viper) 配置管理库
4. [zap](https://github.com/uber-go/zap) 日志库
5. [grpc](https://github.com/grpc/grpc) RPC库
6. [Cobar](https://github.com/spf13/cobra) Command开发库
7. [Opentracing](https://opentracing.io/) 调用链跟踪
8. [go-prometheus](https://github.com/prometheus/client_golang) 服务监控
9. [wire](https://github.com/google/wire) 依赖注入
