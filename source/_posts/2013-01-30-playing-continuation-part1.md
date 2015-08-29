---
title: 玩玩 Continuation（1）
author: Jerry
comments: true
layout: post
permalink: /2013/01/playing-continuation-part1/
categories:
  - Tech
tags:
  - Clojure
  - Continuation
  - HTTP
  - Racket
  - Scheme
  - TR-069
  - Web
keywords: Clojure, Continuation, Racket, Scheme
---

警告：本文可能包含一些错误/不准确的扯淡，如果发现说得不对的，欢迎指正。

<!--more-->

<div id="outline-container-1" class="outline-3">
  <h3 id="sec-1">
    1. 缘起
  </h3>
  
  <div id="text-1" class="outline-text-3">
    <p>
      第一次了解到 Continuation 在 Web/HTTP 领域的使用，还是在上家公司。当时 我们的一个大牛级的架构师巧妙地使用 C 和 Lua 来实现了一套基于 Continuation 的 TR-069 协议实现，了解了他的思路后我发现那是个非常巧妙的 主意，能大幅度简化逻辑的实现（下面为了简化问题，我不再以 TR-069 协议为 例来介绍，而是使用别的更简单的例子以避免牵涉 TR-069 协议的细节）。
    </p>
    
    <p>
      假设我们有一个在线投票系统，让用户选择自己最喜爱的男生/女生（如果是男 生则选择最喜爱的女生，是女生则选择最喜爱的男生），并显示投票结果，其流 程是这样的：
    </p>
    
    <ol>
      <li>
        让用户输入性别
      </li>
      <li>
        如果是男性用户，让用户选择最喜爱的女生
      </li>
      <li>
        如果是女性用户，让用户选择最喜欢的男生
      </li>
      <li>
        展示投票结果
      </li>
    </ol>
    
    <p>
      这其中包含4个页面（选择性别、选择最喜爱的女生、选择最喜爱的男生、投票结 果），使用传统的方法来实现这个流程，其每一个步骤会散落在这几个不同的 Controller 里，并且需要借助 Session 来存储中间变量和当前状态等，本来很 简单的逻辑就被无形中复杂化了。下面的伪代码展示了一个可能的写法（其中每 个函数都是一个 Controller）：
    </p>
    
    <pre class="example">public void selectGender(Request req) {
    Boolean isMale = req.getParameter("gender");
    req.getSession().setAttribute("user-gender", isMale);
}

public void selectFavorites(Request req) {
    Boolean isMale = req.getSession().getAttribute("user-gender");
    if (isMale == null) {
        req.redirect("selectGender");
    }
    if (isMale)
       renderFavoriteGirls();
    else
       renderFavoriteBoys();
}

public void showResults(Request req) {
  ...
}
</pre>
    
    <p>
      可以看到流程的不同步骤散落在不同的 Controller 里，需要我们手工管理好状 态，并且要避免非法情况出现（比如不选择性别就直接投票了）。假如我们能用 下面的“同步”方式来编写这段逻辑，情况就要好很多了。
    </p>
    
    <pre class="example">boolean isMale = getUserGender();
if (isMale) {
    selectFavoriteGirls();
} else {
    selectFavoriteBoys();
}
showPollResult();
</pre>
    
    <p>
      然而想在 Web 编程里实现这样的效果并不是一件容易的事情，因为这样一个流程 里中包含了多次 HTTP 交互，我们需要一种机制来让代码的执行状态得到保存， 在得到用户的请求后恢复。
    </p>
    
    <p>
      第一个可能的选择是线程。但这个方案要求一个这样的流程就占用一个线程来运 行它，这显然会影响服务器的并发和性能。所以这个办法很蠢，基本是不可行的 （几个月前我还差点打算这么干了，囧）
    </p>
    
    <p>
      第二个可能的选择即 Continuation/Coroutine。从维基百科上看， Continuation 可以用来实现 Coroutine，但它本身是比 Coroutine 更强大的一 种执行控制机制，除了Coroutine 以外，它还可以用来实现 Generator， Exception 等。
    </p>
  </div>
</div>

