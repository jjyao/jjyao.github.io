---
layout: post
title: "A Close Look Into PHP zval"
date: 2013-03-17 13:43
comments: true
categories: 
---

最近研究了一下PHP variable的内部实现也就是zval，以及引申出来的Copy on Write, Reference等概念和机制。为了检验一下我是不是真的弄懂了，一个好的方法就是看能不能把它清楚地写下来并让人能够读懂。

<!-- more -->

## 什么是zval
简单的来说zval就是PHP variable的value在低层C的表示。大家都知道PHP主要是用C语言来写的，于是所有东西应该都有一个C的对应，PHP variable的value对应就是zval struct。首先先来看一下zval struct的定义：
``` c
struct {
    union {
        long lval;
        double dval;
        struct {
            char *val;
            int len;
        } str;
        HashTable *ht;
        zend_object_value obj;
    } value;
    zend_uint refcount;
    zend_uchar type;
    zend_uchar is_ref;
} zval;
```
接下来我们就从最简单的开始，逐一解释一下各个字段的含义。
### type
首先是`type`字段，这个顾名思义就是表示PHP variable是什么类型的。由于PHP是动态类型语言，因此需要这个字段来标示，同时根据这个字段可以判断出`value`字段里的值到底是什么。

### value
`value`字段存放的是PHP variable的值，可以看出这是个union类型，也就是说这个字段可以有多种解释，关键看`type`的值是什么，这不就有点像多态么。接下里我们就看看`type`和`value`的对应关系是什么： 

PHP variable type | zval value            | Notes
:----------------:|:---------------------:|:-----------------------------------------------:
Long              | long lval             |
Double            | double dval           |
String            | struct{…} str         |
Resource          | long lval             | 存放的只是resource的identifier,而不是resource本身
Boolean           | long lval             | 0表示FALSE，1表示TRUE
Array             | HashTable *ht         | PHP的很多东西都是用HashTable来实现的
Ojbect            | zend_object_value obj |
NULL              |                       | NULL本身也是一种类型，但它不需要使用`value`字段

其中Long, Double, String, Boolean, NULL应该是很清晰了，用zval可以完全表示，略复杂一点的是Object,Array和Resource。在这里我们就只讨论Ojbect和Array。  

#### Object
从上面的对应表可以看出Object的value是用`zend_object_value`这个struct来表示的，我们就看看这个struct的定义：

``` c
struct {
    zend_object_handle handle;
    zend_object_handlers *handlers;
} zend_object_value;
```

