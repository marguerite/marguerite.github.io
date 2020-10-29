---
title: "openSUSE 下玩围棋"
date: 2013-02-24T00:00:00+08:00
draft: false
---
Update：更新了评论里两位巨巨指正的技术错误和建议到正文中。

主要是整理下 Linux 下面的围棋软件近况、设置方法和已知故障。

#### 引擎

GNUGo 这个是所有围棋软件的后端，简单说就是个棋力十二级的机器人。

不是很明白这里的电脑棋力单位“级”与我们人类的棋力单位“段”的换算关系，因为电脑肯定没法参加人类的段位考试，而两者也很少对弈，围棋引擎很奇怪的，中低段位的人类和引擎对弈，基本很难赢。它的思考方法和这些段位的人类很不一样，比如人类会在开局考虑一些谋篇布局，而电脑只有那几个布局定式，根据你落子的位置优化选出一个; 人类在中盘会考虑大局，而电脑除非你不与它对抗（它就按照定式走），如果出现对抗那它的算法就是怎么堵截你在数学上最优，而人类肯定不是这样，有些禁忌位也一样有人放，即所谓的“妙手”、“妖刀”，当然更多的还是“俗手”、“蠢手”。而高段位的人类和引擎对弈中，引擎被虐的很惨，还不如三段的人类去下，因为高段位的变化很多，思考质量也不亚于电脑。

