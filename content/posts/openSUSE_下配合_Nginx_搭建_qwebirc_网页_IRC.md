---
title: "openSUSE 下配合 Nginx 搭建 qwebirc 网页 IRC"
date: 2013-10-12T00:00:00+08:00
draft: false
---
做这个的初衷很简单：IRC 对小白太远了。

它不像论坛，打开网页就能刷; 不像 QQ/Gtalk，软件是人都能找到，装上就能上。它需要专用客户端（比如 konversation），而且装完你还不能上，还有一堆命令等着你，何况还有 20 年早期互联网流传下来的各种名词、礼仪。总之它是一整个与世隔绝的黑客生态圈。

但是吧，不学还不行，你看哪个开发者不会使用 IRC 的？于是我想先降低一下它对小白们的陌生感，做一个给蠢人用的玩意。（成功把自己从其它蠢人中分离开以示区别哦耶！）

于是最蠢的来了：qwebirc，是一个自由开源的网页 IRC 客户端，最早是给 QuakeNet 游戏网开发的。这是目前我能找到的唯一一个开源的。

看了下，发现它居然不支持多语言！好吧，硬 hack 出了一个繁体中文版（别问为什么不用简体，因为硬 hack 就是只能使用一种语言，没有动态切换，那简体用户繁体一样能看懂，自豪吧？）。

#### 安装依赖（openSUSE 下）

    sudo zypper in nginx python-Twisted python-simplejson java-1_7_0-openjdk

#### 下载

然后随便找个文件夹，因为 qwebirc 和 HTTP 服务器的关系不是常规那样的，常规是在 /srv/www/htdocs（又叫 webroot）下面放一些 html/php 文件，然后 HTTP 服务器让那个文件夹下面的东西能被访问，但是 qwebirc 其实是自己跑起来了一个专用的 http 服务器，为什么呢？据说是通用 HTTP 服务器设计不是用来服务大量的、长期活动的连接的，它们可以处理的是大量的但是都是一次性的连接，这种基于线程或者多进程的服务器（Apache 被点名了）没办法处理太多的这种连接，到时候会把你的网站一起拽下线呦！

但 qwebirc 的专用服务器是开在 9090 端口的，你不想访问 yourircserver.com:9090 这种丑丑的链接吧？所以你需要一个反向代理，还不能用被怒黑的 Apache。。。也就 nginx 了吧？好在我们别的服务器也是用的 nginx，因为 openSUSE 社区穷 VPS 内存小。。。

言归正传。找到文件夹了吧？然后在命令行下载最新的稳定版 qwebirc：

    wget http://qwebirc.org/download-stable-zip

它没后缀但它是个 zip，解压：

    unzip download-stable-zip

进入文件夹：

    cd qwebirc-qwebirc-516de557ddc7

#### 配置

把 config.py.example 复制为 config.py：

    cp -r config.py.example config.py

打开

    vi config.py

要改的我都写好：

连接到的服务器:

    IRCSERVER, IRCPORT = "chat.freenode.net", 6667

因为我们是网页客户端不是网页服务器端，我们要连一个服务器的…

REALNAME:

    REALNAME = "http://irc.suse.org.cn/"

你要提供服务的那个网址

BASE_URL:

    BASE_URL = "http://irc.suse.org.cn/"

同上

    NETWORK_NAME = “openSUSE 中文”

你的客户端显示给客户看的「你叫什么名」字段。这里需要注意，如果你要用中文，要在这个页面的最上面加上这两行：

    #!/usr/bin/python
    # -*- coding: utf-8 -*-

因为它用 python2 写的，默认是英文编码…

Nginx 相关配置:

    FORWARDED_FOR_HEADER="x-forwarded-for"
    FORWARDED_FOR_IPS=["127.0.0.1"]
    ARGS = "-i 127.0.0.1 -p 9090"

简单说就是在你 VPS 上的 127.0.0.1:9090 开一个网页 IRC，然后 Nginx 后面干的事就是把对你的某个网址比如 irc.suse.org.cn 的访问全中转给 127.0.0.1:9090 上。我不太懂，但听起来像是挺安全的，因为我觉得 127.0.0.1 只有登录到那台 VPS 才能用的嘛。

