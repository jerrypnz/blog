---
layout: post
title: 半年 Scala 小感——Monad 篇
date: 2015-08-27 10:38:49 +0800
comments: true
categories: Tech
tags: Scala, FP, Monad
keywords: Scala, FP, Monad
description: 在新公司使用 Scala 小半年了，对其了解逐渐深入，现在是时候 聊聊我对这个语言的感觉了。这一次先说我喜欢的地方：ADT 和 Monad。
---

[上一次]({% post_url 2015-07-21-half-year-of-scala-adt %})聊了聊 Scala
里的代数数据类型（ADT），这一次继续聊聊 Scala 里我比较喜欢的地方：实用
的 Monad。

早在我刚开始学习函数式编程的时候，我就曾试图弄明白 Monad 这个概念，然
而看过很多文章后依然一头雾水。究其原因，可能是我当初使用的语言 Clojure
里，Monad 并不是很常用，所以没有办法通过一些十分简单的例子来理解这个十
分抽象的概念。在当时研究 Continuation 的时候，我甚至计划
[啃下 Continuation Monad]({% post_url 2013-01-30-playing-continuation-part1 %})
这个硬骨头，然而十分惭愧的是最后也不了了之了（最近准备用 Scala 再试试）。

然而这几个月开始学习和使用 Scala，并在实际代码中看到、用到几个很基础的
Monad 之后，我突然茅塞顿开，有点明白 Monad 了。

<!--more-->

先说说我当前对 Monad 很粗浅的理解：Monad 其实就是包含了一个（或多个）
value 的一个 context，其核心操作是 `flatMap` (就是 `bind`，Haskell 里
的 `>>=`)：

```scala
def flatMap[B](f: (A) ⇒ M[B]): M[B]
```

它将自己所包含的这个 value 拿出来，调用传入的函数得到包含另一个 value
的一个新的 context，然后对其执行某种逻辑然后返回。

传给 `flatMap` 的是用户的计算逻辑，而发生在 `flatMap` 实现里的，是每种
Monad 独有的处理，我们可以对一个 Monad 不断调用 `flatMap`，以有趣的方
式串联起多个操作。所以 Monad 还是一种组合不同计算步骤的方式：

> A type with a monad structure defines what it means to chain
> operations, or nest functions of that type together. This allows the
> programmer to build pipelines that process data in steps, in which
> each action is decorated with additional processing rules provided
> by the monad.

我知道，这样还是太抽象，那就以 Scala 里帮助了我理解 Monad 的几个有趣的
类型为例说一说吧。

## `Option[T]`

`Option` 其实就是 Haskell 里的 `Maybe`，它包含 `Some[T]` 和 `None` 两
种可能的取值。一般在 Java 里需要用 `null` 表示一个值不存在，而在 Scala
里这通常用 `Option` 来作为这种值的类型，用 `None` 来表示不存在。

那么对于 `Option` 而言，`flatMap` 都做了什么呢？其实很简单：如果是
`Some[T]`，就取出其包含的值，调用传入的函数得到一个新的 `Option` 并直
接返回；如果是 `None` 就什么都不做，直接返回 `None`。所以它所做的事情，
就是根据取值来决定要不要继续下一步计算。这能让我们写出十分简洁的代码。
比如 Java 里我们需要显式处理 `null`：

```java
public Integer getHeight() {...}
public Integer getWidth() {...}

public Integer getArea() {
    Integer h = getHeight();
    Integer w = getWidth();
    if (h == null || w == null) {
        return null;
    }
    return h * w;
}

```

在 Scala 里使用 Option 就很愉快：

```scala
def getHeight: Option[Int] = {...}
def getWidth: Option[Int] = {...}

def getArea: Option[Int] =
  getHeight flatMap { h ⇒
    getWidth map { w ⇒
      w * h
    }
  }
```

我知道，我知道，嵌套的 `flatMap` 和 `map` 很丑陋，我们还有 `for...yield`：

```scala
def getArea: Option[Int] =
  for {
    h ← getHeight
    w ← getWidth
  } yield h * w
```

