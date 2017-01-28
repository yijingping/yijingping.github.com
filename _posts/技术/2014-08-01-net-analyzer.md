---
layout: post
title: 常用的抓包软件
category: 技术
tags: 抓包
---

做web开发或者app开发的时候，经常会有抓包的需求。下面总结一下我用过的抓包工具，并分析一下优缺点。

# WireShark

Wireshark一直是用的最爽的一个工具，在max和windows下都有一致的界面。

唯一要注意的是Wireshark依赖X11，而OS X Mountain Lion后OS X不再附带X11。因此，需要先安装XQuartz，然后再安装Wireshark。
先启动XQuartz，然后在XQuartz中配置Wireshark的启动路径：```/Applications/Wireshark.app/Contents/MacOS/Wireshark```，以后每次在XQuartz中启动Wireshark即可。

另外，Wireshark没有自带proxy，因此需要自己搭建proxy。

# Fiddler

最好用的抓包工具，支持https、proxy。但是是基于.Net 技术开发的，没办法直接在 Mac/Linux 下使用。

# Charles

自带proxy，支持拦截并修改response，非常好用的一个工具，支持mac。

但是只有30天试用，收费的是50刀。网上有免费得注册码。

一个缺陷是，很容易丢包。像高德地图的api一个都没抓到，百度地图的却都抓到了。

Charles支持Https抓包，但需要在手机上安装一个证书（如果不安转这个证书，在Charles中看到的request和response是乱码）::

```
    下载Charles证书http://www.charlesproxy.com/ssl.zip，解压后导入到iOS设备中（将crt文件作为邮件附件发给自己，再在iOS设备中点击附件即可安装；也可上传至dropbox之类的网盘，通过safari下载安装）
    在Charles的工具栏上点击设置按钮，选择Proxy Settings…
    切换到SSL选项卡，选中Enable SSL Proxying，别急，选完先别关掉，还有下一步
    这一步跟Fiddler不同，Fiddler安装证书后就可以抓HTTPS网址的包了，Charles则麻烦一些，需要在上一步的SSL选项卡的Locations表单填写要抓包的域名和端口，点击Add按钮，在弹出的表单中Host填写域名，比如填api.instagram.com，Port填443
```


# Rythem

也是再带proxy，使用QT写的，支持mac。但是功能太简单，因此没怎么用。
