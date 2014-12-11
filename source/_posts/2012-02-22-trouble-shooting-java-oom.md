---
title: 记一次OOM的排查过程
author: Jerry
comments: true
layout: post
permalink: /2012/02/trouble-shooting-java-oom/
categories:
  - Java
  - 技术
tags:
  - btrace
  - Java
  - OOM
---
在最近公司产品的一次release中，我们遇到了一个Java OOM的问题，追查了几个小时才解决问题，而且事后发现造成问题的原因很简单，但追查的过程我认为值得记录一下。

## 1. 现象

我们的应用是跑在64位的Red Hat Enterprise Linux上的，Heap配置为1G。在那天release跑BAT测试用例的时候，发现不定期地系统会开始不工作，一查后台的日志，能发现不少OutOfMemoryError的异常，如下：

```
java.lang.OutOfMemoryError
	java.util.zip.ZipFile.open(Native Method)
	java.util.zip.ZipFile.(ZipFile.java:112)
	java.util.jar.JarFile.(JarFile.java:127)
	java.util.jar.JarFile.(JarFile.java:92)
	org.apache.catalina.loader.WebappClassLoader.openJARs(WebappClassLoader.java:1544)
	org.apache.catalina.loader.WebappClassLoader.findResourceInternal(WebappClassLoader.java:1763)
	...
```

异常是在ClassLoader打开jar文件以做类加载时抛出的，看堆栈跟踪发现与具体加载的类无关，是随机出现的。Google了一番，发现ZipFile类在打开文件的时候用的是本地内存而不是Java堆（所以OOM后面没有说是heap space）。我用pmap来看了一下进程的内存占用信息，发现竟然能占用到3G多！Heap Space才1G，内存却占用有3G，而且我们没有用DirectBuffer之类的需要占用堆外内存的类，所以我们推测应该是有线程泄漏的情况：每个Java线程都需要堆栈空间，而堆栈空间的确是Java Heap之外的。我们没有改-Xss参数，即线程堆栈大小为默认的1M，所以内存占用有3G，说明系统中存在这大量的线程（几千个）。

不过这里还有一件事有些蹊跷，为什么内存占用到3G多就抛OutOfMemory了？毕竟我们用的是64位OS，地址空间应该超级庞大才对。结果我们发现……

## 2. 32-bit JVM on 64-bit Linux

在这个OOM的问题困扰了我们好半天，导致测试进行不下去的时候我才开始往这个方向想——毕竟谁能想到会发生这么低级的失误，在64位的OS上运行32位的JVM呢？

 开始怀疑这个以后，用file命令查了一下系统JRE里的java可执行文件，结果是：

```
[root@X64_213 bin]# file java
java: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.2.5, dynamically linked (uses shared libs), for GNU/Linux 2.2.5, stripped
```

 果然和自己怀疑的一样。

然后我们换成了64位的JRE，继续测试，但其实真正的问题还没解决，只是暂时被隐藏起来了而已。

## 3. 系统卡死

换成64位JRE以后我们只测试了一会，新的问题就出现了，这次是系统卡死。其实这也在预料之中，只不过在忙于功能测试的时候暂时选择性无视了而已。

用pmap看了一下java进程的内存占用，好家伙，先是12G，接着有一次到了27G！！！系统完全没有响应，ssh也完全卡住了。这次我们不得不认真追查问题了。

用ps -eLf看了一下，java进程竟然有几万个线程！这下可以肯定是系统中某个地方会大量创建线程了。可是比较郁闷的是jstack老会出问题，在dump已经被JIT编译过的本地栈时，会抛出异常，导致完全看不到线程在干什么，卡在什么地方。这时想起来一招，这招在Web系统里很好用：写一个jsp，里面调用Thread.getAllStackTraces()这个方法来获取当前JVM里所有java线程的堆栈跟踪信息，然后输出到网页上。通过这个方法获得的堆栈跟踪不会受JIT编译的影响，很适合做Trouble Shooting用。

于是我们现写了一个这样的JSP页面，把它放到我们的一个Web Context里面，重启了系统。之后再测试，重现问题后马上访问这个JSP页面，果然找到了那些大量产生的线程：原来是java.util.Timer所创建的TimerThread线程，而且它们全都阻塞在Object.wait()里了。

查了一下Timer类的源代码，有如下发现：

1.  java.util.Timer类在构造方法里就会创建线程**并启动它**
2.  如果Timer中没有任务，TimerThread线程会阻塞在task queue的wait上

 所以能得出结论就是系统中有某个地方在大量创建Timer

## 4. 罪魁祸首

我们在代码里寻找所有引用了java.util.Timer的地方，把代码仔细阅读了几遍，没有发现问题。这些Timer都是静态的或者在单例中，没有大量创建的可能性。我们不得不把目光放到第三方包里。

我们这个项目因为用到了Sip协议，所以引用了一个叫jain-sip-ri的第三方Sip协议栈实现。它是此项目中新增的功能所引用的库，在早先的产品里是不存在的，而我们之前也从未出现过类似的问题，所以就把怀疑的目光投向了它：jain-sip-ri

我们写了一段测试代码，直接调用我们代码里使用了jain-sip-ri的部分，果然重现了大量创建线程的问题，于是用btrace跟踪了一下所有调用了java.util.Timer构造方法的地方，脚本如下：

```java
@BTrace public class NewTimer {

    @OnMethod(
      clazz="java.util.Timer",
      method=""
    )
    public static void onnew() {
        println("----------------------------------------");
        jstack();
    }

}
```

