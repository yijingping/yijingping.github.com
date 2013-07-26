使用django debug toolbar
====================================

# 安装
--------------

```
pip install django_debug_toolbar 
```

# 配置
-------------
虽然debug toolbar在非DEBUG状态下是不启用的，但是我还是建议在settings.py显示的注明这一点，如下所示:
在 `settings.py` 文件的末尾添加下面几行：

```
# add for debug toolbar
if DEBUG:
    MIDDLEWARE_CLASSES += ('debug_toolbar.middleware.DebugToolbarMiddleware',)
    INTERNAL_IPS = ('60.247.90.134',)
    INSTALLED_APPS += ( 'debug_toolbar',)
    DEBUG_TOOLBAR_PANELS = (
        'debug_toolbar.panels.version.VersionDebugPanel',
        'debug_toolbar.panels.timer.TimerDebugPanel',
        'debug_toolbar.panels.settings_vars.SettingsVarsDebugPanel',
        'debug_toolbar.panels.headers.HeaderDebugPanel',
        'debug_toolbar.panels.request_vars.RequestVarsDebugPanel',
        'debug_toolbar.panels.template.TemplateDebugPanel',
        'debug_toolbar.panels.sql.SQLDebugPanel',
        'debug_toolbar.panels.signals.SignalDebugPanel',
        'debug_toolbar.panels.logger.LoggingPanel',
    )
```

这个时候访问网站，在右边栏就会出现一系列工具，如下图所示：

![django debug toolbar](/images/django_debug_toolbar.png)

# 改进
-------------------
django debug toolbar 对于json数据、gzip数据是不会返回toolbar的，但是我们写api的时候，通常都会返回gzip的json数据，因此我们需要做一些改进。

1) 编写一个新的middleware, 下载地址：[debugtoolbar4json.py](https://gist.github.com/yijingping/5567579)

```
# -*- coding: utf-8 -*-                                                         
from __future__ import unicode_literals                                         
from cStringIO import StringIO                                                  
import gzip                                                                     
from django.http import HttpResponse                                            
                                                                                
RESPONSE_TYPES = ('application/json',)                                          
class DebugToolbar4JsonMiddleware(object):                                      
    def process_response(self, request, response):                              
        if request.GET.get('debug') == 'true':                                  
            if 'gzip' in response.get('Content-Encoding', ''):                  
                sio = StringIO(response.content)                                
                f = gzip.GzipFile(fileobj=sio)                                  
                content = f.read()                                              
                f.close()                                                       
            else:                                                               
                content = response.content                                      
                                                                                
            if response.get('Content-Type', '').split(';')[0] in RESPONSE_TYPES:
                return HttpResponse('''<html><body> %s </body></html>'''%content)
                                                                                
        return response                                                         
```

2) 将该middleware中加入到settings.py中，注意：确保该middleware在 `debug_toolbar.middleware.DebugToolbarMiddleware` 的后面。

现在，正常访问api url的时候，返回json数据。

当在api的url中加上GET参数debug＝true的时候，则返回带有toolbar的数据。如下图所示：

![django debug toolbar 4 json](/images/django_debug_toolbar_4_json.png)