当然你也可以试试把前两个的 # 号继续留着，然后直接：

    ARGS = "-i 你的公网 IP -p 80"

没试验哦，估计这样就能直接开在你的 80 端口了，也不用 nginx 了。但这样做最大的问题就是你的那个公网 IP 也只能做网页 IRC 了。因为不管你怎么做域名解析，通过浏览器过来的都默认跑 80 端口就来到这里了，如果别的还用（我不懂网络那些哦），可能就会出现我前面 nginx 重定向配置错误时的情况，时而打开网页 IRC，时而打开你别的网站…

而用 nginx 做反向代理的话，那个 IP 还可以干别的。（我说的是 IP 不是域名哦，你不可能 irc.suse.org.cn 又当 wordpress 博客又当网页 IRC 的）

剩下的配置就留默认就好

#### 编译和启动

这里其实我有点不能理解，它编译应该是把 *.py 编译为 *.pyc/pyo 才对，但事实上，你在 javascript 里面的硬翻译，也是由编译来管的…我刚开始先架设了一个英文的，然后本地翻译完，上传，我想 js 应该刷新网页就好了吧，结果不行…debug 了好久，后来在官网 Installation 指南看到

    Run compile.py to "compile the HTML/js/css"

整个人都呆掉了…

开始编译：

    ./compile.py

把它跑起来：

    ./run.py

它写的就比较有节操，自己会 fork 自己到后台，不像我之前写的那些 openshift 跑 python 的教程里面的脚本都要使用 nohup 来跑…

qwebirc 就配置好了。

#### Nginx 反向代理

    cd /etc/nginx

你是直接编辑 /etc/nginx/nginx.conf 还是建一个 /etc/nginx/vhosts.d/irc.suse.org.cn.conf 我管不着，反正里面是像这样的：

    server {
    listen 80;
    listen [::]:80;

    server_name webchat.domain.tld;

    access_log /var/log/qwebirc.access.log;
    error_log /var/log/qwebirc.error.log;

    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_buffering off;

    location / {
        proxy_pass http://127.0.0.1:9090;
        }
    }

它和 Nginx Wiki 上说的那个配置的唯一不同之处在于：

    listen 80;
    listen [::]:80;

Wiki 上用的是：

    listen 127.0.0.1:80;
    listen [::1]:80;

导致我在不明白其原理的时候狂刷 irc.suse.org.cn。。。。

后来才反应过来：Wiki 的教学的场景是说，你在本机（127.0.0.1）跑起来了一个 qwebirc，然后嫌用 127.0.0.1:9090 访问它比较的麻烦，于是做下反向代理让 127.0.0.1 可以直接访问到它。

但我们的场景是说，我在本机（127.0.0.1）跑起来了一个 qwebirc，然后想用 http://irc.suse.org.cn 直接访问它！127.0.0.1:9090 和我没一毛钱关系，因为我没在服务器上登录装图形界面开网页浏览器…

我们要让 Nginx 去监听对 irc.suse.org.cn 的 80 端口访问，然后把它扔给 VPS 上的 127.0.0.1:9090。。。所以直接监听 server_name 的 80 端口就好了嘛…

这个错误很难反应过来，尤其对我这种水货 SA！Nginx 还会正常跑，但你在 VPS 外面是访问不到它的：

qwebirc 提供的服务在 127.0.0.1 不是你的公网 IP，所以你用 xxx.xxx.xxx.xxx:9090 根本打不开…甚至 Nginx 也不知道你在干啥，因为别的配置是没有用 9090 端口的，连 Fallback 这种安慰奖都没有…

nginx 监听着你的内！网！IP！你访问 irc.suse.org.cn，是有这个配置，但是它找了一圈发现没有跟 irc.suse.org.cn: 有关的事啊，这个配置在管 127.0.0.1 呢…

还有注意：proxy_pass http://127.0.0.1:9090; 这个不要手贱把 127.0.0.1 改成 irc.suse.org.cn 了。。。因为那个域名的 A 记录是你的公网 IP，而 qwebirc 并没有对公网提供服务…血泪的教训！

接着重启下 Nginx 服务器：

    sudo systemctl restart nginx.service

就 OK 了！

PS：终于有个顺手的 IRC 去调戏 #kde-cn 的 BadGirl 了…
