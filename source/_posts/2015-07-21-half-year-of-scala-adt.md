---
layout: post
title: 半年 Scala 小感——ADT 篇
date: 2015-07-21 19:22:08 +1200
comments: true
categories: Scala
tags: Scala, FP, ADT
keywords: Scala, FP, ADT
description: 在新公司使用 Scala 小半年了，对其了解逐渐深入，现在是时候 聊聊我对这个语言的感觉了。这一次先说我喜欢的地方：ADT 和 Monad。
---

作为一个 [Scala Shop](http://movio.co/career/#gotoOpenPositions)，我们
的主力编程语言自然非它莫属了。入职小半年，我在工作中也用到了不少 Scala，
对其了解也逐渐深入，现在是时候聊聊我对这个语言的感觉了。

必须声明的是，我还是个 Scala 菜鸟，对 Scala 的很多特性，尤其是类型系统
了解还不够深入，所以这里只谈我了解的部分。这一次我想说的是代数数据类型
（Algebraic Data Type）。

[代数数据类型（ADT）](https://en.wikipedia.org/wiki/Algebraic_data_type)
在函数式编程，尤其是静态类型的 FP 语言中很常见，它是一种组合已有类型形
成新类型的方式。组合方式有两种：Product 或者 Sum。前者几乎所有编程语言
都有，就是记录、Struct 或者 Class 等，它的取值范围是所有字段取值范围的
笛卡尔积；后者一般在 ML、Haskell、Scala、F# 等静态类型函数式语言里才有，
也叫 Tagged Union，每一种可能的取值都叫一个 variant， 它的取值范围是所
有 variant 的并集。比如最典型的 `Option[T]` 类型，取值要么是 `None`，要么
是 `Some[T]`。

第一次听说 Sum Type 这个概念是在学 Standard ML 的时候，当时看到了就眼
前一亮。配合模式匹配，代码写起来太简洁且富有表达力了。而真正让我更加认
同 Sum Type 的是 Scott Wlaschin 在他的
[Functional Design Patterns](http://www.slideshare.net/ScottWlaschin/fp-patterns-buildstufflt)
里说的一句话（顺带一提，强烈推荐仔细看看这个 slides，全是干货）：

> “Make illegal states unrepresentable”

<!--more-->

而实现这一点的关键就是 Sum 类型。说到这里不得不贴上他那个 slides 里的
一页：

{% img /images/201507/sum-type.png %}

可以看到 Sum 类型是一个闭环，类型的定义已经包含了所有可能性，绝无可能
会出现非法状态。而 OO 就无法做到这一点
（
[当然硬说是可能的](http://grundlefleck.github.io/2014/07/17/sealing-algebraic-data-types-in-java.html)
，但是缺少模式匹配的话也并没有什么卵用）。

Scala 当然是支持 Sum Type 的，但是写起来并没有 Haskell，ML 或者 F# 里
那么优雅，上面的例子用 Scala 写的话是这样的：

```scala
sealed abstract class CardType
case object Visa extends CardType
case object MasterCard extends CardType

sealed abstract class PaymentMethod
case object Cash extends PaymentMethod
case class Cheque(chequeNumber: String) extends PaymentMethod
case class Card(cardType: CardType, cardNumber: String) extends PaymentMethod
```

关键就在其中的 `sealed` 关键字，它能禁止一个类在同样的源文件之外被继承，
从而达到了上面说的『闭环』的效果。Scala 里有很多这样的例子：`Option`,
`Try`, `Either` 等等。

使用 case class 来写代码的标准姿势是模式匹配：

```scala
def payMe(method: PaymentMethod) = method match {
  case Cash ⇒
    println("Pay with cash")
  case Cheque(checkNum) ⇒
    println(s"Pay with cheque: $checkNum")
  case Card(Visa, cardNum) ⇒
    println(s"Pay with a Visa card: $cardNum")
  case Card(MasterCard, cardNum) ⇒
    println(s"Pay with a MasterCard: $cardNum")
}

payMe(Cash)
payMe(Cheque("123456"))
payMe(Card(Visa, "54321"))
payMe(Card(MasterCard, "98765"))
```

case class 还是不可变的值类型，是 Scala 中推荐的表达数据的方式。它用起
来也十分方便，`hashCode`，`equals`，`toString` 等方法默认都已经实现好
了。

最后想说的一点是，我其实挺希望 Scala 在设计之初就完全抛弃 OO 的部分，
做成一个完全的 FP 语言，而不是像现在这样混合 OO 和 FP，把语言搞得特别
复杂。我更希望它是一个运行在 JVM 上，语法比起 Haskell 更友好一些，几乎
是纯粹函数式的静态类型语言，类似 JVM 上的 F# 那种意思吧。

P.S. 又是快两个月没有更新博客了，主要原因是，咳咳，买了 PS4。最近的大
部分业余时间都花在 [Witcher 3](http://www.thewitcher.com/) 上了。其实
我挺想写写游戏相关的文章的，就是不知道有没有人愿意看。
