---
layout: post
title: Java 服务端监控方案（二. Ganglia 篇）
date: 2014-07-04 09:27:25 +0800
comments: true
categories: Java
tags: Java, 监控, Linux, Ganglia
keywords: Java, Monitor, Linux, Server, Ganglia
description: 服务端应用的重要一环是监控，本文介绍 Java 的服务端监控解决方案，基于 Ganglia，Nagios 和 JMX。
---

[上一篇文章]({% post_url 2014-06-19-server-side-java-monitoring %})综述
了我们的服务端监控方案，本文则继续介绍我们的监控方案的核心：Ganglia。

Ganglia 是加州大学伯克利分校发起的系统监控项目，为大规模高性能计算集群
而设计，采用了 RRD、XML 等成熟的技术实现，单节点开销很小，提供了相当靠
谱的容错机制，且很容易扩展和加入自定义的 Metric。Ganglia 的一个比较有名
的用户是维基百科，可以通过
[这里](https://ganglia.wikimedia.org/latest/) 访问他们的 Ganglia 实例，
从上面可以看到维基百科的集群运行状况。

## 1. 安装

我们的服务器环境是 Redhat Enterprise Linux 6.4 x86_64 版，本文基于这个发行版来介
绍安装和配置。其他发行版或者架构也都大同小异。

<!--more-->

Ganglia 主要包含三个组件：gmond、gmetad 和 ganglia-web。其中 gmond 是核
心进程，负责发送和接收 metric，每个相关的节点上都需要安装 gmond。
gmetad 是聚合 metric 数据的服务，一般在一台机器上安装即可，如果要考虑高
可用性，也可以安装到多台机器上。ganglia-web 则是基于PHP 的 Web 界面，一
般只在一台机器上安装即可。

默认 gmond 是按多播的方式工作的，这样同一个网络内的每一个 gmond 都能接
收到整个集群所有节点的 metric 数据。这样任何一个监控节点挂了，都可以切
换到其他任意存活节点上。不过我们的集群规模较小，没有采取这个方案，而是
采用单播的方式，由一个节点接收所有数据，也只有这个节点上安装了 gmetad
和 ganglia-web。这样做比较简单，且实践中稳定性完全足够了。

因此，按照部署方式，gmetad 可能是不需要的，下面的步骤中关于 gmetad 的
部分可以忽略。

### 1.1 安装依赖

首先确保下面的包都正确安装了，这些都能在官方源里有（如果没有购买 RHEL
的授权，可以配置使用 CentOS 的源，也一样能用）。

```sh
 yum -y install apr-devel apr-util check-devel cairo-devel \
   pango-devel libxml2-devel rpmbuild glib2-devel \
   dbus-devel freetype-devel fontconfig-devel gcc-c++ \
   expat-devel python-devel libXrender-devel pcre-devel \
   perl-ExtUtils-MakeMaker
```

其次需要安装 `libconfuse`，这个库虽然在源里有，但版本貌似比较老，我是
通过 [RPM find](http://rpm.pbone.net/) 找到 RPM 直接安装的。注意
[libconfuse](http://rpm.pbone.net/index.php3/stat/4/idpl/19571009/dir/redhat_el_6/com/libconfuse-2.7-4.el6.x86_64.rpm.html)
和
[libconfuse-devel](http://rpm.pbone.net/index.php3/stat/4/idpl/19571011/dir/redhat_el_6/com/libconfuse-devel-2.7-4.el6.x86_64.rpm.html)
都需要安装，因为编译 Ganglia 时也需要使用。

最后需要安装 [rrdtools](http://oss.oetiker.ch/rrdtool/pub/?M=D)，这个
源里没有，只能自己编译安装。在其官网下载最新版本的源码，解压后按下面的步
骤编译安装即可。

```sh
./configure --prefix=/usr
make
sudo make install
```

### 1.2 编译 Ganglia

从 Ganglia 官网下载
[最新版本的源码](http://sourceforge.net/projects/ganglia/files/ganglia%20monitoring%20core/3.6.0/)
，解压后按照下面的步骤编译安装：

```sh
./configure --with-gmetad
make
sudo make install
```

按这个配置 Ganglia 会被安装到 `/usr/local` 下，其配置文件在
`/usr/local/etc` 下。

安装好以后强烈建议把 Ganglia 的两个组件 `gmond` 和 `gmetad` 的服务脚本
拷贝到 `/etc/init.d` 下，方便使用 `service` 和 `chkconfig` 来管理
Ganglia 的服务。这两个脚本位于源码目录下的 `gmond/gmond.init` 和
`gmetad/gmetad.init`。拷贝之前注意修改这两个脚本里的 `GMOND` 和
`GMETAD` 两个环境变量的值为正确的可执行文件路径，默认它们都在
`/usr/sbin` 下，要修改成 `/usr/local/sbin`。

```sh
sudo cp gmond/gmond.init /etc/init.d/gmond
sudo cp gmetad/gmetad.init /etc/init.d/gmetad
```

## 2. 配置

注意这里不会提到所有的配置项，而只是列出了关键的部分。实际操作时，请直接在
默认配置文件的基础上修改，里面都有详细的注释对配置项作出了解释。

### 2.1 gmond 配置

gmond 配置文件在 `/usr/local/etc/gmond.conf` 。

cluster 名称的配置，主要是 name 和 owner，自定义即可。同一个集群下所有
的 gmond 都要配置成一样的。

```
/*
 * The cluster attributes specified will be used as part of the <CLUSTER>
 * tag that will wrap all hosts collected by this instance.
 */
cluster {
  name = "my-cluster"
  owner = "jerry"
  latlong = "unspecified"
  url = "unspecified"
}
```

host 配置。注意这个只是给每个 host 取个名字而已，也可以随便命名，不过
建议直接实用主机名。

```
/* The host section describes attributes of the host, like the location */
host {
  location = "host1"
}
```

#### 多播模式配置

这个是默认的方式，基本上不需要修改配置文件，且所有节点的配置是一样的。
这种模式的好处是所有的节点上的 gmond 都有完备的数据，gmetad 连接其中任
意一个就可以获取整个集群的所有监控数据，很方便。

其中可能要修改的是 `mcast_if` 这个参数，用于指定多播的网络接口。如果有
多个网卡，要填写对应的内网接口。

```
/* Feel free to specify as many udp_send_channels as you like.  Gmond
   used to only support having a single channel */
udp_send_channel {
  bind_hostname = yes # Highly recommended, soon to be default.
                       # This option tells gmond to use a source address
                       # that resolves to the machine's hostname.  Without
                       # this, the metrics may appear to come from any
                       # interface and the DNS names associated with
                       # those IPs will be used to create the RRDs.
  mcast_join = 239.2.11.71
  mcast_if = em2
  port = 8649
  ttl = 1
}

/* You can specify as many udp_recv_channels as you like as well. */
udp_recv_channel {
  mcast_join = 239.2.11.71
  mcast_if = em2
  port = 8649
  bind = 239.2.11.71
  retry_bind = true
  # Size of the UDP buffer. If you are handling lots of metrics you really
  # should bump it up to e.g. 10MB or even higher.
  # buffer = 10485760
}
```


#### 单播模式配置

监控机上的接收 Channel 配置。我们使用 UDP 单播模式，非常简单。我们的集
群有部分机器在另一个机房，所以监听了 `0.0.0.0`，如果整个集群都在一个内
网中，建议只 bind 内网地址。如果有防火墙，要打开相关的端口。

```
/* Alternative UDP channel */
udp_recv_channel {
  bind = 0.0.0.0
  port = 8648
}
```

被监控的节点上的发送 Channel 配置。同样很简单，host 填写监控节点的 IP，
端口填写上面配置的监听端口即可。其中 ttl 设定为 1，因为我们是直接发送
到目标节点的，没有经过中间的 gmond 转发。

```
/* Feel free to specify as many udp_send_channels as you like.  Gmond
   used to only support having a single channel */
udp_send_channel {
  bind_hostname = yes # Highly recommended, soon to be default.
                       # This option tells gmond to use a source address
                       # that resolves to the machine's hostname.  Without
                       # this, the metrics may appear to come from any
                       # interface and the DNS names associated with
                       # those IPs will be used to create the RRDs.
  host = 192.168.221.9
  port = 8648
  ttl = 1
}
```

接下来是默认收集和发送的 metric 配置。这个保留默认即可。默认收集的
metric 包括：CPU、Load、Memory、Disk 和 Network，基本上涵盖了操作系统
级的所有参数。下面是一个配置片段，如果我们要添加自定义监控 metric，也
需要按这个格式来配置：

```
collection_group {
  collect_every = 40
  time_threshold = 180
  metric {
    name = "disk_free"
    value_threshold = 1.0
    title = "Disk Space Available"
  }
  metric {
    name = "part_max_used"
    value_threshold = 1.0
    title = "Maximum Disk Space Used"
  }
}
```

### 2.2 gmetad 配置

gmetad 的配置文件位于 `/usr/local/etc/gmetad.conf`。其中最重要的配置项
是 `data_source`:

```
data_source "my-cluster" localhost:8648
```

如果使用的是默认的 8649 端口，则端口部分可以省略。如果有多个集群，则可
以指定多个 `data_source`，每行一个。

最后是 `gridname` 配置，用于给整个 Grid 命名：

```
gridname "My Grid"
```

其他配置保留默认值即可。

### 2.3 ganglia-web 安装和配置

先从官网下载
[ganglia-web 的源码包](http://sourceforge.net/projects/ganglia/files/ganglia-web/3.5.12/)
并解压。
如果使用 Apache 作为 Web 服务器，可以修改 Makefile 中相关的变量并执行
`sudo make install` 来安装。

```sh
# Location where gweb should be installed to (excluding conf, dwoo dirs).
GDESTDIR = /var/www/html/ganglia

# Gweb statedir (where conf dir and Dwoo templates dir are stored)
GWEB_STATEDIR = /var/lib/ganglia-web

# Gmetad rootdir (parent location of rrd folder)
GMETAD_ROOTDIR = /var/lib/ganglia

# User by which your webserver is running
APACHE_USER =  apache
```

我们使用 Nginx 作为 Web Server，可以直接解压安装包到目标位置，修改相关
配置文件即可。要确保系统安装配置好了 Nginx 和 PHP。下面是一个样例
Nginx 配置：

```
server {
    listen 80;
    server_name monitor.xxx.com;
    charset utf8;
    access_log logs/$host.access.log main;
    root /var/www/ganglia;
    index index.php;
    include mime.server.common;
    auth_digest_user_file /var/www/ganglia/users.htdigest;
    
    location ~ .*\.(php|php5)?$ {
        auth_digest 'XXX Monitor System';
        auth_digest_timeout 60s;
        auth_digest_expires 3600s;
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) {
                return 404;
        }
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi.conf;
        fastcgi_param ganglia_secret yourSuperSecret;
        if ($http_authorization ~ username="([^\"]+)") {
            set $htdigest_user $1;
        }
        fastcgi_param  REMOTE_USER   $htdigest_user;
    }
}
```

请根据部署时的实际情况调整里面的相关参数如 ganglia-web 的路径，以及
PHP-FPM 的地址等。

注意我们是启用了认证的，使用的是 HTTP Digest 认证（需要安装第三方Nginx
模块）。使用自带的 HTTP Basic 认证也是可以的，但建议配合 HTTPS 使用以提
升安全性。

其中 `ganglia_secret` 和 `REMOTE_USER` 这两个 fastcgi 参数是用于控制
Ganglia 认证的，其中 `ganglia_secret` 是一个自定义的串，随便填写即可。

ganglia-web 本身也需要配置才能使认证生效。在 ganglia-web 的目录里新增一
个 `conf.php` 文件来写入配置：

```php
<?php

#
# 'readonly': No authentication is required.
#             All users may view all resources.
#             No edits are allowed.
# 'enabled': Guest users may view public clusters.
#            Login is required to make changes.
#            An administrator must configure an
#            authentication scheme and ACL rules.
# 'disabled': Guest users may perform any actions,
#             including edits. No authentication is required.
$conf['auth_system'] = 'enabled';

$acl = GangliaAcl::getInstance();

$acl->addRole('admin', GangliaAcl::ADMIN);

?>
```

其中 `admin` 用户被配置为管理员，你也可以根据需要为其他用户分配不同的
角色。

### 2.4 启动

用 `chkconfig` 增加系统服务，并启动 `gmond` 和 `gmetad` 服务：

```bash
sudo chkconfig --add gmond
sudo chkconfig --add gmetad
sudo service gmond start
sudo service gmetad start
```

启动系统以后，可以通过 Web 界面访问 Ganglia 查看图形。


## 3. 扩展

Ganglia 的扩展性很强，可以很方便使用其提供的一些机制来创建自定义的
metric，以收集应用数据等。它主要提供了以下方式：

1. Python 扩展
2. gmetric 命令

而 ganglia 的数据格式和协议是完全开放的，第三方应用也完全可以按照其格
式发送 metric 数据给 gmond。本系列文章后续的章节会详细介绍的 JMX 和
ganglia 的整合，用的就是这种方式。

Ganglia 官方也将一些
[第三方扩展收集到一起了](https://github.com/ganglia/gmond_python_modules)
，可以用来监控诸如redis、MySQL 等等应用，可以根据需要选用。

这里先介绍通过 Python 扩展的方式来解决 Ganglia 默认的磁盘监控的一点不足
之处。

### 3.1 磁盘监控的小改进

Ganglia 默认的 disk metric，针对的是系统所有的磁盘，它会把所有
磁盘总空间、可用空间加起来。这样当某个分区快满了，别的分区空闲很多的时
候，通过 ganglia 完全发现不了问题。

好在通过自定义扩展，可以很容易解决这个问题。我在网上找到了一个 ganglia
的 multidisk 扩展，并自己小小改进了一下。相关文件在这个
[gist](https://gist.github.com/moonranger/1854f9ef7f04d4e3a24e) 里。

把其中的 `multidisk.py` 文件放到
`/usr/local/lib64/ganglia/python_modules` 下（目录不存在的话先创建出
来），把 `multidisk.pyconf` 配置文件放到 `/usr/local/etc/conf.d` 下，
重启 gmond 即可。

其中配置文件可以根据需要加入分区。其中参数名是挂载点去掉开头的 `/` ，
并将中间的 `/` 替换成 `_` 后，再跟上 `_disk_total` 和 `_disk_free` 。
特例是根分区，名字是 `root_disk_free` 和 `root_disk_total`。这个是我做
出的小 hack，原本这个脚本使用的是设备名（`dev_sda1`）这种。但不同的机
器设备名不尽相同，有时候要统一监控某类分区时不方便。

下面是一个例子，监控根分区和 `/var` 分区。

```
modules {
  module {
    name = 'multidisk'
    language = 'python'
  }
}
 
collection_group {
  collect_every = 120
  time_threshold = 20
 
  metric {
    name = "root_disk_total"
    title = "Root Partition Total"
    value_threshold = 1.0
  }
 
  metric {
    name = "root_disk_free"
    title = "Root Partition Free"
    value_threshold = 1.0
  }
 
  metric {
    name = "var_disk_total"
    title = "Var Partition Total"
    value_threshold = 1.0
  }
 
  metric {
    name = "var_disk_free"
    title = "Var Partition Free"
    value_threshold = 1.0
  }
 
}
```

## 4. 总结

上面差不多把 Ganglia 基本的安装、配置和使用都介绍了一下。图形界面方面
没有介绍太多，但相信读者安装配置好以后，自己看看就能明白，或者去维基百
科的 Ganglia 实例上操作一下也行。

### 一些有用的链接

- [SETUP AND CONFIGURE GANGLIA-3.6 ON CENTOS/RHEL 6.3](http://sachinsharm.wordpress.com/2013/08/17/setup-and-configure-ganglia-3-6-on-centosrhel-6-3/)
- [Monitoring multiple clusters using Ganglia](http://jablonskis.org/2011/monitoring-multiple-clusters-using-ganglia/index.html)

### 系列文章导航

- [Java 服务端监控方案（一. 综述篇）]({% post_url 2014-06-19-server-side-java-monitoring %})
- [Java 服务端监控方案（三. Nagios 篇）]({% post_url 2014-07-22-server-side-java-monitoring-nagios %})
- [Java 服务端监控方案（四. Java 篇）]({% post_url 2014-08-08-server-side-java-monitoring-java %})
