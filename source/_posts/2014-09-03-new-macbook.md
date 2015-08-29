---
layout: post
title: 加入 Mac 神教
date: 2014-09-03 14:39:27 +0800
comments: true
categories: Tech
keywords: Mac Book Pro, MBP, OS X
description: 咬牙入了 2014 新款 MBP，从此加入 Mac 神教
---

在每天上[苹果团](http://www.appletuan.com)看报价几个月后，我终于下狠心
在半个月前下单入了 2014 新款的 Retina Mac Book Pro MGX72。到现在用了半
个月，完全用它取代了公司配的华硕笔记本。目前为止的总体感觉非常好，总结
起来主要有两点：一是屏幕太棒了；二是硬盘太小了。

## 总体感受

Retina 屏幕真不是盖的，看着实在是太舒服了。对于每天面对密密麻麻一堆代码
的码农来说，一块精细的 Retina 屏幕真是个了不起的福利。有了这么高的分辨
率，真是无论用什么字体，什么渲染方法都好看。以前用 Linux 的时候，每次装
完一个新系统，第一件做的事情就是折腾字体：安装各种补丁，折腾
fontconfig 配置（P.S. 推荐所有对字体渲染有要求的 Linux 用户尝试下
[infinality](http://www.infinality.net/blog/)）；相比较之下，用 Mac 就
完全无需折腾，直接用就好了。自带的等宽字体 Monaco 非常漂亮，中文的冬青
黑也非常不错。我甚至修改了我的 Emacs 配置，只有在 Linux 系统下才设置字
体了：

<!--more-->

```cl
(if (and (eq system-type 'gnu/linux) (display-graphic-p))
    (progn
      (set-face-attribute
       'default nil :font "Source Code Pro Bold 9")
      (dolist (charset '(kana han symbol cjk-misc bopomofo))
        (set-fontset-font (frame-parameter nil 'font)
                          charset
                          (font-spec :family "文泉驿等宽微米黑"
                                     :size 13)))))
```

因为默认字体实在是很漂亮，无需更换：

{% img /images/201409/retina-emacs.png %}

128G 的 SSD 着实是个让人蛋疼的地方。现在 SSD 价格已经便宜了很多，不明
白为什么苹果还在低配版的 MBP 上选择 128G 的容量。我这系统才刚用没多久，
硬盘容量就只剩下 30G 左右了，以后估计要经常清理。玩游戏什么的就更别想了，
一个几年前的巫师2就要 18G，新游戏就更别说了。不过 MBP 倒也不是用来玩游戏
的，还是放弃吧。至于什么照片、视频还是别想了，统统扔到移动硬盘里吧。目
前我的 iPhoto 里仅仅是放了我的 iPhone 这一年多的照片，再加上一些婚纱照
什么的就已经占用了 7、8G 了。单反相机的照片必须要老老实实放移动硬盘上
了。

## 软件

说说 Mac 下的软件吧。目前我尝试了的有这些：

#### Alfred 2

早有耳闻的神器，买完第一个装的就是它。我现在还没买 Powerpack （感觉有点
贵），仅仅用它的一些基本功能，感觉很不错的，用来替代 Spotlight是绰绰有
余了。看着网上有人分享的各种 Workflow，还是有点心痒痒的。不过先忍忍，
以后再买吧。

#### IntelliJ IDEA 13

和别的系统上的版本没啥区别，不过得益于 Retina 屏幕，看起来舒服了不少。

#### Homebrew

有了它，安装程序不能更简单了。买完第一批安装的程序就有 Homebrew，然后通过
它安装了 GNU coreutils，zsh，tmux，imagemagick 等等命令行工具。命令行
还是习惯 GNU 的那些，毕竟用惯了 Linux。说起 Homebrew，不得不提
[Gentoo Prefix](https://www.gentoo.org/proj/en/gentoo-alt/prefix/)。作
为 Gentoo 的一部分，它提供了机制在类 UNIX 系统上运行 Gentoo。作为一个
多年的 Gentoo Linux 用户，它对我还是很有吸引力的，只不过它的活跃程度比
不上 Homebrew，另外它需要一个专门的目录作为 Prefix 也不是很对我的胃口。
等以后有时间我再玩玩看。

#### iTerm 2

超级好用的终端，界面美观，功能强大，最最重要的是它竟然能和 tmux 整合，
简直是太棒了，对我这种每天都要连接 N 个服务器，开 N 个 SSH Session 的
人来说相当好用。通过 iTerm 2 使用 tmux，就不用老是按 C-a 这个前缀按键
了，tmux 的 window 也变成了 iTerm 2 里的 tab，直接用 Command 加上数字
就可以切换。

#### QQ for Mac

公司的 IM 就是 QQ，没办法必须用。我最受不了的是这一众国产软件的 Mac 版
都跟着装逼，一个比一个简洁、文艺。Mac 版 QQ 没有了 Win 版的弹窗、乱七八
糟的小按钮、QQ 秀等等，用起来还是很舒服的。不过看起来它也有逐渐变得臃肿
的趋势。最新版里的那个 Swiftly 的功能怎么看怎么多余，这是想山寨一个
Alfred 的节奏吗？

#### Shadowsocks X

新系统怎么少得了科学上网的工具呢。Shadowsocks X 做得很不错，没有复杂的
界面，安安静静呆在状态栏上就工作了，直接全局启用，通过 PAC 来控制，很好
很强大。唯一不爽的是要手工修改 PAC。回头可以考虑写个脚本，配合 Alfred
2 来动态增删 URL。


## 一些小问题

- Home 和 End 键。这两个按键需要用 Fn + 左右箭头，实际使用起来很不爽。
  `C-S-a` 和 `C-S-e` 又不是在哪都能用。
- 没有了 `C-S-Left` 和 `C-S-Right` 按单词选择的功能。可能有替代按键，
  但我目前还没找到。
- 散热问题。为了用 Octopress，我装了个 rvm 和 ruby 1.9.3，结果发现它还
  要装个 gcc 4.6.x 来编译 ruby。编译 gcc 时，那风扇声音叫一个大，而且
  明显感觉电脑更烫。看来以后不敢让这娇贵货干太重的活了……

其实 Mac 对我来说最大的吸引力在于 Retina 屏幕，以及 OS X 这个不怎么需要
折腾的系统，其中前者对我来说才是真正的 deal-breaker。如果 Mac 没有
Retina 屏幕，或者 PC 上也有高分辨率 IPS 屏幕，并且 Linux 系统对其有很好
的支持的话，可能我根本就不考虑 Mac 了。OS X 虽然好用，但对于我来说，一
个配置好的 Linux 桌面也够用了。毕竟作为服务端开发者，大部分时间还是在终
端、浏览器、IDE 和 Emacs 中度过的，而这些在所有类 UNIX 平台都基本一样。
