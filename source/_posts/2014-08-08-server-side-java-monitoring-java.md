---
layout: post
title: Java 服务端监控方案（四. Java 篇）
date: 2014-08-08 14:11:53 +0800
comments: true
categories: Java
tags: Java, 监控, Linux, Ganglia, jmxtrans, JMX
keywords: Java, Monitor, Linux, Server, Ganglia, jmxtrans, JMX
description: 服务端应用的重要一环是监控，本文介绍 Java 的服务端监控解决方案，基于 Ganglia，Nagios 和 JMX。
---

这个漫长的系列文章今天要迎来最后一篇了，也是真正与 Java 有关的部分。前
面介绍了我们的监控方案的
[Ganglia]({% post_url 2014-07-04-server-side-java-monitoring-ganglia %})
和
[Nagios]({% post_url 2014-07-22-server-side-java-monitoring-nagios %})
及其整合的部分，这一次则介绍如何记录 Java 应用内的性能参数并将其暴露给
监控系统。

主要介绍的内容有 JMX 以及将监控 JMX 并发送数据到 Ganglia 的 jmxtrans，
同时还会介绍我实现的一个简单的记录性能参数的方法。

<!--more-->

## 1. JMX

JMX 基本上是 Java 应用监控的标准解决方案，JVM 本身的诸多性能指标如内存
使用、GC、线程等都有对应的 JMX 参数可供监控。自定义 MBean 也是十分简单
的一件事。可以用两种方式来定义 MBean，第一种是通过自定义接口和对应的实
现类，另一种则是实现 `javax.management.DynamicMBean` 接口来定义动态的
MBean。我们采用的是第二种方式，因此略过第一种方式的介绍，有兴趣的读者请
参考[Java Tutorial] [1] 里的教程和 [Javalobby] [2] 上的文章。

下面是我们内部使用的 `MetricMBean`，使用 `DynamicMBean` 实现：

```java
public class MetricsMBean implements DynamicMBean {

    private final Map<String, Metric> metrics;

    public MetricsMBean(Map<String, Metric> metrics) {
        this.metrics = new HashMap<>(metrics);
    }

    @Override
    public Object getAttribute(String attribute)
            throws AttributeNotFoundException,
                   MBeanException,
                   ReflectionException {
        Metric metric = metrics.get(attribute);
        if (metric == null) {
            throw new AttributeNotFoundException("Attribute " + attribute + " not found");
        }
        return metric.getValue();
    }

    @Override
    public void setAttribute(Attribute attribute)
            throws AttributeNotFoundException,
                   InvalidAttributeValueException,
                   MBeanException,
                   ReflectionException {
        // 我们仅仅需要做监控，没有设置属性的需要，所以直接抛异常
        throw new UnsupportedOperationException("Setting attribute is not supported");
    }

    @Override
    public AttributeList getAttributes(String[] attributes) {
        AttributeList attrList = new AttributeList();
        for (String attr : attributes) {
            Metric metric = metrics.get(attr);
            if (metric != null)
                attrList.add(new Attribute(attr, metric.getValue()));
        }
        return attrList;
    }

    @Override
    public AttributeList setAttributes(AttributeList attributes) {
        // 我们仅仅需要做监控，没有设置属性的需要，所以直接抛异常
        throw new UnsupportedOperationException("Setting attribute is not supported");
    }

    @Override
    public Object invoke(String actionName,
                         Object[] params,
                         String[] signature) throws MBeanException, ReflectionException {
        // 方法调用也是不需要实现的
        throw new UnsupportedOperationException("Invoking is not supported");
    }

    @Override
    public MBeanInfo getMBeanInfo() {
        SortedSet<String> names = new TreeSet<>(metrics.keySet());
        List<MBeanAttributeInfo> attrInfos = new ArrayList<>(names.size());
        for (String name : names) {
            attrInfos.add(new MBeanAttributeInfo(name,
                                                 "long",
                                                 "Metric " + name,
                                                 true,
                                                 false,
                                                 false));
        }
        return new MBeanInfo(getClass().getName(),
                             "Application Metrics",
                             attrInfos.toArray(new MBeanAttributeInfo[attrInfos.size()]),
                             null,
                             null,
                             null);
    }

}
```

