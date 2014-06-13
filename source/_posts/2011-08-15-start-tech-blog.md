---
title: 开始写技术博客
author: Jerry
layout: post
permalink: /2011/08/start-tech-blog/
views:
  - 275
categories:
  - 技术
tags:
  - Blog
---
写博客这个活动，从08年到现在已经停滞了3年，现在是时候重新拾起来了。

这三年，我完成了从一个痴迷技术的学生到职业软件工程师的转变，期间参加了真刀真枪的项目研发，遇到了各种各样的技术难题，对自己已经了解的知识有了更深层次的认识，更接触到了曾经从未想象过的技术领域。这段时间里，私以为自己有一些积累和沉淀，也逐渐开始形成了自己的一套知识体系和思维方式——当然还很浅薄，需要更多的锤炼。最大的遗憾就是因为自己的散漫而未能将知识积累的过程记录下来。三年职业生涯过后的现在，我也在逐渐进入一个升华和蜕变的阶段，这一次，我希望自己不要再散漫下去，而是要坚持把自己学到的，想到的，领悟到的，完整记录在这里。

现在翻看自己以前的博客文章，发现有营养的“干货”很少，更多的是心情随笔，对于别人是没有什么价值的。这一次，我希望自己的博客能尽可能多地谈技术话题，最好有一定深度，包含自己思考和发现的过程。我认为，博客一定要能给予阅读者价值，要起到传播知识的作用。这些年的工作中，我的Google Reader中也订阅了很多技术博客，这些对开阔自己视野有很大帮助，真的可以称之为是一笔财富。而现在，我希望通过自己的经营，能让这个博客也能传播知识和价值。

积累需要过程，我要坚持下去，嗯。

下面的代码是测试wp-syntax插件专用，请自觉无视。

<pre lang="python">def hello_world(*args, **kwargs):
	print "Hello world"
	print "Args: ", ', '.join(args)
	print "Keyword Args: ", ", ".join(["%s=%s" % (k, v) for k, v in kwargs.items()])
</pre>