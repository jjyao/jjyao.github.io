---
layout: post
title: "OpenSSL Read Ahead &amp; Epoll"
date: 2012-08-28 12:06
comments: true
categories: Debug
---

Use epoll with a ssl socket and ignore the OpenSSL read ahead feature

<!-- more -->

## Scenario
I write a python program to implement IMAP client that uses the IDLE command. So I have a ssl client socket and I use epoll level triggered mode to implement non-blocking socket.

However, epoll doesn't work correctly. For example, in the IDLE mode, IMAP server sends two lines at a time. My client `select.epoll()` function returns because the ssl socket is readable. Then I read one line and call `select.epoll()` again. Because I use epoll level triggered mode so the `select.epoll()` should return immediately because the socket still has one line that is ready for read. But in fact, when I call `select.epoll()` again, it blocks and doesn't return.

## Bug
OpenSSL has read ahead feature, which means that it may read more data to its internal buffer. So when I read one line, it may indead read two lines, decrypt and store them to the internal buffer. That is why `select.epoll()` blocks because all the data in the underlying socket is read to the OpenSSL internal buffer and the socket is unreadable.

## Solution
Use `pending()` method to see whether ssl has decrypted bytes that are available for read.
