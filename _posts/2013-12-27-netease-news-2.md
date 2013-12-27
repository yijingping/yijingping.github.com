---
layout: post
title: 抓包分析网易新闻app (下)
---

上篇分析了`网易新闻`app的基本结果, 下篇主要分析每个HTTP请求, 并找出其中采用的优化措施

# 1 列表页分析 
--------------

选择菜单`体育`列表页, url为: [http://c.m.163.com/nc/article/list/T1348649079062/0-20.html](http://c.m.163.com/nc/article/list/T1348649079062/0-20.html)

## 1.1 请求

    GET /nc/article/list/T1348649079062/0-20.html HTTP/1.1
    Host: c.m.163.com
    Connection: Keep-Alive
    User-Agent: NTES Android
    Accept-Encoding: gzip
    
1) Connection: Keep-Alive

重用连接, 现在的nginx默认就是打开的

2) Accept-Encoding: gzip

接受gzip压缩的数据,可以节约2/3的带宽

## 1.2 响应

    HTTP/1.1 200 OK
    Server: nginx
    Date: Fri, 27 Dec 2013 14:24:31 GMT
    Content-Type: text/html; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Vary: Accept-Encoding
    Last-Modified: Fri, 27 Dec 2013 14:20:31 GMT
    Expires: Fri, 27 Dec 2013 14:28:31 GMT
    Cache-Control: max-age=240
    via: c-3g-163-com-52.75
    X_cache: HIT from zeta-c3g1.sa.bgp.hz
    Content-Encoding: gzip

    cd4
    ... gzip data ...
    0

1) Content-Type: text/html; charset=UTF-8

奇怪, 应该用json才对啊

2) Transfer-Encoding: chunked

分块传输. 但是本次请求是一次传输完的. 因为0xcd4 表示3284个字节, 最后的0表示还剩0个字节. 


3) Connection: keep-alive

保持连接, 省略断开并重新建立连接的时间

3) 本地缓存

    Last-Modified: Fri, 27 Dec 2013 14:20:31 GMT
    Expires: Fri, 27 Dec 2013 14:28:31 GMT
    Cache-Control: max-age=240

Expires是HTTP1.0就有的, Cache-Control从HTTP1.1才有, 如果客户端是HTTP1.0版本的, 本地缓存会在27 Dec 2013 14:28:31过期.

如果客户端是HTTP1.1的,Cache-Control会重写Expires规则, 也就是在接受到数据的4分钟(240秒)后过期.

本地缓存4分钟,或者在14:28:31秒过期后,客户端访问服务器,并且加上If-Modified-Since(值等于Last-Modified), 如果没有更改,服务端可以返回304 Not Modified

4) 这是cdn还是squid/varnish

    via: c-3g-163-com-52.75
    X_cache: HIT from zeta-c3g1.sa.bgp.hz

5) 数据压缩 

   Content-Encoding: gzip
   Vary: Accept-Encoding

gzip压缩,能节约2/3的带宽. 通过wireshark可以看出, HTTP响应的数据部分3284字节经过gzip解压后,还原为11576字节

## 1.3 第2次请求

这次请求比上次请求多了一个`If-Modified-Since`字段

    If-Modified-Since: Fri, 27 Dec 2013 14:20:31 GMT
    GET /nc/article/list/T1348649079062/0-20.html HTTP/1.1
    If-Modified-Since: Fri, 27 Dec 2013 14:20:31 GMT
    Host: c.m.163.com
    Connection: Keep-Alive
    Accept-Encoding: gzip


## 1.4 第2次响应

由于第二次请求的`If-Modified-Since` < `Last-Modified`, 所以返回新数据

    HTTP/1.1 200 OK
    Server: nginx
    Date: Fri, 27 Dec 2013 15:54:57 GMT
    Content-Type: text/html; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Vary: Accept-Encoding
    Last-Modified: Fri, 27 Dec 2013 15:48:26 GMT
    Expires: Fri, 27 Dec 2013 15:58:57 GMT
    Cache-Control: max-age=240
    via: c-3g-163-com-21.103
    X_cache: HIT from zeta-c3g1.sa.bgp.hz
    Content-Encoding: gzip

    cd0
    ... gzip data ...
    0

## 1.5 304 Not Modified

