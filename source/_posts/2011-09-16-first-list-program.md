---
title: 第一个Lisp程序
author: Jerry
layout: post
permalink: /2011/09/first-list-program/
views:
  - 251
categories:
  - Lisp
tags:
  - Lisp
---
受周围同事以及《黑客与画家》的影响，最近开始学习Common Lisp了。我的目标并不是要用Lisp写程序，而是在学习的过程中去感受它的设计思想，拓展一下自己的思维，我太想了解为什么那么多黑客都如此推崇这个看起来很古怪甚至有点恶心的语言了。

我看的是《Practial Common Lisp》，目前刚看完函数那一章，略有一些体会：

*   <span style="line-height: 18px;">Lisp的语法和语义十分简洁</span>
*   <span style="line-height: 18px;">强大的宏——这应该是Lisp的必杀技之一了吧？</span>
*   <span style="line-height: 18px;">看到了别的语言的影子——或者说了解到那些语言特性的出处原来都是Lisp</span>
*   <span style="line-height: 18px;">看起来好像挺简单，写起程序来真难！</span>

学Lisp让我感觉回到了初学编程的岁月，一切都是新的，都不明白，想写个程序发现障碍太多了，不知道从何下手。本来我想写个通用的冒泡排序程序，发现还有太多技术障碍需要扫清，还是先写个简单的吧——山寨版reduce。不知道什么是reduce？请查阅Python文档中关于built-in function的部分。

<pre lang="lisp">(defun my-reduce (func data &#038;optional initval)
  (let ((result initval))
    (dolist (item data)
      (setf result (funcall func item result)))
    result))

(format t "Plus all: ~d~%" (my-reduce #'+ '(1 2 3 4) 0))
(format t "Multiply all: ~d~%" (my-reduce #'* '(1 2 3 4) 1))
(format t "Find max: ~d~%" (my-reduce #'(lambda (x y) (if (> x y) x y)) '(1 0 5 9 11 3) -1))</pre>

运行结果：

<pre>Plus all: 10
Multiply all: 24
Find max: 11</pre>

我觉得学习一个学习曲线陡峭，并且和已经掌握的语言完全不同的新语言是个很好的训练思维的方式。希望我能坚持把Lisp学下去——这可是[黑客的终极语言][1]呢！

 [1]: http://www.ruanyifeng.com/blog/2010/10/why_lisp_is_superior.html