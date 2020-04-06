---
title: Windows环境下使用Python的Curses库时报错No module named _curses
date: 2019-2-7
toc: true
mathjax: false
tags:
  - python
  - windows环境错误处理
categories: python
typora-root-url: ./
---

#### 一、问题

最近在用python写一个2048的demo时，代码中需要引用到curses库。在windows环境下调试时，除去代码中各种报错之后，代码运行出错。报错如下：

![img](/windows-py-use-curses/5610878-633f20e10c578f8e.jpg)

在网上找了一些解答，简单总结一下解决方法

<!-- more -->

#### 二、解决方法

首先，这个问题产生的根本原因是 **原生curses库不支持windows**。所以我们在安装python后，虽然在`python目录\Lib`文件夹下可以看到curses库，但使用时会产生如上错误，在报错提示的\_\_init\_\_.py文件中确实可以找到`from _curses import *`的语句

要解决这个问题，我们需要使用一个unofficial curses(非官方curses库)来代替自带的curses库。也就是whl包。

在[python whl包下载](https://www.lfd.uci.edu/~gohlke/pythonlibs/#curses)中找到curses，下载与自己python版本对应的whl包（如我的环境是python3.7 ，64位操作系统，下载curses‑2.2+utf8‑cp37‑cp37m‑win_amd64.whl），打开cmd，使用pip安装这个whl包，等待安装完成即可。

> pip install *directory\curses‑2.2+utf8‑cp37‑cp37m‑win_amd64.whl*