运行后的结果如下：

```
java.util.Timer.(Timer.java:106)
    gov.nist.javax.sip.stack.BlockingQueueDispatchAuditor.start(BlockingQueueDispatchAuditor.java:25)
    gov.nist.javax.sip.stack.UDPMessageProcessor.(UDPMessageProcessor.java:134)
    gov.nist.javax.sip.stack.OIOMessageProcessorFactory.createMessageProcessor(OIOMessageProcessorFactory.java:46)
    gov.nist.javax.sip.stack.SIPTransactionStack.createMessageProcessor(SIPTransactionStack.java:2372)
    gov.nist.javax.sip.SipStackImpl.createListeningPoint(SipStackImpl.java:1371)
    gov.nist.javax.sip.SipStackImpl.createListeningPoint(SipStackImpl.java:1537)
    com.workssys.share.oma.SipLayer.createPrivider(SipLayer.java:290)
```

根源就在这个BlockingQueueDispatchAuditor里了，于是下载了一份源代码，找到了相应的方法，如下：

```java
public void start(int interval) {
   if(started) stop();
   started = true;
   timer = new Timer();
   timer.scheduleAtFixedRate(this, interval, interval);
}
```

每次启动的时候BlockingQueueDispatchAuditor都会创建一个Timer。继续顺着stack trace定位上层的代码，有如下发现：

1.  每次createListeningPoint调用，都会导致一个新的MessageProcessor被创建
2.  createListeningPoint后续的动作如果失败了，不会销毁创建出来的MessageProcessor（至少没有cancel Timer）

进一步查看我们自己的代码，发现有一处循环调用createListeningPoint！原来绑定Sip端口的逻辑是从一个起始端口开始不断尝试绑定，失败的话就让端口加1，直到成功为止。虽然这里可能会出现死循环，但通常是不应该出现几百上千个连续端口全都被占用的情况才对。分析到这，我们同事终于想起来一件事：原来他之前为了测试，把绑定IP改成了一个硬编码的值，后来不小心把这个改动commit了。测试Server的IP必然和他之前用的IP不一样，无论用什么端口都必然绑定不上，所以那段代码会无限循环，无限调用createListeningPoint，最后因为jain-sip里的小漏洞而无限创建Timer对象和TimerThread线程，进而导致了这一连串离奇的现象。

## 5. 结论

我觉得找这个问题的过程挺狗血的，唯一有些价值的我认为是以下几点：

1.  64位OS上运行的32位进程，地址空间只有4G，其实这个结论是显而易见的，但容易被忽略
2.  Java Web系统里可以用一个小JSP页面来调用Thread.getAllStackTraces生成所有线程的状态，这是替代jstack的一个很不错的一招！
3.  btrace是个好东西，仅仅是为了这个工具而将运行环境迁移到Java 6上都是很值得的！
4.  最好不要为测试去改被测试的代码，如这里提到的把IP改成硬编码的本机IP，更好的办法应该是提高代码的“可测试性”

## 6. 插曲

这篇文章写出来有两天了，但再最后发出来之前的一刻，发现之前的分析有错！下面是原始的**错误分析**：

<del>我Download下来这个包的源代码，在里面搜索Timer，结果找到了一个叫DefaultSipTimer的类，正好就是java.util.Timer的子类。接着搜索引用了这个类的地方，找到了一个叫SipStackImpl的类里的两处引用，一处是在构造函数里：</del>

```java
String defaultTimerName = configurationProperties.getProperty("gov.nist.javax.sip.TIMER_CLASS_NAME",DefaultSipTimer.class.getName());
try {
    setTimer((SipTimer)Class.forName(defaultTimerName).newInstance());
    getTimer().start(this, configurationProperties);
    if (getThreadAuditor().isEnabled()) {
        // Start monitoring the timer thread
        getTimer().schedule(new PingTimer(null), 0);
    }
} catch (Exception e) {
    logger.logError("Bad configuration value for gov.nist.javax.sip.TIMER_CLASS_NAME", e);
}
```

<del>另一处在一个叫reInitialize的方法：</del>

```java
if(!getTimer().isStarted()) {
    String defaultTimerName = configurationProperties.getProperty("gov.nist.javax.sip.TIMER_CLASS_NAME",DefaultSipTimer.class.getName());
    try {
        setTimer((SipTimer)Class.forName(defaultTimerName).newInstance());
        getTimer().start(this, configurationProperties);
        if (getThreadAuditor().isEnabled()) {
            // Start monitoring the timer thread
            getTimer().schedule(new PingTimer(null), 0);
        }
    } catch (Exception e) {
        logger.logError("Bad configuration value for gov.nist.javax.sip.TIMER_CLASS_NAME", e);
    }
}
```

<del>仔细想一下就能发现这里有个严重的漏洞，如果Timer还没有被启动，就会在这里被重新创建，但同时原来的Timer没有被Cancel掉，其实还是存在的，对应的TimerThread线程也还在跑。所以只要因为某个原因让reInitialize这个方法被不断调用，就会发生无限创建线程的状况！</del>

看一下上面的代码能发现，reInitialize里其实已经检查了SipTimer是否已经启动，只有未启动的时候才会重新创建Timer，而这两处都有调用start，即这里不可能无限创建Timer。我是在发出文章的前一刻看了一眼代码才突然发现这个问题。结论是：一定要细心啊！
