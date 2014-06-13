---
title: 小心 Clojure 的 apply
author: Jerry
comments: true
layout: post
permalink: /2012/12/use-apply-with-caution/
categories:
  - Clojure
  - 技术
tags:
  - Clojure
  - FP
  - nippy
  - OOM
---
Clojure 里的 `apply` 是十分常用的一个函数，它可以方便地让我们将一个 seq作为参数列表传给任意函数。可是今天我遇到一个由不恰当地使用了 `apply` 导致的 OOM 问题，在这里和大家分享一下。

最近大胆在公司的项目中使用了 Clojure 来实现了一个大数据量的分析程序，前 几天上线跑了一阵，效果很好，正高兴呢，今天查日志发现系统在分析月度汇总 数据的时候 OOM 了！今天分析了半天原因，最终定位到了开源 Clojure 序列化 库 [nippy][1] 中的 [这句代码][2] ：

<pre class="example">(apply hash-map (coll-thaw! s))
</pre>

罪魁祸首就是这里的 `apply` ！可以看到这句代码的逻辑就是读取 stream 中的 collection 并将其作为参数传给 `hash-map` 函数用来创建 map，其中 `coll-thaw!` 返回的是一个包含 所有 key/value 的 lazy seq，而在我的场景 里，这样的 map 是十分庞大的，包含几百万的数据。

重点来了：通过分析 `apply` 的源代码，我发现对于变参函数， `apply` 会将 整个seq 全部 realize 之后放到一个 Object 数组里，再调用目标函数：

<pre class="example">static public Object applyToHelper(IFn ifn, ISeq arglist) {
        switch(RT.boundedLength(arglist, 20))
        {
        case 0:
            arglist = null;
            return ifn.invoke();
        case 1:
            return ifn.invoke(Util.ret1(arglist.first(),arglist = null));
        case 2:
            return ifn.invoke(arglist.first()
                    , Util.ret1((arglist = arglist.next()).first(),arglist = null)
            );
        case 3:
            return ifn.invoke(arglist.first()
                    , (arglist = arglist.next()).first()
                    , Util.ret1((arglist = arglist.next()).first(),arglist = null)
            );

                ...

        default:
            return ifn.invoke(arglist.first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , (arglist = arglist.next()).first()
                    , RT.seqToArray(Util.ret1(arglist.next(),arglist = null)));
        }
}

</pre>

就是最后一行的那个 `seqToArray` 将 arglist 的剩余部分全部放到一个 `Object[]` 的。

我的这个 map 极其庞大，本来将其放到内存里就要耗费相当的内存（几个G）， 经过 `apply` 这样一处理，在 nippy 反序列化期间，这个 map 的所有 key/values 相当于被复制成了两份保存在内存里，一份在构造中的 `PersistentHashMap` 里，一份在 `hash-map` 函数的参数列表里，不把 heap 给撑爆了才怪！

后来我将这里的 `apply` 替换成了 `into` ，在生产环境同样的 JVM 配置和输 入数据下，运行成功。

<pre class="example">(into {} (map vec (partition 2 (coll-thaw! s))))
</pre>

其实 nippy 的作者在其他几类数据结构的反序列化中里使用了 `into` ，不知何 故在 map 这要用 `apply` （list 那如果用 `into` 会导致顺序颠倒，暂时没想 到好办法解决）：

<pre class="example">id-list    (apply list (coll-thaw! s))
id-vector  (into  [] (coll-thaw! s))
id-set     (into #{} (coll-thaw! s))
id-map     (apply hash-map (coll-thaw! s))
id-coll    (doall (coll-thaw! s))
id-queue   (into  (PersistentQueue/EMPTY) (coll-thaw! s))
</pre>

我 fork 了[这个项目][3]，修正了这个问题并提了个[Pull Request][4]，看会不会被接 受。

追查问题的过程让我对 Clojure 函数、闭包的实现产生了极大兴趣，是时候打 开反编译工具和 Clojure 源代码看个究竟了。等分析得差不多了，再写文章分 享一下。

 [1]: https://github.com/ptaoussanis/nippy
 [2]: https://github.com/ptaoussanis/nippy/blob/master/src/taoensso/nippy.clj#L196
 [3]: https://github.com/moonranger/nippy
 [4]: https://github.com/ptaoussanis/nippy/pull/3