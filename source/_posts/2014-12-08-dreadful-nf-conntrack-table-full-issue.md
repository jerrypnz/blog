---
layout: post
title: 解决恶心的 nf_conntrack&#58; table full 问题
date: 2014-12-08 13:39:03 +0800
comments: true
categories: Linux
tags: iptables, nf_conntrack, ip_conntrack, netfilter
keywords: iptables, nf_conntrack, ip_conntrack, netfilter, table full
description: 启用了 iptables 且流量较大、并发连接较多的服务器很容易遇到 『nf_conntrack&#58; table full, dropping packet.』的错误，导致大量丢包，影响正常服务。本文介绍我们使用的解决方案。
---

相信有一定 Linux 服务器运维经验的人肯定见过这个问题：`dmesg` 或者
`/var/log/messages` 里大片地输出类似这样的日志：

```
Dec  8 11:22:29 product08 kernel: nf_conntrack: table full, dropping packet.
Dec  8 11:22:29 product08 kernel: nf_conntrack: table full, dropping packet.
```

同时服务器上的各种网络服务耗时大幅上升，各种 timed out，各种丢包，完全
无法正常提供服务。

随着我们流量的提升，最近又开始被这个问题虐了。几个月前第一次遇到此问题
的时候，我们使用了提高 nf_conntrack table size 的方法解决的，事实证明
这是个治标不治本的方法，流量上来了，改得再大最后还是会爆掉。况且 table
太大，占用的内存也会很多。

今天再次研究了一下这个问题，换了另一种更好的方案，尝试解决掉它。简单地
说，方案就是在 iptables raw 表里增加规则，将无需跟踪状态的包标记为
`NOTRACK`，这样就无需耗费 nf_conntrack table entry 来记录其状态了。

<!--more-->

## 1. nf_conntrack 是干嘛的

`nf_conntrack`（在老版本的 Linux 内核中叫 `ip_conntrack`）是一个内核模
块，用于跟踪一个连接的状态的。连接状态跟踪可以供其他模块使用，最常见的
两个使用场景是 iptables 的 `nat` 的 `state` 模块。

iptables 的 `nat` 通过规则来修改目的/源地址，但光修改地址不行，我们还
需要能让回来的包能路由到最初的来源主机。这就需要借助 `nf_conntrack` 来
找到原来那个连接的记录才行。

而 `state` 模块则是直接使用 `nf_conntrack` 里记录的连接的状态来匹配用
户定义的相关规则。例如下面这条 INPUT 规则用于放行 80 端口上的状态为
NEW 的连接上的包。

```
iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
```

## 2. 能否完全禁用 nf_conntrack

能否禁用 `nf_conntrack` 要看服务器干啥了，如果没有使用 `nat` 模块和
`state` 模块是可以禁用掉它的，这样做一劳永逸，再也不用被这个恶心的问题
折磨了。

但是我发现 RHEL 默认的 iptables 规则里都是用到了 `state` 模块的：

```
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -j REJECT --reject-with icmp-host-prohibited
```

按理说打开端口的规则完全不需要使用 `-m state --state NEW`，只要匹配了
目标端口直接放行就好。我理解这样做是为了优化性能。注意看第一条规则，只
要连接状态是 ESTABLISHED 或者 RELATED 直接放行。这样做，只有连接初始化
时的包需要经过下面的重重规则去一条条匹配，一旦建立上连接了（代表被放行
了），后续的包直接通过第一条规则就放行了，省去了一条条匹配后续规则的麻
烦（O(1)复杂度？）。

反之，如果我们不这么写，那连接的每个包都需要经过前面的层层规则过滤之后
才能被放行，对性能有影响（O(n)复杂度？）。当然只有 iptables 规则很多的
时候，这个问题才能突显，且到底能带来多大的影响，我手头没有测试数据，还
不是特别清楚。

保险起见，我们就没有全部禁用掉 `nf_conntrack`，而是采用了 `raw` 表来部
分地禁用连接跟踪。

## 3. 哪些连接可以不必跟踪状态

我们先看看怎么通过 `raw` 表针对部分连接禁用 conntrack。

```
# 针对进入本机的包
iptables -t raw -A PREROUTING -p tcp -m tcp --dport 8080 -j NOTRACK
# 针对从本机出去的包
iptables -t raw -A OUTPUT -p tcp -m tcp --dport 8080 -j NOTRACK
```

重点就是 `-t raw` 和 `-j NOTRACK`，前者指定 table，后者指定 action 为
`NOTRACK`——不跟踪状态。

那么问题来了，要针对哪些连接应用此规则呢？需要 NAT 的肯定不行，各种
redirect 的连接本质上也是 NAT，也一样不行。除此之外的大部分的普通连接
都是可以的。

比如所有 lo 接口上的连接：

```
iptables -t raw -A PREROUTING -i lo -j NOTRACK
iptables -t raw -A OUTPUT -o lo -j NOTRACK
```

在应用和 Nginx 都跑在一台服务器上的情况下，这能禁用掉很多连接的状态跟
踪，因为在并发高的时候，Nginx 和 Upstream 的连接也不是个小数目。

再比如针对流量很大的特定端口：

```
iptables -t raw -A PREROUTING -p tcp -m tcp --dport 8080 -j NOTRACK
iptables -t raw -A OUTPUT -p tcp -m tcp --sport 8080 -j NOTRACK
```

需要特别特别注意的是，被 `NOTRACK` 的包无法匹配上依赖特定状态的其他规
则。比如 INPUT 链上假如有这么一条规则：

```
iptables -A INPUT -p tcp -m state --state NEW -m tcp --dport 8080 -j ACCEPT
```

那么在增加了上面的 `NOTRACK` 规则后，你会发现这个端口的包都被丢掉了。因
为没有了连接状态跟踪，那个 `--state NEW` 不可能匹配得上。上午我就因为
犯了这个错误导致某关键服务宕机了几分钟……

{% img /images/201412/dont-always-test-code.jpg %}

所以要千万小心，一个很好的解决办法是加上这么一条规则（你想 `NOTRACK`
的，肯定是你要放行的）：

```
iptables -A INPUT -m state --state UNTRACKED -j ACCEPT
```

我们目前使用这种方案解决了目前的 nf_conntrack: table full 问题，只是在
流量最大的端口以及 lo 上启用这些规则就已经让该错误没再出现了（在并发没
有降低的情况下），再仔细优化下，把更多的端口加进来应该能解决绝大多数情
况下的问题了。

对专业运维来说，估计本文说的都不是事，但希望我这半吊子运维的这篇文章能
帮助到那些同样要兼职运维工作的苦逼码农们。

Have fun :-)

### 各种链接

- [解决 nf_conntrack: table full, dropping packet 的几种思路](http://jaseywang.me/2012/08/16/%E8%A7%A3%E5%86%B3-nf_conntrack-table-full-dropping-packet-%E7%9A%84%E5%87%A0%E7%A7%8D%E6%80%9D%E8%B7%AF/)
- [Hacker News 上的一篇相关讨论](https://news.ycombinator.com/item?id=4874289)
- [netfilter 链接跟踪机制与NAT原理](http://www.cnblogs.com/liushaodong/archive/2013/02/26/2933593.html) 
