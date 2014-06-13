---
title: 自动同步Xfce和LightDM的墙纸
author: Jerry
layout: post
permalink: /2012/06/sync-xfce-lightdm-wallpaper/
categories:
  - Life Hack
  - UNIX/Linux
tags:
  - LightDM
  - Linux
  - Xfce
---
今天上午小小地hack了一下，解决了一个一直让我很不爽又没有找时间好好解决的问题：XDM和DE的背景不一致。无论使用什么DM，我都希望其背景和我登录以后的桌面背景保持一致，这样登录过程背景不会有变化，不会给人很突兀的感觉。不过一般DM和DE之间不会强耦合，也不会共享同样的桌面背景设置，而是分别独立设置的，所以如果想保持两张墙纸是一样的，需要人工修改配置文件。每次改墙纸都人肉改一次配置文件，这自然是无法忍受的，于是上午写了个脚本，放到local start里，自动同步墙纸设置。

Xfce的桌面配置在~/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-desktop.xml中：

<pre lang="xml" escaped="true">&lt;channel name="xfce4-desktop" version="1.0">
&lt;property name="backdrop" type="empty">
&lt;property name="screen0" type="empty">
&lt;property name="monitor0" type="empty">
&lt;property name="image-path" type="string" value="/home/jerry/Pictures/wallpaper/2012/1070492-1440x900-photo.jpg"/>
&lt;property name="last-image" type="string" value="/home/jerry/Pictures/wallpaper/2012/1070689-1440x900-Irui-Guneden.jpg"/>
&lt;property name="last-single-image" type="string" value="/home/jerry/Pictures/wallpaper/2012/1070492-1440x900-photo.jpg"/>
      &lt;/property>
    &lt;/property>
  &lt;/property>
&lt;property name="desktop-icons" type="empty">
&lt;property name="style" type="int" value="0"/>
&lt;property name="icon-size" type="uint" value="48"/>
&lt;property name="file-icons" type="empty">
&lt;property name="show-removable" type="bool" value="true"/>
&lt;property name="show-trash" type="bool" value="true"/>
    &lt;/property>
  &lt;/property>
&lt;/channel>
</pre>

其中name为image-path的那个property就是墙纸的路径。 而LightDM GTK Greeter的配置则在/etc/lightdm/lightdm-gtk-greeter.conf中：

<pre>[greeter]
background=/home/jerry/Pictures/wallpaper/2012/1070689-1440x900-Irui-Guneden.jpg
theme-name=Adwaita
icon-name=gnome
cursor-name=gnome
font-name=Cantarell 11
xft-antialias=true
xft-dpi=96
xft-hintstyle=hintslight
xft-rgba=rgb
</pre>

 其中的background就是背景的设置了。

脚本只需要把Xfce的桌面背景图片的路径取出来，再将其替换到LightDM配置文件的background中去，就可以了：

<pre lang="bash" escaped="true">#!/bin/bash

XFCE_DESKTOP_CONF=/home/jerry/.config/xfce4/xfconf/xfce-perchannel-xml/xfce4-desktop.xml
LIGHTDM_GTK_GREETER=/etc/lightdm/lightdm-gtk-greeter.conf

if [ ! -f $XFCE_DESKTOP_CONF ]; then
    echo "xfce desktop file does not exist"
    exit 0
fi

if [ ! -f $LIGHTDM_GTK_GREETER ]; then
    echo "LightDM GTK Greeter config file not found"
    exit 0
fi

wallpaper=$(xpath  $XFCE_DESKTOP_CONF "//property[@name='image-path']/@value" 2>/dev/null \
    | grep value= \
    | awk -F\" '{print $2}')

if [ -z "$wallpaper" ]; then
    echo "xfce wallpaper config not found"
    exit 0
fi

sed -i "s#^background=.*#background=$wallpaper#g" $LIGHTDM_GTK_GREETER
</pre>

 其中Xfce墙纸路径是通过xpath工具来获取的，操作XML，grep之类的工具不好使……

在Gentoo下把这个脚本放到/etc/local.d/中，并以.start结尾就可以在Gentoo下自启动了。刚才试了一把，工作得很好，哈哈。

其他的DM和DE，可以自行寻找对应的配置文件并修改脚本，自启动的方法可能也不一样（一般是rc.local），但基本思路就是这样的。如果你也有同样的需求，自己hack试试吧！

此脚本源文件：<a href="https://github.com/moonranger/personal-config/blob/master/scripts/sync-lightdm-wallpaper" target="_blank">https://github.com/moonranger/personal-config/blob/master/scripts/sync-lightdm-wallpaper</a>

Happy Hacking <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_smile.gif' alt=':-)' class='wp-smiley' />