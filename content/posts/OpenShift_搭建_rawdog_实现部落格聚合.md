---
title: "OpenShift 搭建 rawdog 实现部落格聚合"
date: 2013-03-19T00:00:00+08:00
draft: false
---
因为 [Planet openSUSE](https://planet.opensuse.org/) 的管理员一直 unavailable，导致我无法正常的推送对 Planet 的修复和处理中文新成员的加入，考虑到中文博客又太多，总去找一个 unavailable 的人，两边都互相嫌嘛，于是就架设了这个：~~community.suse.org.cn~~ Update: 挂了

这篇文章中的教学其实我只完成了一半，python wsgi 前台是 Arch 维护者 [Felix Yan](http://blog.felixc.at/) 帮写的，所以在前面感谢一下 co-worker。
准备

你要去开一个新 `Application`，类型是 `Python 2.7 Community Cartridge`，然后要新添加一个 `Cron 1.4 Cartridge`。其它的 openshift 必要知识我假设你已经了解了，不了解的话请去看我之前的文章。
rawdog 架设

`rawdog` 是 KDE 社区开发的部落格聚合程序。简单说，这就是一个 python 程序，它只能获取 feed，并输出 HTML，至于怎么让 HTML 能被用户在互联网上看到，这就是 HTTP Server 需要做的事情了。而 openshift python 默认的 HTTP Server 就是 Apache + python wsgi。如果你用 VPS 还可以使用如 Apache，Nginx，Lighttpd 等配置 FastCGI 什么的。HTTP Server 这块我不懂，所以只能拿人家写好的 Server 出来讲。但是一定要记住，只配置好 rawdog 你才成功了一半。

另外这里我用的也不是原装的，是三转之后的 rawdog，就是被 Planet openSUSE 改了一次，我自己又改了一次的。代码在 [susews-planet](https://github.com/marguerite/susews-planet)。

ssh 进你的 application，然后

    cd $OPENSHIFT_DATA_DIR
    git clone https://github.com/marguerite/susews-planet.git
    cd susews-planet

这个你只能使用 openSUSE 的 bento 风格，想改风格请在本地 Git 捡出，然后修改 planetsuse/planet_template.html 和 planetsuse/feedlist_template.html，website/css，website/js，website/images 下的内容。

修改完成后依次使用 l10n-extract-from-template 和 l10n-merge.sh 提取翻译，去 locale 文件夹用 lokalize 翻译 .po 文件，翻译好后运行 l10n-compile-po.sh 生成 .mo 文件。

然后通过各种方法把它上传到上面所说的目录。

接着运行一次：

    ./compile.sh

这会编译 python 的 .pyc 和 .pyo。

然后修改 planetsuse/feeds 文件，加入要聚合的部落格：

    feed 15m http://www.marguerite.su/feed/
      define_name Marguerite Su
      define_nick marguerite
      define_face marguerite
      define_lang cn
      define_irc  marguerite
      define_member 1

这是一个比较全的单个 feed 配置。其中 defineface 会用在 website/hackergotchi/ 下的同名 .png 头像。definemember 是判断你是不是 openSUSE 社员。本质上你可以 define 任何关键词，只要你在前面的两个 template 里有对它们规定风格。

改好了 feeds 和上传了 hackergotchi 头像后。运行：

    ./update.sh

这会预取 feeds，也会提示一些比如 feed 301 重定向了，新 feed 在哪里的消息。

    ./write.sh

这会在 website/global 和 website/[lang] 这样的文件夹下写入 HTML。

至此，rawdog 配置完成。

#### wsgi 伺服器架设

首先，在本地 git clone [openshift 给你的那个 ssh:// 的仓库地址]。

然后修改 setup.py

    from setuptools import setup

    setup(name='openSUSE Chinese Community', version='1.0',
    description='OpenShift Python-2.7 Community Cartridge based application',
    author='Marguerite Su', author_email='marguerite@opensuse.org',
    url='http://www.python.org/sigs/distutils-sig/',

    #  Uncomment one or more lines below in the install_requires section
    #  for the specific client drivers/modules your application needs.
    install_requires=['greenlet', 'gevent', 'bottle',
      #  'MySQL-python',
      #  'pymongo',
      #  'psycopg2',
      ],
    )

重点要改的是 name，author，authoremail 和下面的 installrequires。

接着修改 wsgi/application，这个是 Felix 的工作，我完全看不懂，不过你的 susews-planet/website 的位置和我的一样的话，可以直接复制粘贴：

    #!/usr/bin/env python
    import os
    from bottle import route, default_app, static_file, redirect, HTTPError

    root = os.environ['OPENSHIFT_HOMEDIR'] + 'app-root/runtime/data/susews-planet/website'

    @route('/')
    def homepage():
        redirect('/global/')


    @route('/<path:path>/')
    def static_dir(path):
        return static_file(path + "/index.html", root=root)

    @route('/<path:path>')
    def static(path):
        result = static_file(path, root=root)
        if isinstance(result, HTTPError):
            result = static_file(path + "/index.html", root=root)
            if not isinstance(result, HTTPError):
                redirect(path + "/")
         return result

    application = default_app()

然后我们还要做一件事，因为你的 planetsuse/feeds 和 website/hackergotchi 都在你用 git 修改不到的地方，这给后续更新带来了麻烦，所以要修改下。

首先 ssh 登入把它们两个删掉。

然后把你本地的 planetsuse/feeds 和 website/hackergotchi 复制到 git 根目录下。

接着编辑 .openshift/action_hooks/build，内容如下：

    ln -sf ${OPENSHIFT_REPO_DIR}/hackergotchi ${OPENSHIFT_DATA_DIR}/susews-planet/website/
    ln -sf ${OPENSHIFT_REPO_DIR}/feeds ${OPENSHIFT_DATA_DIR}/susews-planet/planetsuse/

就是把我们 git 上传的这两个文件软链接到它们应该在的地方去。

#### Cron jobs

前面已经说了，rawdog 只管制作 HTML，但是我们需要这些 HTML 始终最新，所以需要用 cron jobs 每小时跑一次它来更新 feeds。

进入到 .openshift/cron/hourly 做一个 fetch.sh，内容如下：

    {
    $LOG_DATE=`date -R`
    echo -n "========== START $LOG_DATE =========="
    ${OPENSHIFT_DATA_DIR}/susews-planet/update.sh 2>&1
    ${OPENSHIFT_DATA_DIR}/susews-planet/write.sh 2>&1
    echo -n "========== END $LOG_DATE =========="
    } > "${OPENSHIFT_DATA_DIR}/log.txt" 2>&1

然后回退到 git 根目录：

    git add .
    git commit -m "add cron and soft link"
    git push

本教学到此就结束了，如果您想去绑定域名，请看我这篇文章：

[openshift 搭建 wordpress 教程](http://marguerite.su/openshift-host-wordpress-tutorial)

如果您想弄个备份，请看我这篇文章：

[openshift 搭建和 IRC 频道互通的 Gtalk 群聊](http://marguerite.su/openshift-setup-a-gtalk-group-chat-with-a-bind-in-irc-channel)

Enjoy life！

#### 2013-10-01 Update: Nginx 相关

Nginx 相对要简单一些，只需要本地捡出，然后相应修改，传到你的 htdocs/子文件夹。写对配置就可以，这是迁移到 community.suse.org.cn 后由我和 phonixlzx 共同修改的 nginx 配置，写到 /etc/nginx/nginx.conf 还是 /etc/nginx/vhosts.d/xxx.conf 要看你的需求了。

    server {
        server_name community.suse.org.cn;
        listen   80;
        listen   [::]:80;

        root /srv/www/htdocs/community.suse.org.cn/website/;
        index index.html;

        # Only match requests for /
        location = / {
            rewrite ^ /global permanent;
        }
    }
