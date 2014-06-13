---
title: 小探 Java 泛型系统的不协变问题和类型推断
author: Jerry
layout: post
permalink: /2013/02/java-generics-invariant-and-inference/
categories:
  - Java
  - 技术
tags:
  - Generics
  - Java
  - Type System
---
这个问题起源于最近在生产代码中写下的这样一个方法：

<pre class="example">protected Set&lt;Class&lt;? extends Module&gt;&gt; getAppModuleClasses() {
    return Sets.&lt;Class&lt;? extends Module&gt;&gt; newHashSet(DAOModule.class,
                                                     ACSModule.class,
                                                     TR069Module.class,
                                                     STUNModule.class);
}
</pre>

那句显式的类型参数实在是太碍眼了，但是去掉它则会导致编译无法通过。在忍 受了这段代码一段时间后，我决定对这个问题好好研究一下。

下面以这段短小的代码来作为例子解释：

<pre class="example">static interface Plant {}
static class Grass implements Plant {}
static class Tree implements Plant {}
static class AppleTree extends Tree {}
static class BananaTree extends Tree {}

public static void main(String[] args) {
    List&lt;Class&lt;? extends Tree&gt;&gt; list1
         = Arrays.asList(AppleTree.class, BananaTree.class);
    List&lt;Class&lt;? extends Plant&gt;&gt; list2
         = Arrays.asList(AppleTree.class, BananaTree.class);
    List&lt;Class&lt;? extends Plant&gt;&gt; list3 
         = Arrays.asList(AppleTree.class,  BananaTree.class, Grass.class);
    List&lt;? extends Class&lt;? extends Plant&gt;&gt; list4
         = Arrays.asList(AppleTree.class,  BananaTree.class);
}
</pre>

上面的代码编译无法通过，读者可以猜测一下是哪一处有问题。

答案是 `list2` 处。但为什么会这样呢？这要分两部分来说明：

1.  泛型的“不协变（invariant）”问题
2.  Java 泛型方法调用的类型推断

<div id="outline-container-1" class="outline-3">
  <h3 id="sec-1">
    Java 泛型的“不协变”问题
  </h3>
  
  <div id="text-1" class="outline-text-3">
    <p>
      其实这个问题是所有接触到 Java 泛型的人很快就会遇到的，应该属于很基础的 内容。Java 数组是协变（covariant）的，而泛型系统在不用 wildcard type 的 情况下是不协变的（invariant）<sup><a class="footref" name="fnr-.1" href="#fn-.1"></a>1</sup>。比如可以把 <code>Integer[]</code> 赋值 <code>Number[]</code> ，但是不能把 <code>List&lt;Integer&gt;</code> 赋值给 <code>List&lt;Number&gt;</code> 。
    </p>
    
    <p>
      但是当出现嵌套的泛型类型加上 wildcard type 时，我们还是容易迷 惑<sup><a class="footref" name="fnr-.2" href="#fn-.2"></a>2</sup>。比如 <code>List&lt;Integer&gt;</code> 可以赋值给 <code>List&lt;? extends Number&gt;</code> ，那么 <code>Set&lt;List&lt;Integer&gt;&gt;</code> 是否可以赋值给 <code>Set&lt;List&lt;? extends Number&gt;&gt;</code> 呢？乍一看好像是可以的，但其实是不行的， 而我犯的就是这个错误。应该牢记，在不使用 wildcard type 的情况下泛型是不 协变的。虽然可以认为 <code>List&lt;Integer&gt;</code> 是 <code>List&lt;? extends Number&gt;</code> 的子类 型，但 <code>Set&lt;List&lt;Integer&gt;&gt;</code> 不是 <code>Set&lt;List&lt;? extends Number&gt;&gt;</code> 的子类 型。为了解决这个问题，我们还是要加上 wildcard，把 <code>Set&lt;List&lt;? extends Number&gt;&gt;</code> 改成 <code>Set&lt;? extends List&lt;? extends Number&gt;&gt;</code> 即可解决问题。
    </p>
    
    <p>
      说了这么多，其实就是一个简单的道理：想获得协变的效果，就要使用 wildcard 加 extends。
    </p>
    
    <p>
      回到前面的例子， <code>list2</code> 那里编译不通过的原因，看一下错误信息，结合上 面的解释应该就很明了：
    </p>
    
    <blockquote>
      <p>
        Type mismatch: cannot convert from <code>List&lt;Class&lt;? extends TypeInference.Tree&gt;&gt;</code> to <code>List&lt;Class&lt;? extends TypeInference.Plant&gt;&gt;</code>
      </p>
    </blockquote>
    
    <p>
      在类型声明处加上 <code>? extends</code> 就可以解决问题， <code>list4</code> 处加上以后编译立 刻能通过了。
    </p>
    
    <p>
      剩下的问题是，为啥 <code>list1</code> 和 <code>list3</code> 两处可以通过编译？
    </p>
  </div>
