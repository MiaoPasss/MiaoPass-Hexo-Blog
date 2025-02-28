---
title: CNAME和域名解析
date: 2025-02-24 13:34:05
categories: 计算机网络
tags: 
- 域名解析
---

# 域名解析协议 Domain Name System
1. DNS让用户能够通过域名访问网页(www.baidu.com -> 36.152.44.95)
2. 计算机会先将域名发送到一个解析域名的服务器上
    2.1. 在网络上有很多服务器，能解析各种各样的域名，比如有专门解析.org的，解析.com的，解析.net的。等等，最主要的有一个根域名服务器(Root Name Server)
    2.2. 域名解析(在服务器上查找IP地址)的过程有两种算法，迭代查询，递归查询。一般是两种查询的结合
    2.3. 本机计算机找到其中一台解析域名的服务器(可能是.com)，如果没有找到对应的IP地址，那么就会去找根域名服务器，根域名服务器知道所有的子服务器，
    所以他肯定知道该域名所对应的IP地址在那个子服务器中，所以告诉第一次查询的服务器要他去另一台服务器上找，找到了，就将其返回给计算机，
    以后在有另一台计算机也通过这个域名访问，那么第一台服务器会有原来的域名IP地址的缓存，就不用去找根服务器了。
    2.4. 找到了，就能找到我们要访问的服务器了。

---

# 根域名
![图片](RootDomain.jpg "域名结构")
Root Domain is the highest hierarchical level of a site and is separated from the Top Level Domain by a dot (e.g. rootdomain.com).

---

# A 记录
Address Record indicates the IP address of a given domain. For example, if you pull the DNS records of cloudflare.com, the A record currently returns an IP address of: 104.17.210.9. A records only hold IPv4 addresses. If a website has an IPv6 address, it will instead use an "AAAA" record.

![图片](ARecord.png "A记录组成部分")

The "@" symbol in this example indicates that this is a record for the root domain, and the "14400" value is the TTL (time to live), listed in seconds. The default TTL for A records is 14,400 seconds. This means that if an A record gets updated, it takes 240 minutes (14,400 seconds) to take effect.

---

# 内容分发网络
内容分发网络 (CDN) 是一个分布在不同地理位置的服务器群，用于缓存靠近最终用户的内容。CDN 可以快速传输加载互联网内容所需的资产，包括 HTML 网页、JavaScript 文件、样式表、图像和视频。

---

# CNAME 别名记录
CNAME记录，也叫别名记录，相当于给A记录中的域名起个小名儿，比如www.xx.com的小名儿就叫www.yy.com好了，然后CNAME记录也和A记录一样，是一种指向关系，把小名儿www.yy.com指向了www.xx.com，然后通过A记录，www.xx.com又指向了对应的IP：

> www.yy.com → www.xx.com → 1.1.1.1

这样一来就能通过它的小名儿直接访问1.1.1.1了。

<br>

## 多个域名指向同一个地址

>www.yy.com → www.xx.com → 1.1.1.1
www.cc.com → www.xx.com → 1.1.1.1
www.kk.com → www.xx.com → 1.1.1.1

突然服务器的IP地址因为一些不可描述的原因要换了，不再是1.1.1.1了，换成了2.2.2.2，这时候你发现，只要把www.xx.com的指向修改一下即可：

> 域名 www.xx.com → 2.2.2.2
这时候你又发现了，原来他的小名儿不需要做更改，直接就能访问服务器，因为他们都只指向了www.xx.com，服务器IP改没改它们不管。

那么假如不用CNAME，直接做A记录，那么当1.1.1.1更改的时候，全部相关A记录指向关系都要做更改
> www.yy.com → 1.1.1.1
www.cc.com → 1.1.1.1
www.xx.com → 1.1.1.1
www.kk.com → 1.1.1.1

<br>

## 使用CDN

假如你是DD公司老板，你公司中的一台IP为1.1.1.1的服务器，注册了域名为www.dd.com，要对外提供客户访问。随着公司越做越大，访问量也越来越多，服务器顶不住了，你去找CDN提供商购买CDN加速服务，这个时候他们要求你的域名做个CNAME指向他们给你的一个域名叫www.dd.cdn.com

> www.dd.com → www.dd.cdn.com

当用户访问www.dd.com的时候，本地DNS会获得CDN提供的CNAME域名：www.dd.cdn.com，然后再次向DNS调度系统发出请求，通过DNS调度系统的智能解析，把离客户端地理位置最近的（或者相对负载低的，主要看CDN那边智能解析的策略）CDN提供商的服务器IP返回给本地DNS，然后再由本地DNS回给客户端，让用户就近取到想要的资源（如访问网站），大大降低了延迟。

![图片](CDN.png "使用CNAME配置使用CDN服务器")

---

# Credits

域名解析：https://www.cloudflare.com/learning/dns/what-is-dns/
CNAME别名记录：https://blog.csdn.net/DD_orz/article/details/100034049

---