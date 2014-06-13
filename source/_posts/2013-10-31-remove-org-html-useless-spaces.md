---
title: 去除 org-mode 输出 HTML 时多余的空格
author: Jerry
layout: post
permalink: /2013/10/remove-org-html-useless-spaces/
categories:
  - 技术
tags:
  - Emacs
  - org-mode
---

用 org-mode 写文档、写 slides 有一段时间了，org-mode + Tex Live 这个组
合真是棒极了，写文档，制作 slides 的效率高多了，输出的 PDF 也很漂亮。但
偶尔也会有输出 HTML 的需要（比如写 Blog），此时有一个不大不小的问题：输
出的 HTML 里的中文段落会多出来很多不必要的空格。

究其原因，Emacs 默认启用了 autofill，会自动将段落的宽度控制在 80 个字符。
当这样的段落被输出成 HTML 的时候，换行符就被替换成了空格。在英文文章里
这不是问题，因为英文本来就是用空格来分割单词，但在中文里就很让人不爽了。

今天有同事问到这个问题，我又花时间 Google 了一下，这次在[水木][1]上找到
了一个还不错的 Workaround：通过 `defadvice` 定义一个“拦截器”，在
org-mode 生成段落 HTML 的函数执行之前，把段落里的中文文字之间的空格去除
掉。代码如下：

```elisp
(defadvice org-html-paragraph (before fsh-org-html-paragraph-advice 
                                      (paragraph contents info) activate) 
  "Join consecutive Chinese lines into a single long line without 
unwanted space when exporting org-mode to html." 
  (let ((fixed-contents) 
        (orig-contents (ad-get-arg 1)) 
        (reg-han "[[:multibyte:]]")) 
    (setq fixed-contents (replace-regexp-in-string 
                          (concat "\$latex " reg-han
                                  "\$ *\n *\$latex " reg-han "\$") 
                          "\\1\\2" orig-contents)) 
    (ad-set-arg 1 fixed-contents)))
```

或者你也可以考虑使用[我的 Emacs 配置][2]，里面自带了很多功能，包括开箱
即用的 org-mode + TexLive 导出功能。

不得不夸一下 elisp 的强大，像上面的 `defadvice` 这种拦截器（或者叫 AOP）
机制，可以让人轻松改变已有的插件的行为而不必改变其代码，真心好用。

 [1]: http://ar.newsmth.net/thread-d98e0223ce6e8f.html
 [2]: https://github.com/moonranger/dotemacs
