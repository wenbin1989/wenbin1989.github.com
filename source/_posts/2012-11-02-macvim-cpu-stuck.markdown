---
layout: post
title: "MacVim处理中文时卡顿、CPU占用过高问题"
date: 2012-11-02 16:34
comments: true
categories: [mac, vim]
keywords: MacVim, vim, mac, cpu, 卡, 卡顿, 占用, 中文, Use Core Text renderer
description: MacVim编辑含有大段中文的markdown文档时，出现卡顿、CPU占用严重过高问题。Preferences对话框、Advanced选项卡中去掉Use Core Text renderer选项，问题解决。
---

问题出现在[MacVim-snapshot-65][1]版本

本人一般用MacVim编辑文档。近期发现编辑含有大段中文的markdown文档时，出现卡顿、MacVim进程CPU占用严重过高问题。
google搜索无果，自行研究，过程如下：

* 打开纯英文markdown文档，不管多长都不会出现卡顿问题。
* 打开大段中文txt文件，未出现卡顿问题。
* 考虑可能是语法高亮造成卡顿问题，关闭语法高亮，大段中文的markdown文档未出现卡顿问题。

虽然关闭语法高亮可以解决卡顿问题，但是。。。这个必须得有啊！！！

继续研究，偶然发现，在*Preferences*对话框中选择*Advanced*选项卡，去掉*Use Core Text renderer*选项的勾勾，
则打开语法高亮后，大段中文也不会造成卡顿，至此问题解决。

虽然去掉*Use Core Text renderer*选项后，渲染效果变差一点点，无法使用透明效果，偶尔出现残像，但总比没有语法高亮强。
Sigh...

[1]: https://github.com/downloads/b4winckler/macvim/MacVim-snapshot-65.tbz
