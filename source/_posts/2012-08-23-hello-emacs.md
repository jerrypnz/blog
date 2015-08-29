---
title: Hello, Emacs
author: Jerry
comments: true
layout: post
permalink: /2012/08/hello-emacs/
categories:
  - Tech
tags:
  - Editor
  - Emacs
---
学不学Emacs，是最近这些年一直让我很纠结的一个问题。早就听说了Emacs的强大，也看到 了用Emacs作为各种语言IDE的威力，但我一直不能下定决心好好学习它。我想主要原因是 Emacs的快捷键，习惯了Vim简短快捷键带来的快感，Emacs的编辑体验实在让人难受。不过， 为了一个更有威力的Clojure开发环境，我决定还是要试一试这个传说中的，伪装成编辑器的 操作系统。不过，挑这个时间开始学Emacs，与最近那个广为流传的关于Emacs和Vim用户的调 查报告毫无关系……

<div id="outline-container-1" class="outline-4">
  <h4 id="sec-1">
    Evil or Not？
  </h4>
  
  <div id="text-1" class="outline-text-4">
    <p>
      作为一个已经十分熟悉Vim的用户，要不要在Emacs里模拟Vim的操作方式是一个十分让人纠结 的问题。从我这两天的使用过程来看，Emacs的文本编辑体验和Vim还是很有差距的。像text objects这种东西，在Emacs里没有对应物，这直接导致很多操作的效率低了不少。在Google 上搜索，在微博上提问的结果都是建议我用Evil Mode——一个加强版的Vi模拟插件。不过考虑 再三，我还是放弃了这个方案。主要原因是想通过学习“原汁原味”的Emacs，看看那些不用 Vi模拟的Emacs用户是怎么操作的。
    </p>
  </div>
</div>

<div id="outline-container-2" class="outline-4">
  <h4 id="sec-2">
    Plugins
  </h4>
  
  <div id="text-2" class="outline-text-4">
    <p>
      我深知想用好Emacs，对插件的管理十分重要，好在Emacs24已经自带了package.el。这两天 试用了一下感觉还是不错的，虽然功能还不算很完备，但我认为够用了。我安装了 emacs-starter-kit，并按照它的建议，在我的init.el里把自己常用的插件列举出来，并自动检查 是否需要下载。
    </p>

```cl
(add-to-list 'package-archives
             '("marmalade" . "http://marmalade-repo.org/packages/"))
(package-initialize)

(when (not package-archive-contents)
  (package-refresh-contents))

(defvar my-packages
  '(starter-kit
    starter-kit-lisp
    starter-kit-eshell
    starter-kit-bindings
    color-theme-solarized
    clojure-mode
    org
    org2blog
    nrepl
    yasnippet))

(dolist (p my-packages)
  (when (not (package-installed-p p))
    (package-install p)))
```
    
    <p>
      另外在参考了几篇关于Emacs配置的文章以后，我想出来一个自己的配置管理方案： 在~/.emacs.d/目录下建立available.d和enabled.d两个子目录，其中available.d里面放配 置文件，需要启用符号链接到enabled.d里面。这个方案简单有效，先用一阵看看，等更熟悉 了以后再慢慢研究更好的方案。
    </p>

```cl
(mapc 'load (directory-files
              (concat user-emacs-directory "enabled.d")
              t
              "\\.el$"))
```

</div>
</div>

<div id="outline-container-3" class="outline-4">
  <h4 id="sec-3">
    Lisp系语言的终极开发环境
  </h4>
  
  <div id="text-3" class="outline-text-4">
    <p>
      最近醉心于Clojure的学习，而Vim + tmux弄出来的简单环境，实在不够好用。而Emacs中 SLIME的强大，我早在去年看《Practical Common Lisp》的时候就听说了。因此，装上 Emacs后的第一件事就是搞定Clojure的开发环境。一番Google后我装上了clojure-mode， SLIME和nREPL，并在在最近的小项目中小试牛刀玩了一下，然后被其强大的功能震撼了。诸 如加载当前buffer到REPL中等功能已经不算什么了，我最喜欢的是它可以用一个快捷键来将光标 处的宏调用展开并pprint到另一个buffer中。我最近的clojure项目中刚好有几个复杂的宏，之前都是人肉在REPL里写一串代码pprint出来，现在用这个功能，按一下快捷键就行，真是爽！哪怕只是为了一个好用的Clojure开发环境，我也应该把 Emacs学下去。
    </p>
  </div>
</div>

<div id="outline-container-4" class="outline-4">
  <h4 id="sec-4">
    elisp初体验
  </h4>
  
  <div id="text-4" class="outline-text-4">
    <p>
      会一点Lisp的好处之一就是从一开始我就可以尝试hack Emacs。昨天使用了一会Emacs后， 我就发现一个想自己hack出来的功能——Vim里的O和o，即在当前行之前/之后插入新行并将光标移动过去。我找了GNU官方的elisp教程翻了下，顺利实现了这个功能并绑定到C-o和M-o上， 用着很舒服，哈哈。
    </p>

```cl
(defun start-newline-next ()
  (interactive)
  (end-of-line)
  (newline-and-indent))

(defun start-newline-prev ()
  (interactive)
  (forward-line -1)
  (start-newline-next))

(global-set-key (kbd "C-o") 'start-newline-next)
(global-set-key (kbd "M-o") 'start-newline-prev)
```

  </div>
</div>

<div id="outline-container-5" class="outline-4">
  <h4 id="sec-5">
    Org-mode
  </h4>
  
  <div id="text-5" class="outline-text-4">
    <p>
      作为名声在外的Emacs必杀级应用，org-mode我也早在玩vimwiki的时候就有所耳闻。 因此装上Emacs后我也立刻就装上了org-mode体验了一下，目前还没有深入研究，等以后差不 多了再分享我的体验。不过这篇日志可是用org-mode写的，哈哈。org2blog这个插件可以直接通过xml-rpc连接Wordpress发布基于org-mode的日志。分类和Tag支持自动补全，编辑的时候可以自动换行，可以自动上传文中file:/开头的链接对应的文件，这些体贴的功能都让我禁不住感叹：so sweet！
    </p>
    
    <p>
      附一张我的Emacs截图（正在写这篇日志）：
    </p>
    
    <p>
      <img src="http://jerrypeng.me/wp-content/uploads/2012/08/wpid-org2blog-wp-wordpress.png" alt="http://jerrypeng.me/wp-content/uploads/2012/08/wpid-org2blog-wp-wordpress.png" />
    </p>
  </div>
</div>

<div id="outline-container-6" class="outline-4">
  <h4 id="sec-6">
    Why Emacs?
  </h4>
  
  <div id="text-6" class="outline-text-4">
    <p>
      前面提到我看过一篇文章描述作者从Vim转到Emacs的过程，其中有他对Emacs的理解，我觉得 这是让我选择投入Emacs怀抱的主要原因。（原文地址： <a href="http://artagnon.com/why-and-how-i-switched-from-vi">http://artagnon.com/why-and-how-i-switched-from-vi</a>）
    </p>
    
    <blockquote>
      <p>
        Then why? Because I came to realize that programmers do a lot more than just basic text editing. Vim and Emacs assist the programmer in different ways. Vim helps the programmer save time on basic text editing functions while deserting him when he wants to do his compiling or debugging or even interact with an external application. Emacs, on the other hand, seems to understand the code and helps the programmer operate on code entities, rather than characters and lines. SLIME is easily the best development environment I have seen. Emacs interactions with external applications nicely. Even documentation is readily available within Emacs.
      </p>
    </blockquote>
  </div>
</div>