</div>

<div id="outline-container-2" class="outline-3">
  <h3 id="sec-2">
    方法调用的类型推断
  </h3>
  
  <div id="text-2" class="outline-text-3">
    <p>
      方法调用的类型推断是个十分复杂的过程，对其完整的规则我还没有一个深入的 理解，说实话试图阅读 Java Language Specification 相关部分对我来说都十分 困难，感觉好难懂，有兴趣的读者可以自行查看相关章节（<a href="http://docs.oracle.com/javase/specs/jls/se5.0/html/expressions.html#15.12.2.7">15.12.2.7 Inferring Type Arguments Based on Actual Arguments</a>）。
    </p>
    
    <p>
      不过对于上面那个简单的例子，我可以得出一个比较 naive 的结论：
    </p>
    
    <blockquote>
      <p>
        对于某个泛型方法 M 中包含的泛型参数 T1..Tn，Java 编译器会根据调用上下文<br /> （Calling Context），包括实际参数和返回值等，推断出尽可能“具体”的实际<br /> 类型。
      </p>
    </blockquote>
    
    <p>
      另外还有一条我还不太确定的结论：如果一个泛型参数同时出现在参数和返回值 中，则类型推断以参数为准，仅当不包含泛型参数的时候才会参考函数返回值。
    </p>
    
    <p>
      看一下上面的例子， <code>list1</code> ， <code>list2</code> ， <code>list3</code> 看起来差不多，为什么 只有 <code>list2</code> 处编译不通过？我们可以结合上面提到的规则看一下：
    </p>
    
    <ol>
      <li>
        根据 <code>list1</code> 处的两个参数 <code>AppleTree.class</code> 和 <code>BananaTree.clas</code> 可 以推断出来的最“具体”的类型是 <code>List&lt;Class&lt;? extends Tree&gt;&gt;</code> ，和前面 <code>list1</code> 的声明完全吻合，所以不受不协变的影响，是合法的。
      </li>
      <li>
        根据 <code>list3</code> 处的三个参数 <code>AppleTree.class</code> ， <code>BananaTree.class</code> 和 <code>Grass.class</code> 可以推断出来的最“具体”的类型是 <code>List&lt;Class&lt;? extends Plant&gt;&gt;</code> ，和 <code>list3</code> 的声明也完全吻合，同理也是合法的。
      </li>
      <li>
        <code>list2</code> 处推断出来的是 <code>List&lt;Class&lt;? extends Tree&gt;&gt;</code> ，和前面声明的 <code>List&lt;Class&lt;? extends Plant&gt;&gt;</code> 不兼容，所以编译报错。
      </li>
    </ol>
    
    <p>
      再回到最开始那个例子，其中的 <code>Module</code> 是 Google Guice 的一个接口，而下 面那句 <code>newHashSet</code> 调用推断出来的是 <code>Set&lt;Class&lt;? extends AbstractModule&gt;&gt;</code> ，所以会报错。只要相应地把方法返回值改成 <code>Set&lt;? extends Class&lt;? extends Module&gt;&gt;</code> 即可解决我最初的问题。
    </p>
    
    <div id="footnotes">
      <h2 class="footnotes">
        Footnotes:
      </h2>
      
      <div id="text-footnotes">
        <p class="footnote">
          <sup><a class="footnum" name="fn-.1" href="#fnr-.1"></a>1</sup> <a href="http://www.ibm.com/developerworks/cn/java/j-jtp01255.html">Java 理论 和实践: 了解泛型</a>
        </p>
        
        <p class="footnote">
          <sup><a class="footnum" name="fn-.2" href="#fnr-.2"></a>2</sup> <a href="http://stackoverflow.com/questions/3546745/multiple-wildcards-on-a-generic-methods-makes-java-compiler-and-me-very-confu">Multiple wildcards on a generic methods makes Java compiler (and me!) very confused</a>
        </p>
      </div>
    </div>
  </div>
</div>