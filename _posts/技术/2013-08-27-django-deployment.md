---
layout: post
title: Nginx + svc + uWSGI + virtualenv 部署 Django 服务
category: 技术
tags: Django
---

原来公司部署Django采用的是Apache+mod_wsgi，后来为了降低服务器压力和增强并发能力，
改用Nginx+uWSGI。

网上有很多中uWSGI+Django的部署方式，但是要注意Django的版本，Django < 1.4 的 部
署和之后的版本略有不同，部署前最好从官方的[uWSGI文档]入手。

最近我刚刚使用Nginx + (svc+uWSGI) + virtualenv +Django1.3 完成了一次部署，这里做一下总结。

[uWSGI文档]: http://projects.unbit.it/uwsgi/ 

# Nginx
    
    upstream mapi_99fang_com_uwsgi {
        server 127.0.0.1:5921;
    }

    server {
        listen          82;
        server_name     xxx.com;
        access_log      /var/log/nginx/mapi_uwsgi_access.log mapi_combined;
        error_log       /var/log/nginx/mapi_uwsgi_error.log;

        location / {
            proxy_next_upstream error timeout http_500 http_503;
            proxy_connect_timeout 4000ms;
            proxy_read_timeout    30s;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Rewrite-URL $request_uri;
            client_max_body_size 10m;

            uwsgi_pass  mapi_99fang_com_uwsgi;
            include uwsgi_params;
        }
    }
 

# svc和uWSGI 

uwsgi的配置参考了[Deploying Django] (http://uwsgi-docs.readthedocs.org/en/latest/WSGIquickstart.html#deploying-django)，
注意django版本>=1.4和<1.4的区别

编辑uwsgi.ini

    [uwsgi]
    # the socket (use the full path to be safe
    socket = 0.0.0.0:5921
    # the virtualenv (full path)
    virtualenv = /opt/tiger/ve1.3
    # the base directory (full path)
    chdir = /opt/tiger/mobile_api/module
    # Django-related settings
    env = DJANGO_SETTINGS_MODULE=settings
    # Django's wsgi file
    module =  django.core.handlers.wsgi:WSGIHandler()
    # process-related settings
    # maximum number of worker processes
    processes = 6
    # threads = 2
    stats = 127.0.0.1:9191

在svc service目录下建立mapi_run目录，文件下新建文件run(可执行文件权限)

    ├── svc_services/
            └── mapi_run/
                    └── run 

run的内容如下：
    
    exec /opt/tiger/ve1.3/bin/uwsgi --ini=/opt/tiger/ve1.3/lib/python2.7/deploy/mobile_api/uwsgi.ini 1> /dev/null 2>> /var/log/tiger/uwsgi_mobile_api.log
    
然后用svc的方式启用uwsgi

    $ cd svc_services
    $ svc -u mapi_run 

下面几个命令会用到:

    $ svcstat mapi_run # 查看mapi_run服务运行状态
    $ svc -d mapi_run # stop mapi_run服务
    $ svc -u mapi_run # stop mapi_run服务

更多svc命令参考 [the svc program]

[the svc program]: http://cr.yp.to/daemontools/svc.html 
