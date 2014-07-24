---
layout: post
title: 小心 Eclipse Java Compiler 编译出来的 Class
date: 2014-07-24 15:22:12 +0800
comments: true
categories: Java
tags: Java, Eclipse, ECJ
keywords: Java, Eclipse, ECJ
description: Eclipse Java Compiler 在源文件无法编译的情况仍会生成 class 文件，如果直接使用这样的 class 可能会带来很多问题。
---

本作坊里，不少项目的升级方式都是直接把 IDE 编译出来的 Class 文件打包放
到服务器上。虽然这种方式实在是让我无力吐槽，并且我也尝试推动了很多项目
往 Gradle 的迁移以实现构建、部署的自动化和规范化，但还有一些历史项目依
然用这种方式部署。在项目无法编译成功的情况下，这种做法会带来很严重的问
题。

Eclipse 的 Java 编译器会在 Java 源文件有错误的情况下仍旧生成 Class。如
果开发者未发现这些错误，并且使用了它生成的这些 Class，那么相关代码会在
运行时直接抛出一个 `java.lang.Error` 。

<!--more-->

可以尝试新建一个 Java 工程，写一段完全不合法的代码，编译出来查看 class
文件来验证。比如下面的例子：


```java
package com.trafree;

public class Foobar {

    public static void main(String[] args) {
        // 缺少分号
        System.out.println("Foobar")
    }
}
```


Eclipse 为其生成的 Class 文件中的字节码如下：


```sh
[~/dev/workspace/amadeus/bin]$ javap -l -c -constants com.trafree.Foobar                                               
Compiled from "Foobar.java"
public class com.trafree.Foobar {
  public com.trafree.Foobar();
    Code:
       0: aload_0       
       1: invokespecial #8        // Method java/lang/Object."<init>":()V
       4: return        
    LineNumberTable:
      line 4: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0       5     0  this   Lcom/trafree/Foobar;

  public static void main(java.lang.String[]);
    Code:
       0: new           #16            // class java/lang/Error
       3: dup           
       4: ldc           #18           _/ String Unresolved compilation problem: \n\tSyntax error, insert \";\" to complete BlockStatements\n
       6: invokespecial #20           /_ Method java/lang/Error."<init>":(Ljava/lang/String;)V
       9: athrow        
    LineNumberTable:
      line 8: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
}
```


可以看到，生成类中的所有方法的行为都是抛出一个 `java.lang.Error`，包
含的消息正是编译错误信息：


```java
Unresolved compilation problem:
    Syntax error, insert ";" to complete BlockStatements
```

如果是缺少依赖的类，比如：


```java
package com.trafree;

// NonExistent 是个不存在的类
import com.trafree.NonExistent;

public class ServletUtil {

    public void doSomething() {
        NonExistent foo = new NonExistent();
    }
}
```


生成的字节码：


```sh
[~/dev/workspace/amadeus/bin]$ javap -l -c -constants com.trafree.ServletUtil                                          
Compiled from "ServletUtil.java"
public class com.trafree.ServletUtil {
  public com.trafree.ServletUtil();
    Code:
       0: new           #8                  // class java/lang/Error
       3: dup           
       4: ldc           #10                 _/ String Unresolved compilation problems: \n\tThe import com.trafree.NonExistent cannot be resolved\n\tNonExistent cannot be resolved to a type\n\tNonExistent cannot be resolved to a type\n
       6: invokespecial #12                 /_ Method java/lang/Error."<init>":(Ljava/lang/String;)V
       9: athrow        
    LineNumberTable:
      line 3: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      10     0  this   Lcom/trafree/ServletUtil;

  public void doSomething();
    Code:
       0: new           #8                  // class java/lang/Error
       3: dup           
       4: ldc           #20                 _/ String Unresolved compilation problems: \n\tNonExistent cannot be resolved to a type\n\tNonExistent cannot be resolved to a type\n
       6: invokespecial #12                 /_ Method java/lang/Error."<init>":(Ljava/lang/String;)V
       9: athrow        
    LineNumberTable:
      line 8: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      10     0  this   Lcom/trafree/ServletUtil;
}

```


可以看到还是一样抛出 `java.lang.Error`，只不过错误信息变成了：


```java
Unresolved compilation problems:
    The import com.trafree.NonExistent cannot be resolved
    NonExistent cannot be resolved to a type
```

猜测 Eclipse Java Compiler 的此行为是为了代码补全而设计的，这样即使编
译会出错的类也能在 Eclipse 被正常补全。但如果想直接使用 Eclipse 编译出
来的 Class，则要十分小心，要确定 Eclipse 正常把项目编译通过了，再使用
其生成的 Class。

P.S. 我知道我们的做法槽点太多，但也只能逐渐推动改进了。好在目前我们大部
分内部项目都已经迁移成 Gradle 项目了，我们也编写了一些脚本用于自动化构
建和部署，能解决本文提到的这些问题了。

P.P.S. Eclipse 的这种行为我感觉很不靠谱，我认为代码在无法通过编译的时候
是不应该生成目标 Class 文件的。IntelliJ IDEA 的做法我就很喜欢，编译不通
过就无法运行，也无法生成 artifact。
