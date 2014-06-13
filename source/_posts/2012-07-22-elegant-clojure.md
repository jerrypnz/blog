---
title: 优雅的Clojure——读clj-http源码有感
author: Jerry
layout: post
permalink: /2012/07/elegant-clojure/
categories:
  - Clojure
tags:
  - Clojure
  - Destructuring
  - FP
---
今天下午陪老婆加班，百无聊赖，找了台电脑上网看Clojure相关的文章。一个链接点到另一个链接，偶然发现了一个叫clj-http的库，是对Apache HttpComponents库的包装，于是在IE6上展现出来的惨不忍睹的Github页面上一点点读完了clj-http的那几百行代码。这段代码很简单，即使是我这个Clojure初学者也能毫不费力地读懂。看完之后不禁为Clojure的优雅和表达力所折服。有一段时间没读到让我觉得如此舒服的代码了，遂拿出来说说，记录下我的理解和感受。

## FP风格的优雅架构

从入口开始看，发现client/request这个函数是如此定义的

<pre lang="clojure">(defn wrap-request
  [request]
  (-> request
    wrap-redirects
    wrap-exceptions
    wrap-decompression
    wrap-input-coercion
    wrap-output-coercion
    wrap-query-params
    wrap-basic-auth
    wrap-user-info
    wrap-accept
    wrap-accept-encoding
    wrap-content-type
    wrap-method
    wrap-url))

(def request
  (wrap-request #'core/request))
</pre>

request其实是core/request这个函数经过n个wrap-xxx的函数包装之后的结果。wrap-request中的->是个宏，展开后的代码其实是

<pre lang="clojure">(wrap-url (wrap-method (wrap-content-type ... （wrap-redirects request)...)))
</pre>

 每一个wrap-xxx都是一个高阶函数，它们将传入的函数做包装，添加一些功能。包装函数之间是正交的。这种设计，类似Python里的装饰器，但得益于->这个宏，用户自己做扩展的时候看起来会更漂亮一些。比如我要添加cookie和XML RPC支持，那么我可以在自己的namespace里写一个wrap-cookie-support和wrap-xml-rpc函数，然后用下面的代码来强化request函数：

<pre lang="clojure">(defn wrap-cookie-support [fn] ...)
(defn wrap-xml-rpc [fn] ...)

(def my-request
  (-> #'client/request
      wrap-cookie-support
      wrap-xml-rpc))
</pre>

 这样我的my-request函数就具有了默认的request的所有功能，加上我自己写的cookie和XML RPC的支持。

等等，你也许会说，这算什么啊？我们的Java也支持这样的功能好不好，这不就是大名鼎鼎的Decorator Pattern——装饰模式吗？这有什么稀奇的？

是，但容我继续展示一些代码并作出解释。

## 数据结构的语法糖

知道我对Java十分讨厌的一点是啥吗？不支持简单的{&#8220;key&#8221; : &#8220;value&#8221;}风格的Map字面量，不支持用[]风格的下标来访问Map中的元素，总之，Map就是简单的class，没有一点语法糖，使用起来全都是难看的方法调用。很多时候因为这个限制，我们不得不用一个个除了几个属性和一堆getter/setter之外什么都没有的Java Bean来表示一些简单的数据结构。在Clojure里，Map可以简单地用{key1 value1 key2 value2}的形式来表达，类似的，List，Vector，Set这些核心数据结构都支持类似的表达法，再加上Destructuring这个功能（接下来会介绍），让核心数据结构的表达力强大了很多。在Clojure里用嵌套的Vector，Map等来表示复杂的数据结构是十分方便的。

到了要回答上面那个问题的时候了。为什么在Clojure里，这种“装饰模式”用起来比Java里要爽。答案就是参数和返回值。

clj-http里的request方法的参数与返回值都是map，里面用:content-type, :method, :url, :body等关键字来表示所需要的不同参数，返回值里也用:status, :body, :headers等来表达HTTP响应的不同部分。用map的一大好处是，map并没有限定你可以往里面放什么内容，这意味着当我需要添加功能的时候，完全可以扩充参数和返回值，在已有的map里用新的关键字来表达扩充的内容。比如，HTTP Basic认证的wrapper，我们看它的代码：

<pre lang="clojure">(defn wrap-basic-auth [client]
  (fn [req]
    (if-let [[user password] (:basic-auth req)]
      (client (-> req (dissoc :basic-auth)
                      (assoc-in [:headers "Authorization"]
                        (basic-auth-value user password))))
      (client req))))
</pre>

它使用了请求map里的:basic-auth关键字，用它来存放认证用户名和密码，而这个关键字是核心的request的方法所不关心，也不会用到的。任何一个扩展函数，都可以用这种方式来扩充请求map或者响应map，而不同担心受到限制，这让扩展能力增强了很多。

回过头来想象一下Java的装饰器，要实现类似的效果，我能想到的有两个方法：1. 使用Map；2. 子类化参数或者返回值的类型。对于第一个方法，Java那可怜的表达力会让代码十分蛋疼，恐怕没多少人愿意那样写。对于第二个方法，它可能完全不工作，因为它会让不同装饰器的组合产生问题，而且会产生很多额外的class以及类型转换代码。

我认为Clojure，以及大部分函数式语言在语言层面对基本数据结构的强力支持，才让装饰模式风格变得好用。

## 极大简化代码的参数Destructuring

用map作为参数，在别的语言，包括Python里，可能的一个问题是，很难用简洁的语法来获取其中层层包装的数据。但在Clojure里，有一个叫Destructuring的功能，消除了这个问题。Destructuring有点类似Python里的元组解包，即x, y, z = (&#8216;xvalue&#8217;, &#8216;yvalue&#8217;, &#8216;zvalue&#8217;)这种写法。但它能用在所有的基本数据结构上，包括list， vector，map，set，并且支持嵌套！用Destructuring可以直接在binding form里将一个map类型的参数中某个成员绑定到变量上。

比如下面这个例子：

<pre lang="clojure">(defn wrap-content-type [client]
  (fn [{:keys [content-type] :as req}]
    (if content-type
      (client (-> req (assoc :content-type
                        (content-type-value content-type))))
      (client req))))
</pre>

 里面的匿名函数的参数部分，就是用了destructuring来将map中:content-type的值绑定到同名的content-type变量中，并且将原始的map用:as关键字绑定到req变量上。在函数的body部分，我们就可以直接用conent-type变量了。

再比如核心的request方法，有很长一串destructuring：

<pre lang="clojure">(defn request
  [{:keys [request-method scheme server-name server-port uri query-string
           headers content-type character-encoding body]}]
  (let [http-client (DefaultHttpClient.)]
</pre>

 它将map中request-method, scheme, server-name等等关键字的值都绑定到对应的同名变量上了。

可以看到，Destructuring在保持了用map等数据结构作为参数的灵活性的同时，极大提升了代码的可读性，消除了数据结构操作的语法噪音。

## Clojure，下一个深入学习的语言

也许这些简单的例子，对于熟悉Lisp或者Clojure的人来说没什么，但对于我这个刚刚踏入Lisp大门的初学者来说，是有些震撼的。Clojure真的是一个很酷的语言，它是Lisp但又增加了很多现代特性，使代码简化了很多；它跑在JVM上，能无缝地和Java互操作。作为一个主要使用Java工作的人，我觉得Clojure太有价值了。我准备从现在开始深入学习这个语言并开始用它写一些实用的东西，享受这个语言带来的一切！