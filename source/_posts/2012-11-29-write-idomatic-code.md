---
title: 写出 Idiomatic 的代码
author: Jerry
layout: post
permalink: /2012/11/write-idomatic-code/
categories:
  - Java
  - 技术
tags:
  - Clojure
  - Idiomatic
  - Java
---
Idiomatic 这个词我还是在学习 Python 的时候在一篇介绍Pythonic 和 Python 编程风格的文章 [Code Like a Pythonista: Idiomatic Python][1] 里看到的。这个词的意思是”符合语言习惯的“，用在编程语言里，意思就是代码风格比较符合某个语言一贯的用法。我认为代码漂不漂亮，第一要看的就是它是否 Idiomatic 。

下面说一个例子，是我最近这两天写出来的代码，用的语言是 Java 。这段代码 做的事情是根据参数生成动态的 SQL ，例子中的片段是对一个 IN 子句的处理。 输入的参数一个逗号分割的字符串列表，里面的元素要放到 IN 子句里。我们用的是 `PreparedStatement` ，在 JDBC 里，这种情况需要在 IN 后面写上和参数个 数那么多个 `?` 。这段代码的逻辑就是分割参数，然后拼装出若干个 `?` ，并 逐一将参数放到一个列表里，供后面实际做查询时使用。

说了这么多，下面贴出我最终写出来的代码：

<pre lang="java">if (StringUtils.isNotBlank(criteria.getCategory())) {
    String[] categories = criteria.getCategory().split(",");
    sql.append(" AND d.CATEGORY in (?");
    args.add(categories[0]);
    for (int i = 1; i &lt; categories.length; i++) {
        sql.append(",?");
        args.add(categories[i]);
    }
    sql.append(") ");
}
</pre>

这是一段很朴实，说实话也挺啰嗦的代码，但是我纠结了半天才写出来的。我在 纠结什么呢？对，就是问号中间的那个逗号的处理。用循环来逐一将 `?` 添加 到 `StringBuilder` 中的时候，总会多出来一个逗号（无论是在前面还是后面）， 解决办法一般是：

1.  每次循环添加一个 &#8220;?,&#8221;，结束后把最后一个逗号删除掉
2.  在循环里判断是不是第一次，第一次添加 &#8220;?&#8221;，以后添加 &#8220;,?&#8221;
3.  在循环外先添加第一个，进入循环后添加其他的

还有别的类似的办法，但我相信都差不太多。

你可能已经开始要反对了说这不是典型的用 `StringUtils.join` 的场景吗？

对！我也觉得是，但问题是你怎么构造一个由一堆问号组成的列表/迭代器/数组？ 还是得用循环构造出一个列表，然后 append 吧？可以试着写出来代码，实际不 见得比上面那个短，或者更可读。

其实想用这个方法的话，最好是能有一个 `repeat` 方法用于生成指定长度的， 内容都是某个元素的列表/数组/迭代器。比如在 Clojure 里，上面的代码可以 写成这个样子（省略了与问题无关的部分）：

<pre lang="clojure">(let [category-list "aaa,bbb,ccc,ddd"
      categories (string/split category-list #",")]
  (str "AND d.CATEGORY in ("
       (string/join "," (repeat (count categories) "?"))
       ")"))
</pre>

可以看到一个简单的 join + repeat 就可以解决问题了。可惜的是我这是在写 Java 代码。我找了一圈，没有发现项目中所引用的库有哪个包含此方法（主要参 考的是 guava 库）。

正当我打算自己写一个 repeat 出来的时候，我突然想到了之前在 guava 的文 档上看到过的一段话（原文在此：[Functional idioms in Guava, explained][2]）：

> As of Java 7, functional programming in Java can only be approximated through awkward and verbose use of anonymous classes. This is expected to change in Java 8, but Guava is currently aimed at users of Java 5 and above.
> 
> Excessive use of Guava&#8217;s functional programming idioms can lead to verbose, confusing, unreadable, and inefficient code. These are by far the most easily (and most commonly) abused parts of Guava, and when you go to preposterous lengths to make your code &#8220;a one-liner,&#8221; the Guava team weeps.

大意就是说在 Java 里使用函数式风格会导致十分冗长、让人迷惑、不可读的代 码，因为 Java 里没有 First-class Function 。当初看到这段话，我深受震撼， 因为光看 guava 的 API 文档，以为它是鼓励在 Java 里使用函数式风格的。原 文中列举了两段代码，一个是 Java 中的“函数式”风格：

<pre lang="java">Multiset lengths = HashMultiset.create(
  FluentIterable.from(strings)
    .filter(new Predicate() {
       public boolean apply(String string) {
         return CharMatcher.JAVA_UPPER_CASE.matchesAllOf(string);
       }
     })
    .transform(new Function() {
       public Integer apply(String string) {
         return string.length();
       }
     }));
</pre>

另一个是正常的“命令式”风格：

<pre lang="java">Multiset lengths = HashMultiset.create();
for (String string : strings) {
  if (CharMatcher.JAVA_UPPER_CASE.matchesAllOf(string)) {
    lengths.add(string.length());
  }
}
</pre>

对比就能发现，后者其实更加可读。这让我挺震撼的，因为一般认为命令式风格 的可读性比起函数式要差一些。仔细想一想，其实原因很简单：Java 不是一个支 持函数式编程的语言，无论怎么去模拟（用丑陋的匿名内部类），终究不可能比 得上一个原生的函数式编程语言。

在 Java 里用这种模拟出来的函数式风格还有一个更加严重的问题：性能。 Java 不是 lazy 的，在 Java 里使用 map/filter 等函数，会导致不断的列表创建和元素复制动作，对于数据量大一些的场景是不小的开销。

经过这样一番思索，我最终决定不去写那个花哨的 `repeat` 了，老老实实按照前面那个样子写出来那段代码，因为我现在觉得，那种愚蠢、丑陋、啰嗦的写法， 就是 Idiomatic Java。不排除有一些更好的写法，但我相信不会是用一个山寨 的 `repeat` 和 `join` 去做。

当然，如果你不喜欢这种 Idiomatic Java，你总是有一些 [更酷的，同样是跑在JVM 上的语言][3]可以用嘛 <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_smile.gif' alt=':-)' class='wp-smiley' />

 [1]: http://python.net/~goodger/projects/pycon/2007/idiomatic/handout.html
 [2]: https://code.google.com/p/guava-libraries/wiki/FunctionalExplained
 [3]: http://clojure.org/