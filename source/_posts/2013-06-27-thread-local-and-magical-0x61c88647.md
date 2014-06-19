---
title: ThreadLocal 和神奇的 0x61c88647
author: Jerry
comments: true
layout: post
permalink: /2013/06/thread-local-and-magical-0x61c88647/
categories:
  - Java
  - 技术
tags:
  - Concurrency
  - Java
  - ThreadLocal
keywords: Java, 并发, 线程本地, ThreadLocal, Concurrency
---
我之前提到说想把找工作面试过程中被问到的一些问题进行一下整理和记录，借刚刚完成一个小feature 的空闲时间开始第一篇吧。这一次的主题是 ThreadLocal，以及在探索过程中发现的一个很神奇的魔数 0x61c88647。

当时面试的过程中，我针对面试官的另一个问题提到了一个基于 ThreadLocal 的解决方案，然后面试官就抓住这一点进行了追问，让我画 ThreadLocal 的内部结构图并解释原理。自诩 Java 基础还可以的我，硬是在这翻船了。我当时给出的答案是，ThreadLocal 内部有一个以 Thread Id 为 key 的 map，不同的线程通过自己的 Thread Id 作为索引来查找对应的值。面试官接着追问说这样是不是会带来不必要的 contention，性能不好，我一下就语塞了。还好后来回忆起来 Java 的 Thread 类里有一个字段叫 threadLocals，隐隐觉得不对劲，修改了我的说法，换成了一个比较接近真相的答案。

回来以后赶紧翻开源代码验证一下，以下是我的发现。

<!--more-->

## ThreadLocal 的原理

ThreadLocal，意味着是线程本地的，如果实现层面引入一个让线程共享访问的 Map，岂不很蠢。事实上，Java 1.4 之前的 ThreadLocal 实现好像就是有问题的。目前版本的<span style="line-height: 1.714285714; font-size: 1rem;"> ThreadLocal 的实现则不会引入任何线程间的竞争（唯一共享的 Id 生成器也是个 AtomicInteger），其机制是这样的：</span>

*   <span style="line-height: 1.714285714; font-size: 1rem;">每个线程有一个自己的 ThreadLocalMap 对象</span>
*   <span style="line-height: 1.714285714; font-size: 1rem;">每一个 ThreadLocal 对象有一个创建时生成唯一的 Id</span>
*   <span style="line-height: 1.714285714; font-size: 1rem;">访问一个 ThreadLocal 变量的值，就是用这个唯一 Id 去<strong>本线程</strong>的 ThreadLocalMap 中查找对应的值</span>

## ThreadLocalMap

这是一个定义在 ThreadLocal 类中的访问权限为 package private 的内部类，因此可以在 Thread 类中引用（都是 java.lang 包下的）。它是一个专门为线程本地变量设计的一个特殊的 hash map。它采用的是开地址法而不是链表来解决冲突。比较有趣的地方在于，它要求内部的数组尺寸一定是 2 的 N 次方：

<div class="highlight">
  <pre><span class="cm">/**</span>
<span class="cm"> * The initial capacity -- MUST be a power of two.</span>
<span class="cm"> */</span>
<span class="kd">private</span> <span class="kd">static</span> <span class="kd">final</span> <span class="kt">int</span> <span class="n">INITIAL_CAPACITY</span> <span class="o">=</span> <span class="mi">16</span><span class="o">;</span>

<span class="cm">/**</span>
<span class="cm"> * The table, resized as necessary.</span>
<span class="cm"> * table.length MUST always be a power of two.</span>
<span class="cm"> */</span>
<span class="kd">private</span> <span class="n">Entry</span><span class="o">[]</span> <span class="n">table</span><span class="o">;</span>
</pre>
</div>

Entry 是 WeakReference 的子类，这样不再被使用的 ThreadLocal 可以被检查出来并清除掉。

比较关键的地方是：计算数组下标的方法<del>不是 hash table 中常见的对 size 取模，而</del><span style="color: #0000ff;">（又想当然了，HashMap 里也一样是与运算）</span>是和 size &#8211; 1 进行与运算。

<div class="highlight">
  <pre><span class="kd">private</span> <span class="n">Entry</span> <span class="nf">getEntry</span><span class="o">(</span><span class="n">ThreadLocal</span> <span class="n">key</span><span class="o">)</span> <span class="o">{</span>
    <span class="kt">int</span> <span class="n">i</span> <span class="o">=</span> <span class="n">key</span><span class="o">.</span><span class="na">threadLocalHashCode</span> <span class="o">&</span> <span class="o">(</span><span class="n">table</span><span class="o">.</span><span class="na">length</span> <span class="o">-</span> <span class="mi">1</span><span class="o">);</span>
    <span class="n">Entry</span> <span class="n">e</span> <span class="o">=</span> <span class="n">table</span><span class="o">[</span><span class="n">i</span><span class="o">];</span>
    <span class="k">if</span> <span class="o">(</span><span class="n">e</span> <span class="o">!=</span> <span class="kc">null</span> <span class="o">&&</span> <span class="n">e</span><span class="o">.</span><span class="na">get</span><span class="o">()</span> <span class="o">==</span> <span class="n">key</span><span class="o">)</span>
        <span class="k">return</span> <span class="n">e</span><span class="o">;</span>
    <span class="k">else</span>
        <span class="k">return</span> <span class="nf">getEntryAfterMiss</span><span class="o">(</span><span class="n">key</span><span class="o">,</span> <span class="n">i</span><span class="o">,</span> <span class="n">e</span><span class="o">);</span>
<span class="o">}</span>
</pre>
</div>

