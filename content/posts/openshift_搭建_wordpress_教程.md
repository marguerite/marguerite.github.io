---
title: "openshift 搭建 wordpress 教程"
date: 2013-02-18T00:00:00+08:00
draft: false
---
最近把 suse.ws 搬家到了 Red Hat 的 openshift.com。原因是我个人博客的空间 IP 也被光荣认证了，而 openSUSE 简体中文官方主站不能在简体中文地区不能访问呀。又没有赞助就选了搞基的红帽子搞的 PaaS。

下面把搭建时候的要点简单说一下。

#### 准备

你要有一个 openshift 账户。每个账户可以开三个 Gear，共享 3 GB 硬盘和 1.5 GB 内存（VPS 泪流满面）。

你要会 Git 基本命令。比如 git clone，git add，git rm，git commit。不会的快去学习 codeschool 的免费课程 [Try Git](http://www.codeschool.com/courses/try-git)，半个小时包教包会。

你要安装 [rvm](https://rvm.io)  和 ruby，然后安装 openshift 的命令行客户端 [rhc](https://openshift.redhat.com/community/get-started)，具体的 Ruby 新新人指南可以看我的[上一篇文章](http://www.marguerite.su/2012/11/11/ruby-on-rails-hello-world/)。

#### 安装 WordPress

[生成 ssh 密钥对](http://heylinux.com/archives/956.html)并把公共密钥贴到 openshift 去。

申请一个 Gear，选 wordpress。然后看需要可以添加一个 phpMyAdmin 的 Cartridge。把它给你的数据库 root 用户名和密码以及数据库名记在小本本上别弄丢了。

`git clone` 返回给你的那个 repo，php/wp-config-example.php 改成 php/wp-config.php，把里面的数据库名、root 用户名和密码安装上面的填了。WPLANG 愿意改改一下。

提交一次，留意命令行中的信息，会说数据库的 IP。

然后再到 php/wp-config.php 里把 DB_HOST 改对。用 localhost 是不行的。

接下来就是 wordpress 的 5 分钟安装了。

#### 绑定域名

先配置您的域名的 A 记录。建议最好在域名注册商那里把 NS 记录改成一些免费的 DNS 服务商的 NS 记录，然后在 DNS 服务商的面板里配置（注册商可配置的东西特别少）。比如我的是 cloudns.net。ping 一下 openshift 给你分配的那个 appname-username.rhcloud.com 的二级域名，得到一个 IP 地址。把带 www 和不带 www 的域名的 A 记录都设为那个 IP。

然后安装 rvm 和 ruby 后 `gem install rhc`。您可能需要

    export PATH=$PATH:/home/marguerite/.rvm/gems/ruby-1.9.3-p374/bin

才能使用 rhc（Linux 下）。

`rhc alias add appname 你的域名`可以绑定域名，把带 www 和不带 www 的都绑定上。

***注意***：

1. rhc 可能需要你在本地 ssh 里有你刚生成的那个 ssh 才能正常使用
2. openshift 给你分配的二级域名免费送了一个 https，你千万不要傻到在 WordPress 的 General 设置里***把自己的主域名也输入成 https://***，那样你访问站点的时候 css/js 什么的统统加载不出来的。因为 https 只适用于二级域名，你那个域名不行。

#### 升级和备份 （皮肤、插件、上传文件等）

***绝对要看！不然您会发现 WordPress 有各种各样的问题，比如再次本地提交后之前上传的图片显示不出来了啊之类的。***

openshift 有一个比较神秘的地方在于，你使用 git 管理。但平时你是在 WordPress 后台进行操作的，所以就产生了下面的问题：

***你发现你在后台修改的内容是没法 git pull 取回本地的！***比如你升级了 wordpress 版本，然后在本地 git 仓库里改了下主题，一提交发现你的 wordpress 没有升级！

Red Hat 给出的解释是说：那些叫运行时修改，是跳过 git 的。有以下几个办法可以解决，从难到易：

1. ssh 登录进去，把那些东西都复制到一个文件夹里，tar 压缩那个文件夹，在网页里面下载回来添加到 git。这个不说太具体，和 VPS 的管理一样，您既然都会 ssh 远程登录了，是吧。

2. 使用 `rhc app snapshot save -a {appName}` 把当前 Gear 的快照下载回来，用 app_root/repo 里面的东西覆盖到 git。然后添加、提交。

3. 使用配置文件：

你会在你的 git 仓库里发现这样一个隐藏文件：`.openshift/action_hooks/build`

里面有这样的内容：

    if [ ! -d $OPENSHIFT_DATA_DIR/uploads ]; then
            mkdir $OPENSHIFT_DATA_DIR/uploads
    fi

    ln -sf $OPENSHIFT_DATA_DIR/uploads $OPENSHIFT_REPO_DIR/php/wp-content/

这段内容的作用就是把 uploads 文件夹作为 DATA_DIR。DATA_DIR 的内容是不会被 git 管理也不会被 git 修改的。所以我们照样子抄然后把 uploads 替换为我们的内容比如 themes 和 plugins 即可。

注意：此方法有一个缺点就是还是无法应对升级，另外就是如果你把那些文件夹添加为 DATA_DIR 会产生一个问题就是你发现除了万能的 ssh 运程登录，你几乎没有办法再修改它，相当于一个丢失了 FTP 功能的 PHP 空间，只能在线运行，没法备份和管理。***不要有侥幸心态觉得我这个博客就一直放在这里不动，可以负责任的说您的博客多运行一天，您再出问题进行修复或迁移的难度就增加一分。***

所以个人比较推荐第二种方法。

#### 参考文献

* [openshift 用户手册](https://access.redhat.com/knowledge/docs/en-US/OpenShift/2.0/html/User_Guide/index.html)
* [https://openshift.redhat.com/community/forums/openshift/one-newbie-question](https://openshift.redhat.com/community/forums/openshift/one-newbie-question)
* [OpenShift免费空间绑定顶级域名（图文教程）](http://www.i7086.com/openshiftbangdingdingjiyuming)

#### 小结

以上。

至少我现在的 suse.ws 还没有任何问题地倔强地狂奔在 openshift 上。
