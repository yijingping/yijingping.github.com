---
layout: post
title:  "为什么要使用 Markdown 和 Jekyll"
date:   2015-01-05 19:18:07
author: "易先生个蛋"
category: 思考
tags: Jekyll
---

# 为什么要使用 Markdown 和 Jekyll
这里我发布技术文章和感悟的地方，文章的核心是内容，不是形式，我不愿意花大量的时间在布局和排版上。

在筛选了大部分文档编辑方案后，我发现 Markdown **「易读易写」** 的哲学和我的期望最相符。

Jekyll是一个静态站点生成器，可以根据Markdown文件自动生成静态的html文件。且github pages 支持托管jekyll。

因此我只要在本地编写符合Jekyll规范的Markdown文件，上传到github上，github pages就会自动生成并托管整个网站。

这样做带来的好处是：

1. 专注。只需要关注Markdown内容的编写。无需考虑标签和样式，也不会干扰git的log。
2. 历史版本。通过git历史可以看到自己思维的变迁。
4. 免费，不限流量。
5. 简单。你只要用自己喜欢的编辑器写文章就可以了，其他事情一概不用操心，都由github处理。

下面一步一步来建立我的博客

#1. 使用github pages

建立Github仓库并初始化

1. 在 [https://github.com/username/] 下面建立一个名称为 `username.github.io` 的仓库。注意username必须和 **用户名** 或者 **组织名** 一样。

2. 在这个仓库的 `master` 分支中建立一个 `index.html` 文件，并且写入 “Hello World!”。提交并push到github。

3. 前往 [http://username.github.io] 查看页面。

更详细教程前往 [https://pages.github.com/] 。

到这里我已经学会了把静态文件托管在github上，下一步接着学习如何使用Jekyll托管动态文件。

#2. 使用Jekyll
前面说过，Jekyll是一个静态站点生成器，可以根据Markdown文件自动生成静态的html文件。

1. 安装Jekyll

    Jekyll是用ruby开发的，因此使用之前需要确保系统环境有以下东西：

        1. Ruby：版本必须是 2.0.0以上
        2. RubyGems：Ruby的包管理器
        3. Jekyll：使用命令 `gem install jekyll` 安装

    Jekyll对Windows并不提供官方支持，如果想在Windows上运行，可以参考：[Windows支持文档]

2. 使用Jekyll

    1. 新建一个Jekyll工程。

        执行命令 `jekyll new yourProjectName` ，在 `yourProjectName` 目录下会生成如下文件：

            .
            ├── _config.yml
            ├── _includes
            │   ├── footer.html
            │   ├── head.html
            │   └── header.html
            ├── _layouts
            │   ├── default.html
            │   ├── page.html
            │   └── post.html
            ├── _posts
            │   └── 2015-09-18-welcome-to-jekyll.markdown
            ├── _sass
            │   ├── _base.scss
            │   ├── _layout.scss
            │   └── _syntax-highlighting.scss
            ├── about.md
            ├── css
            │   └── main.scss
            ├── feed.xml
            └── index.html

        具体的含义可以参考 [http://jekyllrb.com/docs/structure/](http://jekyllrb.com/docs/structure/)

        这是最简单的工程结构，为了遵循DRY，我们也可以直接使用一些成熟的 [使用了Jekyll的网站] 模板。

    2. 运行和调试Jekyll

        在 `yourProjectName` 目录执行 `jekyll serve`，
        jekyll会启动一个web server，用于接收http请求。
        只要打开浏览器，前往 [http://localhost:4000/](http://localhost:4000/) 就可以看到生成的网页。

        同时，`jekyll` 还会自动监控当前目录下的文件改动，即时生成html。改变文件后，刷新一下浏览器就可以看到效果。

    3. 添加文章

        所有的文章都放在 `_posts` 目录下。进入该目录，按照格式 `YYYY-MM-DD-name-of-post.markdown` 添加一篇新文章。

        文章格式可以参考 `_posts` 目录下的示例：`XXXX-XX-XX-welcome-to-jekyll.markdown`。

        文章保存后，刷新浏览器，就可以看到新文章了。

    4. 托管到github

        编辑完本地文件后，直接将commits `push` 到 `github` 的 `username.github.io` 主仓库，
        前往 [http://username.github.io](http://username.github.io) 就可以看到你的网站了。
        赶紧把网址分享给你的朋友吧。

    5. 其它

        基础内容到此就为止了，如果想使用更多Jekyll高级功能，请访问 [Jekyll官网]。

        添加留言功能，参考 [友言：一个专业的网站社交评论系统]。

        添加分享功能，参考 [百度分享]。


#3. 使用Markdown语法编辑文章

`_posts`目录下的文章除了头部信息，剩下的都是markdown语法的。

头部信息格式如下：

    ---
    layout: post                        ## 文章使用的模板
    title:  "使用 Markdown 和 Jekyll"    ## 文章的标题
    date:   2015-01-05 19:18:07         ## 发布时间
    author: "易先生个蛋"                 ## 作者
    categories: manage                  ## 文章所属的目录
    ---

body部分的格式请前往 [Markdown 语法说明 (简体中文版)] 学习。


[https://github.com/username/]: https://github.com/username/
[https://pages.github.com/]:    https://pages.github.com/
[http://username.github.io]:    http://username.github.io
[Windows支持文档]:               http://jekyllrb.com/docs/windows/#installation
[使用了Jekyll的网站]:             http://jekyllrb.com/docs/sites/
[Jekyll官网]:                    http://jekyllrb.com/
[友言：一个专业的网站社交评论系统]:  http://www.uyan.cc/
[百度分享]:                      http://share.baidu.com/
[Markdown 语法说明 (简体中文版)]:  http://wowubuntu.com/markdown/