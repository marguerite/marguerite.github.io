---
title: "[Solved] btrfs 开机 initramfs 加载时间过长"
date: 2012-10-20T00:00:00+08:00
draft: false
---
这是困扰了我一年半的一个 bug，昨晚被 btrfs ML 的 cwillu 几封邮件解决了。

最早开始使用 btrfs 是在 10 年 11 月，那时我换上了听歌比 ssd 更有质感的 1TB 西数 7200 转机械硬盘，重装系统就装了那之前炒作的沸沸扬扬的「下一代文件系统」btrfs。不得不说，很少有人写文章像我这么求实了。「下一代文件系统」，那不就是「future never actually comes！」吗？！刚装机就遇到了一个 serious bug，那就是 btrfs 的几个进程 100% cpu 使用率，基本幻灯片。怀疑过 chromium，firefox 的数据库导致碎片，反正怀疑了一大堆，最终是升级到 3.2 莫名其妙的解决了。（后来 OBS 又在这上面吃了一回屎，12.2 发布前几周，RC 2 的时候，升级服务器换了 btrfs，结果卡的 scheduler 一动不动，编译服务变成了一块大网盘，最终跳掉了 RC3）

这次的这个 bug 是 12.2 后带来的。我之前的 RC 版本没注意，但根据我编辑维基的[记录](http://en.opensuse.org/openSUSE:Most_annoying_bugs_12.2_dev)，是在 RC 2 出现的。当时是因为我看到了 lizards 的一篇小短文，叫做「openSUSE 2 秒启动」，本来想试做然后发到 suse.ws 的。结果人家全部两秒，我的 NetworkManager 就要 80000+ms 才能启动。这不是坑人么。于是找到 NM 的维护者 BinLi 报了 bug。

中间很曲折，BinLi 无法复现，后来有个用户说这 bug 他也有，重装改成 ext4 好了。可是我用 Linux 是木有重装机这个概念的。因为垃圾我能清理掉（Windows 完全清不掉！），Linux 又稳定，我自己是打包者也没有「/usr 污染」，于是这台电脑上攒了好多的资料。比如 300GB 的电影。。。

后来我发现，添加 `comment="systemd.automount"` 到 /etc/fstab 的 btrfs 条目后，NM 的时间变正常了。当时唯一剩下的问题就是，我的内核在接通电源后要 135s 才能加载上，也就是说你们都已经见到 KDM 了我还在 Plymouth！

后来内核组的 David Sterba 乱入说这是个 3.4 的已知 bug，要清理缓存（clear cache）。但是添加了 clear_cache 到 /etc/fstab 的 btrfs 条目，重启，删除它，再重启两次让 readhead 生效后（systemd 下解决 bug 总是要重启两次！第一次什么都看不出来！），发现问题依旧。David 就没办法了。

可是系统是我的啊！日子总还得过啊！于是查阅了各种 gentoo/arch 维基，btrfs 官方维基，一点头绪都没有，甚至使用 3.6.2 内核配合 mason 的 btrfs-git 版本统统不管用。最终逼到了官方邮件列表。

mason 让我 `sysrq-w snapshot`。。。结果我什么都不懂，人家嫌我傻，不和我说话。赶紧到 #suse IRC 频道找人学习 magic sysrq 命令。回去跟人家贴命令，这时 cwillu 来了。

一眼认出我的 btrfs 没有 space_cache。什么是 space_cache？

space_cache，意思就是说 btrfs 在挂载前要扫描整个硬盘来预热需要的信息（超慢），用了它呢，直接使用上次卸载的数据来做。

bug 解决：

    UUID=9b9aa9d9-760e-445c-a0ab-68e102d9f02e / btrfs defaults,space_cache,comment=systemd.automount 1 0
    UUID=559dec06-4fd0-47c1-97b8-cc4fa6153fa0 /home btrfs defaults,space_cache,comment=systemd.automount 1 0

UUID 用你的，别的都替换成我这样。

真正的成因就是 David 所说的，3.4 内核之后的一个已知 bug 需要重建缓存（rebuild cache），但是 openSUSE 就完全没用缓存，所以 clear_cache 没用，要用 space_cache 来初始化建立缓存。像我一样问题的，或者用 3.4+ 内核配合 btrfs 的同学们有福了。。。毕竟这问题比较难办，我调动了 SuSE 的资源（难听点说就是卖萌）都无解。。。。有则改之无则加勉吧！

当然，本文省略了很多信息，比如 btrfs 的调试和报 bug 需要提供的信息，以及一些我久病成医看到的已知问题和解决方案。这些都会写到 [http://zh.opensuse.org/openSUSE:Btrfs](https://web.archive.org/web/20130309075037/http://zh.opensuse.org/openSUSE:Btrfs) 中去（现在还没写）。相信我这么不离不弃几乎遇到过 btrfs 诞生至今所有的知名 bug 的人士会把这维基写的很 up to date 很 powerful 的。

openSUSE 的同学还请期待学姐的「两秒光速启动」调教系列哦～

参考文献：

* [bnc#776563](https://bugzilla.novell.com/show_bug.cgi?id=776563)
* [linux-btrfs ML](http://www.spinics.net/lists/linux-btrfs/msg19766.html)
