---
title: 最近在用Clojure做的事情
author: Jerry
layout: post
permalink: /2012/09/what-i-did-with-clojure-recently/
categories:
  - Clojure
  - 技术
tags:
  - Clojure
  - TR-069
  - 文本处理
---
有一段时间没有更新博客了，不是因为有多忙，而是最近做的事情多比较琐碎， 没有多少能单独成篇的内容。所以今天把最近的思考和实践中我觉得有意思或者 有价值的部分拿出来说一说，算是个记录和整理。

<div id="outline-container-1" class="outline-3">
  <h3 id="sec-1">
    第一个Clojure DSL
  </h3>
  
  <div id="text-1" class="outline-text-3">
    <p>
      最近的业余时间很多都贡献给了<a href="https://github.com/moonranger/clj.tr069">clj.tr069</a>这个项目——一个学习Clojure用的习作。 在里面有我写的第一个DSL，用来描述TR-069协议里的数据类型。这个DSL看起来 是这样的：
    </p>
    
    <pre lang="clojure">; Inform
(deftr069type
  ^:top-level
  Inform
  (device-id      :child       <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_biggrin.gif' alt=':D' class='wp-smiley' /> eviceId)
  (events         :child-array :Event         :EventStruct)
  (parameter-list :child-array <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_razz.gif' alt=':P' class='wp-smiley' /> arameterList <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_razz.gif' alt=':P' class='wp-smiley' /> arameterValueStruct)
  (retry-count    :int         :RetryCount)
  (current-time   :dateTime    :CurrentTime)
  (max-envelopes  :int         :MaxEnvelopes))
</pre>
    
    <p>
      这样一段代码描述了TR-069的数据类型Inform，其中列出了每个字段，它的类型 以及相应的tag名称。deftr069type是个宏，展开后其实是一个defrecord和一个 multimethod的实现部分。其中defrecord定义了Clojure的数据类型并实现了一个 用来产生SOAP XML的protocol，而multimethod部分则实现了从SOAP XML提取出字 段并创建对象。
    </p>
    
    <p>
      最终的效果我还是很满意的，很大一部分无用的重复代码通过宏给消除掉，隐藏 在声明式的类型定义DSL之后了。
    </p>
  </div>
</div>

<div id="outline-container-2" class="outline-3">
  <h3 id="sec-2">
    大数据量文本处理
  </h3>
  
  <div id="text-2" class="outline-text-3">
    <p>
      最近工作的内容之一是要对大批量的文本数据文件做预处理。这些文本都是很单 纯的类似CSV的数据记录文件，一行一条记录。我要做的是对这些数据做过滤和 简单的聚合，之后存到中间文件后，通过SQL Loader导入数据库。数据本身并不 复杂，但是量不小，会按10分钟500M左右的量传到我们的系统。
    </p>
    
    <p>
      具体的需求还没下来，我只能先做一些准备工作。正好最近在学Clojure，遂决 定写个HTTP日志分析的小demo玩一下。半个下午的时间给弄出来一个东西，用来 分析nginx的日志，计算出每个IP的访问次数并按顺序排列。核心的代码如下：
    </p>
    
    <pre lang="clojure">(defn parse-ip-freq
  "Parse a nginx log to get IP frequency data"
  [file black-list]
  (let [lines (buffered-line-seq file)
        black-list (set black-list)]
    (->> lines
         (map (comp first extract-log-fields))
         (remove #(or (nil? %) (black-list %)))
         (aggregation identity count)
         (sort-by (comp - second)))))

(defn parse-ip-freq-pmap
  "Parse a nginx log to get IP frequency data"
  [file black-list chunk-size]
  (let [lines (buffered-line-seq file)
        chunks (partition-all chunk-size lines)
        black-list (set black-list)]
    (->> chunks
         (pmap (fn [chunk]
                 (->> chunk
                      (map (comp first extract-log-fields))
                      (remove #(or (nil? %) (black-list %)))
                      (aggregation identity count))))
         (reduce (partial merge-with +))
         (sort-by (comp - second)))))
</pre>
    
    <p>
      第一个是单线程的版本，第二个是基于pmap实现的多线程版本。在我的i3处理器 上测试，后者的速度是前者的两倍，和预期的一样：
    </p>
    
    <pre lang="clojure">user> (pprint
        (time (take 3 (parse-ip-freq
                        "/home/jerry/documents/blog.access.log"
                        ["216.24.201.214" "60.30.32.20"]))))
"Elapsed time: 7694.650284 msecs"
nil
(["218.240.38.162" 6338]
 ["204.236.246.95" 4656]
 ["174.129.46.176" 4436])
user> (pprint
        (time (take 3 (parse-ip-freq-pmap
                        "/home/jerry/documents/blog.access.log"
                        ["216.24.201.214" "60.30.32.20"]
                        1000))))
"Elapsed time: 3857.587226 msecs"
nil
(["218.240.38.162" 6338]
 ["204.236.246.95" 4656]
 ["174.129.46.176" 4436])
</pre>
    
    <p>
      可以看到多线程版本的代码并不比单线程的复杂多少，唯一多出来的就是两点：
    </p>
    
    <ul>
      <li>
        先对原始数据做了partition，分成指定大小的若干chunk再传给pmap处理
      </li>
      <li>
        pmap处理完的结果通过merge-with合并成到一起
      </li>
    </ul>
    
    <p>
      其实这就是很简单清晰的map/reduce程序：将数据划分成大小合适的chunk，在一 个线程池中做并行的map处理，做字段提取和聚合的操作，在主线程中对map的结 果作reduce操作，合并结果。一眼看过去，代码非常清晰易懂。
    </p>
    
    <p>
      不得不再次感叹Clojure的好用，用好Lazy Seq和map，reduce，partition等等 针对sequence的操作，可以很顺利地构造出各种数据处理程序。->, ->>这对宏 更是能让代码的可读性上升一个档次。Rich Hickey替程序员考虑到了太多。
    </p>
    
    <p>
      这个小任务，我决定用Clojure来完成了。
    </p>
  </div>
</div>

<div id="outline-container-3" class="outline-3">
  <h3 id="sec-3">
    Clojure的Sequence抽象
  </h3>
  
  <div id="text-3" class="outline-text-3">
    <p>
      随着对Clojure理解的深入，我越来越喜欢Clojure的Sequence抽象。我认为 Sequence本质上是对顺序产生数据的计算过程的一种抽象，而这种抽象在很多语 言里都存在：Python、Java等都有类似Sequence的Iterator的概念，但他们没有 Sequence那么强大。主要原因我觉得有以下几点：
    </p>
  </div>
  
  <div id="outline-container-3-1" class="outline-4">
    <h4 id="sec-3-1">
      Laziness
    </h4>
    
    <div id="text-3-1" class="outline-text-4">
      <p>
        Clojure的支持lazy seq，可以在真正需要的时候才计算，更妙的是，已经计算 过的部分会被缓存，并且在不需要使用的时候被GC掉。Clojure核心中的很多 sequence相关的函数返回的都是lazy seq，比如map，filter，partition等。使 用这些sequence，其实和自己手写一个for循环，在里面做变换、过滤是一样的， 性能上也没有什么不同，因为map，filter返回的是lazy seq，在创建那一刻其 实什么都还没做。
      </p>
    </div>
  </div>
  
  <div id="outline-container-3-2" class="outline-4">
    <h4 id="sec-3-2">
      Stateless/Thread Safety
    </h4>
    
    <div id="text-3-2" class="outline-text-4">
      <p>
        Java和Python中的iterator是有状态的，它们要记录当前迭代的内部状态，因而 它们也无法做到线程安全。Clojure的seq是线程安全的，它没有内部状态，对于 lazy seq，Clojure能确保运算过程只会运行一次。因此，Clojure的seq十分适 合用来当作State world和Stateless world之间的桥梁。
      </p>
    </div>
  </div>
</div>

<div id="outline-container-4" class="outline-3">
  <h3 id="sec-4">
    关于Memory Mapped Files
  </h3>
  
  <div id="text-4" class="outline-text-3">
    <p>
      写上面的程序IO处理部分时，我想当然地认为用Memory Mapped Files肯定会更 快，因此自己手工基于Java的MappedByteBuffer写了个逐行read文本的 LineReader。结果对一个400多M的nginx日志测试的结果还不如直接用 BufferedReader加FileReader来得快……
    </p>
    
    <p>
      怀疑是自己新写的部分有性能问题，我又写了个小程序测试MappedByteBuffer和 BufferedInputStream的读取性能，结果发现虽然的确MappedByteBuffer更快一 些，但差别没有想象的那么大。
    </p>
    
    <p>
      想了想，像我的这种需求，只需要对文件顺序读取一遍就好了，这种场景 Memory Mapped Files用处应该不大。
    </p>
  </div>
</div>