Scala 里的 `for` 基本上就是 Haskell 里的 `do`，是嵌套的`flatMap`，
`map` 和 `filter` 调用的语法糖。

你可能注意到了上面我用到了一个 `map`，和 `flatMap` 不同的是，它所接受
的函数 `f` 应该返回的是 value 而不是带着 value 的 context，即：

```scala
def flatMap[B](f: (A) ⇒ M[B]): M[B]
def map[B](f: (A) ⇒ B): M[B]
```

这个其实是 `Functor` 的基本操作，有兴趣的同学可以自行了解一下。

## `Try[T]`

`Try[T]` 是个很有意思的 Monad，它包含 `Success[T]` 和 `Failure` 两个取
值，和 `Option[T]` 很像，但不同的是其 `Failure` 包含着一个 `Throwable`。

就像名字所暗示的，它是用来处理异常的。例如我们可以将可能抛出异常的方法
的返回值设计成 `Try[T]` 类型的，然后用 `flatMap` 或者 `for` 来串联多个
函数调用并集中处理异常。同 `Option` 一样，`flatMap` 仅在 `Success` 的
时候调用给定函数，在 `Failure` 的情况下什么都不做，仅仅返回本身。

```scala
def openFile(name: String): Try[File] = Try {...}
def readAsString(file: File): Try[String] = Try {...}
def parseXML(xml: String): Try[Data] = Try {...}

def deserializeFile(name: String): Try[Data] =
  for {
    file ← openFile(name)
    xml ← readAsString(file)
    data ← parseXML(xml)
  } yield (data) recoverWith {
    case e: IOException ⇒
      log.error("Error opening file")
      ...
  }
```

我们可以用上面提供的一系列方法如 `getOrElse`, `recover`, `recoverWith`
等等来进一步处理异常。

比起 Java 的异常处理方式，`Try` 将错误处理编码进类型里，为其提供更丰富
的操作，同时得益于 `for` 的语法糖，使用起来并不会更加麻烦。

## `Future[T]`

首先要说明的是，`Future` 虽然有 `flatMap` 等 Monadic 操作，但
[它不是一个 Monad](http://stackoverflow.com/questions/27454798/is-future-in-scala-a-monad)
，因为其不符合 Monad 定律。这里就不展开细说了，有兴趣的可以自行 Google
一下。

然而这并不妨碍我们像使用 Monad 一样用 `flatMap` 和 `for` 来操作 `Future`。
`Future` 是一个包含将来可能得到的 value 的 context，不难想象，它的
`flatMap` 是用来异步的对一个 `Future` 的结果调用指定函数，并返回一个新的
`Future` 用于承载结果。基于这个，我们能写出非常有意思的异步处理代码：

```scala
def compareSearchEngines(keyword: String): Future[Result] = {
  val googleSearch = httpGet(genGoogleSearchURL(keyword))
  val baiduSearch = httpGet(genBaiduSearchURL(keyword))
  val bingSearch = httpGet(genBingSearchURL(keyword))

  for {
    googleRes ← googleSearch
    baiduRes ← baiduSearch
    bingRes ← bingSearch
  } yield compareResults(keyword, googleRes, baiduRes, bingRes)
}
```

上面的方法并发请求三个搜索引擎的结果，并异步等待他们的结果返回以后，比
较他们的搜索结果。可以看到，其代码相当简洁。

`Future` 也能捕捉可能的异常，并提供了和 `Try` 类似的方法用于处理错误。


## 还需再学习

其实上面列出来的，还是十分简单的几种 Monad，不像 Haskell，Scala 是允许
副作用的，所以也不太需要更高级的 IO Monad, State Monad 等，日常使用的
还是上面列出来的这几个。

然而有了这个基础，我十分期待能学得更多一点。从简单一些的 Reader,
Writer Monad 开始，一步步把 Haskell 标准库里所包含的那些 Monad 都了解
一遍，可能的话，自己在 Scala 里实现一遍作为练习。

其实最后还是要学 Haskell 的，不是吗？;-)
