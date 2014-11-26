---
layout: post
title: 玩玩 core.logic
date: 2014-11-25 23:41:34 +0800
comments: true
categories: Clojure
tags: Clojure, core.logic
keywords: Clojure, core.logic
description: 最近工作中遇到的一个问题，我感觉用逻辑编程可以很轻松解决， 遂决定拿 core.logic 小试牛刀一把。
---

我第一次接触逻辑编程是在
[《七周七语言》](http://book.douban.com/subject/10555435/)中看到的对
Prolog 的介绍。当时一知半解，但感受到了逻辑编程在解决某类问题时的威力。
之后开始玩 Clojure，Prolog 也被我放到一边没有深入学习。

最近在公司做一个项目的时候发现要解决的问题似乎很适合逻辑编程（我直接用
Java 也没太费功夫就实现了，但不管咋样，玩玩呗），遂决定拿来练习一下。
当然我不再打算学习 Prolog 了，而是直接使用 Clojure 的逻辑编程库
[core.logic](https://github.com/clojure/core.logic)。

废话不多说，开始吧。

## 1. 问题描述

这个是航空行业的一个问题，但我相信大家都能明白。简单地说就是要根据一组
航班号确定其代表的出发地、到达地和中转地，例如航班号是 CZ323 和 CZ305，
那么行程是从北京到广州转机到奥克兰。乍一看似乎很简单，但是要考虑一个
特殊情况，且听我道来。

每个航班都会有一个唯一的航班号，形如 MU779，其中 MU 是航空公司的代码，
779 是航班号。每个航班的出发到达地通常是固定的（为简化问题，不考虑航班
排期变化的情况），比如 MU779 是 SHA-AKL 的（上海到奥克兰）。

恶心的是，航班可能有经停。所谓经停，就是航班在中间的某个城市降落，目的
地是这个城市的乘客直接下来，其他乘客也先下去等会，到时间了再跟新来的乘
客一起再回到飞机上继续飞到最终的目的地（就像火车中间停多个站一样）。比
如CZ323 这个航班就是 PEK-CAN-PNH（从北京经停广州飞往金边）。对于这样的
航班，光给我一个航班号是无法确定乘客从哪上，要去哪的，有可能是PEK-CAN，
PEK-PNH，也可能是CAN-PNH。

如果是直达行程，我们就无能为力了，但如果是有转机的情况（转机就是到一个
目的地之后换另一个航班），是能根据后面的航班作出推断的，因为转机肯定要
在一个城市的，我不可能飞到广州然后到深圳转另一个航班（一般的机票检索系
统不会这么干，也几乎没有这样的运价）。比如现在我知道乘客坐的是 CZ323
加上 CZ305 这两个航班，我只需看看 CZ305 航班的出发城市，就知道 CZ323 这
段是到达的什么地方。CZ305 是 CAN-AKL 的，没有经停的航班，那反过来就可以
推断 CZ323 这段是 PEK-CAN 的。简单的说就是**前一段的到达地和下一段的出
发地要一致**。

问题描述完毕，我们要做的就是，给出一组航班号，推断出可能的行程（航班数
据是给定的）。

<!--more-->

## 2. 开始编程

### 数据准备

首先我们需要航班的基础数据，就是一个航班号对应的所有可能的出发到达，这
个用一个简单的 Map 就可以：

```clojure
(def flights-database {"CZ323" [["PEK" "PNH"]
                                ["PEK" "CAN"]
                                ["CAN" "PNH"]]
                       "CZ305" [["CAN" "AKL"]]})
```

实际的应用，这个数据应该从数据库中加载（对于国际航班，数据量会很大）。

### 定义逻辑变量和程序骨架

逻辑编程的核心我认为有两点：

1. 对问题抽象，定义出逻辑变量
2. 定义逻辑变量需要满足的约束

程序运行的过程就是不断的对逻辑变量尝试『合一（unification）』，使其满
足给定的关系，并在失败时『回溯』，尝试别的解。

我们这个问题，未知的是每个航班对应的出发到达，所以首先定义逻辑变量来表
达这些『未知数』，这里同时给出核心函数的框架：

```clojure
(defn infer-orig-dest [flights]
  (let [vars (repeatedly (count flights) lvar)
        ...]
    (run* [q]
      (== q vars)
      ...)))
```

因为我们的逻辑变量个数不是固定的，而是和航班个数一致，所以这里不用
`fresh` 来定义逻辑变量，而是直接调用 `lvar` 函数来生成。`repeatedly`产
生一个 lazy seq，长度由第一个参数 `(count flights)` 指定，内容不断调用
`lvar` 得到的。所以 `vars` 就是一个包含和 `flights` 一样大小的逻辑变量
集合。

下面 body 部分的 `run*` 实际运行逻辑表达式，它定义了一个逻辑变量 `q`
用于最终返回，我们首先要写上一个 `(== q vars)`，含义是直接取 `vars` 作
为我们要的结果。

骨架建好了，下面一步步填充。

### 初始化每个航班可能的出发到达

我们有一组和航班个数相同的，代表最终出发到达地的逻辑变量，现在要做的是
初始化每个逻辑变量可能的值——即航班所有可能的出发到达。

这个在逻辑编程里要使用关系来表达，我们在这里用了两个函数 `everyg` 和
`membero`。其中 `everyg` 类似 `every?`，只不过针对的是关系，含义是对集
合的每个元素调用给定函数，且要满足函数中定义的所有关系。`membero` 则表
达包含关系，即第一个参数指定的逻辑变量被包含在第二个参数指定的集合里——
这里正是通过 `flights-database` 查出的所有可能的出发到达。

为方便调用 `everyg`，这里用 `map` 和 `vector` 先把逻辑变量和对应的航班
组合到一起了，匿名函数里再用 destructuring 解开。

```clojure
(defn init [vars flights]
  (everyg (fn [[var flight]] (membero var (flights-database flight)))
          (map vector vars flights)))
```

这里给出我最初写的版本，是手工用递归写的，更啰嗦一点，从里面也可以看到
逻辑关系函数必须要返回 `succeed` 后者 `failed`。这也是为什么不能用普通
函数的原因：

```clojure
(defn init [vars flights]
  (if (seq vars)
    (all 
     (membero (first vars) (flights-database (first flights)))
     (init (rest vars) (rest flights)))
     succeed))
```

### 定义航班之间的连接关系

现在要定义两个航班之间的连接关系，即相邻的两个航班，前者的到达地和后者
的出发地要一致。我们首先定义这样的一个函数：

```clojure
(defn connectedo [[var1 var2]]
  (fresh [x y z]
    (== var1 [x y])
    (== var2 [y z])))
```

注意函数名是用 o 结尾的，这是 core.logic 的约定：关系函数都以 o 结尾，
前面的 `membero` 就是一个例子。

该函数的参数是一个二元组，其中的 `var1` 是前面航班出发到达，`var2` 是
后面航班的出发到达， 我们要声明的关系是，前者的第二个元素和后者的第一
个元素是相等的。这里先用 `fresh` 定义三个新的临时使用的逻辑变量 `x`
`y` `z`，然后用 `==` 来作合一。注意 `y` 出现了两次，一前一后，因为是同
一个逻辑变量，所以这就间接声明了 `var1` 的尾和 `var2` 的头要一致。`x`
和 `z`则没有别的用途了。

其实本来我是写成这样的：

```clojure
(defn connectedo [[var1 var2]]
  (== (first var1) (second var2)))
```

结构发现直接报错了，告诉我 `var1` 不是 `ISeq` 云云。我还没有特别想明白
为什么，我猜测关系还是应该直接在逻辑变量这一级上表达，无法像操作数据结
构那样取出其中的元素再合一？

### 组合到一起

现在我们可以给出最终的 `infer-orig-dest` 函数了：

```clojure
(defn infer-orig-dest [flights]
  (let [vars (repeatedly (count flights) lvar)
        connections (partition 2 1 vars)]
    (run* [q]
      (== q vars)
      (init vars flights)
      (everyg connectedo connections))))
```

其中 `connections` 是用 `partition` 是两两分组，将所有相邻的的逻辑变量
放到一起，方便后面调用 `connectedo` 函数。不熟悉 `partition` 的，可以
参考下面的代码片段：

```clojure
user> (partition 2 1 [:a :b :c])
((:a :b) (:b :c))
```

`run*` 的 body 部分定义了三个表达式：

- 第一个刚才介绍过，用于得到最终结果
- 第二个是调用 `init`，查询数据确定每个航班所有可能的出发到达
- 第三个是对每个`connnections` 中的元素调用 `connectedo`，必须全部满足
其定义的同城市连接关系才行。

拿前面的例子试验一下：

```clojure
core-logic-playground.core> (infer-orig-dest ["CZ323" "CZ305"])
((["PEK" "CAN"] ["CAN" "AKL"]))
```

哈哈，Bingo！增加些数据，尝试个更复杂的（数据是乱编的，不是实际的航班）：

```clojure
(def flights-database {"CZ323" [["PEK" "PNH"]
                                ["PEK" "CAN"]
                                ["CAN" "PNH"]]
                       "CZ305" [["CAN" "AKL"]
                                ["AKL" "MEL"]
                                ["CAN" "MEL"]]
                       "QF123" [["MEL" "PAR"]
                                ["MEL" "LON"]
                                ["PAR" "LON"]]})
```

试试这个航班组合：

```clojure
core-logic-playground.core> (infer-orig-dest ["CZ323" "CZ305" "QF123"])
((["PEK" "CAN"] ["CAN" "MEL"] ["MEL" "PAR"])
 (["PEK" "CAN"] ["CAN" "MEL"] ["MEL" "LON"]))
```

可以看到有两种可能的结果都被找出来了。


## 3. 总结

到这里这个程序就结束了，完整的程序可以在我的
[Github](https://github.com/moonranger/playground/blob/master/clj-projects/core-logic-playground/src/core_logic_playground/core.clj)
上找到，可以看到它很简单，并且完全是声明式的。当然了，这个问题本身已经
是简化过了的，实际除了要推断出发到达地以外，还要根据第一段的出发日期推
断出后面每一段的出发日期（要考虑航班可能跨天，且出发到达地时区的差异）。
另外它也没有考虑同城市跨机场转机的情况。等有时间了，我准备把它完整实现
出来，真正和 Java 版本比较下。

下面奉上一些关于 core.logic 的链接（其中有一个用类型检查作为示例的教程；
最后的那个 gist 是一个 30 多行的解数独程序，很有意思）：

- https://github.com/clojure/core.logic/wiki/A-Core.logic-Primer
- https://github.com/frenchy64/Logic-Starter/wiki
- http://www.intensivesystems.net/tutorials/logic_prog.html
- https://gist.github.com/swannodette/3217582

Have fun :-)