<div id="outline-container-2" class="outline-3">
  <h3 id="sec-2">
    2. Continuation 的实现
  </h3>
  
  <div id="text-2" class="outline-text-3">
    <p>
      具体到 Continuation 的实现，还有两种不同的方式：First-class Continuation 和 Continuation Passing Style。
    </p>
  </div>
  
  <div id="outline-container-2-1" class="outline-4">
    <h4 id="sec-2-1">
      First-class Continuation
    </h4>
    
    <div id="text-2-1" class="outline-text-4">
      <p>
        这是最完整、最强大的 Continuation 实现，一般是在语言级别实现。目前我知 道的实现了 First-class Continuation 的语言有 Scheme 和 Ruby。 First-class Continuation 的核心是 <code>call/cc</code> 操作，即 Call with Current Continuation。 <code>call/cc</code> 会捕捉当前的执行上下文到一个 Continuation 对象 中，以它为参数调用传递给 <code>call/cc</code> 的函数。说起来很拗口，可以看一个简单 的例子，用 <code>call/cc</code> 来实现简单的 Coroutine：
      </p>
      
      <pre class="example">(define (ping)
  (let loop ((n 1))
    (display (format "ping ~A\n" n))
    (call/cc (lambda (k)
               (set! *ping-cont* k)
               (*pong-cont*)))
    (when (&lt; n 5) (loop (+ n 1)))))

(define (pong)
  (let loop ((n 1))
    (display (format "pong ~A\n" n))
    (call/cc (lambda (k)
               (set! *pong-cont* k)
               (*ping-cont*)))
    (when (&lt; n 5) (loop (+ n 1)))))

(define *ping-cont* ping)
(define *pong-cont* pong)

(ping)
</pre>
      
      <p>
        <code>ping</code> 和 <code>pong</code> 是两个交替运行的 Coroutine，两者的交替是通过 <code>call/cc</code> 来实现的。可以看到 <code>call/cc</code> 处的逻辑就是将当前 Continuation 保存在一个变量里，并调用之前保存的对方的 Continuation。这 段程序运行后会输出这样的结果：
      </p>
      
      <pre class="example">ping 1
pong 1
ping 2
pong 2
ping 3
pong 3
ping 4
pong 4
ping 5
pong 5
</pre>
      
      <p>
        借助 <code>call/cc</code> ，诸如协程（Coroutine），生成器（Generator）之类的流程控 制工具完全可以自己来实现。 <code>call/cc</code> 能自动捕捉执行上下文，这一点必须要 在语言级别来实现，如果语言本身不支持，这种功能是无法通过库来实现的。
      </p>
      
      <p>
        事实上，前面提到的那个在线投票的例子，在 Scheme 中就可以轻松实现。接下 来，我会在 Scheme 的一个方言 Racket 里试着实现出来玩一下，敬请期待后续 文章。
      </p>
    </div>
  </div>
  
  <div id="outline-container-2-2" class="outline-4">
    <h4 id="sec-2-2">
      Continuation Passing Style
    </h4>
    
    <div id="text-2-2" class="outline-text-4">
      <p>
        在不支持 First-class Continuation，但是支持闭包的语言里，可以通过将程 序写成 Continuation Passing Style （CPS）来实现 Continuation。CPS 风格 的程序里，函数接受一个额外的函数作为参数，当函数计算出结果的时候，会以 结果为参数调用参数所指定的函数（Continuation）。这种风格里，代码的逻辑 不再体现为“函数调用 -> 返回 -> 函数调用 -> 返回…”，而是“函数调用 -> 以结果为参数调用Continuation -> 函数调用 -> Continuation…”。
      </p>
      
      <p>
        比如<a href="http://en.wikipedia.org/wiki/Continuation-passing_style">维基百科</a>上的例子：
      </p>
      
      <pre class="example">(define (pyth x y)
 (sqrt (+ (* x x) (* y y))))