我们在第3次请求发出后和2分钟内进行第4次请求, 发现返回304. 并没有重新发送数据, 说明本地缓存起效了.

但是我们第4次响应的Expires和第3次的不同, 这是设计中要相当小心的地方

    // 第3次请求 
    GET /nc/article/list/T1348649079062/0-20.html HTTP/1.1
    If-Modified-Since: Fri, 27 Dec 2013 15:48:26 GMT
    Host: c.m.163.com
    Connection: Keep-Alive
    User-Agent: NTES Android
    Accept-Encoding: gzip
    
    // 第3次响应 
    HTTP/1.1 200 OK
    Server: nginx
    Date: Fri, 27 Dec 2013 16:03:31 GMT
    Content-Type: text/html; charset=UTF-8
    Transfer-Encoding: chunked
    Connection: keep-alive
    Vary: Accept-Encoding
    Last-Modified: Fri, 27 Dec 2013 16:00:36 GMT
    Expires: Fri, 27 Dec 2013 16:07:31 GMT
    Cache-Control: max-age=240
    via: c-3g-163-com-21.104
    X_cache: HIT from zeta-c3g1.sa.bgp.hz
    Content-Encoding: gzip
    
    cd3
    ... gzip data ...
    0
    
    // 第4次请求 
    GET /nc/article/list/T1348649079062/0-20.html HTTP/1.1
    If-Modified-Since: Fri, 27 Dec 2013 16:00:36 GMT
    Host: c.m.163.com
    Connection: Keep-Alive
    User-Agent: NTES Android
    Accept-Encoding: gzip
    
    // 第4次响应 
    HTTP/1.1 304 Not Modified
    Server: nginx
    Date: Fri, 27 Dec 2013 16:03:44 GMT
    Connection: keep-alive
    Last-Modified: Fri, 27 Dec 2013 16:00:36 GMT
    Expires: Fri, 27 Dec 2013 16:07:44 GMT
    Cache-Control: max-age=240
    via: c-3g-163-com-21.104
    X_cache: HIT from zeta-c3g1.sa.bgp.hz

# 2 单页分析

单页和列表页没有什么区别,就是缓存时间由4分钟提高到10分钟

# 3 图片分析

图片和内容来自于不同的域名, 因此要将wireshark的过滤器设置为:`http`, 然后选取获取图片的请求和响应

例如url: [http://s.cimg.163.com/i/img3.cache.netease.com/2008/2013/12/27/2013122716280472651.jpg.320x10000.50.auto.jpg](http://s.cimg.163.com/i/img3.cache.netease.com/2008/2013/12/27/2013122716280472651.jpg.320x10000.50.auto.jpg)

## 3.1 请求

    GET /i/img3.cache.netease.com/2008/2013/12/27/2013122716280472651.jpg.320x10000.50.auto.jpg HTTP/1.1
    Host: s.cimg.163.com
    Connection: Keep-Alive
    Accept-Encoding: gzip

## 3.2 响应

    HTTP/1.1 200 OK
    Server: nginx
    Date: Fri, 27 Dec 2013 10:32:00 GMT
    Content-Type: image/jpeg
    Content-Length: 7739
    Etag: "bf31e5c19a5a8254022e1b89c511b1d109f2c2c0"
    Expires: Sat, 27 Dec 2014 10:32:00 GMT
    Cache-Control: max-age=31536000
    Powered-By-ChinaCache: HIT from 010519735a
    Age: 21787
    Powered-By-ChinaCache: HIT from 01001073S6

    ...data...

1) 本地缓存

    Expires: Sat, 27 Dec 2014 10:32:00 GMT
    Cache-Control: max-age=31536000

本地缓存时间为1年   

2) Etag 

    Etag: "bf31e5c19a5a8254022e1b89c511b1d109f2c2c0"

Etag一般设置为图片的md5. 本地缓存过期后, 再请求该图片,可以根据Etag值判断图片是否修改过. 如果Etag相同, 则未修改, 返回304.

3) CDN

    Powered-By-ChinaCache: HIT from 010519735a
    Age: 21787
    Powered-By-ChinaCache: HIT from 01001073S6

使用了[ChinaCache](http://cn.chinacache.com/)的CDN技术, 使用户从最近的节点获取图片

`Age: 21787`表示该图片在代理(CDN)服务器上从刚生成到现在经历了21787秒
