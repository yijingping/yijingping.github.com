如何使用django debug toolbar
====================================

# 安装
--------------
```
pip install django_debug_toolbar 
```

# 配置
-------------
虽然debug toolbar在非DEBUG状态下是不启用的，但是我还是建议在settings.py显示的注明这一点，如下所示:
在`settings.py`文件的末尾添加下面几行：
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
Format: ![django debug toolbar](#)



