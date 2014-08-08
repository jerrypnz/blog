---
layout: post
title: Java 服务端监控方案（三. Nagios 篇）
date: 2014-07-22 14:30:25 +0800
comments: true
categories: Java
tags: Java, 监控, Linux, Ganglia, Nagios
keywords: Java, Monitor, Linux, Server, Ganglia, Nagios
description: 服务端应用的重要一环是监控，本文介绍 Java 的服务端监控解决方案，基于 Ganglia，Nagios 和 JMX。
---

最近在为可能改变人生的一件事在努力，所以没怎么有空更新博客，无论如何今
天还是抽空把这个系列的第三篇补上吧。

[上一次]({% post_url 2014-07-04-server-side-java-monitoring-ganglia %})
我介绍了我们监控方案中的核心：Ganglia，而这次我们继续介绍用于告警的
Nagios 系统，以及如何让 Nagios 使用 Ganglia 来作为告警的数据源。

[Nagios](http://www.nagios.org/) 原本名叫 NetSaint，是由 Ethan Galstad
和其他一些开发者实现和维护的。它跨平台，可以在主流的所有 UNIX 类操作系
统上运行。Nagios 提供一个基于 PHP 和 CGI 的 Web 界面，并包含很多插件，
用于监控各种不同的网络服务以及网络设备。Nagios 的基本工作方式就是定期
检查用户配置的 host 上的 service，如果发现异常就会产生告警（WARN 或者
CRITICAL 级别），并使用邮件或者短信来通知管理员。

正常的 Nagios 系统需要在被监控的机器上安装
[NPRE](http://exchange.nagios.org/directory/Addons/Monitoring-Agents/NRPE--2D-Nagios-Remote-Plugin-Executor/details)
，用于远程执行指令来获取监控数据。但我们的方案里已经有 Ganglia 来收集
监控数据了，完全可以省略 NPRE，直接从 Ganglia 中获取数据。

下面介绍 Nagios 的安装和配置，以及和 Ganglia 的整合。

<!--more-->

## 1. 安装和运行

Nagios 在 Redhat 的 EPEL 的源里有，但版本比较老（3.5.x），因此我们选择
自行下载源码，编译安装。最新版本的 Nagios 4.x 的源码可以从这里下载：
http://sourceforge.net/projects/nagios/files/nagios-4.x/ 。

### 1.1 编译安装 nagios-core

首先要建立必要的用户和组：

```sh
sudo groupadd nagios
sudo useradd -g nagios nagios
sudo gpasswd -a nobody nagios
```

我们的 Nginx 和 fcgiwrap 的用户是 nobody，所以把它加到了 nagios 组，以
便运行其 CGI 程序。如果使用 Apache 作为 Web Server，则将 apache 用户加
入到 nagios 组即可。

解压源码后进入源码目录，执行下面的命令来编译安装：

```sh
./configure --with-command-group=nagios
make
sudo make install
```

nagios 会被安装到 `/usr/local/nagios` 下，其配置文件在 `etc` 子目录下，
CGI 在 `sbin` 下，PHP 和 HTML 以及静态资源在 `share` 下。

接下来启用 nagios 服务：

```sh
sudo chkconfig --add nagios
sudo chkconfig --level 35 nagios on
```

### 1.2 配置 Nginx

Nagios 自带了 Apache 的配置，可以直接安装好并启用即可。我们使用的
Nginx，相比之下要麻烦很多。首先 Nginx 并不直接支持 CGI，只支持 FastCGI，
因此需要使用 `fcgiwrap` 这个工具。具体的配置这里不再详细叙述，可以参考
[一些网上的教程](http://www.howtoforge.com/serving-cgi-scripts-with-nginx-on-centos-6.0-p2)
。

我们的 Nagios 和 Ganglia 运行在同一台服务器上，且共享了同一个域名，所
以配置放到一起了。其中 `monigor.foobar.com` 是 Ganglia 的地址，而
`monitor.foobar.com/nagios` 则是 Nagios 的地址。

完整配置如下，其中 location 中包含 `/nagios` 的是 Nagios 相关配置：

```nginx
server {
     listen 80;
     server_name monitor.foobar.com;
     charset utf8;
     access_log logs/$host.access.log main;
     root /usr/local/www/ganglia;
     index index.php index.html index.htm;
     include mime.server.common;
     auth_digest_user_file /usr/local/www/ganglia/users.htdigest;

     location ~ ^/nagios/(.*\.php)$ {
        set $phpfile $1;
        alias /usr/local/nagios/share/$phpfile;
        auth_digest 'Foobar Monitor System';
        auth_digest_timeout 60s;
        auth_digest_expires 3600s;
        if ($http_authorization ~ username="([^\"]+)") {
            set $htdigest_user $1;
        }
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME  $request_filename;
        fastcgi_param AUTH_USER $htdigest_user;
        fastcgi_param REMOTE_USER $htdigest_user;
        include fastcgi.conf;
     }

    location ~ .*\.(php|php5)?$ {
        auth_digest 'Foobar Monitor System';
        auth_digest_timeout 60s;
        auth_digest_expires 3600s;
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) {
                return 404;
        }
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi.conf;
        fastcgi_param ganglia_secret super-secret;
        if ($http_authorization ~ username="([^\"]+)") {
            set $htdigest_user $1;
        }
        fastcgi_param  REMOTE_USER    $htdigest_user;
     }

     location /nagios {
        alias /usr/local/nagios/share;
        index index.php index.html index.htm;
     }

     location /nagios {
        alias /usr/local/nagios/share;
        index index.php index.html index.htm;
     }

     location /nagios/cgi-bin/ {
        alias /usr/local/nagios/sbin/;
        auth_digest 'Foobar Monitor System';
        auth_digest_timeout 60s;
        auth_digest_expires 3600s;
        if ($http_authorization ~ username="([^\"]+)") {
            set $htdigest_user $1;
        }
        fastcgi_param SCRIPT_FILENAME  $request_filename;
        fastcgi_param AUTH_USER $htdigest_user;
        fastcgi_param REMOTE_USER $htdigest_user;
        include fastcgi.conf;
        fastcgi_pass unix:/var/run/fcgiwrap.sock;
     }
}
```

如果你打算为 nagios 单独设置一个域名，且 path 不是 /nagios，你还需要修
改 `/usr/share/nagios/etc/cgi.cfg` 文件里的 `url_html_path` 路径为 `/`，
同时 `/usr/share/nagios/share` 里的 `config.inc.php` 也需要一些修改：

```php
$cfg['cgi_base_url']='/cgi-bin'; //默认是 /nagios/cgi-bin
```

### 1.3 运行

启用 Nagios 服务即可：

```sh
sudo service nagios start
```

配置好 Nginx 后重启或者重载配置文件，确保 fcgiwrap 正常运行，访问
`monitor.foobar.com/nagios` 即可看到 Nagios 的运行效果。

## 2. Nagios 配置

Nagios 的配置文件都在 `/usr/local/nagios/etc` 下，主配置文件是
`nagios.cfg`，其中可以通过 `cfg_file` 配置项包含其他配置文件。这些子配
置文件一般都放在 `objects` 子目录下，按照职责分开，比如 `hosts.cfg` 配
置要监控的主机，`contacts.cfg` 是联系人，`commands.cfg` 是命令配置等等。

```cfg
cfg_file=/usr/local/nagios/etc/objects/commands.cfg
cfg_file=/usr/local/nagios/etc/objects/contacts.cfg
cfg_file=/usr/local/nagios/etc/objects/timeperiods.cfg
cfg_file=/usr/local/nagios/etc/objects/templates.cfg
```

Nagios 的配置提供了模板机制，可以把共同的配置项抽取成一个个模版，然后
在定义实际的对象时可以继承这些模板。默认的配置文件已经定义了各种模板，
都在 `templates.cfg` 里。

下面的这些配置都是在默认配置的基础上修改的，并且使用了默认的一些模板。
另外我在 `nagios.cfg` 中把 `localhost.cfg` 给禁用掉了，因为后面和
Ganglia 整合后，不需要再对本机做监控，而是全部基于 Ganglia 的数据来实
现。

```cfg
#cfg_file=/usr/local/nagios/etc/objects/localhost.cfg
```

### 2.1 Host 和 Host Group 配置

首先是主机和组的配置。首先是主机配置：

```cfg
define host{
   use          linux-server         #使用linux-server模板
   host_name    mysql2.foobar.com    #主机名
   alias        MySQL Slave Server 1 #别名
   address      192.168.1.9          #IP地址
   hostgroups   all-servers     #所属组，可以指定多个，逗号隔开
   }
```

然后是主机组配置：

```cfg
define hostgroup{
   hostgroup_name  mysql-slaves                #组名
   alias           MySQL Slaves                #组别名
   members         mysql2.foobar.com,mysql3.foobar.com #组成员
   }
```

可以看到，主机和组的归属关系，即可在 `host` 处指定，也可以在
`hostgroup` 处指定。前者使用成员很多的组，后者适合成员较少的组。我这里
定义了一个叫 `all-servers` 的组，包含所有主机，比较方便用于配置所有服
务器上都需要监控的 `service`，比如 Load、磁盘、内存等。

### 2.2 Command 配置

Command 是 Nagios 实际检查一项服务时要执行的命令。命令可以是 Nagios 内
置的插件提供，第三方插件提供，甚至可以自己写脚本实现。这里列举的是我们
检查 MySQL Slave 状态的一个 Command。

```cfg
define command{
 command_name  check_mysql_slave #指令名称
 #命令行
 command_line /usr/local/nagios/plugins/check_mysql_slavestatus.sh -H $HOSTNAME$ -P 3306 -u xxx -p xxx -w $ARG1$ -c $ARG2$
}
```

这个脚本是网上找的一个第三方插件，其实就是一个 Shell 脚本。可以看到
`command_line` 定义里有一些变量： `$HOSTNAME$`，`$ARG1`，`$ARG2$`。到
Nagios 实际执行的时候，它们都会被替换成实际的值。其中 `$HOSTNAME$` 会
被替换成主机名，`$ARG1` 和 `$ARG2` 则会被替换成 service 定义处指定的参
数。在这个例子中，这两个参数用于指定 MySQL Slave 的 Seconds Behind
Master 的告警阈值，`$ARG1` 是 WARN 阈值，`$ARG2` 是 CRITICAL 阈值。

### 2.3 Service 配置

Service 是最核心的配置，它用于定义某个 host 或 host group 上要检查哪些
服务，使用什么指令来检查。以上面的 MySQL Slaves 为例：

```cfg
define service{
   use                    generic-service    #使用的模板
   hostgroup_name         mysql-slaves       #要检查的主机组
   service_description    MySQL Slave Status #服务描述
   check_command          check_mysql_slave!1200!2400 #命令
   }
```

其中如果使用 `hostgroup_name` 的话，则针对一个主机组，也可以使用
`host_name` 来指定单个主机。

`check_command` 指定命令和参数，用 `!` 分隔。对于这个例子，我们指定
Slave 落后于 Master 的时间在 1200 秒以上则发出 WARN 级别告警，在 2400
秒以上则发出 CRITICAL 级别告警。

上面差不多就是 Nagios 的主要配置了，可以看到还是很简单的。

### 2.4 Contact 配置

Contact 是联系人配置，在 `contacts.cfg` 文件里，主要是用于设置用户和用
户组。

```cfg
define contact{
   contact_name                    jerry
   use                             generic-contact
   alias                           Jerry Peng
   email                           jerry.peng@foobar.com
   }
```

其中最重要的就是 `email` 配置了，这个决定了你能不能收到 Nagios 的告警
邮件。

联系人组方面，我们没有进行细分，仅仅使用了默认的 `admins` 组，默认的服
务模板 `generic-service` 里定义的联系人组也是它，因此这个组里的用户能收
到所有告警邮件。

```cfg
define contactgroup{
   contactgroup_name       admins
   alias                   Nagios Administrators
   members                 jerry,drizzt
   }
```

如果你们的服务比较多，希望不同的管理员负责不同的服务，你可以多定义几个
组，并且在定义 `service` 的时候使用 `contact_groups` 配置项来指定联系
人组。

## 3. 和 Ganglia 的整合

我们的监控方案是以 Ganglia 为核心，因此 Nagios 的告警也主要基于
Ganglia 的数据来实现（当然有些类型的监控直接通过 Nagios 来做更简单，比
如上面举的 MySQL Slave 监控）。两者的整合方式有很多，Ganglia 的
[Github WIKI](https://github.com/ganglia/monitor-core/wiki/Integrating-Ganglia-with-Nagios)
上有一个总结。

我们采用的是第一个方案：
[Ganglia Web Nagios Script](https://github.com/ganglia/ganglia-web/wiki/Nagios-Integration)
。这个方案是 Ganglia 内置的，装好就有，比较简单方便。它是通过 Ganglia
Web 中的一个 PHP 脚本，外加一个 Bash 脚本实现的。原理很简单，Bash 脚本
通过 curl 访问 Web 系统的 PHP 脚本，传入要检查的参数和主机，以及告警阈
值即可。

Command 定义如下：

```cfg
define command{
 command_name  check_ganglia_metric
 command_line  /bin/sh /usr/local/www/ganglia/nagios/check_ganglia_metric.sh host=$HOSTNAME$ metric_name=$ARG1$ operator=$ARG2$ critical_value=$ARG3$
}
```

其中的三个参数分别为要监控的 metric 名称，操作符（more/less），
CRITICAL 阈值。

下面是使用这个 Command 来定义的若干 Service：

- 所有主机的根分区磁盘空间监控，剩余空间小于 20G 时告警：

```cfg
define service{
  use                             generic-service
  hostgroup_name                  all-servers
  service_description             Root Partition Free Space
  check_command                   check_ganglia_metric!root_disk_free!less!20
  }
```

- 所有主机的一分钟 loadavg 检查，大于 16 时告警（我们的都是16核以上的机器）：

```cfg
define service{
   use                             generic-service
   hostgroup_name                  all-servers
   service_description             Load One
   check_command                   check_ganglia_metric!load_one!more!16
   }
```

- 一个 Java 应用的性能参数告警，核心引擎的处理时间超过两秒时告警（该参
  数通过 JMX 暴露出来，使用 jmxtrans发送给 Ganglia，后续章节会对此做详
  细介绍）：

```cfg
define service{
   use                             generic-service
   host_name                       engine1.foobar.com
   service_description             Engine Process Time
   check_command                   check_ganglia_metric!engine1.ProcessTime!more!2000
   }
```

可以看到，任何 Ganglia 中的 Metric 都可以用这样方式拿来做告警，十分方
便。

## 3. 总结

至此，我们的监控方案基本上成形了，通过这个方案，我们既可以查看某个监控
参数的变化情况，也可以针对它们做告警机制。

写了几篇了，与 Java 有关的事情都还没影，的确有点标题党的嫌疑了。不要急，
有了这些基础系统，剩下的与 Java 有关的事情就简单了，无非是想法办法记录
应用内的一些监控参数，并整合到 Ganglia 里而已。下一次我就详细对这个进
行介绍，并分享我们的经验。

### 系列文章导航

- [Java 服务端监控方案（一. 综述篇）]({% post_url 2014-06-19-server-side-java-monitoring %})
- [Java 服务端监控方案（二. Ganglia 篇）]({% post_url 2014-07-04-server-side-java-monitoring-ganglia %})
- [Java 服务端监控方案（四. Java 篇）]({% post_url 2014-08-08-server-side-java-monitoring-java %})
