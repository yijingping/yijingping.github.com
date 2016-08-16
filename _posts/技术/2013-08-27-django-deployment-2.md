---
layout: post
title: Nginx + Supervisor + uWSGI 部署 Django 服务
category: 技术
tags: Django
---
这一篇介绍使用 Nginx + Supervisor + uWSGI 部署 Django 服务。 

高德地图的服务器比较多，仅web服务就100多台，每台服务器的工作都比较单一，所以就没有部署virtualenv。

因此我们为了保证服务器的纯净，也可以不用virtualenv。

下面演示一下，我们的微博爬虫平台的部署

# 下载工程
一般我们的web工程都放在 `/var/www`下面，`0fenbei`是一个组，所以把`weibo-spider`放在`/var/www/0fenbei/`目录下。 

	cd /var/www/0fenbei/
	git clone https://github.com/bowenpay/weibo-spider.git

在工程目录下，有以下文件：

    ├── weibo-spider/
            ├── nginx.conf
            ├── supervisor.conf
            ├── uwsgi.ini
            └── requirements.txt

包含了下面所有的用到的配置文件.


# uWsgi

其具体配置如下:

    [uwsgi]
    chdir=/var/www/0fenbei/weibo-spider
    py-autoreload=3  #实现和django自带server一样更新文件自动重启功能
    module=weibospider.wsgi:application
    master=True
    pidfile=/tmp/weibospider.pid
    vacuum=True   # clear environment on exit
    socket=127.0.0.1:49160
    processes=1    # 启动5个进程
    harakiri=20 # respawn processes taking more than 20 seconds
    max-requests=5000  # 请求5000次后重启
    # daemonize=/var/log/bowenpay/weibo-spider-uwsgi.log # 不使用daemon模式，防止supervisor自动重启

可以通过命令来测试配置是否正常

    # 启动web服务
    uwsgi --http-socket 0.0.0.0:3031 --chdir /var/www/0fenbei/weibo-spider/ --wsgi-file weibospider/wsgi.py --master --processes 4 --threads 2 --stats 0.0.0.0:9191

然后打开浏览器访问: http://你的服务器ip:3031, 看能否访问网站。 测试通过后,可以结束上面的进程。


# 配置Supervisor
`Supervisor`是用来将`uWsgi`做成一个守护进程的，通过`supervisor`命令或者网页服务也能启动和停止`uUwsgi`.

安装和运行`Supervisor`请参考官方文档: [http://supervisord.org/](http://supervisord.org/) 。

下面开始配置`Supervisor`。

    # 使用软链接, 载入 `supervisor` 的配置
    cd /etc/supervisord.d/
    sudo ln -s /var/www/0fenbei/weibo-spider/supervisor.conf weibospider.0fenbei.com.conf 
    
    # supervisor 更新配置,并启动进程
    sudo supervisorctl -c /etc/supervisord.conf
    > reread
    > update
    > status

这个时候可以看到服务已经启动起来了。

`Supervisor`的配置如下:
    
    [program:uwsgi-weibospider.0fenbei.com]
    # 确保uwsgi命令的路径是对的
    command=/usr/bin/uwsgi --ini /var/www/0fenbei/weibo-spider/uwsgi.ini
    directory=/var/www/0fenbei/weibo-spider
    umask=022
    # 以ripple用户运行
    user=ripple
    startsecs=0
    stopwaitsecs=0
    autostart=true
    autorestart=true
    # 注意确保路径存在
    stdout_logfile=/var/log/bowenpay/weibo-spider.stdout.log
    stderr_logfile=/var/log/bowenpay/weibo-spider.stderr.log
    stopsignal=QUIT
    killasgroup=true
    
    [supervisord]

如果不想使用`Supervisor`也可以，只要把上面`uwsgi.ini`文件中关于最后一行关于`daemonize`的注释去掉, 直接用以下命令启动`uWsgi`。

    /usr/bin/uwsgi --ini /var/www/0fenbei/weibo-spider/uwsgi.ini


# 配置Nginx
使用软链接, 将 `nginx.conf` 写入到 `nginx` 的配置中:

    cd /etc/nginx/conf.d/
    sudo ln -s /var/www/0fenbei/weibo-spider/nginx.conf weibospider.0fenbei.com.conf


然后重启 `nginx`

    sudo systemctl restart nginx

`nginx.conf` 的配置内容如下:

    upstream weibospider_0fenbei_uwsgi_backend {
    	# django工程在49160端口启动
        server 127.0.0.1:49160;
    }
    
    server {
        listen   80;
        # 通过该域名访问
        server_name weibospider.0fenbei.com;
    
    	# 禁止访问git文件
        location ^~ /.git {
            deny all;
        }
    	
    	# django的静态文件，直接通过nginx访问，不走uWsgi。 
    	# 静态文件的目录是：/var/www/0fenbei/weibo-spider/static/
        location ^~ /static {
            root   /var/www/0fenbei/weibo-spider; 
            index  index.html;
            expires 1M;
            access_log off;
            add_header Cache-Control "public";
        }
    	# 非静态文件请求都走uWsgi，具体端口在upstream配置
        location / {
            proxy_next_upstream error timeout http_500 http_503;
            proxy_connect_timeout 4000ms;
            proxy_read_timeout    30s;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Rewrite-URL $request_uri;
            client_max_body_size 10m;
    
    		# 通过 upstream 
            uwsgi_pass  weibospider_0fenbei_uwsgi_backend;
            include uwsgi_params;
        }
    
    }
 
# 配置域名
 在你的域名提供服务商处,增加一条域名解析A记录,将域名 `wechatspider.0fenbei.com` 解析到 你的服务器ip。
 成功后。
 
 这个时候打开 [http://wechatspider.0fenbei.com](http://wechatspider.0fenbei.com) ,就能成功访问网站了。
 
 