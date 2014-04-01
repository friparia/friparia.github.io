---
layout: post
title: "vim总结"
date: 2014-03-31 13:54:35 +0800
comments: true
categories: coding-tools
tags: vim
---
这两天在看vim，顺手总结了下不知道但是觉得很有用命令
<!--more-->

* `U` 行撤销，撤销所有在最近编辑的行上的操作
* `ZZ` 保存并退出
* `e` 下一个单词的词末
* `ge` 移动到前一个单词的末尾
* `W` 字串，包括连接符(E同理)
* `0` 移动到一行的第一个字符
* `F` 向左查找
* `t` 查找至其前一个字符
* `ctrl-v` 启动矩形可视模式
* ``”` 跳转到上次离开这个文件时的位置
* ``.` 最后一次修文件的位置
* `.` 重复前一个修改命令
* `ctrl-l` 重画整个屏幕
* `gf` 查找文件中文件名，然后打开(必须在path中或者超链接)
* `gv` 选择上次选过的文本
####寄存器
* `“fyas` 拷贝一个句子到f寄存器

[我的vimrc](https://github.com/friparia/myscripts/blob/master/.vimrc)

####待整理
* 单词组成部分’iskeyword’
* 文本组成对象 text-objects

    
