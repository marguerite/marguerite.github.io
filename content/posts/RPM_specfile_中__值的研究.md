---
title: "RPM specfile 中  值的研究"
date: 2015-08-02T00:00:00+08:00
draft: false
---
研究这个问题是因为论坛上的一个讨论：[zypper reinstall package](https://forum.suse.org.cn/viewtopic.php?f=15&t=4004/)

略过楼主的实现是否正确不谈（因为他不给我看 specfile 估计是公司的软件），简单来说可以分成几个子问题：

第一个是 RPM specfile 的 %postun 部分的 $1 变量到底是什么：

    if [ $1 == 0 ] ; then

$1 的值当然是有意义的，它代表安装在你系统上的同名软件包的版本数。比如你升级软件包，那默认是 1，但在某个状态是 2，因为这时新包装上旧包还没卸载，0 就代表这个包在你系统上已经不存在了。

http://stackoverflow.com/questions/7398834/rpm-upgrade-uninstalls-the-rpm

第二个是 RPM reinstall 的作业流程是怎样的，是遵循升级的流程也就是：

软件包升级的过程是先安装新包后卸载旧包，只有来自旧包的差异文件会被删除。也就是 pre -> install -> post; preun -> uninstall -> postun

还是另有一个流程。$1 值的变化可以推测出这个问题的答案。

第三个是使用 zypper 和 rpm 分别进行重装，$1 值的变动是不是一致的。

这个问题我们需要做一个实证，代码在这里：

https://build.opensuse.org/package/show/home:MargueriteSu/rpm-reinstall-demo

相应的测试用 RPM 也可以在那边取得。下面直接上结果：

    sudo zypper --no-refresh install --force rpm-reinstall-demo-0.0.0-5.1.x86_64.rpm

    drwxr-xr-x 1 root       root     0 8 月   2 22:25 demo

通过 /var/tmp/demo 的时间戳记可以发现安装前后 demo 没有被删除重建。

    sudo rpm --install --replacepkgs rpm-reinstall-demo-0.0.0-5.1.x86_64.rpm

时间戳记还是没变。

    sudo rpm --reinstall rpm-reinstall-demo-0.0.0-5.1.x86_64.rpm

这个时间戳记也没变，但是输出了 $1 的值为 1，也就是说触发了 %postun 过程。之前 rpm –install –replacepkgs 是没有输出的。至于 zypper 我怀疑可能是被压制了输出 :-(

所以结论如下：

估计那位楼主说的重装可能是卸载了再安装，那样 $1 肯定会为 0, 会触发删除 /var/tmp/demo 的操作。

正常的 zypper 重装和 rpm 重装（两种方法）都是不会使 $1 为 0 的，也就是说 /var/tmp/demo 不会被删除掉。也就是说 if [ $1 == 0 ] ; then 的真正作用是为了确保软件包从系统删除时它的一些临时文件会被删除、做出的修改会被回滚等。

使用 rpm –install –replacepkgs 方法重装，不会触发 %post %postun %pre %preun 等过程。

使用 rpm –reinstall 重装，会触发那些过程，但是 $1 值不会为 2, 也不会为 0, 也就是说遵循的依然是升级的流程：先安装新包再卸载旧包

至于 zypper，且不管它是否被压制输出吧，它有 --oldpackage 选项，所以看两者的 --help 我觉得 libzypp 后端使用的应该是 rpm --install --replacepkgs 方法。这个真正测试起来也很简单，在 %post 阶段往 /tmp 里写个文件就好了。安装完毕先手动删除掉，如果重装后发现了这个文件，那么它的 reinstall 就是 rpm --reinstall 方法（因为调用了 %post），否则就是 rpm --install --replacepkgs 方法。

PS:以后博客就用 hexo 来写了。以前 wordpress 的文章换服务器丢了一部分，找到了一部分，有时间会慢慢转换过来。
