---
title: 玩转Emacs的auto-complete
author: Jerry
comments: true
layout: post
permalink: /2012/10/play-emacs-auto-complete/
categories:
  - 技术
tags:
  - auto-complete
  - Clojure
  - Emacs
---
这两天从另一套很不错的Emacs配置[Emacs Live][1]中copy了一段auto-complete和 clojure-mode的配置，用起来很爽，可以通过nREPL来补全Clojure的namespace， class，method等等。不过有一个小小的问题：我的auto-complete是自动触发的， 一般情况下自动触发很好用，速度也很快，但在clojure-mode中补全的时候则会 有卡顿现象。这是因为ac-nrepl是动态发送代码到nREPL中运行以得到补全的候选 列表的，速度必然会相对慢一些，如果是通过快捷键补全可能还好一点，但自动 触发补全时会很影响输入体验。 

用了一会很不爽，就开始了改进之旅。 

<div id="outline-container-1" class="outline-3">
  <h3 id="sec-1">
    1. 禁用自动补全
  </h3>
  
  <div class="outline-text-3" id="text-1">
    <p>
      第一个选择是禁用自动补全并设定一个触发按键，比如 <code>TAB</code> 。这个很简单，只需 要两行代码：
    </p>
    
```cl
(setq ac-auto-start nil)
(ac-set-trigger-key "TAB")
```
    
    <p>
      其中 <code>ac-auto-start</code> 还可以被设定为一个指定的数字，这种情况下只有连续 输入了指定数目的字符以后才会开始自动补全。
    </p>
    
    <p>
      这么做虽然可以解决卡顿问题，但牺牲了自动触发的便利性。尤其是在补全当前 buffer中已经出现的单词时，自动触发会极大改善输入体验，减少按键次数。所 以这个解决办法有点“因噎废食”的意思，我用了一会就放弃了。
    </p></p>
  </div>
</div>

<div id="outline-container-2" class="outline-3">
  <h3 id="sec-2">
    2. 为ac-nrepl单独设定补全按键
  </h3>
  
  <div class="outline-text-3" id="text-2">
    <p>
      翻看了一下auto-complete的文档，发现其中核心的函数 <code>auto-complete</code> 是支 持参数的：
    </p>
    
```cl
(auto-complete &optional SOURCES)
```
    
    <p>
      其中 <code>SOURCES</code> 是一个complete source列表，如果没有指定此参数， <code>auto-complete</code> 会使用buffer局部的 <code>ac-sources</code> 变量。一般情况下，我们 都是针对它去做配置。但我想达到的效果是：自动触发的补全，使用速度较快的 sources，而速度较慢的sources，我希望手动触发补全。所以需要做两件事：
    </p>
    
    <ol>
      <li>
        <b>不要</b> 将速度慢的source（比如ac-nrepl的那一堆）放到 <code>ac-sources</code> 里
      </li>
      <li>
        单独写一个补全函数，使用慢速source，并绑定到别的按键上
      </li>
    </ol>
    
    <p>
      废话不多说，下面是这部分的代码：
    </p>
    
```cl
;; 下面的两行代码要去掉，因为ac-nrepl-setup会将自身
;; 的source加入到ac-sources里
;;(add-hook 'nrepl-mode-hook 'ac-nrepl-setup)
;;(add-hook 'nrepl-interaction-mode-hook 'ac-nrepl-setup)

(defun clojure-complete ()
  (interactive)
  (auto-complete '(ac-source-nrepl-ns
                   ac-source-nrepl-vars
                   ac-source-nrepl-ns-classes
                   ac-source-nrepl-all-classes
                   ac-source-nrepl-java-methods
                   ac-source-nrepl-static-methods)))

(add-hook 'nrepl-mode-hook
          (lambda ()
            (local-set-key (kbd "C-M-/") 'clojure-complete)))
(add-hook 'nrepl-interaction-mode-hook
          (lambda ()
            (local-set-key (kbd "C-M-/") 'clojure-complete)))
```
    
    <p>
      很简单，就是新写了个 <code>clojure-complete</code> 函数，里面使用ac-nrepl的source 来调用 <code>auto-complete</code> ，然后将它绑定到nrepl mode的 <code>C-M-/</code> 上了。原来 的配置里setup的部分 <b>一定要去掉</b> ，否则 <code>ac-sources</code> 里还是有ac-nrepl的 那些source，问题依旧。
    </p>
    
    <p>
      使用这个方案玩了一下，目前感觉良好。有什么好点子也希望读者能评论、分享 一下！
    </p>
    
    <p>
      另外我的Emacs配置都在此： <a href="https://github.com/moonranger/dotfiles/tree/master/emacs">https://github.com/moonranger/dotfiles/tree/master/emacs</a>
    </p>
    
    <p>
      Happy hacking Emacs！ <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_smile.gif' alt=':-)' class='wp-smiley' />
    </p>
  </div>
</div>

 [1]: https://github.com/overtone/emacs-live
