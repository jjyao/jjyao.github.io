---
layout: post
title: "Refactor with Vim"
date: 2012-12-09 23:28
comments: true
categories: 
---

使用IDE的一个大好处就是可以方便的进行重构，想想Eclipse那彪悍的重构功能吧。可是对于我们这种喜欢用Vim的人来说，要如何来进行代码重构呢？接下来我就介绍一些Tips，来让Vim也能干重构的事。
<!-- more -->

## 变量名、函数名、类名变更
如果这个变更只在一个文件中进行那就很方便了，可以使用Vim的替换命令：
```
:%s/old/new/g
```

可万一现在有很多文件呢，我们需要将每个出现的地方都进行变更，这时我们可以这样做：
首先先写一个shell的函数，然后放到bashrc或者zshrc中去
```
function vr() {
    vim -c "bufdo %s/$1/$2/gc | update" $(ack-grep -i -l "$1") # 这里也可以使用grep命令，只是我觉得ack-grep要更好用一些
}
```
这样当想要替换的时候，只要在shell中执行以下命令就可以了：
```
vr old new
```
这样当前目录及子目录下所有文件中的*old*都会替换成*new*，同时每次替换都会要求确认，这样就防止了可能出现的错误