</pre>
      
      <p>
        转换成 CPS 风格后：
      </p>
      
      <pre class="example">(define (pyth&#038; x y k)
 (*&#038; x x (lambda (x2)
          (*&#038; y y (lambda (y2)
                   (+&#038; x2 y2 (lambda (x2py2)
                              (sqrt&#038; x2py2 k))))))))
</pre>
      
      <p>
        可以看到，程序逻辑流的推进，在直接模式（Direct Style）下，是体现为嵌套 的函数调用，很简单直接；但在 CPS 下，程序逻辑体现为 Continuation 的嵌 套，Continuation 之间的推进是控制在用户手里的。换言之，CPS 风格中，代 码控制流是掌握在用户手中的。
      </p>
      
      <p>
        （忽然意识到，其实很多异步 IO 库比如 Python 的 Tornado 和 Node.js 就是 在让用户以 CPS 风格写程序，Callback 本质上就是 Continuation）
      </p>
      
      <p>
        借助 CPS，很多不支持 First-class Continuation 的语言，比如 Clojure，理 论上也可以实现上面提到的那种风格的程序，但一个很严重的问题是 CPS 风格 的程序可读性非常不好，逻辑流看起来并不像 Direct Style 那么清晰，这点有 背初衷——简化 Web 逻辑的表达。
      </p>
      
      <p>
        但借助一个强大的武器——Monad 的帮助，我们可以把 CPS 风格隐藏在背后，让 程序的代码本身依然保持 Direct Style 式的可读性。
      </p>
    </div>
  </div>
  
  <div id="outline-container-2-3" class="outline-4">
    <h4 id="sec-2-3">
      Continuation Monad
    </h4>
    
    <div id="text-2-3" class="outline-text-4">
      <p>
        关于这个话题，我关注了很久，但到现在都还没完全理解——主要原因还是 对 Monad 理解不深，更别提实际使用了。所以这里我还没有办法展开说得太深 入，但可以呈现其最终的效果。
      </p>
      
      <p>
        简单的说，<a href="http://www.intensivesystems.net/tutorials/cont_m.html">Continuation Monad</a> 可以把代码转换成 CPS 风格，转换的细节隐藏 在 Monad 的实现背后。我现在还没办法更深入介绍其中原理，但可以呈现一个 由<a href="http://www.intensivesystems.net/tutorials/web_sessions.html">别人写出来的例子</a>：
      </p>
      
      <pre class="example">(def web-app (web-while (constantly true)
                        (web-seq get-name
                                 get-age
                                 get-gender
                                 (web-cond
                                   male? male-options
                                   :else female-options)
                                 show-summary)))

(defroutes new-file-routes
           (ANY "/app-url"
                (handle-request request web-app)))
</pre>
      
      <p>
        可以看到这段代码里实现了一个 Web Flow：即不断循环（ <code>web-while</code> ）以下 过程：
      </p>
      
      <ol>
        <li>
          获取姓名、年龄、性别（通过表单让用户输入）
        </li>
        <li>
          根据用户是男性还是女性呈现不同的选项（ <code>web-cond</code> ）
        </li>
        <li>
          展示结果
        </li>
      </ol>
      
      <p>
        和我上面举的例子如出一辙——通过这种“看起来像是 Direct Style”的代码来 实现跨多次 HTTP 交互的流程。在其背后，它使用了 Continuation Monad 来将 这段代码转换成 CPS 风格，并在 <code>handle-request</code> 里去推动流程的运转。
      </p>
      
      <p>
        <code>handle-request</code> 的源码里也能看出来一些端倪：
      </p>
      
      <pre class="example">(defn handle-request [request session-start]
  (let [session-id (get (:params request) :session-id (new-id))
        session (get @sessions session-id)
        [result next-handler] (if session
                                (session request)
                                (run-cont
                                  (session-start
                                    [{:app-context {:session-id session-id}}
                                    request])))]
    (dosync (ref-set sessions (assoc @sessions session-id next-handler)))
    result))
</pre>
      
      <p>
        可以看到，基本逻辑就是从 <code>session-start</code> 开始调用，得到下一步的 Continuation，存储在 <code>sessions</code> map里，之后的请求处理就是不断调用这些 Continuation，推动流程并得到下一个 Continuation （ <code>next-handler</code> ）。
      </p>
      
      <p>
        这个实在是让人眼前一亮，在我的<a href="https://github.com/moonranger/clj.tr069">clj.tr069</a>里，我一直试图寻找一个方案来简 化协议逻辑的编写，而不必使用复杂的状态机。我想，这就是一个答案了。
      </p>
    </div>
  </div>
</div>

<div id="outline-container-3" class="outline-3">
  <h3 id="sec-3">
    3. 总结
  </h3>
  
  <div id="text-3" class="outline-text-3">
    <p>
      今天说的这些，似乎没太多人研究，网上搜索到的也都是有年头的资料。可能实 际的 Web 应用很少有这种需要。即使是 TR-069 这样的有复杂交互过程的协议， 也并没有复杂到不可接受的程度。研究这些纯粹是处于好玩:-)
    </p>
    
    <p>
      接下来我想继续挖掘一下这个主题玩一下，要探索的主要有：
    </p>
    
    <ol>
      <li>
        玩玩 Scheme 的 call/cc 和 Racket 中已经实现的 Web Continuation
      </li>
      <li>
        深入学习一下 Monad，搞清楚 Continuation Monad 的原理
      </li>
      <li>
        借助 Continuation Monad 来继续实现 clj.tr069
      </li>
    </ol>
    
    <p>
      敬请期待 <img src='http://jerrypeng.me/wp-includes/images/smilies/icon_smile.gif' alt=':-)' class='wp-smiley' />
    </p>
  </div>
</div>
