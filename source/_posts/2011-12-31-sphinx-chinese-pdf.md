---
title: 用Sphinx生成中文PDF
author: Jerry
layout: post
permalink: /2011/12/sphinx-chinese-pdf/
categories:
  - 技术
tags:
  - LaTeX
  - reStructuredText
  - Sphinx
---
记录一下用Sphinx生成中文PDF的关键配置，需要和Texlive 2011配合使用。

## 1. 中文字体

修改sphinx工程的conf.py，加入以下内容：

<pre lang="python">latex_elements = {
...
# Additional stuff for the LaTeX preamble.
'preamble': '''
\usepackage{xeCJK}
\setCJKmainfont[BoldFont=SimHei, ItalicFont=KaiTi_GB2312]{SimSun}
\setCJKmonofont[Scale=0.9]{KaiTi_GB2312}
\setCJKfamilyfont{song}[BoldFont=SimSun]{SimSun}
\setCJKfamilyfont{sf}[BoldFont=SimSun]{SimSun}
''',
}</pre>

字体根据个人喜好可以随意更改，要查询Linux下可用的中文字体，用以下命令：

<pre lang="bash">fc-list :lang=zh</pre>

## 2. 中文首行缩进

仍然是更改latex_elements的preamble，加入一下内容：

<pre lang="python">\usepackage{indentfirst}
\setlength{\parindent}{2em}</pre>

这样每个段落的首行会缩进两个字符。

最终的conf.py中的preamble配置：

<pre lang="python">latex_elements = {
...
# Additional stuff for the LaTeX preamble.
'preamble': '''
\usepackage{xeCJK}
\usepackage{indentfirst}
\setlength{\parindent}{2em}
\setCJKmainfont[BoldFont=SimHei, ItalicFont=KaiTi_GB2312]{SimSun}
\setCJKmonofont[Scale=0.9]{KaiTi_GB2312}
\setCJKfamilyfont{song}[BoldFont=SimSun]{SimSun}
\setCJKfamilyfont{sf}[BoldFont=SimSun]{SimSun}
''',
}</pre>

生成的PDF效果图：

<div id="attachment_71" class="wp-caption alignnone" style="width: 611px">
  <a href="http://jerrypeng.me/wp-content/uploads/2011/12/chinese-pdf.png"><img class="size-full wp-image-71" title="中文PDF" src="http://jerrypeng.me/wp-content/uploads/2011/12/chinese-pdf.png" alt="中文PDF效果图" width="601" height="301" /></a><p class="wp-caption-text">
    中文PDF效果图
  </p>
</div>