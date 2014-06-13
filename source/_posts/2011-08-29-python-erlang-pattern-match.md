---
title: 用Python山寨一个Erlang风格的模式匹配
author: Jerry
comments: true
layout: post
permalink: /2011/08/python-erlang-pattern-match/
views:
  - 530
categories:
  - Python
tags:
  - Erlang
  - Pattern Match
  - Python
---

## 1. Intro

最近在做的项目，是一个用Python实现的代理程序，服务端程序是另一个同事用
Erlang实现的。虽然我对Erlang了解不多，但对其必杀特性之一模式匹配
（Pattern Matching）早有耳闻，因此花了点时间了解了一下。模式匹配是
Erlang语言的核心特性之一，在case, receive等语句中都可以使用，但这篇文章
只考虑函数定义中的模式匹配，因为我试图用Python去山寨的也是这一特性。

模式匹配是用声明式的风格来描述本来需要用一连串的if&#8230;else&#8230;才
能表达的逻辑，方便维护和修改。下面这段从learnyousomeerlang网站上找到的
代码是一个很好的例子（原
文:<http://learnyousomeerlang.com/syntax-in-functions>)：

通常的 if&#8230;else&#8230; 式写法：

```erlang
function greet(Gender,Name)
    if Gender == male then
        print("Hello, Mr. %s!", Name)
    else if Gender == female then
        print("Hello, Mrs. %s!", Name)
    else
        print("Hello, %s!", Name)
end
```

使用模式匹配的写法：

```erlang
greet(male, Name) ->;
    io:format("Hello, Mr. ~s!", [Name]);
greet(female, Name) ->;
    io:format("Hello, Mrs. ~s!", [Name]);
greet(_, Name) ->;
    io:format("Hello, ~s!", [Name]).
```

基于函数的模式匹配乍一看有点像Java，C++里函数重载，但函数重载是静态语言
的编译时特性，模式匹配则完全是运行时行为。模式匹配的更多细节可以参考
Erlang的教程、文章，这里不再详述。

下面介绍如何用Python实现一个最简单的模式匹配功能。

## 2. What It Looks Like

第一个要考虑的问题是如何用Python已有的语言特性来表达模式匹配。因为
Python的函数不支持这种特性，所以必须要将函数强化一下才有可能实现模式匹
配。毫无疑问，decorator必然会派上用场的。decorator本身是支持参数化构造
的——这意味着用decorator的参数来表达模式最合适不过了。

顺着这个思路，上面那段函数定义的等价Python实现可以出炉了：

```python
@match("male")
def greet(male, name):
    print "Hello, Mr. %s" % name

@match("female")
def greet(female, name):
    print "Hello, Mrs. %s" % name

@match()
def greet(_, name):
    print "Hello, %s" % name
```

match这个装饰器的目的就是为对应的函数描述其需要满足的Pattern，match的参
数和函数本身的参数对应，但可能少于函数实际参数的个数，因为并不是每一个
参数都需要做模式匹配。比如上面头两个greet函数都只匹配第一个参数的值。关
于这一点，看函数调用的写法就能明白了：

```python
greet('male', 'Cooper')
greet('female', 'Geller')
greet('', 'Drizzt')
```

运行结果：

```
Hello, Mr. Cooper
Hello, Mrs. Geller
Hello, Drizzt
```

现在的问题是如何实现match这个decorator了。

## 3. How It Is Implemented

基本的思路是这样的：

1.  按函数名将一组函数及其pattern组织到一起
2.  将这些函数替换成一个统一的入口函数
3.  入口函数根据参数去寻找最匹配的实现函数并调用

下面是对应的代码：

```python
class PatternError(Exception):
    pass

class _PatternGroup(object):
    """match matching for python"""
    patterns_matchers = {}

    def __init__(self):
        super(_PatternGroup, self).__init__()
        self._rules = []

    def _add_pattern(self, fn, *args, **kwargs):
        self._rules.append(((args, kwargs), fn))
        return self._run

    def _run(self, *args, **kwargs):
        retval = None
        for rule, fn in self._rules:
            if self._match(rule, (args, kwargs)):
                retval = fn(*args, **kwargs)
                break
        else:
            raise PatternError("No match found")
        return retval

    def _match(self, rule, val):
        pargs, pkwargs = rule
        args, kwargs = val
        for actual, expected in itertools.izip(args, pargs):
            if actual != expected:
                return False
        for key, expectedval in pkwargs.iteritems():
            if not key in kwargs or kwargs[key] != expectedval:
                return False
        return True

def match(*args, **kwargs):
    def _do_match(fn):
        fname = fn.__name__ #FIXME Problem 1: Should not only use function name as the key
        if fname in _PatternGroup.patterns_matchers:
            return _PatternGroup.patterns_matchers[fname]._add_pattern(fn, *args, **kwargs)
        else:
            matcher = _PatternGroup()
            _PatternGroup.patterns_matchers[fname] = matcher
            return matcher._add_pattern(fn, *args, **kwargs)
    return _do_match
```

非常简陋，还有很多问题：

1.  用函数的名称来当作一组pattern的key是很不靠谱的，一旦有不同模块的同名函数就有问题
2.  因为self的缘故，不支持class的instance method
3.  match的算法很简陋，找到第一个能匹配上的就返回了，实际应该找到最匹配的函数

实现完以后发现这个东西没有太多实用价值，只能当作玩具而已，在这里记录下
来，当作分享自己探索和玩乐的过程。

完整的源代码：
<https://github.com/moonranger/python/blob/master/practice/pattern_match.py>

## 4. What Others Have Done

在实现完自己的简单Pattern Matching后，我也在Google上搜索了一番，还真有人跟我做了同样的事情，  
而且比我的要成熟得多，有兴趣的话可以看看人家的实现。

Python Pattern Matching  
<http://svn.colorstudy.com/home/ianb/recipes/patmatch.py>

Thoughts About the Erlang Runtime  
<http://blog.ianbicking.org/2008/06/09/thoughts-about-the-erlang-runtime/>
