---
title: 用正确的方式启动Java服务
author: Jerry
comments: true
layout: post
permalink: /2012/06/start-java-service-the-right-way/
categories:
  - Java
  - Tech
tags:
  - Java
  - Python
---
今天解决的一个问题，简单记录一下。

前些日子写了一套安装部署脚本，一直工作很正常，结果这些天增加一个小功能后出现了一个很怪异的问题，脚本会在特定的地方无限阻塞。新增的功能很简单，就是在安装好我们的系统以后将其启动。而经过观察，脚本就是在启动完系统之后阻塞的。

我们的脚本是这样一个调用关系：

main.py &#8211;> start.sh &#8211;> java

阻塞是发生在main.py脚本里的，它用subprocess模块来启动start.sh并捕获其输出，但发生阻塞时，start.sh已经结束了。这让人很迷惑，于是用strace来观察main.py到底在干什么，结果是它阻塞在read上了，对应于Python代码的Popen.readline方法里：

<pre lang="python">process = subprocess.Popen(args,
                           stdout=subprocess.PIPE,
                           stderr=subprocess.STDOUT,
                           env=dup_env,
                           close_fds=True)
try:
    while True:
        line = process.stdout.readline()
        if not line:
            break
        line = line.rstrip()
        logging.info('   %s' % line)
    process.wait()
    if process.returncode != 0:
        raise PackageError('Phase %s of package %s returned non-zero: %s' %
                (phase, self.name, process.returncode))
except KeyboardInterrupt:
    logging.error('&gt;&gt;&gt; Terminating...')
    os.kill(process.pid, signal.SIGINT)
    sys.exit(-1)
</pre>

于是进入/proc/PID/fd/里面察看这个fd，发现是一个管道：

<pre>[root@PN1 fd]# ls -lh
total 0
lr-x------ 1 root root 64 Jun  1 14:11 0 -&gt; /tmp/tmp.YdbnE13625
lrwx------ 1 root root 64 Jun  1 14:11 1 -&gt; /dev/pts/0
lrwx------ 1 root root 64 Jun  1 14:13 2 -&gt; /dev/pts/0
l-wx------ 1 root root 64 Jun  1 14:11 3 -&gt; /var/log/tms_install.log
lr-x------ 1 root root 64 Jun  1 14:11 4 -&gt; pipe:[47565576]
</pre>

但是start.sh都结束了，这个管道应该关闭了，对应的那句readline调用应该返回空才对。于是用lsof来察看一下，真相大白：

<pre>[root@PN1 fd]# lsof | grep 47565576
java      13324     root    2w     FIFO        0,6      47565576 pipe
java      13353     root    2w     FIFO        0,6      47565576 pipe
python    13627     root    4r     FIFO        0,6      47565576 pipe
java      13828     root    2w     FIFO        0,6      47565576 pipe
java      13870     root    2w     FIFO        0,6      47565576 pipe
java      13968     root    2w     FIFO        0,6      47565576 pipe
</pre>

原来是start.sh启动的java进程还打开着那个管道。查看了一下start.sh，里面是直接用java xxxx来启动系统的，这样java进程就继承了start.sh打开的文件描述符，其中stdout正好就是main.py调用start.sh时，subprocess.Popen所开启的管道。虽然start.sh已经结束了，但是管道还没有被关闭，加上java进程又是个服务，会一直运行且不会实际地往stdout上写数据，main.py就永远阻塞在read调用里了。

知道了这个问题，就要考虑解决办法。关键问题就是要处理好stdin，stdout和stderr，要将其正确重定向，除此之外，还要考虑禁止HUP信号等问题。Google了一番，有这些办法解决问题：

*   用nohup配合重定向来启动Java进程
*   使用Apache Commons Daemon（<http://commons.apache.org/daemon/>）
*   可以使用start-stop-daemon等标准daemon工具来启动Java服务

nohup比较简单直接，配合重定向就可以达到需要的效果。最终这个问题我也是采用nohup来解决的。