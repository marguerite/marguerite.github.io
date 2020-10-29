---
title: "openshift 搭建和 IRC 频道互通的 Gtalk 群聊"
date: 2013-03-17T00:00:00+08:00
draft: false
---
Long long ago，大概八九个月前，看到翁学天的 ikde 群居然能和 IRC 互通，感觉很惊艳，于是也想弄一个替代现在的 opensuse_zh@im.partych.at。

Party Chat 是一个很好的免折腾 Gtalk 群聊主机服务，架设在 Amazon 上面，但是有以下缺点：

* 不能和 IRC 互通。作为一个开源社区，要是没有利用上 IRC，总感觉少了点什么。
* 在线时间不能保证。也就是说你没有 schduled_maintenance 的权利，只能它在线你就聊，它离线你就歇息。
* 大于 300 人的群拒绝服务。也就是说你的群是顶着地雷的，到了 300 人整个群就会消失，因为 Party Chat 不能承担那些多出来的流量钱。你交钱也不给你提供额外服务。

所以几十个人的小社群还是可以应付的，但是人满了之后迁移成本特别大。所以就想着趁人少迁移到 ikde 那种高级群里去。

打听了一下，这种群聊是使用依云([@lilydjwg](http://lilydjwg.is-programmer.com/) 写的两个软件在 VPS 上搭建的，分别是 [xmpptalk](https://github.com/lilydjwg/xmpptalk) 和 [ircbindxmpp](https://github.com/lilydjwg/ircbindxmpp)。

但问题出来了，我没有 VPS。当时 [openshift](https://openshift.redhat.com/) 这种 PaaS 已经出现了，只是还比较新鲜，我不会用。于是去年 9 月的一次尝试就可耻地失败了。

当时（甚至至今）网上关于这两个软件的文章只有两篇：

* 依云自己的[介绍性文章](http://lilydjwg.is-programmer.com/2012/5/14/xmpptalk-chatroom-bot-and-xmpp-group-recommandation.33537.html)，但是太粗略了。
* StarBrilliant 的稍微详细一些的[文章](http://m13253.blogspot.jp/2012/08/xmpptalk-get-started.html)，但是还是有点犯了程序员的通病：普通用户不知所云。如果你是一个已有 VPS 的博主，看了也得折腾一会儿，要是麻瓜的话，恐怕也就只能看个热闹了。

于是逼迫我们群里的 douglarek 写了一篇在 openshift 上折腾它们的文章给我：

[Openshift 折腾 xmpptalk](http://blog.icocoa.me/xmpptalk-on-openshift/)

但实际上这篇文章很水。反正看过就知道啦，到处都是「凑合」。实际上你看它也根本架设不起来（好多致命细节都没说）。

当时我就是在这种情况下开始折腾的。

首先我开了 `Do it yourself` 应用，然后进去编译，结果很悲剧，红帽给的内存太小，mongodb 编译不成功; 后来我使用 Build Service 模拟了一个 openshift 环境（一个 x86_64 的 RHEL），然后编译编译，我犯懒了…

今年三月份挖出了作者，又折腾了一个多礼拜，总算搞定了。下面是正文：

#### 注册一个 Do it yourself 应用

注册、配置 ssh 和怎么使用 ssh 登入服务器的教学见 douglarek 和我之前几篇讲 openshift 的教学，都是最基本的英文，基本能注册 GMail 的都会弄，不能注册 GMail 的基本你也不能在墙外看见我的文章。

#### 编译

基本的 `Do it yourself` 应用是带有以下部分的：Git，bz2/gz 的解压软件。所以你这些是不需要的。

ssh 进去之后，`cd $OPENSHIFT_DATA_DIR`，这个文件夹的完整位置是在 `app-root/runtime/data`，当然你也可以使用 `echo $OPENSHIFT_DATA_DIR` 看到它的完整路径。

在这个文件夹里使用 wget URL 去下载源代码，git clone 捡出 git 仓库。这些我都假设你已经知道了。

所有的编译过程都是源自依云的 scripts/quick_install.sh，但是我更新了一些东西，所以最好还是看我的。

#### 编译 Python 2.7

下载 python 2.7.3 和 3.3 的源代码（你看到这篇的文章的时候可能已经更新了，所以请去 python.org 下载最新版）。然后解压。

编译 2.7.3 是因为后面我们有个程序要用 2to3 把针对 Python 2 gen 写的代码转义到 Python 3 gen。

    mkdir python2.7 (我们要把最终的 python 2.7 装进去，因为你没有权限往 /usr 装)
    cd Python-2.7.3
    // 依云原版的多了个 `--with-computed-gotos`，但这个选项已经没有了，变默认了。另外一个 `--with-wide-unicode` 作废了，现在默认就是 long。
    ./configure --enable-shared --with-threads --enable-ipv6 --with-system-expat --with-system-ffi --prefix=${OPENSHIFT_DATA_DIR}/python2.7/
    make -j4
    make install

这样 python2.7 里就有 python 了。

#### 编译 Python 3.3

这才是主依赖，因为依云的代码是基于 3.2 版 Python 写的。

    mkdir python3.3
    cd Python-3.3
    ./configure --enable-shared --with-threads --with-computed-gotos --enable-ipv6 --with-system-expat --with-system-ffi --prefix=${OPENSHIFT_DATA_DIR}/python3.3/
    make -j4
    make install

需要注意的就是：`--prefix=${OPENSHIFT_DATA_DIR}/pythonX.X/` 一定要有，而且必须写在最后，否则不起作用。至于原因我也不知道，openshift 就是这样的。如果你看了 douglarek 的文章的话，你可能会知道一个 `${OPENSHIFT_RUNTIME_DIR}`，这个也作废了。

#### 编译 python3-distribute

去 PYPI 下载 [distribute](https://pypi.python.org/pypi/distribute)。

这时编译需要用我们前面刚制作好的 python 3.3 来编译它。所以我们需要修改一下系统的 `PATH` 和 `LDLIBRARYPATH` 让系统能够找到 python3

    export PATH=${OPENSHIFT_DATA_DIR}/python3.3/bin:$PATH
    export LD_LIBRARY_PATH=${OPENSHIFT_DATA_DIR}/python3.3/lib:$LD_LIBRARY_PATH

这个 export 命令只对当前 Shell 有效，也就是说你 ssh logout 再登入，是要重新做的。

编译：

    cd distribute-0.6.35
    // 我们把所有的子模块都装到 python3.3 目录，下同
    python3 setup.py install --prefix=${OPENSHIFT_DATA_DIR}/python3.3/

#### 编译 python3-dns

去 dnspython.org 下载 Python 3.x stable 对应的软件包。我这时是 1.10.0，解压用 unzip 命令。编译方法同 distribute。

#### 编译 pyxmpp2

要用依云修改的：

    git clone https://github.com/lilydjwg/pyxmpp2.git

这时需要注意的是，最新版有问题，我们要使用 b3a01f03a10e319a833d2f57639bb696790c930e 这个 commit 的版本：

    git checkout b3a01f03a10e319a833d2f57639bb696790c930e pyxmpp2

以后的编译方法同 distribute。

#### 编译 pymongo

下载在 [PYPI](https://pypi.python.org/pypi/pymongo)。

编译方法同 distribute。

#### 安装 mongodb 的预制包

前面已经说了，在 openshift 上编译的话内存会耗尽，所以我们用编译好的。下载在 mongodb.org，要下载 Linux 64-bit 的版本。

然后解压，把 bin 目录下的所有东西都复制到 python3.3/bin 中去就可以了：

    cp -r mongodb-*/bin/* python3.3/bin/

#### 用 Python 2.7.3 安装 mongokit

下载 `github.com/namlook/mongokit.git`

    git clone https://github.com/namlook/mongokit.git

因为要用 python 2.7.3 编译，所以我们要像前面一样把 python2.7/bin 和 python2.7/lib 导入到环境变量中去，具体方法见 distribute。

编译：

    // 就这个不是 python3
    python setup.py install --prefix=${OPENSHIFT_DATA_DIR}/python2.7/

接着我们要做 2to3，方法是：

    cd python2.7/lib/python2.7/site-packages/mongokit-*-py2.7.egg/mongokit
    2to3 -w .

然后把 mongokit-*-py2.7.egg 文件夹复制到 python3.3 的相应位置去：

    cp -r python2.7/lib/python2.7/site-packages/mongokit-*-py2.7.egg python3.3/lib/python3.3/site-packages/mongokit-*-py3.3.egg
    // * 的位置请自己补充

至此，xmpptalk 的依赖我们就安装完了。下面安装 ircbindxmpp 的。

#### 编译 tornado

捡出地址在 github.com/facebook/tornado。编译方法同 distribute。

下面我们的操作都将在 python3.3/lib/python3.3/site-packages 下进行。 取得一些 winterpy 的小脚本

    git clone https://github.com/lilydjwg/winterpy.git

然后把里面的 pylib/myutils.py 和 pylib/xmppbot.py 复制到 site-packages 目录下，winterpy 就可以删掉了。

至此，ircbindxmpp 的依赖安装完毕。下面获取它们两个。

在 site-packages 中捡出它们：

    git clone https://github.com/lilydjwg/xmpptalk
    git clone https://github.com/lilydjwg/ircbindxmpp

#### 一些清理

如果你不准备未来升级你的程序的依赖，那么你可以把 python3.3 下除了 bin 和 lib 的文件夹全部干掉节省空间。也可以不干掉。

除了 python3.3 这个目录外的其它编译中间文件目录都可以 rm -rf 掉。

#### 准备配置

你需要两个 xmpp 账户，一个用来做 Gtalk 群聊，比如 talk@suse.org.cn，另一个用来转发 IRC 上的消息，比如 irc@suse.org.cn。xmpp 账户怎么来你可以看 StarBrilliant 的文章学习配置 prosody，也可以像我一样直接在我的 Google Apps 里新建两个账户（Google 的服务据说发送太多链接和说话太勤对方会收不到，但是目前我还没遇到）。所以我就用 Google Apps 为例了。怎么使用 prosody 我也不会，我的 bundle 里也没有 prosody 的依赖和主包，估计 openshift 在你不绑定域名（见我搭建 wordpress 那篇文章）的情况下也跑不起来（因为要设 SRV 记录，红帽提供的二级域名肯定没这功能）。

注册 Google Apps，需要你有一个个人持有的域名。这个教学我省略了，因为太多了。只讲几个新建账户的注意事项：

你建完账户是 temporary password，可以在该账户的明细里看到，首次登入是会要你改的，所以不要建完就去配置了，那样不会成功。一定要登入改密码。

如果你用 Google Apps 弄过 Gtalk 就知道需要[设置一个 DNS 的 SRV 记录](http://support.google.com/a/bin/answer.py?hl=zh-Hans&answer=34143)，才能在非 Google 的客户端比如 Pidgin 上登入。但是这段 SRV 记录并不全，你之所以能在 Pidgin 上登入是你告诉了 Pidgin 我这就是 Google Talk。xmpptalk 是不知道的，它会一直 ping 你的域名的 A 记录，但那上面通常都是个博客或者别的什么，肯定不是 xmpp 服务器。所以你要多加下面一段 SRV 记录，和 Google 给的一起，加到你的 DNS 服务商比如 cloudns.net（不是每个 DNS 服务商都支持用 SRV 记录，至少我知道的免费的就这一个，反正你的域名注册商甚至共享主机商提供的 DNS 都不行），至于怎么改域名的 DNS 请自行搜索。

    _xmpp-client._tcp.你的域名. IN SRV 5 0 5222 xmpp-server.l.google.com.
    _xmpp-client._tcp.你的域名. IN SRV 20 0 5222 alt1.xmpp-server.l.google.com.
    _xmpp-client._tcp.你的域名. IN SRV 20 0 5222 alt2.xmpp-server.l.google.com.
    _xmpp-client._tcp.你的域名. IN SRV 20 0 5222 alt3.xmpp-server.l.google.com.
    _xmpp-client._tcp.你的域名. IN SRV 20 0 5222 alt4.xmpp-server.l.google.com.

至于各项的意思，搜索一下就能明白，反正 cloudns 是蛮简单的，就顺序填空…

等 DNS 刷新好了之后，然后请把两个账户相互加为好友。不然到了 ircbindxmpp 的时候，由于不是好友，转发机器人没法把 IRC 说的话转发到 Gtalk 群聊里去。

这样你的账户就准备好了。我们开始配置。

#### 开始配置

我们需要改两个配置文件：site-packages 中的 xmpptalk/config.py 和 ircbindxmpp/config.py。没有就把 config.py.example 复制一下：

    cp -r config.py.example config.py

下面先说 xmpptalk。

    TODO: your bot's JID： 这个是要写群聊机器人使用的账户，比如 talk@suse.org.cn/bot
    TODO: your JID：这个是管理员的 Gtalk 账户，比如我的 marguerite@opensuse.org
    TODO: type a random string below：这个需要一个随机字符串，可以用 cat /dev/urandom | tr -dc A-Za-z0-9_ | fold -w 12 | head -5 得到一堆。
    nickchangeinterval = datetime.timedelta(days=10)：这个是群成员改昵称的间隔，默认是 10 天。
    TODO: select a database：这里唯一需要改的是 host，不能用 localhost，可以用 echo $OPENSHIFT_INTERNAL_IP 得到一个 IP 地址，用它。
    warnv105 = True：这个改成 False，不然用 Gtalk 105 版的就会持续收到警告说你的客户端不安全之类的，但问题我们群里有台湾地区的用户，我们面临的问题人家不需要。
    TODO: the password of your bot：群聊机器人的密码。

这样就好了。其它的不用管（如果你用 Prosody 的话可能还需要管一个 ~~# Which IP to connect?~~）。[Update] 依云酱说一般不需要，这个是给 Prosody 不支持 IPv6 准备的，一般填也填 IPv6 地址。

然后你还需要新建一个 xmpptalk/mongodb.conf，指定数据库的存放位置和日志的存放位置，完整见下：

    dbpath=/var/lib/openshift/513c7ceb5973cab51b0002c5/app-root/runtime/data/openshift-xmpptalk-ircbindxmpp-bundle/data
    logpath=/var/lib/openshift/513c7ceb5973cab51b0002c5/app-root/runtime/data/openshift-xmpptalk-ircbindxmpp-bundle/log/mongodb.log
    logappend=true
    #auth = true
    smallfiles = true
    nojournal = true
    bind_ip = 127.5.76.1

其中 `/var/lib/openshift/513c7ceb5973cab51b0002c5/app-root/runtime/data/` 就是 `$OPENSHIFT_DATA_DIR`，可以用 `echo $OPENSHIFT_DATA_DIR` 得到，`openshift-xmpptalk-ircbindxmpp-bundle` 是我预制的那个 bundle（在 git 上），data 和 log 是我自己建的。那个 bind_ip 就是前面刚讲的 `$OPENSHIFT_INTERNAL_IP`。在配置文件中是不能使用 openshfit 预定义的宏的，因为使用这个配置的程序读不出。

然后新建一个 start_mongo.sh 来启动 mongod 的 daemon，内容如下：

    mongod --config ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/xmpptalk/mongodb.conf --fork

那个 --config 跟的也是 mongodb.conf 所在的完整路径。

ircbindxmpp 的配置：

    targetJID = JID('talk@suse.org.cn')：这是你的群聊机器人的账户
    BotJID = JID('irc@suse.org.cn/bot')：这是你的转发机器人的账户
    'channel': 'opensuse-cn',：你要互通的 IRC 频道
    'nick': 'SuSE-Lady'：机器人在频道上的名字
    'realname': 'Gtalk Bot'：机器人在频道说话时显示的名字，我们互通 Gtalk 自然叫 Gtalk
    password = ''：你的转发机器人的密码。

#### 运行

export 环境变量（如果你已经登出了的话）。

进入 xmpptalk 文件夹：

    // 启动数据库
    chmod +x start_mongo.sh
    ./start_mongo.sh
    // 初始化数据库
    python3 dbman.py
    // 运行群聊机器人
    nohup python3 main.py > ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/xmpptalk.log 2>&1 &

其中 nohup 是不让 python3 进程一直占用你的 shell，后面那一套是把返回在 shell 屏幕上的 runtime 日志给重定向到一个文件中去。不然的话，在你 ssh 退出之后，这些 python3 进程会被 openshift 杀死，你的群就掉线了。

这时您就可以加群聊机器人与其它人通过机器人聊天了。

进去 ircbindxmpp 文件夹：

    // 运行 IRC 转发机器人
    nohup python3 ircbindxmpp > ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/ircbindxmpp.log 2>&1 &

原理同上。这时您就可以在 IRC 频道里看到你的机器人上线了。

停止机器人：

    killall python3

简单粗暴但行之有效。

你也可以做个重启脚本 restart.sh 放在根目录下面（一般 mongodb 跑起来就不用我们去管它了，所以不包括这个）：

    export PATH=${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/bin:$PATH
    export LD_LIBRARY_PATH=${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib:$LD_LIBRARY_PATH
    killall python3
    #killall mongod
    #mongod --config ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/xmpptalk/mongodb.conf --fork
    nohup python3 ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/xmpptalk/main.py > ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/xmpptalk.log 2>&1 &
    nohup python3 ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/ircbindxmpp/ircbindxmpp > ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/ircbindxmpp.log 2>&1 &

uncomment mongod 那两行可以连 mongod 一起重启。

#### 特殊注意：

是不是以为万事大吉了？还没呢…红帽说了，后台程序（在网页访问不到的），两天就会被 idle。也就是说两天后两个机器人就都掉线了。怎么办呢？

`Do-it-yourself` 应用的 Git（就是网页上让你 clone 的那个 URL）里面有一个 diy 文件夹，里面是一个 ruby 写的超小网页服务器，我们把那个跑起来，不就有一个非后台应用在跑了么？

    git clone ssh://*************@talk-**********.rhcloud.com/~/git/talk.git/

~~然后啥也不用干，直接 touch 个什么东西，提交回去就行了：~~ [Update] 要改 ./openshift/action_hooks/{start,stop}，不然提交的时候会停掉服务器上正在跑的程序只跑一个 ruby 的网页服务器。

    cd talk/.openshift/action_hooks/

在 start 尾部添加以下几行：

    export PATH=${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/bin:$PATH
    export LD_LIBRARY_PATH=${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib:$LD_LIBRARY_PATH
    nohup python3 ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/xmpptalk/main.py > ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/xmpptalk.log 2>&1 &
    nohup python3 ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/ircbindxmpp/ircbindxmpp > ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/ircbindxmpp.log 2>&1 &

在 stop 里面的 exit 0 上面插入一行：

    killall python3

然后提交：

    git add .
    git commit -m "initial commit"
    git push

然后 `.openshift/action_hooks/start` 就会在服务器跑，就会把那个网页服务器跑起来了。

#### 额外

###### Prosody

这个确实不懂。不过需要提醒大家注意的一点是，如果你打算在 openshift 上编译并搭建它，你需要了解 openshift 不是每个端口都可以对外的。大部分你熟悉的端口都已经[在内外两边都为官方应用保留了](https://openshift.redhat.com/community/kb/kb-e1038-i-cant-bind-to-a-port)，内部你只可以用 15000 – 35530，对外只可以 bind 到 `${OPENSHIFT_INTERNAL_PORT}` 也就是 8080，会通过 80 端口转发给外部。

###### 群日志

在 mongodb 数据库里。可以用 mongoexport 导出。

[Update] 今天他们非要一个 IRC 日志，我就直接把日志做成 cron jobs 放到我们之前跑起来的那个 ruby 网页服务器上了。

[Update1] 他们非要 HTML 格式的日志。好吧：[https://talk-marguerite.rhcloud.com/index.html](https://talk-marguerite.rhcloud.com/index.html)

把以下内容保存为 log_to_web.sh，放在 .openshfit/cron/hourly 并 `chmod +x` 后提交：

    ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/bin/mongoexport --host ${OPENSHIFT_INTERNAL_IP}:27017 -d talk -c log --csv -f time,jid,msg -o ${OPENSHIFT_REPO_DIR}/diy/chat.txt
    ${OPENSHIFT_DATA_DIR}/text2html.sh ${OPENSHIFT_REPO_DIR}/diy/chat.txt > ${OPENSHIFT_REPO_DIR}/diy/index.html
    rm -rf ${OPENSHIFT_REPO_DIR}/diy/chat.txt
    killall ruby
    nohup $OPENSHIFT_REPO_DIR/diy/testrubyserver.rb $OPENSHIFT_INTERNAL_IP $OPENSHIFT_REPO_DIR/diy > $OPENSHIFT_HOMEDIR/diy-0.1/logs/diy_server.log 2>&1 &

解释下就是导出 log，转为 HTML，然后重启网页服务器生效。

然后把以下内容保存为 text2html.sh，放在 $OPENSHIFT_DATA_DIR 下面然后 chmod +x。

<pre>
    #!/bin/bash
    echo "
            <!DOCTYPE html>
            <html>
                <head>
                    <meta charset='utf-8'>
                    <title>`date -R`</title>
                    <style>
                        body {
                            background: #dbe1ed;
                            font-family: "WenQuanYi Micro Hei",sans-serif;
                            font-size: 16px;
                            color: #333;
                        }
                        #content {
                            max-width: 900px;
                            margin: 5px auto;
                        }
                        p {
                            border: 1px solid #aaa;
                            padding: 2px 10px;
                            border-radius: 8px;
                            -moz-border-radius: 8px;
                            box-shadow: 0 8px 6px -6px #999;
                            line-height: 150%;
                        }
                        p:nth-child(odd) {
                            background: #92d841;
                            text-shadow: 0 1px 0 #adff4d;
                        }
                        p:nth-child(even) {
                            background: #d3d3d3;
                            text-shadow: 0 1px 0 #fff;
                        }
                        h2 {
                            margin: 5px auto;
                            text-shadow: 0 1px 0 #fff;
                            color: #555;
                            max-width: 900px;
                            text-align: center;
                        }
                        span {
                            display: block;
                            font-size: 11px;
                            color: #999;
                            magin: 0 auto 15px auto;
                            width: 900px;
                            text-align: center;
                            line-height: 200%;
                            text-shadow: 0 1px 0 #fff;
                        }
                    </style>
                </head>
            <body>"
    echo '<h2>openSUSE 中文社團聊天記錄'
    echo "`date -R`"
    echo '</h2><div id='content'>'
    cat "$1" | while read p; do
            echo "<p>"
            echo $p
            echo "</p>"
    done
    echo '</div>
                    <span>2013 版權所有 suse.ws。這是 openSUSE 中文社區的官方聊天室。</span>
                    <span>添加 talk@suse.ws 爲 Gtalk 好友或 IRC 加入 #opensuse-cn 頻道即可參與。</span>
            </body>
        </html>'
</pre>

###### 备份方案

你当然可以使用我在之前搭建 wordpress 的文章里讲的 snapshot 方法。不过下面介绍一种自动的：使用 [Dropbox-Uploader](https://github.com/andreafabrizi/Dropbox-Uploader) 配合 openshift 的 Cron Cartridge。

首先在网页版的 My Applications 里点进去，然后进入你的 Do-it-yourself 应用，点 add cartridge，加一个 Cron 1.4 的 cartridge。

然后 ssh 进你的应用程序：

    cd $OPENSHIFT_DATA_DIR
    git clone https://github.com/andreafabrizi/Dropbox-Uploader

然后把里面那个脚本复制到上级目录，文件夹可以删了。需要修改一个地方：

    sed -i "s/CONFIG_FILE=~/CONFIG_FILE=\./" ./dropbox_uploader.sh

不然会提示权限不够。再：

    chmod +x dropbox_uploader.sh
    ./dropbox_uploader.sh

一次，跟随设置。这样以后你就可以 ./dropbox_uploader.sh upload 文件 了。

先备份一下我们整个的 openshift-xmpptalk-ircbindxmpp-bundle 吧。注意，以下所有脚本都是在 ${OPENSHIFT_DATA_DIR} 运行的。这是 dump_all.sh 脚本的内容：

    pushd ${OPENSHIFT_DATA_DIR}
    # 压缩整个 openshift-xmpptalk-ircbindxmpp-bundle 文件夹
    tar -cjvf openshift-xmpptalk-ircbindxmpp-bundle-`date +%Y-%m-%d`.tar.bz2 openshift-xmpptalk-ircbindxmpp-bundle/
    # 上传到 Dropbox
    ./dropbox_uploader.sh upload `ls | grep tar.bz2`
    # 删除 tar
    rm -rf openshift-xmpptalk-ircbindxmpp-bundle-*.tar.bz2
    popd

然后我们编写一个 backup_data.sh 备份脚本，需要备份的有数据库，config.py，mongodb.conf，和 data 与 log 文件夹，以及 mongodump 导出的群消息日志：

    # 备份 mongodb
    ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/bin/mongodump --host ${OPENSHIFT_INTERNAL_IP}:27017 -d talk -o ${OPENSHIFT_DATA_DIR}/backup/database/
    # 导出聊天记录
    ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/bin/mongoexport --host ${OPENSHIFT_INTERNAL_IP}:27017 -d talk -c log --csv -f time,jid,msg -o ${OPENSHIFT_DATA_DIR}/backup/chatlog.csv
    # 备份 config(s)
    mkdir -p ${OPENSHIFT_DATA_DIR}/backup/config/
    cp -r ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/xmpptalk/config.py ${OPENSHIFT_DATA_DIR}/backup/config/xmpptalk.config.py
    cp -r ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/ircbindxmpp/config.py ${OPENSHIFT_DATA_DIR}/backup/config/ircbindxmpp.config.py
    cp -r ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/lib/python3.3/site-packages/xmpptalk/mongodb.conf ${OPENSHIFT_DATA_DIR}/backup/config/
    # 备份 log(s)
    mkdir -p ${OPENSHIFT_DATA_DIR}/backup/log/
    cp -r ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/log/* ${OPENSHIFT_DATA_DIR}/backup/log/
    # 备份裸数据库
    mkdir -p ${OPENSHIFT_DATA_DIR}/backup/raw/
    cp -r ${OPENSHIFT_DATA_DIR}/openshift-xmpptalk-ircbindxmpp-bundle/data/* ${OPENSHIFT_DATA_DIR}/backup/raw/
    # 改名
    mv ${OPENSHIFT_DATA_DIR}/backup ${OPENSHIFT_DATA_DIR}/backup-`date +%Y-%m-%d-%H_%M_%S`
    pushd ${OPENSHIFT_DATA_DIR}
    tar -cjf `ls | grep backup-20`.tar.bz2 backup-*/
    # 上传
    ./dropbox_uploader.sh upload *.tar.bz2
    # 清理
    rm -rf backup-20*
    popd

现在是 cron 设置了。还记得刚才你弄的那个 talk 的 Git 吗？用那个去搞（以下命令是在本地的）：

    cd talk
    cd .openshift/cron

能够看到下面有 minutely 什么的这样的文件夹。openshift 的 cron 就是把脚本扔到这样的文件夹里，就会按照文件夹名所说的时间来跑。我们的 dump_all.sh 一个月执行一次就好了，而 backup_data.sh 可以每天执行一次。于是就把那两个脚本也放到相应的文件夹里去（服务器上的是给你手动跑的），不要忘记 chmod +x 哦！然后：

     git add .
    git commit -m "scheduled backup"
    git push

完成，enjoy！
呜谢

* 依云酱
* douglarek

