---
layout: post
title: RHEL/CentOS 5 下 NAT 转发不工作的一个原因
date: 2014-07-30 21:54:45 +0800
comments: true
categories: Linux
tags: iptables, Linux, NAT, RHEL, CentOS, RH-Firewall-1-INPUT
keywords: iptables, Linux, NAT, RHEL, CentOS, RH-Firewall-1-INPUT
description: RHEL/CentOS 5 下面如果 NAT 转发规则不工作，不妨仔细看看 FORWARD 链的内容，你可能会有惊（keng）喜（die）的发现
---

**TL;DR** 如果你发现 RHEL/CentOS 5 下用 iptables 做的 NAT 转发规则不管
 用，请用 `iptables -L -nv` 检查一下 FORWARD 链里的内容，如果里面有一条
 直接转到 `RH-Firewall-1-INPUT` 的规则，那么你很有可能跟我们一样被坑了。
 尝试在 `RH-Firewall-1-INPUT` 链里把目标端口打开，那些规则应该就可以工
 作了。

公司的服务器上因为种种原因做了不少 iptables NAT 规则，用于做端口映射。
我们发现有的规则可以工作，有的则不然。但这些规则基本都长一个样，除了端
口和转发的目标 IP 各有不同以外。

<!--more-->

我们的规则很简单：

```
-A PREROUTING -p tcp --dport 443 -j DNAT --to 192.168.1.2:8443
-A POSTROUTING -d 192.168.1.2 -p tcp --dport 8443 -j SNAT --to 192.168.1.1
```

就是把一台服务器（`192.168.1.1`）上的 `443` 端口转发到另一台
（`192.168.1.2`）的 `8443` 上而已。但我们发现它死活不工作。但别的端口（比
如 `80` 转到 `8080`）的类似规则又是可以的：

```
-A PREROUTING -p tcp --dport 80 -j DNAT --to 192.168.1.3:8080
-A POSTROUTING -d 192.168.1.3 -p tcp --dport 8080 -j SNAT --to 192.168.1.1
```

各种 Google 搜索都没找到答案，无奈只能仔细检查下 iptables 的规则，结果
发现了一个线索：我们的 `8443` 端口没有打开，而 `8080` 是打开的！果断尝试
把 `8443` 也给打开，果然可以了。

不过问题是，为什么 `INPUT` 规则会对 NAT 转发造成影响呢？按照 iptables 的
工作方式，只有目的地址是本机的包才会经过 `INPUT` 链，而转发的包只会经过
`FORWARD` 链。好吧，答案其实很简单，怪我们没看仔细。RHEL/CentOS 5 的
iptables 会建立一个名叫 `RH-Firewall-1-INPUT` 的链，并且把 `INPUT` 和
`FORWARD` 都转到这条链上。

```sh
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [9418420:35897535679]
:RH-Firewall-1-INPUT - [0:0]
-A INPUT -j RH-Firewall-1-INPUT
-A FORWARD -j RH-Firewall-1-INPUT #FORWARD 直接挂到 RH-Firewall-1-INPUT 上了
-A RH-Firewall-1-INPUT -i lo -j ACCEPT
#......省略若干规则
-A RH-Firewall-1-INPUT -j REJECT --reject-with icmp-host-prohibited
```

这种做法个人觉得很坑爹，转发的包管那么多干嘛，让目标去判断要不要
ACCEPT 就好，你就老老实实 FORWARD 嘛。果然在 RHEL/CentOS 6 以后的版本
里，这个 `RH-Firewall-1-INPUT` 被干掉了。


