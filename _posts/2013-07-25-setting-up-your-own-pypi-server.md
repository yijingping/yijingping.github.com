如何搭建自己的pypi私有源服务器
===========================
通常我们使用pip安装python包，都会默认从 https://pypi.python.org/pypi 上安装，非常方便。

但是有些是公司内部的项目，不方便放到外网上去，这个时候我们就要搭建自己的内网pypi源服务器，安全并且拥有同样的舒适体验。

python官方有个pypi私有源实现的说明：http://wiki.python.org/moin/PyPiImplementations ，并 且列出了几个比较成熟的实现方案:

* [PyPI] (aka [CheeseShop]) - The reference implementation, powering the main index.
* [ClueReleaseManager]
* [EggBasket] - A simple, lightweight Python Package Index (aka Cheeseshop) clone.
* [haufe.eggserver] - Grok-based local repository with upload and no security model.
* [Plone Software Center]
* [chishop] - django based
* [pypiserver] - minimal pypi server, easy to install & use

[PyPI]: http://wiki.python.org/moin/CheeseShopDev
[CheeseShop]: http://wiki.python.org/moin/CheeseShop
[ClueReleaseManager]: http://pypi.python.org/pypi/ClueReleaseManager
[EggBasket]: http://chrisarndt.de/projects/eggbasket/
[haufe.eggserver]: http://pypi.python.org/pypi/haufe.eggserver
[Plone Software Center]: http://tarekziade.wordpress.com/2008/03/20/how-to-run-your-own-private-pypi-cheeseshop-server/
[chishop]: https://github.com/ask/chishop
[pypiserver]: http://pypi.python.org/pypi/pypiserver
	
我选择pypiserver，因为他最小而且使用简单。下面是搭建的过程。

安装和快速上手
-----------------------

    ```
    $ pip install pypiserver
    $ mkdir ~/packages
    # copy some source packages or eggs to this directory
    $ pypi-server -p 8080 ~/packages
    $ pip install -i http://localhost:8080/simple/ ...
    ```

改进
----------------
我们用supervisor来管理pypi-server。

* 安装supervisor

    ```
    $ sudo apt-get install supervisor
    $ ps aux|grep supervisor # 查看后台是否已经运行起来了
    ```

* 安装pypi server 

    ```
    $ virtualenv pypienv    # 建立一个virtaulenv

    $ source PATH/TO/pypienv/bin/activate
    $ pip install pypiserver   #安装pypi server

    $ mkdir PATH/TO/pypi-packages # 建立存放packages的文件夹
    ```

* 编写脚本/PATH/TO/run-pypi.py，作用是在virtualenv中启动pypiserver。

    ```
    #!/bin/sh                      
    # 启动virtualenv                                                                                    
    . PATH/TO/pypienv/bin/activate                 
    # 使用端口号3141，因为pypi与π谐音，π≈3.141
    exec pypi-server -p 3141 /PATH/TO/pypi-packages  
    ```

* 在/etc/supervisor/conf.d增加配置文件pypi-server.conf， 配置如下：

    ```
    [program:pypi-server]                 
    directory=/PATH/TO/   
    command=sh run-pypi.sh                
    autostart=true                        
    autorestart=true                      
    redirect_stderr=true                  
    ```

* 重启supervisor

    ```
    $ sudo /etc/init.d/supervisor stop
    $ sudo /etc/init.d/supervisor start
    ```

    这时候在浏览器中访问 [http://localhost:3134/](http://localhost:3134/) ，就可以看到pypiserver的欢迎页面了。

* 上传package

    方法1：将/PATH/TO/pypi-packages当成git仓库管理起来，通过git push来管理packages

    方法2：用ftp的方式上传

    建议使用方法1

* 下载package
    
    ```
    $ pip install -i http://localhost:3134/simple/ some-package
    ```






 
