---
layout: post
title: 抓包分析网易新闻app (上)
---

通过wireshark抓包来分析`网易新闻`这个app

# 1 准备 
--------------

在eclipse打开一个AVD, 安装`网易新闻`app, 然后打开wireshark, 监听电脑的数据流


# 2 猜测app使用的协议 域名和ip
----------------
我在wireshark中找出了一个类似是`网易新闻`app的请求, 如下:

![](/assets/images/netease_news/firstfind.png)

从该请求可得到基本信息如下:

    scheme: HTTP
    host: c.3g.163.com
    ip: 123.58.189.26

再查看一些其他请求, 发现还有一些其他域名和ip

如c.m.163.com和img4.cache.netease.com, 123.58.189.25

整理后,发现主要的请求来自c.m.163.com和c.3g.163.com, 这两个域名都映射到123.58.189.25 123.58.189.26两个ip, 部分图片的请求来自img4.cache.netease.com, 可以忽略

因此我们假设过滤条件为: 

    所有ip去往123.58.189.26/25的HTTP请求
    或者
    所有ip来自123.58.189.26/25的HTTP响应


# 3 设置过滤器,监测app启动请求

在检测的时候将过滤器设置为: 

    http and (ip.dst == 123.58.189.26 or ip.dst == 123.58.189.25 or ip.src == 123.58.189.26 or ip.src == 123.58.189.25)


在`网易新闻`app中删除缓存,重新打开app, 检测到多条HTTP请求和响应

![](/assets/images/netease_news/1-4.png)

可以在浏览器中输入请求的url, 查看其数据. (当然,在wireshark中也能看,但是没有浏览器直观, 建议在chrome浏览器中安装jsonview插件查看)

## 3.1 请求1 和 响应1

url: [http://c.3g.163.com/nc/uc/prize/list/](http://c.3g.163.com/nc/uc/prize/list/)

内容猜测:  个人中心的活动信息

android截图: 

![](/assets/images/netease_news/ns1.png)


## 3.2 请求2 和 响应2

url: [http://c.3g.163.com/mobilerec/mobile?uid=000000000000000](http://c.3g.163.com/mobilerec/mobile?uid=000000000000000)

内容猜测: 主页列表为我推荐数据 


## 3.3 请求3 和 响应3

url: [http://c.m.163.com/nc/ad/headline/android/0-3.html](http://c.m.163.com/nc/ad/headline/android/0-3.html)

内容猜测: 主页列表顶部图片轮播数据, 显示0-3条 

## 3.4 请求4 和 相应4

url: [http://c.m.163.com/nc/article/headline/T1347415223240/0-20.html](http://c.m.163.com/nc/article/headline/T1347415223240/0-20.html) 

内容猜测:  头条列表(headline/T1348649079062)的0-20条的数据, T1348649079062表示分类. 将url中的headline替换为list, 得到的数据相同,说明headline和list的处理方法是相同的.

android截图: 

![](/assets/images/netease_news/ns2.png)

# 4 监测app菜单切换请求

下面尝试切换主页的菜单, 从头条切换到娱乐, 再切换到体育,检测到2条HTTP请求和2条HTTP响应

## 4.1 菜单`娱乐`项的请求和响应

url: [http://c.m.163.com/nc/article/list/T1348648517839/0-20.html](http://c.m.163.com/nc/article/list/T1348648517839/0-20.html)

内容猜测: 娱乐列表(T1348648517839), 从第0条开始,显示20条数据, 服务端已经做了静态化处理

android截图: 

![](/assets/images/netease_news/ns3.png)


## 4.2 菜单`体育`项 请求和响应

url: [http://c.m.163.com/nc/article/list/T1348649079062/0-20.html](http://c.m.163.com/nc/article/list/T1348649079062/0-20.html)

内容猜测: 体育列表(T1348649079062), 从第0条开始,显示20条数据, 服务端也做了静态化处理

android截图: 

![](/assets/images/netease_news/ns4.png)

## 4.3 监测app新闻详情页请求

从列表页直接跳到详情页

url: [http://c.m.163.com/nc/article/9H36QHED00963VRR/full.html](http://c.m.163.com/nc/article/9H36QHED00963VRR/full.html)

内容猜测: 新闻详情(article)(9H36QHED00963VRR), 服务端已经做了静态化处理, 9H2G8JKE00964KL5可能是文章id的md5

android截图: 

![](/assets/images/netease_news/ns5.png)

# 5 监测app专题列表

从头条列表中点击某个专题,进入专题列表(special)

url: [http://c.m.163.com/nc/special/S1388104506515.html](http://c.m.163.com/nc/special/S1388104506515.html)

其中S1388104506515, 表示当前专题的id. 

# 6 监测翻页

在菜单`娱乐`列表中, 往下翻两页

第1页url: [http://c.m.163.com/nc/article/list/T1348648517839/0-20.html](http://c.m.163.com/nc/article/list/T1348648517839/0-20.html)

第2页url: [http://c.m.163.com/nc/article/list/T1348648517839/20-20.html](http://c.m.163.com/nc/article/list/T1348648517839/20-20.html)

第3页url: [http://c.m.163.com/nc/article/list/T1348648517839/40-20.html](http://c.m.163.com/nc/article/list/T1348648517839/40-20.html)

wireshark截图:

![](/assets/images/netease_news/ns6.png)

可以看出使用类似 `where type = 1348668517839 limit 0, 20` 语句来进行分页的. 其中1348668517839是类型,不是时间戳, 因为我上午和下午访问时,这个类型字段都没有变,但是url对应的内容已经更新成最新的了.

实际上,这样子会有问题, 如果用户访问第1页和第2页之间有新数据插入, 则访问第2页的时候会可能会出现重复的数据. 解决的方法是增加一个排序字段,如id或者时间戳, 并将查询的类型改为: `where time < atimestamp and type = 1348668517839 limit 0, 20`
