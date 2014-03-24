---
layout: post
title: Django Debug Toolbar 的使用和改进
---

#. 安装
--------------

```
$ pip install django_debug_toolbar 
```

#. 配置
-------------
虽然debug toolbar在非DEBUG状态下是不启用的，但是我还是建议在settings.py显示的注明这一点，如下所示:
在 `settings.py` 文件的末尾添加下面几行：

{% gist 6093747 %}

这个时候访问网站，在右边栏就会出现一系列工具，如下图所示：

![django debug toolbar](/assets/images/django_debug_toolbar.png)

# 改进
-------------------
django debug toolbar 对于json数据、gzip数据是不会返回toolbar的，但是我们写api的时候，通常都会返回gzip的json数据，因此我们需要做一些改进。

1) 编写一个新的middleware

{% gist 5567579 %}

2) 将该middleware中加入到settings.py中，注意：确保该middleware在 `debug_toolbar.middleware.DebugToolbarMiddleware` 的后面。

现在，正常访问api url的时候，返回json数据。

当在api的url中加上GET参数debug＝true的时候，则返回带有toolbar的数据。如下图所示：

![django debug toolbar 4 json](/assets/images/django_debug_toolbar_4_json.png)
