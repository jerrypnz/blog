---
layout: post
title: 撸了个玩具级 Lisp 解释器
date: 2015-03-22 14:35:30 +1300
comments: true
categories: Java
tags: Lisp, Java
keywords: Lisp, Java
description: 几个月开坑写了一个十分简单的 Lisp 解释器，最近终于完成比较核心的功能了。
---

相信 10 个 Lisper 中有 9 个会写个自己的 Lisp 解释器玩吧，我当然也不能
免俗了。去年 12 月初还在国内的时候就开了个坑写了这么个简单的
[toylisp](https://github.com/moonranger/toylisp)，两天时间就弄出来一个
具备词法作用域的核心 Lisp，但是没有宏，所以功能有限，只具备核心的几个
函数和 special form：`lambda`，`def`，`car`，`cdr`，`eq?`，`cond`，
`quote` 等，还有几个基本的算术函数。像 `defun`，`if`，`let` 等可以宏来
实现的操作符就没有定义。

最近的闲暇时间又将其重新拾起来了，这次将宏实现出来了，并加入了一个小小
的核心库供解释器启动时自动加载。几个基本的 Lisp 解释器和配套功能算是有
了，现在可以拿出来说说了。

<!-- more -->

## 1. John McCarthy 的论文

强烈推荐所有的 Lisper 都读一遍 John McCarthy 最早的那篇关于Lisp 的论文
[Recursive Functions of Symbolic Expressions and Their Computation by Machine](http://www-formal.stanford.edu/jmc/recursive.pdf)
，看看大神是如何一步步将 Lisp 的核心展现在你眼前。他先引入数学中的偏函
数、条件表达式、递归函数定义等概念，厘清了函数和 form （我还是不要翻译
为『形式』了）的区别，然后开始介绍符号表达式极其表现形式 S-exp，和针对
S-exp 的基本操作如 `car`, `cdr`, `cons`, `eq`, `atom` 等。紧接着就是最
精彩的部分，即用 S-exp 来表达函数定义并通过一个通用的 `apply` 和
`eval` 函数来执行。其中的 `eval` 就是 Lisp 语义的核心了，读懂了这个，
也就知道如何写 Lisp 解释器了。

可以看到，John McCarthy 设计的根本不是一个编程语言，而自成体系的一套图
灵机等价物（但更简单），并不与特定的计算机架构相关。真正实现一个 Lisp，
只需要在底层提供一个 `cons` 存储机制和相关的那几个核心操作即可，上层的
一切都可以在这个基础上构建出来。这个 idea 实在太棒了，甚至让人怀疑
[我们的世界是不是用 Lisp 构建出的](https://xkcd.com/224/)（大误）。

## 2. toylisp 实现细节

[toylisp](https://github.com/moonranger/toylisp) 并没有严格按照论文中
的描述来实现，其 `eval` 不是用 Lisp 实现的，而是直接在宿主语言 Java 中
实现的，因为将来打算针对其做一些优化。

下面介绍一些我觉得实现过程中比较有意思的地方。

### 闭包

我的解释器实现采用了词法作用域，并实现了闭包功能。在着手写代码的时候，
我卡壳了。我当时在想：闭包引用了其定义时所在的环境，但是在调用闭包时还
有另一个调用环境，怎么同时处理这两个环境呢？哪个更优先？是否需要合并两
者？

在怎么做都觉得不对劲的时候，我还是没忍住 Google 了一下，然后在万能的
Stack Overflow 上找到了
[答案](http://stackoverflow.com/questions/2384157/how-is-lexical-scoping-implemented)
。原来正确的做法是捕获函数定义时的环境，但在调用闭包的时候 *不要* 传入
环境，而是仅仅传递参数即可（本来也要做的事情）。函数执行时所在的环境是
其定义环境，而不是执行环境。仔细想想，这不正符合词法作用域的定义吗？变
量作用域是静态的，是程序的词法结构决定的。

### Reader

我觉得 Reader 是一个简单的 Lisp 解释器中相对最复杂的地方，至少对我来说
是这样。我原本的 Reader 实现混乱不堪，各种标志位和 `if`, `else`。不过
在实现宏的时候我重构了一下，写了一个递归下降的 parser。对，我现在终于
明白什么是递归下降的 parser 了，其实背后的思想很简单，写出来的实现可读
性也很不错。

### Backquote

虽然不是必须的，但 Backquote 对于方便宏的编写至关重要。实现到到这里我
又有点没灵感了，这次可耻地参考了下 Clojure 的实现方式。简单地说，对于
Backquote 这样处理即可：

```cl
; Before
`(a (b1 b2) ,c ,@d)
; After
(concat (list (quote a)) (list (concat (list (quote b1)) (list (quote b2)))) (list c) d)
```

（一点都不简单好吗！）

其实就是把 Backquote 中的所有 S-exp 改写成用 `concat` 和 `list` 来组装，
这样得到的 S-exp 是等价的。对于其中包含的 Symbol，全部 `quote` 一下，
而通过 `,` unquote 掉的则不需要作处理，对于 `,@` 则更进一步地去掉
`list`调用，这样 unquote 的内容就能和外层 S-exp 合并了。这部分我是在
Reader 中实现的。


## 3. 未来想做的

如果就此止步，实在是不怎么好玩，也就是用 Java 翻译了一下 McCarthy 的论
文而已，没啥意思。但我觉得有几个点是可以继续投入精力做做，并且挺『高大
上』的，哈哈。

### 重新实现 eval

虽然 `eval` 是个很自然的递归函数，但如果实现也还是完全用递归，性能会很
不好（一小破玩具解释器还说毛性能）。我相信有办法改写 `eval` 使其不
必用递归，接下来要研究研究。

### TCO

是的，尾递归优化。我希望我的小 Lisp 更接近 Scheme，更鼓励使用递归、不
可变数据等。这种情况下 TCO 是个很有价值的功能。我现在还对其完全没有概
念，一番 Google 之后发现可能会用到 CPS 变换、trampoline 等高大上的技术，
只是看看就觉得好牛逼，有时间要好好钻研下。

### Compiler

不完全算是编译器，可能更接近一个宏展开器（当然未来不排除做成全功能的
compiler）的东西。现在的实现里，宏展开做得非常简单，就是每次调用时展开，
这会带来不必要的额外开销。我想实现一个 Compiler，能在程序载入的时候将
所有的宏都展开。不过这可能并不是很简单的事，因为宏可能会调用其他的函数，
必须确保宏展开的时候，其依赖是已经加载好了的。

### Java Interop

就算把上面的所有全部搞定，这个解释器也无法用来写一个真正有用的程序，因
为缺少了太多 API 了。但只要加入了 Java Interop 的功能，让其可以调用任
意 Java 代码，那就不一样了。


## 4. 写在最后

现在还不确定这些新坑能填多久，能不能填起来，但无论如何写 Lisp 解释器真
是很有意思的一件事：简单一个解释器半天功夫就能写出来，但高阶的功能每个
都是个不小的挑战。再次贴出这个小解释器的
[链接](https://github.com/moonranger/toylisp)，欢迎 fork、star、提 issue。

## 5. 参考

- Why Lisp？ http://c2.com/cgi/wiki?WhyLisp
- The Root of Lisp http://ep.yimg.com/ty/cdn/paulgraham/jmc.ps