围棋引擎流行的能跑在 Linux 下的有这些：[Fuego](http://fuego.sourceforge.net/)（棋力 9 段，9×9 棋盘）, [GNU Go](http://www.gnu.org/software/gnugo/devel.html)（棋力 12 级）, [MoGo](https://www.lri.fr/~teytaud/mogo.html), [Pachi](http://pachi.or.cz/)（棋力 4 段），其中开源最强的是 Fuego（因为闭源引擎用的多数不是蒙地卡罗方法，强是应该的）。

当然这些强弱是对有几年棋龄的业余棋手而言的，我们这种入门小菜哪个都下不过，随意选个就开始好了。一般发行版都会带 GNUGo。我们就用它了。

另外关于围棋程序比较好的网站是：[Sensei's Library](http://senseis.xmp.net/?GoPlayingPrograms)，有各种引擎比较详细的介绍和测评，至于界面，还能用的我都在下面整理出来了，完整列表见此。
界面

* [gGo](http://www.pandanet.co.jp/java/gGo/) Pandanet 电脑围棋界比较有名的熊猫网出的。Java 写的，最新版本发布于 2004 年。
* [RubyGo](http://rubygo.rubyforge.org/) Ruby 写的，最新版本发布于 2005 年。
* [qGo](http://qgo.sourceforge.net/) Qt3 写的，最新版本发布于 2008 年。
* [CGoban](http://www.gokgs.com/help/app/main.html) 这其实是围棋网 KGS 的官方客户端。Java 写的，版本也比较老。
* [glGo](http://www.pandanet.co.jp/English/glgo/) Pandanet 出的，最后更新于 2008 年，但是依赖很烦，依然依赖 python 2.5。没有源代码（网页打不开），非开源软件。

这些都是老黄历了，网上宣传的基本也都是这些，因为这是 IGS 或者 KGS 这些业内比较知名的组织开发的。但是在现今的 Linux 上都比较难跑。我整理了一份比较好跑的界面：

* [qGo2](https://github.com/pzorin/qgo) 这是 qGo 的 Qt4 移植，可以说几多波折，开发者换了一个又一个，因为它太强大了，支持这些战网：IGS, WING, LGS, CyberORO, Tygem, Tom, and eWeiqi 的对弈和观战（看好了后面两个是中国的），还是目前最强大的 sgf（通用围棋棋谱格式，用于复盘和自己打谱）编辑器。
* [quarry](http://gitorious.org/quarry) 这是一个本地围棋和五子棋客户端。用 GNUGo 或 GRhino 做后端。配置 GTP 后端的时候要加 -mode gtp --quiet 参数才能把 /usr/bin/gnugo 跑起来。
* [kigo](http://www.kde.org/applications/games/kigo/) 只用于 KDE，自动识别后端。我目前用这个。但是有一个 Bug 就是电脑不会认输，走不下去了就一直下虚手（跳过），需要你手动结束游戏才能统计“目”。

其中前两者我都有打包，但是还没有推送到 openSUSE 的 games 源，迫不及待的话可以用我的 [home:MargueriteSu:branches:M17N](https://build.opensuse.org/project/show?project=home%3AMargueriteSu%3Abranches%3AM17N) 源里的 RPM。

以下是 Java 的，我不喜欢，但是肯定能用，也还在开发：

* [GoGui](http://gogui.sourceforge.net/) 本地客户端，支持 GNUGo。
* [Jago](http://jagoclient.sourceforge.net/)。只能在线上 IGS，没法玩机器人。

我是 KDE 所以不是很理 GNOME，似乎 GNOME 可以用这个：

* [ccGo](http://ccdw.org/~cjj/prog/ccgo/)。Gtkmm 2.4 写的，最后更新于 2010 年。

#### 在线

以上主要满足的是你打谱和被电脑虐的需求，有些时候新手还是需要和人类进行一些交流的，不然被电脑虐出来的棋风比较诡异。两个在线网站，都需要 Java：

* [日本雅虎](http://games.yahoo.co.jp/) 注册方法见[这个帖子](http://forum.ubuntu.org.cn/viewtopic.php?f=34&t=290181)，但是日本围棋比我们流行，棋力都很彪悍。
* [Pandanet](http://www.pandanet-igs.com/communities/gopanda2) 熊猫网，最顶级的，什么人都有，欧洲人主要都是些物理学家什么的，逻辑能力特别强，也比较难对付。你可以观战然后下他们的 sgf 自己复盘。
* [Peepo](http://www.peepo.com/) Pachi 引擎的前端网站。
* [天元围棋](http://www.twgo.net/) 这是台湾同胞的围棋网站，可以用 wine 跑起来和他们玩。
* [无争](http://www.wuzheng.me/) 楼下巨巨推荐的在线慢棋网，日本雅虎下的太快了（因为日本人赶时间）。

#### Android/iPad 客户端

* [Panda-Tetsuki](http://www.gentgo.be/tetsuki/)。这个是最好的手机客户端，没有之一。可以上战网也可以自己玩。
* [Go Free](https://play.google.com/store/apps/details?id=uk.co.aifactory.gofree&hl=zh_CN)。显示是核心开发者开发的，除了电脑傻别的什么都很好=。=
* [飞燕围棋](http://swiftgo.duapp.com/)。楼下巨巨推荐的，看了下确实蛮好，可惜桌面客户端是 Java 而且不继续开发了，而且 apk 没有进 Google Play，但是没关系，我们可以把 SGF 棋谱下载回来用 qGo2 打谱。

#### 教材

Google Play 有一个叫“围棋教室”的小软件，用它一晚上就能入门了。比一些围棋书还快。强烈推荐。

电子书一般去 [SimpleCD](http://simplecd.me/) 上面找，但是问题就是，因为中国是发源地嘛，中文书一般偏难，能出书的都是九段起步了，书中选的案例也都是九段比赛，而日文书选的案例一般也是棋圣战这样的水平，你根本看不懂。也就是说人家根本没有考虑我们这些小菜的需求。规则都不懂就上大师傅对你们都是煎熬，小学学围棋那种情绪都还记得吧。

我学的时候妈妈推荐我上的这个网站 [Life in 19×19](http://www.lifein19x19.com/forum/viewforum.php?f=6)。听名字你就知道是个老外的围棋网站，里面的 instruction 都是面向没有经过五千年文化熏陶的野蛮人的，非常适合二逼青年入门（连英语都学了），当然回头你还是要看一些基础的中文围棋书，不然你连什么是“打”、“提”、“冲”都不知道，平时也没法跟领导下棋是不是？

至于死活题也一样，尽量别用中文的什么段位考试题啊、死活 5000 题啊，那些都太难了，不适用于娱乐需求。去 [Go Problems](http://www.goproblems.com/)，这上面的死活题简单，可以娱乐使用。楼下巨巨也推荐了一个 [Go Game Guru](http://gogameguru.com/)，特点是网站特别漂亮…

楼下的两位巨巨都推荐用日本棋院的书入门，俺娘说俺太菜不适合，不过还是放出来：

[围棋吧](http://www.weiqibar.com/forum.php?mod=forumdisplay&fid=40)的电子棋书版块。

#### 精神

昭和棋圣 · 吴清源的《中的精神》必须看。要领会围棋是一种中和的精神，不争是非，清静无为。所以那些在贴吧秀 GNUGo 的棋力如何弱爆了的下的都是伪棋，厉害点的算个将棋，不厉害的那就把围棋下成兵棋推演了。我们要下王道之棋，娱乐为主。

#### 结

总之本篇算是一个 Linux 下如何入门围棋以及自娱自乐的教程吧。Enjoy！