其中 Metric 是我们设计的一个接口，用于定义不同的监控指标：

```java
public interface Metric {

    long getValue();
}
```

最后是一个工具类 `Metrics` 用于注册和创建 MBean：

```java
public class Metrics {

    private static final Logger log = LoggerFactory.getLogger(Metrics.class);
    private static final Metrics instance = new Metrics();
    private Map<String, Metric> metrics = new HashMap<>();

    public static Metrics instance() {
        return instance;
    }

    private Metrics() {
    }

    public Metrics register(String name, Metric metric) {
        metrics.put(name, metric);
        return this;
    }

    public void createMBean() {
        MetricsMBean mbean = new MetricsMBean(metrics);
        MBeanServer server = ManagementFactory.getPlatformMBeanServer();
        try {
            final String name = MetricsMBean.class.getPackage().getName() +
                                ":type=" +
                                MetricsMBean.class.getSimpleName();
            log.debug("Registering MBean: {}", name);
            server.registerMBean(mbean, new ObjectName(name));
        } catch (Exception e) {
            log.warn("Error registering trafree metrics mbean", e);
        }
    }

}
```

在应用启动的时候这样调用以注册指标并创建 MBean：

```java
// createMaxValueMetric 和 createCountMetric 可以基于同一份数据来得到
// 最大值和次数的指标，详见下面 AverageMetric 的具体实现。
Metrics.instance()
       .register("SearchAvgTime", MetricLoggers.searchTime)
       .register("SearchMaxTime", MetricLoggers.searchTime.createMaxValueMetric())
       .register("SearchCount", MetricLoggers.searchTime.createCountMetric())
       .createMBean();
```

其中注册时指定的名称也是最后从通过 JMX 看到的属性名。

当然上面只是我们内部的监控框架的做法，你需要关注的是如何实现自定义
MBean 而已。

上面提到的 `Metric` 接口，我并没有给出实现。下面介绍我们内部常用的一个
实现 `AverageMetric` （平均值指标）。它可以记录某个性能数值，并计算单
位时间内的平均值，最大值和次数。例如上面的 `MetricLoggers` 中定义的
`searchTime`，它用来记录我们系统的搜索功能的一分钟平均耗时，一分钟最大
耗时和一分钟的搜索次数。

```java
public class MetricLoggers {
    public static final AverageMetric searchTime = new AverageMetric();
}
```

在实际的搜索功能处记录耗时：

```java
long startTime = System.currentTimeMillis();
doSearch(request);
long timeCost = System.currentTimeMillis() - startTime;

MetricLoggers.searchTime.log(timeCost);
```

这样通过 JMX 就可以监控到我们系统过去一分钟内的平均搜索耗时，最大搜索
耗时以及搜索次数。

下面是 `AverageMetric` 类的具体实现，比较长，请慢慢看。基本思路就是使
用 AtomicReference 和一个值对象，通过非阻塞算法来实现并发。经过测试，
在并发度不高的情况下性能不错，但在线程很多，竞争激烈的时候不是很好。再
次重申，这个实现仅供参考。

