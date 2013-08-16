---
layout: post
title: php正则表达式-非贪婪匹配和递归匹配
tag: [php, 正则]
category: php
---

###贪婪匹配与非贪婪匹配

php中所有的正则表达式默认都是贪婪的，即一个标识符（如\*）会尽可能多地匹配字符。

举个例子，如模式p\*，它先匹配一个p，然后是0个或者多个字符。对于字符串“php”。在贪婪模式下，它能找到一个匹配，首先它匹配一个p，接着它匹配hp。而在非贪婪模式下，它能找到两个匹配。开始他能匹配p然后h，但是接下来它并不是继续匹配p，而是把最后一个p留下，当做下一次匹配的开始。看下面的代码：

	//preg_match_all函数输出匹配到的个数
	$match = array();
	print preg_match_all('/p.*/', "php", $match);  // 贪婪，输出1
	print preg_match_all('/p.*?/', "php", $match); // 非贪婪，输出2

贪婪匹配也叫最大匹配，非贪婪匹配又名最小匹配。再看另外一个例子：
	
	贪婪模式下：
	$html = '<b>I am bold.</b> <i>I am italic.</i> <b>I am also bold.</b>';
	preg_match_all('/<b>(.+)</b>/', $html, $bolds);
	print_r($bolds[1]);
	Array
	(
	    [0] => I am bold.</b> <i>I am italic.</i> <b>I am also bold.

	)

	非贪婪模式下：
	$html = '<b>I am bold.</b> <i>I am italic.</i> <b>I am also bold.</b>';
	preg_match_all('/<b>(.+?)</b>/', $html, $bolds);
	print_r($bolds[1]);
	Array
	(
	    [0] => I am bold.
	    [1] => I am also bold.
	)

再来看一个更复杂的，为了方便阅读拆分成多段：

	    const URL_REGEXP = '/
	    	^(\/[\d]+)?
	    	\/([^\d\?\/\\][^\/]*?)?
	    	(?:\/([^\?\/\\][^\/]*?))?
	    	(?:\/([^\?]+?))?
	    	(?:\/?\?.*)?
	    $/is';

这个正则表达式的状态转换图是：
![状态转换图](/assets/images/regularexperssion.jpg)

假设带匹配的字符串 $str = /a/b/,通过上面的正则表达式匹配到的结果是什么呢？

?:表示只匹配不捕获，也就是不会出现在结果中。对于$str,它的匹配结果如下：

	array(5) {
	  [0]=>
	  string(5) "/a/b/"
	  [1]=>
	  string(0) ""
	  [2]=>
	  string(1) "a"
	  [3]=>
	  string(0) "" //这个为什么是空呢？
	  [4]=>
	  string(2) "b/"
	}
刚开始没搞懂为什么匹配到的结果是这个，后来和同事在邮件中讨论才搞明白，一下是邮件原文：



>![正则表达式](/assets/images/regularexperssion2.png)
>含“?:”的括号中表达式只匹配不捕获（non-capturing subpatterns），

>匹配“/a”后分析表达式       (?:\/([^\?\/\\][^\/]\*?))?

>根据默认的贪婪原则，括号后“?”先尝试匹配1次括号中表达式

>根据非贪婪原则，"\*?"先尝试匹配0次"[^\/]"，即匹配"/b"，但这样匹配后，后面的正则表达式无法成功匹配剩下的"/"，所以回溯

>"\*?"尝试匹配1次"[^\/]"，无法匹配 "/b/"，。。。

>“匹配1次括号中表达式”失败，进行回溯

>括号后“?”尝试匹配0次，即该表达式匹配空字符串，其中红色部分为不含“?:”的第三个括号，所以匹配结果中第3个为空（去掉“?:”后该表达式两个括号结果都为空）

>感觉贪婪、非贪婪匹配纠结的地方是 需要在整个正则表达式成功匹配的前提下 使子模式尽可能多（少）的匹配，否则需要回溯，直到匹配成功


###递归匹配

递归匹配用符号(?R)来表示，指正则表达式本身。php文档里的[描述](http://php.net/manual/en/regexp.reference.recursive.php):

>Consider the problem of matching a string in parentheses, allowing for unlimited nested parentheses. Without the use of recursion, the best that can be done is to use a pattern that matches up to some fixed depth of nesting.

看下面的代码：

	$str = '{"name":"percent_param", "default":{"24小时回复率":["24小时回复量","售后平台维权量"]}}';
	preg_match_all("/{[^{}]*((?R)*)}/", $str, $t);
	var_dump($t);

打印结果：

	array(2) {
	  [0]=>
	  array(1) {
	    [0]=>
	    string(103) "{"name":"percent_param", "default":{"24小时回复率":["24小时回复量","售后平台维权量"]}}"
	  }
	  [1]=>
	  array(1) {
	    [0]=>
	    string(67) "{"24小时回复率":["24小时回复量","售后平台维权量"]}"
	  }
	}

这里要理解为什么$t\[1\] = {"24小时回复率":["24小时回复量","售后平台维权量"]}。如果把正则表达式改成"/{\[^{}\]*(?R*)}/"呢？结果就很明显了，$t\[1\]是正则表达式匹配迭代中最后一次的捕获结果。

><span style="color:red">PREG_PATTERN_ORDER:Orders results so that $matches[0] is an array of full pattern matches, $matches[1] is an array of strings matched by the first parenthesized subpattern, and so on.If no order flag is given, PREG_PATTERN_ORDER is assumed.</span>


###参考链接
1、[Choosing Greedy or Nongreedy Matches](http://docstore.mik.ua/orelly/webprog/pcook/ch13_05.htm)

2、[PHP中的递归正则](http://iregex.org/blog/recursive-regex-in-php.html)