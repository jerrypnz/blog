---
title: Cron 导致的乱码问题
author: Jerry
comments: true
layout: post
permalink: /2013/08/mojibake-caused-by-cron/
categories:
  - UNIX/Linux
  - 技术
tags:
  - cron
  - crontab
  - LANG
  - Linux
  - 乱码
---

几天前为公司的生产环境写了个应用内存监控工具，定时通过 jstat 查看当前机
器上所有的 Tomcat 进程的 GC 信息，发现 Eden 和 Old 占用持续超过一定阈值
时自动重启相关进程（我知道这方案很山寨，求别黑……）。部署上后看似很
Happy，让服务质量有所提升。

但好景不长，没过多久，多个系统纷纷反映出现了奇怪的乱码问题，而这些系统
都是被这个监控脚本重启过的。这个监控脚本是通过 crontab 定期执行的，联想
到以前遇到过的 crontab 下 PATH 不对导致的 command not found 问题，我不
禁把怀疑的目光再次投向了 cron 这货。一番 Google 后果不其然，很多人都遇
到了同样的问题（Google 出来的第一页记录有几个日文网页，CJK 伤不起啊），
而原因其实都是一样的： **cron 会使用一个最小化的环境来执行任务。**

<!--more-->

可以用 cron 来执行一下 env 来验证这一点：

```sh
*/1 * * * * /usr/bin/env > /tmp/cronenv
```

在我的机器（Gentoo 64位，vixie-cron）上输出的结果是这样的：

```sh
[~]$ cat /tmp/cronenv
SHELL=/bin/sh
USER=jerry
PATH=/usr/bin:/bin
PWD=/home/jerry
SHLVL=1
HOME=/home/jerry
LOGNAME=jerry
_=/usr/bin/env
```

可以看到里面只有几个最基本的环境变量，语言相关的 LANG，LC_* 那一堆环境
变量都未定义，因此，通过 cron 运行的任务所使用的语言都是默认值，即
POSIX，而这会造成很恶心的乱码问题（对 Java 程序而言问题尤其严重，因为在
Linux 上 Java 的默认 charset 就是由 LANG 等环境变量决定的）。

解决办法有这些：

   - 在 `/etc/environment` 里定义 `LANG` 等关键的环境变量
   - 在 crontab 文件的头部定义环境变量
   - 如果执行的任务是个脚本，在脚本头部用 `export` 定义并导出环境变量

其中第一个方法依赖 PAM 的 pam_env.so 模块，需要 cron 的实现支持 PAM 且
配置文件里启用了此模块。我试验了下，在 Redhat Enterprise Linux 和
CentOS 上是可以工作的，但在我的 Gentoo 上不工作——即使我手工在
`/etc/pam.d/cron` 里增加 pam_env.so 也不管用，尚不知道原因何在（我用的
是 vixie-cron，且启用了 pam USE flag，应该是支持 PAM 的）。所以第一个
方法不保证在所有的发行版上都工作，使用前最好验证一下。

个人认为第二个方法和第三个方法比较靠谱，尤其是第二个，基本上所有的
cron 实现都支持在 crontab 里写环境变量：

```sh
LANG=zh_CN.UTF-8
*/1 * * * * /usr/bin/env > /tmp/cronenv
```

这样修改一下 crontab 就能看到，此处声明的环境变量生效了：

```sh
[~/tmp/vixie-cron-4.1]$ cat /tmp/cronenv
SHELL=/bin/sh
USER=jerry
PATH=/usr/bin:/bin
PWD=/home/jerry
LANG=zh_CN.UTF-8
SHLVL=1
HOME=/home/jerry
LOGNAME=jerry
_=/usr/bin/env
```

不太理解为什么 cron 会这样设计，感觉 `/etc/profile` 和 `~/.profile` 是
标准的环境变量机制，为什么 cron 不会加载它们。这是处于安全性的考虑还是
性能考虑呢？我 Google 了一圈也没找到一个合适的答案，还是以后再探索吧。

P.S. 突然很好奇主要的几个 Linux init 实现如 sysvinit，openrc，upstart
等是怎么解决环境变量的问题，印象中系统服务没遇到过语言相关的问题，所以
环境变量应该是被正确设置了的。有时间研究下去。