```java
public class TimeWindowSupport {
    final long timeWindow;

    TimeWindowSupport(long timeWindow) {
        this.timeWindow = timeWindow;
    }

    long currentSlot() {
        return System.currentTimeMillis() / timeWindow;
    }
}


public class AverageMetric extends TimeWindowSupport implements Metric {

    final AtomicReference<Value> currentValue = new AtomicReference<Value>();
    private volatile Value lastValue = null;

    public AverageMetric(long timeWindow) {
        super(timeWindow);
    }

    public AverageMetric() {
        super(TimeUnit.MINUTES.toMillis(1));
    }

    public Value getLastValue() {
        long slot = currentSlot();
        while(true) {
            Value curValue = currentValue.get();
            if (curValue != null && slot != curValue.slot) {
                if (currentValue.compareAndSet(curValue, Value.create(slot))) {
                    lastValue = curValue;
                    break;
                }
            } else {
                break;
            }
        }
        return lastValue;
    }

    public void log(long value) {
        long slot = currentSlot();
        while (true) {
            Value curValue = currentValue.get();
            if (curValue == null) {
                if (currentValue.compareAndSet(null, Value.create(slot, value)))
                    return;
            } else if (slot == curValue.slot) {
                if (currentValue.compareAndSet(curValue, curValue.add(value)))
                    return;
            } else {
                if (currentValue.compareAndSet(curValue, Value.create(slot, value))) {
                    lastValue = curValue;
                    return;
                }
            }
        }
    }

    /**
     * 基于同样的数据，创建一个计数度量，其返回值是过去的单位时间内的log事件发生次数
     *
     * @return 返回计数度量
     */
    public Metric createCountMetric() {
        return new Metric() {
            @Override
            public long getValue() {
                Value val = getLastValue();
                if (val != null)
                    return (long) val.n;
                else
                    return 0L;
            }
        };
    }

    /**
     * 基于同样的数据，创建一个最大值度量，其返回值是过去的单位时间内记录的最大数值
     *
     * @return 返回最大值度量
     */
    public Metric createMaxValueMetric() {
        return new Metric() {
            @Override
            public long getValue() {
                Value val = getLastValue();
                if (val != null)
                    return val.max;
                else
                    return 0L;
            }
        };
    }

    @Override
    public long getValue() {
        Value lastValue =  getLastValue();
        long lastSlot = currentSlot() - 1;
        if (lastValue != null && lastValue.n != 0 && lastSlot == lastValue.slot)
            return lastValue.total / lastValue.n;
        else
            return 0L;
    }

    static class Value {
        final long slot;
        final int n;
        final long total;
        final long max;

        Value(long slot, int n, long total, long max) {
            this.slot = slot;
            this.n = n;
            this.total = total;
            this.max = max;
        }

        static Value create(long slot, long value) {
            return new Value(slot, 1, value, value);
        }

        static Value create(long slot) {
            return new Value(slot, 0, 0, 0);
        }

        Value add(long value) {
            return new Value(this.slot,
                             this.n + 1,
                             this.total + value,
                             (value > this.max) ? value : this.max);
        }
    }
}
```

[1]: http://docs.oracle.com/javase/tutorial/jmx/mbeans/standard.html
[2]: http://www.javalobby.org/java/forums/t49130.html

## 2. jmxtrans

