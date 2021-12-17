---
layout: post
title: "JMeter中cookie未随请求一起发送"
subtitle: ""
author: ""
header-style: text
tags:
  - jmeter
  - 测试
---



![](/blog/img/20190116235505954.png)

可看到 Cookie Policy那里有很多种策略可选，默认是standard，我这里换成了netscape才生效，反正发现不起作用时各种换着试一试。

下面是各种策略的说明：

- Compatibility

  > 这种兼容性设计要求是适应尽可能多的不同的服务器，尽管不是完全按照标准来实现的。如果你遇到了解析Cookies的问题，你就可能要用到这一个规范。
  > 有太多的web站点是用CGI脚本去实现的，而导致只有将所有的Cookies都放入Request header才可以正常的工作。这种情况下最好设置http.protocol.single-cookie-header参数为true。

- RFC2965

  > RFC2965定义了版本2并且尝试去弥补在版本1中Cookie的RFC2109标准的缺点。RFC2965是，并规定RFC2965最终取代RFC2109.
  > 发送RFC2965标准Cookies的服务端，将会使用Set-Cookie2 header添加到Set-Cookie Header信息中，RFC2965 Cookies是区分端口的。

- Ignore Cookies

  > 此规格忽略所有Cookie 。被用来防止HttpClient接受和发送的Cookie。

- Netscape

  > Netscape是最原始的Cookies规范，同时也是RFC2109的基础。尽管如此，还是在很多重要的方面与RFC2109不同，可能需要特定服务器才可以兼容。

- RFC2109

  > RFC2109是W3C组织第一次推出的官方Cookies标准。理论上，所有使用版本1Cookies的服务端都应该使用此标准。HttpClient已经将此标准设定为默认。
  >
  > 遗憾的是，许多服务端不正确的实现了标准或者仍然使用Netscape标准。所以有时感到此标准太多于严格。
  > RFC2109是HttpClient使用的默认Cookies协议。