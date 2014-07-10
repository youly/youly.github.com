---
layout: post
title: https通信原理
category: 协议
tags: [protocol, https]
---

###什么是https
[https](http://en.wikipedia.org/wiki/HTTP_Secure)，简单的说就是HTTP协议+SSL协议。由于使用HTTP协议传输的数据没有加密，有可能被第三方窃取甚至修改，Netscape公司于1994年创造了https。SSL协议在通信过程中起了两个作用：

1、确保通信双方身份没有伪造，通过数字证书实现。

2、确保与服务器之间的通讯不能被第三方窃取、修改。

了解https通信过程还需知道两种加密方式：非对称加密和对称加密。客户端与服务器进行安全数据交互之前，需在非对称秘钥的加密保护下生成用于会话期数据加密的对称秘钥。

###SSL连接建立过程
客户端和服务器交换数据之前有个握手协议，具体过程是：

1、客户端初始化支持的加密算法、压缩算法、SSL协议版本，并向服务器发起消息结构体为client hello的连接

2、服务端收到消息，根据客户端提供的加密算法列表、压缩算法列表来选取后续使用的对称加密算法、压缩算法和SSL协议版本，并返回结构体为server hello的消息。

3、如果客户端需要服务器身份认证，服务器在返回server hello的消息后需立刻返回数字证书信息（通常服务器认证是必须的），并发送hello done的消息。

4、客户端验证证书是否有效，并发送用证书公钥加密的对称秘钥。

5、服务器收到加密的对称秘钥，使用自己的私钥解密，并发送通过对称秘钥加密的的会话完成信息。

握手完成之后客户端和服务器可以安全交换信息了。

###什么是数字证书
数字证书之于服务器，就像身份证之于人。数字证书由被公认的证书机构颁发。通常操作系统里都会内置一些知名证书机构的根证书。证书内容由以下几个部分组成（[来自wikipedia](http://en.wikipedia.org/wiki/Public_key_certificate)）：

>Serial Number: Used to uniquely identify the certificate.
<br />Subject: The person, or entity identified.
<br />Signature Algorithm: The algorithm used to create the signature.
<br />Signature: The actual signature to verify that it came from the issuer.
<br />Issuer: The entity that verified the information and issued the certificate.
<br />Valid-From: The date the certificate is first valid from.
<br />Valid-To: The expiration date.
<br />Key-Usage: Purpose of the public key (e.g. encipherment, signature, certificate signing...).
<br />Public Key: The public key.
<br />Thumbprint Algorithm: The algorithm used to hash the public key certificate.
<br />Thumbprint (also known as fingerprint): The hash itself, used as an abbreviated form of the public key certificate.

从上面的引用中知道，证书主要由三部分组成：证书内容、证书签名、证书指纹。那么拿到一个证书，如何验证证书的有效性呢？先看下面的一幅图：
![签名和认证](/assets/images/digtal_certificate_sign.png)

颁发证书的机构会将证书内容使用指定的哈希函数生成内容hash，然后用自己的私钥加密哈希形成签名。证书指纹用于确定证书没有被修改。解密验证过程如上右图，如果最终得到的签名与客户端使用指定的签名算法得到的签名不一致，则证书是不可信的。

一般我们得到的证书都不是根证书机构直接颁发的，而是经过了中间机构。这里有一个信任链机制，客户端或从证书链的底端逐级网上找到根证书颁发机构，然后使用颁发机构的公钥逐级往下验证证书的有效性，过程如下图：
![证书链](/assets/images/digtal_certificate_sign1.gif)

###参考
1、[How does HTTPS actually work?](http://robertheaton.com/2014/03/27/how-does-https-actually-work/)

2、[WikiPedia:HTTP_Secure](http://en.wikipedia.org/wiki/HTTP_Secure)

3、[数字证书原理](http://www.cnblogs.com/JeffreySun/archive/2010/06/24/1627247.html)

4、[Debugging SSL communications](http://prefetch.net/articles/debuggingssl.html)

5、[应用 openssl 工具进行 SSL 故障分析](http://www.ibm.com/developerworks/cn/linux/l-cn-sclient/)

6、[证书链(The Certificate Chains)](http://lukejin.iteye.com/blog/587200)

7、[How to create a self-signed SSL Certificate](http://www.akadia.com/services/ssh_test_certificate.html)