有了 JMX，我们还缺少最后一环：将监控数据发给我们前面辛苦搭建的监控系统。
我们的核心系统是 Ganglia，所以要将数据发送给它。我们选择的是
[jmxtrans](https://github.com/jmxtrans/jmxtrans) 这个解决方案。它本身
也是用 Java 实现的，使用 JSON 作为配置文件。

### 2.1 安装

它提供了
[deb，rpm 和标准的 zip 包](https://github.com/jmxtrans/jmxtrans/downloads)
，很方便安装。按照发行版选择安装即可。

### 2.2 配置

jmxtrans 的配置文件在 `/var/lib/jmxtrans` 下，使用 JSON 格式。针对要监
控的每个应用创建一个 JSON 文件，按下面的格式配置即可。下面我附加了注释，
但实际的配置文件如果有这种注释貌似会报错，请注意。

```json
{
  "servers" : [ {
    "host" : "localhost", // JMX IP
    "port" : "19008", // JMX 端口
    // 别名，用于Ganglia对参数来源的识别，写成本机IP和Hostname即可
    "alias" : "192.168.221.29:fly2save02",
    "queries" : [
    {
      "outputWriters" : [ {
        "@class" : "com.googlecode.jmxtrans.model.output.GangliaWriter",
        "settings" : {
          "groupName" : "myapp", //Ganglia里的参数组名
          "host" : "192.168.1.9", //Ganglia的IP
          "port" : 8648, //Ganglia的端口
          "slope" : "BOTH",
          "units" : "bytes", //参数单位
          "tmax" : 60,
          "dmax" : 0,
          "sendMetadata": 30
        }
      } ],
      "obj" : "java.lang:type=Memory", //要监控的 MBean 的标识
      "resultAlias" : "app", //别名，使用别名可以避免名称过长
      "attr" : [ "HeapMemoryUsage", "NonHeapMemoryUsage" ] //要监控的MBean属性
    },
    // 要监控多个 MBean，需要写多组 query，其中 outputWriters 部分会冗
    // 余，这个比较恶心。
    {
      "outputWriters" : [ {
        "@class" : "com.googlecode.jmxtrans.model.output.GangliaWriter",
        "settings" : {
          "groupName" : "myapp",
          "host" : "192.168.1.9",
          "port" : 8648,
          "slope" : "BOTH",
          "tmax" : 60,
          "dmax" : 0,
          "sendMetadata": 30
        }
      } ],
      "obj" : "com.trafree.metrics:type=MetricsMBean", //我们应用的MBean
      "resultAlias" : "app"
      //未指定attr意味着要监控所有属性
    }
  ]
  } ]
}
```

更详细的配置请参考[官方WIKI](https://github.com/jmxtrans/jmxtrans/wiki/GangliaWriter)。

### 2.3 运行

首先应用一定要打开 JMX Remote，为应用添加如下的 JVM 参数。

```
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=19008
-Dcom.sun.management.jmxremote.local.only=true
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

我们的应用和 jmxtrans 是运行在同一台机器上的，所以把 `local.only` 改成
了 `true`，仅允许本地连接，同时去掉了认证和 SSL 的支持。如果你们的部署
方式不同，请按需求调整。

jmxtrans 的运行很简单，启动相应的服务即可（确保 `java` 在 `PATH` 里）：

```sh
chkconfig --add jmxtrans
/etc/init.d/jmxtrans start
```

## 3. 总结以及其他解决方案介绍

至此，我们的完整监控方案基本成型了。借助 Ganglia，Nagios，JMX 和
jmxtrans，我们可以完整地监控从 OS 到应用的方方面面，可以很轻松地做告警
支持，也可以很方便地查看历史趋势。

下面 Show 两张图，是我们的核心机票检索引擎的性能参数在 Ganglia 和
Nagios 里的样子：

- Ganglia 的聚合视图，堆叠展示多个实例上的同一指标

{% img /images/201408/ganglia1.jpg %}

- 从 Nagios 里看到的这些服务的状态，若从 OK 变成 WARN/CRITICAL，我们会马上
  收到邮件

{% img /images/201408/nagios1.jpg %}

终于完成了这个系列的文章，欢迎读者留下自己的想法，欢迎交流。

### 3.1 其他方案

在研究这些的时候，我也发现了一些其他的解决方案，在这里一并提一下，感兴
趣的可以深入研究下（欢迎交流）：

- [collectd](https://collectd.org/) 是 Ganglia 的一个不错的替代品，貌
  似更加轻量一些，性能也很不错，应该更适合小集群。他也可以和 Nagios 很
  好地整合。
- [Metrics](http://metrics.codahale.com/) 是一个 Java 库，提供了用于记
  录系统指标的各种工具，基本上是我们自己实现的 `MetricMBean` 的最佳替代
  品，功能强大，并且支持很多常用组件如 Jetty，Ehcache，Log4j 等，并且可
  以发送数据到 Ganglia。如果早点发现这个，我可能就不会自己写上面介绍的
  那一套方案了。对了，它还有
  [Clojure 绑定](https://github.com/sjl/metrics-clojure)，如果是
  Clojure 应用，那更可以考虑使用它了。

### 系列文章导航

- [Java 服务端监控方案（一. 综述篇）]({% post_url 2014-06-19-server-side-java-monitoring %})
- [Java 服务端监控方案（二. Ganglia 篇）]({% post_url 2014-07-04-server-side-java-monitoring-ganglia %})
- [Java 服务端监控方案（三. Nagios 篇）]({% post_url 2014-07-22-server-side-java-monitoring-nagios %})
