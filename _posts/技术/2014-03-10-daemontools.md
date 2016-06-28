---
layout: post
title: Daemontools使用
category: 技术
tags: 运维
---

linux主要使用supervise来管理常驻进程。基于supervise的两个比较重要的工具是[Daemontools]和[Supervisor]。

实际上，supervise也算Daemontools的一个工具。Daemontools是svscanboot，svscan，supervise，svc，svok，svstat等一系列工具的合集。

[Daemontools]: http://cr.yp.to/daemontools.html 
[Supervisor]: http://supervisord.org


安装
----
执行安装后，所有的命令都放到了`/command`目录，并软链到`/usr/local/bin/`下面。

并且新建了`/service`目录来放置常驻脚本。


关系
----
安装完后，可以看到两个进程启动了。

    root     19907  0.0  0.0   1936   508 ?        Ss   17:12   0:00 /bin/sh /command/svscanboot
    root     19909  0.0  0.0   1880   376 ?        S    17:12   0:00 svscan /service

svscanboot启动svscan监视`/service`目录，svscan则为`/service`的每个进程都启动一个supervise服务。

`supervise s` 执行`./s/run`，如果`s/down`文件存在，则需要使用svc手动启用。（机器重启的时候防止自动启用）

如果往`/service`下面加入服务脚本，则可以在后台看到下面的进程。

    root@test2:/opt/tiger/graphite_client/tsar-client_run# ps aux|grep supervise
    root      3945  0.0  0.0   3932    40 ?        S     2013   0:00 supervise location_search_8920
    root      3946  0.0  0.0   3932    28 ?        S     2013   0:00 supervise fenci_run
    root      3952  0.0  0.0   3932    76 ?        S     2013  44:04 supervise tsar-client_run
    root      3953  0.0  0.0   3932    52 ?        S     2013   0:00 supervise sentinel_run
    root      3954  0.0  0.0   3932    20 ?        S     2013   0:00 supervise qiuzu_solr

supervise的状态信息以2进制的形式存放在`s/supervise`下面，并且提供了下面的工具来操作：

* svstat： 读取状态信息
* svc： 启动/停止/挂起等
* svok： 检查是否运行成功
* svscan：可靠的启动`/service`目录下的服务。如果某个服务加入后，没有启动，可以调用此命令，强制启动。 

加入一个新服务
--------------
最简单的方式是建立一个文件夹

    testsvc
    ├── main.py
    └── run

写入文件内容：
    
    yijingping@yjp-pc:~/testsvc$ cat main.py 
    #!/usr/bin/python
    import time
    import logging
    
    while True:
        time.sleep(1)
        logging.info('sleep 1 second')
        logging.error('sleep 1 second')

    yijingping@yjp-pc:~/testsvc$ cat run
    #!/bin/sh
    exec ./main.py 1>/var/log/main.py.log 2>&1

然后在`/service`目录下建立软链接

    $ sudo ln -s /path/to/testsvc

这个时候可以检查一下服务是否正在运行：

    yijingping@yjp-pc:~$ sudo svstat /service/testsvc
    /service/testsvc: up (pid 4204) 962 seconds
    yijingping@yjp-pc:~$ ps aux|grep supervise
    root      4203  0.0  0.0   1716   248 ?        S    09:37   0:00 supervise testsvc
    1000      5631  0.0  0.0   3784   792 pts/3    S+   09:54   0:00 grep supervise
    yijingping@yjp-pc:~$ tree /service/testsvc
    /service/testsvc
    ├── main.py
    ├── run
    └── supervise [error opening dir]
    
    1 directory, 2 files

上面这种方式的坏处是必须以root用户运行，如果想以其他用户运行，则需要做如下改进，假设用户为tiger，id为1001：

    tigersvc
    ├── main.py
    ├── real_run
    └── run

文件内容：

    tiger@yjp-pc:~/tigersvc$ cat run
    #!/bin/sh
    who=$(id -u)
    if [ $who -eq 0 ]; then
            exec /usr/local/bin/setuidgid tiger ./real_run
    elif [ $who -eq 1001 ];then
            exec ./real_run
    else
            echo "neither root nor tiger"
    fi
    tiger@yjp-pc:~/tigersvc$ cat real_run 
    #!/bin/sh
    exec ./main.py 1>/var/log/tiger/main2.py.log 2>&1

加入服务后，查看后台进程以tiger为用户在运行：

    yijingping@yjp-pc:/service$ ps aux|grep main.py
    tiger    24682  0.0  0.1   7052  3924 ?        S    13:47   0:00 /usr/bin/python ./main.py

管理服务
-------
使用svstat来查看服务

    yijingping@yjp-pc:/service$ sudo svstat testsvc
    testsvc: down 20 seconds, normally up
    yijingping@yjp-pc:/service$ sudo svstat tigersvc
    tigersvc: up (pid 25046) 230 seconds

使用svc来管理服务

    command mnemonic    signal  action
    svc -u  up                  bring service up
    svc -d  down                put service down (stays down)
    svc -o  once                run service once (don't restart)
    svc -k  kill        SIGKILL send service KILL signal

如果要重启，必须先`svc -d s`，再`svc -u s`。

其他工具
--------
log工具：

* The readproctitle program 
* The multilog program 
* The tai64n program 
* The tai64nlocal program

环境工具：

* The setuidgid program 
* The envuidgid program 
* The envdir program 
* The softlimit program 
* The setlock program
