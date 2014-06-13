---
title: Python Unicode字符串格式化中的一个陷阱
author: Jerry
comments: true
layout: post
permalink: /2012/03/python-unicode-format-pitfall/
categories:
  - Python
  - 技术
tags:
  - Python
  - Unicode
---
今天帮同事研究一个莫名其妙的UnicodeDecodeError时发现了Python字符串格式化中的一个小陷阱，在此记录一下。原本的代码过于复杂，有太多与问题无关的东西，所以我在ipython里简单试验复现了问题，过程如下：

<pre lang="python">In [4]: a = '你好世界'

In [5]: print 'Say this: %s' % a
Say this: 你好世界

In [6]: print 'Say this: %s and say that: %s' % (a, 'hello world')
Say this: 你好世界 and say that: hello world

In [7]: print 'Say this: %s and say that: %s' % (a, u'hello world')
---------------------------------------------------------------------------
UnicodeDecodeError                        Traceback (most recent call last)

/home/jerry/ in ()

UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 10: ordinal not in range(128)

In [8]: a
Out[8]: '\xe4\xbd\xa0\xe5\xa5\xbd\xe4\xb8\x96\xe7\x95\x8c'
</pre>

看到In [7]之后的那个很怪异的UnicodeDecodeError没？它和上一句的唯一区别是’hello world‘变成了一个unicode对象而非str对象。但问题是，&#8217;hello world&#8217;只是单纯的英文字符串，不包含任何ASCII之外的字符，怎么会无法decode呢？再仔细看看异常附带的message，里面提到了0xe4，这个显然不是’hello world‘里面的，所以只能怀疑那句中文了，In [8]把它的字节序列打印了出来，果然就是它，第一个就是0xe4。

看来在字符串格式化的时候Python试图将a decode成unicode对象，并且decode时用的还是默认的ASCII编码而非实际的UTF-8编码。那这又是怎么回事呢？？下面继续我们的试验：

<pre lang="python">In [9]: 'Say this: %s' % 'hello'
Out[9]: 'Say this: hello'

In [10]: 'Say this: %s' % u'hello'
Out[10]: u'Say this: hello'
</pre>

仔细看，In [9]中的&#8217;hello&#8217;是普通的字符串，结果也是字符串（str对象），而In [10]中的&#8217;hello&#8217;变成了unicode对象，格式化的结果也变成unicode了（注意结果开头的那个u）。

所以真相是这样的：Python在格式化字符串的时候有一些隐藏着的小动作：如果%s对应的参数里有unicode，那么最终的结果也是unicode。在这种情况下模版字符串以及所有的%s参数中的str都会被decode成unicode，然而这个decode是隐式的，用户无法指定其使用的charset，Python只能用默认的ASCII。如果正好里面有非ASCII编码的字符串，就完蛋了……

看看Python文档怎么说的：

> If format is a Unicode object, or if any of the objects being converted using the %s conversion are Unicode objects, the result will also be a Unicode object.

如果代码里混合着str和unicode，这种问题很容易出现。在同事的代码里，中文字符串是用户输入的，经过了正确的编码处理，是以UTF-8编码的str对象；但那个惹事的unicode对象，虽然其内容都是ASCII码，但其来源是sqlite3数据库查询的结果，而<span style="color: #0000ff;">sqlite的API返回的字符串都是unicode对象</span>，所以导致了这么怪异的结果。

Python 2的str和unicode真的挺坑爹，已经被它们害过好几次了。Python 3在这方面很有提升，期待它的全面普及！