---
layout: post
title: "Review LightMail Server Development"
date: 2013-05-08 18:02
comments: true
categories: 
published: false
---

* 文档一定要看全，要仔细看，静下心来看，直到把所有的细节都弄明白。现在弄多了碎片阅读后，产生了阅读障碍，对大篇幅的技术文档没有耐心读完。
* 在服务器端, 高并发，I/O bound的程序，异步编程比同步编程有巨大的优势，虽然说写出来的代码可能比较难阅读。
* Openssl会占很多内存，MODE_RELEASE_BUFFERS可以减少一点内存的使用
* 编写daemon程序一定要做好十足的错误处理，daemon进程是不能跪的
* 能用编写良好的第三方库的时候就不要自己写
* 不能把EC2当做Linode用，要看使用说明书
