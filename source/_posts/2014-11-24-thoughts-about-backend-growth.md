---
layout: post
title: 对后端系统规模上升的一些思考
date: 2014-11-24 10:14:19 +0800
comments: true
categories: 技术
tags: Docker, Service Discovery, Zookeeper, etcd
keywords: Docker, Service Discovery, Zookeeper, etcd
description: 公司的服务器越来越多，上面运行的服务也越来越多，运维越来越复杂，应对故障的响应速度也不够。这篇文件记录下自己的一些研究和思考，主要关于 Docker、服务发现、配置管理等。
---

随着[公司](http://www.trafree.com)业务的增长，我们的服务器数量越来越多，
上面运行的各种服务也越来越多；系统的架构也在逐渐复杂化，一个业务往往需
要调用后端的多个服务才能完成，服务间有着复杂的依赖关系。这些给我们的运
维带来了很大的麻烦，系统发布、监控、扩容等等都随着服务器和服务数量的上
升变得越来越麻烦，遇到故障尤其是整个服务器的硬件故障的时候，恢复时间也
越来越长。如果说当前还能忍受的话，那当规模再增长一倍的时候，运维的复杂
度就会不可控了。

最近针对这个话题我做了些研究，也有一些思考，还不成熟，但在这里先记录一
下。


## 1. 服务发现

我们当前还是用的最原始的配置文件和 DNS 来做服务发现，Host、端口都是写
在配置文件里的，发生变更的时候只能修改配置文件并重启服务。所以当某台机
器挂掉的时候，依赖它上面服务的其他系统也都全部会出问题。而应急的步骤都
是先在别的机器上运行新的实例，修改配置文件并重启关联的其他系统。这样做
费时、费力、且会有一个时间窗口内系统无法提供服务。

当然我们关键的服务是通过 Nginx 来做了负载均衡/主备的。但这样做还是有两
个问题：

<!--more-->

1. Nginx 本身成为一个故障点
2. 连接数量翻倍

其中第二个问题曾导致我们的环境出现了
[nf_conntrack table full](https://www.google.co.jp/search?client=safari&rls=en&q=nf_conntrack+table+full+dropping+packet&ie=UTF-8&oe=UTF-8&gfe_rd=cr&ei=efpuVIODIOSN8QeEwIHgDA)
的问题。我们的关键服务都是多实例负载均衡的，当系统并发上升到一定程度的
时候，某些服务器，尤其是跑着 Nginx 的机器很容易出现这个错误。

那如何解决这个问题呢？刚好前些日子在
[Tim 大神的博客](http://timyang.net/distributed/service-architecture/)
上看到了一篇文章为我指明了方向。

文章里提到，目前成熟的分布式服务多使用基于 ZooKeeper 的配置服务来实现
的。因此顺着这个思路往下 Google 了一下，发现它是一个靠谱的，可以考虑的
方案。

ZooKeeper 本身是个强一致性的分布式配置服务，是分布式系统的基础设施，可
以用来实现配置管理、服务发现、Leader 选举、分布式锁等等。Netflix 开源
了一个名为 [curator](http://curator.apache.org)(现在是 Apache 的顶级项
目）的库，里面实现了不少 ZooKeeper 常用的使用模式，以 Recipes 的方式供
用户直接使用。这其中就包含服务发现的功能。

其[文档](http://curator.apache.org/curator-x-discovery/index.html)中详
细介绍了实现，我这里也简单介绍一下：

1. 每个服务在 ZooKeeper 里都有一个专门的 Path
2. 每个服务实例在启动时都在这个 Path 下注册一个 Node，并附带本实例的
   Host 和端口等信息
3. 服务使用者从 ZooKeeper 里查询 Path 下的节点以获取当前活跃的实例和他
   们的 Host、端口等，使用者可以在客户端作负载均衡

其中服务实例注册的 Node 类型是 ephemeral node，这种类型的节点只有在客
户端保持着连接的时候才有效。所以当某个服务实例被停止或者出现网络异常的
时候，对应的节点也会被删掉。因此，任何时候从 ZooKeeper 里查询到的都是
当前活跃的实例。借助 ZooKeeper 的推送功能，服务的消费者可以得知实例的
变化，从而可以从容应对服务实例的宕机和新实例的添加，无需重启。

总结一下这个方案的好处：

1. 配置的解耦，服务的消费者只需要知道服务在 ZooKeeper 中的注册路径即可，
   无需配置 Host、端口等（这对于虚拟化或者容器化的方式尤其有用，详见下
   面 Docker 部分的描述）
2. 客户端的负载均衡，省去了额外的 Nginx，节省连接且去掉了单点
3. 很容易动态增减实例
4. ZooKeeper 本身是一个强一致性的集群，可以做到很高的可用性，消除了单
   点

比起用 DNS 和配置文件，这个方案优势实在太明显了，我认为是需要迈出去的
第一步。

## 2. 配置管理

我们系统的配置，目前绝大多数用的还是最原始的配置文件方式。对于实例很
多的服务，配置管理也是一个很麻烦的事。每次有修改配置的需求时，需要把相
关的配置文件全部修改一遍并挨个重启系统。这种做法太过于原始了，成本太高
了，且随着实例数的上升线性上升。

引入 ZooKeeper 或者类似的系统，配置管理的问题也可以很自然地得到解决：
直接使用它们就好了。易变的配置项全部都注册到 ZooKeeper 中，借助推送，
每个服务实例都可以获取到最新的配置。程序只需要保证能动态调整相关的配置
参数就行（写成 `static final xxx` 的都可以改改了）。

我在 Curator 里没有看到现成的配置管理工具，但是借助其
[Framework](http://curator.apache.org/curator-framework/index.html) 和
[Node Cache](http://curator.apache.org/curator-recipes/node-cache.html)
应该很容易自己实现一个。


## 3. Docker

在实现了上面的目标以后，我认为还可以使用现在红得发紫的新兴技术 Docker
来更进一步简化部署和运维，以便扫清进一步增长的障碍。

Docker 的背后，还是已经存在多年的 Linux 技术如 LXC、Aufs、cgroup 等。
借助这些技术，我们可以在一台 Linux 机器（Host）上运行多个互相隔离的容
器，它们和 Host 共享内核，但是有自己独立的进程空间、文件系统空间，在容
器内的进程看来好像是独立的机器一样。看起来容器和 VMWare，Xen，KVM 等虚
拟机差不多，但原理完全不同——前者是虚拟硬件设备，后者只是 Linux 内核做
出的隔离而已。容器更加轻量，因此性能损耗小很多，一台 Host 上可以运行很
多的容器。Docker 把这些技术的使用简化了很多，让人可以通过命令很简单地
管理容器镜像和运行容器。Docker 借助 Aufs 实现了镜像的『继承』机制，从
而节省磁盘空间并且简化镜像的创建和管理。

使用 Docker 能带来下面这些好处：

1. 可以简化开发、测试和部署的环境准备，消除环境变更带来的问题。通过
Dockerfile 可以很容易脚本化镜像的创建。
2. 简化部署。运维不再需要了解应用的细节，直接管理容器即可。
3. 很容易实现自动化。

不过下面这个问题也让我比较顾虑：

Docker 容器的 IP 地址是随机分配的。对于应用本身，通过前面提到的服务发
现机制可以消除随机 IP 带来的影响，但是对于一些基础设施如 Nginx，MySQL，
Redis 等，可能就是问题。比如 Nginx 中配置的 Upstream 如果重启并且 IP
变化了，Nginx 如何快速获取这个变化并重新配置？
[解决方案倒也有](http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/)
，需要借助类似 ZooKeeper 的 [etcd](https://github.com/coreos/etcd)。通
过类似的思路，我们完全可以统一使用 ZooKeeper 来实现。不过我认为还是比
较麻烦，并且 Redis Sentinel、Redis 主从、MySQL 主从能否用类似的方法实
现动态发现也是个问题。


## 4. 结论

对于服务发现和配置管理，我认为是需要迈出的第一步，它可以为更进一步的演
进扫清障碍。对于 Docker，我目前还是稍微持保守态度一些。只要减小应用对
环境的依赖，并借助 chef，puppet 之类的配置管理工具，大规模的部署应该一
样不会太复杂。

## 5. 参考

- Curator Service Discovery 实例：
http://aredko.blogspot.jp/2013/10/coordination-and-service-discovery-with.html
- 开源服务发现方案（有ZooKeeper之外的方案）：
  http://jasonwilder.com/blog/2014/02/04/service-discovery-in-the-cloud/
- Pinterest 对 ZooKeeper 服务发现的改进（主要防止 ZooKeeper 本身出故
  障）：
  http://engineering.pinterest.com/post/77933733851/zookeeper-resilience-at-pinterest
- Docker 核心技术概览：http://www.infoq.com/cn/articles/docker-core-technology-preview