在这里要说明的是一个ojbect的data并不是直接存放在zend_object_value里面的，而是放在object store中，也就是说每个object都有一个对应的object store（这应该也是用HashTable来实现的吧），里面放着object的data，也就是object属性的值，这些值本身也是一个个的zval。接下里PHP要做的就是建立object和object store的一一对应关系，而这个就是通过`zend_object_handle handle`来实现的，这是一个long值，相当于object store的identifier,通过这个handle就可以找到object的data在那里了。而`zend_object_handlers *handlers`指向了一些处理函数，比如用来访问object属性的函数等等，这个在这里就不再详述了，有兴趣的可以看[这里](https://wiki.php.net/internals/engine/objects)。讲到这里，我们应该知道了对于object的zval来说本身并没有存放object的data，而只是存放了一个整数型的handle。认识到这一点很重要，因为这会对接下来讨论的copy by reference和copy by value产生直接的影响。

#### Array
PHP的Array是用HashTable来实现的，通过这个就可以存放Array中的Key-Value了。关于这个HashTable具体是如何实现的，可以看[这里](http://nikic.github.com/2012/03/28/Understanding-PHPs-internal-array-implementation.html)。我根据那个tutorial画了个示意图如下：

{% img /images/post/a-close-look-into-php-zval/zval.6.png %}

在这里我们要知道的是Key-Value中的value也就是一个个的zval，就和object的属性值一样，HashTable中就存放着指向这些zval的指针。

### refcount & if_ref
看完了`type`和`value`字段，接下来就看看`refcount`和`if_ref`字段，这两个字段关系到gc,关系到copy by value和copy by reference，所以非常重要，需要仔细讲解。`refcount`表明当前有多少个variable指向了这个zval，这个信息就可以用来gc，当`refcount`变为0后，这个zval就可以被回收。`if_ref`表明当前指向这个zval的variable是不是reference。在PHP中一个variable可以是value type就像这样`$b = $a`或者是reference type就像这样`$b = &$a`。在语义上这两者是不同的，因此也就拥有不同的行为，我们首先要区分这两种类型的variable，这个通过代码解释最好不过了。

``` PHP
$a = 'hello';
$b = $a; // $b is value type
$a = 'world';
var_dump($a); // 'world'
var_dump($b); // 'hello'

$a = 'hello';
$b = &$a; // $b is reference type
$a = 'world';
var_dump($a); // 'world'
var_dump($b); // 'world'
```

从上面一段代码可以看出如果$b是value type的话，它的值并不会随着$a的改变而改变，就好像在执行`$b = $a`时，`$a`的值(zval)被复制了一份，也就是说`$a`和`$b`拥有自己独立的zval，两者互不影响。而如果是reference type的话，就如同`$a`和`$b`共用了一个zval，不管是改变`$a`，还是`$b`，其实都是改变一个zval。但事实没有这么简单，如果真像上面所说的那样执行`$b = $a`时，zval复制了一份，那么效率也太低了，因为有可能`$b`和`$a`在接下来的使用过程中都是只读的，它们本可以共用一个zval而不会出问题。那么PHP究竟是如何实现的呢，答案就是通过`refcount`和`if_ref`实现了copy on write机制。而这个copy on write机制对于不同type的变量也会有些不同的表现，接下来我们就通过一系列的图例来理解这个机制。

{% img /images/post/a-close-look-into-php-zval/zval.1.png %}
{% img /images/post/a-close-look-into-php-zval/zval.2.png %}
{% img /images/post/a-close-look-into-php-zval/zval.3.png %}

前两张图大家应该很好理解，可以清楚的看到copy on write的发生并可以看到`if_ref`和`refcount`的作用，PHP正是依赖于`if_ref`和`refcount`来决定是否需要copy on write。而从第三张图中可以看到在没有`write`发生的情况下就发生了`copy`，而这是为什么呢，貌似`$a`、`$b`和`$c`可以共用一个zval的呀。原因其实很简单，那就是如果这三个变量共用一个zval那么`is_ref`的值就不知道该填什么了，`0`或`1`都不正确，因为这三个变量中既有value type又有reference type。如果`is_ref`为`0`那么`$a`和`$c`是reference关系这个信息就无法表达,PHP会认为`$a`、`$b`和`$c`是value type，改变其中任何一个的值并不会对另外两个造成影响，这显然不对。如果`is_ref`为`1`那么PHP会认为`$a`、`$b`和`$c`是reference type，改变其中任何一个的值都会反映在另外两个上，这显然也是不对的。综上所述在这种情况下只能copy出一份zval出来，这样才能保证接下来不管是操作`$a`、`$b`或者`$c`都不会出问题。之前的这三种图展示的是type为Long的variable的情况，而Boolean、Double、String和Long是一样的，就不再画出来了，而Array和Object比较复杂，情况也特殊，所以拿出来单独讨论。

{% img /images/post/a-close-look-into-php-zval/zval.4.png %}
{% img /images/post/a-close-look-into-php-zval/zval.5.png %}

从上面两张图看出Array和Object的行为存在着显著的不同，Array拥有`by-value semantics`而Object拥有`by-reference semantics`。Array的表现更像之前提到的Long type，也会有Copy on Write行为，而它的Copy从上图中可以看出比较智能，只Copy那些必须的zval。而Object则不一样，不管是value type还是reference type，它们都共用一个object store，也就是说任何属性的值发生了变化，所有指向这个object的variable都能知道，就好像Copy on Write对Object不起作用了。为了强制Copy，最后执行了`$c = &$a`这条语句，为什么这条语句能强制使zval copy一份的原理在上面已经讲过了，这里就不再复述。这里有趣的是zval确实copy了一份，但是两个zval还是指向了同一个object store，也就是说不管通过`$a`、`$b`或者`$c`改变了某个属性，这三个变量都能知道。那么zval的copy到底做了什么呢？还记得上面提到过`zend_object_value`中只是存放了一个整型的`handle`字段，而通过这个字段可以找到object store在哪，那么zval的copy也就只会将这个`handle`字段copy一份，于是两个zval中拥有同样的`handle`值，它们就当然指向了同一个object store。那么如何让Object发生像Array那样类似的Copy呢，PHP为此提供了`clone`关键字。  

说到这差不多把zval给讲清楚了，同时也讲了一下Copy on Write机制，Value type和Reference type的区别。有了上面的解释，我们也应该能知道函数传参的Pass by Value和Pass by Reference的区别了，在这里就不多说了。如果有人想看更多类似我上面的图的话，可以看[这里](http://andrey.hristov.com/projects/php_stuff/Internals%20Exposed.pdf)。如果有人想问我是怎么知道低层zval的变化的话，可以看[这里](http://www.php.net/manual/en/function.debug-zval-dump.php)。最后还可以看看这个和zval有关的[问题](http://stackoverflow.com/questions/10057671/how-foreach-actually-works/)。

## 参考资料
1. [Extension Writing Part II: Parameters, Arrays, and ZVALs](http://devzone.zend.com/317/extension-writing-part-ii-parameters-arrays-and-zvals/)
2. [PHP's Source Code For PHP Developers - Part 3 - Variables](http://blog.ircmaxell.com/2012/03/phps-source-code-for-php-developers_21.html)
3. [Zend Engine 2 - Internals Exposed](http://andrey.hristov.com/projects/php_stuff/Internals%20Exposed.pdf)
4. [Understanding PHP's internal array implementation (PHP's Source Code for PHP Developers - Part 4 )](http://nikic.github.com/2012/03/28/Understanding-PHPs-internal-array-implementation.html)
