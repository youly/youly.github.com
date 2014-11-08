---
layout: post
title: 计算机编码
category: 原来如此
tags: [编码, python]
---

###遇到的情况
用户给商品添加评价，结果报错：

    uncategorized SQLException for SQL []; SQL state [HY000]; error code [1366]; Incorrect string value: '\xF0\x9F\x8E\x81\xE7\x9A...'

java后端采用utf8编码，mysql也采用utf8编码，都正常，怎么会报错呢？

先来看看 \'\xF0\x9F\x8E\x81\xE7\x9A...\'是什么：

使用python，将字节还原成utf8编码，然后再打印出来：

![unicode](/assets/images/unicode.png)

去[这里](http://www.utf8-chartable.de/unicode-utf8-table.pl)查询知道，\U0001f381 代表 WRAPPED PRESENT，以utf8编码后16进制为：f0 9f 8e 81， 长度为四个字节，再去mysql官网[查询](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8.html), mysql的utf8编码实际仅仅支持最多3个字节的编码！

如何解决这个问题呢？可以设置Mysql服务器编码为utf8mb4，也可以在业务层过滤掉超过四个字节的编码。

把unicode范围\u0000-\uFFFF内的字符替换掉，来自stackoversflow的[代码](http://stackoverflow.com/questions/14981109/checking-utf-8-data-type-3-byte-or-4-byte-unicode)：

    public static String withNonBmpStripped( String input ) {
            if( input == null ) throw new IllegalArgumentException("input");
                return input.replaceAll("[^\\u0000-\\uFFFF]", "");
    }

或者适用java内置的方法：

    public static boolean isEntirelyInBasicMultilingualPlane(String text) {
        for (int i = 0; i < text.length(); i++) {
            if (Character.isSurrogate(text.charAt(i))) {
                return false;
            }
        }
        return true;
    }

###什么是编码

信息是什么？对计算机来说，是一系列的0、1串，而对于人来说，则是一系列可以理解的数字、符号、表情等。

所谓编码(codec)，是以某种规则（映射关系）把信息从一种形式对应到另外一种形式的过程。

###字符集
字符集，信息表示形式的集合。字符集的概念依托于编码(codec)，有了编码才会有字符集。每种编码都有一个有限的表示集合。比如二进制编码的字符集是\{0, 1\}，gbk编码的字符集是一系列的中文符号。

###二进制文件与文本文件
二进制文件可以包含任何形式的数据，比如图形、声音，以二进制编码。

文本文件是某种字符编码的文件，仅仅包含这个编码字符集内的字符，例如ASCII文本文件里面不会出现超过127的字符。

###Unicode与UTF-8、UTF-16
Unicode， 统一（唯一）编码，能表示所有字符的字符集，不仅仅局限于某国语言。

Unicode仅仅是一个概念，不能被电脑表示、不能用来传输，于是就有了UTF(Unicode Transfer Format)，将unicode转换成1到6个字节。

###源文件编码
源文件编码(source code encode)， 由操作系统默认编码和编辑器决定。编写完代码点击保存时，编辑器会以特定的编码（通常是操作系统默认编码）来保存文件。

有的编辑器会在源文件开头增加编码标识，以便读取时以此种编码标识解码。

###运行时编码
程序运行时采用的一种编码，相对程序来说，通常与源文件编码不同。如python运行时以unicode来表示所有的字符。

###参考
1、[All About Python and Unicode](http://pythonic.zoomquiet.io/data/20110415091609/index.html)

2、[Unicode-utf8字符表](http://www.utf8-chartable.de/unicode-utf8-table.pl)

3、[Wikipedia:Plane_Unicode](http://en.wikipedia.org/wiki/Plane_(Unicode))

4、[Mysql The utf8 Character Set (3-Byte UTF-8 Unicode Encoding)](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8.html)

