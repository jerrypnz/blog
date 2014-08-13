---
layout: post
title: Nginx 基于客户端 IP 来开启/关闭认证
date: 2014-08-13 16:47:29 +0800
comments: true
categories: Linux
tags: nginx, satisfy
keywords: nginx, satisfy
description: 配置 Nginx 以实现特定网段的 Client 无需认证，其他需要认证
---

前些日子帮助公司在搭建了一个内部资源的导航页面，方便公司员工访问各种常
用的系统。因为这个页面包含一些敏感信息，我们希望对其做认证，但仅当从外
网访问的时候才开启，当从公司内网访问的时候，则无需输入账号密码。

按照这个需求研究了下 Nginx 配置，发现 `satisfy` 指令可以很好地解决这个
问题。下面贴出来我们的配置：

```nginx
server {
    listen 80;
    server_name intra.foobar.com;
    charset utf-8;

    satisfy any;
    allow 192.168.0.0/22;
    allow 111.222.333.1/32;
    deny all;
    auth_basic "Intranet";
    auth_basic_user_file /usr/local/nginx/conf/intranet-htpasswd;
    
    location / {
        root /usr/local/www/intranet;
        index index.html;
    }
}
```

其中 `satisfy` 作用于所有的 access phase handler，参数值有两种：`all`
和 `any`，分别代表所有规则都满足和任一规则满足。将其配置为 `any`，并配
合 auth_basic 和 access 模块就可以基于 IP 地址来开启/关闭认证。

以我们的规则为例，`192.168.0.0/22` 这个内网网段和 `111.222.333.1`（虚
构的）这个 IP 的 Client 是可以访问的，而其他用户则需要进行认证。

官方文档上的说法是 `satisfy` 支持 `ngx_http_access_module`,
`ngx_http_auth_basic_module` 和 `ngx_http_auth_request_module`，不确定
对于第三方的认证模块如 `ngx_http_auth_digest`，`satisfy` 是否也能工作。

详细的说明可以参考 [Nginx 官方文档](http://nginx.org/en/docs/http/ngx_http_core_module.html#satisfy)。
