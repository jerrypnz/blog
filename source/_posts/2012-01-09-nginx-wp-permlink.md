---
title: 在nginx上配置WordPress的Pretty Permlink
author: Jerry
comments: true
layout: post
permalink: /2012/01/nginx-wp-permlink/
categories:
  - Tech
tags:
  - Blog
  - nginx
  - Wordpress
---
WordPress的pretty permlink（不带index.php）只支持Apache，且需要mod_rewrite，用别的Web服务器时只能用PATHINFO风格的permlink，里面有一个难看的index.php。这篇文章介绍如何在nginx下配置出真正的pretty permlink来。

## 1. 插件：nginx Compatibility

如果Wordpress检测不到mod\_rewrite的存在，那么配置的permlink里都有index.php部分。nginx Compatibility插件可以欺骗Wordpress，让其以为mod\_rewrite存在，从而可以配置pretty permlink。此插件还做了一些其他的针对nginx的兼容性修正。

The plugin solves two problems:

1.  When WordPress detects that FastCGI PHP SAPI is in use, it <a href="http://blog.sjinks.pro/wordpress/510-wordpress-fastcgi-and-301-redirect/" rel="nofollow">disregards the redirect status code</a> passed to `wp_redirect`. Thus, all 301 redrects become 302 redirects which may not be good for SEO. The plugin overrides`wp_redirect` when it detects that nginx is used.
2.  When WordPress detects that `mod_rewrite` is not loaded (which is the case for nginx as it does not load any Apache modules) it falls back to <a href="http://codex.wordpress.org/Using_Permalinks#PATHINFO:_.22Almost_Pretty.22" rel="nofollow">PATHINFO permalinks</a> in Permalink Settings page. nginx itself has built-in support for URL rewriting and does not need PATHINFO permalinks. Thus, when the plugin detects that nginx is used, it makes WordPress think that `mod_rewrite` is loaded and it is OK to use pretty permalinks.

插件的主页在此：<http://wordpress.org/extend/plugins/nginx-compatibility/>

## 2. nginx配置

关键的配置其实就一句话：

<pre lang="c">try_files $uri $uri/ /index.php;
</pre>

这样nginx会先尝试$uri或者$uri/，如果这两个资源都不存在，就会转到/index.php。

下面是我的nginx配置：

<pre lang="c">server
{
    listen       80;
    server_name jerrypeng.me www.jerrypeng.me;
    index index.html index.htm index.php;
    root  /home/wwwroot/wordpress;

    location /
    {
        try_files $uri $uri/ /index.php;
    }

    location ~ .*\.(php|php5)?$
    {
        fastcgi_pass  unix:/tmp/php-cgi.sock;
        fastcgi_index index.php;
        include fcgi.conf;
    }

    location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|js|css)$
    {
        expires      max;
    }

    log_format  access  '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" $http_x_forwarded_for';
    access_log  /home/wwwlogs/blog.access.log  access;
}
</pre>