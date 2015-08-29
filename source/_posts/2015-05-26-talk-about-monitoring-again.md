---
layout: post
title: 再聊监控
date: 2015-05-26 20:30:44 +1200
comments: true
categories: Tech
tags: monitoring, riemann, influxdb, grafana
keywords: monitoring, riemann, influxdb, grafana
description: 在新公司再次研究监控系统，这次使用了一套完全不同的方案。
---

去年我曾写过一个[系列文章]({% post_url 2014-06-19-server-side-java-monitoring %})
介绍在上一家公司搭建的监控系统，那个方案基于 Ganglia 和 Nagios，
效果很不错，对我们监控性能状况，及时发现问题起了不小的作用。

不巧的是，在新公司入职不久，我又开始和同事一起研究起监控系统来。作为技
术上比较『追新』的公司，我们也把视线放到了比较新的一些方案上了。目前我
们的监控系统已经上线运行了一些时间，效果也很不错。在此对用到的主要子系
统做一个介绍，希望能对有兴趣或者正在研究监控方案的朋友们带来一些帮助。

## 绝对的核心 Riemann

[Riemann](http://riemann.io/index.html) 是一个分布式监控系统，但它做的
事情其实非常简单，就是接收事件，按照配置的 `stream` 对事件进行处理并转
发到 InfluxDB、Graphite 或者产生告警、发邮件等。官网的一张架构图很好地
解释了这一点：

<div style="background: #B70034; width: 500px; padding: 10px; margin: 10px;">
{% img /images/201505/riemann-arch-diagram-min.png %}
</div>

Riemann 是用 Clojure 实现的，作为一个 Clojure 拥趸，这一点极大地吸引了
我。

<!--more-->

Riemann 的配置文件用的也是 Clojure，你需要使用其提供的 DSL 来编写
`stream` 来实现想要的告警等逻辑。Riemann 中的 `stream` 是指以 event 为
参数的函数，`stream` 可以组合形成一个树形结构，事件就在 `stream` 中流
动，触发处理逻辑。例如下面的片段：

```clojure
(streams
  (where (and (service #"^jvm")
              (state "critical"))
         (email "foo@bar.com")))
```

匹配状态是 `critical`，服务名以 jvm 开头的事件，然后发送邮件。

Riemann 提供了大量的函数帮助编写 `stream`，例如按时间窗口聚合，事件映
射等等，详细可以参考其 [API 文档](http://riemann.io/api.html)。

Riemann 还有个 `index` 的概念，可以用于在内存中短暂存储事件并提供一个
查询机制，配合 [Riemann Dash](http://riemann.io/dashboard.html) 可以用
来实现一个能实时查看系统状态的 Dashboard。

<div style="width: 700px; margin: 10px;">
{% img /images/201505/dash-riak-min.png %}
</div>

Riemann 使用 protocol buffer 作为接口，有不同语言的
[Client Library](http://riemann.io/clients.html) 可供使用。对于 JVM 的
监控，我们还是扰不开 JMX：
[riemann-jmx](https://github.com/twosigma/riemann-jmx) 可以从 JMX MBean
收集数据并发送到 Riemann。

不得不说 Riemann 是有一定的学习曲线的，需要了解 Clojure，并不像
Ganglia 那样配置好就可以使用。但显然它更加灵活，用它作为一个 Middleman
可以很容易实现各种复杂需求。

## InfluxDB 和 Graphana

前面提到了 Riemann 只有一个 in-memory 的 `index` 存储，但我们往往需要
持久化一部分监控数据，方便日后查询和比对。这时我们就需要一个存储机制了。
目前比较流行的方案有下面这些：

- Graphite
- InfluxDB
- OpenTSDB

Riemann 对他们都有支持，可以很简单地将事件转发过去。

我们使用了 [InfluxDB](http://influxdb.com/)，一个分布式 time series 数
据库。它提供了一个类似 SQL 的语言来查询数据和一个简单的界面来展现数据。
但想绘制比较强大的图表，还需要其他工具，比如
[Graphana](http://grafana.org/)。

Graphana 同样支持不同的数据后端，前面提到的三个系统它都支持，可以很容
易集成到一起。

## 其他

虽然和监控关系不是特别大，我们还使用了 Logstash 来收集和存储日志，而
Logstash 也是可以和 Riemann 集成的，可以用 Logstash 来从日志中提取感兴
趣的指标，然后发到 Riemann 中作进一步处理。通过它可以实现监控日志中出
现的异常，并基于异常作告警处理等。
