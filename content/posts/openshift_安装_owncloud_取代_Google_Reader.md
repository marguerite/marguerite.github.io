---
title: "openshift 安装 owncloud 取代 Google Reader"
date: 2013-03-15T00:00:00+08:00
draft: false
---
Google Reader’s sunset is the dawn of ownCloud news.

翻译过来就是 GR 的夕阳正是 ownCloud 新闻订阅的佛晓。

相信 Google Reader 在 7 月 1 号关闭对大家都是一个打击，虽然大家的未读都是 1000+。但是一种生活方式突然变了，总归有点怅然。至少我听闻这个消息的时候第一反应就是：妈呀！我的那么多红心怎么办！

是小企鹅输入法的作者翁学天让我意识到了：哦，好在还有替代。虽然我是它的简体中文翻译者，但是我以前真的不知道嘿嘿。

首先这个替代目前装起来还是有一点困难的。所以需要一个这样的中文教学来教大家怎么才能装上有 News 的 ownCloud。

我们开始吧。（以 openSUSE 为例）

#### 下载

首先我们需要 Git。因为现在只有 Git 版的 ownCloud Apps 才有这个功能。

    sudo zypper in git

安装好了 Git，我们需要下载这些源码

    git clone https://github.com/owncloud/core

这是核心组件。

    git clone https://github.com/owncloud/apps

这是带 News 的 apps。

    git clone https://github.com/owncloud/themes

这是新的 responsive 主题。

    git clone https://github.com/owncloud/3rdparty

这是一些第三方的比较好用的 apps。当然如果不用也可以不装（我装了也没发现有太大用）。

    cd apps
    git clone https://github.com/owncloud/mail

这是 ownCloud 新写的邮件收发界面，愿意尝试的可以下。

然后，把 apps, themes, 3rdparty 文件夹下的所有内容都放到 core 里相关文件夹下面。

于是你就有了完整版的 ownCloud 6.0 alpha。

#### 注册 OpenShift 应用

下面去要去 openshift 申请一个账户。

然后在「我的应用程序」去新建一个应用程序。

选 PHP 5.3, 然后为你的 Web app 设置一个好记的网址。

注意下面的「Scaling」要设成「是」。OpenShift 默认可以建立三个应用程序，每个应用程序 1 GB 空间，但是 1 GB 空间对于一个个人网盘有点少。所以我们选了「是」就是说如果我装满了，就再挤占我另外的一个 Gear 的空间，直到三个都用光。

点「创建应用」，然后在右手边，会有一个添加「Cartridge」的按钮，进去添加一个 MySQL 的 Cartridge。

之后应该返回给你一个框，需要你的 SSH 公钥。Linux 下只需要

    ssh-keygen 

就可以在 $HOME/.ssh/isa.pub 生成一串公钥，把那个贴过去就好了。 安装

你需要用刚才装的 Git 去克隆 OpenShift 给你的那个 URL。

    git clone 那串 URL

下载来之后，执行以下操作：

    cd owncloud
    git rm -r php/index.php

然后添加一个远程 Git 仓库，抓取 openshift 的人调教好的 ownCloud。

    git remote add upstream -m master git://github.com/ichristo/owncloud-openshift-quickstart.git
    git pull -s recursive -X theirs upstream master

好了吧，但是这个是稳定版的，没有 News。我们就要把我们刚才准备好的 core 复制到 php/ 文件夹下面去覆盖它们：

    cp -r core/* owncloud/php/*

为什么要这么做，因为红帽改的这个有一些 openshift 专有的文件，可以直接让 owncloud 跑起来而不用我们去麻烦的设置。

下面上传

    git add .
    git commit -m "initial commit"
    git push

输入你刚才 ssh-keygen 时留的密码。然后就去你的网址使用吧。

#### 背景

openshift 是红帽旗下的 PaaS（平台即服务），意思就是给你一个你账户几乎可以完全控制的在线空间，你可以装软件，跑服务什么的。

ownCloud 是 KDE 社区催生的一个 idea，意思就是为目前的 Dropbox 之类的商业网盘提供一个开源替代。所以和 KDE 整合的也不错，也有 Android/iPhone 客户端。但唯一让普通人忘而却步的就是，它需要一个个人空间才能装。

#### 其它话题

###### ownCloud 相关

它有一个 MySQL bug，就是你如果服务器上已经有了 ownCloud，那再上传新的 ownCloud 的时候就会各种用户名已存在。我没有 debug 出来究竟在哪儿，不过那些东西没有写在它的 ownCloud 数据库里，而是在那个名为 mysql 数据库的 users 和 db 表里，在里面写了你已有的用户名。再次安装不会先删除这两个表里的用户名。

但是实际测试我发现，即使添加一个 phpMyAdmin cartridge 删掉了也装不上。原因是服务器上已经有了一个 config。所以你还要 ssh 到服务器上去

    ssh 刚才那个 git 地址里没有 git:// 一直到 rhcloud.com 那串

然后进入 runtime/repo/php/config 去把那个 config.php 删除。

但是实际上我这么做了也没有成功，估计是还有别的地方没找到。但实际上这样升级并不好，因为这样做实际上是新装一个 ownCloud。你现有的 ownCloud 里的东西你要使用备份功能导出，如果你 3GB 都用了呢？来回 6GB，你有那个网速么…

目前比较好的升级方法有一种。就是想办法获取你服务器上的应用程序的拷贝，拿回来填充你的 Git，这样你的本地应用程序和服务器上的总是同步的。填充好了之后，再用新的 ownCloud 去填充一次旧版，然后再提交。具体获取拷贝的方法在我 openShift 搭建 wordpress 教程的最后有讲，而填充新版的教学在本文前面有讲。

###### Google Reader 相关

根据翁学天的扫盲。feed 只能返回最近的 10 篇文章。GR 之所以能够一直拉一直拉是因为 Google 缓存了在你之前订阅这个 feed 的人返回的那 10 篇。所以想通过 feed 取回我所有的红心是不可能了。

去 GR 发现它出了一个 takeout 功能，就是以前的导出，但是因为要关了嘛，所以能导出你以前的红心啊什么的个人历史为 JSON 格式。看了看，发现里面是很全的，但都是网址。而我的订阅里有些网址已经关站了，即使我求人写个 JSON 渲染程序也拉不到那个 URL 了（估计很快有会有程序员写这种程序了，比如 GR JSON 2 PDF 等）。

反正最后不知道你们怎么弄的，我是把 feed 手动加到 News 里（解决迁移问题），然后在 「GR 设置」- 「文件夹管理」里把红心设为 Public，接着用 Google Chrome 的网页截图扩展抓取整个网页抓了半个晚上把我的红心分页全部保存为图片，然后使用 ImageMagick 的 convert：

    convert images/* my-google-reader-fav.pdf

弄成一个 pdf 然后传到我刚弄好的 ownCloud 里去的。

好么，May the force be with you。

PS：按照 shellex 的说法：「任何时候都应该假设单一在线服务是不可靠的。当然，这与技术无关，只要将服务交与了某个服务提供商，就意味着正在失去对他们的控制力。」

「像 RMS 那样苦行僧般地活着我做不到也不会去做，像 Linux 那样的桌面我也不会有太多机会去用，但是并不代表他们就不重要。相反，他们很重要，他们的存在本身就是底线。只有支持他们，当我想从某个团体中收回控制时，才有选择的权力。」

我们是不是该考虑给自己的其它唯一的东西找一个替代呢。比如多为 openSUSE 做贡献之类的…或者，「说不定哪天 Google Talk 也关了，小姐请问我可以得到你的手机号码吗？」
