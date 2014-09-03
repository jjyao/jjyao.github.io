---
layout: post
title: "PHP Single Quoted String"
date: 2012-08-07 21:38
comments: true
categories: Debug
---

Use PHP single quoted string in a case where I should use double quoted string instead

<!-- more -->

## Scenario
I write a Python Twisted line receiver server, so the server reads one line at a time. Then I write a PHP client to send lines to server.

``` php
socket_write($socket, 'it is one line\r\n', strlen);
```

However, my server can't receive a whole line and blocked to wait for **\r\n**

## Bug
In a single quoted string, escape sequences you might be used to, such as \r or \n, will be output literally as specified rahter than having any special meaning. So my server receive \r\n literally and doesn't think that it is a new line character.

## Solution
Change single quotes to double quotes

``` php
socket_write($socket, "it is one line\r\n", strlen);
```