联想到上面说的要求内部数组的尺寸为 2 的 N 次方，可以得出的结论是，这里取的下标就是 hash code 的低 N &#8211; 1 位（因为 2^N &#8211; 1 的二进制表示就是 N &#8211; 1 个 1）。为什么要这么干？这就要看 hash code 是怎么来的了。

## 神奇的 0x61c88647

继续看 ThreadLocal 的源码，找到生成 hash code 的地方：

<div class="highlight">
  <pre><span class="cm">/**</span>
<span class="cm"> * The difference between successively generated hash codes - turns</span>
<span class="cm"> * implicit sequential thread-local IDs into near-optimally spread</span>
<span class="cm"> * multiplicative hash values for power-of-two-sized tables.</span>
<span class="cm"> */</span>
<span class="kd">private</span> <span class="kd">static</span> <span class="kd">final</span> <span class="kt">int</span> <span class="n">HASH_INCREMENT</span> <span class="o">=</span> <span class="mh">0x61c88647</span><span class="o">;</span>

<span class="cm">/**</span>
<span class="cm"> * Returns the next hash code.</span>
<span class="cm"> */</span>
<span class="kd">private</span> <span class="kd">static</span> <span class="kt">int</span> <span class="nf">nextHashCode</span><span class="o">()</span> <span class="o">{</span>
    <span class="k">return</span> <span class="n">nextHashCode</span><span class="o">.</span><span class="na">getAndAdd</span><span class="o">(</span><span class="n">HASH_INCREMENT</span><span class="o">);</span>
<span class="o">}</span>
</pre>
</div>

原来 hash code 从 0 开始不断累加 0x61c88647 生成的。其目的在 HASH_INCREMENT 的注释中已经提到了：为了让 hash code 能更好地分布在尺寸为 2 的 N 次方的数组里。我们不妨来验证一下。这里用 Clojure 来演示一串这样的 hash code 所生成的数组下标：

<div class="highlight">
  <pre><span class="p">(</span><span class="kd">defn </span><span class="nv">magic-hash</span> <span class="p">[</span><span class="nv">n</span><span class="p">]</span>
  <span class="p">(</span><span class="nf">-&gt;&gt;</span> <span class="p">(</span><span class="nb">iterate </span><span class="o">#</span><span class="p">(</span><span class="nb">+ </span><span class="nv">%</span> <span class="mi"></span><span class="nv">x61c88647</span><span class="p">)</span> <span class="mi"></span><span class="p">)</span>
       <span class="p">(</span><span class="nb">map </span><span class="o">#</span><span class="p">(</span><span class="nb">bit-and </span><span class="p">(</span><span class="nb">dec </span><span class="nv">n</span><span class="p">)</span> <span class="nv">%</span><span class="p">))))</span>

<span class="p">(</span><span class="nb">take </span><span class="mi">16</span> <span class="p">(</span><span class="nf">magic-hash</span> <span class="mi">16</span><span class="p">))</span>
<span class="c1">;(0 7 14 5 12 3 10 1 8 15 6 13 4 11 2 9)</span>
<span class="p">(</span><span class="nb">take </span><span class="mi">32</span> <span class="p">(</span><span class="nf">magic-hash</span> <span class="mi">32</span><span class="p">))</span>
<span class="c1">;(0 7 14 21 28 3 10 17 24 31 6 13 20 27 2 9 16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25)</span>
<span class="p">(</span><span class="nb">take </span><span class="mi">64</span> <span class="p">(</span><span class="nf">magic-hash</span> <span class="mi">64</span><span class="p">))</span>
<span class="c1">;(0 7 14 21 28 35 42 49 56 63 6 13 20 27 34 41 48 55 62 5 12 19 26 33 40 47 54 61 4 11 18 25 32 39 46 53 60 3 10 17 24 31 38 45 52 59 2 9 16 23 30 37 44 51 58 1 8 15 22 29 36 43 50 57)</span>
</pre>
</div>

可以看到分布真的十分均匀，而且目测没有任何冲突（要验证一下），相当神奇。

你可能要问，为什么不直接用自增的方式，直接按 0,1,2 这样分配 id，并对 size 取模后生成下标。我的理解是，随着不用的 ThreadLocal 变量被回收掉，这种自增的方式的性能会越来越差，因为临近的 slot 为空的可能性很小。而 ThreadLocal 实际所采用的方式，其下标是在跳跃分布，这样即使出现冲突，在临近找到空 slot 的可能性更大一些，性能也会更好。

为什么 0x61c88647 这个数会有这么神奇的属性？我 Google 了一下，在<a href="http://www.javaspecialists.eu/archive/Issue164.html" target="_blank">这篇文章</a>中找到了一些说法：

> This number represents the golden ratio (sqrt(5)-1) times two to the power of 31. The result is then a golden number, either 2654435769 or -1640531527.

其中 1640531527 就是 0x61c88647 的十进制表示。文章后面的结论和上面说的是一个意思：

> We established thus that the HASH\_INCREMENT has something to do with fibonacci hashing, using the golden ratio. If we look carefully at the way that hashing is done in the ThreadLocalMap, we see why this is necessary. The standard java.util.HashMap uses linked lists to resolve clashes. The ThreadLocalMap simply looks for the next available space and inserts the element there. It finds the first space by bit masking, thus only the lower few bits are significant. If the first space is full, it simply puts the element in the next available space. The HASH\_INCREMENT spaces the keys out in the sparce hash table, so that the possibility of finding a value next to ours is reduced.

看来这个数与黄金比例、Fibonacci 数有些神秘的关系……作为一个数学弱爆了的屌丝码农，我无法做出更好的解释。有数学比较好的读者看到这篇文章的话，求科普。
