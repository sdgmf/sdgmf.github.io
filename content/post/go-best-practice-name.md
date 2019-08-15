---
title: "Golang最佳实践(基础篇)-命名"
date: 2019-08-14T17:21:46+08:00
publishdate: 2019-08-14
lastmod: 2019-08-14
draft: false
tags: [golang]
---

良好的命名和统一的编程风格可以让Go程序员容易理解你编写的程序。

## 基本规则

Go约定使用MixedCaps或者mixedCaps的形式，而不是下划线来书写多个单词的名字。

## 项目名(仓库名)

1. 项目名(仓库名）的命名可以使用字母、数字。
2. 多个单词建议采用中划线分隔，目前github中大多数项目都是使用用中划线，不建议采用驼峰式分隔，不要使用下划线(kubernetes中的组件名称不允许使用下划线)
3. 命名可以是对项目功能的描述；也可以是一个代号（如神话人物的名字，或者希腊语）,适合采用代号的项目有两种，一种是公司的基础组件或者开源项目，一般这种项目都有详细的文档，
4. 项目名（仓库名）要尽量避免重复，如果可能重复要添加必要的前缀或者后缀做区分。
5. 命名尽量在三个单词以内。

正确:

```txt
user、user-api、user-service,product、product-search、redis-go,druid、zeus、kubernetes.
```

错误:

```txt
user_api、Product
```

## 包名

1. 包名应该尽量简短，简明,一个单词来命名,不要用多个单词命名。
2. 其不需要在所有源代码中是唯一的。
3. 不要使用import . "xxx" 导入程序包。
4. 包内struct、function不要带入包名。
5. 包名和目录名保持一致。
6. 避免使用诸如base，util或common之类的包名称,按照提供的内容命名您的包，而不是它们包含的包。

## Get方法

> Go不提供对Get方法和Set方法的自动支持。你自己提供Get方法和Set方法是没有错的，通常这么做是合适的。但是，在Get方法的名字中加上Get，是不符合语言习惯的，并且也没有必要。如果你有一个域叫做owner（小写，不被导出），则Get方法应该叫做Owner（大写，被导出），而不是GetOwner。对于要导出的，使用大写名字，提供了区别域和方法的钩子。Set方法，如果需要，则可以叫做SetOwner。这些名字在实际中都很好读:

```golang
owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}
```

## 变量名

1. 作用域越小，命名应该越简短。如在for循环内部用i表示index。
2. 变量名不应该包含类型，变量的名称应描述其内容，而不是内容的类型.
3. struct 的行为方法，命名不要采用self,this等方式的命名。

## 接口名

> 按照约定，单个方法的接口使用方法名加上“er”后缀来命名，或者类似的修改来构造一个施动者名词：Reader，Writer，Formatter，CloseNotifier等。
> 有许多这样的名字，最有效的方式就是尊重它们，以及它们所体现的函数名字。Read，Write，Close，Flush，String等，都具有规范的签名和含义。为了避免混淆，不要为你的方法使用这些名字，除非其具有相同的签名和含义。反过来讲，如果你的类型实现了一个和众所周知的类型具有相同含义的方法，那么就使用相同的名字和签名；例如，为你的字符串转换方法起名为String，而不是ToString。

## New方法

1. 当包里只有一个对象时，比如gin包，New方法可以直接写成gin.New(),而不是gin.NewEngine()

## Constant

1. 建议采用驼峰式命名，不要采用全部大写，全部大写加下划线的方式很难阅读。

## 参考

[https://golang.org/doc/effective_go.html](https://golang.org/doc/effective_go.html)

[https://www.kancloud.cn/kancloud/effective](https://www.kancloud.cn/kancloud/effective)

[https://dave.cheney.net/2019/01/29/you-shouldnt-name-your-variables-after-their-types-for-the-same-reason-you-wouldnt-name-your-pets-dog-or-cat](https://dave.cheney.net/2019/01/29/you-shouldnt-name-your-variables-after-their-types-for-the-same-reason-you-wouldnt-name-your-pets-dog-or-cat)

[https://dave.cheney.net/2019/01/08/avoid-package-names-like-base-util-or-common](https://dave.cheney.net/2019/01/08/avoid-package-names-like-base-util-or-common)