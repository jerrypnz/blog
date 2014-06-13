---
title: 又被 Python 的 Unicode 坑了
author: Jerry
comments: true
layout: post
permalink: /2014/02/python-2-unicode-print-pitfall/
categories:
  - Python
  - 技术
tags:
  - Python
  - Unicode
---

很久没更新博客了，不是没什么可写，而是因为工作生活的种种原因没有心情更
新。趁着今天有点时间，还是开个头继续写吧。

这次遇到的问题还是与 Python 的 Unicode 有关。请看下面的代码：

```bash
[~/tmp]$ cat test.py
#coding:utf8

foo = u'测试'

print foo
[~/tmp]$ python test.py
测试
[~/tmp]$ python test.py > /tmp/foobar.txt
Traceback (most recent call last):
  File "test.py", line 5, in <module>
    print foo
UnicodeEncodeError: 'ascii' codec can't encode \
  characters in position 0-1: ordinal not in range(128)
```

简简单单打印一个 Unicode 对象，直接输出不会报错，但重定向到文件就会挂掉，
这实在让人想“呵呵”。

一番 Google 之后在 [Stackoverflow][1] 上找到了答案。原来 Python 2 里的
print 会针对 unicode 参数进行自动 encode。如果输出的目标是一个终端，它
会使用终端的编码（可通过 `sys.stdout.encoding` 获得）；如果输出的目标是
管道（重定向或者使用管道符），则无法获取对应的编码，此时 Python 会使用
系统默认的编码。

可以 print 一下 `sys.stdout.encoding` 查看终端编码：

```bash
[~/tmp]$ python -c 'import sys; print sys.stdout.encoding'
UTF-8
[~/tmp]$ python -c 'import sys; print sys.stdout.encoding' | tee /tmp/foo.txt
None
```

果然在使用管道的情况下，终端编码是 `None` 。

解决办法也很简单，就是在 print 之前手工 encode 一下，比如在 Linux 环境
完全可以一律 encode 成 UTF-8。如果想做得更好一点，也可以根据
`sys.stdout.encoding` 以及是否有重定向来判断决定。

刚开始写博客的时候就遇到过[Unicode格式化的坑][2]，没想到快两年后还会再
遇到类似问题。不得不说 Python 2 的默认编码，以及 Unicode 和 Str 这块容
易出问题。可惜的是，虽然 Python 3 完美地解决了这些问题，但都 3.X 了，还
是没能大面积普及起来，真是遗憾。

 [1]: http://stackoverflow.com/questions/17419126/understanding-python-unicode-and-linux-terminal
 [2]: http://jerrypeng.me/2012/03/python-unicode-format-pitfall/
